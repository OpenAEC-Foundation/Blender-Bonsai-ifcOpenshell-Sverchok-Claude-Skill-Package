---
name: blender-core-gpu
description: >
  Use when drawing custom overlays, viewport visualizations, or offscreen renders in
  Blender Python. Prevents the critical mistake of using the deprecated bgl module
  (removed in Blender 5.0) instead of the gpu module. Covers gpu.shader, gpu.batch,
  gpu.state, SpaceView3D draw handlers, built-in shaders, and BGL-to-gpu migration.
  Keywords: gpu module, bgl, draw handler, viewport overlay, offscreen rendering,
  SpaceView3D, shader, UNIFORM_COLOR, Blender Python drawing.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-core-gpu

## Quick Reference

### Critical Warnings

**NEVER** use the `bgl` module — it is deprecated since Blender 3.5 and **completely removed in Blender 5.0**. ALL drawing code MUST use the `gpu` module.

**NEVER** use shader name `3D_UNIFORM_COLOR` or `3D_FLAT_COLOR` in Blender 4.0+ — the `3D_` prefix was removed. Use `UNIFORM_COLOR`, `POLYLINE_UNIFORM_COLOR`, etc.

**NEVER** forget to restore `gpu.state` to defaults at the end of draw callbacks — failing to restore state corrupts rendering for ALL subsequent draw handlers and Blender's own UI.

**NEVER** create `gpu.types.GPUOffScreen` outside a valid OpenGL/GPU context — offscreen buffers require a graphics context (viewport draw callback or `bpy.types.SpaceView3D`).

**NEVER** call `gpu.shader.from_builtin()` or create batches from a background thread — ALL GPU operations MUST execute on the main thread.

**ALWAYS** remove draw handlers in addon `unregister()` — leaked handlers cause crashes or persistent ghost overlays.

**ALWAYS** set `viewportSize` and `lineWidth` uniforms when using `POLYLINE_*` shaders — these are REQUIRED uniforms (Blender 4.0+).

**ALWAYS** call `offscreen.free()` when done with offscreen buffers to prevent GPU memory leaks.

### Module Overview

| Module | Purpose |
|--------|---------|
| `gpu` | Core GPU module — shader creation, state management |
| `gpu.shader` | Shader compilation, built-in shader access |
| `gpu.types` | GPU types: `GPUShader`, `GPUBatch`, `GPUOffScreen`, `GPUTexture` |
| `gpu.state` | OpenGL-like state: blend, depth, line width, viewport |
| `gpu.texture` | Texture creation from images |
| `gpu.matrix` | Model-view-projection matrix stack |
| `gpu.select` | GPU-based selection utilities |
| `gpu.platform` | GPU backend info (vendor, renderer, version) |
| `gpu_extras.batch` | `batch_for_shader()` helper |
| `gpu_extras.presets` | `draw_circle_2d()`, `draw_texture_2d()` |

### Built-in Shaders (Blender 4.0+)

| Shader Name | Geometry | Use Case |
|-------------|----------|----------|
| `'UNIFORM_COLOR'` | Triangles, points | Flat colored filled geometry |
| `'POLYLINE_UNIFORM_COLOR'` | Lines | Lines with configurable width |
| `'FLAT_COLOR'` | Triangles | Per-vertex colored (flat shaded) |
| `'POLYLINE_FLAT_COLOR'` | Lines | Per-vertex colored lines |
| `'SMOOTH_COLOR'` | Triangles | Per-vertex colored (smooth shaded) |
| `'POLYLINE_SMOOTH_COLOR'` | Lines | Smooth per-vertex colored lines |
| `'IMAGE'` | Triangles | Textured quads/geometry |
| `'IMAGE_COLOR'` | Triangles | Textured with color tint |

#### Shader Name Version Map

| Blender 3.x Name | Blender 4.0+ Name |
|-------------------|-------------------|
| `'3D_UNIFORM_COLOR'` | `'UNIFORM_COLOR'` |
| `'3D_FLAT_COLOR'` | `'FLAT_COLOR'` |
| `'3D_SMOOTH_COLOR'` | `'SMOOTH_COLOR'` |
| `'3D_IMAGE'` | `'IMAGE'` |
| `'2D_UNIFORM_COLOR'` | `'UNIFORM_COLOR'` |
| `'2D_FLAT_COLOR'` | `'FLAT_COLOR'` |
| `'2D_SMOOTH_COLOR'` | `'SMOOTH_COLOR'` |
| `'2D_IMAGE'` | `'IMAGE'` |

In Blender 4.0+, the `2D_`/`3D_` distinction is removed. A single `'UNIFORM_COLOR'` works in both 2D and 3D contexts.

---

## Essential Patterns

### Pattern 1: Basic Line Drawing with POLYLINE Shader

```python
# Blender 4.0+/5.x — Draw colored lines in 3D viewport
import bpy
import gpu
from gpu_extras.batch import batch_for_shader

def draw_lines():
    coords = [(0, 0, 0), (5, 0, 0), (5, 5, 0), (0, 5, 0), (0, 0, 0)]
    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINE_STRIP', {"pos": coords})

    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(2.0)

    shader.bind()
    region = bpy.context.region
    shader.uniform_float("viewportSize", (region.width, region.height))
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (1.0, 0.0, 0.0, 0.8))
    batch.draw(shader)

    # ALWAYS restore state
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)

_handle = bpy.types.SpaceView3D.draw_handler_add(draw_lines, (), 'WINDOW', 'POST_VIEW')
# Remove: bpy.types.SpaceView3D.draw_handler_remove(_handle, 'WINDOW')
```

### Pattern 2: Filled Triangles with UNIFORM_COLOR

```python
# Blender 4.0+/5.x — Draw a filled quad (two triangles)
import gpu
from gpu_extras.batch import batch_for_shader

def draw_filled():
    verts = [(0, 0, 0), (5, 0, 0), (5, 5, 0), (0, 5, 0)]
    indices = ((0, 1, 2), (2, 3, 0))
    shader = gpu.shader.from_builtin('UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'TRIS', {"pos": verts}, indices=indices)

    gpu.state.blend_set('ALPHA')
    shader.bind()
    shader.uniform_float("color", (0.0, 0.5, 1.0, 0.3))
    batch.draw(shader)
    gpu.state.blend_set('NONE')
```

### Pattern 3: Draw Handler Registration/Removal

```python
# Blender 3.x/4.x/5.x — Proper handler lifecycle in an addon
import bpy

_draw_handle = None

def register():
    global _draw_handle
    _draw_handle = bpy.types.SpaceView3D.draw_handler_add(
        draw_callback, (), 'WINDOW', 'POST_VIEW'
    )

def unregister():
    global _draw_handle
    if _draw_handle is not None:
        bpy.types.SpaceView3D.draw_handler_remove(_draw_handle, 'WINDOW')
        _draw_handle = None
```

#### Draw Handler Layer Options

| Layer | When | Coordinate Space |
|-------|------|------------------|
| `'PRE_VIEW'` | Before 3D scene | 3D world space |
| `'POST_VIEW'` | After 3D scene, before gizmos | 3D world space |
| `'POST_PIXEL'` | After everything, on top | 2D pixel space |

### Pattern 4: Version-Compatible Shader Selection

```python
# Blender 3.x/4.x/5.x — Version-safe shader selection
import bpy
import gpu

def get_line_shader():
    if bpy.app.version >= (4, 0, 0):
        return gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    else:
        return gpu.shader.from_builtin('3D_UNIFORM_COLOR')

def get_flat_shader():
    if bpy.app.version >= (4, 0, 0):
        return gpu.shader.from_builtin('UNIFORM_COLOR')
    else:
        return gpu.shader.from_builtin('3D_UNIFORM_COLOR')
```

### Pattern 5: Offscreen Rendering

```python
# Blender 4.x/5.x — Render to offscreen buffer
import gpu

offscreen = gpu.types.GPUOffScreen(512, 512)
with offscreen.bind():
    fb = gpu.state.active_framebuffer_get()
    fb.clear(color=(0.0, 0.0, 0.0, 0.0))
    # Draw operations here (shaders, batches)

texture = offscreen.texture_color  # GPUTexture
offscreen.free()  # ALWAYS free when done
```

### Pattern 6: GPU State Management

```python
# Blender 3.5+/4.x/5.x — Full state management pattern
import gpu

def draw_with_state():
    # Save implicit state by setting explicitly
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.depth_mask_set(True)
    gpu.state.line_width_set(2.0)
    gpu.state.point_size_set(5.0)
    gpu.state.face_culling_set('BACK')

    # ... draw calls ...

    # ALWAYS restore defaults
    gpu.state.blend_set('NONE')
    gpu.state.depth_test_set('NONE')
    gpu.state.depth_mask_set(False)
    gpu.state.line_width_set(1.0)
    gpu.state.point_size_set(1.0)
    gpu.state.face_culling_set('NONE')
```

---

## Common Operations

### Drawing 2D Overlays (POST_PIXEL)

```python
# Blender 4.0+/5.x — 2D HUD overlay in pixel coordinates
import bpy
import gpu
from gpu_extras.batch import batch_for_shader

def draw_2d_overlay():
    verts = [(10, 10), (200, 10), (200, 50), (10, 50)]
    indices = ((0, 1, 2), (2, 3, 0))
    shader = gpu.shader.from_builtin('UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'TRIS', {"pos": verts}, indices=indices)

    gpu.state.blend_set('ALPHA')
    shader.bind()
    shader.uniform_float("color", (0.0, 0.0, 0.0, 0.5))
    batch.draw(shader)
    gpu.state.blend_set('NONE')

# POST_PIXEL = 2D pixel space overlay
_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_2d_overlay, (), 'WINDOW', 'POST_PIXEL'
)
```

### Drawing Textured Quads

```python
# Blender 4.0+/5.x — Draw image texture in viewport
import gpu
from gpu_extras.batch import batch_for_shader

def draw_image(texture):
    shader = gpu.shader.from_builtin('IMAGE')
    batch = batch_for_shader(
        shader, 'TRI_FAN',
        {
            "pos": ((0, 0), (200, 0), (200, 200), (0, 200)),
            "texCoord": ((0, 0), (1, 0), (1, 1), (0, 1)),
        },
    )
    shader.bind()
    shader.uniform_sampler("image", texture)
    batch.draw(shader)
```

### Per-Vertex Colored Geometry

```python
# Blender 4.0+/5.x — Smooth colored triangle
import gpu
from gpu_extras.batch import batch_for_shader

shader = gpu.shader.from_builtin('SMOOTH_COLOR')
batch = batch_for_shader(shader, 'TRIS', {
    "pos": [(0, 0, 0), (5, 0, 0), (2.5, 5, 0)],
    "color": [(1, 0, 0, 1), (0, 1, 0, 1), (0, 0, 1, 1)],
})
shader.bind()
batch.draw(shader)
```

### Loading Image as GPU Texture

```python
# Blender 4.0+/5.x — Load image to GPU texture (replaces bgl texture binding)
import bpy
import gpu

image = bpy.data.images.load("/path/to/image.png")
texture = gpu.texture.from_image(image)
# Use texture with IMAGE shader via shader.uniform_sampler("image", texture)
```

---

## BGL-to-gpu Migration Checklist

When migrating code from `bgl` to `gpu`:

1. **Remove** ALL `import bgl` statements
2. **Replace** `bgl.glEnable(bgl.GL_BLEND)` → `gpu.state.blend_set('ALPHA')`
3. **Replace** `bgl.glDisable(bgl.GL_BLEND)` → `gpu.state.blend_set('NONE')`
4. **Replace** `bgl.glLineWidth(w)` → `gpu.state.line_width_set(w)`
5. **Replace** `bgl.glEnable(bgl.GL_DEPTH_TEST)` → `gpu.state.depth_test_set('LESS_EQUAL')`
6. **Replace** `bgl.glDisable(bgl.GL_DEPTH_TEST)` → `gpu.state.depth_test_set('NONE')`
7. **Replace** `bgl.glDepthMask(GL_TRUE)` → `gpu.state.depth_mask_set(True)`
8. **Replace** `bgl.glPointSize(s)` → `gpu.state.point_size_set(s)`
9. **Replace** `bgl.glEnable(bgl.GL_LINE_SMOOTH)` → use `POLYLINE_*` shaders (no direct replacement)
10. **Replace** `image.gl_load()` / `bgl.glBindTexture()` → `gpu.texture.from_image(image)`
11. **Replace** `3D_UNIFORM_COLOR` → `POLYLINE_UNIFORM_COLOR` (for lines) or `UNIFORM_COLOR` (for triangles)
12. **Replace** `3D_FLAT_COLOR` → `POLYLINE_FLAT_COLOR` (for lines) or `FLAT_COLOR` (for triangles)
13. **Add** `viewportSize` and `lineWidth` uniforms for ALL `POLYLINE_*` shaders
14. **Add** state restoration at end of every draw callback
15. **Test** on Apple Silicon (Metal backend) — bgl was already non-functional there

### BGL-to-gpu API Mapping

| bgl (REMOVED in 5.0) | gpu replacement (3.5+) | Notes |
|---|---|---|
| `bgl.glEnable(bgl.GL_BLEND)` | `gpu.state.blend_set('ALPHA')` | Options: `'ALPHA'`, `'ADDITIVE'`, `'ADDITIVE_PREMUL'`, `'MULTIPLY'` |
| `bgl.glDisable(bgl.GL_BLEND)` | `gpu.state.blend_set('NONE')` | |
| `bgl.glLineWidth(w)` | `gpu.state.line_width_set(w)` | Also set `lineWidth` uniform for POLYLINE shaders |
| `bgl.glEnable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('LESS_EQUAL')` | Options: `'NONE'`, `'LESS'`, `'LESS_EQUAL'`, `'EQUAL'`, `'GREATER'` |
| `bgl.glDisable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('NONE')` | |
| `bgl.glDepthMask(GL_TRUE)` | `gpu.state.depth_mask_set(True)` | |
| `bgl.glPointSize(s)` | `gpu.state.point_size_set(s)` | |
| `bgl.glEnable(bgl.GL_LINE_SMOOTH)` | *(no direct replacement)* | Use `POLYLINE_*` shaders for antialiased lines |
| `image.gl_load()` / `bgl.glBindTexture()` | `gpu.texture.from_image(image)` | Completely different API |

---

## Version Timeline

| Version | Change |
|---------|--------|
| Blender 3.0 | `gpu` module stable, `gpu_extras.batch` available |
| Blender 3.5 | `bgl` module officially deprecated |
| Blender 4.0 | `3D_`/`2D_` shader name prefixes removed; `POLYLINE_*` shaders added |
| Blender 5.0 | `bgl` module completely removed; `gpu` is the ONLY drawing API |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for gpu, gpu.shader, gpu.state, gpu.types, gpu_extras
- [references/examples.md](references/examples.md) — Working code examples for all drawing scenarios
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with GPU drawing

### Cross-References

- [blender-core-api](../blender-core-api/SKILL.md) — bpy.context, draw handler registration, restricted contexts

### Official Sources

- https://docs.blender.org/api/current/gpu.html
- https://docs.blender.org/api/current/gpu.shader.html
- https://docs.blender.org/api/current/gpu.types.html
- https://docs.blender.org/api/current/gpu.state.html
- https://docs.blender.org/api/current/gpu_extras.batch.html
- https://docs.blender.org/api/current/gpu_extras.presets.html
