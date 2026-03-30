---
name: blender-syntax-rendering
description: >
  Use when configuring render settings or automating renders in Blender Python. Prevents
  the version pitfall of using 'BLENDER_EEVEE' (renamed to 'BLENDER_EEVEE_NEXT' in 4.2).
  Covers render engine selection (EEVEE/Cycles/Workbench), output format setup, camera
  configuration, batch rendering, and scene.render.* settings.
  Keywords: render, EEVEE, Cycles, Workbench, render settings, output format, camera, batch render, scene.render, BLENDER_EEVEE_NEXT, resolution, render from script, set render resolution, save image.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-rendering

## Quick Reference

### Critical Warnings

**NEVER** use `'BLENDER_EEVEE'` in Blender 4.2-4.4 — the identifier is `'BLENDER_EEVEE_NEXT'` in those versions.

**NEVER** call `bpy.ops.render.render()` without `write_still=True` for still images — the render completes but the image is never saved to disk.

**NEVER** set Cycles GPU device without calling `cycles_prefs.get_devices()` first — the device list is empty until explicitly refreshed.

**NEVER** use `scene.use_nodes = True` in Blender 5.0+ — the compositor is always active; the attribute is deprecated.

**NEVER** use `scene.node_tree` in Blender 5.0+ — use `scene.compositing_node_group` instead.

**NEVER** use `eevee.use_gtao` in Blender 5.0+ — use `view_layer.eevee.ambient_occlusion_distance` instead.

**ALWAYS** use the version-safe EEVEE selection pattern when targeting multiple Blender versions.

### EEVEE Engine Identifier Matrix

| Blender Version | Engine Identifier | Notes |
|----------------|-------------------|-------|
| 3.0 - 4.1 | `'BLENDER_EEVEE'` | Original EEVEE |
| 4.2 - 4.4 | `'BLENDER_EEVEE_NEXT'` | Rewritten EEVEE engine |
| 5.0+ | `'BLENDER_EEVEE'` | Identifier reverted to original |

### Render Engine Decision Tree

```
What render engine does the task require?
├── Photo-realistic / ray-traced → CYCLES
│   └── GPU available?
│       ├── Yes → cycles.device = 'GPU' (call get_devices() first)
│       └── No  → cycles.device = 'CPU'
├── Real-time / fast preview → EEVEE (use version-safe pattern)
│   └── Blender version?
│       ├── 3.x / 4.0-4.1 / 5.0+ → 'BLENDER_EEVEE'
│       └── 4.2 - 4.4            → 'BLENDER_EEVEE_NEXT'
└── Solid/wireframe viewport look → WORKBENCH
    └── scene.render.engine = 'BLENDER_WORKBENCH'
```

### Render Pass Names (5.0 Changes)

| Old Name (3.x / 4.x) | New Name (5.0+) |
|-----------------------|-----------------|
| `"DiffCol"` | `"Diffuse Color"` |
| `"GlossCol"` | `"Glossy Color"` |
| `"TransCol"` | `"Transmission Color"` |
| `"IndexMA"` | `"Material Index"` |
| `"IndexOB"` | `"Object Index"` |
| `"Z"` | `"Depth"` |

---

## Essential Patterns

### Pattern 1: Version-Safe EEVEE Selection

ALWAYS use this pattern when targeting multiple Blender versions:

```python
# Blender 3.x / 4.x / 5.x: Version-safe EEVEE selection
import bpy
scene = bpy.context.scene

if bpy.app.version >= (5, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE'
elif bpy.app.version >= (4, 2, 0):
    scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    scene.render.engine = 'BLENDER_EEVEE'
```

### Pattern 2: Render Settings Configuration

```python
# Blender 3.x / 4.x / 5.x: Configure render output
scene = bpy.context.scene
render = scene.render

# Resolution
render.resolution_x = 1920
render.resolution_y = 1080
render.resolution_percentage = 100  # Scale factor (1-100)

# Output format
render.image_settings.file_format = 'PNG'   # 'PNG','JPEG','OPEN_EXR','TIFF','BMP'
render.image_settings.color_mode = 'RGBA'   # 'BW', 'RGB', 'RGBA'
render.image_settings.color_depth = '16'    # '8', '16', '32' (format-dependent)
render.image_settings.compression = 15      # PNG compression (0-100)

# Output path
render.filepath = "/tmp/render_output/frame_"

# Frame range
scene.frame_start = 1
scene.frame_end = 250
scene.frame_step = 1
render.fps = 24
```

### Pattern 3: Cycles Configuration

```python
# Blender 3.x / 4.x / 5.x: Cycles setup
scene.render.engine = 'CYCLES'
cycles = scene.cycles

# Device selection
cycles.device = 'GPU'  # 'CPU' or 'GPU'

# Sampling
cycles.samples = 256
cycles.preview_samples = 64
cycles.use_adaptive_sampling = True
cycles.adaptive_threshold = 0.01

# Denoiser
cycles.use_denoising = True
cycles.denoiser = 'OPENIMAGEDENOISE'  # 'OPENIMAGEDENOISE' or 'OPTIX'

# Light paths
cycles.max_bounces = 12
cycles.diffuse_bounces = 4
cycles.glossy_bounces = 4
cycles.transmission_bounces = 12
cycles.transparent_max_bounces = 8
```

### Pattern 4: Cycles GPU Setup

ALWAYS call `get_devices()` before enabling GPU devices:

```python
# Blender 3.x / 4.x / 5.x: Cycles GPU configuration
prefs = bpy.context.preferences
cycles_prefs = prefs.addons['cycles'].preferences

# Set compute type FIRST
cycles_prefs.compute_device_type = 'CUDA'  # 'CUDA','OPTIX','HIP','ONEAPI','METAL'

# ALWAYS refresh device list before enabling
cycles_prefs.get_devices()

# Enable all available devices
for device in cycles_prefs.devices:
    device.use = True
```

### Pattern 5: Camera Creation and Configuration

```python
# Blender 3.x / 4.x / 5.x: Camera setup
cam_data = bpy.data.cameras.new("RenderCamera")
cam_obj = bpy.data.objects.new("RenderCamera", cam_data)
bpy.context.collection.objects.link(cam_obj)  # REQUIRED: link to collection

# Set as active scene camera
scene.camera = cam_obj

# Lens
cam_data.type = 'PERSP'        # 'PERSP', 'ORTHO', 'PANO'
cam_data.lens = 35.0            # Focal length in mm (perspective only)
cam_data.ortho_scale = 10.0     # Orthographic scale (ortho only)

# Clipping
cam_data.clip_start = 0.1
cam_data.clip_end = 1000.0

# Position and orientation
cam_obj.location = (10.0, -10.0, 8.0)
cam_obj.rotation_euler = (1.1, 0.0, 0.785)
```

### Pattern 6: Depth of Field

```python
# Blender 3.x / 4.x / 5.x: Depth of field
cam_data.dof.use_dof = True
cam_data.dof.focus_object = bpy.data.objects.get("FocusTarget")
cam_data.dof.aperture_fstop = 2.8

# Sensor settings
cam_data.sensor_width = 36.0   # mm
cam_data.sensor_height = 24.0  # mm
```

---

## Common Operations

### Render a Still Image

```python
# Blender 3.x / 4.x / 5.x: Render and save a still image
scene = bpy.context.scene
scene.render.filepath = "/tmp/output.png"
scene.render.image_settings.file_format = 'PNG'
bpy.ops.render.render(write_still=True)  # write_still=True is REQUIRED
```

### Render an Animation

```python
# Blender 3.x / 4.x / 5.x: Render animation sequence
scene = bpy.context.scene
scene.frame_start = 1
scene.frame_end = 250
scene.render.filepath = "/tmp/renders/anim_"
bpy.ops.render.render(animation=True)
```

### Batch Render Multiple Cameras

```python
# Blender 3.x / 4.x / 5.x: Batch render from all cameras
import bpy

scene = bpy.context.scene
cameras = [obj for obj in bpy.data.objects if obj.type == 'CAMERA']

for cam in cameras:
    scene.camera = cam
    scene.render.filepath = f"/tmp/renders/{cam.name}_"
    bpy.ops.render.render(write_still=True)
```

### Headless Rendering (Command Line)

```bash
# Render with a script (no GUI)
blender --background scene.blend --python render_script.py
```

```python
# render_script.py
import bpy
scene = bpy.context.scene
scene.render.filepath = "/tmp/headless_render.png"
bpy.ops.render.render(write_still=True)
```

### Create Light Sources

```python
# Blender 3.x / 4.x / 5.x: Light types
# Point light
point_data = bpy.data.lights.new("PointLight", type='POINT')
point_data.energy = 1000.0      # Watts
point_data.shadow_soft_size = 0.25
point_data.color = (1.0, 0.95, 0.9)
point_obj = bpy.data.objects.new("PointLight", point_data)
bpy.context.collection.objects.link(point_obj)

# Sun light
sun_data = bpy.data.lights.new("SunLight", type='SUN')
sun_data.energy = 5.0
sun_data.angle = 0.00918  # Angular diameter in radians

# Area light
area_data = bpy.data.lights.new("AreaLight", type='AREA')
area_data.shape = 'RECTANGLE'  # 'SQUARE','RECTANGLE','DISK','ELLIPSE'
area_data.size = 2.0
area_data.size_y = 1.0         # Only for RECTANGLE/ELLIPSE
area_data.energy = 500.0

# Spot light
spot_data = bpy.data.lights.new("SpotLight", type='SPOT')
spot_data.spot_size = 0.785    # Cone angle in radians
spot_data.spot_blend = 0.15    # Soft edge (0=hard, 1=fully soft)
```

### Light Type Reference

| Type | String | Use Case |
|------|--------|----------|
| Point | `'POINT'` | Omnidirectional (lamps, bulbs) |
| Sun | `'SUN'` | Directional (exteriors); position irrelevant, only rotation matters |
| Spot | `'SPOT'` | Cone-shaped (stage lighting, flashlights) |
| Area | `'AREA'` | Surface-emitting (soft shadows, interiors) |

### EEVEE-Specific Settings (Version-Dependent)

```python
# Blender 4.2+: EEVEE settings via scene.eevee
eevee = scene.eevee

# Shadows
eevee.shadow_cube_size = '1024'
eevee.shadow_cascade_size = '2048'

# Ray tracing (4.2+)
eevee.ray_tracing_method = 'SCREEN'  # 'SCREEN' or 'PROBE'

# Ambient Occlusion
# Blender 4.x:
#   eevee.use_gtao = True            # REMOVED in 5.0
#   eevee.gtao_distance = 1.0        # REMOVED in 5.0
# Blender 5.0+:
#   view_layer.eevee.ambient_occlusion_distance = 1.0
```

### Compositor Node Access (Version-Dependent)

```python
# Blender 3.x / 4.x: Compositor node tree
scene.use_nodes = True           # Enable compositor  (DEPRECATED in 5.0)
tree = scene.node_tree           # Access node tree   (REMOVED in 5.0)

# Blender 5.0+: Compositor node group
tree = scene.compositing_node_group  # New accessor
```

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for RenderSettings, CyclesRenderSettings, SceneEEVEE, Camera, Light types, render operators
- [references/examples.md](references/examples.md) — Working code examples for common rendering workflows
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with rendering API, version-specific pitfalls

### Official Sources

- https://docs.blender.org/api/current/bpy.types.RenderSettings.html
- https://docs.blender.org/api/current/bpy.types.SceneEEVEE.html
- https://docs.blender.org/api/current/bpy.ops.render.html
- https://docs.blender.org/api/current/bpy.types.Camera.html
- https://docs.blender.org/api/current/bpy.types.Light.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/
