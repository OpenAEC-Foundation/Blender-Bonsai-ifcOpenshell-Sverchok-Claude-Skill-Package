---
name: blender-errors-context
description: >
  Use when debugging Blender RuntimeError from restricted context, operator poll() failures,
  or wrong context for bpy.ops calls. Prevents the #1 Blender Python error: calling operators
  or accessing context attributes from wrong areas (e.g., timer callbacks, draw handlers).
  Covers context override removal in 4.0+, temp_override migration, and context-dependent
  attribute access patterns.
  Keywords: RuntimeError, restricted context, poll failure, context override, temp_override,
  bpy.ops error, wrong context, modal context, Blender context error.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
dependencies:
  - blender-syntax-operators
  - blender-core-api
---

# Blender Context Errors — Diagnosis and Resolution

## Purpose

This skill enables diagnosis and resolution of Blender Python context errors. Context errors are the most common category of Blender scripting failures. They occur when code accesses `bpy.context` members or calls `bpy.ops` operators in situations where the required context is unavailable, restricted, or incorrectly configured.

## Scope

- RuntimeError from restricted context access (handlers, timers, draw callbacks)
- Operator `poll()` failures and incorrect context for `bpy.ops` calls
- Context override removal in Blender 4.0+ and `temp_override()` migration
- Modal operator context issues
- Context-dependent attribute access (`bpy.context.active_object`, `bpy.context.edit_object`)
- Stale references after undo, data modification, or mode changes

**Version coverage:** Blender 3.x, 4.x, 5.x. Version-breaking changes are marked explicitly.

**Dependencies:** This skill builds on `blender-core-api` (context system basics) and `blender-syntax-operators` (operator patterns).

---

## 1. Context Error Categories

### 1.1 RuntimeError: Restricted Context

**When:** Code runs in a context where `bpy.context` is partially or fully unavailable.

**Restricted contexts:**
- `depsgraph_update_post` / `depsgraph_update_pre` handlers
- `render_pre` / `render_post` / `render_write` handlers
- `draw()` methods of `bpy.types.Panel`, `bpy.types.Header`, `bpy.types.Menu`
- `bpy.app.timers` callbacks
- Background threads (NEVER access `bpy.context` from threads)

**Error messages:**
```
RuntimeError: restricted context element 'active_object'
RuntimeError: Calling operator "bpy.ops.object.select_all" error, can't modify blend data in this state
```

**Rule:** In restricted contexts, use `bpy.data` for read-only access. NEVER call `bpy.ops` or modify blend data. To defer work to a safe context, use `bpy.app.timers.register()`.

```python
# Blender 3.2+
import bpy

def my_handler(scene, depsgraph):
    # WRONG: bpy.context.active_object raises RuntimeError
    # WRONG: bpy.ops.object.delete() raises RuntimeError

    # CORRECT: read-only access via bpy.data
    obj = bpy.data.objects.get("Cube")
    if obj is not None:
        print(obj.location)

    # CORRECT: defer modifications to a timer
    def deferred():
        bpy.data.objects["Cube"].location.x += 1.0
        return None  # do not repeat
    bpy.app.timers.register(deferred)

bpy.app.handlers.depsgraph_update_post.append(my_handler)
```

### 1.2 Operator poll() Failures

**When:** An operator's `poll()` classmethod returns `False` because required context conditions are not met.

**Error message:**
```
RuntimeError: Operator bpy.ops.object.mode_set.poll() failed, context is incorrect
```

**Common causes:**
| Operator | Required context |
|----------|-----------------|
| `bpy.ops.object.mode_set()` | Active object exists, is visible, is selectable |
| `bpy.ops.mesh.subdivide()` | Active object is mesh, in Edit Mode |
| `bpy.ops.object.parent_set()` | 2+ objects selected, one active |
| `bpy.ops.uv.unwrap()` | Active mesh in Edit Mode, UV map exists |
| `bpy.ops.object.convert()` | Active object exists, correct type |

**Rule:** ALWAYS check `poll()` before calling an operator, OR set up the required context explicitly.

```python
# Blender 3.2+
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Ensure context: select and activate
bpy.context.view_layer.objects.active = obj
obj.select_set(True)

# Check poll before calling
if bpy.ops.object.mode_set.poll():
    bpy.ops.object.mode_set(mode='EDIT')
```

### 1.3 Context Override Errors (Version-Critical)

**BREAKING CHANGE in Blender 4.0:** Dictionary-based context overrides are REMOVED.

```python
# Blender 3.x ONLY — REMOVED in 4.0
override = bpy.context.copy()
override['area'] = area
override['region'] = region
bpy.ops.view3d.camera_to_view(override)  # TypeError in 4.0+
```

**Blender 3.2+ (REQUIRED in 4.0+):** Use `bpy.context.temp_override()`.

```python
# Blender 3.2+ (works in 3.2, 3.3, ..., 4.0, 4.1, ..., 5.x)
import bpy

area = next(a for a in bpy.context.screen.areas if a.type == 'VIEW_3D')
region = next(r for r in area.regions if r.type == 'WINDOW')

with bpy.context.temp_override(area=area, region=region):
    bpy.ops.view3d.camera_to_view()
```

**Rule:** ALWAYS use `bpy.context.temp_override()` for context overrides. NEVER use dictionary-based overrides — they fail silently in 3.6 and raise TypeError in 4.0+.

### 1.4 Modal Operator Context

**When:** Modal operators lose context between `invoke()` and `modal()` calls, or access area/region data that changes between frames.

**Rule:** Store required context references during `invoke()`. Do NOT rely on `bpy.context.area` or `bpy.context.region` persisting in `modal()`.

```python
# Blender 3.2+
import bpy

class EXAMPLE_OT_modal(bpy.types.Operator):
    bl_idname = "example.modal_op"
    bl_label = "Modal Example"

    def invoke(self, context, event):
        # Store references during invoke
        self._area = context.area
        self._region = context.region
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'ESC':
            return {'CANCELLED'}
        # Use stored references, not context.area
        if self._area.type != 'VIEW_3D':
            return {'CANCELLED'}
        return {'PASS_THROUGH'}
```

### 1.5 Context-Dependent Attribute Access

**When:** Code accesses `bpy.context` attributes that are only available in specific editor types or modes.

**Attributes with context requirements:**
| Attribute | Requires |
|-----------|----------|
| `context.edit_object` | Object in Edit Mode |
| `context.active_bone` | Armature in Edit Mode |
| `context.active_pose_bone` | Armature in Pose Mode |
| `context.sculpt_object` | Object in Sculpt Mode |
| `context.image_paint_object` | Object in Texture Paint Mode |
| `context.selected_editable_bones` | Armature in Edit Mode |
| `context.visible_pose_bones` | Armature in Pose Mode |

**Rule:** ALWAYS check mode before accessing mode-specific context attributes. These attributes return `None` or raise `AttributeError` when the mode does not match.

```python
# Blender 3.2+
import bpy

# WRONG: may be None if not in Edit Mode
# verts = bpy.context.edit_object.data.vertices

# CORRECT: check mode first
obj = bpy.context.active_object
if obj is not None and obj.mode == 'EDIT':
    edit_obj = bpy.context.edit_object
    # work with edit_obj
```

---

## 2. Stale Reference Errors

### 2.1 Undo Invalidates All References

**When:** After `bpy.ops.ed.undo()` or user Ctrl+Z, ALL previously stored references to Blender data become invalid.

**Error message:**
```
ReferenceError: StructRNA of type Object has been removed
```

**Rule:** NEVER store persistent references to `bpy.types` instances across operations that may trigger undo. Re-fetch by name after undo.

```python
# Blender 3.2+
import bpy

# WRONG: reference becomes stale after undo
obj = bpy.data.objects["Cube"]
bpy.ops.ed.undo()
print(obj.location)  # ReferenceError

# CORRECT: re-fetch after undo
obj_name = "Cube"
bpy.ops.ed.undo()
obj = bpy.data.objects.get(obj_name)
if obj is not None:
    print(obj.location)
```

### 2.2 Data Modification Invalidates References

**When:** Operations that add or remove data blocks (objects, meshes, materials) may invalidate existing Python references to the same collection.

**Rule:** After any operation that modifies `bpy.data` collections, re-fetch references by name. NEVER iterate a collection while modifying it.

```python
# Blender 3.2+
import bpy

# WRONG: modifying collection during iteration
for obj in bpy.data.objects:
    bpy.data.objects.remove(obj)  # invalidates iterator

# CORRECT: collect names first, then remove
names = [obj.name for obj in bpy.data.objects if obj.type == 'MESH']
for name in names:
    obj = bpy.data.objects.get(name)
    if obj is not None:
        bpy.data.objects.remove(obj)
```

### 2.3 Mode Change Invalidates Mesh References

**When:** Switching between Object Mode and Edit Mode invalidates `bmesh.from_edit_mesh()` references and mesh data access patterns.

**Rule:** ALWAYS obtain fresh mesh/bmesh references after mode changes. Free BMesh instances when done.

```python
# Blender 3.2+
import bpy
import bmesh

obj = bpy.context.active_object
bpy.ops.object.mode_set(mode='EDIT')

bm = bmesh.from_edit_mesh(obj.data)
bm.verts.ensure_lookup_table()

# ... perform edits ...

bmesh.update_edit_mesh(obj.data)
# Do NOT call bm.free() for from_edit_mesh — Blender manages it

bpy.ops.object.mode_set(mode='OBJECT')
# bm is now INVALID — do not use
```

---

## 3. Resolution Decision Tree

When encountering a context error, follow this sequence:

```
1. Is the error "restricted context element"?
   YES → Code runs in a handler, timer, or draw callback.
         → Use bpy.data for read-only access.
         → Defer modifications with bpy.app.timers.register().

2. Is the error "poll() failed"?
   YES → Check operator requirements (mode, selection, active object).
         → Set up context: activate object, set mode, ensure selection.
         → Use temp_override() if area/region context is needed.

3. Is the error about context overrides (TypeError)?
   YES → Blender 4.0+ removed dict-based overrides.
         → Migrate to bpy.context.temp_override().
         → See §1.3 for migration pattern.

4. Is the error "StructRNA has been removed"?
   YES → Reference invalidated by undo or data modification.
         → Re-fetch by name from bpy.data.
         → See §2 for patterns.

5. Is the attribute None unexpectedly?
   YES → Context attribute requires specific mode or editor type.
         → Check mode before access.
         → See §1.5 for attribute requirements.
```

---

## 4. Version Migration Reference

### 4.0 Context Override Removal

| Aspect | Blender 3.x | Blender 4.0+ |
|--------|-------------|--------------|
| Context override syntax | `op(override_dict, ...)` | REMOVED — TypeError |
| Recommended syntax | `temp_override()` (3.2+) | `temp_override()` (required) |
| `bpy.context.copy()` for overrides | Works but deprecated | REMOVED |

### temp_override() Availability

| Blender Version | `temp_override()` | Dict overrides |
|-----------------|-------------------|----------------|
| 3.0–3.1 | NOT available | Works |
| 3.2–3.6 | Available | Works (deprecated) |
| 4.0+ | Required | REMOVED |

### Version Detection Pattern

```python
# Blender 3.2+
import bpy

def call_with_override(area, region, op_func, **kwargs):
    """Call operator with context override, version-aware."""
    if bpy.app.version >= (3, 2, 0):
        with bpy.context.temp_override(area=area, region=region):
            return op_func(**kwargs)
    else:
        override = bpy.context.copy()
        override['area'] = area
        override['region'] = region
        return op_func(override, **kwargs)
```

**Note:** The version-detection pattern above is provided for migration of legacy code. For new code, ALWAYS target Blender 3.2+ and use `temp_override()` exclusively.

---

## 5. Threading Rules

**Rule:** NEVER access `bpy.context`, `bpy.ops`, or modify `bpy.data` from background threads.

**Error:**
```
RuntimeError: can't modify blend data outside main thread
```

**Resolution:** Use `bpy.app.timers.register()` to schedule work on the main thread.

```python
# Blender 3.2+
import bpy
import threading

def background_work():
    result = expensive_computation()

    def apply_result():
        bpy.data.objects["Cube"].location.x = result
        return None  # do not repeat
    bpy.app.timers.register(apply_result)

thread = threading.Thread(target=background_work)
thread.start()
```

---

## 6. Quick Reference — Error Message Index

| Error Message | Section | Resolution |
|---------------|---------|------------|
| `RuntimeError: restricted context element '...'` | §1.1 | Use `bpy.data`, defer with timers |
| `RuntimeError: Operator ... poll() failed` | §1.2 | Set up context, check poll() |
| `TypeError: ... takes N positional arguments` (4.0+) | §1.3 | Migrate to `temp_override()` |
| `ReferenceError: StructRNA of type ... has been removed` | §2.1 | Re-fetch by name |
| `RuntimeError: can't modify blend data outside main thread` | §5 | Use `bpy.app.timers.register()` |
| `AttributeError: 'NoneType' has no attribute ...` on context | §1.5 | Check mode before access |

---

## References

- `references/methods.md` — Context-related API signatures
- `references/examples.md` — Working error resolution examples
- `references/anti-patterns.md` — Common context mistakes with corrections

## Approved Sources

- [Blender Python API Documentation](https://docs.blender.org/api/current/)
- [Blender 4.0 Release Notes — Python API](https://wiki.blender.org/wiki/Reference/Release_Notes/4.0/Python_API)
- [Blender 3.2 Release Notes — temp_override](https://wiki.blender.org/wiki/Reference/Release_Notes/3.2/Python_API)
