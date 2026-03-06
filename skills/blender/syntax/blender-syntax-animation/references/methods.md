# blender-syntax-animation: API Method Reference

Sources:
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

---

## bpy_struct.keyframe_insert() — Blender 3.x/4.x/5.x

Available on any `bpy_struct` with animatable properties.

```python
bpy_struct.keyframe_insert(
    data_path: str,          # RNA path to the property
    index: int = -1,         # Array index (-1 = all)
    frame: float = current,  # Frame number
    group: str = "",         # Action group name
    options: set = set()     # Insertion options
) -> bool                    # True if keyframe was inserted
```

**Options set values:**

| Option | Description |
|--------|-------------|
| `'INSERTKEY_NEEDED'` | Only insert if value differs from existing |
| `'INSERTKEY_VISUAL'` | Insert based on visual transform (with constraints applied) |
| `'INSERTKEY_XYZ_TO_RGB'` | Map XYZ channels to RGB colors in FCurve display |
| `'INSERTKEY_REPLACE'` | Replace existing keyframe at same frame |
| `'INSERTKEY_AVAILABLE'` | Only insert on already-existing FCurves |
| `'INSERTKEY_CYCLE_AWARE'` | Take cycle offset into account |

**Common animatable data_path values:**

| data_path | Property | Type |
|-----------|----------|------|
| `"location"` | Object location | `Vector(3)` |
| `"rotation_euler"` | Euler rotation | `Vector(3)` |
| `"rotation_quaternion"` | Quaternion rotation | `Vector(4)` |
| `"rotation_axis_angle"` | Axis-angle rotation | `Vector(4)` |
| `"scale"` | Object scale | `Vector(3)` |
| `"hide_viewport"` | Viewport visibility | `bool` |
| `"hide_render"` | Render visibility | `bool` |
| `"color"` | Object color | `Vector(4)` |
| `"energy"` (Light) | Light intensity | `float` |
| `"lens"` (Camera) | Focal length | `float` |

```python
bpy_struct.keyframe_delete(
    data_path: str,
    index: int = -1,
    frame: float = current,
    group: str = ""
) -> bool
```

---

## bpy_struct.driver_add() / driver_remove() — Blender 3.x/4.x/5.x

```python
bpy_struct.driver_add(
    path: str,           # RNA path to the property
    index: int = -1      # Array index (-1 = all)
) -> FCurve | list[FCurve]
# Returns single FCurve if index specified, list if index=-1 for vector property.
# The returned FCurve has a .driver attribute (bpy.types.Driver).

bpy_struct.driver_remove(
    path: str,
    index: int = -1
) -> bool
```

---

## bpy.types.AnimData — Blender 3.x/4.x/5.x

Every ID data block can hold animation data via `.animation_data`.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `action` | `Action | None` | Active action providing keyframe data |
| `action_blend_type` | `str` | Blending mode for action: `'REPLACE'`, `'COMBINE'`, `'ADD'`, `'SUBTRACT'`, `'MULTIPLY'` |
| `action_extrapolation` | `str` | `'HOLD'`, `'HOLD_FORWARD'`, `'NOTHING'` |
| `action_influence` | `float` | Influence of action (0.0 to 1.0) |
| `drivers` | `AnimDataDrivers` | Collection of driver FCurves |
| `nla_tracks` | `NlaTracks` | Collection of NLA tracks |
| `use_nla` | `bool` | Enable NLA evaluation |
| `use_tweak_mode` | `bool` | True when editing an NLA strip's action |

### Methods

```python
# Create animation data on an object (Blender 3.x/4.x/5.x)
obj.animation_data_create() -> AnimData

# Clear animation data from an object
obj.animation_data_clear()
```

---

## bpy.types.Action — Blender 3.x/4.x/5.x

Actions store FCurves and are shared via `bpy.data.actions`.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `fcurves` | `ActionFCurves` | Collection of FCurves in this action |
| `groups` | `ActionGroups` | Named groups for organizing FCurves |
| `frame_range` | `Vector(2)` | Start and end frame of the action (read-only) |
| `id_root` | `str` | ID type this action is intended for (e.g., `'OBJECT'`) |
| `use_frame_range` | `bool` | Enable manual frame range |
| `frame_start` | `float` | Manual start frame (when `use_frame_range` is True) |
| `frame_end` | `float` | Manual end frame |
| `use_fake_user` | `bool` | Prevent purge when no users |
| `use_cyclic` | `bool` | Enable cyclic evaluation |

### ActionFCurves Methods

```python
# Create new FCurve (Blender 3.x/4.x/5.x)
action.fcurves.new(
    data_path: str,       # RNA path
    index: int = 0,       # Array index
    action_group: str = "" # Group name
) -> FCurve

# Find an existing FCurve
action.fcurves.find(
    data_path: str,
    index: int = 0
) -> FCurve | None

# Remove an FCurve
action.fcurves.remove(fcurve: FCurve)
```

---

## bpy.types.FCurve — Blender 3.x/4.x/5.x

An FCurve maps a single property to a function of time.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `data_path` | `str` | RNA path to the animated property |
| `array_index` | `int` | Index for vector properties |
| `keyframe_points` | `FCurveKeyframePoints` | Collection of keyframe points |
| `modifiers` | `FCurveModifiers` | Collection of FCurve modifiers |
| `driver` | `Driver | None` | Driver object (for driver FCurves) |
| `group` | `ActionGroup | None` | Action group this FCurve belongs to |
| `color` | `Vector(3)` | Display color in Graph Editor |
| `color_mode` | `str` | `'AUTO_RAINBOW'`, `'AUTO_RGB'`, `'AUTO_YRGB'`, `'CUSTOM'` |
| `mute` | `bool` | Silence this FCurve |
| `hide` | `bool` | Hide in the Graph Editor |
| `lock` | `bool` | Prevent editing |
| `select` | `bool` | Selection state |
| `is_valid` | `bool` | True if the FCurve path resolves |
| `is_empty` | `bool` | True if no keyframes or modifiers |
| `auto_smoothing` | `str` | `'NONE'`, `'CONT_ACCEL'` |
| `extrapolation` | `str` | `'CONSTANT'`, `'LINEAR'` |

### Methods

```python
# Evaluate FCurve at a frame (Blender 3.x/4.x/5.x)
fcurve.evaluate(frame: float) -> float

# Get frame range
fcurve.range() -> tuple[float, float]  # (min_frame, max_frame)

# Recalculate after manual changes — MUST CALL
fcurve.update()

# Convert between keyframes and samples
fcurve.convert_to_samples(start: int, end: int)
fcurve.convert_to_keyframes(start: int, end: int)
```

---

## bpy.types.FCurveKeyframePoints — Blender 3.x/4.x/5.x

Collection of keyframe points on an FCurve.

### Methods

```python
# Add keyframe points (Blender 3.x/4.x/5.x)
fcurve.keyframe_points.add(count: int)
# Adds `count` keyframe points. Set .co on each afterward.

# Insert keyframe with immediate sort
fcurve.keyframe_points.insert(
    frame: float,
    value: float,
    options: set = set(),    # {'REPLACE', 'NEEDED', 'FAST'}
    keyframe_type: str = 'KEYFRAME'  # 'KEYFRAME','BREAKDOWN','MOVING_HOLD','EXTREME','JITTER'
) -> Keyframe

# Remove a keyframe
fcurve.keyframe_points.remove(keyframe: Keyframe, fast: bool = False)
# fast=True skips re-sorting (call fcurve.update() when done)
```

---

## bpy.types.Keyframe — Blender 3.x/4.x/5.x

Individual keyframe point on an FCurve.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `co` | `Vector(2)` | Keyframe coordinates: `(frame, value)` |
| `co_ui` | `Vector(2)` | Coordinates in UI space |
| `interpolation` | `str` | Interpolation type (see table below) |
| `easing` | `str` | `'AUTO'`, `'EASE_IN'`, `'EASE_OUT'`, `'EASE_IN_OUT'` |
| `type` | `str` | `'KEYFRAME'`, `'BREAKDOWN'`, `'MOVING_HOLD'`, `'EXTREME'`, `'JITTER'` |
| `handle_left` | `Vector(2)` | Left Bezier handle position |
| `handle_right` | `Vector(2)` | Right Bezier handle position |
| `handle_left_type` | `str` | Handle type (see table below) |
| `handle_right_type` | `str` | Handle type |
| `select_left_handle` | `bool` | Selection state of left handle |
| `select_right_handle` | `bool` | Selection state of right handle |
| `select_control_point` | `bool` | Selection state of the keyframe |
| `amplitude` | `float` | Amplitude for elastic easing |
| `back` | `float` | Amount for back easing |
| `period` | `float` | Period for elastic easing |

**Interpolation types:**

| Value | Description |
|-------|-------------|
| `'CONSTANT'` | No interpolation (step function) |
| `'LINEAR'` | Linear interpolation |
| `'BEZIER'` | Bezier curve interpolation |
| `'SINE'` | Sinusoidal easing |
| `'QUAD'` | Quadratic easing |
| `'CUBIC'` | Cubic easing |
| `'QUART'` | Quartic easing |
| `'QUINT'` | Quintic easing |
| `'EXPO'` | Exponential easing |
| `'CIRC'` | Circular easing |
| `'BACK'` | Back easing (overshoot) |
| `'BOUNCE'` | Bounce easing |
| `'ELASTIC'` | Elastic easing |

**Handle types:**

| Value | Description |
|-------|-------------|
| `'FREE'` | Fully manual control |
| `'ALIGNED'` | Handles stay aligned |
| `'VECTOR'` | Points directly at adjacent keyframe |
| `'AUTO'` | Automatic smooth interpolation |
| `'AUTO_CLAMPED'` | Automatic, clamped to prevent overshoot |

---

## bpy.types.FCurveModifiers — Blender 3.x/4.x/5.x

### Methods

```python
fcurve.modifiers.new(type: str) -> FModifier
fcurve.modifiers.remove(modifier: FModifier)
```

**Modifier types:**

| Type | Description |
|------|-------------|
| `'GENERATOR'` | Polynomial generator |
| `'FNGENERATOR'` | Built-in function generator (sin, cos, etc.) |
| `'ENVELOPE'` | Envelope modifier |
| `'CYCLES'` | Repeat/mirror animation cycles |
| `'NOISE'` | Add noise variation |
| `'LIMITS'` | Clamp values to min/max range |
| `'STEPPED'` | Stepped interpolation (frame hold) |

---

## bpy.types.Driver — Blender 3.x/4.x/5.x

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `type` | `str` | `'AVERAGE'`, `'SUM'`, `'SCRIPTED'`, `'MIN'`, `'MAX'` |
| `expression` | `str` | Python expression (for `'SCRIPTED'` type) |
| `variables` | `ChannelDriverVariables` | Collection of driver variables |
| `use_self` | `bool` | Allow `self` reference in expression (default: False) |
| `is_valid` | `bool` | True if expression compiles and variables resolve |
| `is_simple_expression` | `bool` | True if expression uses only built-in math |

### ChannelDriverVariables Methods

```python
driver.variables.new() -> DriverVariable
driver.variables.remove(variable: DriverVariable)
```

---

## bpy.types.DriverVariable — Blender 3.x/4.x/5.x

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Variable name (used in expression) |
| `type` | `str` | Variable type (see table below) |
| `targets` | `list[DriverTarget]` | Target references (1 or 2 depending on type) |
| `is_name_valid` | `bool` | True if name is a valid Python identifier |

**Variable types:**

| Type | Targets | Description |
|------|---------|-------------|
| `'SINGLE_PROP'` | 1 | Value of an RNA property |
| `'TRANSFORMS'` | 1 | Transform channel of an object/bone |
| `'ROTATION_DIFF'` | 2 | Rotation difference between two bones |
| `'LOC_DIFF'` | 2 | Distance between two objects/bones |

---

## bpy.types.DriverTarget — Blender 3.x/4.x/5.x

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | `ID` | Target data block (Object, Armature, etc.) |
| `id_type` | `str` | ID type filter (`'OBJECT'`, `'SCENE'`, `'MESH'`, etc.) |
| `data_path` | `str` | RNA path to property (for `SINGLE_PROP`) |
| `bone_target` | `str` | Bone name (for armature targets) |
| `transform_type` | `str` | Transform channel (for `TRANSFORMS` type) |
| `rotation_mode` | `str` | `'AUTO'`, `'XYZ'`, `'XZY'`, `'YXZ'`, `'YZX'`, `'ZXY'`, `'ZYX'`, `'QUATERNION'`, `'SWING_TWIST_X/Y/Z'` |
| `transform_space` | `str` | `'WORLD_SPACE'`, `'TRANSFORM_SPACE'`, `'LOCAL_SPACE'` |

**transform_type values (for TRANSFORMS):**

| Value | Description |
|-------|-------------|
| `'LOC_X'`, `'LOC_Y'`, `'LOC_Z'` | Location channels |
| `'ROT_X'`, `'ROT_Y'`, `'ROT_Z'`, `'ROT_W'` | Rotation channels |
| `'SCALE_X'`, `'SCALE_Y'`, `'SCALE_Z'` | Scale channels |
| `'SCALE_AVG'` | Average scale |

---

## bpy.types.NlaTracks — Blender 3.x/4.x/5.x

### Methods

```python
anim_data.nla_tracks.new() -> NlaTrack
# Creates a new NLA track. prev parameter removed in 4.0+.

anim_data.nla_tracks.remove(track: NlaTrack)
```

---

## bpy.types.NlaTrack — Blender 3.x/4.x/5.x

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Track name |
| `strips` | `NlaStrips` | Collection of NLA strips |
| `is_solo` | `bool` | Solo this track (mutes all others) |
| `mute` | `bool` | Mute this track |
| `lock` | `bool` | Prevent editing |
| `select` | `bool` | Selection state |
| `is_override_data` | `bool` | True if this is library override data |

### NlaStrips Methods

```python
track.strips.new(
    name: str,
    start: int,          # Start frame on NLA timeline
    action: Action       # Action to use
) -> NlaStrip

track.strips.remove(strip: NlaStrip)
```

---

## bpy.types.NlaStrip — Blender 3.x/4.x/5.x

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Strip name |
| `action` | `Action` | Source action |
| `blend_type` | `str` | `'REPLACE'`, `'COMBINE'`, `'ADD'`, `'SUBTRACT'`, `'MULTIPLY'` |
| `frame_start` | `float` | Start frame on NLA timeline |
| `frame_end` | `float` | End frame on NLA timeline |
| `action_frame_start` | `float` | Start frame within the action |
| `action_frame_end` | `float` | End frame within the action |
| `repeat` | `float` | Number of times to repeat |
| `scale` | `float` | Speed multiplier |
| `blend_in` | `float` | Blend in duration (frames) |
| `blend_out` | `float` | Blend out duration (frames) |
| `use_auto_blend` | `bool` | Automatic blend in/out |
| `use_reverse` | `bool` | Play in reverse |
| `mute` | `bool` | Mute this strip |
| `influence` | `float` | Influence amount (0.0 to 1.0) |
| `strip_time` | `float` | Current time within the strip |
| `type` | `str` | `'CLIP'`, `'TRANSITION'`, `'META'`, `'SOUND'` (read-only) |
| `extrapolation` | `str` | `'HOLD'`, `'HOLD_FORWARD'`, `'NOTHING'` |

---

## bpy.types.BoneCollection — Blender 4.0+/5.x

Replaces bone layers (`bone.layers[i]`) and bone groups (`pose.bone_groups`) from Blender 3.x.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Collection name |
| `is_visible` | `bool` | Visibility in viewport (replaces layer visibility) |
| `is_editable` | `bool` | Whether bones in collection are editable |
| `is_local_override` | `bool` | True if this is a library override |
| `bones` | `bpy_prop_collection` of `Bone` | Bones in this collection (read-only) |
| `color_tag` | `str` | Color tag for display (replaces bone_groups color) |

**color_tag values:** `'DEFAULT'`, `'THEME01'` through `'THEME20'`, `'COLOR_01'` through `'COLOR_20'`

### Methods

```python
# Create a new bone collection (Blender 4.0+)
armature.collections.new(name: str) -> BoneCollection

# Remove a bone collection
armature.collections.remove(collection: BoneCollection)

# Assign a bone to the collection
collection.assign(bone: Bone) -> None

# Unassign a bone from the collection
collection.unassign(bone: Bone) -> None
```

### Armature.collections (BoneCollections)

```python
armature.collections.new(name: str) -> BoneCollection
armature.collections.remove(collection: BoneCollection)
armature.collections.get(name: str) -> BoneCollection | None
armature.collections.active  # -> BoneCollection | None (get/set)
armature.collections.active_index  # -> int (get/set)
```

---

## bpy.types.EditBone — Blender 3.x/4.x/5.x

ONLY accessible in Edit Mode (`bpy.ops.object.mode_set(mode='EDIT')`).

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Bone name |
| `head` | `Vector(3)` | Head (root) position in armature space |
| `tail` | `Vector(3)` | Tail (tip) position in armature space |
| `roll` | `float` | Roll angle around bone axis |
| `parent` | `EditBone | None` | Parent bone |
| `use_connect` | `bool` | Connect head to parent's tail |
| `use_deform` | `bool` | Bone deforms geometry |
| `use_local_location` | `bool` | Use local space for location |
| `use_inherit_rotation` | `bool` | Inherit rotation from parent |
| `use_inherit_scale` | `str` | `'FULL'`, `'FIX_SHEAR'`, `'ALIGNED'`, `'NONE'`, etc. |
| `length` | `float` | Distance from head to tail |
| `envelope_distance` | `float` | Envelope deform distance |
| `envelope_weight` | `float` | Envelope deform weight |
| `head_radius` | `float` | B-bone head radius |
| `tail_radius` | `float` | B-bone tail radius |
| `matrix` | `Matrix(4x4)` | Bone matrix in armature space (read-only) |
| `collections` | `bpy_prop_collection` | Bone collections this bone belongs to (4.0+) |

### Armature.edit_bones Methods (Edit Mode only)

```python
armature.edit_bones.new(name: str) -> EditBone
armature.edit_bones.remove(bone: EditBone)
armature.edit_bones.get(name: str) -> EditBone | None
armature.edit_bones.active  # -> EditBone | None (get/set)
```

---

## bpy.types.PoseBone — Blender 3.x/4.x/5.x

Accessible in Object Mode and Pose Mode via `armature_obj.pose.bones`.

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Bone name |
| `location` | `Vector(3)` | Pose-space location offset |
| `rotation_euler` | `Euler(3)` | Euler rotation |
| `rotation_quaternion` | `Quaternion(4)` | Quaternion rotation |
| `rotation_mode` | `str` | `'QUATERNION'`, `'XYZ'`, `'XZY'`, etc. |
| `scale` | `Vector(3)` | Scale |
| `matrix` | `Matrix(4x4)` | Pose-space transform matrix |
| `matrix_basis` | `Matrix(4x4)` | Local transform matrix |
| `bone` | `Bone` | Reference to rest-pose bone data |
| `constraints` | `PoseBoneConstraints` | Constraints on this pose bone |
| `custom_shape` | `Object | None` | Custom display shape |

---

## bpy.types.ShapeKey — Blender 3.x/4.x/5.x

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Shape key name |
| `value` | `float` | Current blend value (0.0 to 1.0 by default) |
| `slider_min` | `float` | Minimum slider value |
| `slider_max` | `float` | Maximum slider value |
| `data` | `bpy_prop_collection` | Per-vertex shape data |
| `mute` | `bool` | Mute this shape key |
| `relative_key` | `ShapeKey` | Reference shape for relative blending |
| `vertex_group` | `str` | Vertex group for limiting effect |

### Key (shape key container) Properties

```python
# Access shape key container
mesh = obj.data
key = mesh.shape_keys  # bpy.types.Key | None

# Access individual shape keys
key.key_blocks  # bpy_prop_collection of ShapeKey
key.key_blocks["Basis"]
key.key_blocks["MyShape"].value = 0.5

# Reference key (basis)
key.reference_key  # -> ShapeKey (the basis)
```

---

## Scene Timeline Properties — Blender 3.x/4.x/5.x

| Property | Type | Description |
|----------|------|-------------|
| `scene.frame_start` | `int` | Animation start frame |
| `scene.frame_end` | `int` | Animation end frame |
| `scene.frame_current` | `int` | Current frame (no depsgraph update) |
| `scene.frame_step` | `int` | Frame step for playback |
| `scene.render.fps` | `int` | Frames per second |
| `scene.render.fps_base` | `float` | FPS base divisor |

### Methods

```python
scene.frame_set(frame: int, subframe: float = 0.0)
# Sets frame AND triggers depsgraph evaluation.
# ALWAYS use frame_set() instead of frame_current when
# subsequent code reads evaluated data.
```

---

## Common Constraint Types — Blender 3.x/4.x/5.x

Added via `obj.constraints.new(type_string)` or `pose_bone.constraints.new(type_string)`.

| Type String | Constraint | Key Properties |
|------------|------------|----------------|
| `'COPY_LOCATION'` | Copy Location | `target`, `use_x/y/z`, `influence` |
| `'COPY_ROTATION'` | Copy Rotation | `target`, `use_x/y/z`, `mix_mode` |
| `'COPY_SCALE'` | Copy Scale | `target`, `use_x/y/z` |
| `'COPY_TRANSFORMS'` | Copy Transforms | `target`, `mix_mode` |
| `'LIMIT_LOCATION'` | Limit Location | `use_min_x/y/z`, `min_x/y/z`, `owner_space` |
| `'LIMIT_ROTATION'` | Limit Rotation | `use_limit_x/y/z`, `min_x/y/z` |
| `'LIMIT_SCALE'` | Limit Scale | `use_min_x/y/z`, `min_x/y/z` |
| `'TRACK_TO'` | Track To | `target`, `track_axis`, `up_axis` |
| `'IK'` | Inverse Kinematics | `target`, `pole_target`, `chain_count`, `iterations` |
| `'CHILD_OF'` | Child Of | `target`, `use_location_x/y/z` |
| `'FLOOR'` | Floor | `target`, `use_rotation`, `offset` |
| `'DAMPED_TRACK'` | Damped Track | `target`, `track_axis` |
| `'LOCKED_TRACK'` | Locked Track | `target`, `track_axis`, `lock_axis` |
| `'STRETCH_TO'` | Stretch To | `target`, `rest_length` |
| `'CLAMP_TO'` | Clamp To | `target` (curve object) |
| `'ACTION'` | Action | `target`, `action`, `frame_start/end` |
