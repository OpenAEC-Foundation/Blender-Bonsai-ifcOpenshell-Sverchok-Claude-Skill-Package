# sverchok-syntax-data: Working Examples

> All examples verified against Sverchok v1.4.0+ source code.
> Blender 4.0+/5.x.

---

## 1. Data Nesting Fundamentals

### 1.1 Correct Nesting Per Socket Type

```python
# Sverchok v1.4.0+
# --- SvVerticesSocket: Level 3 ---
# ONE object with 4 vertices (a quad)
vertices_1obj = [[(0,0,0), (1,0,0), (1,1,0), (0,1,0)]]

# TWO objects
vertices_2obj = [
    [(0,0,0), (1,0,0), (1,1,0), (0,1,0)],   # Object 0: quad
    [(2,0,0), (3,0,0), (2.5,1,0)],            # Object 1: triangle
]

# --- SvStringsSocket: Level 2 ---
# ONE object with 5 numeric values
numbers_1obj = [[1.0, 2.0, 3.0, 4.0, 5.0]]

# THREE objects with varying value counts
numbers_3obj = [[10, 20], [30], [40, 50, 60]]

# --- SvMatrixSocket: Level 1 ---
from mathutils import Matrix
matrices = [Matrix(), Matrix.Translation((1, 0, 0))]

# --- Edge data: Level 2 ---
edges_1obj = [[(0,1), (1,2), (2,3), (3,0)]]

# --- Face data: Level 2 ---
faces_1obj = [[(0,1,2,3)]]  # One quad face
faces_2obj = [[(0,1,2,3)], [(0,1,2)]]  # Object 0: quad, Object 1: triangle
```

### 1.2 Detecting Nesting Levels

```python
# Sverchok v1.4.0+
from sverchok.data_structure import get_data_nesting_level, describe_data_shape

# Scalars
get_data_nesting_level(42)                          # 0
get_data_nesting_level(3.14)                        # 0

# Flat list
get_data_nesting_level([1, 2, 3])                   # 1

# Number socket data (level 2)
get_data_nesting_level([[1, 2, 3]])                 # 2

# Vertex socket data (level 3)
get_data_nesting_level([[(0,0,0), (1,0,0)]])        # 3

# Empty structures
get_data_nesting_level([])                          # 1
get_data_nesting_level([[]])                        # 2

# Debugging with describe_data_shape
describe_data_shape([[(0,0,0), (1,0,0)]])
# "Level 3: list [1] of list [2] of tuple [3] of float"

describe_data_shape([[1, 2, 3], [4, 5]])
# "Level 2: list [2] of list [3] of int"

describe_data_shape(None)
# "Level 0: NoneType"
```

### 1.3 Adjusting Nesting Levels

```python
# Sverchok v1.4.0+
from sverchok.data_structure import (
    ensure_nesting_level, ensure_min_nesting,
    flatten_data, graft_data
)

# Wrap up to target level
ensure_nesting_level(5, 0)                # 5
ensure_nesting_level(5, 1)                # [5]
ensure_nesting_level(5, 2)               # [[5]]
ensure_nesting_level([1, 2, 3], 2)        # [[1, 2, 3]]
ensure_nesting_level([(0,0,0)], 3)        # [[(0,0,0)]]

# ensure_nesting_level RAISES if already too deep
# ensure_nesting_level([[[1]]], 2)  # TypeError!

# ensure_min_nesting is safe with deeper nesting
ensure_min_nesting([[[1]]], 2)            # [[[1]]] (no change)
ensure_min_nesting([1, 2], 2)             # [[1, 2]]

# Flatten (reduce nesting)
flatten_data([[[1, 2], [3, 4]], [[5, 6]]], target_level=1)
# [1, 2, 3, 4, 5, 6]

flatten_data([[[1, 2], [3, 4]], [[5, 6]]], target_level=2)
# [[1, 2], [3, 4], [5, 6]]

# Graft (add nesting at a specific depth)
graft_data([[1, 2], [3, 4]], item_level=0, wrap_level=1)
# [[[1], [2]], [[3], [4]]]
```

---

## 2. List Matching

### 2.1 Basic Matching with match_long_repeat

```python
# Sverchok v1.4.0+
from sverchok.data_structure import match_long_repeat

# Matching vertex objects with scale values
verts = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]  # 2 objects
scales = [[1.5]]  # 1 object

matched_verts, matched_scales = match_long_repeat([verts, scales])
# matched_verts:  [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]  (unchanged)
# matched_scales: [[1.5], [1.5]]  (last element repeated)

# Three inputs of different lengths
a = [1, 2, 3, 4, 5]
b = [10, 20]
c = [100]

ma, mb, mc = match_long_repeat([a, b, c])
# ma: [1, 2, 3, 4, 5]
# mb: [10, 20, 20, 20, 20]
# mc: [100, 100, 100, 100, 100]
```

### 2.2 Convenient Iteration with zip_long_repeat

```python
# Sverchok v1.4.0+
from sverchok.data_structure import zip_long_repeat

verts = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]
scales = [[2.0]]

# zip_long_repeat = zip(*match_long_repeat(lists))
for v, s in zip_long_repeat(verts, scales):
    # Iteration 1: v=[(0,0,0), (1,0,0)], s=[2.0]
    # Iteration 2: v=[(2,0,0), (3,0,0), (4,0,0)], s=[2.0]
    scaled = [(x*s[0], y*s[0], z*s[0]) for x, y, z in v]
```

### 2.3 All Five Modes Compared

```python
# Sverchok v1.4.0+
from sverchok.data_structure import list_match_func

a = [1, 2, 3, 4]
b = [10, 20]

# REPEAT: repeat last element of shorter list
list_match_func["REPEAT"]([a, b])
# [[1, 2, 3, 4], [10, 20, 20, 20]]

# CYCLE: cycle shorter list from beginning
list_match_func["CYCLE"]([a, b])
# [[1, 2, 3, 4], [10, 20, 10, 20]]

# SHORT: truncate to shortest
list_match_func["SHORT"]([a, b])
# [[1, 2], [10, 20]]

# XREF: cross product (all combinations)
list_match_func["XREF"]([a, b])
# [[1, 1, 2, 2, 3, 3, 4, 4], [10, 20, 10, 20, 10, 20, 10, 20]]
# = 4 * 2 = 8 pairs

# XREF2: cross product (reversed nesting)
list_match_func["XREF2"]([a, b])
# [[1, 2, 3, 4, 1, 2, 3, 4], [10, 10, 10, 10, 20, 20, 20, 20]]
```

### 2.4 Using fullList and repeat_last

```python
# Sverchok v1.4.0+
from sverchok.data_structure import fullList, fullList_deep_copy, repeat_last, repeat_last_for_length

# fullList: extend IN-PLACE (shallow copies of last element)
data = [1, 2, 3]
fullList(data, 7)
# data is now [1, 2, 3, 3, 3, 3, 3]

# fullList_deep_copy: extend IN-PLACE with deep copies (safe for mutable items)
data = [[1, 2], [3, 4]]
fullList_deep_copy(data, 4)
# data is now [[1, 2], [3, 4], [3, 4], [3, 4]]  (each [3,4] is independent copy)

# repeat_last: infinite iterator (ALWAYS pair with a terminating consumer)
gen = repeat_last([10, 20, 30])
result = [next(gen) for _ in range(8)]
# [10, 20, 30, 30, 30, 30, 30, 30]

# repeat_last_for_length: returns new list of exact length
repeat_last_for_length([5, 10], 5)
# [5, 10, 10, 10, 10]

repeat_last_for_length([5, 10, 15, 20], 2)
# [5, 10]  (truncates if source is longer)
```

### 2.5 NumPy Matching

```python
# Sverchok v1.4.0+
import numpy as np
from sverchok.data_structure import (
    numpy_list_match_func,
    numpy_full_list, numpy_full_list_cycle
)

arr_a = np.array([[0, 0, 0], [1, 0, 0], [2, 0, 0]])  # 3 rows
arr_b = np.array([[1, 1, 1]])                           # 1 row

# Match by repeating last row
matched = numpy_list_match_func["REPEAT"]([arr_a, arr_b])
# matched[0]: shape (3, 3) — unchanged
# matched[1]: shape (3, 3) — [[1,1,1], [1,1,1], [1,1,1]]

# Extend single array
result = numpy_full_list(np.array([1, 2, 3]), 6)
# array([1, 2, 3, 3, 3, 3])

result = numpy_full_list_cycle(np.array([1, 2, 3]), 7)
# array([1, 2, 3, 1, 2, 3, 1])
```

---

## 3. SvRecursiveNode Examples

### 3.1 Minimal SvRecursiveNode

```python
# Sverchok v1.4.0+
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.utils.nodes_mixins.recursive_nodes import SvRecursiveNode

class SvDoubleVerticesNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    """Doubles all vertex coordinates"""
    bl_idname = 'SvDoubleVerticesNode'
    bl_label = 'Double Vertices'

    def sv_init(self, context):
        s = self.inputs.new('SvVerticesSocket', "Vertices")
        s.is_mandatory = True
        s.nesting_level = 3
        s.default_mode = 'NONE'

        self.outputs.new('SvVerticesSocket', "Vertices")

    def process_data(self, params):
        verts, = params
        # verts is a single object's vertex list: [(x,y,z), ...]
        return [(v[0]*2, v[1]*2, v[2]*2) for v in verts]
```

### 3.2 SvRecursiveNode with Multiple Inputs and Outputs

```python
# Sverchok v1.4.0+
import bpy
from bpy.props import FloatProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode
from sverchok.utils.nodes_mixins.recursive_nodes import SvRecursiveNode

class SvOffsetVerticesNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    """Offset vertices and report distances"""
    bl_idname = 'SvOffsetVerticesNode'
    bl_label = 'Offset Vertices'

    falloff: FloatProperty(name="Falloff", default=1.0, update=updateNode)

    def sv_init(self, context):
        v = self.inputs.new('SvVerticesSocket', "Vertices")
        v.is_mandatory = True
        v.nesting_level = 3
        v.default_mode = 'NONE'

        d = self.inputs.new('SvStringsSocket', "Distance")
        d.nesting_level = 2
        d.default_mode = 'EMPTY_LIST'
        d.pre_processing = 'ONE_ITEM'  # one distance per object

        self.outputs.new('SvVerticesSocket', "Vertices")
        self.outputs.new('SvStringsSocket', "Distances")

    def process_data(self, params):
        verts, dist = params
        if not dist:
            dist = 1.0
        new_verts = [(v[0], v[1], v[2] + dist * self.falloff) for v in verts]
        distances = [dist * self.falloff] * len(verts)
        return new_verts, distances

    def draw_buttons(self, context, layout):
        layout.prop(self, 'falloff')

    def draw_buttons_ext(self, context, layout):
        layout.prop(self, 'list_match')  # Show matching mode in sidebar
```

### 3.3 SvRecursiveNode with Dynamic Socket Configuration

```python
# Sverchok v1.4.0+
import bpy
from bpy.props import EnumProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode
from sverchok.utils.nodes_mixins.recursive_nodes import SvRecursiveNode

class SvFlexibleNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    bl_idname = 'SvFlexibleNode'
    bl_label = 'Flexible Node'

    mode: EnumProperty(
        name="Mode",
        items=[("PER_VERT", "Per Vertex", ""), ("PER_OBJECT", "Per Object", "")],
        default="PER_VERT",
        update=updateNode
    )

    def sv_init(self, context):
        s = self.inputs.new('SvStringsSocket', "Values")
        s.is_mandatory = True
        s.nesting_level = 2
        self.outputs.new('SvStringsSocket', "Result")

    def pre_setup(self):
        # Dynamic nesting level based on mode
        if self.mode == 'PER_VERT':
            self.inputs['Values'].nesting_level = 2
        else:
            self.inputs['Values'].nesting_level = 1

    def process_data(self, params):
        values, = params
        return [v * 2 for v in values]
```

### 3.4 SvRecursiveNode with BMesh Support

```python
# Sverchok v1.4.0+
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.utils.nodes_mixins.recursive_nodes import SvRecursiveNode

class SvBMeshAnalyzerNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    """Analyze BMesh geometry"""
    bl_idname = 'SvBMeshAnalyzerNode'
    bl_label = 'BMesh Analyzer'

    def sv_init(self, context):
        v = self.inputs.new('SvVerticesSocket', "Vertices")
        v.is_mandatory = True
        v.nesting_level = 3

        e = self.inputs.new('SvStringsSocket', "Edges")
        e.nesting_level = 2
        e.default_mode = 'EMPTY_LIST'

        f = self.inputs.new('SvStringsSocket', "Faces")
        f.nesting_level = 2
        f.default_mode = 'EMPTY_LIST'

        self.outputs.new('SvStringsSocket', "FaceCount")

    # Enable bmesh construction
    build_bmesh = True
    bmesh_inputs = [0, 1, 2]  # indices of verts, edges, faces sockets

    def process_data(self, params):
        bmesh_list, = params  # first param is now list of bmesh objects
        counts = []
        for bm in bmesh_list:
            counts.append(len(bm.faces))
            bm.free()  # ALWAYS free bmesh when done
        return counts
```

---

## 4. Vectorize Decorator Examples

### 4.1 Single Output Function

```python
# Sverchok v1.4.0+
from typing import List, Tuple
from sverchok.utils.vectorize import vectorize

def offset_vertices(*, vertices: List[Tuple[float, float, float]],
                       offset: float) -> list:
    """Offset all vertices by a scalar amount in Z."""
    return [(v[0], v[1], v[2] + offset) for v in vertices]

# In a node's process() method:
verts = self.inputs['Vertices'].sv_get()    # [[(0,0,0),(1,0,0)], [(2,0,0)]]
offsets = self.inputs['Offset'].sv_get()     # [[1.0], [2.0]]

fn = vectorize(offset_vertices, match_mode="REPEAT")
result = fn(vertices=verts, offset=offsets)  # MUST use keyword arguments
# result: [[(0,0,1),(1,0,1)], [(2,0,2)]]
self.outputs['Vertices'].sv_set(result)
```

### 4.2 Multiple Output Function

```python
# Sverchok v1.4.0+
from typing import List, Tuple
from sverchok.utils.vectorize import vectorize

def analyze_edges(*, vertices: List[Tuple[float, float, float]],
                     edges: List[Tuple[int, int]]) -> Tuple[list, list]:
    """Return edge midpoints and lengths."""
    import math
    midpoints = []
    lengths = []
    for e in edges:
        v0, v1 = vertices[e[0]], vertices[e[1]]
        mid = ((v0[0]+v1[0])/2, (v0[1]+v1[1])/2, (v0[2]+v1[2])/2)
        length = math.sqrt(sum((a-b)**2 for a, b in zip(v0, v1)))
        midpoints.append(mid)
        lengths.append(length)
    return midpoints, lengths

# Usage — Tuple[list, list] return annotation signals 2 outputs:
fn = vectorize(analyze_edges, match_mode="REPEAT")
mids, lens = fn(vertices=verts_data, edges=edges_data)
```

### 4.3 Using as a Decorator

```python
# Sverchok v1.4.0+
from sverchok.utils.vectorize import vectorize
from typing import List

@vectorize(match_mode="REPEAT")
def scale_values(*, values: List[float], factor: float) -> list:
    return [v * factor for v in values]

# Direct call with nested data:
result = scale_values(values=[[1,2,3],[4,5]], factor=[[2],[3]])
# result: [[2,4,6], [12,15]]
```

---

## 5. match_sockets Examples

### 5.1 Object-Level Iteration

```python
# Sverchok v1.4.0+
from sverchok.utils.vectorize import match_sockets

# Process vertices with per-vertex colors
verts = [[(0,0,0), (1,0,0), (1,1,0)],     # Object 0: 3 verts
         [(2,0,0), (3,0,0)]]                # Object 1: 2 verts
colors = [[(1,0,0), (0,1,0), (0,0,1)]]     # 1 object, 3 colors

result_verts = []
result_colors = []
for v, c in match_sockets(verts, colors):
    # Object 0: v has 3 verts, c has 3 colors (exact match)
    # Object 1: v has 2 verts, c from repeated object, padded to length 2
    result_verts.append(v)
    result_colors.append(c)
```

### 5.2 Three-Input Matching

```python
# Sverchok v1.4.0+
from sverchok.utils.vectorize import match_sockets

verts = [[(0,0,0), (1,0,0)]]          # 1 object, 2 verts
radii = [[0.1, 0.2]]                   # 1 object, 2 radii
segments = [[8]]                        # 1 object, 1 segment count

for v, r, s in match_sockets(verts, radii, segments):
    # v = [(0,0,0), (1,0,0)]
    # r = [0.1, 0.2]
    # s = [8, 8]  (extended to match length 2)
    for vert, radius, seg in zip(v, r, s):
        create_sphere(vert, radius, seg)
```

---

## 6. Recursive Processing Examples

### 6.1 recurse_fx: Unary Leaf Transform

```python
# Sverchok v1.4.0+
from sverchok.utils.sv_itertools import recurse_fx

# Double all values at any nesting depth
data = [[[1, 2], [3]], [[4, 5, 6]]]
result = recurse_fx(data, lambda x: x * 2)
# [[[2, 4], [6]], [[8, 10, 12]]]

# Round all floats
data = [[1.234, 5.678], [9.012]]
result = recurse_fx(data, lambda x: round(x, 1))
# [[1.2, 5.7], [9.0]]

# Convert to string
result = recurse_fx([1, [2, [3]]], str)
# ['1', ['2', ['3']]]
```

### 6.2 recurse_fxy: Binary Leaf Operations

```python
# Sverchok v1.4.0+
from sverchok.utils.sv_itertools import recurse_fxy

# Add corresponding elements (with repeat-last matching)
result = recurse_fxy([1, 2, 3], [10, 20], lambda x, y: x + y)
# [11, 22, 33]  (20 is repeated for the third element)

# Multiply nested structures
result = recurse_fxy([[1, 2], [3, 4]], [[10], [20, 30]], lambda x, y: x * y)
# [[10, 20], [60, 120]]

# Scalar broadcast to list
result = recurse_fxy(5, [1, 2, 3], lambda x, y: x + y)
# [6, 7, 8]
```

### 6.3 recurse_f_level_control

```python
# Sverchok v1.4.0+
from sverchok.utils.sv_itertools import recurse_f_level_control
from sverchok.data_structure import match_long_repeat

def my_func(params, constant, matching_f):
    values, weights = matching_f(params)
    return [v * w * constant for v, w in zip(values, weights)]

data_values = [[1, 2, 3], [4, 5]]
data_weights = [[0.5]]

result = recurse_f_level_control(
    [data_values, data_weights],
    constant=2.0,
    main_func=my_func,
    matching_f=match_long_repeat,
    desired_levels=[1, 1],
    concatenate="APPEND"
)
# Each sub-list processed separately with matching
```

### 6.4 process_matched (Engine Behind SvRecursiveNode)

```python
# Sverchok v1.4.0+
from sverchok.utils.sv_itertools import process_matched

def double_verts(params):
    """Process function receives matched params at target nesting."""
    verts, = params
    return [(v[0]*2, v[1]*2, v[2]*2) for v in verts]

# Input: 2 objects of vertices
input_verts = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]

result = process_matched(
    [input_verts],           # params: list of inputs
    double_verts,            # function to apply
    "REPEAT",                # matching mode
    [3],                     # input_nesting: level 3 for vertices
    1                        # outputs_num: single output
)
# result: [[(0,0,0), (2,0,0)], [(4,0,0), (6,0,0), (8,0,0)]]
```

### 6.5 process_matched with Multiple Outputs

```python
# Sverchok v1.4.0+
from sverchok.utils.sv_itertools import process_matched

def analyze_verts(params):
    """Return transformed vertices AND count."""
    verts, scales = params
    scaled = [(v[0]*scales[0], v[1]*scales[0], v[2]*scales[0]) for v in verts]
    return scaled, [len(verts)]

input_verts = [[(0,0,0), (1,0,0)], [(2,0,0)]]
input_scales = [[2.0], [3.0]]

result = process_matched(
    [input_verts, input_scales],
    analyze_verts,
    "REPEAT",
    [3, 2],      # verts at level 3, scales at level 2
    2            # 2 outputs
)
# result[0]: scaled vertices per object
# result[1]: vertex count per object
```

---

## 7. Complete Node Example: Full Pipeline

```python
# Sverchok v1.4.0+ — Complete custom node using SvRecursiveNode
import bpy
from bpy.props import FloatProperty, EnumProperty
from mathutils import Vector
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode
from sverchok.utils.nodes_mixins.recursive_nodes import SvRecursiveNode

class SvScaleFromCenterNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    """Scale vertices from their centroid"""
    bl_idname = 'SvScaleFromCenterNode'
    bl_label = 'Scale From Center'
    sv_category = 'Transforms'

    scale_factor: FloatProperty(
        name="Scale", default=1.0, update=updateNode
    )

    def sv_init(self, context):
        v = self.inputs.new('SvVerticesSocket', "Vertices")
        v.is_mandatory = True
        v.nesting_level = 3
        v.default_mode = 'NONE'

        s = self.inputs.new('SvStringsSocket', "Factor")
        s.nesting_level = 2
        s.default_mode = 'EMPTY_LIST'
        s.pre_processing = 'ONE_ITEM'

        self.outputs.new('SvVerticesSocket', "Vertices")
        self.outputs.new('SvVerticesSocket', "Centers")

    def process_data(self, params):
        verts, factor = params
        if not factor:
            factor = self.scale_factor

        # Calculate centroid
        n = len(verts)
        if n == 0:
            return [], [(0, 0, 0)]
        cx = sum(v[0] for v in verts) / n
        cy = sum(v[1] for v in verts) / n
        cz = sum(v[2] for v in verts) / n

        # Scale from centroid
        scaled = []
        for v in verts:
            sv = (
                cx + (v[0] - cx) * factor,
                cy + (v[1] - cy) * factor,
                cz + (v[2] - cz) * factor,
            )
            scaled.append(sv)

        return scaled, [(cx, cy, cz)]

    def draw_buttons(self, context, layout):
        layout.prop(self, 'scale_factor')

    def draw_buttons_ext(self, context, layout):
        layout.prop(self, 'list_match')

    def rclick_menu(self, context, layout):
        layout.prop_menu_enum(self, "list_match", text="List Match")
```
