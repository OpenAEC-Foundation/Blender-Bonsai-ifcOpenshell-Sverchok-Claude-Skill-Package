---
name: ifcos-errors-patterns
description: 'Catalogs common IfcOpenShell error patterns including RuntimeError on invalid entities, AttributeError from wrong PascalCase, TypeError from wrong parameter types, entity invalidation after removal, and debugging strategies for IFC processing scripts. Activates when debugging IfcOpenShell errors, handling exceptions, or troubleshooting IFC processing failures.'
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Error Patterns and Debugging

## Quick Reference

### Decision Tree: Diagnosing IfcOpenShell Errors

```
Error occurred?
├── RuntimeError
│   ├── "entity not found in schema" → Schema mismatch (§1)
│   ├── "Failed to process shape" → Geometry failure (§4)
│   ├── Entity reference invalid → Entity was removed (§5)
│   └── Invalid STEP data → File corruption (§7)
│
├── AttributeError
│   ├── "entity has no attribute 'X'" → Wrong attribute name / wrong entity type (§2)
│   ├── NoneType has no attribute → Null return not checked (§6)
│   └── 'parent' / 'children' → IFC uses relationships, not tree attributes (§3)
│
├── TypeError
│   ├── Wrong argument type to create_entity → Type mismatch in attributes (§8)
│   ├── Expected entity, got string → Entity reference required (§8)
│   └── Missing positional argument → settings missing for geom calls (§4)
│
├── ValueError
│   ├── GUID format wrong → Use ifcopenshell.guid.new() (§9)
│   └── Coordinate scale wrong → Unit conversion missing (§10)
│
└── Silent failures (no error but wrong output)
    ├── Orphaned entities → Missing relationship creation (§3)
    ├── Duplicate property sets → Not checking existing psets (§11)
    ├── Invalid file → Missing OwnerHistory in IFC2X3 (§12)
    └── Wrong spatial location → Missing spatial containment (§3)
```

### Critical Warnings

- **ALWAYS** use `ifcopenshell.api.run()` for mutations. NEVER modify entity attributes directly for relationships, spatial containment, or property sets.
- **ALWAYS** check `model.schema` before using schema-specific entities. Entity names differ between IFC2X3, IFC4, and IFC4X3.
- **ALWAYS** check return values for `None` before accessing attributes. Functions like `get_container()`, `get_type()`, and `create_shape()` can return `None`.
- **NEVER** use `model.remove()` directly on products. Use `ifcopenshell.api.run("root.remove_product", ...)` to clean up relationships.
- **NEVER** cache entity references across file modifications. Entity wrappers become invalid after `model.remove()` or `model.undo()`.
- **NEVER** assume coordinate units. ALWAYS call `ifcopenshell.util.unit.calculate_unit_scale(model)` and apply the scale factor.
- **ALWAYS** wrap geometry processing in `try/except RuntimeError`. Not all elements have processable geometry.

---

## Error Category 1: Schema Mismatch Errors

IFC has three major schema versions with different entity names. Using an entity that does not exist in the file's schema raises `RuntimeError`.

```python
# IfcOpenShell — schema-agnostic pattern
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema  # "IFC2X3", "IFC4", or "IFC4X3"

# WRONG: IfcBuiltElement only exists in IFC4X3
element = model.create_entity("IfcBuiltElement", ...)
# RuntimeError: entity 'IfcBuiltElement' not found in schema 'IFC2X3'

# CORRECT: Check schema first
if schema == "IFC4X3":
    element = model.create_entity("IfcBuiltElement", ...)
else:
    element = model.create_entity("IfcBuildingElementProxy", ...)
```

**Key entity differences by schema:**

| Entity/Concept | IFC2X3 | IFC4 | IFC4X3 |
|---------------|--------|------|--------|
| Standard wall | IfcWallStandardCase | IfcWall | IfcWall |
| Building element base | IfcBuildingElement | IfcBuildingElement | IfcBuiltElement |
| OwnerHistory | REQUIRED | OPTIONAL | OPTIONAL |
| Door type | IfcDoorStyle | IfcDoorType | IfcDoorType |
| Window type | IfcWindowStyle | IfcWindowType | IfcWindowType |
| Infrastructure | Not available | Not available | IfcBridge, IfcRoad, IfcAlignment |

---

## Error Category 2: AttributeError on Entity Attributes

IFC entity attributes use PascalCase. Using wrong case or non-existent attribute names raises `AttributeError`.

```python
# WRONG: snake_case or lowercase
wall.global_id       # AttributeError
wall.name            # AttributeError
wall.object_type     # AttributeError

# CORRECT: PascalCase as defined in IFC schema
wall.GlobalId        # "3Ks0WO3qP2xhJ0wBKg$u5H"
wall.Name            # "Wall 001"
wall.ObjectType      # "Standard Wall"
```

**ALWAYS** use `element.get_info()` to discover available attributes:

```python
info = wall.get_info()
# Returns dict: {"id": 42, "type": "IfcWall", "GlobalId": "...", "Name": "...", ...}
print(list(info.keys()))  # See all attribute names
```

---

## Error Category 3: Incorrect Relationship Handling

IFC uses objectified relationships (separate entities). There are NO parent/child attributes on elements.

```python
# WRONG: IFC has no tree-style attributes
storey.parent = building        # AttributeError
wall.ContainedInStructure = storey  # Does NOT work as expected

# CORRECT: Use ifcopenshell.api for relationships
# Spatial hierarchy (aggregation): Project > Site > Building > Storey
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Elements in spatial container (containment)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

**Relationship type reference:**

| Relationship | API Function | Purpose |
|-------------|-------------|---------|
| IfcRelAggregates | `aggregate.assign_object` | Spatial hierarchy decomposition |
| IfcRelContainedInSpatialStructure | `spatial.assign_container` | Element in spatial container |
| IfcRelDefinesByType | `type.assign_type` | Element to Type assignment |
| IfcRelDefinesByProperties | `pset.add_pset` | Property set to element |
| IfcRelAssociatesMaterial | `material.assign_material` | Material to element |
| IfcRelVoidsElement | `void.add_opening` | Opening in element |

---

## Error Category 4: Geometry Processing Failures

`ifcopenshell.geom.create_shape()` invokes OpenCASCADE and can fail for multiple reasons.

```python
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

# ALWAYS wrap in try/except
try:
    shape = ifcopenshell.geom.create_shape(settings, element)
except RuntimeError as e:
    print(f"Geometry failed for {element.is_a()} #{element.id()}: {e}")
    shape = None

# ALWAYS check result before accessing geometry data
if shape is not None:
    verts = shape.geometry.verts
    faces = shape.geometry.faces
```

**Common geometry failure causes:**

| Cause | Symptom | Fix |
|-------|---------|-----|
| No Representation | RuntimeError | Check `element.Representation is not None` |
| Missing settings argument | TypeError | Pass `settings` as first argument |
| Spatial element (IfcSite) | RuntimeError | Filter by elements with Body representation |
| Complex boolean operations | RuntimeError | Set `DISABLE_OPENING_SUBTRACTIONS=True` |
| Corrupt geometry definition | RuntimeError | Skip element, log warning |

**Check for geometry before processing:**

```python
def has_geometry(element):
    """Check if an IFC element has processable 3D geometry."""
    if not hasattr(element, "Representation") or element.Representation is None:
        return False
    for rep in element.Representation.Representations:
        if rep.RepresentationIdentifier in ("Body", "Facetation", "Tessellation"):
            return True
    return False
```

---

## Error Category 5: Entity Invalidation After Removal

Entity wrappers are C++ pointers. After `model.remove()`, references become invalid.

```python
wall = model.by_type("IfcWall")[0]
wall_id = wall.id()
wall_name = wall.Name  # Works fine

model.remove(wall)
# wall is NOW INVALID — the C++ object is destroyed

# WRONG: Using invalidated reference
print(wall.Name)   # RuntimeError or undefined behavior
print(wall.id())   # RuntimeError or undefined behavior

# CORRECT: Extract needed data BEFORE removal
wall_data = {"id": wall.id(), "name": wall.Name, "guid": wall.GlobalId}
ifcopenshell.api.run("root.remove_product", model, product=wall)
# Use wall_data (plain dict) from here
```

**NEVER** iterate and remove simultaneously:

```python
# WRONG: Modifying collection while iterating
for wall in model.by_type("IfcWall"):
    model.remove(wall)  # Invalidates the iteration!

# CORRECT: Collect first, then remove
walls_to_remove = list(model.by_type("IfcWall"))
for wall in walls_to_remove:
    ifcopenshell.api.run("root.remove_product", model, product=wall)
```

---

## Error Category 6: None Return Values Not Checked

Many IfcOpenShell utility functions return `None` when data is not found.

```python
import ifcopenshell.util.element

# WRONG: Assuming result is always valid
container = ifcopenshell.util.element.get_container(wall)
print(container.Name)  # AttributeError if container is None!

element_type = ifcopenshell.util.element.get_type(wall)
print(element_type.Name)  # AttributeError if no type assigned!

# CORRECT: ALWAYS check for None
container = ifcopenshell.util.element.get_container(wall)
if container is not None:
    print(container.Name)

element_type = ifcopenshell.util.element.get_type(wall)
if element_type is not None:
    print(element_type.Name)
```

**Functions that commonly return None:**

| Function | Returns None When |
|----------|------------------|
| `get_container(element)` | Element has no spatial containment |
| `get_type(element)` | No type assigned |
| `get_material(element)` | No material assigned |
| `get_psets(element).get("X")` | Property set does not exist |
| `geom.create_shape(settings, element)` | Geometry processing fails |
| `model.by_guid("...")` | GUID not found in file |

---

## Error Category 7: File Corruption from Direct Removal

Using `model.remove()` directly leaves dangling references in relationship entities.

```python
# WRONG: Direct removal corrupts relationships
model.remove(wall)
# IfcRelContainedInSpatialStructure still references the removed wall
# IfcRelDefinesByProperties still references the removed wall
# File is NOW CORRUPTED

# CORRECT: API handles all relationship cleanup
ifcopenshell.api.run("root.remove_product", model, product=wall)
# Removes wall + cleans up:
# - Spatial containment relationships
# - Property set relationships
# - Material associations
# - Opening relationships
# - Type assignments
# - Placement (if not shared)
# - Representation (if not shared)
```

---

## Error Category 8: TypeError from Wrong Attribute Types

IFC attributes have strict types. Passing wrong types raises `TypeError` or `RuntimeError`.

```python
# WRONG: String where entity is expected
wall.ObjectPlacement = "at origin"
# RuntimeError: Expected IfcObjectPlacement, got str

# WRONG: Single entity where tuple is expected
rel.RelatedElements = wall  # Expects tuple, not single entity

# CORRECT: Use proper entity references
placement = model.create_entity("IfcLocalPlacement", ...)
wall.ObjectPlacement = placement

# CORRECT: Use tuple for aggregate attributes
rel.RelatedElements = (wall1, wall2)
```

---

## Error Category 9: GUID Format Errors

IFC uses 22-character base64 GUIDs, NOT standard 36-character UUIDs.

```python
import uuid

# WRONG: Standard UUID format
wall = model.create_entity("IfcWall",
    GlobalId=str(uuid.uuid4()),  # 36 chars — WRONG format
    Name="Wall")

# CORRECT: Use ifcopenshell.guid
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),  # 22 chars — correct
    Name="Wall")

# BEST: Use ifcopenshell.api (auto-generates GUID)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
```

---

## Error Category 10: Unit Conversion Errors

IFC files can use meters, millimeters, feet, or other units. NEVER assume meters.

```python
import ifcopenshell.util.unit

# ALWAYS get the unit scale factor
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# mm file → 0.001, m file → 1.0, ft file → 0.3048

# Apply to ALL coordinates extracted from the model
raw_coords = [3500.0, 2800.0, 0.0]  # From IFC file
coords_meters = [c * unit_scale for c in raw_coords]
```

---

## Error Category 11: Duplicate Property Sets

Creating a property set without checking for existing ones produces duplicates.

```python
from ifcopenshell.util.element import get_psets

# WRONG: Creates duplicate if Pset_WallCommon already exists
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

# CORRECT: Check first, then create or update
existing = get_psets(wall)
if "Pset_WallCommon" in existing:
    pset = model.by_id(existing["Pset_WallCommon"]["id"])
else:
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"IsExternal": True})
```

---

## Error Category 12: Missing OwnerHistory in IFC2X3

OwnerHistory is REQUIRED on most entities in IFC2X3 but OPTIONAL in IFC4/IFC4X3.

```python
# WRONG: No OwnerHistory in IFC2X3
model = ifcopenshell.file(schema="IFC2X3")
wall = model.create_entity("IfcWallStandardCase",
    GlobalId=ifcopenshell.guid.new(), Name="Wall")
# File is INVALID — validators will reject it

# CORRECT: Use ifcopenshell.api (handles OwnerHistory automatically)
model = ifcopenshell.file(schema="IFC2X3")
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallStandardCase", name="Wall")
# API creates OwnerHistory and assigns it automatically
```

---

## Debugging Strategy

### Step-by-step Debugging Protocol

```python
import ifcopenshell

def debug_ifc_element(model, element):
    """Comprehensive debug output for an IFC element."""
    print(f"=== Debug: {element.is_a()} #{element.id()} ===")
    print(f"Schema: {model.schema}")
    print(f"GlobalId: {element.GlobalId}")
    print(f"Name: {element.Name}")

    # 1. Check all attributes
    info = element.get_info()
    for key, value in info.items():
        print(f"  {key}: {value}")

    # 2. Check spatial containment
    container = ifcopenshell.util.element.get_container(element)
    print(f"Container: {container.Name if container else 'NONE (PROBLEM!)'}")

    # 3. Check type assignment
    elem_type = ifcopenshell.util.element.get_type(element)
    print(f"Type: {elem_type.Name if elem_type else 'None'}")

    # 4. Check property sets
    psets = ifcopenshell.util.element.get_psets(element)
    print(f"Property sets: {list(psets.keys())}")

    # 5. Check geometry
    if hasattr(element, "Representation") and element.Representation:
        reps = [r.RepresentationIdentifier
                for r in element.Representation.Representations]
        print(f"Representations: {reps}")
    else:
        print("Representations: NONE")
```

### Common Debug Techniques

```python
# Inspect entity type hierarchy
print(element.is_a())           # "IfcWall"
print(element.is_a("IfcRoot"))  # True (checks inheritance)

# List all entities of a type
walls = model.by_type("IfcWall")
print(f"Total walls: {len(walls)}")

# Find entity by GlobalId
entity = model.by_guid("3Ks0WO3qP2xhJ0wBKg$u5H")

# Find entity by step ID
entity = model.by_id(42)

# Get inverse relationships (who references this entity?)
refs = model.get_inverse(wall)
for ref in refs:
    print(f"Referenced by: {ref.is_a()} #{ref.id()}")

# Validate file units
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Unit scale to meters: {unit_scale}")

# Check file statistics
print(f"Total entities: {len(model)}")
for entity_type in ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam"]:
    count = len(model.by_type(entity_type))
    if count > 0:
        print(f"  {entity_type}: {count}")
```

---

## Reference Links

- [Error Handling Methods](references/methods.md) — Exception types, validation functions, diagnostic utilities
- [Working Debug Examples](references/examples.md) — End-to-end error handling and debugging code
- [Anti-Patterns](references/anti-patterns.md) — Common IfcOpenShell mistakes with corrections
