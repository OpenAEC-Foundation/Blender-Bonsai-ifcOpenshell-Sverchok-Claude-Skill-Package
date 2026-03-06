---
name: blender-syntax-animation
description: "Covers Blender animation API including keyframe insertion, FCurve access, Action data blocks, NLA system, BoneCollection (4.0+), armature bone layers migration, driver expressions, and timeline control. Activates when creating animations programmatically, managing keyframes, working with armatures, or setting up drivers."
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-animation

## Quick Reference

### Critical Warnings

**ALWAYS** set the property value BEFORE calling `keyframe_insert()` — keyframes record the current value at call time.

**ALWAYS** call `fcurve.update()` after manually adding or modifying keyframe points via `keyframe_points.add()` or direct `.co` assignment.

**NEVER** access `edit_bones` outside of Edit Mode — the collection is empty/inaccessible in Object or Pose Mode.

**NEVER** use `bone.layers` or `pose.bone_groups` in Blender 4.0+ — these are removed. Use `BoneCollection` instead.

**NEVER** use `driver.expression` referencing `self` without enabling `driver.use_self = True`.

**ALWAYS** create `animation_data` before accessing `.action` or `.nla_tracks` — call `obj.animation_data_create()` if `obj.animation_data is None`.

### Decision Tree: Animation Approach

```
Need to animate a property?
├─ Simple keyframe on existing property?
│  └─ Use obj.keyframe_insert(data_path=..., frame=...)
├─ Need programmatic control over curve shape?
│  └─ Use FCurve API: action.fcurves.new() + keyframe_points.add()
├─ Need to combine/layer multiple animations?
│  └─ Use NLA system: nla_tracks.new() + track.strips.new()
├─ Need property driven by expression or other property?
│  └─ Use Drivers: obj.driver_add() + driver.expression
├─ Need to organize armature bones into groups (4.0+)?
│  └─ Use BoneCollection: armature.collections.new()
└─ Need parametric mesh deformation?
   └─ Use Shape Keys: obj.shape_key_add() + keyframe on .value
```

### Version Matrix: Animation Features

| Feature | Blender 3.x | Blender 4.0+ | Blender 5.x |
|---------|-------------|--------------|-------------|
| `keyframe_insert()` | Available | Available | Available |
| `FCurve` / `Action` | Available | Available | Available |
| `NLA` system | Available | Available | Available |
| `bone.layers[i]` | Available | **REMOVED** | **REMOVED** |
| `pose.bone_groups` | Available | **REMOVED** | **REMOVED** |
| `BoneCollection` | Not available | **NEW** | Available |
| `BoneCollection.color_tag` | Not available | **NEW** | Available |
| Drivers | Available | Available | Available |
| Shape Keys | Available | Available | Available |

---

## Essential Patterns

### Pattern 1: Keyframe Insertion

```python
# Blender 3.x/4.x/5.x — insert keyframes on animatable properties
import bpy

obj = bpy.data.objects.get("Cube")
if obj is None:
    raise RuntimeError("Object 'Cube' not found")

# ALWAYS set value BEFORE inserting keyframe
obj.location = (0.0, 0.0, 0.0)
obj.keyframe_insert(data_path="location", frame=1)

obj.location = (5.0, 0.0, 3.0)
obj.keyframe_insert(data_path="location", frame=60)

# Single axis only (index: 0=X, 1=Y, 2=Z, -1=all)
obj.keyframe_insert(data_path="location", frame=30, index=2)  # Z only

# Group parameter organizes FCurves in the Action Editor
obj.keyframe_insert(data_path="rotation_euler", frame=1, group="Transform")
```

**Parameters for `keyframe_insert()`:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `data_path` | `str` | RNA path to property (`"location"`, `"rotation_euler"`, `"scale"`) |
| `index` | `int` | Array index for vector properties (-1 = all, 0 = X, 1 = Y, 2 = Z) |
| `frame` | `float` | Frame number for the keyframe |
| `group` | `str` | Action group name for this FCurve |
| `options` | `set` | `{'INSERTKEY_NEEDED'}`, `{'INSERTKEY_VISUAL'}`, etc. |

### Pattern 2: FCurve Access and Manipulation

```python
# Blender 3.x/4.x/5.x — direct FCurve manipulation
import bpy

obj = bpy.data.objects.get("Cube")

# Ensure animation data exists
anim_data = obj.animation_data
if anim_data is None:
    anim_data = obj.animation_data_create()

# Ensure action exists
action = anim_data.action
if action is None:
    action = bpy.data.actions.new(name="MyAction")
    anim_data.action = action

# Create FCurve for X location
fcurve = action.fcurves.new(data_path="location", index=0)

# Add keyframe points in bulk
fcurve.keyframe_points.add(count=3)
fcurve.keyframe_points[0].co = (1.0, 0.0)    # (frame, value)
fcurve.keyframe_points[1].co = (30.0, 5.0)
fcurve.keyframe_points[2].co = (60.0, 0.0)

# Set interpolation per keyframe
for kp in fcurve.keyframe_points:
    kp.interpolation = 'BEZIER'
    kp.handle_left_type = 'AUTO_CLAMPED'
    kp.handle_right_type = 'AUTO_CLAMPED'

# ALWAYS call update after manual changes
fcurve.update()
```

### Pattern 3: Drivers

```python
# Blender 3.x/4.x/5.x — add driver to a property
import bpy

obj = bpy.data.objects.get("Cube")

# Add driver to Z location (returns FCurve with .driver attribute)
driver_fcurve = obj.driver_add("location", 2)
driver = driver_fcurve.driver

# Set driver type
driver.type = 'SCRIPTED'  # 'AVERAGE', 'SUM', 'SCRIPTED', 'MIN', 'MAX'

# Add a variable referencing another object's transform
var = driver.variables.new()
var.name = "ctrl_z"
var.type = 'TRANSFORMS'

target = var.targets[0]
target.id = bpy.data.objects["ControlEmpty"]
target.transform_type = 'LOC_Z'
target.transform_space = 'WORLD_SPACE'

# Set expression
driver.expression = "ctrl_z * 2.0"

# Remove a driver
# obj.driver_remove("location", 2)
```

### Pattern 4: NLA System

```python
# Blender 3.x/4.x/5.x — NLA workflow
import bpy

obj = bpy.data.objects.get("Cube")
anim_data = obj.animation_data
if anim_data is None:
    anim_data = obj.animation_data_create()

# Create NLA track
track = anim_data.nla_tracks.new()
track.name = "MovementTrack"

# Push action as NLA strip
action = bpy.data.actions.get("MoveAction")
if action is not None:
    strip = track.strips.new(name="Move", start=1, action=action)
    strip.blend_type = 'REPLACE'   # 'REPLACE','COMBINE','ADD','SUBTRACT','MULTIPLY'
    strip.use_auto_blend = True
    strip.repeat = 1.0
    strip.scale = 1.0
    strip.mute = False
```

### Pattern 5: Bone Collections (4.0+)

```python
# Blender 4.0+/5.x — bone collections replace bone layers and bone groups
import bpy

armature_data = bpy.data.armatures.new("MyArmature")
armature_obj = bpy.data.objects.new("MyArmature", armature_data)
bpy.context.collection.objects.link(armature_obj)

# MUST enter Edit Mode to create/edit bones
bpy.context.view_layer.objects.active = armature_obj
bpy.ops.object.mode_set(mode='EDIT')

root = armature_data.edit_bones.new("Root")
root.head = (0, 0, 0)
root.tail = (0, 0, 1)

child = armature_data.edit_bones.new("Child")
child.head = (0, 0, 1)
child.tail = (0, 0, 2)
child.parent = root

bpy.ops.object.mode_set(mode='OBJECT')

# Create bone collections (4.0+ only)
deform_col = armature_data.collections.new("Deform")
control_col = armature_data.collections.new("Controls")

# Assign bones to collections
deform_col.assign(armature_data.bones["Root"])
control_col.assign(armature_data.bones["Child"])

# Set visibility and color
deform_col.is_visible = True
control_col.is_visible = True
control_col.color_tag = 'THEME01'  # THEME01..THEME20 or COLOR_01..COLOR_20
```

### Pattern 6: Shape Keys

```python
# Blender 3.x/4.x/5.x — shape keys for parametric deformation
import bpy

obj = bpy.data.objects.get("Cube")

# Add basis shape key FIRST (required)
basis = obj.shape_key_add(name="Basis", from_mix=False)

# Add deformed shape key
deformed = obj.shape_key_add(name="Open", from_mix=False)
for i, vert in enumerate(deformed.data):
    vert.co.z += 0.5  # Modify relative to Basis

# Animate shape key value
key_block = obj.data.shape_keys.key_blocks["Open"]
key_block.value = 0.0
key_block.keyframe_insert(data_path="value", frame=1)
key_block.value = 1.0
key_block.keyframe_insert(data_path="value", frame=30)
```

---

## Common Operations

### Timeline Control

```python
# Blender 3.x/4.x/5.x — scene timeline settings
import bpy

scene = bpy.context.scene

# Set frame range
scene.frame_start = 1
scene.frame_end = 250

# Set current frame
scene.frame_set(50)         # Updates depsgraph
scene.frame_current = 50    # Direct assignment (no depsgraph update)

# Get FPS
fps = scene.render.fps / scene.render.fps_base
```

### Reading FCurve Data

```python
# Blender 3.x/4.x/5.x — iterate and query FCurves
import bpy

obj = bpy.data.objects.get("Cube")
if obj.animation_data and obj.animation_data.action:
    action = obj.animation_data.action
    for fcurve in action.fcurves:
        print(f"Path: {fcurve.data_path}[{fcurve.array_index}]")
        print(f"  Range: {fcurve.range()}")
        # Evaluate at specific frame
        val = fcurve.evaluate(frame=25.0)
        print(f"  Value at frame 25: {val}")
```

### FCurve Modifiers

```python
# Blender 3.x/4.x/5.x — add modifiers to FCurves
import bpy

obj = bpy.data.objects.get("Cube")
action = obj.animation_data.action
fcurve = action.fcurves[0]

# Add Noise modifier
noise = fcurve.modifiers.new(type='NOISE')
noise.scale = 5.0
noise.strength = 0.3
noise.phase = 0.0

# Add Cycles modifier (loop animation)
cycles = fcurve.modifiers.new(type='CYCLES')
cycles.mode_before = 'REPEAT'
cycles.mode_after = 'REPEAT'
```

### Constraints

```python
# Blender 3.x/4.x/5.x — add constraints to objects
import bpy

obj = bpy.data.objects.get("Cube")

# Copy Location constraint
constraint = obj.constraints.new('COPY_LOCATION')
constraint.name = "Follow"
constraint.target = bpy.data.objects.get("Empty")
constraint.influence = 0.5

# IK constraint on pose bone
armature = bpy.data.objects.get("Armature")
if armature:
    pose_bone = armature.pose.bones.get("Hand")
    if pose_bone:
        ik = pose_bone.constraints.new('IK')
        ik.target = bpy.data.objects.get("IKTarget")
        ik.chain_count = 3
```

### Bone Access by Mode

| Mode | Collection | Access Type | Use Case |
|------|-----------|-------------|----------|
| Edit Mode | `armature.edit_bones` | Read/Write | Create, delete, reparent bones |
| Object Mode | `armature.bones` | Read-only properties | Read rest pose, check collections |
| Pose Mode | `armature.pose.bones` | Transforms, constraints | Animate, add constraints |

---

## Version Migration: Bone Layers to Bone Collections

### Blender 3.x (Legacy — BROKEN in 4.0+)

```python
# BROKEN in 4.0+ — DO NOT USE
# bone.layers[0] = True              # REMOVED
# bone.layers[16] = True             # REMOVED
# pose_bone_group = pose.bone_groups.new(name="Controls")  # REMOVED
# pose_bone_group.color_set = 'THEME01'                    # REMOVED
```

### Blender 4.0+ (Current)

```python
# Blender 4.0+/5.x — bone collections API
armature = bpy.data.armatures["MyArmature"]

# Create collections
col = armature.collections.new("MyGroup")

# Assign bones
col.assign(armature.bones["BoneName"])

# Unassign bones
col.unassign(armature.bones["BoneName"])

# Check membership
is_member = armature.bones["BoneName"] in col.bones

# Color (replaces bone_groups color)
col.color_tag = 'THEME05'

# Visibility (replaces bone layers visibility)
col.is_visible = False
```

### Version-Safe Pattern

```python
import bpy

def assign_bone_to_group(armature_data, bone_name, group_name, color_tag='THEME01'):
    """Assign bone to a named group. Works in 4.0+ only."""
    if bpy.app.version < (4, 0, 0):
        raise RuntimeError("BoneCollection requires Blender 4.0+")
    col = armature_data.collections.get(group_name)
    if col is None:
        col = armature_data.collections.new(group_name)
        col.color_tag = color_tag
    bone = armature_data.bones.get(bone_name)
    if bone is not None:
        col.assign(bone)
    return col
```

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for keyframe_insert, FCurve, Action, NLA, BoneCollection, Driver
- [references/examples.md](references/examples.md) — Working code examples for animation workflows
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with animation APIs

### Official Sources

- https://docs.blender.org/api/current/bpy.types.FCurve.html
- https://docs.blender.org/api/current/bpy.types.Keyframe.html
- https://docs.blender.org/api/current/bpy.types.Action.html
- https://docs.blender.org/api/current/bpy.types.NlaTrack.html
- https://docs.blender.org/api/current/bpy.types.NlaStrip.html
- https://docs.blender.org/api/current/bpy.types.Driver.html
- https://docs.blender.org/api/current/bpy.types.DriverVariable.html
- https://docs.blender.org/api/current/bpy.types.BoneCollection.html
- https://docs.blender.org/api/current/bpy.types.ShapeKey.html
- https://docs.blender.org/api/current/bpy.types.EditBone.html
- https://docs.blender.org/api/current/bpy.types.FCurveKeyframePoints.html
- https://developer.blender.org/docs/release_notes/4.0/upgrading/bone_collections/
