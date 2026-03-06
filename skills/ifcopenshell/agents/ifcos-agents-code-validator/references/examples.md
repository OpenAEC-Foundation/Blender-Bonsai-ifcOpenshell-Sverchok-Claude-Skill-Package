# Before/After Validation Fix Examples

Each example shows code that fails validation (BEFORE) and the corrected version (AFTER), with the validation rule that flagged it.

---

## Example 1: Schema-Agnostic Element Query

**Rule**: V-001 (Schema Check Before Entity Usage)
**Severity**: BLOCKER

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: assumes single schema
import ifcopenshell

model = ifcopenshell.open("building.ifc")

# BLOCKER: IfcBuildingElement does not exist in IFC4X3
elements = model.by_type("IfcBuildingElement")
for elem in elements:
    print(elem.Name)

# BLOCKER: IfcWallStandardCase does not exist in IFC4X3
walls = model.by_type("IfcWallStandardCase")
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")
schema = model.schema

# CORRECT: Version-aware entity selection
if schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")

for elem in elements:
    print(elem.Name)

# CORRECT: Version-safe wall query
if schema in ("IFC2X3", "IFC4"):
    walls = model.by_type("IfcWallStandardCase")
else:  # IFC4X3
    walls = model.by_type("IfcWall")
```

---

## Example 2: Safe Entity Creation with API

**Rule**: V-002 (API Module Existence), V-003 (List Parameters)
**Severity**: BLOCKER

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: multiple issues
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# BLOCKER: Low-level creation, no GlobalId, no OwnerHistory handling
wall = model.create_entity("IfcWall", Name="Wall 001")

# BLOCKER: Hallucinated API call
ifcopenshell.api.run("wall.create", model, name="Wall 002")

# BLOCKER: Singular parameter (v0.8+)
storey = model.by_type("IfcBuildingStorey")[0]
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)
```

### AFTER (passes validation)

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

# CORRECT: API handles GlobalId and OwnerHistory
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# CORRECT: Valid API module
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 002")

# CORRECT: List parameter
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall, wall2], relating_structure=storey)
```

---

## Example 3: Safe Entity Removal

**Rule**: V-004 (Direct Entity Removal), V-005 (Entity Reference After Removal), V-006 (Iterate-and-Remove)
**Severity**: BLOCKER

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: three BLOCKER issues
import ifcopenshell

model = ifcopenshell.open("building.ifc")

# BLOCKER (V-006): Iterate-and-remove
for wall in model.by_type("IfcWall"):
    # BLOCKER (V-004): Direct removal corrupts relationships
    model.remove(wall)

# Also wrong:
wall = model.by_type("IfcWall")[0]
wall_name = wall.Name
model.remove(wall)
# BLOCKER (V-005): Entity reference used after removal
print(f"Removed: {wall.GlobalId}")
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

# CORRECT: Collect first, then remove with API
walls_to_remove = list(model.by_type("IfcWall"))
for wall in walls_to_remove:
    ifcopenshell.api.run("root.remove_product", model, product=wall)

# CORRECT: Extract data before removal
wall = model.by_type("IfcWall")[0]
wall_data = {"name": wall.Name, "guid": wall.GlobalId}
ifcopenshell.api.run("root.remove_product", model, product=wall)
print(f"Removed: {wall_data['guid']}")
```

---

## Example 4: None-Safe Utility Calls

**Rule**: V-007 (None Return Value Check)
**Severity**: BLOCKER

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: no None checks
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")

for wall in model.by_type("IfcWall"):
    # BLOCKER: get_container returns None if wall has no spatial containment
    container = ifcopenshell.util.element.get_container(wall)
    print(f"Wall {wall.Name} is in {container.Name}")

    # BLOCKER: get_type returns None if no type is assigned
    wall_type = ifcopenshell.util.element.get_type(wall)
    print(f"Type: {wall_type.Name}")

    # BLOCKER: get_material returns None if no material is assigned
    material = ifcopenshell.util.element.get_material(wall)
    print(f"Material: {material.Name}")
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")

for wall in model.by_type("IfcWall"):
    container = ifcopenshell.util.element.get_container(wall)
    if container is not None:
        print(f"Wall {wall.Name} is in {container.Name}")
    else:
        print(f"Wall {wall.Name} has no spatial containment")

    wall_type = ifcopenshell.util.element.get_type(wall)
    if wall_type is not None:
        print(f"Type: {wall_type.Name}")
    else:
        print(f"Wall {wall.Name} has no type assigned")

    material = ifcopenshell.util.element.get_material(wall)
    if material is not None:
        print(f"Material: {material.Name}")
    else:
        print(f"Wall {wall.Name} has no material assigned")
```

---

## Example 5: Geometry Processing with Error Handling

**Rule**: V-010 (Geometry Processing Without Error Handling), V-011 (Geometry Iterator for Batch)
**Severity**: BLOCKER (V-010), WARNING (V-011)

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: no error handling, no iterator
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("building.ifc")
settings = ifcopenshell.geom.settings()

# WARNING (V-011): create_shape in loop instead of iterator
for wall in model.by_type("IfcWall"):
    # BLOCKER (V-010): No try/except for RuntimeError
    shape = ifcopenshell.geom.create_shape(settings, wall)
    verts = shape.geometry.verts
    faces = shape.geometry.faces
    print(f"{wall.Name}: {len(verts)//3} vertices")
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("building.ifc")
settings = ifcopenshell.geom.settings()

# CORRECT: Use iterator for batch processing
walls = model.by_type("IfcWall")
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(), include=walls
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        verts = shape.geometry.verts
        faces = shape.geometry.faces
        print(f"{element.Name}: {len(verts)//3} vertices")
        if not iterator.next():
            break
```

For single-element geometry (interactive/debug), keep `create_shape()` but add error handling:

```python
# IfcOpenShell — all schema versions (single element)
try:
    shape = ifcopenshell.geom.create_shape(settings, wall)
    verts = shape.geometry.verts
except RuntimeError as e:
    print(f"Geometry failed for {wall.is_a()} #{wall.id()}: {e}")
    shape = None
```

---

## Example 6: Unit-Aware Coordinate Extraction

**Rule**: V-009 (Unit Scale Application)
**Severity**: WARNING

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: assumes meters
import ifcopenshell
import ifcopenshell.util.placement

model = ifcopenshell.open("building.ifc")

for wall in model.by_type("IfcWall"):
    if wall.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            wall.ObjectPlacement)
        # WARNING: Raw coordinates may be in mm, feet, or inches
        x, y, z = matrix[:, 3][:3]
        print(f"{wall.Name}: ({x:.2f}, {y:.2f}, {z:.2f})")
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.placement
import ifcopenshell.util.unit

model = ifcopenshell.open("building.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

for wall in model.by_type("IfcWall"):
    if wall.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            wall.ObjectPlacement)
        x, y, z = matrix[:, 3][:3] * unit_scale
        print(f"{wall.Name}: ({x:.2f}, {y:.2f}, {z:.2f}) meters")
```

---

## Example 7: Duplicate Property Set Prevention

**Rule**: V-012 (Duplicate Property Set Creation)
**Severity**: WARNING

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: creates duplicate psets
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

for wall in model.by_type("IfcWall"):
    # WARNING: Does not check if Pset_WallCommon already exists
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")
    ifcopenshell.api.run("pset.edit_pset", model,
        pset=pset, properties={"IsExternal": True})
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api
from ifcopenshell.util.element import get_psets

model = ifcopenshell.open("building.ifc")

for wall in model.by_type("IfcWall"):
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

## Example 8: IFC2X3 OwnerHistory Handling

**Rule**: V-013 (OwnerHistory in IFC2X3)
**Severity**: BLOCKER

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: missing OwnerHistory in IFC2X3
import ifcopenshell

model = ifcopenshell.file(schema="IFC2X3")

# BLOCKER: OwnerHistory is REQUIRED in IFC2X3
wall = model.create_entity("IfcWallStandardCase",
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=None,
    Name="Wall 001")
```

### AFTER (passes validation)

```python
# IfcOpenShell — IFC2X3
import ifcopenshell
import ifcopenshell.api

# CORRECT: API handles OwnerHistory automatically
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallStandardCase", name="Wall 001")
# API creates and assigns OwnerHistory for IFC2X3
```

---

## Example 9: Batch API Operations

**Rule**: V-015 (Per-Element API Calls Instead of Batch)
**Severity**: WARNING

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: per-element container assignment
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")
storey = model.by_type("IfcBuildingStorey")[0]

# WARNING: Creates N separate IfcRelContainedInSpatialStructure entities
for wall in model.by_type("IfcWall"):
    ifcopenshell.api.run("spatial.assign_container", model,
        products=[wall], relating_structure=storey)
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")
storey = model.by_type("IfcBuildingStorey")[0]
walls = list(model.by_type("IfcWall"))

# CORRECT: Single API call, one relationship entity
ifcopenshell.api.run("spatial.assign_container", model,
    products=walls, relating_structure=storey)
```

---

## Example 10: Transaction Safety

**Rule**: V-014 (Transaction Safety)
**Severity**: BLOCKER

### BEFORE (fails validation)

```python
# IfcOpenShell — WRONG: unsafe transaction handling
import ifcopenshell

model = ifcopenshell.open("building.ifc")

model.begin_transaction()
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Test Wall")

# BLOCKER: If an exception occurs here, transaction is never closed

# BLOCKER: Calling undo inside active transaction
model.undo()

model.end_transaction()
```

### AFTER (passes validation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")

model.begin_transaction()
try:
    wall = model.create_entity("IfcWall",
        GlobalId=ifcopenshell.guid.new(), Name="Test Wall")
    model.end_transaction()
except Exception:
    model.discard_transaction()
    raise

# Undo AFTER transaction is closed
model.undo()
```

---

## Example 11: Complete Validation Pass — Production-Quality Script

This example shows a complete script that passes all validation rules.

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element
import ifcopenshell.util.unit
import ifcopenshell.geom
import multiprocessing

def analyze_walls(filepath):
    """Extract wall data from an IFC file with full validation compliance."""
    model = ifcopenshell.open(filepath)
    schema = model.schema
    unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

    # V-001: Schema-aware entity selection
    if schema == "IFC4X3":
        walls = model.by_type("IfcWall")
    elif schema == "IFC4":
        walls = model.by_type("IfcWall")
    else:  # IFC2X3
        walls = model.by_type("IfcWallStandardCase")

    results = []
    for wall in walls:
        data = {"guid": wall.GlobalId, "name": wall.Name}

        # V-007: None-safe utility calls
        container = ifcopenshell.util.element.get_container(wall)
        data["storey"] = container.Name if container is not None else None

        wall_type = ifcopenshell.util.element.get_type(wall)
        data["type"] = wall_type.Name if wall_type is not None else None

        # V-012: Check existing psets before creating
        existing = ifcopenshell.util.element.get_psets(wall)
        data["is_external"] = existing.get(
            "Pset_WallCommon", {}).get("IsExternal")

        results.append(data)

    # V-011: Use iterator for batch geometry
    settings = ifcopenshell.geom.settings()
    wall_entities = model.by_type("IfcWall")
    if wall_entities:
        iterator = ifcopenshell.geom.iterator(
            settings, model, multiprocessing.cpu_count(),
            include=wall_entities
        )
        if iterator.initialize():
            while True:
                shape = iterator.get()
                element = model.by_id(shape.id)
                vert_count = len(shape.geometry.verts) // 3
                for r in results:
                    if r["guid"] == element.GlobalId:
                        r["vertex_count"] = vert_count
                        break
                if not iterator.next():
                    break

    return results
```
