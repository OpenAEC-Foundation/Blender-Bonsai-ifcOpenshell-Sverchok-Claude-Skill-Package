# Definitive Masterplan — Blender-Bonsai-IfcOpenShell Claude Skill Package

**Date**: 2026-03-06
**Phase**: 3 — Masterplan Refinement
**Status**: DEFINITIVE — This document drives all skill creation
**Author**: masterplan-refinement agent (Phase 3)

---

## 1. Executive Summary

### Skill Counts

| Package | Syntax | Impl | Errors | Core | Agents | Total |
|---------|--------|------|--------|------|--------|-------|
| Blender | 11 | 6 | 3 | 4 | 2 | **26** |
| IfcOpenShell | 4 | 9 | 3 | 2 | 1 | **19** |
| Bonsai | 4 | 7 | 1 | 1 | 1 | **14** |
| Cross-tech | — | — | — | 1 | 1 | **2** |
| **Total** | **19** | **22** | **7** | **8** | **5** | **61** |

### Build Timeline

- **13 batches**, 3–5 agents per batch
- **Batch 1–8**: Blender + IfcOpenShell in parallel (foundation → syntax → implementation → errors), Bonsai core starts in Batch 8
- **Batch 9–12**: Bonsai syntax + impl (depends on both Blender and IfcOpenShell)
- **Batch 13**: Agent skills + Bonsai errors (depend on everything)
- Estimated duration: 13 quality-gated batch cycles

### Scope Boundaries

**IN scope** (AEC-relevant):
- All Blender Python scripting for modeling, animation, rendering, materials, nodes, data management, GPU drawing, addons/extensions, I/O formats
- All IfcOpenShell API modules (35 modules), util modules (29 modules), geometry processing
- All Bonsai core BIM workflows: modeling, spatial, properties, drawing, QTO, BCF, clash detection, classification
- IFC schemas: IFC2X3, IFC4, IFC4X3
- Blender versions: 3.x, 4.0, 4.1, 4.2 LTS, 4.3, 4.4, 4.5, 5.0, 5.1

**OUT of scope** (not AEC-relevant):
- Blender: sculpting, video editing, motion tracking, Grease Pencil 2D animation, particle/cloth/fluid simulation, audio
- Bonsai: Brickschema (IoT), structural analysis, facility management
- IfcOpenShell: structural analysis modules

**DEFERRED** (later phase):
- Sverchok (all skills) — requires dedicated research phase
- IfcSverchok bridge — requires both Sverchok and IfcOpenShell skills first

### Key Decisions Made During Refinement

| Decision | Rationale |
|----------|-----------|
| Cross-tech reduced from 4 to 2 skills | `aec-core-ifc-fundamentals` redundant with `ifcos-core-concepts`; `aec-core-python-runtime` redundant with per-technology runtime skills |
| IfcOpenShell: `ifcos-syntax-geometry` merged into `ifcos-impl-geometry` | Geometry syntax inseparable from implementation; settings + create_shape + iterator belong together |
| IfcOpenShell: `ifcos-impl-extraction` merged into `ifcos-syntax-elements` + `ifcos-syntax-util` | Extraction is just traversal + utilities; no unique API surface |
| IfcOpenShell: `ifcos-impl-modification` merged into `ifcos-impl-creation` | Same API for create/modify (edit_pset, edit_attributes) |
| Bonsai: `bonsai-impl-export` merged into `bonsai-impl-project` | Bonsai is native IFC — no export concept; saving IS the workflow |
| Bonsai: `bonsai-errors-ifc` + `bonsai-errors-geometry` merged into `bonsai-errors-common` | One consolidated error skill with 31+ documented anti-patterns |
| Blender: `blender-errors-operators` merged into `blender-errors-context` | Operator errors are context errors (poll failures, wrong mode, restricted context) |
| Blender: Added `blender-syntax-materials` | Supplementary research revealed extensive AEC-relevant content: Principled BSDF renames, material assignment, UV mapping |
| Blender: Added `blender-core-runtime` | Supplementary research revealed mathutils, threading, undo, handlers, timers need dedicated skill |

---

## 2. Skill Inventory (DEFINITIVE)

### 2.1 Blender Skills (26)

#### C-01: blender-core-api
- **Category**: core
- **Scope**: bpy module overview (data/context/ops/types/props/utils/app), RNA introspection system, ID data blocks, context system (active object, mode, area), dependency graph (depsgraph) basics, operators vs direct data access.
- **Key API surface**: `bpy.data`, `bpy.context`, `bpy.ops`, `bpy.types`, `depsgraph`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (info_quickstart, info_overview)
- **Research input**: vooronderzoek-blender §1 (L29-78), §9 context, §10 depsgraph
- **Version coverage**: All versions — foundation concepts stable across 3.x/4.x/5.x
- **Dependencies**: None (foundation skill)
- **Complexity**: M

#### C-02: blender-core-versions
- **Category**: core
- **Scope**: Complete version matrix for Blender 3.x/4.0/4.1/4.2/4.3/4.4/4.5/5.0/5.1, ALL breaking changes with migration paths, version detection (`bpy.app.version`), version-safe coding patterns.
- **Key API surface**: `bpy.app.version`, version comparison patterns
- **SOURCES.md URLs**: https://developer.blender.org/docs/release_notes/4.0/python_api/, https://developer.blender.org/docs/release_notes/4.1/python_api/, https://developer.blender.org/docs/release_notes/4.2/python_api/, https://developer.blender.org/docs/release_notes/4.3/python_api/, https://developer.blender.org/docs/release_notes/4.4/python_api/, https://developer.blender.org/docs/release_notes/4.5/python_api/, https://developer.blender.org/docs/release_notes/5.0/python_api/, https://developer.blender.org/docs/release_notes/5.1/python_api/, https://developer.blender.org/docs/release_notes/compatibility/, https://docs.blender.org/api/current/change_log.html
- **Research input**: vooronderzoek-blender §2 (L81-446)
- **Version coverage**: 3.x through 5.1 — THIS IS the version reference skill
- **Dependencies**: None (reference skill)
- **Complexity**: L

#### C-03: blender-core-gpu
- **Category**: core
- **Scope**: gpu module (replaces bgl), built-in shaders (UNIFORM_COLOR, POLYLINE_*, SMOOTH_COLOR, IMAGE), batch_for_shader, gpu.state (blend, depth, line_width), SpaceView3D draw handlers, offscreen rendering, BGL→gpu migration checklist.
- **Key API surface**: `gpu.shader.from_builtin()`, `gpu_extras.batch.batch_for_shader()`, `gpu.state.*`, `gpu.types.GPUOffScreen`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (gpu, gpu.shader, gpu.types), https://developer.blender.org/docs/release_notes/5.0/python_api/
- **Research input**: vooronderzoek-blender §2 BGL migration (L311-435), supplementary-blender-gaps §7.8 (L1738-1829)
- **Version coverage**: bgl deprecated 3.5, removed 5.0; shader name `3D_UNIFORM_COLOR` → `POLYLINE_UNIFORM_COLOR` in 4.0
- **Dependencies**: blender-core-api
- **Complexity**: M

#### C-04: blender-core-runtime
- **Category**: core
- **Scope**: mathutils module (Vector, Matrix, Quaternion, Euler, KDTree, BVHTree, bl_math), Python threading restrictions, undo invalidation of bpy references, application handlers (@persistent), bpy.app.timers, bpy.msgbus, background mode limitations, bpy as module.
- **Key API surface**: `mathutils.*`, `bpy.app.timers`, `bpy.app.handlers`, `bpy.msgbus.subscribe_rna()`, `bpy.app.background`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (mathutils, bpy.app.timers, bpy.msgbus, bpy.app.handlers, info_gotchas)
- **Research input**: supplementary-blender-gaps §7 mathutils (L1468-1851), §8 runtime quirks (L1854-2193)
- **Version coverage**: 5.0 removes IDProperty dict access to RNA props; `del obj['prop']` → `property_unset()`
- **Dependencies**: blender-core-api
- **Complexity**: L

#### S-01: blender-syntax-operators
- **Category**: syntax
- **Scope**: bpy.types.Operator class, execute/invoke/modal methods, poll(), bl_idname naming, bl_options flags, return values, operator properties.
- **Key API surface**: `bpy.types.Operator`, `bpy.ops.*`, `context.window_manager.invoke_props_dialog()`, `modal_handler_add()`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/, https://developer.blender.org/docs/release_notes/4.0/python_api/
- **Research input**: vooronderzoek-blender §4 (L609-751)
- **Version coverage**: 4.0 removes context override dict (must use `temp_override`); 4.2 adds `MODAL_PRIORITY`
- **Dependencies**: blender-core-api
- **Complexity**: M

#### S-02: blender-syntax-properties
- **Category**: syntax
- **Scope**: All bpy.props types (Bool/Int/Float/String/Enum/Vector/Pointer/Collection), PropertyGroup, subtypes, units, update callbacks, dynamic enum items, getter/setter.
- **Key API surface**: `bpy.props.*`, `bpy.types.PropertyGroup`, `bpy.utils.register_class()`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (bpy.props)
- **Research input**: vooronderzoek-blender §5 (L754-915)
- **Version coverage**: 4.1 adds enum ID properties; 5.0 removes `del obj['prop']` for RNA props
- **Dependencies**: blender-syntax-operators
- **Complexity**: M

#### S-03: blender-syntax-panels
- **Category**: syntax
- **Scope**: bpy.types.Panel, draw(), UILayout API (row/column/box/split), bl_space_type, bl_region_type, bl_category, sub-panels, draw_header, menus, UIList.
- **Key API surface**: `bpy.types.Panel`, `bpy.types.UILayout`, `bpy.types.Menu`, `bpy.types.UIList`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §6 (L918-1000+)
- **Version coverage**: 4.1 adds `layout.panel()` for collapsible sections without registration
- **Dependencies**: blender-syntax-properties
- **Complexity**: M

#### S-04: blender-syntax-addons
- **Category**: syntax
- **Scope**: Legacy bl_info dict, blender_manifest.toml (4.2+), register/unregister, multi-file addon structure, AddonPreferences, class naming convention, extension packaging.
- **Key API surface**: `bpy.utils.register_class()`, `bpy.types.AddonPreferences`, `bpy.context.preferences.addons`
- **SOURCES.md URLs**: https://developer.blender.org/docs/handbook/extensions/, https://developer.blender.org/docs/handbook/extensions/addon_dev_setup/, https://developer.blender.org/docs/handbook/extensions/addon_guidelines/, https://developer.blender.org/docs/handbook/extensions/hosted/
- **Research input**: vooronderzoek-blender §3 (L450-606)
- **Version coverage**: 4.2 introduces blender_manifest.toml; legacy bl_info still works through 5.x
- **Dependencies**: blender-syntax-operators, blender-syntax-panels
- **Complexity**: M

#### S-05: blender-syntax-mesh
- **Category**: syntax
- **Scope**: Mesh data (vertices/edges/faces/loops), BMesh creation and editing, from_pydata, UV layers, vertex colors/attributes, normals, foreach_get/set.
- **Key API surface**: `bpy.types.Mesh`, `bmesh`, `bmesh.from_mesh()`, `bmesh.to_mesh()`, `mesh.from_pydata()`, `mesh.attributes`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §7 (L1000+)
- **Version coverage**: 4.0 removes bevel_weight/crease properties (→ attributes); 4.1 removes use_auto_smooth/calc_normals_split; 4.3 splits AttributeGroup
- **Dependencies**: blender-core-api
- **Complexity**: L

#### S-06: blender-syntax-modifiers
- **Category**: syntax
- **Scope**: Modifier stack, obj.modifiers, apply vs preview, evaluated mesh via depsgraph, Geometry Nodes modifier input identifiers, common modifiers for AEC.
- **Key API surface**: `obj.modifiers.new()`, `bpy.ops.object.modifier_apply()`, `obj.evaluated_get()`, `depsgraph`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §8 (L1050+), supplementary-blender-gaps §1.5 (L165-186)
- **Version coverage**: 4.0 requires `temp_override` for modifier_apply; GN modifier inputs use auto-generated `Socket_N` identifiers
- **Dependencies**: blender-core-api, blender-syntax-nodes
- **Complexity**: M

#### S-07: blender-syntax-nodes
- **Category**: syntax
- **Scope**: Node tree architecture (NodeTree, Node, NodeLink, NodeSocket), Geometry Nodes via Python, Shader Nodes scripting, Compositor Nodes, node group interface API, socket types. Covers ALL node systems.
- **Key API surface**: `bpy.data.node_groups`, `NodeTree.interface.new_socket()`, `tree.nodes.new()`, `tree.links.new()`, `NodeTreeInterfacePanel`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (NodeTree, NodeTreeInterface, NodeSocket), https://developer.blender.org/docs/release_notes/4.0/python_api/, https://developer.blender.org/docs/release_notes/5.0/python_api/
- **Research input**: supplementary-blender-gaps §1 (L24-252), vooronderzoek-blender §2 node interface migration (L134-143)
- **Version coverage**: 4.0 moves interface from `tree.inputs/outputs` to `tree.interface.new_socket()`; 4.1 adds interface panels; 5.0 changes compositor from `scene.node_tree` to `scene.compositing_node_group`
- **Dependencies**: blender-core-api
- **Complexity**: L

#### S-08: blender-syntax-animation
- **Category**: syntax
- **Scope**: Keyframe insertion, FCurves (access, creation, update), keyframe interpolation, Drivers (variables, expressions), Constraints, Armatures/Bones/Bone Collections, NLA strips, Actions, Shape Keys.
- **Key API surface**: `keyframe_insert()`, `FCurve`, `Driver`, `driver_add()`, `obj.constraints.new()`, `armature.edit_bones`, `armature.collections`, `NlaTrack`, `ShapeKey`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (FCurve, Keyframe, BoneCollection, EditBone), https://developer.blender.org/docs/release_notes/4.0/python_api/
- **Research input**: supplementary-blender-gaps §2 (L255-583)
- **Version coverage**: 4.0 removes bone.layers and pose.bone_groups (→ bone collections); edit_bones only in Edit Mode
- **Dependencies**: blender-core-api, blender-syntax-properties
- **Complexity**: L

#### S-09: blender-syntax-materials
- **Category**: syntax
- **Scope**: Material creation/assignment, Principled BSDF setup (with 3.x→4.0+ socket rename table), shader node tree building, UV mapping via Python, image texture loading, color space settings, AEC material presets.
- **Key API surface**: `bpy.data.materials.new()`, `mat.node_tree`, `ShaderNodeBsdfPrincipled`, `mesh.uv_layers`, `bpy.data.images.load()`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (Material, ShaderNodeBsdfPrincipled, UVLoopLayers), https://developer.blender.org/docs/release_notes/4.0/python_api/
- **Research input**: supplementary-blender-gaps §3 (L586-795)
- **Version coverage**: 4.0 renames Principled BSDF sockets (Subsurface→Subsurface Weight, etc.); 4.2 removes `mat.blend_method`
- **Dependencies**: blender-syntax-nodes
- **Complexity**: M

#### S-10: blender-syntax-rendering
- **Category**: syntax
- **Scope**: Scene.render settings, EEVEE vs Cycles configuration via Python, camera setup (lens, DOF, clipping), light creation (Point/Sun/Spot/Area), output format, batch rendering, headless rendering, render passes.
- **Key API surface**: `scene.render`, `scene.cycles`, `scene.eevee`, `bpy.data.cameras`, `bpy.data.lights`, `bpy.ops.render.render()`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (RenderSettings, SceneEEVEE), https://developer.blender.org/docs/release_notes/5.0/python_api/
- **Research input**: supplementary-blender-gaps §4 (L799-1062)
- **Version coverage**: EEVEE identifier: `BLENDER_EEVEE` (3.x/4.0-4.1) → `BLENDER_EEVEE_NEXT` (4.2-4.4) → `BLENDER_EEVEE` (5.0+); 5.0 renames render pass names
- **Dependencies**: blender-core-api
- **Complexity**: M

#### S-11: blender-syntax-data
- **Category**: syntax
- **Scope**: Collection management (hierarchy, visibility), library linking/appending, library overrides (4.0+), Asset Browser API (mark, metadata, tags, catalogs), ID data lifecycle (user count, fake user).
- **Key API surface**: `bpy.data.collections`, `bpy.data.libraries.load()`, `obj.asset_mark()`, `AssetMetaData`, `IDOverrideLibrary`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (Collection, BlendDataLibraries, AssetMetaData), https://developer.blender.org/docs/release_notes/5.0/python_api/
- **Research input**: supplementary-blender-gaps §6 (L1245-1465), vooronderzoek-blender §1 data hierarchy (L47-66)
- **Version coverage**: 4.0 removes proxies (→ library overrides); 5.0 removes `context.asset_file_handle`
- **Dependencies**: blender-core-api
- **Complexity**: M

#### I-01: blender-impl-operators
- **Category**: impl
- **Scope**: When to use which operator type (simple/invoke/modal), undo/redo patterns, modal timer pattern, invoke with dialog, context.temp_override usage patterns, operator chaining.
- **Key API surface**: Same as S-01, focused on decision trees and workflow patterns
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §4 (L680-715), §9 context system
- **Version coverage**: Same as S-01
- **Dependencies**: blender-syntax-operators
- **Complexity**: M

#### I-02: blender-impl-addons
- **Category**: impl
- **Scope**: Full addon/extension development workflow, packaging for extensions.blender.org, multi-file structure, testing, migration from bl_info to blender_manifest.toml, wheel bundling.
- **Key API surface**: Same as S-04, focused on workflow
- **SOURCES.md URLs**: https://developer.blender.org/docs/handbook/extensions/, https://developer.blender.org/docs/handbook/extensions/addon_dev_setup/, https://developer.blender.org/docs/handbook/extensions/hosted/
- **Research input**: vooronderzoek-blender §3 (L450-606)
- **Version coverage**: 4.2+ extension system focus
- **Dependencies**: blender-syntax-addons, blender-syntax-operators, blender-syntax-panels
- **Complexity**: M

#### I-03: blender-impl-mesh
- **Category**: impl
- **Scope**: Mesh creation workflows, BMesh vs direct data access decision tree, performance patterns (foreach_get/set), from_pydata usage, mesh editing in edit mode vs object mode, attribute-based workflow (4.0+).
- **Key API surface**: Same as S-05, focused on workflows
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §7
- **Version coverage**: Same as S-05
- **Dependencies**: blender-syntax-mesh
- **Complexity**: M

#### I-04: blender-impl-automation
- **Category**: impl
- **Scope**: Batch operations, headless/background mode rendering, CLI Blender, I/O format scripting (FBX/glTF/USD/OBJ/STL export/import with AEC-relevant parameters), subprocess usage, bpy as module.
- **Key API surface**: `bpy.ops.wm.obj_export`, `bpy.ops.export_scene.fbx`, `bpy.ops.export_scene.gltf`, `bpy.ops.wm.usd_export`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (export_scene, import_scene, wm)
- **Research input**: supplementary-blender-gaps §5 I/O formats (L1065-1242), §8.7-8.8 background mode (L2116-2170)
- **Version coverage**: 4.0 removes Python OBJ/STL exporters (→ `bpy.ops.wm.obj_export/stl_export`)
- **Dependencies**: blender-syntax-rendering, blender-core-runtime
- **Complexity**: L

#### I-05: blender-impl-nodes
- **Category**: impl
- **Scope**: Building Geometry Node groups via Python for AEC (wall generators, parametric structures), procedural material creation, modifier input value setting, node group reuse patterns.
- **Key API surface**: Same as S-07, focused on AEC workflows
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: supplementary-blender-gaps §1 (L24-252), §3.4 AEC materials (L706-725)
- **Version coverage**: Same as S-07
- **Dependencies**: blender-syntax-nodes, blender-syntax-materials, blender-syntax-modifiers
- **Complexity**: L

#### I-06: blender-impl-animation
- **Category**: impl
- **Scope**: Rigging workflows for AEC (building rigs, floor controllers), constraint setup patterns, driver expressions, animation baking, NLA workflow, shape key-driven parametric deformation.
- **Key API surface**: Same as S-08, focused on AEC workflows
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: supplementary-blender-gaps §2 (L255-583)
- **Version coverage**: Same as S-08
- **Dependencies**: blender-syntax-animation
- **Complexity**: M

#### E-01: blender-errors-context
- **Category**: errors
- **Scope**: Context override errors, wrong context for operators, restricted context, temp_override patterns, poll() failures and diagnosis, operator return value errors, registration errors, wrong bl_options. Merged: operator errors are context errors.
- **Key API surface**: `bpy.context`, `bpy.ops`, `poll()`, `temp_override()`
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §9 context system, §11 common errors, §12 AI mistakes
- **Version coverage**: 4.0 context override removal is the #1 breaking change
- **Dependencies**: blender-syntax-operators, blender-core-api
- **Complexity**: M

#### E-02: blender-errors-data
- **Category**: errors
- **Scope**: Orphaned data, reference counting, memory management, stale references after undo, undo invalidation (the #1 runtime danger), ID data lifecycle errors.
- **Key API surface**: `bpy.data`, object references, undo system
- **SOURCES.md URLs**: https://docs.blender.org/api/current/ (info_gotchas)
- **Research input**: vooronderzoek-blender §11 (error patterns), supplementary-blender-gaps §8.2 undo invalidation (L1903-1943)
- **Version coverage**: All versions — undo invalidation is universal
- **Dependencies**: blender-core-api, blender-core-runtime
- **Complexity**: M

#### E-03: blender-errors-version
- **Category**: errors
- **Scope**: Version-specific pitfalls: BGL→gpu (5.0), bone.layers→collections (4.0), EEVEE identifier changes, Principled BSDF renames (4.0), GP rewrite (4.3), mesh attribute migration (4.0), auto_smooth removal (4.1), OBJ/STL operator migration (4.0).
- **Key API surface**: All deprecated/removed APIs with migration paths
- **SOURCES.md URLs**: All version-specific release notes (4.0 through 5.1), https://developer.blender.org/docs/release_notes/compatibility/
- **Research input**: vooronderzoek-blender §2 version matrix (L81-446), supplementary-blender-gaps §4.9 rendering (L1045-1056)
- **Version coverage**: 3.x→5.1 complete migration guide
- **Dependencies**: blender-core-versions
- **Complexity**: L

#### A-01: blender-agents-code-validator
- **Category**: agents
- **Scope**: Validate Blender Python scripts for: correct context usage, API version compatibility, deprecated API calls, anti-pattern detection, naming convention compliance.
- **Key API surface**: All Blender APIs (validation checklist)
- **SOURCES.md URLs**: https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-blender §11-12 (error patterns, AI mistakes)
- **Version coverage**: All versions
- **Dependencies**: All syntax + error skills
- **Complexity**: M

#### A-02: blender-agents-version-migrator
- **Category**: agents
- **Scope**: Auto-detect version issues in scripts, suggest migration from 3.x→4.x→5.x, flag deprecated API calls, provide replacement code, version-safe wrapper generation.
- **Key API surface**: All deprecated/changed APIs
- **SOURCES.md URLs**: All version-specific release notes
- **Research input**: vooronderzoek-blender §2 (L81-446), blender-core-versions
- **Version coverage**: 3.x→5.1
- **Dependencies**: blender-core-versions, blender-errors-version
- **Complexity**: M

---

### 2.2 IfcOpenShell Skills (19)

#### ifcos-core-concepts
- **Category**: core
- **Scope**: IFC fundamentals — entity hierarchy (IfcRoot→IfcObjectDefinition→IfcObject→IfcProduct→IfcElement), spatial structure, ownership model, placement system, representation system, relationship model.
- **Key API surface**: `entity.is_a()`, inheritance chains, `IfcProject/IfcSite/IfcBuilding/IfcBuildingStorey` hierarchy, `IfcFacility` (IFC4X3)
- **SOURCES.md URLs**: https://ifc43-docs.standards.buildingsmart.org/, https://technical.buildingsmart.org/standards/ifc/, https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/
- **Research input**: vooronderzoek-ifcopenshell §8 (L903-1093), fragments/ifcos-schema-versions §2 (L45-220), topic-research/ifcos-schema-version-comparison §2-4 (L50-421)
- **Version coverage**: IFC2X3, IFC4, IFC4X3 — entity hierarchy differs significantly
- **Dependencies**: None
- **Complexity**: M

#### ifcos-core-runtime
- **Category**: core
- **Scope**: Python runtime quirks — C++ binding behavior, entity identity (`is` vs `==`), entity invalidation after removal, `by_type()` returns tuple, thread safety (single-threaded writes only), memory management, attribute access patterns (PascalCase), installation.
- **Key API surface**: `ifcopenshell.entity_instance`, `model.by_type()` index behavior, `model.remove()` invalidation, `model.schema`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html, https://pypi.org/project/ifcopenshell/
- **Research input**: supplementary-ifcos-gaps §7 (L1115-1360), topic-research/ifcos-errors-performance-research §3-4 (L949-1390)
- **Version coverage**: All schemas — runtime behavior is schema-agnostic
- **Dependencies**: ifcos-syntax-fileio
- **Complexity**: M

#### ifcos-syntax-fileio
- **Category**: syntax
- **Scope**: File I/O operations — open, create, write, serialize IFC files. Transactions (begin_transaction/end_transaction/undo/redo).
- **Key API surface**: `ifcopenshell.open()`, `ifcopenshell.file()`, `ifcopenshell.api.project.create_file()`, `file.write()`, `file.to_string()`, `file.add()`, `file.remove()`, `begin_transaction`/`end_transaction`/`undo`/`redo`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html, https://docs.ifcopenshell.org/ifcopenshell-python/hello_world.html
- **Research input**: vooronderzoek-ifcopenshell §2 (L90-165), fragments/ifcos-core-operations §1-3 (L22-245), topic-research/ifcos-core-operations §1 (L10-290)
- **Version coverage**: All schemas — schema string in `file()` constructor
- **Dependencies**: None
- **Complexity**: S

#### ifcos-syntax-api
- **Category**: syntax
- **Scope**: `ifcopenshell.api.run()` / direct module calls — all 30+ API modules, parameter conventions, invocation patterns.
- **Key API surface**: `ifcopenshell.api.run("module.function", model, **kwargs)` and direct `ifcopenshell.api.module.function(model, ...)`. Modules: root, spatial, aggregate, geometry, type, pset, material, context, unit, owner, classification, cost, sequence, system, group, void, boundary, structural, document, drawing, nest, layer, profile, style, constraint, resource, feature, alignment, georeference, project, attribute, pset_template
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- **Research input**: vooronderzoek-ifcopenshell §3 (L167-358), fragments/ifcos-api-categories full file (L1-1170), topic-research/ifcos-errors-performance-research §5 (L1393-1498)
- **Version coverage**: Most modules work across schemas; `api.run()` handles schema differences internally
- **Dependencies**: ifcos-syntax-fileio
- **Complexity**: M

#### ifcos-syntax-elements
- **Category**: syntax
- **Scope**: Element traversal and querying — by_type, by_id, by_guid, inverse references, entity attributes, get_info(), is_a(), GUID utilities.
- **Key API surface**: `model.by_type()`, `model.by_id()`, `model.by_guid()`, `model.get_inverse()`, `model.traverse()`, `entity.get_info()`, `entity.is_a()`, `entity.id()`, `ifcopenshell.guid.new()`, `ifcopenshell.guid.expand()`, `ifcopenshell.guid.compress()`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html
- **Research input**: vooronderzoek-ifcopenshell §4 (L361-424), §8 (L903-1093), topic-research/ifcos-core-operations §2 (L292-629)
- **Version coverage**: Low schema sensitivity — query API is schema-agnostic; entity class names differ per schema
- **Dependencies**: ifcos-syntax-fileio
- **Complexity**: S

#### ifcos-syntax-util
- **Category**: syntax
- **Scope**: All `ifcopenshell.util.*` modules — element, unit, placement, selector, date, shape_builder, schema, classification, geolocation, representation, attribute, cost, sequence, system.
- **Key API surface**: `util.element.get_psets()`, `get_type()`, `get_container()`, `get_material()`, `get_decomposition()`, `get_aggregate()`; `util.unit.calculate_unit_scale()`; `util.placement.get_local_placement()`; `util.selector.filter_elements()`; `util.date.ifc2datetime()`/`datetime2ifc()`; `util.shape_builder.ShapeBuilder`; `util.schema.get_entity_attributes()`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html
- **Research input**: vooronderzoek-ifcopenshell §5 (L427-565), topic-research/ifcos-core-operations §3 (L631-1086)
- **Version coverage**: Medium — `util.element` abstracts schema differences (recommended approach)
- **Dependencies**: ifcos-syntax-fileio
- **Complexity**: M

#### ifcos-impl-creation
- **Category**: impl
- **Scope**: Creating valid IFC files from scratch — minimal valid file pattern (7-step bootstrap), spatial hierarchy setup (Project→Site→Building→Storey), element creation with geometry, property assignment. Also covers modification patterns (edit_pset, edit_attributes).
- **Key API surface**: `root.create_entity`, `unit.assign_unit`, `context.add_context`, `aggregate.assign_object`, `spatial.assign_container`, `geometry.add_wall_representation`, `geometry.assign_representation`, `geometry.edit_object_placement`, `pset.add_pset`, `pset.edit_pset`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html, https://academy.ifcopenshell.org/posts/creating-a-simple-wall-with-property-set-and-quantity-information/
- **Research input**: vooronderzoek-ifcopenshell §10 (L1326-1471), fragments/ifcos-api-categories §Complete Example (L1109-1154)
- **Version coverage**: HIGH — OwnerHistory mandatory in IFC2X3; StandardCase entities differ per schema
- **Dependencies**: ifcos-syntax-fileio, ifcos-syntax-api
- **Complexity**: M

#### ifcos-impl-geometry
- **Category**: impl
- **Scope**: Geometry creation and processing — wall/slab/profile/mesh representations, geometry settings, create_shape, iterator, BRep vs tessellation, ShapeBuilder. Includes geometry syntax (settings, create_shape, iterator).
- **Key API surface**: `ifcopenshell.geom.settings()`, `ifcopenshell.geom.create_shape()`, `ifcopenshell.geom.iterator()`, `geometry.add_wall_representation`, `geometry.add_mesh_representation`, `geometry.add_profile_representation`, `geometry.add_slab_representation`, `geometry.add_boolean`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/ifcopenshell-python/geometry_settings.html, https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html, https://docs.ifcopenshell.org/ifcopenshell-python/geometry_creation.html
- **Research input**: vooronderzoek-ifcopenshell §6 (L570-710), fragments/ifcos-api-categories §3 geometry (L241-425), topic-research/ifcos-core-operations §4 (L1088-1363)
- **Version coverage**: Low — geometry API is mostly schema-agnostic
- **Dependencies**: ifcos-syntax-api, ifcos-impl-creation
- **Complexity**: L

#### ifcos-impl-materials
- **Category**: impl
- **Scope**: Material assignment — single materials, layer sets (walls/slabs), profile sets (beams/columns), constituent sets (doors/windows), material lists.
- **Key API surface**: `material.add_material`, `material.add_material_set`, `material.assign_material`, `material.add_layer`, `material.edit_layer`, `material.add_constituent`, `material.add_profile`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/material/index.html
- **Research input**: vooronderzoek-ifcopenshell §3 material (L312-341), fragments/ifcos-api-categories §6 (L639-752)
- **Version coverage**: Medium — `IfcMaterialConstituentSet` is IFC4+; `IfcMaterialList` is legacy IFC2X3
- **Dependencies**: ifcos-syntax-api
- **Complexity**: M

#### ifcos-impl-relationships
- **Category**: impl
- **Scope**: IFC relationship types — aggregation, containment, type assignment, property assignment, material association, voids/fillings, nesting, grouping.
- **Key API surface**: `aggregate.assign_object`, `spatial.assign_container`, `spatial.reference_structure`, `type.assign_type`, `pset.add_pset`/`assign_pset`, `void.add_opening`, `void.add_filling`, `nest.assign_object`, `group.add_group`/`assign_group`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/spatial/index.html, https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/aggregate/index.html, https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/type/index.html
- **Research input**: vooronderzoek-ifcopenshell §9 (L1096-1315), fragments/ifcos-schema-versions §4 (L299-435), topic-research/ifcos-schema-version-comparison §6 (L460-885)
- **Version coverage**: HIGH — `IfcRelDecomposes` split in IFC4; `IfcRelPositions` new in IFC4X3
- **Dependencies**: ifcos-syntax-api, ifcos-syntax-elements
- **Complexity**: M

#### ifcos-impl-cost
- **Category**: impl
- **Scope**: Cost management — cost schedules, cost items (WBS), cost values (direct/unit/formula), parametric quantity linking.
- **Key API surface**: `ifcopenshell.api.cost.*` (20 functions), `ifcopenshell.util.cost.*` (12 functions)
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/cost/index.html
- **Research input**: supplementary-ifcos-gaps §1 (L23-209)
- **Version coverage**: Low — supported across all schemas
- **Dependencies**: ifcos-syntax-api
- **Complexity**: M

#### ifcos-impl-sequence
- **Category**: impl
- **Scope**: 4D scheduling — work schedules, tasks (WBS), task time/duration, dependencies (FS/SS/FF/SF), work calendars, task-product relationships, cascade_schedule.
- **Key API surface**: `ifcopenshell.api.sequence.*` (40 functions), `ifcopenshell.util.sequence.*` (16 functions)
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/sequence/index.html
- **Research input**: supplementary-ifcos-gaps §2 (L211-491)
- **Version coverage**: Medium — IFC2X3 uses `IfcDateAndTime` entities; IFC4+ uses ISO 8601 strings
- **Dependencies**: ifcos-syntax-api
- **Complexity**: L

#### ifcos-impl-mep
- **Category**: impl
- **Scope**: MEP systems — distribution systems, port-based connectivity, flow segments/fittings/terminals, system traversal.
- **Key API surface**: `ifcopenshell.api.system.*` (12 functions), `ifcopenshell.util.system.*` (8 functions)
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/system/index.html
- **Research input**: supplementary-ifcos-gaps §3 (L493-668)
- **Version coverage**: Medium — IFC2X3 uses `IfcSystem`; IFC4+ uses `IfcDistributionSystem`
- **Dependencies**: ifcos-syntax-api, ifcos-impl-relationships
- **Complexity**: M

#### ifcos-impl-profiles
- **Category**: impl
- **Scope**: Cross-section profiles — 14 parametric profile types (I-shape, rectangle, circle, etc.), arbitrary profiles, profiles in geometry.
- **Key API surface**: `ifcopenshell.api.profile.*` (6 functions): `add_parameterised_profile`, `add_arbitrary_profile`, `add_arbitrary_profile_with_voids`, `edit_profile`, `remove_profile`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/profile/index.html
- **Research input**: supplementary-ifcos-gaps §5 (L746-908)
- **Version coverage**: Low — profiles work across all schemas
- **Dependencies**: ifcos-syntax-api, ifcos-impl-geometry
- **Complexity**: S

#### ifcos-impl-validation
- **Category**: impl
- **Scope**: IFC file validation and georeferencing — `ifcopenshell.validate` module (validate, validate_guid, validate_ifc_header, validate_ifc_applications, assert_valid, assert_valid_inverse), json_logger, LogDetectionHandler, EXPRESS WHERE rules; `ifcopenshell.api.georeference` module (add_georeferencing, edit_georeferencing, edit_true_north, edit_wcs, remove_georeferencing), EPSG codes, IFC2X3 vs IFC4+ differences.
- **Key API surface**: `ifcopenshell.validate.validate()`, `ifcopenshell.validate.json_logger()`, `ifcopenshell.validate.assert_valid()`, `ifcopenshell.api.georeference.add_georeferencing()`, `ifcopenshell.api.georeference.edit_georeferencing()`, `ifcopenshell.api.georeference.edit_true_north()`, `ifcopenshell.api.georeference.edit_wcs()`, `ifcopenshell.api.georeference.remove_georeferencing()`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/autoapi/ifcopenshell/validate/index.html, https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/georeference/index.html
- **Research input**: supplementary-ifcos-gaps §6 (L911-1112)
- **Version coverage**: Medium — georeference API differs between IFC2X3 and IFC4+; validate works across all schemas
- **Dependencies**: ifcos-syntax-api, ifcos-syntax-fileio
- **Complexity**: S

#### ifcos-errors-schema
- **Category**: errors
- **Scope**: Schema version pitfalls — entity availability per schema, attribute changes, StandardCase removal in IFC4X3, OwnerHistory requirements, entity renames, migration patterns with ifcpatch.
- **Key API surface**: `model.schema`, `ifcopenshell.ifcopenshell_wrapper.schema_by_name()`, `ifcpatch.execute()`, schema introspection helpers
- **SOURCES.md URLs**: https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/, https://ifc43-docs.standards.buildingsmart.org/, https://docs.ifcopenshell.org/ifcpatch.html
- **Research input**: vooronderzoek-ifcopenshell §7 (L713-900), §11 errors 1-2 (L1479-1503), fragments/ifcos-schema-versions full file (L1-834), topic-research/ifcos-schema-version-comparison full file (L1-1485)
- **Version coverage**: All three schemas — THIS IS the schema error skill
- **Dependencies**: ifcos-syntax-fileio
- **Complexity**: L

#### ifcos-errors-patterns
- **Category**: errors
- **Scope**: Common error patterns and anti-patterns — 10 documented errors (relationship errors, unit conversion, GUID errors, pset errors, geometry failures, file corruption), correct vs incorrect code pairs.
- **Key API surface**: Diagnostic: `model.schema`, `util.unit.calculate_unit_scale()`, `util.element.get_psets()`, `ifcopenshell.guid.new()`. Recovery: `root.remove_product` (not `model.remove`), batch `assign_container` (not loops)
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/ifcopenshell-python.html
- **Research input**: vooronderzoek-ifcopenshell §11 (L1475-1611), fragments/ifcos-errors-performance §1 (L18-451), topic-research/ifcos-errors-performance-research §1 (L9-639)
- **Version coverage**: Medium — several errors are schema-specific
- **Dependencies**: ifcos-syntax-api, ifcos-syntax-elements
- **Complexity**: M

#### ifcos-errors-performance
- **Category**: errors
- **Scope**: Performance optimization for large models — memory usage patterns (RAM = 10-20x file size), batch vs individual operations, iterator vs create_shape (5-10x speedup), memory management for 100MB+ files, threading constraints.
- **Key API surface**: `ifcopenshell.geom.iterator()` with multiprocessing, `model.by_type(include_subtypes=False)`, batch `assign_container(products=[...])`, `gc.collect()`, `model.garbage_collect()`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html
- **Research input**: vooronderzoek-ifcopenshell §12 (L1615-1703), fragments/ifcos-errors-performance §2 (L454-665), topic-research/ifcos-errors-performance-research §2 (L643-944)
- **Version coverage**: Low — performance patterns are schema-agnostic
- **Dependencies**: ifcos-syntax-fileio, ifcos-impl-geometry
- **Complexity**: M

#### ifcos-agents-code-validator
- **Category**: agents
- **Scope**: Validate IfcOpenShell scripts — check for hallucinated APIs, wrong parameter names, schema compatibility, missing spatial hierarchy, missing context, missing unit setup, correct use of api.run() vs create_entity.
- **Key API surface**: All — validation checklist references all API surfaces
- **SOURCES.md URLs**: All IfcOpenShell URLs
- **Research input**: vooronderzoek-ifcopenshell §13 AI Mistakes (L1706-1735), fragments/ifcos-errors-performance §3 (L668-828), topic-research/ifcos-errors-performance-research §3 (L949-1257), §5 Quick Reference (L1393-1498), §6 Critical Rules (L1502-1519)
- **Version coverage**: All schemas — must validate schema-appropriate usage
- **Dependencies**: All other ifcos skills
- **Complexity**: L

---

### 2.3 Bonsai Skills (14)

#### bonsai-core-architecture
- **Category**: core
- **Scope**: Three-layer architecture (UI/Core/Tool), `@interface` decorator pattern, IfcStore singleton, module registration system, demo module as reference, dependency injection.
- **Key API surface**: `bonsai.bim.ifc.IfcStore`, `bonsai.core.tool.@interface`, `bonsai.tool.*` classes, `bim/module/` structure, `IfcStore.execute_ifc_operator()`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (core/, tool/, bim/ifc.py)
- **Research input**: vooronderzoek-bonsai §2 (Architecture), supplementary-bonsai-gaps §5.1-5.2 (Tool/Core/UI Pattern, IfcStore Deep Dive)
- **Version coverage**: Bonsai v0.8.x (Blender 4.2+); module path `bonsai.*` (NEVER `blenderbim.*`)
- **Dependencies**: blender-core-api, blender-syntax-operators, blender-syntax-addons
- **Complexity**: L

#### bonsai-syntax-elements
- **Category**: syntax
- **Scope**: Creating/editing IFC elements via Bonsai, `root.create_entity`, class assignment (`bim.assign_class`), element↔object linking, `tool.Ifc.get_entity()/get_object()/link()/run()`.
- **Key API surface**: `tool.Ifc.get()`, `tool.Ifc.run()`, `tool.Ifc.get_entity(obj)`, `tool.Ifc.get_object(entity)`, `bpy.ops.bim.assign_class`, `IfcStore.id_map`, `IfcStore.guid_map`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/ifc.py, tool/ifc.py)
- **Research input**: vooronderzoek-bonsai §3 (IFC Integration), §7 (Modeling Workflow), §12 (API Reference)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-core-architecture, ifcos-syntax-api, ifcos-syntax-elements
- **Complexity**: M

#### bonsai-syntax-spatial
- **Category**: syntax
- **Scope**: Spatial hierarchy (Project/Site/Building/Storey/Space), `IfcRelAggregates`, `IfcRelContainedInSpatialStructure`, container assignment, spatial decomposition queries.
- **Key API surface**: `tool.Spatial`, `bonsai.core.spatial`, `spatial.assign_container`, `aggregate.assign_object`, `ifcopenshell.util.element.get_container()`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (tool/spatial.py, core/spatial.py, bim/module/spatial/)
- **Research input**: vooronderzoek-bonsai §4 (Spatial Structure Management)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Complexity**: M

#### bonsai-syntax-properties
- **Category**: syntax
- **Scope**: Property sets (`IfcPropertySet`), quantity sets (`IfcElementQuantity`), pset templates, naming conventions, reading psets, type-driven property inheritance.
- **Key API surface**: `pset.add_pset`, `pset.edit_pset`, `pset.add_qto`, `pset.edit_qto`, `ifcopenshell.util.element.get_psets()`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/pset/)
- **Research input**: vooronderzoek-bonsai §5 (Property Sets and Quantity Sets)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Complexity**: M

#### bonsai-syntax-geometry
- **Category**: syntax
- **Scope**: Representation contexts (Model/Body/Axis/Box/FootPrint, Plan/Annotation), representation types, ShapeBuilder, geometry API, opening voids.
- **Key API surface**: `geometry.add_wall_representation`, `geometry.assign_representation`, `context.add_context`, `ifcopenshell.util.representation.get_context()`, `ShapeBuilder`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/geometry/)
- **Research input**: vooronderzoek-bonsai §9 (Geometry Representations), §7 (Openings)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Complexity**: M

#### bonsai-impl-project
- **Category**: impl
- **Scope**: New IFC project creation, schema selection, MVD, save workflow (native IFC), header metadata, IDS/IfcTester validation, schema migration, georeference setup (add_georeferencing, edit_georeferencing, edit_true_north, edit_wcs, EPSG codes), owner/history tracking.
- **Key API surface**: `bpy.ops.bim.save_project`, `project.edit_header`, `ifctester`, `ifcopenshell.util.schema.Migrator`, `ifcopenshell.api.georeference.add_georeferencing()`, `ifcopenshell.api.georeference.edit_georeferencing()`
- **SOURCES.md URLs**: https://docs.bonsaibim.org/, https://docs.ifcopenshell.org/ifctester.html, https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/project/), https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/georeference/index.html
- **Research input**: vooronderzoek-bonsai §10 (IFC Export Pipeline/Saving), supplementary-ifcos-gaps §6.2 (L1021-1112, ifcopenshell.api.georeference — 5 functions, EPSG codes, IFC2X3 vs IFC4+ differences, anti-patterns)
- **Version coverage**: Bonsai v0.8.x, IFC2X3/IFC4/IFC4X3
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements, ifcos-syntax-fileio
- **Complexity**: M

#### bonsai-impl-modeling
- **Category**: impl
- **Scope**: Wall/slab/column/beam creation, type system, material assignment (layer sets, profile sets, constituent sets), opening creation and filling, predefined types.
- **Key API surface**: `bpy.ops.bim.assign_class`, `type.assign_type`, `material.add_material_set`, `material.add_layer`, `feature.add_feature`, `void.add_filling`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/model/, module/type/, module/material/)
- **Research input**: vooronderzoek-bonsai §6 (Type System), §7 (Modeling Workflow)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-syntax-elements, bonsai-syntax-spatial, bonsai-syntax-geometry, ifcos-syntax-api
- **Complexity**: L

#### bonsai-impl-classification
- **Category**: impl
- **Scope**: Built-in classification systems (Uniclass 2015, OmniClass), custom systems (NL-SfB, MasterFormat), bSDD integration.
- **Key API surface**: `classification.add_classification`, `classification.add_reference`, `classification.edit_classification`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/classification/, module/bsdd/), https://search.bsdd.buildingsmart.org/
- **Research input**: vooronderzoek-bonsai §8 (Classification Systems)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Complexity**: S

#### bonsai-impl-drawing
- **Category**: impl
- **Scope**: 2D drawing generation from 3D BIM, SVG pipeline, drawing types (PLAN_VIEW, SECTION_VIEW, ELEVATION_VIEW), annotations, sheets/documents, SheetBuilder, camera configuration.
- **Key API surface**: `bpy.ops.bim.add_drawing`, `bpy.ops.bim.create_drawing`, `bpy.ops.bim.add_annotation`, `bpy.ops.bim.add_sheet`, `SvgWriter`, `SheetBuilder`, `EPset_Drawing`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/drawing/)
- **Research input**: supplementary-bonsai-gaps §1 (Drawing Module — all 9 subsections)
- **Version coverage**: Bonsai v0.8.x (requires InkScape for SVG post-processing)
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements, bonsai-syntax-geometry, blender-core-gpu
- **Complexity**: L

#### bonsai-impl-qto
- **Category**: impl
- **Scope**: Quantity takeoff from Blender geometry, calculator system (30+ functions), QTO rules (JSON config), standard quantity sets, gross vs net, cost integration.
- **Key API surface**: `bpy.ops.bim.perform_quantity_take_off`, `bpy.ops.bim.calculate_single_quantity`, calculator functions
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/qto/)
- **Research input**: supplementary-bonsai-gaps §2 (QTO Module — all 7 subsections)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-syntax-elements, bonsai-syntax-properties, blender-syntax-mesh
- **Complexity**: M

#### bonsai-impl-bcf
- **Category**: impl
- **Scope**: BIM Collaboration Format (v2.1 + v3.0), BcfStore singleton, topic CRUD, viewpoints, comments, document references.
- **Key API surface**: `bpy.ops.bim.new_bcf_project`, `load_bcf_project`, `save_bcf_project`, `add_bcf_topic`, `add_bcf_viewpoint`, `BcfStore`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/bcf/)
- **Research input**: supplementary-bonsai-gaps §3 (BCF Module — all 8 subsections)
- **Version coverage**: BCF v2.1 + v3.0
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements
- **Complexity**: M

#### bonsai-impl-clash
- **Category**: impl
- **Scope**: Clash detection (intersection/collision/clearance modes), clash sets, element filtering, FCL engine, result visualization, BCF output integration.
- **Key API surface**: `bpy.ops.bim.execute_ifc_clash`, `add_clash_set`, `add_clash_source`, `smart_clash_group`, `ClashDecorator`
- **SOURCES.md URLs**: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai (bim/module/clash/), https://docs.ifcopenshell.org/ifcclash.html
- **Research input**: supplementary-bonsai-gaps §4 (Clash Detection — all 8 subsections)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements
- **Complexity**: M

#### bonsai-errors-common
- **Category**: errors
- **Scope**: All 10 AI common mistakes, 8 common error patterns, module-specific anti-patterns (drawing: 7, QTO: 7, BCF: 7, clash: 7), operator context requirements, `blenderbim→bonsai` migration, operators vs `ifcopenshell.api` decision tree, cache purging, threading limitations.
- **Key API surface**: All error-prone APIs documented across research
- **SOURCES.md URLs**: All Bonsai source URLs
- **Research input**: vooronderzoek-bonsai §11 (Error Patterns), §13 (AI Mistakes), supplementary-bonsai-gaps §1.9, §2.7, §3.8, §4.8, §5.3-5.8
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: All other bonsai skills (references them)
- **Complexity**: L

#### bonsai-agents-ifc-validator
- **Category**: agents
- **Scope**: Validation checklists for IFC models created via Bonsai, schema compliance checks, IfcTester/IDS integration, pre-save validation, ifcopenshell.validate module (validate, assert_valid, json_logger, WHERE rules).
- **Key API surface**: `ifctester`, `ifcopenshell.validate`, `ifcopenshell.validate.json_logger`, `ifcopenshell.validate.assert_valid`, `bpy.ops.bim.validate_*`
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/ifctester.html, https://www.buildingsmart.org/standards/bsi-standards/information-delivery-specification-ids/, https://docs.ifcopenshell.org/autoapi/ifcopenshell/validate/index.html
- **Research input**: vooronderzoek-bonsai §10 (IDS/IfcTester), supplementary-bonsai-gaps §5 (runtime quirks), supplementary-ifcos-gaps §6.1 (L911-1020, ifcopenshell.validate — 6 functions, json_logger, LogDetectionHandler, EXPRESS WHERE rules)
- **Version coverage**: Bonsai v0.8.x
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-properties
- **Complexity**: S

---

### 2.4 Cross-Technology Skills (2)

#### aec-core-bim-workflows
- **Category**: core
- **Scope**: High-level BIM workflow patterns that span technologies — IFC creation pipeline (9-step), property extraction pattern, mesh generation pipeline, LOD/LOI concepts, coordination patterns. Provides the "big picture" for Claude to understand how Blender, Bonsai, and IfcOpenShell work together.
- **Key API surface**: Cross-references to all three technology packages
- **SOURCES.md URLs**: https://docs.ifcopenshell.org/, https://docs.bonsaibim.org/, https://docs.blender.org/api/current/
- **Research input**: vooronderzoek-ecosystem-sources §3 (cross-technology patterns), scope-analysis.md
- **Version coverage**: Technology-agnostic workflow patterns
- **Dependencies**: ifcos-impl-creation, bonsai-core-architecture
- **Complexity**: S

#### aec-agents-workflow-orchestrator
- **Category**: agents
- **Scope**: Auto-detect which skill package to load (Blender vs Bonsai vs IfcOpenShell), route user requests, multi-technology detection, import statement analysis, context-dependent skill selection.
- **Key API surface**: Pattern matching on imports and API calls
- **SOURCES.md URLs**: All technology URLs
- **Research input**: All briefings — orchestrator needs to understand all three technology boundaries
- **Version coverage**: All versions
- **Dependencies**: All skills (routing agent)
- **Complexity**: S

---

## 3. Dependency Graph

```
LEGEND:  ──→  "depends on" (must be built BEFORE)
         ···→  "optional cross-reference"

═══════════════════════════════════════════════════════════════
LAYER 0: FOUNDATION (no dependencies)
═══════════════════════════════════════════════════════════════

  ┌─────────────────┐   ┌─────────────────┐
  │ blender-core-api│   │blender-core-vers│
  │     (C-01)      │   │     (C-02)      │
  └────────┬────────┘   └────────┬────────┘
           │                     │
  ┌─────────────────┐   ┌─────────────────┐
  │ifcos-syntax-fio │   │ifcos-core-concpt│
  └────────┬────────┘   └─────────────────┘
           │

═══════════════════════════════════════════════════════════════
LAYER 1: CORE COMPLETION
═══════════════════════════════════════════════════════════════

  blender-core-api ──→ blender-core-gpu (C-03)
  blender-core-api ──→ blender-core-runtime (C-04)
  ifcos-syntax-fio ──→ ifcos-core-runtime
  ifcos-syntax-fio ──→ ifcos-syntax-api
  ifcos-syntax-fio ──→ ifcos-syntax-elements
  ifcos-syntax-fio ──→ ifcos-syntax-util

═══════════════════════════════════════════════════════════════
LAYER 2: SYNTAX
═══════════════════════════════════════════════════════════════

  blender-core-api ──→ blender-syntax-operators (S-01)
  blender-core-api ──→ blender-syntax-mesh (S-05)
  blender-core-api ──→ blender-syntax-nodes (S-07)
  blender-core-api ──→ blender-syntax-data (S-11)
  blender-core-api ──→ blender-syntax-rendering (S-10)
  blender-core-api+S-02 ──→ blender-syntax-animation (S-08)

  S-01 operators ──→ blender-syntax-properties (S-02)
  S-02 properties ──→ blender-syntax-panels (S-03)
  S-01+S-03 ──→ blender-syntax-addons (S-04)
  S-07 nodes ──→ blender-syntax-materials (S-09)
  S-07+core-api ──→ blender-syntax-modifiers (S-06)

═══════════════════════════════════════════════════════════════
LAYER 3: IMPLEMENTATION
═══════════════════════════════════════════════════════════════

  BLENDER:
  S-01 ──→ blender-impl-operators (I-01)
  S-04 ──→ blender-impl-addons (I-02)
  S-05 ──→ blender-impl-mesh (I-03)
  S-10+C-04 ──→ blender-impl-automation (I-04)
  S-07+S-09+S-06 ──→ blender-impl-nodes (I-05)
  S-08 ──→ blender-impl-animation (I-06)

  IFCOPENSHELL:
  ifcos-syntax-api ──→ ifcos-impl-creation
  ifcos-syntax-api+impl-creation ──→ ifcos-impl-geometry
  ifcos-syntax-api ──→ ifcos-impl-materials
  ifcos-syntax-api+elements ──→ ifcos-impl-relationships
  ifcos-syntax-api ──→ ifcos-impl-cost
  ifcos-syntax-api ──→ ifcos-impl-sequence
  ifcos-syntax-api+impl-rels ──→ ifcos-impl-mep
  ifcos-syntax-api+impl-geom ──→ ifcos-impl-profiles
  ifcos-syntax-api+fio ──→ ifcos-impl-validation

═══════════════════════════════════════════════════════════════
LAYER 4: ERRORS
═══════════════════════════════════════════════════════════════

  S-01+C-01 ──→ blender-errors-context (E-01)
  C-01+C-04 ──→ blender-errors-data (E-02)
  C-02 ──→ blender-errors-version (E-03)

  ifcos-syntax-fio ──→ ifcos-errors-schema
  ifcos-syntax-api+elements ──→ ifcos-errors-patterns
  ifcos-syntax-fio+impl-geom ──→ ifcos-errors-performance

═══════════════════════════════════════════════════════════════
LAYER 5: BONSAI (depends on Blender + IfcOpenShell)
═══════════════════════════════════════════════════════════════

  blender-core-api+S-01+S-04 ──→ bonsai-core-architecture
  bonsai-core-arch+ifcos-syntax-api+ifcos-syntax-elements ──→ bonsai-syntax-elements
  bonsai-syntax-elements ──→ bonsai-syntax-spatial
  bonsai-syntax-elements ──→ bonsai-syntax-properties
  bonsai-syntax-elements ──→ bonsai-syntax-geometry

  bonsai-core-arch+bonsai-syntax-elements+ifcos-syntax-fio ──→ bonsai-impl-project
  bonsai-syntax-*+ifcos-syntax-api ──→ bonsai-impl-modeling
  bonsai-syntax-elements+ifcos-syntax-api ──→ bonsai-impl-classification
  bonsai-core-arch+syntax-elements+geometry+blender-core-gpu ──→ bonsai-impl-drawing
  bonsai-syntax-elements+properties+blender-syntax-mesh ──→ bonsai-impl-qto
  bonsai-core-arch ──→ bonsai-impl-bcf
  bonsai-core-arch ──→ bonsai-impl-clash

  ALL bonsai ──→ bonsai-errors-common

  NOTE: This graph shows primary dependencies. Consult individual
  skill entries in Section 2 for complete dependency lists.

═══════════════════════════════════════════════════════════════
LAYER 6: AGENTS + CROSS-TECH (depend on everything)
═══════════════════════════════════════════════════════════════

  ALL blender syntax+errors ──→ blender-agents-code-validator
  C-02+E-03 ──→ blender-agents-version-migrator
  ALL ifcos ──→ ifcos-agents-code-validator
  bonsai-core-arch+properties ──→ bonsai-agents-ifc-validator
  ifcos-impl-creation+bonsai-core-arch ──→ aec-core-bim-workflows
  ALL ──→ aec-agents-workflow-orchestrator
```

---

## 4. Build Order & Batch Plan

### Constraints
- 3–5 agents per batch
- NEVER two agents writing to same directory
- Quality gate after every batch
- D-008/D-011: Blender + IfcOpenShell built in parallel; Bonsai waits for both (D-011 supersedes D-008's literal ordering)
- Each skill = unique directory → no file conflicts

### Batch Plan

#### Batch 1: Foundation (4 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-core-api` | skills/blender/core/blender-core-api/ |
| A2 | `blender-core-versions` | skills/blender/core/blender-core-versions/ |
| A3 | `ifcos-core-concepts` | skills/ifcopenshell/core/ifcos-core-concepts/ |
| A4 | `ifcos-syntax-fileio` | skills/ifcopenshell/syntax/ifcos-syntax-fileio/ |

**Quality gate**: All 4 skills have valid frontmatter, <500 lines, English-only, deterministic language.

#### Batch 2: Core Completion + Early Syntax (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-core-gpu` | skills/blender/core/blender-core-gpu/ |
| A2 | `blender-core-runtime` | skills/blender/core/blender-core-runtime/ |
| A3 | `blender-syntax-operators` | skills/blender/syntax/blender-syntax-operators/ |
| A4 | `ifcos-syntax-api` | skills/ifcopenshell/syntax/ifcos-syntax-api/ |
| A5 | `ifcos-syntax-elements` | skills/ifcopenshell/syntax/ifcos-syntax-elements/ |

**Quality gate**: Verify cross-references to Batch 1 skills are correct.

#### Batch 3: Syntax A (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-syntax-properties` | skills/blender/syntax/blender-syntax-properties/ |
| A2 | `blender-syntax-mesh` | skills/blender/syntax/blender-syntax-mesh/ |
| A3 | `blender-syntax-nodes` | skills/blender/syntax/blender-syntax-nodes/ |
| A4 | `ifcos-syntax-util` | skills/ifcopenshell/syntax/ifcos-syntax-util/ |
| A5 | `ifcos-core-runtime` | skills/ifcopenshell/core/ifcos-core-runtime/ |

#### Batch 4: Syntax B (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-syntax-panels` | skills/blender/syntax/blender-syntax-panels/ |
| A2 | `blender-syntax-animation` | skills/blender/syntax/blender-syntax-animation/ |
| A3 | `blender-syntax-materials` | skills/blender/syntax/blender-syntax-materials/ |
| A4 | `blender-syntax-data` | skills/blender/syntax/blender-syntax-data/ |
| A5 | `blender-syntax-rendering` | skills/blender/syntax/blender-syntax-rendering/ |

**Note**: `blender-syntax-addons` moved to Batch 5 because it depends on `blender-syntax-panels` (this batch).

#### Batch 5: Syntax C + IfcOS Impl Start (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-syntax-modifiers` | skills/blender/syntax/blender-syntax-modifiers/ |
| A2 | `blender-syntax-addons` | skills/blender/syntax/blender-syntax-addons/ |
| A3 | `ifcos-impl-creation` | skills/ifcopenshell/impl/ifcos-impl-creation/ |
| A4 | `ifcos-impl-materials` | skills/ifcopenshell/impl/ifcos-impl-materials/ |
| A5 | `ifcos-impl-relationships` | skills/ifcopenshell/impl/ifcos-impl-relationships/ |

#### Batch 6: Blender Impl A + IfcOS Impl B (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-impl-operators` | skills/blender/impl/blender-impl-operators/ |
| A2 | `blender-impl-addons` | skills/blender/impl/blender-impl-addons/ |
| A3 | `blender-impl-mesh` | skills/blender/impl/blender-impl-mesh/ |
| A4 | `ifcos-impl-geometry` | skills/ifcopenshell/impl/ifcos-impl-geometry/ |
| A5 | `ifcos-impl-cost` | skills/ifcopenshell/impl/ifcos-impl-cost/ |

#### Batch 7: Blender Impl B + IfcOS Impl C (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-impl-nodes` | skills/blender/impl/blender-impl-nodes/ |
| A2 | `blender-impl-animation` | skills/blender/impl/blender-impl-animation/ |
| A3 | `blender-impl-automation` | skills/blender/impl/blender-impl-automation/ |
| A4 | `ifcos-impl-sequence` | skills/ifcopenshell/impl/ifcos-impl-sequence/ |
| A5 | `ifcos-impl-profiles` | skills/ifcopenshell/impl/ifcos-impl-profiles/ |

#### Batch 8: IfcOS Impl D + Errors + Bonsai Core (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `ifcos-impl-mep` | skills/ifcopenshell/impl/ifcos-impl-mep/ |
| A2 | `ifcos-errors-schema` | skills/ifcopenshell/errors/ifcos-errors-schema/ |
| A3 | `ifcos-errors-patterns` | skills/ifcopenshell/errors/ifcos-errors-patterns/ |
| A4 | `ifcos-errors-performance` | skills/ifcopenshell/errors/ifcos-errors-performance/ |
| A5 | `bonsai-core-architecture` | skills/bonsai/core/bonsai-core-architecture/ |

**Note**: `bonsai-core-architecture` moved here from Batch 9 so that `aec-core-bim-workflows` and `bonsai-syntax-elements` (Batch 9) can depend on it.

#### Batch 9: Blender Errors + Bonsai Syntax Start (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-errors-context` | skills/blender/errors/blender-errors-context/ |
| A2 | `blender-errors-data` | skills/blender/errors/blender-errors-data/ |
| A3 | `blender-errors-version` | skills/blender/errors/blender-errors-version/ |
| A4 | `bonsai-syntax-elements` | skills/bonsai/syntax/bonsai-syntax-elements/ |
| A5 | `aec-core-bim-workflows` | skills/aec-cross-tech/core/aec-core-bim-workflows/ |

**Note**: `bonsai-syntax-elements` moved here from Batch 10 so that the 3 dependent Bonsai syntax skills (Batch 10) can reference it.

#### Batch 10: Bonsai Syntax + IfcOS Validation (4 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `bonsai-syntax-spatial` | skills/bonsai/syntax/bonsai-syntax-spatial/ |
| A2 | `bonsai-syntax-properties` | skills/bonsai/syntax/bonsai-syntax-properties/ |
| A3 | `bonsai-syntax-geometry` | skills/bonsai/syntax/bonsai-syntax-geometry/ |
| A4 | `ifcos-impl-validation` | skills/ifcopenshell/impl/ifcos-impl-validation/ |

#### Batch 11: Bonsai Impl A (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `bonsai-impl-project` | skills/bonsai/impl/bonsai-impl-project/ |
| A2 | `bonsai-impl-modeling` | skills/bonsai/impl/bonsai-impl-modeling/ |
| A3 | `bonsai-impl-classification` | skills/bonsai/impl/bonsai-impl-classification/ |
| A4 | `bonsai-impl-drawing` | skills/bonsai/impl/bonsai-impl-drawing/ |
| A5 | `bonsai-impl-qto` | skills/bonsai/impl/bonsai-impl-qto/ |

#### Batch 12: Bonsai Impl B (3 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `bonsai-impl-bcf` | skills/bonsai/impl/bonsai-impl-bcf/ |
| A2 | `bonsai-impl-clash` | skills/bonsai/impl/bonsai-impl-clash/ |
| A3 | `ifcos-agents-code-validator` | skills/ifcopenshell/agents/ifcos-agents-code-validator/ |

**Note**: `bonsai-errors-common` moved to Batch 13 because it depends on ALL bonsai skills including `bonsai-impl-bcf` and `bonsai-impl-clash` (this batch).

#### Batch 13: Agent Skills + Bonsai Errors (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-agents-code-validator` | skills/blender/agents/blender-agents-code-validator/ |
| A2 | `blender-agents-version-migrator` | skills/blender/agents/blender-agents-version-migrator/ |
| A3 | `bonsai-agents-ifc-validator` | skills/bonsai/agents/bonsai-agents-ifc-validator/ |
| A4 | `aec-agents-workflow-orchestrator` | skills/aec-cross-tech/agents/aec-agents-workflow-orchestrator/ |
| A5 | `bonsai-errors-common` | skills/bonsai/errors/bonsai-errors-common/ |

### Batch Summary

| Batch | Skills | Agents | Focus |
|-------|--------|--------|-------|
| 1 | 4 | 4 | Foundation core (Blender + IfcOpenShell) |
| 2 | 5 | 5 | Core completion + early syntax |
| 3 | 5 | 5 | Syntax A (Blender + IfcOpenShell) |
| 4 | 5 | 5 | Syntax B (Blender) |
| 5 | 5 | 5 | Syntax C + IfcOpenShell impl start |
| 6 | 5 | 5 | Blender impl A + IfcOpenShell impl B |
| 7 | 5 | 5 | Blender impl B + IfcOpenShell impl C |
| 8 | 5 | 5 | IfcOS impl D + errors + Bonsai core |
| 9 | 5 | 5 | Blender errors + Bonsai syntax start |
| 10 | 4 | 4 | Bonsai syntax + IfcOS validation |
| 11 | 5 | 5 | Bonsai impl A |
| 12 | 3 | 3 | Bonsai impl B |
| 13 | 5 | 5 | Agent skills + Bonsai errors |
| **Total** | **61** | **61** | |

---

## 5. Per-Skill Prompts

### Prompt Template

Every skill-writer agent receives this template with skill-specific parameters filled in:

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Create skill: {SKILL_NAME}

BEFORE STARTING, READ:
1. REQUIREMENTS.md — quality criteria
2. WAY_OF_WORK.md — skill structure specification
3. DECISIONS.md — D-003 (English), D-009 (500 lines)
4. {DEPENDENCY_SKILLS} — skills this depends on (read SKILL.md of each)
5. {RESEARCH_DOCS} — research documents for this skill

OUTPUT DIRECTORY: {OUTPUT_DIR}

CREATE THESE FILES:

1. SKILL.md (< 500 lines) with:
   ---
   name: {SKILL_NAME}
   description: "{DESCRIPTION}"
   license: MIT
   compatibility: Designed for Claude Code. Requires Python 3.x.
   metadata:
     author: OpenAEC-Foundation
     version: "1.0"
   ---
   - Quick Reference (critical warnings, decision trees)
   - Essential Patterns (with version annotations)
   - Common Operations (verified code snippets)
   - Reference Links (to references/ files)

2. references/methods.md — Complete API signatures
3. references/examples.md — Working code examples (VERIFIED against sources)
4. references/anti-patterns.md — What NOT to do (from real issues)

APPROVED SOURCES FOR THIS SKILL:
{SOURCES_URLS}

RESEARCH INPUT:
{RESEARCH_SECTIONS}

VERSION ANNOTATIONS REQUIRED: {VERSION_COVERAGE}

STYLE RULES:
- English only (D-003)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- Version-explicit on ALL code blocks
- Verified against official documentation only
- NEVER use: "you might", "consider", "perhaps", "often", "usually"
- Description: lead with third-person verb, include domain trigger keywords, max 1024 chars

VERIFICATION CHECKLIST:
- [ ] SKILL.md has valid YAML frontmatter (name + description with trigger words)
- [ ] SKILL.md < 500 lines
- [ ] English only, zero Dutch
- [ ] Deterministic language throughout
- [ ] Version annotations on all code blocks
- [ ] All references/ files exist and are linked from SKILL.md
- [ ] Code examples verified against approved source URLs
- [ ] Anti-patterns file has at least 3 entries with WHY explanations
```

### Skill-Specific Parameters

Below are the parameters for EACH skill. Plug these into the template above.

#### Blender Skills

| Skill | Output Dir | Research Docs | Source URLs | Version Coverage | Dependencies |
|-------|-----------|---------------|-------------|-----------------|-------------|
| blender-core-api | skills/blender/core/blender-core-api/ | vooronderzoek-blender §1 (L29-78), §9, §10 | Blender Python API (info_quickstart, info_overview) | All 3.x-5.x | None |
| blender-core-versions | skills/blender/core/blender-core-versions/ | vooronderzoek-blender §2 (L81-446) | All version release notes (4.0-5.1), Compatibility Index, Python API Changelog | 3.x-5.1 | None |
| blender-core-gpu | skills/blender/core/blender-core-gpu/ | vooronderzoek-blender §2 BGL (L311-435), supplementary-blender-gaps §7.8 (L1738-1829) | Blender API (gpu, gpu.shader, gpu.types), 5.0 release notes | bgl deprecated 3.5, removed 5.0 | blender-core-api |
| blender-core-runtime | skills/blender/core/blender-core-runtime/ | supplementary-blender-gaps §7 (L1468-1851), §8 (L1854-2193) | Blender API (mathutils, timers, msgbus, handlers, info_gotchas) | 5.0 IDProperty changes | blender-core-api |
| blender-syntax-operators | skills/blender/syntax/blender-syntax-operators/ | vooronderzoek-blender §4 (L609-751) | Blender Python API, 4.0 release notes | 4.0 temp_override, 4.2 MODAL_PRIORITY | blender-core-api |
| blender-syntax-properties | skills/blender/syntax/blender-syntax-properties/ | vooronderzoek-blender §5 (L754-915) | Blender Python API (bpy.props) | 4.1 enum ID props, 5.0 property_unset | blender-syntax-operators |
| blender-syntax-panels | skills/blender/syntax/blender-syntax-panels/ | vooronderzoek-blender §6 (L918-1000+) | Blender Python API | 4.1 layout.panel() | blender-syntax-properties |
| blender-syntax-addons | skills/blender/syntax/blender-syntax-addons/ | vooronderzoek-blender §3 (L450-606) | Extensions Dev Handbook, Add-on Guidelines, Extension Hosting | 4.2 manifest.toml | blender-syntax-operators, blender-syntax-panels |
| blender-syntax-mesh | skills/blender/syntax/blender-syntax-mesh/ | vooronderzoek-blender §7 (L1000+) | Blender Python API | 4.0 attributes, 4.1 auto_smooth, 4.3 AttributeGroup | blender-core-api |
| blender-syntax-modifiers | skills/blender/syntax/blender-syntax-modifiers/ | vooronderzoek-blender §8, supplementary §1.5 (L165-186) | Blender Python API | 4.0 temp_override, GN Socket_N | blender-core-api, blender-syntax-nodes |
| blender-syntax-nodes | skills/blender/syntax/blender-syntax-nodes/ | supplementary-blender-gaps §1 (L24-252), vooronderzoek §2 (L134-143) | Blender API (NodeTree, NodeTreeInterface), 4.0/5.0 notes | 4.0 interface API, 5.0 compositor | blender-core-api |
| blender-syntax-animation | skills/blender/syntax/blender-syntax-animation/ | supplementary-blender-gaps §2 (L255-583) | Blender API (FCurve, BoneCollection), 4.0 bone migration | 4.0 bone collections | blender-core-api, blender-syntax-properties |
| blender-syntax-materials | skills/blender/syntax/blender-syntax-materials/ | supplementary-blender-gaps §3 (L586-795) | Blender API (Material, ShaderNodeBsdfPrincipled), 4.0 notes | 4.0 BSDF renames, 4.2 blend_method | blender-syntax-nodes |
| blender-syntax-rendering | skills/blender/syntax/blender-syntax-rendering/ | supplementary-blender-gaps §4 (L799-1062) | Blender API (RenderSettings, SceneEEVEE), 5.0 notes | EEVEE identifier changes across versions | blender-core-api |
| blender-syntax-data | skills/blender/syntax/blender-syntax-data/ | supplementary-blender-gaps §6 (L1245-1465), vooronderzoek §1 (L47-66) | Blender API (Collection, BlendDataLibraries, AssetMetaData), 5.0 notes | 4.0 library overrides, 5.0 asset context | blender-core-api |
| blender-impl-operators | skills/blender/impl/blender-impl-operators/ | vooronderzoek-blender §4 (L680-715), §9 | Blender Python API | Same as syntax-operators | blender-syntax-operators |
| blender-impl-addons | skills/blender/impl/blender-impl-addons/ | vooronderzoek-blender §3 (L450-606) | Extensions Dev Handbook, Extension Hosting | 4.2+ extension focus | blender-syntax-addons |
| blender-impl-mesh | skills/blender/impl/blender-impl-mesh/ | vooronderzoek-blender §7 | Blender Python API | Same as syntax-mesh | blender-syntax-mesh |
| blender-impl-automation | skills/blender/impl/blender-impl-automation/ | supplementary §5 (L1065-1242), §8.7-8.8 (L2116-2170) | Blender API (export_scene, wm) | 4.0 OBJ/STL operator migration | blender-syntax-rendering, blender-core-runtime |
| blender-impl-nodes | skills/blender/impl/blender-impl-nodes/ | supplementary §1 (L24-252), §3.4 (L706-725) | Blender Python API | Same as syntax-nodes | blender-syntax-nodes, blender-syntax-materials, blender-syntax-modifiers |
| blender-impl-animation | skills/blender/impl/blender-impl-animation/ | supplementary §2 (L255-583) | Blender Python API | Same as syntax-animation | blender-syntax-animation |
| blender-errors-context | skills/blender/errors/blender-errors-context/ | vooronderzoek §9, §11, §12 | Blender Python API | 4.0 context override removal | blender-syntax-operators, blender-core-api |
| blender-errors-data | skills/blender/errors/blender-errors-data/ | vooronderzoek §11, supplementary §8.2 (L1903-1943) | Blender API (info_gotchas) | All versions | blender-core-api, blender-core-runtime |
| blender-errors-version | skills/blender/errors/blender-errors-version/ | vooronderzoek §2 (L81-446), supplementary §4.9 (L1045-1056) | All release notes, Compatibility Index | 3.x→5.1 | blender-core-versions |
| blender-agents-code-validator | skills/blender/agents/blender-agents-code-validator/ | vooronderzoek §11-12 | All Blender sources | All versions | All Blender syntax + error skills |
| blender-agents-version-migrator | skills/blender/agents/blender-agents-version-migrator/ | vooronderzoek §2 (L81-446) | All release notes | 3.x→5.1 | blender-core-versions, blender-errors-version |

#### IfcOpenShell Skills

| Skill | Output Dir | Research Docs | Source URLs | Version Coverage | Dependencies |
|-------|-----------|---------------|-------------|-----------------|-------------|
| ifcos-core-concepts | skills/ifcopenshell/core/ifcos-core-concepts/ | vooronderzoek-ifcos §8, fragments/schema-versions §2, topic/schema §2-4 | IFC4.3 docs, buildingSMART technical, IFC Schema Specs | IFC2X3/IFC4/IFC4X3 | None |
| ifcos-core-runtime | skills/ifcopenshell/core/ifcos-core-runtime/ | supplementary-ifcos-gaps §7, topic/errors §3-4 | docs.ifcopenshell.org entity_instance, PyPI | Schema-agnostic | ifcos-syntax-fileio |
| ifcos-syntax-fileio | skills/ifcopenshell/syntax/ifcos-syntax-fileio/ | vooronderzoek-ifcos §2, fragments/core-ops §1-3, topic/core-ops §1 | docs.ifcopenshell.org file/, hello_world.html | All schemas | None |
| ifcos-syntax-api | skills/ifcopenshell/syntax/ifcos-syntax-api/ | vooronderzoek-ifcos §3, fragments/api-categories, topic/errors §5 | docs.ifcopenshell.org api/index.html | All schemas | ifcos-syntax-fileio |
| ifcos-syntax-elements | skills/ifcopenshell/syntax/ifcos-syntax-elements/ | vooronderzoek-ifcos §4, §8, topic/core-ops §2 | docs.ifcopenshell.org entity_instance/ | All schemas | ifcos-syntax-fileio |
| ifcos-syntax-util | skills/ifcopenshell/syntax/ifcos-syntax-util/ | vooronderzoek-ifcos §5, topic/core-ops §3 | docs.ifcopenshell.org util/index.html | Medium schema sensitivity | ifcos-syntax-fileio |
| ifcos-impl-creation | skills/ifcopenshell/impl/ifcos-impl-creation/ | vooronderzoek-ifcos §10, fragments/api-categories §Example | docs.ifcopenshell.org code_examples.html, academy wall tutorial | HIGH: OwnerHistory in IFC2X3 | ifcos-syntax-fileio, ifcos-syntax-api |
| ifcos-impl-geometry | skills/ifcopenshell/impl/ifcos-impl-geometry/ | vooronderzoek-ifcos §6, fragments/api-categories §3, topic/core-ops §4 | docs.ifcopenshell.org geometry_settings, geometry_processing, geometry_creation | Low | ifcos-syntax-api, ifcos-impl-creation |
| ifcos-impl-materials | skills/ifcopenshell/impl/ifcos-impl-materials/ | vooronderzoek-ifcos §3 material, fragments/api-categories §6 | docs.ifcopenshell.org api/material/ | IFC4+ constituents | ifcos-syntax-api |
| ifcos-impl-relationships | skills/ifcopenshell/impl/ifcos-impl-relationships/ | vooronderzoek-ifcos §9, fragments/schema-versions §4, topic/schema §6 | docs.ifcopenshell.org api/spatial, aggregate, type | HIGH: relationship splits | ifcos-syntax-api, ifcos-syntax-elements |
| ifcos-impl-cost | skills/ifcopenshell/impl/ifcos-impl-cost/ | supplementary-ifcos-gaps §1 | docs.ifcopenshell.org api/cost/ | Low | ifcos-syntax-api |
| ifcos-impl-sequence | skills/ifcopenshell/impl/ifcos-impl-sequence/ | supplementary-ifcos-gaps §2 | docs.ifcopenshell.org api/sequence/ | IFC2X3 date entities vs IFC4+ ISO 8601 | ifcos-syntax-api |
| ifcos-impl-mep | skills/ifcopenshell/impl/ifcos-impl-mep/ | supplementary-ifcos-gaps §3 | docs.ifcopenshell.org api/system/ | IFC2X3 IfcSystem vs IFC4+ IfcDistributionSystem | ifcos-syntax-api, ifcos-impl-relationships |
| ifcos-impl-profiles | skills/ifcopenshell/impl/ifcos-impl-profiles/ | supplementary-ifcos-gaps §5 | docs.ifcopenshell.org api/profile/ | Low | ifcos-syntax-api, ifcos-impl-geometry |
| ifcos-impl-validation | skills/ifcopenshell/impl/ifcos-impl-validation/ | supplementary-ifcos-gaps §6 (L911-1112) | docs.ifcopenshell.org validate/, api/georeference/ | Medium: IFC2X3 vs IFC4+ georeference | ifcos-syntax-api, ifcos-syntax-fileio |
| ifcos-errors-schema | skills/ifcopenshell/errors/ifcos-errors-schema/ | vooronderzoek-ifcos §7, §11, fragments/schema-versions, topic/schema | buildingSMART technical, IFC4.3 docs, ifcpatch docs | All schemas | ifcos-syntax-fileio |
| ifcos-errors-patterns | skills/ifcopenshell/errors/ifcos-errors-patterns/ | vooronderzoek-ifcos §11, fragments/errors-perf §1, topic/errors §1 | docs.ifcopenshell.org ifcopenshell-python.html | Medium | ifcos-syntax-api, ifcos-syntax-elements |
| ifcos-errors-performance | skills/ifcopenshell/errors/ifcos-errors-performance/ | vooronderzoek-ifcos §12, fragments/errors-perf §2, topic/errors §2 | docs.ifcopenshell.org geometry_processing.html | Low | ifcos-syntax-fileio, ifcos-impl-geometry |
| ifcos-agents-code-validator | skills/ifcopenshell/agents/ifcos-agents-code-validator/ | vooronderzoek-ifcos §13, fragments/errors-perf §3, topic/errors §3,5,6 | All IfcOpenShell URLs | All schemas | All ifcos skills |

#### Bonsai Skills

| Skill | Output Dir | Research Docs | Source URLs | Version Coverage | Dependencies |
|-------|-----------|---------------|-------------|-----------------|-------------|
| bonsai-core-architecture | skills/bonsai/core/bonsai-core-architecture/ | vooronderzoek-bonsai §2, supplementary-bonsai §5.1-5.2 | Bonsai source (core/, tool/, bim/ifc.py) | Bonsai v0.8.x | blender-core-api, blender-syntax-operators, blender-syntax-addons |
| bonsai-syntax-elements | skills/bonsai/syntax/bonsai-syntax-elements/ | vooronderzoek-bonsai §3, §7, §12 | Bonsai source (bim/ifc.py, tool/ifc.py) | Bonsai v0.8.x | bonsai-core-architecture, ifcos-syntax-api |
| bonsai-syntax-spatial | skills/bonsai/syntax/bonsai-syntax-spatial/ | vooronderzoek-bonsai §4 | Bonsai source (tool/spatial.py, core/spatial.py, bim/module/spatial/) | Bonsai v0.8.x | bonsai-syntax-elements, ifcos-syntax-api |
| bonsai-syntax-properties | skills/bonsai/syntax/bonsai-syntax-properties/ | vooronderzoek-bonsai §5 | Bonsai source (bim/module/pset/) | Bonsai v0.8.x | bonsai-syntax-elements, ifcos-syntax-api |
| bonsai-syntax-geometry | skills/bonsai/syntax/bonsai-syntax-geometry/ | vooronderzoek-bonsai §9 | Bonsai source (bim/module/geometry/) | Bonsai v0.8.x | bonsai-syntax-elements, ifcos-syntax-api |
| bonsai-impl-project | skills/bonsai/impl/bonsai-impl-project/ | vooronderzoek-bonsai §10, supplementary-ifcos-gaps §6.2 (georeference) | docs.bonsaibim.org, ifctester docs, Bonsai source (bim/module/project/), docs.ifcopenshell.org api/georeference/ | Bonsai v0.8.x, IFC2X3/4/4X3 | bonsai-core-architecture, bonsai-syntax-elements |
| bonsai-impl-modeling | skills/bonsai/impl/bonsai-impl-modeling/ | vooronderzoek-bonsai §6, §7 | Bonsai source (bim/module/model/, type/, material/) | Bonsai v0.8.x | bonsai-syntax-elements, bonsai-syntax-spatial, bonsai-syntax-geometry |
| bonsai-impl-classification | skills/bonsai/impl/bonsai-impl-classification/ | vooronderzoek-bonsai §8 | Bonsai source (bim/module/classification/, bsdd/), bSDD portal | Bonsai v0.8.x | bonsai-syntax-elements |
| bonsai-impl-drawing | skills/bonsai/impl/bonsai-impl-drawing/ | supplementary-bonsai-gaps §1 (all) | Bonsai source (bim/module/drawing/ — 16 files) | Bonsai v0.8.x | bonsai-core-architecture, bonsai-syntax-elements, bonsai-syntax-geometry |
| bonsai-impl-qto | skills/bonsai/impl/bonsai-impl-qto/ | supplementary-bonsai-gaps §2 (all) | Bonsai source (bim/module/qto/ — 7 files) | Bonsai v0.8.x | bonsai-syntax-elements, bonsai-syntax-properties |
| bonsai-impl-bcf | skills/bonsai/impl/bonsai-impl-bcf/ | supplementary-bonsai-gaps §3 (all) | Bonsai source (bim/module/bcf/ — 5 files) | BCF v2.1 + v3.0 | bonsai-core-architecture, bonsai-syntax-elements |
| bonsai-impl-clash | skills/bonsai/impl/bonsai-impl-clash/ | supplementary-bonsai-gaps §4 (all) | Bonsai source (bim/module/clash/ — 6 files), ifcclash docs | Bonsai v0.8.x | bonsai-core-architecture, bonsai-syntax-elements |
| bonsai-errors-common | skills/bonsai/errors/bonsai-errors-common/ | vooronderzoek-bonsai §11, §13, supplementary §1.9, §2.7, §3.8, §4.8, §5.3-5.8 | All Bonsai source URLs | Bonsai v0.8.x | All bonsai skills |
| bonsai-agents-ifc-validator | skills/bonsai/agents/bonsai-agents-ifc-validator/ | vooronderzoek-bonsai §10, supplementary-bonsai §5, supplementary-ifcos-gaps §6.1 (validate) | ifctester docs, IDS standard, docs.ifcopenshell.org validate/ | Bonsai v0.8.x | bonsai-core-architecture, bonsai-syntax-properties |

#### Cross-Tech Skills

| Skill | Output Dir | Research Docs | Source URLs | Version Coverage | Dependencies |
|-------|-----------|---------------|-------------|-----------------|-------------|
| aec-core-bim-workflows | skills/aec-cross-tech/core/aec-core-bim-workflows/ | vooronderzoek-ecosystem-sources §3, scope-analysis.md | All technology docs | All | ifcos-impl-creation, bonsai-core-architecture |
| aec-agents-workflow-orchestrator | skills/aec-cross-tech/agents/aec-agents-workflow-orchestrator/ | All briefings | All technology docs | All | All skills |

---

## 6. Quality Gate Template

Run this checklist after EVERY batch. Based on PROMPT D in session-prompts.md.

### Structural Checks

- [ ] SKILL.md exists in each skill directory
- [ ] YAML frontmatter has `name` and `description` fields
- [ ] `description` contains trigger words (when Claude should load this skill)
- [ ] `description` starts with third-person verb, max 1024 chars
- [ ] SKILL.md < 500 lines (`wc -l`)
- [ ] `references/` directory exists with at least `methods.md`, `examples.md`, `anti-patterns.md`
- [ ] All files referenced in SKILL.md exist in `references/`

### Content Checks

- [ ] English only (grep for Dutch: "gebruik", "voor", "niet", "moet", "deze", "wordt")
- [ ] Deterministic language (grep for banned: "you might", "consider", "perhaps", "often", "usually")
- [ ] ALWAYS/NEVER used for critical patterns
- [ ] Version annotations on code blocks (Blender 3.x/4.x/5.x or IFC2X3/IFC4/IFC4.3)

### Source Verification

- [ ] Code examples traceable to SOURCES.md approved URLs
- [ ] No patterns from unverified blog posts
- [ ] API signatures match official documentation

### Cross-Reference Checks

- [ ] Skill name in frontmatter matches directory name
- [ ] References to other skills use correct names
- [ ] No broken links to reference files
- [ ] Dependency skills referenced correctly

### Quality Checks

- [ ] Code examples are syntactically correct Python
- [ ] Anti-patterns file has at least 3 entries with WHY explanations
- [ ] Decision trees use clear if/then logic
- [ ] Quick Reference section exists at top of SKILL.md
- [ ] L-complexity skills use `references/` for detailed API tables (keeping SKILL.md under 500 lines)

### Severity Levels

- **BLOCKER**: Must fix before batch approval (e.g., >500 lines, missing frontmatter, non-English content)
- **WARNING**: Should fix before next batch (e.g., missing version annotation, weak deterministic language)
- **INFO**: Nice to have (e.g., additional examples, cross-reference suggestions)

### Validation Report Output

Write to: `docs/validation/validation-batch-{N}.md`

---

## Appendix A: Briefing Divergences

Where this masterplan disagrees with individual briefings:

| Briefing | Recommendation | Masterplan Decision | Reason |
|----------|---------------|---------------------|--------|
| cross-tech | 4 cross-tech skills | 2 cross-tech skills | `aec-core-ifc-fundamentals` redundant with `ifcos-core-concepts`; `aec-core-python-runtime` redundant with per-tech runtime skills |
| cross-tech | `ifcos-core-schemas` as separate skill | Merged into `ifcos-core-concepts` + `ifcos-errors-schema` | Schema overview splits into concepts (entity hierarchy) and errors (version pitfalls) |
| cross-tech | 54 total skills | 61 total skills | Individual briefings added 6 IfcOpenShell skills (incl. ifcos-impl-validation), 4 Bonsai skills, 1 Blender skill after gap analysis |
| blender | I-01 depends on E-01 | I-01 depends only on S-01 | Implementation doesn't depend on errors; errors reference implementation |
| cross-tech | Bonsai `impl-export` | Merged into `bonsai-impl-project` | Bonsai is native IFC; no separate export concept |
| cross-tech | `bonsai-errors-ifc` + `bonsai-errors-geometry` | Merged into `bonsai-errors-common` | One consolidated error skill is more efficient |

## Appendix B: Full Source URL Reference

### Blender URLs (from SOURCES.md)
- Blender Python API: https://docs.blender.org/api/current/
- Extensions Developer Handbook: https://developer.blender.org/docs/handbook/extensions/
- Add-on Dev Setup: https://developer.blender.org/docs/handbook/extensions/addon_dev_setup/
- Add-on Guidelines: https://developer.blender.org/docs/handbook/extensions/addon_guidelines/
- Extension Hosting: https://developer.blender.org/docs/handbook/extensions/hosted/
- Blender Source (GitHub): https://github.com/blender/blender
- 4.0 Python API: https://developer.blender.org/docs/release_notes/4.0/python_api/
- 4.1 Python API: https://developer.blender.org/docs/release_notes/4.1/python_api/
- 4.2 Python API: https://developer.blender.org/docs/release_notes/4.2/python_api/
- 4.3 Python API: https://developer.blender.org/docs/release_notes/4.3/python_api/
- 4.4 Python API: https://developer.blender.org/docs/release_notes/4.4/python_api/
- 4.5 Python API: https://developer.blender.org/docs/release_notes/4.5/python_api/
- 5.0 Python API: https://developer.blender.org/docs/release_notes/5.0/python_api/
- 5.1 Python API: https://developer.blender.org/docs/release_notes/5.1/python_api/
- Compatibility Index: https://developer.blender.org/docs/release_notes/compatibility/
- Python API Changelog: https://docs.blender.org/api/current/change_log.html

### IfcOpenShell URLs (from SOURCES.md)
- IfcOpenShell Docs: https://docs.ifcopenshell.org/
- Python API: https://docs.ifcopenshell.org/ifcopenshell-python.html
- Academy: https://academy.ifcopenshell.org/
- PyPI: https://pypi.org/project/ifcopenshell/
- GitHub: https://github.com/IfcOpenShell/IfcOpenShell
- API Index: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- Util Index: https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html
- Entity Instance: https://docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html
- File: https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html
- Hello World: https://docs.ifcopenshell.org/ifcopenshell-python/hello_world.html
- Code Examples: https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html
- Geometry Settings: https://docs.ifcopenshell.org/ifcopenshell-python/geometry_settings.html
- Geometry Processing: https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html
- Geometry Creation: https://docs.ifcopenshell.org/ifcopenshell-python/geometry_creation.html
- IfcTester: https://docs.ifcopenshell.org/ifctester.html
- IfcPatch: https://docs.ifcopenshell.org/ifcpatch.html
- IfcClash: https://docs.ifcopenshell.org/ifcclash.html
- Wall Tutorial: https://academy.ifcopenshell.org/posts/creating-a-simple-wall-with-property-set-and-quantity-information/

### Bonsai URLs (from SOURCES.md)
- Bonsai Docs: https://docs.bonsaibim.org/
- Bonsai Source: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
- Bonsai on Extensions: https://extensions.blender.org/add-ons/bonsai/

### IFC Standard URLs (from SOURCES.md)
- buildingSMART Technical: https://technical.buildingsmart.org/standards/
- IFC4.3 Docs: https://ifc43-docs.standards.buildingsmart.org/
- IFC Schema Specs: https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/
- bSDD: https://search.bsdd.buildingsmart.org/
- IDS Standard: https://www.buildingsmart.org/standards/bsi-standards/information-delivery-specification-ids/

---

*Generated by masterplan-refinement agent, 2026-03-06. This document is SELF-CONTAINED: a new agent reading ONLY this file knows exactly what to do.*
