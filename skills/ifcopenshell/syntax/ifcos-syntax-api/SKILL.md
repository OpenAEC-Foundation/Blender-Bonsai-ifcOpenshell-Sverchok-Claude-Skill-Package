---
name: ifcos-syntax-api
description: >
  Use when writing IfcOpenShell Python code that creates, modifies, or deletes IFC entities.
  Prevents the #1 AI mistake: using create_entity() or direct attribute assignment instead
  of ifcopenshell.api.run(). Covers all 30+ API modules, invocation patterns, parameter
  conventions, and the difference between api.run() and direct module calls.
  Keywords: ifcopenshell.api, api.run, create_entity, IFC, BIM, IfcWall, IfcSlab, API modules, ifcopenshell Python, how to use api.run, ifcopenshell tutorial.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell API Module System

## Quick Reference

### Critical Warnings

- **ALWAYS** use `ifcopenshell.api.run()` or direct module calls for IFC mutations. NEVER modify entity attributes directly (e.g., `wall.Name = "X"`) — use `api.run("attribute.edit_attributes", ...)` instead.
- **ALWAYS** import `ifcopenshell.api` before calling any API function. The module uses lazy loading; functions are NOT available without this import.
- **ALWAYS** pass the `file` (model) object as the first positional argument after the function name in `api.run()`.
- **NEVER** invent API module or function names. There are exactly 35 modules — see the module table below. Hallucinated calls like `api.run("element.create", ...)` or `api.run("wall.add", ...)` do NOT exist.
- **ALWAYS** use keyword arguments for all parameters after the model. Positional arguments beyond the model are NOT supported.
- **NEVER** use `model.create_entity()` for production code. Use `api.run("root.create_entity", ...)` — it generates GlobalIds, sets ownership, and validates predefined types automatically.
- **ALWAYS** set up a complete project before creating elements: IfcProject → units → contexts → spatial hierarchy. See the bootstrap pattern below.
- **NEVER** assume `products` parameters accept single elements. Since IfcOpenShell v0.8+, most relationship functions expect `products` as a **list**, not a single entity.

### Decision Tree: Which API Module to Use

```
What do you need to do?
├── Create/remove/copy IFC entities?
│   └── root (create_entity, remove_product, copy_class, reassign_class)
│
├── Build spatial hierarchy?
│   ├── Project → Site → Building → Storey → Space?
│   │   └── aggregate (assign_object, unassign_object)
│   ├── Place elements in a storey/space?
│   │   └── spatial (assign_container, unassign_container)
│   └── Reference element in multiple spaces?
│       └── spatial (reference_structure, dereference_structure)
│
├── Add geometry to elements?
│   ├── Set up representation contexts first?
│   │   └── context (add_context — root context, then subcontexts)
│   ├── Parametric wall/slab/beam geometry?
│   │   └── geometry (add_wall_representation, add_slab_representation,
│   │       add_profile_representation)
│   ├── Custom mesh geometry?
│   │   └── geometry (add_mesh_representation)
│   ├── Position element in 3D space?
│   │   └── geometry (edit_object_placement)
│   └── Connect geometry to element?
│       └── geometry (assign_representation)
│
├── Set properties on elements?
│   ├── Key-value metadata (name, fire rating, etc.)?
│   │   └── pset (add_pset, edit_pset)
│   └── Measurable quantities (length, area, volume)?
│       └── pset (add_qto, edit_qto)
│
├── Assign materials?
│   ├── Simple single material?
│   │   └── material (add_material, assign_material)
│   ├── Layered construction (walls, slabs)?
│   │   └── material (add_material_set with IfcMaterialLayerSet, add_layer)
│   ├── Profiled sections (beams, columns)?
│   │   └── material (add_material_set with IfcMaterialProfileSet, add_profile)
│   └── Composite (windows, doors)?
│       └── material (add_material_set with IfcMaterialConstituentSet)
│
├── Assign/manage types?
│   └── type (assign_type, unassign_type, map_type_representations)
│
├── Edit entity attributes directly?
│   └── attribute (edit_attributes)
│
├── Create openings/voids?
│   └── void (add_opening, add_filling)
│
├── Set up units?
│   └── unit (assign_unit — defaults to SI)
│
├── Manage project file?
│   └── project (create_file)
│
└── Other domains?
    ├── Classification → classification
    ├── Groups → group
    ├── Layers (CAD) → layer
    ├── Visual styles → style
    ├── Cost management → cost
    ├── Scheduling (4D) → sequence
    ├── MEP systems → system
    ├── Structural analysis → structural
    ├── Documents → document
    ├── Constraints → constraint
    ├── Resources → resource
    ├── Owner/actors → owner
    ├── Profiles → profile
    ├── Nesting → nest
    ├── Boundaries → boundary
    ├── Libraries → library
    ├── Drawing/annotations → drawing
    ├── Georeference → georeference
    ├── Grid → grid
    ├── Infrastructure alignment → alignment
    ├── Coordinate geometry → cogo
    ├── Features → feature
    └── Property set templates → pset_template
```

---

## Essential Patterns

### Pattern 1: Two Equivalent Invocation Styles

```python
# IfcOpenShell v0.8+: both styles produce identical results

import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Style A: api.run() with dot-notation string (original, most documented)
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 1")

# Style B: Direct module call (shorter, IDE-friendly)
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall", name="Wall 1")
```

**ALWAYS** use one style consistently within a project. Both are correct. `api.run()` is more common in documentation and tutorials.

### Pattern 2: Complete Project Bootstrap

```python
# IfcOpenShell: IFC4 (change version/schema for IFC2X3 or IFC4X3)
import ifcopenshell
import ifcopenshell.api

# Step 1: Create file with proper headers (production use)
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Step 2: Create project entity
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")

# Step 3: Assign units (defaults to SI metric)
ifcopenshell.api.run("unit.assign_unit", model)

# Step 4: Create representation contexts
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Step 5: Build spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Default Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
```

**ALWAYS** follow this sequence: file → project → units → contexts → spatial hierarchy. Skipping steps produces invalid IFC files.

### Pattern 3: Create Element with Full Data

```python
# IfcOpenShell: IFC4 (requires completed bootstrap above)

# Create wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")

# Add material to type (best practice: assign to types, not occurrences)
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30/37", category="concrete")
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=concrete)

# Create wall occurrence
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Assign type
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# Add geometry
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)

# Place in spatial structure
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# Add properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90",
    "ThermalTransmittance": 0.24,
})
```

---

## Complete API Module Table (35 Modules)

| Module | Category | Purpose | Key Functions |
|--------|----------|---------|---------------|
| `root` | Core | Create/remove/copy entities | `create_entity`, `remove_product`, `copy_class`, `reassign_class` |
| `spatial` | Spatial | Containment and referencing | `assign_container`, `unassign_container`, `reference_structure` |
| `aggregate` | Spatial | Hierarchical decomposition | `assign_object`, `unassign_object` |
| `geometry` | Geometry | Representations and placement | `add_wall_representation`, `add_mesh_representation`, `add_profile_representation`, `assign_representation`, `edit_object_placement` |
| `context` | Geometry | Representation contexts | `add_context`, `edit_context`, `remove_context` |
| `pset` | Properties | Property and quantity sets | `add_pset`, `edit_pset`, `add_qto`, `edit_qto` |
| `material` | Materials | Material definitions and sets | `add_material`, `assign_material`, `add_material_set`, `add_layer` |
| `type` | Types | Type definitions | `assign_type`, `unassign_type`, `map_type_representations` |
| `attribute` | Data | Direct attribute editing | `edit_attributes` |
| `unit` | Setup | Measurement units | `assign_unit`, `add_si_unit`, `add_conversion_based_unit` |
| `project` | Setup | Project file creation | `create_file`, `append_asset` |
| `owner` | Organization | Actors and roles | `add_person`, `add_organisation`, `set_user` |
| `classification` | Organization | Classification systems | `add_classification`, `add_reference` |
| `group` | Organization | Element grouping | `add_group`, `assign_group`, `unassign_group` |
| `void` | Geometry | Openings and voids | `add_opening`, `add_filling` |
| `style` | Visualization | Visual styles | `add_style`, `add_surface_style`, `assign_representation_styles` |
| `layer` | Visualization | CAD layers | `add_layer`, `assign_layer` |
| `profile` | Geometry | Cross-section profiles | `add_parameterised_profile`, `add_arbitrary_profile` |
| `cost` | Management | Cost scheduling | `add_cost_schedule`, `add_cost_item` |
| `sequence` | Management | Task scheduling (4D) | `add_work_schedule`, `add_task`, `edit_task_time` |
| `resource` | Management | Resource allocation | `add_resource`, `assign_resource` |
| `system` | MEP | Building systems | `add_system`, `assign_system` |
| `structural` | Engineering | Structural analysis | `add_structural_analysis_model`, `add_structural_member` |
| `document` | References | Document management | `add_information`, `assign_document` |
| `constraint` | Design | Design constraints | `add_objective`, `add_metric`, `assign_constraint` |
| `nest` | Organization | Nesting relationships | `assign_object`, `unassign_object` |
| `boundary` | Spatial | Space boundaries | `assign_connection_geometry` |
| `library` | References | Library references | `add_library`, `add_reference`, `assign_reference` |
| `drawing` | Visualization | Drawings/annotations | `add_drawing`, `add_annotation` |
| `georeference` | Location | Map coordinates | set coordinates, define projections |
| `grid` | Planning | Architectural grids | create, add axis |
| `feature` | Geometry | Geometric features | `add_feature` |
| `alignment` | Infrastructure | Road/rail alignment | `add_alignment` |
| `cogo` | Geometry | Coordinate geometry | calculate, transform |
| `pset_template` | Standards | Property set templates | create, assign |
| `control` | Relations | Control relationships | assign, establish |

---

## Common Operations

### Edit Attributes Directly

```python
# IfcOpenShell: all schema versions
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "Wall 001", "Description": "External bearing wall"})
```

### Create and Assign Opening

```python
# IfcOpenShell: all schema versions
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")
ifcopenshell.api.run("void.add_opening", model, opening=opening, element=wall)

# Optionally fill with door
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
ifcopenshell.api.run("void.add_filling", model, opening=opening, element=door)
```

### Assign Classification

```python
# IfcOpenShell: all schema versions
classification = ifcopenshell.api.run("classification.add_classification", model,
    classification="Uniclass 2015")
ifcopenshell.api.run("classification.add_reference", model,
    products=[wall], identification="Ss_20_10_30",
    name="Walls", classification=classification)
```

### Add Visual Style

```python
# IfcOpenShell: all schema versions
style = ifcopenshell.api.run("style.add_style", model, name="Concrete Grey")
ifcopenshell.api.run("style.add_surface_style", model, style=style,
    attributes={"SurfaceColour": {"Red": 0.7, "Green": 0.7, "Blue": 0.7}})
ifcopenshell.api.run("style.assign_representation_styles", model,
    shape_representation=representation, styles=[style])
```

### Safe Element Removal

```python
# IfcOpenShell: all schema versions
# ALWAYS use root.remove_product: it cleans up ALL relationships
ifcopenshell.api.run("root.remove_product", model, product=wall)
# NEVER use model.remove(wall) for products: it leaves dangling references
```

---

## Parameter Conventions

### List Parameters (v0.8+ Breaking Change)

Since IfcOpenShell v0.8, relationship functions use **list** parameters:

```python
# CORRECT (v0.8+): products is a list
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# WRONG: single element (raises TypeError in v0.8+)
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)  # KeyError: 'product'
```

Functions affected: `spatial.assign_container`, `aggregate.assign_object`, `type.assign_type`, `material.assign_material`, `group.assign_group`, `classification.add_reference`, `nest.assign_object`, and others.

### Property Type Mapping (pset.edit_pset)

Python types map automatically to IFC property types:

| Python Type | IFC Type | Example |
|-------------|----------|---------|
| `str` | IfcLabel | `"REI60"` |
| `float` | IfcReal | `0.24` |
| `int` | IfcInteger | `3` |
| `bool` | IfcBoolean | `True` |
| `None` | (deletes property) | `None` |

### Matrix Convention (geometry.edit_object_placement)

Placement uses a 4x4 NumPy transformation matrix:

```python
import numpy
# Identity matrix = origin, no rotation
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.eye(4))
```

**ALWAYS** set `is_si=True` (default) when providing coordinates in meters. Set `is_si=False` only when coordinates match the file's native unit system.

---

## Version Notes

### Schema-Specific Entities

| Entity | IFC2X3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| IfcBuildingStorey | Yes | Yes | Yes |
| IfcSpace | Yes | Yes | Yes |
| IfcFacility | No | No | Yes |
| IfcFacilityPart | No | No | Yes |
| IfcAlignment | No | No | Yes |
| IfcBridge | No | No | Yes |
| IfcRoad | No | No | Yes |

`api.run()` handles most schema differences internally. When creating IFC4X3 infrastructure models, use the infrastructure-specific entity classes.

### project.create_file Parameter

```python
# CORRECT: parameter is "version" (not "schema")
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# WRONG: "schema" is for ifcopenshell.file(), not project.create_file
model = ifcopenshell.api.run("project.create_file", schema="IFC4")  # Unexpected kwarg
```

---

## Dependency

This skill depends on **ifcos-syntax-fileio** for file creation, opening, writing, and transaction management patterns. Use ifcos-syntax-fileio for:
- `ifcopenshell.open()` / `ifcopenshell.file()`
- `model.write()` / `model.to_string()`
- Transaction management (`begin_transaction` / `end_transaction` / `undo` / `redo`)

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all 35 API modules
- [Working Code Examples](references/examples.md) — End-to-end examples per domain
- [Anti-Patterns](references/anti-patterns.md) — Common API mistakes and how to avoid them
