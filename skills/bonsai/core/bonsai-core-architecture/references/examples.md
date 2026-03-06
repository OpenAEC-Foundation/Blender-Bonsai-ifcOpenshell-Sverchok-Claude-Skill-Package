# Bonsai Core Architecture — Working Code Examples

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11
> **Run all examples via**: `blender --background --python script.py` (with Bonsai enabled)

---

## Example 1: Access IFC Model and Query Elements

```python
"""Query all walls in the current IFC project."""
import bpy
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool

# ALWAYS check for None first
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")

# Query by IFC class
walls = model.by_type("IfcWall")
print(f"Found {len(walls)} walls")

for wall in walls:
    # Get linked Blender object
    obj = tool.Ifc.get_object(wall)
    if obj:
        print(f"  {wall.Name} (GlobalId: {wall.GlobalId}) -> Blender: {obj.name}")
    else:
        print(f"  {wall.Name} (GlobalId: {wall.GlobalId}) -> No Blender object")
```

---

## Example 2: Get IFC Entity from Selected Blender Object

```python
"""Get IFC data for the currently selected object."""
import bpy
import bonsai.tool as tool

obj = bpy.context.active_object
if obj is None:
    raise RuntimeError("No active object selected.")

entity = tool.Ifc.get_entity(obj)
if entity is None:
    print(f"'{obj.name}' is not linked to IFC")
else:
    print(f"IFC Class: {entity.is_a()}")
    print(f"Name: {entity.Name}")
    print(f"GlobalId: {entity.GlobalId}")
    print(f"Step ID: {entity.id()}")
```

---

## Example 3: Create a Wall with Full Geometry

```python
"""Create an IfcWall with parametric geometry in Bonsai v0.8.4."""
import bpy
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

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

# Step 5: Create wall geometry (5m long, 3m high, 200mm thick)
wall_representation = ifcopenshell.api.run(
    "geometry.add_wall_representation", model,
    context=body_context,
    length=5.0,
    height=3.0,
    thickness=0.2)

# Step 6: Assign the representation to the wall
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

## Example 4: Build Spatial Structure from Scratch

```python
"""Build a complete IFC spatial hierarchy."""
import ifcopenshell
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

project = model.by_type("IfcProject")[0]

# Build spatial decomposition tree (uses IfcRelAggregates)
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Construction Site")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)

building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Building")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)

ground_floor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[ground_floor], relating_object=building)

first_floor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[first_floor], relating_object=building)

print("Spatial structure created:")
print(f"  Project: {project.Name}")
print(f"    Site: {site.Name}")
print(f"      Building: {building.Name}")
print(f"        Storey: {ground_floor.Name}")
print(f"        Storey: {first_floor.Name}")
```

---

## Example 5: Add Property Sets to an Element

```python
"""Add standard and custom property sets to a wall."""
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

wall = model.by_type("IfcWall")[0]

# Add standard Pset_WallCommon
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall,
    name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset,
    properties={
        "FireRating": "2HR",
        "ThermalTransmittance": 0.25,
        "LoadBearing": True,
        "Combustible": False,
    })

# Add custom property set (use company prefix, NEVER Pset_ or Qto_)
custom_pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall,
    name="Acme_StructuralData")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=custom_pset,
    properties={
        "LoadCapacity": 500.0,
        "SustainableSource": True,
    })

# Read back all property sets
all_psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, props in all_psets.items():
    print(f"\n{pset_name}:")
    for key, value in props.items():
        if key != "id":
            print(f"  {key}: {value}")
```

---

## Example 6: Assign IFC Class via Operator

```python
"""Assign IfcWall class to a Blender mesh object."""
import bpy

# Create a mesh object
bpy.ops.mesh.primitive_cube_add(size=1)
wall_obj = bpy.context.active_object
wall_obj.dimensions = (5.0, 0.2, 3.0)  # length, width, height
bpy.ops.object.transform_apply(scale=True)

# Assign IFC class via Bonsai operator
# ALWAYS check poll() before calling
if bpy.ops.bim.assign_class.poll():
    bpy.ops.bim.assign_class(ifc_class="IfcWall", predefined_type="STANDARD")
    print(f"Assigned IfcWall to {wall_obj.name}")
else:
    print("Cannot assign class — check Bonsai is active and object is selected")
```

---

## Example 7: Create Opening in a Wall

```python
"""Create a door opening in a wall using feature.add_feature()."""
import bpy
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Get the wall
wall = model.by_type("IfcWall")[0]

# Create opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement",
    name="Door Opening")

# Add void to wall — ALWAYS use feature.add_feature(), NEVER void.add_opening()
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening,
    element=wall)

# Create door and fill the opening
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor",
    name="Door-001")

ifcopenshell.api.run("void.add_filling", model,
    opening=opening,
    element=door)

print(f"Opening {opening.GlobalId} added to wall {wall.Name}")
print(f"Door {door.Name} fills opening")
```

---

## Example 8: Type Assignment with Material Layers

```python
"""Create a wall type with material layers and assign to instances."""
import ifcopenshell
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Create wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType",
    name="200mm Insulated Wall")

# Create material layer set
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="200mm Insulated Wall", set_type="IfcMaterialLayerSet")

concrete = ifcopenshell.api.run("material.add_material", model, name="Concrete C25")
insulation = ifcopenshell.api.run("material.add_material", model, name="Mineral Wool")

layer_c = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
layer_i = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)

ifcopenshell.api.run("material.edit_layer", model,
    layer=layer_c, attributes={"LayerThickness": 0.200})
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer_i, attributes={"LayerThickness": 0.050})

# Assign material to type
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=layer_set)

# Assign type to wall instances — ALWAYS use list for related_objects
walls = model.by_type("IfcWall")
if walls:
    ifcopenshell.api.run("type.assign_type", model,
        relating_type=wall_type,
        related_objects=walls,  # MUST be a list
        should_map_representations=True)
    print(f"Assigned type '{wall_type.Name}' to {len(walls)} walls")
```

---

## Example 9: Custom Operator Following Bonsai Pattern

```python
"""Custom Bonsai operator following the three-layer architecture pattern."""
import bpy
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool

class BIM_OT_print_wall_summary(bpy.types.Operator):
    """Print summary of all walls in the IFC model"""
    bl_idname = "bim.print_wall_summary"
    bl_label = "Print Wall Summary"
    bl_options = {"REGISTER", "UNDO"}

    def execute(self, context):
        return IfcStore.execute_ifc_operator(self, context)

    def _execute(self, context):
        model = tool.Ifc.get()
        if model is None:
            self.report({"ERROR"}, "No IFC project loaded")
            return {"CANCELLED"}

        walls = model.by_type("IfcWall")
        for wall in walls:
            obj = tool.Ifc.get_object(wall)
            obj_name = obj.name if obj else "N/A"
            self.report({"INFO"}, f"{wall.Name} ({wall.is_a()}) -> {obj_name}")

        self.report({"INFO"}, f"Total: {len(walls)} walls")
        return {"FINISHED"}

# Registration
def register():
    bpy.utils.register_class(BIM_OT_print_wall_summary)

def unregister():
    bpy.utils.unregister_class(BIM_OT_print_wall_summary)
```

---

## Example 10: Read and Navigate Spatial Structure

```python
"""Navigate the IFC spatial hierarchy tree."""
import ifcopenshell
import ifcopenshell.util.element
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

def print_spatial_tree(element, indent=0):
    """Recursively print the spatial decomposition tree."""
    prefix = "  " * indent
    ifc_class = element.is_a()
    name = element.Name or "(unnamed)"
    print(f"{prefix}{ifc_class}: {name}")

    # Get children via IfcRelAggregates (spatial decomposition)
    if hasattr(element, "IsDecomposedBy"):
        for rel in element.IsDecomposedBy:
            for child in rel.RelatedObjects:
                print_spatial_tree(child, indent + 1)

    # Get contained elements via IfcRelContainedInSpatialStructure
    if hasattr(element, "ContainsElements"):
        for rel in element.ContainsElements:
            for contained in rel.RelatedElements:
                contained_class = contained.is_a()
                contained_name = contained.Name or "(unnamed)"
                print(f"{prefix}  [{contained_class}] {contained_name}")

project = model.by_type("IfcProject")[0]
print_spatial_tree(project)
```

---

## Example 11: Sync Object Placement After Move

```python
"""Move a Blender object and sync placement to IFC."""
import bpy

obj = bpy.context.active_object
if obj is None:
    raise RuntimeError("No active object selected.")

# Move the object in Blender
obj.location = (5.0, 2.0, 0.0)

# ALWAYS sync to IFC after direct matrix/location changes
if bpy.ops.bim.edit_object_placement.poll():
    bpy.ops.bim.edit_object_placement()
    print(f"Placement synced for {obj.name}")
else:
    print("Cannot edit placement — object may not be linked to IFC")
```

---

## Example 12: Quantity Sets

```python
"""Add quantity sets to a wall element."""
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

wall = model.by_type("IfcWall")[0]

# Create quantity set
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall,
    name="Qto_WallBaseQuantities")

# Edit quantities — type detection is automatic
ifcopenshell.api.run("pset.edit_qto", model,
    qto=qto,
    properties={
        "Length": 5.0,        # float + "Length" -> IfcQuantityLength
        "Height": 3.0,        # float + "Height" -> IfcQuantityLength
        "Width": 0.2,         # float (no keyword match) -> IfcQuantityLength
        "NetSideArea": 15.0,  # float + "Area" -> IfcQuantityArea
        "NetVolume": 3.0,     # float + "Volume" -> IfcQuantityVolume
        "GrossFootprintArea": 1.0,  # float + "Area" -> IfcQuantityArea
    })

print(f"Quantity set added to {wall.Name}")
```

---

## Example 13: Save IFC Project

```python
"""Save the current IFC project to disk."""
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Direct file write
model.write(IfcStore.path)
print(f"Saved to: {IfcStore.path}")

# Or via Bonsai operator (includes hooks and notifications)
import bpy
bpy.ops.bim.save_project(filepath="/path/to/project.ifc")
```
