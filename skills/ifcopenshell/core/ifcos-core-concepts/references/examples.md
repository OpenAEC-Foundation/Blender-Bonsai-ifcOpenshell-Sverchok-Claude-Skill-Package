# IFC Core Concepts — Working Code Examples

## 1. Opening and Inspecting an IFC File

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell

# Open an existing file
model = ifcopenshell.open("building.ifc")

# Detect schema version
print(f"Schema: {model.schema}")  # "IFC2X3", "IFC4", or "IFC4X3"

# Get the project
project = model.by_type("IfcProject")[0]
print(f"Project: {project.Name}")

# Count elements by type
walls = model.by_type("IfcWall")
print(f"Walls: {len(walls)}")

# Get all elements (includes all subtypes of IfcElement)
all_elements = model.by_type("IfcElement")
print(f"Total elements: {len(all_elements)}")
```

## 2. Creating a Complete IFC File from Scratch

```python
# Schema: IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

# Create new file with explicit schema
model = ifcopenshell.file(schema="IFC4")

# Create project (ALWAYS do this first)
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Building Project")

# Assign units (ALWAYS set units before creating geometry)
ifcopenshell.api.run("unit.assign_unit", model)

# Create representation context (REQUIRED for geometry)
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Build spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Construction Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])

# Create a wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Add wall representation
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)

# Set wall placement
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.eye(4))

# Assign wall to storey
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# Save
model.write("new_building.ifc")
```

## 3. IFC4X3 Infrastructure Project

```python
# Schema: IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4X3")

# Create project and units
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Highway Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Create site
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Highway Corridor")

# Create road (IFC4X3 entity — does NOT exist in IFC2X3/IFC4)
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway A1")

# Create road parts
road_part1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoadPart", name="Section 1 - Straight")
road_part2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoadPart", name="Section 2 - Curve")

# Build spatial hierarchy
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[road])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=road, products=[road_part1, road_part2])

# Create infrastructure elements
pavement = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPavement", name="Asphalt Layer")
kerb = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcKerb", name="Road Kerb")

# Assign to road part
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=road_part1, products=[pavement, kerb])

model.write("highway_project.ifc")
```

## 4. Traversing Spatial Hierarchy

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")

def print_spatial_tree(element, depth=0):
    """Recursively print the spatial decomposition tree."""
    indent = "  " * depth
    print(f"{indent}{element.is_a()}: {element.Name}")

    # Get child spatial elements via IfcRelAggregates
    if hasattr(element, "IsDecomposedBy"):
        for rel in element.IsDecomposedBy:
            for child in rel.RelatedObjects:
                print_spatial_tree(child, depth + 1)

    # Get contained elements via IfcRelContainedInSpatialStructure
    if hasattr(element, "ContainsElements"):
        for rel in element.ContainsElements:
            for contained in rel.RelatedElements:
                print(f"{indent}  [{contained.is_a()}] {contained.Name}")

# Start from project
project = model.by_type("IfcProject")[0]
print_spatial_tree(project)
```

## 5. Working with Element Types (Occurrence/Type Pattern)

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")

# Get all wall types
wall_types = model.by_type("IfcWallType")
for wt in wall_types:
    print(f"Wall Type: {wt.Name}")

    # Get all occurrences of this type
    occurrences = ifcopenshell.util.element.get_types(wt)
    print(f"  Occurrences: {len(occurrences)}")

# For a specific wall, find its type
wall = model.by_type("IfcWall")[0]
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    print(f"Wall '{wall.Name}' has type '{wall_type.Name}'")
else:
    print(f"Wall '{wall.Name}' has no type assigned")
```

```python
# Schema: IFC4
# Creating and assigning types
model = ifcopenshell.file(schema="IFC4")
# ... (project, site, building, storey setup omitted for brevity)

# Create a wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Concrete Wall 200mm",
    predefined_type="SOLIDWALL")

# Add properties to the type (shared by all occurrences)
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall_type, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": False,
    "LoadBearing": True,
    "FireRating": "REI120"
})

# Create wall occurrences and assign the type
wall1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 002")

ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)
```

## 6. Navigating Relationships

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Get spatial container
container = ifcopenshell.util.element.get_container(wall)
print(f"Container: {container.is_a()} - {container.Name}")

# Get material
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material: {material.is_a()} - {material.Name}")

# Get property sets
psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, properties in psets.items():
    print(f"\nProperty Set: {pset_name}")
    for prop_name, prop_value in properties.items():
        if prop_name != "id":  # Skip the internal ID
            print(f"  {prop_name}: {prop_value}")

# Get openings in a wall
if hasattr(wall, "HasOpenings"):
    for rel in wall.HasOpenings:
        opening = rel.RelatedOpeningElement
        print(f"Opening: {opening.Name}")

        # Check if opening is filled (has door/window)
        if hasattr(opening, "HasFillings"):
            for fill_rel in opening.HasFillings:
                filling = fill_rel.RelatedBuildingElement
                print(f"  Filled by: {filling.is_a()} - {filling.Name}")
```

## 7. Version-Aware Element Collection

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema

def get_building_elements(model):
    """Get all building/built elements regardless of schema version."""
    if model.schema == "IFC4X3":
        return model.by_type("IfcBuiltElement")
    return model.by_type("IfcBuildingElement")

def get_door_type_class(model):
    """Return the correct door type class name for the schema."""
    if model.schema == "IFC2X3":
        return "IfcDoorStyle"
    return "IfcDoorType"

def get_window_type_class(model):
    """Return the correct window type class name for the schema."""
    if model.schema == "IFC2X3":
        return "IfcWindowStyle"
    return "IfcWindowType"

# Usage
elements = get_building_elements(model)
print(f"Building elements ({schema}): {len(elements)}")

door_type_class = get_door_type_class(model)
door_types = model.by_type(door_type_class)
print(f"Door types ({door_type_class}): {len(door_types)}")
```

## 8. Working with Openings and Fillings

```python
# Schema: IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")
# ... (project/spatial setup omitted for brevity)

# Create a wall
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Exterior Wall")

# Create an opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# Create the void relationship (IfcRelVoidsElement)
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Create a door and fill the opening (IfcRelFillsElement)
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Main Entrance Door")
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)
```

## 9. Property Set Operations

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Read all property sets
psets = ifcopenshell.util.element.get_psets(wall)

# Read a specific property
is_external = psets.get("Pset_WallCommon", {}).get("IsExternal")
fire_rating = psets.get("Pset_WallCommon", {}).get("FireRating")

# Add a new property set
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Custom_Properties")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "CustomProperty1": "Value1",
    "CustomNumeric": 42.0,
    "CustomBoolean": True
})

# Add quantity set
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall, name="Qto_WallBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 5.0,
    "Height": 3.0,
    "Width": 0.2,
    "GrossVolume": 3.0,
    "NetVolume": 2.8,
    "GrossSideArea": 15.0,
    "NetSideArea": 13.5
})
```

## 10. Material Assignment

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.file(schema="IFC4")
# ... (project/spatial setup omitted for brevity)

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Layered Wall")

# Simple single material
material = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30/37")
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material)

# Query material
mat = ifcopenshell.util.element.get_material(wall)
print(f"Material: {mat.Name}")
```

## 11. Entity Lookup Methods

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell

model = ifcopenshell.open("building.ifc")

# By type — returns tuple of all matching entities (includes subtypes)
walls = model.by_type("IfcWall")

# By STEP ID — returns single entity
entity = model.by_id(42)

# By GlobalId — returns single entity
entity = model.by_guid("2O2Fr$t4X7Z8xBnfrq9Cl7")

# Get all entities referencing a given entity (inverse references)
inverse = model.get_inverse(wall)

# Traverse all entities referenced by a given entity (recursive)
referenced = model.traverse(wall)

# Get total entity count
print(f"Total entities: {len(list(model))}")
```

## 12. Schema Introspection

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
import ifcopenshell.ifcopenshell_wrapper

# Get schema definition object
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4X3")

# Check if an entity exists in the schema
def entity_exists(schema_name, entity_name):
    """Return True if entity_name exists in the given schema."""
    schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
    try:
        schema.declaration_by_name(entity_name)
        return True
    except RuntimeError:
        return False

# Examples
print(entity_exists("IFC2X3", "IfcRoad"))      # False
print(entity_exists("IFC4X3", "IfcRoad"))      # True
print(entity_exists("IFC4", "IfcSpatialZone"))  # True
print(entity_exists("IFC2X3", "IfcSpatialZone"))  # False

# List all attributes of an entity
entity_decl = schema.declaration_by_name("IfcWall")
for attr in entity_decl.all_attributes():
    print(f"  {attr.name()}: {attr.type_of_attribute()}")

# Get supertype chain
def get_supertype_chain(schema_name, entity_name):
    """Return list of supertypes from entity up to IfcRoot."""
    schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
    entity = schema.declaration_by_name(entity_name)
    chain = []
    parent = entity.supertype()
    while parent:
        chain.append(parent.name())
        parent = parent.supertype()
    return chain

print(get_supertype_chain("IFC4", "IfcWall"))
# ['IfcBuildingElement', 'IfcElement', 'IfcProduct', 'IfcObject',
#  'IfcObjectDefinition', 'IfcRoot']
```
