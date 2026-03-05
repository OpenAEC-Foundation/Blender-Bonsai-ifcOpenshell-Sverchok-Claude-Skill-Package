# SOURCES

Official documentation, repositories, and reference materials used in this project. All skills MUST be verified against these sources. NEVER use unverified blog posts or outdated community content.

---

## Blender

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Blender Python API | https://docs.blender.org/api/current/ | Complete bpy reference |
| Blender Manual | https://docs.blender.org/manual/en/latest/ | Concepts and workflows |
| Blender Extensions Platform | https://extensions.blender.org/ | 4.x extension system |
| Blender Developer Docs | https://developer.blender.org/ | Internal architecture, release notes |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| Blender Source (official) | https://projects.blender.org/blender/blender | Core source code (primary) |
| Blender Source (GitHub mirror) | https://github.com/blender/blender | GitHub mirror for code search |
| Blender Addons | https://projects.blender.org/blender/blender-addons | Official addon examples |

### Version-Specific Release Notes
| Source | URL | Purpose |
|--------|-----|---------|
| Release Notes Index | https://developer.blender.org/docs/release_notes/ | All version release notes |
| 4.0 Python API Changes | https://developer.blender.org/docs/release_notes/4.0/python_api/ | 4.0 breaking changes |
| 4.1 Python API Changes | https://developer.blender.org/docs/release_notes/4.1/python_api/ | 4.1 changes |
| 4.2 Python API Changes | https://developer.blender.org/docs/release_notes/4.2/python_api/ | 4.2 LTS extension system |
| 4.3 Python API Changes | https://developer.blender.org/docs/release_notes/4.3/python_api/ | 4.3 changes |
| 5.0 Python API Changes | https://developer.blender.org/docs/release_notes/5.0/python_api/ | 5.0 breaking changes (BGL removal) |
| Python API Changelog | https://docs.blender.org/api/current/change_log.html | Cumulative API changelog |

---

## Bonsai (formerly BlenderBIM)

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Bonsai Documentation | https://docs.bonsaibim.org/ | User and developer docs |
| BlenderBIM (legacy) | https://blenderbim.org/ | Historical reference only |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| Bonsai in IfcOpenShell repo | https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai | Bonsai source (inside IfcOpenShell) |
| Bonsai Core | (same repo)/src/bonsai/bonsai/core/ | Core module architecture |
| Bonsai Tool | (same repo)/src/bonsai/bonsai/tool/ | Tool implementations |
| Bonsai UI | (same repo)/src/bonsai/bonsai/bim/module/ | UI panels and operators |

---

## IfcOpenShell

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| IfcOpenShell Website | https://ifcopenshell.org/ | Main documentation and guides |
| IfcOpenShell Academy | https://academy.ifcopenshell.org/ | Tutorials and learning paths |
| Python API Reference | https://ifcopenshell.github.io/ifcopenshell-python/ | Auto-generated API docs |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| IfcOpenShell GitHub | https://github.com/IfcOpenShell/IfcOpenShell | Complete source (mono-repo) |
| IfcOpenShell GitHub Org | https://github.com/orgs/IfcOpenShell/repositories | All org repositories |
| ifcopenshell.api | (same repo)/src/ifcopenshell-python/ifcopenshell/api/ | High-level API modules |
| ifcopenshell.util | (same repo)/src/ifcopenshell-python/ifcopenshell/util/ | Utility modules |
| ifcopenshell.geom | (same repo)/src/ifcopenshell-python/ifcopenshell/geom/ | Geometry processing |

---

## IFC Standard

### Official Sources
| Source | URL | Purpose |
|--------|-----|---------|
| buildingSMART | https://www.buildingsmart.org/ | IFC standard owner |
| IFC Specifications | https://standards.buildingsmart.org/IFC/ | Schema specifications |
| IFC4.3 Documentation | https://ifc43-docs.standards.buildingsmart.org/ | Latest IFC standard docs |
| bSDD (Data Dictionary) | https://search.bsdd.buildingsmart.org/ | Classification and property lookup |

### Schema Files
| Source | URL | Purpose |
|--------|-----|---------|
| IFC2x3 Schema | https://standards.buildingsmart.org/IFC/RELEASE/IFC2x3/ | Legacy schema |
| IFC4 Schema | https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ | Current standard |
| IFC4.3 Schema | https://standards.buildingsmart.org/IFC/RELEASE/IFC4_3/ | Latest standard |

---

## Sverchok

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Sverchok Docs (GitHub Pages) | https://nortikin.github.io/sverchok/docs/main.html | Primary documentation |
| Sverchok Docs (ReadTheDocs) | https://sverchok.readthedocs.io/ | Alternative documentation |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| Sverchok GitHub | https://github.com/nortikin/sverchok | Source code and node implementations |

---

## Claude / Anthropic (Skill Development Platform)

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Claude Platform Docs | https://platform.claude.com/docs/en/home | Claude platform reference |
| Agent SDK / Skills Docs | https://platform.claude.com/docs/en/agent-sdk/skills | Skill format specification |
| Build with Claude | https://www.anthropic.com/learn/build-with-claude | Development guides and patterns |

---

## OpenAEC Foundation Projects

### This Project
| Source | URL | Purpose |
|--------|-----|---------|
| This Skill Package | https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package | This repository |
| OpenAEC Foundation (org) | https://github.com/orgs/OpenAEC-Foundation/repositories | All foundation repositories |

### Methodology & Tooling
| Source | URL | Purpose |
|--------|-----|---------|
| ERPNext Skill Package | https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package | Proven methodology template (28 skills) |
| Impertio AI Ecosystem | https://github.com/OpenAEC-Foundation/Impertio-AI-Ecosystem-Deployment | AI deployment patterns and lessons |
| Open-Agents | https://github.com/OpenAEC-Foundation/Open-Agents | Multi-agent orchestration tooling |

### Related AEC Projects
| Source | URL | Purpose |
|--------|-----|---------|
| building.py | https://github.com/OpenAEC-Foundation/building-py | Python library for buildings → Blender/IFC export |
| GIS-to-Blender Automation | https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation | LLM-driven Blender 3D environment generation |
| AEC Scripts | https://github.com/OpenAEC-Foundation/aec-scripts | Python scripts library for AEC sector |
| Monty IFC Viewer | https://github.com/OpenAEC-Foundation/monty-ifc-viewer | IFC model viewer |
| INB Template | https://github.com/OpenAEC-Foundation/inb-template | IFC template for Dutch construction |
| Nextcloud Check-in/out | https://github.com/OpenAEC-Foundation/Nextcloud-Check-in-Check-out-feature | File locking for shared Claude Code projects |

---

## Source Verification Rules

1. **Primary sources ONLY**: Official docs > source code > official tutorials
2. **NEVER trust**: Random blog posts, outdated StackOverflow, AI-generated content without verification
3. **Version-check**: Ensure source matches target version (Blender 3.x vs 4.x, IFC2x3 vs IFC4)
4. **Date-check**: Note when source was last verified
5. **Cross-reference**: If official docs are sparse (e.g., Bonsai), verify against source code
6. **GitHub org pages**: Use org repo listings to discover related tools and sub-projects

## Last Verified
| Technology | Date | By | Notes |
|------------|------|----|-------|
| Blender | 2026-03-05 | Research agent | Vooronderzoek complete, API docs + release notes verified |
| Bonsai | - | Not yet verified | |
| IfcOpenShell | - | Not yet verified | |
| Sverchok | - | Not yet verified | |
| IFC Standard | - | Not yet verified | |
| Claude/Anthropic | - | Not yet verified | |
