# IFC Model Creation — Working Code Examples

All examples use `ifcopenshell.api.run()` and follow the required creation order: file → project → units → contexts → spatial hierarchy → types → elements → properties.

---

## Example 1: Minimal IFC4 Model with Single Wall

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

# 1. Create file
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# 2. Create project
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Minimal Project")

# 3. Assign SI units (meters, radians)
ifcopenshell.api.run("unit.assign_unit", model)

# 4. Geometric context
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# 5. Spatial hierarchy
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

# 6. Create wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=6.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# 7. Write output
model.write("minimal_wall.ifc")
```

---

## Example 2: IFC2X3 Model with OwnerHistory

```python
# IfcOpenShell — IFC2X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC2X3")

# REQUIRED for IFC2X3: set up owner BEFORE creating any entities
person = ifcopenshell.api.run("owner.add_person", model,
    identification="jsmith", family_name="Smith", given_name="Jane")
org = ifcopenshell.api.run("owner.add_organisation", model,
    identification="ACME", name="ACME Architects")
user = ifcopenshell.api.run("owner.add_person_and_organisation", model,
    person=person, organisation=org)
application = ifcopenshell.api.run("owner.add_application", model)
ifcopenshell.api.run("owner.set_user", model, user=user)

# Now entities get OwnerHistory automatically
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Legacy Project")
ifcopenshell.api.run("unit.assign_unit", model)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
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

model.write("legacy_ifc2x3.ifc")
```

---

## Example 3: Multi-Storey Building with Elevation Placement

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy as np

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Office Building")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Main Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Block")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)

# Create storeys at different elevations
storey_defs = [
    ("Basement", -3.0),
    ("Ground Floor", 0.0),
    ("First Floor", 3.5),
    ("Second Floor", 7.0),
    ("Roof", 10.5),
]

storeys = {}
storey_objects = []
for name, elevation in storey_defs:
    s = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcBuildingStorey", name=name)
    matrix = np.eye(4)
    matrix[2][3] = elevation
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=s, matrix=matrix)
    storeys[name] = s
    storey_objects.append(s)

ifcopenshell.api.run("aggregate.assign_object", model,
    products=storey_objects, relating_object=building)

model.write("multi_storey.ifc")
```

---

## Example 4: Wall Type with Material Layer Set

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes model, body context, and storey exist (see Example 1)

# Create wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="WT-200 External Cavity")

# Define material layer set
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="External Cavity Wall", set_type="IfcMaterialLayerSet")

brick = ifcopenshell.api.run("material.add_material", model,
    name="Facing Brick", category="brick")
insulation = ifcopenshell.api.run("material.add_material", model,
    name="Mineral Wool", category="mineral wool")
block = ifcopenshell.api.run("material.add_material", model,
    name="Concrete Block", category="concrete")
plaster = ifcopenshell.api.run("material.add_material", model,
    name="Gypsum Plaster", category="gypsum")

# Add layers in order (outside to inside)
for mat, thickness in [(brick, 0.102), (insulation, 0.100),
                        (block, 0.100), (plaster, 0.013)]:
    layer = ifcopenshell.api.run("material.add_layer", model,
        layer_set=layer_set, material=mat)
    ifcopenshell.api.run("material.edit_layer", model,
        layer=layer, attributes={"LayerThickness": thickness})

# Assign material set to type
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)

# Create wall occurrences with the type
wall1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="North Wall")
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="South Wall")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)
```

---

## Example 5: Door with Opening in Wall

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy as np

# Assumes model, body context, storey, and wall exist (see Example 1)

# Create opening element with geometry
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening 001")
opening_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=0.9, height=2.1, thickness=0.3)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_rep)

# Position opening within wall (offset 2m from wall start)
opening_matrix = np.eye(4)
opening_matrix[0][3] = 2.0  # X offset along wall
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=opening, matrix=opening_matrix)

# Cut opening in wall
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Create door type
door_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoorType", name="DT-900 Single Swing")

# Create door and fill opening
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[door], relating_type=door_type)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[door])
```

---

## Example 6: Slab with Polyline Footprint and Column

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes model, body context, and storey exist (see Example 1)

# Floor slab with L-shaped footprint
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Floor Slab", predefined_type="FLOOR")
slab_rep = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context=body, depth=0.25,
    polyline=[(0.0, 0.0), (12.0, 0.0), (12.0, 6.0),
              (6.0, 6.0), (6.0, 10.0), (0.0, 10.0)])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=slab_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=slab)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[slab])

# Rectangular column
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")
profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcRectangleProfileDef", XDim=0.4, YDim=0.4)
col_rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=col_rep)

import numpy as np
col_matrix = np.eye(4)
col_matrix[0][3] = 6.0  # X position
col_matrix[1][3] = 3.0  # Y position
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=column, matrix=col_matrix)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[column])
```

---

## Example 7: Property Sets and Quantity Sets

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes model and wall exist (see Example 1)

# Standard property set
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "Reference": "WT-200",
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90",
    "ThermalTransmittance": 0.28,
    "AcousticRating": "Rw 52 dB"
})

# Quantity set
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall, name="Qto_WallBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 6.0,
    "Height": 3.0,
    "Width": 0.2,
    "GrossSideArea": 18.0,
    "NetSideArea": 15.9,
    "GrossVolume": 3.6,
    "NetVolume": 3.18
})

# Custom property set (use project prefix, NOT "Pset_")
custom = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="ACME_ConstructionData")
ifcopenshell.api.run("pset.edit_pset", model, pset=custom, properties={
    "CostCode": "STR-W-001",
    "PlannedInstallDate": "2026-06-15",
    "Contractor": "BuildCo Ltd"
})
```

---

## Example 8: Circular Column with I-Profile Beam

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy as np

# Assumes model, body context, and storey exist (see Example 1)

# Circular column
circ_col = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Circular Column")
circ_profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcCircleProfileDef", Radius=0.15)
circ_rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=circ_profile, depth=3.5)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=circ_col, representation=circ_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=circ_col)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[circ_col])

# Steel I-beam
beam = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="Beam B1")
i_profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcIShapeProfileDef",
    OverallWidth=0.200, OverallDepth=0.400,
    WebThickness=0.010, FlangeThickness=0.016)
beam_rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=i_profile, depth=6.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=beam, representation=beam_rep)

# Position beam at storey height, rotated 90 degrees (horizontal)
import math
angle = math.radians(90)
beam_matrix = np.array([
    [1, 0, 0, 0],
    [0, math.cos(angle), -math.sin(angle), 0],
    [0, math.sin(angle),  math.cos(angle), 3.5],
    [0, 0, 0, 1]
], dtype=float)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=beam, matrix=beam_matrix)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[beam])
```

---

## Example 9: Complete Small Building with Multiple Elements

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy as np

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Small House")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Plot 42")
bldg = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="House")
gf = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[bldg], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[gf], relating_object=bldg)

# Create 4 walls forming a room (8m x 6m)
wall_defs = [
    ("North Wall", 8.0, 0.0, 0.0, 0),
    ("East Wall",  6.0, 8.0, 0.0, 90),
    ("South Wall", 8.0, 8.0, 6.0, 180),
    ("West Wall",  6.0, 0.0, 6.0, 270),
]

import math
walls = []
for name, length, x, y, angle_deg in wall_defs:
    w = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=name)
    rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
        context=body, length=length, height=3.0, thickness=0.2)
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=w, representation=rep)

    angle = math.radians(angle_deg)
    matrix = np.array([
        [math.cos(angle), -math.sin(angle), 0, x],
        [math.sin(angle),  math.cos(angle), 0, y],
        [0, 0, 1, 0],
        [0, 0, 0, 1]
    ], dtype=float)
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=w, matrix=matrix)
    walls.append(w)

ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=gf, products=walls)

# Floor slab
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Slab", predefined_type="FLOOR")
slab_rep = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context=body, depth=0.2,
    polyline=[(0.0, 0.0), (8.0, 0.0), (8.0, 6.0), (0.0, 6.0)])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=slab_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=slab)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=gf, products=[slab])

model.write("small_house.ifc")
```
