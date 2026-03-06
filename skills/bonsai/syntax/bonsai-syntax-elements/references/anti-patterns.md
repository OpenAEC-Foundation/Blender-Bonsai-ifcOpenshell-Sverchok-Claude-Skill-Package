# Bonsai Element Access — Anti-Patterns

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11
> Every anti-pattern includes: what's wrong, why it fails, and the correct approach.

---

## AP-01: Not Checking `tool.Ifc.get()` or `IfcStore.get_file()` for None

### WRONG

```python
import bonsai.tool as tool

model = tool.Ifc.get()
walls = model.by_type("IfcWall")  # AttributeError: 'NoneType' has no attribute 'by_type'
```

### CORRECT

```python
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")
walls = model.by_type("IfcWall")
```

### Why It Fails

`tool.Ifc.get()` returns `None` when no IFC project is loaded. This happens when:
- Blender was launched without opening an IFC file
- The project was closed or unloaded
- A Blender file without IFC data is open

Calling `.by_type()` or any other method on `None` raises `AttributeError`.

**Rule**: ALWAYS check for `None` before calling any method on the IFC model.

---

## AP-02: Assuming Every Blender Object Has an IFC Entity

### WRONG

```python
import bpy
import bonsai.tool as tool

for obj in bpy.data.objects:
    entity = tool.Ifc.get_entity(obj)
    print(f"{obj.name}: {entity.is_a()}")  # AttributeError if entity is None
```

### CORRECT

```python
import bpy
import bonsai.tool as tool

for obj in bpy.data.objects:
    entity = tool.Ifc.get_entity(obj)
    if entity is not None:
        print(f"{obj.name}: {entity.is_a()}")
    else:
        print(f"{obj.name}: not linked to IFC")
```

### Why It Fails

Not every Blender object is linked to an IFC entity. Cameras, lights, empties, helper objects, and any object created outside of Bonsai's IFC workflow have `ifc_definition_id == 0` and `tool.Ifc.get_entity()` returns `None`. Calling `.is_a()` on `None` raises `AttributeError`.

**Rule**: ALWAYS check the return value of `tool.Ifc.get_entity()` for `None` before accessing entity attributes.

---

## AP-03: Assuming Every IFC Entity Has a Blender Object

### WRONG

```python
import bonsai.tool as tool

model = tool.Ifc.get()
for wall in model.by_type("IfcWall"):
    obj = tool.Ifc.get_object(wall)
    obj.select_set(True)  # AttributeError if obj is None
```

### CORRECT

```python
import bonsai.tool as tool

model = tool.Ifc.get()
for wall in model.by_type("IfcWall"):
    obj = tool.Ifc.get_object(wall)
    if obj is not None:
        obj.select_set(True)
```

### Why It Fails

Not every IFC entity has a Blender scene representation. Entities without geometry (e.g., `IfcPropertySet`, `IfcMaterial`, `IfcRelContainedInSpatialStructure`) and entities that have not yet been loaded into the Blender scene return `None` from `tool.Ifc.get_object()`.

**Rule**: ALWAYS check the return value of `tool.Ifc.get_object()` for `None` before manipulating the Blender object.

---

## AP-04: Manually Setting `ifc_definition_id` Instead of Using `tool.Ifc.link()`

### WRONG

```python
import bpy
from bonsai.bim.ifc import IfcStore

obj = bpy.context.active_object
obj.BIMProperties.ifc_definition_id = entity.id()
# id_map and guid_map are NOT updated — get_object() will fail
```

### CORRECT

```python
import bonsai.tool as tool

tool.Ifc.link(entity, obj)
# Sets ifc_definition_id AND populates IfcStore.id_map AND IfcStore.guid_map
```

### Why It Fails

Setting `ifc_definition_id` directly only updates one direction of the bidirectional mapping. `IfcStore.id_map` and `IfcStore.guid_map` remain empty for this object. Subsequent calls to `tool.Ifc.get_object(entity)` return `None` because it looks up `IfcStore.id_map`, not the Blender property.

**Rule**: ALWAYS use `tool.Ifc.link()` to establish Blender-IFC mappings. NEVER set `ifc_definition_id` directly.

---

## AP-05: Using `model.by_id()` Without Try/Except for User-Provided IDs

### WRONG

```python
model = tool.Ifc.get()
user_id = int(input("Enter IFC step ID: "))
entity = model.by_id(user_id)  # RuntimeError if ID does not exist
```

### CORRECT

```python
model = tool.Ifc.get()
user_id = int(input("Enter IFC step ID: "))
try:
    entity = model.by_id(user_id)
except RuntimeError:
    print(f"No entity with step ID {user_id}")
    entity = None
```

### Why It Fails

`model.by_id()` raises `RuntimeError` when the given step ID does not exist in the IFC file. This is different from `model.by_type()` which returns an empty list when no entities match. When the step ID comes from user input, configuration, or external data, it may reference a deleted or non-existent entity.

**Rule**: ALWAYS wrap `model.by_id()` and `model.by_guid()` in try/except when the identifier comes from an external source.

---

## AP-06: Caching Entity References Across File Reloads

### WRONG

```python
import bonsai.tool as tool

# Cache entities at startup
model = tool.Ifc.get()
cached_walls = model.by_type("IfcWall")

# ... later, after user reloads or switches IFC file ...
for wall in cached_walls:
    print(wall.Name)  # Stale reference — may crash or return wrong data
```

### CORRECT

```python
import bonsai.tool as tool

# Always re-query from the current model
def get_walls():
    model = tool.Ifc.get()
    if model is None:
        return []
    return model.by_type("IfcWall")

# Use fresh references
for wall in get_walls():
    print(wall.Name)
```

### Why It Fails

`ifcopenshell.entity_instance` objects are bound to their parent `ifcopenshell.file`. When the IFC file is reloaded, closed, or replaced (via `tool.Ifc.set()`, `bpy.ops.bim.load_project()`, or undo/redo), all entity references from the previous file become stale. Accessing attributes on stale entities causes segmentation faults or returns incorrect data.

**Rule**: NEVER cache `ifcopenshell.entity_instance` objects across operations that may reload the IFC file. ALWAYS re-query from `tool.Ifc.get()`.

---

## AP-07: Running `tool.Ifc.*` Outside Blender Context

### WRONG

```python
# standalone_script.py — run with system Python
import bonsai.tool as tool  # ImportError: No module named 'bpy'
entity = tool.Ifc.get_entity(some_object)
```

```bash
$ python3 standalone_script.py
# ImportError: No module named 'bpy'
```

### CORRECT

```bash
# Run via Blender's embedded Python
$ blender --background --python standalone_script.py
```

Or for standalone IFC processing without Bonsai:

```python
# No Bonsai needed — use ifcopenshell directly
import ifcopenshell

model = ifcopenshell.open("project.ifc")
walls = model.by_type("IfcWall")
for wall in walls:
    print(f"{wall.Name}: {wall.GlobalId}")
```

### Why It Fails

`bonsai.tool.*` classes import `bpy` (Blender's Python API module), which is only available inside Blender's embedded Python interpreter. System Python does not have `bpy` installed. For headless IFC operations that do not need Bonsai's UI or Blender scene integration, use `ifcopenshell` directly.

**Rule**: ALWAYS run Bonsai code via `blender --python`. For standalone IFC work, use `ifcopenshell` directly without importing `bonsai.*`.

---

## AP-08: Using `blenderbim.*` Import Paths

### WRONG

```python
from blenderbim.bim.ifc import IfcStore  # ModuleNotFoundError
import blenderbim.tool as tool            # ModuleNotFoundError
```

### CORRECT

```python
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
```

### Why It Fails

Bonsai was renamed from BlenderBIM in 2024 (v0.8.0). ALL module paths changed from `blenderbim.*` to `bonsai.*`. Code using the old import paths raises `ModuleNotFoundError` in Bonsai v0.8.0+.

| Old (BlenderBIM) | New (Bonsai v0.8.0+) |
|---|---|
| `blenderbim.bim.ifc` | `bonsai.bim.ifc` |
| `blenderbim.tool` | `bonsai.tool` |
| `blenderbim.core` | `bonsai.core` |

**Rule**: ALWAYS use `bonsai.*` import paths. NEVER use `blenderbim.*`.

---

## AP-09: Accessing `IfcStore.file` Directly Instead of `IfcStore.get_file()`

### WRONG

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.file  # Direct attribute access — no None safety
walls = model.by_type("IfcWall")  # AttributeError if file is None
```

### CORRECT

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")
walls = model.by_type("IfcWall")
```

### Why It Fails

`IfcStore.file` is a raw attribute that may be `None`. Using `IfcStore.get_file()` is the canonical access method. While both return the same value, using the method makes the None-check idiom consistent and explicit. Direct attribute access encourages skipping the None check.

**Rule**: ALWAYS use `IfcStore.get_file()` or `tool.Ifc.get()`. Avoid accessing `IfcStore.file` directly.

---

## AP-10: Using `model.by_type()` with `include_subtypes=False` Unintentionally

### WRONG

```python
model = tool.Ifc.get()
# In IFC2X3, IfcWallStandardCase is a subtype of IfcWall
walls = model.by_type("IfcWall", include_subtypes=False)
# Misses IfcWallStandardCase instances in IFC2X3 files
```

### CORRECT

```python
model = tool.Ifc.get()
# Default: include_subtypes=True — gets IfcWall AND IfcWallStandardCase
walls = model.by_type("IfcWall")
```

### Why It Fails

In IFC2X3, many element classes have `*StandardCase` subtypes (e.g., `IfcWallStandardCase`, `IfcSlabStandardCase`). Using `include_subtypes=False` skips these subtypes, returning an incomplete list. In IFC4+, the `*StandardCase` subtypes were deprecated, so this issue is less common but still applies to other subtype hierarchies.

**Rule**: ALWAYS use the default `include_subtypes=True` unless you explicitly need to exclude subtypes. Be aware of schema-specific subtype hierarchies.

---

## Summary: Quick Reference

| # | Anti-Pattern | Rule |
|---|-------------|------|
| 01 | No None check on `tool.Ifc.get()` | ALWAYS check for None before using the model |
| 02 | Assuming all objects have IFC entities | ALWAYS check `get_entity()` result for None |
| 03 | Assuming all entities have Blender objects | ALWAYS check `get_object()` result for None |
| 04 | Manually setting `ifc_definition_id` | ALWAYS use `tool.Ifc.link()` for mapping |
| 05 | Uncaught `by_id()` / `by_guid()` errors | ALWAYS try/except with external IDs |
| 06 | Caching entity references across reloads | ALWAYS re-query from current model |
| 07 | Running `tool.Ifc.*` outside Blender | ALWAYS run via `blender --python` |
| 08 | Using `blenderbim.*` imports | ALWAYS use `bonsai.*` |
| 09 | Accessing `IfcStore.file` directly | ALWAYS use `IfcStore.get_file()` or `tool.Ifc.get()` |
| 10 | Unintentional `include_subtypes=False` | ALWAYS default to `include_subtypes=True` |
