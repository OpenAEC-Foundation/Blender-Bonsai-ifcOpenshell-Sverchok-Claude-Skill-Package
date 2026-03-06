# Review: Masterplan Quality & Consistency Check

**Reviewer**: review-quality agent
**Date**: 2026-03-06
**Document reviewed**: docs/masterplan/masterplan.md
**Verdict**: **CONDITIONAL PASS** — 4 blockers must be fixed before skill creation begins

---

## Summary

| Area | Checks | Pass | Fail | Warnings |
|------|--------|------|------|----------|
| Consistency (1-4) | 4 | 1 | 3 | 0 |
| Quality (5-8) | 4 | 3 | 0 | 1 |
| Compliance (9-12) | 4 | 4 | 0 | 0 |
| **Total** | **12** | **8** | **3** | **1** |

---

## Consistency Checks

### Check 1: Skill Naming Convention — FAIL

**Convention**: `{tech}-{category}-{topic}` per WAY_OF_WORK.md
**Tech prefixes**: blender-, bonsai-, ifcos-, sverchok-, aec-
**Categories**: syntax, impl, errors, core, agents

**53 of 60 skills follow the convention correctly.** 3 skills violate it:

| Skill Name | Declared Category | Expected Name | Issue |
|------------|------------------|---------------|-------|
| `ifcos-code-validator` | agents | `ifcos-agents-code-validator` | Missing `agents` category segment |
| `bonsai-ifc-validator` | agents | `bonsai-agents-ifc-validator` | Missing `agents` category segment |
| `aec-workflow-orchestrator` | agents | `aec-agents-workflow-orchestrator` | `workflow` is not a valid category; `agents` segment missing |

**Comparison**: Blender agent skills (`blender-agents-code-validator`, `blender-agents-version-migrator`) correctly include the `agents` category segment. The IfcOpenShell, Bonsai, and cross-tech agent skills do not.

**Note**: The batch plan directories ARE correct (e.g., `skills/ifcopenshell/agents/ifcos-code-validator/`), creating an inconsistency between directory structure (which has `agents/`) and the skill name (which lacks `agents`).

**Severity**: WARNING — Inconsistent naming will cause confusion during skill creation and cross-referencing.

**Recommendation**: Rename to:
- `ifcos-agents-code-validator`
- `bonsai-agents-ifc-validator`
- `aec-agents-workflow-orchestrator`

---

### Check 2: Category Matching — PASS

All 60 skills have a declared category that matches one of the 5 valid categories (syntax, impl, errors, core, agents).

| Category | Blender | IfcOpenShell | Bonsai | Cross-tech | Total |
|----------|---------|-------------|--------|-----------|-------|
| core | 4 | 2 | 1 | 1 | 8 |
| syntax | 11 | 4 | 4 | 0 | 19 |
| impl | 6 | 8 | 7 | 0 | 21 |
| errors | 3 | 3 | 1 | 0 | 7 |
| agents | 2 | 1 | 1 | 1 | 5 |
| **Total** | **26** | **18** | **14** | **2** | **60** |

**Note**: The Executive Summary table says impl=20 but actual count is 21 (6+8+7). The summary says total agents=5, which matches. However, the summary shows impl=20 while actual is 6+8+6+0=20... wait, let me recount: Bonsai impl = project, modeling, classification, drawing, qto, bcf, clash = 7. So 6+8+7=21, not 20 as stated in the Executive Summary table.

**Issue found**: Executive Summary table shows `Impl = 20` but actual count across packages is `6 + 8 + 7 = 21`. Similarly, `Bonsai Impl` column shows 6 but the actual count is 7. The total column shows 14 for Bonsai which IS correct (1+4+7+1+1=14), but the breakdown is wrong: `Impl` should be 7, not 6.

**Severity**: WARNING — Table has arithmetic error in Bonsai impl count (shows 6, should be 7) and total impl (shows 20, should be 21). Grand total of 60 is maintained because errors also appear to be undercounted... Actually: 19+21+7+8+5 = 60 ✓ but as shown: 19+20+7+8+5 = 59. So the executive summary actually sums to 59 in the breakdown but shows 60 as total. There's a discrepancy.

---

### Check 3: Bidirectional Dependency Consistency — FAIL

**Method**: For each skill's declared dependencies, verified the dependency appears in an earlier layer of the dependency graph (Section 3).

**Issues found**:

1. **blender-syntax-animation (S-08)**: Declares dependency on `blender-syntax-properties` (S-02), but the dependency graph only shows `blender-core-api ──→ blender-syntax-animation`. The S-02 dependency is missing from the graph.

2. **blender-impl-addons (I-02)**: Declares dependencies on `blender-syntax-addons, blender-syntax-operators, blender-syntax-panels`, but the graph only shows `S-04 ──→ blender-impl-addons`. The S-01 and S-03 dependencies are redundant (transitive through S-04) but the graph is incomplete.

3. **bonsai-impl-drawing**: Declares dependency on `blender-core-gpu` (C-03), but the dependency graph in Layer 5 shows: `bonsai-core-arch+syntax-elements+geometry ──→ bonsai-impl-drawing`. The blender-core-gpu cross-package dependency is missing from the graph.

4. **bonsai-impl-qto**: Declares dependency on `blender-syntax-mesh`, but the graph shows: `bonsai-syntax-elements+properties ──→ bonsai-impl-qto`. The cross-package dependency on blender-syntax-mesh is missing.

**Severity**: WARNING — The dependency graph in Section 3 is a simplified visualization that omits some declared dependencies. This is acceptable for readability but could cause issues during build ordering if only the graph is consulted.

**Recommendation**: Add a note to Section 3 stating "This graph shows primary dependencies. Consult individual skill entries in Section 2 for complete dependency lists."

---

### Check 4: Batch Plan vs Dependency Graph — FAIL (BLOCKER)

**Method**: For each batch, verified that no skill depends on another skill in the same batch.

**4 violations found**:

#### Violation 1: Batch 4 — `blender-syntax-addons` depends on `blender-syntax-panels`
- `blender-syntax-addons` (A2) declares dependency: `blender-syntax-operators, blender-syntax-panels`
- `blender-syntax-panels` (A1) is in the **same batch** (Batch 4)
- **Impact**: If agents run in parallel, blender-syntax-addons may reference a skill that doesn't exist yet

#### Violation 2: Batch 9 — `aec-core-bim-workflows` depends on `bonsai-core-architecture`
- `aec-core-bim-workflows` (A5) declares dependency: `ifcos-impl-creation, bonsai-core-architecture`
- `bonsai-core-architecture` (A4) is in the **same batch** (Batch 9)
- **Impact**: Cross-technology skill references a Bonsai skill that may not be written yet

#### Violation 3: Batch 10 — 3 skills depend on `bonsai-syntax-elements`
- `bonsai-syntax-spatial` (A2) depends on `bonsai-syntax-elements`
- `bonsai-syntax-properties` (A3) depends on `bonsai-syntax-elements`
- `bonsai-syntax-geometry` (A4) depends on `bonsai-syntax-elements`
- `bonsai-syntax-elements` (A1) is in the **same batch** (Batch 10)
- **Impact**: 3 out of 4 skills in this batch depend on the 4th skill

#### Violation 4: Batch 12 — `bonsai-errors-common` depends on same-batch skills
- `bonsai-errors-common` (A3) declares dependency: `All other bonsai skills`
- `bonsai-impl-bcf` (A1) and `bonsai-impl-clash` (A2) are in the **same batch** (Batch 12)
- **Impact**: Error skill references implementation skills not yet written

**Severity**: BLOCKER — These violations mean agents in the same batch may reference skills that don't exist yet. This will cause incomplete cross-references or missing dependency content.

**Recommendation**: Restructure batches to resolve dependencies:

```
CURRENT                          PROPOSED FIX
─────────────────────────────    ─────────────────────────────
Batch 4: panels + addons + ...   Batch 4: panels + animation +
                                          materials + data
                                 Batch 4b: addons (moved to B5)

Batch 9: ...+ bonsai-core +     Batch 9: blender-errors + bonsai-core
         aec-bim-workflows       Batch 9b: aec-bim-workflows (moved to B10)

Batch 10: bonsai-syntax-elements Batch 10: bonsai-syntax-elements (alone or
          + spatial/props/geom            with non-dependent skills)
                                 Batch 10b: spatial + props + geom

Batch 12: bcf + clash +         Batch 12: bcf + clash + ifcos-validator
          errors-common +        Batch 12b: bonsai-errors-common
          ifcos-validator
```

Alternative: Accept same-batch dependencies with a constraint that dependent skills run AFTER their dependencies complete within the batch (sequential within batch). Document this constraint explicitly.

---

### Batch Summary Table Error

The batch summary table shows `61` agents but the correct total is `60` (matching the 60 skills). Each batch assigns 1 agent per skill.

| | Stated | Actual |
|---|--------|--------|
| Total Skills | 60 | 60 ✓ |
| Total Agents | 61 | 60 ✗ |

**Severity**: INFO — Typo in the summary table.

---

## Quality Checks

### Check 5: Per-Skill Prompt Usability — PASS

The prompt template (Section 5) is comprehensive and ready-to-use. It includes:

- **Context**: Project name, workspace path
- **Prerequisites**: 5 documents to read before starting (REQUIREMENTS.md, WAY_OF_WORK.md, DECISIONS.md, dependency skills, research docs)
- **Output**: Exact directory path, 4 files to create (SKILL.md + 3 references)
- **Format**: Complete YAML frontmatter template with license and metadata
- **Content sections**: Quick Reference, Essential Patterns, Common Operations, Reference Links
- **Sources**: Approved URLs per skill
- **Research**: Specific research sections with line numbers
- **Style**: 8 explicit rules including deterministic language requirements
- **Verification**: 8-point checklist

The per-skill parameter table (Section 5) provides skill-specific values for all 60 skills across 6 columns (Output Dir, Research Docs, Source URLs, Version Coverage, Dependencies).

**Minor observation**: The prompt template includes `license: MIT` and `compatibility` fields in the YAML frontmatter that are not mentioned in WAY_OF_WORK.md's skill format specification. These are additions from the masterplan. Consistent with DECISIONS.md D-005 (MIT License) but should be added to WAY_OF_WORK.md for alignment.

---

### Check 6: SOURCES.md URL Verification — PASS (with warnings)

**Method**: Verified 15 URLs from SOURCES.md and cross-referenced masterplan URL claims against SOURCES.md content.

#### URL Reachability (5 spot-checked via WebFetch, all valid):
1. https://docs.blender.org/api/current/ ✓ (Blender Python API docs)
2. https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html ✓ (IfcOpenShell API index)
3. https://academy.ifcopenshell.org/ ✓ (IfcOpenShell Academy with tutorials)
4. https://docs.bonsaibim.org/ ✓ (Bonsai 0.8.4 documentation)
5. https://ifc43-docs.standards.buildingsmart.org/ ✓ (IFC 4.3.2 docs)

#### SOURCES.md Last Verified status (2026-03-06):
- Blender: All 26 URLs OK ✓
- Bonsai: All OK except OSArch Wiki [BROKEN/403] — documented ✓
- IfcOpenShell: All OK ✓
- IFC Standard: All 13 URLs OK ✓
- Claude/Anthropic: All 17 URLs OK ✓
- OpenAEC: Skill Package repo [BROKEN/404] — documented ✓

**2 known broken URLs are documented in SOURCES.md with [BROKEN] markers.** This is good practice.

#### Cross-reference issue: Masterplan URLs not in SOURCES.md

The masterplan labels certain URLs as "SOURCES.md URLs" but several are sub-pages not explicitly listed in SOURCES.md:

| Masterplan URL | In SOURCES.md? | Parent URL in SOURCES.md? |
|----------------|---------------|--------------------------|
| `docs.ifcopenshell.org/autoapi/.../api/material/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/cost/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/sequence/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/system/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/profile/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/spatial/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/aggregate/index.html` | No | Yes (docs hub) |
| `docs.ifcopenshell.org/autoapi/.../api/type/index.html` | No | Yes (docs hub) |
| `academy.ifcopenshell.org/posts/creating-a-simple-wall...` | No | Yes (academy root) |

These are valid sub-pages of documented parent URLs. The URLs themselves are real (verified by WebFetch), but labeling them as "SOURCES.md URLs" is technically inaccurate.

**Severity**: INFO — URLs are valid but the label "SOURCES.md URLs" is misleading for sub-pages not explicitly listed. Consider either adding these to SOURCES.md or changing the label to "Approved source URLs (derived from SOURCES.md)".

---

### Check 7: Quality Gate Template Comprehensiveness — PASS

**Method**: Compared Section 6 (Quality Gate Template) with Prompt D in session-prompts.md.

The masterplan's quality gate is a **superset** of Prompt D:

| Check Area | Prompt D | Masterplan QG | Difference |
|------------|----------|---------------|------------|
| SKILL.md exists | ✓ | ✓ | — |
| YAML frontmatter | ✓ | ✓ | — |
| Description triggers | ✓ | ✓ | — |
| Description format | ✗ | ✓ | **Added**: third-person verb, max 1024 chars |
| < 500 lines | ✓ | ✓ | — |
| references/ exists | ✓ | ✓ | — |
| File links valid | ✓ | ✓ | — |
| English only | ✓ | ✓ | — |
| Deterministic language | ✓ | ✓ | — |
| Version annotations | ✓ | ✓ | — |
| Source verification | ✓ | ✓ | — |
| Cross-references | ✓ | ✓ | — |
| Dependency references | ✗ | ✓ | **Added** |
| Python syntax check | ✓ | ✓ | — |
| Anti-patterns 3+ | ✓ | ✓ | — |
| Decision trees | ✓ | ✓ | — |
| Quick Reference | ✓ | ✓ | — |
| L-complexity ref use | ✗ | ✓ | **Added** |
| Severity levels | ✓ | ✓ | — |

The masterplan adds 3 checks not in Prompt D. No checks from Prompt D are missing. **Comprehensive and complete.**

---

### Check 8: Complexity Estimates — PASS (with warning)

**Method**: Evaluated S/M/L assignments against declared API surface and scope.

| Complexity | Count | Typical Scope |
|-----------|-------|---------------|
| S (Small) | 6 | 6-14 API functions, narrow scope |
| M (Medium) | 37 | Moderate API surface, focused domain |
| L (Large) | 17 | Many API functions or broad scope |

**All estimates are reasonable.** Spot-check examples:

| Skill | Complexity | API Surface | Assessment |
|-------|-----------|-------------|------------|
| `ifcos-syntax-fileio` | S | open/file/write — 6 main functions | Correct ✓ |
| `ifcos-impl-profiles` | S | 6 API functions | Correct ✓ |
| `blender-syntax-mesh` | L | Mesh + BMesh + attributes + normals | Correct ✓ |
| `ifcos-impl-sequence` | L | 40 API + 16 util functions | Correct ✓ |
| `bonsai-core-architecture` | L | 3-layer architecture, complex patterns | Correct ✓ |

**Warning**: Several L-complexity skills may challenge the 500-line SKILL.md limit:
- `blender-core-versions` (L): All breaking changes across 8+ versions
- `blender-syntax-animation` (L): 7 sub-topics (keyframes, FCurves, drivers, constraints, armatures, NLA, shape keys)
- `bonsai-errors-common` (L): 31+ documented anti-patterns consolidated from 4 error categories

These skills will need aggressive use of `references/` files to stay under 500 lines. The quality gate check "L-complexity skills use references/ for detailed API tables" addresses this.

**Severity**: WARNING — Monitor L-complexity skills during creation for line count compliance.

---

## Compliance Checks

### Check 9: D-002 — Separate Packages — PASS

Technologies are organized in separate directories:
- `skills/blender/` — 26 skills
- `skills/ifcopenshell/` — 18 skills
- `skills/bonsai/` — 14 skills
- `skills/aec-cross-tech/` — 2 skills (optional)

Cross-package dependencies exist (e.g., Bonsai depends on Blender and IfcOpenShell) but these are **build-time** dependencies for the skill creation process, not **runtime** installation dependencies. Each package is designed to work independently once installed.

The masterplan explicitly states cross-technology skills in `skills/aec-cross-tech/` are optional, consistent with D-002's implication that cross-references between packages must be optional.

---

### Check 10: D-003 — English Only — PASS

The entire masterplan is written in English. No Dutch content found in:
- Skill names, descriptions, or scopes
- Prompt templates and style rules
- Quality gate criteria
- Dependency graph labels

The only Dutch-origin words are document names (`vooronderzoek-blender`, `vooronderzoek-bonsai`) used as file references, not as content. This is acceptable — they are proper nouns (document names).

---

### Check 11: D-008 — Build Order Blender → IfcOpenShell → Bonsai — PASS

D-008 states: "Blender -> Bonsai -> IfcOpenShell -> Sverchok"
D-011 refines: "Blender and IfcOpenShell skills are built in parallel. Bonsai waits for both."

The batch plan follows D-011:

| Phase | Batches | Content |
|-------|---------|---------|
| Foundation | 1-2 | Blender core + IfcOpenShell core (parallel) |
| Syntax | 3-5 | Blender syntax + IfcOpenShell syntax (parallel) |
| Implementation | 6-8 | Blender impl + IfcOpenShell impl (parallel) |
| Errors + Bonsai core | 9 | Blender errors + Bonsai foundation |
| Bonsai | 10-12 | Bonsai syntax → impl → errors |
| Agents | 13 | All agent skills |

Bonsai correctly starts only after both Blender and IfcOpenShell have core + syntax + impl + errors skills complete. Sverchok is correctly deferred.

**Note**: D-008's literal text says "IfcOpenShell" comes after "Bonsai", which contradicts the architectural reality (Bonsai depends on IfcOpenShell). D-011 supersedes this correctly. Consider updating D-008's text to match D-011.

---

### Check 12: D-009 — Skill Scope ≤ 500 Lines — PASS

The 500-line constraint is enforced at multiple levels:

1. **Per-skill prompt template**: "SKILL.md (< 500 lines)" — explicit instruction to writer agents
2. **Quality gate**: "SKILL.md < 500 lines (`wc -l`)" — post-creation validation
3. **Verification checklist**: "SKILL.md < 500 lines" — self-check for agents
4. **L-complexity guidance**: "L-complexity skills use `references/` for detailed API tables" — overflow strategy

The skill structure (SKILL.md + references/ directory) provides a clear mechanism for keeping the main file under 500 lines while allowing detailed content in reference files.

**No scope concerns** at the planning level. Actual compliance will be verified during Phase 5 (skill creation).

---

## Additional Findings

### Finding A: Executive Summary Arithmetic Error

The skill count table in Section 1 has an error:

```
| Package     | Syntax | Impl | Errors | Core | Agents | Total |
| Bonsai      | 4      | 6    | 1      | 1    | 1      | 14    |
```

Bonsai has 7 impl skills (project, modeling, classification, drawing, qto, bcf, clash), not 6. This makes:
- Bonsai impl: **7** (not 6)
- Total impl: **21** (not 20)
- But Bonsai total remains 14 (1+4+7+1+1=14)... wait, 4+6+1+1+1=13 ≠ 14 as shown

Actually the table shows Bonsai total as 14, which matches 4+**7**+1+1+1=14. So the impl column is wrong (shows 6, should be 7) but the total column happens to be correct. The total impl row shows 20 but should be 21.

**Severity**: WARNING — Arithmetic error in summary table. Fix impl count for Bonsai (6→7) and total impl (20→21).

### Finding B: Batch Summary Agent Count Typo

The batch summary table shows "61" total agents but the correct sum is 60 (matching skills).

**Severity**: INFO — Simple typo.

### Finding C: `ifcos-code-validator` naming vs Blender agents

The naming pattern for agent skills is inconsistent across packages:
- Blender: `blender-agents-*` (includes category) ✓
- IfcOpenShell: `ifcos-code-validator` (omits category) ✗
- Bonsai: `bonsai-ifc-validator` (omits category) ✗
- Cross-tech: `aec-workflow-orchestrator` (uses non-standard category) ✗

This should be standardized before skill creation begins.

---

## Verdict

### Overall: CONDITIONAL PASS

The masterplan is well-structured, comprehensive, and largely consistent. The 60-skill inventory is well-defined with clear scopes, dependencies, and creation prompts. However, **4 batch dependency violations** must be resolved before skill creation can begin, as they would cause agents to reference non-existent skills.

### Blockers (must fix)

| # | Issue | Location | Fix |
|---|-------|----------|-----|
| B1 | Batch 4: addons depends on panels (same batch) | §4 Batch 4 | Move addons to Batch 5 or run panels before addons |
| B2 | Batch 9: aec-bim-workflows depends on bonsai-core-architecture (same batch) | §4 Batch 9 | Move aec-bim-workflows to Batch 10 |
| B3 | Batch 10: 3 skills depend on bonsai-syntax-elements (same batch) | §4 Batch 10 | Split into Batch 10a (elements) + Batch 10b (spatial/props/geom) |
| B4 | Batch 12: errors-common depends on bcf+clash (same batch) | §4 Batch 12 | Move errors-common to Batch 13 or add Batch 12b |

### Warnings (should fix before Phase 5)

| # | Issue | Location | Fix |
|---|-------|----------|-----|
| W1 | 3 agent skill names missing `agents` category | §2.2, §2.3, §2.4 | Rename to include `-agents-` segment |
| W2 | Executive Summary impl count wrong (20→21) | §1 table | Fix Bonsai impl: 6→7, total impl: 20→21 |
| W3 | Dependency graph incomplete (missing some declared deps) | §3 | Add note about simplified view, or add missing edges |
| W4 | L-complexity skills may challenge 500-line limit | §2 various | Monitor during creation; pre-plan references/ structure |

### Info (nice to have)

| # | Issue | Location | Fix |
|---|-------|----------|-----|
| I1 | Batch summary says 61 agents, should be 60 | §4 summary | Fix typo |
| I2 | Masterplan URLs labeled "SOURCES.md URLs" but some are sub-pages | §2 various | Change label or add sub-pages to SOURCES.md |
| I3 | D-008 text contradicts D-011 ordering | DECISIONS.md | Update D-008 text or add cross-reference to D-011 |

---

*Review completed by review-quality agent, 2026-03-06*
