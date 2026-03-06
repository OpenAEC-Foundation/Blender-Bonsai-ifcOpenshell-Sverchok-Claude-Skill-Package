# blender-syntax-mesh — Anti-Patterns

What NOT to do when working with Blender mesh data. Each anti-pattern includes the wrong code, why it fails, and the correct alternative.

---

## Anti-Pattern 1: Accessing Mesh Data in Edit Mode

**What goes wrong:** Reading `mesh.vertices`, `mesh.polygons`, etc. while in Edit Mode returns stale data or crashes.

```python
# WRONG — data is not synchronized in Edit Mode
bpy.ops.object.mode_set(mode='EDIT')
mesh = bpy.context.active_object.data
for v in mesh.vertices:
    print(v.co)  # STALE DATA — may crash or return outdated values
```

```python
# CORRECT — exit Edit Mode first
bpy.ops.object.mode_set(mode='OBJECT')
mesh = bpy.context.active_object.data
for v in mesh.vertices:
    print(v.co)  # Safe

# OR — use BMesh in Edit Mode
import bmesh
bm = bmesh.from_edit_mesh(bpy.context.edit_object.data)
for v in bm.verts:
    print(v.co)  # Safe
```

**Why:** Blender's mesh data is managed by the edit-mode system. The `bpy.types.Mesh` arrays are only updated when exiting edit mode.

---

## Anti-Pattern 2: Forgetting mesh.update() After from_pydata

**What goes wrong:** Normals are incorrect, display artifacts, missing edges.

```python
# WRONG
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata(verts, edges, faces)
# Missing mesh.update()!
obj = bpy.data.objects.new("Obj", mesh)
bpy.context.collection.objects.link(obj)
# Mesh may render with inverted or missing normals
```

```python
# CORRECT
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata(verts, edges, faces)
mesh.update()  # ALWAYS call after from_pydata
obj = bpy.data.objects.new("Obj", mesh)
bpy.context.collection.objects.link(obj)
```

**Why:** `from_pydata` populates raw arrays but does not compute normals, edges, or tessellation. `mesh.update()` triggers these computations.

---

## Anti-Pattern 3: Not Calling ensure_lookup_table on BMesh

**What goes wrong:** IndexError or undefined behavior when accessing BMesh elements by integer index.

```python
# WRONG
import bmesh
bm = bmesh.new()
bm.from_mesh(mesh)
v = bm.verts[0]  # IndexError or undefined behavior!
```

```python
# CORRECT
bm = bmesh.new()
bm.from_mesh(mesh)
bm.verts.ensure_lookup_table()  # REQUIRED before index access
v = bm.verts[0]  # Safe
```

**Why:** BMesh uses an internal linked list. Integer indexing requires building a lookup table, which is not done automatically for performance reasons.

**Note:** Iterating (`for v in bm.verts:`) does NOT require `ensure_lookup_table()` — only integer index access does.

---

## Anti-Pattern 4: Not Freeing BMesh (Memory Leak)

**What goes wrong:** Memory leaks. In long-running scripts or modal operators, eventually crashes Blender.

```python
# WRONG — memory leak
bm = bmesh.new()
bm.from_mesh(mesh)
# ... do work ...
bm.to_mesh(mesh)
# Missing bm.free()!
```

```python
# CORRECT
bm = bmesh.new()
bm.from_mesh(mesh)
# ... do work ...
bm.to_mesh(mesh)
bm.free()  # ALWAYS free BMesh created with bmesh.new()
```

**Exception:** BMesh from `bmesh.from_edit_mesh()` must NOT be freed — it is owned by Blender's edit mode system. Call `bmesh.update_edit_mesh()` instead.

---

## Anti-Pattern 5: Freeing Edit-Mode BMesh

**What goes wrong:** Crashes Blender.

```python
# WRONG — crashes!
bm = bmesh.from_edit_mesh(obj.data)
# ... edit ...
bm.free()  # DO NOT free edit-mode BMesh!
```

```python
# CORRECT
bm = bmesh.from_edit_mesh(obj.data)
# ... edit ...
bmesh.update_edit_mesh(obj.data)  # Sync changes, do not free
```

**Why:** The BMesh returned by `from_edit_mesh()` is owned by Blender's internal edit-mode state. Freeing it corrupts Blender's memory.

---

## Anti-Pattern 6: Using calc_normals() in Blender 4.0+

**What goes wrong:** `AttributeError` — method was removed.

```python
# WRONG — removed in Blender 4.0
mesh.calc_normals()  # AttributeError
```

```python
# CORRECT — normals are computed automatically
mesh.update()  # This is sufficient
# Access normals via:
for v in mesh.vertices:
    print(v.normal)
for p in mesh.polygons:
    print(p.normal)
```

**Why:** Blender 4.0 removed `calc_normals()` because normals are now always computed on demand.

---

## Anti-Pattern 7: Using use_auto_smooth in Blender 4.1+

**What goes wrong:** `AttributeError` — property was removed.

```python
# WRONG — removed in Blender 4.1
mesh.use_auto_smooth = True
mesh.auto_smooth_angle = 0.523599  # 30 degrees
```

```python
# CORRECT (Blender 4.1+) — auto-smooth is automatic
# Custom normals are applied when:
# 1. Custom split normals exist (set via normals_split_custom_set)
# 2. Sharp edges/faces are marked (via "sharp_edge"/"sharp_face" attributes)
# No property toggle needed.

# Version-safe pattern:
if bpy.app.version < (4, 1, 0):
    mesh.use_auto_smooth = True
    mesh.auto_smooth_angle = 0.523599
# In 4.1+, auto-smooth is always on when needed
```

**Why:** Blender 4.1 removed the toggle because auto-smooth now activates automatically when custom normals or sharp data exists.

---

## Anti-Pattern 8: Using MeshEdge.bevel_weight / .crease in Blender 4.0+

**What goes wrong:** `AttributeError` — properties were removed.

```python
# WRONG — removed in Blender 4.0
for edge in mesh.edges:
    edge.bevel_weight = 0.5  # AttributeError
    edge.crease = 1.0        # AttributeError
```

```python
# CORRECT — use generic attributes (Blender 4.0+)
import numpy as np

# Bevel weight
bevel = mesh.attributes.get("bevel_weight_edge")
if bevel is None:
    bevel = mesh.attributes.new("bevel_weight_edge", 'FLOAT', 'EDGE')
weights = np.full(len(mesh.edges), 0.5, dtype=np.float32)
bevel.data.foreach_set("value", weights)

# Edge crease
crease = mesh.attributes.get("crease_edge")
if crease is None:
    crease = mesh.attributes.new("crease_edge", 'FLOAT', 'EDGE')
creases = np.full(len(mesh.edges), 1.0, dtype=np.float32)
crease.data.foreach_set("value", creases)

mesh.update()
```

**Version-safe pattern:**
```python
if bpy.app.version < (4, 0, 0):
    for edge in mesh.edges:
        edge.bevel_weight = 0.5
else:
    attr = mesh.attributes.get("bevel_weight_edge")
    if attr is None:
        attr = mesh.attributes.new("bevel_weight_edge", 'FLOAT', 'EDGE')
    for i, edge_data in enumerate(attr.data):
        edge_data.value = 0.5
```

---

## Anti-Pattern 9: Wrong foreach_get/set Array Size

**What goes wrong:** Silent data corruption (pre-4.1) or `TypeError` (4.1+).

```python
# WRONG — array too short for 3-component property
import numpy as np
n = len(mesh.vertices)
coords = np.empty(n, dtype=np.float32)  # Should be n * 3!
mesh.vertices.foreach_get("co", coords)  # Silent corruption or TypeError
```

```python
# CORRECT — flat array with correct length
n = len(mesh.vertices)
coords = np.empty(n * 3, dtype=np.float32)  # 3 components per vertex
mesh.vertices.foreach_get("co", coords)
coords = coords.reshape(-1, 3)
```

**Rule:** For multi-component properties, array length = `len(collection) * component_count`:
- `co`: 3 floats → length = `n_verts * 3`
- `uv`: 2 floats → length = `n_loops * 2`
- `color`: 4 floats → length = `n_loops * 4`
- `value`: 1 → length = `n_elements`

---

## Anti-Pattern 10: Wrong dtype for foreach_set

**What goes wrong:** Silent data corruption (pre-4.1) or `TypeError` (4.1+).

```python
# WRONG — float64 instead of float32
import numpy as np
coords = np.random.rand(100 * 3)  # Default dtype is float64
mesh.vertices.foreach_set("co", coords)  # Blender expects float32
```

```python
# CORRECT — explicit float32
coords = np.random.rand(100 * 3).astype(np.float32)
mesh.vertices.foreach_set("co", coords)
```

**Rule:**
- Float properties → `np.float32`
- Integer properties → `np.int32`
- Boolean properties → `bool` or `np.bool_`

---

## Anti-Pattern 11: Not Linking Object to Collection

**What goes wrong:** Object exists in `bpy.data.objects` but is invisible — not in any viewport.

```python
# WRONG — object is orphaned
mesh = bpy.data.meshes.new("Mesh")
obj = bpy.data.objects.new("Object", mesh)
# Object exists but is invisible and will be purged on save
```

```python
# CORRECT — link to collection
mesh = bpy.data.meshes.new("Mesh")
obj = bpy.data.objects.new("Object", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED
```

---

## Anti-Pattern 12: Looking Up Object by Name After Creation

**What goes wrong:** Name collision — Blender auto-appends `.001`, `.002` etc.

```python
# WRONG — name may not match
bpy.data.meshes.new(name="MyMesh")
mesh = bpy.data.meshes["MyMesh"]  # May be "MyMesh.001" if "MyMesh" already existed!
```

```python
# CORRECT — store the reference directly
mesh = bpy.data.meshes.new(name="MyMesh")
# Use 'mesh' variable, never look up by name after creation
```

---

## Anti-Pattern 13: Using calc_normals_split / free_normals_split in 4.1+

**What goes wrong:** `AttributeError` — methods were removed in 4.1.

```python
# WRONG — removed in Blender 4.1
mesh.calc_normals_split()
for loop in mesh.loops:
    print(loop.normal)
mesh.free_normals_split()
```

```python
# CORRECT (Blender 4.1+) — corner_normals is auto-computed
for cn in mesh.corner_normals:
    print(cn.vector)

# Version-safe pattern:
if bpy.app.version >= (4, 1, 0):
    normals = [cn.vector for cn in mesh.corner_normals]
else:
    mesh.calc_normals_split()
    normals = [loop.normal[:] for loop in mesh.loops]
    mesh.free_normals_split()
```

---

## Anti-Pattern 14: Writing to MeshLoop.normal in 4.1+

**What goes wrong:** Read-only property — cannot set directly.

```python
# WRONG — read-only in Blender 4.1+
for loop in mesh.loops:
    loop.normal = (0, 0, 1)  # TypeError: read-only property
```

```python
# CORRECT — use normals_split_custom_set
custom_normals = [(0, 0, 1)] * len(mesh.loops)
mesh.normals_split_custom_set(custom_normals)
mesh.update()
```

---

## Anti-Pattern 15: Using bpy.types.AttributeGroup in 4.3+ Type Checks

**What goes wrong:** `AttributeGroup` no longer exists as a single type in Blender 4.3.

```python
# WRONG — type was split in 4.3
if isinstance(mesh.attributes, bpy.types.AttributeGroup):  # AttributeError in 4.3
    pass
```

```python
# CORRECT (Blender 4.3+) — use geometry-specific type
if isinstance(mesh.attributes, bpy.types.AttributeGroupMesh):
    pass

# Version-safe:
attr_group = mesh.attributes
# Just use it directly — duck typing works fine
attr = attr_group.new("my_attr", 'FLOAT', 'POINT')
```

**Why:** Blender 4.3 split `AttributeGroup` into `AttributeGroupMesh`, `AttributeGroupPointCloud`, `AttributeGroupCurves`, `AttributeGroupGreasePencil`.

---

## Summary: Version-Specific Anti-Pattern Quick Reference

| Anti-Pattern | Affects | Version |
|-------------|---------|---------|
| Access mesh data in Edit Mode | All | All versions |
| Missing `mesh.update()` after `from_pydata()` | All | All versions |
| Missing `ensure_lookup_table()` | All | All versions |
| Missing `bm.free()` | All | All versions |
| Freeing edit-mode BMesh | Crash | All versions |
| `mesh.calc_normals()` | `AttributeError` | 4.0+ |
| `mesh.use_auto_smooth` | `AttributeError` | 4.1+ |
| `MeshEdge.bevel_weight` / `.crease` | `AttributeError` | 4.0+ |
| Wrong `foreach_get/set` array size | Data corruption / `TypeError` | All / 4.1+ strict |
| Wrong dtype for `foreach_set` | Data corruption / `TypeError` | All / 4.1+ strict |
| Not linking object to collection | Invisible object | All versions |
| Name lookup after creation | Wrong object | All versions |
| `calc_normals_split()` / `free_normals_split()` | `AttributeError` | 4.1+ |
| Writing `MeshLoop.normal` | `TypeError` | 4.1+ |
| `bpy.types.AttributeGroup` type check | `AttributeError` | 4.3+ |

---

## Sources

- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/bpy.types.Mesh.html
- https://docs.blender.org/api/current/bmesh.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
