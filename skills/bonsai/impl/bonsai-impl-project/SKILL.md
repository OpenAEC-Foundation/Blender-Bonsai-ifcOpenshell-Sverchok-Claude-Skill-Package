---
name: bonsai-impl-project
description: >
  Use when creating, opening, or configuring Bonsai IFC projects -- schema selection
  (IFC2X3/IFC4/IFC4X3), unit configuration, georeference setup, or project templates.
  Prevents the common mistake of not setting units before creating geometry (defaulting to
  incorrect units). Covers the complete project lifecycle from creation to delivery.
  Keywords: IFC project, schema selection, IFC2X3, IFC4, IFC4X3, units, georeference, coordinate reference system, project template, Bonsai project, CRS, EPSG, WGS84, map coordinates, start new project.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Project Management

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x | IFC2X3 / IFC4 / IFC4X3
> **Dependency**: bonsai-core-architecture (MUST read first)
> **Module path**: `bonsai.*` — NEVER `blenderbim.*`

## Critical Warnings

1. **ALWAYS** use `ifcopenshell.api.project.create_file()` to create new IFC projects. NEVER use `ifcopenshell.file()` directly — it skips required project setup (IfcProject, IfcUnitAssignment, representation contexts).
2. **ALWAYS** set units via `ifcopenshell.api.unit.assign_unit()` immediately after project creation. An IFC file without a unit assignment is invalid per all IFC schemas.
3. **NEVER** describe the Bonsai workflow as "import/export". The IFC file IS the native document.
4. **ALWAYS** use `bonsai.*` imports. NEVER use `blenderbim.*` — renamed in 2024 (v0.8.0+).
5. **ALWAYS** check `IfcStore.get_file()` for `None` before any IFC operation.
6. **ALWAYS** call `ifcopenshell.api.georeference.add_georeferencing()` before `edit_georeferencing()`. Editing non-existent entities raises an error.
7. **NEVER** set Eastings/Northings without confirmed survey data. Wrong values place the building at the wrong location on Earth.
8. **ALWAYS** run scripts via `blender --python`. NEVER import `bonsai.*` from system Python.

## Schema Selection Decision Tree

```
Which IFC schema should I use?
|
+--> Legacy project or government mandate for IFC2X3?
|    +--> YES --> schema="IFC2X3"
|    +--> NO  --> Continue
|
+--> Infrastructure project (roads, bridges, tunnels, rail)?
|    +--> YES --> schema="IFC4X3"
|    +--> NO  --> Continue
|
+--> Default for buildings
     +--> schema="IFC4"
```

| Schema | Use Case | IfcProject Required? | Georeferencing |
|--------|----------|---------------------|----------------|
| `IFC2X3` | Legacy, government compliance | YES | ePSet_MapConversion (property set) |
| `IFC4` | Buildings (recommended default) | YES | IfcMapConversion + IfcProjectedCRS |
| `IFC4X3` | Infrastructure + buildings | YES | IfcMapConversion/IfcMapConversionScaled + IfcProjectedCRS |

## Project Creation

### Decision Tree: How to Create a Project

```
Creating an IFC project?
|
+--> Inside Blender with Bonsai?
|    |
|    +--> Quick start?
|    |    +--> bpy.ops.bim.create_project(template="metric_m")
|    |
|    +--> Custom setup?
|         +--> bpy.ops.bim.create_project() with wizard options
|
+--> Standalone Python (no Blender)?
     |
     +--> model = ifcopenshell.api.project.create_file(version="IFC4")
     +--> ifcopenshell.api.unit.assign_unit(model)
     +--> Create IfcSite, IfcBuilding, IfcStorey
     +--> model.write("project.ifc")
```

### Standalone Project Creation (No Blender)

```python
import ifcopenshell
import ifcopenshell.api

# Step 1: Create file with schema selection
model = ifcopenshell.api.project.create_file(version="IFC4")

# Step 2: Assign units (metric, millimeters for length)
ifcopenshell.api.unit.assign_unit(model,
    length={"is_metric": True, "raw": "MILLIMETRE"},
    area={"is_metric": True, "raw": "SQUARE_METRE"},
    volume={"is_metric": True, "raw": "CUBIC_METRE"})

# Step 3: Create spatial hierarchy
project = model.by_type("IfcProject")[0]  # auto-created by create_file
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Default Site")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Default Building")
storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# Step 4: Create spatial containment hierarchy
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey])

# Step 5: Create representation contexts
model_context = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_context = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_context)
plan_context = ifcopenshell.api.context.add_context(model,
    context_type="Plan")

# Step 6: Save
model.write("project.ifc")
```

### Bonsai Project Creation (Inside Blender)

```python
import bpy

# Method 1: Quick metric project (meters)
bpy.ops.bim.create_project(template="metric_m")

# Method 2: Quick metric project (millimeters)
bpy.ops.bim.create_project(template="metric_mm")

# Method 3: Quick imperial project (feet)
bpy.ops.bim.create_project(template="imperial_ft")

# Method 4: Demo project (pre-populated with example content)
bpy.ops.bim.create_project(template="demo")
```

## Opening and Saving IFC Files

### Decision Tree: File Operations

```
IFC file operation needed?
|
+--> Open existing file?
|    |
|    +--> Inside Blender?
|    |    +--> bpy.ops.bim.load_project(filepath="project.ifc")
|    |
|    +--> Standalone Python?
|         +--> model = ifcopenshell.open("project.ifc")
|
+--> Save project?
|    |
|    +--> Inside Blender?
|    |    +--> bpy.ops.bim.save_project(filepath="project.ifc")
|    |
|    +--> Standalone Python?
|         +--> model.write("project.ifc")
|
+--> Save to different file (Save As)?
     +--> model.write("new_path.ifc")
```

### Standalone File I/O

```python
import ifcopenshell

# Open
model = ifcopenshell.open("/path/to/project.ifc")

# Inspect
print(f"Schema: {model.schema}")        # "IFC2X3", "IFC4", "IFC4X3"
print(f"Project: {model.by_type('IfcProject')[0].Name}")

# Save (overwrites)
model.write("/path/to/project.ifc")

# Save As (new file)
model.write("/path/to/project_v2.ifc")
```

### Bonsai File I/O (Inside Blender)

```python
import bpy
from bonsai.bim.ifc import IfcStore

# Open
bpy.ops.bim.load_project(filepath="/path/to/project.ifc")

# Access the live IFC model
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Save to current path
bpy.ops.bim.save_project()

# Save As
bpy.ops.bim.save_project(filepath="/path/to/project_v2.ifc")
```

## Unit Configuration

### Decision Tree: Unit Selection

```
What unit system?
|
+--> Metric (international)?
|    |
|    +--> Architecture/interiors (mm precision)?
|    |    +--> length: MILLIMETRE
|    |
|    +--> Urban/site planning (m precision)?
|         +--> length: METRE (or no prefix = default SI)
|
+--> Imperial (US/UK)?
     |
     +--> length: "foot" via add_conversion_based_unit
     +--> OR: "inch" for detail work
```

### Unit Assignment Patterns

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.project.create_file(version="IFC4")

# Pattern 1: Default units (mm length, m² area, m³ volume)
ifcopenshell.api.unit.assign_unit(model)

# Pattern 2: Explicit metric (meters)
length_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="LENGTHUNIT", prefix=None)  # No prefix = metres
area_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="AREAUNIT", prefix=None)
volume_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="VOLUMEUNIT", prefix=None)
ifcopenshell.api.unit.assign_unit(model,
    units=[length_unit, area_unit, volume_unit])

# Pattern 3: Explicit metric (millimeters)
length_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="LENGTHUNIT", prefix="MILLI")
area_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="AREAUNIT", prefix=None)
volume_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="VOLUMEUNIT", prefix=None)
ifcopenshell.api.unit.assign_unit(model,
    units=[length_unit, area_unit, volume_unit])

# Pattern 4: Imperial (feet)
length_unit = ifcopenshell.api.unit.add_conversion_based_unit(model,
    name="foot")
area_unit = ifcopenshell.api.unit.add_conversion_based_unit(model,
    name="square foot")
ifcopenshell.api.unit.assign_unit(model,
    units=[length_unit, area_unit])

# Pattern 5: Add monetary unit
currency = ifcopenshell.api.unit.add_monetary_unit(model,
    currency="EUR")
ifcopenshell.api.unit.assign_unit(model, units=[currency])
```

### SI Unit Prefixes

| Prefix | Factor | Typical Use |
|--------|--------|-------------|
| `"MILLI"` | 10⁻³ | mm — architecture, structural (most common) |
| `None` | 1 | m — site planning, infrastructure |
| `"CENTI"` | 10⁻² | cm — rarely used in IFC |
| `"KILO"` | 10³ | km — large-scale infrastructure |

### Conversion-Based Unit Names

| Name | Unit Type | Use |
|------|-----------|-----|
| `"foot"` | LENGTHUNIT | US Imperial length |
| `"inch"` | LENGTHUNIT | US Imperial detail |
| `"yard"` | LENGTHUNIT | US Imperial site |
| `"square foot"` | AREAUNIT | US Imperial area |
| `"cubic foot"` | VOLUMEUNIT | US Imperial volume |
| `"degree"` | PLANEANGLEUNIT | Angle measurement |

## Georeferencing

### Decision Tree: Georeferencing Setup

```
Does the project need georeferencing?
|
+--> No survey data available?
|    +--> Skip georeferencing — add later when data is available
|
+--> Survey data available?
     |
     +--> Step 1: add_georeferencing()
     +--> Step 2: edit_georeferencing() with EPSG code + Eastings/Northings
     +--> Step 3 (optional): edit_true_north() for solar analysis
```

### Georeferencing Setup Pattern

```python
import ifcopenshell
import ifcopenshell.api
from math import cos, sin, radians

model = ifcopenshell.api.project.create_file(version="IFC4")

# Step 1: Initialize georeferencing entities
ifcopenshell.api.georeference.add_georeferencing(model)

# Step 2: Configure CRS and map conversion
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={
        "Name": "EPSG:28992"  # Amersfoort / RD New (Netherlands)
    },
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": cos(radians(0.0)),
        "XAxisOrdinate": sin(radians(0.0)),
        "Scale": 1.0
    })

# Step 3 (optional): Set true north for solar analysis
ifcopenshell.api.georeference.edit_true_north(model,
    true_north=5.0)  # Degrees anticlockwise from Y-axis
```

### Common EPSG Codes for Construction

| EPSG Code | Name | Region |
|-----------|------|--------|
| `EPSG:28992` | Amersfoort / RD New | Netherlands |
| `EPSG:32632` | WGS 84 / UTM zone 32N | Central Europe |
| `EPSG:32633` | WGS 84 / UTM zone 33N | Eastern Europe |
| `EPSG:27700` | OSGB 1936 / British National Grid | United Kingdom |
| `EPSG:2154` | RGF93 v1 / Lambert-93 | France |
| `EPSG:25832` | ETRS89 / UTM zone 32N | Germany, Denmark, Austria |
| `EPSG:2056` | CH1903+ / LV95 | Switzerland |
| `EPSG:3857` | WGS 84 / Pseudo-Mercator | Web mapping (NOT for construction) |
| `EPSG:7856` | GDA2020 / MGA zone 56 | Australia (East) |
| `EPSG:2263` | NAD83 / New York Long Island (ftUS) | New York, USA |

### IFC2X3 Georeferencing (Legacy)

IFC2X3 does NOT have `IfcMapConversion` or `IfcProjectedCRS` entities. Instead, georeferencing uses the `ePSet_MapConversion` property set on `IfcProject`. The `add_georeferencing` API handles this automatically — use the same API calls regardless of schema version.

## Header Metadata

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")

# Edit file header metadata
ifcopenshell.api.project.edit_header(model,
    editor="Jane Architect",
    organization="ACME Architecture",
    application="Bonsai v0.8.4",
    application_version="0.8.4")

model.write("project.ifc")
```

## Owner History Setup

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.project.create_file(version="IFC4")

# Create person + organization (IFC4: optional but recommended)
person = ifcopenshell.api.owner.add_person(model,
    identification="jdoe",
    family_name="Doe",
    given_name="Jane")
org = ifcopenshell.api.owner.add_organisation(model,
    identification="ACME",
    name="ACME Architecture")
person_org = ifcopenshell.api.owner.add_person_and_organisation(model,
    person=person, organisation=org)

# Register application
app = ifcopenshell.api.owner.add_application(model,
    application_full_name="My BIM Tool",
    application_identifier="MyBIMTool",
    version="1.0")

# Configure automatic owner history for new entities
ifcopenshell.api.owner.settings.get_user = lambda ifc: person_org
ifcopenshell.api.owner.settings.get_application = lambda ifc: app
```

## Schema Migration

```python
import ifcopenshell
import ifcopenshell.util.schema

model_2x3 = ifcopenshell.open("legacy.ifc")
migrator = ifcopenshell.util.schema.Migrator()
model_ifc4 = migrator.migrate(model_2x3, "IFC4")
model_ifc4.write("migrated.ifc")
```

**ALWAYS** validate the migrated file with IfcTester after migration.

## Representation Context Setup

Every IFC project MUST have at least one `IfcGeometricRepresentationContext` before any geometry can be assigned.

```python
# Minimum required: Model context + Body sub-context
model_context = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_context = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_context)

# Optional: Plan context for 2D views
plan_context = ifcopenshell.api.context.add_context(model,
    context_type="Plan")
annotation_context = ifcopenshell.api.context.add_context(model,
    context_type="Plan",
    context_identifier="Annotation",
    target_view="PLAN_VIEW",
    parent=plan_context)
```

### Context Identifiers Reference

| Context Type | Identifier | Target View | Purpose |
|-------------|------------|-------------|---------|
| `"Model"` | `"Body"` | `"MODEL_VIEW"` | Primary 3D geometry (REQUIRED) |
| `"Model"` | `"Axis"` | `"GRAPH_VIEW"` | Centerline representations |
| `"Model"` | `"Box"` | `"MODEL_VIEW"` | Bounding box for clash detection |
| `"Model"` | `"FootPrint"` | `"MODEL_VIEW"` | Floor plan footprint |
| `"Model"` | `"Clearance"` | `"MODEL_VIEW"` | Clearance volumes |
| `"Plan"` | `"Annotation"` | `"PLAN_VIEW"` | 2D annotations |
| `"Plan"` | `"Axis"` | `"PLAN_VIEW"` | 2D centerlines |

## Project Lifecycle Summary

```
1. CREATE  --> ifcopenshell.api.project.create_file(version="IFC4")
2. UNITS   --> ifcopenshell.api.unit.assign_unit(model)
3. SPATIAL --> Create IfcSite, IfcBuilding, IfcBuildingStorey
4. CONTEXT --> ifcopenshell.api.context.add_context(model, ...)
5. GEO     --> ifcopenshell.api.georeference.add_georeferencing(model)
6. OWNER   --> ifcopenshell.api.owner.add_person/organisation(model)
7. MODEL   --> Add elements, properties, geometry
8. SAVE    --> model.write("project.ifc")
```

**ALWAYS** follow this order. Steps 5 and 6 are optional but recommended for production deliverables.

## References

- [methods.md](references/methods.md) — Complete API signatures for project, unit, georeference, context, owner modules
- [examples.md](references/examples.md) — Working code examples for project lifecycle workflows
- [anti-patterns.md](references/anti-patterns.md) — What NOT to do in project management

## Sources

- Bonsai Documentation: https://docs.bonsaibim.org
- IfcOpenShell API: https://docs.ifcopenshell.org
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell
- Bonsai source: `src/bonsai/bonsai/bim/module/project/`
