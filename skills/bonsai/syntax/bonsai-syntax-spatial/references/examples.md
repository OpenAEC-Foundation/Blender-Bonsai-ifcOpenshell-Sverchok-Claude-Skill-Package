# Bonsai Spatial Structure — Working Code Examples

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Blender 4.2.0+
> All examples verified against Bonsai source code.

---

## Example 1: Complete Spatial Hierarchy from Scratch (Standalone IfcOpenShell)

```python
# IfcOpenShell v0.8+ — IFC4 — No Blender required
# Creates a complete spatial hierarchy and saves to file

import ifcopenshell
import ifcopenshell.api

# Step 1: Create IFC file with proper headers
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Step 2: Create project root
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Office Complex")

# Step 3: Assign SI units (metric)
ifcopenshell.api.run("unit.assign_unit", model)

# Step 4: Create representation contexts
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Step 5: Create spatial elements
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Main Campus")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
basement = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Basement")
ground = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
first = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")

# Step 6: Set storey elevations
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=basement, attributes={"Elevation": -3.0})
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=ground, attributes={"Elevation": 0.0})
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=first, attributes={"Elevation": 3.5})

# Step 7: Build hierarchy via IfcRelAggregates
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[basement, ground, first], relating_object=building)

# Step 8: Create spaces within storeys
lobby = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSpace", name="Lobby")
office_a = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSpace", name="Office A")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[lobby], relating_object=ground)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[office_a], relating_object=first)

# Step 9: Save
model.write("office_complex.ifc")
```

---

## Example 2: Place Elements in Spatial Containers (Standalone IfcOpenShell)

```python
# IfcOpenShell v0.8+ — IFC4
# Assumes completed spatial hierarchy from Example 1

# Create physical elements
wall_1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall 001")
wall_2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall 002")
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Main Entrance")
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")

# Assign walls to ground floor (IfcRelContainedInSpatialStructure)
# ALWAYS pass products as a list (v0.8+ requirement)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall_1, wall_2], relating_structure=ground)

# Assign door to ground floor
ifcopenshell.api.run("spatial.assign_container", model,
    products=[door], relating_structure=ground)

# Assign column to ground floor (primary containment)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=ground)

# Column spans to first floor: add reference (NOT containment)
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[column], relating_structure=first)
```

---

## Example 3: Query Spatial Relationships (Standalone IfcOpenShell)

```python
# IfcOpenShell v0.8+
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("office_complex.ifc")

# Get all storeys
storeys = model.by_type("IfcBuildingStorey")
for storey in storeys:
    print(f"Storey: {storey.Name}, Elevation: {storey.Elevation}")

# Get the container of a specific wall
wall = model.by_type("IfcWall")[0]
container = ifcopenshell.util.element.get_container(wall)
if container:
    print(f"Wall '{wall.Name}' is in '{container.Name}' ({container.is_a()})")

# Get all elements in a storey (direct children via aggregation)
ground = [s for s in storeys if s.Name == "Ground Floor"][0]
parts = ifcopenshell.util.element.get_parts(ground)
print(f"Ground floor has {len(parts)} direct spatial children")

# Get full decomposition tree (recursive)
decomposition = ifcopenshell.util.element.get_decomposition(ground)
print(f"Ground floor decomposition: {len(decomposition)} total elements")

# Get all contained elements (physical objects in this container)
for rel in ground.ContainsElements:
    for element in rel.RelatedElements:
        print(f"  Contained: {element.is_a()} '{element.Name}'")

# Find referenced structures for multi-storey elements
column = model.by_type("IfcColumn")[0]
if hasattr(column, "ReferencedInStructures"):
    for rel in column.ReferencedInStructures:
        print(f"  Referenced in: {rel.RelatingStructure.Name}")
```

---

## Example 4: Move Element Between Containers (Standalone IfcOpenShell)

```python
# IfcOpenShell v0.8+ — IFC4
# Moving a wall from ground floor to first floor

import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.open("office_complex.ifc")

wall = model.by_type("IfcWall")[0]
first_floor = [s for s in model.by_type("IfcBuildingStorey")
               if s.Name == "First Floor"][0]

# Check current container
current = ifcopenshell.util.element.get_container(wall)
print(f"Before: '{wall.Name}' in '{current.Name}'")

# Reassign to new container — automatically removes from old container
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=first_floor)

# Verify
new_container = ifcopenshell.util.element.get_container(wall)
print(f"After: '{wall.Name}' in '{new_container.Name}'")

model.write("office_complex_updated.ifc")
```

---

## Example 5: Bonsai Operators — Spatial Management (Inside Blender)

```python
# Bonsai v0.8.x — Inside Blender with active IFC project
import bpy
from bonsai.bim.ifc import IfcStore

# Guard: ensure IFC project is loaded
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Get the first storey
storey = model.by_type("IfcBuildingStorey")[0]

# Set as default container for new elements
bpy.ops.bim.set_default_container(container=storey.id())

# Load spatial decomposition tree in the UI
bpy.ops.bim.import_spatial_decomposition()

# Select all objects in a specific container
bpy.ops.bim.select_similar_container(is_recursive=True)

# Assign selected objects to a container
bpy.ops.bim.assign_container(container=storey.id())

# Remove container assignment from selected objects
bpy.ops.bim.remove_container()

# Copy selected objects to another container
first_floor = model.by_type("IfcBuildingStorey")[1]
bpy.ops.bim.copy_to_container(container=first_floor.id())
```

---

## Example 6: Bonsai Operators — Reference Structure (Inside Blender)

```python
# Bonsai v0.8.x — Inside Blender
import bpy
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

first_floor = model.by_type("IfcBuildingStorey")[1]

# Reference selected objects from a specific structure
# (for multi-storey elements like columns)
bpy.ops.bim.reference_from_provided_structure(
    structure=first_floor.id())

# Remove reference
bpy.ops.bim.reference_from_provided_structure(
    structure=first_floor.id(), dereference=True)

# Reference from ALL selected spatial structures to selected non-spatial objects
# (select both the containers and the elements, then run:)
bpy.ops.bim.reference_structure()
```

---

## Example 7: Generate IfcSpace from Walls (Inside Blender)

```python
# Bonsai v0.8.x — Inside Blender
import bpy
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Step 1: Set default container (required for space generation)
storey = model.by_type("IfcBuildingStorey")[0]
bpy.ops.bim.set_default_container(container=storey.id())

# Step 2a: Generate space from cursor position
# Place the 3D cursor inside a room enclosed by walls, then:
bpy.ops.bim.generate_space()

# Step 2b: Alternatively, generate spaces from selected walls
# Select all wall objects that enclose spaces, then:
bpy.ops.bim.generate_spaces_from_walls()
```

---

## Example 8: Delete Spatial Container (Inside Blender)

```python
# Bonsai v0.8.x — Inside Blender
import bpy
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Delete a storey (moves contained elements to parent or uncontains them)
# NEVER delete IfcProject — the operator enforces this
storey_to_delete = model.by_type("IfcBuildingStorey")[-1]
bpy.ops.bim.delete_container(container=storey_to_delete.id())

# Refresh the spatial decomposition tree
bpy.ops.bim.import_spatial_decomposition()
```

---

## Example 9: Container Visibility Control (Inside Blender)

```python
# Bonsai v0.8.x — Inside Blender
import bpy
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

storey = model.by_type("IfcBuildingStorey")[0]

# Hide a container and all its children
bpy.ops.bim.set_container_visibility(
    container=storey.id(),
    should_include_children=True,
    mode="HIDE")

# Show only this container (isolate)
bpy.ops.bim.set_container_visibility(
    container=storey.id(),
    should_include_children=True,
    mode="ISOLATE")

# Show all
bpy.ops.bim.set_container_visibility(
    container=storey.id(),
    should_include_children=True,
    mode="SHOW")

# Toggle IfcSpace visibility (WIRE <-> TEXTURED)
bpy.ops.bim.toggle_spatial_elements(is_visible=True)

# Toggle grid visibility
bpy.ops.bim.toggle_grids(is_visible=True)
```

---

## Example 10: Multi-Building Spatial Structure (Standalone IfcOpenShell)

```python
# IfcOpenShell v0.8+ — IFC4
# Campus with multiple buildings

import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="University Campus")
ifcopenshell.api.run("unit.assign_unit", model)

# One site
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="North Campus")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)

# Multiple buildings on one site
buildings = []
for name in ["Library", "Science Hall", "Admin Building"]:
    bldg = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcBuilding", name=name)
    buildings.append(bldg)

# Assign all buildings to site in one call
ifcopenshell.api.run("aggregate.assign_object", model,
    products=buildings, relating_object=site)

# Add storeys to each building
for bldg in buildings:
    storeys = []
    for i, floor_name in enumerate(["Ground Floor", "First Floor"]):
        storey = ifcopenshell.api.run("root.create_entity", model,
            ifc_class="IfcBuildingStorey", name=floor_name)
        ifcopenshell.api.run("attribute.edit_attributes", model,
            product=storey, attributes={"Elevation": float(i * 3.5)})
        storeys.append(storey)
    ifcopenshell.api.run("aggregate.assign_object", model,
        products=storeys, relating_object=bldg)

model.write("university_campus.ifc")
```

---

## Example 11: Walk the Spatial Tree (Standalone IfcOpenShell)

```python
# IfcOpenShell v0.8+ — Traverse and print the full spatial tree
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("university_campus.ifc")
project = model.by_type("IfcProject")[0]

def print_spatial_tree(element, indent=0):
    prefix = "  " * indent
    elevation = ""
    if hasattr(element, "Elevation") and element.Elevation is not None:
        elevation = f" (Elevation: {element.Elevation}m)"
    print(f"{prefix}{element.is_a()} '{element.Name}'{elevation}")

    # Get child spatial elements (IfcRelAggregates)
    children = ifcopenshell.util.element.get_parts(element)
    for child in children:
        print_spatial_tree(child, indent + 1)

    # Get contained physical elements (IfcRelContainedInSpatialStructure)
    if hasattr(element, "ContainsElements"):
        for rel in element.ContainsElements:
            for contained in rel.RelatedElements:
                print(f"{prefix}  [contained] {contained.is_a()} '{contained.Name}'")

print_spatial_tree(project)
# Output:
# IfcProject 'University Campus'
#   IfcSite 'North Campus'
#     IfcBuilding 'Library'
#       IfcBuildingStorey 'Ground Floor' (Elevation: 0.0m)
#       IfcBuildingStorey 'First Floor' (Elevation: 3.5m)
#     IfcBuilding 'Science Hall'
#       ...
```
