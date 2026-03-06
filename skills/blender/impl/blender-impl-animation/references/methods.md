# blender-impl-animation — Method Reference

## Keyframe Insertion

### `bpy_struct.keyframe_insert()`

Insert a keyframe on the property at the current value.

```python
obj.keyframe_insert(
    data_path: str,         # RNA path: "location", "rotation_euler", "scale", "hide_viewport"
    index: int = -1,        # Array index: -1=all, 0=X, 1=Y, 2=Z
    frame: float = current, # Frame number (defaults to current frame)
    group: str = "",        # Action group name
    options: set = set()    # {'INSERTKEY_NEEDED', 'INSERTKEY_VISUAL', 'INSERTKEY_XYZ_TO_RGB',
                            #  'INSERTKEY_REPLACE', 'INSERTKEY_AVAILABLE', 'INSERTKEY_CYCLE_AWARE'}
) -> bool                   # True if keyframe was inserted
```

**CRITICAL**: Set the property value BEFORE calling this method.

### `bpy_struct.keyframe_delete()`

```python
obj.keyframe_delete(
    data_path: str,
    index: int = -1,
    frame: float = current,
    group: str = ""
) -> bool
```

---

## FCurve Operations

### `Action.fcurves.new()`

```python
action.fcurves.new(
    data_path: str,     # RNA path
    index: int = 0,     # Array index
    action_group: str = "" # Group name
) -> FCurve
```

### `Action.fcurves.find()`

```python
action.fcurves.find(
    data_path: str,
    index: int = 0
) -> FCurve | None
```

### `FCurve.keyframe_points.add()`

```python
fcurve.keyframe_points.add(
    count: int  # Number of keyframe points to add
)
# After adding, set .co = (frame, value) for each point
# ALWAYS call fcurve.update() after manual changes
```

### `FCurve.keyframe_points.insert()`

```python
fcurve.keyframe_points.insert(
    frame: float,
    value: float,
    options: set = set(),     # {'REPLACE', 'NEEDED', 'FAST'}
    keyframe_type: str = 'KEYFRAME'  # 'KEYFRAME', 'BREAKDOWN', 'MOVING_HOLD',
                                      # 'EXTREME', 'JITTER'
) -> Keyframe
```

**Note**: `insert()` creates one keyframe at a time. For bulk operations, use `add()` + manual `.co` assignment + `update()`.

### `FCurve.evaluate()`

```python
fcurve.evaluate(
    frame: float
) -> float  # Interpolated value at given frame
```

### `FCurve.range()`

```python
fcurve.range() -> tuple[float, float]  # (min_frame, max_frame)
```

### `FCurve.update()`

```python
fcurve.update()  # Recalculate curve handles and cache. MUST call after manual edits.
```

### `FCurve.convert_to_samples()`

```python
fcurve.convert_to_samples(
    start: int,
    end: int
)
```

### `FCurve.convert_to_keyframes()`

```python
fcurve.convert_to_keyframes(
    start: int,
    end: int
)
```

---

## FCurve Modifiers

### `FCurve.modifiers.new()`

```python
fcurve.modifiers.new(
    type: str  # 'NULL', 'GENERATOR', 'FNGENERATOR', 'ENVELOPE',
               # 'CYCLES', 'NOISE', 'LIMITS', 'STEPPED'
) -> FModifier
```

**Cycles Modifier Properties:**

| Property | Type | Values |
|----------|------|--------|
| `mode_before` | `str` | `'NONE'`, `'REPEAT'`, `'REPEAT_OFFSET'`, `'MIRROR'` |
| `mode_after` | `str` | `'NONE'`, `'REPEAT'`, `'REPEAT_OFFSET'`, `'MIRROR'` |
| `cycles_before` | `int` | Number of cycles (0 = infinite) |
| `cycles_after` | `int` | Number of cycles (0 = infinite) |

**Noise Modifier Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `scale` | `float` | Scale of the noise |
| `strength` | `float` | Amplitude of the noise |
| `phase` | `float` | Phase offset |
| `offset` | `float` | Value offset |
| `depth` | `int` | Fractal noise depth |

---

## Animation Data

### `ID.animation_data_create()`

```python
obj.animation_data_create() -> AnimData
# Creates AnimData block if not present. ALWAYS check before accessing .action or .nla_tracks.
```

### `ID.animation_data_clear()`

```python
obj.animation_data_clear()
# Removes all animation data (action, NLA tracks, drivers) from the object.
```

---

## NLA System

### `AnimData.nla_tracks.new()`

```python
anim_data.nla_tracks.new() -> NlaTrack
# prev parameter removed in recent Blender versions
```

### `NlaTrack.strips.new()`

```python
track.strips.new(
    name: str,
    start: int,         # Start frame on NLA timeline
    action: Action      # Action data block to use
) -> NlaStrip
```

### NlaStrip Properties

| Property | Type | Description |
|----------|------|-------------|
| `blend_type` | `str` | `'REPLACE'`, `'COMBINE'`, `'ADD'`, `'SUBTRACT'`, `'MULTIPLY'` |
| `use_auto_blend` | `bool` | Auto-blend with adjacent strips |
| `blend_in` | `float` | Blend-in duration in frames |
| `blend_out` | `float` | Blend-out duration in frames |
| `repeat` | `float` | Number of times to loop the action |
| `scale` | `float` | Speed scaling (1.0 = normal) |
| `action_frame_start` | `float` | Start frame within the action |
| `action_frame_end` | `float` | End frame within the action |
| `frame_start` | `float` | Start frame on the NLA timeline (read-only after creation) |
| `frame_end` | `float` | End frame on the NLA timeline |
| `mute` | `bool` | Disable this strip |
| `influence` | `float` | 0.0–1.0 blending influence |
| `strip_time` | `float` | Current time position within strip |
| `use_animated_influence` | `bool` | Enable keyframing of influence |
| `use_animated_time` | `bool` | Enable keyframing of strip time |

---

## Constraints (Animation-Related)

### Follow Path

```python
constraint = obj.constraints.new('FOLLOW_PATH')
```

| Property | Type | Description |
|----------|------|-------------|
| `target` | `Object` | Curve object to follow |
| `use_curve_follow` | `bool` | Rotate to follow curve tangent |
| `forward_axis` | `str` | `'FORWARD_X'`, `'FORWARD_Y'`, `'FORWARD_Z'`, `'TRACK_NEGATIVE_X'`, etc. |
| `up_axis` | `str` | `'UP_X'`, `'UP_Y'`, `'UP_Z'` |
| `offset` | `float` | Offset along path (0–100 or keyframeable) |
| `use_fixed_location` | `bool` | Use fixed position along spline |
| `offset_factor` | `float` | Fixed position factor (0.0–1.0) when use_fixed_location=True |

### Track To

```python
constraint = obj.constraints.new('TRACK_TO')
```

| Property | Type | Description |
|----------|------|-------------|
| `target` | `Object` | Object to track |
| `track_axis` | `str` | `'TRACK_X'`, `'TRACK_Y'`, `'TRACK_Z'`, `'TRACK_NEGATIVE_X'`, `'TRACK_NEGATIVE_Y'`, `'TRACK_NEGATIVE_Z'` |
| `up_axis` | `str` | `'UP_X'`, `'UP_Y'`, `'UP_Z'` |
| `influence` | `float` | 0.0–1.0 |

### Child Of

```python
constraint = obj.constraints.new('CHILD_OF')
```

| Property | Type | Description |
|----------|------|-------------|
| `target` | `Object` | Parent object |
| `use_location_x/y/z` | `bool` | Which location axes to follow |
| `use_rotation_x/y/z` | `bool` | Which rotation axes to follow |
| `use_scale_x/y/z` | `bool` | Which scale axes to follow |
| `influence` | `float` | 0.0–1.0 (keyframeable for switchable parenting) |

---

## Scene Timeline

### `Scene.frame_set()`

```python
scene.frame_set(
    frame: int,
    subframe: float = 0.0
)
# Updates depsgraph. Use this when evaluation at the new frame is needed.
```

### Timeline Properties

| Property | Type | Description |
|----------|------|-------------|
| `scene.frame_start` | `int` | First frame of animation range |
| `scene.frame_end` | `int` | Last frame of animation range |
| `scene.frame_current` | `int` | Current frame (no depsgraph update on assignment) |
| `scene.frame_step` | `int` | Frame step for playback/render |
| `scene.render.fps` | `int` | Frames per second |
| `scene.render.fps_base` | `float` | FPS base divisor (actual fps = fps / fps_base) |

---

## Render Animation

### `bpy.ops.render.render()`

```python
bpy.ops.render.render(
    animation: bool = False,    # True to render full animation
    write_still: bool = False,  # True to save single frame
    scene: str = "",            # Scene name (default: active)
)
```

### Render Output Properties

| Property | Type | Description |
|----------|------|-------------|
| `scene.render.filepath` | `str` | Output path (use `//` prefix for blend-relative) |
| `scene.render.image_settings.file_format` | `str` | `'PNG'`, `'JPEG'`, `'OPEN_EXR'`, `'FFMPEG'` |
| `scene.render.resolution_x` | `int` | Horizontal resolution |
| `scene.render.resolution_y` | `int` | Vertical resolution |
| `scene.render.resolution_percentage` | `int` | Scale factor (100 = full res) |

### FFMPEG Properties (when `file_format='FFMPEG'`)

| Property | Type | Description |
|----------|------|-------------|
| `scene.render.ffmpeg.format` | `str` | `'MPEG4'`, `'AVI'`, `'MKV'`, `'WEBM'` |
| `scene.render.ffmpeg.codec` | `str` | `'H264'`, `'MPEG4'`, `'WEBM'`, `'AV1'`, `'NONE'` |
| `scene.render.ffmpeg.constant_rate_factor` | `str` | `'NONE'`, `'LOSSLESS'`, `'PERC_LOSSLESS'`, `'HIGH'`, `'MEDIUM'`, `'LOW'`, `'VERYLOW'`, `'LOWEST'` |
| `scene.render.ffmpeg.audio_codec` | `str` | `'NONE'`, `'AAC'`, `'AC3'`, `'FLAC'`, `'MP2'`, `'MP3'`, `'OPUS'`, `'VORBIS'` |

---

## Keyframe Point Properties

| Property | Type | Description |
|----------|------|-------------|
| `kp.co` | `Vector2D` | `(frame, value)` |
| `kp.interpolation` | `str` | `'CONSTANT'`, `'LINEAR'`, `'BEZIER'`, `'SINE'`, `'QUAD'`, `'CUBIC'`, `'QUART'`, `'QUINT'`, `'EXPO'`, `'CIRC'`, `'BACK'`, `'BOUNCE'`, `'ELASTIC'` |
| `kp.easing` | `str` | `'AUTO'`, `'EASE_IN'`, `'EASE_OUT'`, `'EASE_IN_OUT'` |
| `kp.handle_left_type` | `str` | `'FREE'`, `'ALIGNED'`, `'VECTOR'`, `'AUTO'`, `'AUTO_CLAMPED'` |
| `kp.handle_right_type` | `str` | Same as `handle_left_type` |
| `kp.handle_left` | `Vector2D` | Left Bezier handle position |
| `kp.handle_right` | `Vector2D` | Right Bezier handle position |
| `kp.type` | `str` | `'KEYFRAME'`, `'BREAKDOWN'`, `'MOVING_HOLD'`, `'EXTREME'`, `'JITTER'` |

---

## Collection Visibility

### `Collection.hide_viewport`

```python
collection.hide_viewport: bool  # Toggle viewport visibility. Keyframeable.
```

### `Collection.hide_render`

```python
collection.hide_render: bool  # Toggle render visibility. Keyframeable.
```

### Object Visibility

```python
obj.hide_viewport: bool   # Viewport visibility. Keyframeable.
obj.hide_render: bool      # Render visibility. Keyframeable.
obj.hide_set(state: bool)  # Set viewport hide state (Blender 4.0+ preferred for UI-driven hide)
obj.hide_get() -> bool     # Get viewport hide state
```
