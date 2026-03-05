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

### Blender (~25 skills)

#### Syntax (10)
| Skill | Scope |
|-------|-------|
| blender-syntax-operators | bpy.types.Operator, execute/invoke/modal, polling, bl_idname, bl_options |
| blender-syntax-properties | bpy.props.*, PropertyGroups, update callbacks, dynamic enums |
| blender-syntax-panels | bpy.types.Panel, draw(), UILayout API, bl_space_type, sub-panels |
| blender-syntax-addons | bl_info (legacy), blender_manifest.toml (4.2+), register/unregister, preferences, multi-file |
| blender-syntax-mesh | Mesh data, BMesh, vertices/edges/faces, UV layers, vertex colors, normals |
| blender-syntax-modifiers | Modifier stack, obj.modifiers, apply vs preview, evaluated mesh |
| blender-syntax-nodes | Geometry Nodes via Python, Shader Nodes scripting, Compositor Nodes, node trees, node groups, socket I/O, custom node groups |
| blender-syntax-animation | Keyframes, FCurves, Drivers, Constraints, Armatures/Bones/Pose, Actions, NLA strips |
| blender-syntax-rendering | Scene.render settings, EEVEE vs Cycles config, camera/light scripting, output format, batch render |
| blender-syntax-data | Collections, library append/link, Asset Browser API, ID data lifecycle, user count, fake user |

#### Implementation (6)
| Skill | Scope |
|-------|-------|
| blender-impl-operators | When to use which operator type, undo/redo, modal patterns, timers |
| blender-impl-addons | Full addon/extension dev workflow, packaging, distribution, testing, migration 3.x→4.2+ |
| blender-impl-mesh | Mesh creation/modification workflows, BMesh vs direct, performance, from_pydata patterns |
| blender-impl-automation | Batch ops, headless rendering, CLI Blender, background scripts, subprocess |
| blender-impl-nodes | Building Geometry Node groups via Python, procedural material creation, parametric workflows |
| blender-impl-animation | Rigging workflows, constraint setup, driver expressions, animation baking, NLA workflow |

#### Errors (4)
| Skill | Scope |
|-------|-------|
| blender-errors-context | Context override errors, wrong context, restricted context, temp_override |
| blender-errors-operators | Poll failures, return values, registration errors, wrong bl_options |
| blender-errors-data | Orphaned data, reference counting, memory management, stale references, undo invalidation |
| blender-errors-version | Version-specific pitfalls: BGL→gpu (5.0), bone.layers→collections (4.0), EEVEE changes, GP rewrite (4.3) |

#### Core (3)
| Skill | Scope |
|-------|-------|
| blender-core-api | bpy module overview, context system, depsgraph, event system (handlers/timers/msgbus) |
| blender-core-versions | Complete version matrix 3.x/4.0/4.1/4.2/4.3/5.0, ALL breaking changes, migration paths |
| blender-core-gpu | gpu module (replaces bgl), shader creation, batch drawing, overlays, gizmos, SpaceView3D draw handlers |

#### Agents (2)
| Skill | Scope |
|-------|-------|
| blender-code-validator | Validate Blender Python scripts, context usage, API version compat |
| blender-version-migrator | Auto-detect version issues, suggest migration from 3.x→4.x→5.x, flag deprecated API |

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

### Sverchok (~8 skills, later phase — research pending)

#### Syntax (3)
| Skill | Scope |
|-------|-------|
| sverchok-syntax-nodes | Core node types (Generators, Transforms, Analyzers, Modifiers), socket types (Vertices/Strings/Matrix), node tree creation via Python |
| sverchok-syntax-scripting | Scripted Node (SN), Script Node Lite (SNL), Formula Node, custom node development |
| sverchok-syntax-data | Data flow patterns, list nesting levels, socket connections, node group I/O |

#### Implementation (2)
| Skill | Scope |
|-------|-------|
| sverchok-impl-parametric | Parametric design workflows, parameter-driven geometry, node graph patterns |
| sverchok-impl-ifcsverchok | IfcSverchok nodes: IFC file generation from node trees, geometry modes (Blender objects vs Sverchok verts/edges/faces), Bonsai integration |

#### Errors (1)
| Skill | Scope |
|-------|-------|
| sverchok-errors-nodes | Common node errors, data mismatch, socket type issues, performance with complex trees |

#### Core (1)
| Skill | Scope |
|-------|-------|
| sverchok-core-overview | Node categories (25+), extension system (IfcSverchok, Topologic, Extra), socket types, data model |

#### Agents (1)
| Skill | Scope |
|-------|-------|
| sverchok-node-advisor | Suggest optimal node combinations for AEC parametric tasks |

---

### Cross-Technology (~5 skills)

| Skill | Scope |
|-------|-------|
| aec-core-ifc-fundamentals | IFC standard, MVD, schema versions, buildingSMART ecosystem |
| aec-core-bim-workflows | BIM workflow patterns, LOD/LOI, coordination, clash detection |
| aec-core-python-runtime | Python runtime quirks per technology: Blender embedded CPython restrictions, IfcOpenShell C++ bindings, threading, memory, undo |
| aec-cross-sverchok-bonsai | IfcSverchok bridge: parametric Sverchok geometry → IFC output → Bonsai import, bidirectional workflow |
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
