# Anti-Patterns Catalog

Anti-patterns detected by the IfcOpenShell Code Validator, with explanations and fixes.

---

## Anti-Pattern 1: Schema-Blind Entity Usage

**Severity**: BLOCKER
**Rule**: V-001
**Frequency**: Very common in AI-generated code

### Description

Code uses IFC entity names without checking `model.schema`, assuming a single schema version. This causes `RuntimeError` when the file uses a different schema than expected.

### Pattern to Detect

```python
# ANTI-PATTERN: No schema check before schema-specific entities
model = ifcopenshell.open("file.ifc")
elements = model.by_type("IfcBuildingElement")    # Fails on IFC4X3
walls = model.by_type("IfcWallStandardCase")       # Fails on IFC4X3
doors = model.by_type("IfcDoorStyle")              # Fails on IFC4+
```

### Why This Is Wrong

- `IfcBuildingElement` was renamed to `IfcBuiltElement` in IFC4X3
- All `*StandardCase` entities were removed in IFC4X3
- `IfcDoorStyle`/`IfcWindowStyle` were replaced by `IfcDoorType`/`IfcWindowType` in IFC4

### Correct Pattern

```python
# CORRECT: Schema-aware branching
schema = model.schema
if schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")
```

---

## Anti-Pattern 2: Direct model.remove() on Products

**Severity**: BLOCKER
**Rule**: V-004
**Frequency**: Very common

### Description

Using `model.remove(entity)` directly on IFC products leaves dangling references in relationship entities (`IfcRelContainedInSpatialStructure`, `IfcRelDefinesByProperties`, `IfcRelDefinesByType`, `IfcRelAssociatesMaterial`). The resulting file is corrupted.

### Pattern to Detect

```python
# ANTI-PATTERN: Direct removal
wall = model.by_type("IfcWall")[0]
model.remove(wall)
# IfcRelContainedInSpatialStructure still references the deleted wall
# IfcRelDefinesByProperties still references the deleted wall
```

### Why This Is Wrong

`model.remove()` is a low-level C++ operation that deletes the entity from memory but does NOT clean up any IFC relationship entities that reference it. The result:
- `IfcRelContainedInSpatialStructure.RelatedElements` contains a null reference
- `IfcRelDefinesByProperties.RelatedObjects` contains a null reference
- File validators reject the output
- Downstream tools crash on null entity references

### Correct Pattern

```python
# CORRECT: API handles all relationship cleanup
ifcopenshell.api.run("root.remove_product", model, product=wall)
```

---

## Anti-Pattern 3: Iterate-and-Remove

**Severity**: BLOCKER
**Rule**: V-006
**Frequency**: Common

### Description

Removing entities inside a `for` loop that iterates over `model.by_type()`. The removal invalidates the iteration sequence, causing skipped entities or crashes.

### Pattern to Detect

```python
# ANTI-PATTERN: Modifying collection during iteration
for wall in model.by_type("IfcWall"):
    model.remove(wall)        # Invalidates iteration
    # OR
    ifcopenshell.api.run("root.remove_product", model, product=wall)
```

### Why This Is Wrong

`model.by_type()` returns a tuple from an internal index. Removing entities modifies that index during iteration, causing undefined behavior: skipped elements, double-processing, or C++ segfaults.

### Correct Pattern

```python
# CORRECT: Materialize list first, then remove
walls_to_remove = list(model.by_type("IfcWall"))
for wall in walls_to_remove:
    ifcopenshell.api.run("root.remove_product", model, product=wall)
```

---

## Anti-Pattern 4: Using Invalidated Entity References

**Severity**: BLOCKER
**Rule**: V-005
**Frequency**: Common

### Description

Accessing attributes of an IFC entity after it has been removed from the model. Entity wrappers are C++ pointers; after removal, the underlying object is destroyed.

### Pattern to Detect

```python
# ANTI-PATTERN: Using entity after removal
wall = model.by_type("IfcWall")[0]
model.remove(wall)
print(wall.Name)      # RuntimeError or segfault
print(wall.GlobalId)  # RuntimeError or segfault
```

### Why This Is Wrong

IfcOpenShell entity instances are thin Python wrappers around C++ objects. When `model.remove()` is called, the C++ object is freed. Any subsequent access to the Python wrapper triggers undefined behavior — crashes, garbage data, or segfaults.

### Correct Pattern

```python
# CORRECT: Extract data BEFORE removal
wall = model.by_type("IfcWall")[0]
wall_data = {"name": wall.Name, "guid": wall.GlobalId, "id": wall.id()}
ifcopenshell.api.run("root.remove_product", model, product=wall)
# Use wall_data (plain dict) from here
print(wall_data["name"])
```

---

## Anti-Pattern 5: create_shape() in a Loop for Batch Geometry

**Severity**: WARNING
**Rule**: V-011
**Frequency**: Very common in AI-generated code

### Description

Calling `ifcopenshell.geom.create_shape(settings, element)` inside a `for` loop to process multiple elements. This is 5-10x slower than using `ifcopenshell.geom.iterator`.

### Pattern to Detect

```python
# ANTI-PATTERN: Sequential geometry processing
settings = ifcopenshell.geom.settings()
for wall in model.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)
    process(shape)
```

### Why This Is Wrong

- `create_shape()` is single-threaded; processes one element at a time
- No geometry caching; identical geometries are recomputed
- No automatic error skipping; one failure stops the loop (unless wrapped in try/except)
- For 1000 elements: ~60 seconds vs ~8 seconds with iterator (8 cores)

### Correct Pattern

```python
# CORRECT: Iterator with multi-threading and caching
import multiprocessing

settings = ifcopenshell.geom.settings()
walls = model.by_type("IfcWall")
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(), include=walls)
if iterator.initialize():
    while True:
        shape = iterator.get()
        process(shape)
        if not iterator.next():
            break
```

---

## Anti-Pattern 6: Missing None Check on Utility Returns

**Severity**: BLOCKER
**Rule**: V-007
**Frequency**: Very common

### Description

Using return values from `ifcopenshell.util.element` functions without checking for None. Functions like `get_container()`, `get_type()`, and `get_material()` return None when no data is found.

### Pattern to Detect

```python
# ANTI-PATTERN: No None guard
container = ifcopenshell.util.element.get_container(wall)
print(container.Name)  # AttributeError if None

wall_type = ifcopenshell.util.element.get_type(wall)
print(wall_type.Name)  # AttributeError if None
```

### Why This Is Wrong

Not all IFC elements have spatial containment, type assignments, or materials. These are optional relationships in IFC. Accessing `.Name` on None raises `AttributeError`.

### Correct Pattern

```python
# CORRECT: Guard against None
container = ifcopenshell.util.element.get_container(wall)
if container is not None:
    print(container.Name)
```

---

## Anti-Pattern 7: Standard UUID for IFC GlobalId

**Severity**: BLOCKER
**Rule**: V-008
**Frequency**: Common

### Description

Using Python's `uuid.uuid4()` to generate GlobalId values for IFC entities. IFC requires 22-character base64-encoded GUIDs, not 36-character standard UUIDs.

### Pattern to Detect

```python
# ANTI-PATTERN: Standard UUID format
import uuid
wall = model.create_entity("IfcWall",
    GlobalId=str(uuid.uuid4()), Name="Wall")
# GlobalId is 36 chars, IFC requires 22 chars
```

### Why This Is Wrong

IFC GUIDs use a compressed base64 encoding defined in ISO 10303-11. Standard UUIDs are 36 characters; IFC GUIDs are 22 characters. Using standard UUIDs produces files that fail IFC validation.

### Correct Pattern

```python
# CORRECT: Use ifcopenshell.guid.new()
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Wall")

# BEST: Use API (auto-generates GUID)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
```

---

## Anti-Pattern 8: Per-Element Container Assignment

**Severity**: WARNING
**Rule**: V-015
**Frequency**: Common

### Description

Calling `spatial.assign_container` inside a loop with a single-element list creates N separate `IfcRelContainedInSpatialStructure` entities instead of one.

### Pattern to Detect

```python
# ANTI-PATTERN: Per-element assignment
for wall in model.by_type("IfcWall"):
    ifcopenshell.api.run("spatial.assign_container", model,
        products=[wall], relating_structure=storey)
# Creates 100 IfcRelContainedInSpatialStructure entities for 100 walls
```

### Why This Is Wrong

Each call creates a new `IfcRelContainedInSpatialStructure` entity. For 100 walls, this creates 100 relationship entities. A single call with all walls creates ONE relationship entity containing all walls, which is both more efficient and produces a cleaner IFC file.

### Correct Pattern

```python
# CORRECT: Single batch call
walls = list(model.by_type("IfcWall"))
ifcopenshell.api.run("spatial.assign_container", model,
    products=walls, relating_structure=storey)
# Creates ONE IfcRelContainedInSpatialStructure for all walls
```

---

## Anti-Pattern 9: Full Model Iteration for Type Filtering

**Severity**: WARNING
**Rule**: V-016
**Frequency**: Occasional

### Description

Iterating over all entities in the model and filtering by `is_a()` instead of using the O(1) `by_type()` index.

### Pattern to Detect

```python
# ANTI-PATTERN: O(n) scan of all entities
walls = [e for e in model if e.is_a("IfcWall")]

# ANTI-PATTERN: Same issue in loop form
for entity in model:
    if entity.is_a("IfcWall"):
        process(entity)
```

### Why This Is Wrong

IfcOpenShell maintains internal class indexes. `model.by_type("IfcWall")` performs an O(1) indexed lookup. Iterating all entities is O(n) where n is the total entity count (tens of thousands to millions in large files).

### Correct Pattern

```python
# CORRECT: O(1) indexed lookup
walls = model.by_type("IfcWall")

for wall in model.by_type("IfcWall"):
    process(wall)
```

---

## Anti-Pattern 10: Unguarded Geometry Processing

**Severity**: BLOCKER
**Rule**: V-010
**Frequency**: Very common

### Description

Calling `ifcopenshell.geom.create_shape()` without `try/except RuntimeError`. Geometry processing fails for many real-world IFC elements due to missing representations, invalid booleans, or corrupt geometry.

### Pattern to Detect

```python
# ANTI-PATTERN: No error handling
shape = ifcopenshell.geom.create_shape(settings, element)
verts = shape.geometry.verts
```

### Why This Is Wrong

`create_shape()` invokes OpenCASCADE for geometry computation. It raises `RuntimeError` for:
- Elements without representation (spatial elements like IfcSite)
- Invalid boolean operations (malformed solids)
- Missing or corrupt geometry definitions
- Complex geometry exceeding computation limits

In production files, 5-20% of elements can fail geometry processing.

### Correct Pattern

```python
# CORRECT: Error-handled geometry processing
try:
    shape = ifcopenshell.geom.create_shape(settings, element)
except RuntimeError as e:
    print(f"Geometry failed for {element.is_a()} #{element.id()}: {e}")
    shape = None

if shape is not None:
    verts = shape.geometry.verts
    faces = shape.geometry.faces
```

---

## Anti-Pattern 11: Duplicate Property Set Creation

**Severity**: WARNING
**Rule**: V-012
**Frequency**: Common

### Description

Creating a property set with `pset.add_pset` without first checking if a property set with that name already exists on the element. This produces duplicate property sets.

### Pattern to Detect

```python
# ANTI-PATTERN: No existence check
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"IsExternal": True})
# If Pset_WallCommon already exists, wall now has TWO copies
```

### Why This Is Wrong

IFC files from authoring tools (Revit, ArchiCAD, Bonsai) already contain standard property sets. Blindly creating new ones produces duplicates, confusing BIM viewers and validation tools.

### Correct Pattern

```python
# CORRECT: Check first, create only if needed
from ifcopenshell.util.element import get_psets

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

## Anti-Pattern 12: Hallucinated API Calls

**Severity**: BLOCKER
**Rule**: V-002
**Frequency**: Very common in AI-generated code

### Description

AI models generate plausible-sounding but non-existent IfcOpenShell API calls. The 35 API modules have specific function names that must be used exactly.

### Pattern to Detect

```python
# ANTI-PATTERN: Hallucinated calls (NONE of these exist)
ifcopenshell.api.run("element.create", model, ...)
ifcopenshell.api.run("wall.create", model, ...)
ifcopenshell.api.run("wall.add", model, ...)
ifcopenshell.api.run("building.create", model, ...)
ifcopenshell.api.run("file.create", model, ...)
model.get_all_walls()
model.add_wall(name="W1")
wall.get_properties()
wall.set_property("Name", "Wall 1")
ifcopenshell.create_project()
```

### Why This Is Wrong

IfcOpenShell has a fixed set of 35 API modules. The module names and function names are specific. Using non-existent calls raises `KeyError` or `AttributeError`.

### Correct Pattern

```python
# CORRECT: Use verified API calls
ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="W1")
ifcopenshell.api.run("project.create_file", version="IFC4")
model.by_type("IfcWall")
ifcopenshell.util.element.get_psets(wall)
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "Wall 1"})
```

---

## Anti-Pattern 13: get_info(recursive=True) on Large Files

**Severity**: WARNING
**Rule**: V-017
**Frequency**: Occasional

### Description

Calling `element.get_info(recursive=True)` materializes the entire entity graph into Python dictionaries. On large files, this consumes gigabytes of memory.

### Pattern to Detect

```python
# ANTI-PATTERN: Recursive info extraction in loop
for wall in model.by_type("IfcWall"):
    info = wall.get_info(recursive=True)  # Materializes full entity graph
    process(info)
```

### Why This Is Wrong

`get_info(recursive=True)` follows all entity references recursively, converting every referenced entity to a Python dict. For a wall with placement, representation, material, property sets, and type references, this can generate thousands of nested dicts. In a loop over hundreds of elements, this exhausts available memory.

### Correct Pattern

```python
# CORRECT: Non-recursive info + specific attribute access
for wall in model.by_type("IfcWall"):
    info = wall.get_info()  # Immediate attributes only
    name = wall.Name
    guid = wall.GlobalId
    psets = ifcopenshell.util.element.get_psets(wall)
```

---

## Summary Table

| # | Anti-Pattern | Severity | Rule | Detection |
|---|-------------|----------|------|-----------|
| 1 | Schema-blind entity usage | BLOCKER | V-001 | Schema-specific entity names without `model.schema` check |
| 2 | Direct model.remove() on products | BLOCKER | V-004 | `model.remove()` called on IfcProduct subclasses |
| 3 | Iterate-and-remove | BLOCKER | V-006 | Removal inside `for entity in model.by_type()` loop |
| 4 | Invalidated entity references | BLOCKER | V-005 | Entity variable accessed after removal |
| 5 | create_shape() loop for batch | WARNING | V-011 | `create_shape()` inside loop for 10+ elements |
| 6 | Missing None check | BLOCKER | V-007 | Utility return accessed without None guard |
| 7 | Standard UUID for GlobalId | BLOCKER | V-008 | `uuid.uuid4()` in GlobalId context |
| 8 | Per-element container assignment | WARNING | V-015 | `assign_container` in loop with single element |
| 9 | Full model iteration | WARNING | V-016 | `for e in model` with `is_a()` filter |
| 10 | Unguarded geometry processing | BLOCKER | V-010 | `create_shape()` without `try/except RuntimeError` |
| 11 | Duplicate property set creation | WARNING | V-012 | `pset.add_pset` without `get_psets()` check |
| 12 | Hallucinated API calls | BLOCKER | V-002 | Non-existent API module or function names |
| 13 | get_info(recursive=True) in loops | WARNING | V-017 | Recursive info in iteration over large collections |
