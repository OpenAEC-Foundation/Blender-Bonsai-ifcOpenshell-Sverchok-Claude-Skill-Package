# blender-impl-animation — Working Examples

## Example 1: Complete Construction Sequence (4D BIM Simulation)

Animates a multi-phase construction schedule where structural elements appear in order.

```python
# Blender 3.x/4.x/5.x — Full 4D construction sequence
import bpy

def clear_scene():
    """Remove default objects."""
    bpy.ops.object.select_all(action='SELECT')
    bpy.ops.object.delete()

def create_building_element(name, location, dimensions, collection_name):
    """Create a simple box representing a building element."""
    bpy.ops.mesh.primitive_cube_add(size=1, location=location)
    obj = bpy.context.active_object
    obj.name = name
    obj.scale = dimensions

    # Move to target collection
    target_col = bpy.data.collections.get(collection_name)
    if target_col is None:
        target_col = bpy.data.collections.new(collection_name)
        bpy.context.scene.collection.children.link(target_col)

    for col in obj.users_collection:
        col.objects.unlink(obj)
    target_col.objects.link(obj)
    return obj

def animate_construction_phase(obj, appear_frame):
    """Make object appear at specified frame with CONSTANT interpolation."""
    # Hidden state
    obj.hide_viewport = True
    obj.hide_render = True
    obj.keyframe_insert(data_path="hide_viewport", frame=1)
    obj.keyframe_insert(data_path="hide_render", frame=1)

    # Visible state
    obj.hide_viewport = False
    obj.hide_render = False
    obj.keyframe_insert(data_path="hide_viewport", frame=appear_frame)
    obj.keyframe_insert(data_path="hide_render", frame=appear_frame)

    # Set CONSTANT interpolation — critical for boolean visibility
    if obj.animation_data and obj.animation_data.action:
        for fc in obj.animation_data.action.fcurves:
            if fc.data_path in ("hide_viewport", "hide_render"):
                for kp in fc.keyframe_points:
                    kp.interpolation = 'CONSTANT'

# Build schedule
clear_scene()

# Phase 1: Foundation (frames 1-30)
slab = create_building_element("Foundation_Slab", (0, 0, 0.15), (10, 10, 0.3), "Phase_Foundation")
animate_construction_phase(slab, appear_frame=1)

footing_positions = [(-4, -4, 0.5), (4, -4, 0.5), (-4, 4, 0.5), (4, 4, 0.5)]
for i, pos in enumerate(footing_positions):
    footing = create_building_element(f"Footing_{i:02d}", pos, (1.5, 1.5, 1.0), "Phase_Foundation")
    animate_construction_phase(footing, appear_frame=10 + i * 5)

# Phase 2: Structure (frames 31-90)
column_positions = [(-4, -4, 3), (4, -4, 3), (-4, 4, 3), (4, 4, 3)]
for i, pos in enumerate(column_positions):
    col = create_building_element(f"Column_{i:02d}", pos, (0.4, 0.4, 5), "Phase_Structure")
    animate_construction_phase(col, appear_frame=35 + i * 8)

beam = create_building_element("Beam_01", (0, 0, 5.5), (10, 0.3, 0.5), "Phase_Structure")
animate_construction_phase(beam, appear_frame=70)

# Phase 3: Envelope (frames 91-150)
wall_data = [
    ("Wall_North", (0, 5, 3), (10, 0.2, 5)),
    ("Wall_South", (0, -5, 3), (10, 0.2, 5)),
    ("Wall_East", (5, 0, 3), (0.2, 10, 5)),
    ("Wall_West", (-5, 0, 3), (0.2, 10, 5)),
]
for i, (name, pos, dims) in enumerate(wall_data):
    wall = create_building_element(name, pos, dims, "Phase_Envelope")
    animate_construction_phase(wall, appear_frame=95 + i * 12)

# Phase 4: Roof (frames 151-180)
roof = create_building_element("Roof_Slab", (0, 0, 5.65), (10.5, 10.5, 0.3), "Phase_Roof")
animate_construction_phase(roof, appear_frame=155)

# Timeline settings
scene = bpy.context.scene
scene.frame_start = 1
scene.frame_end = 200
scene.frame_set(1)
```

---

## Example 2: Architectural Camera Walkthrough

Camera follows a NURBS path through a building interior.

```python
# Blender 3.x/4.x/5.x — Camera walkthrough with path constraint
import bpy
from mathutils import Vector

# Create camera
cam_data = bpy.data.cameras.new("WalkthroughCam")
cam_data.lens = 18          # Ultra-wide for interiors
cam_data.clip_start = 0.1
cam_data.clip_end = 1000

cam_obj = bpy.data.objects.new("WalkthroughCam", cam_data)
bpy.context.scene.collection.objects.link(cam_obj)

# Create walkthrough path
curve_data = bpy.data.curves.new("WalkthroughPath", type='CURVE')
curve_data.dimensions = '3D'
curve_data.resolution_u = 64  # Smooth path

spline = curve_data.splines.new('NURBS')
# Waypoints at eye height (1.7m)
waypoints = [
    (0, -15, 1.7),    # Approach from exterior
    (0, -5, 1.7),     # Entrance
    (0, 0, 1.7),      # Lobby
    (3, 5, 1.7),      # Turn right
    (3, 10, 1.7),     # Hallway
    (0, 15, 1.7),     # End of hall
    (0, 15, 5.0),     # Rise to overview
]

spline.points.add(count=len(waypoints) - 1)
for i, (x, y, z) in enumerate(waypoints):
    spline.points[i].co = (x, y, z, 1.0)  # NURBS w=1.0

spline.use_endpoint_u = True
spline.order_u = 4  # Smooth cubic

path_obj = bpy.data.objects.new("WalkthroughPath", curve_data)
bpy.context.scene.collection.objects.link(path_obj)

# Follow Path constraint
follow = cam_obj.constraints.new('FOLLOW_PATH')
follow.target = path_obj
follow.use_curve_follow = True
follow.forward_axis = 'FORWARD_Y'
follow.up_axis = 'UP_Z'

# Animate path traversal
follow.offset = 0.0
follow.keyframe_insert(data_path="offset", frame=1)
follow.offset = -100.0
follow.keyframe_insert(data_path="offset", frame=300)

# LINEAR interpolation for constant speed
if cam_obj.animation_data and cam_obj.animation_data.action:
    for fc in cam_obj.animation_data.action.fcurves:
        if "offset" in fc.data_path:
            for kp in fc.keyframe_points:
                kp.interpolation = 'LINEAR'

# Set as active camera
bpy.context.scene.camera = cam_obj

# Timeline
scene = bpy.context.scene
scene.frame_start = 1
scene.frame_end = 300
scene.render.fps = 30
```

---

## Example 3: Solar Shadow Study

Animate sun position to study shadows throughout the day.

```python
# Blender 3.x/4.x/5.x — Solar study animation
import bpy
from math import radians, sin, cos, asin, acos, pi

def create_sun_study(latitude=52.0, day_of_year=172,  # June 21
                     start_hour=6.0, end_hour=20.0,
                     frame_start=1, frame_end=240):
    """Create animated sun for shadow analysis.

    Args:
        latitude: Site latitude in degrees (positive = north)
        day_of_year: 1-365 (172 = summer solstice)
        start_hour: Start time (24h format)
        end_hour: End time (24h format)
        frame_start: First animation frame
        frame_end: Last animation frame
    """
    # Create sun light
    sun_data = bpy.data.lights.new("SolarStudy_Sun", type='SUN')
    sun_data.energy = 5.0
    sun_data.angle = radians(0.545)  # Realistic sun disc angle

    sun_obj = bpy.data.objects.new("SolarStudy_Sun", sun_data)
    bpy.context.scene.collection.objects.link(sun_obj)

    # Create look-at target at scene center
    target = bpy.data.objects.new("SunTarget", None)
    bpy.context.scene.collection.objects.link(target)
    target.location = (0, 0, 0)
    target.empty_display_type = 'PLAIN_AXES'
    target.empty_display_size = 0.5

    # Track To constraint
    track = sun_obj.constraints.new('TRACK_TO')
    track.target = target
    track.track_axis = 'TRACK_NEGATIVE_Z'
    track.up_axis = 'UP_Y'

    # Solar declination
    declination = 23.45 * sin(radians(360 / 365 * (day_of_year - 81)))
    lat_rad = radians(latitude)
    dec_rad = radians(declination)

    total_frames = frame_end - frame_start
    total_hours = end_hour - start_hour
    distance = 200  # Far enough to simulate parallel rays

    for i in range(total_frames + 1):
        frame = frame_start + i
        t = i / total_frames
        hour = start_hour + t * total_hours

        # Hour angle (degrees from solar noon)
        hour_angle_deg = (hour - 12.0) * 15.0
        hour_angle_rad = radians(hour_angle_deg)

        # Solar altitude angle
        sin_alt = (sin(lat_rad) * sin(dec_rad) +
                   cos(lat_rad) * cos(dec_rad) * cos(hour_angle_rad))
        altitude = max(0, asin(min(1, max(-1, sin_alt))))

        # Solar azimuth angle
        if cos(altitude) > 0.001:
            cos_azi = ((sin(dec_rad) - sin(altitude) * sin(lat_rad)) /
                       (cos(altitude) * cos(lat_rad)))
            cos_azi = min(1, max(-1, cos_azi))
            azimuth = acos(cos_azi)
            if hour_angle_deg > 0:
                azimuth = 2 * pi - azimuth
        else:
            azimuth = 0

        # Convert to Blender XYZ (Y=North, Z=Up)
        x = distance * cos(altitude) * sin(azimuth)
        y = distance * cos(altitude) * cos(azimuth)
        z = distance * sin(altitude)

        # Only keyframe if sun is above horizon
        if z > 0:
            sun_obj.location = (x, y, z)
            sun_obj.keyframe_insert(data_path="location", frame=frame)

            # Adjust energy based on altitude (dimmer at horizon)
            sun_data.energy = 5.0 * sin(altitude)
            sun_data.keyframe_insert(data_path="energy", frame=frame)

    # Set linear interpolation for smooth movement
    if sun_obj.animation_data and sun_obj.animation_data.action:
        for fc in sun_obj.animation_data.action.fcurves:
            for kp in fc.keyframe_points:
                kp.interpolation = 'LINEAR'

    return sun_obj

# Usage
sun = create_sun_study(latitude=52.37, day_of_year=172,  # Amsterdam, summer solstice
                       start_hour=5, end_hour=22,
                       frame_start=1, frame_end=240)

scene = bpy.context.scene
scene.frame_start = 1
scene.frame_end = 240
scene.render.fps = 24
```

---

## Example 4: NLA-Based Multi-Phase Animation

Combine separate animation actions into a sequenced timeline using NLA.

```python
# Blender 3.x/4.x/5.x — NLA orchestration for construction phases
import bpy

def create_slide_in_action(name, axis_index, distance, duration_frames):
    """Create action that slides object in from offset position.

    Args:
        name: Action name
        axis_index: 0=X, 1=Y, 2=Z
        distance: Offset distance
        duration_frames: Animation length in frames
    """
    action = bpy.data.actions.new(name=name)

    # Location FCurve
    fc = action.fcurves.new(data_path="location", index=axis_index)
    fc.keyframe_points.add(count=2)
    fc.keyframe_points[0].co = (0.0, distance)  # Start offset
    fc.keyframe_points[0].interpolation = 'BEZIER'
    fc.keyframe_points[0].handle_right_type = 'AUTO_CLAMPED'
    fc.keyframe_points[1].co = (float(duration_frames), 0.0)  # Final position
    fc.keyframe_points[1].interpolation = 'BEZIER'
    fc.keyframe_points[1].handle_left_type = 'AUTO_CLAMPED'
    fc.update()

    # Visibility FCurve (appear at frame 0)
    fc_vis = action.fcurves.new(data_path="hide_viewport")
    fc_vis.keyframe_points.add(count=2)
    fc_vis.keyframe_points[0].co = (0.0, 1.0)  # Hidden
    fc_vis.keyframe_points[0].interpolation = 'CONSTANT'
    fc_vis.keyframe_points[1].co = (1.0, 0.0)  # Visible
    fc_vis.keyframe_points[1].interpolation = 'CONSTANT'
    fc_vis.update()

    return action

def apply_nla_sequence(obj, actions_with_timing):
    """Apply multiple actions to an object via NLA tracks.

    Args:
        obj: Target object
        actions_with_timing: list of (action, nla_start_frame) tuples
    """
    anim_data = obj.animation_data
    if anim_data is None:
        anim_data = obj.animation_data_create()

    # Clear existing NLA
    while anim_data.nla_tracks:
        anim_data.nla_tracks.remove(anim_data.nla_tracks[0])

    for action, start_frame in actions_with_timing:
        track = anim_data.nla_tracks.new()
        track.name = f"NLA_{action.name}"

        strip = track.strips.new(
            name=action.name,
            start=start_frame,
            action=action
        )
        strip.blend_type = 'COMBINE'
        strip.use_auto_blend = True

# Create reusable actions
drop_in = create_slide_in_action("DropIn_Z", axis_index=2, distance=15.0, duration_frames=20)
slide_in_x = create_slide_in_action("SlideIn_X", axis_index=0, distance=20.0, duration_frames=30)

# Apply to construction elements
columns = [obj for obj in bpy.data.objects if obj.name.startswith("Column_")]
for i, col_obj in enumerate(columns):
    apply_nla_sequence(col_obj, [
        (drop_in, 30 + i * 15),  # Staggered drop-in
    ])

beams = [obj for obj in bpy.data.objects if obj.name.startswith("Beam_")]
for i, beam_obj in enumerate(beams):
    apply_nla_sequence(beam_obj, [
        (slide_in_x, 90 + i * 10),  # Staggered slide-in
    ])
```

---

## Example 5: Batch Keyframe Operations

Efficiently animate large numbers of objects for AEC visualization.

```python
# Blender 3.x/4.x/5.x — Batch animation utilities
import bpy

def batch_animate_appearance(objects, start_frame, stagger_frames=5):
    """Make objects appear one by one with staggered timing.

    Args:
        objects: List of objects to animate
        start_frame: First object appears at this frame
        stagger_frames: Delay between each object's appearance
    """
    for i, obj in enumerate(objects):
        appear = start_frame + i * stagger_frames

        obj.hide_viewport = True
        obj.hide_render = True
        obj.keyframe_insert(data_path="hide_viewport", frame=appear - 1)
        obj.keyframe_insert(data_path="hide_render", frame=appear - 1)

        obj.hide_viewport = False
        obj.hide_render = False
        obj.keyframe_insert(data_path="hide_viewport", frame=appear)
        obj.keyframe_insert(data_path="hide_render", frame=appear)

    # Batch set CONSTANT interpolation
    for obj in objects:
        if obj.animation_data and obj.animation_data.action:
            for fc in obj.animation_data.action.fcurves:
                if fc.data_path in ("hide_viewport", "hide_render"):
                    for kp in fc.keyframe_points:
                        kp.interpolation = 'CONSTANT'

def batch_animate_scale_in(objects, start_frame, duration=15, stagger=5):
    """Scale objects from 0 to full size with stagger.

    Args:
        objects: List of objects to animate
        start_frame: First object starts scaling at this frame
        duration: Scale animation duration per object
        stagger: Frame delay between each object
    """
    for i, obj in enumerate(objects):
        frame_in = start_frame + i * stagger

        obj.scale = (0.001, 0.001, 0.001)  # Near-zero (not 0 to avoid math issues)
        obj.keyframe_insert(data_path="scale", frame=frame_in)

        obj.scale = (1.0, 1.0, 1.0)
        obj.keyframe_insert(data_path="scale", frame=frame_in + duration)

    # Set BEZIER with ease-out for natural appearance
    for obj in objects:
        if obj.animation_data and obj.animation_data.action:
            for fc in obj.animation_data.action.fcurves:
                if fc.data_path == "scale":
                    for kp in fc.keyframe_points:
                        kp.interpolation = 'BEZIER'
                        kp.easing = 'EASE_OUT'
                        kp.handle_left_type = 'AUTO_CLAMPED'
                        kp.handle_right_type = 'AUTO_CLAMPED'

# Usage: Animate all objects in "Structure" collection
structure_col = bpy.data.collections.get("Phase_Structure")
if structure_col:
    sorted_objs = sorted(structure_col.objects, key=lambda o: o.location.z)
    batch_animate_appearance(sorted_objs, start_frame=30, stagger_frames=8)
```

---

## Example 6: Driver-Based Parametric Animation

Use drivers for procedural animation tied to control objects.

```python
# Blender 3.x/4.x/5.x — Driver-based construction progress
import bpy

def setup_progress_driver(target_obj, control_obj, control_prop_path,
                          expression="var"):
    """Drive object visibility scale by a control property.

    Args:
        target_obj: Object to be driven
        control_obj: Object containing the control property
        control_prop_path: RNA path to control property on control_obj
        expression: Driver expression using 'var'
    """
    # Add driver to Z scale
    driver_fc = target_obj.driver_add("scale", 2)  # Z scale
    driver = driver_fc.driver
    driver.type = 'SCRIPTED'

    var = driver.variables.new()
    var.name = "var"
    var.type = 'SINGLE_PROP'
    var.targets[0].id = control_obj
    var.targets[0].data_path = control_prop_path

    driver.expression = expression

# Create progress control empty
ctrl = bpy.data.objects.new("ProgressControl", None)
bpy.context.scene.collection.objects.link(ctrl)
ctrl.empty_display_type = 'SINGLE_ARROW'

# Custom property for progress (0.0 to 1.0)
ctrl["progress"] = 0.0
# Keyframe the custom property
ctrl.keyframe_insert(data_path='["progress"]', frame=1)
ctrl["progress"] = 1.0
ctrl.keyframe_insert(data_path='["progress"]', frame=200)

# Drive multiple objects from the single control
floors = [obj for obj in bpy.data.objects if obj.name.startswith("Floor_")]
for i, floor_obj in enumerate(floors):
    threshold = i / max(len(floors) - 1, 1)
    # Expression: object appears (scale=1) when progress passes its threshold
    expr = f"1.0 if var > {threshold:.3f} else 0.001"
    setup_progress_driver(floor_obj, ctrl, '["progress"]', expr)
```

---

## Example 7: Render Animation to Image Sequence

Setup and execute animation rendering.

```python
# Blender 3.x/4.x/5.x — Configure and render animation
import bpy

scene = bpy.context.scene

# Frame range
scene.frame_start = 1
scene.frame_end = 250
scene.frame_step = 1

# Resolution
scene.render.resolution_x = 1920
scene.render.resolution_y = 1080
scene.render.resolution_percentage = 100

# FPS
scene.render.fps = 24
scene.render.fps_base = 1.0

# Output as PNG image sequence (recommended for long renders — resumable)
scene.render.filepath = "//render/construction_seq_"
scene.render.image_settings.file_format = 'PNG'
scene.render.image_settings.color_mode = 'RGBA'
scene.render.image_settings.color_depth = '16'
scene.render.image_settings.compression = 50

# Transparent background (for compositing)
scene.render.film_transparent = True

# Render engine
scene.render.engine = 'BLENDER_EEVEE_NEXT'  # Or 'CYCLES'

# Execute render (blocking call)
# bpy.ops.render.render(animation=True)
```
