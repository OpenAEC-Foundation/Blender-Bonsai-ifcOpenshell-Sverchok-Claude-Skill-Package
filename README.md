<p align="center">
  <img src="docs/social-preview.png" alt="73 Deterministic Claude AI Skills for Blender, Bonsai, IfcOpenShell & Sverchok" width="100%">
</p>

<p align="center">
  <a href="#installation"><img src="https://img.shields.io/badge/Claude_Code-Ready-27ca40?style=for-the-badge" alt="Claude Code Ready"></a>
  <a href="#blender-mcp-live-connection-to-blender"><img src="https://img.shields.io/badge/Blender_MCP-Included-FF6B35?style=for-the-badge" alt="Blender MCP Included"></a>
  <a href="#version-compatibility"><img src="https://img.shields.io/badge/Blender-3.x_|_4.x_|_5.x-E87D0D?style=for-the-badge" alt="Blender Versions"></a>
  <a href="#version-compatibility"><img src="https://img.shields.io/badge/IFC-2X3_|_4_|_4X3-6B8FA3?style=for-the-badge" alt="IFC Schema Versions"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge" alt="MIT License"></a>
</p>

<p align="center">
  <strong>73 deterministic skills + Blender MCP integration</strong> enabling Claude AI to generate and execute flawless Blender/BIM/IFC/Sverchok code.<br>
  Built with the <a href="https://github.com/OpenAEC-Foundation/Open-Agents">Open-Agents</a> multi-agent orchestration framework.
</p>

---

## Blender MCP: Live Connection to Blender

This package includes a pre-configured `.mcp.json` that connects Claude Code directly to a running Blender instance via the [Blender MCP](https://github.com/ahujasid/blender-mcp) server. This means Claude can read your scene, execute Python code in Blender, and take viewport screenshots in real time.

```
Claude Code ──MCP Protocol──> blender-mcp server ──TCP:9876──> Blender Addon ──bpy──> Blender Scene
```

| MCP Tool | What it does |
|----------|-------------|
| `get_scene_info` | Read current scene state (objects, materials, settings) |
| `execute_blender_code` | Run Python code directly in Blender |
| `get_viewport_screenshot` | Capture the 3D viewport as an image |
| `get_object_info` | Get details of a specific object |

### Quick start (if you already have the prerequisites)

1. Install the [Blender MCP addon](https://github.com/ahujasid/blender-mcp) in Blender (Edit > Preferences > Add-ons > Install from Disk)
2. In Blender, open the sidebar (`N`), go to the **BlenderMCP** tab, click **"Connect to Claude"**
3. Start Claude Code in this workspace. The `.mcp.json` is already configured.

### Full setup guide

See **[SETUP.md](SETUP.md)** for complete step-by-step installation instructions covering:
- Node.js, Claude Code CLI, uv package manager
- Blender MCP addon installation (the bridge between Claude and Blender)
- Bonsai/IfcOpenShell/Sverchok addon installation
- MCP configuration and troubleshooting

---

## Why This Exists

Claude is powerful, but without domain-specific guidance it generates BIM/IFC code that *looks* correct but **fails in production**.

**The #1 cause of AI-generated IFC failures:**

```python
# WRONG - This creates an orphan entity with no relationships
wall = ifc_file.create_entity("IfcWall")

# CORRECT - API ensures valid ownership, relationships, and GlobalId
wall = ifcopenshell.api.run("root.create_entity", ifc_file,
    ifc_class="IfcWall", name="Wall_001")
```

**This skill package solves it** by giving Claude exact API syntax, decision trees, error diagnostics, and version-aware patterns for every operation.

---

## Skill Packages

| Package | Skills | Syntax | Impl | Errors | Core | Agents |
|---------|:------:|:------:|:----:|:------:|:----:|:------:|
| [Blender](skills/blender/) | **26** | 11 | 6 | 3 | 4 | 2 |
| [IfcOpenShell](skills/ifcopenshell/) | **19** | 4 | 9 | 3 | 2 | 1 |
| [Bonsai](skills/bonsai/) | **14** | 4 | 7 | 1 | 1 | 1 |
| [Sverchok](skills/sverchok/) | **12** | 4 | 5 | 1 | 1 | 1 |
| [Cross-Tech](skills/aec-cross-tech/) | **2** | — | — | — | 1 | 1 |
| **Total** | **73** | **23** | **27** | **8** | **9** | **6** |

Each package is **standalone** — install only the technologies you work with.

See [INDEX.md](INDEX.md) for the full skill catalog with descriptions.

### Skill Categories

| Category | Purpose | Example |
|----------|---------|---------|
| `syntax/` | API syntax, code patterns, method signatures | `blender-syntax-operators` |
| `impl/` | Step-by-step development workflows, decision trees | `bonsai-impl-modeling` |
| `errors/` | Error handling, diagnostics, anti-patterns | `ifcos-errors-schema` |
| `core/` | Cross-cutting: API overview, version matrix, concepts | `blender-core-api` |
| `agents/` | Intelligent orchestration, validation | `aec-agents-workflow-orchestrator` |

---

## Installation

> **First time?** Follow the complete [SETUP.md](SETUP.md) guide. It covers everything from installing Node.js to verifying the Blender MCP connection.

### Claude Code (CLI)

**Option 1 — Copy skills + MCP config into your project:**
```bash
# Clone the repository
git clone https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package.git

# Copy skills into your project
cp -r Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/skills/blender/ your-project/.claude/skills/blender/
cp -r Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/skills/ifcopenshell/ your-project/.claude/skills/ifcopenshell/

# Copy the Blender MCP configuration
cp Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/.mcp.json your-project/.mcp.json
```

**Option 2 — Install via [OpenAEC Workspace Composer](https://github.com/OpenAEC-Foundation/OpenAEC-Workspace-Composer):**

The Workspace Composer sets up a complete workspace with skills and MCP configuration automatically. Select the "Blender/Bonsai" package and the composer handles the rest.

**Option 3 — Install per-package (use only what you need):**
```bash
# Only Blender skills
cp -r skills/blender/ your-project/.claude/skills/blender/

# Only IfcOpenShell skills
cp -r skills/ifcopenshell/ your-project/.claude/skills/ifcopenshell/

# Only Bonsai skills
cp -r skills/bonsai/ your-project/.claude/skills/bonsai/

# Only Sverchok skills
cp -r skills/sverchok/ your-project/.claude/skills/sverchok/

# Cross-technology workflows
cp -r skills/aec-cross-tech/ your-project/.claude/skills/aec-cross-tech/

# Always include the MCP config for Blender connectivity
cp .mcp.json your-project/.mcp.json
```

---

## Version Compatibility

| Technology | Supported Versions |
|------------|--------------------|
| Blender | 3.x, 4.x, 5.x |
| IfcOpenShell | Latest (IFC2X3, IFC4, IFC4X3 schemas) |
| Bonsai | v0.8 (formerly BlenderBIM) |
| Sverchok | Current |

---

## Methodology

Built using the **7-phase research-first methodology** proven in the [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package):

> **Core principle**: Research first, then build. Never create skills based on assumptions.

Skills were created in parallel via [Open-Agents](https://github.com/OpenAEC-Foundation/Open-Agents) multi-agent orchestration, with each skill validated against official documentation and real-world usage.

| Phase | Description | Status |
|:-----:|-------------|:------:|
| 1 | Raw Masterplan | Complete |
| 2 | Deep Research | Complete |
| 3 | Masterplan Refinement | Complete |
| 4 | Topic-Specific Research | Complete |
| 5 | Skill Creation | Complete |
| 6 | Validation | Complete |
| 7 | Publication | Complete |

---

## Documentation

| Document | Purpose |
|----------|---------|
| [SETUP.md](SETUP.md) | Complete setup guide (Blender MCP, addons, Claude Code) |
| [INDEX.md](INDEX.md) | Full skill catalog with descriptions |
| [ROADMAP.md](ROADMAP.md) | Project status (single source of truth) |
| [REQUIREMENTS.md](REQUIREMENTS.md) | What skills must achieve, quality guarantees |
| [DECISIONS.md](DECISIONS.md) | Architectural decisions with rationale |
| [SOURCES.md](SOURCES.md) | Official documentation and reference materials |
| [WAY_OF_WORK.md](WAY_OF_WORK.md) | 7-phase development methodology |
| [LESSONS.md](LESSONS.md) | Lessons learned during development |
| [CHANGELOG.md](CHANGELOG.md) | Version history |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute |

## Related Projects

| Project | Role |
|---------|------|
| [OpenAEC Workspace Composer](https://github.com/OpenAEC-Foundation/OpenAEC-Workspace-Composer) | One-click workspace setup with skills + MCP |
| [Blender MCP](https://github.com/ahujasid/blender-mcp) | The MCP server that bridges Claude to Blender |
| [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) | Proven methodology template |
| [Open-Agents](https://github.com/OpenAEC-Foundation/Open-Agents) | Multi-agent orchestration tooling |
| [Impertio AI Ecosystem](https://github.com/OpenAEC-Foundation/Impertio-AI-Ecosystem-Deployment) | General AI workspace lessons |

---

## Companion Skills: Cross-Technology Integration

This package covers Blender, Bonsai, IfcOpenShell, and Sverchok as individual technologies. When you need to **bridge** these tools with other AEC software, install the companion cross-technology package:

> **[Cross-Tech AEC Integration Skills](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package)** — 15 skills for technology boundaries

Relevant cross-tech skills for Blender/Bonsai users:

| Skill | Boundary | What it adds |
|-------|----------|-------------|
| `crosstech-impl-speckle-blender` | Speckle ↔ Blender | Send/receive BIM data via Speckle, material mapping, Bonsai/IFC through Speckle |
| `crosstech-impl-freecad-ifc-bridge` | FreeCAD ↔ IFC | Round-trip IFC editing between FreeCAD and IfcOpenShell |
| `crosstech-core-ifc-schema-bridge` | IFC ↔ All formats | How IFC entities map to Blender objects, Three.js, web-ifc, FreeCAD, Speckle |
| `crosstech-core-coordinate-systems` | BIM ↔ GIS | CRS transformations, axis conventions (Blender Z-up vs Three.js Y-up) |
| `crosstech-errors-conversion` | Any ↔ Any | Diagnosing IFC conversion errors across all tools |

## License

MIT License — see [LICENSE](LICENSE).

---

*Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.*
