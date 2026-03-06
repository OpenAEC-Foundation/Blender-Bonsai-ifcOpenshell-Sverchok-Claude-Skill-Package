# Migration Examples — Before/After Code

Working code examples for each major migration path. Each example shows the broken (pre-migration) code and the correct (post-migration) code.

---

## 3.x → 4.0 Examples

### Example 1: Context Override Migration

```python
# BEFORE — Blender 3.x (BROKEN in 4.0+)
import bpy

def apply_all_modifiers(obj):
    override = bpy.context.copy()
    override["object"] = obj
    override["active_object"] = obj
    for mod in obj.modifiers:
        bpy.ops.object.modifier_apply(override, modifier=mod.name)
```

```python
# AFTER — Blender 4.0+
import bpy

def apply_all_modifiers(obj):
    for mod in list(obj.modifiers):
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=mod.name)
```

### Example 2: Mesh Bevel Weight Migration

```python
# BEFORE — Blender 3.x (BROKEN in 4.0+)
import bpy

mesh = bpy.context.active_object.data
for edge in mesh.edges:
    if edge.bevel_weight > 0.5:
        edge.select = True
```

```python
# AFTER — Blender 4.0+
import bpy

mesh = bpy.context.active_object.data
bevel_attr = mesh.attributes.get("bevel_weight_edge")
if bevel_attr:
    for i, edge in enumerate(mesh.edges):
        if bevel_attr.data[i].value > 0.5:
            edge.select = True
```

### Example 3: Node Group Interface Migration

```python
# BEFORE — Blender 3.x (BROKEN in 4.0+)
import bpy

node_group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
node_group.inputs.new("NodeSocketFloat", "Width")
node_group.inputs.new("NodeSocketFloat", "Height")
node_group.outputs.new("NodeSocketGeometry", "Result")
```

```python
# AFTER — Blender 4.0+
import bpy

node_group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
node_group.interface.new_socket(
    name="Width", in_out='INPUT', socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Height", in_out='INPUT', socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Result", in_out='OUTPUT', socket_type='NodeSocketGeometry'
)
```

### Example 4: Bone Layer to Collection Migration

```python
# BEFORE — Blender 3.x (BROKEN in 4.0+)
import bpy

armature = bpy.context.active_object.data
for bone in armature.bones:
    if bone.layers[0]:
        bone.layers[1] = True
        bone.layers[0] = False
```

```python
# AFTER — Blender 4.0+
import bpy

armature = bpy.context.active_object.data
source_col = armature.collections.get("Layer 0")
target_col = armature.collections.get("Layer 1")
if not target_col:
    target_col = armature.collections.new("Layer 1")

if source_col:
    for bone in armature.bones:
        if source_col.is_bone_member(bone):
            target_col.assign(bone)
            source_col.unassign(bone)
```

### Example 5: Principled BSDF Socket Names

```python
# BEFORE — Blender 3.x (KeyError in 4.0+)
import bpy

mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs["Subsurface"].default_value = 0.5
bsdf.inputs["Specular"].default_value = 0.3
bsdf.inputs["Transmission"].default_value = 1.0
bsdf.inputs["Clearcoat"].default_value = 0.8
```

```python
# AFTER — Blender 4.0+
import bpy

mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs["Subsurface Weight"].default_value = 0.5
bsdf.inputs["Specular IOR Level"].default_value = 0.3
bsdf.inputs["Transmission Weight"].default_value = 1.0
bsdf.inputs["Coat Weight"].default_value = 0.8
```

### Example 6: OBJ Import/Export Migration

```python
# BEFORE — Blender 3.x (ModuleNotFoundError in 4.0+)
import bpy

bpy.ops.import_scene.obj(filepath="/path/to/model.obj")
bpy.ops.export_scene.obj(filepath="/path/to/output.obj", use_selection=True)
```

```python
# AFTER — Blender 4.0+
import bpy

bpy.ops.wm.obj_import(filepath="/path/to/model.obj")
bpy.ops.wm.obj_export(filepath="/path/to/output.obj", export_selected_objects=True)
```

---

## 4.0 → 4.1 Examples

### Example 7: Auto-Smooth Migration

```python
# BEFORE — Blender 4.0 (BROKEN in 4.1+)
import bpy
import math

obj = bpy.context.active_object
mesh = obj.data
mesh.use_auto_smooth = True
mesh.auto_smooth_angle = math.radians(30)
```

```python
# AFTER — Blender 4.1+
import bpy

obj = bpy.context.active_object
# Auto-smooth is always active in 4.1+
# For angle-based smoothing, add a "Smooth by Angle" modifier:
mod = obj.modifiers.new(name="Smooth by Angle", type='NODES')
# The modifier uses the built-in "Smooth by Angle" node group asset
```

### Example 8: Light Probe Type Migration

```python
# BEFORE — Blender 4.0 (ValueError in 4.1+)
import bpy

bpy.ops.object.lightprobe_add(type='CUBEMAP')
probe = bpy.context.active_object.data
if probe.type == 'CUBEMAP':
    probe.clip_end = 100.0
```

```python
# AFTER — Blender 4.1+
import bpy

bpy.ops.object.lightprobe_add(type='SPHERE')
probe = bpy.context.active_object.data
if probe.type == 'SPHERE':
    probe.clip_end = 100.0
```

---

## 4.1 → 4.2 Examples

### Example 9: Extension Manifest Migration

```python
# BEFORE — Blender 4.1 addon (__init__.py)
bl_info = {
    "name": "My Addon",
    "author": "Developer",
    "version": (1, 2, 0),
    "blender": (4, 1, 0),
    "description": "Does useful things with meshes and provides tools",
    "category": "Mesh",
}
```

```toml
# AFTER — Blender 4.2+ extension (blender_manifest.toml)
schema_version = "1.0.0"
id = "my_addon"
version = "1.2.0"
name = "My Addon"
tagline = "Does useful things with meshes and provides tools"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
tags = ["Mesh"]
license = ["SPDX:MIT"]
```

### Example 10: EEVEE Identifier Migration

```python
# BEFORE — Blender 4.1 (RuntimeError in 4.2)
import bpy
bpy.context.scene.render.engine = 'BLENDER_EEVEE'
```

```python
# AFTER — Blender 4.2–4.x
import bpy
bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'
```

---

## 4.2 → 4.3 Examples

### Example 11: AttributeGroup Split

```python
# BEFORE — Blender 4.2 (AttributeError in 4.3+ for non-mesh)
import bpy

obj = bpy.context.active_object
attrs = obj.data.attributes  # was generic AttributeGroup
color_name = attrs.active_color_name  # Fails on non-mesh in 4.3+
```

```python
# AFTER — Blender 4.3+
import bpy

obj = bpy.context.active_object
attrs = obj.data.attributes  # now type-specific (e.g., AttributeGroupMesh)
if hasattr(attrs, 'active_color_name'):
    color_name = attrs.active_color_name  # Only on AttributeGroupMesh
```

### Example 12: EEVEE Legacy Property Removal

```python
# BEFORE — Blender 4.2 (AttributeError in 4.3+)
import bpy

scene = bpy.context.scene
scene.eevee.use_ssr = True
scene.eevee.use_bloom = True
scene.eevee.use_gtao = True
scene.eevee.use_soft_shadows = True
```

```python
# AFTER — Blender 4.3+
import bpy

# These properties are removed — EEVEE handles them internally.
# Remove ALL references to these properties from scripts.
# No replacement API exists; the engine manages these features automatically.
```

---

## 4.3 → 4.4 Examples

### Example 13: Super Init Required

```python
# BEFORE — Blender 4.3 (TypeError in 4.4+)
import bpy

class MY_OT_example(bpy.types.Operator):
    bl_idname = "my.example"
    bl_label = "Example"

    def __init__(self):
        self.my_var = 0

    def execute(self, context):
        return {'FINISHED'}
```

```python
# AFTER — Blender 4.4+
import bpy

class MY_OT_example(bpy.types.Operator):
    bl_idname = "my.example"
    bl_label = "Example"

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.my_var = 0

    def execute(self, context):
        return {'FINISHED'}
```

### Example 14: Sequence to Strip Rename

```python
# BEFORE — Blender 4.3 (AttributeError in 4.4+)
import bpy

for seq in bpy.context.scene.sequence_editor.sequences_all:
    if isinstance(seq, bpy.types.Sequence):
        print(seq.name)
```

```python
# AFTER — Blender 4.4+
import bpy

for strip in bpy.context.scene.sequence_editor.sequences_all:
    if isinstance(strip, bpy.types.Strip):
        print(strip.name)
```

### Example 15: Slotted Actions (New Animation System)

```python
# BEFORE — Blender 4.3 (legacy, removed in 5.0)
import bpy

action = bpy.data.actions.new("BounceAction")
fcurve = action.fcurves.new(data_path="location", index=2)
fcurve.keyframe_points.add(2)
fcurve.keyframe_points[0].co = (1.0, 0.0)
fcurve.keyframe_points[1].co = (25.0, 3.0)
```

```python
# AFTER — Blender 4.4+ (Slotted Actions)
import bpy

action = bpy.data.actions.new("BounceAction")
slot = action.slots.new()
layer = action.layers.new(name="Layer")
strip = layer.strips.new(type='KEYFRAME')
channelbag = strip.channelbags.new(slot=slot)
fcurve = channelbag.fcurves.new(data_path="location", index=2)
fcurve.keyframe_points.add(2)
fcurve.keyframe_points[0].co = (1.0, 0.0)
fcurve.keyframe_points[1].co = (25.0, 3.0)
```

---

## 4.x → 5.0 Examples

### Example 16: BGL to GPU Module Migration (Complete)

```python
# BEFORE — Blender 4.x (ImportError in 5.0+)
import bpy
import bgl
import gpu
from gpu_extras.batch import batch_for_shader

coords = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')
batch = batch_for_shader(shader, 'LINE_LOOP', {"pos": coords})

def draw():
    bgl.glEnable(bgl.GL_BLEND)
    bgl.glLineWidth(2)
    bgl.glEnable(bgl.GL_DEPTH_TEST)
    shader.bind()
    shader.uniform_float("color", (1, 0, 0, 0.8))
    batch.draw(shader)
    bgl.glDisable(bgl.GL_BLEND)
    bgl.glLineWidth(1)
    bgl.glDisable(bgl.GL_DEPTH_TEST)

handle = bpy.types.SpaceView3D.draw_handler_add(draw, (), 'WINDOW', 'POST_VIEW')
```

```python
# AFTER — Blender 5.0+
import bpy
import gpu
from gpu_extras.batch import batch_for_shader

coords = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
batch = batch_for_shader(shader, 'LINE_LOOP', {"pos": coords})

def draw():
    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(2.0)
    gpu.state.depth_test_set('LESS_EQUAL')
    shader.bind()
    shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (1, 0, 0, 0.8))
    batch.draw(shader)
    # ALWAYS restore state
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)
    gpu.state.depth_test_set('NONE')

handle = bpy.types.SpaceView3D.draw_handler_add(draw, (), 'WINDOW', 'POST_VIEW')
```

### Example 17: Compositor Migration

```python
# BEFORE — Blender 4.x (AttributeError in 5.0+)
import bpy

scene = bpy.context.scene
scene.use_nodes = True
comp_tree = scene.node_tree
render_layers = comp_tree.nodes.get("Render Layers")
composite = comp_tree.nodes.get("Composite")
comp_tree.links.new(render_layers.outputs["Image"], composite.inputs["Image"])
```

```python
# AFTER — Blender 5.0+
import bpy

scene = bpy.context.scene
comp_tree = bpy.data.node_groups.new("MyCompositor", 'CompositorNodeTree')
scene.compositing_node_group = comp_tree
render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
composite = comp_tree.nodes.new('CompositorNodeComposite')
comp_tree.links.new(render_layers.outputs["Image"], composite.inputs["Image"])
```

### Example 18: Scene Properties Dict Access Removal

```python
# BEFORE — Blender 4.x (KeyError in 5.0+)
import bpy

scene = bpy.context.scene
cycles_props = scene['cycles']
samples = cycles_props['samples']
```

```python
# AFTER — Blender 5.0+
import bpy

scene = bpy.context.scene
samples = scene.cycles.samples
```

### Example 19: Image Texture Loading Migration

```python
# BEFORE — Blender 4.x (AttributeError in 5.0+)
import bpy
import bgl

image = bpy.data.images.load("/path/to/texture.png")
image.gl_load()
texture_id = image.bindcode
bgl.glBindTexture(bgl.GL_TEXTURE_2D, texture_id)
```

```python
# AFTER — Blender 5.0+
import bpy
import gpu

image = bpy.data.images.load("/path/to/texture.png")
texture = gpu.texture.from_image(image)
# Use texture in shader via gpu.types.GPUShader uniform
```

---

## 5.0 → 5.1 Examples

### Example 20: VSE Time Property Renames

```python
# BEFORE — Blender 5.0 (deprecated in 5.1, removed in 6.0)
import bpy

strip = bpy.context.scene.sequence_editor.active_strip
duration = strip.frame_final_duration
start = strip.frame_final_start
end = strip.frame_final_end
```

```python
# AFTER — Blender 5.1+
import bpy

strip = bpy.context.scene.sequence_editor.active_strip
duration = strip.duration
start = strip.left_handle
end = strip.right_handle
```

---

## Multi-Version Compatibility Examples

### Example 21: Version-Safe Context Override Wrapper

```python
# Blender 3.x / 4.x / 5.x — Works on ALL versions
import bpy

def safe_modifier_apply(obj, modifier_name):
    """Apply modifier with version-safe context override."""
    if bpy.app.version >= (4, 0, 0):
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        override = {"object": obj, "active_object": obj}
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)
```

### Example 22: Version-Safe EEVEE Identifier

```python
# Blender 3.x / 4.x / 5.x — Works on ALL versions
import bpy

def set_eevee_engine():
    """Set render engine to EEVEE with version-safe identifier."""
    if bpy.app.version >= (5, 0, 0):
        bpy.context.scene.render.engine = 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'
    else:
        bpy.context.scene.render.engine = 'BLENDER_EEVEE'
```

### Example 23: Version-Safe Drawing (3.x + 4.x + 5.x)

```python
# Blender 3.x / 4.x / 5.x — Version-safe drawing module
import bpy
import gpu
from gpu_extras.batch import batch_for_shader

try:
    import bgl
    HAS_BGL = True
except ImportError:
    HAS_BGL = False

def get_line_shader():
    """Get line shader with version-safe name."""
    if bpy.app.version >= (4, 0, 0):
        return gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    else:
        return gpu.shader.from_builtin('3D_UNIFORM_COLOR')

def draw_setup():
    """Set up drawing state with version-safe calls."""
    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(2.0)

def draw_teardown():
    """Restore drawing state to defaults."""
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)
```
