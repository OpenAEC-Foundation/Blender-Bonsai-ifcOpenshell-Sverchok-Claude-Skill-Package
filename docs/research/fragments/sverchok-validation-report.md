# Sverchok Skills Validation Report

**Generated**: 2026-03-07
**Validator**: sv-validator agent
**Skills validated**: 12 / 12
**Target**: All SKILL.md files in `skills/sverchok/`

---

## Summary Table

| # | Skill | Lines | 1. YAML | 2. <500 | 3. English | 4. Deterministic | 5. Version | 6. Refs Exist | 7. Refs Linked | 8. Structure | 9. No Placeholders | 10. Python OK |
|---|-------|-------|---------|---------|------------|-------------------|------------|---------------|----------------|--------------|--------------------|----|
| 1 | sverchok-core-concepts | 321 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 2 | sverchok-syntax-sockets | 398 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 3 | sverchok-syntax-data | 379 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 4 | sverchok-syntax-scripting | 397 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 5 | sverchok-syntax-api | 350 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 6 | sverchok-impl-custom-nodes | 470 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 7 | sverchok-impl-parametric | 477 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 8 | sverchok-impl-ifcsverchok | 382 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 9 | sverchok-impl-topologic | 287 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 10 | sverchok-impl-extensions | 499 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 11 | sverchok-errors-common | 343 | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| 12 | sverchok-agents-code-validator | 453 | PASS | PASS | PASS | PASS | PASS | PASS* | PASS | PASS* | PASS | PASS |

**Result: 12/12 PASS — All skills meet quality criteria. Minor observations noted below.**

---

## Detailed Findings Per Check

### Check 1: YAML Frontmatter Valid

All 12 skills have valid YAML frontmatter with:
- `name` — matches directory name
- `description` — contains trigger words describing when to activate
- `license: MIT` — present in all 12
- `compatibility` — present in all 12
- `metadata` with `author: OpenAEC-Foundation` and `version: '1.0'` — present in all 12

**Additional note**: `sverchok-agents-code-validator` also includes a `dependencies` field listing 3 dependency skills. This is valid extra metadata.

### Check 2: Line Count < 500

| Skill | Lines | Status |
|-------|-------|--------|
| sverchok-core-concepts | 321 | PASS |
| sverchok-syntax-sockets | 398 | PASS |
| sverchok-syntax-data | 379 | PASS |
| sverchok-syntax-scripting | 397 | PASS |
| sverchok-syntax-api | 350 | PASS |
| sverchok-impl-custom-nodes | 470 | PASS |
| sverchok-impl-parametric | 477 | PASS |
| sverchok-impl-ifcsverchok | 382 | PASS |
| sverchok-impl-topologic | 287 | PASS |
| sverchok-impl-extensions | 499 | PASS (at limit) |
| sverchok-errors-common | 343 | PASS |
| sverchok-agents-code-validator | 453 | PASS |

**Observation**: `sverchok-impl-extensions` is at exactly 499 lines — one line under the 500-line limit. Any future additions will exceed the threshold.

### Check 3: English-Only

All 12 skills are written entirely in English. No Dutch or other non-English language detected via comprehensive grep for common Dutch words (`gebruik`, `bestand`, `voorbeeld`, `eigenschap`, `aanmaken`, `verwijderen`, `toevoegen`, `schrijf`, `samenvatting`).

**Result**: PASS for all 12.

### Check 4: Deterministic Language

Searched all SKILL.md files for hedging/non-deterministic phrases: `you might consider`, `you could`, `you may want`, `consider using`, `it might be`, `perhaps`, `maybe you`, `you should consider`.

**Zero matches found.** All skills use deterministic language (ALWAYS/NEVER directives, imperative statements).

**Result**: PASS for all 12.

### Check 5: Version-Explicit

All 12 skills specify version information:

| Skill | Compatibility Field | Body References |
|-------|--------------------|-----------------|
| sverchok-core-concepts | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-syntax-sockets | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-syntax-data | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-syntax-scripting | Blender 4.0+/5.x, Sverchok v1.4.0+ | No explicit code comments |
| sverchok-syntax-api | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-impl-custom-nodes | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-impl-parametric | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-impl-ifcsverchok | Blender 4.0+/5.x, Sverchok v1.4.0+, IfcOpenShell 0.8.x | Yes (code comments) |
| sverchok-impl-topologic | Blender 4.0+/5.x, Sverchok v1.2.0+, TopologicSverchok v0.8.3+ | Yes (table) |
| sverchok-impl-extensions | Blender 4.0+/5.x, Sverchok v1.4.0+ | Yes (code comments) |
| sverchok-errors-common | Blender 4.0+/5.x, Sverchok v1.4.0+ | No explicit code comments |
| sverchok-agents-code-validator | Python 3.x | Body: "Targets Sverchok v1.4.0 on Blender 4.0+" |

**Observation**: `sverchok-agents-code-validator` uses "Requires Python 3.x" in its compatibility field instead of the standard "Requires Blender 4.0+/5.x with Sverchok v1.4.0+". This is acceptable because it is a validation agent that operates on Python code, not within Blender. The body text clarifies the target versions.

**Observation**: `sverchok-impl-topologic` specifies Sverchok v1.2.0+ (not v1.4.0+), which is correct since TopologicSverchok has different version requirements.

**Result**: PASS for all 12.

### Check 6: All references/ Files Exist

All 12 skills have the required `references/` directory containing:
- `methods.md` — present in all 12
- `examples.md` — present in all 12
- `anti-patterns.md` — present in all 12

**Result**: PASS for all 12.

### Check 7: References Are Linked from SKILL.md

All 12 skills contain explicit reference links in their `## Reference Links` section pointing to:
- `[references/methods.md](references/methods.md)`
- `[references/examples.md](references/examples.md)`
- `[references/anti-patterns.md](references/anti-patterns.md)`

**Minor observation**: `sverchok-impl-extensions` uses `--` (double hyphen) instead of `—` (em dash) in its reference link descriptions. This is a cosmetic inconsistency, not a functional issue.

**Minor observation**: `sverchok-agents-code-validator` uses different link labels ("Validation Rules", "Before/After Examples", "Anti-Patterns Catalog") but links to the same standard file names. This is acceptable.

**Result**: PASS for all 12.

### Check 8: Structure Follows Quick Reference > Patterns/Decision Trees > Reference Links

All 12 skills follow the required structural pattern:

| Skill | Quick Reference | Decision Tree(s) | Patterns/Content | Reference Links |
|-------|----------------|-------------------|------------------|-----------------|
| sverchok-core-concepts | Line 13 | Line 38 | Line 64 (Essential Patterns) | Line 311 |
| sverchok-syntax-sockets | Line 13 | Line 50 | Line 134 (Essential Patterns) | Line 388 |
| sverchok-syntax-data | Line 13 | Line 60 | Line 97 (Essential Patterns) | Line 368 |
| sverchok-syntax-scripting | Line 13 | Line 40 | Line 65+ (SNLite sections) | Line 385 |
| sverchok-syntax-api | Line 13 | Line 46 | Line 78 (Essential Patterns) | Line 339 |
| sverchok-impl-custom-nodes | Line 13 | Line 48 | Line 73 (Essential Patterns) | Line 453 |
| sverchok-impl-parametric | Line 13 | Line 41 | Line 78 (Essential Patterns) | Line 461 |
| sverchok-impl-ifcsverchok | Line 13 | Line 50 | Line 79 (Essential Patterns) | Line 366 |
| sverchok-impl-topologic | Line 13 | Line 52 | Line 111 (Essential Patterns) | Line 271 |
| sverchok-impl-extensions | Line 13 | Line 38 | Line 258 (Essential Patterns) | Line 488 |
| sverchok-errors-common | Line 13 | Line 37 | Line 65 (Part A/B) | Line 334 |
| sverchok-agents-code-validator | Line 19* | N/A* | Line 28 (Validation Checklist) | Line 439 |

**Observations**:
- `sverchok-agents-code-validator` uses `## Quick Reference — When to Activate` (line 19) instead of bare `## Quick Reference`. This is an appropriate variant for an agent skill.
- `sverchok-agents-code-validator` does not have a separate "Decision Tree" section but instead has a `## Severity Summary Decision Tree` at line 360. This is acceptable as the skill is a validation agent, not a how-to skill.
- `sverchok-errors-common` organizes content as Part A (7 Common Errors) and Part B (10 AI Mistakes) instead of "Essential Patterns". This is appropriate for an error catalog.
- `sverchok-syntax-scripting` organizes content by scripting node type (SNLite, SN Functor B, Formula Mk5, Profile Mk3) instead of numbered patterns. This is appropriate for a syntax reference.

**Result**: PASS for all 12 (structure variants are appropriate for skill type).

### Check 9: No Placeholder Content

Searched all SKILL.md files for: `[TODO]`, `[TBD]`, `lorem ipsum`, `placeholder`, `coming soon`.

**Zero matches found.**

**Result**: PASS for all 12.

### Check 10: Code Examples Are Syntactically Correct Python

Reviewed all Python code blocks across all 12 skills. All code examples:

- Use correct Python 3 syntax
- Import from valid Sverchok modules (`sverchok.data_structure`, `sverchok.node_tree`, etc.)
- Use correct Blender API patterns (`bpy.props`, `bpy.types.Node`, `bpy.data.node_groups`)
- Follow Sverchok conventions (class inheritance, `sv_init`, `process`, `sv_get`/`sv_set`)
- Use proper annotation syntax (`:` not `=`) for Blender properties

**Specific verifications**:
- SNLite docstring headers use correct socket aliases (`s`, `v`, `m`, `o`, `C`, `S`, `So`, `SF`, `VF`, `D`, `FP`)
- Profile Mk3 DSL examples follow correct syntax (M, L, H, V, X commands)
- Formula Mk5 examples use only whitelisted functions
- IfcSverchok patterns reference correct `SvIfcStore` API
- TopologicSverchok node patterns use correct node names

**No syntax errors detected.**

**Result**: PASS for all 12.

---

## Observations (Non-Blocking)

1. **sverchok-impl-extensions at 499 lines**: At the absolute limit. Any additions will exceed the 500-line threshold. Consider splitting if content grows.

2. **sverchok-agents-code-validator compatibility field**: Uses "Requires Python 3.x" instead of the standard Blender/Sverchok version format. Acceptable because this is a code validation agent, but differs from the pattern used by all other 11 skills.

3. **Reference link dash style**: `sverchok-impl-extensions` uses `--` (double hyphen) while all other skills use `—` (em dash) in reference link descriptions. Cosmetic only.

4. **Sverchok version variance**: `sverchok-impl-topologic` specifies `Sverchok v1.2.0+` while most other skills specify `Sverchok v1.4.0+`. This is correct (TopologicSverchok works with older Sverchok) but worth noting.

---

## Blockers to Fix

**None.** All 12 skills pass all 10 quality criteria. No blockers identified.

---

## Conclusion

All 12 Sverchok skills meet the quality standards. The skills are:
- Well-structured with consistent YAML frontmatter
- Under the 500-line limit (tightest: sverchok-impl-extensions at 499)
- Written entirely in English with deterministic language
- Version-explicit with Sverchok/Blender versions
- Complete with all reference files present and linked
- Free of placeholder content
- Populated with syntactically correct Python code examples
