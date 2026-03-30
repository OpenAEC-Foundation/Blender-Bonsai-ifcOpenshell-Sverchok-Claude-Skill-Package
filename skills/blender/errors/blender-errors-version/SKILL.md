---
name: blender-errors-version
description: >
  Use when debugging Blender AttributeError from removed APIs, ImportError from deprecated
  modules (bgl in 5.0), or breaking changes in operator signatures between versions.
  Prevents silent failures from using APIs that were renamed or removed in newer Blender
  versions. Provides migration patterns from 3.x to 4.x to 5.x with complete replacement
  mappings.
  Keywords: AttributeError, ImportError, deprecated, version error, bgl, migration, breaking change, removed API, 3.x to 4.x, 4.x to 5.x, Blender upgrade, module has no attribute, addon not compatible.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Blender Errors: Version Compatibility

Diagnose and resolve Python API errors caused by Blender version differences. This skill activates when encountering `AttributeError`, `ImportError`, `TypeError`, `KeyError`, or `RuntimeError` in Blender scripts that result from using removed, renamed, or restructured APIs.

**Dependency**: [blender-core-versions](../../core/blender-core-versions/SKILL.md) — version matrix and detection API.

---

## 1. Error Diagnosis Decision Tree

When a Blender Python script raises an error, follow this decision tree:

```
ERROR RECEIVED
│
├── AttributeError?
│   ├── "has no attribute 'bevel_weight'" → Blender 4.0+ mesh attribute migration [E-01]
│   ├── "has no attribute 'crease'" → Blender 4.0+ mesh attribute migration [E-01]
│   ├── "has no attribute 'use_auto_smooth'" → Blender 4.1+ auto-smooth removal [E-02]
│   ├── "has no attribute 'auto_smooth_angle'" → Blender 4.1+ auto-smooth removal [E-02]
│   ├── "has no attribute 'face_maps'" → Blender 4.0+ face maps removal [E-03]
│   ├── "has no attribute 'layers'" (on Bone) → Blender 4.0+ bone collections [E-04]
│   ├── "has no attribute 'bone_groups'" → Blender 4.0+ bone collections [E-04]
│   ├── "has no attribute 'inputs'" (on NodeTree) → Blender 4.0+ interface API [E-05]
│   ├── "has no attribute 'outputs'" (on NodeTree) → Blender 4.0+ interface API [E-05]
│   ├── "has no attribute 'sculpt_tool'" → Blender 5.0+ brush rename [E-06]
│   ├── "has no attribute 'node_tree'" (on Scene) → Blender 5.0+ compositor [E-07]
│   ├── "has no attribute 'use_nodes'" (on Scene) → Blender 5.0+ compositor [E-07]
│   ├── "has no attribute 'end_frame'" (on Strip) → Blender 5.0+ VSE [E-08]
│   ├── "has no attribute 'fcurves'" (on Action) → Blender 5.0+ slotted actions [E-09]
│   ├── "has no attribute 'displacement_method'" (on CyclesMaterial) → Blender 4.1+ [E-10]
│   ├── "has no attribute 'active_color_name'" → Blender 4.3+ AttributeGroup split [E-11]
│   ├── "has no attribute 'use_ssr'" → Blender 4.3+ EEVEE property removal [E-12]
│   ├── "has no attribute 'use_bloom'" → Blender 4.3+ EEVEE property removal [E-12]
│   └── "has no attribute 'calc_normals'" → Blender 4.0+ auto normals [E-13]
│
├── ImportError / ModuleNotFoundError?
│   ├── "No module named 'bgl'" → Blender 5.0+ BGL removal [E-14]
│   ├── "No module named 'io_scene_obj'" → Blender 4.0+ C++ IO [E-15]
│   └── "No module named 'io_mesh_ply'" → Blender 4.0+ C++ IO [E-15]
│
├── TypeError?
│   ├── On bpy.ops call with dict first arg → Blender 4.0+ context override [E-16]
│   ├── "foreach_set() invalid data type" → Blender 4.1+ validation [E-17]
│   ├── On IDProperty type mismatch → Blender 4.2+ static typing [E-18]
│   └── On __init__() argument mismatch → Blender 4.4+ super() required [E-19]
│
├── KeyError?
│   ├── "Subsurface" (on Principled BSDF) → Blender 4.0+ socket renames [E-20]
│   ├── "Specular" (on Principled BSDF) → Blender 4.0+ socket renames [E-20]
│   ├── "cycles" (on scene[]) → Blender 5.0+ dict access removal [E-21]
│   └── Node socket name not found → Version-dependent socket names [E-20]
│
├── RuntimeError?
│   ├── "Embedded ID assignment" → Blender 4.3+ PointerProperty [E-22]
│   └── "BLENDER_EEVEE" engine not found → EEVEE identifier change [E-23]
│
└── ValueError?
    ├── "'CUBEMAP' not in enum" → Blender 4.1+ light probe renames [E-24]
    ├── "'PLANAR' not in enum" → Blender 4.1+ light probe renames [E-24]
    ├── "'3D_UNIFORM_COLOR' not found" → Blender 4.0+ shader renames [E-25]
    └── "'BLENDER_EEVEE_NEXT' not in enum" → Blender 5.0+ EEVEE revert [E-23]
```

---

## 2. Error Patterns: Quick Resolution Table

### Blender 4.0 Breaking Errors

| Error Code | Error Message Pattern | Removed API | Replacement | Since |
|------------|----------------------|-------------|-------------|-------|
| E-01 | `AttributeError: 'MeshEdge' has no attribute 'bevel_weight'` | `mesh.edges[i].bevel_weight` | `mesh.attributes.get("bevel_weight_edge")` | 4.0 |
| E-01 | `AttributeError: 'MeshEdge' has no attribute 'crease'` | `mesh.edges[i].crease` | `mesh.attributes.get("crease_edge")` | 4.0 |
| E-03 | `AttributeError: 'Object' has no attribute 'face_maps'` | `obj.face_maps` | Integer face attributes | 4.0 |
| E-04 | `AttributeError: 'Bone' has no attribute 'layers'` | `bone.layers[i]` | `bone.collections` | 4.0 |
| E-04 | `AttributeError: 'Pose' has no attribute 'bone_groups'` | `pose.bone_groups` | Bone collections + colors | 4.0 |
| E-05 | `AttributeError: 'NodeTree' has no attribute 'inputs'` | `NodeTree.inputs.new()` | `NodeTree.interface.new_socket()` | 4.0 |
| E-13 | `AttributeError: 'Mesh' has no attribute 'calc_normals'` | `mesh.calc_normals()` | Removed (auto-calculated) | 4.0 |
| E-15 | `ModuleNotFoundError: No module named 'io_scene_obj'` | Python OBJ/PLY IO | `bpy.ops.wm.obj_import/export` | 4.0 |
| E-16 | `TypeError: bpy.ops expects keyword arguments` | Dict context override | `context.temp_override()` | 4.0 |
| E-20 | `KeyError: 'Subsurface'` on Principled BSDF | Old socket names | `Subsurface Weight`, etc. | 4.0 |
| E-25 | `ValueError: '3D_UNIFORM_COLOR' not found` | `3D_*` shader names | `POLYLINE_*` / prefix removed | 4.0 |

### Blender 4.1 Breaking Errors

| Error Code | Error Message Pattern | Removed API | Replacement | Since |
|------------|----------------------|-------------|-------------|-------|
| E-02 | `AttributeError: 'Mesh' has no attribute 'use_auto_smooth'` | `mesh.use_auto_smooth` | `mesh.corner_normals` + Smooth by Angle modifier | 4.1 |
| E-02 | `AttributeError: 'Mesh' has no attribute 'auto_smooth_angle'` | `mesh.auto_smooth_angle` | Smooth by Angle modifier | 4.1 |
| E-10 | `AttributeError: 'CyclesMaterialSettings' has no attribute 'displacement_method'` | `mat.cycles.displacement_method` | `mat.displacement_method` | 4.1 |
| E-17 | `TypeError: foreach_set() invalid data type` | Silent failure on bad types | Add type validation | 4.1 |
| E-24 | `ValueError: 'CUBEMAP' not in enum` | `CUBEMAP`/`PLANAR`/`GRID` | `SPHERE`/`PLANE`/`VOLUME` | 4.1 |

### Blender 4.2 Breaking Errors

| Error Code | Error Message Pattern | Removed API | Replacement | Since |
|------------|----------------------|-------------|-------------|-------|
| E-18 | `TypeError: IDProperty type mismatch` | Dynamic IDProperty types | Static typing enforced | 4.2 |
| E-23 | `RuntimeError: 'BLENDER_EEVEE' not found` (in 4.2-4.x) | `BLENDER_EEVEE` | `BLENDER_EEVEE_NEXT` | 4.2 |

### Blender 4.3 Breaking Errors

| Error Code | Error Message Pattern | Removed API | Replacement | Since |
|------------|----------------------|-------------|-------------|-------|
| E-11 | `AttributeError: 'AttributeGroup*' has no attribute 'active_color_name'` | Generic `AttributeGroup` | `AttributeGroupMesh` (type-specific) | 4.3 |
| E-12 | `AttributeError: 'EEVEE' has no attribute 'use_ssr'` | 40+ EEVEE legacy props | Properties removed (engine-internal) | 4.3 |
| E-22 | `RuntimeError: Embedded ID assignment` | Embedded ID to PointerProperty | Do not assign embedded IDs | 4.3 |

### Blender 4.4 Breaking Errors

| Error Code | Error Message Pattern | Removed API | Replacement | Since |
|------------|----------------------|-------------|-------------|-------|
| E-19 | `TypeError: __init__() argument mismatch` | No `super().__init__()` | `super().__init__(*args, **kwargs)` | 4.4 |

### Blender 5.0 Breaking Errors

| Error Code | Error Message Pattern | Removed API | Replacement | Since |
|------------|----------------------|-------------|-------------|-------|
| E-14 | `ModuleNotFoundError: No module named 'bgl'` | `import bgl` | `import gpu` + `gpu.state.*` | 5.0 |
| E-06 | `AttributeError: 'Brush' has no attribute 'sculpt_tool'` | `brush.sculpt_tool` | `brush.sculpt_brush_type` | 5.0 |
| E-07 | `AttributeError: 'Scene' has no attribute 'node_tree'` | `scene.node_tree` (compositor) | `scene.compositing_node_group` | 5.0 |
| E-08 | `AttributeError: 'Strip' has no attribute 'end_frame'` | `strip.end_frame` | `strip.length` | 5.0 |
| E-09 | `AttributeError: 'Action' has no attribute 'fcurves'` | `action.fcurves` (legacy) | Slotted Actions channelbag API | 5.0 |
| E-21 | `KeyError: 'cycles'` on `scene['cycles']` | Dict access to RNA props | `scene.cycles.samples` (attribute) | 5.0 |

---

## 3. Deprecation Timeline

ALWAYS check this timeline before using any API. If the current Blender version is past the removal date, the API WILL raise an error.

| API | Deprecated | Removed | Warning Period |
|-----|-----------|---------|---------------|
| Context override dicts | 3.2 | **4.0** | 3.2 → 3.6 (4 releases) |
| `mesh.calc_normals()` | 3.4 | **4.0** | 3.4 → 3.6 (2 releases) |
| `MeshEdge.bevel_weight` / `.crease` | 3.4 | **4.0** | 3.4 → 3.6 (2 releases) |
| `NodeTree.inputs/outputs` | 3.4 | **4.0** | 3.4 → 3.6 (2 releases) |
| `3D_UNIFORM_COLOR` shader | 3.4 | **4.0** | 3.4 → 3.6 (2 releases) |
| `bgl` module | 3.5 | **5.0** | 3.5 → 4.x (long, 9+ releases) |
| `mesh.use_auto_smooth` | 4.0 | **4.1** | 4.0 only (1 release) |
| Light probe `CUBEMAP`/`PLANAR`/`GRID` | 4.0 | **4.1** | 4.0 only (1 release) |
| `bl_info` dict (for extensions) | 4.2 | Ongoing | Still works as legacy addon |
| `BLENDER_EEVEE` identifier | 4.2 | **4.2** (temp) | Renamed to `_NEXT`, reverted in 5.0 |
| EEVEE legacy properties (40+) | 4.2 | **4.3** | 4.2 only (1 release) |
| `bpy.types.AttributeGroup` (generic) | 4.2 | **4.3** | 4.2 only (1 release) |
| `bpy.types.Sequence` (VSE) | 4.3 | **4.4** | 4.3 only (1 release) |
| `action.fcurves` (legacy) | 4.4 | **5.0** | 4.4 → 4.5 (2 releases) |
| `Image.bindcode` / `gl_load()` | 4.0 | **5.0** | 4.0 → 4.x (long) |
| `scene['cycles']` dict access | 4.2 | **5.0** | 4.2 → 4.x |
| `scene.node_tree` (compositor) | 4.2 | **5.0** | 4.2 → 4.x |
| VSE time property names | **5.1** | **6.0** | 5.1 → 5.x (future) |

---

## 4. Diagnostic Patterns

### Pattern 1: Identify Version from Error

When a user reports an error without specifying their Blender version, ALWAYS determine the version first:

```python
# Blender 3.x / 4.x / 5.x: Print diagnostic info
import bpy
print(f"Blender: {bpy.app.version}")
print(f"Python: {bpy.app.version_string}")
print(f"Cycle: {bpy.app.version_cycle}")
```

### Pattern 2: Version-Safe Error Recovery

ALWAYS use `hasattr()` for graceful degradation. NEVER catch `AttributeError` silently.

```python
# Blender 3.x / 4.x / 5.x: Version-safe accessor with explicit fallback
import bpy

def get_bevel_weight(mesh, edge_index):
    """Get edge bevel weight with version-safe fallback."""
    if bpy.app.version >= (4, 0, 0):
        attr = mesh.attributes.get("bevel_weight_edge")
        if attr:
            return attr.data[edge_index].value
        return 0.0
    else:
        return mesh.edges[edge_index].bevel_weight
```

### Pattern 3: Multi-Version Compatibility Wrapper

For addons targeting multiple Blender versions, use a compatibility module:

```python
# Blender 3.x / 4.x / 5.x: compat.py module pattern
import bpy

BLENDER_4 = bpy.app.version >= (4, 0, 0)
BLENDER_41 = bpy.app.version >= (4, 1, 0)
BLENDER_42 = bpy.app.version >= (4, 2, 0)
BLENDER_43 = bpy.app.version >= (4, 3, 0)
BLENDER_44 = bpy.app.version >= (4, 4, 0)
BLENDER_5 = bpy.app.version >= (5, 0, 0)
BLENDER_51 = bpy.app.version >= (5, 1, 0)
```

---

## 5. Critical Rules

1. NEVER suppress `DeprecationWarning` in Blender scripts — it hides migration signals.
2. NEVER use `try/except AttributeError` as a version check — use `bpy.app.version` or `hasattr()`.
3. ALWAYS check the deprecation timeline BEFORE using any Blender API.
4. ALWAYS test addons on the OLDEST supported version, not just the newest.
5. NEVER assume backward compatibility of `.blend` files across major API rewrites (Grease Pencil 4.3, Slotted Actions 4.4, Compositor 5.0).
6. ALWAYS restore `gpu.state` to defaults at the end of draw callbacks.
7. NEVER use `bgl` in any new code — it is REMOVED in Blender 5.0.
8. ALWAYS use `context.temp_override()` instead of dict overrides in Blender 4.0+.

---

## 6. Reference Links

- **[references/methods.md](references/methods.md)** — Error diagnosis functions, version detection API, compatibility checking utilities
- **[references/examples.md](references/examples.md)** — Error-to-fix migration code for each error pattern (E-01 through E-25)
- **[references/anti-patterns.md](references/anti-patterns.md)** — Version compatibility mistakes that cause errors, with explanations

### Official Documentation

- [Blender 4.0 Python API Changes](https://developer.blender.org/docs/release_notes/4.0/python_api/)
- [Blender 4.1 Python API Changes](https://developer.blender.org/docs/release_notes/4.1/python_api/)
- [Blender 4.2 Python API Changes](https://developer.blender.org/docs/release_notes/4.2/python_api/)
- [Blender 4.3 Python API Changes](https://developer.blender.org/docs/release_notes/4.3/python_api/)
- [Blender 4.4 Python API Changes](https://developer.blender.org/docs/release_notes/4.4/python_api/)
- [Blender 5.0 Python API Changes](https://developer.blender.org/docs/release_notes/5.0/python_api/)
- [Blender 5.1 Python API Changes](https://developer.blender.org/docs/release_notes/5.1/python_api/)
- [Blender API Compatibility Notes](https://developer.blender.org/docs/release_notes/compatibility/)
- [Grease Pencil 4.3 Migration Guide](https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/)
