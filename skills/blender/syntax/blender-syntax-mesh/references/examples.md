# blender-syntax-mesh — Code Examples

Working code examples verified against Blender 4.x API documentation. Each example is self-contained and annotated with version compatibility.

---

## Example 1: Create Mesh with from_pydata

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy

# Create a simple quad
mesh = bpy.data.meshes.new("Quad")
verts = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
edges = []  # Auto-generate from faces
faces = [(0, 1, 2, 3)]

mesh.from_pydata(verts, edges, faces)
mesh.update()  # REQUIRED — recalculates normals and derived data

obj = bpy.data.objects.new("Quad", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED — makes object visible
```

---

## Example 2: Create Mesh with Edges Only (Wireframe)

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy

mesh = bpy.data.meshes.new("Wireframe")
verts = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
edges = [(0, 1), (1, 2), (2, 3), (3, 0)]
faces = []  # No faces — edges only

mesh.from_pydata(verts, edges, faces)
mesh.update()

obj = bpy.data.objects.new("Wireframe", mesh)
bpy.context.collection.objects.link(obj)
```

---

## Example 3: High-Performance Mesh Creation with foreach_set

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import numpy as np

# Generate a grid of triangles
grid_size = 100
n_verts = (grid_size + 1) ** 2
n_tris = grid_size * grid_size * 2

# Create vertex coordinates
x = np.repeat(np.arange(grid_size + 1, dtype=np.float32), grid_size + 1)
y = np.tile(np.arange(grid_size + 1, dtype=np.float32), grid_size + 1)
z = np.zeros(n_verts, dtype=np.float32)
coords = np.column_stack((x, y, z)).ravel()  # Flat array: [x0, y0, z0, x1, y1, z1, ...]

# Create triangle indices
row_len = grid_size + 1
loop_verts = []
for gy in range(grid_size):
    for gx in range(grid_size):
        i = gy * row_len + gx
        # Triangle 1
        loop_verts.extend([i, i + 1, i + row_len + 1])
        # Triangle 2
        loop_verts.extend([i, i + row_len + 1, i + row_len])
loop_verts = np.array(loop_verts, dtype=np.int32)

loop_starts = np.arange(0, n_tris * 3, 3, dtype=np.int32)
loop_totals = np.full(n_tris, 3, dtype=np.int32)

# Build mesh
mesh = bpy.data.meshes.new("Grid")
mesh.vertices.add(n_verts)
mesh.loops.add(n_tris * 3)
mesh.polygons.add(n_tris)

mesh.vertices.foreach_set("co", coords)
mesh.loops.foreach_set("vertex_index", loop_verts)
mesh.polygons.foreach_set("loop_start", loop_starts)
mesh.polygons.foreach_set("loop_total", loop_totals)

mesh.update(calc_edges=True)  # calc_edges=True because edges were not set explicitly
mesh.validate()

obj = bpy.data.objects.new("Grid", mesh)
bpy.context.collection.objects.link(obj)
```

---

## Example 4: Read Mesh Data with foreach_get

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import numpy as np

mesh = bpy.context.active_object.data

# Read vertex positions
n_verts = len(mesh.vertices)
coords = np.empty(n_verts * 3, dtype=np.float32)
mesh.vertices.foreach_get("co", coords)
coords = coords.reshape(-1, 3)
print(f"Vertex count: {n_verts}, bounds: {coords.min(axis=0)} to {coords.max(axis=0)}")

# Read face normals
n_polys = len(mesh.polygons)
normals = np.empty(n_polys * 3, dtype=np.float32)
mesh.polygons.foreach_get("normal", normals)
normals = normals.reshape(-1, 3)

# Read material indices
mat_indices = np.empty(n_polys, dtype=np.int32)
mesh.polygons.foreach_get("material_index", mat_indices)

# Read UV coordinates (if available)
if mesh.uv_layers.active:
    n_loops = len(mesh.loops)
    uvs = np.empty(n_loops * 2, dtype=np.float32)
    mesh.uv_layers.active.data.foreach_get("uv", uvs)
    uvs = uvs.reshape(-1, 2)
```

---

## Example 5: Modify Vertex Positions with foreach_set

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import numpy as np

obj = bpy.context.active_object
mesh = obj.data
n = len(mesh.vertices)

# Read current positions
coords = np.empty(n * 3, dtype=np.float32)
mesh.vertices.foreach_get("co", coords)
coords = coords.reshape(-1, 3)

# Apply sine wave displacement on Z axis
coords[:, 2] = np.sin(coords[:, 0] * 3.0) * 0.5

# Write back
mesh.vertices.foreach_set("co", coords.ravel())
mesh.update()
```

---

## Example 6: BMesh — Object Mode Workflow

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

obj = bpy.context.active_object
mesh = obj.data

# Create BMesh from existing mesh
bm = bmesh.new()
bm.from_mesh(mesh)

# REQUIRED before index access
bm.verts.ensure_lookup_table()
bm.edges.ensure_lookup_table()
bm.faces.ensure_lookup_table()

# Read geometry info
print(f"Verts: {len(bm.verts)}, Edges: {len(bm.edges)}, Faces: {len(bm.faces)}")

# Extrude all faces
result = bmesh.ops.extrude_face_region(bm, geom=bm.faces[:])
extruded_verts = [e for e in result['geom'] if isinstance(e, bmesh.types.BMVert)]
bmesh.ops.translate(bm, vec=(0, 0, 1.0), verts=extruded_verts)

# Write back and clean up
bm.to_mesh(mesh)
mesh.update()
bm.free()  # ALWAYS free in Object Mode
```

---

## Example 7: BMesh — Edit Mode Workflow

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

# Ensure we are in edit mode
obj = bpy.context.edit_object
if obj is None:
    raise RuntimeError("Must be in Edit Mode")

bm = bmesh.from_edit_mesh(obj.data)

# Add a vertex and connect to existing geometry
bm.verts.ensure_lookup_table()
new_vert = bm.verts.new((2, 2, 0))

# Merge by distance (remove doubles)
bmesh.ops.remove_doubles(bm, verts=bm.verts[:], dist=0.001)

# Sync back — do NOT call bm.free()
bmesh.update_edit_mesh(obj.data, loop_triangles=True, destructive=True)
```

---

## Example 8: BMesh — Create Geometry from Scratch

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

bm = bmesh.new()

# Create a pyramid
base = [
    bm.verts.new((-1, -1, 0)),
    bm.verts.new(( 1, -1, 0)),
    bm.verts.new(( 1,  1, 0)),
    bm.verts.new((-1,  1, 0)),
]
apex = bm.verts.new((0, 0, 1.5))

# Base face
bm.faces.new(base)

# Side faces (winding order matters for normals)
for i in range(4):
    bm.faces.new([base[i], base[(i + 1) % 4], apex])

bm.normal_update()

# Write to new mesh
mesh = bpy.data.meshes.new("Pyramid")
bm.to_mesh(mesh)
mesh.update()
bm.free()

obj = bpy.data.objects.new("Pyramid", mesh)
bpy.context.collection.objects.link(obj)
```

---

## Example 9: UV Layer Creation and Assignment

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy

obj = bpy.context.active_object
mesh = obj.data

# Create UV layer
if not mesh.uv_layers:
    uv_layer = mesh.uv_layers.new(name="UVMap")
else:
    uv_layer = mesh.uv_layers.active

# Assign planar UV projection (XY plane)
for poly in mesh.polygons:
    for loop_idx in poly.loop_indices:
        vert_idx = mesh.loops[loop_idx].vertex_index
        co = mesh.vertices[vert_idx].co
        uv_layer.data[loop_idx].uv = (co.x, co.y)
```

---

## Example 10: UV Layer with BMesh

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

obj = bpy.context.active_object
bm = bmesh.new()
bm.from_mesh(obj.data)

# Get or create UV layer
uv_lay = bm.loops.layers.uv.verify()  # Gets active or creates new

# Set UVs per face corner
for face in bm.faces:
    for loop in face.loops:
        loop[uv_lay].uv = (loop.vert.co.x, loop.vert.co.y)

bm.to_mesh(obj.data)
obj.data.update()
bm.free()
```

---

## Example 11: Color Attributes (Blender 4.0+)

**Versions:** Blender 4.0+

```python
import bpy
import numpy as np

obj = bpy.context.active_object
mesh = obj.data

# Create vertex color attribute (RGBA float, per face corner)
if mesh.color_attributes.get("Color") is None:
    mesh.color_attributes.new("Color", type='FLOAT_COLOR', domain='CORNER')

attr = mesh.color_attributes["Color"]

# Set colors — one RGBA per loop
n_loops = len(mesh.loops)
colors = np.zeros(n_loops * 4, dtype=np.float32)
colors[0::4] = 1.0  # Red channel = 1.0
colors[3::4] = 1.0  # Alpha = 1.0
attr.data.foreach_set("color", colors)

mesh.update()
```

---

## Example 12: Generic Attributes (Blender 4.0+)

**Versions:** Blender 4.0+

```python
import bpy
import numpy as np

mesh = bpy.context.active_object.data

# Create per-vertex float attribute
attr = mesh.attributes.new("height", type='FLOAT', domain='POINT')

# Assign values based on Z coordinate
n = len(mesh.vertices)
coords = np.empty(n * 3, dtype=np.float32)
mesh.vertices.foreach_get("co", coords)
z_values = coords[2::3]  # Every 3rd element starting at index 2

attr.data.foreach_set("value", z_values.astype(np.float32))

# Create per-edge boolean attribute
sharp = mesh.attributes.new("my_sharp", type='BOOLEAN', domain='EDGE')
values = np.zeros(len(mesh.edges), dtype=bool)
values[::2] = True  # Mark every other edge
sharp.data.foreach_set("value", values)

mesh.update()
```

---

## Example 13: Access Built-in Attributes (Blender 4.0+ Replacement for Removed Properties)

**Versions:** Blender 4.0+

```python
import bpy
import numpy as np

mesh = bpy.context.active_object.data

# Bevel weight on edges (replaces MeshEdge.bevel_weight removed in 4.0)
bevel = mesh.attributes.get("bevel_weight_edge")
if bevel is None:
    bevel = mesh.attributes.new("bevel_weight_edge", 'FLOAT', 'EDGE')

# Set all edges to bevel weight 0.5
n = len(mesh.edges)
weights = np.full(n, 0.5, dtype=np.float32)
bevel.data.foreach_set("value", weights)

# Edge crease (replaces MeshEdge.crease removed in 4.0)
crease = mesh.attributes.get("crease_edge")
if crease is None:
    crease = mesh.attributes.new("crease_edge", 'FLOAT', 'EDGE')

creases = np.full(n, 1.0, dtype=np.float32)
crease.data.foreach_set("value", creases)

mesh.update()
```

---

## Example 14: Custom Normals

**Versions:** Version-dependent (see comments)

```python
import bpy

obj = bpy.context.active_object
mesh = obj.data

# Blender 3.x: must enable auto-smooth before setting custom normals
if bpy.app.version < (4, 1, 0):
    mesh.use_auto_smooth = True

# Set all loop normals pointing straight up
custom_normals = [(0.0, 0.0, 1.0)] * len(mesh.loops)
mesh.normals_split_custom_set(custom_normals)
mesh.update()

# Read normals
if bpy.app.version >= (4, 1, 0):
    # 4.1+: corner_normals is auto-computed, always available
    for cn in mesh.corner_normals:
        print(cn.vector)
else:
    # 3.x / 4.0: must calc split normals first
    mesh.calc_normals_split()
    for loop in mesh.loops:
        print(loop.normal)
    mesh.free_normals_split()
```

---

## Example 15: Evaluated Mesh (After Modifiers)

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy

obj = bpy.context.active_object
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()

print(f"Original vertices: {len(obj.data.vertices)}")
print(f"Evaluated vertices: {len(mesh_eval.vertices)}")

# Read evaluated data
for v in mesh_eval.vertices:
    print(f"  {v.co}")

# ALWAYS clean up
obj_eval.to_mesh_clear()
```

---

## Example 16: BMesh from Evaluated Object

**Versions:** Blender 4.0+ (depsgraph parameter required)

```python
import bpy
import bmesh

obj = bpy.context.active_object
depsgraph = bpy.context.evaluated_depsgraph_get()

bm = bmesh.new()
bm.from_object(obj, depsgraph)  # Depsgraph REQUIRED in 4.0+

bm.verts.ensure_lookup_table()
print(f"Evaluated vertex count: {len(bm.verts)}")

bm.free()
```

---

## Example 17: BMesh Operations — Triangulate

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

obj = bpy.context.active_object
bm = bmesh.new()
bm.from_mesh(obj.data)

result = bmesh.ops.triangulate(bm, faces=bm.faces[:])
print(f"Triangulated: {len(result['faces'])} triangles created")

bm.to_mesh(obj.data)
obj.data.update()
bm.free()
```

---

## Example 18: BMesh Operations — Subdivide

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

obj = bpy.context.active_object
bm = bmesh.new()
bm.from_mesh(obj.data)

bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=2, use_grid_fill=True)

bm.to_mesh(obj.data)
obj.data.update()
bm.free()
```

---

## Example 19: BMesh Vertex Deform Weights

**Versions:** Blender 3.x / 4.x / 5.x

```python
import bpy
import bmesh

obj = bpy.context.active_object

# Ensure vertex group exists
if "MyGroup" not in obj.vertex_groups:
    vg = obj.vertex_groups.new(name="MyGroup")
else:
    vg = obj.vertex_groups["MyGroup"]

group_index = vg.index

bm = bmesh.new()
bm.from_mesh(obj.data)
bm.verts.ensure_lookup_table()

deform_lay = bm.verts.layers.deform.verify()

# Set weights based on Z height
for vert in bm.verts:
    vert[deform_lay][group_index] = max(0.0, vert.co.z)

bm.to_mesh(obj.data)
obj.data.update()
bm.free()
```

---

## Example 20: Batch Import — CityJSON Buildings to Blender

**Versions:** Blender 4.x+ (AEC use case from OpenAEC)

```python
import bpy

def import_buildings(buildings, collection_name="Buildings"):
    """Import building meshes from parsed CityJSON data.

    Args:
        buildings: list of dicts with 'id', 'vertices' (list of (x,y,z)), 'faces' (list of index tuples)
        collection_name: name of target collection
    """
    collection = bpy.data.collections.new(collection_name)
    bpy.context.scene.collection.children.link(collection)

    imported = 0
    skipped = 0
    for building in buildings:
        try:
            bid = building.get("id", f"building_{imported}")
            verts = building["vertices"]
            faces = building["faces"]

            mesh = bpy.data.meshes.new(bid)
            mesh.from_pydata(verts, [], faces)
            mesh.update()
            mesh.validate(verbose=False)

            obj = bpy.data.objects.new(bid, mesh)
            collection.objects.link(obj)
            imported += 1
        except Exception as e:
            skipped += 1
            print(f"Skipped building {building.get('id', '?')}: {e}")

    print(f"Imported: {imported}, Skipped: {skipped}")
    return imported, skipped
```

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Mesh.html
- https://docs.blender.org/api/current/bmesh.html
- https://docs.blender.org/api/current/bmesh.types.html
- https://docs.blender.org/api/current/bmesh.ops.html
- https://docs.blender.org/api/current/info_best_practice.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
