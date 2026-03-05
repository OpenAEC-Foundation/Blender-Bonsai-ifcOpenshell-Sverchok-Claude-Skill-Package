# Agent Task: Research OpenAEC Code Repositories

WORKSPACE: /mnt/c/Users/Freek Heijting/Documents/GitHub/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
OUTPUT FILE: docs/research/fragments/ecosystem-openaec-code.md

## Your Task
Research 3 OpenAEC Foundation code repositories and extract patterns relevant to our AEC skill package.

## Repos to Research

### 1. building.py
URL: https://github.com/OpenAEC-Foundation/building-py
- What is it? Python library for buildings -> Blender/IFC export
- How does it use IfcOpenShell? Extract API usage patterns
- How does it generate Blender geometry? Extract bpy patterns
- What IFC entities does it create?
- Code patterns we can reference in our skills as real-world examples
- Any anti-patterns or lessons visible in issues/commits?

### 2. GIS-to-Blender Automation
URL: https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
- How does it automate Blender via Python? Extract automation patterns
- LLM-driven 3D environment generation - how does it prompt Blender?
- Headless/CLI Blender usage patterns
- Mesh generation patterns
- Any integration with IFC?

### 3. AEC Scripts
URL: https://github.com/OpenAEC-Foundation/aec-scripts
- What Python scripts exist for AEC?
- Which use IfcOpenShell? Extract patterns
- Which use Blender? Extract patterns
- Common operations that should be covered by our skills
- Anti-patterns visible in the code

## Research Method
- Use `gh repo clone` to clone each repo to /tmp/ for reading
- Read README, main source files, and key examples
- Use WebFetch on GitHub URLs to browse repo structure
- Extract specific code snippets with file attribution

## Output Format
Write to docs/research/fragments/ecosystem-openaec-code.md with:
- Section per repo with findings
- Code snippets extracted (with file path attribution)
- "Skill Relevance" annotation: which of our planned skills each finding impacts
- Anti-patterns found
- Summary table: repo -> key findings -> impacted skills

Write in English only. Use deterministic language.
