# Version Error Migration Examples

Error-to-fix migration code for each error pattern defined in the SKILL.md decision tree.

---

## E-01: MeshEdge bevel_weight / crease (Removed 4.0)

**Error**: `AttributeError: 'MeshEdge' object has no attribute 'bevel_weight'`

```python
# BROKEN in Blender 4.0+ — Direct edge property access removed
weight = mesh.edges[0].bevel_weight  # AttributeError
crease = mesh.edges[0].crease        # AttributeError

# FIX for Blender 4.0+ — Use mesh attributes
bevel_attr = mesh.attributes.get("bevel_weight_edge")
if bevel_attr:
    weight = bevel_attr.data[0].value
else:
    weight = 0.0

crease_attr = mesh.attributes.get("crease_edge")
if crease_attr:
    crease = crease_attr.data[0].value
else:
    crease = 0.0

# To CREATE bevel weight attribute (4.0+)
if "bevel_weight_edge" not in mesh.attributes:
    mesh.attributes.new("bevel_weight_edge", type='FLOAT', domain='EDGE')

# VERSION-SAFE wrapper
import bpy

def get_edge_bevel_weight(mesh, edge_index):
    if bpy.app.version >= (4, 0, 0):
        attr = mesh.attributes.get("bevel_weight_edge")
        return attr.data[edge_index].value if attr else 0.0
    return mesh.edges[edge_index].bevel_weight

def set_edge_bevel_weight(mesh, edge_index, value):
    if bpy.app.version >= (4, 0, 0):
        if "bevel_weight_edge" not in mesh.attributes:
            mesh.attributes.new("bevel_weight_edge", type='FLOAT', domain='EDGE')
        mesh.attributes["bevel_weight_edge"].data[edge_index].value = value
    else:
        mesh.edges[edge_index].bevel_weight = value
```

---

## E-02: Auto Smooth Removal (Removed 4.1)

**Error**: `AttributeError: 'Mesh' object has no attribute 'use_auto_smooth'`

```python
# BROKEN in Blender 4.1+ — Auto smooth properties removed
mesh.use_auto_smooth = True           # AttributeError
mesh.auto_smooth_angle = 0.523599     # AttributeError
mesh.calc_normals_split()             # AttributeError
mesh.create_normals_split()           # AttributeError
mesh.free_normals_split()             # AttributeError

# FIX for Blender 4.1+ — Auto smooth is ALWAYS active
# Reading normals:
corner_normals = mesh.corner_normals  # Read-only, auto-updated

# For angle-based smoothing, use the "Smooth by Angle" modifier:
# This is a built-in geometry node group asset, applied as a modifier.
# There is NO direct Python API equivalent to mesh.auto_smooth_angle.

# For custom normals:
normals_list = [(0.0, 0.0, 1.0)] * len(mesh.loops)
mesh.normals_split_custom_set(normals_list)
# Or from vertices:
normals_vert = [(0.0, 0.0, 1.0)] * len(mesh.vertices)
mesh.normals_split_custom_set_from_vertices(normals_vert)

# VERSION-SAFE pattern
import bpy

def enable_auto_smooth(obj, angle_radians=0.523599):
    mesh = obj.data
    if bpy.app.version >= (4, 1, 0):
        # Auto smooth is always active in 4.1+
        # For angle control, apply "Smooth by Angle" modifier
        pass  # No action needed — auto smooth is implicit
    else:
        mesh.use_auto_smooth = True
        mesh.auto_smooth_angle = angle_radians
```

---

## E-03: Face Maps Removal (Removed 4.0)

**Error**: `AttributeError: 'Object' object has no attribute 'face_maps'`

```python
# BROKEN in Blender 4.0+ — Face maps removed
face_map = obj.face_maps.new(name="Front")  # AttributeError
face_map.add([0, 1, 2, 3])

# FIX for Blender 4.0+ — Integer face attributes
mesh = obj.data
if "face_group" not in mesh.attributes:
    mesh.attributes.new("face_group", type='INT', domain='FACE')
attr = mesh.attributes["face_group"]
for face_idx in [0, 1, 2, 3]:
    attr.data[face_idx].value = 1  # Group ID
```

---

## E-04: Bone Layers and Groups (Removed 4.0)

**Error**: `AttributeError: 'Bone' object has no attribute 'layers'`

```python
# BROKEN in Blender 4.0+ — Bone layers and groups removed
bone.layers[0] = True                        # AttributeError
bone.layers[1] = False                       # AttributeError
group = pose.bone_groups.new(name="IK")      # AttributeError
group.color_set = 'THEME01'

# FIX for Blender 4.0+ — Bone collections
armature = obj.data
collection = armature.collections.new("IK Controls")
collection.assign(bone)
bone.color.palette = 'THEME01'
```

---

## E-05: Node Group Interface (Removed 4.0)

**Error**: `AttributeError: 'ShaderNodeTree' object has no attribute 'inputs'`

```python
# BROKEN in Blender 4.0+ — NodeTree.inputs/outputs removed
node_group.inputs.new("NodeSocketFloat", "Width")
node_group.outputs.new("NodeSocketGeometry", "Result")

# FIX for Blender 4.0+ — interface API
node_group.interface.new_socket(
    name="Width", in_out='INPUT', socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Result", in_out='OUTPUT', socket_type='NodeSocketGeometry'
)

# VERSION-SAFE wrapper
import bpy

def add_node_group_socket(node_group, name, in_out, socket_type):
    if bpy.app.version >= (4, 0, 0):
        node_group.interface.new_socket(
            name=name, in_out=in_out, socket_type=socket_type
        )
    else:
        if in_out == 'INPUT':
            node_group.inputs.new(socket_type, name)
        else:
            node_group.outputs.new(socket_type, name)
```

---

## E-06: Brush sculpt_tool (Removed 5.0)

**Error**: `AttributeError: 'Brush' object has no attribute 'sculpt_tool'`

```python
# BROKEN in Blender 5.0+ — sculpt_tool renamed
brush.sculpt_tool = 'DRAW'       # AttributeError

# FIX for Blender 5.0+
brush.sculpt_brush_type = 'DRAW'

# VERSION-SAFE
import bpy

def set_sculpt_brush_type(brush, brush_type):
    if bpy.app.version >= (5, 0, 0):
        brush.sculpt_brush_type = brush_type
    else:
        brush.sculpt_tool = brush_type
```

---

## E-07: Compositor node_tree (Removed 5.0)

**Error**: `AttributeError: 'Scene' object has no attribute 'node_tree'`

```python
# BROKEN in Blender 5.0+ — scene.node_tree removed for compositor
scene.use_nodes = True          # Also removed in 5.0
comp_tree = scene.node_tree     # AttributeError

# FIX for Blender 5.0+
comp_tree = scene.compositing_node_group
if comp_tree is None:
    comp_tree = bpy.data.node_groups.new("Compositor", 'CompositorNodeTree')
    scene.compositing_node_group = comp_tree

# VERSION-SAFE
import bpy

def get_compositor_tree(scene):
    if bpy.app.version >= (5, 0, 0):
        tree = scene.compositing_node_group
        if tree is None:
            tree = bpy.data.node_groups.new("Compositor", 'CompositorNodeTree')
            scene.compositing_node_group = tree
        return tree
    else:
        scene.use_nodes = True
        return scene.node_tree
```

---

## E-08: VSE end_frame (Removed 5.0)

**Error**: `AttributeError: 'Strip' object has no attribute 'end_frame'`

```python
# BROKEN in Blender 5.0+ — end_frame removed
duration = strip.end_frame - strip.frame_start  # AttributeError

# FIX for Blender 5.0+
duration = strip.length

# VERSION-SAFE
import bpy

def get_strip_duration(strip):
    if bpy.app.version >= (5, 0, 0):
        return strip.length
    return strip.end_frame - strip.frame_start
```

---

## E-09: Legacy Action fcurves (Removed 5.0)

**Error**: `AttributeError: 'Action' object has no attribute 'fcurves'`

```python
# BROKEN in Blender 5.0+ — Legacy action.fcurves removed
for fc in action.fcurves:        # AttributeError
    print(fc.data_path)
for group in action.groups:      # AttributeError
    print(group.name)
root_type = action.id_root       # AttributeError

# FIX for Blender 5.0+ — Slotted Actions channelbag API
for slot in action.slots:
    for layer in action.layers:
        for strip in layer.strips:
            channelbag = strip.channelbag(slot)
            if channelbag:
                for fc in channelbag.fcurves:
                    print(fc.data_path)
root_type = action.slots[0].target_id_type

# Creating keyframes in 5.0+
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
```

---

## E-10: Material Displacement Method (Removed 4.1)

**Error**: `AttributeError: 'CyclesMaterialSettings' object has no attribute 'displacement_method'`

```python
# BROKEN in Blender 4.1+ — Moved from Cycles to Material
mat.cycles.displacement_method = 'BOTH'  # AttributeError

# FIX for Blender 4.1+
mat.displacement_method = 'BOTH'

# VERSION-SAFE
import bpy

def set_displacement_method(mat, method):
    if bpy.app.version >= (4, 1, 0):
        mat.displacement_method = method
    else:
        mat.cycles.displacement_method = method
```

---

## E-11: AttributeGroup Split (Removed 4.3)

**Error**: `AttributeError: 'AttributeGroupPointCloud' object has no attribute 'active_color_name'`

```python
# BROKEN in Blender 4.3+ — Generic AttributeGroup split into type-specific classes
attrs = obj.data.attributes
color = attrs.active_color_name  # AttributeError if obj is not a Mesh

# FIX for Blender 4.3+ — Check object type first
import bpy

if isinstance(obj.data, bpy.types.Mesh):
    color = obj.data.attributes.active_color_name  # Only on AttributeGroupMesh
else:
    color = None  # Not available on non-mesh types
```

---

## E-12: EEVEE Legacy Properties (Removed 4.3)

**Error**: `AttributeError: 'SceneEEVEE' object has no attribute 'use_ssr'`

```python
# BROKEN in Blender 4.3+ — 40+ EEVEE properties removed
scene.eevee.use_ssr = True                   # AttributeError
scene.eevee.use_bloom = True                 # AttributeError
scene.eevee.use_volumetric_lights = True     # AttributeError
scene.eevee.use_gtao = True                  # AttributeError
scene.eevee.bokeh_max_size = 100.0           # AttributeError

# FIX for Blender 4.3+ — EEVEE Next handles these internally
# There are NO replacement properties. Remove these lines entirely.
# EEVEE Next automatically manages SSR, bloom, volumetrics, and AO.
```

---

## E-14: BGL Module Removal (Removed 5.0)

**Error**: `ModuleNotFoundError: No module named 'bgl'`

```python
# BROKEN in Blender 5.0+ — bgl completely removed
import bgl                                    # ModuleNotFoundError
bgl.glEnable(bgl.GL_BLEND)
bgl.glLineWidth(2)
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')

# FIX for Blender 5.0+ (works in 4.0+) — gpu module
import gpu
from gpu_extras.batch import batch_for_shader

gpu.state.blend_set('ALPHA')
gpu.state.line_width_set(2.0)
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')

# Complete draw callback migration:
def draw_callback():
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.depth_mask_set(True)
    gpu.state.line_width_set(2.0)

    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINES', {"pos": coords})
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

---

## E-16: Context Override Dict (Removed 4.0)

**Error**: `TypeError: bpy_struct.default_value: expected keyword arguments, not dict`

```python
# BROKEN in Blender 4.0+ — Dict as first arg to bpy.ops
override = {"object": obj, "active_object": obj}
bpy.ops.object.modifier_apply(override, modifier="Boolean")  # TypeError

override = bpy.context.copy()
override["object"] = obj
bpy.ops.object.select_all(override, action='DESELECT')       # TypeError

# FIX for Blender 4.0+ — temp_override context manager
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Boolean")

with bpy.context.temp_override(object=obj):
    bpy.ops.object.select_all(action='DESELECT')

# VERSION-SAFE wrapper
import bpy

def apply_modifier(obj, modifier_name):
    if bpy.app.version >= (4, 0, 0):
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        override = {"object": obj, "active_object": obj}
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)
```

---

## E-19: Missing super().__init__() (Required 4.4+)

**Error**: `TypeError: MY_OT_operator.__init__() takes 1 positional argument but N were given`

```python
# BROKEN in Blender 4.4+ — Missing super().__init__()
class MY_OT_operator(bpy.types.Operator):
    bl_idname = "my.operator"
    bl_label = "My Operator"

    def __init__(self):            # Crashes in 4.4+
        self.my_data = []

# FIX for Blender 4.4+ — Forward args to parent
class MY_OT_operator(bpy.types.Operator):
    bl_idname = "my.operator"
    bl_label = "My Operator"

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.my_data = []
```

---

## E-20: Principled BSDF Socket Names (Renamed 4.0)

**Error**: `KeyError: 'bpy_prop_collection[key]: key "Subsurface" not found'`

```python
# BROKEN in Blender 4.0+ — Old socket names
principled.inputs["Subsurface"].default_value = 0.5      # KeyError
principled.inputs["Specular"].default_value = 0.5         # KeyError
principled.inputs["Transmission"].default_value = 1.0     # KeyError
principled.inputs["Clearcoat"].default_value = 0.5        # KeyError
principled.inputs["Emission"].default_value = (1,1,1,1)   # KeyError

# FIX for Blender 4.0+ — New socket names
principled.inputs["Subsurface Weight"].default_value = 0.5
principled.inputs["Specular IOR Level"].default_value = 0.5
principled.inputs["Transmission Weight"].default_value = 1.0
principled.inputs["Coat Weight"].default_value = 0.5
principled.inputs["Emission Color"].default_value = (1,1,1,1)

# Full rename map:
# "Subsurface"           → "Subsurface Weight"
# "Subsurface Color"     → "Subsurface Radius"
# "Specular"             → "Specular IOR Level"
# "Sheen"                → "Sheen Weight"
# "Transmission"         → "Transmission Weight"
# "Clearcoat"            → "Coat Weight"
# "Clearcoat Roughness"  → "Coat Roughness"
# "Clearcoat Normal"     → "Coat Normal"
# "Emission"             → "Emission Color"
```

---

## E-21: Dict Access to RNA Properties (Removed 5.0)

**Error**: `KeyError: 'bpy_struct[key]: key "cycles" not found'`

```python
# BROKEN in Blender 5.0+ — Dict-like RNA property access removed
cycles = scene['cycles']            # KeyError
samples = scene['cycles']['samples']

# FIX for Blender 5.0+ — Attribute access
samples = scene.cycles.samples
scene.cycles.samples = 128
```

---

## E-23: EEVEE Identifier Changes (Changed 4.2, Reverted 5.0)

**Error**: `RuntimeError: Render engine 'BLENDER_EEVEE' not found` (in 4.2-4.x)

**Error**: `ValueError: 'BLENDER_EEVEE_NEXT' not in enum` (in 5.0+)

```python
# BROKEN in Blender 4.2-4.x — Old identifier
scene.render.engine = 'BLENDER_EEVEE'      # Not found in 4.2-4.x

# BROKEN in Blender 5.0+ — 4.2 identifier
scene.render.engine = 'BLENDER_EEVEE_NEXT'  # Not found in 5.0+

# FIX — Version-safe EEVEE identifier
import bpy

def get_eevee_identifier():
    if bpy.app.version >= (5, 0, 0):
        return 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        return 'BLENDER_EEVEE_NEXT'
    return 'BLENDER_EEVEE'

scene.render.engine = get_eevee_identifier()
```

---

## E-24: Light Probe Type Renames (Renamed 4.1)

**Error**: `ValueError: bpy_struct: item.attr = val: enum "CUBEMAP" not found`

```python
# BROKEN in Blender 4.1+ — Old enum values
if probe.type == 'CUBEMAP':   # Never matches in 4.1+
    pass
if probe.type == 'PLANAR':    # Never matches
    pass
if probe.type == 'GRID':      # Never matches
    pass

# FIX for Blender 4.1+ — New enum values
if probe.type == 'SPHERE':    # Was CUBEMAP
    pass
if probe.type == 'PLANE':     # Was PLANAR
    pass
if probe.type == 'VOLUME':    # Was GRID
    pass

# VERSION-SAFE mapping
import bpy

PROBE_TYPE_MAP = {
    'CUBEMAP': 'SPHERE',
    'PLANAR': 'PLANE',
    'GRID': 'VOLUME',
}

def get_probe_type(type_name):
    if bpy.app.version >= (4, 1, 0):
        return PROBE_TYPE_MAP.get(type_name, type_name)
    return type_name
```

---

## E-25: GPU Shader Name Changes (Renamed 4.0)

**Error**: `ValueError: '3D_UNIFORM_COLOR' not found in builtin shaders`

```python
# BROKEN in Blender 4.0+ — 3D_ prefix removed
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')  # ValueError
shader = gpu.shader.from_builtin('3D_FLAT_COLOR')     # ValueError
shader = gpu.shader.from_builtin('2D_UNIFORM_COLOR')  # ValueError

# FIX for Blender 4.0+
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
shader = gpu.shader.from_builtin('POLYLINE_FLAT_COLOR')
shader = gpu.shader.from_builtin('UNIFORM_COLOR')

# VERSION-SAFE
import bpy
import gpu

SHADER_NAME_MAP = {
    '3D_UNIFORM_COLOR': 'POLYLINE_UNIFORM_COLOR',
    '3D_FLAT_COLOR': 'POLYLINE_FLAT_COLOR',
    '3D_SMOOTH_COLOR': 'POLYLINE_SMOOTH_COLOR',
    '2D_UNIFORM_COLOR': 'UNIFORM_COLOR',
    '2D_FLAT_COLOR': 'FLAT_COLOR',
    '2D_SMOOTH_COLOR': 'SMOOTH_COLOR',
}

def get_builtin_shader(name):
    if bpy.app.version >= (4, 0, 0):
        name = SHADER_NAME_MAP.get(name, name)
    return gpu.shader.from_builtin(name)
```

---

## VSE Time Properties (Deprecated 5.1, Removed 6.0)

```python
# Blender 5.0 — Current names (DEPRECATED in 5.1)
start = strip.frame_final_start
end = strip.frame_final_end
dur = strip.frame_final_duration

# Blender 5.1+ — New names (forward-compatible)
start = strip.left_handle
end = strip.right_handle
dur = strip.duration

# VERSION-SAFE
import bpy

def get_strip_timing(strip):
    if bpy.app.version >= (5, 1, 0):
        return strip.left_handle, strip.right_handle, strip.duration
    return strip.frame_final_start, strip.frame_final_end, strip.frame_final_duration
```

---

## Sources

- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.2/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
- https://developer.blender.org/docs/release_notes/4.4/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- https://developer.blender.org/docs/release_notes/5.1/python_api/
- https://developer.blender.org/docs/release_notes/compatibility/
