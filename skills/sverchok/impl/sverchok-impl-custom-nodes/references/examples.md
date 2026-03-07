# sverchok-impl-custom-nodes — Working Examples

> Sverchok v1.4.0+ / Blender 4.0+/5.x

## Example 1: Vertex Filter Node (Multi-Output, Properties, UI)

A node that filters vertices by axis threshold, demonstrating multiple outputs, EnumProperty, and sv_draw_buttons.

```python
import bpy
from bpy.props import FloatProperty, EnumProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvVertexFilterNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: filter vertices axis threshold
    Tooltip: Filters vertices by coordinate threshold on selected axis
    """
    bl_idname = 'SvVertexFilterNode'
    bl_label = 'Vertex Filter'
    bl_icon = 'FILTER'

    axis: EnumProperty(
        name='Axis',
        items=[('X', 'X', ''), ('Y', 'Y', ''), ('Z', 'Z', '')],
        default='Z',
        update=updateNode
    )

    threshold: FloatProperty(
        name='Threshold', default=0.0, update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Threshold').prop_name = 'threshold'
        self.outputs.new('SvVerticesSocket', 'Above')
        self.outputs.new('SvVerticesSocket', 'Below')
        self.outputs.new('SvStringsSocket', 'Mask')

    def sv_draw_buttons(self, context, layout):
        layout.prop(self, 'axis', expand=True)

    def process(self):
        if not any(s.is_linked for s in self.outputs):
            return

        verts = self.inputs['Vertices'].sv_get(default=[[]])
        thresh = self.inputs['Threshold'].sv_get(default=[[self.threshold]])
        verts, thresh = match_long_repeat([verts, thresh])

        axis_idx = {'X': 0, 'Y': 1, 'Z': 2}[self.axis]
        out_above, out_below, out_mask = [], [], []

        for vert_list, thresh_list in zip(verts, thresh):
            above, below, mask = [], [], []
            for v, t in zip(vert_list, repeat_last(thresh_list, len(vert_list))):
                if v[axis_idx] >= t:
                    above.append(v)
                    mask.append(1)
                else:
                    below.append(v)
                    mask.append(0)
            out_above.append(above)
            out_below.append(below)
            out_mask.append(mask)

        self.outputs['Above'].sv_set(out_above)
        self.outputs['Below'].sv_set(out_below)
        self.outputs['Mask'].sv_set(out_mask)


def repeat_last(lst, count):
    """Yield elements from lst, repeating the last element to reach count."""
    for i in range(count):
        yield lst[min(i, len(lst) - 1)]


classes = [SvVertexFilterNode]
register, unregister = bpy.utils.register_classes_factory(classes)
```

## Example 2: BMesh Boolean Node (Geometry Processing)

A node that performs BMesh boolean operations on two mesh inputs.

```python
import bpy
import bmesh
from bpy.props import EnumProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata, pydata_from_bmesh

class SvSimpleBooleanNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: boolean union difference intersect mesh
    Tooltip: Performs boolean operation on two meshes using BMesh
    """
    bl_idname = 'SvSimpleBooleanNode'
    bl_label = 'Simple Boolean'

    operation: EnumProperty(
        name='Operation',
        items=[
            ('UNION', 'Union', 'Combine meshes'),
            ('DIFFERENCE', 'Difference', 'Subtract B from A'),
            ('INTERSECT', 'Intersect', 'Keep overlapping volume'),
        ],
        default='UNION',
        update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Verts A')
        self.inputs.new('SvStringsSocket', 'Faces A')
        self.inputs.new('SvVerticesSocket', 'Verts B')
        self.inputs.new('SvStringsSocket', 'Faces B')
        self.outputs.new('SvVerticesSocket', 'Vertices')
        self.outputs.new('SvStringsSocket', 'Edges')
        self.outputs.new('SvStringsSocket', 'Faces')

    def sv_draw_buttons(self, context, layout):
        layout.prop(self, 'operation', text='')

    def process(self):
        if not self.outputs['Vertices'].is_linked:
            return

        verts_a = self.inputs['Verts A'].sv_get()
        faces_a = self.inputs['Faces A'].sv_get()
        verts_b = self.inputs['Verts B'].sv_get()
        faces_b = self.inputs['Faces B'].sv_get()

        out_v, out_e, out_f = [], [], []

        for va, fa, vb, fb in zip(verts_a, faces_a, verts_b, faces_b):
            bm_a = bmesh_from_pydata(va, [], fa, normal_update=True)
            bm_b = bmesh_from_pydata(vb, [], fb, normal_update=True)

            # BMesh boolean
            result = bmesh.ops.boolean(
                bm_a,
                bm=bm_b,
                operation=self.operation
            )

            new_v, new_e, new_f = pydata_from_bmesh(bm_a)
            bm_a.free()
            bm_b.free()

            out_v.append(new_v)
            out_e.append(new_e)
            out_f.append(new_f)

        self.outputs['Vertices'].sv_set(out_v)
        self.outputs['Edges'].sv_set(out_e)
        self.outputs['Faces'].sv_set(out_f)


classes = [SvSimpleBooleanNode]
register, unregister = bpy.utils.register_classes_factory(classes)
```

## Example 3: NumPy Noise Node (Vectorized Performance)

A high-performance vertex displacement node using NumPy.

```python
import bpy
import numpy as np
from bpy.props import FloatProperty, IntProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvNumpyNoiseNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: noise displacement random numpy
    Tooltip: Displaces vertices with random noise using NumPy
    """
    bl_idname = 'SvNumpyNoiseNode'
    bl_label = 'NumPy Noise'

    amplitude: FloatProperty(
        name='Amplitude', default=0.1, min=0.0, update=updateNode
    )
    seed: IntProperty(
        name='Seed', default=0, min=0, update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.sv_new_input('SvStringsSocket', 'Amplitude',
                          prop_name='amplitude')
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def sv_draw_buttons(self, context, layout):
        layout.prop(self, 'seed')

    def process(self):
        if not self.outputs['Vertices'].is_linked:
            return

        verts = self.inputs['Vertices'].sv_get()
        amp = self.inputs['Amplitude'].sv_get(default=[[self.amplitude]])
        verts, amp = match_long_repeat([verts, amp])

        result = []
        for v_list, a_list in zip(verts, amp):
            v_arr = np.array(v_list, dtype=np.float64)
            rng = np.random.default_rng(self.seed)
            noise = rng.uniform(-1, 1, v_arr.shape)

            # Broadcast amplitude across vertices
            a_arr = np.array(a_list, dtype=np.float64)
            if len(a_arr) < len(v_arr):
                a_arr = np.pad(a_arr, (0, len(v_arr) - len(a_arr)),
                               mode='edge')
            a_arr = a_arr[:len(v_arr)].reshape(-1, 1)

            displaced = v_arr + noise * a_arr
            result.append(displaced.tolist())

        self.outputs['Vertices'].sv_set(result)


classes = [SvNumpyNoiseNode]
register, unregister = bpy.utils.register_classes_factory(classes)
```

## Example 4: Animation-Dependent Node (Frame Access)

A node that uses the current frame number, demonstrating `is_animation_dependent`.

```python
import bpy
import math
from bpy.props import FloatProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

class SvAnimatedWaveNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: wave animate sin oscillate
    Tooltip: Generates an animated sine wave driven by the current frame
    """
    bl_idname = 'SvAnimatedWaveNode'
    bl_label = 'Animated Wave'

    is_animation_dependent = True  # Re-evaluate on frame change

    frequency: FloatProperty(
        name='Frequency', default=1.0, min=0.01, update=updateNode
    )
    amplitude: FloatProperty(
        name='Amplitude', default=1.0, min=0.0, update=updateNode
    )

    def sv_init(self, context):
        self.outputs.new('SvStringsSocket', 'Value')
        self.outputs.new('SvStringsSocket', 'Frame')

    def sv_draw_buttons(self, context, layout):
        layout.prop(self, 'frequency')
        layout.prop(self, 'amplitude')

    def process(self):
        if not any(s.is_linked for s in self.outputs):
            return

        frame = bpy.context.scene.frame_current
        value = self.amplitude * math.sin(
            2 * math.pi * self.frequency * frame / 24.0
        )

        self.outputs['Value'].sv_set([[value]])
        self.outputs['Frame'].sv_set([[frame]])


classes = [SvAnimatedWaveNode]
register, unregister = bpy.utils.register_classes_factory(classes)
```

## Example 5: Node with External Dependencies

A node requiring SciPy, demonstrating `sv_dependencies` and error handling.

```python
import bpy
from bpy.props import IntProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode
from sverchok.core.sv_custom_exceptions import SvNoDataError

class SvConvexHullSciPyNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: convex hull scipy spatial
    Tooltip: Computes 3D convex hull using SciPy
    """
    bl_idname = 'SvConvexHullSciPyNode'
    bl_label = 'ConvexHull (SciPy)'

    sv_dependencies = {'scipy'}

    def sv_init(self, context):
        self.sv_new_input('SvVerticesSocket', 'Vertices', is_mandatory=True)
        self.outputs.new('SvVerticesSocket', 'Hull Vertices')
        self.outputs.new('SvStringsSocket', 'Hull Faces')

    def process(self):
        if self.dependency_error:
            raise self.dependency_error

        if not self.outputs['Hull Vertices'].is_linked:
            return

        from scipy.spatial import ConvexHull

        try:
            verts = self.inputs['Vertices'].sv_get()
        except SvNoDataError:
            raise  # Propagate "no data" state

        out_v, out_f = [], []
        for v_list in verts:
            if len(v_list) < 4:
                self.warning("Need at least 4 vertices for 3D convex hull")
                out_v.append([])
                out_f.append([])
                continue

            hull = ConvexHull(v_list)
            hull_verts = [v_list[i] for i in hull.vertices]
            # Remap face indices to hull vertex indices
            idx_map = {old: new for new, old in enumerate(hull.vertices)}
            hull_faces = [tuple(idx_map[i] for i in face)
                          for face in hull.simplices]

            out_v.append(hull_verts)
            out_f.append(hull_faces)

        self.outputs['Hull Vertices'].sv_set(out_v)
        self.outputs['Hull Faces'].sv_set(out_f)


classes = [SvConvexHullSciPyNode]
register, unregister = bpy.utils.register_classes_factory(classes)
```

## Example 6: External Add-on Registration (Full bl_info)

Packaging custom nodes as a standalone Blender add-on.

```python
bl_info = {
    "name": "My Sverchok Nodes",
    "author": "Author Name",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "location": "Node Editor > Sverchok",
    "description": "Custom Sverchok nodes for architectural geometry",
    "category": "Node",
}

import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

class SvMyNodeA(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: custom nodeA
    Tooltip: First custom node
    """
    bl_idname = 'SvMyNodeA'
    bl_label = 'My Node A'

    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Input')
        self.outputs.new('SvStringsSocket', 'Output')

    def process(self):
        if not self.outputs['Output'].is_linked:
            return
        data = self.inputs['Input'].sv_get(default=[[0]])
        self.outputs['Output'].sv_set(data)


class SvMyNodeB(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: custom nodeB
    Tooltip: Second custom node
    """
    bl_idname = 'SvMyNodeB'
    bl_label = 'My Node B'

    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Input')
        self.outputs.new('SvStringsSocket', 'Output')

    def process(self):
        if not self.outputs['Output'].is_linked:
            return
        data = self.inputs['Input'].sv_get(default=[[0]])
        self.outputs['Output'].sv_set(data)


def register():
    bpy.utils.register_class(SvMyNodeA)
    bpy.utils.register_class(SvMyNodeB)

def unregister():
    # ALWAYS unregister in reverse order
    bpy.utils.unregister_class(SvMyNodeB)
    bpy.utils.unregister_class(SvMyNodeA)
```

## Example 7: Node with sv_copy and sv_free (Resource Management)

A node managing external resources that must be cleaned up on copy/delete.

```python
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

class SvCachedDataNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: cache data store
    Tooltip: Caches processed data between evaluations
    """
    bl_idname = 'SvCachedDataNode'
    bl_label = 'Cached Data'

    _cache = {}  # Class-level cache keyed by node_id

    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Data')
        self.outputs.new('SvStringsSocket', 'Cached')

    def sv_copy(self, original):
        """Reset cache for the new copy — do NOT share original's cache."""
        # n_id is automatically refreshed by the copy() @final method
        pass

    def sv_free(self):
        """Clean up cache entry when node is deleted."""
        if self.node_id in self._cache:
            del self._cache[self.node_id]

    def process(self):
        if not self.outputs['Cached'].is_linked:
            return
        data = self.inputs['Data'].sv_get(default=None)
        if data is not None:
            self._cache[self.node_id] = data
        cached = self._cache.get(self.node_id, [[]])
        self.outputs['Cached'].sv_set(cached)


classes = [SvCachedDataNode]
register, unregister = bpy.utils.register_classes_factory(classes)
```
