# Element Traversal — Anti-Patterns

## AP-1: Using STEP IDs as Persistent Identifiers

**WRONG:**
```python
# Storing STEP ID for later reference
wall = model.by_type("IfcWall")[0]
saved_id = wall.id()  # 123
# ... later, after file re-export ...
same_wall = model.by_id(saved_id)  # WRONG: ID may have changed
```

**CORRECT:**
```python
# Use GlobalId for persistent identification
wall = model.by_type("IfcWall")[0]
saved_guid = wall.GlobalId  # "2O2Fr$t4X7Zf8NOew3FLOH"
# ... later, even after file re-export ...
same_wall = model.by_guid(saved_guid)  # CORRECT: GlobalId is stable
```

**Why:** STEP IDs (`#123`) are assigned sequentially during file writing. Every re-export, round-trip, or file merge can reassign these IDs. GlobalIds are 128-bit UUIDs that persist across file versions by design.

---

## AP-2: Manually Constructing GUIDs

**WRONG:**
```python
import uuid

# Manually creating a GUID string
guid = str(uuid.uuid4())[:22]  # WRONG: not valid IFC encoding
guid = "ABCDEF1234567890123456"  # WRONG: not a valid IFC GUID
```

**CORRECT:**
```python
import ifcopenshell.guid

# Generate a properly encoded IFC GUID
guid = ifcopenshell.guid.new()

# Or convert from a standard UUID
import uuid
standard_uuid = str(uuid.uuid4())
ifc_guid = ifcopenshell.guid.compress(standard_uuid)
```

**Why:** IFC GUIDs use a specific 22-character base64 encoding with the character set `0-9`, `A-Z`, `a-z`, `_`, `$`. This is NOT the same as standard UUID string formatting or simple truncation. Using `ifcopenshell.guid.new()` guarantees valid encoding.

---

## AP-3: Using get_info(recursive=True) in Loops

**WRONG:**
```python
# Expanding all references for every wall — extremely slow
for wall in model.by_type("IfcWall"):
    info = wall.get_info(recursive=True)  # WRONG: recursively expands ALL references
    print(info)
```

**CORRECT:**
```python
# Access only the attributes you need
for wall in model.by_type("IfcWall"):
    print(f"{wall.Name}: {wall.GlobalId}")

# Or use get_info() without recursive for flat attribute dump
for wall in model.by_type("IfcWall"):
    info = wall.get_info()  # Fast: no recursive expansion
    print(f"{info['Name']}: {info['GlobalId']}")

# Or use scalar_only for just primitive values
for wall in model.by_type("IfcWall"):
    info = wall.get_info(scalar_only=True)
    print(info)
```

**Why:** `recursive=True` follows every entity reference and expands it into a nested dict. For a wall, this includes its placement, representation, property sets, material, spatial relationships — potentially thousands of entities. On large models, this causes severe performance degradation.

---

## AP-4: Assuming All Entities Have GlobalId

**WRONG:**
```python
for entity in model:
    print(entity.GlobalId)  # WRONG: crashes on non-IfcRoot entities
```

**CORRECT:**
```python
# Check if entity derives from IfcRoot before accessing GlobalId
for entity in model:
    if entity.is_a("IfcRoot"):
        print(entity.GlobalId)

# Or use get_info() which safely includes all available attributes
for entity in model:
    info = entity.get_info()
    guid = info.get("GlobalId", "N/A")
```

**Why:** Only entities that inherit from `IfcRoot` have a `GlobalId` attribute. Low-level entities like `IfcCartesianPoint`, `IfcDirection`, `IfcPropertySingleValue`, `IfcMaterial`, and geometry entities do NOT have GlobalIds. Accessing `.GlobalId` on these entities raises `AttributeError`.

---

## AP-5: Not Handling None Values on Optional Attributes

**WRONG:**
```python
wall = model.by_type("IfcWall")[0]
print(wall.Description.upper())  # WRONG: Description may be None
print(wall.Name + " - modified")  # WRONG: Name may be None
```

**CORRECT:**
```python
wall = model.by_type("IfcWall")[0]
print((wall.Description or "").upper())
print(f"{wall.Name or 'Unnamed'} - modified")

# Or check explicitly
if wall.Description is not None:
    print(wall.Description.upper())
```

**Why:** Many IFC attributes are optional per the schema. `Name`, `Description`, `ObjectType`, `Tag`, and others frequently have `None` values. ALWAYS guard against None before string operations.

---

## AP-6: Accessing wrappedValue Without None Check

**WRONG:**
```python
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    value = prop.NominalValue.wrappedValue  # WRONG: NominalValue may be None
```

**CORRECT:**
```python
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                if prop.is_a("IfcPropertySingleValue") and prop.NominalValue:
                    value = prop.NominalValue.wrappedValue
```

**Why:** `IfcPropertySingleValue.NominalValue` is an optional attribute. Properties can exist with a name but no value. Accessing `.wrappedValue` on `None` raises `AttributeError`.

---

## AP-7: Manual Inverse Traversal Instead of util.element

**WRONG (verbose, error-prone):**
```python
# Finding container manually
def get_container(model, element):
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelContainedInSpatialStructure"):
            return rel.RelatingStructure
    return None

# Finding type manually
def get_type(model, element):
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelDefinesByType"):
            return rel.RelatingType
    return None
```

**CORRECT (use the utility module):**
```python
import ifcopenshell.util.element

container = ifcopenshell.util.element.get_container(wall)
wall_type = ifcopenshell.util.element.get_type(wall)
material = ifcopenshell.util.element.get_material(wall)
psets = ifcopenshell.util.element.get_psets(wall)
```

**Why:** `ifcopenshell.util.element` handles edge cases, schema differences, and relationship chains that manual traversal misses. For example, `get_container()` also checks for aggregation-based containment, and `get_psets()` handles both property sets and quantity sets, including type-level inheritance. ALWAYS prefer utility functions for production code.

---

## AP-8: Hardcoding Schema-Specific Entity Names

**WRONG:**
```python
# Breaks on IFC4X3 where IfcBuildingElement was renamed
elements = model.by_type("IfcBuildingElement")  # WRONG for IFC4X3

# Breaks on IFC4+ where IfcWallStandardCase was removed
standard_walls = model.by_type("IfcWallStandardCase")  # WRONG for IFC4+
```

**CORRECT:**
```python
schema = model.schema

# Schema-aware building element query
if schema in ("IFC2X3", "IFC4"):
    elements = model.by_type("IfcBuildingElement")
elif schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")

# Use parent class to get all walls regardless of schema
walls = model.by_type("IfcWall")  # Works on all schemas, includes subtypes
```

**Why:** Entity class names change between IFC schema versions. `IfcBuildingElement` was renamed to `IfcBuiltElement` in IFC4X3. `IfcWallStandardCase` exists only in IFC2X3. ALWAYS check `model.schema` before using schema-specific entity names.

---

## AP-9: Iterating All Entities When by_type Suffices

**WRONG:**
```python
# Scanning ALL entities to find walls — slow and wasteful
walls = [e for e in model if e.is_a("IfcWall")]
```

**CORRECT:**
```python
# Use the indexed lookup — fast after first call
walls = model.by_type("IfcWall")
```

**Why:** `by_type()` uses an internal index that is built once and cached. Iterating over all entities with `is_a()` checks does a linear scan of every entity in the file. For a model with 100,000 entities, `by_type()` is orders of magnitude faster.

---

## AP-10: Using Deprecated Selector API

**WRONG:**
```python
from ifcopenshell.util.selector import Selector

# Deprecated API
results = Selector().parse(model, ".IfcWall")
```

**CORRECT:**
```python
import ifcopenshell.util.selector

# Modern API
results = ifcopenshell.util.selector.filter_elements(model, "IfcWall")
```

**Why:** The `Selector` class with `.parse()` is the old API. ALWAYS use `filter_elements()` which is the current, maintained interface.

---

## AP-11: Forgetting that by_type Returns a Tuple

**WRONG:**
```python
walls = model.by_type("IfcWall")
walls.append(new_wall)  # WRONG: tuples don't have append()
walls.sort(key=lambda w: w.Name)  # WRONG: tuples don't have sort()
```

**CORRECT:**
```python
walls = list(model.by_type("IfcWall"))  # Convert to list if mutation needed
walls.append(new_wall)
walls.sort(key=lambda w: w.Name or "")
```

**Why:** `by_type()` returns a tuple, not a list. If you need to modify the collection (append, sort, remove), convert to a list first. For read-only iteration, tuples work fine.

---

## AP-12: Not Checking pset.is_a("IfcPropertySet") in Property Traversal

**WRONG:**
```python
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        # WRONG: pset might be IfcElementQuantity, not IfcPropertySet
        for prop in pset.HasProperties:
            pass
```

**CORRECT:**
```python
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                pass
        elif pset.is_a("IfcElementQuantity"):
            for qty in pset.Quantities:
                pass
```

**Why:** `IfcRelDefinesByProperties.RelatingPropertyDefinition` can point to either `IfcPropertySet` or `IfcElementQuantity` (or other subtypes of `IfcPropertySetDefinition`). `IfcElementQuantity` uses `Quantities`, not `HasProperties`. Accessing the wrong attribute raises `AttributeError`.
