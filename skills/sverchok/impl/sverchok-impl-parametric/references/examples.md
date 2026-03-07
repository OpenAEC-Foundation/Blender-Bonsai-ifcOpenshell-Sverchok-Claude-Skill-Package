# sverchok-impl-parametric: Working Code Examples

All examples based on Sverchok v1.4.0+ with Blender 4.0+/5.x.
Sources: https://github.com/nortikin/sverchok, vooronderzoek-sverchok.md §6, §14

---

## Example 1: Complete Structural Column Grid (Node Tree)

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups.new("StructuralGrid", 'SverchCustomTreeType')

with tree.init_tree():
    # X grid spacing: 0, 6, 12, 18, 24
    rx = tree.nodes.new('SvGenNumberRange')
    rx.location = (0, 200)
    rx.mode = 'RANGE_STEP'

    # Y grid spacing: 0, 6, 12
    ry = tree.nodes.new('SvGenNumberRange')
    ry.location = (0, 0)
    ry.mode = 'RANGE_STEP'

    # SNLite: cross-product grid + column geometry
    script = tree.nodes.new('SvScriptNodeLite')
    script.location = (250, 100)

    # Viewer for preview
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (500, 100)

    # Connect ranges to script
    tree.links.new(rx.outputs[0], script.inputs[0])
    tree.links.new(ry.outputs[0], script.inputs[1])

    # Connect script outputs to viewer
    tree.links.new(script.outputs[0], viewer.inputs['vertices'])
    tree.links.new(script.outputs[1], viewer.inputs['faces'])

tree.force_update()
```

SNLite script for the script node:
```python
"""
in  xs  s  default=[]
in  ys  s  default=[]
out verts v
out faces s
"""
import math

radius = 0.15
height = 3.5
seg = 12
angles = [2 * math.pi * i / seg for i in range(seg)]
cx = [radius * math.cos(a) for a in angles]
cy = [radius * math.sin(a) for a in angles]

col_v = []
for z in [0.0, height]:
    for i in range(seg):
        col_v.append((cx[i], cy[i], z))

col_f = []
for i in range(seg):
    j = (i + 1) % seg
    col_f.append((i, j, j + seg, i + seg))

all_v, all_f = [], []
for x in xs:
    for y in ys:
        off = len(all_v)
        for v in col_v:
            all_v.append((v[0] + x, v[1] + y, v[2]))
        for f in col_f:
            all_f.append(tuple(i + off for i in f))

verts = [all_v]
faces = [all_f]
```

---

## Example 2: Multi-Story Floor Plates

```python
"""
in  width    s  default=24.0
in  depth    s  default=12.0
in  floors   s  default=5
in  floor_h  s  default=3.5
in  slab_t   s  default=0.25
out verts    v
out faces    s
"""
n = int(floors)
w, d = width, depth
t = slab_t

all_v = []
all_f = []

for fl in range(n):
    z = fl * floor_h
    idx = len(all_v)
    # Bottom of slab
    all_v.extend([
        (0, 0, z), (w, 0, z), (w, d, z), (0, d, z)
    ])
    # Top of slab
    all_v.extend([
        (0, 0, z + t), (w, 0, z + t), (w, d, z + t), (0, d, z + t)
    ])
    # Bottom face
    all_f.append((idx, idx+1, idx+2, idx+3))
    # Top face
    all_f.append((idx+4, idx+7, idx+6, idx+5))
    # Side faces
    for i in range(4):
        j = (i + 1) % 4
        all_f.append((idx+i, idx+j, idx+j+4, idx+i+4))

verts = [all_v]
faces = [all_f]
```

---

## Example 3: Curtain Wall Mullion Grid

```python
"""
in  wall_w   s  default=24.0
in  wall_h   s  default=12.0
in  div_x    s  default=8
in  div_y    s  default=4
in  mullion_w s  default=0.05
in  mullion_d s  default=0.08
out verts    v
out faces    s
"""
nx, ny = int(div_x), int(div_y)
mw = mullion_w / 2
md = mullion_d

all_v = []
all_f = []

# Vertical mullions
for i in range(nx + 1):
    x = i * wall_w / nx
    idx = len(all_v)
    all_v.extend([
        (x - mw, 0, 0), (x + mw, 0, 0),
        (x + mw, 0, wall_h), (x - mw, 0, wall_h),
        (x - mw, -md, 0), (x + mw, -md, 0),
        (x + mw, -md, wall_h), (x - mw, -md, wall_h),
    ])
    # Front face
    all_f.append((idx, idx+1, idx+2, idx+3))
    # Back face
    all_f.append((idx+4, idx+7, idx+6, idx+5))
    # Side faces
    all_f.append((idx, idx+3, idx+7, idx+4))
    all_f.append((idx+1, idx+5, idx+6, idx+2))

# Horizontal mullions
for j in range(ny + 1):
    z = j * wall_h / ny
    idx = len(all_v)
    all_v.extend([
        (0, 0, z - mw), (wall_w, 0, z - mw),
        (wall_w, 0, z + mw), (0, 0, z + mw),
        (0, -md, z - mw), (wall_w, -md, z - mw),
        (wall_w, -md, z + mw), (0, -md, z + mw),
    ])
    all_f.append((idx, idx+1, idx+2, idx+3))
    all_f.append((idx+4, idx+7, idx+6, idx+5))
    all_f.append((idx, idx+3, idx+7, idx+4))
    all_f.append((idx+1, idx+5, idx+6, idx+2))

verts = [all_v]
faces = [all_f]
```

---

## Example 4: Parametric Louver Array

```python
"""
in  width     s  default=1.2
in  height    s  default=0.08
in  depth     s  default=0.15
in  count     s  default=12
in  spacing   s  default=0.12
in  angle_deg s  default=35.0
out verts     v
out faces     s
"""
import math
from mathutils import Matrix, Vector

n = int(count)
angle = math.radians(angle_deg)
rot = Matrix.Rotation(angle, 4, 'X')

# Single louver blade (flat panel)
blade_v = [
    (-width/2, 0, -height/2), (width/2, 0, -height/2),
    (width/2, 0, height/2), (-width/2, 0, height/2),
    (-width/2, -depth, -height/2), (width/2, -depth, -height/2),
    (width/2, -depth, height/2), (-width/2, -depth, height/2),
]

blade_f = [
    (0,1,2,3), (4,7,6,5), (0,4,5,1), (1,5,6,2), (2,6,7,3), (3,7,4,0)
]

all_v = []
all_f = []
for i in range(n):
    z_offset = i * spacing
    loc = Matrix.Translation(Vector((0, 0, z_offset)))
    transform = loc @ rot
    idx = len(all_v)
    for v in blade_v:
        tv = transform @ Vector(v)
        all_v.append(tuple(tv))
    for f in blade_f:
        all_f.append(tuple(fi + idx for fi in f))

verts = [all_v]
faces = [all_f]
```

---

## Example 5: Spiral Staircase

```python
"""
in  inner_r   s  default=0.5
in  outer_r   s  default=1.5
in  riser_h   s  default=0.17
in  num_steps s  default=20
in  total_rot s  default=360.0
out verts     v
out faces     s
"""
import math

n = int(num_steps)
angle_step = math.radians(total_rot) / n

all_v = []
all_f = []

for i in range(n):
    a0 = i * angle_step
    a1 = (i + 1) * angle_step
    z = i * riser_h

    idx = len(all_v)
    # Four corners of the tread (pie-slice shape)
    all_v.extend([
        (inner_r * math.cos(a0), inner_r * math.sin(a0), z + riser_h),
        (outer_r * math.cos(a0), outer_r * math.sin(a0), z + riser_h),
        (outer_r * math.cos(a1), outer_r * math.sin(a1), z + riser_h),
        (inner_r * math.cos(a1), inner_r * math.sin(a1), z + riser_h),
    ])
    # Tread top face
    all_f.append((idx, idx+1, idx+2, idx+3))

    # Riser face (vertical front)
    idx2 = len(all_v)
    all_v.extend([
        (inner_r * math.cos(a0), inner_r * math.sin(a0), z),
        (outer_r * math.cos(a0), outer_r * math.sin(a0), z),
        (outer_r * math.cos(a0), outer_r * math.sin(a0), z + riser_h),
        (inner_r * math.cos(a0), inner_r * math.sin(a0), z + riser_h),
    ])
    all_f.append((idx2, idx2+1, idx2+2, idx2+3))

verts = [all_v]
faces = [all_f]
```

---

## Example 6: Gable Roof from Footprint

```python
"""
in  width   s  default=12.0
in  depth   s  default=8.0
in  eave_h  s  default=6.0
in  ridge_h s  default=9.0
out verts   v
out faces   s
"""
w, d = width, depth
eh, rh = eave_h, ridge_h
hw = w / 2

all_v = [
    # Eave corners (base of roof)
    (0,   0, eh),   # 0: front-left
    (w,   0, eh),   # 1: front-right
    (w,   d, eh),   # 2: back-right
    (0,   d, eh),   # 3: back-left
    # Ridge line
    (hw,  0, rh),   # 4: front-ridge
    (hw,  d, rh),   # 5: back-ridge
]

all_f = [
    (0, 1, 4),      # Front gable triangle
    (2, 3, 5),      # Back gable triangle
    (0, 4, 5, 3),   # Left roof slope
    (1, 2, 5, 4),   # Right roof slope
]

verts = [all_v]
faces = [all_f]
```

---

## Example 7: Hip Roof from Rectangular Footprint

```python
"""
in  width   s  default=12.0
in  depth   s  default=8.0
in  eave_h  s  default=6.0
in  ridge_h s  default=9.0
out verts   v
out faces   s
"""
w, d = width, depth
eh, rh = eave_h, ridge_h

# Ridge inset (hip offset from shorter side)
inset = min(w, d) / 2

all_v = [
    # Eave corners
    (0, 0, eh),      # 0
    (w, 0, eh),      # 1
    (w, d, eh),      # 2
    (0, d, eh),      # 3
    # Ridge endpoints (inset from gable ends)
    (inset, d/2 if w >= d else inset, rh),           # 4
    (w - inset, d/2 if w >= d else d - inset, rh),   # 5
]

if w >= d:
    # Wider than deep: ridge runs along X
    all_v[4] = (inset, d / 2, rh)
    all_v[5] = (w - inset, d / 2, rh)
    all_f = [
        (0, 1, 5, 4),   # Front slope
        (2, 3, 4, 5),   # Back slope
        (0, 4, 3),      # Left hip triangle
        (1, 2, 5),      # Right hip triangle
    ]
else:
    # Deeper than wide: ridge runs along Y
    all_v[4] = (w / 2, inset, rh)
    all_v[5] = (w / 2, d - inset, rh)
    all_f = [
        (0, 1, 4),      # Front hip triangle
        (2, 3, 5),      # Back hip triangle
        (0, 4, 5, 3),   # Left slope
        (1, 2, 5, 4),   # Right slope
    ]

verts = [all_v]
faces = [all_f]
```

---

## Example 8: Pipe Routing from Coordinate List

```python
"""
in  coords   v
in  radius   s  default=0.05
in  segments s  default=12
out verts    v
out faces    s
"""
import math

n_seg = int(segments)
r = radius
path = coords  # list of (x, y, z) waypoints

if len(path) < 2:
    verts = [[]]
    faces = [[]]
else:
    all_v = []
    all_f = []

    # Generate circle profile at each path point
    angles = [2 * math.pi * i / n_seg for i in range(n_seg)]

    for pi, pt in enumerate(path):
        # Simple circle in XY plane, translated to path point
        for ai, a in enumerate(angles):
            all_v.append((
                pt[0] + r * math.cos(a),
                pt[1] + r * math.sin(a),
                pt[2]
            ))

    # Connect consecutive circles with quad faces
    for pi in range(len(path) - 1):
        base = pi * n_seg
        next_base = (pi + 1) * n_seg
        for ai in range(n_seg):
            aj = (ai + 1) % n_seg
            all_f.append((
                base + ai, base + aj,
                next_base + aj, next_base + ai
            ))

    verts = [all_v]
    faces = [all_f]
```

---

## Example 9: Terrain from CSV Elevation Data

```python
"""
in  xs     s  default=[]
in  ys     s  default=[]
in  zs     s  default=[]
out verts  v
out faces  s
"""
import numpy as np

x_arr = np.array(xs, dtype=float)
y_arr = np.array(ys, dtype=float)
z_arr = np.array(zs, dtype=float)

n = len(x_arr)
if n < 3:
    verts = [[]]
    faces = [[]]
else:
    # Create vertices
    v_list = [(x_arr[i], y_arr[i], z_arr[i]) for i in range(n)]

    # Simple grid triangulation (assumes regular grid)
    # Detect grid dimensions from unique x/y values
    ux = np.unique(x_arr)
    uy = np.unique(y_arr)
    nx_grid = len(ux)
    ny_grid = len(uy)

    f_list = []
    if nx_grid * ny_grid == n:
        for iy in range(ny_grid - 1):
            for ix in range(nx_grid - 1):
                i00 = iy * nx_grid + ix
                i10 = i00 + 1
                i01 = i00 + nx_grid
                i11 = i01 + 1
                f_list.append((i00, i10, i11, i01))

    verts = [v_list]
    faces = [f_list]
```

---

## Example 10: Data-Driven Facade from JSON

```python
"""
in  json_str  s  default='{}'
out verts     v
out faces     s
"""
import json

data = json.loads(json_str) if isinstance(json_str, str) else {}
panels = data.get("panels", [])

all_v = []
all_f = []

for p in panels:
    x = p.get("x", 0)
    z = p.get("z", 0)
    w = p.get("width", 1.0)
    h = p.get("height", 1.0)
    y_off = p.get("depth", 0.0)

    idx = len(all_v)
    all_v.extend([
        (x, y_off, z), (x + w, y_off, z),
        (x + w, y_off, z + h), (x, y_off, z + h)
    ])
    all_f.append((idx, idx+1, idx+2, idx+3))

verts = [all_v]
faces = [all_f]
```

Expected JSON input format:
```json
{
  "panels": [
    {"x": 0, "z": 0, "width": 1.5, "height": 2.8, "depth": 0},
    {"x": 1.6, "z": 0, "width": 1.5, "height": 2.8, "depth": 0},
    {"x": 0, "z": 3.0, "width": 1.5, "height": 2.8, "depth": 0}
  ]
}
```

---

## Example 11: Batch Generation with Blender Object Output

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def generate_column_grid_objects(span_x, span_y, nx, ny, radius, height):
    """Generate column grid as Blender mesh objects.
    Parameters:
        span_x: float — X spacing between columns
        span_y: float — Y spacing between columns
        nx: int — number of columns in X
        ny: int — number of columns in Y
        radius: float — column radius
        height: float — column height
    """
    import math
    seg = 16

    # Create column template mesh
    angles = [2 * math.pi * i / seg for i in range(seg)]
    template_v = []
    for z in [0.0, height]:
        for a in angles:
            template_v.append((radius * math.cos(a), radius * math.sin(a), z))

    template_f = []
    for i in range(seg):
        j = (i + 1) % seg
        template_f.append((i, j, j + seg, i + seg))
    template_f.append(list(range(seg)))
    template_f.append(list(range(seg, 2 * seg)))

    # Create collection for columns
    col_name = "ColumnGrid"
    if col_name not in bpy.data.collections:
        col = bpy.data.collections.new(col_name)
        bpy.context.scene.collection.children.link(col)
    else:
        col = bpy.data.collections[col_name]

    # Generate column objects
    for ix in range(nx):
        for iy in range(ny):
            name = f"Column_{ix}_{iy}"
            mesh = bpy.data.meshes.new(name)
            ox, oy = ix * span_x, iy * span_y
            verts = [(v[0] + ox, v[1] + oy, v[2]) for v in template_v]
            mesh.from_pydata(verts, [], template_f)
            mesh.update()

            if name in bpy.data.objects:
                bpy.data.objects[name].data = mesh
            else:
                obj = bpy.data.objects.new(name, mesh)
                col.objects.link(obj)

# Usage:
# generate_column_grid_objects(6.0, 6.0, 5, 4, 0.15, 3.5)
```

---

## Example 12: NumPy-Accelerated Floor Grid

```python
"""
in  grid_nx  s  default=50  nested=2
in  grid_ny  s  default=50  nested=2
in  cell_size s default=1.0
out verts    v
out faces    s
"""
import numpy as np

nx, ny = int(grid_nx), int(grid_ny)
s = cell_size

# Generate vertices using NumPy (fast for large grids)
x = np.linspace(0, nx * s, nx + 1)
y = np.linspace(0, ny * s, ny + 1)
gx, gy = np.meshgrid(x, y)
gz = np.zeros_like(gx)

v = np.column_stack([gx.ravel(), gy.ravel(), gz.ravel()])

# Generate quad faces using NumPy indexing
row = np.arange(ny)
col = np.arange(nx)
r, c = np.meshgrid(row, col, indexing='ij')
i00 = r * (nx + 1) + c
i10 = i00 + 1
i01 = i00 + (nx + 1)
i11 = i01 + 1
f = np.column_stack([i00.ravel(), i10.ravel(), i11.ravel(), i01.ravel()])

verts = [v.tolist()]
faces = [f.tolist()]
```
