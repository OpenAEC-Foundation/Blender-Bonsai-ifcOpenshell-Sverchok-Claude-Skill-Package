# Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package

> Deterministic Claude Skills for AEC (Architecture, Engineering, Construction) - enabling Claude to write correct, version-aware code for Blender, Bonsai, IfcOpenShell, and Sverchok.

---

## Technology Packages

Each technology is a **standalone installable package**. Use only what you need.

| Package | Technology | Scope | Status |
|---------|-----------|-------|--------|
| `skills/blender/` | Blender 3.x & 4.x | Python API, operators, addons, mesh, modifiers | In Development |
| `skills/bonsai/` | Bonsai (ex-BlenderBIM) | BIM modeling, IFC editing, spatial structure | In Development |
| `skills/ifcopenshell/` | IfcOpenShell | IFC file I/O, API, geometry, schemas (IFC2x3/IFC4/IFC4.3) | In Development |
| `skills/sverchok/` | Sverchok | Visual programming, parametric design | Planned |
| `skills/aec-cross-tech/` | Cross-technology | IFC fundamentals, BIM workflows, auto-detection | In Development |

## Skill Categories (per package)

| Category | Purpose | Example |
|----------|---------|---------|
| `syntax/` | API syntax, code patterns, method signatures | `blender-syntax-operators` |
| `impl/` | Step-by-step development workflows, decision trees | `bonsai-impl-modeling` |
| `errors/` | Error handling, diagnostics, anti-patterns | `ifcos-errors-schema` |
| `core/` | Cross-cutting: API overview, version matrix, concepts | `blender-core-api` |
| `agents/` | Intelligent orchestration, validation | `aec-workflow-orchestrator` |

## Current Progress

**Phase 1: Setup + Raw Masterplan** - Complete

See [ROADMAP.md](ROADMAP.md) for detailed status.

## Documentation

| Document | Purpose |
|----------|---------|
| [ROADMAP.md](ROADMAP.md) | Project status (single source of truth) |
| [REQUIREMENTS.md](REQUIREMENTS.md) | What skills must achieve, quality guarantees |
| [DECISIONS.md](DECISIONS.md) | Architectural decisions with rationale |
| [SOURCES.md](SOURCES.md) | Official documentation and reference materials |
| [WAY_OF_WORK.md](WAY_OF_WORK.md) | 7-phase development methodology |
| [LESSONS.md](LESSONS.md) | Lessons learned during development |
| [CHANGELOG.md](CHANGELOG.md) | Version history |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute |
| [SECURITY.md](SECURITY.md) | Security policy |

## Methodology

Built using the **7-phase research-first methodology** proven in the [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package):

1. Raw Masterplan
2. Deep Research (vooronderzoek per technology)
3. Masterplan Refinement (based on research findings)
4. Topic-Specific Research (per skill)
5. Skill Creation (parallel via [Open-Agents](https://github.com/OpenAEC-Foundation/Open-Agents))
6. Validation
7. Publication

**Core principle**: Research first, then build. Never create skills based on assumptions.

## Installation

> Coming soon - skills are currently in development.

Skills will be installable per technology package for:
- **Claude Code (CLI)** - Copy skill folders to `~/.claude/skills/`
- **Claude.ai Web** - Upload via Settings > Capabilities > Skills
- **Claude Desktop** - Integrated skill loading

## Related Projects

| Project | Role |
|---------|------|
| [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) | Proven methodology template |
| [Open-Agents](https://github.com/OpenAEC-Foundation/Open-Agents) | Multi-agent orchestration tooling |
| [Impertio AI Ecosystem](https://github.com/OpenAEC-Foundation/Impertio-AI-Ecosystem-Deployment) | General AI workspace lessons |

## License

MIT License - see [LICENSE](LICENSE).

---

*Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.*
