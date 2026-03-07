---
name: ifcos-impl-creation
description: "Guides IFC model creation workflows using ifcopenshell.api including creating projects, spatial structure (site/building/storey), walls, slabs, columns, openings, property sets, and type assignments. Covers IFC2X3 OwnerHistory requirements. Activates when building IFC models from scratch, creating BIM elements, or setting up spatial hierarchies."
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Model Creation Workflows

## Quick Reference

### Decision Tree: Starting a New IFC Model

```
Creating a new IFC model?
├── Production model with proper header/metadata?
│   └── YES → model = ifcopenshell.api.run("project.create_file", version="IFC4")
│
└── Bare-minimum test file?
    └── model = ifcopenshell.file(schema="IFC4")
        └── WARNING: No header, no project, no units. Add these manually.
```

### Decision Tree: Choosing IFC Schema Version

```
Which schema version?
├── Building/architecture project?
│   ├── Legacy compatibility needed? → IFC2X3
│   │   └── WARNING: OwnerHistory is REQUIRED on all rooted entities
│   └── Modern project? → IFC4
│
├── Infrastructure project (roads, bridges, tunnels)?
│   └── IFC4X3
│       └── Adds: IfcBridge, IfcRoad, IfcTunnel, IfcAlignment
│
└── Unsure? → Default to IFC4
```

### Decision Tree: Element Creation Order

```
Building an IFC model from scratch?
Follow this EXACT order:

1. Create file         → project.create_file
2. Create IfcProject   → root.create_entity
3. Assign units        → unit.assign_unit
4. Add contexts        → context.add_context (Model + Body subcontext)
5. Create spatial hierarchy:
   a. IfcSite           → root.create_entity + aggregate.assign_object
   b. IfcBuilding       → root.create_entity + aggregate.assign_object
   c. IfcBuildingStorey  → root.create_entity + aggregate.assign_object
6. Create types (optional but recommended):
   a. IfcWallType etc.  → root.create_entity
   b. Assign materials  → material.add_material_set + material.assign_material
7. Create elements:
   a. Create entity     → root.create_entity
   b. Add geometry      → geometry.add_wall_representation (etc.)
   c. Assign geometry   → geometry.assign_representation
   d. Set placement     → geometry.edit_object_placement
   e. Assign container  → spatial.assign_container
   f. Assign type       → type.assign_type
8. Add properties       → pset.add_pset + pset.edit_pset
9. Write file          → model.write("output.ifc")
```

### Critical Warnings

- **ALWAYS** use `ifcopenshell.api.run()` for entity creation. NEVER use `model.create_entity()` directly — the API handles GlobalId generation, OwnerHistory, and validation automatically.
- **ALWAYS** create the spatial hierarchy (Project → Site → Building → Storey) before creating physical elements. Elements without spatial containment are invisible in most BIM viewers.
- **ALWAYS** assign spatial containment via `spatial.assign_container` for physical elements (walls, slabs, columns, doors). Without containment, elements float in the model with no spatial context.
- **ALWAYS** set up geometric contexts (Model → Body subcontext) before creating geometry. Geometry without a context cannot be displayed.
- **ALWAYS** assign units via `unit.assign_unit` immediately after creating the project. Without units, all dimensions are ambiguous.
- **NEVER** skip `geometry.edit_object_placement` after creating an element. Without a placement, the element has no position in 3D space.
- **NEVER** create physical elements and types with mismatched IFC classes. An IfcWall occurrence MUST use IfcWallType, not IfcSlabType.
- **IFC2X3 ONLY**: OwnerHistory is REQUIRED on every rooted entity. The `root.create_entity` API handles this automatically, but ONLY if an owner/user has been set via `owner.set_user`. Call `owner.set_user` before creating any entities in IFC2X3.

---

## Essential Patterns

### Pattern 1: Complete IFC4 Model from Scratch

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

# Step 1: Create file with proper header
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Step 2: Create project
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")

# Step 3: Assign SI units (meters, radians)
ifcopenshell.api.run("unit.assign_unit", model)

# Step 4: Set up geometric context
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Step 5: Create spatial hierarchy
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

# Step 6: Create wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
representation = ifcopenshell.api.run("geometry.add_wall_representation",
    model, context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# Step 7: Add properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90"
})

# Step 8: Write
model.write("output.ifc")
```

### Pattern 2: IFC2X3 Model with Required OwnerHistory

```python
# IfcOpenShell — IFC2X3
import ifcopenshell
import ifcopenshell.api

# Create IFC2X3 file
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")

# REQUIRED for IFC2X3: Set up owner BEFORE creating any entities
person = ifcopenshell.api.run("owner.add_person", model,
    identification="jdoe", family_name="Doe", given_name="John")
org = ifcopenshell.api.run("owner.add_organisation", model,
    identification="ACME", name="ACME Engineering")
user = ifcopenshell.api.run("owner.add_person_and_organisation", model,
    person=person, organisation=org)
application = ifcopenshell.api.run("owner.add_application", model)
ifcopenshell.api.run("owner.set_user", model, user=user)

# Now create entities — OwnerHistory is automatically attached
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Legacy Project")
ifcopenshell.api.run("unit.assign_unit", model)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
```

### Pattern 3: Spatial Hierarchy with Multiple Storeys

```python
# IfcOpenShell — IFC4
# After project, units, and contexts are set up:
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Main Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Block")

# Create multiple storeys
basement = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Basement")
ground = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
first = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")
roof = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Roof")

# Aggregate: Project → Site → Building → all storeys
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[basement, ground, first, roof], relating_object=building)

# Set storey elevations using object placement
import numpy as np
for storey_obj, elevation in [(basement, -3.0), (ground, 0.0),
                               (first, 3.5), (roof, 7.0)]:
    matrix = np.eye(4)
    matrix[2][3] = elevation  # Z translation = elevation
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=storey_obj, matrix=matrix)
```

### Pattern 4: Creating Elements with Type Assignment

```python
# IfcOpenShell — IFC4
# Create a wall type with material layers
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="WT-200 External")

# Set up material layers for the type
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="WT-200", set_type="IfcMaterialLayerSet")
brick = ifcopenshell.api.run("material.add_material", model,
    name="Brick", category="brick")
insulation = ifcopenshell.api.run("material.add_material", model,
    name="Insulation", category="mineral wool")
plaster = ifcopenshell.api.run("material.add_material", model,
    name="Gypsum Plaster", category="gypsum")

layer1 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=brick)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer1, attributes={"LayerThickness": 0.102})
layer2 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer2, attributes={"LayerThickness": 0.080})
layer3 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=plaster)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer3, attributes={"LayerThickness": 0.018})

ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)

# Create wall occurrences and assign the type
wall1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 002")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)
```

### Pattern 5: Creating Openings (Doors and Windows)

```python
# IfcOpenShell — IFC4
# Prerequisites: wall exists, body context exists, storey exists

# Create opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")
opening_rep = ifcopenshell.api.run("geometry.add_wall_representation",
    model, context=body, length=0.9, height=2.1, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=opening)

# Cut opening in wall (boolean subtraction)
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Create door and fill the opening
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[door])
```

### Pattern 6: Property Sets and Quantity Sets

```python
# IfcOpenShell — IFC4
# Add standard property set to wall
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "Reference": "WT-200",
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI60",
    "ThermalTransmittance": 0.28
})

# Add quantity set
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall, name="Qto_WallBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 5.0,
    "Height": 3.0,
    "Width": 0.2,
    "GrossSideArea": 15.0,
    "NetSideArea": 13.11,
    "GrossVolume": 3.0
})

# Custom property set (use project-specific prefix, NOT "Pset_")
custom_pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="ACME_WallData")
ifcopenshell.api.run("pset.edit_pset", model, pset=custom_pset, properties={
    "CostCode": "STR-W-001",
    "InstallDate": "2026-03-15",
    "Inspector": "J. Doe"
})
```

### Pattern 7: Slab and Column Creation

```python
# IfcOpenShell — IFC4
# Slab with polyline footprint
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Floor Slab", predefined_type="FLOOR")
slab_rep = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context=body, depth=0.25,
    polyline=[(0.0, 0.0), (10.0, 0.0), (10.0, 8.0), (0.0, 8.0)])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=slab_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=slab)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[slab])

# Column with profile
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")
profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcRectangleProfileDef", XDim=0.3, YDim=0.3)
col_rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=col_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=column)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[column])
```

---

## Version Notes

### IFC2X3 vs IFC4+ Entity Differences

| Feature | IFC2X3 | IFC4 / IFC4X3 |
|---------|--------|----------------|
| OwnerHistory | REQUIRED on all rooted entities | OPTIONAL |
| IfcWallStandardCase | EXISTS (use for layered walls) | DEPRECATED (use IfcWall with PredefinedType) |
| IfcBuildingStorey | Only valid spatial container | IfcBuildingStorey + IfcSpace + IfcFacility |
| Property set types | IfcPropertySingleValue only | Adds IfcPropertyBoundedValue, IfcPropertyTableValue |
| Material sets | IfcMaterialList (legacy) | IfcMaterialConstituentSet (preferred) |

### IFC4X3 Infrastructure Entities (Not Available in IFC2X3/IFC4)

- `IfcBridge`, `IfcBridgePart`
- `IfcRoad`, `IfcRoadPart`
- `IfcRailway`, `IfcRailwayPart`
- `IfcTunnel`, `IfcTunnelPart`
- `IfcAlignment`, `IfcLinearElement`
- `IfcFacility`, `IfcFacilityPart`

### Aggregation Hierarchy by Schema

| Schema | Standard Hierarchy |
|--------|-------------------|
| IFC2X3 | IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace |
| IFC4 | IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace |
| IFC4X3 | IfcProject → IfcSite → IfcFacility (or IfcBuilding/IfcBridge/IfcRoad) → IfcFacilityPart (or IfcBuildingStorey) → IfcSpace |

### Type-to-Occurrence Mapping (Common Elements)

| Element Type | Occurrence Class | Notes |
|-------------|-----------------|-------|
| IfcWallType | IfcWall | IFC2X3: also IfcWallStandardCase |
| IfcSlabType | IfcSlab | Use predefined_type: FLOOR, ROOF, BASESLAB, LANDING |
| IfcColumnType | IfcColumn | |
| IfcBeamType | IfcBeam | |
| IfcDoorType | IfcDoor | IFC2X3: IfcDoorStyle (not IfcDoorType) |
| IfcWindowType | IfcWindow | IFC2X3: IfcWindowStyle (not IfcWindowType) |
| IfcCurtainWallType | IfcCurtainWall | |

---

## Common Operations

### Set Up Geometric Contexts

```python
# IfcOpenShell — IFC4
# Root context (REQUIRED before any subcontexts)
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")

# Body subcontext (REQUIRED for 3D solid geometry)
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Optional: Axis subcontext (for wall/beam centerlines)
axis = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Axis",
    target_view="GRAPH_VIEW", parent=model3d)

# Optional: Plan context (for 2D representations)
plan = ifcopenshell.api.run("context.add_context", model,
    context_type="Plan")
plan_body = ifcopenshell.api.run("context.add_context", model,
    context_type="Plan", context_identifier="Body",
    target_view="PLAN_VIEW", parent=plan)
```

### Position Elements with Transformation Matrix

```python
# IfcOpenShell — IFC4
import numpy as np

# Place wall at specific X, Y, Z coordinates
matrix = np.eye(4)
matrix[0][3] = 5.0   # X offset = 5 meters
matrix[1][3] = 2.0   # Y offset = 2 meters
matrix[2][3] = 0.0   # Z offset = 0 meters (ground level)

ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix)

# Rotate element 90 degrees around Z axis
import math
angle = math.radians(90)
rotation_matrix = np.array([
    [math.cos(angle), -math.sin(angle), 0, 0],
    [math.sin(angle),  math.cos(angle), 0, 0],
    [0,                0,               1, 0],
    [0,                0,               0, 1]
])
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=rotation_matrix)
```

### Shared Property Sets Across Multiple Elements

```python
# IfcOpenShell — IFC4
# Create pset on first element
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall1, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True, "FireRating": "REI90"
})

# Share same pset with additional elements
ifcopenshell.api.run("pset.assign_pset", model,
    products=[wall1, wall2, wall3], pset=pset)
```

---

## Visual Alternative: IfcSverchok

For node-based visual IFC creation without writing Python scripts, use **IfcSverchok** — a 31-node bridge inside Sverchok that provides the same IFC creation pipeline as a visual node tree:

- `SvIfcCreateProject` → equivalent to `project.create_file` + `root.create_entity(IfcProject)`
- `SvIfcCreateEntity` → equivalent to `root.create_entity`
- `SvIfcApi` → visual wrapper around any `ifcopenshell.api.run()` call
- Two geometry conversion modes: `SvIfcBMeshToIfcRepr` (Blender mesh) and `SvIfcSverchokToIfcRepr` (Sverchok geometry)
- Enable `use_bonsai_file` toggle to operate on Bonsai's active IFC file

Refer to: `sverchok-impl-ifcsverchok`

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all creation-related API methods
- [Working Code Examples](references/examples.md) — End-to-end model creation examples
- [Anti-Patterns](references/anti-patterns.md) — Common creation mistakes and how to avoid them
