# Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package

## Project Identity
- AEC (Architecture, Engineering, Construction) skill package for Claude
- Technologies: Blender, Bonsai (formerly BlenderBIM), IfcOpenShell, Sverchok
- Methodology: 7-phase research-first development (proven in ERPNext Skill Package)
- Reference project: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package

## Session Start Protocol (P-001)
1. Read ROADMAP.md (single source of truth for project status)
2. Read LESSONS.md (check recent lessons)
3. Read docs/masterplan/masterplan.md (current execution plan)
4. Identify current phase and next action
5. Confirm with user before proceeding

## Delegation Protocol (P-002)
CRITICAL: This session is an ORCHESTRATOR, not a worker.

### Rules:
- NEVER do research/writing work directly in this session
- ALWAYS delegate via `oa run` or `oa delegate`
- Each delegated task gets a clear scope and output file
- Monitor workers via `oa status`
- Collect results via `oa collect`
- Validate before accepting (validator-before-apply)

### Delegation Flow:
1. Define task scope + expected output
2. Write task prompt (specific, scoped, with output path)
3. Spawn worker: `oa run "<task>" --name <name>`
4. Monitor: `oa status`
5. Collect: `oa collect <name>`
6. VALIDATE output against quality criteria
7. Accept or respawn with corrections

### Batch Strategy:
- 3-5 agents per batch (optimaal)
- Gescheiden file scopes (NOOIT twee agents op zelfde file)
- Quality gate na elke batch
- Orchestrator doet QA, NIET zelf werk

## Inter-Agent Communication Protocol (P-008)
Agents communicate via `oa send`, `oa inbox`, and `oa broadcast`.

### Communication Patterns:

#### Pattern 1: Dependency Validation
When Agent B depends on Agent A's output:
1. Agent A completes work, sends: `oa send validator-b "Work complete, output at [path]. Ready for review."`
2. Agent B reads Agent A's output directory
3. Agent B validates against quality criteria
4. Agent B sends feedback: `oa send agent-a "QA passed"` or `"QA failed: [issues]"`
5. If failed: Agent A fixes and re-notifies

#### Pattern 2: Broadcast for Phase Transitions
When a batch completes:
1. Orchestrator broadcasts: `oa broadcast "Batch 3A complete. Quality gate passed. Starting 3B."`
2. All agents receive notification
3. Next batch agents know they can reference previous batch output

#### Pattern 3: Peer Review Between Writers
Within a batch of skill writers:
1. Writer A finishes syntax/ skill, sends to Writer B
2. Writer B checks cross-references are consistent with their impl/ skill
3. Writer B confirms or flags inconsistencies
4. Both correct before batch closes

#### Pattern 4: Research Sharing
Research agents share findings that benefit others:
1. Blender researcher discovers pattern relevant to Bonsai
2. Sends: `oa send researcher-bonsai "Found that Blender 4.x changed [X], relevant for Bonsai integration"`
3. Bonsai researcher incorporates finding

### Message Flow Rules:
- Orchestrator NEVER does work, only coordinates and validates
- Workers send completion notifications to orchestrator
- Workers send dependency-relevant findings to peer workers
- Broadcasts for status updates only (not for task content)
- `oa inbox --unread` to check for pending messages before starting new work

## Quality Control Protocol (P-003)
### Validator-Before-Apply:
Before accepting ANY agent output:
1. Check file exists and is complete
2. Check YAML frontmatter (name, description)
3. Check line count (SKILL.md < 500 lines)
4. Check language (English-only)
5. Check style (deterministic: ALWAYS/NEVER, not "consider")
6. Check version-explicit (Blender 3.x/4.x, IFC2x3/IFC4/IFC4.3)
7. Check references/ files exist and are referenced

### Correction Flow:
If validation fails:
1. Document what failed
2. Spawn fix-agent with specific correction instructions
3. Re-validate after fix
4. NEVER accept below quality bar

## Research Protocol (P-004)
### Before creating ANY skill:
1. Topic research document must exist
2. Research must be verified against official docs
3. Code examples must be tested/validated
4. Anti-patterns must be identified from real issues

### Research Sources (priority):
1. Official documentation (docs.blender.org, ifcopenshell.org, blenderbim.org)
2. Source code (GitHub repos)
3. Official examples/tutorials
4. NEVER: random blog posts, outdated StackOverflow

## Skill Standards (P-005)
- English-only (Claude reads English, responds in any language)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- Version-explicit: mark all code with supported versions
- SKILL.md < 500 lines, heavy content in references/
- YAML frontmatter: name + description with trigger words
- Structure: Quick Reference > Decision Trees > Patterns > Reference Links

## Document Sync Protocol (P-006)
After EVERY completed phase/batch:
1. Update ROADMAP.md (status, percentage, next steps)
2. Update LESSONS.md (new discoveries)
3. Commit with descriptive message: "Phase X.Y: [action] [subject]"
4. Push to GitHub
- Timing: IMMEDIATE, not deferred

## Session End Protocol (P-007)
Before ending any session:
1. Update ROADMAP.md with current status
2. Log any new lessons in LESSONS.md
3. Commit all changes
4. Push to GitHub
5. Note next action for next session

## Core Files
| File | Purpose | When to Update |
|------|---------|----------------|
| ROADMAP.md | Single source of truth for status | After every phase/batch |
| LESSONS.md | What we learned | When discovering patterns |
| WAY_OF_WORK.md | Methodology reference | Rarely (methodology is stable) |
| docs/masterplan/masterplan.md | Execution plan | After research phases |
| INDEX.md | Skill catalog | After skill creation |

## Skill Categories
| Category | Purpose | Naming |
|----------|---------|--------|
| syntax/ | API syntax, code patterns | {tech}-syntax-{topic} |
| impl/ | Development workflows | {tech}-impl-{topic} |
| errors/ | Error handling patterns | {tech}-errors-{topic} |
| core/ | Cross-cutting concerns | {tech}-core-{topic} |
| agents/ | Intelligent orchestration | {tech}-{agent-name} |

## Technology Scope
| Tech | Prefix | Versions |
|------|--------|----------|
| Blender | blender- | 3.x, 4.x |
| Bonsai | bonsai- | Current (ex-BlenderBIM) |
| IfcOpenShell | ifcos- | Latest + IFC2x3/IFC4/IFC4.3 |
| Sverchok | sverchok- | Current (later phase) |
| Cross-tech | aec- | N/A |

## Repository Structure
```
project-root/
├── CLAUDE.md, ROADMAP.md, WAY_OF_WORK.md, LESSONS.md
├── docs/masterplan/          # Masterplan versions
├── docs/research/            # Vooronderzoek + topic research
└── skills/                   # SEPARATE PACKAGES per technology
    ├── blender/              # Blender skill package (standalone)
    │   ├── syntax/
    │   ├── impl/
    │   ├── errors/
    │   ├── core/
    │   └── agents/
    ├── bonsai/               # Bonsai skill package (standalone)
    │   └── [same structure]
    ├── ifcopenshell/          # IfcOpenShell skill package (standalone)
    │   └── [same structure]
    ├── sverchok/              # Sverchok skill package (standalone, later)
    │   └── [same structure]
    └── aec-cross-tech/        # Cross-technology skills
        ├── core/
        └── agents/
```
Each technology is a SEPARATE package that can be installed independently.
