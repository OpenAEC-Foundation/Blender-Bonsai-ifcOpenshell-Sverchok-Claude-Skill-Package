# REQUIREMENTS

## What This Skill Package Must Achieve

### Primary Goal
Enable Claude to write correct, version-aware, production-quality code for Blender, Bonsai, IfcOpenShell, and Sverchok - without hallucinating APIs or patterns.

### What Claude Should Do After Loading Skills
1. Recognize which technology the user is working with (Blender, Bonsai, IfcOpenShell, Sverchok, or a combination)
2. Select the correct skill(s) automatically based on the request
3. Write code that is correct for the specified version (Blender 3.x vs 4.x, IFC2x3 vs IFC4 vs IFC4.3)
4. Avoid known anti-patterns and common AI mistakes
5. Follow best practices documented in the skill references

### Quality Guarantees
| Guarantee | Description |
|-----------|-------------|
| Version-correct | Code MUST specify which versions it targets |
| API-accurate | All method signatures verified against official docs |
| Anti-pattern-free | Known mistakes are explicitly documented and avoided |
| Deterministic | Skills use ALWAYS/NEVER language, not suggestions |
| Self-contained | Each skill works independently without requiring other skills |

---

## Per-Technology Requirements

### Blender
| Requirement | Detail |
|-------------|--------|
| Versions | Blender 3.x, 4.x, and 5.x (document ALL breaking changes per version) |
| API coverage | bpy.types, bpy.props, bpy.ops, bpy.data, bpy.context, bpy.msgbus, bpy.app |
| Core areas | Operators, Properties, Panels, Addons/Extensions, Mesh/BMesh, Modifiers |
| Node systems | Geometry Nodes (scripting, node groups, inputs/outputs), Shader Nodes (material scripting), Compositor Nodes |
| Animation | Keyframes, FCurves, Drivers, Constraints, Armatures/Bones, NLA, Actions |
| Rendering | Scene render settings, EEVEE vs Cycles via Python, camera/light scripting, batch rendering |
| Data management | Collections, Libraries (append/link), Asset Browser API, ID data lifecycle |
| Drawing/GPU | gpu module (REQUIRED for 5.0+, replaces bgl), overlays, gizmos |
| Event system | Application handlers, timers, msgbus subscriptions |
| Critical | Context system errors (the #1 AI mistake in Blender) |
| Critical | 4.0 extension system vs legacy addon format |
| Critical | 5.0 BGL removal — ALL drawing code MUST use gpu module |
| Critical | Grease Pencil API rewrite in 4.3 |

### Bonsai
| Requirement | Detail |
|-------------|--------|
| Versions | Current Bonsai (post-BlenderBIM rename) |
| API coverage | tool.Ifc, core modules, Bonsai operators |
| Key areas | Spatial structure, Property sets, Modeling, Classification, Export |
| Critical | IFC schema compliance when creating/editing elements |
| Critical | Bonsai's tool/core/ui architecture pattern |

### IfcOpenShell
| Requirement | Detail |
|-------------|--------|
| Versions | Latest IfcOpenShell + IFC2x3, IFC4, IFC4.3 schema support |
| API coverage | ifcopenshell.open/file, ifcopenshell.api, ifcopenshell.util, ifcopenshell.geom |
| Key areas | File I/O, API categories, Element traversal, Geometry, Utilities |
| Critical | Schema version differences (entity availability, attribute changes) |
| Critical | Correct use of ifcopenshell.api.run() vs direct entity manipulation |

### Sverchok
| Requirement | Detail |
|-------------|--------|
| Versions | Sverchok v1.4.0+ on Blender 4.0+/5.x |
| API coverage | Node tree API, socket types, data nesting system, scripting nodes (SNLite, Functor B, Formula Mk5, Profile Mk3) |
| Core areas | Node system architecture, 18+ node categories, 500+ nodes, update triggers, socket data cache |
| Socket system | All 16 socket types, implicit type conversions, data processing flags |
| Data model | Nesting levels (vertices level 3, edges/faces level 2, matrices level 1), 5 list matching modes |
| Scripting | SNLite socket declaration syntax, type identifiers, built-in aliases, template system |
| Programmatic API | Creating/connecting nodes from Python, parameter sweeps, batch processing |
| Custom nodes | Full node lifecycle, socket creation, property management, BMesh integration, registration |
| Parametric AEC | Structural grids, facade panels, parametric stairs, roof geometry, MEP routing, terrain generation |
| IfcSverchok | 31 IFC nodes, SvIfcStore, geometry conversion modes, 6-step IFC workflow, Bonsai integration |
| Topologic | TopologicSverchok CellComplex workflows, space adjacency graphs, dual graphs, energy simulation |
| Extensions | Sverchok-Extra (surfaces, fields, solids, SDF), Open3d integration, custom extension development |
| Critical | Data nesting errors (the #1 AI mistake in Sverchok) |
| Critical | Missing updateNode() callbacks causing silent failures |
| Critical | Socket data mutation (MUST deep copy before modifying) |
| Critical | List matching mode selection for correct data pairing |

---

## Structural Requirements

### Skill Format
- SKILL.md < 500 lines (heavy content in references/)
- YAML frontmatter with name and description (including trigger words)
- English-only content
- Deterministic language (ALWAYS/NEVER, imperative)

### Package Independence
- Each technology is a SEPARATE installable package
- skills/blender/ works without skills/bonsai/ and vice versa
- Cross-technology skills in skills/aec-cross-tech/ are optional

### Skill Categories (per technology)
| Category | Purpose | Must Include |
|----------|---------|--------------|
| syntax/ | How to write it | Method signatures, code patterns, version notes |
| impl/ | How to build it | Decision trees, workflows, step-by-step |
| errors/ | How to handle failures | Error patterns, diagnostics, recovery |
| core/ | Cross-cutting | API overview, version matrix, concepts |
| agents/ | Orchestration | Validation checklists, auto-detection |

---

## Research Requirements (before creating any skill)

1. Official documentation MUST be consulted and referenced
2. Source code MUST be checked for accuracy
3. Version differences MUST be documented
4. Anti-patterns MUST be identified from real issues (GitHub issues, forums)
5. Code examples MUST be verified (not hallucinated)

## Non-Requirements (explicitly out of scope)
- Teaching users what Blender/BIM/IFC is (skills are for Claude, not tutorials)
- GUI instructions (skills cover Python API, not UI clicks)
- Non-Python workflows (no Blender shader nodes via UI, only via Python)
- Commercial tool integrations (Revit, ArchiCAD, etc.)
