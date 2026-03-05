# Agent Task: Research Claude/Anthropic Skill Development Platform

WORKSPACE: /mnt/c/Users/Freek Heijting/Documents/GitHub/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
OUTPUT FILE: docs/research/fragments/ecosystem-claude-platform.md

## Your Task
Research the Claude/Anthropic skill development platform documentation to optimize our skill package format.

## Sources to Research

### 1. Claude Platform Docs
URL: https://platform.claude.com/docs/en/home
- Current skill format specification
- YAML frontmatter requirements (are there fields we're missing?)
- Skill loading behavior (how does Claude discover and load skills?)
- Skill triggering mechanism (how does description field drive selection?)
- Best practices for skill organization
- Size limits or recommendations

### 2. Agent SDK / Skills Docs
URL: https://platform.claude.com/docs/en/agent-sdk/skills
- Detailed skill specification
- Reference file conventions
- How multiple skills interact when loaded
- Priority/ordering between skills
- Any new features or conventions

### 3. Build with Claude
URL: https://www.anthropic.com/learn/build-with-claude
- Development guides relevant to skill authoring
- Best practices for instruction design (our skills ARE instructions)
- Prompt engineering patterns that apply to SKILL.md content
- Any case studies of skill packages

## Also check
- Claude Code documentation for SKILL.md format
- Any updates to the skill/slash-command specification
- WebSearch for "Claude Code custom skills" or "Claude Code SKILL.md" for latest info

## Research Method
- Use WebFetch to read each URL
- Use WebSearch for supplementary info
- Read our existing WAY_OF_WORK.md and REQUIREMENTS.md for comparison
- Note any gaps between our current format and the official spec

## Also read for comparison
- The ERPNext Skill Package repo for proven patterns: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
- Our WAY_OF_WORK.md in the workspace

## Output Format
Write to docs/research/fragments/ecosystem-claude-platform.md with:
- Current official skill spec (what we know)
- Gap analysis: our format vs official recommendations
- Optimization opportunities for SKILL.md files
- Better trigger word patterns based on platform docs
- Recommendations for our skill package

Write in English only. Use deterministic language.
