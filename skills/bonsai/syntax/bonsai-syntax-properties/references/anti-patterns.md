# Property Set Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+
> Common mistakes when working with IFC property sets in Bonsai and IfcOpenShell.

---

## Anti-Pattern 1: Modifying Property Values Directly

### WRONG

```python
# NEVER modify property NominalValue directly
pset = model.by_type("IfcPropertySet")[0]
for prop in pset.HasProperties:
    if prop.Name == "FireRating":
        prop.NominalValue = model.create_entity("IfcLabel", "REI120")  # WRONG
```

### WHY IT FAILS

- Bypasses pset template validation (type checking against buildingSMART standards)
- Bypasses automatic Python-to-IFC type inference
- Bypasses shared property unsharing (a property shared by multiple psets gets modified for ALL of them)
- Bypasses owner history updates
- Does NOT trigger Bonsai UI refresh

### CORRECT

```python
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "FireRating": "REI120",
})
```

---

## Anti-Pattern 2: Using `Pset_` Prefix for Custom Property Sets

### WRONG

```python
# NEVER use Pset_ prefix for custom property sets
custom = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_CompanyFireData")  # WRONG: Pset_ is reserved
```

### WHY IT FAILS

- `Pset_` prefix is reserved for buildingSMART international standards
- IFC validators flag custom psets with `Pset_` prefix as non-conforming
- Template applicability checks fail because `Pset_CompanyFireData` is not a recognized standard pset
- Bonsai's `is_pset_applicable()` may reject the pset for the element class

### CORRECT

```python
# Use company or project prefix for custom psets
custom = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Acme_FireData")
```

Same rule applies to `Qto_` prefix — NEVER use it for custom quantity sets.

---

## Anti-Pattern 3: Expecting `None` Quantities to Be Kept

### WRONG

```python
# WRONG: expecting None to clear the value but keep the quantity entity
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Height": None,       # This REMOVES the quantity, not clears it
}, should_purge=False)    # ERROR: edit_qto has NO should_purge parameter
```

### WHY IT FAILS

- `edit_qto` does NOT have a `should_purge` parameter (unlike `edit_pset`)
- Quantities set to `None` are ALWAYS removed from the model
- IFC standard does not allow quantity entities with no value
- Passing `should_purge` as a keyword argument causes a `TypeError`

### CORRECT

```python
# Quantities with None are always removed
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Height": None,  # Quantity entity is deleted
})

# To update a quantity, set it to a new value
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Height": 0.0,  # Keeps entity with zero value
})
```

---

## Anti-Pattern 4: Traversing IsDefinedBy Manually to Read Properties

### WRONG

```python
# NEVER traverse relationships manually
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.Name == "Pset_WallCommon":
            for prop in pset.HasProperties:
                if prop.Name == "FireRating":
                    value = prop.NominalValue.wrappedValue  # WRONG
```

### WHY IT FAILS

- Does not handle type-inherited properties (`should_inherit`)
- Does not handle `IfcElementQuantity` vs `IfcPropertySet` distinction
- Does not handle `IfcPropertyEnumeratedValue` or other property subtypes
- Does not handle IFC2X3 vs IFC4 schema differences (`PropertyDefinitionOf` vs `DefinesOccurrence`)
- Verbose, error-prone, and schema-version-dependent

### CORRECT

```python
import ifcopenshell.util.element

# Single property value
fire_rating = ifcopenshell.util.element.get_pset(
    wall, "Pset_WallCommon", "FireRating")

# All properties in a pset
pset_data = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon")

# All psets and qtos
all_data = ifcopenshell.util.element.get_psets(wall)
```

---

## Anti-Pattern 5: Assigning Standard Psets to Wrong Element Classes

### WRONG

```python
# WRONG: Pset_WallCommon is only for IfcWall/IfcWallType
beam = model.by_type("IfcBeam")[0]
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=beam, name="Pset_WallCommon")  # Semantically wrong
```

### WHY IT FAILS

- Standard psets have defined applicability per IFC element class
- `Pset_WallCommon` is only valid for `IfcWall` and `IfcWallType`
- IFC validators flag non-applicable pset assignments
- Bonsai's `tool.Pset.is_pset_applicable()` returns `False` for this combination
- Downstream BIM tools may ignore or misinterpret the data

### CORRECT

```python
# Check applicability first
beam = model.by_type("IfcBeam")[0]

# Use the correct standard pset for the element class
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=beam, name="Pset_BeamCommon")

# Or use a custom pset (no applicability restriction)
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=beam, name="Acme_StructuralData")
```

---

## Anti-Pattern 6: Forgetting Idempotency of add_pset

### WRONG

```python
# WRONG: creating duplicate psets
pset1 = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset1, properties={
    "FireRating": "REI60",
})

# Later in the same code or different function:
pset2 = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
# pset2 IS pset1 — they are the same entity!

ifcopenshell.api.run("pset.edit_pset", model, pset=pset2, properties={
    "LoadBearing": True,
    # FireRating from earlier is still present (not lost)
})
```

### WHY IT MATTERS

- `add_pset` is idempotent: it returns the existing pset if one with the same name exists
- This is NOT a bug — it prevents duplicate psets
- But developers may not realize `pset2 is pset1` and accidentally assume a fresh pset
- Properties from the first `edit_pset` call are preserved, not overwritten

### CORRECT

```python
# Set all properties in a single edit_pset call when possible
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "FireRating": "REI60",
    "LoadBearing": True,
})
```

---

## Anti-Pattern 7: Using `model.remove()` to Delete Properties

### WRONG

```python
# NEVER use model.remove() for properties or psets
model.remove(pset)  # WRONG: leaves dangling IfcRelDefinesByProperties
model.remove(prop)  # WRONG: leaves dangling references in pset.HasProperties
```

### WHY IT FAILS

- `model.remove()` only deletes the entity instance, not its relationships
- Leaves orphaned `IfcRelDefinesByProperties` with invalid references
- Leaves orphaned `IfcPropertyEnumeration` entities
- Does NOT clean up `OwnerHistory` entities
- Produces an invalid IFC file

### CORRECT

```python
# Remove individual properties: set to None in edit_pset
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "OldProperty": None,  # Properly removed with should_purge=True
})

# Remove entire pset: use remove_pset
ifcopenshell.api.run("pset.remove_pset", model, product=wall, pset=pset)
```

---

## Anti-Pattern 8: Confusing Quantity Name Keywords

### WRONG

```python
# WRONG: name does not contain a recognized keyword
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Perimeter": 36.0,  # Defaults to IfcQuantityLength (no keyword match)
    "Value": 100.0,     # Defaults to IfcQuantityLength (no keyword match!)
})
```

### WHY IT MATTERS

- Quantity type is inferred from the property name, NOT the pset name
- `"Perimeter"` defaults to `IfcQuantityLength` (correct by coincidence since it contains no keyword)
- `"Value"` defaults to `IfcQuantityLength` even though it may represent area or volume
- The keyword check is case-insensitive on the property name
- Default for any unrecognized float name is `IfcQuantityLength`

### CORRECT

```python
# Use names that contain the correct type keyword
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Perimeter": 36.0,              # -> IfcQuantityLength (default, correct)
    "TotalExposedArea": 100.0,      # -> IfcQuantityArea (contains "area")
    "TotalConcreteVolume": 50.0,    # -> IfcQuantityVolume (contains "volume")
    "TotalSteelWeight": 1200.0,     # -> IfcQuantityWeight (contains "weight")
    "NumberOfPanels": 12,           # -> IfcQuantityCount (int value)
})
```

**Recognized keywords** (case-insensitive in property name):

| Category | Keywords |
|----------|----------|
| Area | "area" |
| Volume | "volume" |
| Weight | "weight", "mass" |
| Length | "length", "width", "height", "depth", "distance" |
| Time | "time", "duration" |
| Count | (any int value) |

---

## Anti-Pattern 9: Not Checking IfcStore for None in Bonsai Scripts

### WRONG

```python
# WRONG: no guard against missing IFC file
import bpy
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
walls = model.by_type("IfcWall")  # AttributeError if model is None
```

### WHY IT FAILS

- `IfcStore.get_file()` returns `None` when no IFC project is loaded
- Calling `.by_type()` on `None` raises `AttributeError`
- This is the #1 runtime error in Bonsai scripts

### CORRECT

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")

walls = model.by_type("IfcWall")
```

---

## Anti-Pattern 10: Editing Shared Psets Without Unsharing

### WRONG

```python
# WRONG: editing a shared pset affects ALL elements using it
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall1, name="Acme_Data")
ifcopenshell.api.run("pset.assign_pset", model,
    products=[wall2, wall3], pset=pset)

# This modifies the pset for wall1, wall2, AND wall3
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "SpecialValue": "only-for-wall1",  # Applied to ALL three walls!
})
```

### WHY IT MATTERS

- When a pset is shared via `IfcRelDefinesByProperties`, editing it affects all elements
- `edit_pset` handles individual shared property entities (unshares on `get_total_inverses > 1`), but the pset entity itself remains shared
- If you need different values per element, you must unshare first

### CORRECT

```python
# Unshare before making element-specific edits
copies = ifcopenshell.api.run("pset.unshare_pset", model,
    products=[wall1], pset=pset)

# Now wall1 has an independent pset copy
ifcopenshell.api.run("pset.edit_pset", model, pset=copies[0], properties={
    "SpecialValue": "only-for-wall1",
})
# wall2 and wall3 still share the original pset, unaffected
```

---

## Sources

- IfcOpenShell API source: `src/ifcopenshell-python/ifcopenshell/api/pset/`
- Bonsai pset module: `src/bonsai/bonsai/bim/module/pset/`
- Bonsai tool/pset: `src/bonsai/bonsai/tool/pset.py`
- OSArch Community: https://community.osarch.org
- IfcOpenShell GitHub Issues: https://github.com/IfcOpenShell/IfcOpenShell/issues
