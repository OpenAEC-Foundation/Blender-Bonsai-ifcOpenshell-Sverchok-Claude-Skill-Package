# blender-syntax-rendering — Working Examples

All examples verified against official Blender Python API documentation.

---

## Example 1: Complete Still Image Render Setup

```python
# Blender 3.x / 4.x / 5.x — Full render setup from scratch
import bpy

scene = bpy.context.scene
render = scene.render

# Engine (version-safe EEVEE)
if bpy.app.version >= (5, 0, 0):
    render.engine = 'BLENDER_EEVEE'
elif bpy.app.version >= (4, 2, 0):
    render.engine = 'BLENDER_EEVEE_NEXT'
else:
    render.engine = 'BLENDER_EEVEE'

# Resolution
render.resolution_x = 1920
render.resolution_y = 1080
render.resolution_percentage = 100

# Output
render.filepath = "/tmp/render_result.png"
render.image_settings.file_format = 'PNG'
render.image_settings.color_mode = 'RGBA'
render.image_settings.color_depth = '16'
render.image_settings.compression = 15

# Render and save
bpy.ops.render.render(write_still=True)
```

---

## Example 2: Cycles GPU Render with Denoising

```python
# Blender 3.x / 4.x / 5.x — Cycles GPU render pipeline
import bpy

scene = bpy.context.scene
render = scene.render

# Set engine
render.engine = 'CYCLES'
cycles = scene.cycles

# GPU setup
cycles.device = 'GPU'
prefs = bpy.context.preferences
cycles_prefs = prefs.addons['cycles'].preferences
cycles_prefs.compute_device_type = 'CUDA'
cycles_prefs.get_devices()  # REQUIRED: refresh device list
for device in cycles_prefs.devices:
    device.use = True

# Sampling
cycles.samples = 512
cycles.use_adaptive_sampling = True
cycles.adaptive_threshold = 0.01

# Denoising
cycles.use_denoising = True
cycles.denoiser = 'OPENIMAGEDENOISE'

# Light paths
cycles.max_bounces = 12
cycles.diffuse_bounces = 4
cycles.glossy_bounces = 4
cycles.transmission_bounces = 12

# Resolution
render.resolution_x = 3840
render.resolution_y = 2160
render.resolution_percentage = 100

# Output
render.filepath = "/tmp/cycles_render.png"
render.image_settings.file_format = 'PNG'
render.image_settings.color_mode = 'RGBA'

# Render
bpy.ops.render.render(write_still=True)
```

---

## Example 3: Batch Render Multiple Cameras

```python
# Blender 3.x / 4.x / 5.x — Render from every camera in the scene
import bpy
import os

scene = bpy.context.scene
render = scene.render
output_dir = "/tmp/batch_renders/"

# Ensure output directory exists
os.makedirs(output_dir, exist_ok=True)

# Configure output format once
render.image_settings.file_format = 'PNG'
render.image_settings.color_mode = 'RGBA'
render.resolution_x = 1920
render.resolution_y = 1080

# Find all cameras
cameras = [obj for obj in bpy.data.objects if obj.type == 'CAMERA']

if not cameras:
    raise RuntimeError("No cameras found in scene")

# Render from each camera
for cam in cameras:
    scene.camera = cam
    render.filepath = os.path.join(output_dir, f"{cam.name}.png")
    bpy.ops.render.render(write_still=True)
    print(f"Rendered: {cam.name}")
```

---

## Example 4: Animation Render

```python
# Blender 3.x / 4.x / 5.x — Render animation frame range
import bpy

scene = bpy.context.scene
render = scene.render

# Frame range
scene.frame_start = 1
scene.frame_end = 120
scene.frame_step = 1
render.fps = 30

# Output as image sequence (one file per frame)
render.filepath = "/tmp/animation/frame_"
render.image_settings.file_format = 'PNG'
render.image_settings.color_mode = 'RGBA'

# Render entire animation
bpy.ops.render.render(animation=True)
```

---

## Example 5: Camera Creation with Depth of Field

```python
# Blender 3.x / 4.x / 5.x — Create camera with DOF for architectural viz
import bpy
from mathlib import Vector

scene = bpy.context.scene

# Create camera data and object
cam_data = bpy.data.cameras.new("ArchViz_Camera")
cam_obj = bpy.data.objects.new("ArchViz_Camera", cam_data)
bpy.context.collection.objects.link(cam_obj)

# Set as active camera
scene.camera = cam_obj

# Perspective lens
cam_data.type = 'PERSP'
cam_data.lens = 24.0          # Wide angle for architecture
cam_data.clip_start = 0.1
cam_data.clip_end = 500.0

# Sensor (full-frame equivalent)
cam_data.sensor_width = 36.0
cam_data.sensor_height = 24.0

# Position
cam_obj.location = (15.0, -12.0, 6.0)
cam_obj.rotation_euler = (1.22, 0.0, 0.87)

# Depth of field
cam_data.dof.use_dof = True
cam_data.dof.focus_distance = 10.0
cam_data.dof.aperture_fstop = 5.6
cam_data.dof.aperture_blades = 8  # Octagonal bokeh
```

---

## Example 6: Interior Lighting Setup

```python
# Blender 3.x / 4.x / 5.x — Interior lighting with multiple light types
import bpy

collection = bpy.context.collection

# Ceiling area light (primary ambient)
area_data = bpy.data.lights.new("CeilingLight", type='AREA')
area_data.shape = 'RECTANGLE'
area_data.size = 3.0
area_data.size_y = 2.0
area_data.energy = 800.0
area_data.color = (1.0, 0.97, 0.92)  # Warm white
area_obj = bpy.data.objects.new("CeilingLight", area_data)
area_obj.location = (0.0, 0.0, 2.8)
area_obj.rotation_euler = (0.0, 0.0, 0.0)  # Pointing down
collection.objects.link(area_obj)

# Accent spot light
spot_data = bpy.data.lights.new("AccentSpot", type='SPOT')
spot_data.energy = 200.0
spot_data.spot_size = 0.52        # ~30 degrees
spot_data.spot_blend = 0.3
spot_data.color = (1.0, 0.95, 0.85)
spot_obj = bpy.data.objects.new("AccentSpot", spot_data)
spot_obj.location = (2.0, -1.0, 2.5)
spot_obj.rotation_euler = (0.5, 0.0, 0.3)
collection.objects.link(spot_obj)

# Fill point light
point_data = bpy.data.lights.new("FillLight", type='POINT')
point_data.energy = 100.0
point_data.shadow_soft_size = 0.5
point_data.color = (0.85, 0.9, 1.0)  # Cool fill
point_obj = bpy.data.objects.new("FillLight", point_data)
point_obj.location = (-3.0, 2.0, 1.5)
collection.objects.link(point_obj)
```

---

## Example 7: EXR Output for Compositing

```python
# Blender 3.x / 4.x / 5.x — Render to OpenEXR for post-processing
import bpy

scene = bpy.context.scene
render = scene.render

render.engine = 'CYCLES'

# EXR format setup
render.image_settings.file_format = 'OPEN_EXR_MULTILAYER'
render.image_settings.color_depth = '32'
render.image_settings.exr_codec = 'ZIP'

# Enable render passes on view layer
view_layer = scene.view_layers[0]
view_layer.use_pass_diffuse_color = True
view_layer.use_pass_glossy_color = True
view_layer.use_pass_z = True
view_layer.use_pass_normal = True
view_layer.use_pass_mist = True

# Output
render.filepath = "/tmp/compositing_render.exr"
render.resolution_x = 1920
render.resolution_y = 1080

bpy.ops.render.render(write_still=True)
```

---

## Example 8: Headless Batch Render Script

```python
#!/usr/bin/env python3
# render_batch.py — Run with: blender --background scene.blend --python render_batch.py
import bpy
import sys
import os

def configure_render(engine='CYCLES', samples=256, resolution=(1920, 1080)):
    scene = bpy.context.scene
    render = scene.render

    # Engine
    if engine == 'EEVEE':
        if bpy.app.version >= (5, 0, 0):
            render.engine = 'BLENDER_EEVEE'
        elif bpy.app.version >= (4, 2, 0):
            render.engine = 'BLENDER_EEVEE_NEXT'
        else:
            render.engine = 'BLENDER_EEVEE'
    elif engine == 'CYCLES':
        render.engine = 'CYCLES'
        scene.cycles.samples = samples
        scene.cycles.device = 'GPU'
        prefs = bpy.context.preferences
        cycles_prefs = prefs.addons['cycles'].preferences
        cycles_prefs.compute_device_type = 'CUDA'
        cycles_prefs.get_devices()
        for device in cycles_prefs.devices:
            device.use = True

    # Resolution
    render.resolution_x = resolution[0]
    render.resolution_y = resolution[1]
    render.resolution_percentage = 100

    # Output format
    render.image_settings.file_format = 'PNG'
    render.image_settings.color_mode = 'RGBA'

def render_all_cameras(output_dir):
    os.makedirs(output_dir, exist_ok=True)
    scene = bpy.context.scene
    cameras = [obj for obj in bpy.data.objects if obj.type == 'CAMERA']

    for cam in cameras:
        scene.camera = cam
        scene.render.filepath = os.path.join(output_dir, f"{cam.name}.png")
        bpy.ops.render.render(write_still=True)
        print(f"[OK] {cam.name}")

# Execute
configure_render(engine='CYCLES', samples=128, resolution=(3840, 2160))
render_all_cameras("/tmp/batch_output/")
```

Command line usage:
```bash
blender --background project.blend --python render_batch.py
```

---

## Example 9: Orthographic Camera for Plan/Section Views

```python
# Blender 3.x / 4.x / 5.x — Orthographic camera for architectural plans
import bpy
from math import radians

scene = bpy.context.scene

# Create orthographic camera
cam_data = bpy.data.cameras.new("PlanCamera")
cam_data.type = 'ORTHO'
cam_data.ortho_scale = 20.0    # Covers 20m width
cam_data.clip_start = 0.1
cam_data.clip_end = 100.0

cam_obj = bpy.data.objects.new("PlanCamera", cam_data)
bpy.context.collection.objects.link(cam_obj)

# Position: directly above, looking down (plan view)
cam_obj.location = (0.0, 0.0, 30.0)
cam_obj.rotation_euler = (0.0, 0.0, 0.0)  # Looking straight down

scene.camera = cam_obj

# Configure for technical output
render = scene.render
render.resolution_x = 4000
render.resolution_y = 4000
render.resolution_percentage = 100
render.filepath = "/tmp/plan_view.png"
render.image_settings.file_format = 'PNG'
render.image_settings.color_mode = 'RGBA'

bpy.ops.render.render(write_still=True)
```

---

## Example 10: Version-Safe Render Pass Access

```python
# Blender 3.x / 4.x / 5.x — Version-safe render pass name lookup
import bpy

# Render pass name mapping (old → new in 5.0)
PASS_NAMES = {
    "DiffCol": "Diffuse Color",
    "GlossCol": "Glossy Color",
    "TransCol": "Transmission Color",
    "IndexMA": "Material Index",
    "IndexOB": "Object Index",
    "Z": "Depth",
}

def get_pass_name(old_name):
    """Return the correct render pass name for the current Blender version."""
    if bpy.app.version >= (5, 0, 0):
        return PASS_NAMES.get(old_name, old_name)
    return old_name

# Usage
depth_pass = get_pass_name("Z")  # Returns "Depth" in 5.0+, "Z" in 3.x/4.x
```
