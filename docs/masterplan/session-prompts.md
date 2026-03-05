# Session Prompts - Ready to Use

Copy-paste these prompts to start new Claude sessions or oa agents.

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
- It lives INSIDE the IfcOpenShell repository: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
- We already have vooronderzoek-blender.md (54KB) and vooronderzoek-ifcopenshell.md (35KB)
- This is the LAST missing research before we can refine our masterplan

APPROVED SOURCES (from SOURCES.md):
- Bonsai Documentation: https://docs.bonsaibim.org/
- BlenderBIM (legacy): https://blenderbim.org/
- Bonsai source code: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
- Bonsai Core modules: same repo /src/bonsai/bonsai/core/
- Bonsai Tool modules: same repo /src/bonsai/bonsai/tool/

REQUIREMENTS (from REQUIREMENTS.md):
- Minimum 2000 words, English only
- Must include version matrix
- Must include API structure overview
- Must include minimum 10 key concepts
- Must include minimum 5 common error patterns
- Must include code examples per feature area

CONTENT MUST COVER:
1. Bonsai overview: what it is, relationship to BlenderBIM, current status
2. Architecture: tool/core/ui module separation pattern (CRITICAL - this is unique to Bonsai)
   - How bonsai/core/ defines business logic
   - How bonsai/tool/ implements Blender-specific operations
   - How bonsai/bim/module/ organizes UI panels
3. IFC integration: how Bonsai bridges Blender objects and IFC entities
   - tool.Ifc interface
   - How Blender objects map to IFC elements
   - The "active IFC file" concept
4. Spatial structure management:
   - IfcSite, IfcBuilding, IfcBuildingStorey, IfcSpace
   - Creating and navigating spatial hierarchy
   - Containment vs aggregation relationships
5. Property sets and quantity sets:
   - Creating, editing, applying property sets
   - Pset templates
   - Quantity takeoff via Bonsai
6. Type system:
   - IfcWallType, IfcSlabType, IfcColumnType etc.
   - Type assignment and inheritance
   - Parametric type definitions
7. Modeling workflow:
   - Walls, slabs, columns, beams, openings
   - Profile-based vs extrusion-based geometry
   - Void/opening creation
8. Classification systems:
   - Uniclass, OmniClass, NL-SfB integration
   - How to apply classifications via Bonsai
9. Geometry representations:
   - Body, mapped items, bounding box
   - Representation contexts
   - Boolean operations (openings/voids)
10. IFC export pipeline:
    - MVD selection (Coordination View, Design Transfer View)
    - Validation and quality checks before export
    - Schema version selection
11. Bonsai Python API for scripting:
    - How to automate Bonsai operations via Python
    - Key operators (bim.*)
    - Core module function signatures
12. Common error patterns (minimum 5 with code examples):
    - Schema violations
    - Missing required attributes
    - Geometry errors
    - Spatial structure mistakes
    - Type assignment issues
13. AI Common Mistakes section:
    - What Claude typically gets wrong about Bonsai
    - Confusion between Blender operations and BIM operations
    - Missing IFC context when scripting

FORMAT:
- Markdown with tables for version/feature matrices
- Code examples for each major feature area
- Source references (URLs to official docs or source code)
- Deterministic language (facts, not suggestions)

After completing, commit to the repo and push to GitHub.
```

---

## PROMPT B: Fase 3 - Masterplan Verfijning

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
REPO: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Refine the masterplan based on completed research. This is Phase 3 of the 7-phase methodology.

BEFORE STARTING, READ:
1. ROADMAP.md (current status)
2. REQUIREMENTS.md (quality guarantees)
3. DECISIONS.md (architectural decisions)
4. SOURCES.md (approved sources)
5. docs/masterplan/raw-masterplan.md (preliminary skill inventory)
6. docs/research/vooronderzoek-blender.md (Blender research)
7. docs/research/vooronderzoek-bonsai.md (Bonsai research)
8. docs/research/vooronderzoek-ifcopenshell.md (IfcOpenShell research)

DELIVERABLES:
1. docs/masterplan/masterplan.md - REFINED masterplan containing:
   a. DEFINITIVE skill inventory (add/merge/remove skills based on research)
   b. Per skill: exact scope, which vooronderzoek sections it draws from
   c. Dependencies between skills mapped
   d. Build order per technology (core -> syntax -> impl -> errors -> agents)
   e. Parallel execution batches (3 agents per batch, separated file scopes)
   f. Ready-to-use PROMPT per skill (so agents can execute directly)
   g. Quality gate criteria per batch

2. Updated ROADMAP.md with:
   - Definitive skill count per technology
   - Phase 3 marked complete
   - Phase 4/5 detailed next steps

3. Updated DECISIONS.md with any new decisions from the refinement

CONSTRAINTS:
- Skills must be per-technology packages (D-002)
- SKILL.md < 500 lines each (D-009)
- English only (D-003)
- Each skill prompt must reference SOURCES.md URLs
- Each skill prompt must include REQUIREMENTS.md criteria

After completing, commit and push to GitHub. Update README.md if skill counts changed.
```

---

## PROMPT C: Skill Writer Template (voor individuele skills)

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Create skill: {SKILL_NAME}

OUTPUT DIRECTORY: skills/{TECHNOLOGY}/{CATEGORY}/{SKILL_NAME}/

CREATE THESE FILES:
1. SKILL.md (< 500 lines) with:
   ---
   name: {SKILL_NAME}
   description: "{TRIGGER_DESCRIPTION}"
   ---
   - Quick Reference section (critical warnings, decision trees)
   - Essential Patterns (with version annotations)
   - Common Operations (code snippets)
   - Reference Links (to references/ files)

2. references/methods.md - Complete API signatures
3. references/examples.md - Working code examples (verified)
4. references/anti-patterns.md - What NOT to do with explanations

RESEARCH INPUT: {PATH_TO_RESEARCH_DOC}
APPROVED SOURCES: {RELEVANT_URLS_FROM_SOURCES_MD}

REQUIREMENTS (from REQUIREMENTS.md):
- English only
- Deterministic language: "ALWAYS use X when Y" / "NEVER do X because Y"
- Version-explicit: {VERSION_ANNOTATIONS}
- All code verified against official documentation
- Anti-patterns identified from real issues

STRUCTURE REFERENCE: See WAY_OF_WORK.md for detailed skill structure.

After completing, verify:
- [ ] SKILL.md has valid YAML frontmatter
- [ ] SKILL.md < 500 lines
- [ ] English only, no Dutch
- [ ] Deterministic language throughout
- [ ] Version annotations on all code
- [ ] All references/ files exist and are linked from SKILL.md
```

---

## PROMPT D: Batch Validator (na elke batch)

```
PROJECT: Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
WORKSPACE: C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

TASK: Validate batch of skills created by worker agents.

SKILLS TO VALIDATE:
{LIST_OF_SKILL_DIRECTORIES}

VALIDATION CHECKLIST (from REQUIREMENTS.md + DECISIONS.md):
For EACH skill directory:

1. STRUCTURAL:
   - [ ] SKILL.md exists in directory root
   - [ ] YAML frontmatter has 'name' and 'description' fields
   - [ ] SKILL.md < 500 lines (count with `wc -l`)
   - [ ] references/ directory exists
   - [ ] All files referenced in SKILL.md exist in references/

2. CONTENT:
   - [ ] English only (grep for common Dutch words: "gebruik", "voor", "niet", "moet")
   - [ ] Deterministic language (no "you might", "consider", "perhaps", "often")
   - [ ] ALWAYS/NEVER used for critical patterns
   - [ ] Version annotations present (Blender 3.x/4.x or IFC2x3/IFC4/IFC4.3)

3. CROSS-REFERENCES:
   - [ ] Skill name in frontmatter matches directory name
   - [ ] References to other skills use correct names
   - [ ] No broken links to reference files

4. QUALITY:
   - [ ] Code examples are syntactically correct
   - [ ] Anti-patterns file has at least 3 entries
   - [ ] Decision trees use clear if/then logic

OUTPUT: Write validation report to docs/research/validation-batch-{N}.md
Flag any failures with severity: BLOCKER (must fix) or WARNING (should fix).

If blockers found, list specific fix instructions per skill.
```

---

## Usage Order

1. **Now**: Use Prompt A (Bonsai research) - blocks everything else
2. **After Bonsai research**: Use Prompt B (Masterplan refinement)
3. **After masterplan**: Use Prompt C (repeated per skill, in batches of 3)
4. **After each batch**: Use Prompt D (validation)
