# Ecosystem Research: Claude/Anthropic Skill Development Platform

**Date**: 2026-03-05
**Researcher**: research-claude-platform agent
**Status**: Complete

---

## 1. Official Skill Specification (Agent Skills Open Standard)

Claude Code skills follow the [Agent Skills](https://agentskills.io) open standard, which works across multiple AI tools. Claude Code extends the standard with additional features.

### 1.1 Directory Structure (Required)

```
skill-name/
├── SKILL.md              # Required - main instructions
├── scripts/              # Optional - executable code
├── references/           # Optional - documentation loaded on demand
└── assets/               # Optional - templates, images, data files
```

The directory name MUST match the `name` field in the YAML frontmatter.

### 1.2 SKILL.md Format

The file MUST contain YAML frontmatter followed by Markdown content.

#### Required Frontmatter Fields

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | Yes | Max 64 chars. Lowercase letters, numbers, hyphens only. Must not start/end with hyphen. No consecutive hyphens. Must match parent directory name. |
| `description` | Yes (recommended in Claude Code) | Max 1024 chars. Non-empty. Describes what the skill does AND when to use it. No XML tags. |

#### Optional Frontmatter Fields (Agent Skills Standard)

| Field | Constraints |
|-------|-------------|
| `license` | License name or reference to bundled license file |
| `compatibility` | Max 500 chars. Environment requirements (product, packages, network) |
| `metadata` | Arbitrary key-value mapping (e.g., author, version) |
| `allowed-tools` | Space-delimited list of pre-approved tools (experimental) |

#### Claude Code-Specific Frontmatter Fields

These fields extend the open standard and are specific to Claude Code:

| Field | Default | Purpose |
|-------|---------|---------|
| `disable-model-invocation` | `false` | Set `true` to prevent Claude from auto-loading this skill. User must invoke with `/name`. |
| `user-invocable` | `true` | Set `false` to hide from `/` menu. Skill remains available for Claude to auto-invoke. |
| `argument-hint` | - | Hint shown during autocomplete (e.g., `[issue-number]`). |
| `model` | - | Model to use when skill is active. |
| `context` | - | Set to `fork` to run in a forked subagent context. |
| `agent` | `general-purpose` | Which subagent type when `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom. |
| `hooks` | - | Hooks scoped to this skill's lifecycle. |

#### Invocation Control Matrix

| Frontmatter | User can invoke | Claude can invoke | Context behavior |
|-------------|----------------|-------------------|------------------|
| (default) | Yes | Yes | Description always in context; full skill loads when invoked |
| `disable-model-invocation: true` | Yes | No | Description NOT in context; full skill loads when user invokes |
| `user-invocable: false` | No | Yes | Description always in context; full skill loads when invoked |

**Critical distinction**: `user-invocable: false` is a UI setting only. It does NOT prevent Claude from triggering the skill. To prevent model invocation, MUST use `disable-model-invocation: true`.

### 1.3 Markdown Body Content

No format restrictions from the specification. Recommended sections:
- Step-by-step instructions
- Examples of inputs and outputs
- Common edge cases

#### String Substitutions (Claude Code)

| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All arguments passed when invoking |
| `$ARGUMENTS[N]` or `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing SKILL.md |

#### Dynamic Context Injection (Claude Code)

The `` !`command` `` syntax runs shell commands before skill content is sent to Claude. Output replaces the placeholder.

---

## 2. Skill Discovery and Loading Mechanism

### 2.1 Three-Level Progressive Disclosure

Skills use a three-level loading system to minimize context window consumption:

| Level | When Loaded | Token Cost | Content |
|-------|------------|------------|---------|
| **Level 1: Metadata** | Always (at startup) | ~100 tokens per skill | `name` and `description` from YAML frontmatter |
| **Level 2: Instructions** | When skill is triggered | < 5000 tokens recommended | SKILL.md body |
| **Level 3: Resources** | As needed | Effectively unlimited | Bundled files (scripts, references, assets) |

### 2.2 Description-Driven Discovery

Claude uses the `description` field to decide when to invoke a skill. The description is injected into the system prompt at session startup. This is the PRIMARY trigger mechanism.

**Description budget**: All skill descriptions share a budget of 2% of the context window, with a fallback of 16,000 characters. Override with `SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable.

**Description writing rules**:
- ALWAYS write in third person ("Processes Excel files", not "I can help you" or "You can use this")
- Include BOTH what the skill does AND when to use it
- Include specific keywords that match user queries
- Be specific, not vague
- Max 1024 characters

### 2.3 Skill Location Hierarchy

| Priority | Location | Scope |
|----------|----------|-------|
| 1 (highest) | Enterprise managed settings | All org users |
| 2 | Personal `~/.claude/skills/<name>/SKILL.md` | All user's projects |
| 3 | Project `.claude/skills/<name>/SKILL.md` | This project only |
| 4 | Plugin `<plugin>/skills/<name>/SKILL.md` | Where plugin enabled |

When skills share the same name across levels, higher-priority locations win. Plugin skills use `plugin-name:skill-name` namespace (no conflicts).

### 2.4 Automatic Discovery

- **Nested directories**: Claude Code discovers skills from nested `.claude/skills/` (e.g., `packages/frontend/.claude/skills/`) — supports monorepos.
- **Additional directories**: Skills in `--add-dir` directories are loaded automatically with live change detection.
- **Backward compatibility**: `.claude/commands/` files still work and support the same frontmatter. If a skill and command share a name, the skill takes precedence.

---

## 3. Skill Content Best Practices (Official Anthropic Guidance)

### 3.1 Core Principles

1. **Concise is key**: Context window is a shared resource. Challenge every token: "Does Claude really need this explanation?"
2. **Set appropriate degrees of freedom**: Match specificity to task fragility (high freedom for flexible tasks, low freedom for fragile operations).
3. **Test with all models**: Haiku needs more guidance, Opus needs less. Aim for instructions that work across models.

### 3.2 SKILL.md Body Recommendations

- Keep SKILL.md body **under 500 lines**
- Move detailed reference material to separate files
- Use **progressive disclosure**: SKILL.md = overview + navigation; details in referenced files
- Keep file references **one level deep** (no nested references)
- Include **table of contents** for reference files over 100 lines
- Use **forward slashes** in file paths (never backslashes)
- Name files descriptively (`form_validation_rules.md`, not `doc2.md`)

### 3.3 Description Optimization

The description is the MOST critical field for skill activation.

**Effective pattern**:
```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Anti-patterns**:
```yaml
description: Helps with documents       # Too vague
description: Processes data              # Too vague
description: Does stuff with files       # Useless
```

**Naming conventions** (from best practices):
- Prefer gerund form: `processing-pdfs`, `analyzing-spreadsheets`
- Acceptable: noun phrases (`pdf-processing`) or action-oriented (`process-pdfs`)
- Avoid: vague names (`helper`, `utils`), generic (`documents`, `data`), reserved words (`anthropic-*`, `claude-*`)

### 3.4 Content Patterns

**Template pattern**: Provide output format templates with appropriate strictness.

**Examples pattern**: Include input/output pairs for quality-dependent tasks.

**Conditional workflow pattern**: Guide through decision points.

**Feedback loop pattern**: Run validator → fix errors → repeat. Greatly improves output quality.

**Workflow checklist pattern**: For complex multi-step tasks, provide a checklist Claude can track.

### 3.5 Anti-Patterns to Avoid

- Windows-style paths (`scripts\helper.py`)
- Offering too many options without a default
- Deeply nested file references
- Time-sensitive information (use "old patterns" section instead)
- Inconsistent terminology
- Assuming tools/packages are installed
- Voodoo constants (undocumented magic numbers)
- Punting errors to Claude instead of handling them

---

## 4. Gap Analysis: Our Format vs. Official Specification

### 4.1 What We Have (WAY_OF_WORK.md / REQUIREMENTS.md)

Our current skill format:
```yaml
---
name: {tech}-{category}-{topic}
description: "Deterministic [description]. Use this skill when Claude needs to [trigger scenario]..."
---
```

Content sections:
1. Quick Reference (critical warnings, decision trees)
2. Essential Patterns (with version annotations)
3. Common Operations (code snippets)
4. Reference Links (to references/ files)

Directory layout:
```
skill-name/
├── SKILL.md              # Main file, < 500 lines
└── references/
    ├── methods.md        # Complete API signatures
    ├── examples.md       # Working code examples
    └── anti-patterns.md  # What NOT to do
```

### 4.2 Gaps Identified

#### A. Missing Frontmatter Fields

| Field | Status | Action Required |
|-------|--------|----------------|
| `name` | Present | Validate: must match directory name, lowercase+hyphens only, max 64 chars |
| `description` | Present | Verify third-person, max 1024 chars, includes trigger words |
| `license` | Missing | Add `license: MIT` (or project license) |
| `compatibility` | Missing | Add `compatibility: Designed for Claude Code. Requires Python 3.x.` |
| `metadata` | Missing | Consider adding `author`, `version` fields |
| `disable-model-invocation` | Not used | Correct — our skills should auto-invoke |
| `user-invocable` | Not used | Consider for reference-only skills (e.g., core/ category) |
| `allowed-tools` | Not used | Consider for safety on certain skills |

#### B. Naming Convention Mismatch

**Our convention**: `{tech}-{category}-{topic}` (e.g., `blender-syntax-operators`)
**Official convention**: Lowercase, hyphens, max 64 chars, gerund form preferred

**Issue**: Our naming scheme is valid but differs from the recommended gerund pattern. However, our `{tech}-{category}-{topic}` pattern provides better discoverability for domain-specific skills. This is an acceptable deviation.

**Action**: Ensure all names pass validation (no uppercase, no consecutive hyphens, no leading/trailing hyphens, max 64 chars).

#### C. Description Format

**Our pattern**: `"Deterministic [description]. Use this skill when Claude needs to [trigger scenario]..."`
**Official guidance**: Third-person, includes what it does AND when to use it, specific keywords.

**Issue**: Our "Deterministic" prefix wastes description tokens. The word has no trigger value.

**Action**: Remove "Deterministic" prefix. Lead with action verbs in third person. Include domain-specific trigger keywords.

**Before**: `"Deterministic guide for IfcOpenShell API categories. Use this skill when Claude needs to use ifcopenshell.api.run()..."`

**After**: `"Navigates IfcOpenShell API categories (root, spatial, geometry, material, type, pset, classification). Use when writing ifcopenshell.api.run() calls, creating IFC elements, or working with IFC spatial structures."`

#### D. Missing Directory Types

**Our structure**: Only `references/` subdirectory.
**Official structure**: `scripts/`, `references/`, `assets/`

**Action**: Consider adding `scripts/` for validation scripts (e.g., version-checking scripts, IFC schema validators).

#### E. Reference File Organization

**Our approach**: `methods.md`, `examples.md`, `anti-patterns.md`
**Official approach**: Domain-organized files, table of contents for files > 100 lines

**Assessment**: Our structure is valid and well-organized. The three-file pattern (methods, examples, anti-patterns) aligns with the progressive disclosure model. No change needed.

#### F. Content Language Style

**Our standard**: Imperative, deterministic (ALWAYS/NEVER)
**Official guidance**: "Avoid heavy-handed MUSTs and ALWAYS statements. Explain the reasoning so Claude understands the 'why'."

**Issue**: Direct conflict. The official skill-creator skill explicitly recommends AGAINST overuse of ALWAYS/NEVER/MUST. Instead, it recommends explaining the reasoning.

**Recommended action**: Keep deterministic language for truly critical rules (version-specific breaking changes, known API traps). Add brief "why" explanations. Relax ALWAYS/NEVER for preferences vs. requirements.

**Before**: `"ALWAYS use bpy.context.view_layer.objects.active instead of bpy.context.active_object"`
**After**: `"Use bpy.context.view_layer.objects.active instead of bpy.context.active_object — the latter is read-only and raises AttributeError on assignment in Blender 4.x."`

---

## 5. Optimization Opportunities

### 5.1 Description Trigger Optimization

Based on the skill-creator skill's description optimization workflow:

1. Generate 20 trigger eval queries (10 should-trigger, 10 should-not-trigger)
2. Test description activation accuracy
3. Iterate on description wording

**For our domain skills, recommended trigger patterns**:

```yaml
# IfcOpenShell skills
description: "Provides IfcOpenShell API method signatures and usage patterns for ifcopenshell.api.run(), ifcopenshell.file, and ifcopenshell.util. Use when writing Python code that creates, reads, modifies, or validates IFC files, BIM models, or OpenBIM data."

# Blender skills
description: "Guides Blender Python API (bpy) operator registration, property definitions, and panel creation for Blender 3.x and 4.x. Use when writing Blender addons, extensions, operators, or scripts using bpy.types, bpy.props, or bpy.ops."

# Bonsai skills
description: "Covers Bonsai BIM addon patterns including tool.Ifc, spatial structure creation, and property set management. Use when writing Bonsai-specific code, working with IFC in Blender, or extending the Bonsai BIM addon."
```

**Key trigger keywords to include per technology**:
- **Blender**: bpy, addon, extension, operator, modifier, bmesh, mesh, context, panel, property
- **IfcOpenShell**: ifc, ifcopenshell, ifc2x3, ifc4, ifc4.3, bim, openbim, pset, spatial structure
- **Bonsai**: bonsai, blenderbim, tool.Ifc, bim addon, ifc in blender
- **Sverchok**: sverchok, node, parametric, visual programming, data flow

### 5.2 Progressive Disclosure Optimization

Current SKILL.md structure loads ALL content at once. Optimize by:

1. Keep SKILL.md under 500 lines (current requirement matches)
2. Move method signatures entirely to `references/methods.md`
3. Move examples entirely to `references/examples.md`
4. SKILL.md becomes: quick-reference + decision trees + links to reference files
5. Add table of contents to any reference file > 100 lines

### 5.3 Evaluation Framework

Adopt the official evaluation pattern from the skill-creator skill:

```json
{
  "skill_name": "ifcos-api-categories",
  "evals": [
    {
      "id": 1,
      "prompt": "Create an IFC wall element using IfcOpenShell",
      "expected_output": "Code using ifcopenshell.api.run('root.create_entity'...)",
      "files": []
    }
  ]
}
```

Save to `evals/evals.json` per skill for regression testing.

### 5.4 Metadata Fields

Add to all skills:
```yaml
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
  technologies: "blender,ifcopenshell,bonsai,sverchok"
```

### 5.5 Compatibility Field

Add where relevant:
```yaml
compatibility: Designed for Claude Code. Requires Python 3.x and knowledge of Blender/IfcOpenShell APIs.
```

---

## 6. Comparison with ERPNext Skill Package

The ERPNext Skill Package (28 skills) is the reference implementation from which our WAY_OF_WORK.md derives.

### 6.1 ERPNext Patterns We Should Keep

- **Research-first methodology**: 7-phase approach (proven effective)
- **Category system**: syntax, core, impl, errors, agents
- **Deterministic language**: Critical for preventing API hallucination
- **Version matrix**: Essential for multi-version technologies
- **Anti-pattern documentation**: Primary differentiator from generic AI output

### 6.2 ERPNext Patterns to Update Based on New Research

| ERPNext Pattern | Official Guidance | Recommended Update |
|----------------|-------------------|-------------------|
| Heavy ALWAYS/NEVER | Explain the "why" | Add reasoning to critical rules |
| No `license` field | Include license | Add `license: MIT` |
| No `metadata` | Include author/version | Add metadata block |
| No `compatibility` | Include when needed | Add for Claude Code targeting |
| No evaluation framework | Build evals first | Add `evals/evals.json` per skill |
| Fixed description pattern | Optimized trigger words | Rewrite descriptions per §5.1 |

---

## 7. Recommendations for Our Skill Package

### 7.1 Immediate Actions (Before Skill Creation)

1. **Update SKILL.md template** to include all official frontmatter fields:
   ```yaml
   ---
   name: {tech}-{category}-{topic}
   description: "[Third-person action verb] [specific capability]. Use when [specific trigger scenarios with domain keywords]."
   license: MIT
   compatibility: Designed for Claude Code. Requires Python 3.x.
   metadata:
     author: OpenAEC-Foundation
     version: "1.0"
   ---
   ```

2. **Rewrite description pattern**: Remove "Deterministic" prefix. Lead with verbs. Include technology-specific keywords. Max 1024 chars.

3. **Add evaluation framework**: Create `evals/evals.json` for each skill with 3+ test scenarios.

4. **Validate names**: Ensure all skill names pass: lowercase, hyphens only, max 64 chars, match directory name, no consecutive hyphens.

### 7.2 Structural Changes

1. **Add table of contents** to reference files over 100 lines.
2. **Consider `scripts/` directory** for skills that benefit from validation scripts.
3. **Soften language where appropriate**: Use ALWAYS/NEVER only for genuinely critical rules. Add "because [reason]" to each constraint.

### 7.3 Quality Assurance

1. **Test with multiple models**: At minimum Sonnet and Haiku (Opus has better recall, Haiku needs more guidance).
2. **Run trigger evaluations**: 20 queries per skill (10 should-trigger, 10 should-not-trigger).
3. **Monitor context budget**: With 20+ skills, descriptions may exceed the 16k character budget. Keep descriptions concise.

### 7.4 Distribution Strategy

- **Claude Code**: Commit `.claude/skills/` to version control (primary distribution)
- **Plugin format**: Package as Claude Code plugin for easy installation
- **Claude.ai**: Provide ZIP download instructions per skill
- **API**: Document skill_id usage for programmatic access

---

## 8. Sources

- [Agent Skills Open Standard Specification](https://agentskills.io/specification)
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [Agent Skills Overview (Anthropic Platform)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Skill Authoring Best Practices (Anthropic Platform)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Anthropic Skills Repository](https://github.com/anthropics/skills)
- [Skill Creator SKILL.md](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [Claude Custom Skills Help Center](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
- [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package)
- [Claude Agent Skills Deep Dive (Lee Han Chung)](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
