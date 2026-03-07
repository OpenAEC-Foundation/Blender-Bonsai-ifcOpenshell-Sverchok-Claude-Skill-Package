# Skill Package Development Workflow — 0 to 100

**Purpose**: This document captures the complete, proven methodology for building a Claude Skill Package for any technology domain. Give this to an orchestrator agent with a technology name, and it executes the entire pipeline autonomously.

**Proven on**: Blender (26 skills), IfcOpenShell (19 skills), Bonsai (14 skills), Sverchok (in progress), Cross-Tech (2 skills) — total 61+ skills.

**Tooling**: `oa` CLI (Open-Agents) for multi-agent delegation in WSL/Linux.

---

## Prerequisites

Before starting, ensure:
1. Repository structure exists with core files (CLAUDE.md, ROADMAP.md, REQUIREMENTS.md, DECISIONS.md, SOURCES.md, WAY_OF_WORK.md, LESSONS.md, CHANGELOG.md)
2. `skills/{technology}/` directory with subdirectories: `syntax/`, `impl/`, `errors/`, `core/`, `agents/`
3. `docs/research/` and `docs/research/fragments/` and `docs/research/topic-research/` directories
4. `docs/masterplan/` directory
5. `oa` CLI installed and `oa start` executed
6. SOURCES.md populated with official documentation URLs for the target technology

---

## Phase 1: Deep Research (Vooronderzoek)

**Input**: Technology name, official documentation URLs from SOURCES.md
**Output**: `docs/research/vooronderzoek-{technology}.md` (2000+ lines)

### Step 1.1: Spawn Research Agents

Spawn 3 parallel agents, each covering a distinct research domain:

```bash
# Agent 1: Core Architecture & API Surface
oa run "TASK: Deep research on {TECHNOLOGY} CORE ARCHITECTURE.
OUTPUT FILE: Write to {PROJECT_ROOT}/docs/research/fragments/{tech}-core-architecture.md

RESEARCH SCOPE:
- Complete API surface mapping (all modules, classes, methods)
- Architecture overview (how the system is structured internally)
- Data model and object hierarchy
- Version matrix (all supported versions, breaking changes between them)
- Key configuration and initialization patterns

SOURCES: {list URLs from SOURCES.md}
FORMAT: English only. Markdown. Minimum 800 lines. Deterministic language (ALWAYS/NEVER).
Verify ALL API references against source code or official docs." \
--name {tech}-research-core --direct

# Agent 2: Python Integration & Scripting
oa run "TASK: Deep research on {TECHNOLOGY} PYTHON SCRIPTING & INTEGRATION.
OUTPUT FILE: Write to {PROJECT_ROOT}/docs/research/fragments/{tech}-python-integration.md

RESEARCH SCOPE:
- All Python APIs and entry points
- Scripting patterns and code templates
- Common operations with working code examples
- Performance optimization patterns
- Integration with other tools/libraries

SOURCES: {list URLs from SOURCES.md}
FORMAT: English only. Markdown. Minimum 600 lines. Working code examples for each pattern." \
--name {tech}-research-python --direct

# Agent 3: Ecosystem, Extensions & Real-World Usage
oa run "TASK: Deep research on {TECHNOLOGY} ECOSYSTEM & EXTENSIONS.
OUTPUT FILE: Write to {PROJECT_ROOT}/docs/research/fragments/{tech}-ecosystem-extensions.md

RESEARCH SCOPE:
- Extension/plugin ecosystem
- Integration with other technologies
- Real-world usage patterns from open-source projects
- Common anti-patterns found in GitHub issues
- Error patterns and their solutions

SOURCES: {list URLs from SOURCES.md}
FORMAT: English only. Markdown. Minimum 600 lines. Include real error messages and solutions." \
--name {tech}-research-ecosystem --direct
```

### Step 1.2: Monitor & Collect

```bash
oa status                          # Check progress
oa watch {tech}-research-core      # Stream live output
oa collect {tech}-research-core    # Read completed output
```

### Step 1.3: Merge Fragments

After all 3 agents complete, merge fragments into a single vooronderzoek:

```bash
oa run "TASK: Merge 3 research fragments into a unified vooronderzoek document.

INPUT FILES (read these):
- {PROJECT_ROOT}/docs/research/fragments/{tech}-core-architecture.md
- {PROJECT_ROOT}/docs/research/fragments/{tech}-python-integration.md
- {PROJECT_ROOT}/docs/research/fragments/{tech}-ecosystem-extensions.md

OUTPUT FILE: Write to {PROJECT_ROOT}/docs/research/vooronderzoek-{technology}.md

INSTRUCTIONS:
1. Read all 3 fragments
2. Create a unified document with these sections:
   - Header (Date, Status, Author, Scope, Versions Covered)
   - Table of Contents
   - API Overview (from core fragment)
   - Version Matrix (from core fragment)
   - Architecture (from core fragment)
   - Python Scripting (from python fragment)
   - Ecosystem & Extensions (from ecosystem fragment)
   - Common Error Patterns (from ecosystem fragment)
   - AI Common Mistakes (synthesized from all)
   - Real-World Usage (from ecosystem fragment)
3. Remove duplication between fragments
4. Ensure consistent terminology
5. Add cross-references between sections
6. Target: 2000+ lines total

FORMAT: English only. Follow the exact structure of vooronderzoek-blender.md." \
--name {tech}-merge-vooronderzoek --direct
```

### Step 1.4: QA Gate

Verify the vooronderzoek:
- [ ] All major API surfaces covered
- [ ] Version information is explicit and accurate
- [ ] Code examples are working and version-tagged
- [ ] Anti-patterns documented with real error messages
- [ ] Sources traceable to SOURCES.md approved URLs
- [ ] English-only content
- [ ] Minimum 2000 lines

**Update ROADMAP.md**: Mark Phase 1 as COMPLETE.

---

## Phase 2: Raw Masterplan

**Input**: `docs/research/vooronderzoek-{technology}.md`
**Output**: `docs/masterplan/{technology}-masterplan-raw.md`

### Step 2.1: Spawn Masterplan Agent

```bash
oa run "TASK: Create a RAW MASTERPLAN for the {TECHNOLOGY} skill package.

INPUT: Read {PROJECT_ROOT}/docs/research/vooronderzoek-{technology}.md

OUTPUT FILE: Write to {PROJECT_ROOT}/docs/masterplan/{technology}-masterplan-raw.md

INSTRUCTIONS:
1. Analyze the vooronderzoek and identify all distinct skill topics
2. For each skill, define:
   - Skill name: {tech}-{category}-{topic}
   - Category: syntax | impl | errors | core | agents
   - Scope: what API surface it covers
   - Dependencies: which other skills must exist first
   - Key SOURCES.md URLs for this skill
   - Complexity: S (small) | M (medium) | L (large)

3. Organize skills into categories:
   - syntax/: API syntax, code patterns, method signatures
   - impl/: Development workflows, step-by-step guides
   - errors/: Error handling, diagnostics, common mistakes
   - core/: Cross-cutting concepts, architecture overview, version matrix
   - agents/: Validation checklists, code validators

4. Define batch execution order respecting dependencies:
   - Batch 1: Foundation (core + basic syntax, no dependencies)
   - Batch 2: Remaining syntax (depends on core)
   - Batch 3+: Implementation (depends on syntax)
   - Second-to-last: Errors (depends on impl)
   - Last: Agents (depends on everything)

5. Create skill count table:
   | Category | Count | Skills |
   |----------|-------|--------|

RULES:
- Each skill < 500 lines SKILL.md (heavy content in references/)
- No redundancy between skills
- Each skill must be independently useful
- Target: 8-20 skills depending on API surface size

FORMAT: Follow the structure of the existing masterplan.md" \
--name {tech}-masterplan-raw --direct
```

**Update ROADMAP.md**: Mark Phase 2 as COMPLETE.

---

## Phase 3: Masterplan Refinement

**Input**: Raw masterplan + vooronderzoek
**Output**: `docs/masterplan/{technology}-masterplan.md` (DEFINITIVE)

### Step 3.1: Spawn Review Agents (2 parallel)

```bash
# Technical review
oa run "TASK: TECHNICAL REVIEW of {TECHNOLOGY} skill masterplan.

INPUT: Read {PROJECT_ROOT}/docs/masterplan/{technology}-masterplan-raw.md
       AND {PROJECT_ROOT}/docs/research/vooronderzoek-{technology}.md

OUTPUT FILE: Write review to {PROJECT_ROOT}/docs/masterplan/{technology}-review-technical.md

CHECK:
1. Are ALL API surfaces from the vooronderzoek covered by at least one skill?
2. Any skill trying to cover too much? (should be split)
3. Any two skills with overlapping scope? (should be merged)
4. Are dependencies correct? (no circular deps, correct order)
5. Any critical anti-patterns not covered by error skills?
6. Is the batch order optimal for parallel execution?" \
--name {tech}-review-technical --direct

# Practical review
oa run "TASK: PRACTICAL REVIEW of {TECHNOLOGY} skill masterplan.

INPUT: Read {PROJECT_ROOT}/docs/masterplan/{technology}-masterplan-raw.md
       AND {PROJECT_ROOT}/docs/research/vooronderzoek-{technology}.md

OUTPUT FILE: Write review to {PROJECT_ROOT}/docs/masterplan/{technology}-review-practical.md

CHECK:
1. Would each skill actually help Claude write better code?
2. Are skills organized from a USER's perspective (not just API structure)?
3. Do the skill descriptions contain clear trigger words?
4. Are there common workflows missing (what would a developer ask Claude to do)?
5. Is the agent skill (code-validator) comprehensive enough?
6. Can each skill work independently without other skills?" \
--name {tech}-review-practical --direct
```

### Step 3.2: Consolidate Reviews

```bash
oa run "TASK: Consolidate reviews into DEFINITIVE masterplan.

INPUT:
- {PROJECT_ROOT}/docs/masterplan/{technology}-masterplan-raw.md
- {PROJECT_ROOT}/docs/masterplan/{technology}-review-technical.md
- {PROJECT_ROOT}/docs/masterplan/{technology}-review-practical.md

OUTPUT: Write to {PROJECT_ROOT}/docs/masterplan/{technology}-masterplan.md

INSTRUCTIONS:
1. Read all 3 documents
2. Apply all valid feedback from both reviews
3. Merge/split/add/remove skills as recommended
4. Write agent prompts for EACH skill (copy-paste ready for oa run)
5. Finalize batch execution order
6. Mark this as DEFINITIVE

FORMAT: Follow the structure of the existing masterplan.md §2 (Skill Inventory)" \
--name {tech}-masterplan-definitive --direct
```

**Update ROADMAP.md**: Mark Phase 3 as COMPLETE. Record new decisions in DECISIONS.md.

---

## Phase 4: Topic-Specific Research

**Input**: Definitive masterplan
**Output**: `docs/research/topic-research/{tech}-{topic}-research.md` per topic

### Step 4.1: Identify Research Gaps

Read the definitive masterplan. For each skill, check if the vooronderzoek has sufficient detail. Spawn research agents only for skills where:
- API surface is complex and needs deeper investigation
- The vooronderzoek mentions the topic but lacks code examples
- Anti-patterns need real GitHub issue research

### Step 4.2: Spawn Research Agents (3-5 per batch)

```bash
oa run "TASK: Topic-specific research for skill {SKILL_NAME}.
OUTPUT FILE: Write to {PROJECT_ROOT}/docs/research/topic-research/{tech}-{topic}-research.md

FOCUS: {specific research scope from masterplan}
SOURCES: {specific URLs for this topic}
FORMAT: English only. Working code examples. Anti-patterns from real issues." \
--name {tech}-research-{topic} --direct
```

**Only spawn what's needed** — skip topics where the vooronderzoek is already sufficient.

**Update ROADMAP.md**: Mark Phase 4 as COMPLETE.

---

## Phase 5: Skill Creation

**Input**: Definitive masterplan + vooronderzoek + topic research
**Output**: `skills/{technology}/{category}/{skill-name}/SKILL.md` + `references/`

### Step 5.1: Skill Writer Agent Prompt Template

Every skill-writer agent receives this prompt:

```bash
oa run "PROJECT: {PROJECT_NAME}
WORKSPACE: {PROJECT_ROOT}

TASK: Create skill: {SKILL_NAME}
CATEGORY: {CATEGORY}
OUTPUT DIRECTORY: {PROJECT_ROOT}/skills/{technology}/{category}/{skill-name}/

SKILL SCOPE:
{scope from masterplan}

RESEARCH INPUT:
- Main: {PROJECT_ROOT}/docs/research/vooronderzoek-{technology}.md
- Topic: {PROJECT_ROOT}/docs/research/topic-research/{tech}-{topic}-research.md (if exists)

DELIVERABLES:
1. SKILL.md (< 500 lines) — main skill file
2. references/methods.md — complete API signatures
3. references/examples.md — working code examples
4. references/anti-patterns.md — what NOT to do

SKILL.MD STRUCTURE:
\`\`\`yaml
---
name: {skill-name}
description: '{trigger-word-rich description}'
license: MIT
compatibility: 'Designed for Claude Code. Requires {technology} {versions}.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---
\`\`\`

Content sections (in order):
1. # {Skill Title}
2. ## Quick Reference
   - ### Critical Warnings (ALWAYS/NEVER statements)
   - ### Decision Tree (when to use what)
   - ### Version Matrix (if applicable)
3. ## Essential Patterns (5-8 code examples with version tags)
4. ## Common Operations (quick reference tables + snippets)
5. ## Reference Links (to references/ files)

QUALITY RULES:
- English-only (D-003)
- SKILL.md < 500 lines (D-009)
- Deterministic language: ALWAYS/NEVER, not 'you might consider'
- Version-explicit: tag every code example with supported versions
- All API references verified against official docs
- No hallucinated method signatures" \
--name {tech}-skill-{skill-name} --direct
```

### Step 5.2: Execute in Batches

```
Batch 1: Foundation skills (core + basic syntax) — 2-3 agents
  [QA GATE]
Batch 2: Remaining syntax skills — 3-5 agents
  [QA GATE]
Batch 3: Implementation skills — 3-5 agents
  [QA GATE]
Batch 4: Error skills — 1-3 agents
  [QA GATE]
Batch 5: Agent skills — 1 agent
  [QA GATE]
```

### Step 5.3: QA Gate After Each Batch

For each completed skill, verify:
- [ ] SKILL.md exists and is < 500 lines
- [ ] YAML frontmatter is valid (name, description, license, compatibility, metadata)
- [ ] references/methods.md exists
- [ ] references/examples.md exists
- [ ] references/anti-patterns.md exists
- [ ] English-only content
- [ ] Deterministic language (no vague hedging)
- [ ] Version-explicit code examples
- [ ] No duplicate content with other skills

**Update ROADMAP.md** after each batch.

---

## Phase 6: Validation

**Input**: All created skills
**Output**: Validation report + fixes

### Step 6.1: Spawn Validators (2 parallel)

```bash
# Structural validator
oa run "TASK: Structural validation of ALL {TECHNOLOGY} skills.

SKILLS DIRECTORY: {PROJECT_ROOT}/skills/{technology}/

CHECK EVERY SKILL FOR:
1. YAML frontmatter valid (name, description, license, compatibility, metadata)
2. SKILL.md line count < 500
3. references/ directory exists with methods.md, examples.md, anti-patterns.md
4. English-only (no Dutch, German, or other languages)
5. Deterministic language (grep for 'might consider', 'you could', 'perhaps')
6. File naming follows convention: {tech}-{category}-{topic}
7. No empty sections or placeholder text

OUTPUT: Write report to {PROJECT_ROOT}/docs/research/validation-{technology}-structural.md
List EVERY issue found with file path and line number." \
--name {tech}-validator-structural --direct

# Content validator
oa run "TASK: Content validation of ALL {TECHNOLOGY} skills.

SKILLS DIRECTORY: {PROJECT_ROOT}/skills/{technology}/
VOORONDERZOEK: {PROJECT_ROOT}/docs/research/vooronderzoek-{technology}.md

CHECK EVERY SKILL FOR:
1. Version information is explicit and correct
2. API signatures match official documentation
3. Code examples are complete and would actually run
4. Anti-patterns include real error messages
5. Cross-references between skills are correct
6. No contradictions between skills
7. Agent skill has comprehensive validation checklist

OUTPUT: Write report to {PROJECT_ROOT}/docs/research/validation-{technology}-content.md
List EVERY issue found with file path and specific concern." \
--name {tech}-validator-content --direct
```

### Step 6.2: Fix Blockers

For each issue found, spawn a fix agent:

```bash
oa run "TASK: Fix validation issue in {SKILL_NAME}.
ISSUE: {description of the problem}
FILE: {file path}
FIX: {specific fix required}" \
--name {tech}-fix-{issue-id} --direct
```

### Step 6.3: Re-validate

Re-run validators after fixes. Repeat until zero blockers.

**Update ROADMAP.md**: Mark Phase 6 as COMPLETE.

---

## Phase 7: Publication

**Input**: All validated skills
**Output**: Updated project documentation

### Step 7.1: Update Project Files

Update these files to reflect the new skills:

1. **INDEX.md** — Add all new skills to the catalog table
2. **ROADMAP.md** — Update technology from "DEFERRED" to "COMPLETE", update total skill count, add changelog entry
3. **README.md** — Update package table, total skill count, progress section
4. **CHANGELOG.md** — Add milestone entry with date
5. **REQUIREMENTS.md** — Update technology requirements section
6. **DECISIONS.md** — Record any new decisions made during development
7. **LESSONS.md** — Log discoveries and patterns learned

### Step 7.2: Commit & Push

```bash
git add skills/{technology}/ docs/ INDEX.md ROADMAP.md README.md CHANGELOG.md
git commit -m "Phase {N}: Complete {TECHNOLOGY} skill package (N skills)"
git push
```

### Step 7.3: Deploy (Optional)

If deploying to a workspace:
```bash
cp -r skills/{technology}/ {WORKSPACE}/.claude/skills/{technology}/
```

---

## Quick Reference: Full Pipeline Commands

For a new technology `{TECH}`, the complete pipeline is:

```bash
# Phase 1: Research (3 agents)
oa run "{core research prompt}" --name {tech}-research-core --direct
oa run "{python research prompt}" --name {tech}-research-python --direct
oa run "{ecosystem research prompt}" --name {tech}-research-ecosystem --direct
# Wait for completion, then merge:
oa run "{merge prompt}" --name {tech}-merge-vooronderzoek --direct

# Phase 2: Raw Masterplan (1 agent)
oa run "{masterplan prompt}" --name {tech}-masterplan-raw --direct

# Phase 3: Refinement (2 review + 1 consolidate)
oa run "{technical review}" --name {tech}-review-technical --direct
oa run "{practical review}" --name {tech}-review-practical --direct
oa run "{consolidate}" --name {tech}-masterplan-definitive --direct

# Phase 4: Topic Research (as needed)
oa run "{topic research}" --name {tech}-research-{topic} --direct

# Phase 5: Skill Creation (batched)
# For each batch of 3-5 skills:
oa run "{skill prompt}" --name {tech}-skill-{name} --direct

# Phase 6: Validation (2 agents)
oa run "{structural validation}" --name {tech}-validator-structural --direct
oa run "{content validation}" --name {tech}-validator-content --direct

# Phase 7: Publication (orchestrator does this directly)
# Update INDEX.md, ROADMAP.md, README.md, CHANGELOG.md, etc.
```

**Total agents**: ~15-25 depending on skill count and research needs.
**Total duration**: Depends on API surface complexity and agent throughput.

---

## Lessons Learned (from 61+ skills)

1. **Research before action** — never create skills based on assumptions
2. **Flat spawning** — don't nest oa agents inside oa agents (Issue #9/#11)
3. **`--direct` flag always** — prevents output getting lost in /tmp (Issue #10)
4. **3-5 agents per batch** — optimal parallelism without overwhelming QA
5. **Quality gate after EVERY batch** — catch issues early, not at the end
6. **English-only** — skills are for Claude, not for humans (D-003)
7. **< 500 lines** — keep SKILL.md focused, heavy content in references/ (D-009)
8. **No file conflicts** — each agent writes to its own unique directory (L-003)
9. **ROADMAP.md is the single source of truth** — update after every phase (D-006)
10. **Deterministic language** — ALWAYS/NEVER, not "you might consider"
