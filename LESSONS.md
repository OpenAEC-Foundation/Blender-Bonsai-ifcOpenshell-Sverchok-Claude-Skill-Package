# LESSONS LEARNED

## L-001: oa-cli fcntl Module Not Available on Windows
**Date**: 2026-03-05
**Context**: Testing `oa` command on Windows 11
**Issue**: `ModuleNotFoundError: No module named 'fcntl'` - fcntl is Unix-only
**Impact**: oa-cli cannot run on Windows without modification
**Resolution**: Needs Windows-compatible file locking (e.g., msvcrt or portalocker)
**Source repo**: https://github.com/OpenAEC-Foundation/Open-Agents

## L-002: ERPNext Skill Package as Proven Template
**Date**: 2026-03-05
**Context**: Planning methodology for this project
**Finding**: The ERPNext Skill Package successfully produced 28 domain skills using a 7-phase methodology documented in WAY_OF_WORK.md. Key success factors:
- Research BEFORE action (never create skills on assumptions)
- Deterministic content (ALWAYS/NEVER, not "you might consider")
- English-only skills (Claude reads English, responds in any language)
- SKILL.md < 500 lines, heavy content in references/
- ROADMAP.md as single source of truth
- Commit after every phase
**Reference**: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package

## L-003: Skill Inventory is a Starting Point, Not Final
**Date**: 2026-03-05
**Context**: Defining skill list before research
**Finding**: The preliminary skill inventory (~43 skills) is an estimate. The definitive list MUST be established after deep research (Phase 2) and validated in Phase 3. Skills may be added, merged, or removed based on actual API surface and user needs.
