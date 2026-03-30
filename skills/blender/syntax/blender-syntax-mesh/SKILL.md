---
name: blender-syntax-mesh
description: >
  Use when creating or editing mesh geometry in Blender Python -- vertices, edges, faces,
  BMesh operations, or bulk data access. Prevents the performance mistake of accessing
  vertices one-by-one instead of using foreach_get/foreach_set for bulk operations. Covers
  from_pydata, BMesh creation/editing, UV layers, vertex attributes, normals, and loops.
  Keywords: mesh, vertices, edges, faces, BMesh, from_pydata, foreach_get, foreach_set, UV layer, vertex colors, normals, bpy.types.Mesh, read vertex positions, edit mesh data, access face normals.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-mesh

## Quick Reference

### Critical Warnings

**NEVER** access `mesh.vertices`, `mesh.edges`, `mesh.polygons`, or `mesh.loops` while in Edit Mode — data is not synchronized. Use `bmesh.from_edit_mesh()` instead.

**NEVER** forget `mesh.update()` after `mesh.from_pydata()` — normals and derived data will be incorrect.

**NEVER** forget `bm.free()` after BMesh operations in Object Mode — causes memory leaks. Exception: edit-mode BMesh from `bmesh.from_edit_mesh()` must NOT be freed.

**NEVER** access BMesh elements by index without calling `ensure_lookup_table()` first — raises IndexError or undefined behavior.

**NEVER** use `mesh.calc_normals()` in Blender 4.0+ — removed, normals are computed automatically.

**NEVER** use `mesh.use_auto_smooth` or `mesh.auto_smooth_angle` in Blender 4.1+ — removed, auto-smooth is automatic when custom or sharp normals exist.

**NEVER** use `MeshEdge.bevel_weight`, `MeshEdge.crease`, or `MeshVertex.bevel_weight` in Blender 4.0+ — replaced by `mesh.attributes["bevel_weight_edge"]`, `mesh.attributes["crease_edge"]`, etc.

**ALWAYS** call `mesh.update()` after modifying mesh data via `foreach_set` or direct property writes.

**ALWAYS** link created objects to a collection — `bpy.context.collection.objects.link(obj)`. Objects without collection links are invisible.

---

### Version Matrix: Mesh API Breaking Changes

| Feature | Blender 3.x | Blender 4.0 | Blender 4.1 | Blender 4.3 |
|---------|-------------|-------------|-------------|-------------|
| Bevel weight | `edge.bevel_weight` | `mesh.attributes["bevel_weight_edge"]` | Same as 4.0 | Same as 4.0 |
| Edge crease | `edge.crease` | `mesh.attributes["crease_edge"]` | Same as 4.0 | Same as 4.0 |
| Vertex colors | `mesh.vertex_colors` | `mesh.color_attributes` | Same as 4.0 | Same as 4.0 |
| `calc_normals()` | Available | Removed (automatic) | Removed | Removed |
| `use_auto_smooth` | Available | Available | Removed (automatic) | Removed |
| `calc_normals_split()` | Available | Available | Removed | Removed |
| `MeshLoop.normal` | Read/Write | Read/Write | Read-only | Read-only |
| Custom normals | `create/calc/free_normals_split()` | Same as 3.x | `mesh.corner_normals` (auto) | Same as 4.1 |
| `AttributeGroup` | N/A | Single type | Single type | Split into `AttributeGroupMesh` etc. |
| `foreach_set` types | Silent coercion | Silent coercion | Strict `TypeError` | Strict + auto-updates |

---

## Decision Tree: Which Mesh Approach?

```
Need to create mesh from vertex/face data?
├─ Simple mesh (< 10K verts) → mesh.from_pydata()
├─ Large mesh (> 10K verts) → Pre-allocate + foreach_set
└─ Procedural/algorithmic → BMesh (verts.new, faces.new)

Need to read mesh data?
├─ Small mesh (< 1K elements) → Direct iteration (for v in mesh.vertices)
├─ Large mesh (> 1K elements) → foreach_get + numpy
└─ In Edit Mode → bmesh.from_edit_mesh()

Need to modify topology (add/remove verts/faces)?
├─ In Object Mode → BMesh (from_mesh → edit → to_mesh → free)
└─ In Edit Mode → bmesh.from_edit_mesh() → update_edit_mesh()

Need to modify vertex positions only?
├─ Few vertices → Direct: vertex.co = (x, y, z)
├─ Many vertices → foreach_set("co", flat_array)
└─ In Edit Mode → BMesh vert.co = Vector(...)

Need UV data?
├─ Read existing → mesh.uv_layers.active.data[loop_idx].uv
├─ Create new layer → mesh.uv_layers.new(name="UVMap")
├─ Bulk read/write → foreach_get/set on uv_layer.data
└─ In BMesh → bm.loops.layers.uv.active; loop[uv_lay].uv

Need custom attributes?
├─ Create → mesh.attributes.new(name, type, domain)
├─ Read → mesh.attributes["name"].data[i].value
├─ Bulk read/write → attr.data.foreach_get/set("value", array)
└─ Built-in names: "bevel_weight_edge", "crease_edge", "sharp_edge", "sharp_face"
```

---

## Essential Patterns

### Pattern 1: Create Mesh with from_pydata (All Versions)

```python
import bpy

mesh = bpy.data.meshes.new("MyMesh")
verts = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
edges = []  # Empty = auto-generate from faces
faces = [(0, 1, 2, 3)]
mesh.from_pydata(verts, edges, faces)
mesh.update()  # REQUIRED after from_pydata

obj = bpy.data.objects.new("MyObject", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED to make visible
```

### Pattern 2: High-Performance Mesh Creation with foreach_set (All Versions)

```python
import bpy
import numpy as np

n_verts = 10000
n_tris = 9998

mesh = bpy.data.meshes.new("FastMesh")
mesh.vertices.add(n_verts)
mesh.loops.add(n_tris * 3)
mesh.polygons.add(n_tris)

# Vertex coordinates: flat array, length = n_verts * 3
coords = np.random.rand(n_verts * 3).astype(np.float32)
mesh.vertices.foreach_set("co", coords)

# Loop vertex indices: flat array, length = n_tris * 3
loop_verts = np.arange(n_tris * 3, dtype=np.int32)
mesh.loops.foreach_set("vertex_index", loop_verts)

# Polygon definitions
loop_starts = np.arange(0, n_tris * 3, 3, dtype=np.int32)
loop_totals = np.full(n_tris, 3, dtype=np.int32)
mesh.polygons.foreach_set("loop_start", loop_starts)
mesh.polygons.foreach_set("loop_total", loop_totals)

mesh.update(calc_edges=True)
mesh.validate()
```

### Pattern 3: BMesh in Object Mode (All Versions)

```python
import bpy, bmesh

obj = bpy.context.active_object
bm = bmesh.new()
bm.from_mesh(obj.data)

bm.verts.ensure_lookup_table()  # REQUIRED before index access
bm.edges.ensure_lookup_table()
bm.faces.ensure_lookup_table()

# Create geometry
v1 = bm.verts.new((0, 0, 0))
v2 = bm.verts.new((1, 0, 0))
v3 = bm.verts.new((1, 1, 0))
face = bm.faces.new((v1, v2, v3))

# Write back and clean up
bm.to_mesh(obj.data)
obj.data.update()
bm.free()  # ALWAYS free in Object Mode
```

### Pattern 4: BMesh in Edit Mode (All Versions)

```python
import bpy, bmesh

obj = bpy.context.edit_object  # Must be in Edit Mode
bm = bmesh.from_edit_mesh(obj.data)

# Modify geometry
v = bm.verts.new((2, 0, 0))
bm.verts.ensure_lookup_table()

# Sync changes: do NOT call bm.free()
bmesh.update_edit_mesh(obj.data, loop_triangles=True, destructive=True)
```

### Pattern 5: Bulk Data Read with foreach_get (All Versions)

```python
import bpy
import numpy as np

mesh = bpy.context.active_object.data

# Read all vertex positions
n = len(mesh.vertices)
coords = np.empty(n * 3, dtype=np.float32)
mesh.vertices.foreach_get("co", coords)
coords = coords.reshape(-1, 3)  # Shape: (n, 3)

# Read all face normals
n_polys = len(mesh.polygons)
normals = np.empty(n_polys * 3, dtype=np.float32)
mesh.polygons.foreach_get("normal", normals)
normals = normals.reshape(-1, 3)
```

### Pattern 6: UV Layer Access (All Versions)

```python
import bpy

mesh = bpy.context.active_object.data

# Create UV layer if none exists
if not mesh.uv_layers:
    mesh.uv_layers.new(name="UVMap")

uv_layer = mesh.uv_layers.active

# Per-loop UV access
for poly in mesh.polygons:
    for loop_idx in poly.loop_indices:
        uv = uv_layer.data[loop_idx].uv  # mathutils.Vector (2D)
        print(f"Loop {loop_idx}: UV ({uv.x:.3f}, {uv.y:.3f})")
```

### Pattern 7: Generic Attributes (Blender 4.0+)

```python
import bpy
import numpy as np

mesh = bpy.context.active_object.data

# Create a per-vertex float attribute
attr = mesh.attributes.new("my_weight", type='FLOAT', domain='POINT')

# Write values
values = np.linspace(0.0, 1.0, len(mesh.vertices), dtype=np.float32)
attr.data.foreach_set("value", values)

# Read values
result = np.empty(len(mesh.vertices), dtype=np.float32)
attr.data.foreach_get("value", result)

# Access built-in attributes (4.0+ replacements)
bevel = mesh.attributes.get("bevel_weight_edge")
if bevel is None:
    bevel = mesh.attributes.new("bevel_weight_edge", 'FLOAT', 'EDGE')
```

### Pattern 8: Custom Normals (Version-Dependent)

```python
import bpy

mesh = bpy.context.active_object.data

# Blender 3.x: enable auto-smooth first
if bpy.app.version < (4, 1, 0):
    mesh.use_auto_smooth = True

# Set custom per-loop normals (all versions that support it)
# normals: list of (x, y, z) tuples, one per loop
custom_normals = [(0, 0, 1)] * len(mesh.loops)
mesh.normals_split_custom_set(custom_normals)
mesh.update()

# Read normals (4.1+): use corner_normals (auto-computed)
if bpy.app.version >= (4, 1, 0):
    for cn in mesh.corner_normals:
        print(cn.vector)
```

---

## Common Operations

### Read Mesh Data

```python
mesh = obj.data  # bpy.types.Mesh

# Vertices
for v in mesh.vertices:
    v.co          # mathutils.Vector — position
    v.normal      # mathutils.Vector — vertex normal (read-only)
    v.index       # int — index
    v.select      # bool — selection state
    v.groups      # VertexGroupElement collection

# Edges
for e in mesh.edges:
    e.vertices    # int[2] — vertex indices
    e.index       # int
    e.use_seam    # bool — UV seam
    e.use_edge_sharp  # bool — sharp edge

# Polygons (faces)
for p in mesh.polygons:
    p.vertices[:]    # list[int] — vertex indices
    p.normal         # mathutils.Vector — face normal
    p.center         # mathutils.Vector — face center
    p.area           # float — surface area
    p.loop_start     # int — first loop index
    p.loop_total     # int — loop count
    p.material_index # int — material slot
    p.use_smooth     # bool — smooth shading

# Loops (face corners)
for l in mesh.loops:
    l.vertex_index   # int — associated vertex
    l.edge_index     # int — associated edge
    l.normal         # float[3] — loop normal (read-only in 4.1+)
```

### Transform Mesh

```python
import mathutils

mesh = obj.data
mat = mathutils.Matrix.Scale(2.0, 4)
mesh.transform(mat)
mesh.update()
```

### Evaluated Mesh (Post-Modifiers)

```python
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()

for v in mesh_eval.vertices:
    print(v.co)

obj_eval.to_mesh_clear()  # ALWAYS clean up
```

---

## Performance Guide

| Operation | Method | Relative Speed |
|-----------|--------|---------------|
| Read < 1K verts | `for v in mesh.vertices` | 1x (baseline) |
| Read > 1K verts | `foreach_get("co", array)` | 10-100x faster |
| Write positions | `foreach_set("co", array)` | 10-100x faster |
| Create simple mesh | `mesh.from_pydata()` | Good |
| Create large mesh | Pre-allocate + `foreach_set` | Best |
| Topology changes | BMesh | Required |
| Interactive edits | `bmesh.from_edit_mesh()` | Best for edit mode |

**foreach_get/set rules:**
- Array MUST be flat 1D — for `co` (3 floats per vertex): length = `len(vertices) * 3`
- Use `np.float32` for float data, `np.int32` for integer data
- Blender 4.1+: strict type checking — mismatched types raise `TypeError`
- Blender 4.3+: `foreach_set` on attribute data triggers property updates automatically

---

## Attribute Type Reference (Blender 4.0+)

### Data Types

| Type | Description | `foreach_get/set` key |
|------|-------------|----------------------|
| `'FLOAT'` | Single float | `"value"` |
| `'INT'` | Integer | `"value"` |
| `'FLOAT_VECTOR'` | 3D vector | `"vector"` |
| `'FLOAT_COLOR'` | RGBA linear float | `"color"` |
| `'BYTE_COLOR'` | RGBA sRGB byte | `"color"` |
| `'BOOLEAN'` | Boolean | `"value"` |
| `'FLOAT2'` | 2D vector (UV-like) | `"vector"` |
| `'QUATERNION'` | Quaternion | `"value"` |

### Domains

| Domain | Applies to | Size equals |
|--------|-----------|-------------|
| `'POINT'` | Vertices | `len(mesh.vertices)` |
| `'EDGE'` | Edges | `len(mesh.edges)` |
| `'FACE'` | Polygons | `len(mesh.polygons)` |
| `'CORNER'` | Loops | `len(mesh.loops)` |

### Built-in Named Attributes (Blender 4.0+)

| Name | Type | Domain | Replaces |
|------|------|--------|----------|
| `"bevel_weight_vert"` | FLOAT | POINT | `MeshVertex.bevel_weight` |
| `"bevel_weight_edge"` | FLOAT | EDGE | `MeshEdge.bevel_weight` |
| `"crease_vert"` | FLOAT | POINT | Vertex crease |
| `"crease_edge"` | FLOAT | EDGE | `MeshEdge.crease` |
| `"sharp_edge"` | BOOLEAN | EDGE | Edge sharp flag |
| `"sharp_face"` | BOOLEAN | FACE | Inverse of `use_smooth` |
| `".sculpt_mask"` | FLOAT | POINT | `vertex_paint_mask` (4.1+) |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for Mesh, BMesh, MeshVertex, MeshPolygon, MeshLoop, UV layers, Attributes
- [references/examples.md](references/examples.md) — Working code examples: from_pydata, foreach_set, BMesh workflows, UV, attributes, normals
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do: 15 mesh-specific anti-patterns with version context

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Mesh.html
- https://docs.blender.org/api/current/bmesh.html
- https://docs.blender.org/api/current/bmesh.types.html
- https://docs.blender.org/api/current/bmesh.ops.html
- https://docs.blender.org/api/current/bpy.types.MeshVertex.html
- https://docs.blender.org/api/current/bpy.types.MeshPolygon.html
- https://docs.blender.org/api/current/bpy.types.MeshLoop.html
- https://docs.blender.org/api/current/bpy.types.AttributeGroup.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
