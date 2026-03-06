# DECISIONS

Architectural and process decisions with rationale. Each decision is numbered and immutable once recorded. New decisions may supersede old ones but old ones are never deleted.

---

## D-001: 7-Phase Research-First Methodology
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Need a structured approach to build high-quality skills
**Decision**: Adopt the 7-phase methodology proven in the ERPNext Skill Package
**Rationale**: ERPNext project successfully produced 28 domain skills with this approach. Research-first prevents hallucinated content.
**Reference**: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package/blob/main/WAY_OF_WORK.md

## D-002: Separate Packages Per Technology
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Skills cover 4+ technologies that may be used independently
**Decision**: Each technology gets its own directory under skills/ (blender/, bonsai/, ifcopenshell/, sverchok/)
**Rationale**: Users working only with Blender shouldn't need to install Bonsai skills. Each package must be independently installable.
**Implication**: Cross-references between packages must be optional, not required

## D-003: English-Only Skills
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Team works primarily in Dutch, skills target international audience
**Decision**: ALL skill content in English only
**Rationale**: Skills are instructions for Claude, not end-user documentation. Claude reads English and responds in any language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext project.
**Reference**: ERPNext LESSONS_LEARNED.md, lesson on English-only skills

## D-004: Orchestrator-First Delegation
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Need to produce ~43 skills efficiently within tight timeline
**Decision**: Use orchestrator-first pattern with Open-Agents (oa-cli) for parallel execution
**Rationale**: Main session coordinates only, never does work directly. Workers are delegated via `oa run`. Proven in Open-Agents project (lessons L-001 through L-032).
**Reference**: https://github.com/OpenAEC-Foundation/Open-Agents/blob/main/LESSONS.md

## D-005: MIT License
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Need to choose open-source license
**Decision**: MIT License
**Rationale**: Most permissive, maximizes adoption. Consistent with OpenAEC Foundation philosophy.

## D-006: ROADMAP.md as Single Source of Truth
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Need to track project status across multiple sessions and agents
**Decision**: ROADMAP.md is the ONLY place where project status is tracked
**Rationale**: Multiple status locations cause drift and confusion. Single source prevents "which is current?" questions. Proven in both ERPNext and Open-Agents projects.

## D-007: Inter-Agent Communication for Quality
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Parallel agents may produce work with cross-dependencies
**Decision**: Use `oa send`/`oa inbox`/`oa broadcast` for agent-to-agent quality validation
**Rationale**: Agent B that depends on Agent A's output should validate it before building on it. Peer review between writers catches cross-reference inconsistencies.

## D-008: Build Order - Blender First
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Need to decide which technology to develop first
**Decision**: Blender -> Bonsai -> IfcOpenShell -> Sverchok (later)
**Rationale**: Blender is the foundation (Bonsai runs inside Blender, IfcOpenShell is used by Bonsai). Building bottom-up ensures dependencies are available.

## D-009: Skill Line Limit: 500 Lines
**Date**: 2026-03-05
**Status**: ACTIVE
**Context**: Skills need to be concise enough for Claude to load efficiently
**Decision**: SKILL.md must be under 500 lines. Heavy content goes in references/
**Rationale**: Anthropic convention. ERPNext skills ranged 180-427 lines, all well under 500. Keeps the main skill focused on decision trees and quick reference.

## D-010: Cross-Tech Skills Minimal — No Redundancy with Per-Technology Skills
**Date**: 2026-03-06
**Status**: ACTIVE
**Context**: Phase 3 masterplan refinement identified overlap between proposed cross-tech skills and per-technology skills
**Decision**: Cross-tech package contains only 2 skills: `aec-core-bim-workflows` (cross-cutting BIM patterns) and `aec-workflow-orchestrator` (routing agent). IFC fundamentals and Python runtime are covered by their respective technology packages.
**Rationale**: `aec-core-ifc-fundamentals` is redundant with `ifcos-core-concepts`. `aec-core-python-runtime` is redundant with `blender-core-runtime` and `ifcos-core-runtime`. Keeping redundant cross-tech skills would violate the DRY principle and create maintenance burden.

## D-011: Blender and IfcOpenShell Built in Parallel
**Date**: 2026-03-06
**Status**: ACTIVE
**Context**: D-008 specifies Blender → Bonsai → IfcOpenShell build order, but Blender and IfcOpenShell have no mutual dependencies
**Decision**: Blender and IfcOpenShell skills are built in parallel within each batch. Bonsai waits for both. This refines D-008 without contradicting it.
**Rationale**: IfcOpenShell does not depend on Blender. Building them in parallel cuts total batch count from ~20 to 13. Bonsai depends on both, so it starts only after both have core + syntax skills ready.
