# blender-syntax-modifiers: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/bpy.types.ObjectModifiers.html
- https://docs.blender.org/api/current/bpy.types.Modifier.html
- https://docs.blender.org/api/current/bpy.types.NodesModifier.html
- https://docs.blender.org/api/current/bpy.types.Depsgraph.html

---

## Example 1: Add Multiple Modifiers to an Object

```python
# Blender 3.x/4.x/5.x — add and configure multiple modifiers
import bpy

obj = bpy.data.objects.get("Wall")
if obj is None:
    raise RuntimeError("Object 'Wall' not found")

# Add Solidify for wall thickness
solidify = obj.modifiers.new(name="Thickness", type='SOLIDIFY')
solidify.thickness = 0.3
solidify.offset = -1.0  # Extrude outward
solidify.use_even_offset = True

# Add Array for repetition
array = obj.modifiers.new(name="Repeat", type='ARRAY')
array.count = 5
array.use_relative_offset = True
array.relative_offset_displace = (1.0, 0.0, 0.0)

# Add Bevel for edge finishing
bevel = obj.modifiers.new(name="EdgeBevel", type='BEVEL')
bevel.width = 0.02
bevel.segments = 2
bevel.limit_method = 'ANGLE'
bevel.angle_limit = 0.523599  # 30 degrees in radians

print(f"Modifiers on {obj.name}: {[m.name for m in obj.modifiers]}")
```

---

## Example 2: Apply All Modifiers on an Object

```python
# Blender 3.2+/4.x/5.x — apply all modifiers safely
import bpy

obj = bpy.data.objects.get("Wall")
if obj is None:
    raise RuntimeError("Object 'Wall' not found")

# Ensure Object Mode (modifier_apply requires it)
if bpy.context.mode != 'OBJECT':
    bpy.ops.object.mode_set(mode='OBJECT')

# Copy the modifier list (stack changes as modifiers are applied)
modifier_names = [mod.name for mod in obj.modifiers]

with bpy.context.temp_override(object=obj, active_object=obj):
    for name in modifier_names:
        try:
            bpy.ops.object.modifier_apply(modifier=name)
            print(f"Applied: {name}")
        except RuntimeError as e:
            print(f"Failed to apply '{name}': {e}")
            # Some modifiers cannot be applied (e.g., physics without bake)

print(f"Remaining modifiers: {len(obj.modifiers)}")
```

---

## Example 3: Boolean Difference for Window Openings

```python
# Blender 3.x/4.x/5.x — create window openings in a wall
import bpy

wall = bpy.data.objects.get("Wall")
if wall is None:
    raise RuntimeError("Wall object not found")

# Create window opening cutter
bpy.ops.mesh.primitive_cube_add(
    size=1.0,
    location=(2.0, 0.0, 1.5)
)
cutter = bpy.context.active_object
cutter.name = "WindowCutter"
cutter.scale = (1.2, 0.5, 1.5)  # Window dimensions

# Add Boolean modifier
boolean = wall.modifiers.new(name="WindowOpening", type='BOOLEAN')
boolean.operation = 'DIFFERENCE'
boolean.object = cutter
boolean.solver = 'EXACT'  # EXACT handles coplanar faces better for AEC

# Hide the cutter
cutter.hide_set(True)
cutter.hide_render = True

# Optional: apply the boolean
with bpy.context.temp_override(object=wall, active_object=wall):
    bpy.ops.object.modifier_apply(modifier="WindowOpening")

# Clean up cutter after applying
bpy.data.objects.remove(cutter, do_unlink=True)
```

---

## Example 4: Array Along a Curve (Railing/Fence Pattern)

```python
# Blender 3.x/4.x/5.x — distribute posts along a curve
import bpy

# Assumes "Post" object and "RailingPath" curve exist
post = bpy.data.objects.get("Post")
path = bpy.data.objects.get("RailingPath")
if post is None or path is None:
    raise RuntimeError("Post and RailingPath objects required")

# Array modifier fitted to curve
array = post.modifiers.new(name="PostArray", type='ARRAY')
array.fit_type = 'FIT_CURVE'
array.curve = path
array.use_relative_offset = True
array.relative_offset_displace = (0.0, 0.0, 1.0)  # Adjust per object orientation

# Curve modifier to follow the path
curve_mod = post.modifiers.new(name="FollowPath", type='CURVE')
curve_mod.object = path
curve_mod.deform_axis = 'POS_Z'  # Adjust per object orientation
```

---

## Example 5: Read Evaluated Mesh After Modifiers

```python
# Blender 3.x/4.x/5.x — export post-modifier vertex positions
import bpy

obj = bpy.data.objects.get("Building")
if obj is None or obj.type != 'MESH':
    raise RuntimeError("Mesh object 'Building' required")

depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()

try:
    # Collect world-space vertex positions
    vertices = []
    for v in mesh_eval.vertices:
        world_co = obj_eval.matrix_world @ v.co
        vertices.append((world_co.x, world_co.y, world_co.z))

    # Collect face vertex indices
    faces = []
    for poly in mesh_eval.polygons:
        faces.append(tuple(poly.vertices))

    print(f"Evaluated: {len(vertices)} verts, {len(faces)} faces")
    print(f"Original: {len(obj.data.vertices)} verts, {len(obj.data.polygons)} faces")
finally:
    obj_eval.to_mesh_clear()  # ALWAYS clean up
```

---

## Example 6: Geometry Nodes Modifier — Set All Inputs Programmatically

```python
# Blender 4.0+ — configure all GN inputs by name
import bpy

obj = bpy.data.objects.get("Parametric_Wall")
if obj is None:
    raise RuntimeError("Object not found")

# Add Geometry Nodes modifier
mod = obj.modifiers.new(name="WallGen", type='NODES')
node_group = bpy.data.node_groups.get("AEC_WallGenerator")
if node_group is None:
    raise RuntimeError("Node group not found")
mod.node_group = node_group

# Build a name-to-identifier mapping
input_map = {}
for item in mod.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        input_map[item.name] = item.identifier

# Set values by name
params = {
    "Height": 3.0,
    "Thickness": 0.3,
    "Length": 10.0,
    "Segments": 20,
}

for name, value in params.items():
    identifier = input_map.get(name)
    if identifier is None:
        print(f"WARNING: Input '{name}' not found in node group")
        continue
    mod[identifier] = value
    print(f"Set {name} ({identifier}) = {value}")

# Force update
bpy.context.view_layer.update()
```

---

## Example 7: Geometry Nodes — Set Object/Collection Inputs

```python
# Blender 4.0+ — set pointer-type inputs on GN modifier
import bpy

obj = bpy.data.objects.get("Distributor")
mod = obj.modifiers.get("GeoNodes")
if mod is None or mod.node_group is None:
    raise RuntimeError("GN modifier with node group required")

# Find identifiers for pointer inputs
for item in mod.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.socket_type == 'NodeSocketObject':
            # Set an Object input
            target_obj = bpy.data.objects.get("ColumnBase")
            if target_obj:
                mod[item.identifier] = target_obj
                print(f"Set object input '{item.name}' to {target_obj.name}")

        elif item.socket_type == 'NodeSocketCollection':
            # Set a Collection input
            col = bpy.data.collections.get("Furniture")
            if col:
                mod[item.identifier] = col
                print(f"Set collection input '{item.name}' to {col.name}")

        elif item.socket_type == 'NodeSocketMaterial':
            # Set a Material input
            mat = bpy.data.materials.get("Concrete")
            if mat:
                mod[item.identifier] = mat
                print(f"Set material input '{item.name}' to {mat.name}")

bpy.context.view_layer.update()
```

---

## Example 8: Conditional Modifier Application with Version Check

```python
# Blender 3.x/4.x/5.x — version-aware modifier application
import bpy

BLENDER_VERSION = bpy.app.version


def apply_modifier_safe(obj, modifier_name):
    """Apply a modifier using the correct API for the running Blender version."""
    # Verify modifier exists
    mod = obj.modifiers.get(modifier_name)
    if mod is None:
        raise ValueError(f"Modifier '{modifier_name}' not found on '{obj.name}'")

    # Ensure Object Mode
    if bpy.context.mode != 'OBJECT':
        bpy.ops.object.mode_set(mode='OBJECT')

    if BLENDER_VERSION >= (3, 2, 0):
        # Blender 3.2+/4.x/5.x — use temp_override
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        # Blender < 3.2 — legacy dict override
        override = bpy.context.copy()
        override['object'] = obj
        override['active_object'] = obj
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)


# Usage
obj = bpy.data.objects.get("Wall")
if obj:
    apply_modifier_safe(obj, "Solidify")
```

---

## Example 9: Mirror Modifier for Symmetric Buildings

```python
# Blender 3.x/4.x/5.x — mirror building geometry
import bpy

obj = bpy.data.objects.get("BuildingHalf")
if obj is None:
    raise RuntimeError("Object not found")

mirror = obj.modifiers.new(name="Symmetry", type='MIRROR')
mirror.use_axis = (True, False, False)  # Mirror on X axis
mirror.use_bisect_axis = (True, False, False)  # Cut geometry at axis
mirror.use_clip = True  # Prevent vertices from crossing mirror axis
mirror.merge_threshold = 0.001  # Merge vertices at seam

# Use an empty as mirror center (optional)
empty = bpy.data.objects.get("MirrorCenter")
if empty:
    mirror.mirror_object = empty
```

---

## Example 10: Solidify with Variable Thickness via Vertex Group

```python
# Blender 3.x/4.x/5.x — variable wall thickness using vertex group
import bpy

obj = bpy.data.objects.get("Slab")
if obj is None or obj.type != 'MESH':
    raise RuntimeError("Mesh object 'Slab' required")

# Create vertex group for edge thickening
vg = obj.vertex_groups.new(name="EdgeThickness")

# Assign weights (1.0 = full thickness, 0.0 = no thickness)
mesh = obj.data
for v in mesh.vertices:
    # Thicken edges (vertices far from center)
    dist = (v.co.x**2 + v.co.y**2) ** 0.5
    weight = min(dist / 5.0, 1.0)  # Normalize to 0-1 range
    vg.add([v.index], weight, 'REPLACE')

# Add Solidify with vertex group
solidify = obj.modifiers.new(name="VariableThickness", type='SOLIDIFY')
solidify.thickness = 0.5
solidify.vertex_group = "EdgeThickness"
solidify.thickness_vertex_group = 1.0  # Factor for vertex group influence
solidify.use_even_offset = True
```

---

## Example 11: Batch Modify All Objects in a Collection

```python
# Blender 3.x/4.x/5.x — add modifiers to all mesh objects in a collection
import bpy

collection = bpy.data.collections.get("Walls")
if collection is None:
    raise RuntimeError("Collection 'Walls' not found")

for obj in collection.objects:
    if obj.type != 'MESH':
        continue

    # Skip if already has a Solidify modifier
    if obj.modifiers.get("WallThickness") is not None:
        continue

    mod = obj.modifiers.new(name="WallThickness", type='SOLIDIFY')
    mod.thickness = 0.2
    mod.offset = -1.0
    mod.use_even_offset = True
    print(f"Added Solidify to {obj.name}")
```

---

## Example 12: BVHTree from Evaluated Mesh (Blender 4.0+)

```python
# Blender 4.0+ — raycasting on post-modifier geometry
import bpy
from mathutils import Vector
from mathutils.bvhtree import BVHTree

obj = bpy.data.objects.get("Building")
if obj is None:
    raise RuntimeError("Object not found")

depsgraph = bpy.context.evaluated_depsgraph_get()

# Build BVH tree from evaluated object (depsgraph REQUIRED in 4.0+)
bvh = BVHTree.FromObject(obj, depsgraph)

# Raycast
origin = Vector((0, -10, 1.5))
direction = Vector((0, 1, 0))
location, normal, index, distance = bvh.ray_cast(origin, direction)

if location:
    print(f"Hit at {location}, normal {normal}, face index {index}, distance {distance}")
else:
    print("No hit")
```

---

## Example 13: Count Evaluated Vertices for Multiple Objects

```python
# Blender 3.x/4.x/5.x — compare original vs evaluated vertex counts
import bpy

depsgraph = bpy.context.evaluated_depsgraph_get()

for obj in bpy.data.objects:
    if obj.type != 'MESH':
        continue
    if not obj.modifiers:
        continue  # Skip objects without modifiers

    obj_eval = obj.evaluated_get(depsgraph)
    mesh_eval = obj_eval.to_mesh()
    try:
        orig_count = len(obj.data.vertices)
        eval_count = len(mesh_eval.vertices)
        ratio = eval_count / orig_count if orig_count > 0 else 0
        print(f"{obj.name}: {orig_count} -> {eval_count} vertices ({ratio:.1f}x)")
    finally:
        obj_eval.to_mesh_clear()
```
