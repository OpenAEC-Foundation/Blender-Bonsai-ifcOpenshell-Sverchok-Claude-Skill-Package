# blender-core-api: Anti-Patterns

These are confirmed error patterns from the Blender Python API. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/info_overview.html
- vooronderzoek-blender.md §11 (Common Error Patterns) and §12 (AI Common Mistakes)

---

## AP-001: Dict Context Override (Removed in Blender 4.0)

**WHY this is wrong**: The dict-based context override API was removed in Blender 4.0. Scripts using it raise `TypeError` in 4.0+ and cannot run.

```python
# WRONG — Blender 3.x only, BROKEN in 4.0+
override = bpy.context.copy()
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="Subsurf")

# Also WRONG — Blender 3.x only
override = {"object": obj, "active_object": obj}
bpy.ops.object.modifier_apply(override, modifier="Subsurf")
```

```python
# CORRECT — Blender 3.2+/4.x/5.x
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Subsurf")
```

ALWAYS use `context.temp_override()` for Blender 3.2+. NEVER use dict overrides in any new code.

---

## AP-002: Caching bpy.data References Across Undo

**WHY this is wrong**: Blender's undo system rebuilds the entire data model. Any pointer stored before an undo operation becomes a dangling pointer. Accessing it causes crashes or silent memory corruption.

```python
# WRONG — modal operator stores object reference
class MY_OT_bad_modal(bpy.types.Operator):
    _stored_obj = None

    def invoke(self, context, event):
        self._stored_obj = context.active_object  # STORED REFERENCE
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        # After user presses Ctrl+Z, _stored_obj is invalid
        self._stored_obj.location.x += 0.01  # CRASH or CORRUPTION
        return {'RUNNING_MODAL'}
```

```python
# CORRECT — store name, re-fetch each iteration
class MY_OT_safe_modal(bpy.types.Operator):
    _target_name: str = ""

    def invoke(self, context, event):
        self._target_name = context.active_object.name  # Store name only
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        obj = bpy.data.objects.get(self._target_name)
        if obj is None:
            return {'CANCELLED'}  # Object was deleted
        obj.location.x += 0.01   # Fresh reference — safe
        return {'RUNNING_MODAL'}
```

ALWAYS store names or indices, NEVER store direct object references across modal iterations or undo boundaries.

---

## AP-003: Calling bpy.ops from a Background Thread

**WHY this is wrong**: Blender's Python API is not thread-safe. All `bpy.*` calls MUST execute on Blender's main thread. Calling from a background thread causes race conditions, data corruption, or immediate crashes. This is documented as an explicit limitation in the Blender API.

```python
# WRONG — modifying Blender data from a thread
import threading
import bpy

def background_task():
    bpy.data.objects["Cube"].location.x = 5.0  # UNDEFINED BEHAVIOR / CRASH

thread = threading.Thread(target=background_task)
thread.start()
```

```python
# CORRECT — compute in thread, apply on main thread via timer
import threading
import queue
import bpy

result_queue = queue.Queue()

def background_compute():
    """Heavy computation runs here — NO bpy calls."""
    result = sum(range(10_000_000))  # CPU work only
    result_queue.put(result)

def apply_on_main_thread():
    """Called by timer on Blender's main thread."""
    if not result_queue.empty():
        result = result_queue.get()
        bpy.data.objects["Cube"].location.x = result
        return None  # Stop timer
    return 0.05  # Check again in 50ms

thread = threading.Thread(target=background_compute, daemon=True)
thread.start()
bpy.app.timers.register(apply_on_main_thread, first_interval=0.05)
```

ALWAYS use `bpy.app.timers.register()` to apply thread results on the main thread. NEVER call `bpy.*` from a non-main thread.

---

## AP-004: Modifying Data Inside Panel.draw()

**WHY this is wrong**: `draw()` is called every time Blender redraws the UI (frequently). Modifying properties inside `draw()` triggers another redraw, creating an infinite loop. It also violates Blender's draw-state contract, which forbids side effects during rendering.

```python
# WRONG — modifying data in draw callback
class MY_PT_bad(bpy.types.Panel):
    bl_label = "Bad Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'

    def draw(self, context):
        # FORBIDDEN: triggers infinite redraw loop
        context.scene.frame_current += 1
        # FORBIDDEN: operator call in draw
        bpy.ops.object.select_all(action='SELECT')
        self.layout.label(text="This panel is broken")
```

```python
# CORRECT — draw() only reads data and builds UI elements
class MY_PT_good(bpy.types.Panel):
    bl_label = "Good Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'

    def draw(self, context):
        layout = self.layout
        # Read-only access — safe
        layout.label(text=f"Frame: {context.scene.frame_current}")
        # Operators in UI — user triggers them, not draw()
        layout.operator("my.advance_frame", text="Next Frame")
```

ALWAYS keep `draw()` free of data modifications. ALWAYS use operators for user-initiated actions.

---

## AP-005: Accessing Mesh Data in Edit Mode

**WHY this is wrong**: When an object is in Edit Mode, the mesh data in `obj.data.vertices` is NOT synchronized with the edit mesh. The data is stale or incomplete. Blender does not automatically synchronize until you exit Edit Mode.

```python
# WRONG — reading mesh data while in Edit Mode
bpy.ops.object.mode_set(mode='EDIT')
obj = bpy.context.active_object
for v in obj.data.vertices:
    print(v.co)  # Returns pre-edit-mode data, not current edit state
```

```python
# CORRECT option 1 — exit Edit Mode first
bpy.ops.object.mode_set(mode='OBJECT')
obj = bpy.context.active_object
for v in obj.data.vertices:
    print(v.co)  # Current data

# CORRECT option 2 — use bmesh in Edit Mode
import bmesh
obj = bpy.context.edit_object
bm = bmesh.from_edit_mesh(obj.data)
for v in bm.verts:
    print(v.co)  # Live edit data
bmesh.update_edit_mesh(obj.data)  # After modifications
```

ALWAYS use `bmesh.from_edit_mesh()` when reading or modifying mesh data in Edit Mode. NEVER access `obj.data.vertices` while the object is in Edit Mode.

---

## AP-006: Not Calling to_mesh_clear() After to_mesh()

**WHY this is wrong**: `obj_eval.to_mesh()` allocates a temporary `Mesh` object. Without calling `to_mesh_clear()`, this memory is never freed, causing a memory leak. For complex meshes or repeated calls in handlers, this accumulates significant memory.

```python
# WRONG — memory leak
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
for v in mesh_eval.vertices:
    print(v.co)
# Missing: obj_eval.to_mesh_clear()
```

```python
# CORRECT — always paired call
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
try:
    for v in mesh_eval.vertices:
        print(v.co)
finally:
    obj_eval.to_mesh_clear()  # ALWAYS execute, even if exception raised
```

ALWAYS call `to_mesh_clear()` in a `finally` block or immediately after use. NEVER leave a temporary mesh unreleased.

---

## AP-007: Assuming Data Block Name Is Exact

**WHY this is wrong**: When creating a new data block with `bpy.data.*.new(name=...)`, Blender appends `.001`, `.002`, etc. if the name already exists. The returned name may differ from the requested name.

```python
# WRONG — assuming name is exact
bpy.data.meshes.new(name="MyMesh")
mesh = bpy.data.meshes["MyMesh"]  # KeyError if "MyMesh.001" was created instead
```

```python
# CORRECT — use the returned reference directly
mesh = bpy.data.meshes.new(name="MyMesh")
# mesh.name may be "MyMesh" or "MyMesh.001" depending on existing data
# ALWAYS use the variable, NEVER look up by the requested name
obj = bpy.data.objects.new("MyObject", mesh)
bpy.context.collection.objects.link(obj)
```

ALWAYS store and use the reference returned by `.new()`. NEVER look up data by the name you requested to create.

---

## AP-008: Creating Objects Without Linking to a Collection

**WHY this is wrong**: `bpy.data.objects.new()` creates an object that exists in `bpy.data` but is not part of any scene collection. Such objects are invisible in the viewport and are purged on save/reload as orphaned data.

```python
# WRONG — object created but not visible, will be purged
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata([(0,0,0),(1,0,0),(0,1,0)], [], [(0,1,2)])
mesh.update()
obj = bpy.data.objects.new("Object", mesh)
# Missing: link to collection
```

```python
# CORRECT — link to active scene collection
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata([(0,0,0),(1,0,0),(0,1,0)], [], [(0,1,2)])
mesh.update()
obj = bpy.data.objects.new("Object", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED
```

ALWAYS call `collection.objects.link(obj)` after creating an object. NEVER assume an object is visible after creation without explicit linking.

---

## AP-009: Not Calling mesh.update() After from_pydata()

**WHY this is wrong**: `mesh.from_pydata()` populates raw vertex/face data but does not compute normals, edge data, or other derived data. Without `mesh.update()`, the mesh displays incorrectly (wrong normals, missing edges) or causes errors in downstream operations.

```python
# WRONG — missing update causes display errors and wrong normals
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata(verts, [], faces)
obj = bpy.data.objects.new("Object", mesh)
bpy.context.collection.objects.link(obj)
```

```python
# CORRECT — always call update immediately after from_pydata
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata(verts, [], faces)
mesh.update()  # Computes normals, validates, builds edge data
obj = bpy.data.objects.new("Object", mesh)
bpy.context.collection.objects.link(obj)
```

ALWAYS call `mesh.update()` immediately after `mesh.from_pydata()`. NEVER create objects with unupdated meshes.

---

## AP-010: Using Removed 2.7x / Pre-2.80 API

**WHY this is wrong**: Blender 2.80 introduced a complete Python API redesign. Many patterns from 2.7x still appear in online examples and Claude training data. They cause `AttributeError` in any current Blender version.

```python
# WRONG — 2.7x patterns (ALL broken in 2.80+)
bpy.context.scene.objects.link(obj)       # AttributeError
bpy.context.scene.objects.active = obj    # AttributeError
obj.select = True                          # AttributeError
bpy.context.scene.render.layers           # AttributeError (now view_layers)
mesh.uv_textures                          # AttributeError (now uv_layers)
bpy.data.lamps                            # AttributeError (now lights)
bpy.utils.register_module(__name__)       # AttributeError (removed)
```

```python
# CORRECT — 2.80+/3.x/4.x/5.x API
bpy.context.collection.objects.link(obj)  # Link to active collection
bpy.context.view_layer.objects.active = obj
obj.select_set(True)
bpy.context.scene.view_layers             # Correct attribute
mesh.uv_layers                            # Correct attribute
bpy.data.lights                           # Correct attribute
# Register classes individually in register()
bpy.utils.register_class(MyClass)
```

ALWAYS use 2.80+ API patterns. NEVER use 2.7x patterns (`scene.objects.link`, `obj.select`, `bpy.data.lamps`, `render.layers`).

---

## AP-011: Using Dynamic EnumProperty Without Storing Items Reference

**WHY this is wrong**: When `EnumProperty.items` is a callback function, Blender holds a reference to the returned list at C level. Python's garbage collector can free the list if no Python reference exists, causing the dropdown to crash or show garbage data.

```python
# WRONG — returned list is immediately garbage-collected
def get_items(self, context):
    return [(obj.name, obj.name, "") for obj in bpy.data.objects]
    # The list is created and returned, but no Python variable holds it

my_enum: bpy.props.EnumProperty(items=get_items)
# Accessing this property can crash Blender
```

```python
# CORRECT — store reference to prevent garbage collection
_enum_items_cache = []

def get_items(self, context):
    global _enum_items_cache
    _enum_items_cache = [
        (obj.name, obj.name, f"Select {obj.name}", 'OBJECT_DATA', i)
        for i, obj in enumerate(bpy.data.objects)
    ]
    return _enum_items_cache

my_enum: bpy.props.EnumProperty(items=get_items)
```

ALWAYS store dynamic EnumProperty item lists in a module-level variable. NEVER return a locally-created list from an EnumProperty items callback without storing a reference.

---

## AP-012: Using bgl Module in Blender 5.0+

**WHY this is wrong**: The `bgl` module (OpenGL wrapper) was deprecated in Blender 3.5 and completely removed in Blender 5.0. It also does not work on Apple Silicon (Metal backend) in any version. All drawing code must use the `gpu` module.

```python
# WRONG — BROKEN in Blender 5.0, non-functional on Apple M1/M2
import bgl
import gpu

def draw():
    bgl.glEnable(bgl.GL_BLEND)        # AttributeError or ImportError in 5.0
    bgl.glLineWidth(2)                 # BROKEN
    shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')  # ALSO REMOVED in 4.0
    bgl.glDisable(bgl.GL_BLEND)
```

```python
# CORRECT — Blender 4.0+/5.x
import gpu
from gpu_extras.batch import batch_for_shader

def draw():
    gpu.state.blend_set('ALPHA')       # Replaces bgl.glEnable(GL_BLEND)
    gpu.state.line_width_set(2.0)      # Replaces bgl.glLineWidth(2)

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')  # 4.0+
    batch = batch_for_shader(shader, 'LINES', {"pos": [(0,0,0),(1,0,0)]})
    shader.bind()
    shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (1.0, 0.5, 0.0, 1.0))
    batch.draw(shader)

    gpu.state.blend_set('NONE')        # ALWAYS restore state
    gpu.state.line_width_set(1.0)
```

ALWAYS use the `gpu` module for all drawing code. NEVER import `bgl` in new code targeting Blender 4.0+. NEVER use `bgl` if the code must run on Apple Silicon.

---

## AP-013: Calling bpy.ops When Direct Data Access Suffices

**WHY this is wrong**: Operators require correct context (area, mode, selection), trigger redraws, create undo steps, and are significantly slower than direct data manipulation. Using operators for simple data changes adds context dependency and brittleness to scripts.

```python
# WRONG — using operators for simple data operations
bpy.ops.object.select_all(action='DESELECT')
for obj_name in names:
    bpy.data.objects[obj_name].select_set(True)
    bpy.context.view_layer.objects.active = bpy.data.objects[obj_name]
    bpy.ops.object.location_clear()  # Operator just to zero a location

# WRONG — using operator to rename
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.make_single_user(type='ALL', object=True)
```

```python
# CORRECT — direct data manipulation
for obj_name in names:
    obj = bpy.data.objects.get(obj_name)
    if obj:
        obj.location = (0.0, 0.0, 0.0)  # Direct assignment — no context needed

# CORRECT — direct rename
obj = bpy.data.objects.get("OldName")
if obj:
    obj.name = "NewName"
```

ALWAYS prefer direct `bpy.data` manipulation for simple property changes. ONLY use `bpy.ops` when undo support, user feedback, or functionality unavailable through data access is required.
