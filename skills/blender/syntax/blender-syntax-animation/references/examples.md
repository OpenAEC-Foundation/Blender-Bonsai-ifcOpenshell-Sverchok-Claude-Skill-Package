# blender-syntax-animation: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/bpy.types.FCurve.html
- https://docs.blender.org/api/current/bpy.types.Action.html
- https://docs.blender.org/api/current/bpy.types.NlaTrack.html
- https://docs.blender.org/api/current/bpy.types.Driver.html
- https://docs.blender.org/api/current/bpy.types.BoneCollection.html
- https://docs.blender.org/api/current/bpy.types.ShapeKey.html

---

## Example 1: Basic Keyframe Animation

```python
# Blender 3.x/4.x/5.x — animate object location along a path
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Define keyframe positions
keyframes = [
    (1, (0.0, 0.0, 0.0)),
    (30, (5.0, 0.0, 0.0)),
    (60, (5.0, 5.0, 0.0)),
    (90, (0.0, 5.0, 0.0)),
    (120, (0.0, 0.0, 0.0)),
]

for frame, location in keyframes:
    obj.location = location                             # Set value FIRST
    obj.keyframe_insert(data_path="location", frame=frame)  # Then insert keyframe

# Set scene frame range to match animation
bpy.context.scene.frame_start = 1
bpy.context.scene.frame_end = 120
```

---

## Example 2: Animate Rotation and Scale

```python
# Blender 3.x/4.x/5.x — animate rotation and scale
import bpy
from math import radians

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Ensure rotation mode is Euler
obj.rotation_mode = 'XYZ'

# Frame 1: identity
obj.rotation_euler = (0.0, 0.0, 0.0)
obj.scale = (1.0, 1.0, 1.0)
obj.keyframe_insert(data_path="rotation_euler", frame=1)
obj.keyframe_insert(data_path="scale", frame=1)

# Frame 60: rotated and scaled
obj.rotation_euler = (0.0, 0.0, radians(360))
obj.scale = (2.0, 2.0, 2.0)
obj.keyframe_insert(data_path="rotation_euler", frame=60)
obj.keyframe_insert(data_path="scale", frame=60)
```

---

## Example 3: FCurve Direct Manipulation (Bulk Keyframes)

```python
# Blender 3.x/4.x/5.x — create animation via direct FCurve API
import bpy
import math

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Ensure animation data and action exist
if obj.animation_data is None:
    obj.animation_data_create()
if obj.animation_data.action is None:
    obj.animation_data.action = bpy.data.actions.new(name="SineWave")

action = obj.animation_data.action

# Create FCurve for Z location
fcurve = action.fcurves.new(data_path="location", index=2, action_group="Location")

# Generate sine wave keyframes
num_frames = 120
fcurve.keyframe_points.add(count=num_frames)
for i in range(num_frames):
    frame = float(i + 1)
    value = math.sin(frame / 10.0) * 3.0
    fcurve.keyframe_points[i].co = (frame, value)
    fcurve.keyframe_points[i].interpolation = 'BEZIER'
    fcurve.keyframe_points[i].handle_left_type = 'AUTO'
    fcurve.keyframe_points[i].handle_right_type = 'AUTO'

# MUST call update after bulk keyframe manipulation
fcurve.update()

bpy.context.scene.frame_end = num_frames
```

---

## Example 4: FCurve Modifiers — Noise and Cycles

```python
# Blender 3.x/4.x/5.x — add FCurve modifiers
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Create a simple two-keyframe animation first
obj.location = (0.0, 0.0, 0.0)
obj.keyframe_insert(data_path="location", frame=1)
obj.location = (10.0, 0.0, 0.0)
obj.keyframe_insert(data_path="location", frame=60)

# Access the X location FCurve
action = obj.animation_data.action
fcurve_x = action.fcurves.find(data_path="location", index=0)

# Add Cycles modifier to loop the animation
cycles = fcurve_x.modifiers.new(type='CYCLES')
cycles.mode_before = 'REPEAT'       # 'NONE', 'REPEAT', 'REPEAT_OFFSET', 'MIRROR'
cycles.mode_after = 'REPEAT_OFFSET' # Repeat with cumulative offset

# Add Noise modifier for variation
noise = fcurve_x.modifiers.new(type='NOISE')
noise.scale = 3.0        # Noise frequency
noise.strength = 0.5     # Noise amplitude
noise.phase = 0.0        # Phase offset
noise.depth = 0          # Fractal depth (0 = simple noise)
noise.blend_type = 'ADD' # 'REPLACE', 'ADD', 'SUBTRACT', 'MULTIPLY'
```

---

## Example 5: Driver — Property Linked to Another Object

```python
# Blender 3.x/4.x/5.x — driver linking Z scale to Empty's Z location
import bpy

# Create objects
bpy.ops.object.empty_add(type='PLAIN_AXES', location=(0, 0, 0))
controller = bpy.context.active_object
controller.name = "HeightControl"

cube = bpy.data.objects.get("Cube")
if cube is None:
    bpy.ops.mesh.primitive_cube_add(location=(3, 0, 0))
    cube = bpy.context.active_object

# Add driver to cube's Z scale
driver_fcurve = cube.driver_add("scale", 2)  # index 2 = Z
driver = driver_fcurve.driver
driver.type = 'SCRIPTED'

# Add variable
var = driver.variables.new()
var.name = "height"
var.type = 'TRANSFORMS'
target = var.targets[0]
target.id = controller
target.transform_type = 'LOC_Z'
target.transform_space = 'WORLD_SPACE'

# Expression: scale Z = 1 + controller Z location
driver.expression = "1.0 + height"
```

---

## Example 6: Driver — Single Property Reference

```python
# Blender 3.x/4.x/5.x — driver referencing a custom property
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Add custom property
obj["my_factor"] = 1.0

# Add driver to X location
driver_fc = obj.driver_add("location", 0)
driver = driver_fc.driver
driver.type = 'SCRIPTED'

# Variable referencing the custom property
var = driver.variables.new()
var.name = "factor"
var.type = 'SINGLE_PROP'
target = var.targets[0]
target.id_type = 'OBJECT'
target.id = obj
target.data_path = '["my_factor"]'

driver.expression = "factor * 5.0"
```

---

## Example 7: Driver with `use_self`

```python
# Blender 3.x/4.x/5.x — driver using self reference
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Add driver to Y location
driver_fc = obj.driver_add("location", 1)
driver = driver_fc.driver
driver.type = 'SCRIPTED'

# Enable self reference — REQUIRED before using 'self' in expression
driver.use_self = True

# Expression using self to read another property of the same object
driver.expression = "self.location.x * 0.5"
# 'self' refers to the object that owns this driver
```

---

## Example 8: NLA — Layering Multiple Actions

```python
# Blender 3.x/4.x/5.x — combine multiple actions via NLA
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Create two separate actions
# Action 1: horizontal movement
action_move = bpy.data.actions.new(name="HorizontalMove")
fc = action_move.fcurves.new(data_path="location", index=0)
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (60.0, 10.0)
fc.update()

# Action 2: vertical bounce
action_bounce = bpy.data.actions.new(name="VerticalBounce")
fc = action_bounce.fcurves.new(data_path="location", index=2)
fc.keyframe_points.add(count=3)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (30.0, 3.0)
fc.keyframe_points[2].co = (60.0, 0.0)
fc.update()

# Set up NLA
anim_data = obj.animation_data
if anim_data is None:
    anim_data = obj.animation_data_create()

# Track 1: horizontal movement
track1 = anim_data.nla_tracks.new()
track1.name = "Horizontal"
strip1 = track1.strips.new(name="Move", start=1, action=action_move)
strip1.blend_type = 'REPLACE'

# Track 2: vertical bounce (added on top)
track2 = anim_data.nla_tracks.new()
track2.name = "Vertical"
strip2 = track2.strips.new(name="Bounce", start=1, action=action_bounce)
strip2.blend_type = 'ADD'  # Add on top of track 1

# Clear the active action (NLA evaluates from tracks)
anim_data.action = None
```

---

## Example 9: Armature with Bone Collections (4.0+)

```python
# Blender 4.0+/5.x — create armature with organized bone collections
import bpy

# Create armature data
armature_data = bpy.data.armatures.new("CharacterRig")
armature_obj = bpy.data.objects.new("CharacterRig", armature_data)
bpy.context.collection.objects.link(armature_obj)

# Set active and enter Edit Mode
bpy.context.view_layer.objects.active = armature_obj
bpy.ops.object.mode_set(mode='EDIT')

# Create bone hierarchy
root = armature_data.edit_bones.new("Root")
root.head = (0, 0, 0)
root.tail = (0, 0, 0.5)

spine = armature_data.edit_bones.new("Spine")
spine.head = (0, 0, 0.5)
spine.tail = (0, 0, 1.5)
spine.parent = root
spine.use_connect = True

head_bone = armature_data.edit_bones.new("Head")
head_bone.head = (0, 0, 1.5)
head_bone.tail = (0, 0, 2.0)
head_bone.parent = spine
head_bone.use_connect = True

# Exit Edit Mode before working with bone collections
bpy.ops.object.mode_set(mode='OBJECT')

# Create bone collections
deform_col = armature_data.collections.new("Deform")
deform_col.color_tag = 'THEME09'
deform_col.is_visible = True

control_col = armature_data.collections.new("Controls")
control_col.color_tag = 'THEME01'
control_col.is_visible = True

hidden_col = armature_data.collections.new("Hidden")
hidden_col.is_visible = False

# Assign bones to collections
deform_col.assign(armature_data.bones["Root"])
deform_col.assign(armature_data.bones["Spine"])
control_col.assign(armature_data.bones["Head"])

print(f"Deform bones: {[b.name for b in deform_col.bones]}")
print(f"Control bones: {[b.name for b in control_col.bones]}")
```

---

## Example 10: Shape Key Animation with Driver

```python
# Blender 3.x/4.x/5.x — shape key driven by another object
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Add Basis shape key (MUST be first)
if obj.data.shape_keys is None:
    obj.shape_key_add(name="Basis", from_mix=False)

# Add deformed shape key
deformed = obj.shape_key_add(name="Expand", from_mix=False)
for vert in deformed.data:
    vert.co *= 1.5  # Scale all vertices by 1.5x

# Create controller empty
bpy.ops.object.empty_add(type='SINGLE_ARROW', location=(3, 0, 0))
controller = bpy.context.active_object
controller.name = "ShapeControl"

# Add driver to shape key value
key_block = obj.data.shape_keys.key_blocks["Expand"]
driver_fc = key_block.driver_add("value")
driver = driver_fc.driver
driver.type = 'SCRIPTED'

var = driver.variables.new()
var.name = "ctrl"
var.type = 'TRANSFORMS'
target = var.targets[0]
target.id = controller
target.transform_type = 'LOC_Z'
target.transform_space = 'WORLD_SPACE'

# Clamp shape key value between 0 and 1
driver.expression = "max(0.0, min(1.0, ctrl))"
```

---

## Example 11: Batch Keyframe Insertion with Action Groups

```python
# Blender 3.x/4.x/5.x — organized keyframe insertion with groups
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Animate with named groups for Action Editor organization
frames_data = {
    1: {"location": (0, 0, 0), "rotation_euler": (0, 0, 0), "scale": (1, 1, 1)},
    30: {"location": (5, 0, 2), "rotation_euler": (0, 0, 0.785), "scale": (1.5, 1.5, 1.5)},
    60: {"location": (0, 0, 0), "rotation_euler": (0, 0, 0), "scale": (1, 1, 1)},
}

obj.rotation_mode = 'XYZ'

for frame, props in frames_data.items():
    for data_path, value in props.items():
        setattr(obj, data_path, value)
        obj.keyframe_insert(data_path=data_path, frame=frame, group="Transform")
```

---

## Example 12: Reading and Modifying Existing Animation

```python
# Blender 3.x/4.x/5.x — read, filter, and modify existing FCurves
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None or obj.animation_data is None:
    raise RuntimeError("Object 'Cube' has no animation")

action = obj.animation_data.action
if action is None:
    raise RuntimeError("No action assigned")

# Find all location FCurves
for fcurve in action.fcurves:
    if fcurve.data_path == "location":
        axis = ["X", "Y", "Z"][fcurve.array_index]
        print(f"Location {axis}: {len(fcurve.keyframe_points)} keyframes")

        # Double all keyframe values
        for kp in fcurve.keyframe_points:
            kp.co[1] *= 2.0  # Multiply value by 2

        # Set all interpolation to linear
        for kp in fcurve.keyframe_points:
            kp.interpolation = 'LINEAR'

        # MUST call update after modifications
        fcurve.update()

# Remove all scale animation
fcurves_to_remove = [fc for fc in action.fcurves if fc.data_path == "scale"]
for fc in fcurves_to_remove:
    action.fcurves.remove(fc)
```

---

## Example 13: Constraint Animation

```python
# Blender 3.x/4.x/5.x — animate constraint influence
import bpy

obj = bpy.data.objects.get("Cube")
target = bpy.data.objects.get("Empty")
if obj is None or target is None:
    raise RuntimeError("Objects not found")

# Add Copy Location constraint
constraint = obj.constraints.new('COPY_LOCATION')
constraint.name = "FollowTarget"
constraint.target = target

# Animate the constraint influence
constraint.influence = 0.0
constraint.keyframe_insert(data_path="influence", frame=1)

constraint.influence = 1.0
constraint.keyframe_insert(data_path="influence", frame=30)

constraint.influence = 0.0
constraint.keyframe_insert(data_path="influence", frame=60)
```

---

## Example 14: Timeline Scrubbing with Depsgraph Update

```python
# Blender 3.x/4.x/5.x — iterate frames and read evaluated data
import bpy

scene = bpy.context.scene
obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# Iterate through frames and read animated position
positions = []
for frame in range(scene.frame_start, scene.frame_end + 1):
    scene.frame_set(frame)  # Updates depsgraph
    # Read evaluated location (with constraints, drivers, etc.)
    depsgraph = bpy.context.evaluated_depsgraph_get()
    obj_eval = obj.evaluated_get(depsgraph)
    pos = obj_eval.matrix_world.translation.copy()
    positions.append((frame, pos))

print(f"Collected {len(positions)} frame positions")

# Restore original frame
scene.frame_set(scene.frame_start)
```

---

## Example 15: Pose Bone Keyframe Animation

```python
# Blender 3.x/4.x/5.x — animate armature pose bones
import bpy
from math import radians

armature = bpy.data.objects.get("Armature")
if armature is None:
    raise RuntimeError("Armature not found")

# MUST be in Pose Mode or Object Mode to access pose bones
bpy.context.view_layer.objects.active = armature
bpy.ops.object.mode_set(mode='POSE')

# Animate a bone
bone = armature.pose.bones.get("Forearm")
if bone is None:
    raise RuntimeError("Bone 'Forearm' not found")

bone.rotation_mode = 'XYZ'

# Frame 1: rest
bone.rotation_euler = (0, 0, 0)
bone.keyframe_insert(data_path="rotation_euler", frame=1)

# Frame 30: bent
bone.rotation_euler = (radians(-90), 0, 0)
bone.keyframe_insert(data_path="rotation_euler", frame=30)

# Frame 60: rest again
bone.rotation_euler = (0, 0, 0)
bone.keyframe_insert(data_path="rotation_euler", frame=60)

bpy.ops.object.mode_set(mode='OBJECT')
```

---

## Example 16: Action Stashing via NLA

```python
# Blender 3.x/4.x/5.x — stash actions for later use
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

anim_data = obj.animation_data
if anim_data is None:
    anim_data = obj.animation_data_create()

# Create first action
action1 = bpy.data.actions.new(name="Walk")
action1.use_fake_user = True  # Prevent purge
fc = action1.fcurves.new(data_path="location", index=1)
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (60.0, 10.0)
fc.update()

# Create second action
action2 = bpy.data.actions.new(name="Run")
action2.use_fake_user = True
fc = action2.fcurves.new(data_path="location", index=1)
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (30.0, 10.0)
fc.update()

# Push actions to NLA tracks for storage
anim_data.action = action1
track1 = anim_data.nla_tracks.new()
track1.name = "Walk"
strip1 = track1.strips.new(name="Walk", start=1, action=action1)
strip1.mute = True  # Muted = stashed, not actively playing

anim_data.action = action2
track2 = anim_data.nla_tracks.new()
track2.name = "Run"
strip2 = track2.strips.new(name="Run", start=1, action=action2)
strip2.mute = True

# Set active action for editing
anim_data.action = action1
```
