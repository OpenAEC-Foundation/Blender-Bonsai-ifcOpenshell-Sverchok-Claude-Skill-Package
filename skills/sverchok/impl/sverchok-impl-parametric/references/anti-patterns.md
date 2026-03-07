# sverchok-impl-parametric: Anti-Patterns

These are confirmed error patterns for AEC parametric modeling in Sverchok. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://github.com/nortikin/sverchok (v1.4.0+)
- vooronderzoek-sverchok.md §6, §14

---

## AP-001: Hardcoded Positions Instead of Matrix Transforms

**WHY this is wrong**: Hardcoded coordinates make the design non-parametric. Changing grid spacing requires rewriting all vertex positions. Matrix-based placement allows the entire grid to update when a single spacing parameter changes.

```python
# WRONG — hardcoded column positions, not parametric
all_verts = []
for col in column_profile:
    all_verts.append((col[0] + 0,  col[1] + 0,  col[2]))  # Column at (0, 0)
    all_verts.append((col[0] + 6,  col[1] + 0,  col[2]))  # Column at (6, 0)
    all_verts.append((col[0] + 12, col[1] + 0,  col[2]))  # Column at (12, 0)
    # Adding a column requires editing code
```

```python
# CORRECT — parametric placement via matrices
from mathutils import Matrix, Vector

positions = [(ix * span_x, iy * span_y, 0)
             for ix in range(count_x) for iy in range(count_y)]
matrices = [Matrix.Translation(Vector(p)) for p in positions]

# Place column profile at each matrix
all_verts = []
all_faces = []
for mat in matrices:
    offset = len(all_verts)
    for v in column_profile_verts:
        all_verts.append(tuple(mat @ Vector(v)))
    for f in column_profile_faces:
        all_faces.append(tuple(i + offset for i in f))
```

ALWAYS use matrix transforms for element placement. NEVER hardcode coordinates for repeating AEC elements.

---

## AP-002: Flat Vertex Lists Without Sverchok Nesting

**WHY this is wrong**: Sverchok requires ALL socket data in nested list format: `[[object_data]]`. Passing flat lists causes downstream nodes to interpret each vertex as a separate "object", producing wrong geometry. Matrix Apply, Viewer Draw, and all other nodes expect this convention.

```python
# WRONG — flat vertex list
verts = [(0,0,0), (1,0,0), (1,1,0), (0,1,0)]
self.outputs['Vertices'].sv_set(verts)
# Each vertex becomes a separate "object" — downstream nodes break
```

```python
# CORRECT — properly nested
verts = [[(0,0,0), (1,0,0), (1,1,0), (0,1,0)]]  # 1 object, 4 vertices
self.outputs['Vertices'].sv_set(verts)

# Multiple objects (e.g., individual columns):
verts = [
    [(v) for v in column_0_verts],  # Object 0
    [(v) for v in column_1_verts],  # Object 1
]
self.outputs['Vertices'].sv_set(verts)
```

ALWAYS wrap output data in the outer object list `[[...]]`. NEVER pass flat lists to `sv_set()`.

---

## AP-003: Not Using match_long_repeat for Mismatched Inputs

**WHY this is wrong**: AEC workflows frequently combine inputs of different lengths (e.g., 20 grid positions × 1 column profile). Without `match_long_repeat`, Python's `zip()` silently truncates to the shortest list, dropping elements. The result is missing columns, panels, or other elements with no error.

```python
# WRONG — zip truncates to shortest list
positions = [(0,0,0), (6,0,0), (12,0,0)]  # 3 positions
profiles = [cylinder_verts]  # 1 profile

for pos, prof in zip(positions, profiles):
    # Only processes 1 column instead of 3
    pass
```

```python
# CORRECT — match_long_repeat extends shorter lists
from sverchok.data_structure import match_long_repeat

positions = [[(0,0,0), (6,0,0), (12,0,0)]]
profiles = [[cylinder_verts]]

matched = match_long_repeat([positions, profiles])
# profiles now has 3 copies to match 3 positions
```

ALWAYS use `match_long_repeat` when combining inputs that may have different lengths. NEVER use plain `zip()` for Sverchok multi-input processing.

---

## AP-004: Generating Geometry Without Checking is_linked

**WHY this is wrong**: AEC node trees often have many output branches (columns, beams, slabs, mullions). Computing geometry for unconnected outputs wastes significant CPU time. A structural grid with 500 columns takes measurable time; computing it when no viewer is connected is pure waste.

```python
# WRONG — always computes everything
def process(self):
    columns = generate_columns(self.grid_x, self.grid_y)  # Expensive
    beams = generate_beams(self.grid_x, self.grid_y)      # Expensive
    slabs = generate_slabs(self.grid_x, self.grid_y)      # Expensive
    self.outputs['Columns'].sv_set(columns)
    self.outputs['Beams'].sv_set(beams)
    self.outputs['Slabs'].sv_set(slabs)
```

```python
# CORRECT — only compute what is needed
def process(self):
    if not any(o.is_linked for o in self.outputs):
        return

    if self.outputs['Columns'].is_linked:
        columns = generate_columns(self.grid_x, self.grid_y)
        self.outputs['Columns'].sv_set(columns)

    if self.outputs['Beams'].is_linked:
        beams = generate_beams(self.grid_x, self.grid_y)
        self.outputs['Beams'].sv_set(beams)

    if self.outputs['Slabs'].is_linked:
        slabs = generate_slabs(self.grid_x, self.grid_y)
        self.outputs['Slabs'].sv_set(slabs)
```

ALWAYS check `output.is_linked` before computing AEC geometry. ALWAYS return early if no outputs are connected.

---

## AP-005: Building Large Trees Without init_tree()

**WHY this is wrong**: An AEC building model can easily have 50+ nodes (grids, profiles, transforms, viewers). Without `init_tree()`, each of the 50 node additions and 80+ link creations triggers a full tree update. This causes O(n²) event processing — adding 130 operations triggers 130 × 130 = 16,900 update evaluations instead of 1.

```python
# WRONG — massive performance hit for AEC node trees
tree = bpy.data.node_groups.new("Building", 'SverchCustomTreeType')
for i in range(50):
    tree.nodes.new('SvBoxNodeMk2')  # Each triggers TreeEvent
for link in connections:
    tree.links.new(...)  # Each triggers TreeEvent
# Total: 130+ TreeEvents, each re-evaluating entire topology
```

```python
# CORRECT — single update after all construction
tree = bpy.data.node_groups.new("Building", 'SverchCustomTreeType')
with tree.init_tree():
    for i in range(50):
        tree.nodes.new('SvBoxNodeMk2')
    for link in connections:
        tree.links.new(...)
# Single update on exit
tree.force_update()
```

ALWAYS use `with tree.init_tree():` when building AEC node trees programmatically. NEVER add nodes/links without suppressing updates.

---

## AP-006: Wrong Transform Composition Order

**WHY this is wrong**: Matrix multiplication is not commutative. In Blender/Sverchok, the correct order is `Translation @ Rotation @ Scale` (applied right-to-left: scale first, then rotate, then translate). Reversing the order produces elements at wrong positions — columns end up offset from grid intersections after rotation.

```python
# WRONG — scale applied after translation shifts element away from origin
from mathutils import Matrix, Vector
import math

mat_loc = Matrix.Translation(Vector((6, 0, 0)))
mat_rot = Matrix.Rotation(math.radians(45), 4, 'Z')
mat_sca = Matrix.Diagonal(Vector((2, 2, 1, 1)))

transform = mat_sca @ mat_rot @ mat_loc  # WRONG ORDER
# Element is translated, then rotated, then scaled — position is wrong
```

```python
# CORRECT — standard order: Translate @ Rotate @ Scale
transform = mat_loc @ mat_rot @ mat_sca
# Element is scaled at origin, rotated at origin, then moved to position
```

ALWAYS compose transforms as `Translation @ Rotation @ Scale`. For multi-story buildings, use `FloorOffset @ ElementPosition @ ElementRotation`.

---

## AP-007: Ignoring NumPy for Large AEC Models

**WHY this is wrong**: Pure Python loops over thousands of vertices are slow. A 50×50 terrain grid has 2,500 vertices; a column grid with 100 columns at 24 vertices each has 2,400 vertices. NumPy vectorized operations are 10-100× faster for these scales. Using Python loops for large AEC models causes noticeable UI lag in Sverchok.

```python
# WRONG — Python loop for large grid (slow)
verts = []
for i in range(100):
    for j in range(100):
        verts.append((i * spacing, j * spacing, 0))
# 10,000 iterations in pure Python — slow
```

```python
# CORRECT — NumPy vectorized (fast)
import numpy as np

ix = np.arange(100) * spacing
iy = np.arange(100) * spacing
gx, gy = np.meshgrid(ix, iy)
gz = np.zeros_like(gx)
verts = [np.column_stack([gx.ravel(), gy.ravel(), gz.ravel()]).tolist()]
# Single vectorized operation — 10-100× faster
```

ALWAYS use NumPy for AEC geometry generation with more than ~500 vertices. Use `np.meshgrid`, `np.column_stack`, and vectorized math instead of Python loops.

---

## AP-008: Single Object for Entire Building

**WHY this is wrong**: Putting all building geometry (columns, beams, slabs, facade) into a single Sverchok object makes it impossible to selectively show/hide elements, assign materials per element type, or export specific categories to IFC. It also prevents partial updates — changing a facade parameter forces re-computation of columns.

```python
# WRONG — everything in one object
all_v = columns_v + beams_v + slabs_v + facade_v
all_f = columns_f + beams_f + slabs_f + facade_f  # Offset indices manually
verts = [all_v]
faces = [all_f]
```

```python
# CORRECT — separate objects per element type
verts = [columns_v, beams_v, slabs_v, facade_v]
faces = [columns_f, beams_f, slabs_f, facade_f]
# Each becomes a separate Sverchok object — can be managed independently

# Or use separate output sockets:
self.outputs['Columns'].sv_set([columns_v])
self.outputs['Beams'].sv_set([beams_v])
self.outputs['Slabs'].sv_set([slabs_v])
```

ALWAYS separate AEC elements into distinct Sverchok objects or output sockets. NEVER merge all building geometry into a single object.

---

## AP-009: Mutating CSV/JSON Input Data

**WHY this is wrong**: Data from Exchange nodes (CSV In, JSON In) is cached in the socket data cache. Mutating it in-place corrupts the cache, causing incorrect values on subsequent evaluations. The CSV/JSON node will not re-read the file — it returns the corrupted cached data.

```python
# WRONG — mutates cached CSV data
coords = self.inputs['CSV_Data'].sv_get(deepcopy=False)
for row in coords:
    row[2] *= 1000  # Convert m to mm — CORRUPTS CACHE
```

```python
# CORRECT — create new data
coords = self.inputs['CSV_Data'].sv_get(deepcopy=False)
converted = []
for row in coords:
    converted.append([row[0], row[1], row[2] * 1000])  # New list
```

ALWAYS create new data structures when transforming CSV/JSON input. NEVER mutate Exchange node output in-place.

---

## AP-010: Missing Face Winding Consistency

**WHY this is wrong**: AEC models are frequently exported to IFC or used for rendering. Inconsistent face winding (mixing clockwise and counter-clockwise vertex order) produces flipped normals. Flipped normals cause rendering artifacts, incorrect area calculations, and IFC geometry validation failures.

```python
# WRONG — inconsistent winding
faces = [
    (0, 1, 2, 3),   # Counter-clockwise (normal up)
    (7, 6, 5, 4),   # Clockwise (normal down) — INCONSISTENT
]
```

```python
# CORRECT — consistent counter-clockwise winding (Blender convention)
faces = [
    (0, 1, 2, 3),   # CCW — normal points up
    (4, 5, 6, 7),   # CCW — normal points up
]
# Use Blender's Recalculate Normals (Ctrl+N) if unsure
```

ALWAYS use consistent counter-clockwise face winding for AEC geometry. ALWAYS verify normals before IFC export.
