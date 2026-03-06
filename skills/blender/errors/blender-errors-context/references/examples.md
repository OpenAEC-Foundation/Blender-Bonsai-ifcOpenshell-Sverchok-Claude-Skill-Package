# Working Error Resolution Examples

Each example shows a real error scenario, the root cause, and the correct resolution.

---

## Example 1: RuntimeError in depsgraph Handler

### Error

```
RuntimeError: restricted context element 'active_object'
```

### Failing Code

```python
# Blender 3.2+
import bpy

def on_depsgraph_update(scene, depsgraph):
    obj = bpy.context.active_object  # RuntimeError: restricted context
    if obj and obj.location.z < 0:
        obj.location.z = 0

bpy.app.handlers.depsgraph_update_post.append(on_depsgraph_update)
```

### Root Cause

`depsgraph_update_post` runs in a restricted context. `bpy.context.active_object` is not available. Direct modification of blend data from this handler is not permitted.

### Resolution

```python
# Blender 3.2+
import bpy
from bpy.app.handlers import persistent

@persistent
def on_depsgraph_update(scene, depsgraph):
    # Use bpy.data instead of bpy.context for read access
    obj = bpy.data.objects.get("Cube")
    if obj is None:
        return

    if obj.location.z < 0:
        # Defer modification to a timer (runs on main thread, unrestricted)
        def fix_position():
            target = bpy.data.objects.get("Cube")
            if target is not None and target.location.z < 0:
                target.location.z = 0
            return None  # run once
        bpy.app.timers.register(fix_position)

bpy.app.handlers.depsgraph_update_post.append(on_depsgraph_update)
```

---

## Example 2: Operator poll() Failure — No Active Object

### Error

```
RuntimeError: Operator bpy.ops.object.mode_set.poll() failed, context is incorrect
```

### Failing Code

```python
# Blender 3.2+
import bpy

# Script assumes there is always an active object
bpy.ops.object.mode_set(mode='EDIT')
```

### Root Cause

No object is active in the view layer. `mode_set` requires an active, visible, selectable object.

### Resolution

```python
# Blender 3.2+
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found in bpy.data.objects")

# Ensure object is in the current view layer
if obj.name not in bpy.context.view_layer.objects:
    raise RuntimeError(f"Object '{obj.name}' not in active view layer")

# Activate and select
bpy.context.view_layer.objects.active = obj
obj.select_set(True)

# Now poll() will succeed
bpy.ops.object.mode_set(mode='EDIT')
```

---

## Example 3: Dict-Based Context Override Fails in Blender 4.0

### Error

```
TypeError: Converting py args to operator properties: : keyword "area" unrecognized
```

### Failing Code

```python
# Blender 3.x — BREAKS in 4.0+
import bpy

override = bpy.context.copy()
for area in bpy.context.screen.areas:
    if area.type == 'VIEW_3D':
        override['area'] = area
        for region in area.regions:
            if region.type == 'WINDOW':
                override['region'] = region
                bpy.ops.view3d.snap_cursor_to_center(override)
                break
        break
```

### Root Cause

Blender 4.0 removed dictionary-based context overrides. Passing a dict as the first positional argument to an operator is no longer supported.

### Resolution

```python
# Blender 3.2+ (works in 3.2 through 5.x)
import bpy

area = next((a for a in bpy.context.screen.areas if a.type == 'VIEW_3D'), None)
if area is None:
    raise RuntimeError("No 3D Viewport found")

region = next((r for r in area.regions if r.type == 'WINDOW'), None)
if region is None:
    raise RuntimeError("No WINDOW region in 3D Viewport")

with bpy.context.temp_override(area=area, region=region):
    bpy.ops.view3d.snap_cursor_to_center()
```

---

## Example 4: Stale Reference After Undo

### Error

```
ReferenceError: StructRNA of type Object has been removed
```

### Failing Code

```python
# Blender 3.2+
import bpy

cube = bpy.data.objects["Cube"]
bpy.ops.mesh.primitive_uv_sphere_add()

# User presses Ctrl+Z (undo)
bpy.ops.ed.undo()

# cube reference is now stale
print(cube.location)  # ReferenceError
```

### Root Cause

Undo reconstructs the entire data state. All Python references to `bpy.types` instances (objects, meshes, materials, etc.) become invalid.

### Resolution

```python
# Blender 3.2+
import bpy

# Store the name, not the reference
cube_name = "Cube"

bpy.ops.mesh.primitive_uv_sphere_add()
bpy.ops.ed.undo()

# Re-fetch after any undo
cube = bpy.data.objects.get(cube_name)
if cube is not None:
    print(cube.location)
else:
    print(f"Object '{cube_name}' no longer exists after undo")
```

---

## Example 5: Accessing edit_object Outside Edit Mode

### Error

```
AttributeError: 'NoneType' object has no attribute 'data'
```

### Failing Code

```python
# Blender 3.2+
import bpy

# Assumes we are in Edit Mode
mesh = bpy.context.edit_object.data  # AttributeError if not in Edit Mode
```

### Root Cause

`bpy.context.edit_object` is `None` when no object is in Edit Mode. The code does not verify the current mode.

### Resolution

```python
# Blender 3.2+
import bpy

obj = bpy.context.active_object
if obj is None:
    raise RuntimeError("No active object")

if obj.mode != 'EDIT':
    raise RuntimeError(f"Expected Edit Mode, got {obj.mode}")

edit_obj = bpy.context.edit_object
# edit_obj is guaranteed non-None here
mesh = edit_obj.data
```

---

## Example 6: Threading Violation

### Error

```
RuntimeError: can't modify blend data outside main thread
```

### Failing Code

```python
# Blender 3.2+
import bpy
import threading

def compute_and_apply():
    result = heavy_computation()
    bpy.data.objects["Cube"].location.x = result  # RuntimeError

thread = threading.Thread(target=compute_and_apply)
thread.start()
```

### Root Cause

Blender's data API is not thread-safe. All `bpy.data` modifications must occur on the main thread.

### Resolution

```python
# Blender 3.2+
import bpy
import threading

def compute_and_apply():
    result = heavy_computation()

    def apply_on_main_thread():
        obj = bpy.data.objects.get("Cube")
        if obj is not None:
            obj.location.x = result
        return None  # run once

    bpy.app.timers.register(apply_on_main_thread)

thread = threading.Thread(target=compute_and_apply)
thread.start()
```

---

## Example 7: Operator Requires Specific Area Type

### Error

```
RuntimeError: Operator bpy.ops.node.add_node.poll() failed, context is incorrect
```

### Failing Code

```python
# Blender 3.2+
import bpy

# Called from a script or panel — no Node Editor area is active
bpy.ops.node.add_node(type='ShaderNodeBsdfPrincipled')
```

### Root Cause

`bpy.ops.node.*` operators require the active area to be a `NODE_EDITOR`. When called from a script or from a different area type, the poll check fails.

### Resolution

```python
# Blender 3.2+
import bpy

# Find Node Editor area
area = next((a for a in bpy.context.screen.areas if a.type == 'NODE_EDITOR'), None)
if area is None:
    raise RuntimeError("No Node Editor area found — open one first")

region = next(r for r in area.regions if r.type == 'WINDOW')

with bpy.context.temp_override(area=area, region=region):
    bpy.ops.node.add_node(type='ShaderNodeBsdfPrincipled')
```

---

## Example 8: Modifying Data in Panel draw()

### Error

```
RuntimeError: Calling operator "bpy.ops.object.select_all" error, can't modify blend data in this state
```

### Failing Code

```python
# Blender 3.2+
import bpy

class EXAMPLE_PT_panel(bpy.types.Panel):
    bl_label = "Example"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Example"

    def draw(self, context):
        layout = self.layout
        # WRONG: modifying data in draw()
        if context.active_object:
            context.active_object.location.x = 0  # RuntimeError
```

### Root Cause

`Panel.draw()` runs in a restricted context. It is called during screen redraw and must NOT modify blend data or call operators.

### Resolution

```python
# Blender 3.2+
import bpy

class EXAMPLE_PT_panel(bpy.types.Panel):
    bl_label = "Example"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Example"

    def draw(self, context):
        layout = self.layout
        # draw() is read-only: display data, create UI elements
        obj = context.active_object
        if obj:
            layout.label(text=f"X: {obj.location.x:.2f}")
        # Use an operator button to let the user trigger modifications
        layout.operator("example.reset_position")

class EXAMPLE_OT_reset(bpy.types.Operator):
    bl_idname = "example.reset_position"
    bl_label = "Reset Position"

    def execute(self, context):
        if context.active_object:
            context.active_object.location.x = 0
        return {'FINISHED'}
```

---

## Example 9: Batch Operations With temp_override and Selection

### Scenario

Apply a modifier to multiple objects without manual selection.

### Solution

```python
# Blender 3.2+
import bpy

mesh_objects = [obj for obj in bpy.data.objects if obj.type == 'MESH']

for obj in mesh_objects:
    # Ensure the object has the modifier
    if "Subdivision" not in obj.modifiers:
        continue

    # Set as active and selected
    bpy.context.view_layer.objects.active = obj
    obj.select_set(True)

    # Apply modifier — no area override needed for this operator
    bpy.ops.object.modifier_apply(modifier="Subdivision")

    obj.select_set(False)
```

### Alternative: Direct Data Access (Preferred When Possible)

```python
# Blender 3.2+
import bpy

# For evaluation without applying, use depsgraph
depsgraph = bpy.context.evaluated_depsgraph_get()

for obj in bpy.data.objects:
    if obj.type != 'MESH':
        continue
    eval_obj = obj.evaluated_get(depsgraph)
    eval_mesh = eval_obj.to_mesh()
    # Use eval_mesh.vertices, eval_mesh.polygons, etc.
    eval_obj.to_mesh_clear()
```
