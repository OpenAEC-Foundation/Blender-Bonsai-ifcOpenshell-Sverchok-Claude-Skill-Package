# Bonsai Modeling — Code Examples

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x | Blender 4.2.0+
> **Execution**: Run via `blender --background --python script.py` or from Blender's Python console.
> **NEVER** run `bonsai.*` imports from system Python outside Blender.

---

## Example 1: Complete Wall Creation Pipeline

Creates an IfcWall with type, material layers, geometry, and spatial assignment.

```python
"""
Complete wall creation in Bonsai v0.8.x.
Run from Blender Python console or: blender --background --python this_script.py
"""
import bpy
import numpy as np
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

# Step 1: Get the live IFC model
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")

# Step 2: Ensure representation context exists
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

# Step 3: Create wall type with material layers
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType",
    predefined_type="STANDARD",
    name="200mm Concrete Wall")

# Step 4: Configure material layer set
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30", category="concrete")

layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="200mm Concrete", set_type="IfcMaterialLayerSet")

layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer, attributes={"LayerThickness": 0.200})

ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=layer_set)

# Step 5: Create wall occurrence
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# Step 6: Assign type
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall],
    relating_type=wall_type,
    should_map_representations=True)

# Step 7: Create and assign geometry
wall_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context,
    length=5.0,
    height=3.0,
    thickness=0.2)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_repr)

# Step 8: Place in spatial structure
storey = model.by_type("IfcBuildingStorey")[0]
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall],
    relating_structure=storey)

# Step 9: Set placement (5m along X axis)
matrix = np.eye(4)
matrix[0, 3] = 5.0  # X = 5m
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix, is_si=True)

# Step 10: Save
model.write(IfcStore.path)
print(f"Wall created: {wall.GlobalId}")
```

---

## Example 2: Steel Column with Profile Set

Creates an IfcColumn with an I-shape profile material set.

```python
"""
Steel column creation with IfcMaterialProfileSet in Bonsai v0.8.x.
"""
import numpy as np
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

# Create column type
column_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType",
    predefined_type="COLUMN",
    name="HEA200 Steel Column")

# Create material and profile
steel = ifcopenshell.api.run("material.add_material", model,
    name="S355 Steel", category="steel")

profile = model.create_entity("IfcIShapeProfileDef",
    ProfileType="AREA",
    ProfileName="HEA200",
    OverallWidth=0.200,
    OverallDepth=0.190,
    WebThickness=0.0065,
    FlangeThickness=0.010)

# Create profile set
profile_set = ifcopenshell.api.run("material.add_material_set", model,
    name="HEA200 Steel", set_type="IfcMaterialProfileSet")

ifcopenshell.api.run("material.add_profile", model,
    profile_set=profile_set, material=steel, profile=profile)

ifcopenshell.api.run("material.assign_material", model,
    products=[column_type], material=profile_set)

# Create column occurrence
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Col-A1")

ifcopenshell.api.run("type.assign_type", model,
    related_objects=[column],
    relating_type=column_type,
    should_map_representations=True)

# Create profile-based geometry (3m height)
col_repr = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body_context,
    profile=profile,
    depth=3.0,
    cardinal_point=5)  # mid-depth centre

ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=col_repr)

# Place in storey at grid intersection
storey = model.by_type("IfcBuildingStorey")[0]
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column],
    relating_structure=storey)

matrix = np.eye(4)
matrix[0, 3] = 0.0   # X = 0m (grid A)
matrix[1, 3] = 0.0   # Y = 0m (grid 1)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=column, matrix=matrix, is_si=True)

model.write(IfcStore.path)
print(f"Column created: {column.GlobalId}")
```

---

## Example 3: Floor Slab with Custom Outline

Creates an IfcSlab with an arbitrary polygonal outline.

```python
"""
Floor slab with custom polyline outline in Bonsai v0.8.x.
"""
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

# Create slab type with material layers
slab_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlabType",
    predefined_type="FLOOR",
    name="200mm Concrete Floor")

concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C25", category="concrete")

layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="200mm Floor", set_type="IfcMaterialLayerSet")

layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer, attributes={"LayerThickness": 0.200})

ifcopenshell.api.run("material.assign_material", model,
    products=[slab_type], material=layer_set)

# Create slab occurrence
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab",
    predefined_type="FLOOR",
    name="Floor-Level0")

ifcopenshell.api.run("type.assign_type", model,
    related_objects=[slab],
    relating_type=slab_type,
    should_map_representations=True)

# Create slab geometry with L-shaped outline
slab_repr = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context=body_context,
    depth=0.2,
    polyline=[
        (0.0, 0.0),
        (10.0, 0.0),
        (10.0, 5.0),
        (6.0, 5.0),
        (6.0, 8.0),
        (0.0, 8.0),
    ])

ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=slab_repr)

# Assign to storey
storey = model.by_type("IfcBuildingStorey")[0]
ifcopenshell.api.run("spatial.assign_container", model,
    products=[slab],
    relating_structure=storey)

model.write(IfcStore.path)
print(f"Slab created: {slab.GlobalId}")
```

---

## Example 4: Wall with Door Opening

Creates a wall, adds an opening void, and fills it with a door.

```python
"""
Wall with door opening in Bonsai v0.8.x.
Uses feature.add_feature (NEVER void.add_opening).
"""
import numpy as np
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

storey = model.by_type("IfcBuildingStorey")[0]

# Create wall
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-WithDoor")

wall_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_repr)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# Create opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Opening-Door01")

opening_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=0.9, height=2.1, thickness=0.25)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_repr)

# Position opening in the wall (1m from wall start)
opening_matrix = np.eye(4)
opening_matrix[0, 3] = 1.0  # 1m from wall origin
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=opening, matrix=opening_matrix, is_si=True)

# Add opening to wall (creates IfcRelVoidsElement)
# CRITICAL: Use feature.add_feature, NEVER void.add_opening
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)

# Create door and fill the opening
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door-01")

door_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=0.9, height=2.1, thickness=0.05)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=door, representation=door_repr)

# Fill opening with door (creates IfcRelFillsElement)
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)

ifcopenshell.api.run("spatial.assign_container", model,
    products=[door], relating_structure=storey)

model.write(IfcStore.path)
print(f"Wall with door created. Wall: {wall.GlobalId}, Door: {door.GlobalId}")
```

---

## Example 5: Multi-Layer Insulated Wall Type

Creates a wall type with multiple material layers (exterior insulated wall).

```python
"""
Multi-layer wall type in Bonsai v0.8.x.
Typical exterior insulated wall: plaster + concrete + insulation + cladding.
"""
import ifcopenshell
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Create wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType",
    predefined_type="STANDARD",
    name="Insulated Exterior Wall 350mm")

# Create materials
plaster = ifcopenshell.api.run("material.add_material", model,
    name="Interior Plaster", category="gypsum")
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30", category="concrete")
insulation = ifcopenshell.api.run("material.add_material", model,
    name="EPS Insulation", category="plastic")
cladding = ifcopenshell.api.run("material.add_material", model,
    name="Brick Cladding", category="brick")

# Create layer set (layers are ordered inside-to-outside)
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Insulated Exterior 350mm", set_type="IfcMaterialLayerSet")

# Layer 1: Interior plaster (15mm)
l1 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=plaster)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l1, attributes={"LayerThickness": 0.015, "Name": "Interior Plaster"})

# Layer 2: Concrete structure (200mm)
l2 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l2, attributes={"LayerThickness": 0.200, "Name": "Structural Concrete"})

# Layer 3: Insulation (100mm)
l3 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l3, attributes={"LayerThickness": 0.100, "Name": "Thermal Insulation"})

# Layer 4: Cladding (35mm)
l4 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=cladding)
ifcopenshell.api.run("material.edit_layer", model,
    layer=l4, attributes={"LayerThickness": 0.035, "Name": "Brick Cladding"})

# Assign to wall type
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=layer_set)

model.write(IfcStore.path)
print(f"Wall type created: {wall_type.Name} (total: 350mm)")
```

---

## Example 6: Beam with Rectangular Profile

Creates a concrete beam with a rectangular cross-section.

```python
"""
Concrete beam with rectangular profile in Bonsai v0.8.x.
"""
import numpy as np
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

# Create beam type
beam_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeamType",
    predefined_type="BEAM",
    name="300x500 Concrete Beam")

# Create material and profile
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30", category="concrete")

profile = model.create_entity("IfcRectangleProfileDef",
    ProfileType="AREA",
    ProfileName="300x500",
    XDim=0.300,
    YDim=0.500)

profile_set = ifcopenshell.api.run("material.add_material_set", model,
    name="300x500 Concrete", set_type="IfcMaterialProfileSet")
ifcopenshell.api.run("material.add_profile", model,
    profile_set=profile_set, material=concrete, profile=profile)
ifcopenshell.api.run("material.assign_material", model,
    products=[beam_type], material=profile_set)

# Create beam occurrence
beam = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="Beam-B1")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[beam],
    relating_type=beam_type,
    should_map_representations=True)

# Create profile representation (6m span)
beam_repr = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body_context,
    profile=profile,
    depth=6.0,        # 6m span
    cardinal_point=8) # top centre (beam hangs from top)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=beam, representation=beam_repr)

# Place beam at 3m height
storey = model.by_type("IfcBuildingStorey")[0]
ifcopenshell.api.run("spatial.assign_container", model,
    products=[beam], relating_structure=storey)

matrix = np.eye(4)
matrix[2, 3] = 3.0  # Z = 3m (top of storey)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=beam, matrix=matrix, is_si=True)

model.write(IfcStore.path)
print(f"Beam created: {beam.GlobalId}")
```

---

## Example 7: Multiple Elements from Same Type

Creates multiple wall instances from a single wall type.

```python
"""
Multiple wall instances from one type in Bonsai v0.8.x.
Demonstrates type reuse and batch spatial assignment.
"""
import numpy as np
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")
storey = model.by_type("IfcBuildingStorey")[0]

# Get or create wall type (reuse existing if available)
wall_types = model.by_type("IfcWallType")
if wall_types:
    wall_type = wall_types[0]
else:
    wall_type = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWallType", predefined_type="STANDARD",
        name="Standard 200mm Wall")

# Create 4 walls forming a room
wall_configs = [
    {"name": "Wall-North", "length": 6.0, "x": 0.0, "y": 5.0, "angle": 0.0},
    {"name": "Wall-South", "length": 6.0, "x": 0.0, "y": 0.0, "angle": 0.0},
    {"name": "Wall-East",  "length": 5.0, "x": 6.0, "y": 0.0, "angle": np.pi/2},
    {"name": "Wall-West",  "length": 5.0, "x": 0.0, "y": 0.0, "angle": np.pi/2},
]

walls = []
for config in wall_configs:
    # Create occurrence
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=config["name"])

    # Assign shared type
    ifcopenshell.api.run("type.assign_type", model,
        related_objects=[wall],
        relating_type=wall_type,
        should_map_representations=True)

    # Create geometry per wall (different lengths)
    wall_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
        context=body_context,
        length=config["length"],
        height=3.0,
        thickness=0.2)
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=wall, representation=wall_repr)

    # Set placement with rotation
    cos_a = np.cos(config["angle"])
    sin_a = np.sin(config["angle"])
    matrix = np.array([
        [cos_a, -sin_a, 0.0, config["x"]],
        [sin_a,  cos_a, 0.0, config["y"]],
        [0.0,    0.0,   1.0, 0.0],
        [0.0,    0.0,   0.0, 1.0]
    ])
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=wall, matrix=matrix, is_si=True)

    walls.append(wall)

# Batch spatial assignment (all walls at once)
ifcopenshell.api.run("spatial.assign_container", model,
    products=walls,
    relating_structure=storey)

model.write(IfcStore.path)
print(f"Created {len(walls)} walls in storey {storey.Name}")
```

---

## Example 8: Query and Inspect Existing Elements

Read-only operations for inspecting the IFC model.

```python
"""
Querying and inspecting elements in Bonsai v0.8.x.
"""
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.placement
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Query all walls
walls = model.by_type("IfcWall")
print(f"Found {len(walls)} walls")

for wall in walls:
    # Get type
    wall_type = ifcopenshell.util.element.get_type(wall)
    type_name = wall_type.Name if wall_type else "No type"

    # Get spatial container
    container = ifcopenshell.util.element.get_container(wall)
    container_name = container.Name if container else "Not contained"

    # Get material
    material = ifcopenshell.util.element.get_material(wall)
    if material:
        if material.is_a("IfcMaterialLayerSet"):
            mat_info = f"LayerSet: {material.LayerSetName}"
        elif material.is_a("IfcMaterial"):
            mat_info = f"Material: {material.Name}"
        else:
            mat_info = f"{material.is_a()}"
    else:
        mat_info = "No material"

    # Get placement
    if wall.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            wall.ObjectPlacement)
        position = matrix[:3, 3]
        pos_info = f"({position[0]:.2f}, {position[1]:.2f}, {position[2]:.2f})"
    else:
        pos_info = "No placement"

    # Get property sets
    psets = ifcopenshell.util.element.get_psets(wall)

    print(f"  {wall.Name}: type={type_name}, container={container_name}, "
          f"material={mat_info}, position={pos_info}, psets={len(psets)}")
```
