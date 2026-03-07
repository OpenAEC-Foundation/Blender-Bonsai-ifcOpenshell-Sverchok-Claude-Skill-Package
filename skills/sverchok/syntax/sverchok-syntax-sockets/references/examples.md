# sverchok-syntax-sockets — Working Code Examples

Sverchok v1.4.0, Blender 4.0+/5.x
All examples are verified against Sverchok source code.

---

## Example 1: Custom Node with Multiple Socket Types

```python
# Sverchok v1.4.0+ — Custom node defining multiple socket types
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

class SvExampleMultiSocketNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvExampleMultiSocketNode'
    bl_label = 'Example Multi-Socket'
    bl_icon = 'GEOMETRY_NODES'

    count: bpy.props.IntProperty(
        name="Count", default=5, min=1, update=updateNode
    )

    def sv_init(self, context):
        # Input: vertices (orange, nesting_level=3)
        self.inputs.new('SvVerticesSocket', 'Vertices')

        # Input: matrix (teal, nesting_level=1)
        self.inputs.new('SvMatrixSocket', 'Transform')

        # Input: scalar field (dark orange, FieldImplicitConversionPolicy)
        field_sock = self.inputs.new('SvScalarFieldSocket', 'Field')
        # Numbers from SvStringsSocket will auto-convert to SvConstantScalarField

        # Input: number with processing flags
        count_sock = self.inputs.new('SvStringsSocket', 'Count')
        count_sock.use_prop = True
        count_sock.prop_name = 'count'
        count_sock.allow_flatten = True
        count_sock.allow_simplify = True

        # Output: vertices
        self.outputs.new('SvVerticesSocket', 'Vertices')
        # Output: curve
        self.outputs.new('SvCurveSocket', 'Curve')

    def process(self):
        # ALWAYS provide defaults for unconnected sockets
        verts = self.inputs['Vertices'].sv_get(default=[[]])
        transform = self.inputs['Transform'].sv_get(default=[])
        field_data = self.inputs['Field'].sv_get(default=[[]])
        count = self.inputs['Count'].sv_get(default=[[self.count]])

        # Evaluate scalar field at origin
        if field_data and field_data[0]:
            field = field_data[0][0]
            value_at_origin = field.evaluate(0.0, 0.0, 0.0)

        # Output must match nesting_level
        # SvVerticesSocket level=3: [[(x,y,z), ...], ...]
        result_verts = [[(float(i), 0.0, 0.0) for i in range(count[0][0])]]
        self.outputs['Vertices'].sv_set(result_verts)


def register():
    bpy.utils.register_class(SvExampleMultiSocketNode)

def unregister():
    bpy.utils.unregister_class(SvExampleMultiSocketNode)
```

---

## Example 2: Dynamic Socket Type Replacement

```python
# Sverchok v1.4.0+ — Replacing socket type based on enum property
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

MODE_ITEMS = [
    ('VERTICES', 'Vertices', '', 0),
    ('CURVE',    'Curve',    '', 1),
    ('SURFACE',  'Surface',  '', 2),
]

class SvDynamicSocketNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvDynamicSocketNode'
    bl_label = 'Dynamic Socket'

    mode: bpy.props.EnumProperty(
        items=MODE_ITEMS,
        default='VERTICES',
        update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Data')
        self.outputs.new('SvStringsSocket', 'Info')

    def sv_update(self):
        """Called when the node tree topology changes — use for socket replacement."""
        mode_to_type = {
            'VERTICES': 'SvVerticesSocket',
            'CURVE':    'SvCurveSocket',
            'SURFACE':  'SvSurfaceSocket',
        }
        target_type = mode_to_type[self.mode]
        sock = self.inputs.get('Data')
        if sock is None:
            return
        if sock.bl_idname != target_type:
            # ALWAYS sv_forget() before replace_socket()
            sock.sv_forget()
            new_sock = sock.replace_socket(target_type, 'Data')
            # sock is now invalid; use new_sock
            new_sock.label = self.mode.capitalize()

    def process(self):
        sock = self.inputs.get('Data')
        if sock is None or not sock.is_linked:
            self.outputs['Info'].sv_set([[]])
            return
        data = sock.sv_get(default=[[]])
        self.outputs['Info'].sv_set([[len(data)]])
```

---

## Example 3: Accessing socket_id and other

```python
# Sverchok v1.4.0+ — Inspecting socket cache keys and linked sockets
import bpy

tree = bpy.data.node_groups["MyTree"]
node = tree.nodes["MyNode"]

# --- socket_id ---
for sock in node.inputs:
    print(f"Input '{sock.name}': socket_id = {sock.socket_id}")
    # socket_id is stable within a session; re-generated after undo

for sock in node.outputs:
    print(f"Output '{sock.name}': socket_id = {sock.socket_id}")

# --- other ---
# For INPUT sockets:
in_sock = node.inputs['Vertices']
if in_sock.is_linked:
    upstream_sock = in_sock.other
    print(f"Connected from: {upstream_sock.node.name}.{upstream_sock.name}")
    print(f"Upstream type: {upstream_sock.bl_idname}")

# For OUTPUT sockets (ONLY one arbitrary socket returned):
out_sock = node.outputs['Result']
if out_sock.is_linked:
    one_downstream = out_sock.other
    # WARNING: If multiple nodes are connected, this is only ONE of them
    print(f"One downstream: {one_downstream.node.name}.{one_downstream.name}")

    # To get ALL connected sockets for an output:
    all_downstream = [link.to_socket for link in out_sock.links]
    for ds in all_downstream:
        print(f"Downstream: {ds.node.name}.{ds.name}")
```

---

## Example 4: Field Socket — Implicit Conversion

```python
# Sverchok v1.4.0+ — Demonstrating FieldImplicitConversionPolicy
# When SvStringsSocket (numbers) is connected to SvScalarFieldSocket,
# the numbers are automatically wrapped in SvConstantScalarField.

import bpy

tree = bpy.data.node_groups["MyTree"]

with tree.init_tree():
    # Number node → outputs SvStringsSocket data
    num_node = tree.nodes.new('SvNumberNode')
    num_node.location = (0, 0)

    # Iso Surface node → expects SvScalarFieldSocket input
    iso_node = tree.nodes.new('SvExMarchingCubesNode')
    iso_node.location = (200, 0)

    # This connection works because SvScalarFieldSocket uses
    # FieldImplicitConversionPolicy which wraps numbers as SvConstantScalarField
    tree.links.new(
        num_node.outputs[0],       # SvStringsSocket
        iso_node.inputs['Field']   # SvScalarFieldSocket
    )

tree.force_update()

# Inside a node's process() — reading the auto-converted field:
def process(self):
    # Even though a number was connected, sv_get() returns SvScalarField objects
    field_data = self.inputs['Field'].sv_get()
    # field_data: [[SvConstantScalarField], ...]
    for obj_fields in field_data:
        for field in obj_fields:
            val = field.evaluate(1.0, 2.0, 3.0)  # Returns the constant number
            print(val)
```

---

## Example 5: SvCurveSocket with reparametrize

```python
# Sverchok v1.4.0+ — SvCurveSocket reparametrize property
import bpy
from sverchok.node_tree import SverchCustomTreeNode

class SvCurveConsumerNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvCurveConsumerNode'
    bl_label = 'Curve Consumer'

    def sv_init(self, context):
        curve_sock = self.inputs.new('SvCurveSocket', 'Curve')
        # reparametrize=True forces curve to [0, 1] parameter range on input
        # Useful when your algorithm assumes t in [0, 1]
        curve_sock.reparametrize = True

    def process(self):
        curves = self.inputs['Curve'].sv_get(default=[[]])
        for obj_curves in curves:
            for curve in obj_curves:
                t_min, t_max = curve.get_u_bounds()
                # If reparametrize=True, t_min=0.0, t_max=1.0
                mid_point = curve.evaluate((t_min + t_max) / 2)
                print(f"Midpoint: {mid_point}")
```

---

## Example 6: SvDictionarySocket with unzip

```python
# Sverchok v1.4.0+ — Working with SvDictionarySocket
import bpy
from sverchok.utils.dictionary import unzip_dict_recursive

# unzip_dict_recursive converts list-of-dicts to dict-of-lists
data = [
    {'x': 1.0, 'y': 2.0, 'z': 3.0},
    {'x': 4.0, 'y': 5.0, 'z': 6.0},
]
result = unzip_dict_recursive(data)
# result: {'x': [1.0, 4.0], 'y': [2.0, 5.0], 'z': [3.0, 6.0]}

# Inside a node reading dict data:
def process(self):
    dict_data = self.inputs['Dict'].sv_get(default=[[]])
    # dict_data: [[dict, dict, ...], ...]
    for obj_dicts in dict_data:
        unzipped = unzip_dict_recursive(obj_dicts)
        x_values = unzipped.get('x', [])
        print(x_values)
```

---

## Example 7: Using SvObjectSocket with object_kinds

```python
# Sverchok v1.4.0+ — SvObjectSocket filtering by object type
import bpy
from sverchok.node_tree import SverchCustomTreeNode

class SvMeshOnlyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMeshOnlyNode'
    bl_label = 'Mesh Only Input'

    def sv_init(self, context):
        obj_sock = self.inputs.new('SvObjectSocket', 'Object')
        # Filter to only show MESH objects in the picker UI
        obj_sock.object_kinds = 'MESH'
        # Other valid values: 'CURVE', 'SURFACE', 'FONT', 'META',
        #                     'ARMATURE', 'LATTICE', 'EMPTY', 'GPENCIL',
        #                     'LIGHT', 'CAMERA', 'ALL' (default)

    def process(self):
        objects = self.inputs['Object'].sv_get(default=[[]])
        # objects: [[bpy.types.Object, ...], ...]
        for obj_list in objects:
            for obj in obj_list:
                print(obj.name, obj.type)
```

---

## Example 8: SvSolidSocket — requires FreeCAD

```python
# Sverchok v1.4.0+ — SvSolidSocket (FreeCAD/OpenCascade dependency)
import bpy
from sverchok.node_tree import SverchCustomTreeNode

class SvSolidInfoNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvSolidInfoNode'
    bl_label = 'Solid Info'

    def sv_init(self, context):
        # SolidImplicitConversionPolicy: attempts to_solid_recursive()
        self.inputs.new('SvSolidSocket', 'Solid')
        self.outputs.new('SvStringsSocket', 'Volume')

    def process(self):
        # ALWAYS guard against missing FreeCAD
        try:
            import Part
        except ImportError:
            self.outputs['Volume'].sv_set([[0.0]])
            return

        solids = self.inputs['Solid'].sv_get(default=[[]])
        volumes = []
        for obj_solids in solids:
            obj_volumes = []
            for solid in obj_solids:
                # solid is Part.Shape
                obj_volumes.append(solid.Volume)
            volumes.append(obj_volumes)
        self.outputs['Volume'].sv_set(volumes)
```
