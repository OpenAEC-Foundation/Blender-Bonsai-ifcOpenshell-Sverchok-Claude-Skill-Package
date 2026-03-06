# blender-impl-animation — Anti-Patterns

## AP-01: Keyframing Without Setting Value First

**WRONG:**
```python
# Value is still the old/default value — keyframe captures wrong state
obj.keyframe_insert(data_path="location", frame=60)
obj.location = (5, 0, 0)  # Too late — keyframe already recorded old location
```

**CORRECT:**
```python
# ALWAYS set value BEFORE inserting keyframe
obj.location = (5, 0, 0)
obj.keyframe_insert(data_path="location", frame=60)
```

**Why it fails**: `keyframe_insert()` records the CURRENT value of the property at call time. Setting the value after the call means the keyframe stores the previous value.

---

## AP-02: Forgetting fcurve.update() After Manual Edits

**WRONG:**
```python
fc = action.fcurves.new(data_path="location", index=0)
fc.keyframe_points.add(count=3)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (30.0, 5.0)
fc.keyframe_points[2].co = (60.0, 0.0)
# Missing update — handles and cache are stale
```

**CORRECT:**
```python
fc = action.fcurves.new(data_path="location", index=0)
fc.keyframe_points.add(count=3)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (30.0, 5.0)
fc.keyframe_points[2].co = (60.0, 0.0)
fc.update()  # ALWAYS call after manual changes
```

**Why it fails**: Blender caches FCurve data for performance. Without `update()`, Bezier handles are not recalculated and evaluation may return incorrect values.

---

## AP-03: Using LINEAR/BEZIER Interpolation for Boolean Visibility

**WRONG:**
```python
obj.hide_viewport = True
obj.keyframe_insert(data_path="hide_viewport", frame=1)
obj.hide_viewport = False
obj.keyframe_insert(data_path="hide_viewport", frame=30)
# Default BEZIER interpolation: object gradually fades between 0 and 1
# At frame 15 the value is ~0.5, which Blender treats as True (visible)
# Results in unpredictable hide/show behavior
```

**CORRECT:**
```python
obj.hide_viewport = True
obj.keyframe_insert(data_path="hide_viewport", frame=1)
obj.hide_viewport = False
obj.keyframe_insert(data_path="hide_viewport", frame=30)

# ALWAYS use CONSTANT interpolation for boolean properties
if obj.animation_data and obj.animation_data.action:
    for fc in obj.animation_data.action.fcurves:
        if fc.data_path == "hide_viewport":
            for kp in fc.keyframe_points:
                kp.interpolation = 'CONSTANT'
```

**Why it fails**: Boolean properties (`hide_viewport`, `hide_render`) are stored as floats internally. With BEZIER or LINEAR interpolation, intermediate frames get fractional values (0.5), causing unpredictable visibility toggling. CONSTANT ensures instant on/off switching.

---

## AP-04: Accessing animation_data Without Creating It

**WRONG:**
```python
obj = bpy.data.objects.get("Cube")
action = obj.animation_data.action  # AttributeError if animation_data is None
track = obj.animation_data.nla_tracks.new()  # Same crash
```

**CORRECT:**
```python
obj = bpy.data.objects.get("Cube")
anim_data = obj.animation_data
if anim_data is None:
    anim_data = obj.animation_data_create()

action = anim_data.action  # Safe
track = anim_data.nla_tracks.new()  # Safe
```

**Why it fails**: Fresh objects have `animation_data == None`. Accessing `.action` or `.nla_tracks` on `None` raises `AttributeError`. ALWAYS check and create first.

---

## AP-05: Using frame_current Instead of frame_set()

**WRONG:**
```python
# Setting frame without depsgraph update
scene.frame_current = 50
# Constraints, drivers, and modifiers are NOT evaluated at frame 50
value = obj.matrix_world  # Still at previous frame's state
```

**CORRECT:**
```python
# frame_set() triggers full depsgraph evaluation
scene.frame_set(50)
# Now all constraints, drivers, modifiers are evaluated
value = obj.matrix_world  # Correct for frame 50
```

**Why it fails**: `frame_current` is a direct property assignment — it changes the frame counter but does NOT trigger the dependency graph to re-evaluate. `frame_set()` triggers full evaluation, ensuring constraints, drivers, and modifiers reflect the new frame.

---

## AP-06: Direct Object Key Access Without None-Check

**WRONG:**
```python
obj = bpy.data.objects["MyObject"]  # KeyError if not found
cam = bpy.data.objects["Camera"]    # KeyError if renamed/deleted
```

**CORRECT:**
```python
obj = bpy.data.objects.get("MyObject")
if obj is None:
    raise RuntimeError("Object 'MyObject' not found in scene")

cam = bpy.data.objects.get("Camera")
if cam is None:
    raise RuntimeError("Camera not found in scene")
```

**Why it fails**: Direct dictionary-style access raises `KeyError` if the name doesn't match exactly. Objects may be renamed, deleted, or not yet created. `get()` returns `None` safely.

---

## AP-07: Animating Objects Not Linked to Any Collection

**WRONG:**
```python
# Object created but not linked to scene
mesh = bpy.data.meshes.new("TempMesh")
obj = bpy.data.objects.new("TempObj", mesh)
# Forgot to link: bpy.context.scene.collection.objects.link(obj)

obj.location = (1, 2, 3)
obj.keyframe_insert(data_path="location", frame=1)
# Keyframe is inserted but object is invisible and not rendered
```

**CORRECT:**
```python
mesh = bpy.data.meshes.new("TempMesh")
obj = bpy.data.objects.new("TempObj", mesh)
bpy.context.scene.collection.objects.link(obj)  # ALWAYS link first

obj.location = (1, 2, 3)
obj.keyframe_insert(data_path="location", frame=1)
```

**Why it fails**: Objects in `bpy.data.objects` but not linked to any collection exist in memory but have no scene presence. Animation works technically, but the object is never visible in viewport or render.

---

## AP-08: Using bone.layers in Blender 4.0+

**WRONG (Blender 4.0+):**
```python
# REMOVED in 4.0 — will raise AttributeError
bone.layers[0] = True
bone.layers[16] = True
pose.bone_groups.new(name="Controls")
```

**CORRECT (Blender 4.0+):**
```python
# Use BoneCollection API
col = armature_data.collections.new("Controls")
col.assign(armature_data.bones["BoneName"])
col.is_visible = True
col.color_tag = 'THEME01'
```

**Why it fails**: Blender 4.0 replaced the 32-layer bone system and bone groups with `BoneCollection`. The old API attributes (`bone.layers`, `pose.bone_groups`) are completely removed and raise `AttributeError`.

---

## AP-09: Using NLA Strips Without Clearing Active Action

**WRONG:**
```python
# Active action overrides NLA strips
anim_data = obj.animation_data_create()
anim_data.action = my_action  # Active action set

track = anim_data.nla_tracks.new()
strip = track.strips.new("MyStrip", start=1, action=another_action)
# NLA strip is ignored because active action takes priority
```

**CORRECT:**
```python
anim_data = obj.animation_data_create()

# Push action to NLA first, then clear active action
track = anim_data.nla_tracks.new()
strip = track.strips.new("MyStrip", start=1, action=my_action)

# Clear active action so NLA strips take effect
anim_data.action = None
```

**Why it fails**: When `animation_data.action` is set (active action), it overrides ALL NLA strips for the same properties. NLA strips only take effect when there is no active action, or when `anim_data.use_nla` is True and the active action is pushed to NLA.

---

## AP-10: Creating NURBS Path With Wrong Point Count

**WRONG:**
```python
spline = curve_data.splines.new('NURBS')
# NURBS spline starts with 1 point
# Trying to set 5 points but only added 3
spline.points.add(count=3)  # Now has 4 points total (1 default + 3 added)
# Trying to set index 4 — IndexError
spline.points[4].co = (10, 10, 0, 1)
```

**CORRECT:**
```python
spline = curve_data.splines.new('NURBS')
waypoints = [(0, 0, 0), (5, 0, 0), (5, 5, 0), (10, 5, 0), (10, 10, 0)]
# Add count-1 because spline starts with 1 point
spline.points.add(count=len(waypoints) - 1)
for i, (x, y, z) in enumerate(waypoints):
    spline.points[i].co = (x, y, z, 1.0)  # NURBS uses 4D: (x, y, z, weight)
```

**Why it fails**: New NURBS splines start with 1 control point. `points.add(count=N)` adds N more, giving N+1 total. Off-by-one errors are common. Also, NURBS points require 4D coordinates `(x, y, z, w)` where `w` is the weight (typically 1.0).

---

## AP-11: Setting Constraint Influence to 0 Instead of Keyframing It

**WRONG:**
```python
# Want camera to smoothly transition from path-following to free movement
follow = cam_obj.constraints["Follow Path"]
follow.influence = 0.0  # Instant switch — no transition
```

**CORRECT:**
```python
# Keyframe influence for smooth transition
follow = cam_obj.constraints["Follow Path"]
follow.influence = 1.0
follow.keyframe_insert(data_path="influence", frame=100)
follow.influence = 0.0
follow.keyframe_insert(data_path="influence", frame=120)
```

**Why it fails**: Setting influence directly is an instant change. For smooth animation transitions, keyframe the `influence` property so it interpolates between values over time.

---

## AP-12: Rendering Animation Without Setting Output Path

**WRONG:**
```python
bpy.ops.render.render(animation=True)
# Default output path is /tmp/ — frames may be overwritten or lost
```

**CORRECT:**
```python
scene = bpy.context.scene
scene.render.filepath = "//render/my_animation_"  # // = blend-relative
scene.render.image_settings.file_format = 'PNG'
bpy.ops.render.render(animation=True)
```

**Why it fails**: Without explicitly setting `filepath`, Blender uses a system temp directory. Frames may be overwritten by other renders or lost on restart. ALWAYS set a project-relative output path before rendering.

---

## AP-13: Using threading for Animation Playback Control

**WRONG:**
```python
import threading
import bpy

def advance_frame():
    bpy.context.scene.frame_current += 1  # Crash — bpy on non-main thread

thread = threading.Thread(target=advance_frame)
thread.start()
```

**CORRECT:**
```python
import bpy

def advance_frame():
    bpy.context.scene.frame_set(bpy.context.scene.frame_current + 1)
    return 1.0 / 24  # Return interval to call again (24fps)

# Use Blender's timer system for deferred execution
bpy.app.timers.register(advance_frame)
```

**Why it fails**: ALL `bpy` operations MUST run on the main thread. Blender's embedded Python is not thread-safe. Using `threading` for `bpy` calls causes crashes. Use `bpy.app.timers.register()` for deferred/repeated execution.

---

## Summary Table

| # | Anti-Pattern | Severity | Fix |
|---|-------------|----------|-----|
| AP-01 | Keyframe before setting value | Critical | Set value THEN keyframe_insert() |
| AP-02 | Missing fcurve.update() | Critical | ALWAYS call update() after manual edits |
| AP-03 | Non-CONSTANT interpolation on booleans | High | Use CONSTANT for hide_viewport/hide_render |
| AP-04 | animation_data None access | Critical | Check and create with animation_data_create() |
| AP-05 | frame_current vs frame_set() | High | Use frame_set() when evaluation needed |
| AP-06 | Direct key access without None-check | Medium | Use .get() with None check |
| AP-07 | Animating unlinked objects | Medium | Link to collection before animating |
| AP-08 | bone.layers in 4.0+ | Critical | Use BoneCollection API |
| AP-09 | NLA ignored by active action | High | Clear active action or push to NLA |
| AP-10 | NURBS point count off-by-one | Medium | Account for default point; use 4D coords |
| AP-11 | Instant constraint influence change | Medium | Keyframe influence for transitions |
| AP-12 | No output path for render | Medium | Set filepath before render |
| AP-13 | Threading with bpy | Critical | Use bpy.app.timers.register() |
