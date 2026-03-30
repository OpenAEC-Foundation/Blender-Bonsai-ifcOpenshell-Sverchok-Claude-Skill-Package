---
name: bonsai-impl-modeling
description: >
  Use when placing building elements (walls, slabs, columns, beams) in Bonsai or assigning
  IFC types and materials. Prevents the common mistake of creating geometry without assigning
  it to the spatial hierarchy (orphaned elements). Covers the complete element creation
  pipeline: type selection, geometric representation, material layer/profile/constituent
  assignment, and spatial containment.
  Keywords: wall, slab, column, beam, IfcWall, IfcSlab, IFC type, predefined type, material layer, material profile, spatial assignment, BIM modeling, make a wall, create building elements, add floor.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Implementation: BIM Modeling

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.bim.module.model` — NEVER `blenderbim.*`
> **Data layer**: All IFC operations use `ifcopenshell.api.run()`

## 1. Critical Warnings

1. **ALWAYS use `bonsai.*`** — NEVER `blenderbim.*`. The rename occurred in 2024 (v0.8.0).
2. **ALWAYS check `IfcStore.get_file()` for `None`** before any IFC operation.
3. **ALWAYS verify representation context exists** before creating geometry.
4. **ALWAYS use `spatial.assign_container`** for placing elements in storeys — NEVER `aggregate.assign_object`.
5. **ALWAYS use `feature.add_feature`** for openings — NEVER `void.add_opening` (does not exist in v0.8.0+).
6. **ALWAYS pass lists** to `related_objects` / `products` parameters — NEVER single entities.
7. **ALWAYS call `bpy.ops.bim.edit_object_placement()`** after moving Blender objects to sync IFC placement.
8. **NEVER treat Bonsai as an importer/exporter** — IFC IS the native document.

## 2. Element Creation Pipeline

### Decision Tree: Which Workflow?

```
Need to create a building element?
├── Is there an existing IfcElementType for it?
│   ├── YES → Use Type-Based Workflow (§2.1)
│   └── NO  → Create type first (§3), then use Type-Based Workflow
│
├── Element category?
│   ├── Wall         → Layer-based type (IfcMaterialLayerSet)     → §4.1
│   ├── Slab/Plate   → Layer-based type (IfcMaterialLayerSet)     → §4.1
│   ├── Column/Beam  → Profile-based type (IfcMaterialProfileSet) → §4.2
│   ├── Door/Window   → Parametric type (mapped geometry)         → §4.3
│   └── Custom/Other → Constituent set or direct mesh             → §4.4
```

### 2.1 Type-Based Element Creation (Standard Workflow)

Every building element in Bonsai follows this pipeline:

```
Step 1: Get IFC model          → IfcStore.get_file()
Step 2: Ensure context exists  → ifcopenshell.util.representation.get_context()
Step 3: Get/create type        → root.create_entity(ifc_class="IfcWallType")
Step 4: Configure type material → material.add_material_set() + material.assign_material()
Step 5: Create occurrence      → root.create_entity(ifc_class="IfcWall")
Step 6: Assign type            → type.assign_type(related_objects=[wall], relating_type=wall_type)
Step 7: Create representation  → geometry.add_wall_representation() / add_profile_representation()
Step 8: Assign representation  → geometry.assign_representation()
Step 9: Place in spatial tree  → spatial.assign_container(products=[wall], relating_structure=storey)
Step 10: Set placement         → geometry.edit_object_placement()
```

### 2.2 Representation Context Setup

ALWAYS verify context exists before creating geometry:

```python
import ifcopenshell.util.representation

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

if not body_context:
    model_ctx = ifcopenshell.api.run("context.add_context", model,
        context_type="Model")
    body_context = ifcopenshell.api.run("context.add_context", model,
        context_type="Model",
        context_identifier="Body",
        target_view="MODEL_VIEW",
        parent=model_ctx)
```

## 3. Type System

### 3.1 Creating Element Types

```python
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType",
    predefined_type="STANDARD",
    name="200mm Concrete Wall")
```

### 3.2 Type-to-Representation Template Mapping

| IFC Type Class    | Material Set Type         | Representation Template |
|-------------------|---------------------------|------------------------|
| `IfcWallType`     | `IfcMaterialLayerSet`     | `LAYERSET_AXIS2`       |
| `IfcSlabType`     | `IfcMaterialLayerSet`     | `LAYERSET_AXIS3`       |
| `IfcPlateType`    | `IfcMaterialLayerSet`     | `LAYERSET_AXIS3`       |
| `IfcColumnType`   | `IfcMaterialProfileSet`   | `PROFILESET`           |
| `IfcBeamType`     | `IfcMaterialProfileSet`   | `PROFILESET`           |
| `IfcMemberType`   | `IfcMaterialProfileSet`   | `PROFILESET`           |
| `IfcFootingType`  | `IfcMaterialProfileSet`   | `PROFILESET`           |
| `IfcDoorType`     | Direct/Constituent        | `DOOR`                 |
| `IfcWindowType`   | Direct/Constituent        | `WINDOW`               |
| `IfcFurnitureType`| Direct/Constituent        | `MESH`                 |

### 3.3 Predefined Type Enums

#### IfcWallTypeEnum
`STANDARD`, `SOLIDWALL`, `PARTITIONING`, `RETAININGWALL`, `SHEAR`, `PARAPET`, `USERDEFINED`, `NOTDEFINED`

#### IfcSlabTypeEnum
`FLOOR`, `ROOF`, `LANDING`, `BASESLAB`, `USERDEFINED`, `NOTDEFINED`

#### IfcColumnTypeEnum
`COLUMN`, `PILASTER`, `USERDEFINED`, `NOTDEFINED`

#### IfcBeamTypeEnum
`BEAM`, `JOIST`, `HOLLOWCORE`, `LINTEL`, `SPANDREL`, `T_BEAM`, `USERDEFINED`, `NOTDEFINED`

### 3.4 Type Assignment

```python
# Assign type to one or more occurrences
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2],   # ALWAYS a list
    relating_type=wall_type,
    should_map_representations=True)  # Inherits geometry via MappedItem

# Query type of an element
import ifcopenshell.util.element
element_type = ifcopenshell.util.element.get_type(wall1)

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(wall_type)
```

### 3.5 MappedItem Geometry Sharing

When `should_map_representations=True`:
- Geometry is stored ONCE on the type's `IfcRepresentationMap`
- Each occurrence references it via `IfcMappedItem` with a per-instance transformation
- Editing the type geometry updates ALL occurrences automatically

## 4. Material Assignment

### 4.1 Material Layer Sets (Walls, Slabs, Plates)

```python
# Step 1: Create materials
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C25", category="concrete")
insulation = ifcopenshell.api.run("material.add_material", model,
    name="Mineral Wool", category="ite")

# Step 2: Create layer set
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="200mm Insulated Wall", set_type="IfcMaterialLayerSet")

# Step 3: Add layers with thickness
layer_c = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer_c, attributes={"LayerThickness": 0.150})

layer_i = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer_i, attributes={"LayerThickness": 0.050})

# Step 4: Assign to type (NEVER to occurrence directly)
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=layer_set)
```

### 4.2 Material Profile Sets (Columns, Beams)

```python
# Step 1: Create material
steel = ifcopenshell.api.run("material.add_material", model,
    name="S355 Steel", category="steel")

# Step 2: Create profile definition
profile = model.create_entity("IfcIShapeProfileDef",
    ProfileType="AREA", ProfileName="HEA200",
    OverallWidth=0.200, OverallDepth=0.190,
    WebThickness=0.0065, FlangeThickness=0.010)

# Step 3: Create profile set and add profile
profile_set = ifcopenshell.api.run("material.add_material_set", model,
    name="HEA 200 Steel", set_type="IfcMaterialProfileSet")
ifcopenshell.api.run("material.add_profile", model,
    profile_set=profile_set, material=steel, profile=profile)

# Step 4: Assign to type
ifcopenshell.api.run("material.assign_material", model,
    products=[column_type], material=profile_set)
```

### 4.3 Material Constituent Sets (Doors, Windows, Custom)

```python
constituent_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Door Materials", set_type="IfcMaterialConstituentSet")

wood = ifcopenshell.api.run("material.add_material", model,
    name="Oak", category="wood")
glass = ifcopenshell.api.run("material.add_material", model,
    name="Float Glass", category="glass")

ifcopenshell.api.run("material.add_constituent", model,
    constituent_set=constituent_set, material=wood, name="Frame")
ifcopenshell.api.run("material.add_constituent", model,
    constituent_set=constituent_set, material=glass, name="Glazing")

ifcopenshell.api.run("material.assign_material", model,
    products=[door_type], material=constituent_set)
```

### 4.4 Material Assignment Decision Tree

```
What element type?
├── Wall / Slab / Plate → IfcMaterialLayerSet      → assign to TYPE
├── Column / Beam       → IfcMaterialProfileSet     → assign to TYPE
├── Door / Window       → IfcMaterialConstituentSet  → assign to TYPE
├── Generic element     → IfcMaterial (single)       → assign to TYPE or occurrence
└── Composite           → IfcMaterialConstituentSet  → assign to TYPE
```

**Rule**: ALWAYS assign materials to the TYPE, not to individual occurrences. Occurrences inherit materials from their type via `IfcMaterialLayerSetUsage` or `IfcMaterialProfileSetUsage`.

## 5. Geometry Creation

### 5.1 Wall Geometry

```python
representation = ifcopenshell.api.run(
    "geometry.add_wall_representation", model,
    context=body_context,
    length=5.0,       # meters
    height=3.0,       # meters
    thickness=0.2,    # meters
    offset=0.0,
    x_angle=0.0)      # radians, for sloped walls

ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
```

### 5.2 Slab Geometry

```python
representation = ifcopenshell.api.run(
    "geometry.add_slab_representation", model,
    context=body_context,
    depth=0.2,
    polyline=[(0.0, 0.0), (10.0, 0.0), (10.0, 8.0), (0.0, 8.0)])

ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=representation)
```

### 5.3 Profile-Based Geometry (Columns, Beams)

```python
representation = ifcopenshell.api.run(
    "geometry.add_profile_representation", model,
    context=body_context,
    profile=profile_def,    # IfcProfileDef entity
    depth=3.0,              # extrusion length
    cardinal_point=5)       # mid-depth centre

ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
```

#### Cardinal Point Reference

| Value | Position          |
|-------|-------------------|
| 1     | Bottom left       |
| 2     | Bottom centre     |
| 3     | Bottom right      |
| 4     | Mid-depth left    |
| 5     | Mid-depth centre  |
| 6     | Mid-depth right   |
| 7     | Top left          |
| 8     | Top centre        |
| 9     | Top right         |
| 10    | Geometric centroid|

### 5.4 Common Profile Definitions

```python
# Rectangle (columns, beams)
rect_profile = model.create_entity("IfcRectangleProfileDef",
    ProfileType="AREA", ProfileName="400x400",
    XDim=0.4, YDim=0.4)

# I-Shape (steel beams/columns)
i_profile = model.create_entity("IfcIShapeProfileDef",
    ProfileType="AREA", ProfileName="HEA200",
    OverallWidth=0.200, OverallDepth=0.190,
    WebThickness=0.0065, FlangeThickness=0.010)

# Circle (pipes, round columns)
circle_profile = model.create_entity("IfcCircleProfileDef",
    ProfileType="AREA", ProfileName="D300",
    Radius=0.15)

# Or use the API for parameterized profiles
profile = ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class="IfcRectangleProfileDef")
```

## 6. Element Placement

### 6.1 Setting Object Placement

```python
import numpy as np

matrix = np.array([
    [1.0, 0.0, 0.0, 5.0],   # X translation = 5m
    [0.0, 1.0, 0.0, 0.0],   # Y translation = 0m
    [0.0, 0.0, 1.0, 3.0],   # Z translation = 3m (e.g., storey offset)
    [0.0, 0.0, 0.0, 1.0]
], dtype=float)

ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix, is_si=True)
```

### 6.2 Spatial Container Assignment

```python
storey = model.by_type("IfcBuildingStorey")[0]

ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall, column, slab],   # ALWAYS a list
    relating_structure=storey)
```

## 7. Opening and Void Operations

```python
# Step 1: Create the opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Opening-001")

# Step 2: Create opening geometry
opening_repr = ifcopenshell.api.run(
    "geometry.add_wall_representation", model,
    context=body_context,
    length=1.0, height=2.1, thickness=0.25)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_repr)

# Step 3: Add opening to host wall (creates IfcRelVoidsElement)
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)

# Step 4: Fill with door (creates IfcRelFillsElement)
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)
```

## 8. Parametric Arrays (Bonsai-Specific)

Bonsai uses `BBIM_Array` property set for parametric element arrays:

```python
# Array data structure stored in EPset_Parametric
array_data = {
    "children": [],          # GlobalIds of child copies
    "count": 5,              # Number of copies
    "x": 3.0,               # X offset between copies (meters)
    "y": 0.0,               # Y offset
    "z": 0.0,               # Z offset
    "use_local_space": True, # Offset relative to element's local axes
    "method": "OFFSET"       # Offset method
}
```

## 9. Bonsai Operator Quick Reference

| Operator                              | Purpose                           |
|---------------------------------------|-----------------------------------|
| `bpy.ops.bim.assign_class()`         | Assign IFC class to Blender object |
| `bpy.ops.bim.edit_object_placement()`| Sync Blender transform to IFC     |
| `bpy.ops.bim.save_project()`         | Save IFC file                     |
| `bpy.ops.bim.add_type_instance()`    | Place instance of existing type   |

**Rule**: ALWAYS prefer `ifcopenshell.api.run()` over `bpy.ops.bim.*` in scripts — the API is more stable and explicit.

## 10. Reference Links

- **API Signatures**: See `references/methods.md`
- **Complete Code Examples**: See `references/examples.md`
- **Anti-Patterns**: See `references/anti-patterns.md`
