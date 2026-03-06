# Validation Rules — Detection Patterns

Complete catalog of validation rules with grep-able code patterns for each check in the validator.

---

## Rule V-001: Target Version Declaration

**Check ID**: CHECK 1
**Severity**: WARNING

**Detection patterns** (at least one MUST be present):
```python
# Pattern 1: bl_info dict
bl_info = {
    "blender": (X, Y, Z),
    ...
}

# Pattern 2: blender_manifest.toml (file exists in project)
[project]
blender_version_min = "4.2.0"

# Pattern 3: Runtime version check
if bpy.app.version >= (4, 0, 0):
    ...

# Pattern 4: Version constant
BLENDER_MIN_VERSION = (4, 0, 0)
```

**Grep patterns**:
```
bl_info\s*=\s*\{
blender_manifest\.toml
bpy\.app\.version
BLENDER.*VERSION
```

---

## Rule V-002: Restricted Context — draw() Mutation

**Check ID**: CHECK 2.1
**Severity**: BLOCKER

**Detection**: Find all classes inheriting from Panel/Header/Menu/UIList/Gizmo. Inside their `draw()`, `draw_header()`, or `draw_header_preset()` methods, flag any:

```python
# Forbidden patterns inside draw():
bpy.ops.*                              # Any operator call
bpy.data.*.new(...)                    # Data creation
bpy.data.*.remove(...)                 # Data removal
context.scene.property = value         # Property mutation
context.object.location = ...         # Data mutation
```

**Grep patterns**:
```
class\s+\w+\(bpy\.types\.(Panel|Header|Menu|UIList|Gizmo)\)
def draw\(self,\s*context\)
bpy\.ops\.
```

---

## Rule V-003: Restricted Context — Handler/Timer Mutation

**Check ID**: CHECK 2.1
**Severity**: BLOCKER

**Detection**: Find all functions appended to `bpy.app.handlers.*` or registered via `bpy.app.timers.register()`. Flag `bpy.ops.*` calls inside those functions.

```python
# Identify handler registrations
bpy.app.handlers.frame_change_pre.append(my_func)
bpy.app.handlers.depsgraph_update_post.append(my_func)
bpy.app.handlers.load_post.append(my_func)
bpy.app.handlers.render_pre.append(my_func)

# Identify timer registrations
bpy.app.timers.register(my_func)

# Then check my_func body for bpy.ops calls → BLOCKER
```

**Grep patterns**:
```
bpy\.app\.handlers\.\w+\.append\(
bpy\.app\.timers\.register\(
```

---

## Rule V-004: Operator poll() Pre-Check

**Check ID**: CHECK 2.2
**Severity**: WARNING

**Detection**: Flag `bpy.ops.*` calls that lack prerequisite checks. Common required checks:

| Operator | Required Pre-Check |
|----------|--------------------|
| `bpy.ops.object.mode_set(mode='EDIT')` | `context.active_object is not None` |
| `bpy.ops.mesh.subdivide()` | `context.mode == 'EDIT_MESH'` |
| `bpy.ops.object.modifier_apply()` | `context.mode == 'OBJECT'` and modifier exists |
| `bpy.ops.view3d.*` | `area.type == 'VIEW_3D'` |
| `bpy.ops.object.parent_set()` | `len(context.selected_objects) >= 2` |
| `bpy.ops.uv.unwrap()` | `context.mode == 'EDIT_MESH'` |
| `bpy.ops.object.convert()` | `context.active_object is not None` |

---

## Rule V-005: Dict Context Override (Version-Critical)

**Check ID**: CHECK 2.3
**Severity**: BLOCKER on 4.0+, WARNING on 3.2–3.6

**Detection patterns**:
```python
# Pattern 1: context.copy() + operator call with override dict
override = bpy.context.copy()
bpy.ops.something(override, ...)

# Pattern 2: Direct dict construction
override = {'area': area, 'region': region}
bpy.ops.something(override, ...)

# Pattern 3: Dict as first positional arg to any bpy.ops call
bpy.ops.mesh.select_all(override, action='SELECT')
```

**Grep patterns**:
```
bpy\.context\.copy\(\)
bpy\.ops\.\w+\.\w+\(\s*\w+\s*,      # Dict as first arg
```

**Replacement pattern**:
```python
with bpy.context.temp_override(area=area, region=region):
    bpy.ops.something(...)
```

---

## Rule V-006: Removed APIs — Blender 4.0

**Check ID**: CHECK 3.1
**Severity**: BLOCKER

| Removed API | Grep Pattern | Replacement |
|-------------|-------------|-------------|
| `mesh.calc_normals()` | `\.calc_normals\(\)` | Remove call (auto-calculated) |
| `bone.layers` | `bone\.layers\[` | `bone.collections` |
| `armature.bone_groups` | `\.bone_groups` | Removed — use bone collections |
| `obj.face_maps` | `\.face_maps` | Integer face attributes |
| `MeshEdge.bevel_weight` | `\.bevel_weight` (on edge) | `mesh.attributes["bevel_weight_edge"]` |
| `MeshEdge.crease` | `\.crease` (on edge) | `mesh.attributes["crease_edge"]` |
| `NodeTree.inputs.new()` | `\.inputs\.new\(` / `\.outputs\.new\(` | `NodeTree.interface.new_socket()` |
| `mesh.calc_normals_split()` | `\.calc_normals_split\(\)` | Remove call (auto-calculated) |
| `mesh.use_auto_smooth` | `\.use_auto_smooth` | Removed in 4.1 |
| `sculpt.lock_x/y/z` | `\.lock_[xyz]` (on sculpt) | Removed |

---

## Rule V-007: Removed APIs — Blender 5.0

**Check ID**: CHECK 3.1
**Severity**: BLOCKER

| Removed API | Grep Pattern | Replacement |
|-------------|-------------|-------------|
| `import bgl` | `import bgl` / `from bgl import` | `import gpu` |
| `bgl.glEnable(...)` | `bgl\.gl` | `gpu.state.*` |
| `del obj["prop"]` (RNA) | `del\s+\w+\["` | `obj.property_unset("prop")` |
| `scene["cycles"]` | `scene\["cycles"\]` | `scene.cycles` |
| `image.bindcode` | `\.bindcode` | `gpu.texture.from_image()` |
| `import io_scene_obj` | `import io_scene_obj` | Removed — use io_scene_wavefront |

---

## Rule V-008: EEVEE Identifier Mismatch

**Check ID**: CHECK 3.3
**Severity**: BLOCKER

**Detection**: Search for render engine assignment and check version:
```python
# Grep pattern
scene\.render\.engine\s*=\s*['"]BLENDER_EEVEE
```

**Version rules**:
- Target 3.x: MUST use `'BLENDER_EEVEE'`
- Target 4.2–4.x: MUST use `'BLENDER_EEVEE_NEXT'`
- Target 5.0+: MUST use `'BLENDER_EEVEE'`
- Multi-version: MUST use version-conditional

---

## Rule V-009: Stale Data References

**Check ID**: CHECK 4.1
**Severity**: BLOCKER

**Detection flow**:
1. Track all assignments from `bpy.data.*` collections
2. Track all `bpy.ops.*` calls, `bpy.data.*.remove()`, `mode_set()`, file operations
3. If a tracked reference is used AFTER an invalidating operation without re-fetch → BLOCKER

**Common stale reference patterns**:
```python
# Pattern 1: Reference used after remove
obj = bpy.data.objects["Cube"]
bpy.data.objects.remove(obj)
print(obj.name)  # CRASH — ReferenceError

# Pattern 2: Reference used after operator
mesh = bpy.data.meshes["MyMesh"]
bpy.ops.wm.read_homefile()
mesh.vertices  # CRASH — all references invalidated

# Pattern 3: CollectionProperty re-allocation
item = collection.add()
for i in range(100):
    collection.add()  # Re-allocates C array
item.name = "x"  # CRASH — pointer invalid
```

---

## Rule V-010: Name Collision on Data Creation

**Check ID**: CHECK 4.2
**Severity**: WARNING

**Detection pattern**:
```python
# Flag this pattern:
bpy.data.meshes.new("SomeName")    # Line N
bpy.data.meshes["SomeName"]        # Line N+M — unsafe lookup
```

**Grep pattern**:
```
bpy\.data\.\w+\.new\(.*\)$         # .new() without capturing return
bpy\.data\.\w+\["                  # Dict lookup after .new()
```

---

## Rule V-011: Missing Object-Collection Link

**Check ID**: CHECK 4.3
**Severity**: BLOCKER

**Detection flow**:
1. Find all `bpy.data.objects.new(...)` calls
2. Check if corresponding variable is passed to `collection.objects.link()` within the same scope
3. If no link call found → BLOCKER

**Grep patterns**:
```
bpy\.data\.objects\.new\(
\.objects\.link\(
```

---

## Rule V-012: BMesh ensure_lookup_table

**Check ID**: CHECK 4.4
**Severity**: BLOCKER

**Detection**: After `bmesh.new()` or `bmesh.from_edit_mesh()`, check if index access (`bm.verts[i]`, `bm.edges[i]`, `bm.faces[i]`) occurs without prior `ensure_lookup_table()` call on the same element type.

**Grep patterns**:
```
bm\.verts\[\d+\]
bm\.edges\[\d+\]
bm\.faces\[\d+\]
\.ensure_lookup_table\(\)
```

---

## Rule V-013: BMesh Memory Leak

**Check ID**: CHECK 4.4
**Severity**: BLOCKER

**Detection**: After `bmesh.new()`, check that `bm.free()` is called before function exit. Exception: BMesh from `bmesh.from_edit_mesh()` MUST NOT be freed — call `bmesh.update_edit_mesh()` instead.

**Grep patterns**:
```
bmesh\.new\(\)
bm\.free\(\)
bmesh\.from_edit_mesh\(
bmesh\.update_edit_mesh\(
```

---

## Rule V-014: Missing mesh.update() After from_pydata()

**Check ID**: CHECK 4.5
**Severity**: BLOCKER

**Detection**: Find `mesh.from_pydata(...)` calls. The next non-blank line on the same mesh variable MUST be `mesh.update()`.

**Grep patterns**:
```
\.from_pydata\(
\.update\(\)
```

---

## Rule V-015: Threading — bpy in Background Thread

**Check ID**: CHECK 5.1
**Severity**: BLOCKER

**Detection**: Find `threading.Thread(target=func)` or `executor.submit(func)`. Trace `func` body for any `bpy.*` call.

**Grep patterns**:
```
threading\.Thread\(target=
ThreadPoolExecutor
ProcessPoolExecutor
\.submit\(
import bpy   # Inside threaded module
```

---

## Rule V-016: Registration Order

**Check ID**: CHECK 6.1
**Severity**: BLOCKER

**Detection in register() function**:
1. Collect all `bpy.utils.register_class(ClassName)` calls in order
2. Collect all `PointerProperty(type=TypeName)` assignments
3. Verify: every `TypeName` in PointerProperty MUST be registered BEFORE the PointerProperty assignment

**Unregister() check**:
1. Verify reverse order: delete PointerProperties first, then unregister classes in reverse order

---

## Rule V-017: Handler Cleanup

**Check ID**: CHECK 6.2
**Severity**: WARNING

**Detection**: For each `bpy.app.handlers.*.append(func)` in `register()`, verify a corresponding `bpy.app.handlers.*.remove(func)` exists in `unregister()`.

---

## Rule V-018: Missing @persistent on Handler

**Check ID**: CHECK 6.3
**Severity**: BLOCKER

**Detection**: Find all functions appended to `bpy.app.handlers.*`. Check that each has the `@bpy.app.handlers.persistent` decorator.

**Grep patterns**:
```
@bpy\.app\.handlers\.persistent
@persistent
bpy\.app\.handlers\.\w+\.append\(
```

---

## Rule V-019: EnumProperty Items Garbage Collection

**Check ID**: CHECK 6.4
**Severity**: BLOCKER

**Detection**: Find `EnumProperty(items=func_name)` where `func_name` is a callback. Check that the callback stores its return value in a module-level or persistent variable.

```python
# BLOCKER: No caching
def get_items(self, context):
    return [(o.name, o.name, "") for o in bpy.data.objects]

# PASS: Cached
_items_cache = []
def get_items(self, context):
    global _items_cache
    _items_cache = [(o.name, o.name, "") for o in bpy.data.objects]
    return _items_cache
```

---

## Rule V-020: Unnecessary bpy.ops Usage

**Check ID**: CHECK 7.1
**Severity**: WARNING

| bpy.ops Call | Direct Alternative |
|-------------|-------------------|
| `bpy.ops.object.location_clear()` | `obj.location = (0, 0, 0)` |
| `bpy.ops.object.rotation_clear()` | `obj.rotation_euler = (0, 0, 0)` |
| `bpy.ops.object.scale_clear()` | `obj.scale = (1, 1, 1)` |
| `bpy.ops.object.select_all(action='DESELECT')` | Loop: `obj.select_set(False)` |
| `bpy.ops.transform.translate(value=v)` | `obj.location += Vector(v)` |
| `bpy.ops.object.delete()` | `bpy.data.objects.remove(obj)` |

---

## Rule V-021: bgl Import on Blender 5.0+

**Check ID**: CHECK 8
**Severity**: BLOCKER

**Detection**: `import bgl` or `from bgl import *` in any file targeting Blender 5.0+.

**Replacement mapping**:
| bgl Function | gpu Equivalent |
|-------------|---------------|
| `bgl.glEnable(bgl.GL_BLEND)` | `gpu.state.blend_set('ALPHA')` |
| `bgl.glDisable(bgl.GL_BLEND)` | `gpu.state.blend_set('NONE')` |
| `bgl.glLineWidth(w)` | `gpu.state.line_width_set(w)` |
| `bgl.glPointSize(s)` | `gpu.state.point_size_set(s)` |
| `bgl.glEnable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('LESS_EQUAL')` |
| `bgl.glDisable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('NONE')` |
