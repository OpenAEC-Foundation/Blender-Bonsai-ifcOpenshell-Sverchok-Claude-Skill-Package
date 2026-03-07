# RAW Masterplan — Sverchok Claude Skill Package

**Date**: 2026-03-07
**Phase**: S2 — Sverchok Masterplan (Raw)
**Status**: RAW — Requires review before skill creation
**Author**: sv-masterplan-raw agent
**Input**: vooronderzoek-sverchok.md (2352 lines, 78KB)

---

## 1. Executive Summary

### Skill Counts

| Category | Count | Skills |
|----------|-------|--------|
| core | 1 | sverchok-core-concepts |
| syntax | 4 | sverchok-syntax-sockets, sverchok-syntax-data, sverchok-syntax-scripting, sverchok-syntax-api |
| impl | 4 | sverchok-impl-custom-nodes, sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-extensions |
| errors | 1 | sverchok-errors-common |
| agents | 1 | sverchok-agents-code-validator |
| **Total** | **11** | |

### Build Timeline

- **6 batches**, 1–3 skills per batch
- **Batch 1**: Foundation (core-concepts, no deps)
- **Batch 2**: Core syntax (sockets, data — depend on core)
- **Batch 3**: Advanced syntax + custom nodes (scripting, api, custom-nodes — depend on batch 2)
- **Batch 4**: Implementation (parametric, ifcsverchok, extensions — depend on batch 3)
- **Batch 5**: Errors (depends on impl knowledge)
- **Batch 6**: Agents (depends on everything)
- Estimated duration: 6 quality-gated batch cycles

### Scope Boundaries

**IN scope**:
- Sverchok core architecture: node tree system, update system, socket cache, events
- Socket type system: 16 socket types, implicit conversions, processing flags
- Data nesting levels and list matching (the #1 source of errors)
- Python scripting: SNLite, SN Functor B, Formula Mk5, Profile Mk3
- Custom node development: full lifecycle, registration, BMesh integration
- External API: programmatic tree control, batch processing
- Parametric design workflows and real-world AEC patterns
- IfcSverchok: 31 IFC nodes, SvIfcStore, geometry modes, BIM workflow
- Extensions: TopologicSverchok, Sverchok-Extra, extension development pattern
- Error patterns and AI common mistakes (17 documented anti-patterns)
- Advanced data types: curves, surfaces, fields, solids

**OUT of scope**:
- Individual node documentation for all 500+ built-in nodes (reference only)
- Pulga Physics simulation details
- Blender-internal node systems (Geometry Nodes, Shader Nodes)
- Sculpting, video editing, and non-AEC Blender workflows

**Dependencies on other packages**:
- `sverchok-impl-ifcsverchok` references IfcOpenShell API (covered by ifcos-* skills)
- `sverchok-impl-extensions` references TopologicSverchok external dependency
- `sverchok-syntax-api` builds on Blender bpy.data knowledge (covered by blender-core-api)

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Extensions merged into single skill | TopologicSverchok, Sverchok-Extra, and extension dev pattern share the same registration architecture; separating them would create 3 very small skills |
| Data nesting + list matching = one skill | §4 (nesting) and §5 (matching/vectorization) are inseparable — matching only makes sense with correct nesting understanding |
| External API as syntax skill | Programmatic tree access (§9) is API syntax/patterns, not a workflow guide |
| Node categories overview in core-concepts | §6 (500+ node categories) is reference material, not deep API documentation; fits as part of architecture overview |
| Advanced data types in syntax-sockets | Curves, surfaces, fields, solids (§15) are socket data types; belong with socket type documentation |
| Single errors skill | 7 error patterns + 10 AI mistakes = 17 items; fits comfortably in one skill < 500 lines |

---

## 2. Skill Inventory

### 2.1 Core Skills (1)

#### SV-C-01: sverchok-core-concepts
- **Category**: core
- **Scope**: Sverchok architecture overview, SverchCustomTree class, SverchCustomTreeNode base class and mixins (UpdateNodes, NodeUtils, NodeDependencies, NodeDocumentation), socket data cache system, update system (control_center, SearchTree, UpdateTree), event classes (TreeEvent, PropertyEvent, AnimationEvent, etc.), node execution order (topological sort), data propagation between nodes, execution statistics, node categories overview (500+ nodes in 18+ categories).
- **Key API surface**: `SverchCustomTree`, `SverchCustomTreeNode`, `socket_data_cache`, `SearchTree`, `UpdateTree`, `control_center()`, `prepare_input_data()`, event classes from `core/events.py`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
  - https://nortikin.github.io/sverchok/
  - https://sverchok.readthedocs.io/en/latest/nodes.html
- **Research input**: vooronderzoek-sverchok §1 (L30-120), §2 (L122-288), §6 (L720-778)
- **Dependencies**: None (foundation skill)
- **Complexity**: L

### 2.2 Syntax Skills (4)

#### SV-S-01: sverchok-syntax-sockets
- **Category**: syntax
- **Scope**: All 16 socket types (SvStringsSocket, SvVerticesSocket, SvMatrixSocket, SvQuaternionSocket, SvColorSocket, SvCurveSocket, SvSurfaceSocket, SvSolidSocket, SvScalarFieldSocket, SvVectorFieldSocket, SvDictionarySocket, SvObjectSocket, SvFilePathSocket, SvTextSocket, SvDummySocket, SvChameleonSocket), socket base class API (SvSocketCommon), sv_get/sv_set/sv_forget methods, socket properties (color, label, use_prop, prop_name, nesting_level, default_mode, pre_processing), implicit conversion policies (Default, Field, Lenient, Solid), conversion policy registry, lenient socket types, socket data processing flags (flatten, simplify, graft, unwrap, wrap), advanced data types (curves/SvCurve, surfaces/SvSurface, scalar fields/SvScalarField, vector fields/SvVectorField, solids/Part.Shape, dictionaries).
- **Key API surface**: `SvSocketCommon`, `SvStringsSocket`, `SvVerticesSocket`, `SvMatrixSocket`, `SvCurveSocket`, `SvSurfaceSocket`, `SvScalarFieldSocket`, `SvVectorFieldSocket`, `ConversionPolicies`, `DefaultImplicitConversionPolicy`, `FieldImplicitConversionPolicy`, `socket.sv_get()`, `socket.sv_set()`, `socket.sv_forget()`, `socket.replace_socket()`, `socket.socket_id`, `socket.other`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok/blob/master/core/sockets.py
  - https://github.com/nortikin/sverchok/blob/master/core/socket_conversions.py
  - https://github.com/nortikin/sverchok/blob/master/core/socket_data.py
- **Research input**: vooronderzoek-sverchok §3 (L290-431), §15 (L2291-2333)
- **Dependencies**: sverchok-core-concepts
- **Complexity**: M

#### SV-S-02: sverchok-syntax-data
- **Category**: syntax
- **Scope**: Data nesting level convention (levels 0-3), standard data formats per socket type (SvStringsSocket=level 2, SvVerticesSocket=level 3, SvMatrixSocket=level 1), nesting level detection (get_data_nesting_level), the "objects" mental model, list matching system (5 modes: REPEAT, CYCLE, SHORT, XREF, XREF2), core matching functions (match_long_repeat, fullList, repeat_last, list_match_func), NumPy matching variants, the vectorize decorator, SvRecursiveNode mixin, socket processing modes (input preprocessing: flatten/simplify/graft/unwrap/wrap, output postprocessing), match_sockets helper, recursive processing utilities (recurse_fx, recurse_fxy, recurse_f_level_control, process_matched).
- **Key API surface**: `match_long_repeat()`, `fullList()`, `repeat_last()`, `get_data_nesting_level()`, `list_match_func`, `vectorize()`, `SvRecursiveNode`, `match_sockets()`, `recurse_fx()`, `recurse_fxy()`, `numpy_list_match_func`, `DataWalker`, `walk_data()`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok/blob/master/data_structure.py
  - https://github.com/nortikin/sverchok/blob/master/utils/vectorize.py
  - https://github.com/nortikin/sverchok/blob/master/utils/sv_itertools.py
  - https://github.com/nortikin/sverchok/blob/master/utils/nodes_mixins/recursive_nodes.py
- **Research input**: vooronderzoek-sverchok §4 (L435-520), §5 (L522-716)
- **Dependencies**: sverchok-core-concepts
- **Complexity**: L (CRITICAL — #1 error source, must be thorough)

#### SV-S-03: sverchok-syntax-scripting
- **Category**: syntax
- **Scope**: Script Node Lite (SNLite) — socket declaration syntax, socket type identifiers (s/v/m/o/C/S/So/SF/VF/D/FP), options (default, nested, required), built-in aliases (vectorize, bpy, np, bmesh_from_pydata, etc.), special functions (setup(), ui(), sv_internal_links()), custom enums, file handler, includes, template system, persistent per-node storage (get_user_dict). SN Functor B — functor_init/process/draw_buttons functions, pre-defined properties (int_00-04, float_00-04, bool_00-04). Formula Node Mk5 — expression evaluation, safe_eval system, available functions/constants, NumPy mode. Profile Node Mk3 — SVG-like DSL commands (M/L/H/V/C/Q/A/@I/x/X), variable system, architectural profile examples. Other script nodes overview (Generative Art, Formula Interpolate, Mesh Expression, Multi Exec, NumExpr).
- **Key API surface**: `SvScriptNodeLite`, `SvSNFunctorB`, `SvFormulaNodeMk5`, `SvProfileNodeMK3`, SNLite aliases dict, `safe_eval()`, profile DSL commands
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok/blob/master/nodes/script/script1_lite.py
  - https://github.com/nortikin/sverchok/blob/master/nodes/script/sn_functor_b.py
  - https://github.com/nortikin/sverchok/blob/master/nodes/script/formula_mk5.py
  - https://github.com/nortikin/sverchok/blob/master/nodes/script/profile_mk3.py
  - https://github.com/nortikin/sverchok/tree/master/node_scripts/SNLite_templates
- **Research input**: vooronderzoek-sverchok §7 (L782-1199)
- **Dependencies**: sverchok-syntax-sockets, sverchok-syntax-data
- **Complexity**: L

#### SV-S-04: sverchok-syntax-api
- **Category**: syntax
- **Scope**: External API access to Sverchok node trees, accessing trees via bpy.data.node_groups (filtering by bl_idname='SverchCustomTreeType'), creating nodes programmatically, creating links, modifying node properties, triggering tree updates (force_update, update_nodes, sv_process), reading node output data via sv_get(), batch processing with parameter sweeps, init_tree context manager for suppressing intermediate updates, key internal API modules (sverchok.data_structure complete reference, sverchok.utils.sv_itertools).
- **Key API surface**: `bpy.data.node_groups`, `tree.nodes.new()`, `tree.links.new()`, `tree.force_update()`, `tree.update_nodes()`, `tree.init_tree()`, `tree.sv_process`, `node.outputs[].sv_get()`, `updateNode`, `match_long_repeat`, `fullList`, `repeat_last`, `changable_sockets`, `replace_socket`, `multi_socket`, `enum_item_4`, `flat_iter`, `fixed_iter`, `get_edge_loop`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok/blob/master/node_tree.py
  - https://github.com/nortikin/sverchok/blob/master/data_structure.py
  - https://github.com/nortikin/sverchok/blob/master/utils/sv_itertools.py
- **Research input**: vooronderzoek-sverchok §9 (L1469-1598)
- **Dependencies**: sverchok-syntax-sockets, sverchok-syntax-data
- **Complexity**: M

### 2.3 Implementation Skills (4)

#### SV-I-01: sverchok-impl-custom-nodes
- **Category**: impl
- **Scope**: Complete custom node development guide — minimal node template, node docstring format (Triggers/Tooltip), socket creation methods (direct in sv_init, sv_new_input helper), node lifecycle methods (sv_init, process, sv_update, sv_copy, sv_free, sv_draw_buttons, sv_draw_buttons_ext), node properties with updateNode callback, animation/scene dependency flags, node dependency checking (sv_dependencies), registration pattern (register_classes_factory, manual register/unregister), standard process method pattern (output check → read inputs → match lengths → process objects → set outputs), BMesh integration in nodes (bmesh_from_pydata, pydata_from_bmesh), NumPy vectorized operations for performance, error handling in nodes (SvNoDataError), logging (debug/info/warning/error/exception).
- **Key API surface**: `SverchCustomTreeNode`, `bpy.types.Node`, `sv_init()`, `process()`, `sv_update()`, `sv_copy()`, `sv_free()`, `sv_draw_buttons()`, `sv_draw_buttons_ext()`, `updateNode`, `match_long_repeat`, `SvNoDataError`, `bmesh_from_pydata()`, `pydata_from_bmesh()`, `sv_dependencies`, `is_animation_dependent`, `is_scene_dependent`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok/blob/master/node_tree.py
  - https://github.com/nortikin/sverchok/blob/master/data_structure.py
  - https://github.com/nortikin/sverchok/blob/master/utils/sv_bmesh_utils.py
  - https://github.com/nortikin/sverchok/blob/master/core/sv_custom_exceptions.py
- **Research input**: vooronderzoek-sverchok §8 (L1202-1465)
- **Dependencies**: sverchok-syntax-sockets, sverchok-syntax-data, sverchok-core-concepts
- **Complexity**: L

#### SV-I-02: sverchok-impl-parametric
- **Category**: impl
- **Scope**: Parametric design workflows using Sverchok — parametric building elements (Profile → Extrude → IFC), batch element generation with list processing, working with matrices (Matrix construction, rotation, scale, combined transforms, applying to vertices), Blender object integration (reading mesh data from objects, writing generated geometry back to objects), TopologicSverchok for building analysis (CellComplex, space adjacency, dual graph), vectorize utility in SNLite, performance optimization with setup() pre-computation, NumPy acceleration patterns, node categories overview for workflow selection.
- **Key API surface**: `Matrix.Translation()`, `Euler.to_matrix()`, `Matrix.Diagonal()`, `mathutils.Vector`, `bmesh_from_pydata()`, `vectorize()`, `setup()`, `get_user_dict()`, `bpy.data.meshes`, `bpy.data.objects`, `bpy.context.collection.objects.link()`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
  - https://nortikin.github.io/sverchok/
  - https://github.com/wassimj/TopologicSverchok
- **Research input**: vooronderzoek-sverchok §14 (L2145-2287), §6 (L720-778)
- **Dependencies**: sverchok-syntax-scripting, sverchok-syntax-data, sverchok-syntax-sockets
- **Complexity**: M

#### SV-I-03: sverchok-impl-ifcsverchok
- **Category**: impl
- **Scope**: IfcSverchok BIM integration — overview and dependency chain (Bonsai + Sverchok + ifcopenshell), SvIfcStore transient IFC file management (purge, get_file, id_map, use_bonsai_file), SvIfcCore node base class, complete node catalog (31 nodes: 24 IFC nodes + 7 Shape Builder nodes), two geometry modes (from Blender objects via SvIfcBMeshToIfcRepr, from Sverchok geometry via SvIfcSverchokToIfcRepr), IFC file generation workflow (6-step: generate geometry → convert to representation → create entities → build hierarchy → add properties → export), automatic hierarchy completion (ensure_hirarchy), working example (simple IFC wall), integration with Bonsai (use_bonsai_file toggle), known issues and limitations (undo crashes, single transient file, no type/material library, limited geometry types, no validation).
- **Key API surface**: `SvIfcStore`, `SvIfcCore`, `SvIfcCreateEntity`, `SvIfcWriteFile`, `SvIfcReadFile`, `SvIfcCreateFile`, `SvIfcAddPset`, `SvIfcAddSpatialElement`, `SvIfcBMeshToIfcRepr`, `SvIfcSverchokToIfcRepr`, `SvIfcSbRectangle`, `SvIfcSbExtrude`, `SvIfcSbRepresentation`, `SvIfcByType`, `SvIfcByQuery`, `SvIfcApi`, `SvIfcQuickProjectSetup`
- **SOURCES.md URLs**:
  - https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
  - https://community.osarch.org/discussion/284/sverchok-ifc
  - https://mdjska.github.io/GSoC/notes/GSoC_proposal_mdj/
- **Research input**: vooronderzoek-sverchok §10 (L1614-1800)
- **Dependencies**: sverchok-impl-custom-nodes, sverchok-syntax-sockets, sverchok-syntax-data
- **Complexity**: L

#### SV-I-04: sverchok-impl-extensions
- **Category**: impl
- **Scope**: Sverchok extensions ecosystem — TopologicSverchok (non-manifold topology: Vertex/Edge/Wire/Face/Shell/Cell/CellComplex/Cluster hierarchy, 200+ nodes, AEC relevance: envelope analysis, space adjacency, dual graph, energy simulation, IFC integration), Sverchok-Extra (advanced geometry: Surface Extra, Field Extra, Solid Extra, Spatial Extra, SDF Primitives, SDF Operations), other extensions overview (Sverchok-Open3d, Mega-Polis, Ladybug Tools, Sverchok-Bmesh), extension development pattern (bl_info, nodes_index, register/unregister, directory structure), node registration requirements (7 mandatory elements).
- **Key API surface**: `TopologicSverchok` classes (Vertex, Edge, Wire, Face, Shell, Cell, CellComplex, Cluster), `nodes_index()`, `add_node_menu.register()`, `SverchCustomTreeNode` inheritance, `bpy.types.Node` inheritance, extension directory structure
- **SOURCES.md URLs**:
  - https://github.com/wassimj/TopologicSverchok
  - https://topologic.app/software/
  - https://github.com/portnov/sverchok-extra
  - https://github.com/vicdoval/sverchok-open3d
  - https://github.com/victorcalixto/mega-polis
  - https://nortikin.github.io/sverchok/docs/introduction/sverchok_extensions.html
- **Research input**: vooronderzoek-sverchok §11 (L1804-1936)
- **Dependencies**: sverchok-impl-custom-nodes
- **Complexity**: M

### 2.4 Error Skills (1)

#### SV-E-01: sverchok-errors-common
- **Category**: errors
- **Scope**: Common error patterns — nesting level errors (wrong level causes silent data corruption), missing updateNode callback (property changes don't trigger re-evaluation), not checking output connections (unnecessary computation), socket data mutation (modifying input in-place affects upstream), list matching misunderstandings (using zip instead of match_long_repeat), IfcSverchok undo crashes, IfcSverchok purge on re-run (entity IDs change between runs). AI common mistakes — outputting flat data, missing object wrapper for vertices, using __init__ instead of sv_init, wrong SNLite socket type aliases, forgetting updateNode, not matching lists before zipping, creating sockets in process(), assuming matrix data is nested, IfcSverchok entity ID persistence, IfcSverchok single vs double nesting.
- **Key patterns**: 7 common error patterns (§12) + 10 AI-specific anti-patterns (§13) = 17 documented issues
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
  - https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
- **Research input**: vooronderzoek-sverchok §12 (L1939-2026), §13 (L2028-2141)
- **Dependencies**: sverchok-impl-custom-nodes, sverchok-syntax-scripting, sverchok-syntax-data
- **Complexity**: M

### 2.5 Agent Skills (1)

#### SV-A-01: sverchok-agents-code-validator
- **Category**: agents
- **Scope**: Validation agent for Sverchok code — checklist-based validator covering: data nesting correctness (vertices=level 3, strings=level 2, matrices=level 1), updateNode callback presence on all properties, output connection check (early exit pattern), list matching before zip, socket creation only in sv_init/sv_update, deepcopy awareness for input data, correct SNLite socket type aliases, proper node docstring format (Triggers/Tooltip), process method standard pattern compliance, IfcSverchok double-nesting compliance, BMesh cleanup (bm.free()), NumPy vs Python loop optimization opportunities.
- **Validation checklist items**: ~15 automated checks derived from §12 error patterns + §13 AI mistakes
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
- **Research input**: vooronderzoek-sverchok §12 (L1939-2026), §13 (L2028-2141), §8 (L1369-1465)
- **Dependencies**: sverchok-errors-common, all impl skills
- **Complexity**: M

---

## 3. Batch Execution Plan

### Batch 1: Foundation (1 skill)
**Skills**: sverchok-core-concepts
**Rationale**: All other skills depend on understanding Sverchok architecture, node tree system, and update mechanism.
**Agents**: 1

### Batch 2: Core Syntax (2 skills)
**Skills**: sverchok-syntax-sockets, sverchok-syntax-data
**Rationale**: Socket types and data nesting are the two fundamental concepts that all implementation skills require. Data nesting is the #1 error source and must be thoroughly documented early.
**Agents**: 2 (parallel)

### Batch 3: Advanced Syntax + Custom Nodes (3 skills)
**Skills**: sverchok-syntax-scripting, sverchok-syntax-api, sverchok-impl-custom-nodes
**Rationale**: Scripting and API skills depend on socket/data knowledge (batch 2). Custom node development also depends on batch 2 and can run in parallel with syntax skills since it only requires core + sockets + data.
**Agents**: 3 (parallel)

### Batch 4: Implementation (3 skills)
**Skills**: sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-extensions
**Rationale**: Parametric workflows depend on scripting (batch 3). IfcSverchok depends on custom node patterns (batch 3). Extensions depend on custom node development (batch 3).
**Agents**: 3 (parallel)

### Batch 5: Errors (1 skill)
**Skills**: sverchok-errors-common
**Rationale**: Error documentation must reference all implementation patterns to provide correct/incorrect code examples in context.
**Agents**: 1

### Batch 6: Agents (1 skill)
**Skills**: sverchok-agents-code-validator
**Rationale**: The validation agent must reference all error patterns and all implementation skills to build a comprehensive checklist.
**Agents**: 1

### Batch Summary

| Batch | Skills | Count | Depends On | Agents |
|-------|--------|-------|------------|--------|
| 1 | sverchok-core-concepts | 1 | — | 1 |
| 2 | sverchok-syntax-sockets, sverchok-syntax-data | 2 | Batch 1 | 2 |
| 3 | sverchok-syntax-scripting, sverchok-syntax-api, sverchok-impl-custom-nodes | 3 | Batch 2 | 3 |
| 4 | sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-extensions | 3 | Batch 3 | 3 |
| 5 | sverchok-errors-common | 1 | Batch 4 | 1 |
| 6 | sverchok-agents-code-validator | 1 | Batch 5 | 1 |
| **Total** | | **11** | | **Max 3** |

### Dependency Graph

```
sverchok-core-concepts (Batch 1)
├── sverchok-syntax-sockets (Batch 2)
│   ├── sverchok-syntax-scripting (Batch 3)
│   │   └── sverchok-impl-parametric (Batch 4)
│   ├── sverchok-syntax-api (Batch 3)
│   ├── sverchok-impl-custom-nodes (Batch 3)
│   │   ├── sverchok-impl-ifcsverchok (Batch 4)
│   │   ├── sverchok-impl-extensions (Batch 4)
│   │   └── sverchok-errors-common (Batch 5)
│   │       └── sverchok-agents-code-validator (Batch 6)
│   └── ...
└── sverchok-syntax-data (Batch 2)
    ├── sverchok-syntax-scripting (Batch 3)
    ├── sverchok-syntax-api (Batch 3)
    ├── sverchok-impl-custom-nodes (Batch 3)
    └── sverchok-errors-common (Batch 5)
```

---

## 4. Skill-to-Research Mapping

| Skill | Vooronderzoek Sections | Lines | Est. SKILL.md Lines |
|-------|----------------------|-------|-------------------|
| sverchok-core-concepts | §1, §2, §6 | L30-288, L720-778 | ~350 |
| sverchok-syntax-sockets | §3, §15 | L290-431, L2291-2333 | ~300 |
| sverchok-syntax-data | §4, §5 | L435-716 | ~400 |
| sverchok-syntax-scripting | §7 | L782-1199 | ~450 |
| sverchok-syntax-api | §9 | L1469-1598 | ~250 |
| sverchok-impl-custom-nodes | §8 | L1202-1465 | ~400 |
| sverchok-impl-parametric | §14, §6 | L2145-2287, L720-778 | ~300 |
| sverchok-impl-ifcsverchok | §10 | L1614-1800 | ~400 |
| sverchok-impl-extensions | §11 | L1804-1936 | ~300 |
| sverchok-errors-common | §12, §13 | L1939-2141 | ~350 |
| sverchok-agents-code-validator | §12, §13, §8 | L1939-2141, L1369-1465 | ~250 |

All skills estimated under the 500-line SKILL.md limit (D-009).

---

## 5. Cross-Package Dependencies

| Sverchok Skill | External Dependency | Required For |
|---------------|---------------------|--------------|
| sverchok-impl-ifcsverchok | ifcos-core-concepts | Understanding IFC file structure |
| sverchok-impl-ifcsverchok | ifcos-syntax-elements | IfcWall, IfcSlab, IfcColumn entity types |
| sverchok-impl-ifcsverchok | ifcos-impl-creation | ifcopenshell.api patterns used by SvIfcApi node |
| sverchok-impl-parametric | blender-core-api | bpy.data, bpy.context for Blender object integration |
| sverchok-impl-parametric | blender-core-runtime | mathutils (Vector, Matrix, Euler) for transforms |
| sverchok-impl-custom-nodes | blender-syntax-addons | Registration pattern (register_class, bl_info) |
| sverchok-syntax-api | blender-core-api | bpy.data.node_groups access pattern |

**Recommendation**: Sverchok skills should be built AFTER the core Blender and IfcOpenShell packages are complete, as specified in the main masterplan's "DEFERRED" section.

---

*Document generated by sv-masterplan-raw agent. Date: 2026-03-07.*
