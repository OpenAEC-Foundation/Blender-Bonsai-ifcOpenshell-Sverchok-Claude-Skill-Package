# blender-impl-mesh — Methods Reference

Complete API signatures for Mesh, BMesh, and foreach operations used in mesh implementation workflows.

---

## bpy.types.Mesh — Core Methods

### Mesh Creation

```python
mesh = bpy.data.meshes.new(name: str) -> bpy.types.Mesh
```
Creates a new empty mesh data block. ALWAYS follow with geometry population and `mesh.update()`.

```python
mesh.from_pydata(
    vertices: list[tuple[float, float, float]],
    edges: list[tuple[int, int]],
    faces: list[tuple[int, ...]]
) -> None
```
Populate mesh from Python lists. Edges are auto-generated from faces when `edges=[]`. ALWAYS call `mesh.update()` after.

```python
mesh.update(
    calc_edges: bool = True,
    calc_edges_loose: bool = False
) -> None
```
Recalculates derived mesh data (normals, edge cache). REQUIRED after `from_pydata()` and `foreach_set()`.

```python
mesh.validate(
    verbose: bool = False,
    clean_customdata: bool = True
) -> bool
```
Validates mesh data, fixes degenerate geometry. Returns True if corrections were made.

```python
mesh.clear_geometry() -> None
```
Removes all geometry data (vertices, edges, faces, loops) from the mesh.

### Mesh Element Collections

```python
mesh.vertices                  # bpy.types.MeshVertices (read-only collection)
mesh.vertices.add(count: int)  # Pre-allocate vertices
len(mesh.vertices)             # Vertex count

mesh.edges                     # bpy.types.MeshEdges
mesh.edges.add(count: int)

mesh.loops                     # bpy.types.MeshLoops (vertex-face corners)
mesh.loops.add(count: int)

mesh.polygons                  # bpy.types.MeshPolygons
mesh.polygons.add(count: int)
```

### MeshVertex Properties

```python
vertex.co: mathutils.Vector          # Position (x, y, z) — read/write
vertex.normal: mathutils.Vector      # Normal — read-only (auto-calculated)
vertex.index: int                    # Index in mesh.vertices — read-only
vertex.select: bool                  # Selection state
vertex.hide: bool                    # Hidden state
vertex.groups: MeshVertexGroupElement # Vertex group weights
```

### MeshPolygon Properties

```python
polygon.vertices: list[int]          # Vertex indices (read-only tuple)
polygon.normal: mathutils.Vector     # Face normal — read-only
polygon.center: mathutils.Vector     # Face center — read-only
polygon.area: float                  # Face area — read-only
polygon.loop_start: int              # Start index in mesh.loops
polygon.loop_total: int              # Number of loops (= vertex count of face)
polygon.material_index: int          # Material slot index
polygon.use_smooth: bool             # Smooth shading flag
polygon.index: int                   # Index in mesh.polygons
```

### MeshLoop Properties

```python
loop.vertex_index: int               # Index of the vertex this loop uses
loop.edge_index: int                 # Index of the outgoing edge
loop.normal: mathutils.Vector        # Loop normal (per-face-vertex normal)
loop.index: int                      # Index in mesh.loops
```

### MeshEdge Properties

```python
edge.vertices: tuple[int, int]       # Pair of vertex indices — read-only
edge.is_loose: bool                  # True if not used by any face
edge.use_seam: bool                  # UV seam flag
edge.use_sharp: bool                 # Sharp edge flag
edge.index: int                      # Index in mesh.edges
```

---

## foreach_get / foreach_set API

All mesh element collections support bulk read/write via `foreach_get` and `foreach_set`.

### Signatures

```python
collection.foreach_get(attr: str, seq: MutableSequence) -> None
collection.foreach_set(attr: str, seq: Sequence) -> None
```

### Attribute Reference Table

| Collection | Attribute | Type | Dimension | Array dtype |
|-----------|-----------|------|-----------|-------------|
| `mesh.vertices` | `"co"` | float | 3 | `float32` or `float64` |
| `mesh.vertices` | `"normal"` | float | 3 | `float32` |
| `mesh.vertices` | `"select"` | bool | 1 | `bool` or `int32` |
| `mesh.vertices` | `"hide"` | bool | 1 | `bool` or `int32` |
| `mesh.loops` | `"vertex_index"` | int | 1 | `int32` |
| `mesh.loops` | `"normal"` | float | 3 | `float32` |
| `mesh.polygons` | `"loop_start"` | int | 1 | `int32` |
| `mesh.polygons` | `"loop_total"` | int | 1 | `int32` |
| `mesh.polygons` | `"material_index"` | int | 1 | `int32` |
| `mesh.polygons` | `"use_smooth"` | bool | 1 | `bool` or `int32` |
| `mesh.polygons` | `"vertices"` | int | variable | Not supported — use loops |
| `mesh.edges` | `"vertices"` | int | 2 | `int32` |
| `mesh.edges` | `"use_seam"` | bool | 1 | `bool` or `int32` |

### Array Size Rules

- **Total array length** = `len(collection) * dimension`
- For `mesh.vertices.foreach_set("co", arr)`: arr length MUST be `len(mesh.vertices) * 3`
- For `mesh.loops.foreach_set("vertex_index", arr)`: arr length MUST be `len(mesh.loops)`
- Arrays MUST be flat (1D). Use `numpy_array.reshape(-1)` or `.flatten()`.

---

## bmesh Module — Core API

### BMesh Creation and I/O

```python
bm = bmesh.new() -> bmesh.types.BMesh
```
Create a new empty BMesh. ALWAYS call `bm.free()` when done.

```python
bm.from_mesh(
    mesh: bpy.types.Mesh,
    face_normals: bool = True,
    vertex_normals: bool = True,
    use_shape_key: bool = False,
    shape_key_index: int = 0
) -> None
```
Load mesh data into BMesh. Object MUST be in Object Mode.

```python
bm.to_mesh(mesh: bpy.types.Mesh) -> None
```
Write BMesh data back to a Blender mesh.

```python
bm.free() -> None
```
Free BMesh memory. REQUIRED after `bmesh.new()` and `bm.from_mesh()`.

```python
bm = bmesh.from_edit_mesh(mesh: bpy.types.Mesh) -> bmesh.types.BMesh
```
Get BMesh from mesh in Edit Mode. Do NOT call `bm.free()` — use `bmesh.update_edit_mesh()` instead.

```python
bmesh.update_edit_mesh(
    mesh: bpy.types.Mesh,
    loop_triangles: bool = True,
    destructive: bool = True
) -> None
```
Update Edit Mode mesh from BMesh changes.

### BMesh Element Access

```python
bm.verts                              # BMVertSeq
bm.edges                              # BMEdgeSeq
bm.faces                              # BMFaceSeq

bm.verts.ensure_lookup_table()         # REQUIRED before index access
bm.edges.ensure_lookup_table()
bm.faces.ensure_lookup_table()

bm.verts[index]                        # Access by index (after ensure_lookup_table)
bm.verts.new(co: tuple) -> BMVert     # Create new vertex
bm.edges.new(verts: tuple) -> BMEdge  # Create new edge
bm.faces.new(verts: list) -> BMFace   # Create new face
```

### BMVert Properties

```python
vert.co: mathutils.Vector             # Position — read/write
vert.normal: mathutils.Vector         # Normal — read-only
vert.index: int                       # Index (unstable after topology changes)
vert.link_edges: list[BMEdge]         # Connected edges
vert.link_faces: list[BMFace]         # Connected faces
vert.is_boundary: bool                # True if on mesh boundary
vert.is_manifold: bool                # True if manifold vertex
```

### BMFace Properties

```python
face.normal: mathutils.Vector         # Face normal — read-only
face.calc_area() -> float             # Calculate face area
face.calc_center_median() -> Vector   # Face center (median)
face.calc_center_bounds() -> Vector   # Face center (bounding box)
face.material_index: int              # Material slot index
face.smooth: bool                     # Smooth shading flag
face.verts: list[BMVert]              # Ordered vertices
face.edges: list[BMEdge]              # Ordered edges
face.loops: list[BMLoop]              # Face loops
```

### BMesh Calc Methods

```python
bm.calc_volume(signed: bool = False) -> float
```
Calculate mesh volume. Only valid for closed (watertight) meshes.

```python
bm.transform(matrix: mathutils.Matrix, filter: set = None) -> None
```
Transform all vertices by matrix. Use to apply world transform before analysis.

```python
bm.normal_update() -> None
```
Recalculate all normals.

---

## bmesh.ops — Operator Reference (AEC-relevant subset)

### Geometry Creation

```python
bmesh.ops.create_grid(bm, x_segments=1, y_segments=1, size=1.0,
                       matrix=Matrix.Identity(4), calc_uvs=True) -> dict
# Returns: {'verts': [...]}

bmesh.ops.create_cube(bm, size=1.0, matrix=Matrix.Identity(4),
                       calc_uvs=True) -> dict
# Returns: {'verts': [...]}
```

### Extrusion

```python
bmesh.ops.extrude_face_region(bm, geom, use_keep_orig=False,
                               use_normal_flip=False) -> dict
# Returns: {'geom': [...]}  — contains new verts, edges, faces

bmesh.ops.extrude_edge_only(bm, edges, use_normal_flip=False) -> dict
# Returns: {'geom': [...]}

bmesh.ops.extrude_vert_indiv(bm, verts) -> dict
# Returns: {'verts': [...], 'edges': [...]}
```

### Transform

```python
bmesh.ops.translate(bm, vec, verts, space=Matrix.Identity(4)) -> dict
bmesh.ops.rotate(bm, cent, matrix, verts, space=Matrix.Identity(4)) -> dict
bmesh.ops.scale(bm, vec, verts, space=Matrix.Identity(4)) -> dict
bmesh.ops.transform(bm, matrix, verts, space=Matrix.Identity(4),
                     use_shapekey=False) -> dict
```

### Topology Operations

```python
bmesh.ops.subdivide_edges(bm, edges, cuts=1, use_grid_fill=True,
                           use_single_edge=False) -> dict
# Returns: {'geom_inner': [...], 'geom_split': [...], 'geom': [...]}

bmesh.ops.inset_individual(bm, faces, thickness=0.0, depth=0.0,
                            use_even_offset=True) -> dict
bmesh.ops.inset_region(bm, faces, thickness=0.0, depth=0.0,
                        use_boundary=True, use_even_offset=True) -> dict

bmesh.ops.dissolve_verts(bm, verts, use_face_split=False,
                          use_boundary_tear=False) -> dict
bmesh.ops.dissolve_edges(bm, edges, use_verts=True,
                          use_face_split=False) -> dict
bmesh.ops.dissolve_faces(bm, faces, use_verts=True) -> dict

bmesh.ops.recalc_face_normals(bm, faces) -> dict
```

### Mesh Analysis

```python
bmesh.ops.triangulate(bm, faces, quad_method='BEAUTY',
                       ngon_method='BEAUTY') -> dict
# Returns: {'faces': [...], 'face_map': [...]}

bmesh.ops.remove_doubles(bm, verts, dist=0.0001) -> dict
# Returns: {} — modifies in place
```

---

## Mesh Attributes API (Blender 3.2+/4.x/5.x)

### Attribute Management

```python
mesh.attributes                                    # AttributeGroup
mesh.attributes.new(name: str, type: str, domain: str) -> Attribute
mesh.attributes.remove(attribute: Attribute) -> None
mesh.attributes[name: str] -> Attribute            # Access by name
mesh.attributes.active -> Attribute                # Active attribute
```

### Type Values

| Type | Description |
|------|------------|
| `'FLOAT'` | Single float |
| `'INT'` | Single integer |
| `'FLOAT_VECTOR'` | 3D vector |
| `'FLOAT_COLOR'` | RGBA color |
| `'BYTE_COLOR'` | RGBA color (8-bit) |
| `'STRING'` | String |
| `'BOOLEAN'` | Boolean |
| `'FLOAT2'` | 2D vector |
| `'INT8'` | 8-bit integer |
| `'INT32_2D'` | 2D integer vector |
| `'QUATERNION'` | Quaternion (Blender 4.1+) |
| `'FLOAT4X4'` | 4x4 matrix (Blender 4.1+) |

### Domain Values

| Domain | Maps to |
|--------|---------|
| `'POINT'` | Per-vertex |
| `'EDGE'` | Per-edge |
| `'FACE'` | Per-face |
| `'CORNER'` | Per-loop (face corner) |

### Attribute Data Access

```python
attr = mesh.attributes["my_attr"]
for item in attr.data:
    item.value        # For FLOAT, INT, BOOLEAN
    item.vector       # For FLOAT_VECTOR
    item.color        # For FLOAT_COLOR, BYTE_COLOR

# Bulk access via foreach_get/foreach_set
values = [0.0] * len(attr.data)
attr.data.foreach_get("value", values)
attr.data.foreach_set("value", new_values)
```

---

## UV Layer Access

```python
# Read UV data
uv_layer = mesh.uv_layers.active           # Active UV layer
uv_layer = mesh.uv_layers["UVMap"]          # By name
uv_layer = mesh.uv_layers.new(name="MyUV") # Create new

# Access per-loop UVs
for loop_idx, loop in enumerate(mesh.loops):
    uv = uv_layer.data[loop_idx].uv        # mathutils.Vector (2D)
    vert_idx = loop.vertex_index

# Bulk UV access
uv_data = [0.0] * (len(mesh.loops) * 2)
uv_layer.data.foreach_get("uv", uv_data)
```

---

## Object and Mesh Linking

```python
obj = bpy.data.objects.new(name: str, object_data: bpy.types.Mesh) -> bpy.types.Object
bpy.context.collection.objects.link(obj)   # Link to active collection
collection.objects.link(obj)               # Link to specific collection
bpy.data.objects.remove(obj)               # Remove object
bpy.data.meshes.remove(mesh)               # Remove mesh data block
```

---

## Blender 4.0+ Attribute Migration

Edge properties `bevel_weight` and `crease` were moved to the generic attributes system in Blender 4.0. Face Maps were removed entirely.

### Edge Bevel Weight

```python
# Blender 3.x — direct property (REMOVED in 4.0)
for edge in mesh.edges:
    edge.bevel_weight = 1.0

# Blender 4.0+/5.x — via attributes
if "bevel_weight_edge" not in mesh.attributes:
    mesh.attributes.new(name="bevel_weight_edge", type='FLOAT', domain='EDGE')
attr = mesh.attributes["bevel_weight_edge"]
for i in range(len(mesh.edges)):
    attr.data[i].value = 1.0
```

### Edge Crease

```python
# Blender 3.x — direct property (REMOVED in 4.0)
for edge in mesh.edges:
    edge.crease = 0.5

# Blender 4.0+/5.x — via attributes
if "crease_edge" not in mesh.attributes:
    mesh.attributes.new(name="crease_edge", type='FLOAT', domain='EDGE')
attr = mesh.attributes["crease_edge"]
for i in range(len(mesh.edges)):
    attr.data[i].value = 0.5
```

### Face Maps Replacement

```python
# Blender 3.x — Face Maps (REMOVED in 4.0)
# obj.face_maps.new(name="zone_A")

# Blender 4.0+/5.x — use integer face attribute
if "face_zone" not in mesh.attributes:
    mesh.attributes.new(name="face_zone", type='INT', domain='FACE')
attr = mesh.attributes["face_zone"]
for i in range(len(mesh.polygons)):
    attr.data[i].value = 0  # Zone ID
```
