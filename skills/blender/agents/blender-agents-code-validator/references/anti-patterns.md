# Anti-Patterns Catalog

Anti-patterns that the Blender Code Validator catches. Each entry documents a recurring mistake, its detection signature, impact, and fix.

---

## AP-001: Operator Abuse — Using bpy.ops for Data Manipulation

**Severity**: WARNING
**Check**: V-020 (CHECK 7.1)

**Description**: Using `bpy.ops.*` for tasks achievable through direct data access. Operators are designed for user-facing actions with undo support. Direct data access is faster, more reliable, and context-independent.

**Detection signature**:
```python
# Anti-pattern: operator for simple property changes
bpy.ops.object.location_clear()
bpy.ops.object.rotation_clear()
bpy.ops.transform.translate(value=(1, 0, 0))
bpy.ops.object.select_all(action='DESELECT')
bpy.ops.object.delete()
```

**Impact**:
- Operators require correct context (area, mode, active object) — fragile in scripts
- Operators are slower due to undo stack and poll() checks
- Operators fail silently or raise RuntimeError when context is wrong

**Fix**: Use direct data access:
```python
obj.location = (0, 0, 0)
obj.rotation_euler = (0, 0, 0)
obj.location += mathutils.Vector((1, 0, 0))
obj.select_set(False)
bpy.data.objects.remove(obj, do_unlink=True)
```

**Rule**: Use `bpy.ops` ONLY when: (1) undo support is required, (2) the operation has no direct data API, or (3) the call is triggered by a UI button.

---

## AP-002: Fire-and-Forget Data References

**Severity**: BLOCKER
**Check**: V-009 (CHECK 4.1)

**Description**: Storing `bpy.data.*` references in variables and using them across operations that invalidate all bpy references (undo, file load, data removal, mode switch, collection mutation).

**Detection signature**:
```python
# Anti-pattern: reference stored, then used across invalidating operation
obj = bpy.data.objects["Cube"]
bpy.ops.ed.undo()
obj.location.x = 5  # CRASH — ReferenceError

mesh = bpy.data.meshes["Mesh"]
bpy.data.meshes.remove(other_mesh)
mesh.vertices  # CRASH — memory may be re-allocated

items = scene.my_collection
first = items.add()
items.add()  # Re-allocates C array
first.name = "x"  # CRASH — dangling pointer
```

**Impact**: Segmentation fault, `ReferenceError: StructRNA of type Object has been removed`, or silently corrupted data.

**Fix**: Store names/identifiers, re-fetch references after every invalidating operation:
```python
obj_name = "Cube"
bpy.ops.ed.undo()
obj = bpy.data.objects.get(obj_name)
if obj is not None:
    obj.location.x = 5
```

**Rule**: NEVER hold bpy.data references across ANY operation that modifies the data block graph. ALWAYS re-fetch by name using `.get()`.

---

## AP-003: Context Mutation in Draw Callbacks

**Severity**: BLOCKER
**Check**: V-002 (CHECK 2.1)

**Description**: Calling operators, modifying scene data, or changing properties inside `Panel.draw()`, `Header.draw()`, `Menu.draw()`, `Gizmo.draw()`, or `SpaceView3D.draw_handler_add()` callbacks. These run in a restricted context where data mutation causes infinite redraw loops, RuntimeError, or crashes.

**Detection signature**:
```python
class MY_PT_panel(bpy.types.Panel):
    def draw(self, context):
        bpy.ops.object.select_all(action='SELECT')       # FORBIDDEN
        context.scene.frame_current += 1                  # FORBIDDEN
        bpy.data.objects.new("Temp", None)                # FORBIDDEN
        context.active_object.location.x = 0              # FORBIDDEN
```

**Impact**: `RuntimeError: Calling operator in wrong context`, infinite UI redraws, Blender freeze, data corruption.

**Fix**: `draw()` methods MUST be read-only. Only build UI layout and read properties. Mutations go in operators triggered by `layout.operator()`.

```python
class MY_PT_panel(bpy.types.Panel):
    def draw(self, context):
        layout = self.layout
        layout.operator("object.select_all").action = 'SELECT'  # Button, not call
        layout.label(text=f"Frame: {context.scene.frame_current}")  # Read-only
```

**Rule**: NEVER call `bpy.ops`, create/remove data, or mutate properties inside any `draw()` method. The draw callback is for UI layout construction only.

---

## AP-004: Thread-Unsafe bpy Access

**Severity**: BLOCKER
**Check**: V-015 (CHECK 5.1)

**Description**: Accessing `bpy.*` from any Python thread other than the main thread. Blender's data model is not thread-safe. Any bpy access from a background thread causes undefined behavior, race conditions, or segfaults.

**Detection signature**:
```python
import threading

def worker():
    bpy.data.objects["Cube"].location.x = 5  # From thread — UNDEFINED
    bpy.ops.mesh.primitive_cube_add()         # From thread — CRASH

thread = threading.Thread(target=worker)
thread.start()
```

**Impact**: Segmentation fault, corrupted .blend file, random crashes, race conditions in undo stack.

**Fix**: Perform pure computation in threads. Pass results to main thread via `queue.Queue` + `bpy.app.timers.register()`:

```python
import threading
import queue

result_queue = queue.Queue()

def compute():
    result = expensive_calculation()  # No bpy access
    result_queue.put(result)

def apply():
    if result_queue.empty():
        return 0.1
    result = result_queue.get()
    bpy.data.objects["Cube"].location.x = result  # Main thread — safe
    return None

threading.Thread(target=compute).start()
bpy.app.timers.register(apply)
```

**Rule**: NEVER access any `bpy.*` API from a background thread. ALL bpy calls MUST execute on Blender's main thread.

---

## AP-005: Version-Blind Code Generation

**Severity**: WARNING (existing code), BLOCKER (if targeting wrong version)
**Check**: V-006, V-007, V-008 (CHECK 3.1, 3.3)

**Description**: Writing Blender Python code without specifying or checking the target version. This leads to using APIs that have been removed, renamed, or changed behavior across versions.

**Detection signature**:
```python
# No version check anywhere — which Blender is this for?
import bpy

mesh.calc_normals()                    # Removed in 4.0
bone.layers[0] = True                 # Removed in 4.0
import bgl                            # Removed in 5.0
NodeTree.inputs.new("NodeSocketFloat", "Value")  # Removed in 4.0
scene.render.engine = 'BLENDER_EEVEE'  # Wrong for 4.2–4.x
```

**Impact**: `AttributeError`, `ImportError`, `TypeError`, or silent misbehavior. Code written for one Blender version crashes on another.

**Fix**: ALWAYS declare and check target version:
```python
import bpy

# Method 1: Version check at module level
if bpy.app.version < (4, 0, 0):
    raise RuntimeError("This addon requires Blender 4.0 or later")

# Method 2: Feature detection
if hasattr(bpy.types.NodeTree, 'interface'):
    # 4.0+ path
    tree.interface.new_socket("Value", socket_type='NodeSocketFloat')
else:
    # 3.x path
    tree.inputs.new("NodeSocketFloat", "Value")
```

**Rule**: EVERY Blender Python file MUST either declare a minimum version (via `bl_info`, `blender_manifest.toml`, or explicit check) or use feature detection for version-dependent APIs.

---

## AP-006: BMesh Lifecycle Negligence

**Severity**: BLOCKER
**Check**: V-012, V-013, V-014 (CHECK 4.4, 4.5)

**Description**: Mismanaging BMesh objects — forgetting `ensure_lookup_table()` before index access, not calling `bm.free()` after standalone BMesh use, or freeing edit-mode BMesh (which Blender owns).

**Detection signature**:
```python
# Missing ensure_lookup_table
bm = bmesh.new()
bm.from_mesh(mesh)
v = bm.verts[0]           # CRASH — no lookup table

# Missing free
bm = bmesh.new()
bm.from_mesh(mesh)
bm.to_mesh(mesh)
# Missing bm.free()         # Memory leak

# Wrong free on edit-mode BMesh
bm = bmesh.from_edit_mesh(obj.data)
bm.free()                  # WRONG — Blender owns this BMesh

# Missing mesh.update after from_pydata
mesh.from_pydata(verts, edges, faces)
# Missing mesh.update()     # Normals/display broken
```

**Impact**: `IndexError` or undefined behavior, memory leaks accumulating per operation, crash when freeing Blender-owned BMesh, broken normals and rendering.

**Fix**:
```python
# Standalone BMesh — full lifecycle
bm = bmesh.new()
bm.from_mesh(mesh)
bm.verts.ensure_lookup_table()  # Before ANY index access
bm.edges.ensure_lookup_table()
bm.faces.ensure_lookup_table()
# ... operations ...
bm.to_mesh(mesh)
bm.free()                       # ALWAYS free standalone BMesh
mesh.update()                   # Update after modification

# Edit-mode BMesh — Blender owns it
bm = bmesh.from_edit_mesh(obj.data)
bm.verts.ensure_lookup_table()
# ... operations ...
bmesh.update_edit_mesh(obj.data) # NOT bm.free()
```

**Rule**: ALWAYS call `ensure_lookup_table()` before index access. ALWAYS call `bm.free()` on standalone BMesh. NEVER call `bm.free()` on edit-mode BMesh. ALWAYS call `mesh.update()` after `from_pydata()` or `to_mesh()`.

---

## AP-007: Unprotected Handler Registration

**Severity**: BLOCKER (missing @persistent), WARNING (missing cleanup)
**Check**: V-017, V-018 (CHECK 6.2, 6.3)

**Description**: Registering handler callbacks on `bpy.app.handlers.*` without the `@persistent` decorator and/or without removing them in `unregister()`. Without `@persistent`, handlers are silently removed when the user loads a new file. Without cleanup in `unregister()`, handlers accumulate on addon reload.

**Detection signature**:
```python
# Missing @persistent
def on_load(scene):
    print("File loaded")

bpy.app.handlers.load_post.append(on_load)  # Removed on next file load

# Missing cleanup
def register():
    bpy.app.handlers.frame_change_post.append(my_handler)

def unregister():
    pass  # my_handler stays registered → duplicates on addon reload
```

**Impact**: Handler silently stops working after file load. Handler accumulates multiple copies on addon disable/enable cycle, causing duplicate execution and performance degradation.

**Fix**:
```python
from bpy.app.handlers import persistent

@persistent
def on_load(scene):
    print("File loaded")

def register():
    bpy.app.handlers.load_post.append(on_load)

def unregister():
    if on_load in bpy.app.handlers.load_post:
        bpy.app.handlers.load_post.remove(on_load)
```

**Rule**: ALWAYS use `@persistent` on handler callbacks. ALWAYS remove handlers in `unregister()`. Check for existence before removing to avoid ValueError.

---

## AP-008: EnumProperty Dynamic Items Without Cache

**Severity**: BLOCKER
**Check**: V-019 (CHECK 6.4)

**Description**: Using a callback function for `EnumProperty(items=...)` that returns a freshly created list each time without storing it in a persistent variable. Python garbage-collects the list after Blender reads it, causing crash or empty dropdown on next access.

**Detection signature**:
```python
# Anti-pattern: list created and immediately eligible for GC
def get_items(self, context):
    return [(o.name, o.name, "") for o in bpy.data.objects]

my_enum: bpy.props.EnumProperty(items=get_items)
```

**Impact**: Segfault, empty dropdown, or garbled enum entries. Intermittent — depends on when Python's garbage collector runs.

**Fix**: Cache the list in a module-level variable:
```python
_enum_items_cache = []

def get_items(self, context):
    global _enum_items_cache
    _enum_items_cache = [(o.name, o.name, "") for o in bpy.data.objects]
    return _enum_items_cache

my_enum: bpy.props.EnumProperty(items=get_items)
```

**Rule**: ALWAYS cache dynamic EnumProperty items in a module-level or class-level variable that persists beyond the callback's scope.

---

## AP-009: Wrong Registration Order in Addons

**Severity**: BLOCKER
**Check**: V-016 (CHECK 6.1)

**Description**: Registering classes in the wrong order in `register()` — specifically, registering operators and panels before the PropertyGroup classes they depend on, or creating `PointerProperty` references to types not yet registered.

**Detection signature**:
```python
def register():
    bpy.utils.register_class(MY_OT_operator)  # Uses MySettings
    bpy.utils.register_class(MySettings)       # Registered TOO LATE
    bpy.types.Scene.settings = bpy.props.PointerProperty(type=MySettings)

def unregister():
    bpy.utils.unregister_class(MySettings)     # Unregistered TOO EARLY
    del bpy.types.Scene.settings               # Pointer still exists
    bpy.utils.unregister_class(MY_OT_operator)
```

**Impact**: `RuntimeError: Error: Registering operator class` referencing unregistered PropertyGroup. Crash on addon unload when PointerProperty references already-unregistered type.

**Fix**:
```python
def register():
    bpy.utils.register_class(MySettings)       # Dependencies FIRST
    bpy.utils.register_class(MY_OT_operator)
    bpy.types.Scene.settings = bpy.props.PointerProperty(type=MySettings)

def unregister():
    del bpy.types.Scene.settings               # Remove pointers FIRST
    bpy.utils.unregister_class(MY_OT_operator)
    bpy.utils.unregister_class(MySettings)     # Dependencies LAST
```

**Rule**: Register in dependency order (PropertyGroup → Operator → Panel). Unregister in reverse order. Delete PointerProperties BEFORE unregistering their target types.

---

## AP-010: Ignoring Object-Collection Linkage

**Severity**: BLOCKER
**Check**: V-011 (CHECK 4.3)

**Description**: Creating objects via `bpy.data.objects.new()` without linking them to any collection. Objects exist in `bpy.data.objects` but are invisible in the viewport and not saved with the scene in a recoverable way.

**Detection signature**:
```python
mesh = bpy.data.meshes.new("MyMesh")
obj = bpy.data.objects.new("MyObj", mesh)
# No collection.objects.link(obj) call
# Object is orphaned — invisible, cannot be selected
```

**Impact**: Object is invisible. User sees no result. Object consumes memory but serves no purpose. On file save/load, orphaned data blocks with zero users are purged.

**Fix**: ALWAYS link to a collection immediately after creation:
```python
mesh = bpy.data.meshes.new("MyMesh")
obj = bpy.data.objects.new("MyObj", mesh)
bpy.context.collection.objects.link(obj)  # Link to active collection
# OR link to a specific collection:
target_collection = bpy.data.collections.get("MyCollection")
if target_collection is not None:
    target_collection.objects.link(obj)
```

**Rule**: EVERY `bpy.data.objects.new()` call MUST be followed by a `collection.objects.link()` call in the same scope.
