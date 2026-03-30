---
name: bonsai-syntax-properties
description: >
  Use when creating, reading, or editing IFC property sets (Pset) and quantity sets (Qto)
  in Bonsai. Prevents the common mistake of creating property sets without associating them
  to elements via IfcRelDefinesByProperties. Covers IfcPropertySingleValue, IfcPropertyEnumeratedValue,
  IfcQuantityArea/Length/Volume, predefined pset templates, and bulk property management.
  Keywords: property set, Pset, Qto, IfcPropertySingleValue, IfcPropertyEnumeratedValue,
  IfcRelDefinesByProperties, custom properties, bulk property, pset template.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Property Set Management

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.bim.module.pset`, `bonsai.bim.module.pset_template`, `bonsai.bim.module.qto`
> **IFC API**: `ifcopenshell.api.pset`, `ifcopenshell.api.pset_template`

## Critical Warnings

1. **ALWAYS** use `ifcopenshell.api.run("pset.edit_pset", ...)` to modify properties. NEVER set `prop.NominalValue` directly — it bypasses template validation, type inference, and owner history.
2. **ALWAYS** pass `product` (singular) to `pset.add_pset` and `pset.add_qto`. Pass `products` (list) to `pset.assign_pset`.
3. **NEVER** use the `Pset_` or `Qto_` prefix for custom property sets — these are reserved for buildingSMART standards. Use a company or project prefix (e.g., `Acme_StructuralData`).
4. **ALWAYS** check pset applicability before assigning a standard pset. `Pset_WallCommon` only applies to `IfcWall`/`IfcWallType`, not to `IfcBeam`.
5. **NEVER** pass `None` values in `edit_qto` expecting them to be kept. Quantities set to `None` are ALWAYS removed — there is no `should_purge` parameter for quantity sets.
6. **ALWAYS** use `ifcopenshell.util.element.get_psets()` for reading. NEVER traverse `IsDefinedBy` relationships manually.
7. **ALWAYS** use `pset.add_pset()` before `pset.edit_pset()`. The add function is idempotent — it returns the existing pset if one with the same name already exists.
8. **NEVER** assume `bool` maps to `IfcInteger`. Python `bool` is a subclass of `int`, but IfcOpenShell checks `bool` first and maps it to `IfcBoolean`.

## Decision Tree: Property Operations

```
What do you need to do with properties?
|
+--> Read properties from an element?
|    |
|    +--> All psets and qtos?
|    |    --> ifcopenshell.util.element.get_psets(element)
|    |
|    +--> Only property sets?
|    |    --> get_psets(element, psets_only=True)
|    |
|    +--> Only quantity sets?
|    |    --> get_psets(element, qtos_only=True)
|    |
|    +--> Single property value?
|    |    --> get_pset(element, "Pset_WallCommon", "FireRating")
|    |
|    +--> Own psets only (skip type-inherited)?
|         --> get_psets(element, should_inherit=False)
|
+--> Add/edit property sets?
|    |
|    +--> Standard pset (Pset_ prefix)?
|    |    --> add_pset + edit_pset (template auto-loaded)
|    |
|    +--> Custom pset (company prefix)?
|    |    --> add_pset + edit_pset (Python type inference)
|    |
|    +--> Enumerated values?
|    |    --> edit_pset with list values + pset_template
|    |
|    +--> Delete a property?
|         --> edit_pset with value=None, should_purge=True
|
+--> Add/edit quantity sets?
|    |
|    +--> Standard qto (Qto_ prefix)?
|    |    --> add_qto + edit_qto
|    |
|    +--> Custom quantities?
|         --> add_qto + edit_qto (name-based type inference)
|
+--> Create property set templates?
|    --> pset_template.add_pset_template + add_prop_template
|
+--> Remove a pset from an element?
|    --> pset.remove_pset (cleans up all relationships)
|
+--> Share/unshare psets between elements?
     |
     +--> Assign existing pset to more elements?
     |    --> pset.assign_pset(products=[...], pset=pset)
     |
     +--> Make independent copies?
          --> pset.unshare_pset(products=[...], pset=pset)
```

---

## Essential Patterns

### Pattern 1: Create and Edit a Property Set

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+: IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")
wall = model.by_type("IfcWall")[0]

# Step 1: Create pset (idempotent: returns existing if name matches)
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

# Step 2: Edit properties (Python types auto-mapped to IFC types)
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "FireRating": "REI90",          # str  -> IfcLabel
    "ThermalTransmittance": 0.24,   # float -> IfcReal
    "LoadBearing": True,            # bool -> IfcBoolean
    "IsExternal": True,             # bool -> IfcBoolean
    "AcousticRating": 52,           # int  -> IfcInteger
})
```

### Pattern 2: Create and Edit a Quantity Set

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+: IFC4
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall, name="Qto_WallBaseQuantities")

ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 5.0,          # float + "Length" keyword -> IfcQuantityLength
    "Height": 3.0,          # float + "Height" keyword -> IfcQuantityLength
    "NetSideArea": 15.0,    # float + "Area" keyword   -> IfcQuantityArea
    "NetVolume": 3.0,       # float + "Volume" keyword -> IfcQuantityVolume
    "Width": 0.2,           # float + "Width" keyword  -> IfcQuantityLength
})
```

### Pattern 3: Read Properties

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+
import ifcopenshell.util.element

# All psets and qtos (dict of dicts)
all_data = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"id": 42, "FireRating": "REI90", ...}, ...}

# Single pset as dict
pset_data = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon")
# Returns: {"id": 42, "FireRating": "REI90", "LoadBearing": True, ...}

# Single property value directly
fire_rating = ifcopenshell.util.element.get_pset(
    wall, "Pset_WallCommon", "FireRating")
# Returns: "REI90"

# Filter psets only (no qtos)
psets_only = ifcopenshell.util.element.get_psets(wall, psets_only=True)

# Filter qtos only
qtos_only = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# Own psets only (skip inherited from type)
own = ifcopenshell.util.element.get_psets(wall, should_inherit=False)
```

### Pattern 4: Delete Properties

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+

# Delete individual properties: set value to None with should_purge=True (default)
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "OldProperty": None,   # Removed from IFC model
})

# Keep property entity but clear value (should_purge=False)
ifcopenshell.api.run("pset.edit_pset", model, pset=pset,
    properties={"OptionalProp": None}, should_purge=False)
# Property entity remains with NominalValue=None

# Remove entire pset from element (cleans up all relationships)
ifcopenshell.api.run("pset.remove_pset", model, product=wall, pset=pset)
```

### Pattern 5: Property Set Templates

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+: IFC4

# Create a custom pset template
template = ifcopenshell.api.run("pset_template.add_pset_template", model,
    name="Acme_RiskAssessment",
    template_type="PSET_TYPEDRIVENOVERRIDE",
    applicable_entity="IfcWall,IfcWallType")

# Add single-value property template
ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=template,
    name="HighVoltageRisk",
    description="Presence of high voltage hazard",
    template_type="P_SINGLEVALUE",
    primary_measure_type="IfcBoolean")

# Add enumerated-value property template
ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=template,
    name="RiskLevel",
    description="Risk assessment level",
    template_type="P_ENUMERATEDVALUE",
    primary_measure_type="IfcLabel")

# Add quantity template
qto_template = ifcopenshell.api.run("pset_template.add_pset_template", model,
    name="Acme_CustomQuantities",
    template_type="QTO_TYPEDRIVENOVERRIDE",
    applicable_entity="IfcWall")

ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=qto_template,
    name="ExposedArea",
    template_type="Q_AREA")
```

---

## Automatic Type Mapping

### Property Type Mapping (pset.edit_pset)

Python values are auto-mapped to IFC property types. Resolution order:

1. Existing property value type (preserved on update)
2. buildingSMART pset template `PrimaryMeasureType`
3. Entity instance type (`value.is_a()`)
4. Python type inference (fallback)

| Python Type | IFC Type | Example |
|-------------|----------|---------|
| `str` | `IfcLabel` | `"REI60"` |
| `float` | `IfcReal` | `0.24` |
| `bool` | `IfcBoolean` | `True` |
| `int` | `IfcInteger` | `3` |
| `datetime.datetime` | `IfcDateTime` | `datetime(2024,1,1)` |
| `datetime.date` | `IfcDate` | `date(2024,1,1)` |
| `None` | (deletes property) | with `should_purge=True` |

### Quantity Type Mapping (pset.edit_qto)

Quantity types are inferred from the property name and Python value type:

| Value Type | Name Contains (case-insensitive) | IFC Quantity Type |
|-----------|----------------------------------|-------------------|
| `float` | "area" | `IfcQuantityArea` |
| `float` | "volume" | `IfcQuantityVolume` |
| `float` | "weight" or "mass" | `IfcQuantityWeight` |
| `float` | "length", "width", "height", "depth", "distance" | `IfcQuantityLength` |
| `float` | "time" or "duration" | `IfcQuantityTime` |
| `float` | (no keyword match) | `IfcQuantityLength` (default) |
| `int` | (any name) | `IfcQuantityCount` |

---

## Naming Conventions

| Prefix | Meaning | Example |
|--------|---------|---------|
| `Pset_` | buildingSMART standard property set | `Pset_WallCommon`, `Pset_DoorCommon` |
| `Qto_` | buildingSMART standard quantity set | `Qto_WallBaseQuantities` |
| `EPset_` | Internal Bonsai/IfcOpenShell use | `EPset_Parametric`, `EPset_Annotation` |
| Custom | Company- or project-specific | `Acme_StructuralData` |

Standard psets ending with `BaseQuantities` (e.g., `Qto_WallBaseQuantities`) automatically get `MethodOfMeasurement = "BaseQuantities"`.

---

## Bonsai Operators

### Property Set Operators (`bpy.ops.bim.*`)

| Operator | Purpose |
|----------|---------|
| `bim.add_pset` | Add a property set to selected elements |
| `bim.add_qto` | Add a quantity set to selected elements |
| `bim.edit_pset` | Save edited pset values to IFC model |
| `bim.remove_pset` | Remove pset from selected/specified elements |
| `bim.enable_pset_editing` | Enter pset editing mode in UI |
| `bim.disable_pset_editing` | Exit pset editing mode |
| `bim.unshare_pset` | Create independent pset copies for selected elements |
| `bim.copy_property_to_selection` | Copy property value to all selected objects |
| `bim.add_edit_custom_property` | Bulk add/edit properties on selected objects |
| `bim.bulk_remove_psets` | Bulk remove named psets from selected objects |
| `bim.pset_bulk_rename_parameters` | Rename properties across all IfcElements |
| `bim.add_proposed_prop` | Add custom property to a non-template pset |
| `bim.save_pset_as_template` | Save existing pset as reusable template |

### Quantity Calculation Operators

| Operator | Purpose |
|----------|---------|
| `bim.calculate_edge_lengths` | Calculate total edge lengths from mesh |
| `bim.calculate_face_areas` | Calculate total face areas from mesh |
| `bim.calculate_object_volumes` | Calculate object volumes from mesh |
| `bim.calculate_formwork_area` | Calculate formwork area |
| `bim.calculate_single_quantity` | Calculate single quantity via ifc5d |
| `bim.perform_quantity_take_off` | Full QTO on selected objects |

---

## Bonsai Tool Layer: tool.Pset

Key methods available via Bonsai's dependency injection:

| Method | Purpose |
|--------|---------|
| `tool.Pset.get_element_pset(element, name)` | Get pset entity by name from element |
| `tool.Pset.get_pset_props(obj, obj_type)` | Get Blender pset UI properties |
| `tool.Pset.is_pset_applicable(element, name)` | Check if standard pset applies to element class |
| `tool.Pset.get_pset_template(name)` | Get pset template by name |
| `tool.Pset.is_pset_empty(pset)` | Check if pset has no non-None values |
| `tool.Pset.cast_string_to_primitive(value)` | Convert string to Python primitive |

---

## Pset Sharing and Unsharing

Multiple elements can share a single `IfcPropertySet` entity via `IfcRelDefinesByProperties`. Editing a shared property automatically triggers unsharing of individual property entities (checked via `get_total_inverses > 1`).

To explicitly unshare a pset for selected elements:

```python
# Creates independent shallow copies for each product
copies = ifcopenshell.api.run("pset.unshare_pset", model,
    products=[wall1, wall2], pset=shared_pset)
```

To assign an existing pset to additional elements:

```python
ifcopenshell.api.run("pset.assign_pset", model,
    products=[wall3, wall4], pset=existing_pset)
```

---

## Version Notes

| Feature | Bonsai v0.8.x / IfcOpenShell v0.8+ |
|---------|-------------------------------------|
| `product` parameter | Singular in `add_pset`, `add_qto`, `remove_pset` |
| `products` parameter | List in `assign_pset`, `unassign_pset`, `unshare_pset` |
| `should_purge` default | `True` (properties with `None` are removed) |
| Quantity `None` handling | ALWAYS removed (no `should_purge` parameter) |
| Template auto-loading | Built-in buildingSMART templates loaded by schema |
| `MethodOfMeasurement` | Auto-set to `"BaseQuantities"` for standard qto names |

---

## Dependencies

- **ifcos-syntax-api**: For `ifcopenshell.api.run()` invocation patterns and module table
- **bonsai-core-architecture**: For Bonsai three-layer architecture, `tool.Ifc`, `IfcStore`

## Reference Links

- [Method Signatures](references/methods.md) — Complete API signatures for pset, pset_template, and util functions
- [Code Examples](references/examples.md) — End-to-end property management examples
- [Anti-Patterns](references/anti-patterns.md) — Common property set mistakes and how to avoid them
