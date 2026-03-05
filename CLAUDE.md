# Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package

## Project Identity
- AEC (Architecture, Engineering, Construction) skill package for Claude
- Technologies: Blender, Bonsai (formerly BlenderBIM), IfcOpenShell, Sverchok
- Methodology: 7-phase research-first development (proven in ERPNext Skill Package)
- Reference project: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package

## Core Files Map
| File | Domain | Role |
|------|--------|------|
| ROADMAP.md | Status | Single source of truth for project status, progress, next steps |
| LESSONS.md | Knowledge | Numbered lessons (L-XXX) discovered during development |
| DECISIONS.md | Architecture | Numbered decisions (D-XXX) with rationale, immutable once recorded |
| REQUIREMENTS.md | Scope | What skills must achieve, quality guarantees per technology |
| SOURCES.md | References | Official documentation URLs, verification rules, last-verified dates |
| WAY_OF_WORK.md | Methodology | 7-phase process, skill structure, content standards |
| CHANGELOG.md | History | Version history in Keep a Changelog format |
| CONTRIBUTING.md | Community | How to contribute, skill development guidelines |
| SECURITY.md | Security | Vulnerability reporting policy |
| docs/masterplan/masterplan.md | Planning | Execution plan with phases, prompts, dependencies |
| INDEX.md | Catalog | Complete skill catalog with trigger scenarios |
| README.md | Public | GitHub landing page - the project's public face |

---

## Session Start Protocol (P-001)
EVERY session begins with this sequence:

1. **Read ROADMAP.md** → Determine current phase, progress percentage, and "Next Steps" section
2. **Read LESSONS.md** → Check recent lessons that may affect your work
3. **Read DECISIONS.md** → Know all architectural decisions (D-001+) and their constraints
4. **Read REQUIREMENTS.md** → Understand quality guarantees and per-technology requirements
5. **Read docs/masterplan/masterplan.md** → Know the execution plan and current phase details
6. **If researching**: Read **SOURCES.md** → Know approved sources per technology, verification rules
7. **If creating skills**: Read **WAY_OF_WORK.md** → Know skill structure, content standards, naming
8. Identify next action from ROADMAP.md "Next Steps"
9. Confirm with user before proceeding

---

## Delegation Protocol (P-002)
CRITICAL: This session is an ORCHESTRATOR, not a worker.

### Rules:
- NEVER do research/writing work directly in this session
- ALWAYS delegate via `oa run` or `oa delegate`
- Validate before accepting (validator-before-apply)

### What to include in EVERY agent prompt:
- Quality criteria from **REQUIREMENTS.md** (relevant to their task)
- Approved source URLs from **SOURCES.md** (what docs to consult)
- Current status from **ROADMAP.md** (what's done, what's needed)
- Relevant constraints from **DECISIONS.md** (D-003: English-only, D-009: 500 line limit, etc.)
- Skill structure from **WAY_OF_WORK.md** (if writing skills)

### Delegation Flow:
1. Define task scope + expected output path
2. Write task prompt with core file references (see above)
3. Spawn worker: `oa run "<task>" --name <name>`
4. Monitor: `oa status`
5. Collect: `oa collect <name>`
6. VALIDATE output against **REQUIREMENTS.md** quality criteria
7. Accept or respawn with corrections

### Batch Strategy:
- 3-5 agents per batch (optimal per Open-Agents lessons)
- Separated file scopes (NEVER two agents on same file)
- Quality gate after every batch
- Orchestrator does QA, NEVER does the work itself

### Open-Agents CLI Reference (oa-cli)

> **IMPORTANT**: oa-cli runs ONLY in WSL Ubuntu, NOT in Windows PowerShell/CMD/Git Bash.
> The tool depends on `fcntl` (Unix file locking) and `tmux` which are Linux-only.
> All `oa` commands below must be executed in a WSL Ubuntu terminal.

**Spawning agents:**
```bash
oa run "<task description>" --name <agent-name>        # Spawn single worker
oa run "<task>" --name <name> --model claude/opus      # Specify model
oa run "<task>" --name <name> --parent <orchestrator>  # Set parent (hierarchy)
oa delegate "<task>"                                    # Spawn orchestrator + auto-manage workers
oa pipeline "<task>"                                    # Planner -> parallel subtasks -> combiner
```

**Monitoring & collecting:**
```bash
oa status                    # Show all agents with hierarchy and status
oa watch <name>              # Stream agent output in real-time
oa attach <name>             # Switch to agent's tmux window (interactive)
oa collect <name>            # Display completed agent's output
oa dashboard                 # Interactive TUI with rich tables
```

**Communication between agents:**
```bash
oa send <recipient> "<message>"              # Direct message to agent
oa send <recipient> "<msg>" --from <sender>  # Message with custom sender
oa inbox <name>                              # Read agent's messages
oa inbox <name> --unread                     # Only unread messages
oa broadcast "<message>"                     # Send to ALL running agents
```

**Lifecycle:**
```bash
oa start                     # Initialize tmux session + dashboard
oa kill <name>               # Stop a running agent
oa clean                     # Clean up finished agent workspaces
```

**Key rules (from Open-Agents LESSONS.md):**
- L-001: Orchestrator delegates, never does work itself
- L-003: Two agents NEVER write the same file
- L-004: QA after each batch before starting next
- L-010: Claude Code = relay, delegate via `oa run`
- L-025: Optimal 3-5 agents per batch, 5-6 tasks per agent max

**Source**: https://github.com/OpenAEC-Foundation/Open-Agents

---

## Quality Control Protocol (P-003)
### Validation criteria sourced from core files:

**From REQUIREMENTS.md:**
- Skill format requirements (YAML frontmatter, structure)
- Per-technology version coverage
- Package independence (each tech standalone)

**From DECISIONS.md:**
- D-003: English-only content
- D-005: MIT License
- D-009: SKILL.md < 500 lines

**From SOURCES.md:**
- All code verified against listed official sources only
- No unverified blog posts or outdated content

### Validator-Before-Apply checklist:
1. File exists and is complete
2. YAML frontmatter valid (name, description with trigger words)
3. Line count < 500 (SKILL.md)
4. English-only (no Dutch or other languages)
5. Deterministic language (ALWAYS/NEVER, not "you might consider")
6. Version-explicit (Blender 3.x/4.x, IFC2x3/IFC4/IFC4.3)
7. All references/ files exist and are linked from SKILL.md
8. Sources traceable to **SOURCES.md** approved URLs

### Correction Flow:
If validation fails:
1. Document what failed in agent feedback
2. Spawn fix-agent with specific correction instructions
3. Re-validate after fix
4. NEVER accept below quality bar defined in **REQUIREMENTS.md**

---

## Research Protocol (P-004)
### Before ANY research:
1. Read **SOURCES.md** → Know approved sources for the technology
2. Read **REQUIREMENTS.md** → Know what the research must cover
3. Read **DECISIONS.md** → Know constraints (D-003 English-only, etc.)

### During research:
- Use ONLY sources listed in **SOURCES.md** (or add new ones there)
- Verify code examples against official documentation
- Identify anti-patterns from real GitHub issues

### After research:
1. Update **SOURCES.md** "Last Verified" table with verification date
2. Log new discoveries in **LESSONS.md** (numbered L-XXX)
3. If new architectural decisions emerge, record in **DECISIONS.md** (numbered D-XXX)

### Research output location:
- Vooronderzoek: `docs/research/vooronderzoek-{technology}.md`
- Topic research: `docs/research/topic-research/{skill-name}-research.md`
- Research fragments: `docs/research/fragments/`

---

## Skill Standards (P-005)
Defined in detail in **WAY_OF_WORK.md** and **REQUIREMENTS.md**. Quick reference:

- English-only (per **DECISIONS.md** D-003)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- Version-explicit: mark all code with supported versions (per **REQUIREMENTS.md**)
- SKILL.md < 500 lines (per **DECISIONS.md** D-009), heavy content in references/
- YAML frontmatter: name + description with trigger words
- Structure: Quick Reference > Decision Trees > Patterns > Reference Links
- Verify against **SOURCES.md** approved URLs only

---

## Document Sync Protocol (P-006)
After EVERY completed phase/batch, update these files:

1. **ROADMAP.md** → Status, percentage, changelog entry, next steps (MANDATORY)
2. **LESSONS.md** → New patterns or discoveries (if any)
3. **DECISIONS.md** → New architectural decisions (if any)
4. **SOURCES.md** → New sources verified or dates updated (if researching)
5. **CHANGELOG.md** → Milestone entries (for significant completions)
6. Commit with message: `Phase X.Y: [action] [subject]`
7. Push to GitHub
8. **README.md** → Check if landing page needs updating:
   - Skill count changed? Update package table
   - Phase milestone reached? Update "Current Progress" section
   - New documentation added? Update docs table
   - If significant: update repo description/topics via GitHub API

Timing: IMMEDIATE after completion, not deferred.

---

## Session End Protocol (P-007)
Before ending ANY session:

1. **ROADMAP.md** → Update current phase status + "Next Steps" section (CRITICAL - this is how the next session knows where to continue)
2. **LESSONS.md** → Log anything learned during this session
3. **DECISIONS.md** → Record any decisions made
4. **CHANGELOG.md** → Add entry if milestone reached
5. Commit all changes with descriptive message
6. Push to GitHub
7. Verify **README.md** reflects current project state

---

## Inter-Agent Communication Protocol (P-008)
Agents communicate via `oa send`, `oa inbox`, and `oa broadcast`.

### Pattern 1: Dependency Validation
When Agent B depends on Agent A's output:
1. Agent A completes, sends: `oa send validator-b "Done. Output at [path]."`
2. Agent B reads output, validates against **REQUIREMENTS.md** criteria
3. Agent B sends: `oa send agent-a "QA passed"` or `"QA failed: [issues]"`

### Pattern 2: Broadcast for Phase Transitions
1. Orchestrator: `oa broadcast "Batch 3A complete. Starting 3B."`
2. All agents receive; next batch can reference previous output

### Pattern 3: Peer Review Between Writers
1. Writer A (syntax/) sends to Writer B (impl/)
2. Writer B checks cross-references consistency
3. Both correct before batch closes

### Pattern 4: Research Sharing
1. Agent discovers finding relevant to another technology
2. Sends finding via `oa send` to relevant peer
3. Peer incorporates into their research

### Rules:
- Orchestrator coordinates only, NEVER does work
- Workers notify orchestrator on completion
- `oa inbox --unread` before starting new work

---

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
| Blender | blender- | 3.x, 4.x, 5.x |
| Bonsai | bonsai- | Current (ex-BlenderBIM) |
| IfcOpenShell | ifcos- | Latest + IFC2x3/IFC4/IFC4.3 |
| Sverchok | sverchok- | Current (later phase) |
| Cross-tech | aec- | N/A |

## Repository Structure
```
project-root/
├── CLAUDE.md                    # THIS FILE - protocols and instructions
├── ROADMAP.md                   # Status (single source of truth)
├── REQUIREMENTS.md              # Quality guarantees
├── DECISIONS.md                 # Architectural decisions
├── SOURCES.md                   # Official reference URLs
├── WAY_OF_WORK.md               # 7-phase methodology
├── LESSONS.md                   # Lessons learned
├── CHANGELOG.md                 # Version history
├── CONTRIBUTING.md              # Contribution guidelines
├── SECURITY.md                  # Security policy
├── README.md                    # GitHub landing page
├── INDEX.md                     # Skill catalog (after Phase 5)
├── docs/
│   ├── masterplan/              # raw-masterplan.md, masterplan.md
│   └── research/                # vooronderzoek-*.md, topic-research/, fragments/
└── skills/                      # SEPARATE PACKAGES per technology
    ├── blender/{syntax,impl,errors,core,agents}/
    ├── bonsai/{syntax,impl,errors,core,agents}/
    ├── ifcopenshell/{syntax,impl,errors,core,agents}/
    ├── sverchok/{syntax,impl,errors,core,agents}/
    └── aec-cross-tech/{core,agents}/
```
Each technology is a SEPARATE package that can be installed independently.
