# Practical Review — Sverchok Skill Masterplan

**Date**: 2026-03-07
**Reviewer**: sv-review-practical agent
**Input**: sverchok-masterplan-raw.md (327 lines), vooronderzoek-sverchok.md (2352 lines), INDEX.md (61 existing skills)
**Status**: COMPLETE

---

## Review Checklist Results

### 1. Would Each Skill Actually Help Claude Write Better Sverchok Code?

| Skill | Clear Use Case? | Would a Dev Ask Claude? | Verdict |
|-------|----------------|------------------------|---------|
| sverchok-core-concepts | Yes — understanding architecture needed for any Sverchok work | Yes — "how does Sverchok update system work?" | USEFUL |
| sverchok-syntax-sockets | Yes — socket types are foundational for any node connection | Yes — "what socket type for curves?" | USEFUL |
| sverchok-syntax-data | Yes — #1 error source, nesting levels trip up everyone | Yes — "why is my data structure wrong?" | CRITICAL |
| sverchok-syntax-scripting | Yes — SNLite/Formula are the primary custom logic tools | Yes — "write me an SNLite script for X" | USEFUL |
| sverchok-syntax-api | Yes — programmatic tree construction is a real workflow | Yes — "generate a Sverchok node tree from Python" | USEFUL |
| sverchok-impl-custom-nodes | Yes — custom node dev is a core Sverchok skill | Yes — "write a custom Sverchok node that does X" | USEFUL |
| sverchok-impl-parametric | Yes — parametric design is the primary Sverchok use case | Yes — "create a parametric facade" | CRITICAL |
| sverchok-impl-ifcsverchok | Yes — BIM workflow is the package's differentiator | Yes — "connect Sverchok output to IFC" | USEFUL |
| sverchok-impl-extensions | Partial — mixes reference info with dev guide | Rarely — most users use extensions, few develop them | NEEDS WORK |
| sverchok-errors-common | Yes — directly prevents bad code generation | Yes — "why is my Sverchok node broken?" | CRITICAL |
| sverchok-agents-code-validator | Yes — validation catches AI mistakes before user sees them | N/A (agent-internal) | USEFUL |

**Result**: 10 of 11 skills have clear practical value. One skill needs restructuring (see Issue P-01).

---

### 2. Are Skills Organized from a USER's Perspective?

#### Scenario Testing

| User Says | Which Skill Triggers? | Clear Path? |
|-----------|----------------------|-------------|
| "Create a parametric facade" | sverchok-impl-parametric | YES — but see P-02 |
| "Write a custom Sverchok node" | sverchok-impl-custom-nodes | YES |
| "Connect my Sverchok to IFC" | sverchok-impl-ifcsverchok | YES |
| "Write an SNLite script" | sverchok-syntax-scripting | YES |
| "Generate a Sverchok tree from Python" | sverchok-syntax-api | YES |
| "Why is my nesting wrong?" | sverchok-syntax-data + sverchok-errors-common | YES — two skills complement |
| "Create a structural grid" | sverchok-impl-parametric? | UNCLEAR — see P-03 |
| "Generate MEP piping layout" | ??? | MISSING — see P-03 |
| "Create landscape from terrain data" | ??? | MISSING — see P-03 |
| "Use TopologicSverchok for adjacency analysis" | sverchok-impl-extensions | UNCLEAR — see P-01 |
| "What are Sverchok's field nodes?" | sverchok-syntax-sockets | BURIED — advanced data types are a subsection |
| "Create a window pattern on a wall" | sverchok-impl-parametric? | UNCLEAR — no facade-specific guidance |

**Result**: Core scenarios (custom nodes, scripting, IFC) have clear trigger paths. Parametric design scenarios and domain-specific AEC workflows need improvement.

---

### 3. Do Skill Descriptions Contain Clear TRIGGER WORDS?

Comparing against INDEX.md quality standard. Good existing examples:

> `blender-impl-mesh`: "Provides implementation workflows for Blender mesh operations including **creating buildings from vertices, IFC geometry visualization**, mesh analysis tools, **custom mesh generation for AEC**, BMesh algorithms..."

> `ifcos-impl-mep`: "Guides **MEP (Mechanical, Electrical, Plumbing) modeling** in IFC using ifcopenshell.api.system including **IfcSystem, IfcDistributionElement, ports, connections, flow segments, fittings**..."

These descriptions work because they contain **domain terms** users would type, not just API terms.

#### Masterplan Skill Scopes — Trigger Word Assessment

| Skill | Has Domain Trigger Words? | Issue |
|-------|--------------------------|-------|
| sverchok-core-concepts | NO — scope is API-heavy: "SverchCustomTree class, SverchCustomTreeNode base class and mixins..." | Needs domain framing |
| sverchok-syntax-sockets | PARTIAL — lists 16 socket types but no domain context like "choosing the right socket for geometry data" | Needs "when to use" framing |
| sverchok-syntax-data | PARTIAL — mentions "nesting levels" and "list matching" but not "why my vertices are flattened" or "data structure errors" | Needs error-scenario triggers |
| sverchok-syntax-scripting | YES — "SNLite, SN Functor B, Formula Mk5, Profile Mk3" are recognizable terms | OK |
| sverchok-syntax-api | PARTIAL — "External API access" but not "batch processing", "automating Sverchok", "generating trees programmatically" | Needs workflow triggers |
| sverchok-impl-custom-nodes | YES — "custom node development" is a clear trigger | OK |
| sverchok-impl-parametric | PARTIAL — mentions "parametric building elements" but not "facade", "grid", "array of columns", "floor plate" | Needs AEC domain terms |
| sverchok-impl-ifcsverchok | YES — "IfcSverchok BIM integration", "IFC file generation workflow" | OK |
| sverchok-impl-extensions | NO — scope reads like a library catalog, not a "use this when..." guide | Needs use-case framing |
| sverchok-errors-common | YES — lists concrete error patterns users would encounter | OK |
| sverchok-agents-code-validator | N/A — agent skill, not user-facing trigger | OK |

**Result**: 5 of 10 user-facing skills need better trigger words. See Issue P-04.

---

### 4. Are There Common Workflows Missing?

| Workflow | Covered? | Location | Gap? |
|----------|----------|----------|------|
| Parametric facade generation | PARTIAL | sverchok-impl-parametric mentions "parametric building elements" | No facade-specific recipe; no window pattern, mullion grid, or panel division examples |
| Structural grid generation | WEAK | sverchok-impl-parametric mentions "batch element generation with list processing" | No explicit column grid, beam grid, or floor plate workflow |
| MEP parametric generation (pipes, ducts) | MISSING | Not in any skill scope | See P-03 |
| Landscape/terrain from data | MISSING | Not in any skill scope | See P-03 |
| Parametric staircase | MISSING | Not in any skill scope | Common request, omitted |
| Curtain wall system | MISSING | Not in any skill scope | Major facade type, omitted |
| Roof geometry generation | MISSING | Not in any skill scope | Common AEC need |
| Solar/environmental analysis workflow | WEAK | sverchok-impl-extensions mentions Ladybug Tools in overview | No practical integration guidance |

**Result**: The masterplan is strong on Sverchok internals but **weak on AEC domain workflows**. A user asking "generate a column grid" or "create a curtain wall" won't find a clear recipe.

---

### 5. Can Each Skill Work Independently? (D-002)

| Skill | Standalone? | Issue |
|-------|------------|-------|
| sverchok-core-concepts | YES — foundation, no deps | OK |
| sverchok-syntax-sockets | YES with core-concepts loaded | OK |
| sverchok-syntax-data | YES with core-concepts loaded | OK |
| sverchok-syntax-scripting | PARTIAL — needs sockets + data context to be useful | Acceptable — skills reference, don't inline |
| sverchok-syntax-api | PARTIAL — needs sockets + data context | Acceptable |
| sverchok-impl-custom-nodes | PARTIAL — needs sockets + data + core | Acceptable — standard impl pattern |
| sverchok-impl-parametric | YES — workflow-oriented, self-contained patterns | OK |
| sverchok-impl-ifcsverchok | PARTIAL — references ifcos-* skills from other package | Cross-package dep is documented |
| sverchok-impl-extensions | YES — self-contained catalog | OK |
| sverchok-errors-common | YES — error patterns are self-explanatory | OK |
| sverchok-agents-code-validator | YES — checklist-based, self-contained | OK |

**Result**: All skills can work independently with appropriate cross-references. No circular dependencies. PASS.

---

### 6. Is the Agent Skill (code-validator) Comprehensive Enough?

Current checklist (~15 items) covers:
- ✓ Data nesting correctness
- ✓ updateNode callback presence
- ✓ Output connection check pattern
- ✓ List matching before zip
- ✓ Socket creation only in sv_init/sv_update
- ✓ deepcopy awareness
- ✓ SNLite socket type aliases
- ✓ Node docstring format
- ✓ Process method standard pattern
- ✓ IfcSverchok double-nesting
- ✓ BMesh cleanup (bm.free())
- ✓ NumPy optimization opportunities

#### Missing Validation Checks

| Check | Why Important | Severity |
|-------|--------------|----------|
| Import validation (sverchok modules exist) | AI often imports non-existent modules | HIGH |
| Socket name consistency (match sv_init to process) | Common AI mistake — creating socket "Vertices" but reading "Verts" | HIGH |
| `match_long_repeat` return unpacking pattern | AI often does `a, b = match_long_repeat([a, b])` without the `*` unpack | MEDIUM |
| Registration completeness (bl_idname, bl_label, sv_category, bl_icon) | Missing any = node won't register | MEDIUM |
| Property type validation (IntProperty for int, etc.) | AI sometimes uses wrong property type for the data | LOW |
| `sv_get(default=...)` fallback pattern | AI often forgets to provide defaults for optional inputs | MEDIUM |

**Result**: The validator is solid but should add import validation and socket name consistency checks. See Issue P-05.

---

### 7. Any Skill That Seems Too Niche to Be Useful?

| Skill | Too Niche? | Reasoning |
|-------|-----------|-----------|
| sverchok-impl-extensions | BORDERLINE | Mixes 3 different audiences: (1) users of TopologicSverchok, (2) users of Sverchok-Extra, (3) extension developers. Most users will never develop an extension. TopologicSverchok users need practical workflow guidance, not class hierarchy documentation. |
| All others | NO | Each addresses a real, common need |

**Result**: `sverchok-impl-extensions` should be refocused. See Issue P-01.

---

## Issues

### P-01: sverchok-impl-extensions Mixes Audiences

- **ISSUE**: The skill tries to serve three distinct audiences: (1) TopologicSverchok users wanting building topology analysis, (2) Sverchok-Extra users wanting advanced geometry, and (3) developers writing new extensions. These have completely different use cases and trigger words. A user asking "analyze space adjacency with Topologic" will get buried in extension registration boilerplate. A user writing an extension doesn't need the TopologicSverchok class hierarchy.
- **LOCATION**: sverchok-impl-extensions (SV-I-04)
- **RECOMMENDATION**: Split into two skills:
  - `sverchok-impl-topologic` — Focused on TopologicSverchok practical usage: building topology analysis, CellComplex workflows, space adjacency, dual graphs, energy simulation integration. Trigger words: "topology analysis", "space adjacency", "building envelope", "CellComplex", "dual graph".
  - `sverchok-impl-extensions` — Focused on extension ecosystem overview + development pattern. Keep Sverchok-Extra, Open3d, Mega-Polis references here. Trigger words: "write a Sverchok extension", "register custom nodes as extension", "Sverchok-Extra advanced geometry".
  This split adds 1 skill (total: 12) but dramatically improves trigger accuracy. TopologicSverchok has 200+ nodes and is the primary AEC-relevant extension — it deserves dedicated coverage.
- **SEVERITY**: WARNING

### P-02: sverchok-impl-parametric Lacks AEC Domain Recipes

- **ISSUE**: The skill scope describes Sverchok mechanisms ("working with matrices", "Blender object integration") rather than AEC workflows. A user asking "create a parametric facade" or "generate a column grid" will find abstract building blocks but no recognizable recipes. Compare with `bonsai-impl-modeling` from INDEX.md which lists concrete elements: "placing building elements (walls, slabs, columns, beams)".
- **LOCATION**: sverchok-impl-parametric (SV-I-02)
- **RECOMMENDATION**: Restructure the skill scope around AEC domain workflows:
  - **Structural grids**: Column grid generation, beam layouts, floor plate arrays
  - **Facade systems**: Panel division, window patterns, curtain wall mullion grids, parametric openings
  - **Building massing**: Floor plate extrusion, setback generation, atrium voids
  - **Element arrays**: Repeating elements with transforms (balustrades, railings, louvers)
  - **Data-driven generation**: Reading CSV/JSON parameters, spreadsheet-to-geometry workflows
  Keep the matrix operations and Blender integration as supporting techniques, not as the primary organizing principle.
- **SEVERITY**: WARNING

### P-03: Missing Common AEC Workflows

- **ISSUE**: Four common AEC parametric workflows are not covered by any skill:
  1. **MEP parametric generation** — pipe routing, duct runs, cable tray layouts using Sverchok nodes. Users working with mechanical/plumbing systems need parametric duct/pipe generation. The existing `ifcos-impl-mep` covers IFC-side MEP modeling but not Sverchok-side parametric geometry generation for MEP.
  2. **Landscape/terrain from data** — terrain mesh from contour data, point cloud to surface, site grading. Sverchok's surface/field nodes are ideal for this but no skill covers the workflow.
  3. **Parametric staircase** — one of the most frequently requested parametric building elements. Involves profile extrusion, array transforms, and railing generation.
  4. **Roof geometry** — hip/gable/shed roof generation from building footprint. Common AEC need with clear Sverchok patterns (profile extrusion, boolean operations).
- **LOCATION**: Skill inventory — gaps
- **RECOMMENDATION**: Add these as explicit workflow sections within `sverchok-impl-parametric` (preferred) or as a note for future expansion. At minimum, the skill description should include these terms as trigger words so Claude knows to attempt them using the parametric techniques documented. Example scope addition: "Common AEC workflows: structural grids, facade panels, parametric stairs, roof geometry, MEP routing layouts, terrain generation from data."
- **SEVERITY**: WARNING

### P-04: Skill Descriptions Lack Trigger-Word-Rich Framing

- **ISSUE**: 5 of 10 user-facing skills have scope descriptions that read like API documentation rather than skill activation descriptions. The existing INDEX.md skills use a pattern like: "Guides [domain task] including [concrete examples users would search for]". The Sverchok masterplan scopes read more like: "Covers [API classes], [method signatures], [internal architecture]".
- **LOCATION**: sverchok-core-concepts, sverchok-syntax-sockets, sverchok-syntax-data, sverchok-syntax-api, sverchok-impl-extensions
- **RECOMMENDATION**: Each skill description in the final INDEX.md entry should follow this pattern:
  - **sverchok-core-concepts**: "Explains Sverchok parametric node system architecture including node tree execution, data flow between nodes, update triggers, and the socket data cache. Use this when understanding how Sverchok processes node graphs, debugging update issues, or learning Sverchok fundamentals."
  - **sverchok-syntax-sockets**: "Covers all 16 Sverchok socket types including choosing the right socket for geometry, numbers, matrices, curves, surfaces, and fields. Use this when connecting nodes, understanding type conversions, or debugging socket compatibility errors."
  - **sverchok-syntax-data**: "Explains Sverchok's critical data nesting system and list matching — the #1 source of errors. Covers nesting levels for vertices (level 3), edges/faces (level 2), and matrices (level 1), plus the 5 list matching modes. Use this when data looks wrong, vertices are flattened, or list lengths don't match."
  - **sverchok-syntax-api**: "Guides programmatic control of Sverchok node trees from Python including creating nodes, connecting sockets, setting parameters, triggering updates, and batch processing with parameter sweeps. Use this when automating Sverchok, generating trees from scripts, or running batch parametric studies."
  - **sverchok-impl-extensions**: "Covers the Sverchok extension ecosystem including TopologicSverchok for building topology analysis, Sverchok-Extra for advanced geometry, and developing custom extensions. Use this when working with Topologic space adjacency, advanced surface/field operations, or packaging custom nodes as an extension."
  This is a description rewrite task during skill creation, not a masterplan restructure.
- **SEVERITY**: NOTE

### P-05: Agent Validator Missing Import and Socket Name Checks

- **ISSUE**: The code validator agent (SV-A-01) covers 15 checks but misses two high-frequency AI mistakes:
  1. **Import validation**: AI models frequently import non-existent Sverchok modules (e.g., `from sverchok.nodes import ...` instead of the correct utils path). The validator should check that all imports resolve to real Sverchok modules.
  2. **Socket name consistency**: AI models create sockets in `sv_init()` with one name (e.g., `"Vertices"`) and reference them in `process()` with a different name (e.g., `self.inputs["Verts"]`). This causes KeyError at runtime. The validator should cross-check socket names between `sv_init` and `process`.
  3. **match_long_repeat unpacking**: AI models often write `a, b = match_long_repeat([a_in, b_in])` instead of the correct `a, b = match_long_repeat([a_in, b_in])` (which already returns a list of lists, needing index access or star unpacking per Sverchok convention).
  4. **sv_get default pattern**: AI models often forget `sv_get(default=[[]])` for optional inputs, causing SvNoDataError when inputs are unconnected.
- **LOCATION**: sverchok-agents-code-validator (SV-A-01)
- **RECOMMENDATION**: Add these 4 checks to the validator scope, bringing the total to ~19 automated checks. Prioritize import validation and socket name consistency as HIGH severity checks.
- **SEVERITY**: NOTE

### P-06: No "Getting Started" Path for New Sverchok Users

- **ISSUE**: A user who is new to Sverchok and asks "help me get started with Sverchok" or "create my first Sverchok node tree" has no obvious skill trigger. The skills assume the user already knows what they want (scripting, custom nodes, parametric, etc.). Compare with the existing package: `bonsai-impl-project` covers "creating new IFC projects" — the very first step.
- **LOCATION**: Skill inventory — gap
- **RECOMMENDATION**: Ensure `sverchok-core-concepts` explicitly covers a "getting started" workflow: creating a node tree, adding basic nodes (generators, transforms, viewers), connecting them, and seeing output. The skill description should include trigger words like "getting started with Sverchok", "first Sverchok project", "create a Sverchok node tree". This doesn't require a new skill — just an expanded scope for `core-concepts`.
- **SEVERITY**: NOTE

### P-07: sverchok-impl-parametric Overlaps with sverchok-syntax-scripting for SNLite Workflows

- **ISSUE**: The parametric skill lists "vectorize utility in SNLite" and "performance optimization with setup() pre-computation" in its scope. These are SNLite-specific features that are fully covered by `sverchok-syntax-scripting`. A user asking about `vectorize` or `setup()` could trigger either skill, creating confusion about which is authoritative.
- **LOCATION**: sverchok-impl-parametric (SV-I-02) overlaps sverchok-syntax-scripting (SV-S-03)
- **RECOMMENDATION**: Remove `vectorize` and `setup()` from `sverchok-impl-parametric` scope. Instead, `impl-parametric` should reference `syntax-scripting` for SNLite-based patterns and focus on node-tree-level parametric workflows (which nodes to use, how to connect them, what parameters to expose).
- **SEVERITY**: NOTE

---

## Issue Summary

| # | Issue | Location | Severity |
|---|-------|----------|----------|
| P-01 | Extensions skill mixes 3 audiences; TopologicSverchok deserves own skill | SV-I-04 | WARNING |
| P-02 | Parametric skill lacks AEC domain recipes (facade, grid, etc.) | SV-I-02 | WARNING |
| P-03 | 4 common AEC workflows missing (MEP, terrain, stairs, roof) | Skill inventory | WARNING |
| P-04 | 5 skills lack trigger-word-rich descriptions | Multiple | NOTE |
| P-05 | Agent validator missing 4 high-frequency checks | SV-A-01 | NOTE |
| P-06 | No "getting started" path for new Sverchok users | SV-C-01 | NOTE |
| P-07 | Parametric/Scripting scope overlap on vectorize/setup() | SV-I-02 + SV-S-03 | NOTE |

**Warnings**: 3
**Notes**: 4
**Critical**: 0

---

## Verdict: PASS

No critical issues found. The masterplan covers Sverchok's core technical surface thoroughly and the skill structure is sound. All 11 skills have practical value, and the dependency chain is correct.

The 3 warnings should be addressed before skill creation:

1. **Split `sverchok-impl-extensions`** — TopologicSverchok is too important for AEC workflows to be buried in an extensions catalog. Give it a dedicated skill (total becomes 12 skills).
2. **Reframe `sverchok-impl-parametric` around AEC domain workflows** — Users think in terms of "facade", "column grid", "staircase", not "matrix operations" and "list processing". Restructure the scope around recognizable AEC deliverables.
3. **Add missing AEC workflow terms** — Even if full workflow documentation isn't feasible for all scenarios, include MEP routing, terrain generation, stairs, and roof geometry as trigger words so Claude attempts them with the available parametric toolkit.

The 4 notes are quality improvements that can be addressed during skill authoring without changing the masterplan structure.

---

*Review completed by sv-review-practical agent. Date: 2026-03-07.*
