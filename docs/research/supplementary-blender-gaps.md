# Supplementary Research: Blender Python API Gap Domains

**Date**: 2026-03-05
**Status**: COMPLETE
**Author**: Research Worker Agent (Prompt G — Worker 1)
**Scope**: Gap domains NOT covered by vooronderzoek-blender.md
**Versions Covered**: Blender 3.x, 4.0, 4.1, 4.2 LTS, 4.3, 5.0

---

## Table of Contents

1. [Node Systems via Python](#1-node-systems-via-python)
2. [Animation & Rigging](#2-animation--rigging)
3. [Materials & Shading](#3-materials--shading)
4. [Rendering](#4-rendering)
5. [I/O Formats](#5-io-formats)
6. [Collections, Libraries & Assets](#6-collections-libraries--assets)
7. [mathutils & Standalone Modules](#7-mathutils--standalone-modules)
8. [Python Runtime Quirks](#8-python-runtime-quirks)

---

## 1. Node Systems via Python

Node systems in Blender (Geometry Nodes, Shader Nodes, Compositor Nodes) share a common architecture: `NodeTree` objects contain `Node` objects connected via `NodeLink` objects between `NodeSocket` endpoints. All node trees are stored in `bpy.data.node_groups` (for reusable groups) or directly on materials/scenes.

### 1.1 Core Node Tree Architecture

| Component | Type | Access Pattern |
|-----------|------|---------------|
| Node Tree | `bpy.types.NodeTree` | `bpy.data.node_groups["Name"]` |
| Node | `bpy.types.Node` | `tree.nodes["NodeName"]` |
| Node Link | `bpy.types.NodeLink` | `tree.links.new(output_socket, input_socket)` |
| Node Socket | `bpy.types.NodeSocket` | `node.inputs["Name"]` / `node.outputs["Name"]` |

**Node tree types** (the `type` parameter for `bpy.data.node_groups.new()`):

| Tree Type String | Purpose |
|-----------------|---------|
| `'GeometryNodeTree'` | Geometry Nodes |
| `'ShaderNodeTree'` | Material/World shader nodes |
| `'CompositorNodeTree'` | Compositor nodes |
| `'TextureNodeTree'` | Legacy texture nodes |

### 1.2 Creating a Node Tree Programmatically

```python
# Blender 4.0+ — Create a Geometry Nodes tree with interface sockets
import bpy

# Create the node group
node_group = bpy.data.node_groups.new("AEC_WallGenerator", 'GeometryNodeTree')

# Clear default nodes
node_group.nodes.clear()

# Add Group Input and Group Output nodes
group_input = node_group.nodes.new('NodeGroupInput')
group_input.location = (-300, 0)

group_output = node_group.nodes.new('NodeGroupOutput')
group_output.location = (300, 0)

# Create interface sockets (Blender 4.0+ API)
node_group.interface.new_socket(
    name="Geometry",
    in_out='INPUT',
    socket_type='NodeSocketGeometry'
)
node_group.interface.new_socket(
    name="Width",
    in_out='INPUT',
    socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Geometry",
    in_out='OUTPUT',
    socket_type='NodeSocketGeometry'
)

# Add a Cube Mesh node
cube_node = node_group.nodes.new('GeometryNodeMeshCube')
cube_node.location = (0, 0)

# Link nodes: Group Input "Width" -> Cube "Size X"
node_group.links.new(
    group_input.outputs["Width"],   # source socket
    cube_node.inputs["Size"]        # destination socket
)

# Link Cube output -> Group Output
node_group.links.new(
    cube_node.outputs["Mesh"],
    group_output.inputs["Geometry"]
)
```

### 1.3 Node Group Interface: 3.x vs 4.0+ Migration

```python
# Blender 3.x (LEGACY — BROKEN in 4.0+)
node_group.inputs.new("NodeSocketFloat", "My Input")
node_group.outputs.new("NodeSocketGeometry", "Geometry")

# Blender 4.0+ (CURRENT)
node_group.interface.new_socket(
    name="My Input",
    in_out='INPUT',
    socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Geometry",
    in_out='OUTPUT',
    socket_type='NodeSocketGeometry'
)
```

**`interface.new_socket()` parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Display name of the socket |
| `in_out` | `str` | `'INPUT'` or `'OUTPUT'` |
| `socket_type` | `str` | Socket type identifier (see table below) |
| `parent` | `NodeTreeInterfacePanel` | Optional panel to place the socket in |

**Common socket types:**

| Socket Type | Data Type |
|------------|-----------|
| `'NodeSocketFloat'` | Single float |
| `'NodeSocketInt'` | Integer |
| `'NodeSocketBool'` | Boolean |
| `'NodeSocketVector'` | 3D vector |
| `'NodeSocketColor'` | RGBA color |
| `'NodeSocketString'` | String |
| `'NodeSocketGeometry'` | Geometry data |
| `'NodeSocketMaterial'` | Material reference |
| `'NodeSocketObject'` | Object reference |
| `'NodeSocketCollection'` | Collection reference |
| `'NodeSocketImage'` | Image reference |

### 1.4 Interface Panels (Blender 4.1+)

Blender 4.1 added panel support to group interface sockets:

```python
# Blender 4.1+ — Create a panel in the interface
panel = node_group.interface.new_panel("Wall Parameters")
node_group.interface.new_socket(
    name="Height",
    in_out='INPUT',
    socket_type='NodeSocketFloat',
    parent=panel
)
node_group.interface.new_socket(
    name="Thickness",
    in_out='INPUT',
    socket_type='NodeSocketFloat',
    parent=panel
)
```

### 1.5 Assigning a Geometry Node Group to a Modifier

```python
# Blender 4.0+
obj = bpy.context.active_object
modifier = obj.modifiers.new(name="GeoNodes", type='NODES')
modifier.node_group = bpy.data.node_groups["AEC_WallGenerator"]

# Set modifier input values (identifier-based access)
# The identifier corresponds to the socket name in the interface
modifier["Socket_2"] = 3.5  # Set "Width" input value
```

**CRITICAL**: Modifier input identifiers are auto-generated as `Socket_N` (where N is an incrementing integer). The identifier is NOT the socket name. To find the correct identifier, iterate `modifier.node_group.interface.items_tree` and match by name.

```python
# Blender 4.0+ — Find the correct modifier input identifier
for item in modifier.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        print(f"Name: {item.name}, Identifier: {item.identifier}")
        # Use: modifier[item.identifier] = value
```

### 1.6 Shader Node Trees

```python
# Blender 3.x / 4.0+ — Create material with shader nodes
mat = bpy.data.materials.new("AEC_Concrete")
mat.use_nodes = True
tree = mat.node_tree

# Clear defaults
tree.nodes.clear()

# Add Principled BSDF
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (0, 0)

# Add Material Output
output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (300, 0)

# Link Principled BSDF -> Output
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
```

### 1.7 Compositor Node Trees

```python
# Blender 3.x / 4.x — Access compositor
scene = bpy.context.scene
scene.use_nodes = True  # DEPRECATED in 5.0 (always True, setting has no effect)
comp_tree = scene.node_tree  # REMOVED in Blender 5.0

# Blender 5.0+ — Compositor uses compositing_node_group
comp_tree = bpy.data.node_groups.new("MyCompositor", 'CompositorNodeTree')
scene.compositing_node_group = comp_tree

# Add Render Layers node
render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
render_layers.location = (-300, 0)

# Add Composite output
composite = comp_tree.nodes.new('CompositorNodeComposite')
composite.location = (300, 0)

# Link
comp_tree.links.new(render_layers.outputs["Image"], composite.inputs["Image"])
```

### 1.8 Anti-Patterns: Node Systems

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| `tree.inputs.new()` in 4.0+ | API moved to `tree.interface` | Use `tree.interface.new_socket()` |
| `node.inputs[0]` by index | Indices shift when sockets are added/removed | Use `node.inputs["Name"]` by name |
| `scene.node_tree` in 5.0+ | Removed; compositor uses node groups | Use `scene.compositing_node_group` |
| Modifying node trees during render | Race condition with render thread | ALWAYS modify nodes before render starts |
| `modifier["Width"]` by name | Modifier inputs use auto-generated identifiers | Iterate `interface.items_tree` to find identifier |

### Sources
- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://docs.blender.org/api/current/bpy.types.NodeSocket.html
- https://docs.blender.org/api/current/bpy.ops.node.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/

---

## 2. Animation & Rigging

### 2.1 Keyframe Insertion

The primary method for inserting keyframes is `bpy_struct.keyframe_insert()`, available on any Blender data type with animatable properties.

```python
# Blender 3.x / 4.x / 5.x — Insert keyframes on object location
obj = bpy.context.active_object

# Set value and insert keyframe
obj.location = (0.0, 0.0, 0.0)
obj.keyframe_insert(data_path="location", frame=1)

obj.location = (5.0, 0.0, 3.0)
obj.keyframe_insert(data_path="location", frame=60)

# Insert keyframe on a single axis (index parameter)
obj.keyframe_insert(data_path="location", frame=30, index=2)  # Z-axis only
```

**`keyframe_insert()` parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `data_path` | `str` | RNA path to the property (e.g., `"location"`, `"rotation_euler"`) |
| `index` | `int` | Array index for vector properties (-1 = all, 0 = X, 1 = Y, 2 = Z) |
| `frame` | `float` | Frame number to insert the keyframe at |
| `group` | `str` | Name of the action group for this FCurve |
| `options` | `set` | `{'INSERTKEY_NEEDED'}`, `{'INSERTKEY_VISUAL'}`, etc. |

### 2.2 FCurves (Animation Curves)

FCurves are the low-level animation data structure. Each FCurve maps a single property (identified by `data_path` + `array_index`) to a function of time (frame number).

```python
# Blender 3.x / 4.x / 5.x — Access and manipulate FCurves
obj = bpy.context.active_object

# Access animation data
anim_data = obj.animation_data
if anim_data is None:
    anim_data = obj.animation_data_create()

# Access or create an Action
action = anim_data.action
if action is None:
    action = bpy.data.actions.new(name="MyAction")
    anim_data.action = action

# Access FCurves
for fcurve in action.fcurves:
    print(f"Path: {fcurve.data_path}, Index: {fcurve.array_index}")

    # Access keyframe points
    for kp in fcurve.keyframe_points:
        print(f"  Frame: {kp.co[0]}, Value: {kp.co[1]}")

# Create an FCurve manually
fcurve = action.fcurves.new(data_path="location", index=0)  # X location

# Add keyframe points
fcurve.keyframe_points.add(count=3)
fcurve.keyframe_points[0].co = (1.0, 0.0)
fcurve.keyframe_points[1].co = (30.0, 5.0)
fcurve.keyframe_points[2].co = (60.0, 0.0)

# ALWAYS call update after manual keyframe changes
fcurve.update()
```

**FCurve key properties:**

| Property | Type | Description |
|----------|------|-------------|
| `data_path` | `str` | RNA path to the animated property |
| `array_index` | `int` | Index for vector properties |
| `keyframe_points` | `FCurveKeyframePoints` | Collection of keyframe points |
| `modifiers` | `FCurveModifiers` | Modifiers (Noise, Cycles, etc.) |
| `driver` | `Driver` | Driver object (for driver FCurves) |
| `mute` | `bool` | Silence this FCurve |
| `hide` | `bool` | Hide in the Graph Editor |

**FCurve key methods:**

| Method | Description |
|--------|-------------|
| `evaluate(frame)` | Get interpolated value at a frame |
| `range()` | Returns (min_frame, max_frame) tuple |
| `update()` | Recalculate curve after manual changes (MUST call) |
| `convert_to_samples(start, end)` | Convert keyframes to sampled points |
| `convert_to_keyframes(start, end)` | Convert samples back to keyframes |

### 2.3 Keyframe Interpolation

```python
# Blender 3.x / 4.x / 5.x — Set interpolation type
for kp in fcurve.keyframe_points:
    kp.interpolation = 'LINEAR'  # Options: 'CONSTANT', 'LINEAR', 'BEZIER', 'SINE', 'QUAD', etc.
    kp.easing = 'AUTO'           # Options: 'AUTO', 'EASE_IN', 'EASE_OUT', 'EASE_IN_OUT'

    # Bezier handle control
    kp.handle_left_type = 'AUTO_CLAMPED'  # 'FREE', 'ALIGNED', 'VECTOR', 'AUTO', 'AUTO_CLAMPED'
    kp.handle_right_type = 'AUTO_CLAMPED'
```

### 2.4 Drivers

Drivers connect property values to expressions or other property values, enabling procedural animation.

```python
# Blender 3.x / 4.x / 5.x — Add a driver to a property
obj = bpy.context.active_object

# Add driver to Z location
driver_fcurve = obj.driver_add("location", 2)  # Returns FCurve with driver
driver = driver_fcurve.driver

# Set driver type
driver.type = 'SCRIPTED'  # Options: 'AVERAGE', 'SUM', 'SCRIPTED', 'MIN', 'MAX'

# Add a variable
var = driver.variables.new()
var.name = "ctrl_height"
var.type = 'TRANSFORMS'  # Options: 'SINGLE_PROP', 'TRANSFORMS', 'ROTATION_DIFF', 'LOC_DIFF'

# Configure the variable target
target = var.targets[0]
target.id = bpy.data.objects["ControlEmpty"]
target.transform_type = 'LOC_Z'
target.transform_space = 'WORLD_SPACE'

# Set the expression
driver.expression = "ctrl_height * 2.0"

# Remove a driver
obj.driver_remove("location", 2)
```

**Driver variable types:**

| Type | Description |
|------|-------------|
| `'SINGLE_PROP'` | Value of an RNA property |
| `'TRANSFORMS'` | Transform channel of an object/bone |
| `'ROTATION_DIFF'` | Rotation difference between two bones |
| `'LOC_DIFF'` | Distance between two objects/bones |

### 2.5 Constraints

```python
# Blender 3.x / 4.x / 5.x — Add constraints to objects
obj = bpy.context.active_object

# Add a Copy Location constraint
constraint = obj.constraints.new('COPY_LOCATION')
constraint.name = "Follow Target"
constraint.target = bpy.data.objects["TargetEmpty"]
constraint.use_x = True
constraint.use_y = True
constraint.use_z = True
constraint.influence = 0.5

# Add a Limit Location constraint
limit = obj.constraints.new('LIMIT_LOCATION')
limit.use_min_z = True
limit.min_z = 0.0  # Floor constraint
limit.owner_space = 'WORLD'

# Add bone constraint (in Pose Mode)
armature = bpy.data.objects["Armature"]
pose_bone = armature.pose.bones["Bone"]
ik = pose_bone.constraints.new('IK')
ik.target = bpy.data.objects["IKTarget"]
ik.chain_count = 3
```

**Common constraint types for AEC:**

| Type String | Constraint | AEC Use Case |
|------------|------------|-------------|
| `'COPY_LOCATION'` | Copy Location | Align objects to reference points |
| `'COPY_ROTATION'` | Copy Rotation | Mirror orientation from controls |
| `'LIMIT_LOCATION'` | Limit Location | Floor/ceiling boundaries |
| `'TRACK_TO'` | Track To | Camera aim at building |
| `'CHILD_OF'` | Child Of | Switchable parenting |
| `'FLOOR'` | Floor | Keep objects above ground plane |

### 2.6 Armatures and Bone Collections (4.0+)

```python
# Blender 4.0+ — Create armature with bone collections
import bpy

# Create armature
armature_data = bpy.data.armatures.new("BuildingRig")
armature_obj = bpy.data.objects.new("BuildingRig", armature_data)
bpy.context.collection.objects.link(armature_obj)

# MUST enter Edit Mode to create/edit bones
bpy.context.view_layer.objects.active = armature_obj
bpy.ops.object.mode_set(mode='EDIT')

# Create bones
root_bone = armature_data.edit_bones.new("Root")
root_bone.head = (0, 0, 0)
root_bone.tail = (0, 0, 1)

floor_bone = armature_data.edit_bones.new("Floor_01")
floor_bone.head = (0, 0, 0)
floor_bone.tail = (0, 0, 3)
floor_bone.parent = root_bone

# Exit Edit Mode
bpy.ops.object.mode_set(mode='OBJECT')

# Blender 4.0+ — Create bone collections (replaces bone layers and bone groups)
structure_col = armature_data.collections.new("Structure")
controls_col = armature_data.collections.new("Controls")

# Assign bones to collections
structure_col.assign(armature_data.bones["Root"])
controls_col.assign(armature_data.bones["Floor_01"])

# Set collection visibility
structure_col.is_visible = True
controls_col.is_visible = True
```

```python
# Blender 3.x (LEGACY — BROKEN in 4.0+)
# bone.layers[0] = True          # REMOVED
# pose.bone_groups.new(name="X") # REMOVED

# Blender 4.0+ — bone collections replace layers and groups
# Bone colors are now per-collection:
controls_col.color_tag = 'THEME01'  # Use COLOR_01..COLOR_20 or THEME01..THEME20
```

**CRITICAL**: `edit_bones` is ONLY accessible in Edit Mode. `bones` (read-only properties) is accessible in Object/Pose Mode. `pose.bones` (pose transforms, constraints) is accessible in Object/Pose Mode.

### 2.7 Actions and NLA Strips

```python
# Blender 3.x / 4.x / 5.x — NLA workflow
obj = bpy.context.active_object
anim_data = obj.animation_data

# Create an NLA track
track = anim_data.nla_tracks.new()
track.name = "FloorAnimation"

# Push an action as an NLA strip
action = bpy.data.actions["WalkCycle"]
strip = track.strips.new(
    name="Walk",
    start=1,           # Start frame on the NLA timeline
    action=action
)

# Configure the strip
strip.blend_type = 'REPLACE'     # 'REPLACE', 'COMBINE', 'ADD', 'SUBTRACT', 'MULTIPLY'
strip.use_auto_blend = True
strip.repeat = 3.0               # Loop 3 times
strip.scale = 1.0                # Speed multiplier
strip.mute = False

# Stash the current action into NLA (preserves it)
bpy.ops.nla.action_pushdown(channel_index=0)
```

### 2.8 Shape Keys

```python
# Blender 3.x / 4.x / 5.x — Shape keys for parametric deformation
obj = bpy.context.active_object
mesh = obj.data

# Add a basis shape key (MUST be added first)
basis = obj.shape_key_add(name="Basis", from_mix=False)

# Add a deformed shape key
deformed = obj.shape_key_add(name="OpenWindow", from_mix=False)

# Modify shape key vertices (relative to Basis)
for i, vert in enumerate(deformed.data):
    if i in window_vertex_indices:
        vert.co.z += 0.5  # Move window vertices up

# Animate shape key value
key_blocks = mesh.shape_keys.key_blocks["OpenWindow"]
key_blocks.value = 0.0
key_blocks.keyframe_insert(data_path="value", frame=1)
key_blocks.value = 1.0
key_blocks.keyframe_insert(data_path="value", frame=30)

# Add driver to shape key
driver_fc = mesh.shape_keys.key_blocks["OpenWindow"].driver_add("value")
driver = driver_fc.driver
driver.type = 'SCRIPTED'
driver.expression = "var"
var = driver.variables.new()
var.name = "var"
var.targets[0].id = bpy.data.objects["Controller"]
var.targets[0].data_path = 'location.z'
```

### 2.9 Anti-Patterns: Animation & Rigging

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| Forgetting `fcurve.update()` | Keyframe cache is stale | ALWAYS call `update()` after manual keyframe edits |
| Editing `edit_bones` outside Edit Mode | `edit_bones` is empty/inaccessible | ALWAYS enter Edit Mode first with `bpy.ops.object.mode_set(mode='EDIT')` |
| Using `bone.layers` in 4.0+ | Removed, replaced by bone collections | Use `armature.collections.new()` and `collection.assign()` |
| Using `pose.bone_groups` in 4.0+ | Removed | Use bone collections with `color_tag` |
| Inserting keyframes without setting value first | Keyframes record current value | ALWAYS set the property value BEFORE calling `keyframe_insert()` |
| Using `driver.expression` with `self` | `self` is not available in drivers by default | Enable `use_self` on the driver, or use variables |

### Sources
- https://docs.blender.org/api/current/bpy.types.FCurve.html
- https://docs.blender.org/api/current/bpy.types.Keyframe.html
- https://docs.blender.org/api/current/bpy.types.FCurveKeyframePoints.html
- https://docs.blender.org/api/current/bpy.types.NlaStrip.html
- https://docs.blender.org/api/current/bpy.types.NlaTrack.html
- https://docs.blender.org/api/current/bpy.types.ShapeKey.html
- https://docs.blender.org/api/current/bpy.types.BoneCollection.html
- https://docs.blender.org/api/current/bpy.types.EditBone.html
- https://developer.blender.org/docs/release_notes/4.0/upgrading/bone_collections/

---

## 3. Materials & Shading

### 3.1 Material Creation

```python
# Blender 3.x / 4.x / 5.x — Create and assign a material
import bpy

# Create material
mat = bpy.data.materials.new(name="AEC_Concrete")
mat.use_nodes = True  # Enable shader node tree

# Assign to object
obj = bpy.context.active_object
if obj.data.materials:
    obj.data.materials[0] = mat   # Replace first slot
else:
    obj.data.materials.append(mat)  # Add new slot

# Assign to specific faces (by material index)
# In Edit Mode or via mesh data:
for polygon in obj.data.polygons:
    polygon.material_index = 0  # Index into obj.data.materials
```

### 3.2 Shader Node Tree Setup

```python
# Blender 3.x / 4.x / 5.x — Build a complete material node tree
mat = bpy.data.materials.new("AEC_Brick")
mat.use_nodes = True
tree = mat.node_tree

# Clear default nodes
tree.nodes.clear()

# Add Principled BSDF
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (0, 0)

# Add Material Output
output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (400, 0)

# Add Image Texture
tex_node = tree.nodes.new('ShaderNodeTexImage')
tex_node.location = (-400, 0)

# Load image
img = bpy.data.images.load("/path/to/brick_diffuse.png")
tex_node.image = img

# Add Texture Coordinate and Mapping
tex_coord = tree.nodes.new('ShaderNodeTexCoord')
tex_coord.location = (-800, 0)

mapping = tree.nodes.new('ShaderNodeMapping')
mapping.location = (-600, 0)

# Link nodes
tree.links.new(tex_coord.outputs["UV"], mapping.inputs["Vector"])
tree.links.new(mapping.outputs["Vector"], tex_node.inputs["Vector"])
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
```

### 3.3 Principled BSDF Socket Names: 3.x vs 4.0+

This is one of the most impactful breaking changes for material scripts.

```python
# Blender 3.x (LEGACY — BROKEN in 4.0+)
principled.inputs["Subsurface"].default_value = 0.5
principled.inputs["Specular"].default_value = 0.5
principled.inputs["Transmission"].default_value = 1.0
principled.inputs["Coat"].default_value = 0.3
principled.inputs["Sheen"].default_value = 0.1
principled.inputs["Emission"].default_value = (1, 1, 1, 1)

# Blender 4.0+ (CURRENT)
principled.inputs["Subsurface Weight"].default_value = 0.5
principled.inputs["Specular IOR Level"].default_value = 0.5
principled.inputs["Transmission Weight"].default_value = 1.0
principled.inputs["Coat Weight"].default_value = 0.3
principled.inputs["Sheen Weight"].default_value = 0.1
principled.inputs["Emission Color"].default_value = (1, 1, 1, 1)
```

**Complete Principled BSDF socket rename table (3.x -> 4.0+):**

| Blender 3.x Socket Name | Blender 4.0+ Socket Name | Type Change |
|-------------------------|--------------------------|-------------|
| `Subsurface` | `Subsurface Weight` | No |
| `Subsurface Color` | **REMOVED** (use `Base Color`) | Removed |
| `Specular` | `Specular IOR Level` | No |
| `Specular Tint` | `Specular Tint` | Float -> Color |
| `Transmission` | `Transmission Weight` | No |
| `Coat` | `Coat Weight` | No |
| `Sheen` | `Sheen Weight` | No |
| `Emission` | `Emission Color` | No |

**Version-safe pattern:**

```python
# Blender 3.x / 4.0+ — Version-safe Principled BSDF setup
def set_principled_value(node, socket_3x, socket_4x, value):
    """Set Principled BSDF input, handling version differences."""
    if bpy.app.version >= (4, 0, 0):
        socket_name = socket_4x
    else:
        socket_name = socket_3x
    if socket_name in node.inputs:
        node.inputs[socket_name].default_value = value

# Usage
set_principled_value(principled, "Subsurface", "Subsurface Weight", 0.1)
set_principled_value(principled, "Specular", "Specular IOR Level", 0.5)
```

### 3.4 Common AEC Material Setups

```python
# Blender 4.0+ — AEC glass material
def create_glass_material(name="AEC_Glass", ior=1.45, transmission=1.0, roughness=0.0):
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    tree = mat.node_tree
    principled = tree.nodes["Principled BSDF"]

    principled.inputs["Base Color"].default_value = (0.8, 0.9, 1.0, 1.0)
    principled.inputs["Transmission Weight"].default_value = transmission
    principled.inputs["IOR"].default_value = ior
    principled.inputs["Roughness"].default_value = roughness
    principled.inputs["Alpha"].default_value = 0.3

    # Enable alpha blending for EEVEE
    mat.surface_render_method = 'BLENDED'  # Blender 4.2+
    # In Blender 3.x / 4.0-4.1: mat.blend_method = 'BLEND'  (REMOVED in 4.2)

    return mat
```

### 3.5 UV Mapping via Python

```python
# Blender 3.x / 4.x / 5.x — UV layer access
mesh = obj.data

# Access UV layers
uv_layers = mesh.uv_layers
active_uv = uv_layers.active  # Currently active UV map

# Create a new UV layer
new_uv = uv_layers.new(name="ProjectionUV")

# Read UV coordinates (per-loop, not per-vertex)
for loop_idx, loop in enumerate(mesh.loops):
    uv = active_uv.data[loop_idx].uv
    print(f"Loop {loop_idx}: UV = ({uv.x}, {uv.y})")

# Modify UV coordinates
for loop_idx in range(len(mesh.loops)):
    active_uv.data[loop_idx].uv.x *= 2.0  # Scale U
    active_uv.data[loop_idx].uv.y *= 2.0  # Scale V

# Project UVs using an operator (requires Edit Mode)
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.uv.cube_project(cube_size=1.0)
bpy.ops.object.mode_set(mode='OBJECT')
```

**CRITICAL**: UV data is stored per-loop (corner), NOT per-vertex. A vertex shared by multiple faces has separate UV coordinates for each face corner.

### 3.6 Image Textures

```python
# Blender 3.x / 4.x / 5.x — Load and assign image textures
# Load from file
img = bpy.data.images.load("/path/to/texture.png", check_existing=True)

# Create a new image (e.g., for baking)
img = bpy.data.images.new("BakeTarget", width=2048, height=2048)

# Assign to texture node
tex_node = tree.nodes.new('ShaderNodeTexImage')
tex_node.image = img
tex_node.interpolation = 'Linear'     # 'Linear', 'Closest', 'Cubic', 'Smart'
tex_node.projection = 'FLAT'          # 'FLAT', 'BOX', 'SPHERE', 'TUBE'
tex_node.extension = 'REPEAT'         # 'REPEAT', 'EXTEND', 'CLIP'

# Set color space
img.colorspace_settings.name = 'sRGB'        # For color/diffuse textures
img.colorspace_settings.name = 'Non-Color'   # For normal maps, roughness, etc.
```

### 3.7 Anti-Patterns: Materials & Shading

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| Using 3.x Principled BSDF socket names in 4.0+ | Sockets renamed | Use 4.0+ names or version-check pattern |
| `mat.blend_method = 'BLEND'` in 4.2+ | Property removed | Use `mat.surface_render_method = 'BLENDED'` |
| Assigning non-Color images without `Non-Color` space | Incorrect gamma/color interpretation | ALWAYS set `colorspace_settings.name = 'Non-Color'` for normal/roughness maps |
| Accessing UV per-vertex | UV is per-loop (per face corner) | Access via `uv_layers.data[loop_index]` |
| `bpy.data.images.load()` without `check_existing=True` | Duplicate image data blocks | Use `check_existing=True` to reuse |

### Sources
- https://docs.blender.org/api/current/bpy.types.ShaderNodeBsdfPrincipled.html
- https://docs.blender.org/api/current/bpy.types.Material.html
- https://docs.blender.org/api/current/bpy.types.UVLoopLayers.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/

---

## 4. Rendering

### 4.1 Render Settings

```python
# Blender 3.x / 4.x / 5.x — Configure render settings
scene = bpy.context.scene
render = scene.render

# Resolution
render.resolution_x = 1920
render.resolution_y = 1080
render.resolution_percentage = 100

# Output format
render.image_settings.file_format = 'PNG'  # 'PNG', 'JPEG', 'OPEN_EXR', 'TIFF', 'BMP'
render.image_settings.color_mode = 'RGBA'  # 'BW', 'RGB', 'RGBA'
render.image_settings.color_depth = '16'   # '8', '16', '32' (format-dependent)
render.image_settings.compression = 15     # PNG compression (0-100)

# Output path
render.filepath = "/tmp/render_output/frame_"

# Frame range
scene.frame_start = 1
scene.frame_end = 250
scene.frame_step = 1
render.fps = 24
```

### 4.2 Render Engine Configuration

```python
# Blender 3.x — EEVEE
scene.render.engine = 'BLENDER_EEVEE'

# Blender 4.2 - 4.4 — EEVEE Next
scene.render.engine = 'BLENDER_EEVEE_NEXT'

# Blender 5.0+ — EEVEE (identifier changed BACK)
scene.render.engine = 'BLENDER_EEVEE'

# Cycles
scene.render.engine = 'CYCLES'
```

**EEVEE engine identifier history:**

| Blender Version | Engine Identifier |
|----------------|-------------------|
| 3.x | `'BLENDER_EEVEE'` |
| 4.0 - 4.1 | `'BLENDER_EEVEE'` |
| 4.2 - 4.4 | `'BLENDER_EEVEE_NEXT'` |
| 5.0+ | `'BLENDER_EEVEE'` |

**Version-safe EEVEE setup:**

```python
# Blender 3.x / 4.x / 5.x — Version-safe EEVEE selection
if bpy.app.version >= (5, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE'
elif bpy.app.version >= (4, 2, 0):
    scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    scene.render.engine = 'BLENDER_EEVEE'
```

### 4.3 Cycles Configuration

```python
# Blender 3.x / 4.x / 5.x — Cycles-specific settings
scene.render.engine = 'CYCLES'
cycles = scene.cycles

# Device
cycles.device = 'GPU'  # 'CPU' or 'GPU'

# Sampling
cycles.samples = 256                    # Render samples
cycles.preview_samples = 64            # Viewport samples
cycles.use_adaptive_sampling = True
cycles.adaptive_threshold = 0.01

# Denoiser
cycles.use_denoising = True
cycles.denoiser = 'OPENIMAGEDENOISE'   # 'OPENIMAGEDENOISE' or 'OPTIX'

# Light paths
cycles.max_bounces = 12
cycles.diffuse_bounces = 4
cycles.glossy_bounces = 4
cycles.transmission_bounces = 12
cycles.transparent_max_bounces = 8

# GPU compute device setup
prefs = bpy.context.preferences
cycles_prefs = prefs.addons['cycles'].preferences
cycles_prefs.compute_device_type = 'CUDA'  # 'CUDA', 'OPTIX', 'HIP', 'ONEAPI', 'METAL'
cycles_prefs.get_devices()  # Refresh device list
for device in cycles_prefs.devices:
    device.use = True  # Enable all available devices
```

### 4.4 EEVEE Configuration

```python
# Blender 4.2+ — EEVEE settings
eevee = scene.eevee

# Shadows
eevee.shadow_cube_size = '1024'        # Shadow map resolution
eevee.shadow_cascade_size = '2048'

# Ray tracing (EEVEE 4.2+)
eevee.ray_tracing_method = 'SCREEN'    # 'SCREEN' or 'PROBE' (light probes)

# Ambient Occlusion (moved in 5.0)
# Blender 4.x:
# eevee.use_gtao = True                # REMOVED in 5.0
# eevee.gtao_distance = 1.0            # REMOVED in 5.0

# Blender 5.0+:
# view_layer.eevee.ambient_occlusion_distance = 1.0
```

### 4.5 Camera Setup

```python
# Blender 3.x / 4.x / 5.x — Camera configuration
cam_data = bpy.data.cameras.new("AEC_Camera")
cam_obj = bpy.data.objects.new("AEC_Camera", cam_data)
bpy.context.collection.objects.link(cam_obj)

# Set as active camera
scene.camera = cam_obj

# Lens settings
cam_data.type = 'PERSP'           # 'PERSP', 'ORTHO', 'PANO'
cam_data.lens = 35.0              # Focal length in mm (perspective)
cam_data.ortho_scale = 10.0       # Orthographic scale

# Clipping
cam_data.clip_start = 0.1
cam_data.clip_end = 1000.0

# Depth of Field
cam_data.dof.use_dof = True
cam_data.dof.focus_object = bpy.data.objects["FocusTarget"]
cam_data.dof.aperture_fstop = 2.8

# Sensor settings
cam_data.sensor_width = 36.0     # mm (default: 36)
cam_data.sensor_height = 24.0    # mm

# Camera position and orientation
cam_obj.location = (10, -10, 8)
cam_obj.rotation_euler = (1.1, 0.0, 0.785)
```

### 4.6 Lights

```python
# Blender 3.x / 4.x / 5.x — Light creation
# Point light
point_data = bpy.data.lights.new("PointLight", type='POINT')
point_data.energy = 1000.0      # Watts
point_data.shadow_soft_size = 0.25
point_data.color = (1.0, 0.95, 0.9)  # Warm white

point_obj = bpy.data.objects.new("PointLight", point_data)
bpy.context.collection.objects.link(point_obj)

# Sun light (for architectural exterior)
sun_data = bpy.data.lights.new("SunLight", type='SUN')
sun_data.energy = 5.0
sun_data.angle = 0.00918  # Angular diameter (radians)

# Area light (for interior visualization)
area_data = bpy.data.lights.new("AreaLight", type='AREA')
area_data.shape = 'RECTANGLE'   # 'SQUARE', 'RECTANGLE', 'DISK', 'ELLIPSE'
area_data.size = 2.0
area_data.size_y = 1.0          # Only for RECTANGLE/ELLIPSE
area_data.energy = 500.0

# Spot light
spot_data = bpy.data.lights.new("SpotLight", type='SPOT')
spot_data.spot_size = 0.785     # Cone angle in radians (45 degrees)
spot_data.spot_blend = 0.15     # Soft edge (0 = hard, 1 = fully soft)
```

**Light types:**

| Type | String | Description |
|------|--------|-------------|
| Point | `'POINT'` | Omnidirectional light from a point |
| Sun | `'SUN'` | Directional light (position irrelevant, only rotation matters) |
| Spot | `'SPOT'` | Cone-shaped directional light |
| Area | `'AREA'` | Surface-emitting light (soft shadows) |

### 4.7 Batch Rendering

```python
# Blender 3.x / 4.x / 5.x — Batch render multiple cameras
import bpy

scene = bpy.context.scene
cameras = [obj for obj in bpy.data.objects if obj.type == 'CAMERA']

for cam in cameras:
    scene.camera = cam
    scene.render.filepath = f"/tmp/renders/{cam.name}_"
    bpy.ops.render.render(write_still=True)

# Render animation
scene.camera = bpy.data.objects["MainCamera"]
scene.render.filepath = "/tmp/renders/anim_"
bpy.ops.render.render(animation=True)

# Headless rendering (command line)
# blender --background scene.blend --python render_script.py
# In render_script.py:
# bpy.ops.render.render(write_still=True)
```

**`bpy.ops.render.render()` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `animation` | `bool` | `False` | Render entire frame range |
| `write_still` | `bool` | `False` | Save rendered image to output path |
| `use_viewport` | `bool` | `False` | Use viewport render settings |
| `scene` | `str` | `""` | Scene to render (empty = active scene) |

### 4.8 Render Pass Names (5.0 Changes)

```python
# Blender 5.0+ — Render pass names changed
# OLD (3.x / 4.x):     NEW (5.0+):
# "DiffCol"          -> "Diffuse Color"
# "GlossCol"         -> "Glossy Color"
# "TransCol"         -> "Transmission Color"
# "IndexMA"          -> "Material Index"
# "IndexOB"          -> "Object Index"
# "Z"                -> "Depth"
```

### 4.9 Anti-Patterns: Rendering

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| `'BLENDER_EEVEE'` in Blender 4.2-4.4 | Engine was temporarily `'BLENDER_EEVEE_NEXT'` | Use version check for engine identifier |
| `render.render()` without `write_still=True` | Image is rendered but never saved | ALWAYS pass `write_still=True` for still images |
| Setting Cycles GPU without `get_devices()` | Device list is empty until refreshed | ALWAYS call `cycles_prefs.get_devices()` first |
| Using old render pass names in 5.0+ | Pass names were renamed to full words | Use new pass names (e.g., `"Diffuse Color"` not `"DiffCol"`) |
| `scene.use_nodes = True` in 5.0+ | Deprecated, always True | Remove the line; compositor is always active |
| `scene.node_tree` in 5.0+ | Removed | Use `scene.compositing_node_group` |
| `eevee.use_gtao` in 5.0+ | Removed | Use `view_layer.eevee.ambient_occlusion_distance` |

### Sources
- https://docs.blender.org/api/current/bpy.types.RenderSettings.html
- https://docs.blender.org/api/current/bpy.types.SceneEEVEE.html
- https://docs.blender.org/api/current/bpy.ops.render.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/

---

## 5. I/O Formats

### 5.1 I/O Operator Overview

| Format | Export Operator | Import Operator | Since |
|--------|----------------|-----------------|-------|
| FBX | `bpy.ops.export_scene.fbx()` | `bpy.ops.import_scene.fbx()` | 2.5+ |
| glTF/GLB | `bpy.ops.export_scene.gltf()` | `bpy.ops.import_scene.gltf()` | 2.8+ |
| OBJ | `bpy.ops.wm.obj_export()` | `bpy.ops.wm.obj_import()` | 4.0+ (C++) |
| USD | `bpy.ops.wm.usd_export()` | `bpy.ops.wm.usd_import()` | 2.9+ |
| Alembic | `bpy.ops.wm.alembic_export()` | `bpy.ops.wm.alembic_import()` | 2.78+ |
| STL | `bpy.ops.wm.stl_export()` | `bpy.ops.wm.stl_import()` | 4.0+ (C++) |

### 5.2 OBJ/PLY/STL Migration (4.0 Breaking Change)

```python
# Blender 3.x (LEGACY — BROKEN in 4.0+)
bpy.ops.export_scene.obj(filepath="/tmp/model.obj")   # REMOVED
bpy.ops.import_scene.obj(filepath="/tmp/model.obj")    # REMOVED
bpy.ops.export_mesh.stl(filepath="/tmp/model.stl")     # REMOVED
bpy.ops.import_mesh.stl(filepath="/tmp/model.stl")     # REMOVED

# Blender 4.0+ — C++ replacements
bpy.ops.wm.obj_export(filepath="/tmp/model.obj")
bpy.ops.wm.obj_import(filepath="/tmp/model.obj")
bpy.ops.wm.stl_export(filepath="/tmp/model.stl")
bpy.ops.wm.stl_import(filepath="/tmp/model.stl")
```

### 5.3 FBX Export/Import

```python
# Blender 3.x / 4.x / 5.x — FBX export (key parameters for AEC)
bpy.ops.export_scene.fbx(
    filepath="/tmp/building.fbx",
    use_selection=True,           # Export only selected objects
    use_active_collection=False,  # Export active collection only
    global_scale=1.0,
    apply_scale_options='FBX_SCALE_ALL',  # 'FBX_SCALE_NONE', 'FBX_SCALE_UNITS', 'FBX_SCALE_CUSTOM', 'FBX_SCALE_ALL'
    axis_forward='-Z',
    axis_up='Y',
    object_types={'MESH', 'ARMATURE', 'EMPTY', 'CAMERA', 'LIGHT'},
    use_mesh_modifiers=True,      # Apply modifiers
    mesh_smooth_type='EDGE',      # 'OFF', 'FACE', 'EDGE'
    use_triangles=False,          # Triangulate meshes
    use_custom_props=True,        # Export custom properties
    bake_anim=True,               # Include animation
    bake_anim_use_all_actions=True,
    embed_textures=False,         # Embed textures in FBX
    path_mode='AUTO',             # 'AUTO', 'ABSOLUTE', 'RELATIVE', 'MATCH', 'STRIP', 'COPY'
)

# FBX import
bpy.ops.import_scene.fbx(
    filepath="/tmp/building.fbx",
    global_scale=1.0,
    use_manual_orientation=False,
    axis_forward='-Z',
    axis_up='Y',
    use_custom_props=True,
    use_anim=True,
    ignore_leaf_bones=False,
    force_connect_children=False,
)
```

### 5.4 glTF/GLB Export/Import

```python
# Blender 3.x / 4.x / 5.x — glTF export (key parameters for AEC)
bpy.ops.export_scene.gltf(
    filepath="/tmp/building.glb",
    export_format='GLB',           # 'GLB' (binary), 'GLTF_SEPARATE', 'GLTF_EMBEDDED'
    use_selection=True,
    export_apply=True,             # Apply modifiers
    export_texcoords=True,         # UV coordinates
    export_normals=True,
    export_colors=True,            # Vertex colors
    export_cameras=True,
    export_lights=True,
    export_materials='EXPORT',     # 'EXPORT', 'PLACEHOLDER', 'NONE'
    export_image_format='AUTO',    # 'AUTO', 'JPEG', 'NONE'
    export_draco_mesh_compression_enable=False,  # Draco compression
)

# glTF import
bpy.ops.import_scene.gltf(
    filepath="/tmp/building.glb",
    merge_vertices=False,
    import_shading='NORMALS',      # 'NORMALS', 'FLAT', 'SMOOTH'
)
```

### 5.5 USD Export/Import

```python
# Blender 3.x / 4.x / 5.x — USD export
bpy.ops.wm.usd_export(
    filepath="/tmp/building.usdc",
    selected_objects_only=True,
    visible_objects_only=True,
    export_animation=True,
    export_hair=False,
    export_uvmaps=True,
    export_normals=True,
    export_materials=True,
    use_instancing=True,
    evaluation_mode='RENDER',      # 'RENDER' or 'VIEWPORT'
)

# USD import
bpy.ops.wm.usd_import(
    filepath="/tmp/building.usdc",
    scale=1.0,
    set_frame_range=True,
    import_cameras=True,
    import_curves=True,
    import_lights=True,
    import_materials=True,
    import_meshes=True,
    import_volumes=True,
    read_mesh_uvs=True,
    read_mesh_colors=True,
)
```

### 5.6 Alembic Export/Import

```python
# Blender 3.x / 4.x / 5.x — Alembic export
bpy.ops.wm.alembic_export(
    filepath="/tmp/building.abc",
    start=1,
    end=250,
    selected=True,
    visible_objects_only=True,
    flatten=False,                 # Flatten hierarchy
    uvs=True,
    normals=True,
    vcolors=True,
    face_sets=True,
    subdiv_schema=False,
    apply_subdiv=False,
    curves_as_mesh=False,
    use_instancing=True,
    global_scale=1.0,
    export_custom_properties=True,
    as_background_job=False,
)

# Alembic import
bpy.ops.wm.alembic_import(
    filepath="/tmp/building.abc",
    scale=1.0,
    set_frame_range=True,
    validate_meshes=False,
    always_add_cache_reader=False,
    is_sequence=False,
)
```

### 5.7 Anti-Patterns: I/O Formats

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| `bpy.ops.export_scene.obj()` in 4.0+ | Python OBJ exporter removed | Use `bpy.ops.wm.obj_export()` |
| `bpy.ops.export_mesh.stl()` in 4.0+ | Python STL exporter removed | Use `bpy.ops.wm.stl_export()` |
| `bpy.ops.import_scene.obj()` in 4.0+ | Python OBJ importer removed | Use `bpy.ops.wm.obj_import()` |
| FBX export without `apply_scale_options` | Scale mismatch between applications | ALWAYS set `apply_scale_options` explicitly |
| glTF export with `GLTF_EMBEDDED` | Large files, poor compression | Prefer `'GLB'` for single-file binary |
| Alembic without `export_custom_properties` | Custom metadata lost | Set `export_custom_properties=True` for AEC data |
| Not setting axis conventions | Objects rotated 90 degrees on import | ALWAYS verify `axis_forward` and `axis_up` match target application |

### Sources
- https://docs.blender.org/api/current/bpy.ops.export_scene.html
- https://docs.blender.org/api/current/bpy.ops.import_scene.html
- https://docs.blender.org/api/current/bpy.ops.wm.html

---

## 6. Collections, Libraries & Assets

### 6.1 Collection Management

Collections in Blender organize objects in a tree hierarchy. Every scene has a root `scene.collection` (the "Scene Collection") which CANNOT be renamed or deleted.

```python
# Blender 3.x / 4.x / 5.x — Collection operations
import bpy

# Create a collection
building_col = bpy.data.collections.new("Building_A")
bpy.context.scene.collection.children.link(building_col)  # Add to scene

# Create nested collections
floors_col = bpy.data.collections.new("Floors")
building_col.children.link(floors_col)

walls_col = bpy.data.collections.new("Walls")
building_col.children.link(walls_col)

# Add object to collection
obj = bpy.data.objects["Wall_01"]
walls_col.objects.link(obj)

# Remove object from another collection (objects can be in multiple collections)
bpy.context.scene.collection.objects.unlink(obj)

# Toggle collection visibility in viewport
view_layer = bpy.context.view_layer
layer_col = view_layer.layer_collection.children["Building_A"]
layer_col.exclude = True  # Exclude from view layer entirely
layer_col.hide_viewport = True  # Hide in viewport but keep in renders

# Collection visibility per render
building_col.hide_render = True   # Hide in renders
building_col.hide_viewport = True  # Hide in viewport
```

**Collection hierarchy access:**

```python
# Blender 3.x / 4.x / 5.x — Traverse collection hierarchy
def print_collection_tree(collection, indent=0):
    print(" " * indent + collection.name)
    for child in collection.children:
        print_collection_tree(child, indent + 2)
    for obj in collection.objects:
        print(" " * (indent + 2) + f"[Object] {obj.name}")

print_collection_tree(bpy.context.scene.collection)
```

### 6.2 Library Linking

Linking creates a reference to data in another .blend file. Linked data is read-only.

```python
# Blender 3.x / 4.x / 5.x — Link data from external .blend file
filepath = "/path/to/library.blend"

# Method 1: Using bpy.data.libraries.load() (RECOMMENDED)
with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
    # List available collections
    print("Available collections:", data_src.collections)

    # Link specific collections
    data_dst.collections = ["BuildingModule_A", "BuildingModule_B"]

# The linked collections are now in bpy.data.collections
# Link them to the scene
for col in data_dst.collections:
    if col is not None:
        bpy.context.scene.collection.children.link(col)

# Method 2: Using operator
bpy.ops.wm.link(
    filepath=filepath + "/Collection/BuildingModule_A",
    directory=filepath + "/Collection/",
    filename="BuildingModule_A"
)
```

### 6.3 Library Appending

Appending copies data from another .blend file. Appended data is fully local and editable.

```python
# Blender 3.x / 4.x / 5.x — Append data from external .blend file
filepath = "/path/to/library.blend"

with bpy.data.libraries.load(filepath, link=False) as (data_src, data_dst):
    # Append specific objects
    data_dst.objects = ["Column_Type_A"]
    # Append specific materials
    data_dst.materials = ["Concrete_Exposed"]

# Link appended objects to scene
for obj in data_dst.objects:
    if obj is not None:
        bpy.context.collection.objects.link(obj)
```

### 6.4 Library Overrides (4.0+)

Library overrides replaced proxies for creating local editable versions of linked data. Proxies were fully removed in Blender 4.0.

```python
# Blender 4.0+ — Create library overrides
# First, link an object
with bpy.data.libraries.load("/path/to/library.blend", link=True) as (data_src, data_dst):
    data_dst.collections = ["LinkedBuilding"]

for col in data_dst.collections:
    if col is not None:
        bpy.context.scene.collection.children.link(col)

# Select linked object and create override
linked_obj = bpy.data.objects["LinkedBuilding"]
bpy.context.view_layer.objects.active = linked_obj
linked_obj.select_set(True)

# Create library override (makes local editable copy that tracks source)
bpy.ops.object.make_override_library()

# The overridden object can now be modified locally
# Changes are stored as a diff against the linked source
# When the source is updated, non-overridden properties sync automatically
```

**Override properties access:**

```python
# Blender 4.0+ — Check override properties
obj = bpy.data.objects["LinkedBuilding"]
if obj.override_library:
    override = obj.override_library
    print(f"Reference: {override.reference.name}")
    for prop in override.properties:
        print(f"Overridden: {prop.rna_path}")
```

### 6.5 Asset Browser API

```python
# Blender 3.x / 4.x / 5.x — Mark and manage assets
# Mark a data block as an asset
obj = bpy.data.objects["Column_Type_A"]
obj.asset_mark()

# Access asset metadata
metadata = obj.asset_data  # AssetMetaData
metadata.description = "Reinforced concrete column, 300x300mm, for parking structures"
metadata.author = "AEC Library"

# Add tags
metadata.tags.new("concrete", skip_if_exists=True)
metadata.tags.new("structural", skip_if_exists=True)
metadata.tags.new("column", skip_if_exists=True)

# Set catalog (requires UUID string)
metadata.catalog_id = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"

# Generate preview
bpy.ops.ed.lib_id_generate_preview(asset_id=obj.name)

# Clear asset status
obj.asset_clear()
```

```python
# Blender 3.x (LEGACY — BROKEN in 5.0)
# context.asset_file_handle  # REMOVED in 5.0
# bpy.types.AssetHandle      # REMOVED in 5.0

# Blender 4.0+
# context.asset              # AssetRepresentation (4.0+)

# Blender 5.0+
# context.active_file        # REMOVED from asset shelf
# Use context.asset instead
```

### 6.6 Asset Catalogs via Operators

```python
# Blender 4.x / 5.x — Catalog management (operator-based only)
# NOTE: Catalog management is ONLY available through operators.
# There is NO direct data API for catalogs.

# Create a new catalog
bpy.ops.asset.catalog_new()

# Delete a catalog
bpy.ops.asset.catalog_delete()

# Save catalog changes
bpy.ops.asset.catalogs_save()
```

**CRITICAL**: The asset catalog API is limited to operators. There is no direct data-level API to create catalogs with specific names or UUIDs programmatically. For automation, directly edit the `blender_assets.cats.txt` file in the asset library directory.

### 6.7 Anti-Patterns: Collections, Libraries & Assets

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| `bpy.context.scene.collection.name = "X"` | Scene collection CANNOT be renamed | Create child collections for organization |
| Linking without `link=True` in `libraries.load()` | Default is `link=False` (append) | Explicitly set `link=True` for linking |
| Using proxies in 4.0+ | Proxies completely removed | Use `bpy.ops.object.make_override_library()` |
| `context.asset_file_handle` in 5.0+ | Removed | Use `context.asset` (AssetRepresentation) |
| Automating catalogs via `bpy.ops.asset.catalog_new()` | No way to set catalog name/UUID via operator | Edit `blender_assets.cats.txt` directly |
| Forgetting to `unlink()` from old collection | Object appears in multiple collections | Unlink from source collection after linking to target |

### Sources
- https://docs.blender.org/api/current/bpy.types.Collection.html
- https://docs.blender.org/api/current/bpy.types.BlendDataLibraries.html
- https://docs.blender.org/api/current/bpy.ops.asset.html
- https://docs.blender.org/api/current/bpy.types.AssetMetaData.html
- https://docs.blender.org/api/current/bpy.types.IDOverrideLibrary.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/

---

## 7. mathutils & Standalone Modules

### 7.1 Vector

The `mathutils.Vector` class represents 2D, 3D, or 4D vectors. It is the most commonly used math type in Blender scripting.

```python
# Blender 3.x / 4.x / 5.x — Vector operations
from mathutils import Vector

# Creation
v1 = Vector((1.0, 2.0, 3.0))       # 3D
v2 = Vector((4.0, 5.0, 6.0))
v_2d = Vector((1.0, 2.0))           # 2D
v_4d = Vector((1.0, 2.0, 3.0, 1.0)) # 4D (homogeneous)

# Arithmetic
v_sum = v1 + v2                      # (5.0, 7.0, 9.0)
v_diff = v1 - v2                     # (-3.0, -3.0, -3.0)
v_scaled = v1 * 2.0                  # (2.0, 4.0, 6.0)
v_neg = -v1                          # (-1.0, -2.0, -3.0)

# Length and normalization
length = v1.length                   # Euclidean length
length_sq = v1.length_squared        # Faster than length (no sqrt)
v_norm = v1.normalized()             # Returns NEW normalized vector
v1.normalize()                       # Normalizes IN PLACE (modifies v1)

# Dot product and cross product
dot = v1.dot(v2)                     # Scalar dot product
cross = v1.cross(v2)                 # 3D cross product (returns Vector)

# Angle between vectors
angle = v1.angle(v2)                 # Angle in radians
angle_signed = v1.angle_signed(v2, Vector((0, 0, 1)))  # Signed angle around axis

# Projection and reflection
proj = v1.project(v2)               # Project v1 onto v2
refl = v1.reflect(Vector((0, 0, 1))) # Reflect v1 across plane with given normal

# Linear interpolation
v_lerp = v1.lerp(v2, 0.5)           # Halfway between v1 and v2

# Component access
x, y, z = v1.x, v1.y, v1.z
v1.xy                                # Swizzle: returns 2D Vector
v1.xzy                               # Swizzle: rearranged components
```

### 7.2 Matrix

```python
# Blender 3.x / 4.x / 5.x — Matrix operations
from mathutils import Matrix, Vector
import math

# Identity matrix
identity = Matrix.Identity(4)        # 4x4 identity

# Translation matrix
trans = Matrix.Translation(Vector((5.0, 0.0, 3.0)))

# Rotation matrices
rot_x = Matrix.Rotation(math.radians(90), 4, 'X')
rot_y = Matrix.Rotation(math.radians(45), 4, 'Y')
rot_z = Matrix.Rotation(math.radians(30), 4, 'Z')
rot_axis = Matrix.Rotation(math.radians(60), 4, Vector((1, 1, 0)).normalized())

# Scale matrix
scale = Matrix.Diagonal(Vector((2.0, 2.0, 2.0, 1.0)))  # Uniform scale 2x
# Or for non-uniform scale:
scale_nu = Matrix.Diagonal(Vector((1.0, 1.0, 3.0, 1.0)))  # Scale Z by 3

# Matrix multiplication (order matters: applied right-to-left)
transform = trans @ rot_z @ scale    # Scale -> Rotate -> Translate

# Apply matrix to a point
point = Vector((1.0, 0.0, 0.0))
transformed = transform @ point      # Apply full transformation

# Decompose matrix into components
loc, rot, sca = transform.decompose()
# loc: Vector (translation)
# rot: Quaternion (rotation)
# sca: Vector (scale)

# Inverse
inv = transform.inverted()
inv_safe = transform.inverted_safe()  # Returns identity if singular

# Transpose
transposed = transform.transposed()

# Apply to object
obj.matrix_world = transform
```

**CRITICAL**: Matrix multiplication in Blender uses the `@` operator (Python's matrix multiplication operator), NOT `*`. The `*` operator performs element-wise multiplication.

### 7.3 Quaternion

```python
# Blender 3.x / 4.x / 5.x — Quaternion operations
from mathutils import Quaternion, Vector, Euler
import math

# Creation
q_identity = Quaternion()                              # (1, 0, 0, 0) identity
q_axis = Quaternion(Vector((0, 0, 1)), math.radians(90))  # 90 degrees around Z
q_wxyz = Quaternion((1.0, 0.0, 0.0, 0.0))            # Direct w, x, y, z

# Interpolation (SLERP)
q1 = Quaternion(Vector((0, 0, 1)), math.radians(0))
q2 = Quaternion(Vector((0, 0, 1)), math.radians(90))
q_mid = q1.slerp(q2, 0.5)           # Spherical linear interpolation at 50%

# Conversions
euler = q_axis.to_euler()            # Returns Euler
euler_order = q_axis.to_euler('YXZ') # With rotation order
matrix = q_axis.to_matrix()          # Returns 3x3 rotation Matrix
axis, angle = q_axis.to_axis_angle() # Returns (Vector, float)

# Operations
q_conj = q_axis.conjugated()         # Conjugate (inverse for unit quaternions)
q_inv = q_axis.inverted()            # True inverse
q_norm = q_axis.normalized()         # Normalize to unit quaternion
q_combined = q1 @ q2                 # Combine rotations (order matters)

# Rotate a vector
v = Vector((1, 0, 0))
v_rotated = q_axis @ v               # Apply rotation to vector
```

### 7.4 Euler

```python
# Blender 3.x / 4.x / 5.x — Euler angle operations
from mathutils import Euler, Quaternion, Matrix
import math

# Creation
e = Euler((math.radians(90), 0, 0), 'XYZ')  # Rotation order matters

# Available rotation orders: 'XYZ', 'XZY', 'YXZ', 'YZX', 'ZXY', 'ZYX'

# Conversions
quat = e.to_quaternion()
mat = e.to_matrix()                   # Returns 3x3 rotation matrix

# Rotate
e.rotate(Euler((0, math.radians(45), 0)))  # Combine rotations
e.rotate_axis('Z', math.radians(30))       # Rotate around single axis

# Make compatible (avoid gimbal jumps in animation)
e_prev = Euler((math.radians(170), 0, 0))
e_curr = Euler((math.radians(-170), 0, 0))
e_curr.make_compatible(e_prev)  # Adjusts to closest equivalent representation
```

### 7.5 KDTree (Spatial Nearest-Neighbor Queries)

```python
# Blender 3.x / 4.x / 5.x — KDTree for spatial queries
from mathutils.kdtree import KDTree

# Create from mesh vertices
mesh = bpy.context.active_object.data
size = len(mesh.vertices)
kd = KDTree(size)

for i, vert in enumerate(mesh.vertices):
    kd.insert(vert.co, i)    # (coordinate, index)

kd.balance()  # MUST call after all inserts

# Find nearest vertex to a point
query_point = (5.0, 3.0, 0.0)
co, index, dist = kd.find(query_point)
print(f"Nearest vertex: index={index}, distance={dist}")

# Find N nearest vertices
results = kd.find_n(query_point, 10)  # Returns list of (co, index, dist)
for co, index, dist in results:
    print(f"  Vertex {index}: distance={dist}")

# Find all vertices within radius
results = kd.find_range(query_point, 2.0)  # 2.0 meter radius
for co, index, dist in results:
    print(f"  Vertex {index}: distance={dist}")
```

**AEC use cases for KDTree:**
- Finding closest structural elements to a point
- Detecting nearby objects for clash detection
- Spatial queries for element grouping by location

### 7.6 BVHTree (Ray Casting and Nearest Surface)

```python
# Blender 3.x / 4.x / 5.x — BVHTree for ray casting and proximity queries
from mathutils.bvhtree import BVHTree
from mathutils import Vector

# Create from Object (in world space, respects modifiers)
depsgraph = bpy.context.evaluated_depsgraph_get()
obj = bpy.context.active_object
obj_eval = obj.evaluated_get(depsgraph)
bvh = BVHTree.FromObject(obj_eval, depsgraph)

# Create from BMesh
import bmesh
bm = bmesh.new()
bm.from_mesh(obj.data)
bvh = BVHTree.FromBMesh(bm)

# Create from polygon data directly
vertices = [(0,0,0), (1,0,0), (1,1,0), (0,1,0)]
polygons = [(0,1,2,3)]
bvh = BVHTree.FromPolygons(vertices, polygons)

# Ray casting
origin = Vector((0.0, 0.0, 10.0))
direction = Vector((0.0, 0.0, -1.0))
location, normal, index, distance = bvh.ray_cast(origin, direction)
if location:
    print(f"Hit face {index} at {location}, normal: {normal}, dist: {distance}")

# Find nearest point on surface
query = Vector((5.0, 3.0, 0.5))
location, normal, index, distance = bvh.find_nearest(query)
if location:
    print(f"Nearest face {index} at {location}, distance: {distance}")

# Overlap detection between two BVH trees
bvh1 = BVHTree.FromObject(obj1_eval, depsgraph)
bvh2 = BVHTree.FromObject(obj2_eval, depsgraph)
overlapping = bvh1.overlap(bvh2)
# Returns list of (index1, index2) pairs of overlapping polygons
for idx1, idx2 in overlapping:
    print(f"Object 1 face {idx1} overlaps Object 2 face {idx2}")
```

**AEC use cases for BVHTree:**
- Ray casting for sun/shadow analysis
- Clash detection between building elements (overlap)
- Finding closest surface point for snapping/alignment
- Line-of-sight analysis

### 7.7 bl_math Module

```python
# Blender 3.x / 4.x / 5.x — bl_math utility functions
import bl_math

# Linear interpolation
value = bl_math.lerp(0.0, 10.0, 0.25)    # Returns 2.5

# Clamp
clamped = bl_math.clamp(15.0, 0.0, 10.0)  # Returns 10.0
clamped = bl_math.clamp(-5.0)              # Clamp to [0.0, 1.0] -> Returns 0.0

# Smooth step (cubic Hermite interpolation)
smooth = bl_math.smoothstep(0.0, 1.0, 0.5)  # Returns smooth transition value
# Returns 0.0 when value <= from_value, 1.0 when value >= to_value
# Smooth S-curve in between (no abrupt changes)
```

**IMPORTANT**: `bl_math` functions are also available in driver expressions. They operate on floats only, not vectors.

### 7.8 gpu Module (Replaces bgl)

The `gpu` module is the modern graphics API for custom viewport drawing. It replaced the deprecated `bgl` module, which was fully removed in Blender 5.0.

```python
# Blender 4.x / 5.x — GPU drawing (replaces bgl, REQUIRED for 5.0+)
import gpu
from gpu_extras.batch import batch_for_shader
import bpy

# Create a simple line drawing
def draw_callback():
    coords = [
        (0.0, 0.0, 0.0),
        (10.0, 0.0, 0.0),
        (10.0, 10.0, 0.0),
        (0.0, 10.0, 0.0),
        (0.0, 0.0, 0.0),  # Close the loop
    ]

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINE_STRIP', {"pos": coords})

    shader.uniform_float("color", (1.0, 0.0, 0.0, 1.0))  # Red
    shader.uniform_float("lineWidth", 2.0)

    region = bpy.context.region
    rv3d = bpy.context.region_data
    shader.uniform_float("viewportSize", (region.width, region.height))

    batch.draw(shader)

# Register draw handler
handler = bpy.types.SpaceView3D.draw_handler_add(
    draw_callback,
    (),
    'WINDOW',
    'POST_VIEW'    # 'PRE_VIEW', 'POST_VIEW', 'POST_PIXEL'
)

# Unregister draw handler (MUST be done in addon unregister)
bpy.types.SpaceView3D.draw_handler_remove(handler, 'WINDOW')
```

**Built-in shaders:**

| Shader Name | Use Case |
|------------|----------|
| `'UNIFORM_COLOR'` | Flat colored triangles/lines |
| `'POLYLINE_UNIFORM_COLOR'` | Lines with width control |
| `'SMOOTH_COLOR'` | Per-vertex colored geometry |
| `'POLYLINE_SMOOTH_COLOR'` | Per-vertex colored lines |
| `'IMAGE'` | Textured quads |

**Offscreen rendering:**

```python
# Blender 4.x / 5.x — Offscreen rendering with gpu module
import gpu

offscreen = gpu.types.GPUOffScreen(512, 512)
with offscreen.bind():
    # Draw into the offscreen buffer
    fb = gpu.state.active_framebuffer_get()
    fb.clear(color=(0.0, 0.0, 0.0, 0.0))
    # ... draw operations ...

# Access result as texture
texture = offscreen.texture_color

# Cleanup
offscreen.free()
```

**Migration from bgl to gpu:**

```python
# Blender 3.x (LEGACY — bgl, REMOVED in 5.0)
# import bgl
# bgl.glEnable(bgl.GL_BLEND)
# bgl.glLineWidth(2)

# Blender 4.x / 5.x (CURRENT — gpu)
gpu.state.blend_set('ALPHA')
# Line width is controlled via POLYLINE shaders, not global state

# Blender 3.x (LEGACY — bindcode for textures)
# img.gl_load()
# bindcode = img.bindcode

# Blender 5.0+ (CURRENT)
texture = gpu.texture.from_image(img)
```

### 7.9 Anti-Patterns: mathutils & Standalone Modules

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| `matrix1 * matrix2` | Element-wise multiplication, NOT matrix product | Use `matrix1 @ matrix2` (@ operator) |
| Modifying a `normalized()` result | `normalized()` returns a NEW vector | Call `v.normalize()` for in-place, or store result |
| Forgetting `kd.balance()` | KDTree queries return wrong results | ALWAYS call `balance()` after all inserts |
| `BVHTree.FromObject()` without depsgraph | Missing evaluated mesh/modifiers | ALWAYS pass `depsgraph` parameter |
| Using `bgl` in 5.0+ | Module fully removed | Use `gpu` module and `gpu_extras.batch` |
| `img.bindcode` in 5.0+ | Removed | Use `gpu.texture.from_image(img)` |
| `v.length` in tight loops | Involves square root | Use `v.length_squared` for comparisons |

### Sources
- https://docs.blender.org/api/current/mathutils.html
- https://docs.blender.org/api/current/mathutils.kdtree.html
- https://docs.blender.org/api/current/mathutils.bvhtree.html
- https://docs.blender.org/api/current/bl_math.html
- https://docs.blender.org/api/current/gpu.html
- https://docs.blender.org/api/current/gpu.shader.html
- https://docs.blender.org/api/current/gpu.types.html

---

## 8. Python Runtime Quirks

### 8.1 Embedded CPython and Threading

Blender embeds a standard CPython interpreter. The Python GIL (Global Interpreter Lock) exists, but more importantly, the Blender C/C++ codebase is NOT thread-safe. ALL `bpy` API calls MUST happen on the main thread.

```python
# ANTI-PATTERN — NEVER do this (CRASHES Blender)
import threading
def bad_worker():
    bpy.data.objects["Cube"].location.x = 5.0  # CRASH: not on main thread

thread = threading.Thread(target=bad_worker)
thread.start()
```

```python
# Blender 3.x / 4.x / 5.x — Correct thread-safe pattern with queue
import threading
import queue
import bpy

execution_queue = queue.Queue()

def run_in_main_thread(function):
    """Schedule a function to run on the main thread."""
    execution_queue.put(function)

def execute_queued_functions():
    """Timer callback that processes the queue on the main thread."""
    while not execution_queue.empty():
        function = execution_queue.get()
        function()
    return 1.0  # Reschedule after 1 second

# Register the timer
bpy.app.timers.register(execute_queued_functions)

# Now worker threads can schedule main-thread work safely
def worker():
    import time
    time.sleep(2)  # Simulate work
    run_in_main_thread(lambda: setattr(bpy.data.objects["Cube"].location, 'x', 5.0))

thread = threading.Thread(target=worker)
thread.start()
```

### 8.2 Undo Invalidates ALL Python References

This is the single most dangerous runtime behavior. When the user (or script) performs an undo, ALL Python references to Blender data become invalid. Accessing them causes a crash or `ReferenceError`.

```python
# ANTI-PATTERN — Storing references across undo boundaries
obj = bpy.data.objects["Wall"]  # Get reference
bpy.ops.ed.undo()                # UNDO happens
print(obj.name)                   # CRASH or ReferenceError — reference is INVALID

# CORRECT — Re-fetch after any operation that could trigger undo
obj_name = "Wall"  # Store NAME, not reference
bpy.ops.ed.undo()
obj = bpy.data.objects.get(obj_name)  # Re-fetch by name
if obj is not None:
    print(obj.name)
```

**When references are invalidated:**
- Any `bpy.ops.ed.undo()` / `bpy.ops.ed.redo()` call
- File load (`bpy.ops.wm.open_mainfile()`)
- File revert (`bpy.ops.wm.revert_mainfile()`)
- Any operator with undo support that the user undoes interactively

**Safe pattern:**

```python
# Blender 3.x / 4.x / 5.x — Safe reference handling
# ALWAYS store NAMES or identifiers, NEVER store direct bpy references long-term

class SafeObjectRef:
    """Wrapper that stores object name and re-fetches on access."""
    def __init__(self, obj):
        self._name = obj.name

    @property
    def obj(self):
        result = bpy.data.objects.get(self._name)
        if result is None:
            raise RuntimeError(f"Object '{self._name}' no longer exists")
        return result
```

### 8.3 Restricted Context

Many operators require specific context (active object, mode, area type). Calling them in wrong context raises `RuntimeError` with "Operator bpy.ops.xxx.poll() failed".

```python
# ANTI-PATTERN — Calling mesh operator without mesh context
bpy.ops.mesh.extrude_region_move()  # RuntimeError if not in Edit Mode with a mesh

# CORRECT — Ensure correct context
obj = bpy.context.active_object
if obj and obj.type == 'MESH' and obj.mode == 'EDIT':
    bpy.ops.mesh.extrude_region_move()
```

**Context requirements by operator category:**

| Category | Required Mode | Required Object Type | Other Requirements |
|----------|--------------|---------------------|--------------------|
| `bpy.ops.mesh.*` | Edit Mode | MESH | Active mesh object |
| `bpy.ops.object.*` | Object Mode | Any | Active object (most) |
| `bpy.ops.armature.*` | Edit Mode | ARMATURE | Active armature |
| `bpy.ops.pose.*` | Pose Mode | ARMATURE | Active armature |
| `bpy.ops.sculpt.*` | Sculpt Mode | MESH | Active mesh |
| `bpy.ops.node.*` | Any | N/A | Active node editor area |
| `bpy.ops.uv.*` | Edit Mode | MESH | UV editor visible |
| `bpy.ops.render.render()` | Any | N/A | Camera set in scene |

### 8.4 bpy.app.timers

Timers allow deferred execution of functions. They run on the main thread during Blender's event loop.

```python
# Blender 3.x / 4.x / 5.x — Timer registration
import bpy

# One-shot timer (runs once after delay)
def delayed_action():
    print("Executed after 2 seconds")
    return None  # Return None to unregister

bpy.app.timers.register(delayed_action, first_interval=2.0)

# Repeating timer
def periodic_check():
    print(f"Checking at frame {bpy.context.scene.frame_current}")
    return 0.5  # Return float to reschedule after 0.5 seconds

bpy.app.timers.register(periodic_check, first_interval=0.0)

# Unregister a timer
if bpy.app.timers.is_registered(periodic_check):
    bpy.app.timers.unregister(periodic_check)

# Persistent timer (survives file load)
bpy.app.timers.register(periodic_check, first_interval=0.0, persistent=True)
```

**Timer return values:**

| Return Value | Behavior |
|-------------|----------|
| `None` | Timer is unregistered (one-shot) |
| `float` (e.g., `0.5`) | Timer reschedules after that many seconds |

**CRITICAL**: Timer functions execute on the main thread. Long-running timer callbacks freeze the UI.

### 8.5 bpy.msgbus (Message Bus)

The message bus provides property change subscriptions, similar to observer patterns.

```python
# Blender 3.x / 4.x / 5.x — Subscribe to property changes
import bpy

# Owner object (used to unsubscribe later; compared by identity)
owner = object()

# Subscribe to active object location changes
def on_location_changed(*args):
    print(f"Location changed! Args: {args}")

bpy.msgbus.subscribe_rna(
    key=bpy.context.object.location,    # Property to watch
    owner=owner,                         # Subscription owner
    args=("custom_data",),              # Passed to callback
    notify=on_location_changed,          # Callback function
    options=set(),                       # Or {'PERSISTENT'} to survive file loads
)

# Clear subscriptions for an owner
bpy.msgbus.clear_by_owner(owner)

# Subscribe to type-level changes (any object of this type)
bpy.msgbus.subscribe_rna(
    key=(bpy.types.Object, "location"),  # Tuple form: (type, property_name)
    owner=owner,
    args=(),
    notify=on_location_changed,
)

# Publish a message manually (forces notification)
bpy.msgbus.publish_rna(key=bpy.context.object.location)
```

**Key differences from property update callbacks:**
- msgbus callbacks are POSTPONED until all operators finish (not called immediately)
- msgbus callbacks are triggered ONCE per update cycle, even if property changed multiple times
- msgbus subscriptions are cleared on file load unless `PERSISTENT` option is used

### 8.6 bpy.app.handlers (Application Event Handlers)

Handlers are callback functions triggered by application events.

```python
# Blender 3.x / 4.x / 5.x — Application handlers
import bpy
from bpy.app.handlers import persistent

# Handler for file load completion
@persistent  # Survives file load (REQUIRED for load_post handlers)
def on_file_loaded(dummy):
    print(f"File loaded: {bpy.data.filepath}")
    # Re-initialize addon state here

bpy.app.handlers.load_post.append(on_file_loaded)

# Handler for depsgraph update
@persistent
def on_depsgraph_update(scene, depsgraph):
    for update in depsgraph.updates:
        print(f"Updated: {update.id.name}")

bpy.app.handlers.depsgraph_update_post.append(on_depsgraph_update)

# Handler for frame change (animation)
@persistent
def on_frame_change(scene):
    frame = scene.frame_current
    # Update custom calculations per frame

bpy.app.handlers.frame_change_post.append(on_frame_change)

# Remove handler
bpy.app.handlers.load_post.remove(on_file_loaded)
```

**Available handlers:**

| Handler | Trigger | Signature |
|---------|---------|-----------|
| `load_pre` | Before file load | `(filepath)` |
| `load_post` | After file load | `(filepath)` |
| `save_pre` | Before file save | `(filepath)` |
| `save_post` | After file save | `(filepath)` |
| `undo_pre` | Before undo | `(scene)` |
| `undo_post` | After undo | `(scene)` |
| `redo_pre` | Before redo | `(scene)` |
| `redo_post` | After redo | `(scene)` |
| `depsgraph_update_pre` | Before depsgraph eval | `(scene, depsgraph)` |
| `depsgraph_update_post` | After depsgraph eval | `(scene, depsgraph)` |
| `frame_change_pre` | Before frame change | `(scene)` |
| `frame_change_post` | After frame change | `(scene, depsgraph)` |
| `render_pre` | Before render starts | `(scene)` |
| `render_post` | After render completes | `(scene)` |
| `render_init` | Render engine init | `(engine)` |
| `render_complete` | Render fully done | `(scene)` |
| `render_cancel` | Render cancelled | `(scene)` |

**CRITICAL**: `frame_change_pre` and `frame_change_post` are called from ONE thread during rendering, while viewport updates happen on a DIFFERENT thread. If a handler modifies data accessed by the viewport, Blender WILL crash.

### 8.7 Running Python in Background Mode

```bash
# Command line — Background mode (no UI)
blender --background scene.blend --python script.py
blender -b scene.blend -P script.py

# With arguments to the script
blender -b scene.blend -P script.py -- --custom-arg value

# Render from command line
blender -b scene.blend -o /tmp/render_ -f 1
blender -b scene.blend -o /tmp/render_ -a  # Animation
```

```python
# Blender 3.x / 4.x / 5.x — Detect background mode in script
import bpy

if bpy.app.background:
    print("Running in background mode")
    # CANNOT use viewport operators
    # CANNOT use UI-related functions
    # CAN use render, data manipulation, file I/O
```

**Background mode limitations:**
- No viewport operators (anything requiring `SpaceView3D`)
- No UI drawing or interactive elements
- No `bpy.ops.wm.invoke_*` operators
- GPU-based operators may fail without display server
- Operators requiring user interaction (modal, file dialogs) are unavailable

### 8.8 bpy as a Module

Since Blender 2.8, `bpy` can be built and imported as a regular Python module outside of Blender's UI.

```python
# Outside Blender — Using bpy as a Python module
# Install: pip install bpy (or build from source)
import bpy

# Initialize a new scene
bpy.ops.wm.read_homefile(use_empty=True)

# All data operations work
cube = bpy.ops.mesh.primitive_cube_add(size=2.0)
bpy.ops.export_scene.gltf(filepath="/tmp/cube.glb")
```

**Limitations of bpy as a module:**
- No GUI or viewport
- Some operators that require UI context fail
- Must be the same Python version as the Blender build
- GPU-dependent features are unavailable
- Not all addons load correctly

### 8.9 Anti-Patterns: Python Runtime

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|-----------------|
| Calling `bpy` from a thread | Not thread-safe; CRASHES | Use `queue.Queue` + `bpy.app.timers` pattern |
| Storing `bpy.data` references long-term | Invalidated by undo/file operations | Store names/IDs; re-fetch when needed |
| `frame_change_post` handler modifying viewport data | Thread conflict during rendering | Use `depsgraph_update_post` or guard with `bpy.app.is_job_running('RENDER')` |
| Handler without `@persistent` | Removed on file load | ALWAYS use `@persistent` for addon handlers |
| Timer callback doing heavy computation | Freezes Blender UI | Offload work to a thread; use timer only for scheduling |
| Forgetting to remove handlers on addon unregister | Handlers accumulate on addon reload | ALWAYS remove handlers in `unregister()` |
| `scene['cycles']` dict access in 5.0+ | IDProperty access to RNA props removed | Use attribute access: `scene.cycles` |
| Using `del obj['prop']` to reset in 5.0+ | `del` on RNA-defined props removed | Use `obj.property_unset('prop')` |

### Sources
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy.msgbus.html
- https://docs.blender.org/api/current/bpy.app.handlers.html
- https://docs.blender.org/api/current/bpy.app.html
- https://docs.blender.org/api/current/info_gotchas.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/

---

## Consolidated Version Migration Reference

### Blender 4.0 Breaking Changes (Covered in This Document)

| Change | Old API | New API | Section |
|--------|---------|---------|---------|
| Node group interface | `tree.inputs.new()` | `tree.interface.new_socket()` | 1.3 |
| Principled BSDF sockets | `"Subsurface"` | `"Subsurface Weight"` | 3.3 |
| Bone layers | `bone.layers[i]` | `armature.collections` | 2.6 |
| Bone groups | `pose.bone_groups` | Bone collections + `color_tag` | 2.6 |
| OBJ export | `export_scene.obj()` | `wm.obj_export()` | 5.2 |
| STL export | `export_mesh.stl()` | `wm.stl_export()` | 5.2 |
| Proxies | `make_proxy()` | `make_override_library()` | 6.4 |

### Blender 4.2 Breaking Changes (Covered in This Document)

| Change | Old API | New API | Section |
|--------|---------|---------|---------|
| EEVEE identifier | `'BLENDER_EEVEE'` | `'BLENDER_EEVEE_NEXT'` | 4.2 |
| Blend method | `mat.blend_method` | `mat.surface_render_method` | 3.4 |

### Blender 5.0 Breaking Changes (Covered in This Document)

| Change | Old API | New API | Section |
|--------|---------|---------|---------|
| EEVEE identifier | `'BLENDER_EEVEE_NEXT'` | `'BLENDER_EEVEE'` | 4.2 |
| Compositor | `scene.node_tree` | `scene.compositing_node_group` | 1.7 |
| `scene.use_nodes` | Writable | Deprecated (always True) | 1.7 |
| `bgl` module | Available | REMOVED | 7.8 |
| `Image.bindcode` | Available | REMOVED, use `gpu.texture.from_image()` | 7.8 |
| Render pass names | Short (`"DiffCol"`) | Full (`"Diffuse Color"`) | 4.8 |
| IDProperty RNA access | `scene['cycles']` | `scene.cycles` (attribute) | 8.9 |
| Property reset | `del obj['prop']` | `obj.property_unset('prop')` | 8.9 |
| `context.active_file` | Available | REMOVED from asset shelf | 6.5 |
| `AssetHandle` | Available | REMOVED, use `AssetRepresentation` | 6.5 |
| GTAO properties | `eevee.use_gtao` | `view_layer.eevee.ambient_occlusion_distance` | 4.4 |
| File Output node | `file_slots`, `base_path` | `file_output_items`, `directory` | 4.8 |
| 15 bundled modules | Public | Made private (prefixed with `_`) | 8.9 |

---

## Global Sources

- https://docs.blender.org/api/current/ (main API reference)
- https://docs.blender.org/manual/en/latest/ (user manual)
- https://developer.blender.org/docs/release_notes/4.0/python_api/ (4.0 migration)
- https://developer.blender.org/docs/release_notes/5.0/python_api/ (5.0 migration)
- https://developer.blender.org/docs/release_notes/4.0/upgrading/bone_collections/ (bone collections migration)
