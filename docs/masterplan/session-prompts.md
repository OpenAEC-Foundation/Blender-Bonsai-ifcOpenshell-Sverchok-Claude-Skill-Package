# Session Prompts - Ready to Use

Copy-paste these prompts to start new Claude sessions or oa agents.
All prompts reference SOURCES.md for approved documentation URLs.

---

## PROMPT A: Bonsai Vooronderzoek (URGENT - blocks Phase 2 completion)

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
REPO: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Write a comprehensive vooronderzoek (preliminary research) for Bonsai BIM addon.

OUTPUT FILE: docs/research/vooronderzoek-bonsai.md

CONTEXT:
- Bonsai is the BIM addon for Blender (formerly called BlenderBIM)
- It lives INSIDE the IfcOpenShell mono-repo
- We already have vooronderzoek-blender.md (54KB) and vooronderzoek-ifcopenshell.md (35KB)
- This is the LAST missing research before we can refine our masterplan
- Read the existing vooronderzoeken first to understand depth/format expected

APPROVED SOURCES (from SOURCES.md - USE THESE, not random blogs):
Official docs:
- Bonsai Documentation: https://docs.bonsaibim.org/
- BlenderBIM (legacy, historical only): https://blenderbim.org/

Source code (PRIMARY for Bonsai - docs are sparse):
- Bonsai source: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
- Bonsai Core: (same repo)/src/bonsai/bonsai/core/
- Bonsai Tool: (same repo)/src/bonsai/bonsai/tool/
- Bonsai UI:   (same repo)/src/bonsai/bonsai/bim/module/

IFC Standard (for context):
- buildingSMART IFC specs: https://standards.buildingsmart.org/IFC/
- IFC4.3 docs: https://ifc43-docs.standards.buildingsmart.org/
- bSDD (classifications): https://search.bsdd.buildingsmart.org/

Related OpenAEC projects (for cross-references):
- building.py: https://github.com/OpenAEC-Foundation/building-py (Python buildings -> Blender/IFC)
- INB Template: https://github.com/OpenAEC-Foundation/inb-template (Dutch IFC template)

REQUIREMENTS (from REQUIREMENTS.md):
- Minimum 2000 words, English only
- Must include version/feature matrix
- Must include API structure overview
- Must include minimum 10 key concepts
- Must include minimum 5 common error patterns
- Must include code examples per feature area

CONTENT MUST COVER:
1. Bonsai overview: what it is, relationship to BlenderBIM, current status
2. Architecture: tool/core/ui module separation pattern (CRITICAL)
   - How bonsai/core/ defines business logic (read actual source files)
   - How bonsai/tool/ implements Blender-specific operations
   - How bonsai/bim/module/ organizes UI panels and operators
   - The dependency inversion pattern between core and tool
3. IFC integration: how Bonsai bridges Blender objects and IFC entities
   - tool.Ifc interface
   - How Blender objects map to IFC elements
   - The "active IFC file" concept
4. Spatial structure: IfcSite/IfcBuilding/IfcBuildingStorey/IfcSpace
5. Property sets and quantity sets
6. Type system: IfcWallType, IfcSlabType, etc.
7. Modeling workflow: walls, slabs, columns, beams, openings
8. Classification systems: Uniclass, OmniClass, NL-SfB (use bSDD for verification)
9. Geometry representations: body, mapped items, openings/voids
10. IFC export pipeline: MVD, validation, schema version
11. Bonsai Python API: operators (bim.*), core module functions
12. Common error patterns (min 5 with code)
13. AI Common Mistakes section

FORMAT:
- Markdown with tables
- Code examples verified against source code
- Source references with URLs
- Deterministic language (facts, not suggestions)

After completing:
1. Commit to repo and push to GitHub
2. Update SOURCES.md "Last Verified" table for Bonsai
3. Log any key discoveries in LESSONS.md
```

---

## PROMPT A2: Blender Research Verrijking (verbeter bestaand vooronderzoek met nieuwe bronnen)

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Review and enrich the existing Blender vooronderzoek with newly identified sources.

INPUT: docs/research/vooronderzoek-blender.md (54KB, already written)

NEW SOURCES TO VERIFY AGAINST (recently added to SOURCES.md):
- Blender Source (GitHub mirror): https://github.com/blender/blender (use for code search)
- Blender Developer Docs: https://developer.blender.org/ (architecture, release notes)
- Release Notes Index: https://developer.blender.org/docs/release_notes/
- 4.0 Python API: https://developer.blender.org/docs/release_notes/4.0/python_api/
- 4.1 Python API: https://developer.blender.org/docs/release_notes/4.1/python_api/
- 4.2 Python API: https://developer.blender.org/docs/release_notes/4.2/python_api/ (LTS, extension system!)
- 4.3 Python API: https://developer.blender.org/docs/release_notes/4.3/python_api/
- 5.0 Python API: https://developer.blender.org/docs/release_notes/5.0/python_api/ (BGL removal!)
- Python API Changelog: https://docs.blender.org/api/current/change_log.html

Related OpenAEC projects:
- GIS-to-Blender: https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
- AEC Scripts: https://github.com/OpenAEC-Foundation/aec-scripts

CHECK AND ENRICH:
1. Is the version matrix complete? Add 4.1, 4.2, 4.3, 5.0 breaking changes if missing
2. Is the 4.2 extension system documented? (This is the NEW addon format!)
3. Is Blender 5.0 BGL removal mentioned? (Major breaking change for 3D viewport scripts)
4. Are per-version release note URLs included as references?
5. Cross-reference with OpenAEC projects for real-world usage patterns

OUTPUT: Update docs/research/vooronderzoek-blender.md in-place (append new sections, don't delete existing content)

After completing, commit and push. Update SOURCES.md "Last Verified" for Blender.
```

---

## PROMPT A3: IfcOpenShell Research Verrijking

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Review and enrich the existing IfcOpenShell research with newly identified sources.

INPUT FILES:
- docs/research/vooronderzoek-ifcopenshell.md (35KB)
- docs/research/fragments/ (4 files, ~146KB total)
- docs/research/topic-research/ (3 files, ~153KB total)

NEW SOURCES TO VERIFY AGAINST (recently added to SOURCES.md):
- IfcOpenShell Academy: https://academy.ifcopenshell.org/ (tutorials, learning paths)
- Python API Reference: https://ifcopenshell.github.io/ifcopenshell-python/ (auto-generated)
- IfcOpenShell GitHub Org: https://github.com/orgs/IfcOpenShell/repositories (discover sub-projects)
- IFC4.3 Documentation: https://ifc43-docs.standards.buildingsmart.org/
- bSDD: https://search.bsdd.buildingsmart.org/ (classification/property lookup)

Related OpenAEC projects:
- building.py: https://github.com/OpenAEC-Foundation/building-py (Python -> IFC export)
- Monty IFC Viewer: https://github.com/OpenAEC-Foundation/monty-ifc-viewer
- INB Template: https://github.com/OpenAEC-Foundation/inb-template (Dutch IFC)

CHECK AND ENRICH:
1. Is the Academy content referenced? It has step-by-step tutorials
2. Is IFC4.3 fully covered with the new docs URL?
3. Are the auto-generated API docs cross-referenced?
4. Are OpenAEC projects mentioned as real-world usage examples?
5. Is bSDD integration documented (for classification lookups)?

OUTPUT: Update vooronderzoek-ifcopenshell.md in-place. Update SOURCES.md "Last Verified".
```

---

## PROMPT B: Fase 3 - Masterplan Verfijning

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
REPO: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Refine the masterplan based on completed research. This is Phase 3.

BEFORE STARTING, READ THESE FILES IN ORDER:
1. ROADMAP.md → current status
2. REQUIREMENTS.md → quality guarantees per technology
3. DECISIONS.md → architectural constraints (D-001 through D-009)
4. SOURCES.md → approved sources (recently enriched with per-version release notes, OpenAEC projects, Claude platform docs)
5. WAY_OF_WORK.md → skill structure and methodology
6. docs/masterplan/raw-masterplan.md → preliminary skill inventory
7. docs/research/vooronderzoek-blender.md → Blender research (54KB)
8. docs/research/vooronderzoek-bonsai.md → Bonsai research
9. docs/research/vooronderzoek-ifcopenshell.md → IfcOpenShell research (35KB)
10. docs/research/fragments/ → IfcOpenShell deep research (4 files)
11. docs/research/topic-research/ → IfcOpenShell topic research (3 files)

DELIVERABLES:

1. docs/masterplan/masterplan.md - REFINED masterplan:
   a. DEFINITIVE skill inventory (add/merge/remove based on actual API surface from research)
   b. Per skill: exact scope, source sections from vooronderzoek, SOURCES.md URLs
   c. Dependency graph between skills
   d. Build order: core -> syntax -> impl -> errors -> agents (per technology)
   e. Parallel batches (3 agents each, separated file scopes)
   f. Ready-to-use PROMPT per skill with:
      - Specific SOURCES.md URLs for that skill
      - Specific vooronderzoek sections to read
      - REQUIREMENTS.md criteria
      - DECISIONS.md constraints
   g. Quality gate criteria per batch

2. Updated ROADMAP.md:
   - Definitive skill count per technology
   - Phase 3 marked COMPLETE
   - Phase 4/5 detailed with batch schedule

3. Updated DECISIONS.md: any new decisions from refinement

CONSTRAINTS (from DECISIONS.md):
- D-002: Separate packages per technology
- D-003: English only
- D-005: MIT License
- D-008: Build order Blender -> Bonsai -> IfcOpenShell
- D-009: SKILL.md < 500 lines

IMPORTANT: Each skill prompt in the masterplan must include:
- The exact SOURCES.md URLs relevant to that skill
- The exact vooronderzoek sections that provide input
- Version annotations required (from REQUIREMENTS.md)

After completing, commit and push. Update README.md with definitive skill counts.
```

---

## PROMPT C: Skill Writer Template (voor individuele skills)

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Create skill: {SKILL_NAME}

BEFORE STARTING, READ:
- REQUIREMENTS.md → quality criteria
- SOURCES.md → approved documentation URLs
- WAY_OF_WORK.md → skill structure specification
- DECISIONS.md → D-003 (English), D-009 (500 lines)

OUTPUT DIRECTORY: skills/{TECHNOLOGY}/{CATEGORY}/{SKILL_NAME}/

CREATE THESE FILES:

1. SKILL.md (< 500 lines) with:
   ---
   name: {SKILL_NAME}
   description: "{TRIGGER_DESCRIPTION}"
   ---
   - Quick Reference (critical warnings, decision trees)
   - Essential Patterns (with version annotations)
   - Common Operations (verified code snippets)
   - Reference Links (to references/ files)

2. references/methods.md - Complete API signatures
3. references/examples.md - Working code examples (VERIFIED against sources)
4. references/anti-patterns.md - What NOT to do (from real issues)

RESEARCH INPUT: {PATH_TO_RESEARCH_DOC}

APPROVED SOURCES FOR THIS SKILL (from SOURCES.md):
{RELEVANT_URLS}

VERSION ANNOTATIONS REQUIRED: {VERSIONS}

STYLE RULES:
- English only (DECISIONS.md D-003)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- Version-explicit on ALL code blocks
- Verified against official documentation only
- NEVER use: "you might", "consider", "perhaps", "often", "usually"

VERIFICATION CHECKLIST:
- [ ] SKILL.md has valid YAML frontmatter (name + description with trigger words)
- [ ] SKILL.md < 500 lines
- [ ] English only, zero Dutch
- [ ] Deterministic language throughout
- [ ] Version annotations on all code blocks
- [ ] All references/ files exist and are linked from SKILL.md
- [ ] Code examples verified against SOURCES.md URLs
- [ ] Anti-patterns file has at least 3 entries with explanations
```

---

## PROMPT D: Batch Validator (na elke batch)

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Validate batch of skills created by worker agents.

BEFORE STARTING, READ:
- REQUIREMENTS.md → validation criteria
- DECISIONS.md → constraints (D-003 English, D-009 line limit)
- SOURCES.md → approved source URLs for verification

SKILLS TO VALIDATE:
{LIST_OF_SKILL_DIRECTORIES}

VALIDATION CHECKLIST:

1. STRUCTURAL:
   - [ ] SKILL.md exists in directory root
   - [ ] YAML frontmatter has 'name' and 'description' fields
   - [ ] description contains trigger words (when Claude should load this skill)
   - [ ] SKILL.md < 500 lines (count with `wc -l`)
   - [ ] references/ directory exists with at least methods.md, examples.md, anti-patterns.md
   - [ ] All files referenced in SKILL.md exist in references/

2. CONTENT:
   - [ ] English only (grep for Dutch: "gebruik", "voor", "niet", "moet", "deze", "wordt")
   - [ ] Deterministic language (grep for banned: "you might", "consider", "perhaps", "often", "usually")
   - [ ] ALWAYS/NEVER used for critical patterns
   - [ ] Version annotations on code blocks (Blender 3.x/4.x/5.x or IFC2x3/IFC4/IFC4.3)

3. SOURCE VERIFICATION:
   - [ ] Code examples traceable to SOURCES.md approved URLs
   - [ ] No patterns from unverified blog posts
   - [ ] API signatures match official documentation

4. CROSS-REFERENCES:
   - [ ] Skill name in frontmatter matches directory name
   - [ ] References to other skills use correct names
   - [ ] No broken links to reference files

5. QUALITY:
   - [ ] Code examples are syntactically correct Python
   - [ ] Anti-patterns file has at least 3 entries with WHY explanations
   - [ ] Decision trees use clear if/then logic
   - [ ] Quick Reference section exists at top of SKILL.md

OUTPUT: Write validation report to docs/validation/validation-batch-{N}.md
Severity levels: BLOCKER (must fix before merge) | WARNING (should fix) | INFO (nice to have)

If blockers found, generate specific fix prompts per skill that can be copy-pasted.
```

---

## PROMPT E: Sources Verification Session

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Verify and enrich SOURCES.md by checking all listed URLs and discovering new relevant sources.

READ FIRST: SOURCES.md (current state)

FOR EACH TECHNOLOGY SECTION:

1. CHECK all URLs are still valid and current
2. VERIFY the "Purpose" column is accurate
3. DISCOVER new official sources not yet listed:
   - Check GitHub org pages for sub-projects
   - Check official docs for newly added pages
   - Look for migration guides between versions
   - Find official example repositories
4. ADD any newly discovered sources to the appropriate section
5. UPDATE the "Last Verified" table with today's date

SPECIFIC CHECKS:
- Blender: Are there newer release notes (5.1+)? Any new developer docs?
- Bonsai: Is docs.bonsaibim.org still the primary? Any new guides?
- IfcOpenShell: Any new Academy courses? API docs updates?
- IFC Standard: Any new buildingSMART publications?
- Sverchok: Is the ReadTheDocs still maintained?
- Claude/Anthropic: Any new skill development documentation?
- OpenAEC: Any new repos in the org?

OUTPUT: Updated SOURCES.md with verified/new sources and updated dates.
Commit and push after completing.
```

---

## PROMPT F: Deep Research - OpenAEC Ecosystem & Claude Platform Sources

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
REPO: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Deep research on all supplementary sources in SOURCES.md that can contribute to
the quality of our skill package. This includes OpenAEC Foundation projects, Claude/Anthropic
skill development platform, and cross-technology patterns.

READ FIRST:
- SOURCES.md (especially "OpenAEC Foundation Projects" and "Claude / Anthropic" sections)
- REQUIREMENTS.md (what our skills must achieve)
- docs/masterplan/raw-masterplan.md (what skills we're planning)

OUTPUT: docs/research/vooronderzoek-ecosystem-sources.md

---

### PART 1: OpenAEC Foundation Related Projects

Research EACH of these repos. For each: clone/browse, understand what it does, and extract
patterns, code examples, and insights relevant to our skill package.

#### 1a. building.py
URL: https://github.com/OpenAEC-Foundation/building-py
RESEARCH:
- What is it? Python library for buildings -> Blender/IFC export
- How does it use IfcOpenShell? Extract API usage patterns
- How does it generate Blender geometry? Extract bpy patterns
- What IFC entities does it create? Map to our skill inventory
- Code patterns we can reference in our skills as real-world examples
- Any anti-patterns or lessons visible in issues/commits?

#### 1b. GIS-to-Blender Automation
URL: https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
RESEARCH:
- How does it automate Blender via Python? Extract automation patterns
- LLM-driven 3D environment generation - how does it prompt Blender?
- Headless/CLI Blender usage patterns (relevant for blender-impl-automation skill)
- Mesh generation patterns (relevant for blender-syntax-mesh skill)
- Any integration with IFC?

#### 1c. AEC Scripts
URL: https://github.com/OpenAEC-Foundation/aec-scripts
RESEARCH:
- What Python scripts exist for AEC?
- Which use IfcOpenShell? Extract patterns
- Which use Blender? Extract patterns
- Common operations that should be covered by our skills
- Anti-patterns visible in the code

#### 1d. Monty IFC Viewer
URL: https://github.com/OpenAEC-Foundation/monty-ifc-viewer
RESEARCH:
- How does it load/display IFC files?
- IfcOpenShell usage patterns for reading/parsing
- Geometry extraction patterns (ifcopenshell.geom usage)
- Any web-based patterns (could inform future skills)

#### 1e. INB Template
URL: https://github.com/OpenAEC-Foundation/inb-template
RESEARCH:
- What is the Dutch IFC template structure?
- Property set definitions used (relevant for bonsai-syntax-properties)
- Classification system used (relevant for bonsai-impl-classification)
- NL-SfB integration patterns
- Schema version used (IFC2x3? IFC4? IFC4.3?)

#### 1f. Nextcloud Check-in/out
URL: https://github.com/OpenAEC-Foundation/Nextcloud-Check-in-Check-out-feature
RESEARCH:
- File locking patterns for shared Claude Code projects
- Relevant for our multi-agent workflow (agents shouldn't conflict on files)
- Any patterns for concurrent file access that inform our batch strategy?

---

### PART 2: Claude / Anthropic Skill Development Platform

#### 2a. Claude Platform Docs
URL: https://platform.claude.com/docs/en/home
RESEARCH:
- Current skill format specification
- YAML frontmatter requirements (are there fields we're missing?)
- Skill loading behavior (how does Claude discover and load skills?)
- Skill triggering mechanism (how does description field drive selection?)
- Any best practices for skill organization?
- Size limits or recommendations?

#### 2b. Agent SDK / Skills Docs
URL: https://platform.claude.com/docs/en/agent-sdk/skills
RESEARCH:
- Detailed skill specification
- Reference file conventions
- How multiple skills interact when loaded
- Priority/ordering between skills
- Any new features or conventions since ERPNext was developed?

#### 2c. Build with Claude
URL: https://www.anthropic.com/learn/build-with-claude
RESEARCH:
- Development guides relevant to skill authoring
- Best practices for instruction design (our skills ARE instructions)
- Prompt engineering patterns that apply to SKILL.md content
- Any case studies of skill packages?

---

### PART 3: Cross-Technology Patterns

Based on all the above research, identify:

1. COMMON PATTERNS across OpenAEC projects:
   - How do they typically use IfcOpenShell? (create vs read vs modify)
   - How do they typically use Blender? (scripting vs addon vs automation)
   - What IFC operations are most common?
   - What Blender operations are most common?

2. GAPS in our skill inventory:
   - Are there operations used in real OpenAEC projects that our skills don't cover?
   - Are there common workflows we should add skills for?

3. REAL-WORLD EXAMPLES to include in skills:
   - Collect specific code snippets from these repos that demonstrate correct patterns
   - Note which skill each example belongs to
   - These become references/examples.md content

4. ANTI-PATTERNS found in the wild:
   - Common mistakes visible in these repos
   - These become references/anti-patterns.md content

5. CLAUDE SKILL FORMAT INSIGHTS:
   - Any changes to the skill spec since ERPNext?
   - Optimization opportunities for our SKILL.md files
   - Better trigger word patterns based on platform docs

---

FORMAT:
- Markdown with clear sections per source
- Code snippets extracted from repos (with attribution)
- Tables mapping findings to our skill inventory
- "Skill Relevance" annotation for each finding: which skill(s) benefit
- Summary table at the end: source -> key findings -> impacted skills

After completing:
1. Commit and push docs/research/vooronderzoek-ecosystem-sources.md
2. Update SOURCES.md "Last Verified" dates for all researched sources
3. Update LESSONS.md with key discoveries
4. If skill inventory changes needed, note them for Phase 3 masterplan refinement
```

---

## Usage Order

1. **DONE**: Prompt A (Bonsai research) - Phase 2 complete
2. **NOW (parallel)**: Prompt F (ecosystem sources deep research) + Prompt B (masterplan refinement)
3. **Optional parallel**: Prompt A2 + A3 (enrich existing research)
4. **Optional parallel**: Prompt E (verify all source URLs)
5. **After masterplan**: Prompt C (repeated per skill, in batches of 3)
6. **After each batch**: Prompt D (validation)
