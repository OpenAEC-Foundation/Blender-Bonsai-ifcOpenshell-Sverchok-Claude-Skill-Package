# Anti-Patterns: Common IfcOpenShell Mistakes

## Anti-Pattern 1: Using model.remove() Instead of API

**WRONG:**
```python
wall = model.by_type("IfcWall")[0]
model.remove(wall)
# Dangling references in:
# - IfcRelContainedInSpatialStructure
# - IfcRelDefinesByProperties
# - IfcRelAssociatesMaterial
# File is CORRUPTED
```

**CORRECT:**
```python
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run("root.remove_product", model, product=wall)
# API cleans up ALL relationships automatically
```

**Why:** `model.remove()` only deletes the entity itself. All relationship entities that reference the removed entity retain dangling `$` references. IFC viewers and validators will reject the file.

---

## Anti-Pattern 2: Not Checking Schema Before Entity Creation

**WRONG:**
```python
# Assumes IFC4X3 entities exist in all schemas
element = model.create_entity("IfcBuiltElement", ...)
# RuntimeError if schema is IFC2X3 or IFC4

# Assumes IfcWallStandardCase exists in all schemas
wall = model.create_entity("IfcWallStandardCase", ...)
# Deprecated/removed in IFC4X3
```

**CORRECT:**
```python
schema = model.schema
if schema == "IFC2X3":
    wall = model.create_entity("IfcWallStandardCase", ...)
elif schema in ("IFC4", "IFC4X3"):
    wall = model.create_entity("IfcWall", ...)
```

**Why:** IFC2X3, IFC4, and IFC4X3 have different entity names. Using a non-existent entity for the current schema raises `RuntimeError`.

---

## Anti-Pattern 3: Assuming Coordinates Are in Meters

**WRONG:**
```python
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
x, y, z = matrix[:3, 3]
# x could be 3500 (mm) or 11.48 (feet) — NOT necessarily meters
```

**CORRECT:**
```python
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
x, y, z = matrix[:3, 3]
x_meters = x * unit_scale
y_meters = y * unit_scale
z_meters = z * unit_scale
```

**Why:** IFC files commonly use millimeters (Europe), feet (US), or other units. The `calculate_unit_scale()` function returns the factor to convert to SI meters.

---

## Anti-Pattern 4: Not Checking None Returns

**WRONG:**
```python
container = ifcopenshell.util.element.get_container(wall)
print(container.Name)  # AttributeError if container is None

element_type = ifcopenshell.util.element.get_type(wall)
print(element_type.Name)  # AttributeError if no type assigned
```

**CORRECT:**
```python
container = ifcopenshell.util.element.get_container(wall)
if container is not None:
    print(container.Name)

element_type = ifcopenshell.util.element.get_type(wall)
type_name = element_type.Name if element_type else "No type"
```

**Why:** Many IfcOpenShell utility functions return `None` when the requested relationship or data does not exist. Not all elements have spatial containment, type assignments, or materials.

---

## Anti-Pattern 5: Using snake_case for IFC Attributes

**WRONG:**
```python
wall.global_id        # AttributeError
wall.name             # AttributeError
wall.object_placement # AttributeError
wall.is_external      # AttributeError — this is a property, not an attribute
```

**CORRECT:**
```python
wall.GlobalId         # IFC attributes use PascalCase
wall.Name
wall.ObjectPlacement

# Properties are NOT direct attributes — use utility functions
psets = ifcopenshell.util.element.get_psets(wall)
is_external = psets.get("Pset_WallCommon", {}).get("IsExternal")
```

**Why:** IFC schema defines all attributes in PascalCase. Properties (like IsExternal) are stored in separate IfcPropertySet entities linked through IfcRelDefinesByProperties relationships — they are NOT direct entity attributes.

---

## Anti-Pattern 6: Creating Duplicate Property Sets

**WRONG:**
```python
# Called multiple times on the same wall
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
# If called twice: TWO Pset_WallCommon attached to the wall!
```

**CORRECT:**
```python
existing = ifcopenshell.util.element.get_psets(wall)
if "Pset_WallCommon" in existing:
    pset = model.by_id(existing["Pset_WallCommon"]["id"])
else:
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"IsExternal": True})
```

**Why:** `pset.add_pset` always creates a new property set. It does not check for existing ones. Duplicates cause confusion in viewers and validators.

---

## Anti-Pattern 7: Direct Relationship Attribute Assignment

**WRONG:**
```python
wall.ContainedInStructure = storey
# This does NOT create a spatial containment relationship

storey.parent = building
# AttributeError: IFC has no 'parent' attribute
```

**CORRECT:**
```python
# Spatial containment
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# Aggregation hierarchy
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
```

**Why:** IFC uses objectified relationships — separate relationship entities (IfcRelContainedInSpatialStructure, IfcRelAggregates) link elements together. Setting attributes on the element directly does NOT create these relationship entities.

---

## Anti-Pattern 8: Using Standard UUID Instead of IFC GUID

**WRONG:**
```python
import uuid

wall = model.create_entity("IfcWall",
    GlobalId=str(uuid.uuid4()),  # 36-char format "550e8400-e29b-..."
    Name="Wall")
# IFC GlobalId must be exactly 22 characters (base64-encoded)
```

**CORRECT:**
```python
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),  # 22-char format "3Ks0WO3q..."
    Name="Wall")

# Or use API (auto-generates GUID):
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
```

**Why:** IFC uses a compressed 22-character base64 GUID format (ISO 10303-21). Standard UUIDs are 36 characters and will cause validation errors or truncation.

---

## Anti-Pattern 9: Iterating and Removing Simultaneously

**WRONG:**
```python
for wall in model.by_type("IfcWall"):
    if should_remove(wall):
        model.remove(wall)  # Invalidates the iterator!
# Skips elements or crashes due to collection modification during iteration
```

**CORRECT:**
```python
walls_to_remove = [w for w in model.by_type("IfcWall") if should_remove(w)]
for wall in walls_to_remove:
    ifcopenshell.api.run("root.remove_product", model, product=wall)
```

**Why:** `model.by_type()` returns a view of the internal collection. Removing elements during iteration invalidates the iterator, causing skipped elements or crashes.

---

## Anti-Pattern 10: Caching Entity References Across Modifications

**WRONG:**
```python
wall = model.by_type("IfcWall")[0]
# ... later, after other modifications ...
model.undo()
# wall reference is NOW INVALID — the C++ object may be destroyed or different
print(wall.Name)  # RuntimeError or undefined behavior
```

**CORRECT:**
```python
wall = model.by_type("IfcWall")[0]
wall_id = wall.id()  # Save the STEP ID

# ... after modifications or undo ...
model.undo()

# Re-fetch by ID
wall = model.by_id(wall_id)
print(wall.Name)  # Safe — fresh reference
```

**Why:** Entity wrappers are thin wrappers around C++ pointers. After `model.remove()`, `model.undo()`, or `model.redo()`, cached entity references may point to freed memory or different entities.

---

## Anti-Pattern 11: Missing OwnerHistory in IFC2X3

**WRONG:**
```python
model = ifcopenshell.file(schema="IFC2X3")
wall = model.create_entity("IfcWallStandardCase",
    GlobalId=ifcopenshell.guid.new(),
    Name="Wall 001")
# NO OwnerHistory — file is INVALID for IFC2X3
```

**CORRECT:**
```python
model = ifcopenshell.file(schema="IFC2X3")
# Use API — it creates and assigns OwnerHistory automatically
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallStandardCase", name="Wall 001")
```

**Why:** IFC2X3 REQUIRES OwnerHistory on most rooted entities. IFC4 and IFC4X3 made it OPTIONAL. Using `ifcopenshell.api.run("root.create_entity", ...)` handles this difference automatically.

---

## Anti-Pattern 12: Geometry Processing Without Settings

**WRONG:**
```python
# Missing settings argument
shape = ifcopenshell.geom.create_shape(wall)
# TypeError: missing required argument 'settings'

# Using default settings without world coordinates
settings = ifcopenshell.geom.settings()
shape = ifcopenshell.geom.create_shape(settings, wall)
# Coordinates are in LOCAL space — not world position
```

**CORRECT:**
```python
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

try:
    shape = ifcopenshell.geom.create_shape(settings, wall)
except RuntimeError:
    shape = None
```

**Why:** `create_shape()` requires a settings object as the first argument. Without `USE_WORLD_COORDS=True`, coordinates are in the element's local coordinate system, which is rarely what you want for analysis or visualization.

---

## Anti-Pattern 13: Processing All Elements for Geometry

**WRONG:**
```python
# Tries to process geometry for ALL products — many have no geometry
for product in model.by_type("IfcProduct"):
    shape = ifcopenshell.geom.create_shape(settings, product)
    # Fails on IfcSite, IfcBuilding, IfcProject, etc.
```

**CORRECT:**
```python
# Filter to elements with actual geometry
for product in model.by_type("IfcProduct"):
    if not hasattr(product, "Representation") or product.Representation is None:
        continue

    has_body = any(
        r.RepresentationIdentifier in ("Body", "Facetation", "Tessellation")
        for r in product.Representation.Representations
    )
    if not has_body:
        continue

    try:
        shape = ifcopenshell.geom.create_shape(settings, product)
    except RuntimeError:
        continue
```

**Why:** Spatial structure elements (IfcSite, IfcBuilding, IfcBuildingStorey) and some abstract elements have no 3D geometry representation. Attempting to process them raises `RuntimeError`.

---

## Anti-Pattern 14: Using create_entity Instead of API for Complex Operations

**WRONG:**
```python
# Manual property set creation — verbose, error-prone, missing relationship
prop = model.create_entity("IfcPropertySingleValue",
    Name="IsExternal",
    NominalValue=model.create_entity("IfcBoolean", wrappedValue=True))
pset = model.create_entity("IfcPropertySet",
    GlobalId=ifcopenshell.guid.new(),
    Name="Pset_WallCommon",
    HasProperties=[prop])
# MISSING: IfcRelDefinesByProperties to link pset to element!
```

**CORRECT:**
```python
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"IsExternal": True})
# API creates the relationship entity automatically
```

**Why:** `create_entity()` is a low-level function that creates individual entities without relationships. For operations involving relationships (property sets, spatial containment, type assignment, material association), ALWAYS use `ifcopenshell.api.run()` which creates the complete entity graph.

---

## Anti-Pattern 15: Using get_info(recursive=True) on Large Models

**WRONG:**
```python
# Materializes the ENTIRE entity graph into Python dicts
for element in model.by_type("IfcProduct"):
    info = element.get_info(recursive=True)
    # Memory usage explodes — each element expands all references recursively
```

**CORRECT:**
```python
# Use non-recursive get_info (default)
for element in model.by_type("IfcProduct"):
    info = element.get_info()  # Only top-level attributes
    # Access specific nested data as needed
    if element.ObjectPlacement:
        placement_info = element.ObjectPlacement.get_info()
```

**Why:** `get_info(recursive=True)` traverses ALL entity references recursively, converting the entire subgraph to Python dicts. For elements with complex geometry or deep property trees, this can consume gigabytes of memory.

---

## Anti-Pattern 16: Forgetting Spatial Containment

**WRONG:**
```python
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
model.write("output.ifc")
# Wall exists in the file but has NO spatial location
# It will not appear in spatial trees in IFC viewers
```

**CORRECT:**
```python
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
model.write("output.ifc")
```

**Why:** IFC elements MUST be assigned to a spatial container (IfcBuildingStorey, IfcSpace, etc.) through IfcRelContainedInSpatialStructure. Without this relationship, elements are "floating" and will not appear in spatial navigation trees in IFC viewers.

---

## Anti-Pattern 17: Wrong Type Matching Across Schemas

**WRONG:**
```python
# IFC2X3 uses IfcDoorStyle, not IfcDoorType
door_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoorType", name="Single Door")
# RuntimeError in IFC2X3: 'IfcDoorType' not found
```

**CORRECT:**
```python
schema = model.schema
if schema == "IFC2X3":
    door_type_class = "IfcDoorStyle"
else:
    door_type_class = "IfcDoorType"

door_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class=door_type_class, name="Single Door")
```

**Why:** IFC2X3 uses "Style" suffix (IfcDoorStyle, IfcWindowStyle) where IFC4/IFC4X3 uses "Type" suffix (IfcDoorType, IfcWindowType). The API may abstract some of this, but when specifying `ifc_class` directly, you MUST use the correct name for the schema.
