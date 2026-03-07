# Skill Catalog

Complete index of all Claude Code skills in this package for Blender, IfcOpenShell, Bonsai, Sverchok, and Cross-Technology AEC workflows.

## Summary

| Technology | Count |
|---|---|
| Blender | 26 |
| IfcOpenShell | 19 |
| Bonsai | 14 |
| Sverchok | 12 |
| Cross-Tech | 2 |
| **Total** | **73** |

---

## Blender

### Agents

| Skill Name | Description | Directory |
|---|---|---|
| blender-agents-code-validator | Provides a systematic validation checklist for reviewing Blender Python code, checking for deprecated API usage, context errors, version compatibility issues, threading violations, data reference invalidation, incorrect operator calls, and addon/extension structure compliance. | skills/blender/agents/blender-agents-code-validator |
| blender-agents-version-migrator | Provides a systematic migration process for updating Blender Python scripts and addons across major versions (3.x to 4.x to 5.x), covering API renames, removed functions, changed parameters, extension system migration, BGL to GPU module conversion, and bone collection migration. | skills/blender/agents/blender-agents-version-migrator |

### Core

| Skill Name | Description | Directory |
|---|---|---|
| blender-core-api | Guides Blender Python API usage including bpy module structure, RNA data access, context system, dependency graph, and operator invocation. | skills/blender/core/blender-core-api |
| blender-core-gpu | Guides GPU drawing in Blender Python using the gpu module, built-in shaders, batch rendering, gpu.state management, SpaceView3D draw handlers, and offscreen rendering. | skills/blender/core/blender-core-gpu |
| blender-core-runtime | Covers Blender Python runtime behavior including mathutils module (Vector, Matrix, Quaternion, KDTree, BVHTree), threading restrictions, undo/redo invalidation of bpy references, application handlers with @persistent, bpy.app.timers, bpy.msgbus subscriptions, and background mode limitations. | skills/blender/core/blender-core-runtime |
| blender-core-versions | Provides complete Blender Python API version matrix from 3.x through 5.1 with all breaking changes, migration paths, and version-safe coding patterns. | skills/blender/core/blender-core-versions |

### Errors

| Skill Name | Description | Directory |
|---|---|---|
| blender-errors-context | Diagnoses and resolves Blender Python context errors including RuntimeError from restricted context access, operator poll() failures, wrong context for bpy.ops calls, and modal operator context issues. | skills/blender/errors/blender-errors-context |
| blender-errors-data | Diagnoses and resolves Blender Python data access errors including ReferenceError from removed objects, undo-invalidated bpy.data references, stale ID pointers after file operations, and data lifecycle management. | skills/blender/errors/blender-errors-data |
| blender-errors-version | Diagnoses and resolves Blender Python version compatibility errors including AttributeError from removed APIs, ImportError from deprecated modules (bgl in 5.0), breaking changes in operator signatures, and version-specific API differences. | skills/blender/errors/blender-errors-version |

### Impl

| Skill Name | Description | Directory |
|---|---|---|
| blender-impl-addons | Guides complete Blender addon development workflows including project structure for multi-file addons, testing strategies, extension packaging for extensions.blender.org, CI/CD pipelines, dependency management, and addon distribution. | skills/blender/impl/blender-impl-addons |
| blender-impl-animation | Provides implementation workflows for Blender animation including construction sequence animation for AEC, camera walkthrough automation, solar study animations, phasing visualization, NLA workflow orchestration, and batch keyframe operations. | skills/blender/impl/blender-impl-animation |
| blender-impl-automation | Covers Blender automation workflows including batch rendering, headless processing via blender --background, file format I/O automation (OBJ/FBX/STL/USD/glTF), scene assembly scripts, and command-line pipeline integration. | skills/blender/impl/blender-impl-automation |
| blender-impl-mesh | Provides implementation workflows for Blender mesh operations including creating buildings from vertices, IFC geometry visualization, mesh analysis tools, custom mesh generation for AEC, BMesh algorithms, and performance optimization with foreach_get/set. | skills/blender/impl/blender-impl-mesh |
| blender-impl-nodes | Provides implementation workflows for Blender node systems including creating Geometry Nodes modifiers from Python, building shader node trees programmatically, custom node groups, and linking node setups for AEC visualization. | skills/blender/impl/blender-impl-nodes |
| blender-impl-operators | Provides implementation patterns for Blender operators including modal operators with timer callbacks, file browser integration, undo/redo support, batch processing operators, progress reporting, and multi-step workflows. | skills/blender/impl/blender-impl-operators |

### Syntax

| Skill Name | Description | Directory |
|---|---|---|
| blender-syntax-addons | Defines Blender addon and extension development patterns including legacy bl_info dict, blender_manifest.toml (4.2+), register/unregister lifecycle, multi-file addon structure, AddonPreferences, class naming conventions, and extension packaging for extensions.blender.org. | skills/blender/syntax/blender-syntax-addons |
| blender-syntax-animation | Covers Blender animation API including keyframe insertion, FCurve access, Action data blocks, NLA system, BoneCollection (4.0+), armature bone layers migration, driver expressions, and timeline control. | skills/blender/syntax/blender-syntax-animation |
| blender-syntax-data | Covers Blender data management including collections, library overrides, asset system, linked libraries, BlendDataLibraries, data block creation and removal, fake users, and data transfer between files. | skills/blender/syntax/blender-syntax-data |
| blender-syntax-materials | Covers Blender material and shader node API including Principled BSDF input name changes (4.0), material slot assignment, UV mapping, texture node setup, material_slot_add, and node-based material creation. | skills/blender/syntax/blender-syntax-materials |
| blender-syntax-mesh | Covers Blender mesh data access including vertices, edges, faces, loops, BMesh creation and editing, from_pydata, UV layers, vertex colors/attributes, normals, and foreach_get/set for bulk operations. | skills/blender/syntax/blender-syntax-mesh |
| blender-syntax-modifiers | Covers Blender modifier stack API including adding/removing modifiers, applying modifiers via operators, evaluated mesh access via depsgraph, Geometry Nodes modifier input identifiers, and common AEC modifiers (Array, Boolean, Solidify). | skills/blender/syntax/blender-syntax-modifiers |
| blender-syntax-nodes | Explains Blender node tree system including Geometry Nodes, Shader Nodes, Compositor Nodes, NodeTreeInterface API (4.0+), creating and linking nodes programmatically, and node group management. | skills/blender/syntax/blender-syntax-nodes |
| blender-syntax-operators | Defines Blender operator creation patterns including bpy.types.Operator class, execute/invoke/modal methods, poll() functions, bl_idname naming, bl_options flags, return values, operator properties, and context.temp_override. | skills/blender/syntax/blender-syntax-operators |
| blender-syntax-panels | Defines Blender UI panel creation including bpy.types.Panel, draw() method, UILayout API (row/column/box/split), bl_space_type, bl_region_type, bl_category, sub-panels, draw_header, menus, and UIList. | skills/blender/syntax/blender-syntax-panels |
| blender-syntax-properties | Defines all bpy.props types (Bool/Int/Float/String/Enum/Vector/Pointer/Collection), PropertyGroup registration, subtypes, units, update callbacks, dynamic enum items, and getter/setter patterns. | skills/blender/syntax/blender-syntax-properties |
| blender-syntax-rendering | Covers Blender rendering API including render engine selection (EEVEE/Cycles/Workbench), RenderSettings configuration, output format setup, batch rendering via Python, camera setup, scene.render.*, and EEVEE identifier changes across versions. | skills/blender/syntax/blender-syntax-rendering |

---

## IfcOpenShell

### Agents

| Skill Name | Description | Directory |
|---|---|---|
| ifcos-agents-code-validator | Provides a systematic validation checklist for reviewing IfcOpenShell Python code, checking for schema compatibility errors, incorrect API usage, entity reference invalidation, performance anti-patterns, missing error handling, and IFC standard compliance. | skills/ifcopenshell/agents/ifcos-agents-code-validator |

### Core

| Skill Name | Description | Directory |
|---|---|---|
| ifcos-core-concepts | Explains IFC data model fundamentals including entity hierarchy (IfcRoot to IfcElement), spatial structure, ownership model, placement system, representation system, and relationship model across IFC2X3, IFC4, and IFC4X3 schemas. | skills/ifcopenshell/core/ifcos-core-concepts |
| ifcos-core-runtime | Explains IfcOpenShell Python runtime quirks including C++ binding behavior, entity identity (is vs ==), entity invalidation after removal, by_type() return semantics, thread safety constraints, memory management, PascalCase attribute access, and installation patterns. | skills/ifcopenshell/core/ifcos-core-runtime |

### Errors

| Skill Name | Description | Directory |
|---|---|---|
| ifcos-errors-patterns | Catalogs common IfcOpenShell error patterns including RuntimeError on invalid entities, AttributeError from wrong PascalCase, TypeError from wrong parameter types, entity invalidation after removal, and debugging strategies for IFC processing scripts. | skills/ifcopenshell/errors/ifcos-errors-patterns |
| ifcos-errors-performance | Covers IfcOpenShell performance optimization including geometry iterator vs create_shape, by_type caching, batch processing patterns, memory management for large models (100MB+), multiprocessing strategies, and profiling IFC operations. | skills/ifcopenshell/errors/ifcos-errors-performance |
| ifcos-errors-schema | Documents IFC schema-related errors and pitfalls across IFC2X3, IFC4, and IFC4X3 including entity availability differences, attribute type changes, removed/added entities, IfcOpenShell ifcpatch for schema migration, and common SchemaError debugging. | skills/ifcopenshell/errors/ifcos-errors-schema |

### Impl

| Skill Name | Description | Directory |
|---|---|---|
| ifcos-impl-cost | Guides IFC cost management using ifcopenshell.api.cost including cost schedules, cost items, cost values, cost quantities, and 5D BIM workflows. | skills/ifcopenshell/impl/ifcos-impl-cost |
| ifcos-impl-creation | Guides IFC model creation workflows using ifcopenshell.api including creating projects, spatial structure (site/building/storey), walls, slabs, columns, openings, property sets, and type assignments. | skills/ifcopenshell/impl/ifcos-impl-creation |
| ifcos-impl-geometry | Covers IfcOpenShell geometry processing including geometry settings, create_shape() for mesh extraction, geometry iterator for batch processing, representation creation, extrusion/CSG/BRep geometry, and coordinate transformations. | skills/ifcopenshell/impl/ifcos-impl-geometry |
| ifcos-impl-materials | Guides IFC material assignment using ifcopenshell.api including IfcMaterial, IfcMaterialLayerSet, IfcMaterialConstituentSet (IFC4+), IfcMaterialProfileSet, material properties, and style/presentation. | skills/ifcopenshell/impl/ifcos-impl-materials |
| ifcos-impl-mep | Guides MEP (Mechanical, Electrical, Plumbing) modeling in IFC using ifcopenshell.api.system including IfcSystem, IfcDistributionElement, ports, connections, flow segments, fittings, and MEP-specific property sets. | skills/ifcopenshell/impl/ifcos-impl-mep |
| ifcos-impl-profiles | Covers IFC profile definitions using ifcopenshell.api.profile including parametric profiles (I-beam, C-channel, rectangle, circle), arbitrary profiles from polylines, profile-based extrusions for structural elements, and material profile sets. | skills/ifcopenshell/impl/ifcos-impl-profiles |
| ifcos-impl-relationships | Covers IFC relationship management using ifcopenshell.api including spatial containment, aggregation, type assignment, property set association, material association, void relationships, and nesting. | skills/ifcopenshell/impl/ifcos-impl-relationships |
| ifcos-impl-sequence | Guides IFC scheduling and 4D BIM using ifcopenshell.api.sequence including work schedules, tasks, task time relationships, Gantt chart data extraction, and construction sequence modeling. | skills/ifcopenshell/impl/ifcos-impl-sequence |
| ifcos-impl-validation | Implements IfcOpenShell validation workflows including ifcopenshell.validate for schema compliance checking, ifctester for IDS (Information Delivery Specification) validation, georeference validation across IFC2X3 vs IFC4+ differences, and custom validation rule creation. | skills/ifcopenshell/impl/ifcos-impl-validation |

### Syntax

| Skill Name | Description | Directory |
|---|---|---|
| ifcos-syntax-api | Documents the ifcopenshell.api module system with all 30+ API modules, invocation patterns via api.run() and direct module calls, parameter conventions, and module categorization. | skills/ifcopenshell/syntax/ifcos-syntax-api |
| ifcos-syntax-elements | Covers IfcOpenShell element traversal and querying including by_type, by_id, by_guid, inverse references, entity attributes, get_info(), is_a(), GUID utilities, and the universal IFC property extraction pattern. | skills/ifcopenshell/syntax/ifcos-syntax-elements |
| ifcos-syntax-fileio | Handles IfcOpenShell file I/O operations including opening, creating, writing, and serializing IFC files, plus transaction management with undo/redo support. | skills/ifcopenshell/syntax/ifcos-syntax-fileio |
| ifcos-syntax-util | Documents ifcopenshell.util modules for common IFC operations including element utilities, selector syntax, placement helpers, date/unit conversion, cost/schedule utilities, and shape extraction. | skills/ifcopenshell/syntax/ifcos-syntax-util |

---

## Bonsai

### Agents

| Skill Name | Description | Directory |
|---|---|---|
| bonsai-agents-ifc-validator | Provides a systematic validation process for IFC models in Bonsai, checking spatial hierarchy completeness, property set compliance, geometry validity, classification correctness, and IDS (Information Delivery Specification) conformance using ifctester. | skills/bonsai/agents/bonsai-agents-ifc-validator |

### Core

| Skill Name | Description | Directory |
|---|---|---|
| bonsai-core-architecture | Explains Bonsai (formerly BlenderBIM) addon architecture including native IFC workflow, tool.Ifc.get() for IfcOpenShell file access, bpy.ops.bim.* operator namespace, BIM property panels, IFC-backed custom properties, and the relationship between Blender objects and IFC entities. | skills/bonsai/core/bonsai-core-architecture |

### Errors

| Skill Name | Description | Directory |
|---|---|---|
| bonsai-errors-common | Catalogs common Bonsai error patterns including IFC schema violations, spatial hierarchy errors, property set failures, geometry representation issues, operator poll failures, drawing generation errors, and BCF/clash workflow problems. | skills/bonsai/errors/bonsai-errors-common |

### Impl

| Skill Name | Description | Directory |
|---|---|---|
| bonsai-impl-bcf | Guides implementation of BIM Collaboration Format (BCF) workflows in Bonsai including creating BCF topics, adding viewpoints with camera snapshots, managing comments, importing/exporting BCF files (v2.1 and v3.0), and integrating BCF issue tracking with IFC element references. | skills/bonsai/impl/bonsai-impl-bcf |
| bonsai-impl-clash | Guides implementation of clash detection workflows in Bonsai using IfcClash, including defining clash sets with element groups and filters, running intersection tests, reviewing and resolving clashes, smart grouping of results, and integration with BCF for issue tracking. | skills/bonsai/impl/bonsai-impl-clash |
| bonsai-impl-classification | Implements Bonsai classification workflows including applying classification systems (Uniclass 2015, OmniClass, NL-SfB, CCI), bSDD (buildingSMART Data Dictionary) integration for property lookups, managing IfcClassificationReference assignments, and bulk classification operations. | skills/bonsai/impl/bonsai-impl-classification |
| bonsai-impl-drawing | Implements Bonsai drawing and documentation workflows including creating 2D drawings from IFC models, managing drawing views (plans, sections, elevations), annotation placement, sheet layout composition, SVG generation and export, and titleblock management. | skills/bonsai/impl/bonsai-impl-drawing |
| bonsai-impl-modeling | Implements Bonsai BIM modeling workflows including placing building elements (walls, slabs, columns, beams), assigning IFC types and predefined types, material layer/profile/constituent assignment, parametric modeling with type libraries, and element copy/array operations. | skills/bonsai/impl/bonsai-impl-modeling |
| bonsai-impl-project | Implements Bonsai project management workflows including creating new IFC projects with schema selection (IFC2X3/IFC4/IFC4X3), opening and saving native IFC files, project unit configuration (metric/imperial), georeference setup with coordinate reference systems, and project template management. | skills/bonsai/impl/bonsai-impl-project |
| bonsai-impl-qto | Implements Bonsai quantity takeoff (QTO) workflows including calculating quantities from IFC element geometry, using QtoCalculator for automated base quantity computation, managing IfcElementQuantity sets (area, length, volume, weight), custom quantity set definitions, and bulk quantity operations across building elements. | skills/bonsai/impl/bonsai-impl-qto |

### Syntax

| Skill Name | Description | Directory |
|---|---|---|
| bonsai-syntax-elements | Provides Bonsai IFC element access syntax including tool.Ifc.get() for file access, element-to-Blender-object mapping, IfcStore element retrieval, relating IFC entities to scene objects, and the Bonsai data bridge between bpy.types.Object and IFC elements. | skills/bonsai/syntax/bonsai-syntax-elements |
| bonsai-syntax-geometry | Provides Bonsai geometry creation and editing syntax including IFC representation management, profile extrusion, boolean operations, mesh-to-IFC geometry conversion, and Bonsai-specific geometry tools. | skills/bonsai/syntax/bonsai-syntax-geometry |
| bonsai-syntax-properties | Provides Bonsai property set management syntax for creating, reading, and editing IFC property sets (Pset) and quantity sets (Qto). | skills/bonsai/syntax/bonsai-syntax-properties |
| bonsai-syntax-spatial | Provides Bonsai spatial structure syntax for creating and managing IFC spatial hierarchy including IfcSite, IfcBuilding, IfcBuildingStorey, and IfcSpace elements. | skills/bonsai/syntax/bonsai-syntax-spatial |

---

## Sverchok

### Agents

| Skill Name | Description | Directory |
|---|---|---|
| sverchok-agents-code-validator | Validation agent for Sverchok code — runs 19 automated checks on generated Sverchok node code covering data nesting correctness, updateNode callbacks, list matching patterns, socket consistency, import validation, and IfcSverchok compliance. | skills/sverchok/agents/sverchok-agents-code-validator |

### Core

| Skill Name | Description | Directory |
|---|---|---|
| sverchok-core-concepts | Explains Sverchok parametric node system architecture including node tree execution, data flow between nodes, socket data cache, update triggers, and the 18+ node categories with 500+ nodes. | skills/sverchok/core/sverchok-core-concepts |

### Errors

| Skill Name | Description | Directory |
|---|---|---|
| sverchok-errors-common | Documents the 17 most common Sverchok error patterns and AI mistakes — covering data nesting errors, missing updateNode callbacks, socket data mutation, list matching misunderstandings, and IfcSverchok-specific pitfalls. | skills/sverchok/errors/sverchok-errors-common |

### Impl

| Skill Name | Description | Directory |
|---|---|---|
| sverchok-impl-custom-nodes | Complete guide to developing custom Sverchok nodes including the full node lifecycle, socket creation, property management, BMesh integration, and registration patterns. | skills/sverchok/impl/sverchok-impl-custom-nodes |
| sverchok-impl-parametric | Guides AEC parametric design workflows in Sverchok including structural grids, facade panel systems, parametric stairs, roof geometry generation, MEP routing layouts, and terrain generation from data. | skills/sverchok/impl/sverchok-impl-parametric |
| sverchok-impl-ifcsverchok | Covers IfcSverchok BIM integration — generating IFC files from Sverchok geometry using 31 specialized nodes. Explains SvIfcStore transient file management, two geometry conversion modes, the 6-step IFC generation workflow, and Bonsai integration. | skills/sverchok/impl/sverchok-impl-ifcsverchok |
| sverchok-impl-topologic | Covers TopologicSverchok for building topology analysis — non-manifold topology operations, CellComplex workflows, space adjacency graphs, dual graphs, and energy simulation integration. | skills/sverchok/impl/sverchok-impl-topologic |
| sverchok-impl-extensions | Covers the Sverchok extension ecosystem and extension development pattern. Includes Sverchok-Extra for advanced geometry (surfaces, fields, solids, SDF), Open3d integration, and a guide to developing and registering custom extensions. | skills/sverchok/impl/sverchok-impl-extensions |

### Syntax

| Skill Name | Description | Directory |
|---|---|---|
| sverchok-syntax-sockets | Covers all 16 Sverchok socket types including choosing the right socket for geometry, numbers, matrices, curves, surfaces, and fields. Explains socket properties, implicit type conversions, and data processing flags. | skills/sverchok/syntax/sverchok-syntax-sockets |
| sverchok-syntax-data | Explains Sverchok's critical data nesting system and list matching — the #1 source of errors. Covers nesting levels for vertices (level 3), edges/faces (level 2), matrices (level 1), plus the 5 list matching modes. | skills/sverchok/syntax/sverchok-syntax-data |
| sverchok-syntax-scripting | Covers Sverchok Python scripting nodes — SNLite, SN Functor B, Formula Mk5, and Profile Mk3. Explains socket declaration syntax, type identifiers, built-in aliases, special functions, and template system. | skills/sverchok/syntax/sverchok-syntax-scripting |
| sverchok-syntax-api | Guides programmatic control of Sverchok node trees from Python including creating nodes, connecting sockets, setting parameters, triggering updates, and batch processing with parameter sweeps. | skills/sverchok/syntax/sverchok-syntax-api |

---

## Cross-Tech

### Agents

| Skill Name | Description | Directory |
|---|---|---|
| aec-agents-workflow-orchestrator | Orchestrates complex AEC workflows spanning multiple technologies (Blender Python, IfcOpenShell, Bonsai), providing decision trees for technology selection, workflow sequencing for multi-step BIM operations, and coordination patterns for tasks like IFC creation to Blender visualization to Bonsai authoring. | skills/aec-cross-tech/agents/aec-agents-workflow-orchestrator |

### Core

| Skill Name | Description | Directory |
|---|---|---|
| aec-core-bim-workflows | Orchestrates end-to-end BIM workflows combining IfcOpenShell for IFC manipulation, Bonsai for native BIM authoring in Blender, and Blender for 3D operations. | skills/aec-cross-tech/core/aec-core-bim-workflows |

---

**Total skills: 73**
