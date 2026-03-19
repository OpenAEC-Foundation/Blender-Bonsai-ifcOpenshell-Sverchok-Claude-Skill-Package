---
name: sverchok-impl-parametric
description: >
  Use when designing parametric AEC geometry in Sverchok -- structural grids, facade panels,
  parametric stairs, roof geometry, MEP routing, or terrain from data. Prevents the common
  mistake of not using matrix transforms for element placement (using vertex offsets instead,
  which breaks with rotation). Covers matrix-based transforms, Blender object integration,
  and data-driven geometry generation.
  Keywords: parametric design, structural grid, facade panel, stairs, roof, terrain,
  matrix transform, data-driven, AEC geometry, Sverchok parametric.
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-impl-parametric

## Quick Reference

### What This Skill Covers

Parametric AEC (Architecture, Engineering, Construction) design workflows using Sverchok's node-based system. Build structural grids, facade systems, stairs, roofs, MEP routing, and terrain from data — all as reusable parametric node trees.

- **Structural grids**: column grids, beam layouts, floor plates from spacing parameters
- **Facade systems**: panel division, curtain wall mullions, louver arrays
- **Parametric stairs**: straight, L-shaped, spiral with tread/riser profiles
- **Roof geometry**: hip, gable, shed from footprint polygons
- **MEP routing**: pipe and duct paths using spline-based nodes
- **Terrain**: contour/point cloud data to mesh surfaces
- **Supporting techniques**: matrices, Blender object integration, CSV/JSON data input, NumPy acceleration

### Critical Warnings

**NEVER** generate geometry without wrapping output in the Sverchok nesting convention — ALL socket data requires `[[object_data]]` wrapping. Flat vertex lists corrupt downstream nodes.

**NEVER** build parametric node trees without `tree.init_tree()` context manager — every node/link addition without it triggers O(n²) updates.

**NEVER** hardcode absolute coordinates for AEC elements — ALWAYS use matrix transforms so elements respond to parameter changes. Hardcoded positions break parametric intent.

**NEVER** use `deepcopy=False` on `sv_get()` when mutating vertex data for transforms — this corrupts upstream cached data shared across all downstream nodes.

**ALWAYS** use `match_long_repeat` when combining inputs of different lengths (e.g., grid positions × profile shapes) — without it, shorter lists silently truncate.

**ALWAYS** check `output.is_linked` before computing geometry — AEC node trees with many outputs waste significant time computing unused branches.

### Decision Tree

```
What AEC element do you need?
├── Structural grid (columns, beams, floors)
│   ├── Regular grid → Number Range + List Cross → Matrix Apply on profile
│   ├── Irregular grid → CSV In with column positions → Matrix Apply
│   └── Multi-story → Add Z-offset per floor via Matrix Multiply
│
├── Facade panels / curtain wall
│   ├── Flat panels → Subdivide face → Inset → Extrude per cell
│   ├── Louvers → Line + Array via Matrix → rotate per element
│   └── Data-driven → CSV/JSON In → Map Range for panel parameters
│
├── Stairs
│   ├── Straight → Number Range for treads → Box + Matrix offset
│   ├── L-shape → Two straight runs + landing platform
│   └── Spiral → Trigonometric SNLite + Matrix rotation per step
│
├── Roof from footprint
│   ├── Simple gable → Extrude edge + move ridge vertices up
│   ├── Hip roof → Straight Skeleton or inset + raise center
│   └── Shed/mono-pitch → Offset one edge upward
│
├── MEP routing
│   ├── Simple pipe run → Polyline → Pipe node (circle profile + sweep)
│   ├── Duct with bends → Bezier/NURBS spline → Rectangular profile sweep
│   └── From coordinates → CSV In → Vector In → Polyline Viewer
│
└── Terrain from data
    ├── Point cloud → Delaunay 2D triangulation
    ├── Contour lines → Interpolate Z → Surface from curves
    └── Grid DEM → CSV In → Plane grid → set Z from data
```

---

## Essential Patterns

### Pattern 1: Structural Column Grid

Generate a parametric column grid from X/Y spacing and column profile.

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
from mathutils import Matrix, Vector

tree = bpy.data.node_groups.new("ColumnGrid", 'SverchCustomTreeType')

with tree.init_tree():
    # X positions
    range_x = tree.nodes.new('SvGenNumberRange')
    range_x.location = (0, 200)
    range_x.mode = 'RANGE_COUNT'  # start, stop, count

    # Y positions
    range_y = tree.nodes.new('SvGenNumberRange')
    range_y.location = (0, 0)
    range_y.mode = 'RANGE_COUNT'

    # Cross-product for grid positions
    cross = tree.nodes.new('SvListInputNode')  # Use CrossSection or SNLite
    cross.location = (200, 100)

    # Column profile (cylinder)
    cyl = tree.nodes.new('SvCylinderNodeMK2')
    cyl.location = (200, -100)

    # Viewer
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (600, 0)

tree.force_update()
```

### Pattern 2: SNLite Column Grid (Complete)

Use SNLite for full control over a parametric column grid.

```python
"""
in  span_x  s  default=6.0
in  span_y  s  default=6.0
in  count_x s  default=4
in  count_y s  default=3
in  col_radius s default=0.15
in  col_height s default=3.0
out verts   v
out faces   s
"""
import numpy as np
from math import pi, cos, sin

nx, ny = int(count_x), int(count_y)
r, h = col_radius, col_height
seg = 12  # circle segments

# Generate unit column (cylinder)
angles = np.linspace(0, 2 * pi, seg, endpoint=False)
cx = r * np.cos(angles)
cy = r * np.sin(angles)

col_v = []
for z in [0.0, h]:
    for i in range(seg):
        col_v.append((cx[i], cy[i], z))

col_f = []
for i in range(seg):
    j = (i + 1) % seg
    col_f.append((i, j, j + seg, i + seg))
col_f.append(list(range(seg)))
col_f.append(list(range(seg, 2 * seg)))

# Place columns at grid intersections
all_verts = []
all_faces = []
for ix in range(nx):
    for iy in range(ny):
        ox = ix * span_x
        oy = iy * span_y
        offset = len(all_verts)
        for v in col_v:
            all_verts.append((v[0] + ox, v[1] + oy, v[2]))
        for f in col_f:
            all_faces.append(tuple(fi + offset for fi in f))

verts = [all_verts]
faces = [all_faces]
```

### Pattern 3: Matrix-Based Element Placement

Use Sverchok matrix nodes to place AEC elements at grid positions.

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
from mathutils import Matrix, Vector
import math

def create_placement_matrices(positions, rotation_z=0.0):
    """Create 4x4 transform matrices for element placement.
    Parameters:
        positions: list of (x, y, z) tuples
        rotation_z: rotation in radians around Z axis
    Returns:
        list of Matrix — one per position, Sverchok-nested [[mat1, mat2, ...]]
    """
    rot = Matrix.Rotation(rotation_z, 4, 'Z')
    matrices = []
    for pos in positions:
        loc = Matrix.Translation(Vector(pos))
        matrices.append(loc @ rot)
    return [matrices]  # Sverchok nesting: [[matrices]]
```

### Pattern 4: Facade Panel Division

Subdivide a wall face into parametric panels.

```python
"""
in  wall_w   s  default=12.0
in  wall_h   s  default=3.0
in  panels_x s  default=6
in  panels_y s  default=2
in  gap      s  default=0.02
out verts    v
out faces    s
"""
nx, ny = int(panels_x), int(panels_y)
pw = wall_w / nx
ph = wall_h / ny
g = gap

all_v = []
all_f = []
for ix in range(nx):
    for iy in range(ny):
        x0 = ix * pw + g
        x1 = (ix + 1) * pw - g
        z0 = iy * ph + g
        z1 = (iy + 1) * ph - g
        idx = len(all_v)
        all_v.extend([(x0, 0, z0), (x1, 0, z0), (x1, 0, z1), (x0, 0, z1)])
        all_f.append((idx, idx+1, idx+2, idx+3))

verts = [all_v]
faces = [all_f]
```

### Pattern 5: Parametric Staircase

Generate a straight staircase from riser height and tread depth.

```python
"""
in  width      s  default=1.2
in  riser_h    s  default=0.17
in  tread_d    s  default=0.28
in  num_steps  s  default=16
out verts      v
out faces      s
"""
n = int(num_steps)
w = width
rh = riser_h
td = tread_d

all_v = []
all_f = []
for i in range(n):
    x0 = i * td
    x1 = x0 + td
    z0 = i * rh
    z1 = z0 + rh
    idx = len(all_v)
    # Riser face (vertical)
    all_v.extend([(x0, 0, z0), (x0, w, z0), (x0, w, z1), (x0, 0, z1)])
    all_f.append((idx, idx+1, idx+2, idx+3))
    # Tread face (horizontal)
    idx2 = len(all_v)
    all_v.extend([(x0, 0, z1), (x0, w, z1), (x1, w, z1), (x1, 0, z1)])
    all_f.append((idx2, idx2+1, idx2+2, idx2+3))

verts = [all_v]
faces = [all_f]
```

### Pattern 6: Data-Driven Generation from CSV

Read element positions from CSV and generate geometry.

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Node tree approach: CSV In → process data → Matrix Apply → Profile
import bpy

tree = bpy.data.node_groups.new("DataDrivenGrid", 'SverchCustomTreeType')

with tree.init_tree():
    # CSV In node reads external spreadsheet
    csv_node = tree.nodes.new('SvCSVInNode')
    csv_node.location = (0, 0)
    # Set csv_node.csv_file to your file path after creation

    # SNLite processes CSV rows into matrices
    script = tree.nodes.new('SvScriptNodeLite')
    script.location = (200, 0)

    # Profile to be placed at each position
    profile = tree.nodes.new('SvBoxNodeMk2')
    profile.location = (200, -200)

    # Matrix Apply combines position + geometry
    mat_apply = tree.nodes.new('SvMatrixApplyJoinNode')
    mat_apply.location = (400, 0)

    # Viewer output
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (600, 0)

tree.force_update()
```

### Pattern 7: Terrain from Point Data

Generate terrain mesh from elevation data.

```python
"""
in  points  v
out verts   v
out faces   s
"""
# Delaunay 2D triangulation for terrain surface
# Input: list of (x, y, z) points from CSV or point cloud
# Use Sverchok's Delaunay 2D node for triangulation

from mathutils.geometry import delaunay_2d_cdt

pts_2d = [(p[0], p[1]) for p in points]
edges_in = []
faces_in = []

result = delaunay_2d_cdt(pts_2d, edges_in, faces_in, 0, 1e-6)
out_verts_2d, out_edges, out_faces, _, _, _ = result

# Restore Z values from original points
import numpy as np
pts_arr = np.array(points)
out_v = []
for v2d in out_verts_2d:
    # Find nearest original point for Z value
    dists = (pts_arr[:, 0] - v2d[0])**2 + (pts_arr[:, 1] - v2d[1])**2
    nearest = np.argmin(dists)
    out_v.append((v2d[0], v2d[1], pts_arr[nearest, 2]))

verts = [out_v]
faces = [list(out_faces)]
```

---

## Common Operations

### AEC Element Node Recipes

| AEC Element | Key Nodes | Connection Pattern |
|-------------|-----------|-------------------|
| Column grid | Number Range × 2, SNLite, Cylinder, Matrix Apply | Ranges → Cross product → Matrices → Apply to profile |
| Floor plate | Number Range × 2, Plane, Matrix Apply | Ranges → Plane size → Matrix per floor |
| Beam layout | Line, List Repeat, Matrix Apply | Grid lines → Profile sweep → Place |
| Facade panels | Box/Plane, Subdivide, Inset, List processing | Wall → Subdivide → Inset per cell → Gap |
| Curtain wall mullions | Line, Array Modifier, Matrix | Vertical/horizontal lines → Array → Join |
| Louver array | Plane, Matrix Rotation, List Repeat | Panel → Rotate → Array along facade |
| Straight stairs | Box, Number Range, Matrix offset | Tread box → Offset per step via Matrix |
| Spiral stairs | SNLite (trig), Box, Matrix rotation | Angle per step → Tread → Rotate + lift |
| Gable roof | Extrude edge, Move vertices | Footprint top edge → Extrude → Raise ridge |
| Hip roof | Inset Polygon, Move center up | Footprint → Inset → Raise center polygon |
| Pipe routing | Polyline, Circle profile, Sweep | Path points → Polyline → Sweep circle |
| Duct routing | Polyline, Rectangle profile, Sweep | Path points → Polyline → Sweep rectangle |
| Terrain mesh | CSV In, Delaunay 2D or Plane grid | Points → Triangulate or Grid → Set Z |

### Matrix Transform Cheat Sheet

```python
from mathutils import Matrix, Vector, Euler
import math

# Translation only
mat_loc = Matrix.Translation(Vector((x, y, z)))

# Rotation around Z (plan rotation)
mat_rot_z = Matrix.Rotation(math.radians(angle), 4, 'Z')

# Combined: place element at position with rotation
transform = mat_loc @ mat_rot_z

# Multi-story: floor offset
floor_offset = Matrix.Translation(Vector((0, 0, floor_index * floor_height)))
element_transform = floor_offset @ mat_loc @ mat_rot_z

# Scale (non-uniform for panels)
mat_scale = Matrix.Diagonal(Vector((sx, sy, sz, 1.0)))
```

### Data Input Methods

| Source | Sverchok Node | Output | Use Case |
|--------|--------------|--------|----------|
| CSV file | `SvCSVInNode` | Strings/Numbers socket | Column positions, panel sizes, elevation data |
| JSON file | `SvJsonInNode` | Dictionary socket | Complex element definitions, BIM data |
| Blender object | `SvGetObjectsData` | Vertices/Edges/Faces | Site boundary, existing geometry |
| Manual entry | `SvListInputNode` | Strings socket | Small parameter sets |
| Number range | `SvGenNumberRange` | Strings socket | Regular grid spacing |

### NumPy Acceleration for Large AEC Models

```python
"""
in  count s  default=100
in  spacing s default=6.0
out verts v
out faces s
"""
import numpy as np

n = int(count)
s = spacing

# Grid positions via NumPy (fast for large grids)
ix = np.arange(n)
iy = np.arange(n)
gx, gy = np.meshgrid(ix * s, iy * s)
gz = np.zeros_like(gx)

# Flatten to vertex list
v = np.column_stack([gx.ravel(), gy.ravel(), gz.ravel()])

# Generate quad faces for grid
faces_list = []
for i in range(n - 1):
    for j in range(n - 1):
        idx = i * n + j
        faces_list.append((idx, idx + 1, idx + n + 1, idx + n))

verts = [v.tolist()]
faces = [faces_list]
```

### Blender Object Integration

```python
# Write Sverchok output to Blender objects
import bpy, bmesh

def sv_output_to_blender(verts_nested, faces_nested, name_prefix="AEC"):
    """Convert Sverchok nested output to Blender mesh objects.
    Parameters:
        verts_nested: [[verts_obj0], [verts_obj1], ...] — Sverchok format
        faces_nested: [[faces_obj0], [faces_obj1], ...] — Sverchok format
        name_prefix: str — prefix for object names
    """
    for idx, (v_list, f_list) in enumerate(zip(verts_nested, faces_nested)):
        name = f"{name_prefix}_{idx:04d}"
        mesh = bpy.data.meshes.new(name)
        mesh.from_pydata(v_list, [], f_list)
        mesh.update()

        if name in bpy.data.objects:
            bpy.data.objects[name].data = mesh
        else:
            obj = bpy.data.objects.new(name, mesh)
            bpy.context.collection.objects.link(obj)
```

---

## Reference Links

- [references/methods.md](references/methods.md) — Node types, SNLite headers, matrix utilities, and CSV/JSON input patterns for AEC workflows
- [references/examples.md](references/examples.md) — Complete working examples for each AEC element type
- [references/anti-patterns.md](references/anti-patterns.md) — Common mistakes in AEC parametric modeling with Sverchok

### Cross-References

- **sverchok-syntax-scripting** — SNLite node syntax, header format, `vectorize` utility, `setup()` for pre-computation
- **sverchok-core-concepts** — Node tree architecture, update system, socket data cache, data nesting convention
- **blender-core-api** — `bpy.data` access for meshes, objects, collections
- **blender-core-runtime** — `mathutils.Matrix`, `mathutils.Vector`, `mathutils.Euler` for transforms

### Official Sources

- https://github.com/nortikin/sverchok
- https://nortikin.github.io/sverchok/
