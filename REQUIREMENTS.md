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
| Versions | Blender 3.x and 4.x (document breaking changes) |
| API coverage | bpy.types, bpy.props, bpy.ops, bpy.data, bpy.context |
| Key areas | Operators, Properties, Panels, Addons, Mesh/BMesh, Modifiers |
| Critical | Context system errors (the #1 AI mistake in Blender) |
| Critical | 4.0 extension system vs legacy addon format |

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

### Sverchok (later phase)
| Requirement | Detail |
|-------------|--------|
| Versions | Current Sverchok |
| Key areas | Node creation, data flow, parametric design |

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
