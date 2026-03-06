---
name: ifcos-core-runtime
description: "Explains IfcOpenShell Python runtime quirks including C++ binding behavior, entity identity (is vs ==), entity invalidation after removal, by_type() return semantics, thread safety constraints, memory management, PascalCase attribute access, and installation patterns. Activates when debugging IfcOpenShell crashes, entity reference errors, or performance issues."
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Python Runtime

## Quick Reference

### Critical Warnings

- **ALWAYS** use `==` for entity comparison, NEVER `is`. Each query returns a new Python wrapper object.
- **ALWAYS** set entity references to `None` after calling `model.remove()`. The C++ object is deallocated regardless of Python reference count.
- **ALWAYS** keep a reference to the `ifcopenshell.file` object alive while any entities from it are in use. If the file is garbage-collected, all entity wrappers become dangling pointers (segfault).
- **NEVER** write to an `ifcopenshell.file` from multiple threads. The C++ backend has no locking. Concurrent writes corrupt the model or crash.
- **NEVER** iterate all entities and filter manually when `by_type()` exists. `by_type()` uses an internal index and is 10-100x faster.
- **NEVER** use `get_info(recursive=True)` on large models. It materializes the entire entity graph into memory.
- **ALWAYS** use named attribute access (`wall.Name`) instead of positional index access (`wall[2]`). Positional indices vary by entity type and schema version.
- **ALWAYS** check `model.schema` before accessing schema-specific attributes.

### Decision Tree: Entity Reference Safety

```
Working with entity references?
├── Reading attributes?
│   ├── One attribute → entity.AttributeName (PascalCase)
│   ├── All attributes → entity.get_info() (returns dict)
│   └── Bulk read → entity.get_info(scalar_only=True) (faster)
│
├── Comparing entities?
│   ├── Same entity? → entity_a == entity_b (value equality)
│   ├── Same STEP ID? → entity_a.id() == entity_b.id()
│   └── NEVER use → entity_a is entity_b (always False)
│
├── Removing entities?
│   ├── With relationship cleanup → ifcopenshell.api.run("root.remove_product", model, product=entity)
│   ├── Low-level removal → model.remove(entity) (does NOT clean relationships)
│   └── After removal → entity_ref = None (ALWAYS nullify)
│
└── Storing references across operations?
    ├── Store entity.id() instead of the entity object
    ├── Re-fetch with model.by_id(stored_id) when needed
    └── NEVER cache entity objects across remove/undo operations
```

### Decision Tree: Performance

```
Performance-critical operation?
├── Querying entities by type?
│   ├── Use model.by_type("IfcWall") → indexed, returns tuple
│   ├── First call builds index (slow), subsequent calls are instant
│   └── NEVER iterate all entities with manual is_a() filtering
│
├── Reading many attributes?
│   ├── Single entity → entity.get_info() (one C++ round-trip)
│   ├── Scalar values only → entity.get_info(scalar_only=True)
│   └── AVOID get_info(recursive=True) (massive memory allocation)
│
├── Bulk entity creation?
│   ├── < 100 entities → ifcopenshell.api.run() (safe, handles metadata)
│   ├── > 100 entities → model.create_entity() (5-10x faster, no undo tracking)
│   └── Bulk mode → handle GlobalId, OwnerHistory (IFC2X3) manually
│
└── Large model (500MB+ / 100k+ entities)?
    ├── Memory: ifcopenshell.open() loads ENTIRE file into RAM
    ├── Streaming: ifcopenshell.open(path, should_stream=True) (sequential only)
    ├── Parallel: open SEPARATE file instances per thread
    └── Cleanup: del model + gc.collect() to release C++ memory
```

---

## Essential Patterns

### Pattern 1: C++ Binding Architecture

IfcOpenShell Python objects are thin wrappers around C++ objects managed by the `ifcopenshell_wrapper` module.

```python
# Schema-agnostic
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Python wrapper delegates to C++ core
type(wall)                    # <class 'ifcopenshell.entity_instance'>
type(wall.wrapped_data)       # <class 'ifcopenshell_wrapper.entity_instance'>

# The wrapper holds a reference to its parent file
wall.file  # Returns the ifcopenshell.file object
```

**Key implication:** Python garbage collection does NOT control C++ memory. The C++ backend allocates and deallocates independently. Calling `model.remove(entity)` frees the C++ object immediately, even if Python references still exist.

### Pattern 2: Entity Identity

```python
# Schema-agnostic
wall = model.by_type("IfcWall")[0]

# Two queries for the same entity return DIFFERENT Python wrapper objects
wall_a = model.by_id(wall.id())
wall_b = model.by_id(wall.id())

wall_a == wall_b    # True  — value equality via wrapped_data comparison
wall_a is wall_b    # False — different Python wrapper objects

# ALWAYS compare with == or by STEP ID
wall_a.id() == wall_b.id()  # True — same STEP ID (#42)
```

### Pattern 3: Entity Invalidation After Removal

```python
# Schema-agnostic
wall = model.by_type("IfcWall")[0]
wall_id = wall.id()

# Low-level removal deallocates the C++ object
model.remove(wall)

# DANGER: accessing wall after removal causes undefined behavior
# wall.Name  → segfault or garbage data

# ALWAYS nullify references after removal
wall = None

# To re-check existence, query by ID
try:
    still_exists = model.by_id(wall_id)
except RuntimeError:
    still_exists = None  # Entity was removed

# PREFER API removal for products (cleans up relationships)
other_wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run("root.remove_product", model, product=other_wall)
other_wall = None  # Still must nullify
```

### Pattern 4: by_type() Return Semantics

```python
# Schema-agnostic
walls = model.by_type("IfcWall")

type(walls)   # <class 'tuple'> — NOT a list
len(walls)    # Number of IfcWall instances

# First call for a type builds an internal C++ index (slower)
# Subsequent calls for the same type use the cached index (instant)

# To mutate the result, convert to list
wall_list = list(walls)
wall_list.append(some_other_entity)

# Include subtypes (default behavior)
all_walls = model.by_type("IfcWall")  # Includes IfcWallStandardCase in IFC2X3/IFC4

# Exclude subtypes
only_walls = model.by_type("IfcWall", include_subtypes=False)
```

### Pattern 5: PascalCase Attribute Access

```python
# Schema-agnostic
wall = model.by_type("IfcWall")[0]

# Named access — PascalCase matching the IFC schema
name = wall.Name                    # str or None
global_id = wall.GlobalId           # str (22-char encoded GUID)
description = wall.Description      # str or None
object_type = wall.ObjectType       # str or None
owner_history = wall.OwnerHistory   # entity_instance or None

# Positional access — fragile, varies by type and schema
name = wall[2]  # AVOID: index 2 = Name for IfcWall, but varies for other types

# Check if attribute has a value (IFC $null → Python None)
if wall.Description is not None:
    print(wall.Description)

# is_a() checks the full class hierarchy
wall.is_a("IfcWall")              # True
wall.is_a("IfcBuildingElement")   # True (parent class)
wall.is_a("IfcProduct")           # True (grandparent)
wall.is_a("IfcSlab")              # False

# get_info() returns all attributes as a dict
info = wall.get_info()
# {"id": 42, "type": "IfcWall", "GlobalId": "...", "Name": "...", ...}
```

### Pattern 6: Thread Safety

```python
# Schema-agnostic

# SAFE: Read-only operations from multiple threads (no concurrent writes)
import concurrent.futures

def count_type(filepath, ifc_class):
    model = ifcopenshell.open(filepath)  # Separate instance per thread
    return len(model.by_type(ifc_class))

with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = {
        executor.submit(count_type, "model.ifc", cls): cls
        for cls in ["IfcWall", "IfcSlab", "IfcDoor"]
    }

# SAFE: Separate file instances per thread for writes
def process_copy(filepath, output_path):
    local_model = ifcopenshell.open(filepath)
    # Modify local_model freely — independent C++ instance
    local_model.write(output_path)

# UNSAFE: Multiple threads writing to the SAME model object
# This WILL corrupt data or crash — no C++ locks exist
```

### Pattern 7: Memory Management for Large Models

```python
# Schema-agnostic
# Standard open loads entire file into memory
model = ifcopenshell.open("large_building.ifc")  # May use 4-8GB RAM

# Access specific attributes instead of materializing everything
name = wall.Name                                          # 1 attribute
material = ifcopenshell.util.element.get_material(wall)   # Targeted query

# AVOID: get_info(recursive=True) on large models
# deep_info = wall.get_info(recursive=True)  # Massive memory allocation

# Release a large model from memory
del model
import gc
gc.collect()  # Ensures Python wrappers are cleaned up and C++ memory freed
```

### Pattern 8: File Lifecycle — Keep File Reference Alive

```python
# Schema-agnostic
# DANGEROUS: File reference lost while entities are in use
def get_walls():
    model = ifcopenshell.open("model.ifc")
    return model.by_type("IfcWall")  # model goes out of scope after return

walls = get_walls()
# model is garbage-collected → all entity wrappers are dangling pointers
# walls[0].Name  → SEGFAULT

# SAFE: Return both the model and the entities
def get_walls_safe():
    model = ifcopenshell.open("model.ifc")
    walls = model.by_type("IfcWall")
    return model, walls

model, walls = get_walls_safe()
walls[0].Name  # Safe — model reference kept alive
```

---

## Schema-Specific Attribute Differences

```python
# ALWAYS check schema before accessing schema-specific attributes
schema = model.schema  # "IFC2X3", "IFC4", "IFC4X3"

# Entities that differ by schema:
# IFC2X3/IFC4: IfcWallStandardCase exists
# IFC4X3: IfcWallStandardCase does NOT exist — use IfcWall

# IFC2X3: IfcDoorStyle, IfcWindowStyle
# IFC4+:  IfcDoorType, IfcWindowType

# IFC2X3: OwnerHistory is REQUIRED on rooted entities
# IFC4+:  OwnerHistory is OPTIONAL

# IFC4X3: IfcBuiltElement replaces IfcBuildingElement
# IFC4X3: New entities: IfcAlignment, IfcRoad, IfcBridge, IfcFacility

# Accessing a non-existent attribute raises AttributeError
try:
    pt = task.PredefinedType  # Exists in IFC4, not in IFC2X3
except AttributeError:
    pt = None
```

---

## Installation

### pip (Recommended for Most Users)

```bash
pip install ifcopenshell
python -c "import ifcopenshell; print(ifcopenshell.version)"
```

- Pre-built wheels for Python 3.8-3.12 on Linux, macOS, Windows
- Package name is `ifcopenshell` (all lowercase)
- Bundles C++ core and OpenCASCADE dependencies

### conda (Recommended for Complex Environments)

```bash
conda install -c conda-forge ifcopenshell
```

- Better dependency resolution for OpenCASCADE
- More up-to-date than pip releases

### Blender Integration

```bash
# Bonsai addon bundles ifcopenshell — no separate install needed
# For standalone install into Blender's Python:
/path/to/blender/python/bin/python -m pip install ifcopenshell
```

### Platform Notes

| Platform | Note |
|----------|------|
| Windows | pip works out-of-the-box. For Blender: install into Blender's Python. Watch for PATH conflicts. |
| macOS | Works on Intel and Apple Silicon. Use native arm64 Python, not Rosetta. |
| Linux | Works on most distributions. Headless/Docker: no GPU needed. |

---

## Version Detection

```python
import ifcopenshell

# Library version
ifcopenshell.version  # e.g., "0.8.1"

# Schema of an open file
model = ifcopenshell.open("model.ifc")
model.schema             # "IFC2X3", "IFC4", or "IFC4X3"
model.schema_identifier  # e.g., "IFC4_ADD2"
model.schema_version     # e.g., (4, 0, 2, 1)
```

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for entity_instance, file, by_type, remove, schema
- [Working Code Examples](references/examples.md) — End-to-end runtime patterns
- [Anti-Patterns](references/anti-patterns.md) — Common runtime mistakes and how to avoid them
