# Context Error Anti-Patterns

Each anti-pattern shows code that appears correct but fails at runtime due to context errors. The correction explains why the pattern fails and provides the correct approach.

---

## Anti-Pattern 1: Using Dict-Based Context Overrides (Blender 4.0+ Incompatible)

### WRONG

```python
# Blender 3.x ONLY — REMOVED in 4.0
import bpy

override = bpy.context.copy()
override['active_object'] = bpy.data.objects['Cube']
bpy.ops.object.mode_set(override, mode='EDIT')
```

### Why It Fails

Blender 4.0 completely removed dictionary-based context overrides. Passing a dictionary as the first positional argument to any `bpy.ops` call raises `TypeError` in Blender 4.0+. In Blender 3.6, this pattern works but is deprecated and produces console warnings.

### CORRECT

```python
# Blender 3.2+ (works through 5.x)
import bpy

obj = bpy.data.objects.get('Cube')
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

bpy.context.view_layer.objects.active = obj
obj.select_set(True)

with bpy.context.temp_override(active_object=obj):
    bpy.ops.object.mode_set(mode='EDIT')
```

---

## Anti-Pattern 2: Calling bpy.ops in depsgraph Handlers

### WRONG

```python
# Blender 3.2+
import bpy

def auto_smooth(scene, depsgraph):
    for obj in bpy.data.objects:
        if obj.type == 'MESH':
            bpy.context.view_layer.objects.active = obj  # RuntimeError
            bpy.ops.object.shade_smooth()  # RuntimeError

bpy.app.handlers.depsgraph_update_post.append(auto_smooth)
```

### Why It Fails

`depsgraph_update_post` runs in a restricted context. It is NOT permitted to:
- Set `bpy.context.view_layer.objects.active`
- Call any `bpy.ops` function
- Modify blend data directly

Additionally, modifying data from within a depsgraph handler triggers a new depsgraph update, creating an infinite loop.

### CORRECT

```python
# Blender 3.2+
import bpy
from bpy.app.handlers import persistent

_pending_smooth = set()

@persistent
def auto_smooth(scene, depsgraph):
    # Only read data — collect names for deferred processing
    for update in depsgraph.updates:
        if isinstance(update.id, bpy.types.Object) and update.id.type == 'MESH':
            _pending_smooth.add(update.id.name)

    if _pending_smooth:
        bpy.app.timers.register(_apply_smooth)

def _apply_smooth():
    global _pending_smooth
    names = _pending_smooth.copy()
    _pending_smooth.clear()

    for name in names:
        obj = bpy.data.objects.get(name)
        if obj is not None and obj.type == 'MESH':
            for polygon in obj.data.polygons:
                polygon.use_smooth = True
    return None  # run once

bpy.app.handlers.depsgraph_update_post.append(auto_smooth)
```

---

## Anti-Pattern 3: Storing Persistent References to bpy.types Instances

### WRONG

```python
# Blender 3.2+
import bpy

class SceneManager:
    def __init__(self):
        self.cube = bpy.data.objects["Cube"]
        self.material = bpy.data.materials["Material"]

    def update(self):
        # These references may be stale
        self.cube.location.x += 1.0  # potential ReferenceError
        self.material.diffuse_color = (1, 0, 0, 1)  # potential ReferenceError

manager = SceneManager()
```

### Why It Fails

Python references to Blender data blocks (`bpy.types.Object`, `bpy.types.Mesh`, `bpy.types.Material`, etc.) are invalidated by:
- **Undo/redo** — reconstructs all data
- **File load** — replaces all data
- **Data removal** — `bpy.data.objects.remove()`
- **Some operators** — operators that rebuild data internally

After invalidation, accessing any attribute raises `ReferenceError: StructRNA of type Object has been removed`.

### CORRECT

```python
# Blender 3.2+
import bpy

class SceneManager:
    def __init__(self):
        self.cube_name = "Cube"
        self.material_name = "Material"

    def _get_cube(self):
        obj = bpy.data.objects.get(self.cube_name)
        if obj is None:
            raise RuntimeError(f"Object '{self.cube_name}' not found")
        return obj

    def _get_material(self):
        mat = bpy.data.materials.get(self.material_name)
        if mat is None:
            raise RuntimeError(f"Material '{self.material_name}' not found")
        return mat

    def update(self):
        self._get_cube().location.x += 1.0
        self._get_material().diffuse_color = (1, 0, 0, 1)

manager = SceneManager()
```

---

## Anti-Pattern 4: Modifying Data Inside Panel.draw()

### WRONG

```python
# Blender 3.2+
import bpy

class BAD_PT_panel(bpy.types.Panel):
    bl_label = "Bad Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Bad"

    def draw(self, context):
        # Attempting to "auto-fix" values during draw
        obj = context.active_object
        if obj and obj.scale.x < 0:
            obj.scale.x = abs(obj.scale.x)  # RuntimeError in draw()

        self.layout.label(text=f"Scale: {obj.scale.x if obj else 'N/A'}")
```

### Why It Fails

`Panel.draw()` is called during screen redraw, which runs in a restricted context. Any attempt to modify blend data — setting properties, calling operators, writing to data blocks — raises `RuntimeError`. The draw method must be purely read-only.

### CORRECT

```python
# Blender 3.2+
import bpy

class GOOD_PT_panel(bpy.types.Panel):
    bl_label = "Good Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Good"

    def draw(self, context):
        layout = self.layout
        obj = context.active_object
        if obj:
            row = layout.row()
            row.label(text=f"Scale X: {obj.scale.x:.3f}")
            if obj.scale.x < 0:
                row.alert = True
                layout.label(text="Warning: negative scale", icon='ERROR')
            # Use operator button for modifications
            layout.operator("good.fix_scale")

class GOOD_OT_fix_scale(bpy.types.Operator):
    bl_idname = "good.fix_scale"
    bl_label = "Fix Negative Scale"

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        return obj is not None and obj.scale.x < 0

    def execute(self, context):
        context.active_object.scale.x = abs(context.active_object.scale.x)
        return {'FINISHED'}
```

---

## Anti-Pattern 5: Assuming bpy.context.active_object is Always Available

### WRONG

```python
# Blender 3.2+
import bpy

def process_active():
    obj = bpy.context.active_object
    mesh = obj.data  # AttributeError if obj is None
    for vert in mesh.vertices:
        vert.co.z += 1.0
```

### Why It Fails

`bpy.context.active_object` is `None` when:
- No object exists in the scene
- No object is selected
- The active object was deleted
- Running in a context where active_object is not available (handlers, background mode)

Additionally, `obj.data` may not have `.vertices` if the object is not a Mesh (could be Camera, Light, Empty, etc.).

### CORRECT

```python
# Blender 3.2+
import bpy

def process_active():
    obj = bpy.context.active_object
    if obj is None:
        raise RuntimeError("No active object")
    if obj.type != 'MESH':
        raise RuntimeError(f"Active object is {obj.type}, expected MESH")

    mesh = obj.data
    for vert in mesh.vertices:
        vert.co.z += 1.0
    mesh.update()
```

---

## Anti-Pattern 6: Iterating and Modifying bpy.data Collections Simultaneously

### WRONG

```python
# Blender 3.2+
import bpy

# Delete all mesh objects
for obj in bpy.data.objects:
    if obj.type == 'MESH':
        bpy.data.objects.remove(obj)  # Invalidates iterator
```

### Why It Fails

Removing items from a `bpy.data` collection while iterating over it invalidates the iterator. This causes skipped items, crashes, or `ReferenceError`. The same applies to any operation that adds or removes data blocks during iteration.

### CORRECT

```python
# Blender 3.2+
import bpy

# Collect names first, then remove
mesh_names = [obj.name for obj in bpy.data.objects if obj.type == 'MESH']
for name in mesh_names:
    obj = bpy.data.objects.get(name)
    if obj is not None:
        bpy.data.objects.remove(obj)
```

---

## Anti-Pattern 7: Using bpy.context in Background Threads

### WRONG

```python
# Blender 3.2+
import bpy
import threading

def background_export():
    scene = bpy.context.scene  # May crash or return wrong data
    for obj in bpy.context.selected_objects:  # Not thread-safe
        export_object(obj)

thread = threading.Thread(target=background_export)
thread.start()
```

### Why It Fails

`bpy.context` is NOT thread-safe. Accessing it from a background thread can:
- Return incorrect/stale data
- Crash Blender (segfault)
- Raise `RuntimeError: can't modify blend data outside main thread`

Blender's Python API is designed for single-threaded access only. The context is bound to the main thread.

### CORRECT

```python
# Blender 3.2+
import bpy
import threading

def background_export():
    # Collect all needed data on the main thread BEFORE starting the thread
    pass  # This function only does non-Blender computation

# Gather data on main thread
export_data = []
for obj in bpy.context.selected_objects:
    export_data.append({
        'name': obj.name,
        'location': obj.location.copy(),
        'vertices': [v.co.copy() for v in obj.data.vertices] if obj.type == 'MESH' else []
    })

def process_exports():
    # Process only the pre-collected plain Python data
    for data in export_data:
        export_object_data(data)

    def notify_done():
        print("Export complete")
        return None
    bpy.app.timers.register(notify_done)

thread = threading.Thread(target=process_exports)
thread.start()
```

---

## Summary Table

| # | Anti-Pattern | Error | Key Rule |
|---|-------------|-------|----------|
| 1 | Dict-based context overrides | TypeError (4.0+) | ALWAYS use `temp_override()` |
| 2 | `bpy.ops` in depsgraph handlers | RuntimeError: restricted | NEVER call operators in handlers |
| 3 | Persistent bpy.types references | ReferenceError | ALWAYS re-fetch by name |
| 4 | Data modification in `draw()` | RuntimeError: restricted | `draw()` is read-only |
| 5 | Unchecked `active_object` access | AttributeError | ALWAYS check for None and type |
| 6 | Modify collection during iteration | Iterator invalidation | Collect names first, then modify |
| 7 | `bpy.context` in threads | Crash / RuntimeError | NEVER access bpy from threads |
