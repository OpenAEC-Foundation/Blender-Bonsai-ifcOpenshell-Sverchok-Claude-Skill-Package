# Migration Code Examples

Migration code examples for EACH version transition from Blender 3.x through 5.1.

---

## 3.x → 4.0 Migration Examples

### Context Overrides

```python
# Blender 3.x — BROKEN in 4.0+
override = {"object": obj, "active_object": obj, "selected_objects": [obj]}
bpy.ops.object.modifier_apply(override, modifier="Boolean")

# Blender 4.0+ — REQUIRED
with bpy.context.temp_override(object=obj, active_object=obj, selected_objects=[obj]):
    bpy.ops.object.modifier_apply(modifier="Boolean")
```

### Mesh Bevel Weights and Creases

```python
# Blender 3.x — BROKEN in 4.0+
weight = mesh.edges[0].bevel_weight  # AttributeError in 4.0
crease = mesh.edges[0].crease        # AttributeError in 4.0

# Blender 4.0+ — Attribute-based access
bevel_attr = mesh.attributes.get("bevel_weight_edge")
if bevel_attr:
    weight = bevel_attr.data[0].value

crease_attr = mesh.attributes.get("crease_edge")
if crease_attr:
    crease = crease_attr.data[0].value

# To create bevel weight attribute if it does not exist:
if "bevel_weight_edge" not in mesh.attributes:
    mesh.attributes.new("bevel_weight_edge", type='FLOAT', domain='EDGE')
```

### Node Group Interface

```python
# Blender 3.x — BROKEN in 4.0+
node_group.inputs.new("NodeSocketFloat", "Width")
node_group.inputs.new("NodeSocketGeometry", "Geometry")
node_group.outputs.new("NodeSocketGeometry", "Result")

# Blender 4.0+ — interface API
node_group.interface.new_socket(name="Width", in_out='INPUT', socket_type='NodeSocketFloat')
node_group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
node_group.interface.new_socket(name="Result", in_out='OUTPUT', socket_type='NodeSocketGeometry')
```

### Principled BSDF Socket Names

```python
# Blender 3.x — BROKEN in 4.0+
principled.inputs["Subsurface"].default_value = 0.5
principled.inputs["Specular"].default_value = 0.5
principled.inputs["Transmission"].default_value = 1.0

# Blender 4.0+ — Renamed sockets
principled.inputs["Subsurface Weight"].default_value = 0.5
principled.inputs["Specular IOR Level"].default_value = 0.5
principled.inputs["Transmission Weight"].default_value = 1.0
```

### GPU Shader Names

```python
# Blender 3.x — BROKEN in 4.0+
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')
shader_flat = gpu.shader.from_builtin('3D_FLAT_COLOR')
shader_2d = gpu.shader.from_builtin('2D_UNIFORM_COLOR')

# Blender 4.0+ — Prefix removed, polyline variants
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
shader_flat = gpu.shader.from_builtin('POLYLINE_FLAT_COLOR')
shader_2d = gpu.shader.from_builtin('UNIFORM_COLOR')
```

### Bone Layers to Collections

```python
# Blender 3.x — BROKEN in 4.0+
bone.layers[0] = True
bone.layers[1] = False
group = pose.bone_groups.new(name="IK Controls")
group.color_set = 'THEME01'

# Blender 4.0+ — Bone Collections
collection = armature.collections.new("IK Controls")
collection.assign(bone)
# Colors are set on individual bones via bone.color
bone.color.palette = 'THEME01'
```

### OBJ/PLY Import/Export

```python
# Blender 3.x — BROKEN in 4.0+
bpy.ops.import_scene.obj(filepath="model.obj")
bpy.ops.export_scene.obj(filepath="model.obj")

# Blender 4.0+ — C++ IO operators
bpy.ops.wm.obj_import(filepath="model.obj")
bpy.ops.wm.obj_export(filepath="model.obj")
bpy.ops.wm.ply_import(filepath="model.ply")
bpy.ops.wm.ply_export(filepath="model.ply")
```

### Face Maps

```python
# Blender 3.x — BROKEN in 4.0+
face_map = obj.face_maps.new(name="Front")
face_map.add([0, 1, 2, 3])

# Blender 4.0+ — Integer face attributes
mesh = obj.data
if "face_group" not in mesh.attributes:
    mesh.attributes.new("face_group", type='INT', domain='FACE')
attr = mesh.attributes["face_group"]
for i in [0, 1, 2, 3]:
    attr.data[i].value = 1  # assign group ID
```

---

## 4.0 → 4.1 Migration Examples

### Auto Smooth Removal

```python
# Blender 3.x / 4.0 — BROKEN in 4.1+
mesh.use_auto_smooth = True
mesh.auto_smooth_angle = 0.523599  # 30 degrees
mesh.calc_normals_split()
for loop in mesh.loops:
    normal = loop.normal  # was writable

# Blender 4.1+ — corner_normals + Smooth by Angle modifier
# Auto smooth is ALWAYS active; no toggle needed
corner_normals = mesh.corner_normals  # read-only, auto-updated

# For angle-based smoothing, apply the "Smooth by Angle" geometry node modifier
# This is a built-in node group asset, not a Python API call

# For custom normals:
normals_list = [(0.0, 0.0, 1.0)] * len(mesh.loops)
mesh.normals_split_custom_set(normals_list)
```

### Light Probe Types

```python
# Blender 3.x / 4.0 — BROKEN in 4.1+
if probe.type == 'CUBEMAP':
    pass
elif probe.type == 'PLANAR':
    pass
elif probe.type == 'GRID':
    pass

# Blender 4.1+ — Renamed enum values
if probe.type == 'SPHERE':
    pass
elif probe.type == 'PLANE':
    pass
elif probe.type == 'VOLUME':
    pass
```

### Material Displacement Method

```python
# Blender 3.x / 4.0 — BROKEN in 4.1+
mat.cycles.displacement_method = 'BOTH'

# Blender 4.1+ — Property on Material directly
mat.displacement_method = 'BOTH'
```

### foreach_set Validation

```python
# Blender 3.x / 4.0 — Silent failure (dangerous)
mesh.vertices.foreach_set("co", wrong_type_data)  # silently ignored

# Blender 4.1+ — Raises TypeError
try:
    mesh.vertices.foreach_set("co", data)
except TypeError as e:
    print(f"Invalid data type for foreach_set: {e}")
```

---

## 4.1 → 4.2 Migration Examples

### Extension Manifest

```python
# Blender 3.x / 4.0 / 4.1 — bl_info in __init__.py
bl_info = {
    "name": "My BIM Tool",
    "blender": (4, 1, 0),
    "category": "Object",
    "version": (1, 2, 0),
    "description": "BIM modeling tools for AEC",
    "author": "Developer",
}
```

```toml
# Blender 4.2+ — blender_manifest.toml (REQUIRED for extensions)
schema_version = "1.0.0"
id = "my_bim_tool"
version = "1.2.0"
name = "My BIM Tool"
tagline = "BIM modeling tools for AEC"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
tags = ["Object", "Import-Export"]
license = ["SPDX:GPL-3.0-or-later"]

[permissions]
# Only add if addon needs network access
# network = "Downloads BIM library data from server"
```

### EEVEE Identifier Change

```python
# Blender 3.x / 4.0 / 4.1 — 'BLENDER_EEVEE'
scene.render.engine = 'BLENDER_EEVEE'

# Blender 4.2+ — 'BLENDER_EEVEE_NEXT'
scene.render.engine = 'BLENDER_EEVEE_NEXT'

# Version-safe
if bpy.app.version >= (5, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE'  # changed back in 5.0
elif bpy.app.version >= (4, 2, 0):
    scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    scene.render.engine = 'BLENDER_EEVEE'
```

### Statically Typed IDProperties

```python
# Blender 3.x / 4.0 / 4.1 — Dynamic types (implicit conversion)
obj["my_count"] = 5      # int
obj["my_count"] = 3.14   # silently changed to float

# Blender 4.2+ — Static types (conversion rules enforced)
obj["my_count"] = 5      # int
obj["my_count"] = 3.14   # raises TypeError if type was locked as int
# ALWAYS set the correct type on first assignment
```

### Motion Blur Properties

```python
# Blender 4.1 — Cycles/EEVEE-specific properties
scene.cycles.motion_blur_position = 'START'
scene.eevee.use_motion_blur = True

# Blender 4.2+ — Unified render properties
scene.render.motion_blur_position = 'START'
scene.render.use_motion_blur = True
```

---

## 4.2 → 4.3 Migration Examples

### AttributeGroup Split

```python
# Blender 3.x–4.2 — Generic AttributeGroup
attrs = obj.data.attributes  # bpy.types.AttributeGroup
color = attrs.active_color_name  # works on any geometry

# Blender 4.3+ — Type-specific AttributeGroup
attrs = obj.data.attributes  # AttributeGroupMesh for mesh objects
# active_color_name ONLY available on AttributeGroupMesh
if isinstance(obj.data, bpy.types.Mesh):
    color = attrs.active_color_name  # OK
# Use domain_size() to check support
size = attrs.domain_size('POINT')  # returns 0 if unsupported
```

### Grease Pencil Migration

```python
# Blender 3.x–4.2 — Legacy Grease Pencil API
# WARNING: ENTIRE API rewritten in 4.3. NOT backward compatible.
gp = bpy.data.grease_pencils.new("MyGP")
layer = gp.layers.new("Layer")
frame = layer.frames.new(1)
stroke = frame.strokes.new()
stroke.points.add(count=3)

# Blender 4.3+ — New Grease Pencil data structure
# See: https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/
# Files saved in 4.3+ do NOT load in 4.2 or earlier
```

### EEVEE Legacy Property Removal

```python
# Blender 3.x–4.2 — EEVEE properties (REMOVED in 4.3)
scene.eevee.use_ssr = True                    # REMOVED
scene.eevee.use_bloom = True                  # REMOVED
scene.eevee.use_volumetric_lights = True      # REMOVED
scene.eevee.use_gtao = True                   # REMOVED
scene.eevee.bokeh_max_size = 100.0            # REMOVED

# Blender 4.3+ — These features are handled automatically by EEVEE Next
# No replacement properties needed — the engine handles them internally
```

---

## 4.3 → 4.4 Migration Examples

### Subclass Constructor

```python
# Blender 3.x–4.3 — Simple subclassing
class MY_OT_operator(bpy.types.Operator):
    bl_idname = "my.operator"
    bl_label = "My Operator"

    def __init__(self):
        self.my_data = []

# Blender 4.4+ — MUST call super().__init__()
class MY_OT_operator(bpy.types.Operator):
    bl_idname = "my.operator"
    bl_label = "My Operator"

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.my_data = []
```

### Sequence → Strip Rename

```python
# Blender 3.x–4.3 — bpy.types.Sequence
for seq in scene.sequence_editor.sequences_all:
    if seq.type == 'IMAGE':
        print(seq.name)

# Blender 4.4+ — bpy.types.Strip
for strip in scene.sequence_editor.sequences_all:
    if strip.type == 'IMAGE':
        print(strip.name)

# Type annotations
seq: bpy.types.Sequence  # BROKEN in 4.4+
strip: bpy.types.Strip   # Blender 4.4+
```

### Slotted Actions

```python
# Blender 3.x–4.3 — Legacy action.fcurves (REMOVED in 5.0)
action = bpy.data.actions.new("Walk")
fc = action.fcurves.new(data_path="location", index=0)
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (60.0, 5.0)
fc.update()

# Blender 4.4+ — Slotted Actions
action = bpy.data.actions.new("Walk")
slot = action.slots.new()
slot.target_id_type = 'OBJECT'
layer = action.layers.new(name="Base Layer")
strip = layer.strips.new(type='KEYFRAME')
channelbag = strip.channelbags.new(slot=slot)
fc = channelbag.fcurves.new(data_path="location", index=0)
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (60.0, 5.0)
fc.update()

# Assign to object
obj = bpy.context.active_object
if obj.animation_data is None:
    obj.animation_data_create()
obj.animation_data.action = action
obj.animation_data.action_slot = slot
```

---

## 4.x → 5.0 Migration Examples

### BGL to GPU Module

```python
# Blender 3.x / 4.x — bgl module (REMOVED in 5.0)
import bgl
bgl.glEnable(bgl.GL_BLEND)
bgl.glEnable(bgl.GL_DEPTH_TEST)
bgl.glLineWidth(2)
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')
# ... draw ...
bgl.glDisable(bgl.GL_BLEND)
bgl.glDisable(bgl.GL_DEPTH_TEST)
bgl.glLineWidth(1)

# Blender 4.0+ / 5.0 (REQUIRED) — gpu module
import gpu
gpu.state.blend_set('ALPHA')
gpu.state.depth_test_set('LESS_EQUAL')
gpu.state.depth_mask_set(True)
gpu.state.line_width_set(2.0)
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
shader.bind()
shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])
shader.uniform_float("lineWidth", 2.0)
shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))
batch.draw(shader)
# ALWAYS restore state
gpu.state.blend_set('NONE')
gpu.state.depth_test_set('NONE')
gpu.state.depth_mask_set(False)
gpu.state.line_width_set(1.0)
```

### Runtime Properties Access

```python
# Blender 3.x / 4.x — Dict-like access to RNA properties (REMOVED in 5.0)
cycles_settings = bpy.context.scene['cycles']  # dict-like
samples = cycles_settings['samples']

# Blender 5.0+ — Attribute access (REQUIRED)
samples = bpy.context.scene.cycles.samples
```

### Compositor Node Tree

```python
# Blender 3.x / 4.x — scene.node_tree (REMOVED in 5.0)
scene = bpy.context.scene
scene.use_nodes = True
comp_tree = scene.node_tree
render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
composite = comp_tree.nodes.new('CompositorNodeComposite')
comp_tree.links.new(render_layers.outputs["Image"], composite.inputs["Image"])

# Blender 5.0+ — compositing_node_group
scene = bpy.context.scene
comp_tree = bpy.data.node_groups.new("MyCompositor", 'CompositorNodeTree')
scene.compositing_node_group = comp_tree
render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
composite = comp_tree.nodes.new('CompositorNodeComposite')
comp_tree.links.new(render_layers.outputs["Image"], composite.inputs["Image"])
```

### Brush Type Enum

```python
# Blender 3.x / 4.x — sculpt_tool (REMOVED in 5.0)
brush.sculpt_tool = 'DRAW'
brush.sculpt_tool = 'CLAY_STRIPS'

# Blender 5.0+ — sculpt_brush_type
brush.sculpt_brush_type = 'DRAW'
brush.sculpt_brush_type = 'CLAY_STRIPS'
```

### Image Bindcode to GPU Texture

```python
# Blender 3.x / 4.x — Image.bindcode (REMOVED in 5.0)
image = bpy.data.images["MyTexture"]
image.gl_load()
bindcode = image.bindcode

# Blender 5.0+ — gpu.texture.from_image()
import gpu
image = bpy.data.images["MyTexture"]
texture = gpu.texture.from_image(image)
```

### Legacy Action API

```python
# Blender 3.x / 4.0–4.3 — action.fcurves (REMOVED in 5.0)
for fc in action.fcurves:
    print(fc.data_path)
for group in action.groups:
    print(group.name)
root_type = action.id_root

# Blender 5.0+ — Slotted Actions channelbag
for slot in action.slots:
    for layer in action.layers:
        for strip in layer.strips:
            channelbag = strip.channelbag(slot)
            if channelbag:
                for fc in channelbag.fcurves:
                    print(fc.data_path)
root_type = action.slots[0].target_id_type
```

### VSE End Frame

```python
# Blender 3.x / 4.x — end_frame (REMOVED in 5.0)
duration = strip.end_frame - strip.frame_start

# Blender 5.0+ — length property
duration = strip.length
```

---

## 5.0 → 5.1 Migration Examples

### Sculpt Sample Color

```python
# Blender 5.0 — sculpt.sample_color (REMOVED in 5.1)
bpy.ops.sculpt.sample_color(location=(100, 200))

# Blender 5.1+ — paint.sample_color
bpy.ops.paint.sample_color(location=(100, 200))
```

### Python 3.13 Compatibility

```python
# Blender 5.1 upgrades to Python 3.13 (from 3.11 in 5.0)
# Key changes to check:
# - PEP 594: deprecated stdlib modules removed (aifc, audioop, cgi, etc.)
# - PEP 701: f-string syntax changes (more flexible, but edge cases differ)
# - typing module changes
# ALWAYS test addon imports after Python version upgrade
```

### VSE Time Property Deprecations (Removal in 6.0)

```python
# Blender 5.0 — Current names (DEPRECATED in 5.1, removed in 6.0)
start = strip.frame_final_start
end = strip.frame_final_end
dur = strip.frame_final_duration

# Blender 5.1+ — New names (use these for forward compatibility)
start = strip.left_handle
end = strip.right_handle
dur = strip.duration
```

---

## Version-Safe Wrapper Pattern

For addons that must support multiple Blender versions, use wrapper functions:

```python
# Blender 3.x / 4.x / 5.x — Version-safe utility module
import bpy

def get_compositor_tree(scene):
    """Return the compositor node tree for the given scene."""
    if bpy.app.version >= (5, 0, 0):
        tree = scene.compositing_node_group
        if tree is None:
            tree = bpy.data.node_groups.new("Compositor", 'CompositorNodeTree')
            scene.compositing_node_group = tree
        return tree
    else:
        scene.use_nodes = True
        return scene.node_tree

def apply_modifier_safe(obj, modifier_name):
    """Apply a modifier with version-safe context override."""
    if bpy.app.version >= (4, 0, 0):
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        override = {"object": obj, "active_object": obj}
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)

def get_eevee_id():
    """Return the correct EEVEE render engine identifier."""
    if bpy.app.version >= (5, 0, 0):
        return 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        return 'BLENDER_EEVEE_NEXT'
    return 'BLENDER_EEVEE'

def create_node_group_socket(node_group, name, in_out, socket_type):
    """Create a node group socket with version-safe API."""
    if bpy.app.version >= (4, 0, 0):
        node_group.interface.new_socket(name=name, in_out=in_out, socket_type=socket_type)
    else:
        if in_out == 'INPUT':
            node_group.inputs.new(socket_type, name)
        else:
            node_group.outputs.new(socket_type, name)
```

---

## Sources

- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.2/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
- https://developer.blender.org/docs/release_notes/4.4/python_api/
- https://developer.blender.org/docs/release_notes/4.5/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- https://developer.blender.org/docs/release_notes/5.1/python_api/
- https://developer.blender.org/docs/release_notes/compatibility/
