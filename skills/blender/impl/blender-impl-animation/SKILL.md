---
name: blender-impl-animation
description: 'Provides implementation workflows for Blender animation including construction sequence animation for AEC, camera walkthrough automation, solar study animations, phasing visualization, NLA workflow orchestration, and batch keyframe operations. Activates when creating construction animations, camera paths, or time-based visualizations for architecture.'
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-impl-animation

## Quick Reference

### Critical Warnings

**ALWAYS** set the property value BEFORE calling `keyframe_insert()` — keyframes record the current value at call time.

**ALWAYS** call `fcurve.update()` after manually adding or modifying keyframe points via `keyframe_points.add()` or direct `.co` assignment.

**ALWAYS** create `animation_data` before accessing `.action` or `.nla_tracks` — call `obj.animation_data_create()` if `obj.animation_data is None`.

**NEVER** use `bone.layers` or `pose.bone_groups` in Blender 4.0+ — removed. Use `BoneCollection` API.

**NEVER** insert keyframes on objects not linked to any collection — they will exist in `bpy.data` but have no scene presence.

**ALWAYS** call `scene.frame_set(frame)` when you need depsgraph evaluation at that frame (e.g., for visibility keyframes, constraint evaluation). Direct `scene.frame_current = frame` does NOT trigger depsgraph update.

**ALWAYS** use `bpy.data.objects.get("Name")` with None-check instead of direct key access `bpy.data.objects["Name"]` — prevents `KeyError` crashes.

### Decision Tree: AEC Animation Workflow

```
What type of AEC animation do you need?
├─ Construction sequence (4D BIM)?
│  ├─ Objects appear/disappear over time?
│  │  └─ WORKFLOW A: Visibility Phasing
│  └─ Objects move into position (crane sim)?
│     └─ WORKFLOW B: Transform Sequencing
├─ Camera walkthrough / flythrough?
│  ├─ Fixed path (predefined route)?
│  │  └─ WORKFLOW C: Path-Based Camera
│  └─ Point-to-point with smooth transitions?
│     └─ WORKFLOW D: Keyframed Camera
├─ Solar / daylight study?
│  └─ WORKFLOW E: Sun Path Animation
├─ Phasing visualization (design options)?
│  └─ WORKFLOW F: Collection Phasing
└─ Multiple animation clips combined?
   └─ WORKFLOW G: NLA Orchestration
```

### Version Matrix: Implementation Features

| Feature | Blender 3.x | Blender 4.0+ | Blender 5.x |
|---------|-------------|--------------|-------------|
| Visibility/NLA/Path/Constraints | Available | Available | Available |
| `BoneCollection` | Not available | **NEW** | Available |
| `bgl` for overlays | Available | Deprecated | **REMOVED** |

## Workflow A: Visibility Phasing (Construction Sequence)

Use when objects must appear/disappear at specific frames to simulate construction phases.

### Step 1: Organize objects into phase collections

```python
import bpy

# Create phase collections under the scene collection
phase_names = ["Phase_Foundation", "Phase_Structure", "Phase_Envelope", "Phase_Interior"]
phases = {}
for name in phase_names:
    col = bpy.data.collections.new(name)
    bpy.context.scene.collection.children.link(col)
    phases[name] = col
```

### Step 2: Assign objects to phases

```python
# Move objects to phase collections
# ALWAYS unlink from current collection first to avoid duplicates
def assign_to_phase(obj, phase_collection):
    for col in obj.users_collection:
        col.objects.unlink(obj)
    phase_collection.objects.link(obj)
```

### Step 3: Keyframe visibility per phase

```python
def animate_phase_visibility(obj, appear_frame, disappear_frame=None):
    """Keyframe object visibility for construction phasing."""
    # Hidden before appear_frame
    obj.hide_viewport = True
    obj.hide_render = True
    obj.keyframe_insert(data_path="hide_viewport", frame=appear_frame - 1)
    obj.keyframe_insert(data_path="hide_render", frame=appear_frame - 1)

    # Visible at appear_frame
    obj.hide_viewport = False
    obj.hide_render = False
    obj.keyframe_insert(data_path="hide_viewport", frame=appear_frame)
    obj.keyframe_insert(data_path="hide_render", frame=appear_frame)

    if disappear_frame is not None:
        obj.hide_viewport = False
        obj.hide_render = False
        obj.keyframe_insert(data_path="hide_viewport", frame=disappear_frame - 1)
        obj.keyframe_insert(data_path="hide_render", frame=disappear_frame - 1)

        obj.hide_viewport = True
        obj.hide_render = True
        obj.keyframe_insert(data_path="hide_viewport", frame=disappear_frame)
        obj.keyframe_insert(data_path="hide_render", frame=disappear_frame)

    # ALWAYS set CONSTANT interpolation for boolean visibility
    if obj.animation_data and obj.animation_data.action:
        for fc in obj.animation_data.action.fcurves:
            if fc.data_path in ("hide_viewport", "hide_render"):
                for kp in fc.keyframe_points:
                    kp.interpolation = 'CONSTANT'
```

### Step 4: Apply to phase schedule

```python
# Define schedule: {phase_name: (start_frame, end_frame or None)}
schedule = {
    "Phase_Foundation": (1, None),
    "Phase_Structure": (60, None),
    "Phase_Envelope": (120, None),
    "Phase_Interior": (180, None),
}

for phase_name, (start, end) in schedule.items():
    col = bpy.data.collections.get(phase_name)
    if col is None:
        continue
    for obj in col.objects:
        animate_phase_visibility(obj, start, end)

# Set timeline
bpy.context.scene.frame_start = 1
bpy.context.scene.frame_end = 250
```

---

## Workflow B: Transform Sequencing (Crane/Assembly)

Use when objects must animate from off-screen or staging positions to final positions.

```python
import bpy

def animate_assembly(obj, staging_location, final_location,
                     start_frame, end_frame, interpolation='BEZIER'):
    """Animate object from staging to final position."""
    obj.location = staging_location
    obj.keyframe_insert(data_path="location", frame=start_frame)

    obj.location = final_location
    obj.keyframe_insert(data_path="location", frame=end_frame)

    # Set interpolation
    if obj.animation_data and obj.animation_data.action:
        for fc in obj.animation_data.action.fcurves:
            if fc.data_path == "location":
                for kp in fc.keyframe_points:
                    kp.interpolation = interpolation

# Example: Lower beam from above
beam = bpy.data.objects.get("SteelBeam_01")
if beam:
    animate_assembly(
        beam,
        staging_location=(beam.location.x, beam.location.y, beam.location.z + 20),
        final_location=(beam.location.x, beam.location.y, beam.location.z),
        start_frame=60,
        end_frame=90
    )
```

---

## Workflow C: Path-Based Camera Walkthrough

Use for architectural walkthroughs following a predefined curve path.

### Step 1: Create camera and path

```python
import bpy
from mathutils import Vector

# Create camera
cam_data = bpy.data.cameras.new("WalkthroughCam")
cam_data.lens = 24  # Wide angle for architecture
cam_obj = bpy.data.objects.new("WalkthroughCam", cam_data)
bpy.context.scene.collection.objects.link(cam_obj)

# Create path curve (or use existing)
curve_data = bpy.data.curves.new("CameraPath", type='CURVE')
curve_data.dimensions = '3D'
spline = curve_data.splines.new('NURBS')
# Define waypoints
points = [(0, 0, 1.7), (5, 0, 1.7), (5, 10, 1.7), (10, 10, 1.7)]
spline.points.add(count=len(points) - 1)
for i, (x, y, z) in enumerate(points):
    spline.points[i].co = (x, y, z, 1.0)  # NURBS uses 4D (w=1.0)
spline.use_endpoint_u = True
spline.order_u = 3

path_obj = bpy.data.objects.new("CameraPath", curve_data)
bpy.context.scene.collection.objects.link(path_obj)
```

### Step 2: Add Follow Path constraint

```python
# Add Follow Path constraint to camera
follow = cam_obj.constraints.new('FOLLOW_PATH')
follow.target = path_obj
follow.use_curve_follow = True
follow.forward_axis = 'FORWARD_Y'  # Camera looks along Y by default
follow.up_axis = 'UP_Z'

# Animate the path offset
follow.offset = 0.0
follow.keyframe_insert(data_path="offset", frame=1)
follow.offset = -100.0  # Full path traversal
follow.keyframe_insert(data_path="offset", frame=250)

# Set LINEAR interpolation for constant speed
cam_obj_action = cam_obj.animation_data.action
for fc in cam_obj_action.fcurves:
    if "offset" in fc.data_path:
        for kp in fc.keyframe_points:
            kp.interpolation = 'LINEAR'
```

### Step 3: Optional — Track To constraint for look-at target

```python
# Make camera look at a specific point during walkthrough
target = bpy.data.objects.new("CamTarget", None)  # Empty
bpy.context.scene.collection.objects.link(target)
target.location = (5, 5, 3)

track = cam_obj.constraints.new('TRACK_TO')
track.target = target
track.track_axis = 'TRACK_NEGATIVE_Z'
track.up_axis = 'UP_Y'
track.influence = 0.5  # Blend between path direction and target
```

---

## Workflow D: Keyframed Camera (Point-to-Point)

Use for controlled camera transitions between specific viewpoints.

```python
import bpy
from math import radians

def keyframe_camera_viewpoint(cam_obj, location, rotation_euler,
                              frame, interpolation='BEZIER'):
    """Set camera to a viewpoint and keyframe it."""
    cam_obj.location = location
    cam_obj.rotation_euler = rotation_euler
    cam_obj.keyframe_insert(data_path="location", frame=frame)
    cam_obj.keyframe_insert(data_path="rotation_euler", frame=frame)

    if cam_obj.animation_data and cam_obj.animation_data.action:
        for fc in cam_obj.animation_data.action.fcurves:
            for kp in fc.keyframe_points:
                if kp.co[0] == frame:
                    kp.interpolation = interpolation

# Define viewpoints: (location, rotation_euler, frame)
viewpoints = [
    ((20, -20, 10), (radians(60), 0, radians(135)), 1),
    ((5, -15, 5),   (radians(70), 0, radians(160)), 60),
    ((-5, 0, 2),    (radians(85), 0, radians(200)), 120),
    ((0, 10, 15),   (radians(30), 0, radians(250)), 180),
]

cam = bpy.data.objects.get("Camera")
if cam:
    for loc, rot, frame in viewpoints:
        keyframe_camera_viewpoint(cam, loc, rot, frame)
```

---

## Workflow E: Sun Path Animation (Solar Study)

Use for daylight/shadow analysis across time-of-day or seasons.

```python
import bpy
from math import radians, sin, cos

def animate_sun_arc(sun_obj, latitude, start_hour, end_hour,
                    frame_start, frame_end):
    """Animate sun lamp along simplified arc. See references/examples.md for full version."""
    total_frames = frame_end - frame_start
    total_hours = end_hour - start_hour

    for i in range(total_frames + 1):
        frame = frame_start + i
        t = i / total_frames
        hour = start_hour + t * total_hours
        hour_angle = (hour - 12) * 15
        altitude = max(0, min(90, 90 - abs(latitude) + 23.5 * sin(radians(hour_angle))))
        azimuth = hour_angle
        distance = 100
        x = distance * cos(radians(altitude)) * sin(radians(azimuth))
        y = distance * cos(radians(altitude)) * cos(radians(azimuth))
        z = distance * sin(radians(altitude))
        sun_obj.location = (x, y, z)
        sun_obj.keyframe_insert(data_path="location", frame=frame)

    # LINEAR interpolation for smooth arc
    if sun_obj.animation_data and sun_obj.animation_data.action:
        for fc in sun_obj.animation_data.action.fcurves:
            for kp in fc.keyframe_points:
                kp.interpolation = 'LINEAR'

# Setup: create Sun light + Track To constraint pointing at origin
sun_data = bpy.data.lights.new("SunStudy", type='SUN')
sun_data.energy = 5.0
sun_obj = bpy.data.objects.new("SunStudy", sun_data)
bpy.context.scene.collection.objects.link(sun_obj)

track = sun_obj.constraints.new('TRACK_TO')
target = bpy.data.objects.new("SunTarget", None)
bpy.context.scene.collection.objects.link(target)
target.location = (0, 0, 0)
track.target = target
track.track_axis = 'TRACK_NEGATIVE_Z'
track.up_axis = 'UP_Y'

animate_sun_arc(sun_obj, latitude=52.0, start_hour=6, end_hour=18,
                frame_start=1, frame_end=120)
```

---

## Workflow F: Collection Phasing (Design Options)

Use when toggling entire collections to show design alternatives or construction phases.

```python
import bpy

def animate_collection_visibility(collection, visible_start, visible_end=None):
    """Animate collection visibility using exclude flag via view layer.

    NOTE: Collection.hide_viewport can be keyframed directly.
    """
    # Keyframe collection visibility
    collection.hide_viewport = True
    collection.keyframe_insert(data_path="hide_viewport", frame=1)

    collection.hide_viewport = False
    collection.keyframe_insert(data_path="hide_viewport", frame=visible_start)

    if visible_end is not None:
        collection.hide_viewport = False
        collection.keyframe_insert(data_path="hide_viewport", frame=visible_end - 1)
        collection.hide_viewport = True
        collection.keyframe_insert(data_path="hide_viewport", frame=visible_end)

    # CONSTANT interpolation for boolean toggle
    if collection.animation_data and collection.animation_data.action:
        for fc in collection.animation_data.action.fcurves:
            for kp in fc.keyframe_points:
                kp.interpolation = 'CONSTANT'

# Example: Phase schedule
phases = {
    "Demolition": (1, 30),
    "Foundation": (30, 90),
    "Structure": (90, 150),
    "Envelope": (150, 210),
    "Finishes": (210, None),
}

for name, (start, end) in phases.items():
    col = bpy.data.collections.get(name)
    if col is not None:
        animate_collection_visibility(col, start, end)
```

---

## Workflow G: NLA Orchestration

Use when combining multiple animation actions into a sequenced timeline. See [references/examples.md](references/examples.md) for full NLA example.

```python
import bpy

def create_visibility_action(action_name):
    """Create a self-contained visibility action."""
    action = bpy.data.actions.new(name=action_name)
    fc = action.fcurves.new(data_path="hide_viewport")
    fc.keyframe_points.add(count=2)
    fc.keyframe_points[0].co = (0.0, 1.0)   # Hidden
    fc.keyframe_points[1].co = (1.0, 0.0)   # Visible
    for kp in fc.keyframe_points:
        kp.interpolation = 'CONSTANT'
    fc.update()
    return action

def sequence_on_nla(obj, actions_schedule):
    """Place actions on NLA tracks. actions_schedule: list of (action, start_frame, track_name)."""
    anim_data = obj.animation_data if obj.animation_data else obj.animation_data_create()
    for action, start_frame, track_name in actions_schedule:
        track = anim_data.nla_tracks.new()
        track.name = track_name
        strip = track.strips.new(name=action.name, start=start_frame, action=action)
        strip.blend_type = 'REPLACE'
        strip.use_auto_blend = False

# Batch apply: staggered appearance via NLA
collection = bpy.data.collections.get("Structure")
if collection:
    for i, obj in enumerate(collection.objects):
        action = create_visibility_action(f"Appear_{obj.name}")
        sequence_on_nla(obj, [(action, i * 10, f"Phase_{obj.name}")])
```

---

## Batch Keyframe Operations

```python
import bpy

def batch_keyframe_property(objects, data_path, frame, index=-1):
    """Insert keyframes for the current value on multiple objects."""
    for obj in objects:
        try:
            obj.keyframe_insert(data_path=data_path, frame=frame, index=index)
        except TypeError:
            pass  # Property not animatable on this object type

def batch_set_interpolation(objects, interpolation='LINEAR'):
    """Set interpolation type on all keyframes of given objects."""
    for obj in objects:
        if obj.animation_data and obj.animation_data.action:
            for fc in obj.animation_data.action.fcurves:
                for kp in fc.keyframe_points:
                    kp.interpolation = interpolation

def batch_clear_animation(objects):
    """Remove all animation data from objects."""
    for obj in objects:
        if obj.animation_data:
            obj.animation_data_clear()
```

---

## Render Settings for Animation Output

```python
import bpy
scene = bpy.context.scene
scene.frame_start = 1
scene.frame_end = 250
scene.render.filepath = "//render/construction_"  # // = blend-relative
scene.render.image_settings.file_format = 'PNG'   # PNG sequence (resumable)
scene.render.resolution_x = 1920
scene.render.resolution_y = 1080
scene.render.resolution_percentage = 100
scene.render.fps = 24
# For video: file_format='FFMPEG', ffmpeg.format='MPEG4', ffmpeg.codec='H264'
# bpy.ops.render.render(animation=True)
```

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete method signatures for animation workflows
- [references/examples.md](references/examples.md) — Full working examples for AEC animation scenarios
- [references/anti-patterns.md](references/anti-patterns.md) — Common implementation mistakes and corrections

### Official Sources

- https://docs.blender.org/api/current/bpy.types.FCurve.html — FCurve, Action, NlaTrack, NlaStrip
- https://docs.blender.org/api/current/bpy.types.Constraint.html — FollowPath, TrackTo, ChildOf
- https://docs.blender.org/api/current/bpy.types.BoneCollection.html — 4.0+ bone collections
