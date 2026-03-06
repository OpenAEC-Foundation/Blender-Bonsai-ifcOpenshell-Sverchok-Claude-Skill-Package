# blender-core-gpu — Examples Reference

Working code examples for GPU drawing in Blender. All examples are verified against official Blender API documentation.

---

## Example 1: Complete Addon with Viewport Overlay

```python
# Blender 4.0+/5.x — Addon that draws a wireframe box overlay in the 3D viewport
bl_info = {
    "name": "GPU Wireframe Overlay",
    "blender": (4, 0, 0),
    "category": "3D View",
}

import bpy
import gpu
from gpu_extras.batch import batch_for_shader

_draw_handle = None


def draw_wireframe_box():
    """Draw a wireframe box at the world origin."""
    # Define box edges as line pairs
    coords = [
        # Bottom face
        (-1, -1, 0), (1, -1, 0),
        (1, -1, 0), (1, 1, 0),
        (1, 1, 0), (-1, 1, 0),
        (-1, 1, 0), (-1, -1, 0),
        # Top face
        (-1, -1, 2), (1, -1, 2),
        (1, -1, 2), (1, 1, 2),
        (1, 1, 2), (-1, 1, 2),
        (-1, 1, 2), (-1, -1, 2),
        # Vertical edges
        (-1, -1, 0), (-1, -1, 2),
        (1, -1, 0), (1, -1, 2),
        (1, 1, 0), (1, 1, 2),
        (-1, 1, 0), (-1, 1, 2),
    ]

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINES', {"pos": coords})

    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(2.0)

    shader.bind()
    region = bpy.context.region
    shader.uniform_float("viewportSize", (region.width, region.height))
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))
    batch.draw(shader)

    # ALWAYS restore state
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)


def register():
    global _draw_handle
    _draw_handle = bpy.types.SpaceView3D.draw_handler_add(
        draw_wireframe_box, (), 'WINDOW', 'POST_VIEW'
    )


def unregister():
    global _draw_handle
    if _draw_handle is not None:
        bpy.types.SpaceView3D.draw_handler_remove(_draw_handle, 'WINDOW')
        _draw_handle = None


if __name__ == "__main__":
    register()
```

---

## Example 2: Dashed Lines Using Segments

```python
# Blender 4.0+/5.x — Draw dashed lines by creating segmented line pairs
import bpy
import gpu
from gpu_extras.batch import batch_for_shader
from mathutils import Vector


def generate_dashed_coords(start, end, dash_length=0.3, gap_length=0.15):
    """Generate line segment pairs for a dashed line effect."""
    direction = (Vector(end) - Vector(start))
    total_length = direction.length
    direction.normalize()

    coords = []
    dist = 0.0
    while dist < total_length:
        p0 = Vector(start) + direction * dist
        p1_dist = min(dist + dash_length, total_length)
        p1 = Vector(start) + direction * p1_dist
        coords.extend([tuple(p0), tuple(p1)])
        dist += dash_length + gap_length

    return coords


def draw_dashed():
    coords = generate_dashed_coords((0, 0, 0), (10, 0, 0))
    if not coords:
        return

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINES', {"pos": coords})

    gpu.state.blend_set('ALPHA')
    shader.bind()
    region = bpy.context.region
    shader.uniform_float("viewportSize", (region.width, region.height))
    shader.uniform_float("lineWidth", 1.5)
    shader.uniform_float("color", (0.8, 0.8, 0.8, 1.0))
    batch.draw(shader)
    gpu.state.blend_set('NONE')
```

---

## Example 3: Drawing Around Selected Object

```python
# Blender 4.0+/5.x — Draw bounding box around active object
import bpy
import gpu
from gpu_extras.batch import batch_for_shader


def draw_active_object_bounds():
    obj = bpy.context.active_object
    if obj is None:
        return

    # Get world-space bounding box corners
    bbox = [obj.matrix_world @ Vector(corner) for corner in obj.bound_box]

    # 12 edges of a box (pairs of indices into bbox)
    edge_indices = [
        (0, 1), (1, 2), (2, 3), (3, 0),  # Bottom
        (4, 5), (5, 6), (6, 7), (7, 4),  # Top
        (0, 4), (1, 5), (2, 6), (3, 7),  # Vertical
    ]

    coords = []
    for i, j in edge_indices:
        coords.extend([tuple(bbox[i]), tuple(bbox[j])])

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINES', {"pos": coords})

    gpu.state.blend_set('ALPHA')
    shader.bind()
    region = bpy.context.region
    shader.uniform_float("viewportSize", (region.width, region.height))
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (0.0, 1.0, 0.5, 0.9))
    batch.draw(shader)
    gpu.state.blend_set('NONE')


from mathutils import Vector  # Required for bounding box calculation
```

---

## Example 4: 2D HUD Overlay with Text Background

```python
# Blender 4.0+/5.x — Draw a 2D background panel in pixel coordinates
import bpy
import gpu
import blf
from gpu_extras.batch import batch_for_shader


def draw_hud():
    # Draw background rectangle
    x, y = 20, 40
    w, h = 250, 30
    verts = [(x, y), (x + w, y), (x + w, y + h), (x, y + h)]
    indices = ((0, 1, 2), (2, 3, 0))

    shader = gpu.shader.from_builtin('UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'TRIS', {"pos": verts}, indices=indices)

    gpu.state.blend_set('ALPHA')
    shader.bind()
    shader.uniform_float("color", (0.0, 0.0, 0.0, 0.6))
    batch.draw(shader)
    gpu.state.blend_set('NONE')

    # Draw text on top using blf
    font_id = 0
    blf.position(font_id, x + 10, y + 8, 0)
    blf.size(font_id, 14)
    blf.color(font_id, 1.0, 1.0, 1.0, 1.0)
    blf.draw(font_id, "Custom HUD Overlay")


# Register in POST_PIXEL for 2D pixel-space drawing
_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_hud, (), 'WINDOW', 'POST_PIXEL'
)
```

---

## Example 5: Offscreen Rendering to Image

```python
# Blender 4.0+/5.x — Render to offscreen buffer and save as image
import bpy
import gpu
import numpy as np
from gpu_extras.batch import batch_for_shader


def render_to_image(width=256, height=256):
    offscreen = gpu.types.GPUOffScreen(width, height)

    with offscreen.bind():
        fb = gpu.state.active_framebuffer_get()
        fb.clear(color=(0.1, 0.1, 0.1, 1.0))

        # Draw a colored triangle
        shader = gpu.shader.from_builtin('SMOOTH_COLOR')
        batch = batch_for_shader(shader, 'TRIS', {
            "pos": [(-0.8, -0.8), (0.8, -0.8), (0.0, 0.8)],
            "color": [(1, 0, 0, 1), (0, 1, 0, 1), (0, 0, 1, 1)],
        })

        # Set up orthographic projection for 2D drawing
        gpu.matrix.push()
        gpu.matrix.push_projection()
        gpu.matrix.load_identity()
        gpu.matrix.load_projection_matrix(
            gpu.matrix.get_projection_matrix()  # Use current projection
        )

        shader.bind()
        batch.draw(shader)

        gpu.matrix.pop_projection()
        gpu.matrix.pop()

        # Read pixel data
        buffer = fb.read_color(0, 0, width, height, 4, 0, 'FLOAT')

    offscreen.free()

    # Create Blender image from buffer
    buffer.dimensions = width * height * 4
    image = bpy.data.images.new("OffscreenResult", width, height)
    image.pixels.foreach_set(buffer)

    return image
```

---

## Example 6: Version-Compatible Drawing (3.x + 4.x + 5.x)

```python
# Blender 3.x/4.x/5.x — Fully version-compatible draw function
import bpy
import gpu
from gpu_extras.batch import batch_for_shader


def draw_version_compatible():
    coords = [(0, 0, 0), (5, 0, 0), (5, 5, 0), (0, 5, 0), (0, 0, 0)]

    if bpy.app.version >= (4, 0, 0):
        # Blender 4.0+: POLYLINE shaders with required uniforms
        shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
        batch = batch_for_shader(shader, 'LINE_STRIP', {"pos": coords})

        gpu.state.blend_set('ALPHA')
        shader.bind()
        region = bpy.context.region
        shader.uniform_float("viewportSize", (region.width, region.height))
        shader.uniform_float("lineWidth", 2.0)
        shader.uniform_float("color", (1.0, 0.0, 0.0, 1.0))
        batch.draw(shader)
        gpu.state.blend_set('NONE')
    else:
        # Blender 3.x: legacy shader names
        shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')
        batch = batch_for_shader(shader, 'LINE_STRIP', {"pos": coords})

        gpu.state.blend_set('ALPHA')
        gpu.state.line_width_set(2.0)
        shader.bind()
        shader.uniform_float("color", (1.0, 0.0, 0.0, 1.0))
        batch.draw(shader)
        gpu.state.blend_set('NONE')
        gpu.state.line_width_set(1.0)
```

---

## Example 7: Drawing Points

```python
# Blender 4.0+/5.x — Draw points at vertex positions
import bpy
import gpu
from gpu_extras.batch import batch_for_shader


def draw_object_vertices():
    obj = bpy.context.active_object
    if obj is None or obj.type != 'MESH':
        return

    # Get evaluated mesh data
    depsgraph = bpy.context.evaluated_depsgraph_get()
    obj_eval = obj.evaluated_get(depsgraph)
    mesh = obj_eval.to_mesh()

    coords = [tuple(obj.matrix_world @ v.co) for v in mesh.vertices]
    obj_eval.to_mesh_clear()

    if not coords:
        return

    shader = gpu.shader.from_builtin('UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'POINTS', {"pos": coords})

    gpu.state.blend_set('ALPHA')
    gpu.state.point_size_set(6.0)

    shader.bind()
    shader.uniform_float("color", (1.0, 1.0, 0.0, 1.0))
    batch.draw(shader)

    gpu.state.point_size_set(1.0)
    gpu.state.blend_set('NONE')
```

---

## Example 8: Drawing with Depth Testing

```python
# Blender 4.0+/5.x — Lines that respect scene depth (hidden behind objects)
import bpy
import gpu
from gpu_extras.batch import batch_for_shader


def draw_with_depth():
    coords = [(0, 0, 0), (10, 0, 0), (10, 10, 0), (0, 10, 0), (0, 0, 0)]

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINE_STRIP', {"pos": coords})

    # Enable depth testing so lines are hidden behind objects
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.depth_mask_set(True)

    shader.bind()
    region = bpy.context.region
    shader.uniform_float("viewportSize", (region.width, region.height))
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (0.0, 0.8, 1.0, 0.9))
    batch.draw(shader)

    # ALWAYS restore state
    gpu.state.depth_test_set('NONE')
    gpu.state.depth_mask_set(False)
    gpu.state.blend_set('NONE')
```

---

## Example 9: Presets — Quick Circle Drawing

```python
# Blender 4.0+/5.x — Using gpu_extras.presets for quick 2D shapes
import bpy
from gpu_extras.presets import draw_circle_2d


def draw_circles():
    # Draw a white circle at screen position (200, 200) with radius 50
    draw_circle_2d((200, 200), (1.0, 1.0, 1.0, 1.0), 50, segments=64)

    # Draw a red circle
    draw_circle_2d((400, 200), (1.0, 0.0, 0.0, 1.0), 30, segments=32)


_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_circles, (), 'WINDOW', 'POST_PIXEL'
)
```

---

## Example 10: Modal Operator with Dynamic Drawing

```python
# Blender 4.0+/5.x — Modal operator that draws a line from origin to mouse
import bpy
import gpu
from gpu_extras.batch import batch_for_shader
from bpy_extras import view3d_utils


class CUSTOM_OT_draw_line_to_mouse(bpy.types.Operator):
    bl_idname = "custom.draw_line_to_mouse"
    bl_label = "Draw Line to Mouse"

    _handle = None
    _mouse_pos_3d = (0, 0, 0)

    def modal(self, context, event):
        context.area.tag_redraw()

        if event.type == 'MOUSEMOVE':
            # Convert 2D mouse to 3D world coordinate on ground plane
            region = context.region
            rv3d = context.region_data
            coord = (event.mouse_region_x, event.mouse_region_y)
            origin = view3d_utils.region_2d_to_origin_3d(region, rv3d, coord)
            direction = view3d_utils.region_2d_to_vector_3d(region, rv3d, coord)
            if direction.z != 0:
                t = -origin.z / direction.z
                self.__class__._mouse_pos_3d = tuple(origin + direction * t)

        elif event.type in {'RIGHTMOUSE', 'ESC'}:
            self.cancel(context)
            return {'CANCELLED'}

        return {'RUNNING_MODAL'}

    def invoke(self, context, event):
        self.__class__._handle = bpy.types.SpaceView3D.draw_handler_add(
            self.draw_callback, (), 'WINDOW', 'POST_VIEW'
        )
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def cancel(self, context):
        if self.__class__._handle is not None:
            bpy.types.SpaceView3D.draw_handler_remove(
                self.__class__._handle, 'WINDOW'
            )
            self.__class__._handle = None
        context.area.tag_redraw()

    @staticmethod
    def draw_callback():
        coords = [(0, 0, 0), CUSTOM_OT_draw_line_to_mouse._mouse_pos_3d]

        shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
        batch = batch_for_shader(shader, 'LINES', {"pos": coords})

        gpu.state.blend_set('ALPHA')
        shader.bind()
        region = bpy.context.region
        shader.uniform_float("viewportSize", (region.width, region.height))
        shader.uniform_float("lineWidth", 2.0)
        shader.uniform_float("color", (1.0, 1.0, 0.0, 1.0))
        batch.draw(shader)
        gpu.state.blend_set('NONE')


def register():
    bpy.utils.register_class(CUSTOM_OT_draw_line_to_mouse)


def unregister():
    bpy.utils.unregister_class(CUSTOM_OT_draw_line_to_mouse)
```

---

## Example 11: Drawing Image Texture from File

```python
# Blender 4.0+/5.x — Load and display an image in the viewport
import bpy
import gpu
from gpu_extras.batch import batch_for_shader


def draw_image_overlay():
    image = bpy.data.images.get("my_overlay")
    if image is None:
        return

    texture = gpu.texture.from_image(image)
    shader = gpu.shader.from_builtin('IMAGE')

    # Draw at bottom-left corner, 200x200 pixels
    batch = batch_for_shader(
        shader, 'TRI_FAN',
        {
            "pos": ((20, 20), (220, 20), (220, 220), (20, 220)),
            "texCoord": ((0, 0), (1, 0), (1, 1), (0, 1)),
        },
    )

    gpu.state.blend_set('ALPHA')
    shader.bind()
    shader.uniform_sampler("image", texture)
    batch.draw(shader)
    gpu.state.blend_set('NONE')


# Load image once at registration
def register():
    if "my_overlay" not in bpy.data.images:
        bpy.data.images.load("/path/to/overlay.png", check_existing=True)
        bpy.data.images["overlay.png"].name = "my_overlay"

    global _handle
    _handle = bpy.types.SpaceView3D.draw_handler_add(
        draw_image_overlay, (), 'WINDOW', 'POST_PIXEL'
    )
```

---

## Example 12: Batch Caching for Performance

```python
# Blender 4.0+/5.x — Cache shader and batch for performance
import bpy
import gpu
from gpu_extras.batch import batch_for_shader

# Cache at module level — recreate only when data changes
_cached_batch = None
_cached_shader = None


def invalidate_cache():
    global _cached_batch, _cached_shader
    _cached_batch = None
    _cached_shader = None


def ensure_cache():
    global _cached_batch, _cached_shader
    if _cached_shader is None:
        _cached_shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    if _cached_batch is None:
        coords = [(0, 0, 0), (10, 0, 0), (10, 10, 0), (0, 10, 0), (0, 0, 0)]
        _cached_batch = batch_for_shader(
            _cached_shader, 'LINE_STRIP', {"pos": coords}
        )


def draw_cached():
    ensure_cache()

    gpu.state.blend_set('ALPHA')
    _cached_shader.bind()
    region = bpy.context.region
    _cached_shader.uniform_float("viewportSize", (region.width, region.height))
    _cached_shader.uniform_float("lineWidth", 2.0)
    _cached_shader.uniform_float("color", (0.0, 1.0, 0.0, 1.0))
    _cached_batch.draw(_cached_shader)
    gpu.state.blend_set('NONE')
```
