# blender-syntax-animation: Anti-Patterns

Common mistakes when working with Blender animation APIs. Each entry explains WHAT is wrong and WHY it fails.

---

## Anti-Pattern 1: Inserting Keyframe Before Setting Value

```python
# WRONG — keyframe records the OLD value, not the intended one
obj.keyframe_insert(data_path="location", frame=60)
obj.location = (5.0, 0.0, 3.0)
```

**WHY:** `keyframe_insert()` captures the property's current value at call time. If you insert the keyframe before assigning the new value, the keyframe stores the previous value. The new assignment has no effect on the already-recorded keyframe.

```python
# CORRECT — set value FIRST, then insert keyframe
obj.location = (5.0, 0.0, 3.0)
obj.keyframe_insert(data_path="location", frame=60)
```

---

## Anti-Pattern 2: Skipping `fcurve.update()` After Manual Keyframe Manipulation

```python
# WRONG — keyframes may be unsorted, handles wrong, evaluation broken
fcurve.keyframe_points.add(count=10)
for i in range(10):
    fcurve.keyframe_points[i].co = (float(i * 10), float(i))
# Missing fcurve.update() call
```

**WHY:** When keyframe points are added via `keyframe_points.add()` or modified via direct `.co` assignment, Blender does not automatically re-sort keyframes or recalculate Bezier handles. Without `fcurve.update()`, the FCurve data is internally inconsistent, leading to incorrect interpolation, visual glitches, or crashes.

```python
# CORRECT — always call update() after manual FCurve changes
fcurve.keyframe_points.add(count=10)
for i in range(10):
    fcurve.keyframe_points[i].co = (float(i * 10), float(i))
fcurve.update()
```

---

## Anti-Pattern 3: Accessing `edit_bones` Outside of Edit Mode

```python
# WRONG — edit_bones is empty/inaccessible outside Edit Mode
armature = bpy.data.armatures["MyArmature"]
bone = armature.edit_bones.get("Root")  # Returns None
bone.head = (0, 0, 0)                   # AttributeError
```

**WHY:** `armature.edit_bones` is only populated when the armature object is in Edit Mode. In Object Mode or Pose Mode, the collection is empty. Blender uses three separate bone representations (`edit_bones`, `bones`, `pose.bones`) that are mode-dependent. Attempting to read or write `edit_bones` without first entering Edit Mode silently returns `None` on `.get()` or raises an error on direct access.

```python
# CORRECT — enter Edit Mode before accessing edit_bones
bpy.context.view_layer.objects.active = armature_obj
bpy.ops.object.mode_set(mode='EDIT')
bone = armature.edit_bones.get("Root")
bone.head = (0, 0, 0)
bpy.ops.object.mode_set(mode='OBJECT')
```

---

## Anti-Pattern 4: Using `bone.layers` or `pose.bone_groups` in Blender 4.0+

```python
# WRONG — REMOVED in Blender 4.0+
bone.layers[0] = True
bone.layers[16] = True
pose_group = armature_obj.pose.bone_groups.new(name="Controls")
pose_group.color_set = 'THEME01'
```

**WHY:** Blender 4.0 replaced the 32-layer bone system and `pose.bone_groups` with `BoneCollection`. The `bone.layers` array and `pose.bone_groups` collection no longer exist in 4.0+. Code using these attributes raises `AttributeError` at runtime. There is no backward-compatible shim.

```python
# CORRECT — use BoneCollection (Blender 4.0+)
col = armature_data.collections.new("Controls")
col.color_tag = 'THEME01'
col.assign(armature_data.bones["Root"])
col.is_visible = True
```

---

## Anti-Pattern 5: Using `self` in Driver Expression Without `use_self = True`

```python
# WRONG — 'self' is undefined without enabling use_self
driver_fc = obj.driver_add("location", 1)
driver = driver_fc.driver
driver.type = 'SCRIPTED'
driver.expression = "self.location.x * 0.5"  # NameError: 'self' not defined
```

**WHY:** By default, Blender driver expressions run in a restricted namespace where `self` is not available. The `self` keyword only becomes available when `driver.use_self = True` is explicitly set. Without it, Blender cannot resolve `self` and the driver reports an error, evaluating to 0.0 silently or displaying a red highlight in the UI.

```python
# CORRECT — enable use_self before referencing self in expression
driver_fc = obj.driver_add("location", 1)
driver = driver_fc.driver
driver.type = 'SCRIPTED'
driver.use_self = True
driver.expression = "self.location.x * 0.5"
```

---

## Anti-Pattern 6: Accessing `.action` or `.nla_tracks` Without Creating `animation_data`

```python
# WRONG — animation_data is None by default
obj = bpy.data.objects["Cube"]
action = obj.animation_data.action  # AttributeError: NoneType has no attribute 'action'
```

**WHY:** Most Blender objects start without `animation_data`. The `.animation_data` property returns `None` until explicitly created via `obj.animation_data_create()`. Attempting to access `.action`, `.nla_tracks`, or `.drivers` on `None` raises an `AttributeError`.

```python
# CORRECT — create animation_data if it does not exist
obj = bpy.data.objects["Cube"]
if obj.animation_data is None:
    obj.animation_data_create()
action = obj.animation_data.action  # Safe: returns None or Action
```

---

## Anti-Pattern 7: Using `scene.frame_current` When Evaluated Data Is Needed

```python
# WRONG — frame_current does NOT trigger depsgraph evaluation
scene.frame_current = 50
pos = obj.matrix_world.translation  # Still at old frame's position
```

**WHY:** Assigning `scene.frame_current` updates the frame number but does NOT trigger a dependency graph evaluation. Any code reading evaluated properties (like `matrix_world`, constraint results, or shape key values) will get stale data from the previous evaluation. Use `scene.frame_set()` instead, which both sets the frame and triggers a full depsgraph update.

```python
# CORRECT — use frame_set() to trigger depsgraph evaluation
scene.frame_set(50)
pos = obj.matrix_world.translation  # Correct: reads updated position
```

---

## Anti-Pattern 8: Creating NLA Strip Without Clearing Active Action

```python
# WRONG — active action overrides NLA evaluation
track = anim_data.nla_tracks.new()
strip = track.strips.new(name="Walk", start=1, action=walk_action)
# anim_data.action is still set — NLA strips are ignored
```

**WHY:** When `anim_data.action` is set, Blender evaluates it as the "active action" and ignores NLA track evaluation unless tweak mode is active. NLA strips only take effect when `anim_data.action` is `None` (or the strip is being tweaked). Forgetting to clear the active action after pushing strips means the NLA stack has no influence on the animation.

```python
# CORRECT — clear active action so NLA tracks take effect
track = anim_data.nla_tracks.new()
strip = track.strips.new(name="Walk", start=1, action=walk_action)
anim_data.action = None  # NLA tracks now evaluate
```
