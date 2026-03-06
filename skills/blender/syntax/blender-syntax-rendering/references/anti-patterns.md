# blender-syntax-rendering — Anti-Patterns

What NOT to do when scripting Blender rendering, with explanations of WHY each pattern fails.

---

## AP-1: Wrong EEVEE Identifier for Blender Version

**WRONG:**
```python
# Fails in Blender 4.2-4.4 — engine identifier was temporarily changed
scene.render.engine = 'BLENDER_EEVEE'
```

**WHY IT FAILS:** In Blender 4.2 through 4.4, the EEVEE engine was rewritten and the identifier changed to `'BLENDER_EEVEE_NEXT'`. Using `'BLENDER_EEVEE'` in those versions raises a runtime error or silently fails. In Blender 5.0+, the identifier reverted back to `'BLENDER_EEVEE'`.

**CORRECT:**
```python
if bpy.app.version >= (5, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE'
elif bpy.app.version >= (4, 2, 0):
    scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    scene.render.engine = 'BLENDER_EEVEE'
```

---

## AP-2: Rendering Without write_still=True

**WRONG:**
```python
bpy.ops.render.render()  # Image rendered but never saved
```

**WHY IT FAILS:** `bpy.ops.render.render()` without `write_still=True` renders the image to memory only. The result exists in `bpy.data.images["Render Result"]` but is never written to disk. This is the #1 mistake when automating renders.

**CORRECT:**
```python
scene.render.filepath = "/tmp/output.png"
bpy.ops.render.render(write_still=True)
```

---

## AP-3: Setting Cycles GPU Without get_devices()

**WRONG:**
```python
cycles_prefs = bpy.context.preferences.addons['cycles'].preferences
cycles_prefs.compute_device_type = 'CUDA'
# Skipping get_devices() — device list is empty
for device in cycles_prefs.devices:
    device.use = True  # Never executes — no devices in list
```

**WHY IT FAILS:** The `devices` collection is not populated until `get_devices()` is explicitly called. Without it, the loop iterates over an empty collection, no GPU is enabled, and Cycles silently falls back to CPU rendering.

**CORRECT:**
```python
cycles_prefs = bpy.context.preferences.addons['cycles'].preferences
cycles_prefs.compute_device_type = 'CUDA'
cycles_prefs.get_devices()  # REQUIRED: populates device list
for device in cycles_prefs.devices:
    device.use = True
```

---

## AP-4: Using Old Render Pass Names in Blender 5.0+

**WRONG:**
```python
# Blender 5.0+ — old short names no longer exist
depth_pass = render_result.layers[0].passes["Z"]  # KeyError in 5.0+
```

**WHY IT FAILS:** Blender 5.0 renamed all render pass identifiers from abbreviated forms to full words. `"Z"` became `"Depth"`, `"DiffCol"` became `"Diffuse Color"`, etc.

**CORRECT:**
```python
if bpy.app.version >= (5, 0, 0):
    depth_pass = render_result.layers[0].passes["Depth"]
else:
    depth_pass = render_result.layers[0].passes["Z"]
```

---

## AP-5: Using scene.use_nodes in Blender 5.0+

**WRONG:**
```python
# Blender 5.0+ — deprecated attribute
scene.use_nodes = True  # No effect; compositor is always active
```

**WHY IT FAILS:** In Blender 5.0+, the compositor is always active. `scene.use_nodes` is deprecated and has no effect. Code that checks `if scene.use_nodes:` may also behave unexpectedly.

**CORRECT:**
```python
# Blender 5.0+: compositor is always active, no need to enable
# For 3.x/4.x compatibility:
if bpy.app.version < (5, 0, 0):
    scene.use_nodes = True
```

---

## AP-6: Using scene.node_tree in Blender 5.0+

**WRONG:**
```python
# Blender 5.0+ — removed attribute
tree = scene.node_tree  # AttributeError in 5.0+
```

**WHY IT FAILS:** `scene.node_tree` was removed in Blender 5.0. The compositor node tree is now accessed via `scene.compositing_node_group`.

**CORRECT:**
```python
if bpy.app.version >= (5, 0, 0):
    tree = scene.compositing_node_group
else:
    scene.use_nodes = True
    tree = scene.node_tree
```

---

## AP-7: Using eevee.use_gtao in Blender 5.0+

**WRONG:**
```python
# Blender 5.0+ — removed attributes
eevee = scene.eevee
eevee.use_gtao = True        # AttributeError
eevee.gtao_distance = 1.0    # AttributeError
```

**WHY IT FAILS:** Ambient occlusion settings were moved in Blender 5.0 from `scene.eevee` to the view layer EEVEE settings.

**CORRECT:**
```python
if bpy.app.version >= (5, 0, 0):
    view_layer = scene.view_layers[0]
    view_layer.eevee.ambient_occlusion_distance = 1.0
else:
    eevee = scene.eevee
    eevee.use_gtao = True
    eevee.gtao_distance = 1.0
```

---

## AP-8: Forgetting to Link Camera to Collection

**WRONG:**
```python
cam_data = bpy.data.cameras.new("Camera")
cam_obj = bpy.data.objects.new("Camera", cam_data)
scene.camera = cam_obj  # Camera exists but is not in any collection
# Render produces black/empty image — camera is orphaned
```

**WHY IT FAILS:** Objects must be linked to a collection to be part of the scene. An unlinked camera object exists in `bpy.data.objects` but is not visible or functional in the scene. Rendering with an unlinked camera produces undefined results.

**CORRECT:**
```python
cam_data = bpy.data.cameras.new("Camera")
cam_obj = bpy.data.objects.new("Camera", cam_data)
bpy.context.collection.objects.link(cam_obj)  # REQUIRED
scene.camera = cam_obj
```

---

## AP-9: Hardcoding File Format Without Matching Color Depth

**WRONG:**
```python
render.image_settings.file_format = 'JPEG'
render.image_settings.color_depth = '16'  # JPEG only supports 8-bit
render.image_settings.color_mode = 'RGBA'  # JPEG does not support alpha
```

**WHY IT FAILS:** JPEG only supports 8-bit color depth and RGB (no alpha channel). Setting incompatible values causes silent fallback or errors depending on the Blender version.

**CORRECT:**
```python
# JPEG: 8-bit, RGB only
render.image_settings.file_format = 'JPEG'
render.image_settings.color_depth = '8'
render.image_settings.color_mode = 'RGB'
render.image_settings.quality = 90

# PNG: 8 or 16-bit, supports RGBA
render.image_settings.file_format = 'PNG'
render.image_settings.color_depth = '16'
render.image_settings.color_mode = 'RGBA'
```

### Format Compatibility Matrix

| Format | Color Depths | Color Modes | Alpha Support |
|--------|-------------|-------------|---------------|
| PNG | 8, 16 | BW, RGB, RGBA | Yes |
| JPEG | 8 | BW, RGB | No |
| OPEN_EXR | 16, 32 | RGB, RGBA | Yes |
| TIFF | 8, 16 | BW, RGB, RGBA | Yes |
| BMP | 8 | RGB | No |
| HDR | 32 | RGB | No |
| WEBP | 8 | RGB, RGBA | Yes |

---

## AP-10: Rendering in Background Mode with Viewport Operators

**WRONG:**
```python
# In headless mode (blender --background)
bpy.ops.render.view_show()  # Fails — no GUI available
```

**WHY IT FAILS:** Background mode (`blender --background`) has no UI context. Viewport-related operators like `render.view_show()` require a GUI window and will fail with a context error.

**CORRECT:**
```python
# In background mode, render directly without viewport display
bpy.ops.render.render(write_still=True)
# Result is saved to render.filepath; no viewport display needed
```

---

## AP-11: Not Setting render.filepath Before Animation Render

**WRONG:**
```python
bpy.ops.render.render(animation=True)
# Uses default filepath "/tmp/" — frames overwrite each other or go to unexpected location
```

**WHY IT FAILS:** When `animation=True`, Blender appends frame numbers to `render.filepath`. If the path is not explicitly set, frames are written to the default temp directory with default naming, making them hard to find and potentially overwriting other renders.

**CORRECT:**
```python
scene.render.filepath = "/tmp/my_animation/frame_"
bpy.ops.render.render(animation=True)
# Produces: frame_0001.png, frame_0002.png, ...
```

---

## AP-12: Using OPTIX Denoiser Without NVIDIA GPU

**WRONG:**
```python
cycles.use_denoising = True
cycles.denoiser = 'OPTIX'  # Fails on AMD/Intel GPUs
```

**WHY IT FAILS:** OptiX denoising is NVIDIA-exclusive. On non-NVIDIA hardware, setting this causes a runtime error or silent fallback to no denoising.

**CORRECT:**
```python
cycles.use_denoising = True
cycles.denoiser = 'OPENIMAGEDENOISE'  # Works on all hardware (CPU-based)
# Use 'OPTIX' only when confirmed NVIDIA GPU is available
```

---

## Summary Table

| # | Anti-Pattern | Versions Affected | Risk |
|---|-------------|-------------------|------|
| AP-1 | Wrong EEVEE identifier | 4.2-4.4 | Engine selection fails |
| AP-2 | Missing write_still=True | All | Render not saved to disk |
| AP-3 | Missing get_devices() | All | GPU not used, CPU fallback |
| AP-4 | Old render pass names | 5.0+ | KeyError on pass lookup |
| AP-5 | scene.use_nodes | 5.0+ | Deprecated, no effect |
| AP-6 | scene.node_tree | 5.0+ | AttributeError |
| AP-7 | eevee.use_gtao | 5.0+ | AttributeError |
| AP-8 | Unlinked camera | All | Empty/black render |
| AP-9 | Mismatched format/depth | All | Silent fallback or error |
| AP-10 | Viewport ops in background | All (headless) | Context error |
| AP-11 | No filepath for animation | All | Files in wrong location |
| AP-12 | OPTIX on non-NVIDIA | All | Denoiser fails |
