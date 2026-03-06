# Bonsai Skill Inventory Briefing

> **Date**: 2026-03-06 | **Target**: Bonsai v0.8.4 | **Blender**: 4.2.0+ | **Python**: 3.11
> **Status**: FINAL — for masterplan agent consumption

---

## 1. Definitive Skill List (14 Skills)

### 1.1 bonsai-core-architecture
- **Category**: core
- **Scope**: Three-layer architecture (UI/Core/Tool), `@interface` decorator pattern, IfcStore singleton, module registration system, demo module as reference, dependency injection
- **Key API surface**: `bonsai.bim.ifc.IfcStore`, `bonsai.core.tool.@interface`, `bonsai.tool.*` classes, `bim/module/` structure, `IfcStore.execute_ifc_operator()`
- **Source sections**: vooronderzoek §2 (Architecture), supplementary §5.1-5.2 (Tool/Core/UI Pattern, IfcStore Deep Dive)
- **SOURCES.md URLs**: Bonsai Core (`src/bonsai/bonsai/core/`), Bonsai Tool (`src/bonsai/bonsai/tool/`), IfcStore (`src/bonsai/bonsai/bim/ifc.py`)
- **Dependencies**: blender-core-api, blender-syntax-operators, blender-syntax-addons
- **Estimated complexity**: L

### 1.2 bonsai-syntax-elements
- **Category**: syntax
- **Scope**: Creating/editing IFC elements via Bonsai, `root.create_entity`, class assignment (`bim.assign_class`), element↔object linking, `tool.Ifc.get_entity()/get_object()/link()/run()`
- **Key API surface**: `tool.Ifc.get()`, `tool.Ifc.run()`, `tool.Ifc.get_entity(obj)`, `tool.Ifc.get_object(entity)`, `bpy.ops.bim.assign_class`, `IfcStore.id_map`, `IfcStore.guid_map`
- **Source sections**: vooronderzoek §3 (IFC Integration), §7 (Modeling Workflow), §12 (API Reference)
- **SOURCES.md URLs**: Bonsai in IfcOpenShell repo (`src/bonsai/`), IfcStore (`src/bonsai/bonsai/bim/ifc.py`)
- **Dependencies**: bonsai-core-architecture, ifcos-syntax-api, ifcos-syntax-elements
- **Estimated complexity**: M

### 1.3 bonsai-syntax-spatial
- **Category**: syntax
- **Scope**: Spatial hierarchy (Project/Site/Building/Storey/Space), `IfcRelAggregates`, `IfcRelContainedInSpatialStructure`, `IfcRelReferencedInSpatialStructure`, container assignment, spatial decomposition queries
- **Key API surface**: `tool.Spatial`, `bonsai.core.spatial`, `spatial.assign_container`, `aggregate.assign_object`, `spatial.reference_structure`, `ifcopenshell.util.element.get_container()`
- **Source sections**: vooronderzoek §4 (Spatial Structure Management)
- **SOURCES.md URLs**: Tool spatial (`src/bonsai/bonsai/tool/spatial.py`), Core spatial (`src/bonsai/bonsai/core/spatial.py`)
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Estimated complexity**: M

### 1.4 bonsai-syntax-properties
- **Category**: syntax
- **Scope**: Property sets (`IfcPropertySet`), quantity sets (`IfcElementQuantity`), pset templates, naming conventions (Pset_/Qto_/EPset_/custom), reading psets via `ifcopenshell.util.element.get_psets()`, `should_purge` behavior, type-driven property inheritance
- **Key API surface**: `pset.add_pset`, `pset.edit_pset`, `pset.add_qto`, `pset.edit_qto`, `pset_template.add_pset_template`, `ifcopenshell.util.element.get_psets()`
- **Source sections**: vooronderzoek §5 (Property Sets and Quantity Sets)
- **SOURCES.md URLs**: Bonsai module pset (`src/bonsai/bonsai/bim/module/pset/`), ifcopenshell.api.pset
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Estimated complexity**: M

### 1.5 bonsai-syntax-geometry
- **Category**: syntax
- **Scope**: Representation contexts (Model/Body/Axis/Box/FootPrint, Plan/Annotation), representation types (SweptSolid, Tessellation, Clipping, Curve2D, MappedItem), `ShapeBuilder`, geometry API, opening voids (`feature.add_feature`), `IfcRelVoidsElement`/`IfcRelFillsElement`
- **Key API surface**: `geometry.add_wall_representation`, `geometry.assign_representation`, `context.add_context`, `ifcopenshell.util.representation.get_context()`, `ifcopenshell.util.shape_builder.ShapeBuilder`
- **Source sections**: vooronderzoek §9 (Geometry Representations), §7 (Openings)
- **SOURCES.md URLs**: Bonsai module geometry, ifcopenshell.api.geometry, ifcopenshell.util.shape_builder
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-geometry, ifcos-core-concepts
- **Estimated complexity**: M

### 1.6 bonsai-impl-project
- **Category**: impl
- **Scope**: New IFC project creation, schema selection (IFC2X3/IFC4/IFC4X3), MVD selection, save workflow (native IFC — no export), header metadata, IDS/IfcTester validation, schema migration (IFC2X3→IFC4), georeference setup, owner/history tracking
- **Key API surface**: `bpy.ops.bim.save_project`, `project.edit_header`, `ifctester`, `ifcopenshell.util.schema.Migrator`, modules: project, owner, georeference, tester, patch
- **Source sections**: vooronderzoek §10 (IFC Export Pipeline/Saving)
- **SOURCES.md URLs**: Bonsai docs (`docs.bonsaibim.org`), IfcTester docs, Bonsai module project
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements, ifcos-syntax-fileio
- **Estimated complexity**: M

### 1.7 bonsai-impl-modeling
- **Category**: impl
- **Scope**: Wall/slab/column/beam creation (both parametric tool and generic mesh workflow), type system (`IfcElementType`, `IfcRelDefinesByType`, MappedItem sharing), material assignment (layer sets, profile sets, constituent sets), opening creation and filling, predefined types
- **Key API surface**: `bpy.ops.bim.assign_class`, `type.assign_type`, `material.add_material_set`, `material.add_layer`, `material.add_profile`, `feature.add_feature`, `void.add_filling`, modules: model, type, material, void/feature, covering, profile
- **Source sections**: vooronderzoek §6 (Type System), §7 (Modeling Workflow)
- **SOURCES.md URLs**: Bonsai module model, type, material; IfcOpenShell api/type, api/material
- **Dependencies**: bonsai-syntax-elements, bonsai-syntax-spatial, bonsai-syntax-geometry, ifcos-syntax-api
- **Estimated complexity**: L

### 1.8 bonsai-impl-classification
- **Category**: impl
- **Scope**: Built-in classification systems (Uniclass 2015, OmniClass), custom systems (NL-SfB, MasterFormat), multiple simultaneous classifications, bSDD integration, reading classifications from elements
- **Key API surface**: `classification.add_classification`, `classification.add_reference`, `classification.edit_classification`, `classification.remove_reference`, modules: classification, bsdd
- **Source sections**: vooronderzoek §8 (Classification Systems)
- **SOURCES.md URLs**: Bonsai module classification, bSDD portal
- **Dependencies**: bonsai-syntax-elements, ifcos-syntax-api
- **Estimated complexity**: S

### 1.9 bonsai-impl-drawing
- **Category**: impl
- **Scope**: 2D drawing generation from 3D BIM, SVG pipeline (underlay/linework/annotations), drawing types (PLAN_VIEW, SECTION_VIEW, ELEVATION_VIEW, REFLECTED_PLAN_VIEW, MODEL_VIEW), annotations (12+ types), sheets/documents, SheetBuilder, camera configuration, InkScape dependency, drawing styles, titleblock templates
- **Key API surface**: `bpy.ops.bim.add_drawing`, `bpy.ops.bim.create_drawing`, `bpy.ops.bim.add_annotation`, `bpy.ops.bim.add_sheet`, `bpy.ops.bim.create_sheets`, `SvgWriter`, `SheetBuilder`, `BIMCameraProperties`, `EPset_Drawing`; modules: drawing (16 files, 40+ operators)
- **Source sections**: supplementary §1 (Drawing Module — all 9 subsections)
- **SOURCES.md URLs**: Drawing module source (`src/bonsai/bonsai/bim/module/drawing/`), Core drawing, Tool drawing, Bonsai Drawing Guide
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements, bonsai-syntax-geometry, blender-core-gpu
- **Estimated complexity**: L

### 1.10 bonsai-impl-qto
- **Category**: impl
- **Scope**: Quantity takeoff from Blender geometry, calculator system (30+ functions: linear/area/volume/weight), QTO rules (JSON config), standard quantity sets (Qto_WallBaseQuantities etc.), gross vs. net distinction, cost integration, unit conversion
- **Key API surface**: `bpy.ops.bim.perform_quantity_take_off`, `bpy.ops.bim.calculate_single_quantity`, calculator functions (get_net_volume, get_gross_volume, etc.), `BIMQtoProperties`, modules: qto (7 files, 8 operators)
- **Source sections**: supplementary §2 (QTO Module — all 7 subsections)
- **SOURCES.md URLs**: QTO module source (`src/bonsai/bonsai/bim/module/qto/`)
- **Dependencies**: bonsai-syntax-elements, bonsai-syntax-properties, blender-syntax-mesh
- **Estimated complexity**: M

### 1.11 bonsai-impl-bcf
- **Category**: impl
- **Scope**: BIM Collaboration Format (v2.1 + v3.0), `BcfStore` singleton, topic CRUD, viewpoints (camera/visibility/clipping), comments, document references, BIM snippets, georeference offset handling, drag-and-drop .bcf support
- **Key API surface**: `bpy.ops.bim.new_bcf_project`, `load_bcf_project`, `save_bcf_project`, `add_bcf_topic`, `add_bcf_viewpoint`, `activate_bcf_viewpoint`, `add_bcf_comment`, `BcfStore`, `BCFProperties`, modules: bcf (5 files, 30+ operators)
- **Source sections**: supplementary §3 (BCF Module — all 8 subsections)
- **SOURCES.md URLs**: BCF module source (`src/bonsai/bonsai/bim/module/bcf/`), IfcOpenShell BCF docs
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements
- **Estimated complexity**: M

### 1.12 bonsai-impl-clash
- **Category**: impl
- **Scope**: Clash detection (intersection/collision/clearance modes), clash sets (Group A/B with IFC sources), element filtering, FCL engine, result visualization (ClashDecorator), smart grouping, BCF output integration, clash configuration import/export
- **Key API surface**: `bpy.ops.bim.execute_ifc_clash`, `add_clash_set`, `add_clash_source`, `select_clash`, `smart_clash_group`, `BIMClashProperties`, `ClashSet`, `ClashSource`, modules: clash (6 files, 12 operators)
- **Source sections**: supplementary §4 (Clash Detection — all 8 subsections)
- **SOURCES.md URLs**: Clash module source (`src/bonsai/bonsai/bim/module/clash/`), IfcClash docs
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements, bonsai-impl-bcf (optional)
- **Estimated complexity**: M

### 1.13 bonsai-errors-common
- **Category**: errors
- **Scope**: All 10 AI common mistakes, 8 common error patterns, module-specific anti-patterns (drawing: 7, QTO: 7, BCF: 7, clash: 7), operator context requirements, `blenderbim→bonsai` migration, operators vs. `ifcopenshell.api` decision tree, data synchronization pitfalls, cache purging, threading limitations
- **Key API surface**: All error-prone APIs documented across research
- **Source sections**: vooronderzoek §11 (Error Patterns), §13 (AI Mistakes), supplementary §1.9, §2.7, §3.8, §4.8, §5.3-5.8
- **SOURCES.md URLs**: All Bonsai source URLs for cross-reference
- **Dependencies**: All other bonsai skills (references them)
- **Estimated complexity**: L

### 1.14 bonsai-ifc-validator
- **Category**: agents
- **Scope**: Validation checklists for IFC models created via Bonsai, schema compliance checks, IfcTester/IDS integration, pre-save validation, spatial structure completeness, property set conformance
- **Key API surface**: `ifctester`, `ifcopenshell.validate`, `bpy.ops.bim.validate_*`
- **Source sections**: vooronderzoek §10 (IDS/IfcTester), supplementary §5 (runtime quirks)
- **SOURCES.md URLs**: IfcTester docs, IDS Standard
- **Dependencies**: bonsai-core-architecture, bonsai-syntax-properties, ifcos-core-schemas
- **Estimated complexity**: S

---

## 2. Merge/Split/Add/Remove Recommendations

### Module → Skill Mapping (51 bim/module/ directories)

| Module(s) | Decision | Target Skill | Rationale |
|-----------|----------|-------------|-----------|
| `project` | Map | bonsai-impl-project | Core workflow, well-researched |
| `spatial`, `aggregate`, `nest` | Merge | bonsai-syntax-spatial | All spatial hierarchy related |
| `pset`, `pset_template`, `attribute` | Merge | bonsai-syntax-properties | All property-related |
| `geometry`, `context`, `profile`, `cad` | Merge | bonsai-syntax-geometry | All geometry/representation |
| `model`, `type`, `material`, `void`, `covering` | Merge | bonsai-impl-modeling | All physical element creation |
| `classification`, `bsdd` | Merge | bonsai-impl-classification | Both classification systems |
| `drawing`, `document`, `layer` | Merge | bonsai-impl-drawing | All documentation/2D output |
| `qto`, `cost` | Merge | bonsai-impl-qto | Cost integrates with QTO |
| `bcf` | Map | bonsai-impl-bcf | Standalone, well-researched |
| `clash` | Map | bonsai-impl-clash | Standalone, well-researched |
| `owner`, `georeference`, `gis`, `tester`, `patch`, `unit` | Merge | bonsai-impl-project | Project setup/maintenance |
| `root` | Absorb | bonsai-syntax-elements | Core element creation |
| `search`, `csv`, `diff` | Absorb | bonsai-errors-common | Utility operations, mention in errors/tips |
| `style` | Absorb | bonsai-impl-modeling | Visual style goes with materials |
| `debug`, `demo`, `web`, `misc` | Skip | — | Internal/development only |
| `boundary` | Skip | — | Niche (space boundaries), no research |
| `brick` | Skip | — | IoT/Brickschema, outside AEC scope |
| `constraint` | Skip | — | Niche, no research |
| `fm` | Skip | — | Facility management, no research |
| `ifcgit` | Skip | — | Experimental version control |
| `library` | Skip | — | External library refs, niche |
| `resource` | Skip | — | Construction resources, no research |
| `sequence` | Skip (future) | — | 4D scheduling, no Bonsai research yet |
| `structural` | Skip | — | Structural analysis, niche |
| `system` | Skip (future) | — | MEP systems, no Bonsai research yet |
| `alignment` | Skip | — | Infrastructure alignment, niche |

### Key Splits

- **bonsai-impl-drawing** MUST be standalone — 16 files, 40+ operators, complex SVG pipeline
- **bonsai-errors-common** consolidates all anti-patterns from separate modules into one skill (31+ documented anti-patterns)

### Key Merges

- **spatial + aggregate + nest** → single spatial skill (all describe hierarchical IFC relationships)
- **qto + cost** → single QTO skill (cost integration is a QTO sub-feature)
- **geometry + context + profile + cad** → single geometry skill (all deal with representation)

### Additions vs. Raw Masterplan

| Raw Masterplan Skill | Change | New Skill |
|---------------------|--------|-----------|
| bonsai-impl-export | **Remove** | Merged into bonsai-impl-project (no export in Bonsai — native IFC) |
| bonsai-errors-ifc | **Merge** | Combined into bonsai-errors-common |
| bonsai-errors-geometry | **Merge** | Combined into bonsai-errors-common |
| — | **Add** | bonsai-impl-drawing |
| — | **Add** | bonsai-impl-qto |
| — | **Add** | bonsai-impl-bcf |
| — | **Add** | bonsai-impl-clash |

**Net result**: 12 skills in raw masterplan → 14 skills after research.

---

## 3. Key Patterns Summary

### The Tool/Core/UI Pattern (CRITICAL)

Every Bonsai module follows a three-layer architecture:

```
UI Layer      → bonsai/bim/module/<name>/operator.py  (Blender operators)
Core Layer    → bonsai/core/<name>.py                  (pure logic, no bpy)
Tool Layer    → bonsai/tool/<name>.py                  (Blender-specific impl)
```

**Call flow**: Operator → collects context → calls `core.module.function(tool.Ifc, tool.Module, params)` → core calls tool methods via dependency injection → tool accesses bpy/IfcStore.

The `@interface` decorator in `core/tool.py` enforces contracts on tool implementations.

### Top 5 Bonsai Operator Patterns

1. **IfcStore.execute_ifc_operator(self, context)** — Wraps `_execute()` with IFC transaction management, undo integration, and error handling. MOST operators use this.
2. **tool.Ifc.run("api.module.function", **kwargs)** — Preferred way to call ifcopenshell.api against the active file. Handles file reference automatically.
3. **tool.Ifc.get_entity(obj) / get_object(entity)** — Bidirectional lookup between Blender objects and IFC entities via IfcStore.id_map/guid_map.
4. **BIMObjectProperties.ifc_definition_id** — Every Blender object linked to IFC stores its entity ID here. `0` = no IFC link.
5. **Data cache + purge pattern** — Each module has `data.py` with lazy-loaded caches. Direct API calls MUST purge relevant caches (e.g., `PsetData.purge()`).

### Top 5 Anti-Patterns

1. **Using `blenderbim.*` imports** — ALWAYS use `bonsai.*` (renamed 2024, v0.8.0+)
2. **Using `void.add_opening()`** — Non-existent; use `feature.add_feature()` for openings
3. **Treating Bonsai as importer/exporter** — It's native IFC authoring; IFC IS the document
4. **Modifying IFC via API without cache purge** — Blender UI won't update; ALWAYS purge module data caches
5. **Moving objects without `bim.edit_object_placement()`** — Standard Blender transforms do NOT update IfcLocalPlacement

### When to Use Bonsai Operators vs ifcopenshell.api

| Scenario | Use |
|----------|-----|
| Interactive modeling (add walls, slabs) | `bpy.ops.bim.*` operators |
| Batch property modifications | `ifcopenshell.api` directly + cache purge |
| Read-only IFC queries | `tool.Ifc.get()` + ifcopenshell queries |
| Custom entity creation with full control | `ifcopenshell.api` + manual sync |
| Headless/no-UI scripts | `ifcopenshell.api` directly (MUST) |
| Scripts that should be stable across versions | `ifcopenshell.api` (more stable than operators) |

---

## 4. Source Coverage Map

| Skill | Research Sections | Source Code URLs |
|-------|-------------------|-----------------|
| bonsai-core-architecture | vooronderzoek §2, supplementary §5.1-5.2 | `src/bonsai/bonsai/core/`, `src/bonsai/bonsai/tool/`, `src/bonsai/bonsai/bim/ifc.py` |
| bonsai-syntax-elements | vooronderzoek §3, §7, §12 | `src/bonsai/bonsai/bim/ifc.py`, `src/bonsai/bonsai/tool/ifc.py` |
| bonsai-syntax-spatial | vooronderzoek §4 | `src/bonsai/bonsai/tool/spatial.py`, `src/bonsai/bonsai/core/spatial.py`, `src/bonsai/bonsai/bim/module/spatial/` |
| bonsai-syntax-properties | vooronderzoek §5 | `src/bonsai/bonsai/bim/module/pset/`, ifcopenshell `api/pset/` |
| bonsai-syntax-geometry | vooronderzoek §9 | `src/bonsai/bonsai/bim/module/geometry/`, ifcopenshell `api/geometry/`, `util/shape_builder.py` |
| bonsai-impl-project | vooronderzoek §10 | `src/bonsai/bonsai/bim/module/project/`, `src/bonsai/bonsai/bim/module/tester/` |
| bonsai-impl-modeling | vooronderzoek §6, §7 | `src/bonsai/bonsai/bim/module/model/`, `module/type/`, `module/material/` |
| bonsai-impl-classification | vooronderzoek §8 | `src/bonsai/bonsai/bim/module/classification/`, `module/bsdd/` |
| bonsai-impl-drawing | supplementary §1 (all) | `src/bonsai/bonsai/bim/module/drawing/` (16 files), `core/drawing.py`, `tool/drawing.py` |
| bonsai-impl-qto | supplementary §2 (all) | `src/bonsai/bonsai/bim/module/qto/` (7 files) |
| bonsai-impl-bcf | supplementary §3 (all) | `src/bonsai/bonsai/bim/module/bcf/` (5 files) |
| bonsai-impl-clash | supplementary §4 (all) | `src/bonsai/bonsai/bim/module/clash/` (6 files) |
| bonsai-errors-common | vooronderzoek §11, §13, supplementary §1.9, §2.7, §3.8, §4.8, §5.3-5.8 | All module sources (for anti-pattern verification) |
| bonsai-ifc-validator | vooronderzoek §10 (IDS) | ifctester source, `src/bonsai/bonsai/bim/module/tester/` |

All source code URLs are relative to: `https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/`

---

## 5. Bonsai-IfcOpenShell Boundary

### Decision Criteria

| Criterion | Bonsai Skill | IfcOpenShell Skill |
|-----------|-------------|-------------------|
| Requires `bpy` (Blender context) | YES | NO |
| Uses `bonsai.tool.*` or `bonsai.core.*` | YES | NO |
| Calls `bpy.ops.bim.*` operators | YES | NO |
| Uses IfcStore / id_map / guid_map | YES | NO |
| Pure `ifcopenshell.api.run()` without Blender | NO | YES |
| Pure `ifcopenshell.open()`/`.write()` file I/O | NO | YES |
| `ifcopenshell.util.*` utility functions | Mention in context | YES (primary) |
| `ifcopenshell.geom` geometry processing | NO | YES |
| Schema-level concepts (IFC2X3/IFC4/IFC4X3 differences) | NO | YES |

### Shared Concepts (document in BOTH but with different emphasis)

| Concept | Bonsai Skill Emphasis | IfcOpenShell Skill Emphasis |
|---------|----------------------|---------------------------|
| Spatial structure | Bonsai UI workflow + tool.Spatial | ifcopenshell.api.spatial + ifcopenshell.api.aggregate |
| Property sets | Bonsai property panels + cache purge | ifcopenshell.api.pset + ifcopenshell.util.element.get_psets() |
| Type system | MappedItem in Blender viewport | ifcopenshell.api.type API parameters |
| Geometry | Blender mesh ↔ IFC representation sync | ifcopenshell.api.geometry + ShapeBuilder |
| Classification | Bonsai UI integration + bSDD | ifcopenshell.api.classification |

### Rules

1. If the operation requires a running Blender instance with Bonsai loaded → **Bonsai skill**
2. If the operation works in standalone Python (no bpy) → **IfcOpenShell skill**
3. If the operation bridges both (e.g., `tool.Ifc.run()` wrapping `ifcopenshell.api`) → **Bonsai skill** documents the Bonsai-side, references the IfcOpenShell skill for API details
4. Anti-patterns specific to the Bonsai↔IfcOpenShell interaction (cache purge, sync, operator vs API) → **bonsai-errors-common**
5. Schema version differences (entity availability, attribute changes) → **IfcOpenShell skills** exclusively

---

## Summary Statistics

| Metric | Value |
|--------|-------|
| Total Bonsai skills | 14 |
| Syntax skills | 4 |
| Implementation skills | 6 |
| Error skills | 1 |
| Core skills | 1 |
| Agent skills | 1 |
| Small (S) | 2 |
| Medium (M) | 8 |
| Large (L) | 4 |
| Modules covered | ~40 of 51 |
| Modules skipped | ~11 (niche/experimental) |
| Research sections used | All (vooronderzoek §1-13, supplementary §1-5, scope-analysis §3) |
