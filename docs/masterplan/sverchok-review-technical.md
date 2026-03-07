# Technical Review — Sverchok Skill Masterplan

**Date**: 2026-03-07
**Reviewer**: sv-review-technical agent
**Input**: sverchok-masterplan-raw.md (327 lines), vooronderzoek-sverchok.md (2352 lines)
**Status**: COMPLETE

---

## Review Checklist Results

### 1. API Surface Coverage (§1–§15)

| Research Section | Lines | Mapped Skill(s) | Status |
|-----------------|-------|-----------------|--------|
| §1 Sverchok Overview & Architecture | L30–120 | sverchok-core-concepts | COVERED |
| §2 Node System & Data Flow | L122–288 | sverchok-core-concepts | COVERED |
| §3 Socket Type System | L290–431 | sverchok-syntax-sockets | COVERED |
| §4 Data Nesting Levels (CRITICAL) | L435–520 | sverchok-syntax-data | COVERED |
| §5 List Matching & Vectorization | L522–716 | sverchok-syntax-data | COVERED |
| §6 Node Categories | L720–778 | sverchok-core-concepts, sverchok-impl-parametric | COVERED |
| §7 Python Scripting | L782–1199 | sverchok-syntax-scripting | COVERED |
| §8 Custom Node Development | L1202–1465 | sverchok-impl-custom-nodes | COVERED |
| §9 External API Access | L1469–1598 | sverchok-syntax-api | COVERED |
| §10 IfcSverchok | L1614–1800 | sverchok-impl-ifcsverchok | COVERED |
| §11 Extensions Ecosystem | L1804–1936 | sverchok-impl-extensions | COVERED |
| §12 Common Error Patterns | L1939–2026 | sverchok-errors-common | COVERED |
| §13 AI Common Mistakes | L2028–2141 | sverchok-errors-common | COVERED |
| §14 Real-World Usage Patterns | L2145–2287 | sverchok-impl-parametric | COVERED |
| §15 Advanced Data Types | L2291–2333 | sverchok-syntax-sockets | COVERED |

**Result**: All 15 research sections are mapped to at least one skill. No orphaned API surfaces found.

---

### 2. Skill Size Assessment (> 450 line threshold)

| Skill | Est. Lines | Verdict |
|-------|-----------|---------|
| sverchok-core-concepts | ~350 | OK |
| sverchok-syntax-sockets | ~300 | OK |
| sverchok-syntax-data | ~400 | OK |
| sverchok-syntax-scripting | ~450 | AT LIMIT |
| sverchok-syntax-api | ~250 | OK |
| sverchok-impl-custom-nodes | ~400 | OK |
| sverchok-impl-parametric | ~300 | OK |
| sverchok-impl-ifcsverchok | ~400 | OK |
| sverchok-impl-extensions | ~300 | OK |
| sverchok-errors-common | ~350 | OK |
| sverchok-agents-code-validator | ~250 | OK |

- **ISSUE**: `sverchok-syntax-scripting` is estimated at exactly 450 lines, at the threshold. The source research (§7, L782–1199 = 417 lines of dense content) covers SNLite (socket declaration, type identifiers, options, built-in aliases, special functions, custom enums, file handler, includes, templates, persistent storage), SN Functor B, Formula Mk5, Profile Mk3, AND 6 other script nodes. This is a substantial amount of material.
- **LOCATION**: sverchok-syntax-scripting (SV-S-03)
- **RECOMMENDATION**: Monitor during skill creation. If the skill exceeds 450 lines, split into `sverchok-syntax-snlite` (SNLite + SN Functor B, the primary scripting nodes) and `sverchok-syntax-formula-profile` (Formula Mk5 + Profile Mk3 + other script nodes). A natural split point exists between line 1063 and 1064 of the research.
- **SEVERITY**: WARNING

---

### 3. Scope Overlap Analysis

#### 3a. Socket Processing Flags/Modes — Dual Coverage

- **ISSUE**: Socket data processing flags (flatten, simplify, graft, unwrap, wrap) appear in both `sverchok-syntax-sockets` (scope: "socket data processing flags") and `sverchok-syntax-data` (scope: "socket processing modes (input preprocessing: flatten/simplify/graft/unwrap/wrap, output postprocessing)"). Both skills document the same mechanism from different angles.
- **LOCATION**: sverchok-syntax-sockets (SV-S-01) + sverchok-syntax-data (SV-S-02)
- **RECOMMENDATION**: Assign clear ownership. `sverchok-syntax-sockets` should define the flag properties on socket objects (what they are, where they live). `sverchok-syntax-data` should document the preprocessing/postprocessing pipeline behavior (how they transform data). Add a cross-reference note in the masterplan.
- **SEVERITY**: NOTE

#### 3b. TopologicSverchok — Dual Coverage

- **ISSUE**: TopologicSverchok is covered in both `sverchok-impl-parametric` ("TopologicSverchok for building analysis (CellComplex, space adjacency, dual graph)") and `sverchok-impl-extensions` (detailed class hierarchy, 200+ nodes, AEC relevance). This creates ambiguity about which skill is authoritative.
- **LOCATION**: sverchok-impl-parametric (SV-I-02) + sverchok-impl-extensions (SV-I-04)
- **RECOMMENDATION**: `sverchok-impl-extensions` should own the TopologicSverchok library documentation (class hierarchy, node categories, architecture). `sverchok-impl-parametric` should only reference TopologicSverchok as a workflow example ("for building analysis, see sverchok-impl-extensions") without duplicating library-level details.
- **SEVERITY**: WARNING

#### 3c. match_long_repeat / fullList — Referenced in Multiple Skills

- **ISSUE**: The core list matching functions (`match_long_repeat`, `fullList`, `repeat_last`) appear in `sverchok-syntax-data` (primary definition), `sverchok-syntax-api` (API reference), and `sverchok-impl-custom-nodes` (process method pattern).
- **LOCATION**: sverchok-syntax-data, sverchok-syntax-api, sverchok-impl-custom-nodes
- **RECOMMENDATION**: This is acceptable — `syntax-data` owns the authoritative documentation, while `syntax-api` and `impl-custom-nodes` reference them in usage context. No action needed, but skill authors should cross-reference rather than re-documenting.
- **SEVERITY**: NOTE

---

### 4. Dependency Correctness

#### Dependency Graph Validation

```
Batch 1: core-concepts (no deps)                    ✓
Batch 2: syntax-sockets (← core), syntax-data (← core)   ✓
Batch 3: syntax-scripting (← sockets, data),              ✓
         syntax-api (← sockets, data),                     ✓
         impl-custom-nodes (← sockets, data, core)         ✓
Batch 4: impl-parametric (← scripting, data, sockets),    ✓
         impl-ifcsverchok (← custom-nodes, sockets, data), ✓
         impl-extensions (← custom-nodes)                   ✓
Batch 5: errors-common (← custom-nodes, scripting, data)  ⚠
Batch 6: agents-code-validator (← errors-common, all impl) ✓
```

**Circular dependencies**: None found. ✓

#### 4a. Missing Dependency — errors-common → impl-ifcsverchok

- **ISSUE**: `sverchok-errors-common` documents 4 IfcSverchok-specific error patterns (§12.6: Undo Crashes, §12.7: Purge on Re-run, §13.9: Entity ID Persistence, §13.10: Double-Nested Lists) but does NOT list `sverchok-impl-ifcsverchok` as a dependency. The explicit dependency list is: `sverchok-impl-custom-nodes, sverchok-syntax-scripting, sverchok-syntax-data`.
- **LOCATION**: sverchok-errors-common (SV-E-01)
- **RECOMMENDATION**: Add `sverchok-impl-ifcsverchok` to the dependency list of `sverchok-errors-common`. The batch ordering (Batch 5 after Batch 4) already ensures the correct build sequence, but the explicit dependency metadata should be accurate for traceability.
- **SEVERITY**: WARNING

#### 4b. Vague Dependency — agents-code-validator

- **ISSUE**: The dependency for `sverchok-agents-code-validator` states "sverchok-errors-common, all impl skills" — "all impl skills" is imprecise.
- **LOCATION**: sverchok-agents-code-validator (SV-A-01)
- **RECOMMENDATION**: Replace "all impl skills" with explicit list: `sverchok-errors-common, sverchok-impl-custom-nodes, sverchok-impl-ifcsverchok, sverchok-impl-parametric, sverchok-impl-extensions, sverchok-syntax-scripting`.
- **SEVERITY**: NOTE

---

### 5. Anti-Pattern Coverage in sverchok-errors-common

#### §12 Common Error Patterns (7 items)

| # | Pattern | Covered in errors-common? |
|---|---------|--------------------------|
| 12.1 | Nesting level errors (wrong level → silent corruption) | ✓ |
| 12.2 | Missing updateNode callback | ✓ |
| 12.3 | Not checking output connections | ✓ |
| 12.4 | Socket data mutation (modifying input in-place) | ✓ |
| 12.5 | List matching misunderstandings (zip vs match_long_repeat) | ✓ |
| 12.6 | IfcSverchok undo crashes | ✓ |
| 12.7 | IfcSverchok purge on re-run (entity IDs change) | ✓ |

#### §13 AI Common Mistakes (10 items)

| # | Pattern | Covered in errors-common? |
|---|---------|--------------------------|
| 13.1 | Outputting flat data | ✓ |
| 13.2 | Missing object wrapper for vertices | ✓ |
| 13.3 | Using `__init__` instead of `sv_init` | ✓ |
| 13.4 | Wrong SNLite socket type aliases | ✓ |
| 13.5 | Forgetting updateNode | ✓ (overlaps 12.2) |
| 13.6 | Not matching lists before zipping | ✓ (overlaps 12.5) |
| 13.7 | Creating sockets in `process()` | ✓ |
| 13.8 | Assuming matrix data is nested | ✓ |
| 13.9 | IfcSverchok entity ID persistence | ✓ |
| 13.10 | IfcSverchok single vs double nesting | ✓ |

**Result**: All 17 anti-patterns (7 + 10) are captured. ✓

---

### 6. Batch Order Optimality

Current plan: **6 batches**, max 3 parallel agents per batch.

| Batch | Parallelism | Duration factor |
|-------|------------|----------------|
| 1 | 1 agent | 1× |
| 2 | 2 agents | 1× |
| 3 | 3 agents | 1× |
| 4 | 3 agents | 1× |
| 5 | 1 agent | 1× |
| 6 | 1 agent | 1× |

**Critical path analysis**: core → sockets/data → scripting/api/custom-nodes → parametric/ifc/extensions → errors → agents = 6 sequential steps. This is the minimum possible given the dependency chain.

- **ISSUE**: Could `errors-common` be parallelized with Batch 4? It explicitly depends on Batch 3 skills only (custom-nodes, scripting, data). However, it documents IfcSverchok errors which require `impl-ifcsverchok` knowledge. After fixing the missing dependency (issue 4a), errors-common correctly belongs in Batch 5.
- **LOCATION**: Batch execution plan
- **RECOMMENDATION**: No change needed. The 6-batch plan is the minimum possible given the true dependency chain. Batch order is optimal.
- **SEVERITY**: NOTE

---

### 7. Nesting Level System Coverage in sverchok-syntax-data

The skill scope explicitly covers:
- ✓ Nesting level convention (levels 0–3)
- ✓ Standard formats per socket type (SvStringsSocket=L2, SvVerticesSocket=L3, SvMatrixSocket=L1)
- ✓ Nesting level detection (`get_data_nesting_level`)
- ✓ The "objects" mental model (outermost list = objects)
- ✓ List matching system (5 modes: REPEAT, CYCLE, SHORT, XREF, XREF2)
- ✓ Core matching functions
- ✓ NumPy matching variants
- ✓ `vectorize` decorator with annotation-based level detection
- ✓ `SvRecursiveNode` mixin with `nesting_level`, `default_mode`, `pre_processing`
- ✓ Socket processing modes (flatten/simplify/graft/unwrap/wrap)

**Key API surface** includes all critical functions: `match_long_repeat()`, `fullList()`, `repeat_last()`, `get_data_nesting_level()`, `list_match_func`, `vectorize()`, `SvRecursiveNode`, `match_sockets()`, `recurse_fx()`, `recurse_fxy()`.

- **ISSUE**: The scope mentions "SvStringsSocket=level 2, SvVerticesSocket=level 3, SvMatrixSocket=level 1" but does not explicitly mention edge data (level 2 via SvStringsSocket) and face data (level 2 via SvStringsSocket). Given that nesting is the #1 error source, edge/face data formats should be explicitly called out in the skill scope.
- **LOCATION**: sverchok-syntax-data (SV-S-02)
- **RECOMMENDATION**: Add "edge data=level 2, face data=level 2 (both via SvStringsSocket)" to the scope description. This is documented in the research (§4.2, L473–487) and is essential for mesh-related workflows.
- **SEVERITY**: NOTE

**Result**: Coverage is thorough. The nesting level system is appropriately prioritized as CRITICAL with complexity L. ✓

---

### 8. IfcSverchok Skill Comprehensiveness

The skill scope covers all key aspects from §10:
- ✓ Overview and dependency chain (Bonsai + Sverchok + ifcopenshell)
- ✓ SvIfcStore transient file management (purge, get_file, id_map, use_bonsai_file)
- ✓ SvIfcCore node base class
- ✓ Complete node catalog (31 nodes: 24 IFC + 7 Shape Builder)
- ✓ Two geometry modes (BMesh → IFC repr, Sverchok → IFC repr)
- ✓ 6-step IFC file generation workflow
- ✓ Automatic hierarchy completion (ensure_hirarchy)
- ✓ Working example (simple IFC wall)
- ✓ Bonsai integration (use_bonsai_file toggle)
- ✓ Known issues and limitations (6 documented)
- ✓ Cross-package dependencies correctly identified (ifcos-core-concepts, ifcos-syntax-elements, ifcos-impl-creation)

- **ISSUE**: The `SvIfcCore.process()` method uses a unique double-nested input processing pattern (`zip_long_repeat` applied twice, research §10.2 L1674–1686). This differs from standard Sverchok node processing and is a source of confusion. It should be explicitly highlighted in the skill scope.
- **LOCATION**: sverchok-impl-ifcsverchok (SV-I-03)
- **RECOMMENDATION**: Add "SvIfcCore double-nested input processing pattern" as an explicit scope item. This unique pattern explains why IfcSverchok nodes expect inputs formatted differently from standard nodes.
- **SEVERITY**: NOTE

**Result**: The IfcSverchok skill is comprehensive for the BIM workflow. ✓

---

## Additional Issues

### 9. sv_deep_copy Not Explicitly Assigned

- **ISSUE**: The `sv_deep_copy` function (§2.5, L270–278) is a performance-relevant optimization — a custom deep copy faster than Python's `copy.deepcopy` for nested list structures. It's used internally by the socket data cache but understanding when `deepcopy=True` vs `False` matters is important for node developers. No skill scope explicitly names this function.
- **LOCATION**: sverchok-core-concepts (SV-C-01) or sverchok-syntax-sockets (SV-S-01)
- **RECOMMENDATION**: Include `sv_deep_copy` in the key API surface of `sverchok-core-concepts` (where `socket_data_cache` is documented) or `sverchok-syntax-sockets` (where `sv_get(deepcopy=True)` is documented).
- **SEVERITY**: NOTE

### 10. §15 Missing from Research Table of Contents

- **ISSUE**: The research document's Table of Contents (L11–27) lists §1–§14 but the body contains §15 (Advanced Data Types, L2291–2333). The masterplan correctly references §15 for `sverchok-syntax-sockets`.
- **LOCATION**: vooronderzoek-sverchok.md (research document)
- **RECOMMENDATION**: Not a masterplan issue — the masterplan correctly handles §15. Noted for completeness only. The research ToC should be updated by the research team.
- **SEVERITY**: NOTE

---

## Issue Summary

| # | Issue | Location | Severity |
|---|-------|----------|----------|
| 1 | `sverchok-syntax-scripting` at 450-line limit | SV-S-03 | WARNING |
| 2 | TopologicSverchok dual coverage | SV-I-02 + SV-I-04 | WARNING |
| 3 | `errors-common` missing dependency on `impl-ifcsverchok` | SV-E-01 | WARNING |
| 4 | Socket processing flags/modes dual coverage | SV-S-01 + SV-S-02 | NOTE |
| 5 | match_long_repeat referenced in multiple skills | SV-S-02 + SV-S-04 + SV-I-01 | NOTE |
| 6 | Vague "all impl skills" dependency | SV-A-01 | NOTE |
| 7 | Edge/face data formats not explicit in syntax-data scope | SV-S-02 | NOTE |
| 8 | SvIfcCore double-nested processing not highlighted | SV-I-03 | NOTE |
| 9 | `sv_deep_copy` not in any skill's key API surface | SV-C-01 or SV-S-01 | NOTE |
| 10 | §15 missing from research ToC | vooronderzoek-sverchok.md | NOTE |

**Warnings**: 3
**Notes**: 7
**Critical**: 0

---

## Verdict: PASS

No critical issues found. The masterplan correctly maps all 15 research sections to 11 skills with appropriate granularity. All 17 anti-patterns from §12 and §13 are captured. The batch order is optimal for the dependency chain. The nesting level system and IfcSverchok BIM workflow are comprehensively covered.

The 3 warnings should be addressed before skill creation begins:
1. Monitor `sverchok-syntax-scripting` size during authoring; have a split plan ready.
2. Clarify TopologicSverchok ownership between `impl-parametric` and `impl-extensions`.
3. Add `impl-ifcsverchok` to the explicit dependency list of `errors-common`.

---

*Review completed by sv-review-technical agent. Date: 2026-03-07.*
