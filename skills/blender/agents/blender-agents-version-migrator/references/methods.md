# API Migration Mappings — Complete Reference

Complete API migration mappings organized by version transition. Each section lists every removed, renamed, or restructured API with its exact replacement.

---

## 3.x → 4.0 API Mappings

### Context Override System

| Removed API | Replacement | Notes |
|------------|-------------|-------|
| `bpy.ops.*(override_dict, ...)` | `with bpy.context.temp_override(**kwargs): bpy.ops.*(...)` | Dict override was the first positional arg |
| `context.copy()` for override dicts | `context.temp_override()` context manager | No manual dict construction needed |

### Mesh Data Properties

| Removed API | Replacement | Notes |
|------------|-------------|-------|
| `mesh.edges[i].bevel_weight` | `mesh.attributes.get("bevel_weight_edge").data[i].value` | Returns None if attribute not present |
| `mesh.edges[i].crease` | `mesh.attributes.get("crease_edge").data[i].value` | Returns None if attribute not present |
| `mesh.vertices[i].bevel_weight` | `mesh.attributes.get("bevel_weight_vert").data[i].value` | Vertex-level bevel weight |
| `obj.face_maps` | Integer face attributes via `mesh.attributes` | Create INT domain FACE attribute |
| `mesh.calc_normals()` | Remove entirely | Normals are auto-calculated in 4.0+ |
| `filename` parameter (IO ops) | `filepath` parameter | Affects all import/export operators |

### Armature / Bone System

| Removed API | Replacement | Notes |
|------------|-------------|-------|
| `bone.layers[i] = True` | `bone.collections["name"].assign(bone)` | 32 layers → unlimited named collections |
| `armature.layers[i]` | `armature.collections` | Armature-level layer access |
| `pose.bone_groups` | Bone collections with `palette.colors` | Color assignment via collection |
| `pose.bone_groups["name"].color_set` | `bone_collection.color_tag` | Predefined color themes |

### Node Interface

| Removed API | Replacement | Notes |
|------------|-------------|-------|
| `NodeTree.inputs.new(type, name)` | `NodeTree.interface.new_socket(name=name, in_out='INPUT', socket_type=type)` | Argument order changed |
| `NodeTree.outputs.new(type, name)` | `NodeTree.interface.new_socket(name=name, in_out='OUTPUT', socket_type=type)` | Argument order changed |
| `NodeTree.inputs.remove(socket)` | `NodeTree.interface.remove(item)` | Item from `interface.items_tree` |
| `NodeTree.inputs["name"]` | `NodeTree.interface.items_tree["name"]` | Access by name |

### Principled BSDF Socket Renames

| Old Socket Name | New Socket Name (4.0+) |
|----------------|----------------------|
| `Subsurface` | `Subsurface Weight` |
| `Subsurface Color` | `Subsurface Radius` (repurposed) |
| `Specular` | `Specular IOR Level` |
| `Specular Tint` | `Specular Tint` (now color input) |
| `Transmission` | `Transmission Weight` |
| `Sheen` | `Sheen Weight` |
| `Clearcoat` | `Coat Weight` |
| `Clearcoat Roughness` | `Coat Roughness` |
| `Clearcoat Normal` | `Coat Normal` |

### GPU Shader Renames

| Old Shader Name | New Shader Name (4.0+) |
|----------------|----------------------|
| `3D_UNIFORM_COLOR` | `POLYLINE_UNIFORM_COLOR` |
| `3D_FLAT_COLOR` | `POLYLINE_FLAT_COLOR` |
| `3D_SMOOTH_COLOR` | `SMOOTH_COLOR` |
| `3D_POLYLINE_UNIFORM_COLOR` | `POLYLINE_UNIFORM_COLOR` |
| `2D_UNIFORM_COLOR` | `UNIFORM_COLOR` |
| `2D_FLAT_COLOR` | `FLAT_COLOR` |
| `2D_SMOOTH_COLOR` | `SMOOTH_COLOR` |
| `2D_IMAGE` | `IMAGE` |

### IO Operators

| Removed API | Replacement |
|------------|-------------|
| `bpy.ops.import_scene.obj()` | `bpy.ops.wm.obj_import()` |
| `bpy.ops.export_scene.obj()` | `bpy.ops.wm.obj_export()` |
| `bpy.ops.import_mesh.ply()` | `bpy.ops.wm.ply_import()` |
| `bpy.ops.export_mesh.ply()` | `bpy.ops.wm.ply_export()` |
| `io_scene_obj` module | Removed (C++ implementation) |
| `io_mesh_ply` module | Removed (C++ implementation) |

### Animation

| Removed API | Replacement |
|------------|-------------|
| `anim_utils.bake_action(obj, ...)` (positional args) | `anim_utils.bake_action(obj, bake_options=BakeOptions(...))` |

---

## 4.0 → 4.1 API Mappings

### Mesh Normals

| Removed API | Replacement | Notes |
|------------|-------------|-------|
| `mesh.use_auto_smooth` | Remove (always active) | Auto-smooth is permanently enabled |
| `mesh.auto_smooth_angle` | "Smooth by Angle" geometry nodes modifier | Add modifier via bpy.ops |
| `mesh.calc_normals_split()` | Remove | Use `mesh.corner_normals` (auto-updated) |
| `mesh.create_normals_split()` | Remove | No longer needed |
| `mesh.free_normals_split()` | Remove | No longer needed |
| `MeshLoop.normal` (write) | `mesh.normals_split_custom_set()` | Read access still works |

### Light Probe Enum Renames

| Old Enum Value | New Enum Value (4.1+) |
|---------------|---------------------|
| `CUBEMAP` | `SPHERE` |
| `PLANAR` | `PLANE` |
| `GRID` | `VOLUME` |

### Material Properties

| Removed API | Replacement |
|------------|-------------|
| `mat.cycles.displacement_method` | `mat.displacement_method` |
| `probe.show_data` | `probe.use_data_display` |

### Sequencer

| Removed API | Replacement |
|------------|-------------|
| `SUBSAMPLING_3x3` filter | `BOX` filter |

---

## 4.1 → 4.2 API Mappings

### Extension System

| Legacy (3.x–4.1) | Extension System (4.2+) |
|-------------------|------------------------|
| `bl_info` dict in `__init__.py` | `blender_manifest.toml` file |
| `bl_info["name"]` | `name` field in TOML |
| `bl_info["description"]` | `tagline` field (max 64 chars) |
| `bl_info["blender"]` tuple | `blender_version_min` string |
| `bl_info["category"]` | `tags` array |
| `bl_info["version"]` | `version` string (semver) |

### Render Engine Identifier

| Old Identifier | New Identifier (4.2+) |
|---------------|---------------------|
| `BLENDER_EEVEE` | `BLENDER_EEVEE_NEXT` |

### Property Moves

| Old Location | New Location (4.2+) |
|-------------|-------------------|
| `scene.cycles.motion_blur_position` | `scene.render.motion_blur_position` |
| `scene.eevee.use_motion_blur` | `scene.render.use_motion_blur` |
| `blend_method` on material | `surface_render_method` |
| `object.load_reference_image` | `object.empty_image_add` |

### Compositor Mask Node

| Old Property | New Property (4.2+) |
|-------------|-------------------|
| `mask_node.width` | `mask_node.mask_width` |
| `mask_node.height` | `mask_node.mask_height` |

### IDProperties

| Old Behavior | New Behavior (4.2+) |
|-------------|-------------------|
| Dynamic type changes on IDProperties | `TypeError` raised — types are statically enforced |

---

## 4.2 → 4.3 API Mappings

### AttributeGroup Split

| Removed API | Replacement (4.3+) |
|------------|-------------------|
| `bpy.types.AttributeGroup` | `bpy.types.AttributeGroupMesh` |
| `bpy.types.AttributeGroup` | `bpy.types.AttributeGroupPointCloud` |
| `bpy.types.AttributeGroup` | `bpy.types.AttributeGroupCurves` |
| `bpy.types.AttributeGroup` | `bpy.types.AttributeGroupGreasePencil` |

Mesh-only properties (`active_color`, `active_color_index`, `default_color_name`, `render_color_index`) are accessible ONLY on `AttributeGroupMesh`.

### EEVEE Legacy Properties Removed

40+ EEVEE properties removed in 4.3. Key removals:

| Removed Property | Status |
|-----------------|--------|
| `scene.eevee.use_ssr` | Removed — engine-internal |
| `scene.eevee.use_bloom` | Removed — engine-internal |
| `scene.eevee.use_gtao` | Removed — engine-internal |
| `scene.eevee.use_volumetric_*` | Removed — engine-internal |
| `scene.eevee.bokeh_*` | Removed — engine-internal |
| `scene.eevee.shadow_*` | Removed — engine-internal |
| `scene.eevee.use_shadow_*` | Removed — engine-internal |
| `scene.eevee.use_soft_shadows` | Removed — engine-internal |

### Reroute Node

| Old Approach | New Approach (4.3+) |
|-------------|-------------------|
| Direct socket type modification on reroute | `reroute_node.socket_idname` property |

### Grease Pencil

The ENTIRE Grease Pencil Python API is rewritten in 4.3. There is NO simple mapping — a complete code rewrite is required. Consult the [Grease Pencil 4.3 Migration Guide](https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/).

---

## 4.3 → 4.4 API Mappings

### Class Initialization

| Old Pattern | New Pattern (4.4+) |
|------------|-------------------|
| `def __init__(self):` (no super) | `def __init__(self, *args, **kwargs): super().__init__(*args, **kwargs)` |

This applies to ALL `bpy.types` subclasses: Operator, Panel, Menu, Header, PropertyGroup, Node, etc.

### VSE Type Rename

| Old Type | New Type (4.4+) |
|----------|----------------|
| `bpy.types.Sequence` | `bpy.types.Strip` |
| All `Sequence` subclasses | Corresponding `Strip` subclasses |

### Slotted Actions (New in 4.4)

| Legacy API | Slotted Actions API (4.4+) |
|-----------|--------------------------|
| `action.fcurves.new(data_path, index)` | `channelbag.fcurves.new(data_path, index)` via slot/layer/strip/channelbag |
| Direct action.fcurves access | `action.slots[0]` → layer → strip → channelbag → fcurves |

### Paint System

| Old API | New API (4.4+) |
|---------|---------------|
| `paint.brush` (read-write) | `paint.brush` (read-only) |

### GPU Drawing

| Old Requirement | New Requirement (4.4+) |
|----------------|----------------------|
| `pos` attribute any type | `pos` attribute MUST use F32 FLOAT for polyline drawing |

---

## 4.4 → 4.5 API Mappings

Blender 4.5 LTS introduces minor deprecations only. No major API removals.

---

## 4.x → 5.0 API Mappings

### BGL Module Removal

| Removed API (bgl) | Replacement (gpu) |
|-------------------|-------------------|
| `import bgl` | `import gpu` |
| `bgl.glEnable(bgl.GL_BLEND)` | `gpu.state.blend_set('ALPHA')` |
| `bgl.glDisable(bgl.GL_BLEND)` | `gpu.state.blend_set('NONE')` |
| `bgl.glLineWidth(w)` | `gpu.state.line_width_set(w)` |
| `bgl.glEnable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('LESS_EQUAL')` |
| `bgl.glDisable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('NONE')` |
| `bgl.glDepthMask(GL_TRUE)` | `gpu.state.depth_mask_set(True)` |
| `bgl.glDepthMask(GL_FALSE)` | `gpu.state.depth_mask_set(False)` |
| `bgl.glPointSize(s)` | `gpu.state.point_size_set(s)` |
| `bgl.glEnable(bgl.GL_LINE_SMOOTH)` | Use `POLYLINE_*` shaders (no direct replacement) |
| `image.gl_load()` | `gpu.texture.from_image(image)` |
| `image.bindcode` | `gpu.texture.from_image(image)` |
| `gpu.types.GPUShader()` constructor | `gpu.shader.create_from_info()` |

### Compositor

| Removed API | Replacement (5.0+) |
|------------|-------------------|
| `scene.node_tree` (compositor) | `scene.compositing_node_group` |
| `scene.use_nodes` (compositor) | Remove — compositing always active |

### Properties

| Removed API | Replacement (5.0+) |
|------------|-------------------|
| `scene['cycles']` dict access | `scene.cycles.samples` (attribute access) |

### Sculpt / Brush

| Removed API | Replacement (5.0+) |
|------------|-------------------|
| `brush.sculpt_tool` | `brush.sculpt_brush_type` |

### VSE

| Removed API | Replacement (5.0+) |
|------------|-------------------|
| `strip.end_frame` | `strip.length` |

### Animation

| Removed API | Replacement (5.0+) |
|------------|-------------------|
| `action.fcurves` (legacy direct access) | Slotted Actions: `action.slots` → `layers` → `strips` → `channelbags` → `fcurves` |

### Render Engine

| Old Identifier | New Identifier (5.0+) |
|---------------|---------------------|
| `BLENDER_EEVEE_NEXT` | `BLENDER_EEVEE` (reverted to original name) |

### Bundled Modules

| Removed Module | Status (5.0+) |
|---------------|--------------|
| `animsys_refactor` | Removed — was private bundled module |
| Other `_` prefixed internal modules | Check before importing |

---

## 5.0 → 5.1 API Mappings

### Sculpt

| Old API | New API (5.1+) |
|---------|---------------|
| `sculpt.sample_color` | `paint.sample_color` |

### VSE Time Properties (Deprecated — Removed in 6.0)

| Deprecated Name | New Name (5.1+) |
|----------------|----------------|
| `frame_final_duration` | `duration` |
| `frame_final_start` | `left_handle` |
| `frame_final_end` | `right_handle` |
| `frame_start` | `media_start` |
| `frame_offset_start` | Removed concept |
| `frame_offset_end` | Removed concept |

### Python Version

| Blender 5.0 | Blender 5.1 |
|-------------|-------------|
| Python 3.11 | Python 3.13 |

Check stdlib compatibility — Python 3.12 removed `distutils`, Python 3.13 introduces further changes.
