# blender-core-gpu — Methods Reference

Complete API signatures for the `gpu` module and related submodules.

---

## gpu.shader

### gpu.shader.from_builtin(shader_name: str) → GPUShader

Returns a built-in shader by name.

```python
# Blender 4.0+/5.x
shader = gpu.shader.from_builtin('UNIFORM_COLOR')
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
shader = gpu.shader.from_builtin('SMOOTH_COLOR')
shader = gpu.shader.from_builtin('POLYLINE_SMOOTH_COLOR')
shader = gpu.shader.from_builtin('FLAT_COLOR')
shader = gpu.shader.from_builtin('POLYLINE_FLAT_COLOR')
shader = gpu.shader.from_builtin('IMAGE')
shader = gpu.shader.from_builtin('IMAGE_COLOR')
```

**Blender 3.x names** (REMOVED in 4.0):
`'3D_UNIFORM_COLOR'`, `'3D_FLAT_COLOR'`, `'3D_SMOOTH_COLOR'`, `'3D_IMAGE'`,
`'2D_UNIFORM_COLOR'`, `'2D_FLAT_COLOR'`, `'2D_SMOOTH_COLOR'`, `'2D_IMAGE'`

### gpu.shader.create_from_info(shader_info: GPUShaderCreateInfo) → GPUShader

Creates a shader from a `GPUShaderCreateInfo` object (advanced custom shaders).

```python
# Blender 3.4+
shader_info = gpu.types.GPUShaderCreateInfo()
# ... configure shader_info ...
shader = gpu.shader.create_from_info(shader_info)
```

### gpu.shader.unbind() → None

Unbinds the active shader.

```python
gpu.shader.unbind()
```

---

## gpu.types.GPUShader

### GPUShader.bind() → None

Binds the shader for drawing. MUST be called before setting uniforms or drawing batches.

```python
shader.bind()
```

### GPUShader.uniform_float(name: str, value) → None

Sets a float uniform. Value can be float, tuple, or list.

```python
shader.uniform_float("color", (1.0, 0.0, 0.0, 1.0))        # vec4
shader.uniform_float("lineWidth", 2.0)                       # float
shader.uniform_float("viewportSize", (1920.0, 1080.0))       # vec2
shader.uniform_float("ModelViewProjectionMatrix", matrix)     # mat4
```

### GPUShader.uniform_int(name: str, value) → None

Sets an integer uniform.

```python
shader.uniform_int("image", 0)  # Texture slot
```

### GPUShader.uniform_bool(name: str, value) → None

Sets a boolean uniform.

```python
shader.uniform_bool("srgbTarget", [True])
```

### GPUShader.uniform_sampler(name: str, texture: GPUTexture) → None

Binds a texture to a sampler uniform.

```python
shader.uniform_sampler("image", texture)
```

### GPUShader.uniform_block(name: str, ubo: GPUUniformBuf) → None

Binds a uniform buffer object to a named block.

```python
ubo = gpu.types.GPUUniformBuf(data)
shader.uniform_block("block_name", ubo)
```

### GPUShader.attr_from_name(name: str) → int

Returns the attribute location by name.

```python
loc = shader.attr_from_name("pos")
```

### GPUShader.format_calc() → GPUVertFormat

Calculates the vertex format for this shader.

```python
fmt = shader.format_calc()
```

### GPUShader.name → str (read-only)

Returns the shader name.

---

## gpu_extras.batch

### batch_for_shader(shader, type, content, *, indices=None) → GPUBatch

Creates a `GPUBatch` from shader attributes.

**Parameters:**
- `shader` (`GPUShader`) — The shader to create the batch for
- `type` (`str`) — Primitive type: `'POINTS'`, `'LINES'`, `'TRIS'`, `'LINE_STRIP'`, `'LINE_LOOP'`, `'TRI_STRIP'`, `'TRI_FAN'`, `'LINES_ADJ'`, `'TRIS_ADJ'`, `'LINE_STRIP_ADJ'`
- `content` (`dict`) — Dict of attribute name → list of values
- `indices` (`tuple/list`, optional) — Index buffer for indexed drawing

```python
from gpu_extras.batch import batch_for_shader

# Lines
batch = batch_for_shader(shader, 'LINES', {"pos": [(0,0,0), (1,0,0)]})

# Indexed triangles
batch = batch_for_shader(
    shader, 'TRIS',
    {"pos": [(0,0,0), (1,0,0), (0.5,1,0)]},
    indices=[(0, 1, 2)]
)

# Per-vertex color
batch = batch_for_shader(shader, 'TRIS', {
    "pos": [(0,0,0), (1,0,0), (0.5,1,0)],
    "color": [(1,0,0,1), (0,1,0,1), (0,0,1,1)],
})

# Textured quad
batch = batch_for_shader(shader, 'TRI_FAN', {
    "pos": ((0, 0), (100, 0), (100, 100), (0, 100)),
    "texCoord": ((0, 0), (1, 0), (1, 1), (0, 1)),
})
```

### Attribute Names per Built-in Shader

| Shader | Required Attributes |
|--------|-------------------|
| `UNIFORM_COLOR` | `pos` |
| `POLYLINE_UNIFORM_COLOR` | `pos` |
| `FLAT_COLOR` | `pos`, `color` |
| `POLYLINE_FLAT_COLOR` | `pos`, `color` |
| `SMOOTH_COLOR` | `pos`, `color` |
| `POLYLINE_SMOOTH_COLOR` | `pos`, `color` |
| `IMAGE` | `pos`, `texCoord` |
| `IMAGE_COLOR` | `pos`, `texCoord`, `color` |

### Uniform Names per Built-in Shader

| Shader | Required Uniforms |
|--------|------------------|
| `UNIFORM_COLOR` | `color` (vec4) |
| `POLYLINE_UNIFORM_COLOR` | `color` (vec4), `lineWidth` (float), `viewportSize` (vec2) |
| `FLAT_COLOR` | *(none — color from attribute)* |
| `POLYLINE_FLAT_COLOR` | `lineWidth` (float), `viewportSize` (vec2) |
| `SMOOTH_COLOR` | *(none — color from attribute)* |
| `POLYLINE_SMOOTH_COLOR` | `lineWidth` (float), `viewportSize` (vec2) |
| `IMAGE` | `image` (sampler) |
| `IMAGE_COLOR` | `image` (sampler), `color` (vec4) |

---

## gpu.types.GPUBatch

### GPUBatch.draw(shader: GPUShader) → None

Draws the batch using the specified shader. Shader MUST be bound first.

```python
shader.bind()
batch.draw(shader)
```

### GPUBatch.program_set(shader: GPUShader) → None

Sets the shader program for this batch.

### GPUBatch.vertbuf_add(buf: GPUVertBuf) → None

Adds a vertex buffer to this batch.

---

## gpu.state

All functions below are available in Blender 3.5+ (when `gpu.state` was stabilized).

### gpu.state.blend_set(mode: str) → None

Sets blend mode.

**Options:** `'NONE'`, `'ALPHA'`, `'ALPHA_PREMULT'`, `'ADDITIVE'`, `'ADDITIVE_PREMULT'`, `'MULTIPLY'`, `'SUBTRACT'`, `'INVERT'`

```python
gpu.state.blend_set('ALPHA')     # Standard alpha blending
gpu.state.blend_set('NONE')      # Disable blending (default)
```

### gpu.state.blend_get() → str

Returns the current blend mode.

### gpu.state.depth_test_set(mode: str) → None

Sets depth testing mode.

**Options:** `'NONE'`, `'ALWAYS'`, `'LESS'`, `'LESS_EQUAL'`, `'EQUAL'`, `'GREATER'`, `'GREATER_EQUAL'`

```python
gpu.state.depth_test_set('LESS_EQUAL')  # Standard depth testing
gpu.state.depth_test_set('NONE')         # Disable (default)
```

### gpu.state.depth_test_get() → str

Returns the current depth test mode.

### gpu.state.depth_mask_set(value: bool) → None

Enables/disables writing to the depth buffer.

```python
gpu.state.depth_mask_set(True)   # Enable depth writes
gpu.state.depth_mask_set(False)  # Disable (default)
```

### gpu.state.depth_mask_get() → bool

Returns whether depth writing is enabled.

### gpu.state.line_width_set(width: float) → None

Sets the global line width in pixels.

```python
gpu.state.line_width_set(2.0)   # 2-pixel wide lines
gpu.state.line_width_set(1.0)   # Default
```

### gpu.state.line_width_get() → float

Returns the current line width.

### gpu.state.point_size_set(size: float) → None

Sets the point size in pixels.

```python
gpu.state.point_size_set(5.0)   # 5-pixel points
gpu.state.point_size_set(1.0)   # Default
```

### gpu.state.point_size_get() → float

Returns the current point size.

### gpu.state.face_culling_set(mode: str) → None

Sets face culling mode.

**Options:** `'NONE'`, `'FRONT'`, `'BACK'`

```python
gpu.state.face_culling_set('BACK')   # Cull back faces
gpu.state.face_culling_set('NONE')   # No culling (default)
```

### gpu.state.front_facing_set(invert: bool) → None

Sets whether to invert front-facing detection.

### gpu.state.program_point_size_set(enable: bool) → None

Enables/disables program-controlled point size.

### gpu.state.viewport_set(x: int, y: int, width: int, height: int) → None

Sets the viewport rectangle.

### gpu.state.viewport_get() → tuple[int, int, int, int]

Returns the current viewport as `(x, y, width, height)`.

```python
x, y, w, h = gpu.state.viewport_get()
```

### gpu.state.scissor_set(x: int, y: int, width: int, height: int) → None

Sets the scissor rectangle for clipping.

### gpu.state.scissor_get() → tuple[int, int, int, int]

Returns the current scissor rectangle.

### gpu.state.scissor_test_set(enable: bool) → None

Enables/disables scissor testing.

### gpu.state.active_framebuffer_get() → GPUFrameBuffer

Returns the active framebuffer.

```python
fb = gpu.state.active_framebuffer_get()
fb.clear(color=(0.0, 0.0, 0.0, 0.0))
```

### gpu.state.clip_distances_set(value: int) → None

Sets the number of enabled clip distances.

---

## gpu.types.GPUOffScreen

### GPUOffScreen(width: int, height: int, *, format: str = 'RGBA8') → GPUOffScreen

Creates an offscreen render buffer.

```python
offscreen = gpu.types.GPUOffScreen(512, 512)
offscreen = gpu.types.GPUOffScreen(1024, 1024, format='RGBA16F')
```

### GPUOffScreen.bind() → context manager

Binds the offscreen buffer for drawing.

```python
with offscreen.bind():
    fb = gpu.state.active_framebuffer_get()
    fb.clear(color=(0.0, 0.0, 0.0, 0.0))
    # ... draw operations ...
```

### GPUOffScreen.unbind(restore: bool = True) → None

Unbinds the offscreen buffer.

### GPUOffScreen.free() → None

Frees GPU memory. ALWAYS call when done.

### GPUOffScreen.texture_color → GPUTexture (read-only)

Returns the color texture of the offscreen buffer.

### GPUOffScreen.width → int (read-only)

### GPUOffScreen.height → int (read-only)

---

## gpu.texture

### gpu.texture.from_image(image: bpy.types.Image) → GPUTexture

Creates a GPU texture from a Blender image data block.

```python
import bpy
import gpu

image = bpy.data.images.load("/path/to/image.png")
texture = gpu.texture.from_image(image)
```

**Replaces** (Blender < 5.0): `image.gl_load()` / `bgl.glBindTexture()` / `image.bindcode`

---

## gpu.matrix

### gpu.matrix.get_model_view_matrix() → Matrix

Returns the current model-view matrix.

### gpu.matrix.get_projection_matrix() → Matrix

Returns the current projection matrix.

### gpu.matrix.get_normal_matrix() → Matrix

Returns the current normal matrix.

### gpu.matrix.push() / gpu.matrix.pop() → None

Push/pop the matrix stack.

```python
gpu.matrix.push()
gpu.matrix.translate((1.0, 0.0, 0.0))
# ... draw ...
gpu.matrix.pop()
```

### gpu.matrix.push_projection() / gpu.matrix.pop_projection() → None

Push/pop the projection matrix stack.

### gpu.matrix.load_matrix(matrix: Matrix) → None

Loads a model-view matrix.

### gpu.matrix.load_projection_matrix(matrix: Matrix) → None

Loads a projection matrix.

### gpu.matrix.load_identity() → None

Loads the identity matrix.

### gpu.matrix.multiply_matrix(matrix: Matrix) → None

Multiplies the current model-view matrix.

### gpu.matrix.scale(scale: tuple) → None

Applies scale transformation.

### gpu.matrix.scale_uniform(scale: float) → None

Applies uniform scale.

### gpu.matrix.translate(offset: tuple) → None

Applies translation.

---

## gpu_extras.presets

### draw_circle_2d(position, color, radius, *, segments=32)

Draws a 2D circle outline.

```python
from gpu_extras.presets import draw_circle_2d
draw_circle_2d((100, 100), (1.0, 1.0, 1.0, 1.0), 50)
```

### draw_texture_2d(texture, position, width, height)

Draws a textured 2D rectangle.

```python
from gpu_extras.presets import draw_texture_2d
draw_texture_2d(texture, (0, 0), 256, 256)
```

---

## gpu.platform

### gpu.platform.vendor_get() → str

Returns the GPU vendor string.

### gpu.platform.renderer_get() → str

Returns the GPU renderer string.

### gpu.platform.version_get() → str

Returns the GPU driver version string.

### gpu.platform.device_type_get() → str

Returns the device type (e.g., `'APPLE'`, `'NVIDIA'`, `'AMD'`, `'INTEL'`).

### gpu.platform.backend_type_get() → str

Returns the graphics backend: `'OPENGL'`, `'VULKAN'`, or `'METAL'`.

```python
backend = gpu.platform.backend_type_get()
# Returns 'OPENGL', 'VULKAN', or 'METAL'
```

---

## SpaceView3D Draw Handler API

### bpy.types.SpaceView3D.draw_handler_add(callback, args, region_type, draw_type) → handle

Registers a draw callback.

**Parameters:**
- `callback` (`callable`) — Function to call during drawing
- `args` (`tuple`) — Arguments to pass to callback
- `region_type` (`str`) — `'WINDOW'`, `'HEADER'`, `'UI'`, `'TOOLS'`
- `draw_type` (`str`) — `'PRE_VIEW'`, `'POST_VIEW'`, `'POST_PIXEL'`, `'BACKDROP'`

```python
handle = bpy.types.SpaceView3D.draw_handler_add(
    my_draw_func, (arg1, arg2), 'WINDOW', 'POST_VIEW'
)
```

### bpy.types.SpaceView3D.draw_handler_remove(handle, region_type) → None

Removes a previously registered draw handler.

```python
bpy.types.SpaceView3D.draw_handler_remove(handle, 'WINDOW')
```

**ALWAYS** call this in addon `unregister()` to prevent leaked handlers.
