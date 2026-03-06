# ROADMAP

## Current Phase: 4/5 - Topic Research + Skill Creation (READY TO START)
## Overall Progress: 45%

## Phase Status
| Phase | Status | Progress |
|-------|--------|----------|
| 1. Setup + Raw Masterplan | COMPLETE | 100% |
| 2. Deep Research (vooronderzoek) | COMPLETE | 100% |
| 3. Masterplan Refinement | COMPLETE | 100% |
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

## Skill Inventory (DEFINITIVE — finalized in Phase 3, fixed post-review)
| Technology | Skills | Status | Notes |
|------------|--------|--------|-------|
| Blender | 26 | Defined in masterplan | 11 syntax, 6 impl, 3 errors, 4 core, 2 agents |
| Bonsai | 14 | Defined in masterplan | 4 syntax, 7 impl, 1 errors, 1 core, 1 agents |
| IfcOpenShell | 19 | Defined in masterplan | 4 syntax, 9 impl, 3 errors, 2 core, 1 agents |
| Sverchok | ~8 | DEFERRED (later phase) | Requires dedicated research phase |
| Cross-Tech | 2 | Defined in masterplan | 1 core (bim-workflows), 1 agents (orchestrator) |
| **Total** | **61** | **0% skills created** | See docs/masterplan/masterplan.md |

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
- 2026-03-06: Phase 3 COMPLETE — definitive masterplan written (60 skills, 13 batches)
- 2026-03-06: Skill inventory finalized: 26 Blender + 18 IfcOpenShell + 14 Bonsai + 2 Cross-tech = 60
- 2026-03-06: Cross-tech reduced from 4 to 2 skills (redundancies removed)
- 2026-03-06: Decisions D-010, D-011 added
- 2026-03-06: Masterplan reviewed (3 agents), fixed: 4 batch dependency blockers, +1 skill (ifcos-impl-validation), naming conventions, arithmetic
- 2026-03-06: CLAUDE.md + Blender MCP research completed (research-claude-md-and-mcp.md, research-blender-mcp.md)
- 2026-03-06: Total skills: 61 (26 Blender + 19 IfcOpenShell + 14 Bonsai + 2 Cross-tech)

## Next Steps
1. **IMMEDIATE**: Design template CLAUDE.md for the skills package (guides Claude when using Blender/Bonsai/IfcOpenShell)
2. **Phase 5**: Begin skill creation using masterplan batch plan (13 batches, 3-5 agents per batch)
   - Start with Batch 1: foundation skills (blender-core-api, blender-core-versions, ifcos-core-concepts, ifcos-syntax-fileio)
   - Use Prompt C template with skill-specific parameters from masterplan §5
   - Quality gate after each batch using Prompt D / masterplan §6
3. **Blender MCP integration**: Research Blender MCP server for live Claude-to-Blender interaction
4. **All research prompts DONE**: A, A2, A3, B, E, F, G complete — see session-prompts.md
