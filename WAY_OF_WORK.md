# WAY OF WORK

## Overview
This project follows the 7-phase research-first methodology proven in the ERPNext Skill Package development. The methodology ensures deterministic, high-quality skills by mandating deep research before any skill creation.

**Core principle**: You cannot create deterministic skills for something you don't deeply understand.

## The 7 Phases

### Phase 1: Raw Masterplan
- Define project scope and technology coverage
- Create preliminary skill inventory (estimate, not final)
- Set up repository structure and core files
- Establish protocols (CLAUDE.md)
- Output: raw-masterplan.md, CLAUDE.md, ROADMAP.md

### Phase 2: Deep Research (Vooronderzoek)
- One comprehensive research document per technology
- Cover: API surface, version differences, architecture, common patterns, anti-patterns
- Minimum 2000 words per technology
- Must include: version matrix, API overview, code examples, error patterns
- Output: vooronderzoek-{technology}.md

### Phase 3: Masterplan Refinement
- Review research against preliminary skill inventory
- Add, merge, or remove skills based on findings
- Define dependencies between skills
- Write ready-to-use prompts for each skill (so agents can execute directly)
- Output: masterplan.md (final version), updated ROADMAP.md

### Phase 4: Topic-Specific Research
- Before each skill: focused research document
- Only the information that specific skill needs
- Verify against official documentation
- Collect and validate code examples
- Identify anti-patterns from real issues
- Output: topic-research/{skill-name}-research.md

### Phase 5: Skill Creation
- Transform research into deterministic skills
- Follow skill structure strictly (see below)
- Execute in batches of 3 agents via oa-cli
- Quality gate after every batch
- Output: skills/source/{category}/{skill-name}/

### Phase 6: Validation
- Structural validation (frontmatter, line count, references)
- Content validation (deterministic language, version-explicit, English-only)
- Cross-reference validation (skills reference each other correctly)
- Functional validation (test with real Claude Code questions)
- Output: validation report

### Phase 7: Publication
- Update INDEX.md with complete skill catalog
- Write README.md with installation instructions
- Final ROADMAP.md update to 100%
- Release tag on GitHub

## Skill Structure

### Directory Layout
```
skill-name/
├── SKILL.md              # Main file, < 500 lines
└── references/
    ├── methods.md        # Complete API signatures
    ├── examples.md       # Working code examples
    └── anti-patterns.md  # What NOT to do
```

### SKILL.md Format
```yaml
---
name: {tech}-{category}-{topic}
description: "Deterministic [description]. Use this skill when Claude needs to [trigger scenario]..."
---
```

Content sections (in order):
1. Quick Reference (critical warnings, decision trees)
2. Essential Patterns (with version annotations)
3. Common Operations (code snippets)
4. Reference Links (to references/ files)

### Naming Convention
- `{tech}-{category}-{topic}`
- Tech prefixes: blender-, bonsai-, ifcos-, sverchok-, aec-
- Categories: syntax, impl, errors, core, agents
- Examples: blender-syntax-operators, ifcos-impl-creation, aec-core-bim-workflows

## Content Standards

### DO:
- Use imperative, deterministic language: "ALWAYS use X when Y", "NEVER do X because Y"
- Verify all code against official documentation
- Include version-specific information (Blender 3.x/4.x, IFC2x3/IFC4/IFC4.3)
- Provide working examples
- Document anti-patterns with explanations
- Use decision trees for common choices
- Keep SKILL.md under 500 lines

### DON'T:
- Use vague language: "you might consider", "it's often good practice"
- Make assumptions about API behavior
- Copy from outdated sources
- Skip version compatibility information
- Include speculative features
- Write skills in any language other than English

## Orchestration Model

### Delegation-First Architecture
- Main session = ORCHESTRATOR (coordinates, validates, never does work)
- Workers = agents spawned via `oa run` or `oa delegate`
- Validators = agents that check worker output before acceptance

### Batch Execution
- 3-5 agents per batch (optimal per Open-Agents lessons)
- Each agent writes to its own unique directory (no file conflicts)
- Quality gate between batches
- Orchestrator does QA after each batch before starting next

### Inter-Agent Communication
- `oa send`: Direct messages between agents (dependency validation)
- `oa broadcast`: Phase transition announcements
- `oa inbox`: Check for messages before starting new work
- Workers notify orchestrator on completion
- Validators send pass/fail feedback to workers

## Version Control Discipline
- Commit after EVERY completed phase
- Push to GitHub immediately (not deferred)
- Commit message format: "Phase X.Y: [action] [subject]"
- ROADMAP.md updated with every commit
- NEVER track status in multiple places (ROADMAP.md is the ONLY source)

## Session Recovery Protocol
When starting a new session or recovering from interruption:
1. Read ROADMAP.md (what's done, what's next)
2. Read LESSONS.md (recent discoveries)
3. Check git log (last commits)
4. Identify where we left off
5. Confirm with user before continuing

## Key Lesson: English-Only Skills
Skills are instructions FOR Claude, not for end users. Claude reads English and responds in ANY language the user speaks. Creating bilingual skills doubles maintenance with zero functional benefit. ALL skills MUST be in English.

## Reference Projects
- ERPNext Skill Package: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
- Open-Agents (orchestration tooling): https://github.com/OpenAEC-Foundation/Open-Agents
- Impertio AI Ecosystem (general lessons): https://github.com/OpenAEC-Foundation/Impertio-AI-Ecosystem-Deployment
