# DEFINITIVE Masterplan — Sverchok Claude Skill Package

**Date**: 2026-03-07
**Phase**: S2 — Sverchok Masterplan (Definitive)
**Status**: DEFINITIVE — Approved for skill creation
**Author**: sv-masterplan-def agent
**Input**: sverchok-masterplan-raw.md, sverchok-review-technical.md, sverchok-review-practical.md

---

## 1. Executive Summary

### Skill Counts

| Category | Count | Skills |
|----------|-------|--------|
| core | 1 | sverchok-core-concepts |
| syntax | 4 | sverchok-syntax-sockets, sverchok-syntax-data, sverchok-syntax-scripting, sverchok-syntax-api |
| impl | 5 | sverchok-impl-custom-nodes, sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-topologic, sverchok-impl-extensions |
| errors | 1 | sverchok-errors-common |
| agents | 1 | sverchok-agents-code-validator |
| **Total** | **12** | |

### Build Timeline

- **7 batches**, 1–3 skills per batch
- **Batch 1**: Foundation (core-concepts, no deps)
- **Batch 2**: Core syntax (sockets, data — depend on core)
- **Batch 3**: Advanced syntax + custom nodes (scripting, api, custom-nodes — depend on batch 2)
- **Batch 4**: Implementation domain (parametric, ifcsverchok, topologic — depend on batch 3)
- **Batch 5**: Extensions (extensions — depends on custom-nodes)
- **Batch 6**: Errors (depends on impl knowledge including ifcsverchok)
- **Batch 7**: Agents (depends on everything)
- Estimated duration: 7 quality-gated batch cycles

### Scope Boundaries

**IN scope**:
- Sverchok core architecture: node tree system, update system, socket cache, events
- Socket type system: 16 socket types, implicit conversions, processing flags
- Data nesting levels and list matching (the #1 source of errors)
- Python scripting: SNLite, SN Functor B, Formula Mk5, Profile Mk3
- Custom node development: full lifecycle, registration, BMesh integration
- External API: programmatic tree control, batch processing
- AEC parametric design workflows: structural grids, facade panels, stairs, roof geometry, MEP routing, terrain from data
- IfcSverchok: 31 IFC nodes, SvIfcStore, geometry modes, BIM workflow
- TopologicSverchok: building topology, CellComplex, space adjacency, dual graph, energy simulation
- Extensions: Sverchok-Extra, Open3d, extension development pattern
- Error patterns and AI common mistakes (17 documented anti-patterns)
- Advanced data types: curves, surfaces, fields, solids

**OUT of scope**:
- Individual node documentation for all 500+ built-in nodes (reference only)
- Pulga Physics simulation details
- Blender-internal node systems (Geometry Nodes, Shader Nodes)
- Sculpting, video editing, and non-AEC Blender workflows

**Dependencies on other packages**:
- `sverchok-impl-ifcsverchok` references IfcOpenShell API (covered by ifcos-* skills)
- `sverchok-impl-topologic` references TopologicSverchok external dependency
- `sverchok-syntax-api` builds on Blender bpy.data knowledge (covered by blender-core-api)

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| TopologicSverchok split into dedicated skill | TopologicSverchok has 200+ nodes and is the primary AEC-relevant extension — it deserves dedicated coverage separate from generic extension development (review P-01) |
| Extensions skill focused on ecosystem + dev pattern | Sverchok-Extra, Open3d, and extension development share the same registration architecture; TopologicSverchok serves a distinct AEC audience |
| Data nesting + list matching = one skill | §4 (nesting) and §5 (matching/vectorization) are inseparable — matching only makes sense with correct nesting understanding |
| External API as syntax skill | Programmatic tree access (§9) is API syntax/patterns, not a workflow guide |
| Node categories overview in core-concepts | §6 (500+ node categories) is reference material, not deep API documentation; fits as part of architecture overview |
| Advanced data types in syntax-sockets | Curves, surfaces, fields, solids (§15) are socket data types; belong with socket type documentation |
| Single errors skill | 7 error patterns + 10 AI mistakes = 17 items; fits comfortably in one skill < 500 lines |
| Parametric skill framed around AEC domain | Users think "facade", "column grid", "staircase" — not "matrix operations"; scope reframed around recognizable AEC deliverables (review P-02, P-03) |

---

## 2. Skill Inventory

### 2.1 Core Skills (1)

#### SV-C-01: sverchok-core-concepts
- **Category**: core
- **Description**: Explains Sverchok parametric node system architecture including node tree execution, data flow between nodes, update triggers, and the socket data cache. Use this when understanding how Sverchok processes node graphs, debugging update issues, learning Sverchok fundamentals, or getting started with Sverchok for the first time.
- **Scope**: Sverchok architecture overview, SverchCustomTree class, SverchCustomTreeNode base class and mixins (UpdateNodes, NodeUtils, NodeDependencies, NodeDocumentation), socket data cache system (including sv_deep_copy), update system (control_center, SearchTree, UpdateTree), event classes (TreeEvent, PropertyEvent, AnimationEvent, etc.), node execution order (topological sort), data propagation between nodes, execution statistics, node categories overview (500+ nodes in 18+ categories), getting started workflow (creating a node tree, adding basic nodes, connecting and viewing output).
- **Key API surface**: `SverchCustomTree`, `SverchCustomTreeNode`, `socket_data_cache`, `sv_deep_copy`, `SearchTree`, `UpdateTree`, `control_center()`, `prepare_input_data()`, event classes from `core/events.py`
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
- **Description**: Covers all 16 Sverchok socket types including choosing the right socket for geometry, numbers, matrices, curves, surfaces, and fields. Explains socket properties, implicit type conversions, and data processing flags. Use this when connecting nodes, understanding type conversions, debugging socket compatibility errors, or choosing the right socket type for your data.
- **Scope**: All 16 socket types (SvStringsSocket, SvVerticesSocket, SvMatrixSocket, SvQuaternionSocket, SvColorSocket, SvCurveSocket, SvSurfaceSocket, SvSolidSocket, SvScalarFieldSocket, SvVectorFieldSocket, SvDictionarySocket, SvObjectSocket, SvFilePathSocket, SvTextSocket, SvDummySocket, SvChameleonSocket), socket base class API (SvSocketCommon), sv_get/sv_set/sv_forget methods, socket properties (color, label, use_prop, prop_name, nesting_level, default_mode, pre_processing), implicit conversion policies (Default, Field, Lenient, Solid), conversion policy registry, lenient socket types, socket data processing flag definitions (flatten, simplify, graft, unwrap, wrap — what they are, where they live on socket objects), advanced data types (curves/SvCurve, surfaces/SvSurface, scalar fields/SvScalarField, vector fields/SvVectorField, solids/Part.Shape, dictionaries).
- **Note**: Socket processing flag *behavior* (how they transform data in the preprocessing/postprocessing pipeline) is documented in sverchok-syntax-data. This skill covers flag *definitions* on socket objects.
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
- **Description**: Explains Sverchok's critical data nesting system and list matching — the #1 source of errors. Covers nesting levels for vertices (level 3), edges/faces (level 2), matrices (level 1), plus the 5 list matching modes. Use this when data looks wrong, vertices are flattened, list lengths don't match, or you're debugging data structure errors.
- **Scope**: Data nesting level convention (levels 0-3), standard data formats per socket type (SvStringsSocket=level 2, SvVerticesSocket=level 3, SvMatrixSocket=level 1, edge data=level 2 via SvStringsSocket, face data=level 2 via SvStringsSocket), nesting level detection (get_data_nesting_level), the "objects" mental model, list matching system (5 modes: REPEAT, CYCLE, SHORT, XREF, XREF2), core matching functions (match_long_repeat, fullList, repeat_last, list_match_func), NumPy matching variants, the vectorize decorator, SvRecursiveNode mixin, socket processing mode behavior (input preprocessing pipeline: flatten/simplify/graft/unwrap/wrap, output postprocessing — how these flags transform data at runtime), match_sockets helper, recursive processing utilities (recurse_fx, recurse_fxy, recurse_f_level_control, process_matched).
- **Note**: Socket processing flag *definitions* (what they are, where they live on socket objects) are documented in sverchok-syntax-sockets. This skill covers flag *behavior* in the data pipeline.
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
- **Description**: Covers Sverchok's Python scripting nodes — SNLite, SN Functor B, Formula Mk5, and Profile Mk3. Explains socket declaration syntax, type identifiers, built-in aliases, special functions, and template system. Use this when writing SNLite scripts, creating formulas, defining parametric profiles, or embedding custom Python logic in Sverchok node trees.
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
- **Note**: If skill exceeds 450 lines during authoring, split into `sverchok-syntax-snlite` (SNLite + SN Functor B) and `sverchok-syntax-formula-profile` (Formula Mk5 + Profile Mk3 + other script nodes). Natural split point at research L1063.

#### SV-S-04: sverchok-syntax-api
- **Category**: syntax
- **Description**: Guides programmatic control of Sverchok node trees from Python including creating nodes, connecting sockets, setting parameters, triggering updates, and batch processing with parameter sweeps. Use this when automating Sverchok, generating trees from scripts, running batch parametric studies, or building Sverchok workflows programmatically.
- **Scope**: External API access to Sverchok node trees, accessing trees via bpy.data.node_groups (filtering by bl_idname='SverchCustomTreeType'), creating nodes programmatically, creating links, modifying node properties, triggering tree updates (force_update, update_nodes, sv_process), reading node output data via sv_get(), batch processing with parameter sweeps, init_tree context manager for suppressing intermediate updates, key internal API modules (sverchok.data_structure complete reference, sverchok.utils.sv_itertools).
- **Key API surface**: `bpy.data.node_groups`, `tree.nodes.new()`, `tree.links.new()`, `tree.force_update()`, `tree.update_nodes()`, `tree.init_tree()`, `tree.sv_process`, `node.outputs[].sv_get()`, `updateNode`, `match_long_repeat`, `fullList`, `repeat_last`, `changable_sockets`, `replace_socket`, `multi_socket`, `enum_item_4`, `flat_iter`, `fixed_iter`, `get_edge_loop`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok/blob/master/node_tree.py
  - https://github.com/nortikin/sverchok/blob/master/data_structure.py
  - https://github.com/nortikin/sverchok/blob/master/utils/sv_itertools.py
- **Research input**: vooronderzoek-sverchok §9 (L1469-1598)
- **Dependencies**: sverchok-syntax-sockets, sverchok-syntax-data
- **Complexity**: M

### 2.3 Implementation Skills (5)

#### SV-I-01: sverchok-impl-custom-nodes
- **Category**: impl
- **Description**: Complete guide to developing custom Sverchok nodes including the full node lifecycle, socket creation, property management, BMesh integration, and registration patterns. Use this when writing a custom Sverchok node, creating a new node type, or packaging nodes as an add-on.
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
- **Description**: Guides AEC parametric design workflows in Sverchok including structural grids, facade panel systems, parametric stairs, roof geometry generation, MEP routing layouts, and terrain generation from data. Covers matrix-based transforms, Blender object integration, and data-driven geometry generation. Use this when creating parametric building elements, generating column grids, designing curtain walls, creating floor plate arrays, building parametric staircases, generating roof geometry, routing MEP systems, or creating terrain from contour/point data.
- **Scope**: AEC domain parametric workflows —
  - **Structural grids**: Column grid generation, beam layouts, floor plate arrays, repeating structural elements
  - **Facade systems**: Panel division, window patterns, curtain wall mullion grids, parametric openings, louver arrays
  - **Stairs**: Parametric staircase generation, tread/riser profiles, railing generation, spiral stairs
  - **Roof geometry**: Hip/gable/shed roof generation from building footprint, dormer placement, ridge/valley calculation
  - **MEP routing**: Pipe routing, duct run generation, cable tray layouts, fitting placement using Sverchok path/spline nodes
  - **Terrain from data**: Terrain mesh from contour data, point cloud to surface, site grading, topographic generation
  - **Supporting techniques**: Working with matrices (Matrix construction, rotation, scale, combined transforms, applying to vertices), Blender object integration (reading mesh data from objects, writing generated geometry back to objects), batch element generation with list processing, data-driven generation (CSV/JSON parameters to geometry), NumPy acceleration patterns, node categories overview for workflow selection.
- **Key API surface**: `Matrix.Translation()`, `Euler.to_matrix()`, `Matrix.Diagonal()`, `mathutils.Vector`, `bmesh_from_pydata()`, `bpy.data.meshes`, `bpy.data.objects`, `bpy.context.collection.objects.link()`
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
  - https://nortikin.github.io/sverchok/
- **Research input**: vooronderzoek-sverchok §14 (L2145-2287), §6 (L720-778)
- **Dependencies**: sverchok-syntax-scripting, sverchok-syntax-data, sverchok-syntax-sockets
- **Complexity**: M
- **Note**: For SNLite-based techniques (vectorize, setup()), reference sverchok-syntax-scripting. For TopologicSverchok building analysis, reference sverchok-impl-topologic.

#### SV-I-03: sverchok-impl-ifcsverchok
- **Category**: impl
- **Description**: Covers IfcSverchok BIM integration — generating IFC files from Sverchok geometry using 31 specialized nodes. Explains SvIfcStore transient file management, the two geometry conversion modes, the 6-step IFC generation workflow, and Bonsai integration. Use this when connecting Sverchok output to IFC, generating BIM models from parametric geometry, or creating IFC files programmatically through Sverchok node trees.
- **Scope**: IfcSverchok BIM integration — overview and dependency chain (Bonsai + Sverchok + ifcopenshell), SvIfcStore transient IFC file management (purge, get_file, id_map, use_bonsai_file), SvIfcCore node base class, SvIfcCore double-nested input processing pattern (zip_long_repeat applied twice — differs from standard Sverchok node processing, explains why IfcSverchok nodes expect inputs formatted differently), complete node catalog (31 nodes: 24 IFC nodes + 7 Shape Builder nodes), two geometry modes (from Blender objects via SvIfcBMeshToIfcRepr, from Sverchok geometry via SvIfcSverchokToIfcRepr), IFC file generation workflow (6-step: generate geometry → convert to representation → create entities → build hierarchy → add properties → export), automatic hierarchy completion (ensure_hirarchy), working example (simple IFC wall), integration with Bonsai (use_bonsai_file toggle), known issues and limitations (undo crashes, single transient file, no type/material library, limited geometry types, no validation).
- **Key API surface**: `SvIfcStore`, `SvIfcCore`, `SvIfcCreateEntity`, `SvIfcWriteFile`, `SvIfcReadFile`, `SvIfcCreateFile`, `SvIfcAddPset`, `SvIfcAddSpatialElement`, `SvIfcBMeshToIfcRepr`, `SvIfcSverchokToIfcRepr`, `SvIfcSbRectangle`, `SvIfcSbExtrude`, `SvIfcSbRepresentation`, `SvIfcByType`, `SvIfcByQuery`, `SvIfcApi`, `SvIfcQuickProjectSetup`
- **SOURCES.md URLs**:
  - https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
  - https://community.osarch.org/discussion/284/sverchok-ifc
  - https://mdjska.github.io/GSoC/notes/GSoC_proposal_mdj/
- **Research input**: vooronderzoek-sverchok §10 (L1614-1800)
- **Dependencies**: sverchok-impl-custom-nodes, sverchok-syntax-sockets, sverchok-syntax-data
- **Complexity**: L

#### SV-I-04: sverchok-impl-topologic
- **Category**: impl
- **Description**: Covers TopologicSverchok for building topology analysis — non-manifold topology operations, CellComplex workflows, space adjacency graphs, dual graphs, and energy simulation integration. Use this when analyzing building topology, computing space adjacency, creating CellComplex models, generating dual graphs from building geometry, performing envelope analysis, or integrating Topologic with IFC/energy workflows.
- **Scope**: TopologicSverchok practical usage — non-manifold topology class hierarchy (Vertex/Edge/Wire/Face/Shell/Cell/CellComplex/Cluster), 200+ nodes organized by topology class, AEC-relevant workflows (envelope analysis, space adjacency computation, dual graph generation for circulation analysis, energy simulation input preparation, IFC integration), CellComplex construction from building geometry, querying topological relationships (adjacency, containment, connectivity), practical examples (room adjacency from BIM model, building envelope analysis, space connectivity graph).
- **Key API surface**: TopologicSverchok classes (`Vertex`, `Edge`, `Wire`, `Face`, `Shell`, `Cell`, `CellComplex`, `Cluster`), topology query methods, adjacency/connectivity operations
- **SOURCES.md URLs**:
  - https://github.com/wassimj/TopologicSverchok
  - https://topologic.app/software/
- **Research input**: vooronderzoek-sverchok §11 (L1804-1936, TopologicSverchok sections)
- **Dependencies**: sverchok-impl-custom-nodes
- **Complexity**: M

#### SV-I-05: sverchok-impl-extensions
- **Category**: impl
- **Description**: Covers the Sverchok extension ecosystem and extension development pattern. Includes Sverchok-Extra for advanced geometry (surfaces, fields, solids, SDF), Open3d integration, and a guide to developing and registering custom extensions. Use this when working with Sverchok-Extra advanced geometry, using Open3d point cloud operations, writing a Sverchok extension, registering custom nodes as an extension, or packaging nodes for distribution.
- **Scope**: Sverchok extensions ecosystem — Sverchok-Extra (advanced geometry: Surface Extra, Field Extra, Solid Extra, Spatial Extra, SDF Primitives, SDF Operations), other extensions overview (Sverchok-Open3d, Mega-Polis, Ladybug Tools, Sverchok-Bmesh), extension development pattern (bl_info, nodes_index, register/unregister, directory structure), node registration requirements (7 mandatory elements).
- **Note**: For TopologicSverchok building topology analysis, see sverchok-impl-topologic.
- **Key API surface**: `nodes_index()`, `add_node_menu.register()`, `SverchCustomTreeNode` inheritance, `bpy.types.Node` inheritance, extension directory structure
- **SOURCES.md URLs**:
  - https://github.com/portnov/sverchok-extra
  - https://github.com/vicdoval/sverchok-open3d
  - https://github.com/victorcalixto/mega-polis
  - https://nortikin.github.io/sverchok/docs/introduction/sverchok_extensions.html
- **Research input**: vooronderzoek-sverchok §11 (L1804-1936, non-Topologic sections)
- **Dependencies**: sverchok-impl-custom-nodes
- **Complexity**: M

### 2.4 Error Skills (1)

#### SV-E-01: sverchok-errors-common
- **Category**: errors
- **Description**: Documents the 17 most common Sverchok error patterns and AI mistakes — covering data nesting errors, missing updateNode callbacks, socket data mutation, list matching misunderstandings, and IfcSverchok-specific pitfalls. Use this when debugging Sverchok node behavior, diagnosing silent data corruption, or understanding why generated Sverchok code fails.
- **Scope**: Common error patterns — nesting level errors (wrong level causes silent data corruption), missing updateNode callback (property changes don't trigger re-evaluation), not checking output connections (unnecessary computation), socket data mutation (modifying input in-place affects upstream), list matching misunderstandings (using zip instead of match_long_repeat), IfcSverchok undo crashes, IfcSverchok purge on re-run (entity IDs change between runs). AI common mistakes — outputting flat data, missing object wrapper for vertices, using __init__ instead of sv_init, wrong SNLite socket type aliases, forgetting updateNode, not matching lists before zipping, creating sockets in process(), assuming matrix data is nested, IfcSverchok entity ID persistence, IfcSverchok single vs double nesting.
- **Key patterns**: 7 common error patterns (§12) + 10 AI-specific anti-patterns (§13) = 17 documented issues
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
  - https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
- **Research input**: vooronderzoek-sverchok §12 (L1939-2026), §13 (L2028-2141)
- **Dependencies**: sverchok-impl-custom-nodes, sverchok-syntax-scripting, sverchok-syntax-data, sverchok-impl-ifcsverchok
- **Complexity**: M

### 2.5 Agent Skills (1)

#### SV-A-01: sverchok-agents-code-validator
- **Category**: agents
- **Description**: Validation agent for Sverchok code — runs ~19 automated checks on generated Sverchok node code covering data nesting correctness, updateNode callbacks, list matching patterns, socket consistency, import validation, and IfcSverchok compliance. Use this agent to validate any generated Sverchok Python code before presenting it to the user.
- **Scope**: Validation agent for Sverchok code — checklist-based validator covering:
  1. Data nesting correctness (vertices=level 3, edges/faces=level 2, strings=level 2, matrices=level 1)
  2. updateNode callback presence on all properties
  3. Output connection check (early exit pattern)
  4. List matching before zip (match_long_repeat usage)
  5. Socket creation only in sv_init/sv_update
  6. deepcopy awareness for input data
  7. Correct SNLite socket type aliases
  8. Proper node docstring format (Triggers/Tooltip)
  9. Process method standard pattern compliance
  10. IfcSverchok double-nesting compliance
  11. BMesh cleanup (bm.free())
  12. NumPy vs Python loop optimization opportunities
  13. Import validation (verify all sverchok module imports resolve to real modules)
  14. Socket name consistency (cross-check socket names between sv_init and process)
  15. match_long_repeat return unpacking pattern (correct star unpacking or index access)
  16. sv_get default pattern (sv_get(default=...) for optional inputs to prevent SvNoDataError)
  17. Registration completeness (bl_idname, bl_label, sv_category, bl_icon)
  18. Property type validation (correct property type for data)
  19. Edge/face data format validation (level 2 via SvStringsSocket)
- **Validation checklist items**: ~19 automated checks derived from §12 error patterns + §13 AI mistakes + review feedback
- **SOURCES.md URLs**:
  - https://github.com/nortikin/sverchok
- **Research input**: vooronderzoek-sverchok §12 (L1939-2026), §13 (L2028-2141), §8 (L1369-1465)
- **Dependencies**: sverchok-errors-common, sverchok-impl-custom-nodes, sverchok-impl-ifcsverchok, sverchok-impl-parametric, sverchok-impl-topologic, sverchok-impl-extensions, sverchok-syntax-scripting
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

### Batch 4: Implementation Domain (3 skills)
**Skills**: sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-topologic
**Rationale**: Parametric workflows depend on scripting (batch 3). IfcSverchok depends on custom node patterns (batch 3). TopologicSverchok depends on custom node development (batch 3). All three serve distinct AEC domain audiences and can run in parallel.
**Agents**: 3 (parallel)

### Batch 5: Extensions (1 skill)
**Skills**: sverchok-impl-extensions
**Rationale**: Extensions skill depends on custom node development (batch 3) and benefits from TopologicSverchok being documented separately (batch 4) to maintain clear scope boundaries. Separated from batch 4 to avoid overloading the impl batch.
**Agents**: 1

### Batch 6: Errors (1 skill)
**Skills**: sverchok-errors-common
**Rationale**: Error documentation must reference all implementation patterns including IfcSverchok (batch 4) to provide correct/incorrect code examples in context.
**Agents**: 1

### Batch 7: Agents (1 skill)
**Skills**: sverchok-agents-code-validator
**Rationale**: The validation agent must reference all error patterns and all implementation skills to build a comprehensive checklist.
**Agents**: 1

### Batch Summary

| Batch | Skills | Count | Depends On | Agents |
|-------|--------|-------|------------|--------|
| 1 | sverchok-core-concepts | 1 | — | 1 |
| 2 | sverchok-syntax-sockets, sverchok-syntax-data | 2 | Batch 1 | 2 |
| 3 | sverchok-syntax-scripting, sverchok-syntax-api, sverchok-impl-custom-nodes | 3 | Batch 2 | 3 |
| 4 | sverchok-impl-parametric, sverchok-impl-ifcsverchok, sverchok-impl-topologic | 3 | Batch 3 | 3 |
| 5 | sverchok-impl-extensions | 1 | Batch 3 | 1 |
| 6 | sverchok-errors-common | 1 | Batch 4 | 1 |
| 7 | sverchok-agents-code-validator | 1 | Batch 6 | 1 |
| **Total** | | **12** | | **Max 3** |

### Dependency Graph

```
sverchok-core-concepts (Batch 1)
├── sverchok-syntax-sockets (Batch 2)
│   ├── sverchok-syntax-scripting (Batch 3)
│   │   └── sverchok-impl-parametric (Batch 4)
│   ├── sverchok-syntax-api (Batch 3)
│   ├── sverchok-impl-custom-nodes (Batch 3)
│   │   ├── sverchok-impl-ifcsverchok (Batch 4)
│   │   ├── sverchok-impl-topologic (Batch 4)
│   │   ├── sverchok-impl-extensions (Batch 5)
│   │   └── sverchok-errors-common (Batch 6)
│   │       └── sverchok-agents-code-validator (Batch 7)
│   └── ...
└── sverchok-syntax-data (Batch 2)
    ├── sverchok-syntax-scripting (Batch 3)
    ├── sverchok-syntax-api (Batch 3)
    ├── sverchok-impl-custom-nodes (Batch 3)
    └── sverchok-errors-common (Batch 6)
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
| sverchok-impl-parametric | §14, §6 | L2145-2287, L720-778 | ~350 |
| sverchok-impl-ifcsverchok | §10 | L1614-1800 | ~400 |
| sverchok-impl-topologic | §11 (Topologic) | L1804-1870 | ~250 |
| sverchok-impl-extensions | §11 (non-Topologic) | L1870-1936 | ~250 |
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

## 6. Review Feedback Incorporation

All review items from both the technical and practical reviews have been addressed:

| Review Item | Source | Resolution |
|-------------|--------|------------|
| Split sverchok-impl-extensions for TopologicSverchok | P-01 | Created SV-I-04 (topologic) and SV-I-05 (extensions) |
| Reframe parametric around AEC domain | P-02, P-03 | Restructured scope: structural grids, facades, stairs, roof, MEP, terrain |
| Add trigger-word-rich descriptions | P-04 | All 12 skills now have Description field with domain terms |
| Add 4 validator checks | P-05 | Checks 13-16 added: import validation, socket name consistency, match_long_repeat unpacking, sv_get default |
| Getting started path | P-06 | Added to core-concepts scope |
| Remove vectorize/setup overlap | P-07 | Removed from parametric, reference to scripting added |
| Add impl-ifcsverchok to errors-common deps | T-4a | Added to dependency list |
| Replace 'all impl skills' in validator deps | T-4b | Explicit list of 7 dependencies |
| Add edge/face data formats | T-7 | Added "edge data=level 2, face data=level 2" to syntax-data |
| Add SvIfcCore double-nested processing | T-8 | Added to impl-ifcsverchok scope |
| Socket processing flag ownership | T-3a | Cross-reference notes added to sockets and data skills |
| TopologicSverchok dual coverage | T-3b | Resolved by split — topologic owns library, parametric references it |
| sv_deep_copy assignment | T-9 | Added to core-concepts key API surface |
| Scripting size warning | T-1 | Split plan note added to scripting skill |

---

*Document generated by sv-masterplan-def agent. Date: 2026-03-07.*
