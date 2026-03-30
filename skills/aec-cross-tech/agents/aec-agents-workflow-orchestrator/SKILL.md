---
name: aec-agents-workflow-orchestrator
description: >
  Use when a task spans multiple AEC technologies (Blender, IfcOpenShell, Bonsai, Sverchok)
  and you need to decide which tool handles which step, or when orchestrating multi-step BIM
  pipelines such as IFC creation to visualization to authoring. Prevents the common mistake of
  mixing IfcStore.get_file() with ifcopenshell.open() in the same context, or calling Bonsai
  operators outside a Bonsai-loaded session. Provides decision trees for technology selection,
  workflow sequencing, and bridge patterns between tools.
  Keywords: cross-technology, workflow orchestration, BIM pipeline, technology selection, Blender IfcOpenShell Bonsai integration, multi-step AEC, bridge pattern, which tool should I use, combine Blender and IFC, multi-tool workflow, how to connect tools.
license: MIT
compatibility: "Designed for Claude Code. Requires Python 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# AEC Workflow Orchestrator

> **Scope**: Multi-technology workflow orchestration for Blender + IfcOpenShell + Bonsai
> **Version coverage**: Blender 3.x-5.x | IfcOpenShell 0.8.x | IFC2X3/IFC4/IFC4X3 | Bonsai v0.8.x
> **Role**: Agent-level skill — routes tasks, sequences operations, coordinates technology boundaries

## Critical Warnings

1. **ALWAYS** determine the execution context FIRST: standalone Python, Blender Python, or Bonsai-loaded. The available API surface differs per context.
2. **NEVER** mix `IfcStore.get_file()` (Bonsai) with `ifcopenshell.open()` on the same file. Bonsai owns the in-memory IFC graph.
3. **ALWAYS** use `ifcopenshell.api.run()` for IFC mutations in ALL contexts. NEVER modify entity attributes directly.
4. **ALWAYS** call `bpy.ops.bim.edit_object_placement()` after moving Blender objects when Bonsai is active.
5. **NEVER** use `blenderbim.*` imports. Use `bonsai.*` (renamed in v0.8.0, 2024).
6. **ALWAYS** set up units and geometric contexts before creating geometry.
7. **NEVER** use `void.add_opening()` in Bonsai v0.8.0+. Use `feature.add_feature()`.

---

## Quick Reference: Technology Selection Decision Tree

```
What does the task require?
|
+-- Pure IFC data manipulation (no 3D viewport needed)?
|   |
|   +-- Single file processing?
|   |   --> Standalone IfcOpenShell
|   |   --> Import: ifcopenshell, ifcopenshell.api
|   |   --> File: ifcopenshell.open(path) or project.create_file
|   |
|   +-- Batch processing multiple files?
|       --> Standalone IfcOpenShell in a loop
|       --> pathlib.Path.glob("*.ifc") iteration pattern
|       --> NO Blender dependency required
|
+-- IFC data + 3D visualization (view geometry in viewport)?
|   |
|   +-- Read-only visualization (no BIM editing)?
|   |   --> IfcOpenShell geometry extraction + Blender bpy
|   |   --> ifcopenshell.geom.create_shape() -> bpy mesh
|   |   --> Lightweight, no Bonsai dependency
|   |
|   +-- BIM authoring (create/edit IFC elements with visual feedback)?
|       --> Bonsai-loaded context (Blender + Bonsai addon)
|       --> IfcStore.get_file() for IFC access
|       --> bpy.ops.bim.* for operations with Blender sync
|
+-- Pure Blender 3D operations (no IFC/BIM)?
|   --> Standard Blender Python (bpy)
|   --> No IfcOpenShell or Bonsai needed
|   --> Refer to blender-core-api skill
|
+-- Parametric/generative design (arrays, patterns, data-driven geometry)?
|   |
|   +-- Visual node-based workflow?
|   |   --> Sverchok (SverchCustomTreeType node trees)
|   |   --> Refer to sverchok-core-concepts, sverchok-impl-parametric
|   |
|   +-- Generate IFC from parametric geometry?
|   |   --> IfcSverchok (31-node IFC bridge inside Sverchok)
|   |   --> Refer to sverchok-impl-ifcsverchok
|   |
|   +-- Building topology analysis (adjacency, envelope, dual graph)?
|       --> TopologicSverchok
|       --> Refer to sverchok-impl-topologic
|
+-- Headless automation (CI/CD, server, batch)?
    |
    +-- IFC-only operations?
    |   --> python script.py (standalone IfcOpenShell)
    |
    +-- Blender operations needed?
        --> blender --background --python script.py
        --> NEVER use viewport-dependent operators in background mode
```

---

## Workflow Patterns

### Pattern A: IFC-First Workflow

**Use when**: The primary deliverable is an IFC file. Blender is optional (for visualization only).

```
Step 1: Create/Open IFC (IfcOpenShell)
  --> ifcopenshell.api.run("project.create_file") or ifcopenshell.open()

Step 2: Build structure (IfcOpenShell)
  --> Create spatial hierarchy: Project -> Site -> Building -> Storey
  --> Set units and geometric contexts

Step 3: Create elements (IfcOpenShell)
  --> root.create_entity for each element
  --> geometry.add_*_representation for shapes
  --> spatial.assign_container for placement

Step 4: Enrich (IfcOpenShell)
  --> pset.add_pset + pset.edit_pset for properties
  --> type.assign_type for type assignments
  --> material.assign_material for materials

Step 5 (optional): Visualize (Blender)
  --> ifcopenshell.geom.create_shape() -> bpy mesh
  --> OR open in Bonsai for interactive inspection

Step 6: Export (IfcOpenShell)
  --> model.write("output.ifc")
```

### Pattern B: Blender-First Workflow

**Use when**: Starting from Blender geometry that needs BIM data attached.

```
Step 1: Create geometry (Blender bpy)
  --> bpy.data.meshes.new() + mesh.from_pydata()
  --> OR use bpy.ops.mesh.primitive_*_add()
  --> OR import from external format (FBX, OBJ, glTF)

Step 2: Prepare for BIM (Blender)
  --> Organize into collections matching spatial hierarchy
  --> Name objects according to IFC conventions
  --> Clean geometry (manifold, correct normals)

Step 3: Assign IFC identity (Bonsai)
  --> Load Bonsai addon
  --> Create IFC project: bpy.ops.bim.create_project()
  --> Assign IFC classes to Blender objects
  --> bpy.ops.bim.assign_class() for each object

Step 4: Enrich (Bonsai + IfcOpenShell)
  --> Add property sets via ifcopenshell.api.run("pset.*")
  --> Assign types, materials, classifications
  --> Build spatial containment

Step 5: Save (Bonsai)
  --> model.write("output.ifc") via IfcStore
```

### Pattern C: Bonsai-Native Workflow

**Use when**: Full BIM authoring with live IFC integration.

```
Step 1: Initialize project (Bonsai)
  --> bpy.ops.bim.create_project(schema="IFC4")
  --> Sets up IfcProject, default units, contexts automatically

Step 2: Build spatial structure (Bonsai)
  --> bpy.ops.bim.add_building() / bpy.ops.bim.add_storey()
  --> Spatial hierarchy managed by Bonsai operators

Step 3: Create BIM elements (Bonsai)
  --> bpy.ops.bim.add_wall() / add_slab() / add_column()
  --> Each operator creates IFC entity + Blender mesh simultaneously
  --> Bidirectional sync is automatic

Step 4: Edit properties (Bonsai + IfcOpenShell)
  --> model = IfcStore.get_file()
  --> ifcopenshell.api.run("pset.add_pset", model, ...)
  --> OR use Bonsai property panels

Step 5: Validate and save (Bonsai)
  --> Validate spatial containment, property completeness
  --> bpy.ops.bim.save_project() — writes .ifc directly
```

---

## Multi-Step Orchestration: Common AEC Pipelines

### Pipeline 1: IFC Creation + Validation + Visualization

```
[IfcOpenShell: Create]  -->  [IfcOpenShell: Validate]  -->  [Blender: Visualize]
                                                              (optional)
Technologies: IfcOpenShell (required), Blender (optional)
Execution: Standalone Python -> (optional) Blender Python
```

**Sequencing rules:**
- Create the complete IFC model BEFORE validation
- Validate BEFORE visualization (do not visualize broken models)
- Visualization requires `ifcopenshell.geom` (compiled C++ extension)

### Pipeline 2: Multi-File Property Enrichment

```
[IfcOpenShell: Open files]  -->  [IfcOpenShell: Read/Map]  -->  [IfcOpenShell: Enrich]  -->  [IfcOpenShell: Write]
Technologies: IfcOpenShell only
Execution: Standalone Python
```

**Sequencing rules:**
- Open source files read-only; write to separate output directory
- NEVER modify GlobalIds on existing entities
- Preserve OwnerHistory on IFC2X3 files (MANDATORY)

### Pipeline 3: Blender Model to IFC Export

```
[Blender: Geometry]  -->  [Bonsai: Assign IFC]  -->  [IfcOpenShell: Enrich]  -->  [Bonsai: Save]
Technologies: Blender (required), Bonsai (required), IfcOpenShell (required)
Execution: Blender Python with Bonsai addon loaded
```

**Sequencing rules:**
- Geometry MUST be clean before IFC assignment (manifold, correct normals)
- Assign IFC classes BEFORE adding properties
- Create spatial hierarchy BEFORE assigning containers

### Pipeline 4: IFC Analysis + Report Generation

```
[IfcOpenShell: Open]  -->  [IfcOpenShell: Query/Traverse]  -->  [Python: Generate Report]
Technologies: IfcOpenShell (required), standard Python (csv, json)
Execution: Standalone Python
```

**Sequencing rules:**
- Use `ifcopenshell.util.element.get_psets()` for property extraction
- Use `model.by_type()` for element enumeration
- Handle schema differences: IFC2X3 vs IFC4 entity names

### Pipeline 5: Bonsai Authoring + Standalone Post-Processing

```
[Bonsai: Author]  -->  [Save .ifc]  -->  [IfcOpenShell: Post-process]  -->  [Write .ifc]
Technologies: Bonsai (step 1), IfcOpenShell standalone (step 2)
Execution: Blender + Bonsai -> Standalone Python (separate process)
```

**Sequencing rules:**
- Save from Bonsai FIRST, THEN open in standalone IfcOpenShell
- NEVER have Bonsai and standalone IfcOpenShell open the same file simultaneously
- Post-processing runs in a SEPARATE Python process (not inside Blender)

---

## Technology Bridge Patterns

### Bridge 1: IfcOpenShell -> Blender (Geometry Transfer)

```python
# Convert IFC geometry to Blender mesh
# Context: Blender Python | Version: Blender 3.x-5.x, IfcOpenShell 0.8.x
import ifcopenshell
import ifcopenshell.geom
import bpy

ifc_file = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

for element in ifc_file.by_type("IfcProduct"):
    if element.Representation is None:
        continue
    shape = ifcopenshell.geom.create_shape(settings, element)
    verts = shape.geometry.verts
    faces = shape.geometry.faces
    vertices = [(verts[i], verts[i+1], verts[i+2]) for i in range(0, len(verts), 3)]
    triangles = [(faces[i], faces[i+1], faces[i+2]) for i in range(0, len(faces), 3)]
    mesh = bpy.data.meshes.new(name=element.Name or element.is_a())
    mesh.from_pydata(vertices, [], triangles)
    mesh.update()
    obj = bpy.data.objects.new(element.Name or element.is_a(), mesh)
    bpy.context.collection.objects.link(obj)
```

### Bridge 2: Blender -> IfcOpenShell (Geometry Export)

```python
# Extract Blender mesh data for IFC representation
# Context: Blender Python | Version: Blender 3.x-5.x
import bpy
import ifcopenshell
import ifcopenshell.api

obj = bpy.context.active_object
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh = obj_eval.to_mesh()

verts = [tuple(v.co) for v in mesh.vertices]
faces = [tuple(p.vertices) for p in mesh.polygons]

obj_eval.to_mesh_clear()  # ALWAYS clean up evaluated mesh

# Create IFC representation from extracted data
# Use ifcopenshell.api for tessellated representation
```

### Bridge 3: Bonsai <-> IfcOpenShell (Bidirectional Entity Access)

```python
# Context: Blender with Bonsai loaded | Version: Bonsai v0.8.x
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell.util.element

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Blender object -> IFC entity
obj = bpy.context.active_object
ifc_id = obj.BIMObjectProperties.ifc_definition_id
if ifc_id == 0:
    raise RuntimeError("Object is not linked to IFC")
entity = model.by_id(ifc_id)

# IFC entity -> Blender object (via IfcStore maps)
blender_obj = IfcStore.id_map.get(entity.id())

# Read properties from IFC entity
psets = ifcopenshell.util.element.get_psets(entity)
container = ifcopenshell.util.element.get_container(entity)
element_type = ifcopenshell.util.element.get_type(entity)
```

### Bridge 4: Context Detection (Runtime Environment Probe)

```python
# Detect available technologies at runtime
def detect_context():
    """Determine which AEC technologies are available."""
    context = {"blender": False, "bonsai": False, "ifcopenshell": False}

    try:
        import bpy
        context["blender"] = True
        context["blender_version"] = bpy.app.version
    except ImportError:
        pass

    try:
        import ifcopenshell
        context["ifcopenshell"] = True
    except ImportError:
        pass

    if context["blender"]:
        try:
            from bonsai.bim.ifc import IfcStore
            context["bonsai"] = True
            context["bonsai_has_file"] = IfcStore.get_file() is not None
        except ImportError:
            pass

    return context
```

---

## Execution Context Compatibility Matrix

| Operation | Standalone Python | Blender Python | Bonsai Context |
|-----------|------------------|---------------|---------------|
| `ifcopenshell.open()` | YES | YES | NO (use IfcStore) |
| `ifcopenshell.api.run()` | YES | YES | YES |
| `ifcopenshell.geom.create_shape()` | YES | YES | YES |
| `bpy.data.*` | NO | YES | YES |
| `bpy.ops.mesh.*` | NO | YES | YES |
| `bpy.ops.bim.*` | NO | NO | YES |
| `IfcStore.get_file()` | NO | NO | YES |
| `tool.Ifc.get_entity()` | NO | NO | YES |
| Background/headless mode | YES (python) | YES (blender --background) | YES (blender --background) |

## Version-Specific Routing Rules

| Condition | Route To | Reason |
|-----------|----------|--------|
| IFC4X3 infrastructure entities needed | IfcOpenShell with schema="IFC4X3" | IfcRoad, IfcBridge only in IFC4X3 |
| IFC2X3 file detected | IfcOpenShell with OwnerHistory handling | OwnerHistory is MANDATORY in IFC2X3 |
| Blender 5.0+ drawing code | `gpu` module exclusively | `bgl` module REMOVED in Blender 5.0 |
| Blender 4.0+ context overrides | `context.temp_override()` | Dict-based overrides REMOVED in 4.0 |
| Bonsai v0.8.0+ void operations | `feature.add_feature()` | `void.add_opening()` replaced |
| IfcBuildingElement vs IfcBuiltElement | Check `model.schema` | IFC4X3 renamed to IfcBuiltElement |

---

## Technology Selection by Task Type

| Task | Primary Technology | Secondary | Notes |
|------|-------------------|-----------|-------|
| Create IFC from scratch | IfcOpenShell | — | No Blender needed |
| Validate IFC model | IfcOpenShell | — | `ifcopenshell.validate` |
| Extract quantities from IFC | IfcOpenShell | — | `ifcopenshell.util.element` |
| Visualize IFC geometry | IfcOpenShell + Blender | — | `geom.create_shape()` -> bpy mesh |
| Author BIM model interactively | Bonsai | IfcOpenShell | Bonsai handles sync |
| Batch enrich IFC files | IfcOpenShell | — | Loop with `pathlib.glob()` |
| Generate construction drawings | Bonsai | IfcOpenShell | Bonsai drawing module |
| Clash detection | Bonsai | IfcOpenShell | Bonsai clash module |
| Quantity takeoff | Bonsai OR IfcOpenShell | — | Bonsai QTO or standalone |
| IFC to CSV/JSON report | IfcOpenShell | Python stdlib | `ifcopenshell.util.element` |
| Custom Blender addon with BIM | Blender + Bonsai | IfcOpenShell | Follow Bonsai module pattern |
| Parametric/generative design | Sverchok | Blender | Node-based visual programming |
| IFC from parametric geometry | IfcSverchok (Sverchok) | IfcOpenShell | 31-node visual IFC pipeline |
| Building topology analysis | TopologicSverchok | IfcOpenShell | Adjacency, envelope, dual graph |
| CI/CD model validation | IfcOpenShell | — | Standalone, no GUI |

---

## Reference Links

- [Workflow Methods](references/methods.md) — Technology-specific workflow entry points and method signatures
- [End-to-End Examples](references/examples.md) — Complete orchestrated workflow examples
- [Anti-Patterns](references/anti-patterns.md) — Cross-technology workflow mistakes to avoid

### Dependency Skills

- `blender-core-api` — Blender Python API fundamentals
- `ifcos-core-concepts` — IFC data model and IfcOpenShell concepts
- `bonsai-core-architecture` — Bonsai addon architecture and patterns
- `aec-core-bim-workflows` — Cross-technology BIM workflow patterns

### Official Sources

- IfcOpenShell Documentation: https://docs.ifcopenshell.org
- Bonsai Documentation: https://docs.bonsaibim.org
- Blender Python API: https://docs.blender.org/api/current/index.html
- IFC4X3 Standard: https://ifc43-docs.standards.buildingsmart.org/
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell
- OSArch Community: https://community.osarch.org
