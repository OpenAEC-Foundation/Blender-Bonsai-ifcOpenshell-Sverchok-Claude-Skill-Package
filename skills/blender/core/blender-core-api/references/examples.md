# blender-core-api: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/info_quickstart.html
- https://docs.blender.org/api/current/info_overview.html
- https://docs.blender.org/api/current/bpy.types.Depsgraph.html

---

## Example 1: Basic Data Access

```python
# Blender 3.x/4.x/5.x — access objects, meshes, materials
import bpy

# Access by name (safe pattern — never assume exact name)
obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Access nested data
mesh = obj.data  # bpy.types.Mesh
print(f"Object: {obj.name}, type: {obj.type}")
print(f"Mesh vertex count: {len(mesh.vertices)}")
print(f"Location: {obj.location[:]}")

# Iterate all objects of a specific type
for obj in bpy.data.objects:
    if obj.type == 'MESH':
        print(f"Mesh object: {obj.name}")
```

---

## Example 2: Creating a Mesh Object

```python
# Blender 3.x/4.x/5.x — create mesh from vertex/face data
import bpy

verts = [
    (0.0, 0.0, 0.0),
    (1.0, 0.0, 0.0),
    (1.0, 1.0, 0.0),
    (0.0, 1.0, 0.0),
]
edges = []   # Empty — auto-computed from faces
faces = [(0, 1, 2, 3)]

mesh = bpy.data.meshes.new("QuadMesh")
mesh.from_pydata(verts, edges, faces)
mesh.update()  # REQUIRED after from_pydata

obj = bpy.data.objects.new("QuadObject", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED to appear in viewport

print(f"Created: {obj.name} with {len(mesh.vertices)} vertices")
```

---

## Example 3: Context Override — Blender 4.0+ (temp_override)

```python
# Blender 3.2+/4.x/5.x — override context for operators
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Apply all modifiers on a specific object
with bpy.context.temp_override(object=obj, active_object=obj):
    for mod in obj.modifiers[:]:  # Copy list — modifiers change during apply
        bpy.ops.object.modifier_apply(modifier=mod.name)

# Override area type for viewport-specific operators
def run_in_viewport(callback):
    """Find first VIEW_3D area and run callback with it as context."""
    for window in bpy.context.window_manager.windows:
        for area in window.screen.areas:
            if area.type == 'VIEW_3D':
                with bpy.context.temp_override(window=window, area=area):
                    callback()
                return
    raise RuntimeError("No 3D Viewport found")

run_in_viewport(lambda: bpy.ops.view3d.snap_cursor_to_center())
```

---

## Example 4: Dependency Graph — Evaluated Mesh

```python
# Blender 3.x/4.x/5.x — read post-modifier mesh data
import bpy

obj = bpy.context.active_object
if obj is None or obj.type != 'MESH':
    raise RuntimeError("Select a mesh object")

# Get the depsgraph
depsgraph = bpy.context.evaluated_depsgraph_get()

# Get evaluated object (with all modifiers applied)
obj_eval = obj.evaluated_get(depsgraph)

# Get evaluated mesh (temporary — MUST be cleared)
mesh_eval = obj_eval.to_mesh()

print(f"Original vertex count: {len(obj.data.vertices)}")
print(f"Evaluated vertex count: {len(mesh_eval.vertices)}")

# Read data
for v in mesh_eval.vertices:
    world_co = obj_eval.matrix_world @ v.co  # World-space position
    _ = world_co  # use it

# ALWAYS clear the temporary mesh
obj_eval.to_mesh_clear()
```

---

## Example 5: Iterating All Object Instances

```python
# Blender 3.x/4.x/5.x — iterate ALL instances including collection/particle instances
import bpy

depsgraph = bpy.context.evaluated_depsgraph_get()

for instance in depsgraph.object_instances:
    obj = instance.object          # Evaluated object
    matrix = instance.matrix_world # World-space transform

    if instance.is_instance:
        # This is a collection instance or particle instance
        parent = instance.parent   # Parent that spawned this instance
        print(f"Instance of '{obj.name}' from '{parent.name}' at {matrix.translation[:]}")
    else:
        # This is a base object (not instanced)
        print(f"Object '{obj.name}' at {matrix.translation[:]}")
```

---

## Example 6: Depsgraph Update Handler

```python
# Blender 3.x/4.x/5.x — respond to scene changes
import bpy
from bpy.app.handlers import persistent

@persistent  # REQUIRED: survives file load/save
def on_scene_updated(scene, depsgraph):
    """Called after every dependency graph evaluation."""
    for update in depsgraph.updates:
        id_block = update.id
        if update.is_updated_geometry:
            print(f"Geometry changed: {id_block.name}")
        if update.is_updated_transform:
            print(f"Transform changed: {id_block.name}")
        if update.is_updated_shading:
            print(f"Shading changed: {id_block.name}")

# Register
bpy.app.handlers.depsgraph_update_post.append(on_scene_updated)

# Unregister (call in addon unregister())
def cleanup():
    if on_scene_updated in bpy.app.handlers.depsgraph_update_post:
        bpy.app.handlers.depsgraph_update_post.remove(on_scene_updated)
```

---

## Example 7: Minimal Operator (Addon Pattern)

```python
# Blender 3.x/4.x/5.x — complete minimal addon with operator
import bpy

class MY_OT_move_origin(bpy.types.Operator):
    """Move active object to world origin"""  # Docstring = tooltip
    bl_idname = "my.move_to_origin"           # Lowercase: "module.name"
    bl_label = "Move to Origin"
    bl_options = {'REGISTER', 'UNDO'}         # UNDO creates undo step on FINISHED

    @classmethod
    def poll(cls, context):
        """Operator is available only when an active object exists."""
        return context.active_object is not None

    def execute(self, context):
        context.active_object.location = (0.0, 0.0, 0.0)
        return {'FINISHED'}  # Creates undo step


def register():
    bpy.utils.register_class(MY_OT_move_origin)


def unregister():
    bpy.utils.unregister_class(MY_OT_move_origin)


if __name__ == "__main__":
    register()
    # Test:
    bpy.ops.my.move_to_origin()
```

---

## Example 8: RNA Introspection

```python
# Blender 3.x/4.x/5.x — inspect RNA properties of a type
import bpy

obj = bpy.context.active_object

# Access the RNA type descriptor
rna_type = obj.bl_rna

print(f"Type name: {rna_type.name}")
print(f"Description: {rna_type.description}")

# List all properties
for prop in rna_type.properties:
    if prop.identifier.startswith('_'):
        continue  # skip internal
    print(f"  {prop.identifier}: type={prop.type}, description={prop.description}")

# Inspect a specific property
loc_prop = rna_type.properties['location']
print(f"location: array_length={loc_prop.array_length}, subtype={loc_prop.subtype}")

# Inspect type of a specific object's data
mesh_rna = bpy.types.Mesh.bl_rna
for func in mesh_rna.functions:
    print(f"Method: {func.identifier}")
```

---

## Example 9: ID Data Block Reference Counting

```python
# Blender 3.x/4.x/5.x — manage data block lifetime
import bpy

# Create a material
mat = bpy.data.materials.new("MyMaterial")
print(f"Users: {mat.users}")       # 0 — no one uses it yet

# Assign to object
obj = bpy.context.active_object
if obj.data.materials:
    obj.data.materials[0] = mat
else:
    obj.data.materials.append(mat)

print(f"Users: {mat.users}")       # 1 — assigned to one mesh

# Prevent purge on save when users drop to 0
mat.use_fake_user = True           # Survives save even if unused

# Check if data would be purged without fake user
print(f"Would be orphaned: {mat.users == 0 and not mat.use_fake_user}")
```

---

## Example 10: Deferred Execution with Timer

```python
# Blender 3.x/4.x/5.x — run code on main thread after delay
import bpy
import queue

_work_queue = queue.Queue()


def do_background_computation(data):
    """Run in a thread — NO bpy calls allowed here."""
    import time
    time.sleep(1)  # Simulate expensive work
    return data * 2


def apply_results():
    """Called on main thread by timer — bpy calls safe here."""
    while not _work_queue.empty():
        result = _work_queue.get_nowait()
        bpy.context.scene.my_result = result
        print(f"Applied result: {result}")
    return None  # Return None to stop the timer


def start_background_work(data):
    import threading

    def worker():
        result = do_background_computation(data)
        _work_queue.put(result)

    thread = threading.Thread(target=worker, daemon=True)
    thread.start()

    # Register a timer to apply results on main thread
    bpy.app.timers.register(apply_results, first_interval=0.1)


# Usage: start_background_work(42)
```

---

## Example 11: Version-Aware Code Pattern

```python
# Blender 3.x/4.x/5.x — handle API differences by version
import bpy

BLENDER_VERSION = bpy.app.version  # e.g., (4, 2, 0)


def apply_modifier(obj, modifier_name):
    """Apply a modifier, using correct API for the current Blender version."""
    if BLENDER_VERSION >= (3, 2, 0):
        # Blender 3.2+/4.x/5.x: use temp_override
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        # Blender < 3.2: use dict override (deprecated but functional)
        override = bpy.context.copy()
        override['object'] = obj
        override['active_object'] = obj
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)


def get_node_group_socket_api():
    """Return correct API for node group socket creation."""
    if BLENDER_VERSION >= (4, 0, 0):
        return "interface"  # node_group.interface.new_socket(...)
    else:
        return "legacy"     # node_group.inputs.new(...)
```
