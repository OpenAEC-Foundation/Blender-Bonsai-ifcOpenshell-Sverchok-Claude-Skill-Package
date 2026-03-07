# Open-Agents (oa-cli) Improvements Report

**Date**: 2026-03-07
**Author**: sv-oa-research agent
**Source**: GitHub issues analysis + session workflow lessons

---

## 1. Summary of ALL Open Issues on Open-Agents Repo

### Issue #9: Bug — oa agents ignore `oa run` delegation and use Claude Code Agent tool instead
- **Status**: OPEN
- **Author**: FreekHeijting
- **Type**: Bug
- **Summary**: When agents are instructed to delegate via `oa run`, they use Claude Code's built-in Agent tool instead. Sub-agents become invisible to `oa status`, can't use `oa send`/`oa inbox`, and output lands in `/tmp`.
- **Root Cause**: Claude Code's Agent tool is a first-class tool in its palette and takes priority over bash-based `oa run`. The instruction to use `oa run` competes with Claude Code's own delegation patterns.
- **Proposed Solutions**:
  1. Stronger CLAUDE.md injection with CRITICAL-level instructions to use `oa run`
  2. Project-level CLAUDE.md override
  3. `--disallowedTools Agent` flag (nuclear option — disables Agent tool entirely)
- **Workaround**: Use FLAT spawning — meta-orchestrator spawns ALL agents directly, no nested delegation.

### Issue #10: Bug — oa agents write output to /tmp, `--direct` flag not default
- **Status**: OPEN
- **Author**: FreekHeijting
- **Type**: Bug
- **Summary**: Agent output written to `/tmp/oa-agent-*/` is volatile, invisible to users, and lost on reboot. The `--direct` flag exists but isn't the default. Orchestrators spawning sub-workers don't pass `--direct`.
- **Impact**: 176KB of research output was stranded in `/tmp` and had to be manually rescued.
- **Proposed Solutions**:
  1. **Option A (recommended)**: Make `--direct` the default behavior
  2. **Option B**: `oa collect --files` command to copy output from `/tmp` to repo
  3. **Option C**: Change `shared_results_dir` in spawner.py to use project root instead of `/tmp`
  4. **Option D**: Post-completion hook that auto-copies `output/` after `.done` marker

### Issue #11: Feature — Nested agent spawning (oa agent spawns child oa agents)
- **Status**: OPEN
- **Author**: FreekHeijting
- **Type**: Feature Request
- **Summary**: Directly related to #9. When an oa orchestrator agent should spawn child oa agents, it uses Claude Code's internal Agent tool instead. Children are invisible to `oa status`.
- **Proposed Solutions**:
  1. CLAUDE.md injection with explicit `oa run` instructions
  2. PATH setup in agent startup
  3. `oa delegate` command for native orchestration
  4. `--can-spawn` flag for agent configuration

### Issue #12: Feature — Structured task prompt template for `oa run`
- **Status**: OPEN
- **Author**: FreekHeijting
- **Type**: Feature Request
- **Summary**: Free-form task prompts produce inconsistent agent output. 5 essential prompt elements identified through 61+ skills development:
  1. **Absolute file paths** — INPUT and OUTPUT locations
  2. **Explicit scope** — bullet-pointed list of what to cover
  3. **Reference files** — existing file as format template
  4. **Quality rules** — inline constraints (English-only, line limits, etc.)
  5. **Source URLs** — approved official documentation
- **Proposed Implementation**: `--template` flag or prompt builder that ensures all 5 elements present
- **Additional Patterns**: `--direct` should be DEFAULT, phase overlap is possible, research-fragments → merge → review → create is a proven pipeline

---

## 2. Concrete Workflow Improvements Discovered

### 2.1 The 5-Element Task Prompt (L-010)
Every `oa run` prompt MUST include:
1. Absolute file paths (input + output)
2. Explicit scope (bullet list)
3. Reference files (format template)
4. Quality rules (inline constraints)
5. Source URLs (approved docs only)

**Impact**: Dramatically reduces inconsistent agent output, wrong file locations, and missed content.

### 2.2 Flat Spawning Pattern (L-004)
Never nest oa agents inside oa agents. Always spawn from the top-level session:
```
Meta-orchestrator (Claude Code session)
├── worker-1 (oa agent)
├── worker-2 (oa agent)
└── worker-3 (oa agent)
```
NOT:
```
Meta-orchestrator → orchestrator (oa) → worker (oa)  ← BROKEN
```

### 2.3 Always Use `--direct` Flag (Issue #10)
Every `oa run` command must include `--direct` to prevent output getting lost in `/tmp`. This should become the default in oa-cli.

### 2.4 Phase Overlap When Dependencies Allow (L-011)
The 7-phase methodology doesn't require strict sequential execution. Foundation skills (no deps) can start before the phase is "officially" complete. Critical path analysis determines parallelism.

### 2.5 Stage → Merge → Verify → Cleanup Pattern (L-005)
Workers write to staging area, orchestrator merges into main files. Pattern proven reliable for multi-agent writing tasks.

### 2.6 3-5 Agents Per Batch Optimal (Workflow doc)
Optimal parallelism without overwhelming QA. Quality gate after EVERY batch catches issues early.

### 2.7 Skill Description Improvement (L-006)
Remove "Deterministic" prefix. Lead with third-person action verbs, include domain-specific trigger keywords, explain the "why" behind constraints. Follows official Anthropic skill-creator guidance.

---

## 3. Lessons for AI-Deployment-Lessons Repository

The following lessons from this project should be contributed to the `Impertio-AI-Ecosystem-Deployment` lessons repository (or a dedicated AI-Deployment-Lessons repo if created):

### Lesson: Multi-Agent Output Management
- **Category**: `claude/` or `dev/`
- **Key insight**: Agent output written to `/tmp` is volatile. Always use `--direct` flag or equivalent mechanism to write to persistent project directories.
- **Pattern**: Define explicit OUTPUT paths in every agent prompt. Never rely on default workspace locations.

### Lesson: Agent Delegation Hierarchy
- **Category**: `claude/`
- **Key insight**: Claude Code's built-in Agent tool takes priority over bash-based delegation tools (`oa run`). Nested agent spawning through oa-cli doesn't work reliably.
- **Pattern**: Use flat spawning from top-level session. If you need hierarchy, implement it via task dependencies, not agent nesting.

### Lesson: Structured Task Prompts for Autonomous Agents
- **Category**: `claude/`
- **Key insight**: Free-form prompts produce inconsistent results. 5 required elements: absolute paths, explicit scope, reference files, quality rules, source URLs.
- **Pattern**: Create a prompt template/checklist. The task prompt IS the agent's full context — it must be self-contained.

### Lesson: Multi-Agent Skill Package Pipeline
- **Category**: `claude/`
- **Key insight**: Research → Masterplan → Review → Create → Validate is a proven pipeline for knowledge-intensive work. 15-25 agents total for a full technology skill package.
- **Pattern**: 7-phase methodology with QA gates between phases. 3-5 agents per batch. Phase overlap possible when dependency graph allows.

### Lesson: `fcntl` Module Not Available on Windows
- **Category**: `dev/`
- **Key insight**: Python's `fcntl` module is Unix-only. oa-cli (Open-Agents) fails on Windows with `ModuleNotFoundError`.
- **Pattern**: Use cross-platform file locking (e.g., `portalocker` or `msvcrt` fallback).

### Lesson: IFC Property Extraction Is Universal
- **Category**: `dev/`
- **Key insight**: The same IFC property extraction pattern appears in every IFC-related project: `element.IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties → IfcPropertySingleValue.NominalValue.wrappedValue`. Works for both Python (IfcOpenShell) and TypeScript (web-ifc).

---

## 4. Recommendations for Open-Agents CLAUDE.md Workspace Improvements

### 4.1 Default `--direct` Mode
**Priority: HIGH**
Make `--direct` the default for `oa run`. The `/tmp` workspace should only hold state files (CLAUDE.md, .done marker), not user output. This is the most impactful single change.

### 4.2 CLAUDE.md Injection Improvements
**Priority: HIGH**
When oa-cli generates the agent's CLAUDE.md, include:
```markdown
## CRITICAL RULES
- NEVER use the built-in Agent tool for spawning sub-agents
- ALWAYS use `oa run` via the Bash tool for delegation
- The oa CLI is at: {OA_PATH}
- ALWAYS write output to: {PROJECT_ROOT}/{output_dir}/
```

### 4.3 Structured Task Prompt Template
**Priority: MEDIUM**
Implement `--template` flag or prompt builder per Issue #12. Minimum viable version:
```bash
oa run --template standard \
  --input /path/to/input \
  --output /path/to/output \
  --scope "bullet,points,of,scope" \
  --name agent-name --direct
```

### 4.4 `oa collect --files` Command
**Priority: MEDIUM**
Add ability to rescue output files from `/tmp` workspaces:
```bash
oa collect agent-name --files           # list output files
oa collect agent-name --files-to dir/   # copy to specific directory
```

### 4.5 Post-Completion Output Hook
**Priority: LOW**
After `.done` marker is created, automatically copy `output/` contents to a persistent location. Log the paths in `oa status` output.

### 4.6 Agent PATH Configuration
**Priority: LOW**
Ensure `oa` binary is in PATH inside agent workspaces. Currently agents can't find `oa` because it's at `/home/freek/.local/bin/oa` which may not be in the agent's PATH.

### 4.7 `--disallowedTools` Integration
**Priority: EXPERIMENTAL**
Consider adding `--no-agent-tool` flag to `oa run` that passes `--disallowedTools Agent` to Claude Code, forcing agents to use `oa run` for delegation. Aggressive but effective.

---

## 5. Cross-Reference: Issues ↔ Lessons ↔ Improvements

| Issue | Related Lessons | Recommended Fix |
|-------|----------------|-----------------|
| #9 (Agent tool override) | L-004 | CLAUDE.md injection (4.2), --disallowedTools (4.7) |
| #10 (/tmp output loss) | L-010 | Default --direct (4.1), oa collect --files (4.4) |
| #11 (Nested spawning) | L-004 | Flat spawning pattern, CLAUDE.md injection (4.2) |
| #12 (Structured prompts) | L-010, L-011 | --template flag (4.3) |

---

## 6. Current oa-cli Maturity Assessment

| Capability | Status | Notes |
|-----------|--------|-------|
| Single agent spawning | Working | `oa run` works reliably |
| `--direct` mode | Working | But must be explicit |
| Agent messaging (`oa send/inbox`) | Working | Inter-agent comms functional |
| Agent status (`oa status`) | Working | Shows running agents |
| Nested spawning | Broken | Agents use Claude Code Agent tool instead |
| Output persistence | Broken | Defaults to volatile `/tmp` |
| Orchestration patterns | Partial | Flat spawning works, nested doesn't |
| Structured prompts | Missing | No template system yet |
| Windows support | Broken | `fcntl` module missing |

**Overall**: oa-cli v0.2.0 is usable for flat, single-level agent spawning with explicit `--direct` flags and well-structured prompts. The 4 open issues represent the critical gaps preventing reliable multi-agent orchestration workflows.
