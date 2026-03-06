# Agent Task: Synthesize Ecosystem Research into Final Document

WORKSPACE: /mnt/c/Users/Freek Heijting/Documents/GitHub/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
OUTPUT FILE: docs/research/vooronderzoek-ecosystem-sources.md

## Your Task
Combine the 3 research fragments into the final vooronderzoek-ecosystem-sources.md document, and add Part 3: Cross-Technology Patterns analysis.

## Input Files (READ ALL FIRST)
1. docs/research/fragments/ecosystem-openaec-code.md (building.py, GIS-to-Blender, AEC Scripts)
2. docs/research/fragments/ecosystem-openaec-ifc.md (Monty IFC Viewer, INB Template, Nextcloud)
3. docs/research/fragments/ecosystem-claude-platform.md (Claude platform docs)
4. docs/masterplan/raw-masterplan.md (our planned skill inventory - for gap analysis)
5. REQUIREMENTS.md (what our skills must achieve)

## Document Structure
Create docs/research/vooronderzoek-ecosystem-sources.md with this structure:

### Part 1: OpenAEC Foundation Related Projects
- Combine findings from fragments 1 and 2
- Organize by repo with findings, code patterns, skill relevance

### Part 2: Claude / Anthropic Skill Development Platform
- Include findings from fragment 3
- Focus on format spec, optimization opportunities

### Part 3: Cross-Technology Patterns (YOU WRITE THIS)
Based on ALL research, identify:
1. COMMON PATTERNS across OpenAEC projects (how they use IfcOpenShell, Blender)
2. GAPS in our skill inventory (operations used in real projects we don't cover)
3. REAL-WORLD EXAMPLES to include in skills (code snippets from repos)
4. ANTI-PATTERNS found in the wild
5. CLAUDE SKILL FORMAT INSIGHTS (changes/optimizations for our format)

### Summary Table
At the end: source -> key findings -> impacted skills

## Quality Requirements
- English only
- Minimum 2000 words total
- Deterministic language (ALWAYS/NEVER, not "you might")
- Code snippets with attribution
- All findings mapped to our skill inventory

After writing, also update SOURCES.md "Last Verified" dates for all researched sources.
