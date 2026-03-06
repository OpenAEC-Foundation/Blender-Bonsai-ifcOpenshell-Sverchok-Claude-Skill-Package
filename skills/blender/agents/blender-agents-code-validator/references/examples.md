# Before/After Validation Examples

Each example shows code that fails validation, the check that catches it, and the corrected version.

---

## Example 1: Dict Context Override → temp_override

**Check**: V-005 (CHECK 2.3)
**Severity**: BLOCKER on Blender 4.0+
**Target**: Blender 4.0+

### Before (FAILS validation)
```python
import bpy

def apply_modifier(obj, modifier_name):
    override = bpy.context.copy()
    override['active_object'] = obj
    override['object'] = obj
    bpy.ops.object.modifier_apply(override, modifier=modifier_name)
```

### Validator Output
```
BLOCKER [CHECK 2.3] Dict-style context override used — removed in Blender 4.0.
  File: modifier_utils.py:4
  Pattern: bpy.context.copy() + operator call with dict as first argument
  Fix: Replace with context.temp_override()
```

### After (PASSES validation)
```python
import bpy

def apply_modifier(obj, modifier_name):
    with bpy.context.temp_override(active_object=obj, object=obj):
        bpy.ops.object.modifier_apply(modifier=modifier_name)
```

---

## Example 2: Stale Reference After Collection Mutation

**Check**: V-009 (CHECK 4.1)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy

def populate_collection(scene):
    items = scene.my_custom_collection
    first = items.add()
    first.name = "Initial"

    for i in range(50):
        items.add()

    # first pointer is now invalid — C array was re-allocated
    first.value = 42  # CRASH
```

### Validator Output
```
BLOCKER [CHECK 4.1] Stale CollectionProperty reference: 'first' assigned at line 5,
  used at line 11 after 50 .add() calls that re-allocate the C array.
  Fix: Re-fetch reference by index after mutations.
```

### After (PASSES validation)
```python
import bpy

def populate_collection(scene):
    items = scene.my_custom_collection
    first = items.add()
    first.name = "Initial"

    for i in range(50):
        items.add()

    # Re-fetch after mutations
    first = items[0]
    first.value = 42  # Safe
```

---

## Example 3: bpy.ops in Panel.draw()

**Check**: V-002 (CHECK 2.1)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy

class MY_PT_panel(bpy.types.Panel):
    bl_label = "My Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        layout = self.layout
        if context.active_object:
            # FORBIDDEN: operator call inside draw()
            bpy.ops.object.select_all(action='DESELECT')
            layout.label(text=f"Active: {context.active_object.name}")
```

### Validator Output
```
BLOCKER [CHECK 2.1] bpy.ops call inside Panel.draw() method.
  File: my_panel.py:12
  Pattern: bpy.ops.object.select_all() in draw() of class inheriting bpy.types.Panel
  Fix: Remove operator call from draw(). Use an operator button instead.
```

### After (PASSES validation)
```python
import bpy

class MY_PT_panel(bpy.types.Panel):
    bl_label = "My Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        layout = self.layout
        if context.active_object:
            layout.label(text=f"Active: {context.active_object.name}")
            layout.operator("object.select_all", text="Deselect All").action = 'DESELECT'
```

---

## Example 4: BMesh Without ensure_lookup_table() and free()

**Check**: V-012, V-013 (CHECK 4.4)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy
import bmesh

def get_vertex_positions(obj):
    mesh = obj.data
    bm = bmesh.new()
    bm.from_mesh(mesh)

    positions = []
    for i in range(len(bm.verts)):
        v = bm.verts[i]  # IndexError — no ensure_lookup_table()
        positions.append(v.co.copy())

    return positions  # Memory leak — no bm.free()
```

### Validator Output
```
BLOCKER [CHECK 4.4] BMesh index access without ensure_lookup_table().
  File: mesh_utils.py:10
  Pattern: bm.verts[i] without prior bm.verts.ensure_lookup_table()

BLOCKER [CHECK 4.4] BMesh created with bmesh.new() but never freed.
  File: mesh_utils.py:6
  Pattern: bmesh.new() without bm.free() before function exit
```

### After (PASSES validation)
```python
import bpy
import bmesh

def get_vertex_positions(obj):
    mesh = obj.data
    bm = bmesh.new()
    bm.from_mesh(mesh)
    bm.verts.ensure_lookup_table()

    positions = []
    for i in range(len(bm.verts)):
        v = bm.verts[i]  # Safe — lookup table built
        positions.append(v.co.copy())

    bm.free()  # Properly freed
    return positions
```

---

## Example 5: Threading Violation

**Check**: V-015 (CHECK 5.1)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy
import threading

def process_objects():
    for obj in bpy.data.objects:  # bpy access in thread — UNDEFINED BEHAVIOR
        obj.location.z += 1.0

thread = threading.Thread(target=process_objects)
thread.start()
```

### Validator Output
```
BLOCKER [CHECK 5.1] bpy.data access inside threading.Thread target function.
  File: threaded_ops.py:5
  Pattern: bpy.data.objects accessed in function passed to threading.Thread()
  Fix: Move bpy access to main thread via queue + bpy.app.timers
```

### After (PASSES validation)
```python
import bpy
import threading
import queue

result_queue = queue.Queue()

def compute_offsets():
    # Pure computation — no bpy access
    offsets = {name: 1.0 for name in ["Cube", "Sphere", "Cone"]}
    result_queue.put(offsets)

def apply_results():
    if result_queue.empty():
        return 0.1  # Check again in 0.1s
    offsets = result_queue.get()
    for name, offset in offsets.items():
        obj = bpy.data.objects.get(name)
        if obj is not None:
            obj.location.z += offset
    return None  # Stop timer

thread = threading.Thread(target=compute_offsets)
thread.start()
bpy.app.timers.register(apply_results)
```

---

## Example 6: Missing @persistent and Handler Cleanup

**Check**: V-017, V-018 (CHECK 6.2, 6.3)
**Severity**: BLOCKER (missing @persistent), WARNING (missing cleanup)

### Before (FAILS validation)
```python
import bpy

def on_frame_change(scene):
    print(f"Frame: {scene.frame_current}")

def register():
    bpy.app.handlers.frame_change_post.append(on_frame_change)

def unregister():
    pass  # Handler not removed
```

### Validator Output
```
BLOCKER [CHECK 6.3] Handler 'on_frame_change' appended to frame_change_post
  without @bpy.app.handlers.persistent decorator. Handler will be removed on file load.
  File: my_addon.py:3

WARNING [CHECK 6.2] Handler 'on_frame_change' registered in register() but not
  removed in unregister().
  File: my_addon.py:10
```

### After (PASSES validation)
```python
import bpy
from bpy.app.handlers import persistent

@persistent
def on_frame_change(scene):
    print(f"Frame: {scene.frame_current}")

def register():
    bpy.app.handlers.frame_change_post.append(on_frame_change)

def unregister():
    bpy.app.handlers.frame_change_post.remove(on_frame_change)
```

---

## Example 7: Wrong Registration Order

**Check**: V-016 (CHECK 6.1)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy

class MySettings(bpy.types.PropertyGroup):
    scale: bpy.props.FloatProperty(default=1.0)

class MY_OT_apply(bpy.types.Operator):
    bl_idname = "my.apply"
    bl_label = "Apply"

    def execute(self, context):
        s = context.scene.my_settings.scale
        context.active_object.scale = (s, s, s)
        return {'FINISHED'}

def register():
    bpy.utils.register_class(MY_OT_apply)
    bpy.utils.register_class(MySettings)  # TOO LATE
    bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MySettings)

def unregister():
    bpy.utils.unregister_class(MySettings)  # WRONG ORDER
    bpy.utils.unregister_class(MY_OT_apply)
    del bpy.types.Scene.my_settings  # TOO LATE — type already unregistered
```

### Validator Output
```
BLOCKER [CHECK 6.1] PointerProperty(type=MySettings) at line 19 references MySettings,
  but MySettings is registered at line 18 (after MY_OT_apply at line 17).
  PropertyGroup types MUST be registered before any PointerProperty referencing them.

BLOCKER [CHECK 6.1] unregister() deletes Scene.my_settings at line 23,
  after MySettings is unregistered at line 21. Delete PointerProperty FIRST.
```

### After (PASSES validation)
```python
import bpy

class MySettings(bpy.types.PropertyGroup):
    scale: bpy.props.FloatProperty(default=1.0)

class MY_OT_apply(bpy.types.Operator):
    bl_idname = "my.apply"
    bl_label = "Apply"

    def execute(self, context):
        s = context.scene.my_settings.scale
        context.active_object.scale = (s, s, s)
        return {'FINISHED'}

def register():
    bpy.utils.register_class(MySettings)  # FIRST: dependency
    bpy.utils.register_class(MY_OT_apply)
    bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MySettings)

def unregister():
    del bpy.types.Scene.my_settings  # FIRST: remove pointer
    bpy.utils.unregister_class(MY_OT_apply)
    bpy.utils.unregister_class(MySettings)  # LAST: dependency
```

---

## Example 8: Name Collision and Missing Collection Link

**Check**: V-010, V-011 (CHECK 4.2, 4.3)
**Severity**: WARNING (name collision), BLOCKER (missing link)

### Before (FAILS validation)
```python
import bpy

def create_cube():
    mesh = bpy.data.meshes.new("Cube")
    verts = [(-1,-1,-1), (1,-1,-1), (1,1,-1), (-1,1,-1),
             (-1,-1,1), (1,-1,1), (1,1,1), (-1,1,1)]
    faces = [(0,1,2,3), (4,5,6,7), (0,1,5,4),
             (2,3,7,6), (0,3,7,4), (1,2,6,5)]
    mesh.from_pydata(verts, [], faces)

    obj = bpy.data.objects.new("Cube", mesh)
    # Object is invisible — not linked to any collection

    # Later, dangerous lookup:
    my_mesh = bpy.data.meshes["Cube"]  # May be "Cube.001"
```

### Validator Output
```
BLOCKER [CHECK 4.5] mesh.from_pydata() at line 9 not followed by mesh.update().

BLOCKER [CHECK 4.3] Object created at line 11 but never linked to a collection.
  Pattern: bpy.data.objects.new() without subsequent collection.objects.link()

WARNING [CHECK 4.2] Name collision risk at line 14.
  Pattern: bpy.data.meshes["Cube"] after .new("Cube") — name may have suffix.
  Fix: Use the return value from .new() instead of dict lookup.
```

### After (PASSES validation)
```python
import bpy

def create_cube():
    mesh = bpy.data.meshes.new("Cube")
    verts = [(-1,-1,-1), (1,-1,-1), (1,1,-1), (-1,1,-1),
             (-1,-1,1), (1,-1,1), (1,1,1), (-1,1,1)]
    faces = [(0,1,2,3), (4,5,6,7), (0,1,5,4),
             (2,3,7,6), (0,3,7,4), (1,2,6,5)]
    mesh.from_pydata(verts, [], faces)
    mesh.update()  # Required after from_pydata

    obj = bpy.data.objects.new("Cube", mesh)
    bpy.context.collection.objects.link(obj)  # Link to active collection

    # Use return value — no dict lookup needed
```

---

## Example 9: bgl Usage on Blender 5.0+

**Check**: V-021 (CHECK 8)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy
import bgl

def draw_callback():
    bgl.glEnable(bgl.GL_BLEND)
    bgl.glLineWidth(2.0)
    # ... draw code ...
    bgl.glDisable(bgl.GL_BLEND)
    bgl.glLineWidth(1.0)
```

### Validator Output
```
BLOCKER [CHECK 8] 'import bgl' — bgl module removed in Blender 5.0.
  File: draw_overlay.py:2
  Fix: Replace with gpu module equivalents.
```

### After (PASSES validation)
```python
import bpy
import gpu
from gpu_extras.batch import batch_for_shader

def draw_callback():
    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(2.0)
    # ... draw code using gpu.shader and batch_for_shader ...
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)
```

---

## Example 10: EnumProperty Items Garbage Collection

**Check**: V-019 (CHECK 6.4)
**Severity**: BLOCKER

### Before (FAILS validation)
```python
import bpy

class MY_OT_select(bpy.types.Operator):
    bl_idname = "my.select"
    bl_label = "Select Object"

    def get_objects(self, context):
        return [(o.name, o.name, "") for o in bpy.data.objects]

    target: bpy.props.EnumProperty(items=get_objects)

    def execute(self, context):
        obj = bpy.data.objects.get(self.target)
        return {'FINISHED'}
```

### Validator Output
```
BLOCKER [CHECK 6.4] EnumProperty callback 'get_objects' returns a freshly
  created list without caching. The list is garbage-collected by Python,
  causing crash or empty dropdown.
  File: select_op.py:8
  Fix: Cache the returned list in a module-level variable.
```

### After (PASSES validation)
```python
import bpy

_object_items_cache = []

class MY_OT_select(bpy.types.Operator):
    bl_idname = "my.select"
    bl_label = "Select Object"

    def get_objects(self, context):
        global _object_items_cache
        _object_items_cache = [(o.name, o.name, "") for o in bpy.data.objects]
        return _object_items_cache

    target: bpy.props.EnumProperty(items=get_objects)

    def execute(self, context):
        obj = bpy.data.objects.get(self.target)
        return {'FINISHED'}
```

---

## Example 11: Multi-Version Context Override Wrapper

**Check**: V-005 (CHECK 2.3)
**Severity**: Demonstrates version-safe pattern

### Version-Safe Implementation (PASSES all versions)
```python
import bpy
from contextlib import contextmanager

@contextmanager
def safe_context_override(**kwargs):
    """Version-safe context override for Blender 3.2+."""
    if bpy.app.version >= (4, 0, 0):
        with bpy.context.temp_override(**kwargs):
            yield
    elif bpy.app.version >= (3, 2, 0):
        with bpy.context.temp_override(**kwargs):
            yield
    else:
        override = bpy.context.copy()
        override.update(kwargs)
        yield override

# Usage for 4.0+:
with safe_context_override(active_object=obj, object=obj):
    bpy.ops.object.modifier_apply(modifier="MyMod")

# Usage for 3.x (pre-3.2) requires different call pattern:
if bpy.app.version < (3, 2, 0):
    override = bpy.context.copy()
    override['active_object'] = obj
    bpy.ops.object.modifier_apply(override, modifier="MyMod")
else:
    with safe_context_override(active_object=obj, object=obj):
        bpy.ops.object.modifier_apply(modifier="MyMod")
```

---

## Example 12: Undo-Safe Modal Operator

**Check**: V-009 (CHECK 4.1)
**Severity**: Demonstrates safe reference handling

### Before (FAILS validation)
```python
import bpy

class MY_OT_mover(bpy.types.Operator):
    bl_idname = "my.mover"
    bl_label = "Move Object"
    bl_options = {'REGISTER', 'UNDO'}

    _target = None  # Stored bpy reference — invalidated on undo

    def invoke(self, context, event):
        self._target = context.active_object
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'MOUSEMOVE':
            self._target.location.x = event.mouse_region_x * 0.01  # CRASH after undo
        elif event.type == 'ESC':
            return {'CANCELLED'}
        return {'RUNNING_MODAL'}
```

### After (PASSES validation)
```python
import bpy

class MY_OT_mover(bpy.types.Operator):
    bl_idname = "my.mover"
    bl_label = "Move Object"
    bl_options = {'REGISTER', 'UNDO'}

    _target_name: str = ""  # Store name, not reference

    def invoke(self, context, event):
        self._target_name = context.active_object.name
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        obj = bpy.data.objects.get(self._target_name)  # Re-fetch every frame
        if obj is None:
            return {'CANCELLED'}
        if event.type == 'MOUSEMOVE':
            obj.location.x = event.mouse_region_x * 0.01  # Safe
        elif event.type == 'ESC':
            return {'CANCELLED'}
        return {'RUNNING_MODAL'}
```
