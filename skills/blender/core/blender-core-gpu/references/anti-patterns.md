# blender-core-gpu — Anti-Patterns Reference

What NOT to do with GPU drawing in Blender, with explanations of WHY each pattern is wrong.

---

## Anti-Pattern 1: Using bgl Module

### WRONG — bgl is deprecated (3.5) and removed (5.0)

```python
# WRONG — bgl is deprecated since 3.5, REMOVED in 5.0
import bgl

def draw():
    bgl.glEnable(bgl.GL_BLEND)
    bgl.glLineWidth(2)
    bgl.glEnable(bgl.GL_DEPTH_TEST)
    # ... draw ...
    bgl.glDisable(bgl.GL_BLEND)
    bgl.glLineWidth(1)
    bgl.glDisable(bgl.GL_DEPTH_TEST)
```

### CORRECT — gpu.state

```python
# CORRECT — gpu.state (Blender 3.5+/4.x/5.x)
import gpu

def draw():
    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(2.0)
    gpu.state.depth_test_set('LESS_EQUAL')
    # ... draw ...
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)
    gpu.state.depth_test_set('NONE')
```

**WHY**: The `bgl` module is an OpenGL wrapper. Blender now supports Metal (macOS) and Vulkan backends where OpenGL calls have no effect or crash. The `gpu` module provides a backend-independent abstraction. `bgl` is already non-functional on Apple Silicon Macs.

---

## Anti-Pattern 2: Using Old Shader Names in Blender 4.0+

### WRONG — 3D_/2D_ prefix shader names

```python
# WRONG — '3D_UNIFORM_COLOR' was removed in Blender 4.0
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')

# WRONG — '2D_UNIFORM_COLOR' was also removed in Blender 4.0
shader = gpu.shader.from_builtin('2D_UNIFORM_COLOR')
```

### CORRECT — Use unprefixed names

```python
# CORRECT — Blender 4.0+
shader = gpu.shader.from_builtin('UNIFORM_COLOR')          # For triangles/points
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')  # For lines
```

**WHY**: Blender 4.0 unified the 2D and 3D shader variants. The `3D_` and `2D_` prefixes were removed. Using old names raises a `ValueError` in Blender 4.0+.

---

## Anti-Pattern 3: Missing POLYLINE Uniforms

### WRONG — POLYLINE shader without required uniforms

```python
# WRONG — POLYLINE_UNIFORM_COLOR requires viewportSize and lineWidth uniforms
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
batch = batch_for_shader(shader, 'LINES', {"pos": coords})

shader.bind()
shader.uniform_float("color", (1, 0, 0, 1))
batch.draw(shader)  # CRASHES or renders incorrectly — missing uniforms
```

### CORRECT — Set all required uniforms

```python
# CORRECT — All POLYLINE_* shaders require viewportSize and lineWidth
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
batch = batch_for_shader(shader, 'LINES', {"pos": coords})

shader.bind()
region = bpy.context.region
shader.uniform_float("viewportSize", (region.width, region.height))
shader.uniform_float("lineWidth", 2.0)
shader.uniform_float("color", (1, 0, 0, 1))
batch.draw(shader)
```

**WHY**: `POLYLINE_*` shaders compute line width on the GPU and need the viewport dimensions to correctly calculate pixel sizes. Without these uniforms, lines render with zero width or cause GPU errors.

---

## Anti-Pattern 4: Not Restoring GPU State

### WRONG — State leak corrupts other draw handlers

```python
# WRONG — Leaving blend mode and depth test enabled
def draw():
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.line_width_set(3.0)

    shader.bind()
    batch.draw(shader)
    # Missing state restoration! Corrupts all subsequent drawing.
```

### CORRECT — Always restore to defaults

```python
# CORRECT — Restore every state change
def draw():
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.line_width_set(3.0)

    shader.bind()
    batch.draw(shader)

    # ALWAYS restore state
    gpu.state.blend_set('NONE')
    gpu.state.depth_test_set('NONE')
    gpu.state.line_width_set(1.0)
```

**WHY**: GPU state is global. Your draw callback runs alongside Blender's own rendering and other addons' callbacks. Leaving non-default state corrupts ALL subsequent draw operations, causing visual glitches, invisible gizmos, or broken UI elements.

---

## Anti-Pattern 5: Not Removing Draw Handlers on Unregister

### WRONG — Leaked draw handler

```python
# WRONG — Handler leaks when addon is disabled
def register():
    bpy.types.SpaceView3D.draw_handler_add(draw_fn, (), 'WINDOW', 'POST_VIEW')

def unregister():
    pass  # Handler is leaked! Causes ghost overlays or crashes.
```

### CORRECT — Store and remove handle

```python
# CORRECT — Store handle, remove on unregister
_handle = None

def register():
    global _handle
    _handle = bpy.types.SpaceView3D.draw_handler_add(
        draw_fn, (), 'WINDOW', 'POST_VIEW'
    )

def unregister():
    global _handle
    if _handle is not None:
        bpy.types.SpaceView3D.draw_handler_remove(_handle, 'WINDOW')
        _handle = None
```

**WHY**: Draw handlers are NOT automatically cleaned up when an addon is disabled. A leaked handler continues to be called every frame, referencing potentially freed resources. This causes ghost overlays that persist until Blender restarts, or crashes when the callback references deleted objects.

---

## Anti-Pattern 6: Creating Shader/Batch Inside Draw Callback

### WRONG — Recreating shader and batch every frame

```python
# WRONG — Creating shader and batch 60+ times per second
def draw():
    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')  # Created every frame
    batch = batch_for_shader(shader, 'LINES', {"pos": coords})  # Created every frame

    shader.bind()
    # ... draw ...
```

### CORRECT — Cache shader and batch

```python
# CORRECT — Create once, reuse across frames
_shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
_batch = batch_for_shader(_shader, 'LINES', {"pos": coords})

def draw():
    _shader.bind()
    # ... draw using _batch ...
```

**WHY**: Draw callbacks execute every frame (60+ fps). Creating new shader and batch objects inside the callback causes severe GPU memory allocation overhead, leading to frame drops and potential memory leaks. Cache these objects at module level or in a class, and only recreate them when the underlying data changes.

---

## Anti-Pattern 7: GPU Operations from Background Thread

### WRONG — GPU calls from a thread

```python
# WRONG — GPU calls from a background thread
import threading

def background_draw():
    shader = gpu.shader.from_builtin('UNIFORM_COLOR')  # CRASHES
    batch = batch_for_shader(shader, 'TRIS', {"pos": verts})
    offscreen = gpu.types.GPUOffScreen(512, 512)  # CRASHES

threading.Thread(target=background_draw).start()
```

### CORRECT — GPU calls on main thread only

```python
# CORRECT — All GPU operations on the main thread
def main_thread_draw():
    shader = gpu.shader.from_builtin('UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'TRIS', {"pos": verts})

# Use from draw callback (main thread) or bpy.app.timers (main thread)
bpy.types.SpaceView3D.draw_handler_add(main_thread_draw, (), 'WINDOW', 'POST_VIEW')
```

**WHY**: All GPU operations require the main thread's OpenGL/Vulkan/Metal context. GPU calls from background threads cause undefined behavior — typically segmentation faults. Use `bpy.app.timers.register()` to schedule work on the main thread if needed.

---

## Anti-Pattern 8: Not Freeing GPUOffScreen

### WRONG — Offscreen buffer memory leak

```python
# WRONG — offscreen buffer never freed
def render_something():
    offscreen = gpu.types.GPUOffScreen(1024, 1024)
    with offscreen.bind():
        # ... draw ...
    texture = offscreen.texture_color
    # offscreen is never freed! GPU memory leak.
```

### CORRECT — Always free offscreen buffers

```python
# CORRECT — Free when done
def render_something():
    offscreen = gpu.types.GPUOffScreen(1024, 1024)
    with offscreen.bind():
        # ... draw ...
    texture = offscreen.texture_color
    # ... use texture ...
    offscreen.free()  # ALWAYS free
```

**WHY**: `GPUOffScreen` allocates GPU framebuffer memory. Python garbage collection does NOT automatically free GPU resources. Each leaked offscreen buffer permanently consumes GPU VRAM until Blender exits. For a 1024x1024 RGBA buffer, that is 4 MB of VRAM per leak.

---

## Anti-Pattern 9: Using image.gl_load() in Blender 5.0+

### WRONG — Old texture loading API

```python
# WRONG — gl_load() is part of the removed bgl pipeline
image = bpy.data.images.load("/path/to/image.png")
image.gl_load()  # REMOVED in Blender 5.0
bindcode = image.bindcode  # No longer works
```

### CORRECT — gpu.texture.from_image()

```python
# CORRECT — gpu texture API (Blender 3.5+)
image = bpy.data.images.load("/path/to/image.png")
texture = gpu.texture.from_image(image)
# Use with shader.uniform_sampler("image", texture)
```

**WHY**: `image.gl_load()` and `image.bindcode` were part of the raw OpenGL pipeline removed with `bgl`. The `gpu.texture.from_image()` function works across all graphics backends (OpenGL, Vulkan, Metal).

---

## Anti-Pattern 10: Drawing in Restricted Contexts

### WRONG — GPU drawing from Panel.draw()

```python
# WRONG — Panel.draw() is a read-only UI context
class MY_PT_panel(bpy.types.Panel):
    bl_label = "My Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'

    def draw(self, context):
        # GPU drawing in panel draw() — WRONG
        shader = gpu.shader.from_builtin('UNIFORM_COLOR')
        batch = batch_for_shader(shader, 'TRIS', {"pos": verts})
        shader.bind()
        batch.draw(shader)  # Does nothing or causes errors
```

### CORRECT — Use SpaceView3D.draw_handler_add()

```python
# CORRECT — Register a dedicated draw handler
handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_function, (), 'WINDOW', 'POST_VIEW'
)
```

**WHY**: `Panel.draw()` runs in Blender's UI rendering context, which is separate from the 3D viewport's rendering pipeline. GPU draw calls in panel draw callbacks either have no effect or interfere with UI rendering. ALWAYS use `SpaceView3D.draw_handler_add()` for viewport drawing.

---

## Anti-Pattern 11: Ignoring Backend Compatibility

### WRONG — Assuming OpenGL is always available

```python
# WRONG — Making OpenGL-specific assumptions
def draw():
    # Assuming OpenGL is the backend
    # Using hardcoded GL constants or behaviors
    gpu.state.line_width_set(5.0)  # Line width > 1 may not work on all backends
```

### CORRECT — Check backend and handle gracefully

```python
# CORRECT — Backend-aware drawing
def draw():
    backend = gpu.platform.backend_type_get()
    # POLYLINE shaders handle line width consistently across backends
    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    # lineWidth uniform works on all backends
    shader.uniform_float("lineWidth", 5.0)
```

**WHY**: Blender supports OpenGL, Vulkan (Linux/Windows), and Metal (macOS). Wide lines via `gpu.state.line_width_set()` may be clamped to 1.0 on some backends. The `POLYLINE_*` shaders implement line width in the geometry shader, working consistently across all backends.

---

## Anti-Pattern 12: Hardcoding Viewport Dimensions

### WRONG — Hardcoded viewport size

```python
# WRONG — Viewport size is hardcoded
shader.uniform_float("viewportSize", (1920.0, 1080.0))  # Wrong on other resolutions
```

### CORRECT — Query actual viewport size

```python
# CORRECT — Get actual viewport dimensions
region = bpy.context.region
shader.uniform_float("viewportSize", (region.width, region.height))

# Alternative using gpu.state
viewport = gpu.state.viewport_get()
shader.uniform_float("viewportSize", (viewport[2], viewport[3]))
```

**WHY**: Viewport size changes with window resizing, multi-monitor setups, split areas, and different display resolutions. Hardcoded values cause incorrect line widths and positions on any viewport that is not exactly the hardcoded size.
