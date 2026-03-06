# Cross-Technology Briefing: Dependencies, Build Order & Batch Plan

**Date:** 2026-03-06
**Author:** interpret-cross-tech agent
**Purpose:** Backbone document for the masterplan — maps all skill dependencies, defines build order, and designs parallel batch strategy
**Status:** COMPLETE

---

## 1. Cross-Technology Skills

These skills span multiple technologies and belong in `skills/aec-cross-tech/`.

| Skill | Scope | Why Cross-Tech |
|-------|-------|----------------|
| `aec-core-ifc-fundamentals` | IFC standard, MVD, schema versions (IFC2x3/IFC4/IFC4.3), buildingSMART ecosystem, entity hierarchy, relationship types | Needed by both Bonsai and IfcOpenShell users; foundational BIM knowledge |
| `aec-core-python-runtime` | Blender embedded CPython restrictions, IfcOpenShell C++ bindings, threading limitations, memory management, undo system constraints | Every addon/script must respect these runtime rules |
| `aec-core-bim-workflows` | BIM workflow patterns, LOD/LOI, coordination, property extraction pattern (L-007), IFC creation pipeline (9-step), mesh generation pipeline | Patterns observed across all OpenAEC repos |
| `aec-workflow-orchestrator` | Auto-detect which skill package to load (Blender vs Bonsai vs IfcOpenShell), route user requests, multi-technology detection | Agent-level skill for Claude to select correct package |

**Note:** `aec-cross-sverchok-bonsai` (IfcSverchok bridge) is DEFERRED until Sverchok research is complete.

---

## 2. Dependency Graph

### 2.1 Cross-Tech Dependencies

```
aec-core-ifc-fundamentals → bonsai-syntax-elements
aec-core-ifc-fundamentals → bonsai-syntax-spatial
aec-core-ifc-fundamentals → bonsai-syntax-properties
aec-core-ifc-fundamentals → ifcos-core-schemas
aec-core-ifc-fundamentals → ifcos-core-concepts
aec-core-python-runtime   → blender-syntax-addons
aec-core-python-runtime   → blender-core-api
```

### 2.2 Blender Internal Dependencies

```
blender-core-api          → blender-syntax-operators
blender-core-api          → blender-syntax-properties
blender-core-api          → blender-syntax-panels
blender-core-api          → blender-syntax-data
blender-core-api          → blender-core-gpu
blender-core-versions     → blender-errors-version
blender-syntax-operators  → blender-impl-operators
blender-syntax-operators  → blender-errors-operators
blender-syntax-properties → blender-syntax-panels
blender-syntax-properties → blender-impl-addons
blender-syntax-addons     → blender-impl-addons
blender-syntax-mesh       → blender-syntax-modifiers
blender-syntax-mesh       → blender-impl-mesh
blender-syntax-nodes      → blender-impl-nodes
blender-syntax-animation  → blender-impl-animation
blender-syntax-rendering  → blender-impl-automation
blender-syntax-data       → blender-errors-data
blender-impl-operators    → blender-errors-context
blender-core-api          → blender-errors-context
blender-errors-context    → blender-code-validator (agent)
blender-core-versions     → blender-version-migrator (agent)
blender-errors-version    → blender-version-migrator (agent)
```

### 2.3 IfcOpenShell Internal Dependencies

```
ifcos-core-schemas        → ifcos-syntax-fileio
ifcos-core-concepts       → ifcos-syntax-api
ifcos-core-concepts       → ifcos-syntax-elements
ifcos-syntax-fileio       → ifcos-syntax-api
ifcos-syntax-fileio       → ifcos-syntax-elements
ifcos-syntax-api          → ifcos-syntax-geometry
ifcos-syntax-api          → ifcos-impl-creation
ifcos-syntax-elements     → ifcos-syntax-util
ifcos-syntax-elements     → ifcos-impl-extraction
ifcos-syntax-api          → ifcos-impl-modification
ifcos-syntax-geometry     → ifcos-errors-geometry
ifcos-core-schemas        → ifcos-errors-schema
ifcos-impl-creation       → ifcos-code-validator (agent)
ifcos-errors-schema       → ifcos-code-validator (agent)
```

### 2.4 Bonsai Internal Dependencies

```
bonsai-core-architecture  → bonsai-syntax-elements
bonsai-core-architecture  → bonsai-syntax-spatial
bonsai-syntax-elements    → bonsai-syntax-properties
bonsai-syntax-elements    → bonsai-syntax-geometry
bonsai-syntax-spatial     → bonsai-impl-project
bonsai-syntax-elements    → bonsai-impl-modeling
bonsai-syntax-properties  → bonsai-impl-classification
bonsai-impl-modeling      → bonsai-impl-export
bonsai-syntax-elements    → bonsai-errors-ifc
bonsai-syntax-geometry    → bonsai-errors-geometry
bonsai-errors-ifc         → bonsai-ifc-validator (agent)
```

### 2.5 Cross-Package Dependencies (Blender → Bonsai)

```
blender-syntax-operators  → bonsai-syntax-elements (Bonsai operators extend Blender operator system)
blender-syntax-properties → bonsai-syntax-properties (Bonsai properties use bpy.props)
blender-syntax-panels     → bonsai-core-architecture (Bonsai UI uses Blender panel system)
blender-syntax-addons     → bonsai-core-architecture (Bonsai is a Blender addon)
blender-core-api          → bonsai-core-architecture (Bonsai depends on bpy)
```

### 2.6 Cross-Package Dependencies (IfcOpenShell → Bonsai)

```
ifcos-core-schemas        → bonsai-syntax-elements (Bonsai creates IFC entities)
ifcos-syntax-api          → bonsai-syntax-elements (Bonsai delegates to ifcopenshell.api)
ifcos-syntax-elements     → bonsai-syntax-properties (property traversal pattern)
ifcos-syntax-geometry     → bonsai-syntax-geometry (geometry representations)
```

---

## 3. Build Order

Based on D-008 (Blender → Bonsai → IfcOpenShell) and the dependency graph above.

### Phase 1: Foundation (no dependencies)

| # | Skill | Tech | Category | Rationale |
|---|-------|------|----------|-----------|
| 1 | `aec-core-ifc-fundamentals` | cross-tech | core | IFC knowledge needed by both Bonsai and IfcOpenShell |
| 2 | `aec-core-python-runtime` | cross-tech | core | Runtime constraints affect all code |
| 3 | `blender-core-api` | blender | core | Foundation for all Blender skills |
| 4 | `blender-core-versions` | blender | core | Version matrix needed by all version-aware skills |
| 5 | `ifcos-core-schemas` | ifcopenshell | core | IFC schema knowledge, independent reference |
| 6 | `ifcos-core-concepts` | ifcopenshell | core | IFC relationship/placement concepts |

### Phase 2: Syntax Layer (depends on Phase 1 core skills)

| # | Skill | Tech | Depends On |
|---|-------|------|------------|
| 7 | `blender-syntax-operators` | blender | blender-core-api |
| 8 | `blender-syntax-properties` | blender | blender-core-api |
| 9 | `blender-syntax-addons` | blender | aec-core-python-runtime |
| 10 | `blender-syntax-mesh` | blender | (standalone) |
| 11 | `blender-syntax-data` | blender | blender-core-api |
| 12 | `ifcos-syntax-fileio` | ifcopenshell | ifcos-core-schemas |
| 13 | `ifcos-syntax-elements` | ifcopenshell | ifcos-core-concepts, ifcos-syntax-fileio |
| 14 | `ifcos-syntax-api` | ifcopenshell | ifcos-core-concepts, ifcos-syntax-fileio |

### Phase 3: Extended Syntax + First Impl (depends on Phase 2)

| # | Skill | Tech | Depends On |
|---|-------|------|------------|
| 15 | `blender-syntax-panels` | blender | blender-syntax-properties |
| 16 | `blender-syntax-modifiers` | blender | blender-syntax-mesh |
| 17 | `blender-syntax-nodes` | blender | (standalone) |
| 18 | `blender-syntax-animation` | blender | (standalone) |
| 19 | `blender-syntax-rendering` | blender | (standalone) |
| 20 | `blender-core-gpu` | blender | blender-core-api |
| 21 | `ifcos-syntax-geometry` | ifcopenshell | ifcos-syntax-api |
| 22 | `ifcos-syntax-util` | ifcopenshell | ifcos-syntax-elements |
| 23 | `bonsai-core-architecture` | bonsai | blender-core-api, blender-syntax-addons |

### Phase 4: Implementation Skills (depends on Phases 2–3)

| # | Skill | Tech | Depends On |
|---|-------|------|------------|
| 24 | `blender-impl-operators` | blender | blender-syntax-operators |
| 25 | `blender-impl-addons` | blender | blender-syntax-addons, blender-syntax-properties |
| 26 | `blender-impl-mesh` | blender | blender-syntax-mesh |
| 27 | `blender-impl-nodes` | blender | blender-syntax-nodes |
| 28 | `blender-impl-animation` | blender | blender-syntax-animation |
| 29 | `blender-impl-automation` | blender | blender-syntax-rendering |
| 30 | `ifcos-impl-creation` | ifcopenshell | ifcos-syntax-api |
| 31 | `ifcos-impl-extraction` | ifcopenshell | ifcos-syntax-elements, ifcos-syntax-util |
| 32 | `ifcos-impl-modification` | ifcopenshell | ifcos-syntax-api |
| 33 | `bonsai-syntax-elements` | bonsai | bonsai-core-architecture, aec-core-ifc-fundamentals |
| 34 | `bonsai-syntax-spatial` | bonsai | bonsai-core-architecture, aec-core-ifc-fundamentals |
| 35 | `bonsai-syntax-properties` | bonsai | bonsai-syntax-elements |
| 36 | `bonsai-syntax-geometry` | bonsai | bonsai-syntax-elements |

### Phase 5: Error Skills + Bonsai Impl (depends on Phases 3–4)

| # | Skill | Tech | Depends On |
|---|-------|------|------------|
| 37 | `blender-errors-context` | blender | blender-core-api, blender-impl-operators |
| 38 | `blender-errors-operators` | blender | blender-syntax-operators |
| 39 | `blender-errors-data` | blender | blender-syntax-data |
| 40 | `blender-errors-version` | blender | blender-core-versions |
| 41 | `ifcos-errors-schema` | ifcopenshell | ifcos-core-schemas |
| 42 | `ifcos-errors-geometry` | ifcopenshell | ifcos-syntax-geometry |
| 43 | `bonsai-impl-project` | bonsai | bonsai-syntax-spatial |
| 44 | `bonsai-impl-modeling` | bonsai | bonsai-syntax-elements |
| 45 | `bonsai-impl-classification` | bonsai | bonsai-syntax-properties |
| 46 | `bonsai-impl-export` | bonsai | bonsai-impl-modeling |

### Phase 6: Error Skills (Bonsai) + Cross-Tech Workflows

| # | Skill | Tech | Depends On |
|---|-------|------|------------|
| 47 | `bonsai-errors-ifc` | bonsai | bonsai-syntax-elements |
| 48 | `bonsai-errors-geometry` | bonsai | bonsai-syntax-geometry |
| 49 | `aec-core-bim-workflows` | cross-tech | aec-core-ifc-fundamentals, ifcos-impl-creation |

### Phase 7: Agent Skills (depend on everything above)

| # | Skill | Tech | Depends On |
|---|-------|------|------------|
| 50 | `blender-code-validator` | blender | blender-errors-context, blender-errors-version |
| 51 | `blender-version-migrator` | blender | blender-core-versions, blender-errors-version |
| 52 | `ifcos-code-validator` | ifcopenshell | ifcos-errors-schema, ifcos-impl-creation |
| 53 | `bonsai-ifc-validator` | bonsai | bonsai-errors-ifc |
| 54 | `aec-workflow-orchestrator` | cross-tech | all above |

---

## 4. Parallel Batch Plan

Rules (from CLAUDE.md):
- 3–5 agents per batch
- NEVER two agents on same file/directory
- Quality gate after every batch
- Each skill = unique directory → no file conflicts between different skills

### Batch 1: Foundation Core (4 agents)

| Agent | Skill | Directory | Notes |
|-------|-------|-----------|-------|
| A1 | `aec-core-ifc-fundamentals` | skills/aec-cross-tech/core/aec-core-ifc-fundamentals/ | IFC standard overview |
| A2 | `aec-core-python-runtime` | skills/aec-cross-tech/core/aec-core-python-runtime/ | Runtime constraints |
| A3 | `blender-core-api` | skills/blender/core/blender-core-api/ | Blender API foundation |
| A4 | `blender-core-versions` | skills/blender/core/blender-core-versions/ | Version matrix |

**Quality gate:** Verify all 4 skills have valid frontmatter, <500 lines, English-only, deterministic language.

### Batch 2: IfcOpenShell Core + Blender Syntax A (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `ifcos-core-schemas` | skills/ifcopenshell/core/ifcos-core-schemas/ |
| A2 | `ifcos-core-concepts` | skills/ifcopenshell/core/ifcos-core-concepts/ |
| A3 | `blender-syntax-operators` | skills/blender/syntax/blender-syntax-operators/ |
| A4 | `blender-syntax-properties` | skills/blender/syntax/blender-syntax-properties/ |
| A5 | `blender-syntax-mesh` | skills/blender/syntax/blender-syntax-mesh/ |

**Quality gate:** Verify cross-references to Batch 1 skills are correct.

### Batch 3: Blender Syntax B + IfcOpenShell Syntax A (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-syntax-addons` | skills/blender/syntax/blender-syntax-addons/ |
| A2 | `blender-syntax-data` | skills/blender/syntax/blender-syntax-data/ |
| A3 | `blender-syntax-nodes` | skills/blender/syntax/blender-syntax-nodes/ |
| A4 | `ifcos-syntax-fileio` | skills/ifcopenshell/syntax/ifcos-syntax-fileio/ |
| A5 | `ifcos-syntax-api` | skills/ifcopenshell/syntax/ifcos-syntax-api/ |

### Batch 4: Extended Syntax (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-syntax-panels` | skills/blender/syntax/blender-syntax-panels/ |
| A2 | `blender-syntax-modifiers` | skills/blender/syntax/blender-syntax-modifiers/ |
| A3 | `blender-syntax-animation` | skills/blender/syntax/blender-syntax-animation/ |
| A4 | `blender-syntax-rendering` | skills/blender/syntax/blender-syntax-rendering/ |
| A5 | `ifcos-syntax-elements` | skills/ifcopenshell/syntax/ifcos-syntax-elements/ |

### Batch 5: Remaining Syntax + Bonsai Core (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-core-gpu` | skills/blender/core/blender-core-gpu/ |
| A2 | `ifcos-syntax-geometry` | skills/ifcopenshell/syntax/ifcos-syntax-geometry/ |
| A3 | `ifcos-syntax-util` | skills/ifcopenshell/syntax/ifcos-syntax-util/ |
| A4 | `bonsai-core-architecture` | skills/bonsai/core/bonsai-core-architecture/ |
| A5 | `aec-core-bim-workflows` | skills/aec-cross-tech/core/aec-core-bim-workflows/ |

### Batch 6: Blender Implementation (4 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-impl-operators` | skills/blender/impl/blender-impl-operators/ |
| A2 | `blender-impl-addons` | skills/blender/impl/blender-impl-addons/ |
| A3 | `blender-impl-mesh` | skills/blender/impl/blender-impl-mesh/ |
| A4 | `blender-impl-nodes` | skills/blender/impl/blender-impl-nodes/ |

### Batch 7: Remaining Blender Impl + IfcOpenShell Impl (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-impl-animation` | skills/blender/impl/blender-impl-animation/ |
| A2 | `blender-impl-automation` | skills/blender/impl/blender-impl-automation/ |
| A3 | `ifcos-impl-creation` | skills/ifcopenshell/impl/ifcos-impl-creation/ |
| A4 | `ifcos-impl-extraction` | skills/ifcopenshell/impl/ifcos-impl-extraction/ |
| A5 | `ifcos-impl-modification` | skills/ifcopenshell/impl/ifcos-impl-modification/ |

### Batch 8: Bonsai Syntax (4 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `bonsai-syntax-elements` | skills/bonsai/syntax/bonsai-syntax-elements/ |
| A2 | `bonsai-syntax-spatial` | skills/bonsai/syntax/bonsai-syntax-spatial/ |
| A3 | `bonsai-syntax-properties` | skills/bonsai/syntax/bonsai-syntax-properties/ |
| A4 | `bonsai-syntax-geometry` | skills/bonsai/syntax/bonsai-syntax-geometry/ |

### Batch 9: Error Skills (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `blender-errors-context` | skills/blender/errors/blender-errors-context/ |
| A2 | `blender-errors-operators` | skills/blender/errors/blender-errors-operators/ |
| A3 | `blender-errors-data` | skills/blender/errors/blender-errors-data/ |
| A4 | `blender-errors-version` | skills/blender/errors/blender-errors-version/ |
| A5 | `ifcos-errors-schema` | skills/ifcopenshell/errors/ifcos-errors-schema/ |

### Batch 10: Bonsai Impl + Remaining Errors (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `ifcos-errors-geometry` | skills/ifcopenshell/errors/ifcos-errors-geometry/ |
| A2 | `bonsai-impl-project` | skills/bonsai/impl/bonsai-impl-project/ |
| A3 | `bonsai-impl-modeling` | skills/bonsai/impl/bonsai-impl-modeling/ |
| A4 | `bonsai-impl-classification` | skills/bonsai/impl/bonsai-impl-classification/ |
| A5 | `bonsai-impl-export` | skills/bonsai/impl/bonsai-impl-export/ |

### Batch 11: Bonsai Errors + Agent Skills (5 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `bonsai-errors-ifc` | skills/bonsai/errors/bonsai-errors-ifc/ |
| A2 | `bonsai-errors-geometry` | skills/bonsai/errors/bonsai-errors-geometry/ |
| A3 | `blender-code-validator` | skills/blender/agents/blender-code-validator/ |
| A4 | `blender-version-migrator` | skills/blender/agents/blender-version-migrator/ |
| A5 | `ifcos-code-validator` | skills/ifcopenshell/agents/ifcos-code-validator/ |

### Batch 12: Final Agent Skills (2 agents)

| Agent | Skill | Directory |
|-------|-------|-----------|
| A1 | `bonsai-ifc-validator` | skills/bonsai/agents/bonsai-ifc-validator/ |
| A2 | `aec-workflow-orchestrator` | skills/aec-cross-tech/agents/aec-workflow-orchestrator/ |

**Total: 12 batches, 54 skills, 3–5 agents per batch.**

### Batch Summary Table

| Batch | Skills | Agents | Focus |
|-------|--------|--------|-------|
| 1 | 4 | 4 | Foundation core (cross-tech + Blender) |
| 2 | 5 | 5 | IfcOpenShell core + Blender syntax A |
| 3 | 5 | 5 | Blender syntax B + IfcOpenShell syntax A |
| 4 | 5 | 5 | Extended syntax (Blender + IfcOpenShell) |
| 5 | 5 | 5 | GPU + remaining syntax + Bonsai core |
| 6 | 4 | 4 | Blender implementation |
| 7 | 5 | 5 | Blender impl + IfcOpenShell impl |
| 8 | 4 | 4 | Bonsai syntax |
| 9 | 5 | 5 | Error skills (Blender + IfcOpenShell) |
| 10 | 5 | 5 | Bonsai impl + remaining errors |
| 11 | 5 | 5 | Bonsai errors + agent skills |
| 12 | 2 | 2 | Final agent skills |

---

## 5. Ecosystem Insights for Masterplan

### From LESSONS.md and vooronderzoek-ecosystem-sources.md

| Lesson | Impact | Required Action |
|--------|--------|-----------------|
| **L-006**: "Deterministic" prefix wastes tokens | All skill descriptions | Remove prefix. Lead with third-person action verbs. Include domain trigger keywords. Max 1024 chars. |
| **L-007**: Universal IFC property extraction pattern | `bonsai-syntax-properties`, `ifcos-syntax-elements`, `ifcos-impl-extraction` | All three skills MUST include the canonical `IsDefinedBy → IfcRelDefinesByProperties → HasProperties` traversal chain as primary example |
| **L-009**: Missing frontmatter fields | All skills | Update template to include `license: MIT`, `compatibility`, `metadata` fields before Phase 5 |

### Additional Ecosystem Insights

| Finding | Source | Impact |
|---------|--------|--------|
| Description-driven discovery uses 2% context budget | ecosystem-sources §2.2 | With 54 skills, descriptions must be concise (~300 chars each) to fit within 16k char budget |
| 3-level progressive disclosure (metadata → instructions → resources) | ecosystem-sources §2.2 | SKILL.md body = Level 2 (<5000 tokens). References = Level 3. |
| IFC creation follows 9-step pipeline | ecosystem-sources §3.2 | `ifcos-impl-creation` must use this as canonical workflow |
| Mesh pipeline (verts+faces) is universal across platforms | ecosystem-sources §3.3 | `blender-impl-mesh` should reference cross-platform equivalents |
| Mixed `run()` vs `create_entity()` is an anti-pattern | ecosystem-sources §3.8 | `ifcos-syntax-api` must clarify when to use which |
| NL-SfB classification via IfcProjectLibrary files | ecosystem-sources §3.5 | `bonsai-impl-classification` should document both Dutch (NL-SfB) and international (Uniclass, OmniClass) |
| No existing Blender export in OpenAEC tooling | L-008 | Confirms our skills fill a real unserved gap |
| ALWAYS/NEVER overuse conflicts with official guidance | ecosystem-sources §2.4 | Use deterministic language sparingly, only for truly critical rules. Add "because [reason]". |

### Template Update Required

Before Phase 5 (skill creation), update the SKILL.md template:

```yaml
---
name: {tech}-{category}-{topic}
description: "[Third-person verb] [capability]. Use when [trigger scenarios with domain keywords]."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---
```

---

## 6. Scope Boundary Decisions

### INCLUDE (AEC-relevant)

| Domain | Technology | Justification |
|--------|-----------|---------------|
| Mesh/BMesh modeling | Blender | Core geometry operations for AEC |
| Operators/Properties/Panels | Blender | Foundation for all Blender addons |
| Addon/Extension development | Blender | Needed to build BIM tools |
| Node systems (Geometry Nodes) | Blender | Parametric design in AEC |
| Animation (keyframes, drivers) | Blender | Construction sequencing, 4D BIM visualization |
| Rendering (EEVEE/Cycles config) | Blender | Architectural visualization, batch rendering |
| Materials/Shading | Blender | Arch-viz material setup |
| I/O formats (FBX, glTF, USD) | Blender | Data exchange with other AEC tools |
| GPU drawing | Blender | Custom overlays for BIM visualization |
| Data management (collections, assets) | Blender | Project organization |
| Context system + errors | Blender | #1 AI mistake in Blender scripting |
| Version migration (3.x→4.x→5.x) | Blender | Critical for production code |
| All core BIM domains | Bonsai | Primary Bonsai functionality |
| All API modules | IfcOpenShell | Complete IFC programmatic access |
| File I/O, Element CRUD, Spatial | IfcOpenShell | Core operations |
| Cost, Scheduling, MEP | IfcOpenShell | Full BIM discipline coverage |

### EXCLUDE (out of scope)

| Domain | Technology | Reason |
|--------|-----------|--------|
| Sculpting | Blender | Not AEC-relevant, artistic workflow |
| Video Editing (VSE) | Blender | Not AEC-relevant |
| Motion Tracking | Blender | Not AEC-relevant |
| Grease Pencil 2D animation | Blender | Not AEC-relevant (GP for annotations is covered via Bonsai drawing) |
| Particle simulation | Blender | Not AEC-relevant (physics simulation) |
| Cloth/Fluid simulation | Blender | Not AEC-relevant |
| Audio (aud module) | Blender | Not AEC-relevant |
| Freestyle rendering | Blender | Niche, not priority for AEC |
| Brickschema (IoT) | Bonsai | Too specialized, very few users |
| Structural analysis | IfcOpenShell/Bonsai | Requires domain-specific expertise beyond skill scope |
| Facility management | Bonsai | Limited API coverage, not priority |

### DEFER (later phase)

| Domain | Technology | Conditions for Inclusion |
|--------|-----------|--------------------------|
| Sverchok (all) | Sverchok | After Blender/Bonsai/IfcOpenShell packages are complete and validated |
| IfcSverchok bridge | Sverchok + IfcOpenShell | After Sverchok research is complete; requires both Sverchok and IfcOpenShell skills |
| Georeferencing | Bonsai/IfcOpenShell | Include if research confirms sufficient API coverage |
| Drawing/2D | Bonsai | Include if research confirms stable API |

---

## Appendix: Skill Count Summary

| Package | Syntax | Impl | Errors | Core | Agents | Total |
|---------|--------|------|--------|------|--------|-------|
| Blender | 10 | 6 | 4 | 3 | 2 | **25** |
| Bonsai | 4 | 4 | 2 | 1 | 1 | **12** |
| IfcOpenShell | 5 | 3 | 2 | 2 | 1 | **13** |
| Cross-tech | — | — | — | 3 | 1 | **4** |
| **Total** | **19** | **13** | **8** | **9** | **5** | **54** |

**Sverchok (deferred):** 3 syntax + 2 impl + 1 errors + 1 core + 1 agent = 8 skills (later phase)

---

*Generated by interpret-cross-tech agent, 2026-03-06*
