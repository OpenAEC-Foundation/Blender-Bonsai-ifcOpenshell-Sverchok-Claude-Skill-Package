# Project Workflow Examples

> **Version**: IfcOpenShell v0.8.x | Bonsai v0.8.x | IFC2X3 / IFC4 / IFC4X3

## Example 1: Complete IFC4 Building Project (Standalone)

Full project setup from scratch without Blender, suitable for automated workflows.

```python
import ifcopenshell
import ifcopenshell.api
from math import cos, sin, radians

# ─── Step 1: Create file ───
model = ifcopenshell.api.project.create_file(version="IFC4")

# ─── Step 2: Configure owner history ───
person = ifcopenshell.api.owner.add_person(model,
    identification="jdoe",
    family_name="Doe",
    given_name="Jane")
org = ifcopenshell.api.owner.add_organisation(model,
    identification="ACME",
    name="ACME Architecture BV")
person_org = ifcopenshell.api.owner.add_person_and_organisation(model,
    person=person, organisation=org)
app = ifcopenshell.api.owner.add_application(model,
    application_full_name="My BIM Pipeline",
    application_identifier="MyBIMPipeline",
    version="1.0")

# Enable auto owner history on entity creation
ifcopenshell.api.owner.settings.get_user = lambda ifc: person_org
ifcopenshell.api.owner.settings.get_application = lambda ifc: app

# ─── Step 3: Assign units (metric, millimetres) ───
length_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="LENGTHUNIT", prefix="MILLI")
area_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="AREAUNIT", prefix=None)
volume_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="VOLUMEUNIT", prefix=None)
angle_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="PLANEANGLEUNIT", prefix=None)
ifcopenshell.api.unit.assign_unit(model,
    units=[length_unit, area_unit, volume_unit, angle_unit])

# ─── Step 4: Create spatial hierarchy ───
project = model.by_type("IfcProject")[0]
project.Name = "Residential Building A"
project.LongName = "Residential Building A — Phase 1"

site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Plot 12")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Building A")
storey_0 = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
storey_1 = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="First Floor")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey_0, storey_1])

# ─── Step 5: Create representation contexts ───
model_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_ctx)
axis_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Axis",
    target_view="GRAPH_VIEW",
    parent=model_ctx)
plan_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Plan")
anno_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Plan",
    context_identifier="Annotation",
    target_view="PLAN_VIEW",
    parent=plan_ctx)

# ─── Step 6: Set up georeferencing ───
ifcopenshell.api.georeference.add_georeferencing(model)
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": cos(radians(0.0)),
        "XAxisOrdinate": sin(radians(0.0)),
        "Scale": 1.0
    })
ifcopenshell.api.georeference.edit_true_north(model,
    true_north=5.0)

# ─── Step 7: Edit file header ───
ifcopenshell.api.project.edit_header(model,
    editor="Jane Doe",
    organization="ACME Architecture BV",
    application="My BIM Pipeline",
    application_version="1.0")

# ─── Step 8: Save ───
model.write("residential_building_a.ifc")

print(f"Created {model.schema} project with {len(model.by_type('IfcBuildingStorey'))} storeys")
```

---

## Example 2: IFC2X3 Legacy Project

Creating an IFC2X3 project for legacy compliance.

```python
import ifcopenshell
import ifcopenshell.api

# IFC2X3 project
model = ifcopenshell.api.project.create_file(version="IFC2X3")

# Units (IFC2X3 uses same API)
ifcopenshell.api.unit.assign_unit(model)  # Default: mm, m², m³

# Spatial hierarchy
project = model.by_type("IfcProject")[0]
project.Name = "Legacy Project"

site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Site A")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Building 1")
storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Level 0")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey])

# Contexts
model_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_ctx)

# IFC2X3 georeferencing uses property sets (handled automatically)
ifcopenshell.api.georeference.add_georeferencing(model)
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:27700"},
    coordinate_operation={
        "Eastings": 530000.0,
        "Northings": 180000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": 1.0,
        "XAxisOrdinate": 0.0,
        "Scale": 1.0
    })

model.write("legacy_project.ifc")
```

---

## Example 3: IFC4X3 Infrastructure Project

Setting up a project for infrastructure (road, bridge, tunnel).

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.project.create_file(version="IFC4X3")

# Units: metres for infrastructure
length_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="LENGTHUNIT", prefix=None)  # metres
area_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="AREAUNIT", prefix=None)
volume_unit = ifcopenshell.api.unit.add_si_unit(model,
    unit_type="VOLUMEUNIT", prefix=None)
ifcopenshell.api.unit.assign_unit(model,
    units=[length_unit, area_unit, volume_unit])

# Spatial hierarchy (infrastructure uses IfcSite directly)
project = model.by_type("IfcProject")[0]
project.Name = "Highway A1 Extension"

site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="A1 Extension km 42-45")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])

# Representation context
model_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_ctx)

# Georeferencing with IfcMapConversionScaled (IFC4X3 feature)
ifcopenshell.api.georeference.add_georeferencing(model,
    ifc_class="IfcMapConversionScaled")
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 150000.0,
        "Northings": 400000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": 1.0,
        "XAxisOrdinate": 0.0,
        "Scale": 0.999960  # Grid scale factor for RD
    })

model.write("highway_a1.ifc")
```

---

## Example 4: Imperial Project (US)

Creating a project with US imperial units.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.project.create_file(version="IFC4")

# Imperial units
length_unit = ifcopenshell.api.unit.add_conversion_based_unit(model,
    name="foot")
area_unit = ifcopenshell.api.unit.add_conversion_based_unit(model,
    name="square foot")
volume_unit = ifcopenshell.api.unit.add_conversion_based_unit(model,
    name="cubic foot")
angle_unit = ifcopenshell.api.unit.add_conversion_based_unit(model,
    name="degree")
ifcopenshell.api.unit.assign_unit(model,
    units=[length_unit, area_unit, volume_unit, angle_unit])

# Spatial hierarchy
project = model.by_type("IfcProject")[0]
project.Name = "US Office Building"

site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Main Site")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Office Tower")
storey_1 = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="1st Floor")
storey_2 = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="2nd Floor")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey_1, storey_2])

# Context
model_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_ctx)

# US georeferencing (NAD83 State Plane)
ifcopenshell.api.georeference.add_georeferencing(model)
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:2263"},  # NAD83 / New York Long Island
    coordinate_operation={
        "Eastings": 980000.0,
        "Northings": 210000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": 1.0,
        "XAxisOrdinate": 0.0,
        "Scale": 1.0
    })

model.write("us_office_building.ifc")
```

---

## Example 5: Bonsai Project in Blender

Complete Bonsai workflow inside Blender.

```python
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

# Step 1: Create new project
bpy.ops.bim.create_project(template="metric_mm")

# Step 2: Access the live IFC model
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Step 3: Rename project
project = model.by_type("IfcProject")[0]
ifcopenshell.api.attribute.edit_attributes(model,
    product=project,
    attributes={"Name": "My Blender Project", "LongName": "My Building Project"})

# Step 4: Add georeferencing
ifcopenshell.api.georeference.add_georeferencing(model)
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": 1.0,
        "XAxisOrdinate": 0.0,
        "Scale": 1.0
    })

# Step 5: Save
bpy.ops.bim.save_project(filepath="/path/to/my_project.ifc")
```

---

## Example 6: Project Template Management (Library Append)

Transferring types and materials from a template library file to a new project.

```python
import ifcopenshell
import ifcopenshell.api

# Create new project
model = ifcopenshell.api.project.create_file(version="IFC4")
ifcopenshell.api.unit.assign_unit(model)

# Open template library
library = ifcopenshell.open("office_template_library.ifc")

# Transfer wall types from library to project
project = model.by_type("IfcProject")[0]
for wall_type in library.by_type("IfcWallType"):
    new_type = ifcopenshell.api.project.append_asset(model,
        library=library,
        element=wall_type)
    print(f"Imported: {new_type.Name}")

# Transfer material definitions
for material in library.by_type("IfcMaterial"):
    ifcopenshell.api.project.append_asset(model,
        library=library,
        element=material)

model.write("new_project_from_template.ifc")
```

---

## Example 7: Schema Migration (IFC2X3 to IFC4)

```python
import ifcopenshell
import ifcopenshell.util.schema
import ifcopenshell.validate
import logging

# Open legacy file
model_2x3 = ifcopenshell.open("legacy_building.ifc")
print(f"Source schema: {model_2x3.schema}")

# Migrate
migrator = ifcopenshell.util.schema.Migrator()
model_ifc4 = migrator.migrate(model_2x3, "IFC4")
print(f"Target schema: {model_ifc4.schema}")

# Validate the migrated file
logger = logging.getLogger("ifcopenshell.validate")
logger.setLevel(logging.WARNING)
handler = logging.StreamHandler()
logger.addHandler(handler)
ifcopenshell.validate.validate(model_ifc4, logger)

# Save
model_ifc4.write("migrated_building.ifc")
```

---

## Example 8: Querying Project Setup Information

```python
import ifcopenshell
import ifcopenshell.util.unit
import ifcopenshell.util.element

model = ifcopenshell.open("project.ifc")

# ─── Schema info ───
print(f"Schema: {model.schema}")

# ─── Project info ───
project = model.by_type("IfcProject")[0]
print(f"Project: {project.Name}")
print(f"Long name: {project.LongName}")

# ─── Unit info ───
units = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
if units:
    print(f"Length unit: {units}")

# ─── Spatial hierarchy ───
sites = model.by_type("IfcSite")
buildings = model.by_type("IfcBuilding")
storeys = model.by_type("IfcBuildingStorey")
print(f"Sites: {len(sites)}, Buildings: {len(buildings)}, Storeys: {len(storeys)}")

# ─── Georeferencing ───
map_conversions = model.by_type("IfcMapConversion")
if map_conversions:
    mc = map_conversions[0]
    print(f"Eastings: {mc.Eastings}, Northings: {mc.Northings}")
    crs = mc.TargetCRS
    if crs:
        print(f"CRS: {crs.Name}")
else:
    print("No georeferencing configured")
```

---

## Example 9: Minimal Valid IFC File

The absolute minimum required to create a schema-valid IFC4 file.

```python
import ifcopenshell
import ifcopenshell.api

# This single call creates IfcProject + units + contexts
model = ifcopenshell.api.project.create_file(version="IFC4")

# Add minimum spatial hierarchy (required by most validators)
project = model.by_type("IfcProject")[0]
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Site")
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])

model.write("minimal.ifc")
# Result: ~25 lines, valid IFC4 file
```

---

## Example 10: Bonsai Opening, Editing, and Saving

```python
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

# Open existing project
bpy.ops.bim.load_project(filepath="/path/to/existing_project.ifc")

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Inspect current state
print(f"Schema: {model.schema}")
print(f"Path: {IfcStore.path}")
print(f"Storeys: {len(model.by_type('IfcBuildingStorey'))}")

# Add a new storey
building = model.by_type("IfcBuilding")[0]
new_storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Roof Level")
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[new_storey])

# Save
bpy.ops.bim.save_project()
```
