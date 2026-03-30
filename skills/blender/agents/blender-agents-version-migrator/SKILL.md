---
name: blender-agents-version-migrator
description: >
  Use when migrating, porting, or upgrading Blender Python scripts and addons across major
  versions (3.x to 4.x to 5.x). Provides a systematic migration process covering API
  renames, removed functions, changed parameters, extension system migration, BGL to GPU
  module conversion, and bone collection migration. Prevents incomplete migrations that
  compile but fail at runtime.
  Keywords: migration, porting, upgrade, version, 3.x to 4.x, 4.x to 5.x, API rename, bgl to gpu, extension migration, bl_info to manifest, bone collection, script stopped working after update, upgrade addon to new Blender.
license: MIT
compatibility: "Designed for Claude Code. Requires Python 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Blender Agent: Version Migrator

Systematic migration process for Blender Python code across major versions. This agent skill activates when migrating, porting, updating, or upgrading Blender Python scripts or addons between versions 3.x, 4.x, and 5.x.

**Dependencies**:
- [blender-core-versions](../../core/blender-core-versions/SKILL.md) — version matrix, detection API, breaking changes
- [blender-errors-version](../../errors/blender-errors-version/SKILL.md) — error diagnosis for version-related failures

---

## 1. Quick Reference: Migration Paths

### Supported Migration Paths

| From | To | Complexity | Key Changes |
|------|----|-----------|-------------|
| 3.x | 4.0 | HIGH | Context overrides, mesh attributes, node interface, bone collections |
| 4.0 | 4.1 | MEDIUM | Auto-smooth removal, light probe renames, displacement method move |
| 4.1 | 4.2 | MEDIUM | Extension manifest, EEVEE rename, static IDProperties |
| 4.2 | 4.3 | HIGH | Grease Pencil rewrite, AttributeGroup split, EEVEE property removal |
| 4.3 | 4.4 | MEDIUM | super().__init__ required, Sequence→Strip, Slotted Actions |
| 4.4 | 4.5 | LOW | Minor deprecations only |
| 4.x | 5.0 | HIGH | BGL removal, compositor change, slotted actions mandatory |
| 5.0 | 5.1 | LOW | Python 3.13, VSE property renames (deprecated) |
| 3.x | 5.x | CRITICAL | ALL of the above — execute sequentially |

### Migration Execution Order

For a 3.x → 5.x migration, ALWAYS apply changes in this exact order:
1. 3.x → 4.0 changes
2. 4.0 → 4.1 changes
3. 4.1 → 4.2 changes
4. 4.2 → 4.3 changes
5. 4.3 → 4.4 changes
6. 4.4 → 5.0 changes
7. 5.0 → 5.1 changes

NEVER skip intermediate versions. Each version removes APIs deprecated in prior versions.

---

## 2. Migration Agent Process

When migrating Blender Python code, follow these steps in order:

### Step 1: Identify Source and Target Versions

```python
# Determine current version compatibility from code signals
# Check for these indicators:
# - bl_info dict → 3.x / 4.0 / 4.1 addon
# - blender_manifest.toml → 4.2+ extension
# - import bgl → pre-5.0 code
# - context override dicts on bpy.ops → pre-4.0 code
# - mesh.use_auto_smooth → pre-4.1 code
# - NodeTree.inputs.new() → pre-4.0 code
```

### Step 2: Scan for Affected APIs

Search the codebase for ALL patterns listed in the Find-and-Replace Tables (Section 3). Mark each occurrence with the version transition that affects it.

### Step 3: Apply Changes Per Version Transition

Apply changes from the relevant migration checklists below. Process ONE version transition at a time.

### Step 4: Update Metadata

- Replace `bl_info` with `blender_manifest.toml` if targeting 4.2+
- Update minimum version requirements
- Update any version checks in the code

### Step 5: Validate

- Search for ANY remaining references to removed APIs
- Verify all `import` statements reference available modules
- Check that `gpu.state` is restored at end of draw callbacks
- Confirm `super().__init__(*args, **kwargs)` in all Blender type subclasses (4.4+)

---

## 3. Find-and-Replace Tables

### 3.x → 4.0 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `bpy.ops.*.call(override_dict,` (dict as first arg) | `with bpy.context.temp_override(**kwargs):` | Context |
| `mesh.edges[i].bevel_weight` | `mesh.attributes.get("bevel_weight_edge").data[i].value` | Mesh |
| `mesh.edges[i].crease` | `mesh.attributes.get("crease_edge").data[i].value` | Mesh |
| `obj.face_maps` | Integer face attributes | Mesh |
| `mesh.calc_normals()` | Remove call (auto-calculated) | Mesh |
| `bone.layers[i]` | `bone.collections` | Armature |
| `pose.bone_groups` | Bone collections with colors | Armature |
| `NodeTree.inputs.new(type, name)` | `NodeTree.interface.new_socket(name=name, in_out='INPUT', socket_type=type)` | Nodes |
| `NodeTree.outputs.new(type, name)` | `NodeTree.interface.new_socket(name=name, in_out='OUTPUT', socket_type=type)` | Nodes |
| `node.inputs["Subsurface"]` | `node.inputs["Subsurface Weight"]` | Shader |
| `node.inputs["Specular"]` | `node.inputs["Specular IOR Level"]` | Shader |
| `node.inputs["Transmission"]` | `node.inputs["Transmission Weight"]` | Shader |
| `gpu.shader.from_builtin('3D_UNIFORM_COLOR')` | `gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')` | GPU |
| `gpu.shader.from_builtin('3D_FLAT_COLOR')` | `gpu.shader.from_builtin('POLYLINE_FLAT_COLOR')` | GPU |
| `bpy.ops.import_scene.obj(` | `bpy.ops.wm.obj_import(` | IO |
| `bpy.ops.export_scene.obj(` | `bpy.ops.wm.obj_export(` | IO |
| `filename=` (in IO operators) | `filepath=` | IO |

### 4.0 → 4.1 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `mesh.use_auto_smooth = True` | Remove (always active) | Mesh |
| `mesh.auto_smooth_angle` | Use "Smooth by Angle" modifier | Mesh |
| `mesh.calc_normals_split()` | Remove (use `mesh.corner_normals`) | Mesh |
| `mesh.create_normals_split()` | Remove | Mesh |
| `mesh.free_normals_split()` | Remove | Mesh |
| `probe.type == 'CUBEMAP'` | `probe.type == 'SPHERE'` | Light Probe |
| `probe.type == 'PLANAR'` | `probe.type == 'PLANE'` | Light Probe |
| `probe.type == 'GRID'` | `probe.type == 'VOLUME'` | Light Probe |
| `mat.cycles.displacement_method` | `mat.displacement_method` | Material |
| `SUBSAMPLING_3x3` | `BOX` | Sequencer |

### 4.1 → 4.2 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `bl_info = {` (if targeting 4.2+) | `blender_manifest.toml` file | Extension |
| `BLENDER_EEVEE` (render engine) | `BLENDER_EEVEE_NEXT` | Render |
| `scene.cycles.motion_blur_position` | `scene.render.motion_blur_position` | Render |
| `scene.eevee.use_motion_blur` | `scene.render.use_motion_blur` | Render |
| `object.load_reference_image` | `object.empty_image_add` | Object |

### 4.2 → 4.3 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `bpy.types.AttributeGroup` | `AttributeGroupMesh` / `AttributeGroupPointCloud` / etc. | Attributes |
| `scene.eevee.use_ssr` | Remove (engine-internal) | EEVEE |
| `scene.eevee.use_bloom` | Remove (engine-internal) | EEVEE |
| `scene.eevee.*` (40+ legacy props) | Remove or check 4.3 release notes | EEVEE |
| ALL Grease Pencil API calls | Complete rewrite required | Grease Pencil |

### 4.3 → 4.4 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `class MyOp(bpy.types.Operator):` (no super init) | Add `super().__init__(*args, **kwargs)` | Classes |
| `bpy.types.Sequence` | `bpy.types.Strip` | VSE |
| `action.fcurves.new(` (for new code) | Slotted Actions API | Animation |
| `paint.brush` (write access) | Read-only; use brush asset system | Paint |

### 4.x → 5.0 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `import bgl` | `import gpu` | Drawing |
| `bgl.glEnable(bgl.GL_BLEND)` | `gpu.state.blend_set('ALPHA')` | Drawing |
| `bgl.glDisable(bgl.GL_BLEND)` | `gpu.state.blend_set('NONE')` | Drawing |
| `bgl.glLineWidth(w)` | `gpu.state.line_width_set(w)` | Drawing |
| `bgl.glEnable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('LESS_EQUAL')` | Drawing |
| `bgl.glDisable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('NONE')` | Drawing |
| `bgl.glPointSize(s)` | `gpu.state.point_size_set(s)` | Drawing |
| `bgl.glDepthMask(GL_TRUE)` | `gpu.state.depth_mask_set(True)` | Drawing |
| `image.gl_load()` / `image.bindcode` | `gpu.texture.from_image(image)` | Drawing |
| `gpu.types.GPUShader()` | `gpu.shader.create_from_info()` | Drawing |
| `scene['cycles']` | `scene.cycles.property_name` (attribute access) | Properties |
| `scene.node_tree` (compositor) | `scene.compositing_node_group` | Compositor |
| `scene.use_nodes` (compositor) | Remove (always active) | Compositor |
| `brush.sculpt_tool` | `brush.sculpt_brush_type` | Sculpt |
| `strip.end_frame` (VSE) | `strip.length` | VSE |
| `action.fcurves` (legacy) | Slotted Actions channelbag API | Animation |
| `BLENDER_EEVEE_NEXT` | `BLENDER_EEVEE` (reverted) | Render |

### 5.0 → 5.1 Replacements

| Find | Replace With | Category |
|------|-------------|----------|
| `sculpt.sample_color` | `paint.sample_color` | Sculpt |
| `frame_final_duration` | `duration` (deprecated, removed in 6.0) | VSE |
| `frame_final_start` | `left_handle` (deprecated, removed in 6.0) | VSE |
| `frame_final_end` | `right_handle` (deprecated, removed in 6.0) | VSE |

---

## 4. Decision Tree: Compatibility Shims vs Clean Migration

```
MIGRATION STRATEGY DECISION
│
├── Target SINGLE Blender version?
│   └── YES → Clean migration: remove ALL legacy code, use target API only
│
├── Target VERSION RANGE (e.g., 3.x + 4.x)?
│   ├── Range spans 3.x and 4.0+ ?
│   │   └── YES → Use version-safe wrappers with bpy.app.version checks
│   ├── Range spans 4.x and 5.0+ ?
│   │   └── YES → Use version-safe wrappers; NEVER import bgl at module level
│   └── Range within single major (e.g., 4.0–4.4)?
│       └── Use hasattr() feature detection for minor version differences
│
└── Target LATEST only (drop legacy)?
    └── Clean migration: target version API only, no compatibility code
```

### When to Use Compatibility Shims

ALWAYS use compatibility shims when:
- The addon targets 2+ major Blender versions simultaneously
- The addon is distributed to users on different Blender versions

NEVER use compatibility shims when:
- The migration targets a single Blender version
- The code is an internal tool pinned to one Blender version

### Compatibility Module Pattern

```python
# compat.py: Version-safe compatibility layer
import bpy

BLENDER_4 = bpy.app.version >= (4, 0, 0)
BLENDER_41 = bpy.app.version >= (4, 1, 0)
BLENDER_42 = bpy.app.version >= (4, 2, 0)
BLENDER_43 = bpy.app.version >= (4, 3, 0)
BLENDER_44 = bpy.app.version >= (4, 4, 0)
BLENDER_5 = bpy.app.version >= (5, 0, 0)
BLENDER_51 = bpy.app.version >= (5, 1, 0)

def apply_modifier(obj, modifier_name):
    if BLENDER_4:
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        override = {"object": obj, "active_object": obj}
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)
```

### Conditional Import Pattern for BGL → gpu

```python
# NEVER import bgl at module level in multi-version addons
import gpu

# For addons targeting 3.x–4.x (before 5.0 removal):
try:
    import bgl
    HAS_BGL = True
except ImportError:
    HAS_BGL = False  # Blender 5.0+

# ALWAYS use gpu module for new code regardless of version
```

---

## 5. EEVEE Identifier Migration

The EEVEE render engine identifier changed across versions. ALWAYS use this lookup:

| Blender Version | EEVEE Identifier |
|----------------|-----------------|
| 3.x | `BLENDER_EEVEE` |
| 4.0 – 4.1 | `BLENDER_EEVEE` |
| 4.2 – 4.x | `BLENDER_EEVEE_NEXT` |
| 5.0+ | `BLENDER_EEVEE` |

```python
def get_eevee_identifier():
    if bpy.app.version >= (5, 0, 0):
        return 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        return 'BLENDER_EEVEE_NEXT'
    else:
        return 'BLENDER_EEVEE'
```

---

## 6. Extension System Migration (4.2+)

When migrating from legacy addon to extension system:

1. Create `blender_manifest.toml` in addon root
2. Remove `bl_info` dict from `__init__.py`
3. Set `blender_version_min = "4.2.0"` (or target minimum)
4. Add `id` matching the package directory name
5. Add `tagline` (max 64 chars, no trailing punctuation)
6. Add `type = "add-on"` or `type = "theme"`
7. Add network permissions if addon accesses internet:
   `[permissions]` → `network = "Reason for access"`
8. Add `license = ["SPDX:GPL-3.0-or-later"]` (or appropriate SPDX)

---

## 7. Critical Migration Rules

1. NEVER use `import bgl` in code targeting Blender 5.0+ — the module is REMOVED.
2. ALWAYS restore `gpu.state` to defaults at the end of every draw callback.
3. NEVER pass a dict as the first argument to `bpy.ops` in Blender 4.0+ — use `context.temp_override()`.
4. ALWAYS add `super().__init__(*args, **kwargs)` to Blender type subclasses in 4.4+.
5. NEVER suppress `DeprecationWarning` — it hides migration signals.
6. NEVER use `try/except AttributeError` as a version check — use `bpy.app.version` or `hasattr()`.
7. ALWAYS set `POLYLINE_UNIFORM_COLOR` shader uniforms: `viewportSize` and `lineWidth`.
8. ALWAYS migrate Grease Pencil code completely when targeting 4.3+ — partial migration breaks.
9. NEVER assign embedded IDs to `PointerProperty` in 4.3+.
10. ALWAYS check the deprecation timeline in [blender-errors-version](../../errors/blender-errors-version/SKILL.md) before using any API.

---

## 8. Reference Links

- **[references/methods.md](references/methods.md)** — Complete API migration mappings organized by version transition
- **[references/examples.md](references/examples.md)** — Before/after code examples for each migration path
- **[references/anti-patterns.md](references/anti-patterns.md)** — Common migration mistakes with explanations

### Official Documentation

- [Blender 4.0 Python API Changes](https://developer.blender.org/docs/release_notes/4.0/python_api/)
- [Blender 4.1 Python API Changes](https://developer.blender.org/docs/release_notes/4.1/python_api/)
- [Blender 4.2 Python API Changes](https://developer.blender.org/docs/release_notes/4.2/python_api/)
- [Blender 4.3 Python API Changes](https://developer.blender.org/docs/release_notes/4.3/python_api/)
- [Blender 4.4 Python API Changes](https://developer.blender.org/docs/release_notes/4.4/python_api/)
- [Blender 4.5 Python API Changes](https://developer.blender.org/docs/release_notes/4.5/python_api/)
- [Blender 5.0 Python API Changes](https://developer.blender.org/docs/release_notes/5.0/python_api/)
- [Blender 5.1 Python API Changes](https://developer.blender.org/docs/release_notes/5.1/python_api/)
- [Blender API Compatibility Notes](https://developer.blender.org/docs/release_notes/compatibility/)
- [Grease Pencil 4.3 Migration Guide](https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/)
