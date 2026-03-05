# Bonsai BIM — Comprehensive Pre-Research Document

> **Target version**: Bonsai v0.8.4 | Minimum Blender: 4.2.0 | Python 3.11
> **Date**: 2026-03-05
> **Formerly**: BlenderBIM (renamed 2024)
> **Module path**: `bonsai.bim.ifc` — NEVER `blenderbim.bim.ifc`

---

## Table of Contents

1. [Bonsai Overview](#1-bonsai-overview)
2. [Architecture](#2-architecture)
3. [IFC Integration](#3-ifc-integration)
4. [Spatial Structure Management](#4-spatial-structure-management)
5. [Property Sets and Quantity Sets](#5-property-sets-and-quantity-sets)
6. [Type System](#6-type-system)
7. [Modeling Workflow](#7-modeling-workflow)
8. [Classification Systems](#8-classification-systems)
9. [Geometry Representations](#9-geometry-representations)
10. [IFC Export Pipeline (Saving)](#10-ifc-export-pipeline-saving)
11. [Common Error Patterns](#11-common-error-patterns)
12. [Bonsai Python API Reference](#12-bonsai-python-api-reference)
13. [AI Common Mistakes](#ai-common-mistakes)
14. [Sources](#sources)

---

## 1. Bonsai Overview

### What Is Bonsai?

**Bonsai** is a free, open-source Blender add-on for native IFC (Industry Foundation Classes) authoring. It operates as an OpenBIM platform directly within Blender's 3D viewport, enabling architects, engineers, and BIM professionals to create, analyse, and modify fully standards-compliant IFC models without a proprietary intermediate format.

Bonsai was formerly known as the **BlenderBIM Add-on**. The rename to Bonsai occurred in **2024**, with the v0.8.0 release (published 1 September 2024) being the first release under the new name. The project identity is otherwise unchanged — same codebase, same team, same philosophy.

### Repository and Monorepo Context

Bonsai lives inside the **IfcOpenShell monorepo** on GitHub:

```
https://github.com/IfcOpenShell/IfcOpenShell
└── src/
    └── bonsai/          ← Bonsai add-on source root
        └── bonsai/
            ├── __init__.py
            ├── core/
            ├── tool/
            ├── bim/
            ├── libs/
            └── wheels/
```

This co-location means Bonsai is always developed and released in lock-step with IfcOpenShell-python, which it depends on for all IFC data operations.

### Lead Developer and Community

- **Lead developer**: Dion Moult
- **License**: LGPL-3.0 / GPL-2.0 (open source, community-driven)
- **Governance**: volunteer contributors via GitHub, OSArch community forums, and OSArch chat
- **Funding**: ecosystem grants and collective support (Open Collective)

### Version Information

| Property | Value |
|---|---|
| Current version | **v0.8.4** |
| Minimum Blender version | **4.2.0** (LTS; added in v0.8.0) |
| Python version | 3.11 (Blender 4.2–4.3) |
| Platforms | Linux 64-bit, macOS Intel/Silicon, Windows 64-bit |

> **Note**: The v0.8.0 release introduced Blender 4.2 support and was the first release under the Bonsai name. Blender 4.2 is therefore the minimum supported version for all v0.8.x releases.

### Key Features

1. **Native IFC authoring**: the IFC file is the document — no import/export translation step.
2. **IfcOpenShell integration**: all IFC data operations delegate to IfcOpenShell-python.
3. **3D viewport BIM**: full BIM authoring inside Blender's 3D viewport with spatial context, property panels, and type libraries.
4. **Automated drawing generation**: 2D drawing extraction from IFC geometry.
5. **MEP, structural, and GIS**: mechanical/electrical/plumbing systems, structural analysis integration, and georeferencing tools.
6. **Scheduling, costing, and FM**: project sequencing (4D), quantity take-off (5D), and facility management (6D).
7. **BCF support**: Building Collaboration Format issue tracking.
8. **Clash detection**: federation and multi-discipline conflict checking.

---

## 2. Architecture

### Three-Layer Architecture

Bonsai is structured as three conceptually distinct layers, each with a well-defined responsibility:

```
┌─────────────────────────────────────────────────────┐
│  DELIVERY LAYER  — bim/module/                      │
│  Blender UI: operators, properties, panels, gizmos  │
├─────────────────────────────────────────────────────┤
│  DOMAIN LAYER   — core/ + tool/                     │
│  Abstract use-case logic (core) + concrete impls    │
│  (tool), injected via @interface contract           │
├─────────────────────────────────────────────────────┤
│  DATA LAYER     — IfcStore + IFC in-memory graph    │
│  Live ifcopenshell.file instance, id_map, guid_map  │
└─────────────────────────────────────────────────────┘
```

The delivery layer executes use-cases by calling into the domain layer through **dependency injection**: operators pass concrete `tool.*` implementations to `core.*` functions. The core never directly imports Blender (`bpy`); this keeps business logic independently testable.

### Tool / Core / UI Module Separation Pattern

Every functional area follows the same three-part split:

| Sub-layer | Location | Responsibility |
|---|---|---|
| **UI** (delivery) | `bim/module/<name>/ui.py`, `operator.py`, `prop.py` | Blender panels, operator classes, custom properties |
| **Core** (abstract domain) | `core/<name>.py` | High-level use-case flow; no Blender imports |
| **Tool** (concrete domain) | `tool/<name>.py` | Blender-specific implementation; scene, file I/O, IFC graph |

Example call flow for a typical operator:

```python
# bim/module/spatial/operator.py  (Delivery Layer)
class BIM_OT_assign_container(Operator):
    def execute(self, context):
        return IfcStore.execute_ifc_operator(self, context)

    def _execute(self, context):
        core.spatial.assign_container(
            tool.Ifc,        # injected concrete tool
            tool.Spatial,
            structure=self.structure,
        )
```

```python
# core/spatial.py  (Domain Layer — no bpy imports)
def assign_container(ifc, spatial, structure):
    element = ifc.get_entity(bpy.context.active_object)
    spatial.assign_container(element, structure)
    ifc.run("aggregate.assign_object", ...)
```

### The `@interface` Decorator (`core/tool.py`)

`core/tool.py` defines every tool interface as a class decorated with `@interface`. This decorator acts as a **design-by-contract enforcement mechanism**: it prevents any concrete tool class from being used unless it implements all required methods.

```python
# bonsai/core/tool.py (simplified)
def interface(cls):
    """Marks a class as a tool interface contract."""
    # Wraps all methods with a NotImplementedError guard
    # so that concrete implementations must override them.
    return cls

@interface
class Ifc:
    def get(self): ...
    def set(self, ifc): ...
    def run(self, command, **kwargs): ...
    def get_entity(self, obj): ...
    def get_object(self, element): ...
    def link(self, element, obj): ...

@interface
class Spatial:
    def assign_container(self, element, structure): ...
```

The concrete implementations live in `tool/ifc.py`, `tool/spatial.py`, etc. and are imported by operators as `tool.Ifc`, `tool.Spatial`, and so on.

### Registered Modules

Bonsai registers approximately **51–56 functional modules** (exact count varies across patch releases). Each module corresponds to a distinct IFC domain area. As of v0.8.0 the `bim/module/` directory contains:

```
aggregate       alignment       attribute       bcf
boundary        brick           bsdd            cad
clash           classification  constraint      context
cost            covering        csv             debug
demo            diff            document        drawing
fm              geometry        georeference    gis
ifcgit          layer           library         material
nest            owner           patch           profile
project         pset            pset_template   qto
resource        search          sequence        spatial
structural      style           system          tester
type            unit            void            web
```

### Demo Module (Reference Implementation)

The `demo` module (`bim/module/demo/`) is a heavily commented **reference implementation** intended for new contributors. It demonstrates the full module pattern including all optional files.

### Module File Structure

A complete module may contain any subset of:

| File | Purpose |
|---|---|
| `__init__.py` | Package init, imports sub-modules |
| `operator.py` | `bpy.types.Operator` subclasses |
| `prop.py` | `bpy.types.PropertyGroup` subclasses |
| `ui.py` | Panel `draw()` methods, menu contributions |
| `data.py` | Cached/computed UI data (refresh patterns) |
| `decorator.py` | `bpy.types.SpaceView3D` draw callbacks (overlays) |
| `gizmo.py` | `bpy.types.Gizmo` / `GizmoGroup` subclasses |

### Blender Registration

```python
# bonsai/__init__.py (schematic)
bl_info = {
    "name": "Bonsai",
    "version": (0, 8, 4),
    "blender": (4, 2, 0),
    "category": "System",
}

def register():
    register_classes(classes)
    for mod in modules.values():
        mod.register()
    bpy.types.Scene.BIMProperties = PointerProperty(type=BIMProperties)

def unregister():
    for mod in reversed(list(modules.values())):
        mod.unregister()
    unregister_classes(classes)
```

---

## 3. IFC Integration

### Native IFC Authoring Philosophy

Bonsai's central design principle is **"the IFC file IS the document"**. Unlike traditional BIM tools that maintain a proprietary internal model and export to IFC on demand, Bonsai works directly with an `ifcopenshell.file` instance held in memory at all times:

- Every object created in the Blender scene is simultaneously an IFC entity.
- Geometry, properties, relationships, and metadata are written directly to the IFC graph.
- There is **no "export to IFC" step**; saving the project writes the live IFC file to disk.
- There is **no "import from IFC" step**; opening a project loads the IFC file directly into memory.

### IfcStore — Central State Manager

`IfcStore` (in `bonsai/bim/ifc.py`) is a **static class** that acts as the single source of truth for all IFC state within a Blender session:

```python
from bonsai.bim.ifc import IfcStore

# Key attributes:
IfcStore.path      # str  — absolute path to the .ifc file on disk
IfcStore.file      # ifcopenshell.file | None — in-memory IFC instance
IfcStore.schema    # schema definition string — "IFC2X3", "IFC4", or "IFC4X3"
IfcStore.id_map    # dict — IFC id → Blender object
IfcStore.guid_map  # dict — GlobalId → Blender object
```

### `tool.Ifc` — High-Level IFC Tool API

#### `tool.Ifc.get()`

Returns the live `ifcopenshell.file` instance. Returns `None` if no IFC file is loaded.

```python
import bonsai.tool as tool

ifc_file = tool.Ifc.get()
if ifc_file:
    walls = ifc_file.by_type("IfcWall")
```

#### `tool.Ifc.set(ifc)`

Sets the active IFC file and triggers post-load hooks:

```python
import ifcopenshell
import bonsai.tool as tool

new_file = ifcopenshell.open("path/to/model.ifc")
tool.Ifc.set(new_file)
```

#### `tool.Ifc.run(command, **kwargs)`

Executes an `ifcopenshell.api` command against the active IFC file. This is the **preferred way to modify IFC data**:

```python
# Create a new IfcWall
wall = tool.Ifc.run(
    "root.create_entity",
    ifc_class="IfcWall",
    name="EXT-WALL-01",
)

# Assign it to a spatial container
tool.Ifc.run(
    "spatial.assign_container",
    relating_structure=storey,
    product=wall,
)
```

#### `tool.Ifc.get_entity(obj)`

Given a Blender `bpy.types.Object`, returns the corresponding `ifcopenshell.entity_instance`, or `None` if the object is not linked to IFC:

```python
import bpy
import bonsai.tool as tool

obj = bpy.context.active_object
entity = tool.Ifc.get_entity(obj)
if entity:
    print(entity.GlobalId)
    print(entity.is_a())     # e.g. "IfcWall"
```

#### `tool.Ifc.get_object(element)`

Given an `ifcopenshell.entity_instance`, returns the linked Blender object:

```python
wall_entity = ifc_file.by_guid("3mRi3zZPnErAj...")
obj = tool.Ifc.get_object(wall_entity)
if obj:
    obj.select_set(True)
```

#### `tool.Ifc.link(element, obj)`

Establishes the bidirectional link between an IFC entity and a Blender object:

```python
tool.Ifc.link(wall_entity, blender_obj)
# Now tool.Ifc.get_entity(blender_obj) == wall_entity
# And tool.Ifc.get_object(wall_entity)  == blender_obj
```

### Blender Object ↔ IFC Entity Synchronization

```
Blender Object                        IFC Entity
──────────────                        ──────────
BIMObjectProperties                   ifcopenshell.entity_instance
  .ifc_definition_id ──────────────→  .id()
                     ←────────────── IfcStore.id_map
  .name                               .Name
  Matrix (location/rotation/scale)    IfcLocalPlacement / IfcAxis2Placement3D
  Mesh geometry                       IfcShapeRepresentation
```

### Scripting in Blender Python Console

```python
from bonsai.bim.ifc import IfcStore

# Get file
ifc = IfcStore.get_file()

# Inspect store state
print(IfcStore.path)    # "/home/user/project/model.ifc"
print(IfcStore.schema)  # "IFC4"

# Query entities
for wall in ifc.by_type("IfcWall"):
    blender_obj = IfcStore.id_map.get(wall.id())
    if blender_obj:
        print(f"{wall.Name} → {blender_obj.name}")
```

> **CRITICAL**: ALWAYS import from `bonsai.bim.ifc`, NEVER from `blenderbim.bim.ifc`.

---

## 4. Spatial Structure Management

### IFC Spatial Hierarchy

The IFC standard defines a hierarchical spatial decomposition tree:

```
IfcProject
  └── IfcSite
        └── IfcBuilding
              └── IfcBuildingStorey
                    └── IfcSpace
```

- **IfcProject** — root of the IFC model; every valid IFC file must have exactly one
- **IfcSite** — geographic land parcel (optional but recommended)
- **IfcBuilding** — physical building structure
- **IfcBuildingStorey** — floor level (e.g., "Ground Floor", "Level 1")
- **IfcSpace** — discrete spatial zone within a storey (room, corridor, void)

### Key IFC Relationships

#### `IfcRelAggregates`

Decomposes spatial structure elements into a parent–child hierarchy.

| Relationship | Example |
|---|---|
| IfcProject → IfcSite | Site belongs to project |
| IfcSite → IfcBuilding | Building is on site |
| IfcBuilding → IfcBuildingStorey | Storey is part of building |
| IfcBuildingStorey → IfcSpace | Space is within storey |

#### `IfcRelContainedInSpatialStructure`

Places **physical elements** (walls, doors, columns) within a spatial zone. An element can only be **contained in one** spatial structure element at a time.

#### `IfcRelReferencedInSpatialStructure`

Non-hierarchical spatial reference — an element can **reference multiple** spatial structures simultaneously (e.g., multi-storey columns).

### Spatial Structure Relationship Summary

| Relationship | IFC Class | Direction | Cardinality | API Function |
|---|---|---|---|---|
| Spatial decomposition | `IfcRelAggregates` | parent → children | 1:N | `aggregate.assign_object` |
| Physical containment | `IfcRelContainedInSpatialStructure` | zone → elements | 1:N (exclusive) | `spatial.assign_container` |
| Spatial reference | `IfcRelReferencedInSpatialStructure` | zone → elements | M:N | `spatial.reference_structure` |

### Python API

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")

project  = model.by_type("IfcProject")[0]
site     = model.by_type("IfcSite")[0]
building = model.by_type("IfcBuilding")[0]
storey   = model.by_type("IfcBuildingStorey")[0]

# Build the spatial decomposition tree (IfcRelAggregates)
ifcopenshell.api.aggregate.assign_object(
    model, products=[site], relating_object=project)
ifcopenshell.api.aggregate.assign_object(
    model, products=[building], relating_object=site)
ifcopenshell.api.aggregate.assign_object(
    model, products=[storey], relating_object=building)

# Place a wall in the storey (IfcRelContainedInSpatialStructure)
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.spatial.assign_container(
    model,
    products=[wall],
    relating_structure=storey
)

# Multi-storey reference (IfcRelReferencedInSpatialStructure)
column = model.by_type("IfcColumn")[0]
level2 = model.by_type("IfcBuildingStorey")[1]
ifcopenshell.api.spatial.reference_structure(
    model,
    products=[column],
    relating_structure=level2
)

# Remove from container
ifcopenshell.api.spatial.unassign_container(
    model,
    products=[wall]
)
```

### Default Project Structure on New IFC File

When a user creates a new IFC project via **File → New IFC Project**, Bonsai automatically generates:

```
IfcProject  "My Project"
  └── IfcSite  "My Site"
        └── IfcBuilding  "My Building"
              └── IfcBuildingStorey  "My Storey"  (Elevation: 0.0)
```

### Bonsai UI

- **Container**: Properties panel → Object Properties → IFC Object → **Container**
- **Spatial Decomposition**: Properties panel → Project Overview → **Spatial Decomposition**

### Bonsai Source Code

| Component | Path |
|---|---|
| Tool class | `src/bonsai/bonsai/tool/spatial.py` |
| Core logic | `src/bonsai/bonsai/core/spatial.py` |
| Operators | `src/bonsai/bonsai/bim/module/spatial/operator.py` |
| UI panel | `src/bonsai/bonsai/bim/module/spatial/ui.py` |

---

## 5. Property Sets and Quantity Sets

### Overview and Naming Conventions

IFC uses **property sets** (`IfcPropertySet`) and **quantity sets** (`IfcElementQuantity`) to attach structured data to building elements. They are linked via `IfcRelDefinesByProperties`.

#### Prefix Conventions

| Prefix | Meaning | Example |
|---|---|---|
| `Pset_` | buildingSMART international standard | `Pset_WallCommon`, `Pset_DoorCommon` |
| `Qto_` | buildingSMART standard quantity set | `Qto_WallBaseQuantities` |
| `EPset_` | Internal Bonsai / IfcOpenShell use | `EPset_Parametric`, `EPset_Annotation` |
| Custom | Company- or project-specific | `Acme_StructuralData`, `ABC_FireRisk` |

**Rules**:
- Do **NOT** use the `Pset_` or `Qto_` prefix for custom property sets — these are reserved for buildingSMART standards
- Use a company or project identifier as the prefix for custom sets
- `EPset_` sets are used internally by Bonsai to store parametric data

### Python API: Property Sets

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")
wall  = model.by_type("IfcWall")[0]

# Create an empty property set
pset = ifcopenshell.api.pset.add_pset(
    model,
    product=wall,
    name="Pset_WallCommon"
)

# Edit properties
ifcopenshell.api.pset.edit_pset(
    model,
    pset=pset,
    properties={
        "FireRating":            "2HR",
        "ThermalTransmittance":  0.25,
        "LoadBearing":           True,
        "Combustible":           False,
        "OldProperty":           None,  # None + should_purge=True → deleted
    },
    should_purge=True
)

# Custom pset with company prefix
custom_pset = ifcopenshell.api.pset.add_pset(
    model, product=wall, name="Acme_StructuralData")
ifcopenshell.api.pset.edit_pset(
    model, pset=custom_pset,
    properties={"LoadCapacity": 500.0, "SustainableSource": True}
)

# Assign existing pset to multiple elements
ifcopenshell.api.pset.assign_pset(
    model, products=[wall1, wall2], pset=pset
)
```

### Python API: Quantity Sets

```python
# Create quantity set
qto = ifcopenshell.api.pset.add_qto(
    model,
    product=wall,
    name="Qto_WallBaseQuantities"
)

# Edit quantities
ifcopenshell.api.pset.edit_qto(
    model,
    qto=qto,
    properties={
        "Length":       12,    # int → IfcQuantityCount
        "NetVolume":    7.2,   # float + "Volume" keyword → IfcQuantityVolume
        "NetSideArea":  4.2,   # float + "Area" keyword   → IfcQuantityArea
        "Height":       None,  # None → always removed
    }
)
```

**Automatic quantity type detection**:

| Condition | IFC Quantity Type |
|---|---|
| Python `int` | `IfcQuantityCount` |
| `float` + name contains "Area" | `IfcQuantityArea` |
| `float` + name contains "Volume" | `IfcQuantityVolume` |
| `float` + name contains "Weight" | `IfcQuantityWeight` |
| `float` + name contains "Length" | `IfcQuantityLength` |

### Property Set Templates

```python
# Define a template
template = ifcopenshell.api.pset_template.add_pset_template(
    model,
    name="Acme_RiskAssessment",
    template_type="PSET_TYPEDRIVENOVERRIDE",
    applicable_entity="IfcWall,IfcWallType"
)

# Add property definitions to template
ifcopenshell.api.pset_template.add_prop_template(
    model,
    pset_template=template,
    name="HighVoltageRisk",
    description="Presence of high voltage hazard",
    template_type="P_SINGLEVALUE",
    primary_measure_type="IfcBoolean"
)
```

### Reading Property Sets

```python
import ifcopenshell.util.element

# Get all psets and qtos
all_psets = ifcopenshell.util.element.get_psets(wall)

# Filter to property sets only
props_only = ifcopenshell.util.element.get_psets(wall, psets_only=True)

# Filter to quantity sets only
qtos_only = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# Own psets only (skip inherited from type)
own_psets = ifcopenshell.util.element.get_psets(wall, should_inherit=False)
```

---

## 6. Type System

### Overview

The IFC type system provides **shared definitions** for groups of identical or similar elements. A type element (`IfcElementType` subclass) stores:
- Common attributes and properties shared across all instances
- Material composition (layer sets, profile sets, constituent sets)
- Geometric representation template (MappedItem)
- `PredefinedType` classification

The link between occurrences and their type is `IfcRelDefinesByType`.

### Common IFC Type Classes

| Occurrence Class | Type Class | Typical Use |
|---|---|---|
| `IfcWall` | `IfcWallType` | Walls with defined layer composition |
| `IfcSlab` | `IfcSlabType` | Floor/roof slabs |
| `IfcColumn` | `IfcColumnType` | Structural columns |
| `IfcBeam` | `IfcBeamType` | Structural beams |
| `IfcDoor` | `IfcDoorType` | Door families with shared geometry |
| `IfcWindow` | `IfcWindowType` | Window families |
| `IfcPipeSegment` | `IfcPipeSegmentType` | MEP pipe segments |

### PredefinedType Enums (IfcWallTypeEnum)

| Value | Description |
|---|---|
| `STANDARD` | Basic wall with constant cross-section extruded vertically |
| `SOLIDWALL` | Massive load-bearing masonry or concrete |
| `PARTITIONING` | Non-load-bearing lightweight partition |
| `RETAININGWALL` | Retaining wall against soil/water pressure |
| `SHEAR` | Lateral-force-resisting wall |
| `PARAPET` | Protective barrier at roof/balcony edge |
| `USERDEFINED` | Custom type via `ElementType` attribute |
| `NOTDEFINED` | Unspecified |

### Python API: Type Assignment

```python
import ifcopenshell
import ifcopenshell.api

model     = ifcopenshell.open("project.ifc")
wall_type = model.by_type("IfcWallType")[0]
wall1     = model.by_type("IfcWall")[0]
wall2     = model.by_type("IfcWall")[1]

# Assign type to occurrences (list required)
ifcopenshell.api.type.assign_type(
    model,
    related_objects=[wall1, wall2],
    relating_type=wall_type,
    should_map_representations=True   # instances inherit type geometry via MappedItem
)

# Remove type assignment
ifcopenshell.api.type.unassign_type(
    model,
    related_objects=[wall1]
)

# Utility functions
import ifcopenshell.util.element
wall_type = ifcopenshell.util.element.get_type(wall1)
occurrences = ifcopenshell.util.element.get_types(wall_type)
```

### Material Sets

#### Material Layer Sets (Walls, Slabs)

```python
layer_set = ifcopenshell.api.material.add_material_set(
    model, name="200mm Insulated Wall", set_type="IfcMaterialLayerSet")

concrete   = ifcopenshell.api.material.add_material(model, name="Concrete C25")
insulation = ifcopenshell.api.material.add_material(model, name="Mineral Wool")

layer_c = ifcopenshell.api.material.add_layer(
    model, layer_set=layer_set, material=concrete)
layer_i = ifcopenshell.api.material.add_layer(
    model, layer_set=layer_set, material=insulation)

ifcopenshell.api.material.edit_layer(
    model, layer=layer_c, attributes={"LayerThickness": 0.200})
ifcopenshell.api.material.edit_layer(
    model, layer=layer_i, attributes={"LayerThickness": 0.050})

# Assign to wall type
ifcopenshell.api.material.assign_material(
    model, products=[wall_type], material=layer_set)
```

#### Material Profile Sets (Columns, Beams)

```python
profile_set = ifcopenshell.api.material.add_material_set(
    model, name="HEA 200 Steel", set_type="IfcMaterialProfileSet")

steel = ifcopenshell.api.material.add_material(model, name="S355 Steel")
profile = model.create_entity("IfcIShapeProfileDef",
    ProfileType="AREA", ProfileName="HEA200",
    OverallWidth=0.200, OverallDepth=0.190,
    WebThickness=0.0065, FlangeThickness=0.010)

ifcopenshell.api.material.add_profile(
    model, profile_set=profile_set, material=steel, profile=profile)

ifcopenshell.api.material.assign_material(
    model, products=[column_type], material=profile_set)
```

### Type Geometry Sharing: MappedItem

When `type.assign_type` is called with `should_map_representations=True`, Bonsai replaces each occurrence's body representation with an `IfcMappedItem` that references the type's `IfcRepresentationMap`:

```
IfcWallType
  └── RepresentationMaps
        └── IfcRepresentationMap
              └── MappedRepresentation (IfcShapeRepresentation)
                    └── IfcExtrudedAreaSolid (actual geometry)

IfcWall (instance 1)
  └── Representation
        └── IfcShapeRepresentation
              └── IfcMappedItem
                    ├── MappingSource → IfcRepresentationMap (above)
                    └── MappingTarget → IfcCartesianTransformationOperator3D
```

The geometry is stored **once** on the type; all instances reference it with per-instance transformation matrices.

---

## 7. Modeling Workflow

### Assigning an IFC Class to an Object

Any Blender mesh object can be converted to an IFC element by assigning it a class.

**Via UI**: Properties panel → **BIM Tool** tab → IFC Class dropdown → click **Assign IFC Class**

**Via Python**:
```python
import bpy
bpy.ops.bim.assign_class(ifc_class="IfcWall", predefined_type="", userdefined_type="")
```

### Walls

#### Wall Tool (Recommended)

1. Press **SHIFT+SPACEBAR** to open the tool selector
2. Press **6** (or click **Wall** in the toolbar) to activate the Wall tool
3. Click two points in the 3D viewport to define the wall's start and end
4. The tool creates a parametric `IfcWall` with a `SweptSolid` (ExtrudedAreaSolid) representation

#### Generic Mesh Workflow

```python
import bpy

# Create a mesh scaled to wall dimensions
bpy.ops.mesh.primitive_cube_add(size=1)
wall_obj = bpy.context.active_object
wall_obj.dimensions = (5.0, 0.2, 3.0)  # length, width, height
bpy.ops.object.transform_apply(scale=True)

# Assign IfcWall class
bpy.ops.bim.assign_class(ifc_class="IfcWall")
```

### Slabs

Activate via **SHIFT+SPACEBAR** → **Slab** tool. Or use generic mesh workflow:

```python
bpy.ops.bim.assign_class(ifc_class="IfcSlab", predefined_type="FLOOR")
```

**Predefined types for IfcSlab**: `FLOOR`, `ROOF`, `LANDING`, `BASESLAB`, `USERDEFINED`, `NOTDEFINED`

### Columns

```python
bpy.ops.mesh.primitive_cylinder_add(radius=0.2, depth=3.0)
bpy.ops.bim.assign_class(ifc_class="IfcColumn", predefined_type="COLUMN")
```

### Beams

```python
bpy.ops.bim.assign_class(ifc_class="IfcBeam", predefined_type="BEAM")
```

### Openings (IfcOpeningElement)

```python
import bpy
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

ifc_file = IfcStore.get_file()

# Step 1: Create void box and assign IfcOpeningElement
bpy.ops.mesh.primitive_cube_add(size=1)
void_obj = bpy.context.active_object
void_obj.dimensions = (1.0, 0.25, 2.1)  # 1000mm wide, 250mm deep, 2100mm high
bpy.ops.object.transform_apply(scale=True)
bpy.ops.bim.assign_class(ifc_class="IfcOpeningElement")

# Step 2: Add the opening to the host wall via ifcopenshell.api
wall_element    = ifc_file.by_id(wall_ifc_id)
opening_element = ifc_file.by_id(opening_ifc_id)

# Creates IfcRelVoidsElement
ifcopenshell.api.run("feature.add_feature",
    ifc_file,
    feature=opening_element,
    element=wall_element
)

# Step 3: Fill with door (creates IfcRelFillsElement)
door_element = ifc_file.by_id(door_ifc_id)
ifcopenshell.api.run("void.add_filling",
    ifc_file,
    opening=opening_element,
    element=door_element
)
```

**Note**: In IfcOpenShell v0.8.0+, use `feature.add_feature()` for voids. The `void` module has been superseded by `feature`.

### IFC Class Assignment Summary

| Element | IFC Class | Predefined Type |
|---|---|---|
| Wall | `IfcWall` | `STANDARD`, `SHEAR` |
| Slab (floor) | `IfcSlab` | `FLOOR` |
| Slab (roof) | `IfcSlab` | `ROOF` |
| Column | `IfcColumn` | `COLUMN` |
| Beam | `IfcBeam` | `BEAM` |
| Opening void | `IfcOpeningElement` | `OPENING`, `RECESS` |
| Door | `IfcDoor` | `DOOR` |
| Window | `IfcWindow` | `WINDOW` |

---

## 8. Classification Systems

### Built-in Classification Systems

Bonsai v0.8.4 ships with the following classification systems pre-loaded:

| System | Origin | Scope |
|---|---|---|
| **Uniclass 2015** | UK (NBS) | Unified construction classification |
| **OmniClass** | North America | Construction information classification |

### Additional / Custom Classification Systems

| System | Origin | Notes |
|---|---|---|
| **NL-SfB** | Netherlands | Dutch building elements coding |
| **MasterFormat** | North America | CSI construction specifications |
| **ETIM** | Europe | Technical product classification |
| Custom CSV/XML | User-defined | Import via classification manager |

Multiple classification systems **can be assigned simultaneously** to a single element — IFC supports multiple `IfcRelAssociatesClassification` relationships per product.

### Python API

```python
import ifcopenshell
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

ifc_file = IfcStore.get_file()

# Add Uniclass 2015 classification library to project
classification_system = ifcopenshell.api.run(
    "classification.add_classification",
    ifc_file,
    classification="Uniclass2015"
)

# Assign a classification reference to an element
wall_element = ifc_file.by_id(wall_ifc_id)
reference = ifcopenshell.api.run(
    "classification.add_reference",
    ifc_file,
    product=wall_element,
    identification="Ss_20_10_30",
    name="Walls",
    classification=classification_system
)

# Edit a classification
ifcopenshell.api.run(
    "classification.edit_classification",
    ifc_file,
    classification=classification_system,
    attributes={
        "Name": "Uniclass 2015",
        "Edition": "1.8",
        "EditionDate": "2023-01-01",
    }
)

# Remove a reference
ifcopenshell.api.run(
    "classification.remove_reference",
    ifc_file,
    reference=reference,
    product=wall_element
)
```

### Reading Classifications from an Element

```python
from bonsai.bim.ifc import IfcStore

ifc_file = IfcStore.get_file()
element  = ifc_file.by_id(element_ifc_id)

for association in element.HasAssociations:
    if association.is_a("IfcRelAssociatesClassification"):
        ref = association.RelatingClassification
        print(f"Code: {ref.Identification}")
        print(f"Name: {ref.Name}")
```

---

## 9. Geometry Representations

### Representation Contexts

#### Model Context (3D)

| Sub-context | Description |
|---|---|
| **Body** | Main solid geometry — the primary 3D representation |
| **Axis** | Centerline/axis representation (walls, beams, columns) |
| **Box** | Bounding box representation for clash detection |
| **FootPrint** | 2D floor footprint projection |

#### Plan Context (2D)

| Sub-context | Description |
|---|---|
| **Annotation** | 2D annotations, dimensions, text |
| **Axis** | 2D centerline for plan views |

### Representation Types

#### SweptSolid — ExtrudedAreaSolid

Most common for parametric elements (walls, columns, beams):

```
IfcShapeRepresentation
  RepresentationType = "SweptSolid"
  Items = [IfcExtrudedAreaSolid]
    SweptArea = IfcRectangleProfileDef
    ExtrudedDirection = IfcDirection(0., 0., 1.)
    Depth = 3000.0  (wall height)
```

#### Tessellation — IfcPolygonalFaceSet

Used when importing Blender meshes directly into IFC:

```
IfcShapeRepresentation
  RepresentationType = "Tessellation"
  Items = [IfcPolygonalFaceSet]
    Coordinates = IfcCartesianPointList3D [...]
    Faces = [IfcIndexedPolygonalFace [1,2,3], ...]
```

#### Clipping — BooleanClippingResult

SweptSolid with boolean subtraction operations (e.g., angled wall tops):

```
IfcShapeRepresentation
  RepresentationType = "Clipping"
  Items = [IfcBooleanClippingResult]
    Operator = DIFFERENCE
    FirstOperand = IfcExtrudedAreaSolid
    SecondOperand = IfcHalfSpaceSolid
```

#### Curve2D — IfcPolyline

2D plan representations:

```
IfcShapeRepresentation
  ContextOfItems = Plan/Annotation
  RepresentationType = "Curve2D"
  Items = [IfcPolyline]
```

### MappedItems — Type Geometry Sharing

`IfcMappedItem` stores geometry once on the type and all instances reference it via a mapping transformation. Created automatically when `type.assign_type` is called with `should_map_representations=True`.

### IfcRelVoidsElement

```
IfcWall ──────────────────────────────────────────────┐
IfcRelVoidsElement                                     │
  RelatingBuildingElement = IfcWall  ←────────────────┘
  RelatedOpeningElement   = IfcOpeningElement ─────────┐
IfcRelFillsElement                                     │
  RelatingOpeningElement = IfcOpeningElement ←─────────┘
  RelatedBuildingElement = IfcDoor
```

### Geometry API

```python
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

ifc_file = IfcStore.get_file()

# Get or verify Body context
body_context = ifcopenshell.util.representation.get_context(
    ifc_file, "Model", "Body", "MODEL_VIEW")

if not body_context:
    model_ctx = ifcopenshell.api.run("context.add_context", ifc_file,
        context_type="Model")
    body_context = ifcopenshell.api.run("context.add_context", ifc_file,
        context_type="Model",
        context_identifier="Body",
        target_view="MODEL_VIEW",
        parent=model_ctx)

# Add a wall representation
representation = ifcopenshell.api.run("geometry.add_wall_representation",
    ifc_file,
    context=body_context,
    length=5.0,
    height=3.0,
    thickness=0.2)

# Assign to element
ifcopenshell.api.run("geometry.assign_representation", ifc_file,
    product=element,
    representation=representation)

# Remove a representation
ifcopenshell.api.run("geometry.remove_representation", ifc_file,
    representation=representation)
```

### ShapeBuilder — Parametric Geometry

```python
import ifcopenshell.util.shape_builder

ifc_file = IfcStore.get_file()
builder = ifcopenshell.util.shape_builder.ShapeBuilder(ifc_file)

# Create extruded rectangle (e.g., column)
extrusion = builder.extrude(
    ifc_file.createIfcRectangleProfileDef(
        ProfileType="AREA", XDim=0.4, YDim=0.4),
    magnitude=3.0,
    extrusion_vector=ifc_file.createIfcDirection((0., 0., 1.))
)
representation = builder.get_representation(body_context, [extrusion])
```

---

## 10. IFC Export Pipeline (Saving)

### Core Concept: Bonsai Does NOT Export IFC

Bonsai uses **native IFC authoring**. The IFC file IS the project document. There is no export step, no translation layer, and no conversion loss.

**ALWAYS** think of the Bonsai workflow as: Open `.ifc` → edit in Blender → save `.ifc`

**NEVER** use Bonsai as an IFC exporter from a native Blender mesh workflow.

### Native Save

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
model.write("/path/to/project.ifc")

# Or via operator
import bpy
bpy.ops.bim.save_project(filepath="/path/to/project.ifc")
```

### MVD Selection: Model View Definitions

| MVD | Schema | Use Case |
|---|---|---|
| `DesignTransferView` | IFC4 | Full BIM capability, default for IFC4 |
| `CoordinationView 2.0` | IFC2X3 | Older coordination workflows |
| `ReferenceView` | IFC4 | Read-only geometry exchange |

**Schema versions**: `IFC2X3` (legacy), `IFC4` (recommended), `IFC4X3` (latest, infrastructure)

### Header Metadata

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")
ifcopenshell.api.run("project.edit_header", model,
    editor="Jane Architect",
    organization="ACME Architecture",
    application="Bonsai v0.8.4",
    application_version="0.8.4"
)
model.write("project.ifc")
```

### IDS/IfcTester Validation

**IDS (Information Delivery Specification)** is a buildingSMART standard for defining what data must be present in an IFC file.

```python
# pip install ifctester
import ifctester
import ifctester.ids
import ifctester.reporter
import ifcopenshell

ids   = ifctester.ids.open("specification.ids")
model = ifcopenshell.open("project.ifc")
ids.validate(model)

reporter = ifctester.reporter.Console(ids)
reporter.report()
```

**Via Bonsai UI**: `BIM > IFC Tester` panel

### Schema Migration: IFC2X3 → IFC4

```python
import ifcopenshell
import ifcopenshell.util.schema

model_2x3 = ifcopenshell.open("legacy.ifc")  # schema="IFC2X3"
migrator  = ifcopenshell.util.schema.Migrator()
model_ifc4 = migrator.migrate(model_2x3, "IFC4")
model_ifc4.write("migrated.ifc")
```

**ALWAYS** validate the migrated file with IfcTester after migration.

---

## 11. Common Error Patterns

### Error 1: Python Version Mismatch

**Problem**: Running Bonsai Python modules outside of Blender's bundled Python (Bonsai v0.8.4 requires Python 3.11).

**Wrong**:
```bash
$ python3 -c "from bonsai.bim.ifc import IfcStore"
# ImportError: No module named 'bpy'
```

**Correct**:
```bash
$ blender --background --python my_script.py
```

**Rule**: ALWAYS run Bonsai scripts via `blender --python` or from the Blender internal Python console. NEVER import `bonsai.*` from system Python.

---

### Error 2: IFC/Blend Desync

**Problem**: Editing the `.ifc` file externally while Blender has the project open.

**Wrong**:
```bash
$ nano project.ifc  # NEVER DO THIS while Blender has it open
```

**Correct**: After any external modification, use: `File > Reload IFC Project`

**Rule**: NEVER edit the `.ifc` file on disk while Blender has the project open.

---

### Error 3: Module Path AttributeError (blenderbim → bonsai rename)

**Problem**: Using old `blenderbim` module path (renamed in 2024).

**Wrong**:
```python
from blenderbim.bim.ifc import IfcStore  # ModuleNotFoundError
import blenderbim.tool as tool
```

**Correct**:
```python
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
```

| Old (BlenderBIM) | New (Bonsai) |
|---|---|
| `blenderbim.bim.ifc` | `bonsai.bim.ifc` |
| `blenderbim.tool` | `bonsai.tool` |
| `blenderbim.core` | `bonsai.core` |
| `blenderbim.bim.module.*` | `bonsai.bim.module.*` |

**Rule**: ALWAYS use `bonsai.*` import paths. NEVER use `blenderbim.*` in Bonsai v0.8.0+.

---

### Error 4: Geometry Corruption on Move

**Problem**: Moving Blender objects with the standard `G` key does NOT update `IfcLocalPlacement`.

**Wrong**:
```python
obj = bpy.context.active_object
obj.location = (5.0, 0.0, 0.0)
# IFC placement is NOT updated
```

**Correct**:
```python
obj.location = (5.0, 0.0, 0.0)
bpy.ops.bim.edit_object_placement()  # Sync IFC placement
```

**Rule**: ALWAYS call `bpy.ops.bim.edit_object_placement()` after any direct matrix/location change.

---

### Error 5: Missing Representation Context

**Problem**: Adding geometry without a valid representation context.

**Wrong**:
```python
# Trying to add geometry without verifying context exists
ifcopenshell.api.run("geometry.add_wall_representation", model,
    element=wall, length=5.0, height=3.0, thickness=0.2)
```

**Correct**:
```python
body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

if not body_context:
    model_context = ifcopenshell.api.run("context.add_context", model,
        context_type="Model")
    body_context = ifcopenshell.api.run("context.add_context", model,
        context_type="Model",
        context_identifier="Body",
        target_view="MODEL_VIEW",
        parent=model_context)
```

**Rule**: ALWAYS verify or create representation contexts before adding geometry.

---

### Error 6: GLIBC Incompatibility (Linux)

**Error**:
```
/lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.34' not found
```

Bonsai v0.8.4 requires **GLIBC 2.34+** (Ubuntu 22.04+ equivalent).

**Diagnosis**:
```bash
ldd --version | head -1
# Must show GLIBC 2.34 or higher
```

**Rule**: Bonsai v0.8.4 REQUIRES Ubuntu 22.04+ equivalent on Linux. NEVER attempt to run on Ubuntu 20.04 without building from source.

---

### Error 7: Module Caching After Update

**Problem**: Old `.pyc` bytecode cache from previous Bonsai version causes `AttributeError` or `ImportError` after upgrade.

**Fix**:
```bash
# Linux/macOS
find ~/.config/blender/4.2/scripts/addons/bonsai/ -name "__pycache__" -type d -exec rm -rf {} +
```

Then restart Blender completely.

**Rule**: ALWAYS clear `__pycache__` directories after upgrading Bonsai. NEVER assume a Blender restart alone clears bytecode caches.

---

### Error 8: Structural Subcontext Not Found

**Problem**: Adding structural analysis representations without the required context.

**Error**: `RuntimeError: No structural analysis view context found`

**Correct**:
```python
structural_context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model",
    context_identifier="Reference",
    target_view="GRAPH_VIEW",
    parent=model_context
)
```

**Rule**: ALWAYS create the `GRAPH_VIEW` structural context before adding structural analysis representations.

---

## 12. Bonsai Python API Reference

### Import Paths

```python
# Core IFC store
from bonsai.bim.ifc import IfcStore

# Tool classes
import bonsai.tool as tool

# Core functions
import bonsai.core.spatial
import bonsai.core.geometry

# IfcOpenShell
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element
import ifcopenshell.util.placement
import ifcopenshell.util.representation
```

### IfcStore Class Overview

```python
from bonsai.bim.ifc import IfcStore

# Key static attributes
IfcStore.path          # str: absolute path to .ifc file on disk
IfcStore.file          # ifcopenshell.file: the live in-memory IFC model
IfcStore.schema        # str: "IFC2X3", "IFC4", or "IFC4X3"
IfcStore.id_map        # dict: {ifc_id (int) -> blender_object}
IfcStore.guid_map      # dict: {GlobalId (str) -> blender_object}
IfcStore.edited_objs   # set: objects with pending IFC changes

# Key static methods
model = IfcStore.get_file()             # Returns ifcopenshell.file or None
schema = IfcStore.get_schema()          # Returns schema string
obj = IfcStore.get_element(ifc_id)      # Blender object for given IFC id
```

**ALWAYS check for `None`**:

```python
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")
walls = model.by_type("IfcWall")
```

### tool.Ifc.Operator Base Class

```python
import bpy
from bonsai.bim.ifc import IfcStore

class BIM_OT_my_custom_operator(bpy.types.Operator):
    bl_idname = "bim.my_custom_operator"
    bl_label = "My Custom BIM Operation"
    bl_options = {"REGISTER", "UNDO"}

    def execute(self, context):
        return IfcStore.execute_ifc_operator(self, context)

    def _execute(self, context):
        model = IfcStore.get_file()
        # IFC modifications here
        return {"FINISHED"}
```

### Core Function Pattern

```python
# bonsai/core/spatial.py — domain layer (no bpy imports)
def assign_container(ifc, spatial, structure_obj, element_obj):
    container = ifc.get_entity(structure_obj)
    element   = ifc.get_entity(element_obj)
    ifcopenshell.api.run("spatial.assign_container",
        ifc.get(), relating_structure=container, product=element)

# Calling from operator:
import bonsai.core.spatial as core
import bonsai.tool as tool

core.assign_container(
    ifc=tool.Ifc,
    spatial=tool.Spatial,
    structure_obj=storey_obj,
    element_obj=wall_obj
)
```

### Major `tool.*` Classes Reference

| Tool Class | Purpose | Key Methods |
|---|---|---|
| `tool.Ifc` | IFC model access and entity↔object mapping | `get()`, `get_entity(obj)`, `get_object(entity)`, `run(usecase, **kwargs)` |
| `tool.Spatial` | Spatial container hierarchy | `get_container(element)`, `get_decomposed_elements(container)` |
| `tool.Geometry` | Mesh/representation management | `get_active_representation(obj)`, `edit_object_placement(obj)` |
| `tool.Blender` | Blender scene utilities | `get_obj_ifc_definition_id(obj)`, `set_active_object(obj)` |
| `tool.Pset` | Property sets management | `get_element_pset(element, pset_name)`, `get_pset_props(obj, pset_name)` |
| `tool.Type` | IFC type (IfcTypeObject) management | `get_type(element)`, `assign_type(relating_type, related_objects)` |
| `tool.Material` | Material/layer set management | `get_material(element)`, `get_style(material)` |
| `tool.Context` | Representation context management | `get_context(model, type, id, target_view)` |
| `tool.Style` | Visual style/surface style management | `get_style(obj)`, `get_surface_style(obj)` |
| `tool.Drawing` | 2D drawing/documentation | `get_active_drawing()`, `get_drawing_elements(drawing)` |

### Complete Example: Create a Wall from Scratch

```python
"""
Complete example: Create an IfcWall with geometry in Bonsai v0.8.4
Run from Blender's Python console or via: blender --python script.py
"""
import bpy
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool

# Step 1: Get the live IFC model
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Step 2: Get or create representation context
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

# Step 3: Get the target storey
storey = model.by_type("IfcBuildingStorey")[0]

# Step 4: Create the IfcWall entity
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall",
    name="Wall-001")

# Step 5: Create geometry
wall_representation = ifcopenshell.api.run(
    "geometry.add_wall_representation", model,
    context=body_context,
    length=5.0,     # 5 meters long
    height=3.0,     # 3 meters tall
    thickness=0.2   # 200mm thick
)

# Step 6: Assign the representation
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall,
    representation=wall_representation)

# Step 7: Place in spatial container
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey,
    product=wall)

# Step 8: Save
model.write(IfcStore.path)
print(f"Wall created: {wall.GlobalId}")
```

---

## AI Common Mistakes

The following are the 10 most common mistakes AI assistants make when generating Bonsai code or documentation. **All Bonsai users and AI assistants MUST be aware of these.**

### Mistake 1: Using `blenderbim.*` Instead of `bonsai.*`

**Wrong**: `from blenderbim.bim.ifc import IfcStore`
**Correct**: `from bonsai.bim.ifc import IfcStore`

Bonsai was renamed from BlenderBIM in 2024. All module paths changed. Code using `blenderbim.*` will fail with `ModuleNotFoundError` in Bonsai v0.8.0+. **NEVER** use `blenderbim.*` import paths.

---

### Mistake 2: Using `void.add_opening()` (Non-Existent Module)

**Wrong**:
```python
ifcopenshell.api.run("void.add_opening", model, opening=opening, element=wall)
```

**Correct**:
```python
ifcopenshell.api.run("feature.add_feature", model, feature=opening, element=wall)
```

The `void` module does **NOT** exist in ifcopenshell v0.8.0+. It was moved to the `feature` module. **ALWAYS** use `feature.add_feature()` for opening voids.

---

### Mistake 3: Treating Bonsai as an Importer/Exporter

**Wrong mental model**: "Import the IFC file into Blender, do BIM work, export back to IFC."

**Correct mental model**: "Open the IFC file natively. The IFC IS the document. Save directly to IFC."

Bonsai is a **native IFC authoring tool**. There is no import/export pipeline. **NEVER** describe Bonsai's workflow as "import" or "export."

---

### Mistake 4: Using Wrong Blender Version

**Wrong**: Recommending or using Blender versions below 4.2.0.

Bonsai v0.8.4 requires **Blender 4.2.0 as minimum**. It will **NOT** load in Blender 3.x, 4.0, or 4.1. **ALWAYS** specify Blender 4.2.0+ when documenting Bonsai v0.8.4 requirements.

---

### Mistake 5: Running `tool.*` Outside Blender Context

**Wrong**:
```python
# Standalone Python script outside Blender
import bonsai.tool as tool
entity = tool.Ifc.get_entity(some_object)  # Fails — no bpy context
```

`bonsai.tool.*` classes depend on `bpy`. They **CANNOT** be used in standalone Python scripts. For headless/standalone IFC processing, use `ifcopenshell` directly.

---

### Mistake 6: Forgetting to Check `IfcStore.get_file()` for `None`

**Wrong**:
```python
model = IfcStore.file  # May be None
walls = model.by_type("IfcWall")  # AttributeError
```

**Correct**:
```python
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project is currently loaded.")
walls = model.by_type("IfcWall")
```

**ALWAYS** check for `None` before using the model object.

---

### Mistake 7: Confusing IfcRelAggregates with IfcRelContainedInSpatialStructure

| Relationship | Purpose | Example |
|---|---|---|
| `IfcRelAggregates` | Spatial decomposition hierarchy | Site → Building → Storey |
| `IfcRelContainedInSpatialStructure` | Element containment in spatial zone | Wall contained IN a Storey |

**Wrong**:
```python
# Using aggregation to contain a wall in a storey
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=storey, product=wall  # WRONG for physical elements
)
```

**Correct**:
```python
# For element containment (wall in storey):
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, product=wall)

# For spatial hierarchy (storey in building):
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, product=storey)
```

**ALWAYS** use `spatial.assign_container` for placing physical elements in a storey.

---

### Mistake 8: Passing Single Object Instead of List to `type.assign_type`

**Wrong**:
```python
ifcopenshell.api.run("type.assign_type", model,
    relating_type=wall_type,
    related_object=wall_instance  # Single object — WRONG
)
```

**Correct**:
```python
ifcopenshell.api.run("type.assign_type", model,
    relating_type=wall_type,
    related_objects=[wall_instance]  # List, even for single object
)
```

The `related_objects` parameter expects a **list**. **ALWAYS** wrap single objects in a list.

---

### Mistake 9: Assuming Pset Properties Are Committed Without Calling `edit_pset()`

**Wrong**:
```python
pset_props = obj.PsetProperties
pset_props["FireRating"] = "REI60"
# Changes are NOT written to IFC — only Blender UI is updated
```

**Correct**:
```python
import ifcopenshell.api
pset_entity = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset_entity,
    properties={"FireRating": "REI60"})
```

**ALWAYS** call `pset.edit_pset()` to write properties to the IFC model. **NEVER** assume modifying Blender UI properties alone updates the IFC data.

---

### Mistake 10: Using Deprecated BlenderBIM Operator Names

**Preferred approach**: Use `ifcopenshell.api.run()` directly over `bpy.ops.bim.*` operators in Python scripts:

```python
# Preferred: stable, version-independent
import ifcopenshell.api
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, product=element)
```

**ALWAYS** prefer `ifcopenshell.api.run()` over Bonsai `bpy.ops.bim.*` operators in scripts, as the API is more stable and explicit.

---

## Sources

### Official Bonsai / IfcOpenShell Sources

| Source | URL | Description |
|---|---|---|
| Bonsai Official Site | https://bonsaibim.org | Homepage, downloads, release notes |
| Bonsai Documentation | https://docs.bonsaibim.org | User documentation, tutorials |
| IfcOpenShell GitHub | https://github.com/IfcOpenShell/IfcOpenShell | Source code monorepo (`src/bonsai/` for Bonsai) |
| IfcOpenShell Docs | https://docs.ifcopenshell.org | Python API documentation |

### Community Sources

| Source | URL | Description |
|---|---|---|
| OSArch Wiki | https://wiki.osarch.org | Open-source AEC wiki, Bonsai guides |
| OSArch Community | https://community.osarch.org | Forums, Q&A, community support |

### IFC Standards

| Source | URL | Description |
|---|---|---|
| buildingSMART IFC Standards | https://standards.buildingsmart.org/IFC | IFC4, IFC4X3 schemas |
| IDS Specification | https://standards.buildingsmart.org/IDS | Information Delivery Specification |

### Key GitHub Paths in IfcOpenShell Monorepo

- `src/bonsai/` — Bonsai Blender add-on source
- `src/bonsai/bonsai/bim/ifc.py` — `IfcStore` class definition
- `src/bonsai/bonsai/tool/` — All `tool.*` class implementations
- `src/bonsai/bonsai/core/` — Core business logic functions
- `src/bonsai/bonsai/bim/module/` — Per-module operators and UI
- `src/ifcopenshell/` — Core ifcopenshell Python library
- `src/ifcopenshell/api/` — All `ifcopenshell.api.run()` use cases

---

*Assembled by: orch-bonsai-research (orchestrator, depth 0)*
*Workers: bonsai-overview, bonsai-spatial, bonsai-modeling, bonsai-errors*
*Date: 2026-03-05 | Version: Bonsai v0.8.4 | Minimum Blender: 4.2.0*
