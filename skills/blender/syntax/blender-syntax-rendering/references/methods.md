# blender-syntax-rendering — Method Reference

## bpy.types.RenderSettings (scene.render)

All versions: Blender 3.x / 4.x / 5.x unless noted.

### Resolution and Scaling

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `resolution_x` | `int` | `1920` | Horizontal pixel count |
| `resolution_y` | `int` | `1080` | Vertical pixel count |
| `resolution_percentage` | `int` | `100` | Scale factor applied to resolution (1-100) |
| `pixel_aspect_x` | `float` | `1.0` | Horizontal pixel aspect ratio |
| `pixel_aspect_y` | `float` | `1.0` | Vertical pixel aspect ratio |

### Engine Selection

| Property | Type | Values | Description |
|----------|------|--------|-------------|
| `engine` | `str` | `'BLENDER_EEVEE'`, `'BLENDER_EEVEE_NEXT'`, `'CYCLES'`, `'BLENDER_WORKBENCH'` | Active render engine (EEVEE identifier is version-dependent) |

### Output Settings

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `filepath` | `str` | `"/tmp/"` | Output file path (for stills and animation frames) |
| `use_file_extension` | `bool` | `True` | Append file extension automatically |
| `use_overwrite` | `bool` | `True` | Overwrite existing files |
| `use_placeholder` | `bool` | `False` | Create empty placeholder frames before rendering |

### Frame Range and Timing

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `fps` | `int` | `24` | Frames per second |
| `fps_base` | `float` | `1.0` | FPS base divisor (effective FPS = fps / fps_base) |
| `frame_map_old` | `int` | `100` | Old frame mapping value |
| `frame_map_new` | `int` | `100` | New frame mapping value |

Frame range is on `scene` (not `render`):

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `scene.frame_start` | `int` | `1` | First frame of animation |
| `scene.frame_end` | `int` | `250` | Last frame of animation |
| `scene.frame_step` | `int` | `1` | Frame step (skip frames) |
| `scene.frame_current` | `int` | `1` | Current frame |

### Performance

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `threads_mode` | `str` | `'AUTO'` | `'AUTO'` or `'FIXED'` |
| `threads` | `int` | `1` | Thread count (only when `threads_mode='FIXED'`) |
| `use_persistent_data` | `bool` | `False` | Keep render data in memory between frames (Cycles only) |

---

## bpy.types.ImageFormatSettings (render.image_settings)

| Property | Type | Values | Description |
|----------|------|--------|-------------|
| `file_format` | `str` | `'PNG'`, `'JPEG'`, `'OPEN_EXR'`, `'OPEN_EXR_MULTILAYER'`, `'TIFF'`, `'BMP'`, `'HDR'`, `'TARGA'`, `'CINEON'`, `'DPX'`, `'WEBP'` | Output image format |
| `color_mode` | `str` | `'BW'`, `'RGB'`, `'RGBA'` | Color channels (RGBA only for formats that support alpha) |
| `color_depth` | `str` | `'8'`, `'16'`, `'32'` | Bit depth (available values depend on format) |
| `compression` | `int` | `15` | PNG compression level (0-100) |
| `quality` | `int` | `90` | JPEG/WEBP quality (0-100) |
| `exr_codec` | `str` | `'ZIP'` | EXR compression codec: `'NONE'`, `'PXR24'`, `'ZIP'`, `'PIZ'`, `'RLE'`, `'ZIPS'`, `'DWAA'`, `'DWAB'` |
| `color_management` | `str` | `'FOLLOW_SCENE'` | `'FOLLOW_SCENE'` or `'OVERRIDE'` |

### Format-Specific Color Depth Availability

| Format | Available Depths |
|--------|-----------------|
| PNG | `'8'`, `'16'` |
| JPEG | `'8'` |
| OPEN_EXR | `'16'`, `'32'` |
| TIFF | `'8'`, `'16'` |
| BMP | `'8'` |
| HDR | `'32'` |

---

## bpy.types.CyclesRenderSettings (scene.cycles)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `device` | `str` | `'CPU'` | `'CPU'` or `'GPU'` |
| `feature_set` | `str` | `'SUPPORTED'` | `'SUPPORTED'` or `'EXPERIMENTAL'` |
| `samples` | `int` | `4096` | Final render samples |
| `preview_samples` | `int` | `1024` | Viewport preview samples |
| `use_adaptive_sampling` | `bool` | `True` | Enable adaptive sampling |
| `adaptive_threshold` | `float` | `0.01` | Noise threshold for adaptive sampling |
| `time_limit` | `float` | `0.0` | Time limit per frame in seconds (0 = no limit) |
| `use_denoising` | `bool` | `True` | Enable render denoising |
| `denoiser` | `str` | `'OPENIMAGEDENOISE'` | `'OPENIMAGEDENOISE'` or `'OPTIX'` (OPTIX requires NVIDIA GPU) |
| `denoising_input_passes` | `str` | `'RGB_ALBEDO_NORMAL'` | `'RGB'`, `'RGB_ALBEDO'`, `'RGB_ALBEDO_NORMAL'` |

### Light Path Bounces

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `max_bounces` | `int` | `12` | Total maximum bounces |
| `diffuse_bounces` | `int` | `4` | Maximum diffuse bounces |
| `glossy_bounces` | `int` | `4` | Maximum glossy bounces |
| `transmission_bounces` | `int` | `12` | Maximum transmission bounces |
| `transparent_max_bounces` | `int` | `8` | Maximum transparent bounces |
| `volume_bounces` | `int` | `0` | Maximum volume bounces |

### Cycles Preferences (GPU Setup)

Access: `bpy.context.preferences.addons['cycles'].preferences`

| Property / Method | Type | Description |
|-------------------|------|-------------|
| `compute_device_type` | `str` | `'CUDA'`, `'OPTIX'`, `'HIP'`, `'ONEAPI'`, `'METAL'`, `'NONE'` |
| `get_devices()` | method | Refresh available device list. MUST call before enabling devices |
| `devices` | collection | Iterable of `CyclesDeviceSettings` (after `get_devices()`) |
| `devices[i].use` | `bool` | Enable/disable individual compute device |
| `devices[i].name` | `str` | Human-readable device name |
| `devices[i].type` | `str` | `'CPU'`, `'CUDA'`, `'OPTIX'`, `'HIP'`, `'ONEAPI'`, `'METAL'` |

---

## bpy.types.SceneEEVEE (scene.eevee)

Available in Blender 4.2+. Properties vary by version.

| Property | Type | Default | Version | Description |
|----------|------|---------|---------|-------------|
| `shadow_cube_size` | `str` | `'512'` | 4.2+ | Shadow map resolution for point/spot lights |
| `shadow_cascade_size` | `str` | `'1024'` | 4.2+ | Shadow map resolution for sun lights |
| `ray_tracing_method` | `str` | `'SCREEN'` | 4.2+ | `'SCREEN'` (screen-space) or `'PROBE'` (light probes) |
| `use_gtao` | `bool` | `False` | 4.x only | Enable ambient occlusion. **REMOVED in 5.0** |
| `gtao_distance` | `float` | `0.2` | 4.x only | AO sampling distance. **REMOVED in 5.0** |

Blender 5.0+ AO replacement: `view_layer.eevee.ambient_occlusion_distance`

---

## bpy.types.Camera (bpy.data.cameras)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `type` | `str` | `'PERSP'` | `'PERSP'`, `'ORTHO'`, `'PANO'` |
| `lens` | `float` | `50.0` | Focal length in mm (perspective mode) |
| `lens_unit` | `str` | `'MILLIMETERS'` | `'MILLIMETERS'` or `'FOV'` |
| `angle` | `float` | — | Field of view in radians (when `lens_unit='FOV'`) |
| `ortho_scale` | `float` | `6.0` | Orthographic scale |
| `clip_start` | `float` | `0.1` | Near clipping distance |
| `clip_end` | `float` | `1000.0` | Far clipping distance |
| `sensor_width` | `float` | `36.0` | Sensor width in mm |
| `sensor_height` | `float` | `24.0` | Sensor height in mm |
| `sensor_fit` | `str` | `'AUTO'` | `'AUTO'`, `'HORIZONTAL'`, `'VERTICAL'` |
| `shift_x` | `float` | `0.0` | Horizontal lens shift |
| `shift_y` | `float` | `0.0` | Vertical lens shift |

### Camera Depth of Field (cam_data.dof)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `use_dof` | `bool` | `False` | Enable depth of field |
| `focus_object` | `Object` | `None` | Object to use as focus point |
| `focus_distance` | `float` | `10.0` | Manual focus distance (when no focus_object) |
| `aperture_fstop` | `float` | `2.8` | F-stop value |
| `aperture_blades` | `int` | `0` | Number of blades (0 = circular) |
| `aperture_rotation` | `float` | `0.0` | Blade rotation in radians |
| `aperture_ratio` | `float` | `1.0` | Anamorphic ratio |

---

## bpy.types.Light (bpy.data.lights)

### Light Creation

```python
light_data = bpy.data.lights.new(name, type=TYPE)
# TYPE: 'POINT', 'SUN', 'SPOT', 'AREA'
```

### Common Properties (all light types)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `type` | `str` | — | `'POINT'`, `'SUN'`, `'SPOT'`, `'AREA'` (set at creation) |
| `color` | `float[3]` | `(1,1,1)` | RGB color |
| `energy` | `float` | `10.0` | Light power in Watts (point/spot/area) or arbitrary (sun) |
| `specular_factor` | `float` | `1.0` | Specular contribution multiplier |
| `use_shadow` | `bool` | `True` | Enable shadow casting |
| `shadow_soft_size` | `float` | `0.25` | Shadow softness (point/spot only) |

### Spot Light Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `spot_size` | `float` | `0.785` | Cone angle in radians |
| `spot_blend` | `float` | `0.15` | Edge softness (0 = hard, 1 = fully soft) |
| `show_cone` | `bool` | `False` | Display cone shape in viewport |

### Area Light Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `shape` | `str` | `'SQUARE'` | `'SQUARE'`, `'RECTANGLE'`, `'DISK'`, `'ELLIPSE'` |
| `size` | `float` | `0.25` | Size (width for rectangle, radius for disk) |
| `size_y` | `float` | `0.25` | Height (only for `'RECTANGLE'` and `'ELLIPSE'`) |

### Sun Light Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `angle` | `float` | `0.00918` | Angular diameter in radians (shadow softness) |

---

## bpy.ops.render

### render.render()

```python
bpy.ops.render.render(
    animation=False,      # bool — Render full frame range
    write_still=False,    # bool — Save image to filepath
    use_viewport=False,   # bool — Use viewport render settings
    scene="",             # str  — Scene name (empty = active scene)
    layer="",             # str  — Render layer name (empty = all)
)
```

**ALWAYS** pass `write_still=True` for still images. Without it, the image is rendered to memory but never written to disk.

### render.view_show()

Opens the render result window. No parameters. Only works with a GUI (not in background mode).

### render.view_cancel()

Cancel the current render. No parameters.

---

## Compositing (Version-Dependent)

### Blender 3.x / 4.x

```python
scene.use_nodes = True       # Enable compositor
tree = scene.node_tree       # Access compositor node tree
```

### Blender 5.0+

```python
# scene.use_nodes is deprecated (always True)
# scene.node_tree is removed
tree = scene.compositing_node_group  # New accessor
```
