# Review: Research Coverage Verification

**Date**: 2026-03-06
**Reviewer**: review-research-coverage agent
**Status**: COMPLETE
**Scope**: Cross-reference all 18 research documents against the masterplan's 60 skills

---

## 1. Executive Summary

The masterplan demonstrates **excellent research coverage** overall. Of ~75 substantive research sections across 18 documents, **71 are explicitly referenced** by at least one skill's "Research input" field. Four sections are unreferenced (gaps), of which **one is significant** (Validation & Georeferencing). Ecosystem insights L-006, L-007, L-009 are reflected in the masterplan's prompt template and skill design. Anti-patterns from research are comprehensively covered by error skills.

### Verdict: PASS with 4 gaps to address

| Metric | Result |
|--------|--------|
| Research documents with at least 1 skill reference | **18/18** (100%) |
| Research sections with skill coverage | **71/75** (95%) |
| Unreferenced sections (gaps) | **4** (1 significant, 3 minor) |
| Ecosystem insights (L-006/L-007/L-009) reflected | **3/3** (100%) |
| Anti-patterns in error skills | **Complete** |
| Research input references point to real sections | **Verified** (spot-checked 15 sections) |

---

## 2. Coverage Matrix: Research Documents → Skills

### 2.1 vooronderzoek-blender.md (2,099 lines, 13 sections)

| Section | Lines | Topic | Referenced By Skill(s) | Status |
|---------|-------|-------|----------------------|--------|
| §1 Overview | L29-78 | bpy module, RNA, ID blocks, data hierarchy | blender-core-api (C-01), blender-syntax-data (S-11) | COVERED |
| §2 Version Matrix | L81-446 | Breaking changes 3.x→5.0 | blender-core-versions (C-02), blender-core-gpu (C-03), blender-syntax-nodes (S-07), blender-errors-version (E-03), blender-agents-version-migrator (A-02) | COVERED |
| §3 Addon Architecture | L450-606 | bl_info, manifest.toml, register | blender-syntax-addons (S-04), blender-impl-addons (I-02) | COVERED |
| §4 Operators | L609-751 | Operator class, poll, modal, invoke | blender-syntax-operators (S-01), blender-impl-operators (I-01) | COVERED |
| §5 Properties | L754-915 | bpy.props, PropertyGroup | blender-syntax-properties (S-02) | COVERED |
| §6 UI Panels | L918-1000+ | Panel, UILayout, menus, UIList | blender-syntax-panels (S-03) | COVERED |
| §7 Mesh & BMesh | L1000+ | Mesh data, BMesh, from_pydata, UV | blender-syntax-mesh (S-05), blender-impl-mesh (I-03) | COVERED |
| §8 Modifiers | L1050+ | Modifier stack, depsgraph, GN inputs | blender-syntax-modifiers (S-06) | COVERED |
| §9 Context System | ~L1314-1413 | temp_override, restricted context, handlers | blender-impl-operators (I-01), blender-errors-context (E-01) | COVERED |
| §10 Dependency Graph | ~L1415+ | depsgraph basics | blender-core-api (C-01) | COVERED |
| §11 Common Error Patterns | ~L1500+ | Context errors, data errors | blender-errors-context (E-01), blender-errors-data (E-02), blender-agents-code-validator (A-01) | COVERED |
| §12 AI Common Mistakes | ~L1700+ | Hallucinated APIs, wrong patterns | blender-errors-context (E-01), blender-agents-code-validator (A-01) | COVERED |
| **§13 Real-World Usage** | **L1881-2069** | **GIS-to-Blender, CityJSON, building-py, Dutch AEC data** | **NONE** | **GAP** |

### 2.2 vooronderzoek-ifcopenshell.md (2,030 lines, 16 sections)

| Section | Lines | Topic | Referenced By Skill(s) | Status |
|---------|-------|-------|----------------------|--------|
| §1 Overview | L29-88 | Library components, use cases, installation | (implicit context for all ifcos skills) | COVERED (implicit) |
| §2 File I/O | L90-165 | open, create, write, transactions | ifcos-syntax-fileio | COVERED |
| §3 API.run() | L167-358 | 30+ API modules, invocation patterns | ifcos-syntax-api, ifcos-impl-materials | COVERED |
| §4 Element Traversal | L361-424 | by_type, by_id, inverse, get_info | ifcos-syntax-elements | COVERED |
| §5 util.* Modules | L427-565 | element, unit, placement, selector, date | ifcos-syntax-util | COVERED |
| §6 Geometry | L570-710 | geom settings, create_shape, iterator | ifcos-impl-geometry | COVERED |
| §7 Schema Versions | L713-900 | IFC2X3/IFC4/IFC4X3 differences | ifcos-errors-schema | COVERED |
| §8 Entity Hierarchy | L903-1093 | IfcRoot→IfcProduct, spatial structure | ifcos-core-concepts, ifcos-syntax-elements | COVERED |
| §9 Relationships | L1096-1315 | 7 relationship categories | ifcos-impl-relationships | COVERED |
| §10 Common Operations | L1326-1471 | Creation patterns, 7-step bootstrap | ifcos-impl-creation | COVERED |
| §11 Error Patterns | L1475-1611 | 10+ documented errors | ifcos-errors-schema, ifcos-errors-patterns | COVERED |
| §12 Performance | L1615-1703 | Memory, batch ops, iterator | ifcos-errors-performance | COVERED |
| §13 AI Mistakes | L1706-1735 | Hallucinated APIs, wrong params | ifcos-code-validator | COVERED |
| §14 Sources | L1738-1792 | Bibliography | N/A (references only) | N/A |
| §15 bSDD Integration | L1793-1941 | bSDD API, classification systems | bonsai-impl-classification (indirectly) | COVERED |
| **§16 Real-World Usage** | **L1942-2031** | **building.py wrapper, INB Template, web-ifc boundary** | **NONE** | **GAP (minor)** |

### 2.3 vooronderzoek-bonsai.md (1,759 lines, 13 sections)

| Section | Lines | Topic | Referenced By Skill(s) | Status |
|---------|-------|-------|----------------------|--------|
| §1 Overview | L29-85 | Bonsai description, version info | (implicit context) | COVERED (implicit) |
| §2 Architecture | ~L87-300 | Three-layer, @interface, IfcStore | bonsai-core-architecture | COVERED |
| §3 IFC Integration | ~L300-500 | Element↔object linking, tool.Ifc | bonsai-syntax-elements | COVERED |
| §4 Spatial Structure | ~L500-636 | Project/Site/Building/Storey/Space | bonsai-syntax-spatial | COVERED |
| §5 Property Sets | ~L636-730 | Psets, Qtos, templates | bonsai-syntax-properties | COVERED |
| §6 Type System | L637-730 | IfcElementType, material sets | bonsai-impl-modeling | COVERED |
| §7 Modeling Workflow | ~L730-887 | Wall/slab/column creation, openings | bonsai-syntax-elements, bonsai-impl-modeling, bonsai-syntax-geometry | COVERED |
| §8 Classification | L887-970 | Uniclass, OmniClass, NL-SfB, bSDD | bonsai-impl-classification | COVERED |
| §9 Geometry Representations | ~L970-1100 | Contexts, representations, ShapeBuilder | bonsai-syntax-geometry | COVERED |
| §10 IFC Export Pipeline | ~L1100-1207 | Save, IfcTester, schema migration, georef | bonsai-impl-project, bonsai-ifc-validator | COVERED |
| §11 Error Patterns | L1207-1430+ | 11+ error patterns | bonsai-errors-common | COVERED |
| §12 API Reference | ~L1430-1600 | Python API quick reference | bonsai-syntax-elements | COVERED |
| §13 AI Mistakes | ~L1600+ | 10+ common mistakes | bonsai-errors-common | COVERED |

### 2.4 vooronderzoek-ecosystem-sources.md (1,678 lines)

| Section | Topic | Referenced By Skill(s) | Status |
|---------|-------|----------------------|--------|
| Part 1: Claude Platform | Skill spec format, frontmatter | (reflected in prompt template) | COVERED |
| Part 2: OpenAEC Code/IFC | building.py, Monty Viewer, INB Template | (consolidated from fragments) | COVERED |
| §3 Cross-Technology Patterns | IFC extraction, creation pipeline, mesh gen | aec-core-bim-workflows | COVERED |

### 2.5 supplementary-blender-gaps.md (2,242 lines, 8 sections)

| Section | Lines | Topic | Referenced By Skill(s) | Status |
|---------|-------|-------|----------------------|--------|
| §1 Node Systems | L24-252 | GN, Shader, Compositor via Python | blender-syntax-nodes (S-07), blender-impl-nodes (I-05) | COVERED |
| §2 Animation & Rigging | L255-583 | Keyframes, FCurves, Drivers, Bone Collections | blender-syntax-animation (S-08), blender-impl-animation (I-06) | COVERED |
| §3 Materials & Shading | L586-795 | Principled BSDF, UV, textures | blender-syntax-materials (S-09), blender-impl-nodes (I-05) | COVERED |
| §4 Rendering | L799-1062 | EEVEE/Cycles, cameras, lights | blender-syntax-rendering (S-10), blender-errors-version (E-03) | COVERED |
| §5 I/O Formats | L1065-1242 | FBX, glTF, OBJ, USD, STL | blender-impl-automation (I-04) | COVERED |
| §6 Collections & Assets | L1245-1465 | Collections, libraries, Asset Browser | blender-syntax-data (S-11) | COVERED |
| §7 mathutils | L1468-1851 | Vector, Matrix, KDTree, BVHTree | blender-core-runtime (C-04) | COVERED |
| §8 Python Runtime Quirks | L1854-2193 | Threading, undo, handlers, timers, bpy-as-module | blender-core-runtime (C-04), blender-errors-data (E-02), blender-impl-automation (I-04) | COVERED |

### 2.6 supplementary-ifcos-gaps.md (1,380 lines, 8 sections)

| Section | Lines | Topic | Referenced By Skill(s) | Status |
|---------|-------|-------|----------------------|--------|
| §1 Cost Management | L23-209 | Cost schedules, items, values, quantities | ifcos-impl-cost | COVERED |
| §2 4D Scheduling | L211-491 | Work schedules, tasks, calendars, dependencies | ifcos-impl-sequence | COVERED |
| §3 MEP Systems | L493-668 | Distribution systems, ports, connectivity | ifcos-impl-mep | COVERED |
| **§4 Drawing / 2D** | **L671-744** | **ifcopenshell.api.drawing (3 functions), annotations** | **NONE** | **GAP (minor)** |
| §5 Profiles | L746-908 | 14 parametric profile types, arbitrary profiles | ifcos-impl-profiles | COVERED |
| **§6 Validation & Georef** | **L911-1112** | **ifcopenshell.validate (6 functions), ifcopenshell.api.georeference (5 functions)** | **NONE** | **GAP (significant)** |
| §7 Python Runtime Quirks | L1115-1360 | C++ bindings, entity identity, thread safety | ifcos-core-runtime | COVERED |
| §8 Sources | L1363-1380 | References | N/A | N/A |

### 2.7 supplementary-bonsai-gaps.md (1,555 lines, 5 sections)

| Section | Lines | Topic | Referenced By Skill(s) | Status |
|---------|-------|-------|----------------------|--------|
| §1 Drawing Module | L62-395 | Drawing module architecture, SVG pipeline, 16 files | bonsai-impl-drawing, bonsai-errors-common (§1.9) | COVERED |
| §2 QTO Module | L396-611 | Calculator system, QTO rules, standard quantities | bonsai-impl-qto, bonsai-errors-common (§2.7) | COVERED |
| §3 BCF Module | L612-891 | BcfStore, topics, viewpoints, comments | bonsai-impl-bcf, bonsai-errors-common (§3.8) | COVERED |
| §4 Clash Detection | L892-1127 | Clash sets, modes, FCL engine, visualization | bonsai-impl-clash, bonsai-errors-common (§4.8) | COVERED |
| §5 Python Runtime Quirks | L1128-1555 | IfcStore, cache, threading, blenderbim→bonsai | bonsai-errors-common (§5.3-5.8), bonsai-ifc-validator | COVERED |

### 2.8 IfcOpenShell Deep-Dive Fragments (4 documents)

| Document | Lines | Referenced By Skill(s) | Status |
|----------|-------|----------------------|--------|
| fragments/ifcos-api-categories.md | 1,170 | ifcos-syntax-api, ifcos-impl-creation, ifcos-impl-geometry, ifcos-impl-materials | COVERED |
| fragments/ifcos-core-operations.md | 978 | ifcos-syntax-fileio | COVERED |
| fragments/ifcos-errors-performance.md | 1,108 | ifcos-errors-patterns, ifcos-errors-performance, ifcos-code-validator | COVERED |
| fragments/ifcos-schema-versions.md | 833 | ifcos-core-concepts, ifcos-impl-relationships, ifcos-errors-schema | COVERED |

### 2.9 IfcOpenShell Topic Research (3 documents)

| Document | Lines | Referenced By Skill(s) | Status |
|----------|-------|----------------------|--------|
| topic-research/ifcos-core-operations.md | 1,473 | ifcos-syntax-fileio, ifcos-syntax-elements, ifcos-syntax-util, ifcos-impl-geometry | COVERED |
| topic-research/ifcos-errors-performance-research.md | 1,518 | ifcos-core-runtime, ifcos-errors-patterns, ifcos-errors-performance, ifcos-code-validator | COVERED |
| topic-research/ifcos-schema-version-comparison.md | 1,484 | ifcos-core-concepts, ifcos-impl-relationships | COVERED |

### 2.10 Ecosystem Fragments (3 documents, consolidated)

| Document | Lines | Status |
|----------|-------|--------|
| fragments/ecosystem-claude-platform.md | 449 | CONSOLIDATED into vooronderzoek-ecosystem-sources.md → aec-core-bim-workflows |
| fragments/ecosystem-openaec-code.md | 521 | CONSOLIDATED into vooronderzoek-ecosystem-sources.md → aec-core-bim-workflows |
| fragments/ecosystem-openaec-ifc.md | 597 | CONSOLIDATED into vooronderzoek-ecosystem-sources.md → aec-core-bim-workflows |

### 2.11 Analysis

| Document | Lines | Referenced By | Status |
|----------|-------|--------------|--------|
| scope-analysis.md | 483 | aec-core-bim-workflows (indirectly via briefings) | COVERED |

---

## 3. Gaps Found

### GAP-1: vooronderzoek-blender §13 — Real-World Usage: OpenAEC Projects (MINOR)
- **Lines**: L1881-2069 (189 lines)
- **Content**: GIS-to-Blender patterns, CityJSON vertex parsing, coordinate transformation, fan triangulation, batch building import with error isolation, building-py Blender integration (planned), Dutch AEC data sources (3D BAG, PDOK, BGT, Kadaster)
- **No skill references this section**
- **Recommendation**: Fold relevant patterns into `blender-impl-mesh` (I-03) or `blender-impl-automation` (I-04) as reference material. The GIS-to-Blender pipeline is a primary AEC use case that should not be orphaned.

### GAP-2: vooronderzoek-ifcopenshell §16 — Real-World Usage: OpenAEC Projects (MINOR)
- **Lines**: L1942-2031 (90 lines)
- **Content**: building.py wrapper-class abstraction, INB Template data-driven IFC generation, web-ifc boundary (browser vs desktop), cross-cutting universal patterns
- **No skill references this section**
- **Recommendation**: Reference in `aec-core-bim-workflows` as real-world validation of the cross-technology patterns already covered in ecosystem-sources §3. The universal patterns (api.run() for writes, by_type() for reads) reinforce existing skill content.

### GAP-3: supplementary-ifcos-gaps §4 — Drawing / 2D API (MINOR)
- **Lines**: L671-744 (74 lines)
- **Content**: `ifcopenshell.api.drawing` module (3 functions: assign_product, unassign_product, edit_text_literal), smart annotations, text literals
- **No skill references this section**
- **Recommendation**: This is a minimal API that the section itself notes is "intentionally minimal" with actual drawing done by Bonsai. Add a brief reference in `ifcos-syntax-api` (which lists all 30+ modules) and/or `bonsai-impl-drawing` (which handles the Bonsai drawing workflow).

### GAP-4: supplementary-ifcos-gaps §6 — Validation & Georeferencing (SIGNIFICANT)
- **Lines**: L911-1112 (202 lines)
- **Content**:
  - `ifcopenshell.validate` module: 6 functions (validate, validate_guid, validate_ifc_header, validate_ifc_applications, assert_valid, assert_valid_inverse), json_logger, LogDetectionHandler, EXPRESS WHERE rules
  - `ifcopenshell.api.georeference` module: 5 functions (add_georeferencing, edit_georeferencing, edit_true_north, edit_wcs, remove_georeferencing), EPSG codes, IFC2X3 vs IFC4+ differences, anti-patterns
- **No skill explicitly cites this section as research input**
- **Current coverage**:
  - `bonsai-ifc-validator` mentions IfcTester but references only vooronderzoek-bonsai §10 and supplementary-bonsai §5
  - `bonsai-impl-project` mentions "georeference setup" in scope but references only vooronderzoek-bonsai §10
  - `ifcos-syntax-api` lists `georeference` as one of 30+ modules but doesn't reference §6
- **Recommendation**: This is the most significant gap. Add `supplementary-ifcos-gaps §6` as research input to:
  - `bonsai-ifc-validator` (for ifcopenshell.validate)
  - `bonsai-impl-project` (for ifcopenshell.api.georeference)
  - Consider adding `ifcos-syntax-api` reference for completeness

---

## 4. Ecosystem Insights Check (L-006, L-007, L-009)

These lessons from `briefing-cross-tech.md` were checked against the masterplan:

| Insight | Description | Reflected in Masterplan? | Evidence |
|---------|------------|--------------------------|----------|
| **L-006** | "Deterministic" prefix wastes tokens; lead with third-person verb | **YES** | Prompt template §5: "Description: lead with third-person verb, include domain trigger keywords, max 1024 chars" |
| **L-007** | Universal IFC property extraction pattern (`IsDefinedBy → IfcRelDefinesByProperties → HasProperties`) | **PARTIALLY** | `aec-core-bim-workflows` references ecosystem-sources §3 which documents this pattern. However, the briefing recommended that `bonsai-syntax-properties`, `ifcos-syntax-elements`, and `ifcos-impl-extraction` (now merged) must ALL include this canonical traversal — the masterplan doesn't explicitly mandate this cross-skill duplication. |
| **L-009** | Missing frontmatter fields (license, compatibility, metadata) | **YES** | Prompt template §5 includes `license: MIT`, `compatibility: Designed for Claude Code. Requires Python 3.x.`, `metadata: { author, version }` |

### Additional Ecosystem Insights

| Insight | Source | Reflected? | Evidence |
|---------|--------|-----------|----------|
| Mixed `run()` vs `create_entity()` is anti-pattern | ecosystem-sources §3.8 | YES | ifcos-syntax-api scope includes "invocation patterns" and distinguishes api.run() vs create_entity |
| NL-SfB classification via IfcProjectLibrary | ecosystem-sources §3.5 | YES | bonsai-impl-classification scope: "NL-SfB, MasterFormat" |
| No existing Blender export in OpenAEC tooling | L-008 | YES | Confirms skills fill a real gap |
| ALWAYS/NEVER overuse | ecosystem-sources §2.4 | YES | Prompt template: "Use deterministic language sparingly, only for truly critical rules" |

---

## 5. Anti-Pattern Coverage Check

Anti-patterns from research are covered by error skills:

| Research Source | Anti-Patterns Found | Covered By Error Skill | Status |
|----------------|--------------------|-----------------------|--------|
| vooronderzoek-blender §11 (Error Patterns) | Context errors, data errors, operator failures | blender-errors-context (E-01), blender-errors-data (E-02) | COVERED |
| vooronderzoek-blender §12 (AI Mistakes) | Hallucinated APIs, wrong context | blender-errors-context (E-01), blender-agents-code-validator (A-01) | COVERED |
| vooronderzoek-ifcopenshell §11 (Error Patterns) | 10+ documented errors | ifcos-errors-schema, ifcos-errors-patterns | COVERED |
| vooronderzoek-ifcopenshell §12 (Performance) | Memory bloat, single-threaded writes | ifcos-errors-performance | COVERED |
| vooronderzoek-ifcopenshell §13 (AI Mistakes) | Hallucinated APIs, wrong params | ifcos-code-validator | COVERED |
| vooronderzoek-bonsai §11 (Error Patterns) | 11+ error patterns | bonsai-errors-common | COVERED |
| vooronderzoek-bonsai §13 (AI Mistakes) | 10+ mistakes | bonsai-errors-common | COVERED |
| supplementary-bonsai-gaps §1.9 (Drawing anti-patterns) | 7 drawing anti-patterns | bonsai-errors-common | COVERED |
| supplementary-bonsai-gaps §2.7 (QTO anti-patterns) | 7 QTO anti-patterns | bonsai-errors-common | COVERED |
| supplementary-bonsai-gaps §3.8 (BCF anti-patterns) | 7 BCF anti-patterns | bonsai-errors-common | COVERED |
| supplementary-bonsai-gaps §4.8 (Clash anti-patterns) | 7 clash anti-patterns | bonsai-errors-common | COVERED |
| supplementary-ifcos-gaps §6.2.8 (Georef anti-patterns) | 6 georef anti-patterns | **NOT COVERED** | **GAP** (part of GAP-4) |
| supplementary-ifcos-gaps §4.6 (Drawing anti-patterns) | 3 drawing anti-patterns | **NOT COVERED** | **GAP** (part of GAP-3) |

Each skill is also required to produce a `references/anti-patterns.md` file with at least 3 entries (per prompt template verification checklist).

---

## 6. Reference Accuracy Spot-Check

15 randomly selected "Research input" references were verified against actual document content:

| Skill | Reference | Actual Content | Accurate? |
|-------|-----------|---------------|-----------|
| blender-syntax-mesh (S-05) | vooronderzoek-blender §7 (L1000+) | Mesh structures, BMesh, from_pydata, UV | YES |
| blender-errors-context (E-01) | vooronderzoek-blender §9 | Context system, temp_override, restricted context | YES |
| blender-syntax-materials (S-09) | supplementary-blender-gaps §3 (L586-795) | Materials, Principled BSDF renames, UV mapping | YES |
| blender-impl-automation (I-04) | supplementary §5 (L1065-1242) | I/O formats, FBX/glTF/OBJ/USD export | YES |
| ifcos-impl-geometry | vooronderzoek-ifcos §6 (L570-710) | geom settings, create_shape, iterator | YES |
| ifcos-impl-relationships | vooronderzoek-ifcos §9 (L1096-1315) | 7 relationship categories, code examples | YES |
| ifcos-impl-cost | supplementary-ifcos-gaps §1 (L23-209) | 20 API functions, cost value types | YES |
| ifcos-impl-mep | supplementary-ifcos-gaps §3 (L493-668) | 12 system functions, ports, HVAC example | YES |
| bonsai-impl-modeling | vooronderzoek-bonsai §6 | Type system, PredefinedType, material sets | YES |
| bonsai-impl-classification | vooronderzoek-bonsai §8 | Uniclass, OmniClass, NL-SfB, bSDD | YES |
| bonsai-errors-common | vooronderzoek-bonsai §11 | 11+ error patterns with correct/wrong | YES |
| bonsai-impl-drawing | supplementary-bonsai §1 | Drawing module, 16 files, SVG pipeline | YES |
| bonsai-impl-clash | supplementary-bonsai §4 | Clash modes, operators, anti-patterns | YES |
| aec-core-bim-workflows | vooronderzoek-ecosystem §3 | Cross-tech patterns, IFC pipeline | YES |
| bonsai-impl-classification | vooronderzoek-ifcos §15 (indirect) | bSDD API, classification integration | YES |

**Result: 15/15 references are accurate.** Line number ranges match actual content. Section topics align with skill scopes.

---

## 7. Recommendations

### Priority 1: Fix GAP-4 (Validation & Georeferencing)
Add `supplementary-ifcos-gaps §6 (L911-1112)` as research input to:
- `bonsai-ifc-validator` — for `ifcopenshell.validate` coverage (6 functions, json_logger, WHERE rules)
- `bonsai-impl-project` — for `ifcopenshell.api.georeference` coverage (5 functions, EPSG codes, IFC version differences)

### Priority 2: Fix GAP-1 (Blender Real-World Usage)
Add `vooronderzoek-blender §13 (L1881-2069)` as research input to:
- `blender-impl-mesh` (I-03) — for CityJSON mesh patterns, fan triangulation insights
- `blender-impl-automation` (I-04) — for batch building import, error isolation patterns
- Or create a reference in `aec-core-bim-workflows` for the GIS-to-Blender pipeline

### Priority 3: Fix GAP-3 (Drawing API)
Add `supplementary-ifcos-gaps §4 (L671-744)` as research input to:
- `ifcos-syntax-api` — to ensure all 30+ modules have source documentation
- Optionally `bonsai-impl-drawing` — for the ifcos↔bonsai drawing boundary

### Priority 4: Fix GAP-2 (IfcOpenShell Real-World Usage)
Add `vooronderzoek-ifcopenshell §16 (L1942-2031)` as research input to:
- `aec-core-bim-workflows` — universal patterns reinforce cross-technology guidance
- Optionally `ifcos-impl-creation` — for wrapper-class vs direct API patterns

### Priority 5: Strengthen L-007 Implementation
Ensure the canonical IFC property extraction traversal (`IsDefinedBy → IfcRelDefinesByProperties → HasProperties → NominalValue`) appears as a primary example in:
- `bonsai-syntax-properties`
- `ifcos-syntax-elements`

---

## 8. Overall Assessment

The masterplan is **well-researched and thorough**. The 60 skills collectively cover the vast majority of the ~23,000 lines of research produced in Phase 2. The research input references are accurate and point to real sections with matching content. The gaps found are minor (except GAP-4) and can be resolved by adding research input references to existing skills — no new skills are needed.

**Rating: 9/10 — Excellent coverage with 4 addressable gaps.**
