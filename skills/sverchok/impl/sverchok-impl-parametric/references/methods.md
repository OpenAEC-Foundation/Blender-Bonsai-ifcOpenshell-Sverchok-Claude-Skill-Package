# sverchok-impl-parametric: Method Reference

Sources: https://github.com/nortikin/sverchok (v1.4.0+),
vooronderzoek-sverchok.md §6, §14

---

## Generator Nodes for AEC Elements

### SvBoxNodeMk2 — Box Generator

```python
node = tree.nodes.new('SvBoxNodeMk2')
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `Size` | `FloatProperty` | `1.0` | Overall box size |
| `Div X` | `IntProperty` | `1` | Subdivisions along X |
| `Div Y` | `IntProperty` | `1` | Subdivisions along Y |
| `Div Z` | `IntProperty` | `1` | Subdivisions along Z |

**Outputs**: `Vertices`, `Edges`, `Faces`

Use for: column bases, floor slabs, wall panels, stair treads.

### SvCylinderNodeMK2 — Cylinder Generator

```python
node = tree.nodes.new('SvCylinderNodeMK2')
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `RadTop` | `FloatProperty` | `1.0` | Top radius |
| `RadBot` | `FloatProperty` | `1.0` | Bottom radius |
| `Vertices` | `IntProperty` | `32` | Circumference segments |
| `Height` | `FloatProperty` | `2.0` | Cylinder height |
| `Caps` | `BoolProperty` | `True` | Generate cap faces |

**Outputs**: `Vertices`, `Edges`, `Faces`

Use for: round columns, pipes, ducts, handrail posts.

### SvPlaneNode — Plane Generator

```python
node = tree.nodes.new('SvPlaneNode')
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `Size X` | `FloatProperty` | `1.0` | Width |
| `Size Y` | `FloatProperty` | `1.0` | Depth |
| `Div X` | `IntProperty` | `1` | X subdivisions |
| `Div Y` | `IntProperty` | `1` | Y subdivisions |

**Outputs**: `Vertices`, `Edges`, `Faces`

Use for: floor plates, wall faces, terrain base grids, facade panels.

### SvLineNodeMK4 — Line Generator

```python
node = tree.nodes.new('SvLineNodeMK4')
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `Num` | `IntProperty` | `2` | Number of vertices |
| `Size` | `FloatProperty` | `1.0` | Line length |

**Outputs**: `Vertices`, `Edges`

Use for: beam centerlines, mullion paths, pipe routing segments.

### SvCircleNode — Circle Generator

```python
node = tree.nodes.new('SvCircleNode')
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `Num Verts` | `IntProperty` | `24` | Number of vertices |
| `Radius` | `FloatProperty` | `1.0` | Circle radius |

**Outputs**: `Vertices`, `Edges`

Use for: pipe cross-sections, column profiles, circular openings.

---

## Number / Range Nodes

### SvGenNumberRange — Number Range

```python
node = tree.nodes.new('SvGenNumberRange')
node.mode = 'RANGE_COUNT'  # Options: RANGE, RANGE_COUNT, RANGE_STEP, COUNT
```

| Mode | Inputs | Description |
|------|--------|-------------|
| `RANGE` | Start, Stop | Integers from start to stop |
| `RANGE_COUNT` | Start, Stop, Count | N evenly spaced values |
| `RANGE_STEP` | Start, Stop, Step | Values at step intervals |
| `COUNT` | Start, Step, Count | N values from start at step |

Use for: grid spacing, floor levels, stair step positions.

### SvMapRangeNode — Map Range

```python
node = tree.nodes.new('SvMapRangeNode')
```

Maps values from one range to another. Use for normalizing CSV data to geometry dimensions.

---

## Transform Nodes

### SvMatrixApplyJoinNode — Matrix Apply

```python
node = tree.nodes.new('SvMatrixApplyJoinNode')
```

**Inputs**: `Vertices`, `Matrices`
**Outputs**: `Vertices`

Applies each matrix to the input geometry, producing one copy per matrix. Primary node for instancing elements at grid positions.

### SvMatrixInNodeMK4 — Matrix In

```python
node = tree.nodes.new('SvMatrixInNodeMK4')
```

**Inputs**: `Location`, `Scale`, `Rotation`, `Angle`
**Outputs**: `Matrix`

Constructs a 4×4 transform matrix from components. Use to build placement matrices from grid coordinate lists.

### SvMatrixMathNode — Matrix Math

```python
node = tree.nodes.new('SvMatrixMathNode')
```

**Inputs**: `A`, `B`
**Operations**: Multiply, Invert

Use for composing transforms (e.g., floor offset × column position).

---

## List Processing Nodes

### List Cross / List Levels

For generating grid positions from X and Y range lists:

```
[Range X] → [List Repeat (wrap)] → combine with
[Range Y] → [List Repeat (wrap)] → [Vector In] → [Matrix In] → [Matrix Apply]
```

### SvListJoinNode — List Join

```python
node = tree.nodes.new('SvListJoinNode')
```

Joins multiple geometry outputs into a single list. Use for combining columns, beams, and slabs into one output.

---

## Exchange Nodes (Data Input)

### SvCSVInNode — CSV Input

```python
node = tree.nodes.new('SvCSVInNode')
# After creation, set the file path:
# node.csv_file = "/path/to/data.csv"
```

Reads CSV files into Sverchok data streams. Each column becomes a separate output.

**Common CSV format for AEC**:
```csv
x,y,z,type,size
0,0,0,column,0.3
6,0,0,column,0.3
12,0,0,column,0.3
```

### SvJsonInNode — JSON Input

```python
node = tree.nodes.new('SvJsonInNode')
```

Reads JSON files into Sverchok dictionary data.

**Common JSON format for AEC**:
```json
{
  "grid": {"span_x": 6.0, "span_y": 6.0, "floors": 5},
  "columns": [{"x": 0, "y": 0}, {"x": 6, "y": 0}]
}
```

---

## Visualization Nodes

### SvViewerDrawMk4 — Viewer Draw

```python
node = tree.nodes.new('SvViewerDrawMk4')
```

**Inputs**: `vertices`, `edges`, `faces`, `matrix`

Draws geometry in the viewport without creating Blender objects. Use for rapid preview during parametric design iteration.

### SvBmeshViewerNodeMK2 — BMesh Viewer

```python
node = tree.nodes.new('SvBmeshViewerNodeMK2')
```

Creates actual Blender mesh objects from Sverchok data. Use when the geometry needs to be exported or used by other Blender features.

---

## SNLite Header Syntax for AEC Scripts

```python
"""
in  param_name  s  default=1.0           # Strings (numbers) socket
in  param_name  s  default=1  nested=2   # Nested number input
in  verts_in    v                          # Vertices socket
in  matrices    m                          # Matrix socket
out verts_out   v                          # Vertices output
out faces_out   s                          # Faces output (as number lists)
"""
```

### Socket Type Codes

| Code | Socket Type | Use Case |
|------|------------|----------|
| `s` | SvStringsSocket | Numbers, indices, face lists, generic data |
| `v` | SvVerticesSocket | Vertex coordinates, vectors |
| `m` | SvMatrixSocket | 4×4 transform matrices |

### SNLite setup() Function

Pre-compute expensive data once (e.g., lookup tables, trigonometric arrays):

```python
def setup():
    import numpy as np
    sin_table = np.sin(np.linspace(0, 2 * np.pi, 360))
    cos_table = np.cos(np.linspace(0, 2 * np.pi, 360))
    return locals()
```

Variables returned by `setup()` are available in the main script body without re-computation.

---

## mathutils Reference for AEC Transforms

### Matrix Construction

```python
from mathutils import Matrix, Vector, Euler
import math

# Translation
Matrix.Translation(Vector((x, y, z)))  # Returns 4×4 Matrix

# Rotation (angle in radians, size, axis)
Matrix.Rotation(math.radians(90), 4, 'Z')  # 90° around Z

# Scale (diagonal matrix)
Matrix.Diagonal(Vector((sx, sy, sz, 1.0)))  # Non-uniform scale

# Euler to rotation matrix
Euler((rx, ry, rz), 'XYZ').to_matrix().to_4x4()
```

### Composition Order

```python
# Standard AEC order: Scale → Rotate → Translate
transform = mat_translation @ mat_rotation @ mat_scale

# Multi-story placement: Floor offset → Element position → Rotation
final = floor_matrix @ position_matrix @ orientation_matrix
```

---

## Curve/Spline Nodes for MEP Routing

### Polyline from Points

```python
# Create a polyline path from coordinate list
node = tree.nodes.new('SvPolylineViewerNode')
```

### SvPipeNode — Pipe Generator

Sweeps a circular profile along a path. Use for pipe/conduit routing.

### Profile Sweep

For non-circular profiles (rectangular ducts):
```
[Path Vertices] → [Polyline] → [Sweep] ← [Rectangle Profile]
```

---

## match_long_repeat for AEC Data

```python
from sverchok.data_structure import match_long_repeat

# Grid positions (12 positions) × column profiles (1 profile)
positions = [[(0,0,0), (6,0,0), (12,0,0), ...]]  # 12 positions
profiles = [[cylinder_verts]]  # 1 profile

# match_long_repeat repeats the single profile for all 12 positions
matched = match_long_repeat([positions, profiles])
# profiles becomes [[cylinder_verts] * 12]
```
