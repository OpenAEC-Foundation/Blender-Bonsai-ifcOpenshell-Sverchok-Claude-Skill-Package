# ROADMAP

## Current Phase: 3 - Masterplan Refinement (READY TO START)
## Overall Progress: 35%

## Phase Status
| Phase | Status | Progress |
|-------|--------|----------|
| 1. Setup + Raw Masterplan | COMPLETE | 100% |
| 2. Deep Research (vooronderzoek) | COMPLETE | 100% |
| 3. Masterplan Refinement | READY TO START | 0% |
| 4. Topic-Specific Research | PARTIALLY STARTED | 15% |
| 5. Skill Creation | TODO | 0% |
| 6. Validation | TODO | 0% |
| 7. Publication | TODO | 0% |

## Phase 2 Detail: Research Status (ALL COMPLETE)
| Technology | Vooronderzoek | Size | Status |
|------------|--------------|------|--------|
| Blender | vooronderzoek-blender.md | 54KB | COMPLETE |
| Bonsai | vooronderzoek-bonsai.md | 54KB | COMPLETE |
| IfcOpenShell | vooronderzoek-ifcopenshell.md | 35KB | COMPLETE |
| Sverchok | - | - | LATER PHASE |

## Phase 4 Early Start: IfcOpenShell Topic Research
| Document | Size | Status |
|----------|------|--------|
| fragments/ifcos-api-categories.md | 42KB | Complete |
| fragments/ifcos-core-operations.md | 31KB | Complete |
| fragments/ifcos-errors-performance.md | 37KB | Complete |
| fragments/ifcos-schema-versions.md | 35KB | Complete |
| topic-research/ifcos-core-operations.md | 49KB | Complete |
| topic-research/ifcos-errors-performance-research.md | 50KB | Complete |
| topic-research/ifcos-schema-version-comparison.md | 53KB | Complete |

## Skill Inventory (preliminary, finalized in Phase 3)
| Technology | Estimated Skills | Status | Notes |
|------------|-----------------|--------|-------|
| Blender | ~25 (was ~15) | Scope expanded after gap analysis | Added: nodes, animation, rendering, data, gpu, versions |
| Bonsai | ~12+ | Research complete, gaps identified | Missing: drawing, QTO, BCF, clash, MEP |
| IfcOpenShell | ~13+ | Research complete + topic research started | Missing: cost, scheduling, MEP, drawing |
| Sverchok | ~8 | Preliminary inventory, research pending | Includes IfcSverchok (IFC bridge) |
| Cross-Tech | ~5 (was ~3) | Not started | Added: python-runtime, sverchok-bonsai bridge |
| **Total** | **~63+** | **0% skills created** | Scope analysis: docs/research/scope-analysis.md |

## Scope Analysis (2026-03-05)
Key findings from complete API surface mapping:
- **Blender**: 50+ operator categories, 30+ data types, 8+ standalone modules. Current research covers ~40% of Python API surface. Major gaps: node systems, animation, materials, rendering, simulation, I/O.
- **IfcOpenShell**: 30+ api sub-modules, 15+ util sub-modules. Current research covers ~50%. Gaps: cost, scheduling, MEP, drawing, validation.
- **Bonsai**: 40+ bim/module directories. Current research covers ~35%. Gaps: drawing, QTO, BCF, clash, MEP, facility management.
- **Sverchok**: 25+ node categories, 3 scripting modes. ZERO coverage (deferred).
- **Python runtime differences**: Blender Python is embedded CPython with restrictions (no direct threading, restricted context in callbacks/handlers, undo invalidates references). IfcOpenShell uses C++ bindings. Skills MUST cover these runtime quirks.
- See full analysis: `docs/research/scope-analysis.md`

## Changelog
- 2026-03-05: Project initialized, git repo created, pushed to GitHub
- 2026-03-05: Directory structure with separate technology packages
- 2026-03-05: CLAUDE.md with protocols P-001 through P-008
- 2026-03-05: Core docs suite: ROADMAP, WAY_OF_WORK, LESSONS, REQUIREMENTS, DECISIONS, SOURCES, CHANGELOG, CONTRIBUTING, SECURITY
- 2026-03-05: Raw masterplan created in docs/masterplan/
- 2026-03-05: GitHub repo configured (description, topics, MIT license)
- 2026-03-05: Blender vooronderzoek completed (54KB)
- 2026-03-05: IfcOpenShell vooronderzoek completed (35KB) + 7 topic research docs
- 2026-03-05: CLAUDE.md rewritten with action-oriented core file references in all protocols
- 2026-03-05: SOURCES.md enriched with per-version release notes, OpenAEC projects, Claude platform docs
- 2026-03-05: Bonsai vooronderzoek completed (54KB) - Phase 2 COMPLETE
- 2026-03-05: Session prompts updated with source-specific URLs (6 prompts: A, A2, A3, B, C, D, E)
- 2026-03-05: Scope analysis completed - mapped ALL Python API surfaces for 4 technologies
- 2026-03-05: Blender skill inventory expanded ~15 → ~25 (nodes, animation, rendering, gpu, versions)
- 2026-03-05: REQUIREMENTS.md updated with Blender 5.x + expanded key areas
- 2026-03-05: 4 lessons consolidated to AI deployment repo (DEV_003, DEV_004, GH_004, CC_009)
- 2026-03-05: Prompt G added for supplementary research on gap domains

## Next Steps
1. **PRIORITY**: Run Prompt G (supplementary research) to cover identified gaps
   - Blender: node systems, animation/rigging, materials, rendering, I/O formats
   - IfcOpenShell: cost, scheduling, MEP, drawing, validation
   - Bonsai: drawing, QTO, BCF, clash detection
   - Python runtime quirks per technology
2. **Then Phase 3**: Use Prompt B to refine masterplan with ALL research + scope analysis
   - Read all 3 vooronderzoeken (143KB) + scope-analysis.md + supplementary research
   - Finalize skill inventory based on complete API surface
   - Decide scope boundaries (AEC-relevant vs skip)
   - Write ready-to-use prompts per skill with SOURCES.md URLs
   - Define parallel execution batches
3. **Optional parallel**: Use Prompt A2/A3 to enrich existing research with new sources
4. **Optional parallel**: Use Prompt E to verify all SOURCES.md URLs
5. **After Phase 3**: Begin Phase 4/5 skill creation in parallel batches
6. **Parallel track**: oa-cli testing in WSL Ubuntu
