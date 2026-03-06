---
name: blender-core-versions
description: "Provides complete Blender Python API version matrix from 3.x through 5.1 with all breaking changes, migration paths, and version-safe coding patterns. Activates when writing version-compatible scripts, migrating addons between Blender versions, or checking API deprecations."
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Blender Core Versions — Python API Version Matrix

## 1. Quick Reference

### Version Detection

```python
# Blender 3.x / 4.x / 5.x — ALWAYS use tuple comparison for version checks
import bpy
major, minor, patch = bpy.app.version  # e.g., (4, 2, 0)

# Version-safe conditional
if bpy.app.version >= (4, 0, 0):
    # Blender 4.0+ code path
    pass
else:
    # Blender 3.x fallback
    pass
```

See [references/methods.md](references/methods.md) for complete version detection API.

### Python Version Matrix

| Blender | Python | VFX Platform |
|---------|--------|-------------|
| 3.x     | 3.10   | 2023        |
| 4.0     | 3.10   | 2023        |
| 4.1     | 3.11   | 2024        |
| 4.2 LTS | 3.11   | 2024        |
| 4.3     | 3.11   | 2024        |
| 4.4     | 3.11   | 2024        |
| 4.5 LTS | 3.11   | 2024        |
| 5.0     | 3.11   | 2025        |
| 5.1     | 3.13   | 2026        |

### Critical Breaking Changes Summary

| Change | Removed In | Migration |
|--------|-----------|-----------|
| Context override dicts on `bpy.ops` | 4.0 | `context.temp_override()` |
| `MeshEdge.bevel_weight` / `.crease` | 4.0 | `mesh.attributes.get("bevel_weight_edge")` |
| `obj.face_maps` | 4.0 | Integer face attributes |
| `bone.layers[i]` / `pose.bone_groups` | 4.0 | `bone.collections` |
| `NodeTree.inputs/outputs.new()` | 4.0 | `NodeTree.interface.new_socket()` |
| Principled BSDF socket names | 4.0 | Renamed: `Subsurface Weight`, `Specular IOR Level`, etc. |
| `3D_UNIFORM_COLOR` shader name | 4.0 | `POLYLINE_UNIFORM_COLOR` |
| Python OBJ/PLY importers | 4.0 | `bpy.ops.wm.obj_import/export` |
| `Mesh.use_auto_smooth` | 4.1 | `Mesh.corner_normals` + Smooth by Angle modifier |
| Light probe `CUBEMAP`/`PLANAR`/`GRID` | 4.1 | `SPHERE` / `PLANE` / `VOLUME` |
| `mat.cycles.displacement_method` | 4.1 | `mat.displacement_method` |
| `blender_manifest.toml` required | 4.2 | Replace `bl_info` dict with TOML manifest |
| EEVEE → `BLENDER_EEVEE_NEXT` | 4.2 | Check identifier string |
| `bpy.types.AttributeGroup` | 4.3 | `AttributeGroupMesh`, `AttributeGroupPointCloud`, etc. |
| Grease Pencil API rewrite | 4.3 | Complete API rewrite; see migration guide |
| EEVEE legacy properties | 4.3 | 40+ properties removed |
| Subclass `__init__` must call `super()` | 4.4 | Forward `*args, **kwargs` to parent |
| `bpy.types.Sequence` → `bpy.types.Strip` | 4.4 | Rename all Sequence references |
| Slotted Actions system | 4.4 | `action.layers[0].strips[0].channelbag()` |
| `bgl` module | 5.0 | `gpu` module (MANDATORY) |
| `scene['cycles']` dict access | 5.0 | Attribute access on RNA properties |
| `scene.node_tree` (compositor) | 5.0 | `scene.compositing_node_group` |
| `brush.sculpt_tool` | 5.0 | `brush.sculpt_brush_type` |
| `action.fcurves` legacy API | 5.0 | Slotted Actions channelbag API |
| VSE `end_frame` | 5.0 | `length` property |
| VSE strip time property renames | 5.1 | `frame_final_duration` → `duration`, etc. (removal in 6.0) |
| Python 3.13 | 5.1 | Check stdlib changes |

---

## 2. Essential Patterns

### Version-Safe Conditionals

ALWAYS use `bpy.app.version` tuple comparison. NEVER parse `bpy.app.version_string`.

```python
# Blender 3.x / 4.x / 5.x — Version-safe conditional pattern
import bpy

if bpy.app.version >= (5, 0, 0):
    # Blender 5.0+ code
    comp_tree = scene.compositing_node_group
elif bpy.app.version >= (4, 0, 0):
    # Blender 4.0–4.x code
    comp_tree = scene.node_tree
else:
    # Blender 3.x code
    comp_tree = scene.node_tree
```

### Feature Detection Pattern

ALWAYS prefer `hasattr()` checks over version comparisons when testing for a specific API.

```python
# Blender 3.x / 4.x / 5.x — Feature detection pattern
import bpy

# Prefer feature detection over version check
if hasattr(bpy.types, "AttributeGroupMesh"):
    # Blender 4.3+ — type-specific attribute groups
    pass
else:
    # Blender < 4.3 — generic AttributeGroup
    pass

# For method existence
mesh = bpy.context.active_object.data
if hasattr(mesh, "corner_normals"):
    # Blender 4.1+ — use corner_normals
    normals = mesh.corner_normals
else:
    # Blender < 4.1 — use legacy auto_smooth
    mesh.use_auto_smooth = True
```

### Context Override Pattern

```python
# Blender 3.x — LEGACY context override (BROKEN in 4.0+)
override = {"object": obj, "active_object": obj}
bpy.ops.object.modifier_apply(override, modifier="Boolean")

# Blender 4.0+ — temp_override (REQUIRED)
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Boolean")

# Version-safe wrapper
def apply_modifier(obj, modifier_name):
    if bpy.app.version >= (4, 0, 0):
        with bpy.context.temp_override(object=obj, active_object=obj):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        override = {"object": obj, "active_object": obj}
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)
```

### Node Group Interface Pattern

```python
# Blender 3.x — LEGACY (BROKEN in 4.0+)
node_group.inputs.new("NodeSocketFloat", "Width")
node_group.outputs.new("NodeSocketGeometry", "Geometry")

# Blender 4.0+ — interface API (REQUIRED)
node_group.interface.new_socket(
    name="Width", in_out='INPUT', socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry'
)
```

### EEVEE Identifier Pattern

```python
# Blender 3.x — 'BLENDER_EEVEE'
# Blender 4.2–4.x — 'BLENDER_EEVEE_NEXT'
# Blender 5.0+ — 'BLENDER_EEVEE' (changed back)
import bpy

def get_eevee_identifier():
    if bpy.app.version >= (5, 0, 0):
        return 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        return 'BLENDER_EEVEE_NEXT'
    else:
        return 'BLENDER_EEVEE'

scene = bpy.context.scene
scene.render.engine = get_eevee_identifier()
```

### Extension Manifest Pattern

```python
# Blender 3.x / 4.0 / 4.1 — bl_info dict in __init__.py
bl_info = {
    "name": "My Addon",
    "blender": (4, 0, 0),
    "category": "Object",
    "version": (1, 0, 0),
    "description": "My addon description",
}

# Blender 4.2+ — blender_manifest.toml (REQUIRED for extensions)
# File: blender_manifest.toml
# schema_version = "1.0.0"
# id = "my_addon"
# version = "1.0.0"
# name = "My Addon"
# tagline = "Short description up to 64 chars"
# maintainer = "Dev Name <email@example.com>"
# type = "add-on"
# blender_version_min = "4.2.0"
```

### GPU Drawing Pattern (BGL → gpu Migration)

```python
# Blender 3.x — bgl module (BROKEN in 5.0)
import bgl
bgl.glEnable(bgl.GL_BLEND)
bgl.glLineWidth(2)
shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')

# Blender 4.0+ — gpu module (REQUIRED for 5.0+)
import gpu
gpu.state.blend_set('ALPHA')
gpu.state.line_width_set(2.0)
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')

# ALWAYS restore gpu state at end of draw callbacks
gpu.state.blend_set('NONE')
gpu.state.line_width_set(1.0)
```

### Compositor Access Pattern

```python
# Blender 3.x / 4.x — scene.node_tree
scene = bpy.context.scene
scene.use_nodes = True
comp_tree = scene.node_tree

# Blender 5.0+ — compositing_node_group
scene = bpy.context.scene
comp_tree = bpy.data.node_groups.new("MyComp", 'CompositorNodeTree')
scene.compositing_node_group = comp_tree
```

### Slotted Actions Pattern (4.4+)

```python
# Blender 3.x / 4.0–4.3 — Legacy action.fcurves
action = bpy.data.actions.new("MyAction")
fcurve = action.fcurves.new(data_path="location", index=0)

# Blender 4.4+ — Slotted Actions (legacy removed in 5.0)
action = bpy.data.actions.new("MyAction")
slot = action.slots.new()
layer = action.layers.new(name="Layer")
strip = layer.strips.new(type='KEYFRAME')
channelbag = strip.channelbags.new(slot=slot)
fcurve = channelbag.fcurves.new(data_path="location", index=0)
```

---

## 3. Common Operations — Migration Checklists

### 3.x → 4.0 Migration Checklist

- [ ] Replace ALL context override dicts with `context.temp_override()`
- [ ] Replace `mesh.edges[i].bevel_weight` with `mesh.attributes.get("bevel_weight_edge")`
- [ ] Replace `mesh.edges[i].crease` with `mesh.attributes.get("crease_edge")`
- [ ] Replace `obj.face_maps` with integer face attributes
- [ ] Replace `bone.layers[i]` with `bone.collections`
- [ ] Replace `pose.bone_groups` with bone collections + colors
- [ ] Replace `NodeTree.inputs/outputs.new()` with `NodeTree.interface.new_socket()`
- [ ] Update Principled BSDF socket names (`Subsurface` → `Subsurface Weight`, etc.)
- [ ] Replace `3D_UNIFORM_COLOR` with `POLYLINE_UNIFORM_COLOR`
- [ ] Replace `Mesh.calc_normals()` (removed, now auto-calculated)
- [ ] Replace Python OBJ/PLY IO with `bpy.ops.wm.obj_import/export`
- [ ] Remove `filename` params, use `filepath` instead
- [ ] Update `anim_utils.bake_action()` to use `BakeOptions` dataclass

### 4.0 → 4.1 Migration Checklist

- [ ] Remove `mesh.use_auto_smooth` and `mesh.auto_smooth_angle`
- [ ] Use `mesh.corner_normals` for normal access
- [ ] Use "Smooth by Angle" modifier for angle-based smoothing
- [ ] Rename light probe types: `CUBEMAP`→`SPHERE`, `PLANAR`→`PLANE`, `GRID`→`VOLUME`
- [ ] Move `mat.cycles.displacement_method` to `mat.displacement_method`
- [ ] Add type checking for `foreach_set()` calls (now raises `TypeError`)
- [ ] Rename sequencer filter `SUBSAMPLING_3x3` → `BOX`
- [ ] Use socket identifiers instead of indices for dynamic sockets

### 4.1 → 4.2 Migration Checklist

- [ ] Create `blender_manifest.toml` for extension system
- [ ] Handle statically typed IDProperties (no implicit type changes)
- [ ] Update EEVEE identifier: `BLENDER_EEVEE` → `BLENDER_EEVEE_NEXT`
- [ ] Move `scene.cycles.motion_blur_position` to `scene.render.motion_blur_position`
- [ ] Move `scene.eevee.use_motion_blur` to `scene.render.use_motion_blur`
- [ ] Replace `blend_method` with `surface_render_method`
- [ ] Replace `object.load_reference_image` with `object.empty_image_add`
- [ ] Update compositor mask node properties: `width`→`mask_width`, `height`→`mask_height`

### 4.2 → 4.3 Migration Checklist

- [ ] Replace `bpy.types.AttributeGroup` usage with type-specific classes
- [ ] Migrate ALL Grease Pencil code to new API
- [ ] Remove references to EEVEE legacy properties (contact shadows, SSR, bloom, etc.)
- [ ] Update reroute node handling to use `socket_idname` property
- [ ] Do NOT assign embedded IDs to `PointerProperty`

### 4.3 → 4.4 Migration Checklist

- [ ] Add `super().__init__(*args, **kwargs)` to ALL Blender type subclasses
- [ ] Rename `bpy.types.Sequence` references to `bpy.types.Strip`
- [ ] Adopt Slotted Actions API for new animation code
- [ ] Update `paint.brush` usage (now read-only property)
- [ ] Ensure `pos` attribute uses F32 FLOAT for GPU polyline drawing

### 4.x → 5.0 Migration Checklist

- [ ] Remove ALL `import bgl` — replace with `gpu` module
- [ ] Replace `scene['cycles']` dict access with attribute access
- [ ] Replace `scene.node_tree` with `scene.compositing_node_group`
- [ ] Replace `brush.sculpt_tool` with `brush.sculpt_brush_type`
- [ ] Replace legacy `action.fcurves` with Slotted Actions channelbag API
- [ ] Replace VSE `end_frame` with `length`
- [ ] Update EEVEE identifier from `BLENDER_EEVEE_NEXT` back to `BLENDER_EEVEE`
- [ ] Remove usage of bundled private modules (`animsys_refactor`, etc.)
- [ ] Replace `Image.bindcode` with `gpu.texture.from_image()`
- [ ] Replace `gpu.types.GPUShader()` constructor with `gpu.shader.create_from_info()`

### 5.0 → 5.1 Migration Checklist

- [ ] Replace `sculpt.sample_color` with `paint.sample_color`
- [ ] Update to Python 3.13 — check stdlib compatibility
- [ ] Prepare for VSE time property renames (deprecated, removed in 6.0):
  `frame_final_duration` → `duration`, `frame_final_start` → `left_handle`, etc.

---

## 4. Reference Links

- **[references/methods.md](references/methods.md)** — Complete version detection API: `bpy.app.version`, `bpy.app.version_string`, `bpy.app.version_cycle`
- **[references/examples.md](references/examples.md)** — Migration code examples for EACH version transition (3.x→4.0, 4.0→4.1, 4.1→4.2, 4.2→4.3, 4.3→4.4, 4.4→5.0, 5.0→5.1)
- **[references/anti-patterns.md](references/anti-patterns.md)** — Common version migration mistakes with explanations

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
- [Blender API Change Log](https://docs.blender.org/api/current/change_log.html)
