# Error Handling Methods Reference

## Exception Types

### ifcopenshell.Error

Base exception for IfcOpenShell-specific errors. Raised when the library encounters invalid IFC data.

```python
import ifcopenshell

try:
    model = ifcopenshell.open("corrupt_file.ifc")
except ifcopenshell.Error as e:
    print(f"IFC parsing error: {e}")
```

### RuntimeError

Raised by the C++ core for:
- Entity not found in schema
- Invalid entity references (after removal)
- Geometry processing failures (OpenCASCADE)
- Invalid STEP data parsing

```python
# Schema mismatch
try:
    model = ifcopenshell.file(schema="IFC2X3")
    element = model.create_entity("IfcBuiltElement",
        GlobalId=ifcopenshell.guid.new())
except RuntimeError as e:
    # "entity 'IfcBuiltElement' not found in schema 'IFC2X3'"
    print(f"Schema error: {e}")

# Geometry failure
try:
    shape = ifcopenshell.geom.create_shape(settings, element)
except RuntimeError as e:
    # "Failed to process shape"
    print(f"Geometry error: {e}")
```

### AttributeError

Raised when accessing non-existent attributes on entities. Common causes:
- Wrong PascalCase spelling
- Accessing attribute that does not exist in entity's IFC class
- Accessing attribute on `None` return value

```python
try:
    name = wall.name  # WRONG: should be wall.Name
except AttributeError:
    name = wall.Name  # CORRECT: PascalCase
```

### TypeError

Raised when passing wrong types to entity constructors or attribute setters.

```python
try:
    wall.ObjectPlacement = "origin"  # String instead of entity
except (TypeError, RuntimeError):
    # Must pass IfcLocalPlacement entity, not string
    pass
```

### FileNotFoundError

Standard Python exception when IFC file path does not exist.

```python
try:
    model = ifcopenshell.open("nonexistent.ifc")
except FileNotFoundError:
    print("IFC file not found")
```

---

## Diagnostic Methods

### element.get_info()

Returns dict with all attribute names and values for an entity.

```python
info = wall.get_info()
# Returns: {
#   "id": 42,
#   "type": "IfcWall",
#   "GlobalId": "3Ks0WO3qP2xhJ0wBKg$u5H",
#   "OwnerHistory": #1=IfcOwnerHistory(...),
#   "Name": "Wall 001",
#   "Description": None,
#   "ObjectType": None,
#   "ObjectPlacement": #5=IfcLocalPlacement(...),
#   "Representation": #10=IfcProductDefinitionShape(...),
#   "Tag": None
# }
```

**Parameters:**
- `recursive=False` (default) — returns entity references as-is
- `recursive=True` — expands all entity references to nested dicts (WARNING: expensive for large entities)
- `include_identifier=True` (default) — includes `id` and `type` fields

### element.get_info_2(include_identifier=True)

Alternative that returns IfcOpenShell-specific value types instead of Python primitives. Useful for round-tripping data.

### element.is_a(ifc_class=None)

Checks entity type, supports inheritance checking.

```python
wall.is_a()            # "IfcWall" — returns type name
wall.is_a("IfcWall")   # True — exact match
wall.is_a("IfcRoot")   # True — checks inheritance hierarchy
wall.is_a("IfcSlab")   # False
```

### element.id()

Returns the STEP file line number (entity ID) of the element.

```python
step_id = wall.id()  # e.g., 42 (corresponds to #42= in STEP file)
```

### model.get_inverse(element)

Returns all entities that reference the given element. Essential for finding relationships.

```python
refs = model.get_inverse(wall)
for ref in refs:
    print(f"{ref.is_a()} #{ref.id()}")
# Output example:
# IfcRelContainedInSpatialStructure #50
# IfcRelDefinesByProperties #60
# IfcRelAssociatesMaterial #70
```

### model.by_type(ifc_class, include_subtypes=True)

Returns all entities of a given type.

```python
# All walls including IfcWallStandardCase
walls = model.by_type("IfcWall")

# Only exact IfcWall (no subtypes) — faster
walls_exact = model.by_type("IfcWall", include_subtypes=False)
```

### model.by_guid(guid)

Returns entity with the given GlobalId, or raises an error if not found.

```python
try:
    entity = model.by_guid("3Ks0WO3qP2xhJ0wBKg$u5H")
except RuntimeError:
    print("GUID not found")
```

### model.by_id(step_id)

Returns entity with the given STEP ID. O(1) lookup.

```python
entity = model.by_id(42)  # Fast direct lookup
```

---

## Validation Utility Functions

### ifcopenshell.util.element.get_container(element)

Returns the spatial container (IfcBuildingStorey, IfcSpace, etc.) or `None`.

```python
import ifcopenshell.util.element

container = ifcopenshell.util.element.get_container(wall)
# Returns: IfcBuildingStorey entity, or None
```

### ifcopenshell.util.element.get_type(element)

Returns the type entity (IfcWallType, etc.) or `None`.

```python
wall_type = ifcopenshell.util.element.get_type(wall)
# Returns: IfcWallType entity, or None
```

### ifcopenshell.util.element.get_psets(element, psets_only=False, qtos_only=False)

Returns all property sets and/or quantity sets as nested dict.

```python
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {
#   "Pset_WallCommon": {"id": 100, "FireRating": "REI60", "IsExternal": True},
#   "BaseQuantities": {"id": 101, "NetArea": 25.5, "GrossVolume": 5.1}
# }

# Only property sets
psets = ifcopenshell.util.element.get_psets(wall, psets_only=True)

# Only quantity sets
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
```

### ifcopenshell.util.element.get_material(element)

Returns the material or material set for an element, or `None`.

```python
material = ifcopenshell.util.element.get_material(wall)
# Returns: IfcMaterial, IfcMaterialLayerSet, IfcMaterialProfileSet, etc., or None
```

### ifcopenshell.util.unit.calculate_unit_scale(model)

Returns the scale factor to convert from file units to SI (meters).

```python
import ifcopenshell.util.unit

scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# mm file: 0.001
# m file:  1.0
# ft file: 0.3048
```

### ifcopenshell.util.unit.get_project_unit(model, unit_type)

Returns the IfcNamedUnit entity for a given unit type.

```python
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
area_unit = ifcopenshell.util.unit.get_project_unit(model, "AREAUNIT")
```

### ifcopenshell.guid.new()

Generates a new valid IFC GUID (22-character base64 format).

```python
import ifcopenshell.guid

guid = ifcopenshell.guid.new()  # e.g., "3Ks0WO3qP2xhJ0wBKg$u5H"
```

### ifcopenshell.guid.expand(ifc_guid)

Converts IFC GUID to standard UUID string.

```python
uuid_str = ifcopenshell.guid.expand("3Ks0WO3qP2xhJ0wBKg$u5H")
# Returns: "550e8400-e29b-41d4-a716-446655440000"
```

### ifcopenshell.guid.compress(uuid_str)

Converts standard UUID to IFC GUID format.

```python
ifc_guid = ifcopenshell.guid.compress("550e8400-e29b-41d4-a716-446655440000")
# Returns: 22-character IFC GUID
```

---

## Safe Removal API

### ifcopenshell.api.run("root.remove_product", model, product=element)

Removes a product entity and cleans up ALL relationships.

```python
import ifcopenshell.api

# Safe removal — handles relationship cleanup
ifcopenshell.api.run("root.remove_product", model, product=wall)

# What it cleans up:
# - IfcRelContainedInSpatialStructure referencing the wall
# - IfcRelDefinesByProperties referencing the wall
# - IfcRelAssociatesMaterial referencing the wall
# - IfcRelVoidsElement (openings in the wall)
# - IfcRelDefinesByType referencing the wall
# - IfcLocalPlacement (if not shared with other elements)
# - IfcProductDefinitionShape (if not shared)
```

### model.garbage_collect()

Removes unreferenced entities from the file. Call after bulk removals.

```python
# After removing many entities
for wall in walls_to_remove:
    ifcopenshell.api.run("root.remove_product", model, product=wall)

# Clean up orphaned entities
model.garbage_collect()
```
