# Blender Skill Inventory Briefing

**Date**: 2026-03-06
**Author**: interpret-blender agent
**Purpose**: Condensed briefing for masterplan agent to define definitive Blender skill inventory
**Sources**: vooronderzoek-blender.md, supplementary-blender-gaps.md, scope-analysis.md, raw-masterplan.md

---

## 1. Definitive Skill List (26 Skills)

### Syntax Skills (11)

#### S-01: blender-syntax-operators
- **Category**: syntax
- **Scope**: bpy.types.Operator class, execute/invoke/modal methods, poll(), bl_idname naming, bl_options flags, return values, operator properties.
- **Key API**: `bpy.types.Operator`, `bpy.ops.*`, `context.window_manager.invoke_props_dialog()`, `modal_handler_add()`
- **Sources**: vooronderzoek §4 (L609-751)
- **SOURCES.md URLs**: Blender Python API, 4.0 Python API Changes
- **Version sensitivity**: 4.0 removes context override dict (must use `temp_override`); 4.2 adds `MODAL_PRIORITY`
- **Dependencies**: blender-core-api, blender-errors-context
- **Complexity**: M

#### S-02: blender-syntax-properties
- **Category**: syntax
- **Scope**: All bpy.props types (Bool/Int/Float/String/Enum/Vector/Pointer/Collection), PropertyGroup, subtypes, units, update callbacks, dynamic enum items, getter/setter.
- **Key API**: `bpy.props.*`, `bpy.types.PropertyGroup`, `bpy.utils.register_class()`
- **Sources**: vooronderzoek §5 (L754-915)
- **SOURCES.md URLs**: Blender Python API (`bpy.props.html`)
- **Version sensitivity**: 4.1 adds enum ID properties; 5.0 removes `del obj['prop']` for RNA props (use `property_unset`)
- **Dependencies**: blender-syntax-operators (properties on operators)
- **Complexity**: M

#### S-03: blender-syntax-panels
- **Category**: syntax
- **Scope**: bpy.types.Panel, draw(), UILayout API (row/column/box/split), bl_space_type, bl_region_type, bl_category, sub-panels, draw_header, menus, UIList.
- **Key API**: `bpy.types.Panel`, `bpy.types.UILayout`, `bpy.types.Menu`, `bpy.types.UIList`
- **Sources**: vooronderzoek §6 (L918-1000+)
- **SOURCES.md URLs**: Blender Python API
- **Version sensitivity**: 4.1 adds `layout.panel()` for collapsible sections without registration
- **Dependencies**: blender-syntax-properties
- **Complexity**: M

#### S-04: blender-syntax-addons
- **Category**: syntax
- **Scope**: Legacy bl_info dict, blender_manifest.toml (4.2+), register/unregister, multi-file addon structure, AddonPreferences, class naming convention, extension packaging.
- **Key API**: `bpy.utils.register_class()`, `bpy.types.AddonPreferences`, `bpy.context.preferences.addons`
- **Sources**: vooronderzoek §3 (L450-606)
- **SOURCES.md URLs**: Extensions Developer Handbook, Add-on Guidelines, Extension Hosting
- **Version sensitivity**: 4.2 introduces blender_manifest.toml; legacy bl_info still works through 5.x
- **Dependencies**: blender-syntax-operators, blender-syntax-panels
- **Complexity**: M

#### S-05: blender-syntax-mesh
- **Category**: syntax
- **Scope**: Mesh data (vertices/edges/faces/loops), BMesh creation and editing, from_pydata, UV layers, vertex colors/attributes, normals, foreach_get/set.
- **Key API**: `bpy.types.Mesh`, `bmesh`, `bmesh.from_mesh()`, `bmesh.to_mesh()`, `mesh.from_pydata()`, `mesh.attributes`
- **Sources**: vooronderzoek §7 (L1000+)
- **SOURCES.md URLs**: Blender Python API
- **Version sensitivity**: 4.0 removes bevel_weight/crease properties (→ attributes); 4.1 removes use_auto_smooth/calc_normals_split; 4.3 splits AttributeGroup into type-specific classes
- **Dependencies**: blender-core-api
- **Complexity**: L

#### S-06: blender-syntax-modifiers
- **Category**: syntax
- **Scope**: Modifier stack, obj.modifiers, apply vs preview, evaluated mesh via depsgraph, Geometry Nodes modifier input identifiers, common modifiers for AEC.
- **Key API**: `obj.modifiers.new()`, `bpy.ops.object.modifier_apply()`, `obj.evaluated_get()`, `depsgraph`
- **Sources**: vooronderzoek §8 (L1050+), supplementary §1.5 (L165-186)
- **SOURCES.md URLs**: Blender Python API
- **Version sensitivity**: 4.0 requires `temp_override` for modifier_apply; GN modifier inputs use auto-generated `Socket_N` identifiers
- **Dependencies**: blender-core-api, blender-syntax-nodes
- **Complexity**: M

#### S-07: blender-syntax-nodes
- **Category**: syntax
- **Scope**: Node tree architecture (NodeTree, Node, NodeLink, NodeSocket), Geometry Nodes via Python, Shader Nodes scripting, Compositor Nodes, node group interface API, socket types. Covers ALL node systems.
- **Key API**: `bpy.data.node_groups`, `NodeTree.interface.new_socket()`, `tree.nodes.new()`, `tree.links.new()`, `NodeTreeInterfacePanel`
- **Sources**: supplementary §1 (L24-252), vooronderzoek §2 node interface migration (L134-143)
- **SOURCES.md URLs**: Blender Python API (NodeTree, NodeTreeInterface, NodeSocket), 4.0/5.0 release notes
- **Version sensitivity**: 4.0 moves interface from `tree.inputs/outputs` to `tree.interface.new_socket()`; 4.1 adds interface panels; 5.0 changes compositor from `scene.node_tree` to `scene.compositing_node_group`
- **Dependencies**: blender-core-api
- **Complexity**: L (broad topic — use references/ for socket type tables and node type lists)

#### S-08: blender-syntax-animation
- **Category**: syntax
- **Scope**: Keyframe insertion, FCurves (access, creation, update), keyframe interpolation, Drivers (variables, expressions), Constraints, Armatures/Bones/Bone Collections, NLA strips, Actions, Shape Keys.
- **Key API**: `keyframe_insert()`, `FCurve`, `Driver`, `driver_add()`, `obj.constraints.new()`, `armature.edit_bones`, `armature.collections`, `NlaTrack`, `ShapeKey`
- **Sources**: supplementary §2 (L255-583)
- **SOURCES.md URLs**: Blender Python API (FCurve, Keyframe, BoneCollection, EditBone), 4.0 bone collections migration
- **Version sensitivity**: 4.0 removes bone.layers and pose.bone_groups (→ bone collections); edit_bones only in Edit Mode
- **Dependencies**: blender-core-api, blender-syntax-properties
- **Complexity**: L (broad topic — keep SKILL.md focused on decision trees, detail in references/)

#### S-09: blender-syntax-materials
- **Category**: syntax
- **Scope**: Material creation/assignment, Principled BSDF setup (with 3.x→4.0+ socket rename table), shader node tree building, UV mapping via Python, image texture loading, color space settings, AEC material presets (glass, concrete).
- **Key API**: `bpy.data.materials.new()`, `mat.node_tree`, `ShaderNodeBsdfPrincipled`, `mesh.uv_layers`, `bpy.data.images.load()`
- **Sources**: supplementary §3 (L586-795)
- **SOURCES.md URLs**: Blender Python API (Material, ShaderNodeBsdfPrincipled, UVLoopLayers), 4.0 release notes
- **Version sensitivity**: 4.0 renames Principled BSDF sockets (Subsurface→Subsurface Weight, etc.); 4.2 removes `mat.blend_method` (→ `mat.surface_render_method`)
- **Dependencies**: blender-syntax-nodes
- **Complexity**: M

#### S-10: blender-syntax-rendering
- **Category**: syntax
- **Scope**: Scene.render settings, EEVEE vs Cycles configuration via Python, camera setup (lens, DOF, clipping), light creation (Point/Sun/Spot/Area), output format, batch rendering, headless rendering, render passes.
- **Key API**: `scene.render`, `scene.cycles`, `scene.eevee`, `bpy.data.cameras`, `bpy.data.lights`, `bpy.ops.render.render()`
- **Sources**: supplementary §4 (L799-1062)
- **SOURCES.md URLs**: Blender Python API (RenderSettings, SceneEEVEE), 5.0 release notes
- **Version sensitivity**: EEVEE identifier: `BLENDER_EEVEE` (3.x/4.0-4.1) → `BLENDER_EEVEE_NEXT` (4.2-4.4) → `BLENDER_EEVEE` (5.0+); 5.0 renames render pass names; 5.0 removes eevee.use_gtao
- **Dependencies**: blender-core-api
- **Complexity**: M

#### S-11: blender-syntax-data
- **Category**: syntax
- **Scope**: Collection management (hierarchy, visibility), library linking/appending, library overrides (4.0+), Asset Browser API (mark, metadata, tags, catalogs), ID data lifecycle (user count, fake user).
- **Key API**: `bpy.data.collections`, `bpy.data.libraries.load()`, `obj.asset_mark()`, `AssetMetaData`, `IDOverrideLibrary`
- **Sources**: supplementary §6 (L1245-1465), vooronderzoek §1 data hierarchy (L47-66)
- **SOURCES.md URLs**: Blender Python API (Collection, BlendDataLibraries, AssetMetaData), 5.0 release notes
- **Version sensitivity**: 4.0 removes proxies (→ library overrides); 5.0 removes `context.asset_file_handle` (→ `context.asset`)
- **Dependencies**: blender-core-api
- **Complexity**: M

### Implementation Skills (6)

#### I-01: blender-impl-operators
- **Category**: impl
- **Scope**: When to use which operator type (simple/invoke/modal), undo/redo patterns, modal timer pattern, invoke with dialog, context.temp_override usage patterns, operator chaining.
- **Sources**: vooronderzoek §4 (L680-715), §9 context system
- **Dependencies**: blender-syntax-operators, blender-errors-context
- **Complexity**: M

#### I-02: blender-impl-addons
- **Category**: impl
- **Scope**: Full addon/extension development workflow, packaging for extensions.blender.org, multi-file structure, testing, migration from bl_info to blender_manifest.toml, wheel bundling.
- **Sources**: vooronderzoek §3 (L450-606)
- **SOURCES.md URLs**: Extensions Developer Handbook, Add-on Dev Setup, Extension Hosting
- **Dependencies**: blender-syntax-addons, blender-syntax-operators, blender-syntax-panels
- **Complexity**: M

#### I-03: blender-impl-mesh
- **Category**: impl
- **Scope**: Mesh creation workflows, BMesh vs direct data access decision tree, performance patterns (foreach_get/set), from_pydata usage, mesh editing in edit mode vs object mode, attribute-based workflow (4.0+).
- **Sources**: vooronderzoek §7
- **Dependencies**: blender-syntax-mesh, blender-core-api
- **Complexity**: M

#### I-04: blender-impl-automation
- **Category**: impl
- **Scope**: Batch operations, headless/background mode rendering, CLI Blender, I/O format scripting (FBX/glTF/USD/OBJ/STL export/import with AEC-relevant parameters), subprocess usage, bpy as module.
- **Sources**: supplementary §5 I/O formats (L1065-1242), §8.7-8.8 background mode (L2116-2170)
- **SOURCES.md URLs**: Blender Python API (export_scene, import_scene, wm)
- **Version sensitivity**: 4.0 removes Python OBJ/STL exporters (→ `bpy.ops.wm.obj_export/stl_export`)
- **Dependencies**: blender-syntax-rendering, blender-core-runtime
- **Complexity**: L

#### I-05: blender-impl-nodes
- **Category**: impl
- **Scope**: Building Geometry Node groups via Python for AEC (wall generators, parametric structures), procedural material creation, modifier input value setting, node group reuse patterns.
- **Sources**: supplementary §1 (L24-252), §3.4 AEC materials (L706-725)
- **Dependencies**: blender-syntax-nodes, blender-syntax-materials, blender-syntax-modifiers
- **Complexity**: L

#### I-06: blender-impl-animation
- **Category**: impl
- **Scope**: Rigging workflows for AEC (building rigs, floor controllers), constraint setup patterns, driver expressions, animation baking, NLA workflow, shape key-driven parametric deformation.
- **Sources**: supplementary §2 (L255-583)
- **Dependencies**: blender-syntax-animation
- **Complexity**: M

### Error Skills (3)

#### E-01: blender-errors-context
- **Category**: errors
- **Scope**: Context override errors, wrong context for operators, restricted context, temp_override patterns, poll() failures and diagnosis, operator return value errors, registration errors, wrong bl_options. Merged topic: operator errors are context errors.
- **Sources**: vooronderzoek §9 context system, §11 common errors, §12 AI mistakes
- **Dependencies**: blender-syntax-operators, blender-core-api
- **Complexity**: M

#### E-02: blender-errors-data
- **Category**: errors
- **Scope**: Orphaned data, reference counting, memory management, stale references after undo, undo invalidation (the #1 runtime danger), ID data lifecycle errors.
- **Sources**: vooronderzoek §11 (error patterns), supplementary §8.2 undo invalidation (L1903-1943)
- **Dependencies**: blender-core-api, blender-core-runtime
- **Complexity**: M

#### E-03: blender-errors-version
- **Category**: errors
- **Scope**: Version-specific pitfalls: BGL→gpu (5.0), bone.layers→collections (4.0), EEVEE identifier changes, Principled BSDF renames (4.0), GP rewrite (4.3), mesh attribute migration (4.0), auto_smooth removal (4.1), OBJ/STL operator migration (4.0).
- **Sources**: vooronderzoek §2 version matrix (L81-446), supplementary §4.9 rendering (L1045-1056)
- **Dependencies**: blender-core-versions
- **Complexity**: L

### Core Skills (4)

#### C-01: blender-core-api
- **Category**: core
- **Scope**: bpy module overview (data/context/ops/types/props/utils/app), RNA introspection system, ID data blocks, context system (active object, mode, area), dependency graph (depsgraph) basics, operators vs direct data access.
- **Key API**: `bpy.data`, `bpy.context`, `bpy.ops`, `bpy.types`, `depsgraph`
- **Sources**: vooronderzoek §1 (L29-78), §9 context, §10 depsgraph
- **SOURCES.md URLs**: Blender Python API (info_quickstart, info_overview)
- **Dependencies**: None (foundation skill)
- **Complexity**: M

#### C-02: blender-core-versions
- **Category**: core
- **Scope**: Complete version matrix for Blender 3.x/4.0/4.1/4.2/4.3/5.0/5.1, ALL breaking changes with migration paths, version detection (`bpy.app.version`), version-safe coding patterns.
- **Sources**: vooronderzoek §2 (L81-446)
- **SOURCES.md URLs**: All version-specific release notes (4.0 through 5.2), Compatibility Index, Python API Changelog
- **Dependencies**: None (reference skill)
- **Complexity**: L

#### C-03: blender-core-gpu
- **Category**: core
- **Scope**: gpu module (replaces bgl), built-in shaders (UNIFORM_COLOR, POLYLINE_*, SMOOTH_COLOR, IMAGE), batch_for_shader, gpu.state (blend, depth, line_width), SpaceView3D draw handlers, offscreen rendering, BGL→gpu migration checklist.
- **Key API**: `gpu.shader.from_builtin()`, `gpu_extras.batch.batch_for_shader()`, `gpu.state.*`, `gpu.types.GPUOffScreen`
- **Sources**: vooronderzoek §2 BGL migration (L311-435), supplementary §7.8 (L1738-1829)
- **SOURCES.md URLs**: Blender Python API (gpu, gpu.shader, gpu.types), 5.0 release notes
- **Version sensitivity**: bgl deprecated 3.5, removed 5.0; shader name `3D_UNIFORM_COLOR` → `POLYLINE_UNIFORM_COLOR` in 4.0
- **Dependencies**: blender-core-api
- **Complexity**: M

#### C-04: blender-core-runtime
- **Category**: core
- **Scope**: mathutils module (Vector, Matrix, Quaternion, Euler, KDTree, BVHTree, bl_math), Python threading restrictions, undo invalidation of bpy references, application handlers (@persistent), bpy.app.timers, bpy.msgbus, background mode limitations, bpy as module.
- **Key API**: `mathutils.*`, `bpy.app.timers`, `bpy.app.handlers`, `bpy.msgbus.subscribe_rna()`, `bpy.app.background`
- **Sources**: supplementary §7 mathutils (L1468-1851), §8 runtime quirks (L1854-2193)
- **SOURCES.md URLs**: Blender Python API (mathutils, bpy.app.timers, bpy.msgbus, bpy.app.handlers, info_gotchas)
- **Version sensitivity**: 5.0 removes IDProperty dict access to RNA props; `del obj['prop']` → `property_unset()`
- **Dependencies**: blender-core-api
- **Complexity**: L

### Agent Skills (2)

#### A-01: blender-agents-code-validator
- **Category**: agents
- **Scope**: Validate Blender Python scripts for: correct context usage, API version compatibility, deprecated API calls, anti-pattern detection, naming convention compliance.
- **Sources**: vooronderzoek §11-12 (error patterns, AI mistakes)
- **Dependencies**: All syntax + error skills
- **Complexity**: M

#### A-02: blender-agents-version-migrator
- **Category**: agents
- **Scope**: Auto-detect version issues in scripts, suggest migration from 3.x→4.x→5.x, flag deprecated API calls, provide replacement code, version-safe wrapper generation.
- **Sources**: vooronderzoek §2 (L81-446), blender-core-versions
- **Dependencies**: blender-core-versions, blender-errors-version
- **Complexity**: M

---

## 2. Merge/Split Recommendations

### MERGE (from raw masterplan)
| Raw Skill | Merge Into | Reason |
|-----------|-----------|--------|
| blender-errors-operators | blender-errors-context | Operator errors are almost always context errors (poll failures, wrong mode, restricted context). One skill covers both. |

### SPLIT (from raw masterplan)
| Raw Skill | Split Into | Reason |
|-----------|-----------|--------|
| blender-syntax-nodes | Keep unified | Supplementary §1 shows GN/Shader/Compositor share the same NodeTree/Node/NodeLink API. Split is unnecessary if references/ holds node type tables. Use complexity L. |
| blender-syntax-animation | Keep unified | Supplementary §2 shows animation+rigging are tightly coupled (armatures need keyframes, constraints need drivers). Use complexity L with references/ for API tables. |

### ADD (not in raw masterplan)
| New Skill | Reason |
|-----------|--------|
| blender-syntax-materials | Supplementary §3 reveals extensive AEC-relevant content: Principled BSDF socket renames (4.0 breaking change), material assignment, UV mapping, color spaces. This was entirely missing from raw masterplan. |
| blender-core-runtime | Supplementary §7-8 cover mathutils (KDTree, BVHTree critical for AEC clash detection) and Python runtime quirks (threading, undo, handlers, timers, msgbus). Too large for blender-core-api. |

### REMOVE (from raw masterplan)
| Raw Skill | Reason |
|-----------|--------|
| blender-syntax-rendering (as named) | KEEP but renamed — the raw masterplan scope was unclear. Now explicitly covers camera/lights/engine config. |
| None removed | All raw masterplan skills are AEC-relevant. No removals needed. |

### NET CHANGE: 25 → 26 skills (+1 material, +1 runtime, -1 merged errors)

---

## 3. Key Patterns Summary

### Top 10 Blender Python Patterns for AEC

**1. Context temp_override (4.0+)**
```python
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Boolean")
```

**2. BMesh edit cycle**
```python
bm = bmesh.new()
bm.from_mesh(obj.data)
# ... modify geometry ...
bm.to_mesh(obj.data)
bm.free()
```

**3. Geometry Node group creation (4.0+)**
```python
ng = bpy.data.node_groups.new("WallGen", 'GeometryNodeTree')
ng.interface.new_socket(name="Height", in_out='INPUT', socket_type='NodeSocketFloat')
```

**4. Version-safe Principled BSDF**
```python
name = "Subsurface Weight" if bpy.app.version >= (4, 0, 0) else "Subsurface"
principled.inputs[name].default_value = 0.5
```

**5. Collection hierarchy for AEC**
```python
building = bpy.data.collections.new("Building_A")
bpy.context.scene.collection.children.link(building)
floors = bpy.data.collections.new("Floors")
building.children.link(floors)
```

**6. Headless batch rendering**
```python
# CLI: blender --background scene.blend --python render.py
for cam in [o for o in bpy.data.objects if o.type == 'CAMERA']:
    scene.camera = cam
    scene.render.filepath = f"/tmp/{cam.name}_"
    bpy.ops.render.render(write_still=True)
```

**7. KDTree spatial queries (clash detection)**
```python
kd = KDTree(len(mesh.vertices))
for i, v in enumerate(mesh.vertices):
    kd.insert(v.co, i)
kd.balance()
nearby = kd.find_range(point, radius=2.0)
```

**8. Application handler with @persistent**
```python
@persistent
def on_load(filepath):
    # Re-initialize addon state
bpy.app.handlers.load_post.append(on_load)
```

**9. Library link + override**
```python
with bpy.data.libraries.load(path, link=True) as (src, dst):
    dst.collections = ["Module_A"]
# Then: bpy.ops.object.make_override_library()
```

**10. Safe reference handling (undo-proof)**
```python
obj_name = obj.name  # Store NAME, not reference
# ... after undo/load ...
obj = bpy.data.objects.get(obj_name)  # Re-fetch
```

### Top 5 Version Pitfalls

| # | Change | Versions | Impact |
|---|--------|----------|--------|
| 1 | Context override dict → `context.temp_override()` | 3.x → 4.0 | CRITICAL — breaks ALL operator scripts using old override pattern |
| 2 | Node group `tree.inputs/outputs` → `tree.interface.new_socket()` | 3.x → 4.0 | HIGH — breaks all GN/shader node creation scripts |
| 3 | Principled BSDF socket renames (Subsurface→Subsurface Weight, etc.) | 3.x → 4.0 | HIGH — breaks all material scripts |
| 4 | BGL module completely removed → `gpu` module | ≤4.x → 5.0 | CRITICAL — breaks ALL viewport drawing addons |
| 5 | EEVEE engine identifier flip-flop | 3.x→4.2→5.0 | MEDIUM — `BLENDER_EEVEE` → `BLENDER_EEVEE_NEXT` → `BLENDER_EEVEE` |

### Top 5 Anti-Patterns

| # | Anti-Pattern | Consequence | Fix |
|---|-------------|-------------|-----|
| 1 | Calling `bpy.*` from a thread | CRASH — Blender is not thread-safe | Use `queue.Queue` + `bpy.app.timers` for main-thread execution |
| 2 | Storing `bpy.data` references across undo | CRASH or `ReferenceError` — undo invalidates ALL Python references | Store names/IDs, re-fetch via `bpy.data.objects.get()` |
| 3 | `matrix1 * matrix2` for transforms | Element-wise multiplication, NOT matrix product | Use `matrix1 @ matrix2` (@ operator) |
| 4 | Calling operators without checking poll | `RuntimeError: Operator bpy.ops.xxx.poll() failed` | Check mode, object type, and selection before calling |
| 5 | Forgetting `fcurve.update()` after manual keyframe edits | Stale keyframe cache, wrong animation | ALWAYS call `update()` after modifying keyframe_points |

---

## 4. Source Coverage Map

| Skill | Vooronderzoek Section | Supplementary Section | Key SOURCES.md URLs |
|-------|----------------------|----------------------|---------------------|
| S-01 operators | §4 Operators (L609-751) | — | Blender Python API |
| S-02 properties | §5 Properties (L754-915) | — | Blender Python API (`bpy.props.html`) |
| S-03 panels | §6 UI Panels (L918-1000+) | — | Blender Python API |
| S-04 addons | §3 Addon Arch (L450-606) | — | Extensions Dev Handbook, Add-on Guidelines |
| S-05 mesh | §7 Mesh & BMesh | — | Blender Python API |
| S-06 modifiers | §8 Modifiers | §1.5 GN modifier (L165-186) | Blender Python API |
| S-07 nodes | §2 node interface migration (L134-143) | §1 Node Systems (L24-252) | Blender API (NodeTree, NodeTreeInterface), 4.0/5.0 notes |
| S-08 animation | — | §2 Animation & Rigging (L255-583) | Blender API (FCurve, BoneCollection), 4.0 bone migration |
| S-09 materials | — | §3 Materials & Shading (L586-795) | Blender API (Material, ShaderNodeBsdfPrincipled), 4.0 notes |
| S-10 rendering | — | §4 Rendering (L799-1062) | Blender API (RenderSettings, SceneEEVEE), 5.0 notes |
| S-11 data | §1 data hierarchy (L47-66) | §6 Collections/Libraries/Assets (L1245-1465) | Blender API (Collection, BlendDataLibraries, AssetMetaData) |
| I-01 impl-ops | §4 modal/invoke (L680-715), §9 context | — | Blender Python API |
| I-02 impl-addons | §3 (L450-606) | — | Extensions Dev Handbook, Extension Hosting |
| I-03 impl-mesh | §7 | — | Blender Python API |
| I-04 impl-automation | — | §5 I/O Formats (L1065-1242), §8.7-8.8 (L2116-2170) | Blender API (export_scene, wm) |
| I-05 impl-nodes | — | §1 (L24-252), §3.4 AEC mats (L706-725) | Blender Python API |
| I-06 impl-animation | — | §2 (L255-583) | Blender Python API |
| E-01 errors-context | §9 Context, §11 Errors, §12 AI | §8.3 Restricted context (L1946-1970) | Blender Python API |
| E-02 errors-data | §11 Error patterns | §8.2 Undo invalidation (L1903-1943) | Blender API (info_gotchas) |
| E-03 errors-version | §2 Version Matrix (L81-446) | §4.9 Rendering (L1045-1056) | All release notes (4.0-5.2), Compatibility Index |
| C-01 core-api | §1 Overview (L29-78), §9, §10 | — | Blender API (info_quickstart, info_overview) |
| C-02 core-versions | §2 (L81-446) | All version-specific changes | All release notes, Compatibility Index, API Changelog |
| C-03 core-gpu | §2 BGL migration (L311-435) | §7.8 gpu module (L1738-1829) | Blender API (gpu, gpu.shader, gpu.types), 5.0 notes |
| C-04 core-runtime | — | §7 mathutils (L1468-1851), §8 Runtime (L1854-2193) | Blender API (mathutils, timers, msgbus, handlers, info_gotchas) |
| A-01 code-validator | §11-12 | — | All Blender sources |
| A-02 version-migrator | §2 (L81-446) | All migration sections | All release notes |

---

## 5. Build Order Recommendation

Based on dependency analysis:

```
Batch 1 (Foundation):  C-01 core-api, C-02 core-versions
Batch 2 (Core):        C-03 core-gpu, C-04 core-runtime
Batch 3 (Syntax A):    S-01 operators, S-02 properties, S-03 panels, S-04 addons
Batch 4 (Syntax B):    S-05 mesh, S-06 modifiers, S-07 nodes
Batch 5 (Syntax C):    S-08 animation, S-09 materials, S-10 rendering, S-11 data
Batch 6 (Impl):        I-01 through I-06
Batch 7 (Errors):      E-01 errors-context, E-02 errors-data, E-03 errors-version
Batch 8 (Agents):      A-01 code-validator, A-02 version-migrator
```

**Parallelization**: Within each batch, skills with no mutual dependencies can be written in parallel by separate agents.

---

## 6. Complexity Summary

| Complexity | Count | Skills |
|-----------|-------|--------|
| S (Small) | 0 | — |
| M (Medium) | 17 | S-01,S-02,S-03,S-04,S-06,S-09,S-10,S-11, I-01,I-02,I-03,I-06, E-01,E-02, C-01,C-03, A-01,A-02 |
| L (Large) | 9 | S-05,S-07,S-08, I-04,I-05, E-03, C-02,C-04 |

**L-complexity skills** will need careful use of `references/` directory to stay under 500 lines in SKILL.md.
