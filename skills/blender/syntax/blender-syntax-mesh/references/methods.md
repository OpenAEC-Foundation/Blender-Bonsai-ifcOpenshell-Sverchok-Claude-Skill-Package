# blender-syntax-mesh — Method Reference

Complete API signatures for Blender mesh data access. All signatures verified against Blender 4.x official documentation.

---

## bpy.types.Mesh(ID)

### Properties

| Property | Type | Access | Version Notes |
|----------|------|--------|---------------|
| `vertices` | `MeshVertices` | R | All versions |
| `edges` | `MeshEdges` | R | All versions |
| `polygons` | `MeshPolygons` | R | All versions |
| `loops` | `MeshLoops` | R | All versions |
| `loop_triangles` | `MeshLoopTriangles` | R | Computed tessellation |
| `uv_layers` | `UVLoopLayers` | R | All versions |
| `vertex_colors` | `LoopColors` | R | Legacy; use `color_attributes` in 4.0+ |
| `color_attributes` | `AttributeGroupMesh` | R | 4.0+ |
| `attributes` | `AttributeGroupMesh` | R | 4.0+; type split in 4.3 |
| `materials` | `IDMaterials` | R | All versions |
| `shape_keys` | `Key` | R | All versions |
| `vertex_normals` | `MeshNormalValue[]` | R | Per-vertex normals |
| `polygon_normals` | `MeshNormalValue[]` | R | Per-face normals |
| `corner_normals` | `MeshNormalValue[]` | R | 4.1+ (replaces split normals cache) |
| `has_custom_normals` | `bool` | R | All versions |
| `is_editmode` | `bool` | R | All versions |
| `total_vert_sel` | `int` | R | Edit mode only |
| `total_edge_sel` | `int` | R | Edit mode only |
| `total_face_sel` | `int` | R | Edit mode only |
| `use_auto_smooth` | `bool` | RW | **Removed in 4.1** |
| `auto_smooth_angle` | `float` | RW | **Removed in 4.1** |

### Methods

```python
Mesh.from_pydata(vertices, edges, faces, shade_flat=True)
```
- `vertices`: sequence of `(x, y, z)` float tuples
- `edges`: sequence of `(v1, v2)` int tuples (pass `[]` to auto-generate from faces)
- `faces`: sequence of `(v1, v2, v3, ...)` int tuples (3+ vertices per face, supports n-gons)
- `shade_flat`: bool (default `True` in 4.0+) — flat shading on created faces
- ALWAYS call `mesh.update()` after this method

```python
Mesh.update(calc_edges=False, calc_edges_loose=False)
```
- Recalculates derived mesh data (normals, tessellation, edge lists)
- `calc_edges=True`: recompute edge list from polygon data (use after `foreach_set` mesh creation)
- `calc_edges_loose=True`: tag loose edges

```python
Mesh.validate(verbose=False, clean_customdata=True) -> bool
```
- Returns `True` if mesh had invalid geometry (fixes in place)
- `verbose=True`: print warnings to console
- `clean_customdata=True`: remove unused custom data layers

```python
Mesh.transform(matrix, shape_keys=False)
```
- `matrix`: `mathutils.Matrix` (4×4) applied to all vertex positions
- `shape_keys=True`: also transform shape key data

```python
Mesh.flip_normals()
```
- Reverses winding order of all polygons

```python
Mesh.clear_geometry()
```
- Removes all geometry; preserves shape keys and materials

```python
Mesh.calc_tangents(uvmap="")
```
- Computes tangent space per loop. Requires UV map.
- `uvmap`: name of UV layer (empty string = active UV layer)

```python
Mesh.normals_split_custom_set(normals)
```
- `normals`: sequence of `(x, y, z)` tuples, one per loop (length = `len(mesh.loops)`)
- Sets custom split normals per face corner

```python
Mesh.normals_split_custom_set_from_vertices(normals)
```
- `normals`: sequence of `(x, y, z)` tuples, one per vertex (length = `len(mesh.vertices)`)
- Interpolates to per-loop normals

```python
Mesh.count_selected_items() -> tuple[int, int, int]
```
- Returns `(selected_verts, selected_edges, selected_faces)`

### Removed Methods

| Method | Removed In | Replacement |
|--------|-----------|-------------|
| `calc_normals()` | 4.0 | Automatic |
| `create_normals_split()` | 4.1 | Automatic |
| `calc_normals_split()` | 4.1 | `mesh.corner_normals` (auto-computed property) |
| `free_normals_split()` | 4.1 | Automatic |

---

## bpy.types.MeshVertex(bpy_struct)

| Property | Type | Access | Notes |
|----------|------|--------|-------|
| `co` | `float[3]` | RW | Vertex position |
| `normal` | `float[3]` | R | Vertex normal |
| `index` | `int` | R | Index in `mesh.vertices` |
| `hide` | `bool` | RW | Hidden in edit mode |
| `select` | `bool` | RW | Selection state |
| `groups` | `VertexGroupElement[]` | R | Vertex group memberships |
| `undeformed_co` | `float[3]` | R | Position before deformation |
| `bevel_weight` | `float` | RW | **Removed in 4.0** — use attribute `"bevel_weight_vert"` |

---

## bpy.types.MeshEdge(bpy_struct)

| Property | Type | Access | Notes |
|----------|------|--------|-------|
| `vertices` | `int[2]` | R | Endpoint vertex indices |
| `index` | `int` | R | Index in `mesh.edges` |
| `hide` | `bool` | RW | Hidden in edit mode |
| `select` | `bool` | RW | Selection state |
| `use_seam` | `bool` | RW | UV seam flag |
| `use_edge_sharp` | `bool` | RW | Sharp edge flag |
| `is_loose` | `bool` | R | Not connected to any face |
| `bevel_weight` | `float` | RW | **Removed in 4.0** — use attribute `"bevel_weight_edge"` |
| `crease` | `float` | RW | **Removed in 4.0** — use attribute `"crease_edge"` |

---

## bpy.types.MeshPolygon(bpy_struct)

| Property | Type | Access | Notes |
|----------|------|--------|-------|
| `vertices` | `int[]` | R | Vertex indices (3+ for n-gons) |
| `loop_start` | `int` | R | First loop index |
| `loop_total` | `int` | R | Number of loops/corners |
| `loop_indices` | range | R | Range `loop_start` to `loop_start + loop_total` |
| `normal` | `float[3]` | R | Face normal vector |
| `center` | `float[3]` | R | Geometric center |
| `area` | `float` | R | Surface area |
| `material_index` | `int` | RW | Material slot index |
| `hide` | `bool` | RW | Hidden in edit mode |
| `select` | `bool` | RW | Selection state |
| `use_smooth` | `bool` | RW | Smooth shading flag |

**Method:** `flip()` — reverses vertex winding, recalculates normal

---

## bpy.types.MeshLoop(bpy_struct)

| Property | Type | Access | Notes |
|----------|------|--------|-------|
| `vertex_index` | `int` | R | Associated vertex index |
| `edge_index` | `int` | R | Associated edge index |
| `index` | `int` | R | Index in `mesh.loops` |
| `normal` | `float[3]` | R | Loop normal (**read-only since 4.1**) |
| `tangent` | `float[3]` | R | Requires `calc_tangents()` first |
| `bitangent` | `float[3]` | R | Requires `calc_tangents()` first |
| `bitangent_sign` | `float` | R | +1 or -1 |

---

## UV Layer API

### mesh.uv_layers (UVLoopLayers)

```python
mesh.uv_layers.new(name="", do_init=True) -> MeshUVLoopLayer
mesh.uv_layers.remove(layer)
mesh.uv_layers.active                # Active UV layer (MeshUVLoopLayer or None)
mesh.uv_layers.active_index          # int — active layer index
mesh.uv_layers[0]                    # Access by index
mesh.uv_layers["UVMap"]              # Access by name
```

### MeshUVLoopLayer

| Property | Type | Notes |
|----------|------|-------|
| `name` | `str` | Layer name |
| `data` | `MeshUVLoop[]` | Per-loop UV data (length = `len(mesh.loops)`) |
| `active` | `bool` | Active UV layer flag |
| `active_render` | `bool` | Active for rendering |

### MeshUVLoop

| Property | Type | Notes |
|----------|------|-------|
| `uv` | `float[2]` | UV coordinate (mathutils.Vector 2D) |
| `pin_uv` | `bool` | UV is pinned |
| `select` | `bool` | UV vertex selected |
| `select_edge` | `bool` | UV edge selected |

---

## foreach_get / foreach_set

Methods on any `bpy_prop_collection`. Bulk read/write across the Python/C boundary.

```python
collection.foreach_get(attr: str, seq) -> None
collection.foreach_set(attr: str, seq) -> None
```

### Rules

- `seq` MUST be a flat 1D sequence (list, tuple, or numpy array)
- For multi-component properties (e.g., `co` = 3 floats): length = `len(collection) * component_count`
- Supported types: `bool`, `int`, `float`
- **Blender 4.1+**: raises `TypeError` for type mismatches (previously silent coercion)
- **Blender 4.3+**: `foreach_set` on attribute data triggers property updates automatically

### Common foreach_get/set Attributes

| Collection | Attribute | Components | Type |
|------------|-----------|-----------|------|
| `mesh.vertices` | `"co"` | 3 | float |
| `mesh.vertices` | `"normal"` | 3 | float (read-only) |
| `mesh.vertices` | `"select"` | 1 | bool |
| `mesh.vertices` | `"hide"` | 1 | bool |
| `mesh.edges` | `"vertices"` | 2 | int |
| `mesh.edges` | `"select"` | 1 | bool |
| `mesh.polygons` | `"vertices"` | variable | int (use per-polygon) |
| `mesh.polygons` | `"loop_start"` | 1 | int |
| `mesh.polygons` | `"loop_total"` | 1 | int |
| `mesh.polygons` | `"normal"` | 3 | float (read-only) |
| `mesh.polygons` | `"material_index"` | 1 | int |
| `mesh.polygons` | `"use_smooth"` | 1 | bool |
| `mesh.loops` | `"vertex_index"` | 1 | int |
| `mesh.loops` | `"edge_index"` | 1 | int |
| `mesh.loops` | `"normal"` | 3 | float (read-only in 4.1+) |
| `uv_layer.data` | `"uv"` | 2 | float |
| `attr.data` | `"value"` | 1 | varies by attribute type |
| `attr.data` | `"vector"` | 3 | float (FLOAT_VECTOR) |
| `attr.data` | `"color"` | 4 | float (FLOAT_COLOR/BYTE_COLOR) |

---

## Attribute System (Blender 4.0+)

### AttributeGroup / AttributeGroupMesh (4.3+)

```python
mesh.attributes.new(name: str, type: str, domain: str) -> Attribute
mesh.attributes.remove(attribute)
mesh.attributes.get(name: str) -> Attribute or None
mesh.attributes.active              # Currently active attribute
mesh.attributes.active_color        # Active color attribute (4.3: AttributeGroupMesh only)
mesh.attributes.domain_size(domain) # Element count for domain (4.3+)
```

### Data Types

| Enum Value | Description | Components |
|------------|-------------|-----------|
| `'FLOAT'` | Single float | 1 |
| `'INT'` | Integer | 1 |
| `'FLOAT_VECTOR'` | 3D vector | 3 |
| `'FLOAT_COLOR'` | RGBA linear float | 4 |
| `'BYTE_COLOR'` | RGBA sRGB byte | 4 |
| `'BOOLEAN'` | Boolean | 1 |
| `'INT8'` | 8-bit integer | 1 |
| `'FLOAT2'` | 2D vector | 2 |
| `'QUATERNION'` | Quaternion | 4 |
| `'FLOAT4X4'` | 4×4 matrix | 16 |

### Domains

| Enum Value | Element Type | Size |
|------------|-------------|------|
| `'POINT'` | Vertices | `len(mesh.vertices)` |
| `'EDGE'` | Edges | `len(mesh.edges)` |
| `'FACE'` | Polygons | `len(mesh.polygons)` |
| `'CORNER'` | Loops | `len(mesh.loops)` |

---

## bmesh Module

### Module-Level Functions

```python
bmesh.new(use_operators=True) -> BMesh
```
- Creates empty BMesh. `use_operators=True` allocates extra memory for `bmesh.ops` support.

```python
bmesh.from_edit_mesh(mesh: bpy.types.Mesh) -> BMesh
```
- Returns BMesh for a mesh in **edit mode**. Do NOT call `free()` on this.

```python
bmesh.update_edit_mesh(mesh: bpy.types.Mesh, loop_triangles=True, destructive=True)
```
- Syncs BMesh edits back to the mesh in edit mode.
- `destructive=True`: geometry was added or removed.

---

## bmesh.types.BMesh

### Properties

| Property | Type | Access |
|----------|------|--------|
| `verts` | `BMVertSeq` | R |
| `edges` | `BMEdgeSeq` | R |
| `faces` | `BMFaceSeq` | R |
| `loops` | `BMLoopSeq` | R |
| `select_mode` | `set` | RW — `{'VERT'}`, `{'EDGE'}`, `{'FACE'}` |
| `select_history` | `BMEditSelSeq` | R |
| `is_valid` | `bool` | R |
| `is_wrapped` | `bool` | R — True when owned by Blender (edit mode) |

### Methods

```python
bm.from_mesh(mesh, face_normals=True, vertex_normals=True, use_shape_key=False, shape_key_index=0)
bm.to_mesh(mesh: bpy.types.Mesh)
bm.from_object(object, depsgraph, *, cage=False, face_normals=True, vertex_normals=True)
bm.copy() -> BMesh
bm.clear()
bm.free()
bm.normal_update()
bm.transform(matrix, *, filter=None)
bm.calc_volume(*, signed=False) -> float
bm.calc_loop_triangles() -> list[tuple[BMLoop, BMLoop, BMLoop]]
bm.select_flush(select: bool)
bm.select_flush_mode(*, flush_down=False)
```

---

## BMesh Element Types

### BMVert

| Property | Type | Access |
|----------|------|--------|
| `co` | `Vector` | RW |
| `normal` | `Vector` | RW |
| `index` | `int` | RW |
| `hide` | `bool` | RW |
| `select` | `bool` | RW |
| `tag` | `bool` | RW |
| `link_edges` | `BMElemSeq[BMEdge]` | R |
| `link_faces` | `BMElemSeq[BMFace]` | R |
| `link_loops` | `BMElemSeq[BMLoop]` | R |
| `is_boundary` | `bool` | R |
| `is_manifold` | `bool` | R |
| `is_wire` | `bool` | R |

### BMEdge

| Property | Type | Access |
|----------|------|--------|
| `verts` | `BMElemSeq[BMVert]` (len 2) | R |
| `index` | `int` | RW |
| `smooth` | `bool` | RW |
| `seam` | `bool` | RW |
| `link_faces` | `BMElemSeq[BMFace]` | R |
| `is_boundary` | `bool` | R |
| `is_manifold` | `bool` | R |
| `is_wire` | `bool` | R |

**Methods:** `calc_length()`, `calc_face_angle(fallback=None)`, `other_vert(vert)`

### BMFace

| Property | Type | Access |
|----------|------|--------|
| `verts` | `BMElemSeq[BMVert]` | R |
| `edges` | `BMElemSeq[BMEdge]` | R |
| `loops` | `BMElemSeq[BMLoop]` | R |
| `normal` | `Vector` | RW |
| `index` | `int` | RW |
| `smooth` | `bool` | RW |
| `material_index` | `int` | RW |

**Methods:** `calc_area()`, `calc_perimeter()`, `calc_center_median()`, `calc_center_bounds()`, `normal_update()`, `normal_flip()`

### BMLoop

| Property | Type | Access |
|----------|------|--------|
| `vert` | `BMVert` | R |
| `edge` | `BMEdge` | R |
| `face` | `BMFace` | R |
| `index` | `int` | RW |
| `link_loop_next` | `BMLoop` | R |
| `link_loop_prev` | `BMLoop` | R |
| `link_loop_radial_next` | `BMLoop` | R |
| `link_loop_radial_prev` | `BMLoop` | R |

**Methods:** `calc_angle()`, `calc_normal()`, `calc_tangent()`

---

## BMesh Sequence Operations

### BMVertSeq

```python
bm.verts.new(co=(0, 0, 0), example=None) -> BMVert
bm.verts.remove(vert)
bm.verts.ensure_lookup_table()    # REQUIRED before integer indexing
bm.verts.index_update()           # Updates .index on all verts
bm.verts.sort(key=None, reverse=False)
bm.verts.layers                   # BMLayerAccessVert
```

### BMEdgeSeq

```python
bm.edges.new(verts, example=None) -> BMEdge  # verts: 2-element sequence
bm.edges.get(verts, fallback=None) -> BMEdge  # Lookup existing edge
bm.edges.remove(edge)
bm.edges.ensure_lookup_table()
bm.edges.index_update()
```

### BMFaceSeq

```python
bm.faces.new(verts, example=None) -> BMFace  # verts: 3+ BMVert sequence
bm.faces.get(verts, fallback=None) -> BMFace  # Lookup existing face
bm.faces.remove(face)
bm.faces.ensure_lookup_table()
bm.faces.index_update()
bm.faces.active                    # Active BMFace or None
```

---

## BMesh Layer System

Access: `bm.{verts|edges|faces|loops}.layers.{layer_type}`

### Layer Types per Element

| Element | Layer Types |
|---------|------------|
| `verts.layers` | `float`, `int`, `bool`, `float_vector`, `float_color`, `color`, `string`, `shape`, `deform`, `skin` |
| `edges.layers` | `float`, `int`, `bool`, `float_vector`, `float_color`, `color`, `string` |
| `faces.layers` | `float`, `int`, `bool`, `float_vector`, `float_color`, `color`, `string` |
| `loops.layers` | `float`, `int`, `bool`, `float_vector`, `float_color`, `string`, `uv`, `color` |

### BMLayerCollection

```python
layer_col = bm.loops.layers.uv
layer = layer_col.active           # Active layer
layer = layer_col.verify()         # Get or create active
layer = layer_col.new(name="")     # Create named layer
layer_col.remove(layer)
layer_col.get(key, default=None)
layer_col.keys()
layer_col.values()
layer_col.items()
```

---

## Key bmesh.ops Signatures

```python
bmesh.ops.triangulate(bm, *, faces, quad_method='BEAUTY', ngon_method='BEAUTY')
# Returns: {'faces': [...], 'edges': [...], 'face_map': dict, 'face_map_double': dict}

bmesh.ops.extrude_face_region(bm, *, geom, edges_exclude=set(),
    use_keep_orig=False, use_normal_flip=False)
# Returns: {'geom': [BMVert|BMEdge|BMFace, ...]}

bmesh.ops.subdivide_edges(bm, *, edges, cuts=1, smooth=0.0,
    smooth_falloff='LINEAR', use_grid_fill=False)
# Returns: {'geom_inner': ..., 'geom_split': ..., 'geom': ...}

bmesh.ops.remove_doubles(bm, *, verts, dist=0.0001)
# Merges vertices by distance. No return value.

bmesh.ops.translate(bm, *, vec, verts, space=Matrix())
bmesh.ops.rotate(bm, *, cent, matrix, verts, space=Matrix())
bmesh.ops.scale(bm, *, vec, verts, space=Matrix())

bmesh.ops.convex_hull(bm, *, input, use_existing_faces=False)
# Returns: {'geom': ..., 'geom_interior': ..., 'geom_unused': ..., 'geom_holes': ...}

bmesh.ops.bisect_plane(bm, *, geom, plane_co, plane_no, dist=0.0001,
    clear_outer=False, clear_inner=False)
# Returns: {'geom_cut': [...], 'geom': [...]}
```

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Mesh.html
- https://docs.blender.org/api/current/bmesh.html
- https://docs.blender.org/api/current/bmesh.types.html
- https://docs.blender.org/api/current/bmesh.ops.html
- https://docs.blender.org/api/current/bpy.types.MeshVertex.html
- https://docs.blender.org/api/current/bpy.types.MeshPolygon.html
- https://docs.blender.org/api/current/bpy.types.MeshLoop.html
- https://docs.blender.org/api/current/bpy.types.MeshUVLoopLayer.html
- https://docs.blender.org/api/current/bpy.types.AttributeGroup.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
