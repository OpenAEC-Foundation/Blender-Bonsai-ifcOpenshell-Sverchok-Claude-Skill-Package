# ifcos-core-runtime — Anti-Patterns

## Anti-Pattern 1: Using `is` for Entity Comparison

```python
# WRONG
wall_a = model.by_id(42)
wall_b = model.by_id(42)
if wall_a is wall_b:  # ALWAYS False — different Python wrapper objects
    process(wall_a)

# CORRECT
if wall_a == wall_b:  # Value equality via wrapped_data comparison
    process(wall_a)

# ALSO CORRECT
if wall_a.id() == wall_b.id():  # Compare STEP IDs
    process(wall_a)
```

**Why:** Each call to `by_id()`, `by_type()`, or any query creates a new Python `entity_instance` wrapper around the same C++ object. Python's `is` checks object identity (memory address), which is different for each wrapper. Use `==` or compare `.id()` values.

---

## Anti-Pattern 2: Accessing Entities After Removal

```python
# WRONG
wall = model.by_type("IfcWall")[0]
model.remove(wall)
print(wall.Name)  # SEGFAULT or garbage data — C++ object is deallocated

# CORRECT
wall = model.by_type("IfcWall")[0]
wall_name = wall.Name  # Read BEFORE removal
model.remove(wall)
wall = None  # Nullify reference
print(wall_name)  # Use the previously read value
```

**Why:** `model.remove()` deallocates the C++ entity object immediately. Python garbage collection has no control over this — the C++ backend owns the memory. Any access to the wrapper after removal dereferences a freed pointer.

---

## Anti-Pattern 3: Letting the File Object Be Garbage-Collected

```python
# WRONG
def get_walls():
    model = ifcopenshell.open("building.ifc")
    return model.by_type("IfcWall")

walls = get_walls()
# model is garbage-collected — all wrappers are dangling pointers
walls[0].Name  # SEGFAULT

# CORRECT
def get_walls():
    model = ifcopenshell.open("building.ifc")
    return model, model.by_type("IfcWall")

model, walls = get_walls()
walls[0].Name  # Safe — model is alive
```

**Why:** Entity wrappers hold a reference to their parent file's C++ data. When the Python `ifcopenshell.file` object is garbage-collected, the C++ data store is freed. All entity wrappers from that file become dangling pointers.

---

## Anti-Pattern 4: Manual Iteration Instead of by_type()

```python
# WRONG (10-100x slower)
walls = []
for entity in model:
    if entity.is_a("IfcWall"):
        walls.append(entity)

# CORRECT
walls = model.by_type("IfcWall")  # Uses internal C++ index
```

**Why:** `by_type()` builds a C++ index on the first call for each type. Subsequent calls are O(1) lookups. Manual iteration visits every entity in the file and performs a type check on each one — this is O(n) on every call with no caching.

---

## Anti-Pattern 5: Concurrent Writes to a Shared Model

```python
# WRONG — data corruption or crash
import threading

def add_walls(model, start, count):
    for i in range(start, start + count):
        ifcopenshell.api.run("root.create_entity", model,
            ifc_class="IfcWall", name=f"Wall {i}")

t1 = threading.Thread(target=add_walls, args=(model, 0, 100))
t2 = threading.Thread(target=add_walls, args=(model, 100, 100))
t1.start()
t2.start()
t1.join()
t2.join()

# CORRECT — separate file instances per thread
def add_walls_safe(filepath, output, start, count):
    local_model = ifcopenshell.open(filepath)
    for i in range(start, start + count):
        ifcopenshell.api.run("root.create_entity", local_model,
            ifc_class="IfcWall", name=f"Wall {i}")
    local_model.write(output)
```

**Why:** The C++ backend has no mutex locks or atomic operations. Concurrent writes from multiple threads cause data races that corrupt the internal entity store or trigger memory access violations. Use one file instance per thread.

---

## Anti-Pattern 6: Using get_info(recursive=True) on Large Models

```python
# WRONG — massive memory allocation
model = ifcopenshell.open("large_building.ifc")  # 500MB file
wall = model.by_type("IfcWall")[0]
info = wall.get_info(recursive=True)  # Materializes entire entity graph into nested dicts

# CORRECT — access specific attributes
name = wall.Name
global_id = wall.GlobalId
material = ifcopenshell.util.element.get_material(wall)

# ALSO CORRECT — non-recursive get_info()
info = wall.get_info()  # Flat dict, entity references as entity_instance objects
scalar_info = wall.get_info(scalar_only=True)  # Scalars only, faster
```

**Why:** `recursive=True` follows every entity reference and converts the entire reachable graph into nested Python dicts. On a large model, this can allocate gigabytes of memory. Access specific attributes directly or use `get_info()` without recursion.

---

## Anti-Pattern 7: Positional Attribute Access

```python
# WRONG — fragile, index varies by entity type and schema version
wall = model.by_type("IfcWall")[0]
name = wall[2]  # Index 2 is Name for IfcWall, but NOT for other types

# CORRECT — named attribute access
name = wall.Name
global_id = wall.GlobalId
description = wall.Description
```

**Why:** Positional indices are determined by the entity's attribute order in the EXPRESS schema. Different entity types have different attribute orders, and future schema versions may change them. Named access is self-documenting and stable across schema versions.

---

## Anti-Pattern 8: Using model.remove() for Products

```python
# WRONG — orphaned relationships remain
wall = model.by_type("IfcWall")[0]
model.remove(wall)
# IfcRelContainedInSpatialStructure still references a null entity
# IfcRelDefinesByProperties still references a null entity

# CORRECT — API handles relationship cleanup
ifcopenshell.api.run("root.remove_product", model, product=wall)
wall = None
```

**Why:** `model.remove()` only removes the entity itself. It sets references from other entities to null, but does NOT remove the relationship entities (IfcRelContainedInSpatialStructure, IfcRelDefinesByProperties, etc.). These orphaned relationships create an invalid IFC file. The API function `root.remove_product` cleans up all incoming relationships.

---

## Anti-Pattern 9: Assuming Schema Without Checking

```python
# WRONG — crashes on IFC2X3 files
model = ifcopenshell.open("unknown_file.ifc")
wall = model.by_type("IfcWall")[0]
predefined_type = wall.PredefinedType  # AttributeError in IFC2X3

# CORRECT — check schema first
schema = model.schema
if schema in ("IFC4", "IFC4X3"):
    predefined_type = wall.PredefinedType
else:
    predefined_type = None

# ALSO WRONG — using IFC4X3-only entities without checking
facility = model.by_type("IfcFacility")  # Empty tuple in IFC2X3/IFC4
```

**Why:** IFC2X3, IFC4, and IFC4X3 have different entity types and attributes. `IfcWallStandardCase` exists in IFC2X3/IFC4 but not IFC4X3. `PredefinedType` exists on some entities in IFC4 but not IFC2X3. `IfcFacility` exists only in IFC4X3. ALWAYS check `model.schema` before accessing schema-specific features.

---

## Anti-Pattern 10: Removing Entities During Iteration

```python
# WRONG — modifying collection during iteration
for wall in model.by_type("IfcWall"):
    if wall.Name == "Temporary":
        model.remove(wall)  # Undefined behavior — tuple is invalidated

# CORRECT — collect IDs first, then remove
wall_ids_to_remove = [
    w.id() for w in model.by_type("IfcWall") if w.Name == "Temporary"
]
for wid in wall_ids_to_remove:
    try:
        wall = model.by_id(wid)
        ifcopenshell.api.run("root.remove_product", model, product=wall)
        wall = None
    except RuntimeError:
        pass  # Already removed (cascading)
```

**Why:** `by_type()` returns a tuple built from the internal index. Removing entities invalidates the index. Iterating over a stale tuple while modifying the underlying data store causes undefined behavior. Collect entity IDs first, then remove in a separate pass.

---

## Anti-Pattern 11: Caching Entity Objects Across Operations

```python
# WRONG — cached entities may be invalidated by undo
model.begin_transaction()
wall = model.by_type("IfcWall")[0]
model.end_transaction()
model.undo()
# wall may now be invalid if the transaction created it

# CORRECT — re-fetch after undo/redo
model.begin_transaction()
wall = model.by_type("IfcWall")[0]
wall_id = wall.id()
model.end_transaction()
model.undo()
try:
    wall = model.by_id(wall_id)  # Re-fetch to verify existence
except RuntimeError:
    wall = None
```

**Why:** Undo/redo operations modify the C++ entity store. Entities created within a transaction are deallocated when the transaction is undone. Cached Python wrappers become dangling pointers. ALWAYS re-fetch entities by ID after undo/redo operations.

---

## Anti-Pattern 12: Ignoring by_type() Return Type

```python
# WRONG — tuple has no append() method
walls = model.by_type("IfcWall")
walls.append(new_wall)  # AttributeError: 'tuple' object has no attribute 'append'

# CORRECT — convert to list if mutation is needed
walls = list(model.by_type("IfcWall"))
walls.append(new_wall)
```

**Why:** `by_type()` returns a tuple (immutable) for performance reasons. If you need to modify the collection, explicitly convert to a list first.

---

## Summary Table

| Anti-Pattern | Symptom | Fix |
|---|---|---|
| `is` comparison | Equality check always False | Use `==` or `.id()` comparison |
| Access after removal | Segfault or garbage data | Set reference to `None` after `remove()` |
| File GC'd | Segfault on entity access | Keep file reference alive |
| Manual iteration | 10-100x slower queries | Use `by_type()` |
| Concurrent writes | Data corruption or crash | One file instance per thread |
| `recursive=True` | Out-of-memory | Use direct attribute access |
| Positional access | Wrong attribute, schema-fragile | Use PascalCase named access |
| `model.remove()` for products | Orphaned relationships | Use `root.remove_product` API |
| Schema assumption | AttributeError | Check `model.schema` first |
| Remove during iteration | Undefined behavior | Collect IDs first, then remove |
| Cache across undo | Dangling pointer | Re-fetch by ID after undo/redo |
| Append to by_type() result | AttributeError on tuple | Convert to `list()` first |
