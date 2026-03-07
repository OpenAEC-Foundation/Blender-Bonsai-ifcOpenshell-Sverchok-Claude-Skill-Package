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

## L-004: oa Agents Cannot Self-Spawn Child oa Agents
**Date**: 2026-03-06
**Context**: Attempted nested delegation — an oa orchestrator agent instructed to spawn sub-agents via `oa run`
**Issue**: Claude Code prefers its built-in Agent tool over bash `oa run`. Result: sub-agents run as in-session agents (invisible to `oa status`) instead of as oa-managed agents.
**Root Causes**:
1. Claude Code's Agent tool is higher priority than bash commands
2. `oa` may not be in PATH inside agent workspace
3. No mechanism to force Claude Code to use `oa run` over Agent tool
**Resolution**: Use FLAT spawning — meta-orchestrator spawns ALL agents directly, no nested delegation. Filed as issue: https://github.com/OpenAEC-Foundation/Open-Agents/issues/11
**Workaround**: Spawn all workers from the top-level session, not from oa agents.

## L-005: A2/A3 Worker Output Merge Pattern Works
**Date**: 2026-03-06
**Context**: Verifying A2 (Blender) and A3 (IfcOpenShell) enrichment worker outputs
**Finding**: The pattern of workers writing to `worker-output/` staging area, followed by orchestrator merging into main files, works correctly. Both merges verified as COMPLETE. Worker-output dir can be safely deleted after verification.
**Implication**: This pattern (stage → merge → verify → cleanup) is reliable for future multi-agent writing tasks.

## L-006: Official Skill Description Pattern Conflicts with Our "Deterministic" Prefix
**Date**: 2026-03-06
**Context**: Ecosystem research — Claude skill platform analysis (Prompt F)
**Finding**: The official Anthropic skill-creator skill explicitly recommends AGAINST overuse of ALWAYS/NEVER/MUST. Our current description pattern `"Deterministic [description]..."` wastes tokens on a non-trigger word. Official guidance: lead with third-person action verbs, include domain-specific trigger keywords, explain the "why" behind constraints.
**Action**: Remove "Deterministic" prefix from all skill descriptions. Keep deterministic language only for truly critical rules (version-specific breaking changes, known API traps). Add brief "because [reason]" to each constraint.
**Reference**: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices

## L-007: IFC Property Extraction Pattern Is Universal
**Date**: 2026-03-06
**Context**: Ecosystem research — cross-repository analysis (Prompt F)
**Finding**: The same property extraction pattern appears in every IFC-related OpenAEC repo (building.py, Monty IFC Viewer, IfcGit-4-Nextcloud, INB Template): `element.IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties → IfcPropertySingleValue.NominalValue.wrappedValue`. This is the canonical pattern regardless of whether the API is IfcOpenShell (Python) or web-ifc (TypeScript/WASM).
**Implication**: This pattern MUST be the primary example in `bonsai-syntax-properties`, `ifcos-syntax-elements`, and `ifcos-impl-extraction` skills.

## L-008: building.py Has No Blender Export — Our Skills Fill This Gap
**Date**: 2026-03-06
**Context**: Ecosystem research — building.py repository analysis (Prompt F)
**Finding**: building.py exports to IFC, Speckle, FreeCAD, Revit, and DXF but has zero Blender (bpy) integration. The GIS-to-Blender repo is an empty placeholder. This confirms our skill package fills a real gap: providing Claude with the knowledge to generate Blender/bpy code for AEC workflows.
**Implication**: The Blender skills are not redundant with existing OpenAEC tooling — they address an unserved need.

## L-010: Effective oa Agent Task Prompts Need 5 Elements
**Date**: 2026-03-07
**Context**: Spawning research and skill-creation agents via oa run --direct
**Finding**: oa agents perform significantly better when their task prompt includes ALL 5 elements:
1. **Absolute file paths** for both INPUT (what to read) and OUTPUT (where to write)
2. **Explicit scope** with bullet points of what to cover (not vague "research X")
3. **Reference files** to follow for format/structure (e.g., "read this existing skill for format")
4. **Quality rules** inline (English-only, < 500 lines, deterministic language, version-explicit)
5. **Source URLs** from SOURCES.md (approved official docs only)
Without these, agents produce inconsistent output, write to wrong locations, or miss critical content. The --direct flag is essential to prevent output getting lost in /tmp.
**Implication**: The task prompt IS the agent's full context. There's no "conversation" — it must be self-contained.

## L-011: Phases Can Be Overlapped When Dependencies Allow
**Date**: 2026-03-07
**Context**: Sverchok skill package development — Phase S3 + S5 overlap
**Finding**: The 7-phase methodology doesn't require strict sequential execution. When a skill has NO dependencies on later phases, it can start before the phase is "officially" complete. Example: `sverchok-core-concepts` (Batch 1, no deps) was started while the definitive masterplan was still being written — the research was sufficient.
**Implication**: Critical path analysis of the dependency graph determines what can be parallelized. Foundation skills can always start early.

## L-009: Skill Template Needs Additional Frontmatter Fields
**Date**: 2026-03-06
**Context**: Ecosystem research — gap analysis against Agent Skills open standard (Prompt F)
**Finding**: Our current SKILL.md template is missing `license`, `compatibility`, and `metadata` fields that the official specification supports. Adding these improves discoverability and compliance. Recommended additions: `license: MIT`, `compatibility: Designed for Claude Code. Requires Python 3.x.`, `metadata: { author: OpenAEC-Foundation, version: "1.0" }`.
**Action**: Update the SKILL.md template before starting skill creation (Phase 5).
