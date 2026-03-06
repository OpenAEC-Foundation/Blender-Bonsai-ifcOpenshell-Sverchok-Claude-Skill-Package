# ifcos-syntax-api — Working Code Examples

All examples verified against IfcOpenShell v0.8.4. Each example is self-contained and runnable.

---

## Example 1: Complete Minimal IFC File

Creates a valid IFC4 file with project, spatial hierarchy, a wall with geometry, properties, and materials.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

# --- Step 1: File and Project Setup ---
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Example Project")
ifcopenshell.api.run("unit.assign_unit", model)

# --- Step 2: Representation Contexts ---
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# --- Step 3: Spatial Hierarchy ---
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

# --- Step 4: Wall Type with Material ---
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="WT-200 Concrete")
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30/37", category="concrete")
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=concrete)

# --- Step 5: Wall Occurrence ---
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# --- Step 6: Wall Geometry ---
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.eye(4))

# --- Step 7: Spatial Containment ---
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# --- Step 8: Properties ---
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90",
    "ThermalTransmittance": 0.24,
})

# --- Step 9: Quantities ---
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall, name="Qto_WallBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 5.0,
    "Height": 3.0,
    "Width": 0.2,
    "GrossVolume": 3.0,
})

# --- Step 10: Save ---
model.write("complete_example.ifc")
print(f"Saved IFC4 file with {len(model)} entities")
```

---

## Example 2: Multi-Storey Building with Spatial References

Demonstrates aggregation hierarchy and spatial referencing for multi-storey elements.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Multi-Storey")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding")
ground = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
first = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[ground, first], relating_object=building)

# Wall contained in ground floor
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ground Wall")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=ground)

# Column spanning both floors: contained in ground, referenced in first
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Multi-Storey Column")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=ground)
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[column], relating_structure=first)

model.write("multi_storey.ifc")
```

---

## Example 3: Layered Wall Material Construction

Demonstrates composite wall materials with multiple layers.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Material Example")
ifcopenshell.api.run("unit.assign_unit", model)

# Create wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Cavity Wall 300mm")

# Create material layer set
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Cavity Wall 300", set_type="IfcMaterialLayerSet")

# Create individual materials
brick = ifcopenshell.api.run("material.add_material", model,
    name="Face Brick", category="brick")
cavity = ifcopenshell.api.run("material.add_material", model,
    name="Air Cavity", category=None)
insulation = ifcopenshell.api.run("material.add_material", model,
    name="Mineral Wool", category="glass")
block = ifcopenshell.api.run("material.add_material", model,
    name="Concrete Block", category="block")
plaster = ifcopenshell.api.run("material.add_material", model,
    name="Gypsum Plaster", category="gypsum")

# Add layers (order matters: exterior to interior)
l1 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=brick)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l1, attributes={"LayerThickness": 0.102})

l2 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=cavity)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l2, attributes={"LayerThickness": 0.050, "IsVentilated": True})

l3 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l3, attributes={"LayerThickness": 0.080})

l4 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=block)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l4, attributes={"LayerThickness": 0.100})

l5 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=plaster)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l5, attributes={"LayerThickness": 0.013})

# Assign layer set to wall type (best practice)
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)

# Create wall occurrences and assign type
wall1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall 001")
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall 002")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)

model.write("layered_wall.ifc")
```

---

## Example 4: Opening with Door Filling

Demonstrates creating a wall opening and filling it with a door.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Void Example")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Spatial structure
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Wall
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall with Door")
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# Opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")
opening_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=0.9, height=2.1, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_rep)

# Position opening within wall (offset 1m from wall start)
matrix = numpy.eye(4)
matrix[0, 3] = 1.0  # X offset
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=opening, matrix=matrix)

# Create void relationship
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Door filling the opening
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
door_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoorType", name="Single Swing Left",
    predefined_type="DOOR")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[door], relating_type=door_type)
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[door], relating_structure=storey)

model.write("wall_with_door.ifc")
```

---

## Example 5: Custom Mesh Geometry

Demonstrates creating arbitrary geometry from vertices and faces.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Mesh Example")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create custom element
element = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingElementProxy", name="Custom Shape")

# Define a simple box mesh (1m x 1m x 1m)
vertices = [
    [0, 0, 0], [1, 0, 0], [1, 1, 0], [0, 1, 0],  # bottom
    [0, 0, 1], [1, 0, 1], [1, 1, 1], [0, 1, 1],  # top
]
faces = [
    [0, 1, 2, 3],  # bottom
    [4, 7, 6, 5],  # top
    [0, 4, 5, 1],  # front
    [1, 5, 6, 2],  # right
    [2, 6, 7, 3],  # back
    [3, 7, 4, 0],  # left
]

mesh_rep = ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body, vertices=vertices, faces=faces)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=mesh_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=element)

model.write("custom_mesh.ifc")
```

---

## Example 6: Classification and Grouping

Demonstrates classifying elements and organizing them into groups.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Classification Example")
ifcopenshell.api.run("unit.assign_unit", model)

# Create elements
wall1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Fire Wall 1")
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Fire Wall 2")
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Fire Door 1")

# Add classification system
classification = ifcopenshell.api.run("classification.add_classification", model,
    classification="Uniclass 2015")

# Classify elements
ifcopenshell.api.run("classification.add_reference", model,
    products=[wall1, wall2], identification="Ss_20_10_30",
    name="Walls", classification=classification)
ifcopenshell.api.run("classification.add_reference", model,
    products=[door], identification="Pr_20_31_30",
    name="Doorsets", classification=classification)

# Group elements by fire zone
fire_zone = ifcopenshell.api.run("group.add_group", model,
    name="Fire Zone A", description="Ground floor fire compartment")
ifcopenshell.api.run("group.assign_group", model,
    products=[wall1, wall2, door], group=fire_zone)

model.write("classified_grouped.ifc")
```

---

## Example 7: Cost Schedule with Items

Demonstrates 5D BIM cost management.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Cost Example")
ifcopenshell.api.run("unit.assign_unit", model)

# Create cost schedule
schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Bill of Quantities", predefined_type="NOTDEFINED")

# Add cost items
foundations = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=foundations,
    attributes={"Name": "Foundations", "Identification": "1.0"})

# Sub-item
concrete_found = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=foundations)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=concrete_found,
    attributes={"Name": "Concrete Foundations", "Identification": "1.1"})

# Add quantity to cost item
quantity = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=concrete_found, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=quantity, attributes={"VolumeValue": 45.0})

# Add cost value
cost_value = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=concrete_found)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=cost_value,
    attributes={"AppliedValue": 150.0, "UnitBasis": None})

model.write("cost_schedule.ifc")
```

---

## Example 8: Work Schedule (4D BIM)

Demonstrates scheduling tasks and linking them to elements.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Schedule Example")
ifcopenshell.api.run("unit.assign_unit", model)

# Create work schedule
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Construction Programme")

# Add tasks
task_foundations = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Foundations",
    identification="A")
ifcopenshell.api.run("sequence.edit_task_time", model,
    task=task_foundations,
    attributes={
        "ScheduleStart": "2026-01-15",
        "ScheduleFinish": "2026-03-01",
    })

task_structure = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Superstructure",
    identification="B")
ifcopenshell.api.run("sequence.edit_task_time", model,
    task=task_structure,
    attributes={
        "ScheduleStart": "2026-03-01",
        "ScheduleFinish": "2026-06-01",
    })

# Create dependency: structure starts after foundations finish
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_foundations,
    related_process=task_structure)

# Link task to physical element
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Structure Wall")
ifcopenshell.api.run("sequence.assign_product", model,
    relating_product=wall, related_object=task_structure)

model.write("work_schedule.ifc")
```

---

## Example 9: Owner and Actor Information

Demonstrates setting up ownership history.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Owner Example")

# Create person
person = ifcopenshell.api.run("owner.add_person", model,
    identification="jdoe", family_name="Doe", given_name="John")

# Create organisation
org = ifcopenshell.api.run("owner.add_organisation", model,
    identification="ACME", name="ACME Engineering Ltd")

# Combine person and organisation
user = ifcopenshell.api.run("owner.add_person_and_organisation", model,
    person=person, organisation=org)

# Set as active user (affects OwnerHistory on new entities)
ifcopenshell.api.run("owner.set_user", model, user=user)

model.write("owner_info.ifc")
```

---

## Example 10: Visual Styles

Demonstrates adding surface color styles to elements.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Style Example")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Styled Wall")
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)

# Create and assign style
style = ifcopenshell.api.run("style.add_style", model, name="Warm Concrete")
ifcopenshell.api.run("style.add_surface_style", model,
    style=style,
    attributes={
        "SurfaceColour": {"Red": 0.85, "Green": 0.75, "Blue": 0.65},
        "Transparency": 0.0,
    })
ifcopenshell.api.run("style.assign_representation_styles", model,
    shape_representation=representation, styles=[style])

model.write("styled_elements.ifc")
```

---

## Example 11: MEP System

Demonstrates creating a building system with connected elements.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="MEP Example")
ifcopenshell.api.run("unit.assign_unit", model)

# Create distribution system
hvac_system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=hvac_system, attributes={"Name": "HVAC Supply Air"})

# Create duct segments
duct1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Supply Duct 001")
duct2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Supply Duct 002")

# Assign to system
ifcopenshell.api.run("system.assign_system", model,
    products=[duct1, duct2], system=hvac_system)

model.write("mep_system.ifc")
```

---

## Example 12: Copy and Reassign Elements

Demonstrates duplicating and reclassifying entities.

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Copy Example")
ifcopenshell.api.run("unit.assign_unit", model)

# Create original wall with properties
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Original Wall")
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "FireRating": "REI60",
})

# Copy: duplicates entity with psets, placement, materials (NOT geometry)
wall_copy = ifcopenshell.api.run("root.copy_class", model, product=wall)
print(wall_copy.Name)  # "Original Wall" (inherited)

# Reassign class: convert wall copy to slab (retains geometry/relationships)
slab = ifcopenshell.api.run("root.reassign_class", model,
    product=wall_copy, ifc_class="IfcSlab", predefined_type="FLOOR")
print(slab.is_a())  # "IfcSlab"

# Safe removal
ifcopenshell.api.run("root.remove_product", model, product=slab)

model.write("copy_reassign.ifc")
```

---

## Sources

- https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- https://blenderbim.org/docs-python/ifcopenshell-python/code_examples.html
