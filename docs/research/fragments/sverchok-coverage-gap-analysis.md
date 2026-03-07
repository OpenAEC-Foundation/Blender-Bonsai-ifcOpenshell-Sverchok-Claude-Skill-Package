# Sverchok Coverage Gap Analysis

**Date**: 2026-03-07
**Author**: sv-coverage-analyst agent
**Purpose**: Assess whether the 12-skill Sverchok package provides COMPLETE coverage for a fully integrated workspace alongside Blender (26 skills), Bonsai (14 skills), and IfcOpenShell (19 skills).

---

## Executive Summary

| Criterion | Verdict | Summary |
|-----------|---------|---------|
| **A. Integration points with other packages** | **WARNING** | Sverchok→Blender and Sverchok→IfcOpenShell covered well. Sverchok→Bonsai partially covered. Cross-tech skills have ZERO Sverchok references. |
| **B. Vooronderzoek coverage completeness** | **PASS** | All 15 sections of the vooronderzoek are mapped to at least one planned skill. No orphaned research. |
| **C. Workspace context (skills/CLAUDE.md)** | **WARNING** | Sverchok mentioned as "(optional)" with no runtime rules, no version detection, no error patterns. Insufficient. |
| **D. Bidirectional cross-skill references** | **FAIL** | Sverchok skills reference Blender/IfcOpenShell/Bonsai. ZERO existing Blender, IfcOpenShell, Bonsai, or cross-tech skills reference Sverchok. |
| **E. Skill-per-API-surface ratio** | **PASS** | 12 skills is proportionally adequate given Sverchok's focused scope within the AEC context. |

**Overall Assessment**: The Sverchok skill package design is **architecturally sound** but has **integration gaps** — primarily missing bidirectional cross-references from existing packages and inadequate workspace context. These are fixable without adding new skills.

---

## Criterion A: Integration Points with Other Packages

### A1: Sverchok ↔ Blender — **PASS**

The integration surface between Sverchok and Blender is covered across multiple skills:

| Integration Point | Covering Skill | Coverage Quality |
|---|---|---|
| `bpy.data.node_groups` access | sverchok-syntax-api | Full — filtering by `bl_idname='SverchCustomTreeType'` |
| Blender object mesh data read/write | sverchok-impl-parametric | Full — §14.4 and §14.5 of vooronderzoek |
| `mathutils` (Vector, Matrix, Euler) | sverchok-impl-parametric | Full — §14.3 with code examples |
| BMesh integration | sverchok-impl-custom-nodes | Full — `bmesh_from_pydata`, `pydata_from_bmesh` |
| Blender operator registration | sverchok-impl-custom-nodes | Full — `register_classes_factory`, `bl_info` |
| Node tree system (SverchCustomTree) | sverchok-core-concepts | Full — tree class, events, update system |

**Finding**: All critical Blender integration points are well-covered by the planned Sverchok skills.

### A2: Sverchok ↔ IfcOpenShell (via IfcSverchok) — **PASS**

| Integration Point | Covering Skill | Coverage Quality |
|---|---|---|
| 31-node IfcSverchok catalog | sverchok-impl-ifcsverchok | Full — all 24 IFC + 7 Shape Builder nodes |
| SvIfcStore transient file management | sverchok-impl-ifcsverchok | Full — purge, get_file, id_map |
| SvIfcCore double-nested processing | sverchok-impl-ifcsverchok | Full — explicitly called out per review T-8 |
| Two geometry conversion modes | sverchok-impl-ifcsverchok | Full — SvIfcBMeshToIfcRepr + SvIfcSverchokToIfcRepr |
| 6-step IFC generation workflow | sverchok-impl-ifcsverchok | Full — complete pipeline documented |
| ifcopenshell.api bridge (SvIfcApi node) | sverchok-impl-ifcsverchok | Full — references ifcos-impl-creation |
| IfcSverchok error patterns | sverchok-errors-common | Full — undo crashes, purge, entity ID persistence, double-nesting |
| IfcSverchok validator checks | sverchok-agents-code-validator | Full — checks 10 and 19 |

**Finding**: IfcSverchok bridge is comprehensively covered. Cross-package dependencies are explicitly declared (→ ifcos-core-concepts, ifcos-syntax-elements, ifcos-impl-creation).

### A3: Sverchok ↔ Bonsai (via IfcSverchok `use_bonsai_file`) — **WARNING**

| Integration Point | Covering Skill | Coverage Quality |
|---|---|---|
| `use_bonsai_file` toggle | sverchok-impl-ifcsverchok | **Partial** — listed in scope under "Integration with Bonsai" |
| `tool.Ifc.get()` bridge | sverchok-impl-ifcsverchok | **Mentioned** — get_file() returns `tool.Ifc.get()` |
| Bonsai → IfcSverchok (loading files) | sverchok-impl-ifcsverchok | **Mentioned** — bidirectional workflow noted |
| Dependency chain (Bonsai must be enabled) | sverchok-impl-ifcsverchok | **Documented** — in architecture section |

**Finding**: The `use_bonsai_file` integration is documented but may need deeper coverage:
1. The vooronderzoek (§10.7) has only 4 lines on this integration — the research itself is thin here.
2. No documentation of what happens when `use_bonsai_file=True` and Bonsai's active file changes, or when multiple Bonsai projects are open.
3. The Bonsai skills (14 skills) contain ZERO mentions of Sverchok or IfcSverchok — the integration is purely one-directional from the Sverchok side.

### A4: TopologicSverchok ↔ IFC — **WARNING**

| Integration Point | Covering Skill | Coverage Quality |
|---|---|---|
| IFC → topology conversion | sverchok-impl-topologic | **Planned** — "IFC integration" listed in scope |
| Building envelope from CellComplex | sverchok-impl-topologic | **Planned** — AEC workflows documented |
| Space adjacency / dual graph | sverchok-impl-topologic | **Planned** — key use case |
| Energy simulation integration | sverchok-impl-topologic | **Mentioned** — in AEC workflows |

**Finding**: TopologicSverchok IFC integration is covered at the workflow level. However:
1. The vooronderzoek §11.1 mentions "IFC nodes (ifcopenshell)" as an optional TopologicSverchok integration but provides no detail on the actual API.
2. The link from TopologicSverchok → IfcOpenShell (reading IFC geometries for topology analysis) is not explicitly cross-referenced in any IfcOpenShell skill.
3. TopologicSverchok itself has 200+ nodes — the planned skill focuses on AEC workflows rather than exhaustive node documentation, which is the right scoping decision.

---

## Criterion B: Vooronderzoek Coverage Completeness

| Vooronderzoek Section | Lines | Mapped To Skill(s) | Covered? |
|---|---|---|---|
| §1 Overview & Architecture | L30-120 | sverchok-core-concepts | YES |
| §2 Node System & Data Flow | L122-288 | sverchok-core-concepts | YES |
| §3 Socket Type System | L290-431 | sverchok-syntax-sockets | YES |
| §4 Data Nesting Levels | L435-520 | sverchok-syntax-data | YES |
| §5 List Matching & Vectorization | L522-716 | sverchok-syntax-data | YES |
| §6 Node Categories | L720-778 | sverchok-core-concepts, sverchok-impl-parametric | YES |
| §7 Python Scripting | L782-1199 | sverchok-syntax-scripting | YES |
| §8 Custom Node Development | L1202-1465 | sverchok-impl-custom-nodes | YES |
| §9 External API Access | L1469-1598 | sverchok-syntax-api | YES |
| §10 IfcSverchok | L1614-1800 | sverchok-impl-ifcsverchok | YES |
| §11 Extensions Ecosystem | L1804-1936 | sverchok-impl-topologic, sverchok-impl-extensions | YES |
| §12 Common Error Patterns | L1939-2026 | sverchok-errors-common | YES |
| §13 AI Common Mistakes | L2028-2141 | sverchok-errors-common | YES |
| §14 Real-World Usage Patterns | L2145-2287 | sverchok-impl-parametric | YES |
| §15 Advanced Data Types | L2291-2333 | sverchok-syntax-sockets | YES |

**Finding**: 15/15 sections fully mapped. No orphaned research content.

### Capabilities NOT documented in vooronderzoek that could matter:

1. **Pulga Physics** — explicitly out of scope (masterplan §1). Correct decision for AEC focus.
2. **Geometry Nodes interop** — Sverchok and Geometry Nodes are parallel systems in Blender. No integration exists; correctly excluded.
3. **Monad/Group nodes** — mentioned in scope-analysis but not in vooronderzoek. Minor gap — group nodes are a UI convenience, not an API surface.
4. **Exchange nodes** (JSON, CSV, SVG, DXF) — mentioned in scope-analysis §4 but not researched. Could be useful for AEC data import workflows.

**Verdict**: PASS. All critical capabilities are covered. The Exchange nodes (§4 in scope-analysis: JSON/CSV/SVG/DXF import/export) are a minor gap that could be addressed within sverchok-impl-parametric's "data-driven generation (CSV/JSON parameters to geometry)" scope item.

---

## Criterion C: Workspace Context (skills/CLAUDE.md)

The workspace CLAUDE.md at `skills/CLAUDE.md` contains:

| Content | Present? | Quality |
|---|---|---|
| Sverchok listed as a technology | YES | Listed as "(optional)" — correct |
| Sverchok runtime rules | **NO** | No rules about node tree execution, socket cache, update triggers |
| Sverchok version detection | **NO** | No `sverchok.__version__` or equivalent |
| Sverchok error patterns | **NO** | No mention of nesting errors, updateNode, etc. |
| Sverchok-Blender interaction | **NO** | No mention of `bpy.data.node_groups` filtering for SverchCustomTreeType |
| Sverchok-IFC interaction | **NO** | No mention of IfcSverchok or SvIfcStore |
| Sverchok testing | **NO** | No headless testing pattern for Sverchok scripts |

**Finding**: The workspace CLAUDE.md provides zero runtime guidance for Sverchok. When the 12 Sverchok skills are built, the CLAUDE.md should be updated with:

1. **Sverchok Runtime Rules** section (parallel to Blender/IfcOpenShell sections):
   - Data nesting convention (vertices=level 3, strings=level 2, matrices=level 1)
   - Always use `updateNode` callback on properties
   - Socket data is cached in `socket_data_cache` — use `deepcopy=True` or don't mutate
   - IfcSverchok nodes share a single transient file (`SvIfcStore`)
2. **Version detection**:
   ```python
   # Sverchok availability
   import addon_utils
   is_enabled, is_loaded = addon_utils.check("sverchok")
   ```
3. **Key breaking changes** table entry for Sverchok (if any across versions)
4. **Testing pattern**:
   ```bash
   blender --background --python my_sverchok_script.py  # Sverchok addon must be enabled
   ```

**Verdict**: WARNING — requires CLAUDE.md update upon skill completion.

---

## Criterion D: Bidirectional Cross-Skill References

### Sverchok → Other Packages (outbound references)

| Sverchok Skill | References | Quality |
|---|---|---|
| sverchok-impl-ifcsverchok | ifcos-core-concepts, ifcos-syntax-elements, ifcos-impl-creation | Explicit in masterplan §5 |
| sverchok-impl-parametric | blender-core-api, blender-core-runtime | Explicit in masterplan §5 |
| sverchok-impl-custom-nodes | blender-syntax-addons | Explicit in masterplan §5 |
| sverchok-syntax-api | blender-core-api | Explicit in masterplan §5 |

**Outbound**: Well-documented. All 7 cross-package dependencies are explicitly declared.

### Other Packages → Sverchok (inbound references)

| Package | Skills | Mentions of Sverchok | Quality |
|---|---|---|---|
| Blender (26 skills) | All SKILL.md files | **ZERO** | Missing |
| IfcOpenShell (19 skills) | All SKILL.md files | **ZERO** | Missing |
| Bonsai (14 skills) | All SKILL.md files | **ZERO** | Missing |
| Cross-tech (2 skills) | aec-core-bim-workflows, aec-agents-workflow-orchestrator | **ZERO** | Critical gap |
| Workspace CLAUDE.md | 1 file | **1 mention** ("optional") | Insufficient |

**Finding**: This is the most significant gap. Key skills that SHOULD reference Sverchok:

1. **blender-syntax-nodes** — Should mention that Sverchok uses its own node tree type (`SverchCustomTreeType`) alongside Blender's built-in node systems (Geometry Nodes, Shader, Compositor).
2. **blender-core-api** — Should mention `bpy.data.node_groups` contains Sverchok trees when the addon is active.
3. **blender-impl-mesh** — Should mention that Sverchok generates mesh data via `bmesh_from_pydata` / `pydata_from_bmesh` and can be used as an alternative geometry generation pipeline.
4. **ifcos-impl-creation** — Should mention IfcSverchok as a visual/node-based alternative to scripted IFC creation.
5. **ifcos-impl-geometry** — Should mention that IfcSverchok provides visual geometry→IFC conversion via `SvIfcSverchokToIfcRepr` and `SvIfcBMeshToIfcRepr`.
6. **bonsai-core-architecture** — Should mention that IfcSverchok can operate on Bonsai's active IFC file via `use_bonsai_file` toggle.
7. **bonsai-impl-modeling** — Should reference Sverchok as a parametric modeling alternative for complex/repetitive geometry.
8. **aec-core-bim-workflows** — CRITICAL — Should include Sverchok in the workflow orchestration as a parametric design step before BIM authoring.
9. **aec-agents-workflow-orchestrator** — CRITICAL — Decision tree should include "Is this a parametric/generative design task? → Use Sverchok" pathway.

**Verdict**: FAIL — 9 skills need Sverchok cross-references added. The 2 cross-tech skills are the most critical gaps.

---

## Criterion E: Skill-per-API-Surface Ratio

### Comparative Analysis

| Package | Skills | API Surface | Ratio | Notes |
|---|---|---|---|---|
| **Blender** | 26 | ~900 classes, 50+ operator categories, 30+ data types, 5+ standalone modules | 26 / ~1000 = **1:38** | Largest API, appropriately large skill count |
| **IfcOpenShell** | 19 | 30+ API modules, 20+ util modules, geometry engine, validation | 19 / ~55 = **1:3** | Higher density — each skill covers multiple modules |
| **Bonsai** | 14 | ~40 bim/module/ directories, operators, UI | 14 / ~40 = **1:3** | Similar density to IfcOpenShell |
| **Sverchok** | 12 | 500+ nodes (reference only), 16 socket types, ~10 script types, 31 IFC nodes, 200+ Topologic nodes, extension system | 12 / ~15 API domains = **1:1.25** | Highest density per domain |

### Domain-per-Skill Mapping

| Sverchok Skill | Domains Covered |
|---|---|
| sverchok-core-concepts | Architecture, node tree, update system, events, 500+ node categories overview |
| sverchok-syntax-sockets | 16 socket types, conversions, processing flags, advanced data types |
| sverchok-syntax-data | Data nesting (4 levels), 5 matching modes, vectorize, SvRecursiveNode |
| sverchok-syntax-scripting | SNLite, SN Functor B, Formula Mk5, Profile Mk3, templates |
| sverchok-syntax-api | External tree control, batch processing, init_tree |
| sverchok-impl-custom-nodes | Node lifecycle, properties, BMesh, registration, error handling |
| sverchok-impl-parametric | 6 AEC workflow families (grids, facades, stairs, roofs, MEP, terrain) |
| sverchok-impl-ifcsverchok | 31 nodes, SvIfcStore, geometry modes, IFC workflow, Bonsai integration |
| sverchok-impl-topologic | CellComplex, adjacency, dual graphs, envelope, energy simulation |
| sverchok-impl-extensions | Sverchok-Extra, Open3d, extension development pattern |
| sverchok-errors-common | 17 anti-patterns (7 error patterns + 10 AI mistakes) |
| sverchok-agents-code-validator | 19 automated validation checks |

**Finding**: 12 skills covering 12 distinct domains. This is proportionally adequate because:

1. **Sverchok's AEC-relevant API surface is smaller** than Blender's general-purpose API — correctly scoped.
2. **500+ built-in nodes are explicitly excluded** from individual documentation — correct per masterplan ("reference only").
3. **Each skill averages 250-400 estimated lines** — well within the 500-line limit, suggesting no skill is overloaded.
4. **The split decision for TopologicSverchok** (from extensions into its own skill) was the right call per review P-01 — it has 200+ nodes and distinct AEC audience.

**Verdict**: PASS — 12 skills is proportionally adequate. Adding more skills would risk diminishing returns.

---

## Detailed Findings

### F1: Three Sverchok Skills Already Built

As of this analysis, 3 of the 12 planned skills have been created:
- `sverchok-core-concepts` (Batch 1)
- `sverchok-syntax-sockets` (Batch 2)
- `sverchok-syntax-data` (Batch 2)

This means 9 skills are still pending. The built skills can serve as templates for the remaining ones.

### F2: Exchange Nodes Gap (Minor)

The scope-analysis identifies Exchange nodes (JSON, CSV, SVG, DXF import/export) as a Sverchok capability. These are not explicitly covered by any planned skill, though `sverchok-impl-parametric` includes "data-driven generation (CSV/JSON parameters to geometry)" in its scope. This is a minor gap — Exchange nodes are utility nodes, not a distinct API domain.

**Recommendation**: Ensure `sverchok-impl-parametric` briefly mentions Exchange category nodes (CSV In, JSON In) as data sources for parametric workflows.

### F3: Monad/Group Node Gap (Minor)

Sverchok group nodes (Monads) enable reusable sub-trees. Not mentioned in the vooronderzoek or any planned skill. This is a UI-level feature that doesn't expose significant Python API surface.

**Recommendation**: Add a brief mention in `sverchok-core-concepts` under the node categories overview.

### F4: Cross-Tech Orchestrator Missing Sverchok Pathway

The `aec-agents-workflow-orchestrator` should include Sverchok in its decision tree:
- "Is this a parametric/generative design task with many repetitive elements?" → Use Sverchok
- "Do you need to generate IFC from parametric geometry?" → Use IfcSverchok
- "Do you need building topology analysis?" → Use TopologicSverchok

This is the highest-priority gap to fix.

### F5: INDEX.md Not Yet Updated

The `INDEX.md` currently lists 61 skills across 4 sections (Blender, IfcOpenShell, Bonsai, Cross-Tech). It does not include the Sverchok section. This will need to be updated as Sverchok skills are completed.

---

## Recommendations

### Priority 1: Fix Cross-References (Criterion D — FAIL)

When the Sverchok skill package is complete, add cross-references to these existing skills:

| Skill to Modify | Addition |
|---|---|
| `aec-agents-workflow-orchestrator` | Add Sverchok pathway in decision tree for parametric/generative design and IFC generation from node trees |
| `aec-core-bim-workflows` | Add Sverchok as a parametric design step in BIM workflow chains |
| `blender-syntax-nodes` | Note that `SverchCustomTreeType` is a third-party node tree type alongside GN/Shader/Compositor |
| `blender-impl-mesh` | Mention Sverchok as alternative geometry pipeline using `bmesh_from_pydata` |
| `ifcos-impl-creation` | Mention IfcSverchok as visual alternative for IFC creation |
| `ifcos-impl-geometry` | Reference IfcSverchok geometry conversion nodes |
| `bonsai-core-architecture` | Document `use_bonsai_file` integration with IfcSverchok |
| `bonsai-impl-modeling` | Reference Sverchok for parametric element generation |
| `skills/CLAUDE.md` | Add Sverchok runtime rules section |

### Priority 2: Update Workspace Context (Criterion C — WARNING)

Add to `skills/CLAUDE.md`:

```markdown
### Sverchok Runtime
- Sverchok uses its own node tree type (`SverchCustomTreeType`) accessible via `bpy.data.node_groups`
- Data flows through socket cache — nesting levels are CRITICAL: vertices=level 3, strings=level 2, matrices=level 1
- Always use `updateNode` callback on node properties or changes won't trigger re-evaluation
- IfcSverchok shares a single transient IFC file via `SvIfcStore` — purged on every full tree update
- Socket data mutation affects upstream nodes — use `deepcopy=True` on `sv_get()`
```

### Priority 3: Minor Content Additions (Criterion B — PASS with notes)

1. Add Exchange node mention to `sverchok-impl-parametric` scope
2. Add Monad/Group node mention to `sverchok-core-concepts` scope
3. Ensure `sverchok-impl-ifcsverchok` covers edge cases of `use_bonsai_file` (multi-file scenarios)

### No New Skills Needed

The 12-skill architecture is sufficient. All gaps can be addressed through:
- Cross-reference additions to existing skills (~9 skill modifications)
- Workspace CLAUDE.md update
- Minor scope expansions within planned skills

---

## Appendix: Masterplan vs. Original 61-Skill Plan

The original masterplan (masterplan.md) explicitly deferred Sverchok:

> **DEFERRED** (later phase):
> - Sverchok (all skills) — requires dedicated research phase
> - IfcSverchok bridge — requires both Sverchok and IfcOpenShell skills first

The sverchok-masterplan.md was created as a separate phase (S2) after the core 61 skills. This is architecturally correct — the Sverchok package builds ON TOP of the existing skills rather than duplicating their content. Key evidence:

1. `sverchok-impl-ifcsverchok` depends on `ifcos-core-concepts` and `ifcos-syntax-elements` — uses, doesn't duplicate
2. `sverchok-impl-parametric` depends on `blender-core-api` and `blender-core-runtime` — uses, doesn't duplicate
3. `sverchok-impl-custom-nodes` depends on `blender-syntax-addons` — uses, doesn't duplicate

The 12+61 = 73 total skills provide complete coverage when the cross-references are bidirectional.

---

*Document generated by sv-coverage-analyst agent. Date: 2026-03-07.*
