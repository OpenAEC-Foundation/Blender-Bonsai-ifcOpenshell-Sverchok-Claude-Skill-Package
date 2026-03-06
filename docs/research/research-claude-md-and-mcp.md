# Research: CLAUDE.md Best Practices for Claude Code

**Date**: 2026-03-06
**Researcher**: research-claude-md agent
**Status**: Complete

---

## Table of Contents

1. [Official Format and Structure Recommendations](#1-official-format-and-structure-recommendations)
2. [File Hierarchy and Loading Order](#2-file-hierarchy-and-loading-order)
3. [How CLAUDE.md Files Load](#3-how-claudemd-files-load)
4. [Size Limits and Recommendations](#4-size-limits-and-recommendations)
5. [What Should Go Into CLAUDE.md](#5-what-should-go-into-claudemd)
6. [What Should NOT Go Into CLAUDE.md](#6-what-should-not-go-into-claudemd)
7. [How CLAUDE.md Interacts with Skills](#7-how-claudemd-interacts-with-skills)
8. [How to Reference Skills from CLAUDE.md](#8-how-to-reference-skills-from-claudemd)
9. [The @import System](#9-the-import-system)
10. [Organizing Rules with .claude/rules/](#10-organizing-rules-with-clauderules)
11. [Monorepo and Large Team Management](#11-monorepo-and-large-team-management)
12. [Auto Memory System](#12-auto-memory-system)
13. [Writing Effective Instructions](#13-writing-effective-instructions)
14. [Common Failure Patterns](#14-common-failure-patterns)
15. [Skills Best Practices (Complementary)](#15-skills-best-practices-complementary)
16. [Sources](#16-sources)

---

## 1. Official Format and Structure Recommendations

### File Naming
- The filename is **case-sensitive**: it must be exactly `CLAUDE.md` (uppercase CLAUDE, lowercase .md).
- Alternative location: `.claude/CLAUDE.md` (within the .claude config directory).
- Personal per-project preferences: `CLAUDE.local.md` (automatically added to `.gitignore`).

### Format
There is **no required format** for CLAUDE.md files. However, the official documentation recommends:

- **Use markdown**: headers and bullets to group related instructions.
- **Keep it human-readable**: treat it like documentation that both humans and Claude need to understand quickly.
- **Use clear sections**: Claude scans structure the same way readers do -- organized sections are easier to follow than dense paragraphs.

### The /init Command
Run `/init` to generate a starting CLAUDE.md automatically. Claude analyzes the codebase and creates a file with build commands, test instructions, and project conventions it discovers. If a CLAUDE.md already exists, `/init` suggests improvements rather than overwriting it. The output should be refined with instructions Claude would not discover on its own.

### Example of a Well-Structured CLAUDE.md
From the official best practices documentation:

```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

---

## 2. File Hierarchy and Loading Order

CLAUDE.md files can live in several locations, each with a different scope. **More specific locations take precedence over broader ones.**

| Scope | Location | Purpose | Shared With |
|-------|----------|---------|-------------|
| **Managed policy** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`; Linux/WSL: `/etc/claude-code/CLAUDE.md`; Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | Organization-wide instructions managed by IT/DevOps | All users in organization |
| **Project instructions** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team-shared instructions for the project | Team members via source control |
| **User instructions** | `~/.claude/CLAUDE.md` | Personal preferences for all projects | Just you (all projects) |
| **Local instructions** | `./CLAUDE.local.md` | Personal project-specific preferences, not checked into git | Just you (current project) |

### Loading Priority
- **User-level rules** (`~/.claude/rules/`) are loaded **before** project rules, giving project rules **higher priority**.
- **Managed policy** CLAUDE.md files **cannot be excluded** -- they always apply regardless of individual settings.
- When multiple CLAUDE.md files exist, all are loaded and Claude attempts to follow all instructions. There is no explicit override system; rather, more specific locations have higher priority.

### Priority Order (highest to lowest)
1. Managed policy (enterprise, cannot be excluded)
2. Project CLAUDE.md (`./.claude/CLAUDE.md` or `./CLAUDE.md`)
3. Project `.claude/rules/*.md` files
4. User CLAUDE.md (`~/.claude/CLAUDE.md`)
5. User rules (`~/.claude/rules/*.md`)
6. Local overrides (`./CLAUDE.local.md`)

---

## 3. How CLAUDE.md Files Load

### Startup Loading
Claude Code reads CLAUDE.md files by **walking up the directory tree** from the current working directory, checking each directory along the way for `CLAUDE.md` and `CLAUDE.local.md` files. If you run Claude Code in `foo/bar/`, it loads instructions from both `foo/bar/CLAUDE.md` and `foo/CLAUDE.md`.

### On-Demand Loading (Subdirectories)
CLAUDE.md files in **subdirectories** under the current working directory are **not loaded at launch**. They are included **when Claude reads files in those subdirectories** during the session. This is lazy loading -- when Claude starts working in a `frontend/` directory, it picks up the `frontend/CLAUDE.md` at that point.

### Additional Directories
The `--add-dir` flag gives Claude access to additional directories. By default, **CLAUDE.md files from these directories are NOT loaded**. To also load them, set the environment variable:
```bash
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 claude --add-dir ../shared-config
```

### After Compaction
CLAUDE.md **fully survives compaction**. After `/compact`, Claude re-reads CLAUDE.md from disk and re-injects it fresh into the session. If an instruction disappeared after compaction, it was given only in conversation, not written to CLAUDE.md.

---

## 4. Size Limits and Recommendations

### Official Recommendation
- **Target under 200 lines** per CLAUDE.md file (from official Anthropic documentation).
- Longer files consume more context and **reduce adherence**.
- If instructions are growing large, split them using `@imports` or `.claude/rules/` files.

### Community Consensus
- Ideally **50-100 lines** in root CLAUDE.md with `@imports` for detailed sections.
- Under **300 lines** as an absolute maximum.
- Fewer than **150-200 instructions total** across all loaded files (frontier thinking LLMs can follow approximately 150-200 instructions with reasonable consistency).
- Claude Code's system prompt already contains approximately 50 instructions, consuming roughly a third of reliable instruction capacity.

### Auto Memory Size Limit
- The first **200 lines** of `MEMORY.md` (auto memory) are loaded at the start of every conversation. Content beyond line 200 is not loaded.
- This limit applies only to MEMORY.md. CLAUDE.md files are loaded **in full** regardless of length, though shorter files produce better adherence.

### Skills Size Limit
- Keep `SKILL.md` under **500 lines**.
- The skill description budget scales dynamically at **2% of the context window**, with a fallback of **16,000 characters**.

---

## 5. What Should Go Into CLAUDE.md

Based on both official documentation and community best practices:

| Category | Examples |
|----------|----------|
| **Bash commands Claude cannot guess** | `npm run build:prod`, `make test-integration` |
| **Code style rules that differ from defaults** | "Use 2-space indentation", "Use ES modules not CommonJS" |
| **Testing instructions and preferred test runners** | "Run `pytest -x` for tests", "Prefer single test runs" |
| **Repository etiquette** | Branch naming conventions, PR conventions, commit message format |
| **Architectural decisions specific to the project** | "API handlers live in `src/api/handlers/`" |
| **Developer environment quirks** | Required env vars, special setup steps |
| **Common gotchas or non-obvious behaviors** | "The auth module uses a custom token format" |
| **Project context** | What the project is, tech stack, purpose |

### The WHAT/WHY/HOW Framework
A well-structured CLAUDE.md addresses three dimensions:
1. **WHAT**: Technology stack, project architecture, monorepo structure, codebase organization
2. **WHY**: Project purpose and functional intent of different components
3. **HOW**: Workflow instructions, tool usage, testing procedures, verification methods

---

## 6. What Should NOT Go Into CLAUDE.md

| Category | Reason |
|----------|--------|
| **Anything Claude can figure out by reading code** | Wastes tokens; Claude infers patterns well |
| **Standard language conventions Claude already knows** | Claude knows Python PEP 8, JS conventions, etc. |
| **Detailed API documentation** | Link to docs instead of embedding them |
| **Information that changes frequently** | Will become stale and misleading |
| **Long explanations or tutorials** | Move to separate files or skills |
| **File-by-file descriptions of the codebase** | Claude can read the code directly |
| **Self-evident practices** | "Write clean code" adds no value |
| **Code snippets** | They become out-of-date quickly; use `file:line` references instead |
| **Code style / linting rules** | Use deterministic tools (linters/formatters) instead |
| **Task-specific instructions** | Use Skills for on-demand context |

### Key Principle
For each line in CLAUDE.md, ask: **"Would removing this cause Claude to make mistakes?"** If not, delete it. Bloated CLAUDE.md files cause Claude to ignore actual instructions.

---

## 7. How CLAUDE.md Interacts with Skills

### Fundamental Distinction
- **CLAUDE.md** = always-loaded project context. Loaded at the start of every session. Every conversation starts with this context.
- **Skills** = on-demand task-specific workflows. Only load when invoked or when Claude determines they are relevant.

From the official documentation:
> "Rules load into context every session or when matching files are opened. For task-specific instructions that don't need to be in context all the time, use skills instead, which only load when you invoke them or when Claude determines they're relevant to your prompt."

### Loading Behavior Comparison

| Feature | CLAUDE.md | Skills |
|---------|-----------|--------|
| **When loaded** | Every session at startup | On-demand when relevant |
| **Token cost** | Always consumed | Only when activated |
| **Content type** | Project-wide standards | Task-specific workflows |
| **Who writes** | Developer/team | Developer/team |
| **Scope** | Project, user, or org | Project, personal, enterprise, or plugin |

### Skills Loading Priority
Skills have their own priority hierarchy:
1. **Enterprise** (managed settings) -- highest priority
2. **Personal** (`~/.claude/skills/<skill-name>/SKILL.md`)
3. **Project** (`.claude/skills/<skill-name>/SKILL.md`)
4. **Plugin** (uses `plugin-name:skill-name` namespace, no conflicts)

When skills share the same name across levels, **higher-priority locations win**: enterprise > personal > project. If a skill and a legacy command (`.claude/commands/`) share the same name, the **skill takes precedence**.

### Context Budget for Skills
At startup, **only skill descriptions** (name + description from frontmatter) are loaded into context. The full skill content only loads when the skill is invoked. The description budget scales dynamically at 2% of the context window, with a fallback of 16,000 characters.

### Invocation Control
Skills can be configured to control who invokes them:
- **Default**: Both user and Claude can invoke
- **`disable-model-invocation: true`**: Only the user can invoke (manual `/skill-name` only)
- **`user-invocable: false`**: Only Claude can invoke (background knowledge, hidden from `/` menu)

### CLAUDE.md and Skill Context
When a skill runs with `context: fork` (in a subagent), the subagent **also loads CLAUDE.md** alongside the skill content. This means CLAUDE.md instructions apply even within skill-driven subagent execution.

---

## 8. How to Reference Skills from CLAUDE.md

There is no special syntax to reference skills from within CLAUDE.md. Skills are discovered automatically based on:
1. Their location in `.claude/skills/` directories
2. Their `description` field in SKILL.md frontmatter
3. Pattern matching against user prompts

However, you can mention skills in CLAUDE.md for human readers:
```markdown
# Workflows
- For deployment, use the /deploy skill
- For code review, use the /review skill
```

The relationship is complementary: CLAUDE.md provides the always-on context (standards, conventions, architecture), while skills provide the on-demand capabilities (workflows, tasks, domain knowledge).

### When to Use Which

| Scenario | Use CLAUDE.md | Use Skills |
|----------|---------------|------------|
| Code style standards | Yes | No |
| Build/test commands | Yes | No |
| Architecture overview | Yes | No |
| Deployment workflow | No | Yes |
| Code review checklist | No | Yes |
| API conventions (always needed) | Yes | No |
| API conventions (sometimes needed) | No | Yes |
| Domain knowledge | No | Yes |
| Personal preferences | Yes (in ~/.claude/) | No |

---

## 9. The @import System

CLAUDE.md files can import additional files using `@path/to/import` syntax. Imported files are expanded and loaded into context at launch alongside the CLAUDE.md that references them.

### Syntax
```markdown
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
```

### Rules
- Both **relative and absolute paths** are allowed
- Relative paths resolve **relative to the file containing the import**, not the working directory
- Imported files can **recursively import** other files, with a **maximum depth of five hops**
- The first time Claude Code encounters external imports in a project, it shows an **approval dialog** listing the files

### For Private Preferences
Use `CLAUDE.local.md` for per-project preferences not checked into version control.

### Cross-Worktree Personal Instructions
If you work across multiple git worktrees, use a home-directory import:
```markdown
# Individual Preferences
- @~/.claude/my-project-instructions.md
```

### Important Distinction: @ vs No @
- Using `@` syntax: file is **always loaded at startup** (increases token usage)
- Referencing without `@`: Claude decides **when to load** the additional context (lighter startup)

---

## 10. Organizing Rules with .claude/rules/

For larger projects, the `.claude/rules/` directory provides modular, maintainable organization.

### Structure
```
your-project/
├── .claude/
│   ├── CLAUDE.md           # Main project instructions
│   └── rules/
│       ├── code-style.md   # Code style guidelines
│       ├── testing.md      # Testing conventions
│       └── security.md     # Security requirements
```

### Loading Behavior
- Rules **without** `paths` frontmatter are loaded at launch with the same priority as `.claude/CLAUDE.md`
- Rules **with** `paths` frontmatter only load when Claude is working with files matching the specified patterns

### Path-Specific Rules
```yaml
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules
- All API endpoints must include input validation
- Use the standard error response format
```

### Glob Patterns
| Pattern | Matches |
|---------|---------|
| `**/*.ts` | All TypeScript files in any directory |
| `src/**/*` | All files under `src/` directory |
| `*.md` | Markdown files in the project root |
| `src/components/*.tsx` | React components in a specific directory |

### Brace Expansion
```yaml
---
paths:
  - "src/**/*.{ts,tsx}"
  - "lib/**/*.ts"
  - "tests/**/*.test.ts"
---
```

### Symlinks
The `.claude/rules/` directory supports symlinks for sharing rules across projects:
```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

### User-Level Rules
Personal rules in `~/.claude/rules/` apply to every project on your machine. User-level rules are loaded **before** project rules, giving project rules higher priority.

---

## 11. Monorepo and Large Team Management

### claudeMdExcludes Setting
Skip specific CLAUDE.md files by path or glob pattern:
```json
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

- Patterns are matched against **absolute file paths** using glob syntax
- Can be configured at any settings layer: user, project, local, or managed policy
- **Arrays merge across layers**
- Managed policy CLAUDE.md files **cannot be excluded**

### Organization-Wide Deployment
Organizations can deploy a centrally managed CLAUDE.md at the managed policy location. This file cannot be excluded by individual settings. Deploy with MDM, Group Policy, Ansible, or similar tools.

### Monorepo Strategy
From community best practices:
- Root CLAUDE.md defines the **WHEN** (standards, overarching rules)
- Subtree CLAUDE.md files define the **HOW** (service-specific conventions)
- One author reduced documentation from 47,000 to 9,000 words (80% reduction) by distributing content across service-specific files

### Recommended Structure
```
monorepo/
├── CLAUDE.md                  # Root: common standards
├── frontend/
│   ├── CLAUDE.md              # Frontend-specific rules
│   └── .claude/rules/
│       └── components.md      # Component conventions
├── backend/
│   ├── CLAUDE.md              # Backend-specific rules
│   └── .claude/rules/
│       └── api-design.md      # API conventions
└── shared/
    └── CLAUDE.md              # Shared library conventions
```

---

## 12. Auto Memory System

Claude Code has two complementary memory systems, both loaded at the start of every conversation:

| Feature | CLAUDE.md files | Auto Memory |
|---------|-----------------|-------------|
| **Who writes it** | You | Claude |
| **What it contains** | Instructions and rules | Learnings and patterns |
| **Scope** | Project, user, or org | Per working tree |
| **Loaded into** | Every session | Every session (first 200 lines) |
| **Use for** | Coding standards, workflows, architecture | Build commands, debugging insights, preferences |

### Auto Memory Storage
Each project gets its own memory directory at `~/.claude/projects/<project>/memory/`. The `<project>` path is derived from the git repository, so all worktrees share one auto memory directory.

### Enable/Disable
Auto memory is on by default. Toggle via `/memory` command or settings:
```json
{
  "autoMemoryEnabled": false
}
```

### View and Edit
The `/memory` command lists all CLAUDE.md and rules files loaded in the current session, lets you toggle auto memory, and provides a link to open memory files.

---

## 13. Writing Effective Instructions

### Specificity
Write instructions that are concrete enough to verify:
- "Use 2-space indentation" instead of "Format code properly"
- "Run `npm test` before committing" instead of "Test your changes"
- "API handlers live in `src/api/handlers/`" instead of "Keep files organized"

### Consistency
If two rules contradict each other, Claude may pick one arbitrarily. Review CLAUDE.md files periodically to remove outdated or conflicting instructions.

### Emphasis
You can add emphasis (e.g., "IMPORTANT" or "YOU MUST") to improve adherence for critical instructions.

### Version Control
Check CLAUDE.md into git so your team can contribute. The file compounds in value over time.

### Treat It Like Code
Review CLAUDE.md when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts.

### Use Hooks for Guaranteed Actions
Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens. Use hooks for actions that must happen every time with zero exceptions.

---

## 14. Common Failure Patterns

From the official best practices documentation:

1. **The over-specified CLAUDE.md**: If CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise.
   - Fix: Ruthlessly prune. If Claude already does something correctly without the instruction, delete it or convert it to a hook.

2. **Correcting over and over**: Context gets polluted with failed approaches.
   - Fix: After two failed corrections, `/clear` and write a better initial prompt.

3. **Kitchen sink session**: Mixing unrelated tasks fills context with irrelevant information.
   - Fix: `/clear` between unrelated tasks.

4. **Trust-then-verify gap**: Claude produces plausible-looking code without handling edge cases.
   - Fix: Always provide verification (tests, scripts, screenshots).

5. **Infinite exploration**: Unscoped investigations fill context by reading hundreds of files.
   - Fix: Scope narrowly or use subagents.

### System Reminder Warning
Claude Code presents CLAUDE.md with a system reminder: "this context may or may not be relevant to your tasks. You should not respond to this context unless it is highly applicable." This means only universally applicable content receives consistent attention.

---

## 15. Skills Best Practices (Complementary)

### Skill Structure
```
my-skill/
├── SKILL.md           # Main instructions (required)
├── template.md        # Template for Claude to fill in
├── examples/
│   └── sample.md      # Example output
└── scripts/
    └── validate.sh    # Script Claude can execute
```

### Frontmatter Fields
```yaml
---
name: my-skill
description: What this skill does
disable-model-invocation: true
allowed-tools: Read, Grep
context: fork
agent: Explore
---
```

### Key Skill Authoring Principles
1. **Concise is key**: Only add context Claude does not already have
2. **Keep SKILL.md under 500 lines**: Move detailed reference to separate files
3. **Progressive disclosure**: SKILL.md serves as overview; Claude loads detail files on-demand
4. **Avoid deeply nested references**: Keep references one level deep from SKILL.md
5. **Test with all models**: What works for Opus may need more detail for Haiku
6. **Write descriptions in third person**: "Processes Excel files" not "I can help you process Excel files"
7. **Use consistent naming**: Lowercase letters, numbers, and hyphens only

### MCP Tool References in Skills
Use fully qualified tool names: `ServerName:tool_name`
```markdown
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
```

---

## 16. Sources

### Official Anthropic Documentation
- [How Claude remembers your project (Memory/CLAUDE.md)](https://code.claude.com/docs/en/memory) -- Primary official reference
- [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) -- Official best practices guide
- [Extend Claude with Skills](https://code.claude.com/docs/en/skills) -- Official skills documentation
- [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) -- Platform-level skill guidance

### Community Resources
- [Writing a Good CLAUDE.md (HumanLayer)](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [How to Write a Good CLAUDE.md File (Builder.io)](https://www.builder.io/blog/claude-md-guide)
- [How I Organized My CLAUDE.md in a Monorepo (DEV Community)](https://dev.to/anvodev/how-i-organized-my-claudemd-in-a-monorepo-with-too-many-contexts-37k7)
- [Claude Code Best Practices (GitHub - shanraisshan)](https://github.com/shanraisshan/claude-code-best-practice)
- [Claude Code Best Practices (Morph)](https://www.morphllm.com/claude-code-best-practices)
- [Claude Code Best Practices (SFEIR Institute)](https://institute.sfeir.com/en/claude-code/claude-code-resources/best-practices/)
- [CLAUDE.md Setup Guide (DEV Community)](https://dev.to/dembsky/claudemd-how-to-set-up-project-instructions-for-claude-code-5gna)
- [Using CLAUDE.MD Files (Claude Blog)](https://claude.com/blog/using-claude-md-files)
- [Trail of Bits Claude Code Config (GitHub)](https://github.com/trailofbits/claude-code-config)
