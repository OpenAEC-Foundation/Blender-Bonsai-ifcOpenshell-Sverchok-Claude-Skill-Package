# Agent Task: Research OpenAEC IFC & Infrastructure Repos

WORKSPACE: /mnt/c/Users/Freek Heijting/Documents/GitHub/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
OUTPUT FILE: docs/research/fragments/ecosystem-openaec-ifc.md

## Your Task
Research 3 OpenAEC Foundation repositories focused on IFC and infrastructure patterns.

## Repos to Research

### 1. Monty IFC Viewer
URL: https://github.com/OpenAEC-Foundation/monty-ifc-viewer
- How does it load/display IFC files?
- IfcOpenShell usage patterns for reading/parsing
- Geometry extraction patterns (ifcopenshell.geom usage)
- Any web-based patterns

### 2. INB Template
URL: https://github.com/OpenAEC-Foundation/inb-template
- What is the Dutch IFC template structure?
- Property set definitions used (relevant for bonsai-syntax-properties skill)
- Classification system used (relevant for bonsai-impl-classification skill)
- NL-SfB integration patterns
- Schema version used (IFC2x3? IFC4? IFC4.3?)

### 3. Nextcloud Check-in/out
URL: https://github.com/OpenAEC-Foundation/Nextcloud-Check-in-Check-out-feature
- File locking patterns for shared projects
- Relevant for our multi-agent workflow
- Patterns for concurrent file access that inform our batch strategy

## Research Method
- Use `gh repo clone` to clone each repo to /tmp/ for reading
- Read README, main source files, and key examples
- Use WebFetch on GitHub URLs if needed
- Extract specific code snippets with file attribution

## Output Format
Write to docs/research/fragments/ecosystem-openaec-ifc.md with:
- Section per repo with findings
- Code snippets extracted (with file path attribution)
- "Skill Relevance" annotation: which skills each finding impacts
- Anti-patterns found
- Summary table: repo -> key findings -> impacted skills

Write in English only. Use deterministic language.
