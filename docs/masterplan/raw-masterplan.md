# Raw Masterplan v1 - Blender-Bonsai-IfcOpenShell-Sverchok Skill Package

## Date: 2026-03-05
## Status: PRELIMINARY (will be refined after Phase 2 research)

---

## Vision
Create a comprehensive Claude Skill Package for AEC (Architecture, Engineering, Construction) technologies, enabling Claude to write correct, version-aware, production-quality code for Blender, Bonsai, IfcOpenShell, and Sverchok.

## Technology Order
1. **Blender** - Foundation (Python API, addon development)
2. **Bonsai** - BIM layer on top of Blender
3. **IfcOpenShell** - IFC engine used by Bonsai
4. Sverchok - Visual programming (later phase)

## Preliminary Skill Inventory

### Blender (~15 skills)

#### Syntax (6)
| Skill | Scope |
|-------|-------|
| blender-syntax-operators | bpy.types.Operator, execute/invoke/modal, polling, bl_idname, bl_options |
| blender-syntax-properties | bpy.props.*, PropertyGroups, update callbacks, dynamic enums |
| blender-syntax-panels | bpy.types.Panel, draw(), UILayout API, bl_space_type |
| blender-syntax-addons | bl_info, register/unregister, preferences, sub-modules, multi-file |
| blender-syntax-mesh | Mesh data, BMesh, vertices/edges/faces, bpy.data.meshes |
| blender-syntax-modifiers | Modifier stack, obj.modifiers, apply vs preview, GN modifier |

#### Implementation (4)
| Skill | Scope |
|-------|-------|
| blender-impl-operators | When to use which operator type, undo/redo, modal patterns |
| blender-impl-addons | Full addon dev workflow, packaging, distribution, testing |
| blender-impl-mesh | Mesh creation/modification workflows, BMesh vs direct, performance |
| blender-impl-automation | Batch ops, headless rendering, CLI Blender, background scripts |

#### Errors (3)
| Skill | Scope |
|-------|-------|
| blender-errors-context | Context override errors, wrong context, restricted context |
| blender-errors-operators | Poll failures, return values, registration errors |
| blender-errors-data | Orphaned data, reference counting, memory management |

#### Core (1)
| Skill | Scope |
|-------|-------|
| blender-core-api | bpy module overview, context system, depsgraph, 3.x vs 4.x changes |

#### Agents (1)
| Skill | Scope |
|-------|-------|
| blender-code-validator | Validate Blender Python scripts, context usage, API version compat |

---

### Bonsai (~12 skills)

#### Syntax (4)
| Skill | Scope |
|-------|-------|
| bonsai-syntax-elements | Creating/editing IFC elements, tool.Ifc, element manipulation |
| bonsai-syntax-properties | Property sets, quantity sets, IFC properties via API |
| bonsai-syntax-spatial | Spatial structure (Site/Building/Storey/Space), decomposition |
| bonsai-syntax-geometry | Geometry representations, mapped items, body geometry, openings |

#### Implementation (4)
| Skill | Scope |
|-------|-------|
| bonsai-impl-project | BIM project setup, IFC schema selection, project structure |
| bonsai-impl-modeling | Wall/slab/column creation, type assignment, parametric objects |
| bonsai-impl-classification | Classification systems (Uniclass, OmniClass, NL-SfB), materials |
| bonsai-impl-export | IFC export, MVD selection, quality checks, schema validation |

#### Errors (2)
| Skill | Scope |
|-------|-------|
| bonsai-errors-ifc | Schema violations, missing attributes, invalid relationships |
| bonsai-errors-geometry | Invalid representations, boolean failures, geometry errors |

#### Core (1)
| Skill | Scope |
|-------|-------|
| bonsai-core-architecture | Internal architecture, module system, tool/core/ui separation |

#### Agents (1)
| Skill | Scope |
|-------|-------|
| bonsai-ifc-validator | Validate IFC models, completeness, schema compliance |

---

### IfcOpenShell (~13 skills)

#### Syntax (5)
| Skill | Scope |
|-------|-------|
| ifcos-syntax-fileio | ifcopenshell.open(), file.write(), create_entity(), schema |
| ifcos-syntax-api | ifcopenshell.api.run() - root, spatial, geometry, type, pset, material |
| ifcos-syntax-elements | by_type(), by_id(), by_guid(), inverse references, traversal |
| ifcos-syntax-geometry | ifcopenshell.geom, shape processing, settings, BRep/tessellation |
| ifcos-syntax-util | ifcopenshell.util.* - element, unit, placement, selector, date |

#### Implementation (3)
| Skill | Scope |
|-------|-------|
| ifcos-impl-creation | Creating IFC files from scratch, minimal valid IFC, element workflow |
| ifcos-impl-extraction | Data extraction, quantity takeoff, property queries, export |
| ifcos-impl-modification | Modifying existing IFC, property editing, geometry manipulation |

#### Errors (2)
| Skill | Scope |
|-------|-------|
| ifcos-errors-schema | IFC2x3 vs IFC4 vs IFC4.3, entity compatibility, attribute errors |
| ifcos-errors-geometry | Geometry processing errors, shape creation failures, placement |

#### Core (2)
| Skill | Scope |
|-------|-------|
| ifcos-core-schemas | IFC schema overview, entity hierarchy, attribute types, version selection |
| ifcos-core-concepts | IFC fundamentals: ownership, placement, representation, relationships |

#### Agents (1)
| Skill | Scope |
|-------|-------|
| ifcos-code-validator | Validate IfcOpenShell scripts, schema compat, API usage |

---

### Cross-Technology (~3 skills)

| Skill | Scope |
|-------|-------|
| aec-core-ifc-fundamentals | IFC standard, MVD, schema versions, buildingSMART ecosystem |
| aec-core-bim-workflows | BIM workflow patterns, LOD/LOI, coordination, clash detection |
| aec-workflow-orchestrator | Auto-detect which skill set to use based on user request |

---

## Execution Strategy

### Parallel Execution via Open-Agents
- Phase 2: 3 research agents in parallel (one per technology)
- Phase 4: 3 research agents per batch (topic-specific)
- Phase 5: 3 writer agents per batch, quality gate between batches
- Phase 6: 2 validator agents

### Dependency Architecture
```
Layer 1 (Foundation): Syntax Skills
  No dependencies, pure API reference

Layer 2 (Cross-cutting): Core Skills
  Reference syntax skills

Layer 3 (Workflows): Implementation Skills
  Depend on syntax + core

Layer 4 (Robustness): Error Handling Skills
  Reference all other layers

Layer 5 (Orchestration): Agent Skills
  Orchestrate all other skills
```

### Build Order per Technology
1. Core skills first (foundation knowledge)
2. Syntax skills (API patterns)
3. Implementation skills (depend on syntax)
4. Error skills (depend on all above)
5. Agent skills (orchestrate everything)

---

## Quality Criteria
- SKILL.md < 500 lines
- English-only
- Deterministic language (ALWAYS/NEVER)
- Version-explicit (Blender 3.x/4.x, IFC2x3/IFC4/IFC4.3)
- Valid YAML frontmatter
- All referenced files exist
- Cross-references between skills are correct

## This masterplan will be refined after Phase 2 (Deep Research).
