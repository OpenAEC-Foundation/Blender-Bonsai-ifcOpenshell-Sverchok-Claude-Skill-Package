# Complete Scope Analysis: All Python-Scriptable Domains

**Date**: 2026-03-05
**Status**: PRELIMINARY (to be refined in Phase 3)
**Purpose**: Map EVERYTHING you can do via Python in each technology

---

## 1. Blender — Complete Python API Surface

### Application Modules (bpy.*)
| Module | Purpose | In Current Research? |
|--------|---------|---------------------|
| `bpy.context` | Active context (scene, object, mode, area) | YES - extensively |
| `bpy.data` | All blend-file data (30+ collection types) | YES - extensively |
| `bpy.ops` | 50+ operator categories (see below) | PARTIAL - operators covered generically |
| `bpy.props` | Property definitions for UI/addons | YES - extensively |
| `bpy.types` | All Blender types (900+ classes) | YES - key types covered |
| `bpy.utils` | Utilities (register, paths, previews) | PARTIAL |
| `bpy.app` | App info, version, paths, handlers, timers | PARTIAL - handlers covered |
| `bpy.msgbus` | Property change notifications (subscribe_rna) | MENTIONED only |
| `bpy.path` | Path utilities | NO |

### Standalone Modules
| Module | Purpose | In Current Research? |
|--------|---------|---------------------|
| `mathutils` | Vector, Matrix, Quaternion, Euler, Color, KDTree, BVHTree | NO |
| `bmesh` | Low-level mesh editing | YES - section 7 |
| `gpu` | GPU drawing (replaces bgl), shaders, batches | PARTIAL - migration only |
| `aud` | Audio system | NO |
| `bl_math` | Additional math functions (lerp, clamp, smoothstep) | NO |
| `freestyle` | Non-photorealistic line rendering | NO |
| `idprop` | ID property access | NO |
| `imbuf` | Image buffer operations | NO |

### bpy.ops Operator Categories (COMPLETE — 50+ categories)
| Category | Domain | In Scope? |
|----------|--------|-----------|
| `bpy.ops.object` | Object manipulation, transforms, constraints | PARTIAL |
| `bpy.ops.mesh` | Mesh editing, primitives, normals, UV | PARTIAL |
| `bpy.ops.node` | Node editor operations | NO |
| `bpy.ops.anim` | Animation, keyframes | NO |
| `bpy.ops.armature` | Armature/bone editing | NO |
| `bpy.ops.action` | Action/dopesheet | NO |
| `bpy.ops.graph` | Graph editor (FCurves) | NO |
| `bpy.ops.nla` | NLA editor | NO |
| `bpy.ops.constraint` | Constraint operations | NO |
| `bpy.ops.pose` | Pose mode operations | NO |
| `bpy.ops.curve` | Curve editing | NO |
| `bpy.ops.curves` | Hair curves (4.0+) | NO |
| `bpy.ops.surface` | NURBS surfaces | NO |
| `bpy.ops.font` | Text objects | NO |
| `bpy.ops.lattice` | Lattice deformation | NO |
| `bpy.ops.sculpt` | Sculpting operations | NO |
| `bpy.ops.brush` | Brush management | NO |
| `bpy.ops.dpaint` | Dynamic paint | NO |
| `bpy.ops.camera` | Camera operations | NO |
| `bpy.ops.render` | Rendering | NO |
| `bpy.ops.scene` | Scene management | NO |
| `bpy.ops.material` | Material operations | NO |
| `bpy.ops.texture` | Texture operations | NO |
| `bpy.ops.world` | World/environment | NO |
| `bpy.ops.image` | Image editor operations | NO |
| `bpy.ops.uv` | UV editing | NO |
| `bpy.ops.gpencil` | Grease Pencil (legacy) | NO |
| `bpy.ops.grease_pencil` | Grease Pencil (4.3+) | NO |
| `bpy.ops.particle` | Particle system | NO |
| `bpy.ops.cloth` | Cloth simulation | NO |
| `bpy.ops.fluid` | Fluid simulation | NO |
| `bpy.ops.boid` | Boid particles | NO |
| `bpy.ops.rigidbody` | Rigid body physics | NO |
| `bpy.ops.collection` | Collection management | MENTIONED |
| `bpy.ops.asset` | Asset browser operations | NO |
| `bpy.ops.file` | File operations (import/export) | NO |
| `bpy.ops.wm` | Window manager, modal, file dialogs | PARTIAL |
| `bpy.ops.screen` | Screen/workspace | NO |
| `bpy.ops.workspace` | Workspace management | NO |
| `bpy.ops.text` / `text_editor` | Text editor | NO |
| `bpy.ops.console` | Python console | NO |
| `bpy.ops.info` | Info area | NO |
| `bpy.ops.ed` | General editor ops | NO |
| `bpy.ops.clip` | Motion tracking/compositing | NO |
| `bpy.ops.mask` | Masking | NO |
| `bpy.ops.marker` | Timeline markers | NO |
| `bpy.ops.geometry` | Geometry operations | NO |
| `bpy.ops.gizmogroup` | Gizmo operations | NO |
| `bpy.ops.extensions` | Extension management (4.2+) | NO |
| `bpy.ops.preferences` | User preferences | NO |
| `bpy.ops.cycles` | Cycles-specific | NO |
| `bpy.ops.export_scene` / `import_scene` | Scene import/export | NO |
| `bpy.ops.export_anim` / `import_anim` | Animation import/export | NO |
| `bpy.ops.import_curve` | Curve import | NO |
| `bpy.ops.cachefile` | Alembic/USD cache | NO |

### bpy.data Collection Types (COMPLETE — 30+ types)
| Collection | Data Type | In Scope? |
|------------|-----------|-----------|
| `bpy.data.objects` | Objects | YES |
| `bpy.data.meshes` | Mesh data | YES |
| `bpy.data.materials` | Materials | MENTIONED |
| `bpy.data.node_groups` | Node groups (GN, Shader, Compositor) | MENTIONED |
| `bpy.data.armatures` | Armature data | MENTIONED |
| `bpy.data.actions` | Animation actions | MENTIONED |
| `bpy.data.collections` | Collections | MENTIONED |
| `bpy.data.scenes` | Scenes | PARTIAL |
| `bpy.data.cameras` | Camera data | MENTIONED |
| `bpy.data.lights` | Light data | MENTIONED |
| `bpy.data.worlds` | World settings | MENTIONED |
| `bpy.data.images` | Image data | MENTIONED |
| `bpy.data.textures` | Texture data | MENTIONED |
| `bpy.data.curves` | Curve data | NO |
| `bpy.data.fonts` | Font data | NO |
| `bpy.data.lattices` | Lattice data | NO |
| `bpy.data.grease_pencils` | Grease Pencil data | MENTIONED |
| `bpy.data.particles` | Particle settings | NO |
| `bpy.data.brushes` | Brushes | NO |
| `bpy.data.palettes` | Color palettes | NO |
| `bpy.data.paint_curves` | Paint curves | NO |
| `bpy.data.speakers` | Audio speakers | NO |
| `bpy.data.texts` | Text blocks | NO |
| `bpy.data.libraries` | Linked libraries | NO |
| `bpy.data.screens` | Screen layouts | NO |
| `bpy.data.workspaces` | Workspaces | NO |
| `bpy.data.window_managers` | Window managers | PARTIAL |
| `bpy.data.linestyles` | Freestyle line styles | NO |
| `bpy.data.shape_keys` | Shape key data | NO |
| `bpy.data.masks` | Masks | NO |
| `bpy.data.movieclips` | Motion tracking clips | NO |
| `bpy.data.cache_files` | Alembic/USD caches | NO |
| `bpy.data.volumes` | OpenVDB volumes | NO |
| `bpy.data.pointclouds` | Point clouds | NO |
| `bpy.data.hair_curves` | Hair curves (4.0+) | NO |

### Blender Python Domains Summary
| Domain | Sub-areas | Current Coverage |
|--------|-----------|-----------------|
| **Modeling** | Mesh, BMesh, Curves, NURBS, Text, Lattice, Metaball | GOOD (mesh/BMesh) |
| **Node Systems** | Geometry Nodes, Shader Nodes, Compositor, World Nodes | GOOD (supplementary-blender-gaps §1) |
| **Animation** | Keyframes, FCurves, Drivers, NLA, Actions, Shape Keys | GOOD (supplementary-blender-gaps §2) |
| **Rigging** | Armatures, Bones, Constraints, Pose Mode, IK/FK | GOOD (supplementary-blender-gaps §2) |
| **Materials** | Shader node trees, Material settings, Textures, Images | GOOD (supplementary-blender-gaps §3) |
| **Rendering** | EEVEE, Cycles, Output settings, Camera, Lights, World | GOOD (supplementary-blender-gaps §4) |
| **Simulation** | Particles, Cloth, Fluid, Rigid Body, Dynamic Paint, Boids | NONE (out of AEC scope) |
| **Sculpting** | Sculpt mode, Brushes, Multires, Dynamic Topology | NONE (out of AEC scope) |
| **UV/Texturing** | UV editing, Texture painting, Baking | PARTIAL (UV in supplementary-blender-gaps §3) |
| **Compositing** | Compositor nodes, Render layers, Post-processing | PARTIAL (supplementary-blender-gaps §1) |
| **Motion Tracking** | Camera tracking, Object tracking, Plane track | NONE (out of AEC scope) |
| **Grease Pencil** | 2D animation, GP objects, Layers, Frames | POOR (migration only) |
| **Video Editing** | VSE sequences, Strips, Effects, Audio | NONE (out of AEC scope) |
| **Asset System** | Asset Browser, Asset libraries, Tags, Catalogs | GOOD (supplementary-blender-gaps §6) |
| **I/O Formats** | FBX, OBJ, glTF, USD, Alembic, STL, PLY, SVG | GOOD (supplementary-blender-gaps §5) |
| **Addon/Extension** | bl_info, manifest, register, preferences, sub-modules | GOOD |
| **Context/Event** | Context, Handlers, Timers, msgbus, Modal | GOOD (expanded in supplementary-blender-gaps §8) |
| **Drawing/GPU** | gpu module, Shaders, Batches, Overlays, Gizmos | GOOD (supplementary-blender-gaps §7) |
| **Scripting** | Text editor, Console, Script running, bpy as module | PARTIAL (background mode in supplementary-blender-gaps §8) |

---

## 2. IfcOpenShell — Complete Python API Surface

### Core Modules
| Module | Purpose | In Current Research? |
|--------|---------|---------------------|
| `ifcopenshell.open()` | Open/parse IFC files | YES |
| `ifcopenshell.file` | File object, create_entity, by_type, by_id, by_guid | YES |
| `ifcopenshell.geom` | Geometry processing, shape settings | YES |
| `ifcopenshell.api` | High-level authoring API (30+ sub-modules) | YES |
| `ifcopenshell.util` | Utility functions (20+ sub-modules) | YES |
| `ifcopenshell.express` | EXPRESS schema parsing | NO |
| `ifcopenshell.guid` | GUID generation/compression | NO |
| `ifcopenshell.template` | IFC file templates | PARTIAL |
| `ifcopenshell.validate` | IFC validation | YES (supplementary-ifcos-gaps §6) |
| `ifcopenshell.draw` | 2D drawing generation | NO |

### ifcopenshell.api Sub-modules (COMPLETE — 30+ categories)
| Module | Purpose | In Current Research? |
|--------|---------|---------------------|
| `api.aggregate` | Spatial aggregation (assign/unassign) | YES |
| `api.boundary` | Space boundaries | NO |
| `api.classification` | Classification systems | PARTIAL |
| `api.constraint` | Design constraints | NO |
| `api.context` | Geometric representation contexts | YES |
| `api.cost` | Cost items, schedules, rates | YES (supplementary-ifcos-gaps §1) |
| `api.document` | Document references | NO |
| `api.drawing` | 2D drawing generation, annotations | YES (supplementary-ifcos-gaps §4) |
| `api.geometry` | Geometry creation, representations | YES |
| `api.georeference` | Map coordinates, georeferencing | YES (supplementary-ifcos-gaps §6) |
| `api.group` | Grouping elements | NO |
| `api.grid` | Grid systems | NO |
| `api.layer` | Presentation layers | NO |
| `api.library` | External library references | NO |
| `api.material` | Materials, material sets | YES |
| `api.nest` | Nesting relationships | NO |
| `api.owner` | Ownership, history tracking | PARTIAL |
| `api.profile` | Cross-section profiles | YES (supplementary-ifcos-gaps §5) |
| `api.project` | Project creation/setup | YES |
| `api.pset` | Property sets (add/edit/remove) | YES |
| `api.resource` | Construction resources | NO |
| `api.root` | Entity creation (create_entity) | YES |
| `api.sequence` | Work schedules, tasks, 4D simulation | YES (supplementary-ifcos-gaps §2) |
| `api.spatial` | Spatial containment, structure | YES |
| `api.structural` | Structural analysis models | NO |
| `api.style` | Visual styles, surface styles | NO |
| `api.system` | MEP systems, distribution | YES (supplementary-ifcos-gaps §3) |
| `api.type` | Type assignment | YES |
| `api.unit` | Unit assignment | YES |
| `api.void` | Openings, voids | PARTIAL |

### ifcopenshell.util Sub-modules
| Module | Purpose | In Current Research? |
|--------|---------|---------------------|
| `util.element` | Element info, traversal | YES |
| `util.unit` | Unit conversion | YES |
| `util.placement` | Local placement calculations | YES |
| `util.selector` | CSS-like element selection | YES |
| `util.date` | Date/duration parsing | YES |
| `util.cost` | Cost calculations | YES (supplementary-ifcos-gaps §1) |
| `util.sequence` | Schedule analysis | YES (supplementary-ifcos-gaps §2) |
| `util.classification` | Classification lookups | NO |
| `util.constraint` | Constraint utilities | NO |
| `util.geolocation` | Coordinate transformations | YES (supplementary-ifcos-gaps §6) |
| `util.pset` | Property set utilities | PARTIAL |
| `util.representation` | Representation utilities | PARTIAL |
| `util.shape` | Shape analysis | PARTIAL |
| `util.system` | System traversal | YES (supplementary-ifcos-gaps §3) |
| `util.type` | Type utilities | PARTIAL |
| `util.doc` | Documentation generation | NO |

### IFC Schema Coverage
| Schema | Status |
|--------|--------|
| IFC2x3 TC1 | YES - covered in research |
| IFC4 Add2 TC1 | YES - covered in research |
| IFC4x1 | NO - not mentioned |
| IFC4x2 | NO - not mentioned |
| IFC4x3 Add2 | YES - covered in research |

### IfcOpenShell Domain Summary
| Domain | Coverage |
|--------|----------|
| **File I/O** (open, write, create) | GOOD |
| **Element CRUD** (create, read, update, delete) | GOOD |
| **Spatial Structure** (site, building, storey, space) | GOOD |
| **Properties** (pset, qto) | GOOD |
| **Geometry** (representations, shapes, processing) | GOOD |
| **Materials** (single, sets, layers, constituents) | PARTIAL |
| **Types** (type assignment, occurrences) | GOOD |
| **Classification** (systems, references) | PARTIAL |
| **Cost Management** (cost items, schedules) | GOOD (supplementary-ifcos-gaps §1) |
| **4D Scheduling** (tasks, work plans, calendars) | GOOD (supplementary-ifcos-gaps §2) |
| **MEP Systems** (distribution, ports, flow) | GOOD (supplementary-ifcos-gaps §3) |
| **Structural Analysis** (loads, reactions, models) | NONE |
| **Drawing/2D** (annotations, dimensions, sheets) | PARTIAL (supplementary-ifcos-gaps §4) |
| **Documents** (references, information) | NONE |
| **Profiles** (cross-sections, parametric profiles) | GOOD (supplementary-ifcos-gaps §5) |
| **Georeferencing** (coordinates, CRS) | GOOD (supplementary-ifcos-gaps §6) |
| **Validation** (schema compliance, rules) | GOOD (supplementary-ifcos-gaps §6) |

---

## 3. Bonsai — Complete Module Surface

### Architecture: core/ + tool/ + bim/module/
| Module | Purpose | In Current Research? |
|--------|---------|---------------------|
| `aggregate` | Spatial aggregation | PARTIAL |
| `bcf` | BIM Collaboration Format | YES (supplementary-bonsai-gaps §3) |
| `blenderbim` | Legacy bridge | NO |
| `boundary` | Space boundaries | NO |
| `brick` | Brickschema (IoT/building data) | NO |
| `classification` | Classification systems | YES |
| `clash` | Clash detection | YES (supplementary-bonsai-gaps §4) |
| `constraint` | IFC constraints | NO |
| `context` | Geometric contexts | PARTIAL |
| `cost` | Cost management | NO |
| `covering` | Coverings (finishes) | NO |
| `debug` | Debugging tools | NO |
| `document` | Document management | NO |
| `drawing` | 2D drawing generation | YES (supplementary-bonsai-gaps §1) |
| `fm` | Facility management | NO |
| `georeference` | Georeferencing | NO |
| `geometry` | Geometry operations | YES |
| `group` | Element grouping | NO |
| `ifc` | Core IFC operations | YES |
| `library` | Library management | NO |
| `light` | Lighting (IES) | NO |
| `material` | Material management | YES |
| `misc` | Miscellaneous | NO |
| `model` | Parametric modeling (walls, slabs, etc.) | YES |
| `nest` | Nesting | NO |
| `owner` | Ownership tracking | PARTIAL |
| `patch` | IFC patching/migration | NO |
| `profile` | Profile definitions | NO |
| `project` | Project setup | YES |
| `pset` | Property sets | YES |
| `qto` | Quantity takeoff | YES (supplementary-bonsai-gaps §2) |
| `resource` | Resource management | NO |
| `root` | Root entity operations | YES |
| `search` | Element search/filter | NO |
| `sequence` | 4D scheduling | NO |
| `spatial` | Spatial structure | YES |
| `structural` | Structural analysis | NO |
| `style` | Visual styles | NO |
| `system` | MEP systems | NO |
| `tester` | IFC testing/validation | NO |
| `type` | Type management | YES |
| `unit` | Unit management | YES |
| `void` | Openings | PARTIAL |

### Bonsai Domain Summary
| Domain | Coverage |
|--------|----------|
| **Core BIM modeling** (walls, slabs, columns, beams) | GOOD |
| **Spatial structure** | GOOD |
| **Properties & quantities** | GOOD |
| **Classification** | PARTIAL |
| **Materials** | PARTIAL |
| **Drawing/2D** (sheets, annotations, dimensions) | GOOD (supplementary-bonsai-gaps §1) |
| **Cost management** | NONE |
| **4D scheduling** | NONE |
| **MEP systems** | NONE |
| **Clash detection** | GOOD (supplementary-bonsai-gaps §4) |
| **BCF** (issue tracking) | GOOD (supplementary-bonsai-gaps §3) |
| **Facility management** | NONE |
| **Structural analysis** | NONE |
| **Georeferencing** | NONE |
| **Quantity takeoff** | GOOD (supplementary-bonsai-gaps §2) |
| **Brickschema** (IoT) | NONE |

---

## 4. Sverchok — Complete Node Category Surface

### Node Categories
| Category | Purpose | In Current Research? |
|----------|---------|---------------------|
| Generators | Primitive geometry creation (box, sphere, torus, etc.) | NO |
| Generators Extended | Advanced generators (generative art, L-systems, etc.) | NO |
| Transforms | Move, rotate, scale, mirror, randomize | NO |
| Analyzers | Measure, calculate distances, areas, volumes, normals | NO |
| Modifier Change | Modify existing geometry (subdivide, decimate, etc.) | NO |
| Modifier Make | Create geometry from geometry (extrude, solidify, etc.) | NO |
| Number | Numeric operations, ranges, formulas | NO |
| Vector | Vector math, interpolation, fields | NO |
| Matrix | Matrix operations, apply transforms | NO |
| Logic | Boolean, comparison, switching, gates | NO |
| List Main | List operations (join, split, slice, sort) | NO |
| List Structure | Structural list operations (levels, nesting) | NO |
| List Masks | Filtering, masking operations | NO |
| List Mutators | Modify list contents | NO |
| CAD | CAD-specific operations (intersect, offset, boolean) | NO |
| Viz | Visualization (viewer, mesh viewer, curve viewer) | NO |
| Text | String operations, text generation | NO |
| Scene | Blender scene interaction | NO |
| Object | Blender object data access | NO |
| BPY data | Direct bpy data access from node trees | NO |
| Layout | UI layout nodes (frame, reroute) | NO |
| Network | Network/internet operations | NO |
| Script | Python scripting (SN, SN Lite, Formula) | NO |
| Spatial | Spatial operations (KDTree, BVHTree) | NO |
| Exchange | Import/export (JSON, CSV, SVG, DXF) | NO |
| Pulga Physics | Physics simulation in node trees | NO |
| Solid | Solid modeling (FreeCAD kernel) | NO |
| Field | Field operations (scalar, vector fields) | NO |
| Beta/Alpha Nodes | Experimental features | NO |

### Sverchok Python Integration Points
| Feature | Purpose | In Current Research? |
|---------|---------|---------------------|
| Scripted Node (SN) | Custom node with full Python | NO |
| Script Node Lite (SNL) | Lightweight custom node | NO |
| Formula Node | Math expressions | NO |
| Profile Node | Parametric 2D profiles | NO |
| Monad / Node Group | Reusable node groups | NO |
| External API | Accessing Sverchok from scripts | NO |
| Custom node development | Creating new node types | NO |

### Sverchok Extensions & Integrations
| Extension | Purpose | In Current Research? |
|-----------|---------|---------------------|
| **IfcSverchok** | IFC nodes for Sverchok — exposes IfcOpenShell API as visual nodes | NO |
| TopologicSverchok | Topological modeling (non-manifold topology for architecture) | NO |
| Sverchok-Extra | Additional nodes (NURBS, Curves, advanced geometry) | NO |
| Sverchok-Open3d | Point cloud and triangle mesh processing | NO |
| Ladybug Tools | Environmental analysis (EnergyPlus weather files) | NO |

### IfcSverchok Detail (CRITICAL for this project)
- **Source**: `IfcOpenShell/IfcOpenShell/src/ifcsverchok/` (lives INSIDE the IfcOpenShell mono-repo!)
- **Status**: Alpha — separate from Sverchok upstream, not shipped by default
- **Purpose**: Visual node programming for IFC/BIM data generation
- **Features**:
  - Creates IFC files from Sverchok node trees
  - Two geometry modes: from Blender objects OR from Sverchok geometry (verts/edges/faces)
  - Exposes IfcOpenShell API as visual nodes
  - Compatible with Bonsai (output can be imported)
  - "Re-run all nodes" button creates fresh IFC file
  - "Write File" button for export
- **Key insight**: This is the BRIDGE between Sverchok (parametric) and Bonsai (BIM)
- **GSoC 2022**: Google Summer of Code project expanded node coverage
- **Dependencies**: Requires both Sverchok AND IfcOpenShell installed

### Sverchok Coverage: ZERO (but scope is now clearer)
Sverchok is marked as "later phase" in the roadmap and has NO research coverage yet.
However, IfcSverchok makes Sverchok AEC-critical — it's not just "visual programming"
but a parametric BIM authoring tool when combined with IFC nodes.

---

## Gap Analysis Summary

### Critical Gaps (HIGH impact, missing entirely)
| Technology | Domain | Why Critical |
|------------|--------|-------------|
| Blender | Node Systems (GN, Shader, Compositor) | Most-used Python scripting domain in modern Blender |
| Blender | Animation/Rigging | Core Blender workflow, heavy Python scripting |
| Blender | Materials/Shading | Every project needs material setup |
| Blender | Rendering (EEVEE/Cycles config) | Batch rendering is a top automation use case |
| Blender | Simulation (particles, physics) | Common in VFX/architecture visualization |
| Blender | I/O Formats (FBX, glTF, USD) | Import/export scripting |
| IfcOpenShell | Cost Management | Important BIM discipline |
| IfcOpenShell | 4D Scheduling | Growing BIM requirement |
| IfcOpenShell | MEP Systems | Entire MEP discipline |
| IfcOpenShell | Drawing/2D | Document production |
| Bonsai | Drawing/2D | Primary Bonsai workflow for documentation |
| Bonsai | Clash Detection | Key BIM coordination tool |
| Bonsai | Quantity Takeoff | Critical for cost estimation |
| Bonsai | BCF | Industry standard issue tracking |
| Sverchok | Everything | Zero coverage |

### Moderate Gaps (MEDIUM impact)
| Technology | Domain |
|------------|--------|
| Blender | mathutils (Vector, Matrix, Quaternion) |
| Blender | UV editing/texturing |
| Blender | Asset system |
| Blender | Scripting (bpy as module, headless) |
| IfcOpenShell | Validation |
| IfcOpenShell | Georeferencing |
| IfcOpenShell | Profiles |
| Bonsai | Facility management |
| Bonsai | Georeferencing |

### Well Covered (GOOD)
| Technology | Domain |
|------------|--------|
| Blender | Mesh/BMesh modeling |
| Blender | Operators, Properties, Panels |
| Blender | Addon/Extension development |
| Blender | Context system, error patterns |
| Blender | Version matrix (3.x/4.x/5.0) |
| IfcOpenShell | File I/O, Element CRUD |
| IfcOpenShell | Spatial structure, Properties |
| IfcOpenShell | Geometry processing |
| IfcOpenShell | Schema version differences |
| Bonsai | Core modeling, Spatial, Properties |
| Bonsai | Architecture (core/tool/ui pattern) |

---

## Action Items for Phase 3

1. **Decide scope boundaries**: Not everything needs a skill. Prioritize by AEC relevance.
2. **AEC-relevant Blender domains to ADD**: Nodes (GN for parametric), Materials, Rendering, I/O (FBX/glTF/IFC)
3. **AEC-irrelevant to SKIP**: Video editing, motion tracking, sculpting, Grease Pencil 2D animation
4. **IfcOpenShell domains to ADD**: Cost, Scheduling, MEP, Drawing
5. **Bonsai domains to ADD**: QTO, BCF, Drawing, Clash detection
6. **Sverchok**: Defer until core packages are complete, then research

---

## Sources

- [Blender Python API](https://docs.blender.org/api/current/index.html)
- [Blender Operators](https://docs.blender.org/api/current/bpy.ops.html)
- [Blender 5.1 Python API Changes](https://developer.blender.org/docs/release_notes/5.1/python_api/)
- [IfcOpenShell Documentation](https://docs.ifcopenshell.org/)
- [IfcOpenShell API Modules](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html)
- [IfcOpenShell Python](https://docs.ifcopenshell.org/ifcopenshell-python.html)
- [Bonsai Documentation](https://docs.bonsaibim.org/)
- [Bonsai Source Code](https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai)
- [Sverchok Documentation](https://nortikin.github.io/sverchok/)
- [Sverchok Node Categories](https://sverchok.readthedocs.io/en/latest/nodes.html)
- [Sverchok GitHub](https://github.com/nortikin/sverchok)
