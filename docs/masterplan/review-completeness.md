# Completeness Review — Masterplan v2026-03-06

**Reviewer**: review-completeness agent
**Date**: 2026-03-06
**Document reviewed**: docs/masterplan/masterplan.md (1236 lines)
**Reference documents**: scope-analysis.md, briefing-blender.md, briefing-ifcopenshell.md, briefing-bonsai.md, briefing-cross-tech.md, SOURCES.md

---

## Check 1: All Skills from Briefings Represented in Masterplan

**Verdict: PASS**

All skills from all four briefings are accounted for in the masterplan — either kept as-is, or explicitly merged/removed with documented rationale.

### Blender (briefing-blender.md → masterplan)

| Briefing Skill | Masterplan Status |
|---|---|
| 26 skills (C-01..C-04, S-01..S-11, I-01..I-06, E-01..E-03, A-01..A-02) | All 26 present ✓ |
| blender-errors-operators (proposed merge into E-01) | Merged into blender-errors-context — documented in Key Decisions ✓ |
| blender-syntax-materials (proposed addition) | Added as S-09 ✓ |
| blender-core-runtime (proposed addition) | Added as C-04 ✓ |

### IfcOpenShell (briefing-ifcopenshell.md → masterplan)

| Briefing Skill | Masterplan Status |
|---|---|
| 18 skills (all listed in briefing §1) | All 18 present ✓ |
| ifcos-syntax-geometry | Merged into ifcos-impl-geometry — documented in Key Decisions ✓ |
| ifcos-impl-extraction | Merged into ifcos-syntax-elements + ifcos-syntax-util — documented ✓ |
| ifcos-impl-modification | Merged into ifcos-impl-creation — documented ✓ |
| 5 new skills (cost, sequence, mep, profiles, runtime) | All added ✓ |

### Bonsai (briefing-bonsai.md → masterplan)

| Briefing Skill | Masterplan Status |
|---|---|
| 14 skills (all listed in briefing §1) | All 14 present ✓ |
| bonsai-impl-export (raw masterplan proposal) | Merged into bonsai-impl-project — documented ✓ |
| bonsai-errors-ifc + bonsai-errors-geometry | Merged into bonsai-errors-common — documented ✓ |

### Cross-Tech (briefing-cross-tech.md → masterplan)

| Briefing Skill | Masterplan Status |
|---|---|
| aec-core-ifc-fundamentals | Removed — redundant with ifcos-core-concepts. Documented in Appendix A ✓ |
| aec-core-python-runtime | Removed — redundant with per-tech runtime skills. Documented in Appendix A ✓ |
| aec-core-bim-workflows | Kept ✓ |
| aec-workflow-orchestrator | Kept ✓ |
| aec-cross-sverchok-bonsai | Deferred — documented in Scope Boundaries ✓ |

---

## Check 2: Every Skill Has All Required Fields

**Verdict: PASS**

Required fields: scope, API surface, SOURCES.md URLs, research input, version coverage, dependencies, complexity.

All 60 skills in Section 2 (Skill Inventory) were checked. Every skill has:

| Field | Present in all 60? |
|---|---|
| Category | ✓ |
| Scope | ✓ |
| Key API surface | ✓ |
| SOURCES.md URLs | ✓ |
| Research input | ✓ |
| Version coverage | ✓ |
| Dependencies | ✓ |
| Complexity (S/M/L) | ✓ |

Additionally, Section 5 (Per-Skill Prompts) provides a complete parameter table for all 60 skills with output directories, research docs, source URLs, version coverage, and dependencies — matching the Section 2 definitions.

---

## Check 3: SOURCES.md URLs Spot-Check (10 Random Skills)

**Verdict: PASS**

10 skills were randomly selected and their SOURCES.md URLs were verified against both Appendix B (Full Source URL Reference) and the actual SOURCES.md file.

| # | Skill | URL(s) in Skill Definition | In Appendix B? | In SOURCES.md? |
|---|---|---|---|---|
| 1 | blender-core-api | `https://docs.blender.org/api/current/` | ✓ | ✓ |
| 2 | blender-syntax-addons | `https://developer.blender.org/docs/handbook/extensions/` + 3 more | ✓ | ✓ |
| 3 | ifcos-syntax-fileio | `https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html` | ✓ | ✓ |
| 4 | ifcos-impl-creation | `https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html`, `https://academy.ifcopenshell.org/posts/creating-a-simple-wall-with-property-set-and-quantity-information/` | ✓ | ✓ |
| 5 | ifcos-errors-schema | `https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/`, `https://ifc43-docs.standards.buildingsmart.org/`, `https://docs.ifcopenshell.org/ifcpatch.html` | ✓ | ✓ |
| 6 | ifcos-impl-profiles | `https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/profile/index.html` | ✓ | ✓ |
| 7 | bonsai-core-architecture | `https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai` | ✓ | ✓ |
| 8 | bonsai-impl-clash | Bonsai source + `https://docs.ifcopenshell.org/ifcclash.html` | ✓ | ✓ |
| 9 | blender-core-versions | 10 release note URLs (4.0-5.1) + compatibility + changelog | ✓ | ✓ |
| 10 | aec-core-bim-workflows | `https://docs.ifcopenshell.org/`, `https://docs.bonsaibim.org/`, `https://docs.blender.org/api/current/` | ✓ | ✓ |

All 10 spot-checked skills have URLs that match SOURCES.md approved sources.

---

## Check 4: Dependency Graph Covers All Cross-Skill Dependencies

**Verdict: FAIL (minor)**

The dependency graph in Section 3 is a **simplified visualization** that omits several cross-package dependencies listed in the skill definitions (Section 2). While the batch plan correctly respects all dependencies, the graph itself is incomplete.

### Missing Dependencies in Graph

| Skill | Dependency in Skill Definition | In Dependency Graph? |
|---|---|---|
| bonsai-impl-drawing | `blender-core-gpu` | ❌ Not shown |
| bonsai-impl-qto | `blender-syntax-mesh` | ❌ Not shown |
| bonsai-syntax-elements | `ifcos-syntax-elements` | ❌ Only shows `ifcos-syntax-api` |
| bonsai-impl-modeling | `ifcos-syntax-api` | ❌ Only shows `bonsai-syntax-*` |
| bonsai-impl-project | `bonsai-syntax-elements`, `ifcos-syntax-fileio` | ❌ Only shows `bonsai-core-arch` |
| bonsai-impl-classification | `ifcos-syntax-api` | ❌ Only shows `bonsai-syntax-elements` |

### Impact Assessment

These omissions do NOT affect the batch plan — all dependency ordering is respected in the batch assignments. The graph is a visualization aid, not the authoritative dependency source. However, the inconsistency could mislead future agents.

### Recommendation

Add missing cross-package dependencies to the graph, especially the Bonsai→Blender edges (`blender-core-gpu`, `blender-syntax-mesh`) and the Bonsai→IfcOpenShell edges that are implicit in the `bonsai-syntax-*` wildcard.

---

## Check 5: Batch Plan Covers All 60 Skills

**Verdict: PASS (with minor typo)**

### Skill Count per Batch

| Batch | Skills | Count |
|---|---|---|
| 1 | blender-core-api, blender-core-versions, ifcos-core-concepts, ifcos-syntax-fileio | 4 |
| 2 | blender-core-gpu, blender-core-runtime, blender-syntax-operators, ifcos-syntax-api, ifcos-syntax-elements | 5 |
| 3 | blender-syntax-properties, blender-syntax-mesh, blender-syntax-nodes, ifcos-syntax-util, ifcos-core-runtime | 5 |
| 4 | blender-syntax-panels, blender-syntax-addons, blender-syntax-animation, blender-syntax-materials, blender-syntax-data | 5 |
| 5 | blender-syntax-modifiers, blender-syntax-rendering, ifcos-impl-creation, ifcos-impl-materials, ifcos-impl-relationships | 5 |
| 6 | blender-impl-operators, blender-impl-addons, blender-impl-mesh, ifcos-impl-geometry, ifcos-impl-cost | 5 |
| 7 | blender-impl-nodes, blender-impl-animation, blender-impl-automation, ifcos-impl-sequence, ifcos-impl-profiles | 5 |
| 8 | ifcos-impl-mep, ifcos-errors-schema, ifcos-errors-patterns, ifcos-errors-performance | 4 |
| 9 | blender-errors-context, blender-errors-data, blender-errors-version, bonsai-core-architecture, aec-core-bim-workflows | 5 |
| 10 | bonsai-syntax-elements, bonsai-syntax-spatial, bonsai-syntax-properties, bonsai-syntax-geometry | 4 |
| 11 | bonsai-impl-project, bonsai-impl-modeling, bonsai-impl-classification, bonsai-impl-drawing, bonsai-impl-qto | 5 |
| 12 | bonsai-impl-bcf, bonsai-impl-clash, bonsai-errors-common, ifcos-code-validator | 4 |
| 13 | blender-agents-code-validator, blender-agents-version-migrator, bonsai-ifc-validator, aec-workflow-orchestrator | 4 |
| **Total** | | **60** ✓ |

All 60 skills are accounted for. No duplicates, no omissions.

### Minor Typo

The Batch Summary table (line 938) states **61 agents** in the total row. The correct count is **60** (4+5+5+5+5+5+5+4+5+4+5+4+4 = 60). This is a typo.

### Dependency Order Verification

All batch assignments respect dependencies. Verified examples:
- bonsai-core-architecture (Batch 9) depends on blender-core-api (Batch 1), blender-syntax-operators (Batch 2), blender-syntax-addons (Batch 4) ✓
- bonsai-impl-drawing (Batch 11) depends on blender-core-gpu (Batch 2) ✓
- ifcos-impl-mep (Batch 8) depends on ifcos-impl-relationships (Batch 5) ✓
- aec-workflow-orchestrator (Batch 13) depends on all skills ✓

---

## Check 6: Skills Dropped Without Explanation

**Verdict: PASS**

Every skill that was proposed in a briefing but not present in the final masterplan has an explicit explanation:

| Dropped/Merged Skill | Source Briefing | Explanation Location |
|---|---|---|
| aec-core-ifc-fundamentals | cross-tech | Appendix A, line 1174: redundant with ifcos-core-concepts |
| aec-core-python-runtime | cross-tech | Appendix A, line 1174: redundant with per-tech runtime skills |
| ifcos-syntax-geometry | ifcopenshell | Key Decisions table, line 52: merged into ifcos-impl-geometry |
| ifcos-impl-extraction | ifcopenshell | Key Decisions table, line 53: merged into ifcos-syntax-elements + ifcos-syntax-util |
| ifcos-impl-modification | ifcopenshell | Key Decisions table, line 54: merged into ifcos-impl-creation |
| bonsai-impl-export | bonsai/cross-tech | Key Decisions table, line 55: merged into bonsai-impl-project |
| bonsai-errors-ifc | bonsai/cross-tech | Key Decisions table, line 56: merged into bonsai-errors-common |
| bonsai-errors-geometry | bonsai/cross-tech | Key Decisions table, line 56: merged into bonsai-errors-common |
| blender-errors-operators | blender | Key Decisions table, line 57: merged into blender-errors-context |
| aec-cross-sverchok-bonsai | cross-tech | Scope Boundaries DEFERRED section, line 44-45 |

No unexplained drops.

---

## Check 7: Scope Boundary Completeness (vs. scope-analysis.md)

**Verdict: PASS**

### Blender API Surface Coverage

| scope-analysis.md Domain | Masterplan Skill(s) | Covered? |
|---|---|---|
| bpy.data (30+ collections) | blender-core-api, blender-syntax-data | ✓ |
| bpy.ops (50+ categories) | blender-syntax-operators, blender-impl-operators | ✓ |
| bpy.types (Operator, Panel, etc.) | blender-syntax-operators, panels, properties | ✓ |
| bpy.props | blender-syntax-properties | ✓ |
| bmesh | blender-syntax-mesh | ✓ |
| gpu module | blender-core-gpu | ✓ |
| mathutils | blender-core-runtime | ✓ |
| Nodes (Geometry, Shader, Compositor) | blender-syntax-nodes, blender-impl-nodes | ✓ |
| Animation/FCurve/Drivers | blender-syntax-animation, blender-impl-animation | ✓ |
| Materials/Shaders | blender-syntax-materials | ✓ |
| Rendering (EEVEE/Cycles) | blender-syntax-rendering | ✓ |
| I/O formats (FBX/glTF/USD/OBJ/STL) | blender-impl-automation | ✓ |
| Extensions/Addons | blender-syntax-addons, blender-impl-addons | ✓ |
| Sculpting, Video Editing, Motion Tracking | OUT of scope | ✓ (explicitly excluded) |
| Particle/Cloth/Fluid simulation | OUT of scope | ✓ (explicitly excluded) |
| Grease Pencil 2D, Audio | OUT of scope | ✓ (explicitly excluded) |

### IfcOpenShell API Surface Coverage

| scope-analysis.md Domain | Masterplan Skill(s) | Covered? |
|---|---|---|
| 30+ API sub-modules (root, spatial, aggregate, geometry, type, pset, material, context, unit, owner, classification, cost, sequence, system, group, void, boundary, structural, document, drawing, nest, layer, profile, style, constraint, resource, feature, alignment, georeference, project, attribute, pset_template) | ifcos-syntax-api (all listed in scope) + 8 dedicated impl skills | ✓ |
| 20+ util sub-modules | ifcos-syntax-util | ✓ |
| ifcopenshell.geom (geometry processing) | ifcos-impl-geometry | ✓ |
| IFC schemas (IFC2X3, IFC4, IFC4X3) | ifcos-core-concepts, ifcos-errors-schema | ✓ |
| Structural analysis | OUT of scope | ✓ (explicitly excluded) |

### Bonsai Module Coverage

| scope-analysis.md Domain | Masterplan Skill(s) | Covered? |
|---|---|---|
| Core architecture (IfcStore, Tool/Core/UI) | bonsai-core-architecture | ✓ |
| Modeling (walls, slabs, columns, beams) | bonsai-impl-modeling | ✓ |
| Spatial structure | bonsai-syntax-spatial | ✓ |
| Property sets / quantities | bonsai-syntax-properties, bonsai-impl-qto | ✓ |
| Classification | bonsai-impl-classification | ✓ |
| Drawing/documentation | bonsai-impl-drawing | ✓ |
| BCF collaboration | bonsai-impl-bcf | ✓ |
| Clash detection | bonsai-impl-clash | ✓ |
| Brickschema, structural analysis, FM | OUT of scope | ✓ (explicitly excluded) |

### Sverchok

Correctly DEFERRED in masterplan with rationale: "requires dedicated research phase."

### Missing AEC-Relevant Domains?

No missing AEC-relevant domains found. All domains identified in scope-analysis.md are either:
1. Covered by a dedicated skill
2. Covered within the scope of a broader skill (e.g., smaller API modules in ifcos-syntax-api)
3. Explicitly excluded with rationale
4. Explicitly deferred with rationale

---

## Summary of Issues Found

| # | Issue | Severity | Location |
|---|---|---|---|
| 1 | Dependency graph omits 6 cross-package dependencies | WARNING | Section 3, lines 763-785 |
| 2 | Batch summary table says "61 agents" — should be "60" | INFO | Section 4, line 938 |

---

## Overall Verdict: **READY** (with minor fixes recommended)

The masterplan is comprehensive and complete. All 60 skills from all four briefings are accounted for, every skill has all required fields, SOURCES.md URLs check out, no skills were dropped without explanation, and the scope boundary is complete against scope-analysis.md.

The two issues found are minor:
1. **WARNING**: The dependency graph visualization should be updated to include the 6 missing cross-package dependencies. This does not affect the batch plan (which correctly respects all dependencies).
2. **INFO**: Fix the "61" → "60" typo in the batch summary total.

Neither issue blocks skill creation. The masterplan is ready to drive batch execution.

---

*Review completed by review-completeness agent, 2026-03-06.*
