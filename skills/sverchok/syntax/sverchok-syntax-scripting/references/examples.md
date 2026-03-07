# sverchok-syntax-scripting — Working Examples

> Sverchok v1.4.0 · Blender 4.0+/5.x

## SNLite Examples

### Example 1: Flower Petal Generator

```python
"""
in   n_petals s        default=8       nested=2
in   vp_petal s        default=10      nested=2
in   profile_radius s  default=2.3     nested=2
in   amp s             default=1.0     nested=2
out  verts v
out  edges s
"""

import numpy as np
from math import pi

TAU = 2 * pi
N = int(n_petals * vp_petal)

pi_vals = np.tile(np.linspace(0, TAU, int(vp_petal), endpoint=False), int(n_petals))
amps = np.cos(pi_vals) * amp
theta = np.linspace(0, TAU, N, endpoint=False)

circle_coords = np.array([np.sin(theta), np.cos(theta), np.zeros(N)])
coords = circle_coords.T * (profile_radius + amps.reshape((-1, 1)))

verts = [coords.tolist()]
edges = [[(i, (i + 1) % N) for i in range(N)]]
```

### Example 2: KDTree Neighbor Search

```python
"""
in  verts  v
in  radius s  default=0.3  nested=2
out edges  s
"""
import mathutils

size = len(verts)
kd = mathutils.kdtree.KDTree(size)
for i, xyz in enumerate(verts):
    kd.insert(xyz, i)
kd.balance()

edges = [[]]
edge_set = set()
r = radius
for idx, vtx in enumerate(verts):
    n_list = kd.find_range(vtx, r)
    for co, index, dist in n_list:
        if index == idx:
            continue
        edge_set.add(tuple(sorted([idx, index])))

edges[0] = list(edge_set)
```

### Example 3: Persistent Counter

```python
"""
in  trigger s  default=0
out count   s
"""
storage = get_user_dict()
if 'counter' not in storage:
    storage['counter'] = 0
storage['counter'] += 1
count = [[storage['counter']]]
```

### Example 4: Custom Enum Selection

```python
"""
enum operation = Add Subtract Multiply Divide
in  a s  default=1.0
in  b s  default=1.0
out result s
"""
if operation == "Add":
    result = [[a + b]]
elif operation == "Subtract":
    result = [[a - b]]
elif operation == "Multiply":
    result = [[a * b]]
elif operation == "Divide":
    result = [[a / b if b != 0 else 0]]
```

### Example 5: Setup Function with One-Time Init

```python
"""
in  index s  default=0  nested=2
out color v
"""

def setup():
    import colorsys
    palette = []
    for i in range(12):
        h = i / 12.0
        r, g, b = colorsys.hsv_to_rgb(h, 0.8, 0.9)
        palette.append((r, g, b))
    return locals()

# palette is available here from setup()
idx = int(index) % len(palette)
color = [[palette[idx]]]
```

### Example 6: BMesh Operations

```python
"""
in  verts v
in  faces s
out verts_out v
out faces_out s
"""
bm = bmesh_from_pydata(verts, [], faces)

# Subdivide all faces
import bmesh
bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=1)

v_out, e_out, f_out = pydata_from_bmesh(bm)
bm.free()

verts_out = [v_out]
faces_out = [f_out]
```

### Example 7: Required Inputs

```python
"""
in  verts  v  .  required=True
in  scale  s  default=1.0
out result v
"""
# Node will NOT process if verts socket is unconnected
result = [[(v[0]*scale, v[1]*scale, v[2]*scale) for v in verts]]
```

---

## SN Functor B Examples

### Example 1: Circle Generator with UI

```python
import bpy
from sverchok.data_structure import updateNode
from math import sin, cos, pi

def functor_init(self, context):
    self.inputs.new('SvStringsSocket', 'radius')
    self.inputs.new('SvStringsSocket', 'segments')
    self.outputs.new('SvVerticesSocket', 'verts')
    self.outputs.new('SvStringsSocket', 'edges')

def process(self):
    if not self.inputs['radius'].is_linked:
        return

    radius_data = self.inputs['radius'].sv_get()
    segments_data = self.inputs['segments'].sv_get(default=[[24]])

    all_verts = []
    all_edges = []

    for radius_list, seg_list in zip(radius_data, segments_data):
        for radius, segments in zip(radius_list, seg_list):
            segments = int(segments)
            verts = []
            edges = []
            for i in range(segments):
                angle = 2 * pi * i / segments
                verts.append((radius * cos(angle), radius * sin(angle), 0))
                edges.append((i, (i + 1) % segments))
            all_verts.append(verts)
            all_edges.append(edges)

    self.outputs['verts'].sv_set(all_verts)
    self.outputs['edges'].sv_set(all_edges)

def draw_buttons(self, context, layout):
    layout.label(text="Circle Generator")
```

### Example 2: Using Pre-defined Properties

```python
from sverchok.data_structure import updateNode

def functor_init(self, context):
    self.inputs.new('SvVerticesSocket', 'verts')
    self.outputs.new('SvVerticesSocket', 'verts_out')
    self.float_00 = 1.0   # Scale X
    self.float_01 = 1.0   # Scale Y
    self.float_02 = 1.0   # Scale Z
    self.bool_00 = False   # Uniform scale

def process(self):
    if not self.inputs['verts'].is_linked:
        return
    verts = self.inputs['verts'].sv_get()

    if self.bool_00:
        sx = sy = sz = self.float_00
    else:
        sx, sy, sz = self.float_00, self.float_01, self.float_02

    out = [[(v[0]*sx, v[1]*sy, v[2]*sz) for v in obj] for obj in verts]
    self.outputs['verts_out'].sv_set(out)

def draw_buttons(self, context, layout):
    layout.prop(self, 'bool_00', text='Uniform')
    layout.prop(self, 'float_00', text='Scale X')
    if not self.bool_00:
        layout.prop(self, 'float_01', text='Scale Y')
        layout.prop(self, 'float_02', text='Scale Z')
```

---

## Formula Mk5 Examples

### Example 1: Helix Coordinates (3 outputs)

```
output_dimensions = 3

Formula 1: radius * cos(t)
Formula 2: radius * sin(t)
Formula 3: pitch * t / (2 * pi)
```

Connect a Number Range node to `t`, set `radius` and `pitch` as additional inputs.

### Example 2: Parametric Surface

```
output_dimensions = 3

Formula 1: (2 + cos(v)) * cos(u)
Formula 2: (2 + cos(v)) * sin(u)
Formula 3: sin(v)
```

Feed `u` and `v` from two Number Range nodes (0 to 2*pi).

### Example 3: Conditional Expression

```
Formula 1: max(0, sin(x) * amplitude)
```

### Example 4: Vector Output

```
output_type = Vertices
Formula 1: Vector((x, sin(x) * a, 0))
```

---

## Profile Mk3 Examples

### Example 1: I-Beam (Structural)

```
default width = 0.2
default height = 0.4
default flange_thickness = 0.02
default web_thickness = 0.01

let hw = {width / 2}
let hh = {height / 2}
let ft = {flange_thickness}
let wt = {web_thickness / 2}

M -{hw} -{hh}
L {hw} -{hh}
L {hw} {-hh + ft}
L {wt} {-hh + ft}
L {wt} {hh - ft}
L {hw} {hh - ft}
L {hw} {hh}
L -{hw} {hh}
L -{hw} {hh - ft}
L -{wt} {hh - ft}
L -{wt} {-hh + ft}
L -{hw} {-hh + ft}
X
```

### Example 2: Rounded Rectangle

```
default w = 1.0
default h = 0.6
default r = 0.1

let hw = {w / 2}
let hh = {h / 2}

M {hw - r} -{hh}
L {hw} -{hh + r}
Q {hw} {-hh} {hw - r} {-hh} n=8
L {hw} {hh - r}
Q {hw} {hh} {hw - r} {hh} n=8
L -{hw + r} {hh}
Q -{hw} {hh} -{hw} {hh - r} n=8
L -{hw} {-hh + r}
Q -{hw} {-hh} {-hw + r} {-hh} n=8
X
```

### Example 3: Simple L-Shape

```
default a = 1.0
default b = 0.5
default t = 0.1

M 0 0
L {a} 0
L {a} {t}
L {t} {t}
L {t} {b}
L 0 {b}
X
```

### Example 4: Using Arc Command

```
default radius = 0.5
default width = 2.0

M 0 0
H {width} ;
A {radius} {radius} 0 0 1 {width} {radius * 2} n=16
H 0 ;
X
```

### Example 5: NURBS Interpolation Curve

```
default h = 1.0
default w = 0.5

M 0 0
@I 3  0 {h*0.3}  {w*0.5} {h*0.6}  {w} {h} n=20 ;
```
