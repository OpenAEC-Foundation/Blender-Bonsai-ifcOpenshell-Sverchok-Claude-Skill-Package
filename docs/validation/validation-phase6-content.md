# Phase 6 Content Validation Report

**Date:** 2026-03-06
**Validator:** validate-content agent
**Scope:** All 61 SKILL.md files under skills/

## Validation Criteria

1. **YAML Frontmatter** — Has valid `name` and `description` fields; description starts with third-person verb; contains trigger words
2. **English Only** — No Dutch words found: "gebruik", "voor", "niet", "moet", "deze", "wordt", "een", "het", "zijn", "van"
3. **Deterministic Language** — No banned phrases: "you might", "consider", "perhaps", "often", "usually", "it is recommended", "you could"
4. **Name Match** — Frontmatter `name` matches directory name

## Results Table

| # | Skill | Frontmatter | English | Deterministic | Name Match | Status |
|---|-------|-------------|---------|---------------|------------|--------|
| 1 | aec-agents-workflow-orchestrator | PASS | PASS | PASS | PASS | PASS |
| 2 | aec-core-bim-workflows | PASS | PASS | PASS | PASS | PASS |
| 3 | blender-agents-code-validator | PASS | PASS | PASS | PASS | PASS |
| 4 | blender-agents-version-migrator | PASS | PASS | PASS | PASS | PASS |
| 5 | blender-core-api | PASS | PASS | PASS | PASS | PASS |
| 6 | blender-core-gpu | PASS | PASS | PASS | PASS | PASS |
| 7 | blender-core-runtime | PASS | PASS | PASS | PASS | PASS |
| 8 | blender-core-versions | PASS | PASS | PASS | PASS | PASS |
| 9 | blender-errors-context | PASS | PASS | PASS | PASS | PASS |
| 10 | blender-errors-data | PASS | PASS | PASS | PASS | PASS |
| 11 | blender-errors-version | PASS | PASS | PASS | PASS | PASS |
| 12 | blender-impl-addons | PASS | PASS | PASS | PASS | PASS |
| 13 | blender-impl-animation | PASS | PASS | PASS | PASS | PASS |
| 14 | blender-impl-automation | PASS | PASS | PASS | PASS | PASS |
| 15 | blender-impl-mesh | PASS | PASS | PASS | PASS | PASS |
| 16 | blender-impl-nodes | PASS | PASS | PASS | PASS | PASS |
| 17 | blender-impl-operators | PASS | PASS | PASS | PASS | PASS |
| 18 | blender-syntax-addons | PASS | PASS | PASS | PASS | PASS |
| 19 | blender-syntax-animation | PASS | PASS | PASS | PASS | PASS |
| 20 | blender-syntax-data | PASS | PASS | PASS | PASS | PASS |
| 21 | blender-syntax-materials | PASS | PASS | PASS | PASS | PASS |
| 22 | blender-syntax-mesh | PASS | PASS | PASS | PASS | PASS |
| 23 | blender-syntax-modifiers | PASS | PASS | PASS | PASS | PASS |
| 24 | blender-syntax-nodes | PASS | PASS | PASS | PASS | PASS |
| 25 | blender-syntax-operators | PASS | PASS | PASS | PASS | PASS |
| 26 | blender-syntax-panels | PASS | PASS | PASS | PASS | PASS |
| 27 | blender-syntax-properties | PASS | PASS | PASS | PASS | PASS |
| 28 | blender-syntax-rendering | PASS | PASS | PASS | PASS | PASS |
| 29 | bonsai-agents-ifc-validator | PASS | PASS | PASS | PASS | PASS |
| 30 | bonsai-core-architecture | PASS | PASS | PASS | PASS | PASS |
| 31 | bonsai-errors-common | PASS | PASS | PASS | PASS | PASS |
| 32 | bonsai-impl-bcf | PASS | PASS | PASS | PASS | PASS |
| 33 | bonsai-impl-clash | PASS | PASS | PASS | PASS | PASS |
| 34 | bonsai-impl-classification | PASS | PASS | PASS | PASS | PASS |
| 35 | bonsai-impl-drawing | PASS | PASS | PASS | PASS | PASS |
| 36 | bonsai-impl-modeling | PASS | PASS | PASS | PASS | PASS |
| 37 | bonsai-impl-project | PASS | PASS | PASS | PASS | PASS |
| 38 | bonsai-impl-qto | PASS | PASS | PASS | PASS | PASS |
| 39 | bonsai-syntax-elements | PASS | PASS | PASS | PASS | PASS |
| 40 | bonsai-syntax-geometry | PASS | PASS | PASS | PASS | PASS |
| 41 | bonsai-syntax-properties | PASS | PASS | PASS | PASS | PASS |
| 42 | bonsai-syntax-spatial | PASS | PASS | PASS | PASS | PASS |
| 43 | ifcos-agents-code-validator | PASS | PASS | PASS | PASS | PASS |
| 44 | ifcos-core-concepts | PASS | PASS | PASS | PASS | PASS |
| 45 | ifcos-core-runtime | PASS | PASS | **FAIL** | PASS | **FAIL** |
| 46 | ifcos-errors-patterns | PASS | PASS | PASS | PASS | PASS |
| 47 | ifcos-errors-performance | PASS | PASS | PASS | PASS | PASS |
| 48 | ifcos-errors-schema | PASS | PASS | PASS | PASS | PASS |
| 49 | ifcos-impl-cost | PASS | PASS | PASS | PASS | PASS |
| 50 | ifcos-impl-creation | PASS | PASS | PASS | PASS | PASS |
| 51 | ifcos-impl-geometry | PASS | PASS | PASS | PASS | PASS |
| 52 | ifcos-impl-materials | PASS | PASS | **FAIL** | PASS | **FAIL** |
| 53 | ifcos-impl-mep | PASS | PASS | PASS | PASS | PASS |
| 54 | ifcos-impl-profiles | PASS | PASS | PASS | PASS | PASS |
| 55 | ifcos-impl-relationships | PASS | PASS | PASS | PASS | PASS |
| 56 | ifcos-impl-sequence | PASS | PASS | PASS | PASS | PASS |
| 57 | ifcos-impl-validation | PASS | PASS | PASS | PASS | PASS |
| 58 | ifcos-syntax-api | PASS | PASS | PASS | PASS | PASS |
| 59 | ifcos-syntax-elements | PASS | PASS | PASS | PASS | PASS |
| 60 | ifcos-syntax-fileio | PASS | PASS | PASS | PASS | PASS |
| 61 | ifcos-syntax-util | PASS | PASS | PASS | PASS | PASS |

## Failure Details

### 1. ifcos-core-runtime — Deterministic Language FAIL

- **File:** `skills/ifcopenshell/core/ifcos-core-runtime/SKILL.md` line 322
- **Banned phrase:** "often" (1 occurrence)
- **Context:** `- Often more up-to-date than pip`
- **Fix:** Replace with deterministic phrasing, e.g., `- More up-to-date than pip` or `- ALWAYS more up-to-date than pip`

### 2. ifcos-impl-materials — Deterministic Language FAIL

- **File:** `skills/ifcopenshell/impl/ifcos-impl-materials/SKILL.md` line 367
- **Banned phrase:** "often" (1 occurrence)
- **Context:** `Materials define physical properties. Styles define visual appearance. These are separate concepts in IFC but often applied together.`
- **Fix:** Replace with deterministic phrasing, e.g., `These are separate concepts in IFC but are typically applied together.` or `These are separate concepts in IFC. Apply them together.`

## Additional Notes

### ALWAYS/NEVER Directive Coverage

- **blender-agents-code-validator** contains no ALWAYS/NEVER directives. This is a warning (not a failure) — agent skills benefit from explicit ALWAYS/NEVER rules for critical validation patterns.

### Dutch Language Check

- Zero Dutch words detected across all 61 skills. All content is English-only.

### Frontmatter Quality

- All 61 skills have valid YAML frontmatter with `name` and `description` fields.
- All descriptions start with a third-person verb.
- All frontmatter `name` values match their directory names.

## Summary

| Metric | Count |
|--------|-------|
| **Total skills validated** | 61 |
| **PASS** | 59 |
| **FAIL** | 2 |
| **Warnings** | 1 |

### Failures Needing Attention

1. **ifcos-core-runtime** — Remove "Often" on line 322 (deterministic language violation)
2. **ifcos-impl-materials** — Remove "often" on line 367 (deterministic language violation)

### Pass Rate: 96.7% (59/61)
