# blender-impl-mesh — Anti-Patterns Reference

Common mistakes and what NOT to do with Blender mesh operations. Each anti-pattern includes the incorrect code, the correct alternative, and the reason.

---

## Anti-Pattern 1: Accessing Mesh Data in Edit Mode

**WRONG** — mesh element data is NOT synchronized during Edit Mode:

```python
# WRONG — Blender 3.x/4.x/5.x
bpy.ops.object.mode_set(mode='EDIT')
mesh = obj.data
for v in mesh.vertices:       # Data is STALE — not updated in Edit Mode
    print(v.co)
```

**CORRECT** — use BMesh to access Edit Mode data:

```python
# CORRECT — Blender 3.x/4.x/5.x
bpy.ops.object.mode_set(mode='EDIT')
bm = bmesh.from_edit_mesh(obj.data)
for v in bm.verts:
    print(v.co)
bmesh.update_edit_mesh(obj.data)  # Do NOT call bm.free()
```

**Why:** Blender does not synchronize `mesh.vertices`/`mesh.edges`/`mesh.polygons` while in Edit Mode. The data returned is from before entering Edit Mode.

---

## Anti-Pattern 2: Forgetting bm.free() After bmesh.new()

**WRONG** — causes memory leak:

```python
# WRONG — Blender 3.x/4.x/5.x
bm = bmesh.new()
bm.from_mesh(obj.data)
area = sum(f.calc_area() for f in bm.faces)
# Missing bm.free() — BMesh remains in memory
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x
bm = bmesh.new()
bm.from_mesh(obj.data)
area = sum(f.calc_area() for f in bm.faces)
bm.free()  # ALWAYS free when using bmesh.new() or bm.from_mesh()
```

**Why:** `bmesh.new()` and `bm.from_mesh()` allocate memory that is NOT garbage-collected by Python. Each leaked BMesh persists until Blender restarts.

**Exception:** When using `bmesh.from_edit_mesh()`, do NOT call `bm.free()`. Use `bmesh.update_edit_mesh()` instead.

---

## Anti-Pattern 3: Calling bm.free() on Edit Mode BMesh

**WRONG** — corrupts Edit Mode state:

```python
# WRONG — Blender 3.x/4.x/5.x
bm = bmesh.from_edit_mesh(obj.data)
# ... modifications ...
bm.free()  # WRONG — frees the Edit Mode BMesh, causing data corruption
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x
bm = bmesh.from_edit_mesh(obj.data)
# ... modifications ...
bmesh.update_edit_mesh(obj.data)  # Updates the mesh display, no free needed
```

**Why:** `bmesh.from_edit_mesh()` returns Blender's internal Edit Mode BMesh. Freeing it destroys Blender's own reference, causing crashes or data loss.

---

## Anti-Pattern 4: Skipping mesh.update() After from_pydata()

**WRONG** — normals and internal state remain invalid:

```python
# WRONG — Blender 3.x/4.x/5.x
mesh = bpy.data.meshes.new("MyMesh")
mesh.from_pydata(verts, [], faces)
# Missing mesh.update() — normals wrong, edge cache stale
obj = bpy.data.objects.new("MyObj", mesh)
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x
mesh = bpy.data.meshes.new("MyMesh")
mesh.from_pydata(verts, [], faces)
mesh.update()     # Recalculates normals, edge cache, loose edges
mesh.validate()   # Catches degenerate geometry
obj = bpy.data.objects.new("MyObj", mesh)
```

**Why:** `from_pydata()` only sets raw vertex/face data. Without `mesh.update()`, derived data (normals, edge cache) remains in an invalid state. Shading, collisions, and modifiers will produce incorrect results.

---

## Anti-Pattern 5: Index Access Without ensure_lookup_table()

**WRONG** — raises IndexError:

```python
# WRONG — Blender 3.x/4.x/5.x
bm = bmesh.new()
v1 = bm.verts.new((0, 0, 0))
v2 = bm.verts.new((1, 0, 0))
print(bm.verts[0])  # IndexError — lookup table not built
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x
bm = bmesh.new()
v1 = bm.verts.new((0, 0, 0))
v2 = bm.verts.new((1, 0, 0))
bm.verts.ensure_lookup_table()  # Build index → element mapping
print(bm.verts[0])              # Works
```

**Why:** BMesh does not maintain an index-based lookup table by default. After adding or removing elements, the table must be explicitly rebuilt. Iteration (`for v in bm.verts`) works without it; only `bm.verts[i]` requires it.

---

## Anti-Pattern 6: Using calc_normals() in Blender 4.0+

**WRONG** — method removed in Blender 4.0:

```python
# WRONG — Blender 4.0+
mesh.from_pydata(verts, [], faces)
mesh.calc_normals()  # AttributeError — REMOVED in Blender 4.0
mesh.update()
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x (universal)
mesh.from_pydata(verts, [], faces)
mesh.update()  # Normals are auto-calculated in Blender 4.0+
```

**Why:** `mesh.calc_normals()` was removed in Blender 4.0. Normals are now automatically calculated by `mesh.update()` and other internal operations. Calling it raises `AttributeError`.

---

## Anti-Pattern 7: Forgetting to_mesh_clear() After Evaluated Mesh

**WRONG** — causes memory leak:

```python
# WRONG — Blender 3.x/4.x/5.x
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
verts = [v.co.copy() for v in mesh_eval.vertices]
# Missing obj_eval.to_mesh_clear() — temporary mesh leaks memory
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
verts = [v.co.copy() for v in mesh_eval.vertices]
obj_eval.to_mesh_clear()  # ALWAYS clean up evaluated mesh
```

**Why:** `obj_eval.to_mesh()` creates a temporary mesh that is NOT automatically freed. Each call without `to_mesh_clear()` leaks memory.

---

## Anti-Pattern 8: Per-Element Loops on Large Meshes

**WRONG** — causes unacceptable latency on large meshes:

```python
# WRONG — Blender 3.x/4.x/5.x (slow for > 1000 vertices)
coords = []
for v in mesh.vertices:    # Python loop — O(n) with high per-element overhead
    coords.append(v.co.copy())
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x (10-100x faster)
import numpy as np
count = len(mesh.vertices)
coords = np.empty(count * 3, dtype=np.float64)
mesh.vertices.foreach_get("co", coords)
coords = coords.reshape(-1, 3)
```

**Why:** Each `mesh.vertices[i]` access in Python crosses the C/Python boundary. `foreach_get`/`foreach_set` performs bulk transfer in a single C call, avoiding per-element overhead. For meshes with > 1000 vertices, the difference is 10-100x.

---

## Anti-Pattern 9: Wrong Array Shape for foreach_set

**WRONG** — causes crash or silent data corruption:

```python
# WRONG — Blender 3.x/4.x/5.x
import numpy as np
coords = np.array([(0,0,0), (1,0,0), (0,1,0)], dtype=np.float32)
mesh.vertices.foreach_set("co", coords)  # WRONG — shape is (3, 3), must be (9,)
```

**CORRECT:**

```python
# CORRECT — Blender 3.x/4.x/5.x
import numpy as np
coords = np.array([(0,0,0), (1,0,0), (0,1,0)], dtype=np.float32)
mesh.vertices.foreach_set("co", coords.reshape(-1))  # Flatten to (9,)
```

**Why:** `foreach_set` expects a flat (1D) array. For vertex coordinates, the array length must be exactly `len(mesh.vertices) * 3`. Passing a 2D array causes undefined behavior: Blender may crash, silently corrupt data, or raise a cryptic error.

---

## Anti-Pattern 10: Using Legacy Attributes in Blender 4.0+

**WRONG** — direct edge properties removed in Blender 4.0:

```python
# WRONG — Blender 4.0+
for edge in mesh.edges:
    edge.bevel_weight = 1.0  # AttributeError — REMOVED in Blender 4.0
    edge.crease = 0.5        # AttributeError — REMOVED in Blender 4.0
```

**CORRECT:**

```python
# CORRECT — Blender 4.0+/5.x
if "bevel_weight_edge" not in mesh.attributes:
    mesh.attributes.new(name="bevel_weight_edge", type='FLOAT', domain='EDGE')
attr = mesh.attributes["bevel_weight_edge"]
for i in range(len(mesh.edges)):
    attr.data[i].value = 1.0

if "crease_edge" not in mesh.attributes:
    mesh.attributes.new(name="crease_edge", type='FLOAT', domain='EDGE')
attr = mesh.attributes["crease_edge"]
for i in range(len(mesh.edges)):
    attr.data[i].value = 0.5
```

**Why:** Blender 4.0 moved `bevel_weight`, `crease`, and Face Maps to the generic attributes system. Direct edge properties were removed. Use `mesh.attributes` for version-compatible code.

---

## Summary: Quick Checklist

| Check | Rule |
|-------|------|
| Edit Mode data access | Use `bmesh.from_edit_mesh()`, never `mesh.vertices` |
| `bmesh.new()` / `bm.from_mesh()` | ALWAYS call `bm.free()` when done |
| `bmesh.from_edit_mesh()` | NEVER call `bm.free()` — use `bmesh.update_edit_mesh()` |
| After `mesh.from_pydata()` | ALWAYS call `mesh.update()` then `mesh.validate()` |
| BMesh index access `bm.verts[i]` | ALWAYS call `ensure_lookup_table()` first |
| `mesh.calc_normals()` | REMOVED in Blender 4.0 — normals are auto-calculated |
| `obj_eval.to_mesh()` | ALWAYS call `obj_eval.to_mesh_clear()` after |
| Large meshes (> 1000 verts) | Use `foreach_get`/`foreach_set`, never per-element loops |
| `foreach_set` arrays | MUST be flat (1D) with correct dtype and length |
| Edge bevel_weight/crease | Use `mesh.attributes` in Blender 4.0+ |
