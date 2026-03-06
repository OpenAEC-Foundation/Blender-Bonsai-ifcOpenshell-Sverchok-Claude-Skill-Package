---
name: blender-errors-data
description: "Diagnoses and resolves Blender Python data access errors including ReferenceError from removed objects, undo-invalidated bpy.data references, stale ID pointers after file operations, and data lifecycle management. Covers safe data access patterns, ID reference caching pitfalls, and proper use of bpy.data collections across all Blender versions."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-errors-data

## Quick Reference

### Critical Rules

**NEVER** store direct `bpy.data` references (objects, meshes, materials) across undo/redo, file load, or file revert operations — ALL Python pointers to Blender data are invalidated.

**NEVER** access a data block after calling `bpy.data.<collection>.remove()` on it — the StructRNA is freed and access raises `ReferenceError`.

**NEVER** assume `bpy.data.<collection>.new(name)` returns the exact name you passed — Blender appends `.001`, `.002`, etc. when a name already exists.

**NEVER** keep references to `CollectionProperty` items across `.add()` or `.remove()` calls — the underlying C array is re-allocated, invalidating all prior element pointers.

**NEVER** use `del obj["prop"]` to reset RNA-defined properties in Blender 5.0+ — use `obj.property_unset("prop")` instead.

**NEVER** use `obj["cycles"]` dict-like access for system properties in Blender 5.0+ — system properties are separated from custom properties.

**ALWAYS** store object/data block names (strings) or stable identifiers instead of direct `bpy` references when the reference must survive across operator execution, undo, or file operations.

**ALWAYS** re-fetch references from `bpy.data.<collection>` using `.get(name)` before accessing data that may have been invalidated.

**ALWAYS** store the return value of `bpy.data.<collection>.new()` directly in a variable — do not look up by name afterward.

**ALWAYS** use `bpy.data.objects.get(name)` (returns `None`) instead of `bpy.data.objects[name]` (raises `KeyError`) when the data block may not exist.

### Error Identification Table

| Error Message | Root Cause | Section |
|---------------|------------|---------|
| `ReferenceError: StructRNA of type X has been removed` | Accessing data block after `remove()` call | [Removed Data Access](#error-1-referenceerror-from-removed-data) |
| `ReferenceError` after Ctrl+Z | Undo invalidated all bpy references | [Undo Invalidation](#error-2-undo-invalidates-all-references) |
| `KeyError: 'bpy_prop_collection[key] not found'` | Name collision — Blender renamed the data block | [Name Collision](#error-3-name-collision-on-data-creation) |
| Crash / segfault after `CollectionProperty.add()` | Stale pointer to re-allocated array element | [Stale Collection References](#error-4-stale-collectionproperty-references) |
| `ReferenceError` after file load/revert | File operation rebuilt entire data model | [File Operation Invalidation](#error-5-file-operations-invalidate-references) |
| `KeyError` with `del obj["prop"]` in 5.0 | IDProperty separation — dict access removed for system props | [5.0 IDProperty Changes](#error-6-blender-50-idproperty-separation) |

### Decision Tree: Safe Data Access

```
Need to reference a Blender data block?
├── Will you use it immediately (same function, no operators called)?
│   └── YES → Direct reference is safe: obj = bpy.data.objects["Cube"]
├── Will an operator with undo run between store and access?
│   └── YES → Store NAME, re-fetch later: name = obj.name
├── Will undo/redo possibly occur (modal operator, interactive use)?
│   └── YES → Store NAME, re-fetch on every access
├── Will file load/revert occur?
│   └── YES → Store NAME, re-fetch after load_post handler fires
└── Will CollectionProperty.add()/remove() be called?
    └── YES → Re-fetch ALL element references after modification
```

### Decision Tree: Diagnosing ReferenceError

```
Got ReferenceError: StructRNA of type X has been removed?
├── Did you call bpy.data.<collection>.remove() on this data?
│   └── YES → Do not access the variable after remove()
├── Did undo/redo happen since you obtained the reference?
│   └── YES → Re-fetch by name from bpy.data
├── Did a file load/revert happen?
│   └── YES → Re-fetch by name from bpy.data
├── Is this inside a modal operator?
│   └── YES → Store name in operator property, re-fetch each modal() call
└── Did you switch modes (Edit → Object)?
    └── YES → Re-fetch mesh data references after mode switch
```

---

## Error 1: ReferenceError from Removed Data

When a data block is removed via `bpy.data.<collection>.remove()`, ANY Python variable still pointing to that data raises `ReferenceError` on access.

```python
# Blender 3.x/4.x/5.x — BROKEN
mesh = bpy.data.meshes.new("TempMesh")
bpy.data.meshes.remove(mesh)
print(mesh.name)  # ReferenceError: StructRNA of type Mesh has been removed

# Blender 3.x/4.x/5.x — CORRECT
mesh = bpy.data.meshes.new("TempMesh")
mesh_name = mesh.name  # Save name before removal if needed
bpy.data.meshes.remove(mesh)
mesh = None  # Explicitly clear the reference
```

**Cascading removal**: Removing an object does NOT automatically remove its data. Removing a mesh does NOT automatically remove objects using it. Track dependencies manually.

```python
# Blender 3.x/4.x/5.x — Safe removal of object AND its data
obj = bpy.data.objects.get("MyObject")
if obj:
    mesh = obj.data
    bpy.data.objects.remove(obj)
    if mesh and mesh.users == 0:
        bpy.data.meshes.remove(mesh)
```

---

## Error 2: Undo Invalidates ALL References

When undo or redo occurs, Blender rebuilds the entire data model from its undo history. ALL Python references to `bpy.types.ID` subclasses (objects, meshes, materials, scenes, etc.) become invalid. This is the single most dangerous data access pitfall.

**When invalidation occurs:**
- `bpy.ops.ed.undo()` / `bpy.ops.ed.redo()`
- Interactive Ctrl+Z / Ctrl+Shift+Z by the user
- `bpy.ops.wm.open_mainfile()` (file load)
- `bpy.ops.wm.revert_mainfile()` (file revert)
- Any operator with built-in undo support that the user subsequently undoes

```python
# Blender 3.x/4.x/5.x — BROKEN: reference held across undo
obj = bpy.data.objects["Cube"]
bpy.ops.ed.undo()
obj.location.x = 1.0  # ReferenceError or crash

# Blender 3.x/4.x/5.x — CORRECT: store name, re-fetch
obj_name = bpy.data.objects["Cube"].name
bpy.ops.ed.undo()
obj = bpy.data.objects.get(obj_name)
if obj is not None:
    obj.location.x = 1.0
```

See [references/examples.md](references/examples.md) for the `SafeObjectRef` wrapper pattern.

---

## Error 3: Name Collision on Data Creation

Blender enforces unique names within each `bpy.data` collection. When you create a data block with a name that already exists, Blender silently appends `.001`, `.002`, etc.

```python
# Blender 3.x/4.x/5.x — BROKEN: assuming exact name
bpy.data.meshes.new(name="MyMesh")
mesh = bpy.data.meshes["MyMesh"]  # KeyError if "MyMesh" already existed — it became "MyMesh.001"

# Blender 3.x/4.x/5.x — CORRECT: capture return value
mesh = bpy.data.meshes.new(name="MyMesh")  # Returns the actual data block
print(mesh.name)  # May be "MyMesh.001" — use this variable, never look up by name
```

**Renaming also triggers this**: If you set `obj.name = "Cube"` but "Cube" already exists, Blender renames the OTHER object to "Cube.001". Code storing the old name of the other object will break.

---

## Error 4: Stale CollectionProperty References

`CollectionProperty` elements are stored in a contiguous C array. Calling `.add()` or `.remove()` may re-allocate the entire array, invalidating ALL Python references to existing elements.

```python
# Blender 3.x/4.x/5.x — BROKEN: stale pointer after re-allocation
items = bpy.context.scene.my_collection
first_item = items.add()
first_item.name = "First"

for i in range(100):
    items.add()  # May re-allocate entire array

first_item.name = "Updated"  # CRASH or undefined behavior — pointer is invalid

# Blender 3.x/4.x/5.x — CORRECT: re-fetch by index after modifications
items = bpy.context.scene.my_collection
items.add()
items[len(items) - 1].name = "First"

for i in range(100):
    items.add()

items[0].name = "Updated"  # Safe — accessed by index, not stale pointer
```

---

## Error 5: File Operations Invalidate References

File load and revert rebuild the entire data model, identical to undo invalidation. References obtained before the file operation are invalid.

```python
# Blender 3.x/4.x/5.x — BROKEN
obj = bpy.data.objects["Wall"]
bpy.ops.wm.open_mainfile(filepath="/path/to/other.blend")
print(obj.name)  # ReferenceError — entire data model was replaced

# Blender 3.x/4.x/5.x — CORRECT: use load_post handler
from bpy.app.handlers import persistent

@persistent
def on_file_loaded(filepath):
    obj = bpy.data.objects.get("Wall")
    if obj:
        print(f"Found Wall in new file: {obj.name}")

bpy.app.handlers.load_post.append(on_file_loaded)
```

---

## Error 6: Blender 5.0 IDProperty Separation

Blender 5.0 separates system-defined properties (RNA properties defined via `bpy.props`) from user-defined custom properties. Dict-like access to system properties is removed.

```python
# Blender 4.x — Works (but deprecated)
value = bpy.context.scene["cycles"]  # Accesses Cycles settings via dict syntax
del obj["my_rna_prop"]               # Resets RNA property via dict deletion

# Blender 5.0+ — BROKEN
value = bpy.context.scene["cycles"]  # KeyError — system props not in dict
del obj["my_rna_prop"]               # Fails for RNA-defined properties

# Blender 5.0+ — CORRECT
value = bpy.context.scene.cycles     # Access via attribute
obj.property_unset("my_rna_prop")    # Reset RNA property to default

# Custom properties still use dict access in ALL versions
obj["my_custom_prop"] = 42           # Works in 3.x/4.x/5.x
value = obj["my_custom_prop"]        # Works in 3.x/4.x/5.x
del obj["my_custom_prop"]            # Works in 3.x/4.x/5.x
```

**Migration for legacy files**: Blender 5.0 auto-duplicates old IDProperties into system storage when opening pre-5.0 files. Clean up with:

```python
# Blender 5.0+ — Cleanup duplicated IDProperties from legacy files
for ob in bpy.data.objects:
    sys_props = ob.bl_system_properties_get()
    # Migrate values if needed, then remove old custom prop duplicates
```

---

## Mode Switching Invalidates Mesh Data

Switching between Edit Mode and Object Mode re-allocates mesh data arrays. References to `mesh.vertices`, `mesh.polygons`, `mesh.edges`, and UV layers obtained before the switch are invalid.

```python
# Blender 3.x/4.x/5.x — BROKEN
verts = bpy.context.active_object.data.vertices
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.object.mode_set(mode='OBJECT')
print(verts[0].co)  # Crash or stale data — verts pointer is invalid

# Blender 3.x/4.x/5.x — CORRECT: re-fetch after mode switch
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.object.mode_set(mode='OBJECT')
verts = bpy.context.active_object.data.vertices  # Re-fetch
print(verts[0].co)  # Safe
```

---

## Safe Data Access Patterns Summary

| Pattern | When to Use | Implementation |
|---------|-------------|----------------|
| Store name, re-fetch | Modal operators, handlers, any long-lived reference | `name = obj.name` → `bpy.data.objects.get(name)` |
| Use `.get()` not `[]` | Any lookup where data may not exist | `bpy.data.objects.get("X")` returns `None` |
| Capture return value | After `.new()` calls | `mesh = bpy.data.meshes.new("X")` |
| Re-fetch after collection mutation | After `.add()` / `.remove()` on CollectionProperty | Access by index, not stored reference |
| Null-check after re-fetch | After undo, file load, or any invalidation point | `if obj is not None:` |
| Use `@persistent` on handlers | Any handler in an addon | Without it, handler is removed on file load |

---

## Reference Links

- [references/methods.md](references/methods.md) — Data access API signatures for bpy.data collections, ID lifecycle, and property access
- [references/examples.md](references/examples.md) — Complete error resolution examples with safe patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Common data access mistakes with explanations

### Official Sources

- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/info_gotchas_internal_data_and_python_objects.html
- https://docs.blender.org/api/current/info_gotchas_crashes.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/
