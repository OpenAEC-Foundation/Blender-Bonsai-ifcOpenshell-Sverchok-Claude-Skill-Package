# SOURCES

Official documentation, repositories, and reference materials used in this project. All skills MUST be verified against these sources. NEVER use unverified blog posts or outdated community content.

---

## Blender

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Blender Python API | https://docs.blender.org/api/current/ | Complete bpy reference (currently Blender 5.0) |
| Blender Manual | https://docs.blender.org/manual/en/latest/ | Concepts and workflows |
| Blender Extensions Platform | https://extensions.blender.org/ | 4.2+ extension system (replaces bundled addons) |
| Blender Developer Docs | https://developer.blender.org/ | Internal architecture, release notes, build guides |
| Compatibility Index | https://developer.blender.org/docs/release_notes/compatibility/ | All major breaking changes across versions (through 6.0) |

### Extension Development
| Source | URL | Purpose |
|--------|-----|---------|
| Extensions Developer Handbook | https://developer.blender.org/docs/handbook/extensions/ | Official extension development guide |
| Add-on Development Setup | https://developer.blender.org/docs/handbook/extensions/addon_dev_setup/ | Setting up addon development environment |
| Add-on Guidelines | https://developer.blender.org/docs/handbook/extensions/addon_guidelines/ | Standards and best practices for extensions |
| Extension Hosting | https://developer.blender.org/docs/handbook/extensions/hosted/ | Publishing extensions on blender.org |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| Blender Source (official) | https://projects.blender.org/blender/blender | Core source code (primary) |
| Blender Source (GitHub mirror) | https://github.com/blender/blender | GitHub mirror for code search |
| Blender Addons (legacy) | https://projects.blender.org/blender/blender-addons | Legacy addon examples (through 4.1 only, archived) |
| Official Extensions Source | https://projects.blender.org/extensions | Extensions maintained by Blender developers (4.2+) |

### Developer Community
| Source | URL | Purpose |
|--------|-----|---------|
| Blender Developer Forum | https://devtalk.blender.org/ | Official developer discussion, Python API Q&A |
| Blender Developers Blog | https://code.blender.org/ | Official development blog |

### Version-Specific Release Notes
| Source | URL | Purpose |
|--------|-----|---------|
| Release Notes Index | https://developer.blender.org/docs/release_notes/ | All version release notes |
| 4.0 Python API Changes | https://developer.blender.org/docs/release_notes/4.0/python_api/ | 4.0 breaking changes |
| 4.1 Python API Changes | https://developer.blender.org/docs/release_notes/4.1/python_api/ | 4.1 changes |
| 4.2 Python API Changes | https://developer.blender.org/docs/release_notes/4.2/python_api/ | 4.2 LTS extension system |
| 4.3 Python API Changes | https://developer.blender.org/docs/release_notes/4.3/python_api/ | 4.3 Grease Pencil API rewrite |
| 4.4 Python API Changes | https://developer.blender.org/docs/release_notes/4.4/python_api/ | 4.4 changes (March 2025) |
| 4.5 Python API Changes | https://developer.blender.org/docs/release_notes/4.5/python_api/ | 4.5 LTS GPUShader deprecation (July 2025) |
| 5.0 Python API Changes | https://developer.blender.org/docs/release_notes/5.0/python_api/ | 5.0 breaking changes (BGL removal) |
| 5.1 Python API Changes | https://developer.blender.org/docs/release_notes/5.1/python_api/ | 5.1 Python 3.13 upgrade (beta, March 2026) |
| 5.2 Python API Changes | https://developer.blender.org/docs/release_notes/5.2/python_api/ | 5.2 LTS (alpha, July 2026) |
| Python API Changelog | https://docs.blender.org/api/current/change_log.html | Cumulative API changelog |

---

## Bonsai (formerly BlenderBIM)

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Bonsai Documentation | https://docs.bonsaibim.org/ | User and developer docs (v0.8.4 stable) |
| Bonsai Website | https://bonsaibim.org/ | Main website (replaces blenderbim.org) |
| Bonsai on Blender Extensions | https://extensions.blender.org/add-ons/bonsai/ | Official distribution (Blender 4.2+) |
| Bonsai Docs (IfcOpenShell hub) | https://docs.ifcopenshell.org/bonsai.html | Bonsai docs in unified IfcOpenShell documentation |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| Bonsai in IfcOpenShell repo | https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai | Bonsai source (v0.8.0 default branch; stable is 0.8.4) |
| Bonsai Core | (same repo)/src/bonsai/bonsai/core/ | Core module architecture |
| Bonsai Tool | (same repo)/src/bonsai/bonsai/tool/ | Tool implementations |
| Bonsai UI | (same repo)/src/bonsai/bonsai/bim/module/ | UI panels and operators |

### Community
| Source | URL | Purpose |
|--------|-----|---------|
| OSArch Forum (Bonsai) | https://community.osarch.org/discussions/tagged/bonsai-bim/p1 | Community discussions and Q&A |
| OSArch Wiki (Bonsai) | https://wiki.osarch.org/index.php/BlenderBIM_Add-on | Community wiki page |

---

## IfcOpenShell

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| IfcOpenShell Website | https://ifcopenshell.org/ | Main website (v0.8.4 current) |
| IfcOpenShell Documentation Hub | https://docs.ifcopenshell.org/ | Unified docs for all tools (Python API, C++ API, utilities) |
| Python API Reference | https://docs.ifcopenshell.org/ifcopenshell-python.html | Auto-generated API docs (replaces old GitHub Pages URL) |
| IfcOpenShell Academy | https://academy.ifcopenshell.org/ | Tutorials (archived, last content Oct 2021) |
| IfcOpenShell on PyPI | https://pypi.org/project/ifcopenshell/ | Python package (v0.8.4, supports Python 3.9-3.14) |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| IfcOpenShell GitHub | https://github.com/IfcOpenShell/IfcOpenShell | Complete source (mono-repo) |
| IfcOpenShell GitHub Org | https://github.com/orgs/IfcOpenShell/repositories | All org repositories (32 repos) |
| ifcopenshell.api | (same repo)/src/ifcopenshell-python/ifcopenshell/api/ | High-level API modules |
| ifcopenshell.util | (same repo)/src/ifcopenshell-python/ifcopenshell/util/ | Utility modules |
| ifcopenshell.geom | (same repo)/src/ifcopenshell-python/ifcopenshell/geom/ | Geometry processing |

### Tool Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| IfcTester Documentation | https://docs.ifcopenshell.org/ifctester.html | IDS authoring and IFC validation |
| IfcDiff Documentation | https://docs.ifcopenshell.org/ifcdiff.html | Comparing changes between IFC models |
| IfcPatch Documentation | https://docs.ifcopenshell.org/ifcpatch.html | Predetermined modifications on IFC files |

---

## IFC Standard

### Official Sources
| Source | URL | Purpose |
|--------|-----|---------|
| buildingSMART | https://www.buildingsmart.org/ | IFC standard owner |
| buildingSMART Technical Portal | https://technical.buildingsmart.org/standards/ | Central standards overview (IFC, IDS, BCF, bSDD) |
| IFC Specifications | https://standards.buildingsmart.org/IFC/ | Schema specifications |
| IFC4.3 Documentation | https://ifc43-docs.standards.buildingsmart.org/ | IFC 4.3.2.0 standard docs (current ISO standard) |
| bSDD (Data Dictionary) | https://search.bsdd.buildingsmart.org/ | Classification and property lookup |
| IDS Standard | https://www.buildingsmart.org/standards/bsi-standards/information-delivery-specification-ids/ | Information Delivery Specification v1.0 (approved) |

### Schema Files
| Source | URL | Purpose |
|--------|-----|---------|
| IFC2x3 Schema | https://standards.buildingsmart.org/IFC/RELEASE/IFC2x3/ | Legacy schema |
| IFC4 Schema | https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ | Current standard |
| IFC4.3 Schema | https://standards.buildingsmart.org/IFC/RELEASE/IFC4_3/ | Latest standard (IFC4X3_ADD2) |

### IFC5 Development (Alpha)
| Source | URL | Purpose |
|--------|-----|---------|
| IFC5 Development Repository | https://github.com/buildingSMART/IFC5-development | Next-gen IFC standard (Entity Component System) |
| IFC5 Learning Site | https://ifc5.technical.buildingsmart.org/ | IFC5 educational resources and viewer |

### buildingSMART Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| buildingSMART GitHub | https://github.com/buildingSMART | Official repos (IFC schemas, bSDD, IDS, BCF) |
| buildingSMART Community GitHub | https://github.com/buildingsmart-community | Community-driven projects and tools |

---

## Sverchok

### Official Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Sverchok Docs (GitHub Pages) | https://nortikin.github.io/sverchok/docs/main.html | Primary documentation (v1.4.0) |
| Sverchok Project Site | https://nortikin.github.io/sverchok/ | Project homepage with links to docs and downloads |

### Source Code
| Source | URL | Purpose |
|--------|-----|---------|
| Sverchok GitHub | https://github.com/nortikin/sverchok | Source code and node implementations (tested with Blender 5.1) |
| Sverchok Releases | https://github.com/nortikin/sverchok/releases | Release history (latest: v1.3.0, Sep 2024) |

### IfcSverchok (IFC/BIM Nodes for Sverchok)
| Source | URL | Purpose |
|--------|-----|---------|
| IfcSverchok Source | https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok | IFC nodes source (inside IfcOpenShell mono-repo) |
| IfcSverchok Releases | https://github.com/IfcOpenShell/IfcOpenShell/releases?q=ifcsverchok | IfcSverchok release builds |
| GSoC 2022 Proposal | https://github.com/opencax/GSoC/issues/43 | Visual programming nodes for BIM data |
| OSArch Discussion | https://community.osarch.org/discussion/284/sverchok-ifc | Community discussion and examples |

### Sverchok Extensions
| Source | URL | Purpose |
|--------|-----|---------|
| TopologicSverchok | https://github.com/wassimj/TopologicSverchok | Non-manifold topology for architecture |
| Sverchok-Extra | https://github.com/portnov/sverchok-extra | Additional NURBS/advanced geometry nodes |
| Sverchok Extensions Docs | https://nortikin.github.io/sverchok/docs/introduction/sverchok_extensions.html | Extension system documentation |

---

## Claude / Anthropic (Skill Development Platform)

### Claude Platform Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Claude Platform Docs | https://platform.claude.com/docs/en/home | Claude platform reference (canonical URL) |
| Agent SDK / Skills Docs | https://platform.claude.com/docs/en/agent-sdk/skills | Skill format specification |
| Build with Claude (Anthropic Academy) | https://www.anthropic.com/learn/build-with-claude | Development guides and patterns |
| Claude API Release Notes | https://platform.claude.com/docs/en/release-notes/overview | Platform release notes |

### Claude Code Documentation
| Source | URL | Purpose |
|--------|-----|---------|
| Claude Code Overview | https://code.claude.com/docs/en/overview | Main Claude Code documentation |
| Claude Code Skills Guide | https://code.claude.com/docs/en/skills | Creating/managing Skills in Claude Code |
| Claude Code Changelog | https://code.claude.com/docs/en/changelog | Claude Code version history |

### Agent Skills Standard
| Source | URL | Purpose |
|--------|-----|---------|
| Agent Skills Open Standard | https://agentskills.io | Open standard for Agent Skills (adopted by 30+ tools) |
| Agent Skills Specification | https://github.com/agentskills/agentskills | Specification and reference library |
| Anthropic Skills Repository | https://github.com/anthropics/skills | Official example skills and spec |
| Agent Skills Overview (API) | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview | Agent Skills architecture |
| Agent Skills Best Practices | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices | Authoring guide for effective skills |
| Agent Skills Quickstart | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart | Tutorial for pre-built Agent Skills |

### Agent SDK
| Source | URL | Purpose |
|--------|-----|---------|
| Agent SDK Overview | https://platform.claude.com/docs/en/agent-sdk/overview | Agent SDK documentation |
| Agent SDK Quickstart | https://platform.claude.com/docs/en/agent-sdk/quickstart | Getting started with Agent SDK |
| Agent SDK TypeScript Reference | https://platform.claude.com/docs/en/agent-sdk/typescript | TypeScript SDK API reference |
| Agent SDK Python Reference | https://platform.claude.com/docs/en/agent-sdk/python | Python SDK API reference |

---

## OpenAEC Foundation Projects

### This Project
| Source | URL | Purpose |
|--------|-----|---------|
| This Skill Package | https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package | This repository (may be private/renamed - verify) |
| OpenAEC Foundation (org) | https://github.com/orgs/OpenAEC-Foundation/repositories | All foundation repositories (35 repos) |
| OpenAEC Foundation Website | https://openaec-foundation.github.io/website/ | Official OpenAEC Foundation website |

### Methodology & Tooling
| Source | URL | Purpose |
|--------|-----|---------|
| ERPNext Skill Package | https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package | Proven methodology template (28 skills) |
| Open-Agents | https://github.com/OpenAEC-Foundation/Open-Agents | Multi-agent orchestration tooling |

### Related AEC Projects
| Source | URL | Purpose |
|--------|-----|---------|
| building.py | https://github.com/OpenAEC-Foundation/building-py | Python library for buildings → Blender/IFC export (30 stars) |
| Monty IFC Viewer | https://github.com/OpenAEC-Foundation/monty-ifc-viewer | Web-based IFC viewer (Three.js, v1.0.1) |
| INB Template | https://github.com/OpenAEC-Foundation/inb-template | IFC template for Dutch construction (IFC4x3) |
| openaec-reports | https://github.com/OpenAEC-Foundation/openaec-reports | Professional engineering report generation (PDF) |
| Open PDF Studio | https://github.com/OpenAEC-Foundation/open-pdf-studio | Open-source PDF editor/annotator (Tauri 2) |
| warmteverliesberekening | https://github.com/OpenAEC-Foundation/warmteverliesberekening | Heat loss calculations per ISSO 51:2023 (Rust) |
| dynlex | https://github.com/OpenAEC-Foundation/dynlex | Natural-language programming language compiler (LLVM) |

---

## Source Verification Rules

1. **Primary sources ONLY**: Official docs > source code > official tutorials
2. **NEVER trust**: Random blog posts, outdated StackOverflow, AI-generated content without verification
3. **Version-check**: Ensure source matches target version (Blender 3.x vs 4.x vs 5.x, IFC2x3 vs IFC4 vs IFC4.3)
4. **Date-check**: Note when source was last verified
5. **Cross-reference**: If official docs are sparse (e.g., Bonsai), verify against source code
6. **GitHub org pages**: Use org repo listings to discover related tools and sub-projects
7. **Agent Skills standard**: All skills should conform to the Agent Skills open standard (agentskills.io)

## Last Verified
| Technology | Date | By | Notes |
|------------|------|----|-------|
| Blender | 2026-03-05 | e-blender-sources | All URLs OK. Added 4.4/4.5/5.1/5.2 release notes, compatibility index, extension dev docs |
| Bonsai | 2026-03-05 | e-bonsai-ifcos-sources | blenderbim.org→bonsaibim.org redirect. v0.8.4 stable. Added Extensions listing, OSArch community |
| IfcOpenShell | 2026-03-05 | e-bonsai-ifcos-sources | FIXED: Python API docs URL (moved to docs.ifcopenshell.org). v0.8.4 stable. Academy archived |
| IFC Standard | 2026-03-05 | e-bonsai-ifcos-sources | IFC 4.3.2.0 current. IFC5 alpha in development. IDS v1.0 approved. Added buildingSMART GitHub |
| Sverchok | 2026-03-05 | e-sverchok-claude-openaec-sources | ReadTheDocs removed (stale v0.5). GitHub Pages is canonical (v1.4.0). Tested with Blender 5.1 |
| Claude/Anthropic | 2026-03-05 | e-sverchok-claude-openaec-sources | Added Agent Skills standard, Claude Code docs, Agent SDK references. docs.anthropic.com→platform.claude.com |
| OpenAEC Foundation | 2026-03-05 | e-sverchok-claude-openaec-sources | Removed 4 broken repos. Added 4 new repos. 35 repos total in org |
