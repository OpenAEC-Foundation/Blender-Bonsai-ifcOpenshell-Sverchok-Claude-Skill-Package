---
name: blender-impl-mesh
description: >
  Use when generating mesh geometry for AEC applications -- buildings from vertices,
  IFC geometry visualization, parametric elements, or custom mesh tools. Prevents the
  performance mistake of creating vertices one-by-one instead of using from_pydata or
  BMesh batch operations. Covers mesh creation, BMesh algorithms, foreach_get/set
  optimization, and mesh analysis tools.
  Keywords: mesh generation, from_pydata, BMesh, parametric, building geometry, mesh
  analysis, foreach_get, foreach_set, vertices, AEC mesh, custom mesh tool.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-impl-mesh

## Quick Reference

### Decision Tree: Which Mesh API to Use

```
Need to create mesh geometry?
├── Simple shape (< 1000 verts)?
│   └── Use mesh.from_pydata() → Pattern 1
├── Large mesh (> 1000 verts)?
│   └── Use pre-allocate + foreach_set → Pattern 2
├── Need topology edits (extrude, inset, dissolve)?
│   └── Use BMesh → Pattern 3
├── Need post-modifier mesh data?
│   └── Use Depsgraph evaluated mesh → Pattern 4
└── Visualizing IFC geometry in Blender?
    └── Use foreach_set with IFC vertex data → Pattern 5
```

### Decision Tree: BMesh Entry Point

```
Which BMesh context?
├── Object Mode (standalone processing)?
│   ├── bm = bmesh.new()
│   ├── bm.from_mesh(mesh_data)
│   ├── ... modify ...
│   ├── bm.to_mesh(mesh_data)
│   └── bm.free()  ← ALWAYS free
├── Edit Mode (interactive tool)?
│   ├── bm = bmesh.from_edit_mesh(obj.data)
│   ├── ... modify ...
│   └── bmesh.update_edit_mesh(obj.data)  ← NO free
└── New geometry from scratch?
    ├── bm = bmesh.new()
    ├── ... create verts/edges/faces ...
    ├── bm.to_mesh(new_mesh)
    └── bm.free()
```

### Critical Warnings

**NEVER** access `mesh.vertices`, `mesh.edges`, or `mesh.polygons` in Edit Mode — data is not synchronized. Use `bmesh.from_edit_mesh()` instead.

**NEVER** skip `bm.free()` after `bmesh.new()` or `bm.from_mesh()` — causes memory leaks.

**NEVER** skip `mesh.update()` after `mesh.from_pydata()` — normals and internal state remain invalid.

**NEVER** skip `bm.verts.ensure_lookup_table()` before index access — raises `IndexError`.

**NEVER** use `mesh.calc_normals()` in Blender 4.0+ — removed, normals are auto-calculated.

**ALWAYS** call `obj_eval.to_mesh_clear()` after reading evaluated mesh data — prevents memory leaks.

**ALWAYS** call `mesh.validate()` after building mesh from raw data — catches degenerate geometry silently.

---

## Essential Patterns

### Pattern 1: Create Mesh from Vertices (from_pydata)

Use for simple meshes under ~1000 vertices. Suitable for building floor plans, room outlines, simple architectural elements.

```python
# Blender 3.x/4.x/5.x
import bpy

def create_mesh_from_pydata(name, verts, edges, faces):
    """Create a mesh object from vertex/edge/face data.

    Args:
        name: Object and mesh name
        verts: List of (x, y, z) tuples
        edges: List of (v1, v2) tuples. Pass [] if faces are provided.
        faces: List of vertex index tuples, e.g. [(0,1,2,3)]
    Returns:
        The created bpy.types.Object
    """
    mesh = bpy.data.meshes.new(name)
    mesh.from_pydata(verts, edges, faces)
    mesh.update()     # REQUIRED: recalculates normals and internal state
    mesh.validate()   # Catches degenerate geometry

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)  # REQUIRED: makes object visible
    return obj
```

**from_pydata rules:**
- Pass `edges=[]` when providing faces — edges are auto-generated from face definitions.
- Vertex indices in faces MUST be valid indices into the verts list.
- Face winding order determines normal direction (counter-clockwise = outward by Blender convention).
- ALWAYS call `mesh.update()` after `from_pydata()`.

### Pattern 2: High-Performance Mesh Creation (foreach_set)

Use for meshes with > 1000 vertices. Required for IFC geometry visualization and large AEC models. 10-100x faster than per-element access.

```python
# Blender 3.x/4.x/5.x
import bpy
import numpy as np

def create_mesh_fast(name, coords, face_indices, face_sizes=None):
    """Create mesh using pre-allocation and foreach_set.

    Args:
        name: Mesh and object name
        coords: numpy array of shape (N, 3), dtype float32 — vertex positions
        face_indices: flat numpy array of vertex indices for all faces, dtype int32
        face_sizes: numpy array of verts-per-face, dtype int32. Default: all triangles.
    Returns:
        The created bpy.types.Object
    """
    vert_count = len(coords)
    loop_count = len(face_indices)

    if face_sizes is None:
        face_count = loop_count // 3
        face_sizes = np.full(face_count, 3, dtype=np.int32)
    else:
        face_count = len(face_sizes)

    mesh = bpy.data.meshes.new(name)
    mesh.vertices.add(vert_count)
    mesh.loops.add(loop_count)
    mesh.polygons.add(face_count)

    # Set vertex coordinates — flatten to (N*3,) for foreach_set
    mesh.vertices.foreach_set("co", coords.reshape(-1).astype(np.float32))

    # Set loop vertex indices
    mesh.loops.foreach_set("vertex_index", face_indices.astype(np.int32))

    # Set polygon loop_start and loop_total
    loop_starts = np.zeros(face_count, dtype=np.int32)
    loop_starts[1:] = np.cumsum(face_sizes[:-1])
    mesh.polygons.foreach_set("loop_start", loop_starts)
    mesh.polygons.foreach_set("loop_total", face_sizes)

    mesh.update()
    mesh.validate()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj
```

**foreach_set rules:**
- Input array MUST be flat (1D). For coordinates: shape `(N*3,)`, not `(N, 3)`.
- dtype MUST match: `float32` for coordinates/normals, `int32` for indices.
- Array length MUST exactly match `count * attribute_dimension`. Mismatches cause crashes.
- `loop_start` values MUST be cumulative sums of `loop_total` values.
- Call `mesh.update()` after all `foreach_set` calls.

### Pattern 3: BMesh for Topology Operations

Use BMesh when you need extrude, inset, dissolve, merge, or other topology-modifying operations. Required for parametric building element generation.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh

def bmesh_create_and_extrude(name, base_verts, base_face, extrude_height):
    """Create a mesh from a base polygon and extrude upward.

    Args:
        name: Object name
        base_verts: List of (x, y, z) for the base polygon
        base_face: Tuple of vertex indices for the base face
        extrude_height: Height to extrude along Z
    Returns:
        The created bpy.types.Object
    """
    bm = bmesh.new()

    # Create base vertices
    bm_verts = [bm.verts.new(co) for co in base_verts]
    bm.verts.ensure_lookup_table()

    # Create base face
    base = bm.faces.new([bm_verts[i] for i in base_face])

    # Extrude face upward
    result = bmesh.ops.extrude_face_region(bm, geom=[base])
    extruded_verts = [e for e in result['geom'] if isinstance(e, bmesh.types.BMVert)]
    bmesh.ops.translate(bm, vec=(0, 0, extrude_height), verts=extruded_verts)

    # Write to mesh
    mesh = bpy.data.meshes.new(name)
    bm.to_mesh(mesh)
    bm.free()  # ALWAYS free

    mesh.update()
    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj
```

**BMesh rules:**
- ALWAYS call `bm.verts.ensure_lookup_table()` (and edges/faces) before index access.
- ALWAYS call `bm.free()` when using `bmesh.new()` or `bm.from_mesh()`.
- NEVER call `bm.free()` when using `bmesh.from_edit_mesh()` — use `bmesh.update_edit_mesh()` instead.
- After `bmesh.ops.*` calls, element references may be invalidated. Re-query from results.
- BMesh element `.index` values are NOT stable across topology operations. Use `ensure_lookup_table()` after modifications.

### Pattern 4: Read Evaluated (Post-Modifier) Mesh

Use to read mesh data after all modifiers are applied. Required for mesh analysis and export workflows.

```python
# Blender 3.x/4.x/5.x
import bpy

def get_evaluated_mesh_data(obj):
    """Get vertex and face data from evaluated (post-modifier) mesh.

    Args:
        obj: bpy.types.Object with mesh data
    Returns:
        Tuple of (vertices_list, faces_list)
    """
    depsgraph = bpy.context.evaluated_depsgraph_get()
    obj_eval = obj.evaluated_get(depsgraph)
    mesh_eval = obj_eval.to_mesh()

    verts = [v.co.copy() for v in mesh_eval.vertices]
    faces = [list(p.vertices) for p in mesh_eval.polygons]

    obj_eval.to_mesh_clear()  # ALWAYS clean up
    return verts, faces
```

### Pattern 5: IFC Geometry Visualization in Blender

Use to display IFC model geometry in Blender viewport. Combines IfcOpenShell geometry processing with foreach_set for performance.

```python
# Blender 3.x/4.x/5.x: requires ifcopenshell
import bpy
import numpy as np
import ifcopenshell
import ifcopenshell.geom

def visualize_ifc_element(ifc_file, element, collection=None):
    """Create Blender mesh from IFC element geometry.

    Args:
        ifc_file: ifcopenshell.file object
        element: IFC element with geometry (e.g., IfcWall, IfcSlab)
        collection: Target collection. Default: active collection.
    Returns:
        The created bpy.types.Object, or None if no geometry
    """
    settings = ifcopenshell.geom.settings()
    settings.set("use-world-coords", True)

    try:
        shape = ifcopenshell.geom.create_shape(settings, element)
    except RuntimeError:
        return None  # Element has no geometry representation

    # Extract geometry data
    verts_flat = shape.geometry.verts     # Flat list: [x0,y0,z0, x1,y1,z1, ...]
    faces_flat = shape.geometry.faces     # Flat list: [i0,i1,i2, i3,i4,i5, ...]

    coords = np.array(verts_flat, dtype=np.float32).reshape(-1, 3)
    indices = np.array(faces_flat, dtype=np.int32)

    vert_count = len(coords)
    tri_count = len(indices) // 3

    mesh = bpy.data.meshes.new(element.Name or f"IFC_{element.id()}")
    mesh.vertices.add(vert_count)
    mesh.loops.add(tri_count * 3)
    mesh.polygons.add(tri_count)

    mesh.vertices.foreach_set("co", coords.flatten())
    mesh.loops.foreach_set("vertex_index", indices)

    loop_starts = np.arange(0, tri_count * 3, 3, dtype=np.int32)
    loop_totals = np.full(tri_count, 3, dtype=np.int32)
    mesh.polygons.foreach_set("loop_start", loop_starts)
    mesh.polygons.foreach_set("loop_total", loop_totals)

    mesh.update()
    mesh.validate()

    obj = bpy.data.objects.new(element.Name or f"IFC_{element.id()}", mesh)
    target = collection or bpy.context.collection
    target.objects.link(obj)
    return obj
```

---

## Common Operations

See [references/examples.md](references/examples.md) for complete working code examples including:

- Building floor plans from 2D coordinates with BMesh extrusion
- Mesh analysis (area, volume, bounding box) for quantity takeoff
- Bulk vertex read/write with foreach_get/foreach_set and numpy
- Custom float attributes (Blender 3.2+/4.x/5.x)
- Material assignment to specific faces
- Parametric column generation
- Window openings via BMesh inset
- Planar UV mapping

---

## Version-Specific Notes

| Feature | Blender 3.x | Blender 4.0+ |
|---------|-------------|--------------|
| `mesh.calc_normals()` | Available | **REMOVED** — normals auto-calculated |
| `edge.bevel_weight` | Direct property | Use `mesh.attributes["bevel_weight_edge"]` |
| `edge.crease` | Direct property | Use `mesh.attributes["crease_edge"]` |
| Face Maps | `obj.face_maps` | **REMOVED** — use integer face attributes |
| Custom attributes | `mesh.vertex_colors`, `mesh.uv_layers` (legacy) | `mesh.attributes` (preferred), legacy still works |
| Mesh boolean | `bpy.ops.mesh.intersect_boolean()` | Same API, improved solver |

See [references/methods.md](references/methods.md) for Blender 4.0+ attribute migration code examples.

---

## Performance Guidelines

| Scenario | Method | Relative Speed |
|----------|--------|---------------|
| Read < 1000 verts | `for v in mesh.vertices` | 1x (baseline) |
| Read > 1000 verts | `foreach_get("co", array)` | 10-100x |
| Write vertices | `foreach_set("co", array)` | 10-100x |
| Create simple mesh | `mesh.from_pydata()` | Good |
| Create large mesh (> 10K verts) | Pre-allocate + `foreach_set` | Fastest |
| Topology changes | BMesh | Required |
| Interactive edits | `bmesh.from_edit_mesh()` | Best for Edit Mode |

For AEC models exceeding 10,000 vertices, ALWAYS use `foreach_get`/`foreach_set` with numpy arrays. Per-element Python loops on large meshes cause unacceptable latency in Blender's main thread.

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for Mesh, BMesh, and foreach operations
- [references/examples.md](references/examples.md) — Working code examples for AEC mesh workflows
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with mesh operations

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Mesh.html
- https://docs.blender.org/api/current/bmesh.html
- https://docs.blender.org/api/current/bmesh.types.html
- https://docs.blender.org/api/current/bmesh.ops.html
- https://docs.blender.org/api/current/bpy.types.MeshVertex.html
- https://docs.blender.org/api/current/bpy.types.MeshPolygon.html
- https://docs.blender.org/api/current/bpy.types.MeshLoop.html
