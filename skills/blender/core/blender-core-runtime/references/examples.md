# blender-core-runtime: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/mathutils.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy.msgbus.html
- https://docs.blender.org/api/current/bpy.app.handlers.html
- https://docs.blender.org/api/current/info_gotchas.html

---

## Example 1: Vector Operations

```python
# Blender 3.x/4.x/5.x — common vector operations
from mathutils import Vector
import math

# Creation and arithmetic
v1 = Vector((1.0, 2.0, 3.0))
v2 = Vector((4.0, 5.0, 6.0))
v_sum = v1 + v2                      # (5.0, 7.0, 9.0)
v_scaled = v1 * 2.0                  # (2.0, 4.0, 6.0)

# Normalization — two patterns
v_norm = v1.normalized()             # Returns NEW vector (v1 unchanged)
v1_copy = v1.copy()
v1_copy.normalize()                  # Modifies v1_copy IN PLACE

# Dot and cross products
dot = v1.dot(v2)                     # 32.0
cross = v1.cross(v2)                 # (-3.0, 6.0, -3.0)

# Angle between vectors
angle_rad = v1.angle(v2)            # Radians
angle_deg = math.degrees(angle_rad)

# Projection and interpolation
proj = v1.project(v2)               # Project v1 onto v2
v_mid = v1.lerp(v2, 0.5)           # Midpoint

# Distance between two points (FAST pattern — no sqrt)
dist_sq = (v1 - v2).length_squared  # Use for comparisons
dist = (v1 - v2).length             # Only when actual distance needed

# Swizzle access
xy = v1.xy                          # 2D Vector (1.0, 2.0)
xzy = v1.xzy                       # Rearranged (1.0, 3.0, 2.0)
```

---

## Example 2: Matrix Transformations

```python
# Blender 3.x/4.x/5.x — building and applying transformation matrices
from mathutils import Matrix, Vector
import math

# Build individual matrices
trans = Matrix.Translation(Vector((5.0, 0.0, 3.0)))
rot = Matrix.Rotation(math.radians(45), 4, 'Z')
scale = Matrix.Diagonal(Vector((2.0, 2.0, 2.0, 1.0)))

# Combine: Scale → Rotate → Translate (right-to-left application)
# CRITICAL: Use @ operator, NEVER *
transform = trans @ rot @ scale

# Apply to a point
point = Vector((1.0, 0.0, 0.0))
result = transform @ point

# Decompose back to components
loc, rot_q, sca = transform.decompose()
# loc: Vector, rot_q: Quaternion, sca: Vector

# Inverse
inv = transform.inverted()
identity_check = transform @ inv  # Should be identity

# Safe inverse (returns identity if singular)
inv_safe = transform.inverted_safe()

# Apply to Blender object
import bpy
obj = bpy.data.objects.get("Cube")
if obj:
    obj.matrix_world = transform
```

---

## Example 3: Quaternion Rotations

```python
# Blender 3.x/4.x/5.x — quaternion operations for smooth rotations
from mathutils import Quaternion, Vector
import math

# Create from axis-angle
q_90z = Quaternion(Vector((0, 0, 1)), math.radians(90))

# Rotate a vector
v = Vector((1, 0, 0))
v_rotated = q_90z @ v  # (0, 1, 0) approximately

# SLERP — smooth interpolation between rotations
q_start = Quaternion(Vector((0, 0, 1)), math.radians(0))
q_end = Quaternion(Vector((0, 0, 1)), math.radians(180))
q_mid = q_start.slerp(q_end, 0.5)  # 90 degrees

# Combine rotations (order matters)
q_combined = q_90z @ Quaternion(Vector((1, 0, 0)), math.radians(45))

# Convert to other representations
euler = q_90z.to_euler('XYZ')
matrix_3x3 = q_90z.to_matrix()
axis, angle = q_90z.to_axis_angle()
```

---

## Example 4: Euler Angles with Gimbal-Safe Animation

```python
# Blender 3.x/4.x/5.x — Euler rotation with make_compatible
from mathutils import Euler
import math

# Rotation order matters
e = Euler((math.radians(90), 0, 0), 'XYZ')

# Convert to quaternion (ALWAYS preferred for interpolation)
q = e.to_quaternion()

# Avoid gimbal flips in animation keyframes
e_prev = Euler((math.radians(170), 0, 0))
e_curr = Euler((math.radians(-170), 0, 0))
e_curr.make_compatible(e_prev)  # Adjusts to equivalent closest to e_prev
# Now interpolation between e_prev and e_curr is smooth
```

---

## Example 5: KDTree for Nearest Vertex Queries

```python
# Blender 3.x/4.x/5.x — find nearest vertices to a point
import bpy
from mathutils.kdtree import KDTree

# Build KDTree from mesh vertices
obj = bpy.context.active_object
mesh = obj.data
kd = KDTree(len(mesh.vertices))

for i, vert in enumerate(mesh.vertices):
    kd.insert(vert.co, i)

kd.balance()  # MANDATORY — queries fail without this

# Find single nearest vertex
query = (5.0, 3.0, 0.0)
co, index, dist = kd.find(query)
print(f"Nearest vertex: index={index}, distance={dist:.3f}")

# Find 10 nearest vertices
for co, index, dist in kd.find_n(query, 10):
    print(f"  Vertex {index}: dist={dist:.3f}")

# Find all vertices within 2.0 units
for co, index, dist in kd.find_range(query, 2.0):
    print(f"  Vertex {index} within range: dist={dist:.3f}")
```

---

## Example 6: BVHTree for Ray Casting and Clash Detection

```python
# Blender 3.x/4.x/5.x — ray casting and overlap detection
import bpy
from mathutils import Vector
from mathutils.bvhtree import BVHTree

depsgraph = bpy.context.evaluated_depsgraph_get()

# Build BVHTree from evaluated object (includes modifiers)
obj = bpy.context.active_object
obj_eval = obj.evaluated_get(depsgraph)
bvh = BVHTree.FromObject(obj_eval, depsgraph)

# Ray cast — shoot ray downward from above
origin = Vector((0.0, 0.0, 10.0))
direction = Vector((0.0, 0.0, -1.0))
location, normal, index, distance = bvh.ray_cast(origin, direction)

if location is not None:
    print(f"Hit face {index} at {location}, distance={distance:.3f}")
else:
    print("No intersection")

# Find nearest point on surface
query = Vector((5.0, 3.0, 0.5))
location, normal, index, distance = bvh.find_nearest(query)
if location is not None:
    print(f"Nearest surface point on face {index}, distance={distance:.3f}")

# Clash detection between two objects
obj1 = bpy.data.objects["Building_A"]
obj2 = bpy.data.objects["Building_B"]
bvh1 = BVHTree.FromObject(obj1.evaluated_get(depsgraph), depsgraph)
bvh2 = BVHTree.FromObject(obj2.evaluated_get(depsgraph), depsgraph)

overlaps = bvh1.overlap(bvh2)
if overlaps:
    print(f"CLASH DETECTED: {len(overlaps)} overlapping face pairs")
    for idx1, idx2 in overlaps[:5]:
        print(f"  Object A face {idx1} ↔ Object B face {idx2}")
else:
    print("No clashes detected")
```

---

## Example 7: Thread-Safe Background Processing

```python
# Blender 3.x/4.x/5.x — offload work to thread, update bpy on main thread
import bpy
import threading
import queue

# Shared queue for main-thread execution
_main_queue = queue.Queue()

def _process_main_queue():
    """Timer callback — processes queued functions on the main thread."""
    while not _main_queue.empty():
        fn = _main_queue.get()
        try:
            fn()
        except Exception as e:
            print(f"Error in queued function: {e}")
    return 0.1  # Reschedule every 100ms

# Register the timer
bpy.app.timers.register(_process_main_queue)

def schedule_on_main(fn):
    """Schedule a callable to run on the main thread."""
    _main_queue.put(fn)

# Example: heavy computation in background thread
def compute_volumes():
    """Worker thread — computes volumes, then updates objects on main thread."""
    import time
    time.sleep(3)  # Simulate expensive computation

    # NEVER access bpy here — schedule it instead
    def update_results():
        obj = bpy.data.objects.get("ResultDisplay")
        if obj:
            obj["computed_volume"] = 42.5
            obj.location.z = 5.0

    schedule_on_main(update_results)

# Start background work
thread = threading.Thread(target=compute_volumes, daemon=True)
thread.start()
```

---

## Example 8: Application Handlers — Full Addon Pattern

```python
# Blender 3.x/4.x/5.x — handler registration in addon lifecycle
import bpy
from bpy.app.handlers import persistent

@persistent
def on_file_loaded(filepath):
    """Re-initialize addon state after file load."""
    print(f"File loaded: {bpy.data.filepath}")
    # Re-scan scene, rebuild caches, etc.

@persistent
def on_depsgraph_update(scene, depsgraph):
    """React to scene changes."""
    for update in depsgraph.updates:
        if update.is_updated_geometry:
            print(f"Geometry changed: {update.id.name}")

@persistent
def on_undo_post(scene):
    """Rebuild internal state after undo — references are invalid."""
    print("Undo detected — rebuilding state from names")
    # All stored bpy references are NOW INVALID
    # Re-fetch everything by name

_handlers = [
    (bpy.app.handlers.load_post, on_file_loaded),
    (bpy.app.handlers.depsgraph_update_post, on_depsgraph_update),
    (bpy.app.handlers.undo_post, on_undo_post),
]

def register():
    for handler_list, handler_fn in _handlers:
        if handler_fn not in handler_list:
            handler_list.append(handler_fn)

def unregister():
    for handler_list, handler_fn in _handlers:
        if handler_fn in handler_list:
            handler_list.remove(handler_fn)
```

---

## Example 9: Message Bus — Watch Object Properties

```python
# Blender 3.x/4.x/5.x — subscribe to property changes via msgbus
import bpy

_msgbus_owner = object()  # Unique owner for cleanup

def on_active_object_location_changed(*args):
    obj = bpy.context.active_object
    if obj:
        print(f"{obj.name} moved to {obj.location[:]}")

def on_any_object_name_changed(*args):
    print("Some object was renamed")

def register_subscriptions():
    # Watch active object's location (instance-level)
    if bpy.context.active_object:
        bpy.msgbus.subscribe_rna(
            key=bpy.context.active_object.location,
            owner=_msgbus_owner,
            args=(),
            notify=on_active_object_location_changed,
        )

    # Watch ALL object name changes (type-level)
    bpy.msgbus.subscribe_rna(
        key=(bpy.types.Object, "name"),
        owner=_msgbus_owner,
        args=(),
        notify=on_any_object_name_changed,
        options={'PERSISTENT'},  # Survives file load
    )

def unregister_subscriptions():
    bpy.msgbus.clear_by_owner(_msgbus_owner)
```

---

## Example 10: Timer — One-Shot and Repeating

```python
# Blender 3.x/4.x/5.x — timer patterns
import bpy

# One-shot timer: execute once after delay
def show_notification():
    # This runs on the main thread, safe to call bpy
    bpy.context.window_manager.popup_menu(
        lambda self, context: self.layout.label(text="Task complete!"),
        title="Info", icon='INFO')
    return None  # One-shot: unregister after execution

bpy.app.timers.register(show_notification, first_interval=5.0)

# Repeating timer: poll external resource
_poll_count = 0

def poll_external_service():
    global _poll_count
    _poll_count += 1

    # Simulate checking an external service
    if _poll_count >= 10:
        print("Polling complete — unregistering")
        return None  # Stop after 10 polls

    print(f"Poll #{_poll_count}")
    return 2.0  # Re-run every 2 seconds

bpy.app.timers.register(poll_external_service, first_interval=0.0)

# Persistent timer (survives file load)
def persistent_monitor():
    if bpy.data.is_dirty:
        print("Unsaved changes detected")
    return 30.0  # Check every 30 seconds

bpy.app.timers.register(persistent_monitor, persistent=True)
```

---

## Example 11: Background Mode Script

```python
# Blender 3.x/4.x/5.x — headless batch processing script
# Run with: blender -b scene.blend -P this_script.py -- --output /tmp/result
import bpy
import sys

def main():
    if not bpy.app.background:
        print("WARNING: This script is designed for background mode")

    # Parse custom arguments after "--"
    argv = sys.argv
    if "--" in argv:
        custom_args = argv[argv.index("--") + 1:]
    else:
        custom_args = []

    output_path = "/tmp/render"
    for i, arg in enumerate(custom_args):
        if arg == "--output" and i + 1 < len(custom_args):
            output_path = custom_args[i + 1]

    # Background-safe operations
    scene = bpy.context.scene
    scene.render.filepath = output_path
    scene.render.image_settings.file_format = 'PNG'

    # Render (works in background mode)
    bpy.ops.render.render(write_still=True)
    print(f"Rendered to {output_path}")

if __name__ == "__main__":
    main()
```

---

## Example 12: BVHTree from BMesh (Edit Mode Compatible)

```python
# Blender 3.x/4.x/5.x — BVHTree from BMesh for edit-mode queries
import bpy
import bmesh
from mathutils import Vector
from mathutils.bvhtree import BVHTree

obj = bpy.context.edit_object  # Must be in edit mode
bm = bmesh.from_edit_mesh(obj.data)
bm.faces.ensure_lookup_table()

# Build BVHTree from BMesh (local space)
bvh = BVHTree.FromBMesh(bm)

# Ray cast in local space
origin = Vector((0, 0, 5))
direction = Vector((0, 0, -1))
location, normal, index, distance = bvh.ray_cast(origin, direction)

if location is not None:
    face = bm.faces[index]
    print(f"Hit face {index} with {len(face.verts)} vertices")

# Convert local result to world space
if location is not None:
    world_loc = obj.matrix_world @ location
    print(f"World space hit: {world_loc[:]}")
```

---

## Example 13: Safe Reference Wrapper for Modal Operators

```python
# Blender 3.x/4.x/5.x — undo-safe reference wrapper
import bpy

class SafeRef:
    """Stores object by name, re-fetches on access. Undo-safe."""

    def __init__(self, bpy_object):
        self._name = bpy_object.name
        self._collection = type(bpy_object).__name__  # "Object", "Mesh", etc.

    @property
    def resolve(self):
        """Re-fetch the object by name. Returns None if deleted."""
        collection_map = {
            "Object": bpy.data.objects,
            "Mesh": bpy.data.meshes,
            "Material": bpy.data.materials,
            "Collection": bpy.data.collections,
        }
        collection = collection_map.get(self._collection)
        if collection is None:
            return None
        return collection.get(self._name)

    @property
    def exists(self):
        return self.resolve is not None

# Usage in a modal operator
class MY_OT_safe_modal(bpy.types.Operator):
    bl_idname = "my.safe_modal"
    bl_label = "Safe Modal"

    def invoke(self, context, event):
        # Store name reference, NOT bpy pointer
        self._target = SafeRef(context.active_object)
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        obj = self._target.resolve
        if obj is None:
            self.report({'WARNING'}, "Target object was deleted")
            return {'CANCELLED'}

        if event.type == 'LEFTMOUSE' and event.value == 'PRESS':
            obj.location.x += 1.0
            return {'RUNNING_MODAL'}

        if event.type == 'ESC':
            return {'CANCELLED'}

        return {'PASS_THROUGH'}
```

---

## Example 14: bl_math in Driver Expressions

```python
# Blender 3.x/4.x/5.x — using bl_math functions in drivers
import bpy

obj = bpy.data.objects.get("Cube")
if obj:
    # Add driver on Z location
    fcurve = obj.driver_add("location", 2)  # Z axis
    driver = fcurve.driver

    # Use bl_math.smoothstep in driver expression
    driver.type = 'SCRIPTED'

    # Add variable for frame
    var = driver.variables.new()
    var.name = "frame"
    var.type = 'SINGLE_PROP'
    var.targets[0].id_type = 'SCENE'
    var.targets[0].id = bpy.context.scene
    var.targets[0].data_path = "frame_current"

    # Expression using bl_math (available in driver namespace)
    driver.expression = "smoothstep(1, 100, frame) * 10.0"
    # Object smoothly rises from Z=0 to Z=10 over frames 1-100
```

---

## Example 15: Handling Blender 5.0 Property Changes

```python
# Version-aware property access pattern
import bpy

obj = bpy.data.objects.get("MyObject")
if obj is None:
    raise RuntimeError("Object not found")

# Setting custom properties (works in ALL versions)
obj["my_custom_prop"] = 42

# Resetting RNA-defined properties
if bpy.app.version >= (5, 0, 0):
    # Blender 5.0+ — del on RNA props is REMOVED
    obj.property_unset("location")  # Reset to default
else:
    # Blender 3.x/4.x — del works on RNA props
    del obj["location"]

# Accessing render engine settings
scene = bpy.context.scene
if bpy.app.version >= (5, 0, 0):
    # Blender 5.0+ — dict-style access to RNA props is REMOVED
    cycles = scene.cycles  # Attribute access ONLY
else:
    # Blender 3.x/4.x — both work
    cycles = scene.cycles
    # cycles_dict = scene["cycles"]  # Also works but avoid for forward compat
```
