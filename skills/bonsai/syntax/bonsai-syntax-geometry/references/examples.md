# Geometry Creation Code Examples

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Blender 4.2.0+
> All examples are source-verified. All dimensions are in SI meters.

---

## Example 1: Complete Project with Wall Geometry (Standalone)

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation

# Bootstrap project
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Geometry Demo")
ifcopenshell.api.run("unit.assign_unit", model)

# Create representation contexts
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Spatial hierarchy
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

# Create wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

model.write("wall_demo.ifc")
```

---

## Example 2: Column with Rectangular Profile Extrusion

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes: model, body context, storey already created (see Example 1)

# Create profile
profile = ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class="IfcRectangleProfileDef")
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=profile, attributes={"XDim": 0.4, "YDim": 0.4})

# Create column with profile extrusion
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0, cardinal_point=5)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=column)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=storey)
```

---

## Example 3: Beam with I-Shape Profile

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes: model, body context, storey already created

# Create I-shape profile (IPE 300 approximation)
profile = ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=profile, attributes={
        "OverallWidth": 0.150,
        "OverallDepth": 0.300,
        "WebThickness": 0.0071,
        "FlangeThickness": 0.0107,
    })

# Create beam with profile extrusion along Y-axis
beam = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="Beam B1")
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=6.0, cardinal_point=5)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=beam, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=beam)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[beam], relating_structure=storey)
```

---

## Example 4: Slab with Custom Footprint

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes: model, body context, storey already created

slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Floor Slab")

# L-shaped footprint (coordinates in meters)
footprint = [
    (0.0, 0.0), (10.0, 0.0), (10.0, 4.0),
    (6.0, 4.0), (6.0, 8.0), (0.0, 8.0), (0.0, 0.0)
]

representation = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context=body, depth=0.25, polyline=footprint)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=slab)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[slab], relating_structure=storey)
```

---

## Example 5: Wall with Boolean Clipping (Angled Top)

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes: model, body context, storey already created

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Gable Wall")

# Wall with angled top: clipping plane cuts at 2.5m height, tilted normal
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=8.0, height=5.0, thickness=0.3,
    clippings=[
        {"location": (4.0, 0.0, 2.5), "normal": (0.0, 0.0, 1.0)},   # Horizontal cut at 2.5m
        {"location": (4.0, 0.0, 5.0), "normal": (-1.0, 0.0, 0.5)},   # Angled cut (left slope)
        {"location": (4.0, 0.0, 5.0), "normal": (1.0, 0.0, 0.5)},    # Angled cut (right slope)
    ])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
```

---

## Example 6: Arbitrary Mesh Representation (Box)

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api

# Assumes: model, body context already created

element = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingElementProxy", name="Custom Geometry")

# Vertices and faces as list-of-lists (one sub-list per representation item)
vertices = [[
    (0.0, 0.0, 0.0), (1.0, 0.0, 0.0), (1.0, 1.0, 0.0), (0.0, 1.0, 0.0),
    (0.0, 0.0, 1.0), (1.0, 0.0, 1.0), (1.0, 1.0, 1.0), (0.0, 1.0, 1.0),
]]
faces = [[
    (0, 1, 2, 3), (4, 5, 6, 7), (0, 1, 5, 4),
    (1, 2, 6, 5), (2, 3, 7, 6), (3, 0, 4, 7),
]]

representation = ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body, vertices=vertices, faces=faces)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=representation)
```

---

## Example 7: Opening in Wall (Bonsai v0.8+ Pattern)

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

# Assumes: model, body context, wall (with geometry) already created

# Create opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# Opening geometry (rectangular void: 0.9m wide, 2.1m high, 0.3m thick)
opening_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=0.9, height=2.1, thickness=0.3)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_repr)

# Position the opening (1.5m from wall start, at floor level)
matrix = numpy.eye(4)
matrix[0][3] = 1.5  # X offset
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=opening, matrix=matrix)

# Attach opening to wall (creates IfcRelVoidsElement)
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)

# Create and insert door
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door D1")
ifcopenshell.api.run("feature.add_filling", model,
    opening=opening, element=door)
```

---

## Example 8: ShapeBuilder — Custom Extruded Shape

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api
from ifcopenshell.util.shape_builder import ShapeBuilder

# Assumes: model, body context already created

builder = ShapeBuilder(model)

# Create L-shaped profile using polyline
l_shape = builder.polyline([
    (0.0, 0.0), (0.3, 0.0), (0.3, 0.05),
    (0.05, 0.05), (0.05, 0.3), (0.0, 0.3), (0.0, 0.0)
], closed=True)
profile = builder.profile(l_shape, name="L-Bracket")

# Extrude to 3D solid
solid = builder.extrude(profile, magnitude=2.5)

# Create IfcShapeRepresentation (auto-detects RepresentationType)
representation = builder.get_representation(body, [solid])

# Assign to element
element = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcMember", name="L-Bracket 001")
ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=representation)
```

---

## Example 9: ShapeBuilder — Profile with Void (Hollow Section)

```python
# IfcOpenShell standalone — IFC4
from ifcopenshell.util.shape_builder import ShapeBuilder

builder = ShapeBuilder(model)

# Outer rectangle (200mm x 200mm)
outer = builder.rectangle(size=(0.2, 0.2))

# Inner rectangle (160mm x 160mm, centered)
inner = builder.rectangle(size=(0.16, 0.16), position=(0.02, 0.02))

# Profile with void
profile = builder.profile(outer, name="Hollow Section", inner_curves=(inner,))

# Extrude
solid = builder.extrude(profile, magnitude=4.0)
representation = builder.get_representation(body, [solid])
```

---

## Example 10: ShapeBuilder — Boolean with Half-Space Solid

```python
# IfcOpenShell standalone — IFC4
from ifcopenshell.util.shape_builder import ShapeBuilder

builder = ShapeBuilder(model)

# Create base solid (extruded rectangle)
rect = builder.rectangle(size=(1.0, 0.5))
profile = builder.profile(rect)
solid = builder.extrude(profile, magnitude=3.0)

# Create clipping plane at 2m height, angled
plane = builder.plane(
    location=(0.0, 0.0, 2.0),
    normal=(0.0, 0.0, 1.0),
)
half_space = builder.half_space_solid(plane, agreement_flag=False)

# Apply boolean via API
representation = builder.get_representation(body, [solid])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=representation)

# Add boolean clipping
ifcopenshell.api.run("geometry.add_boolean", model,
    first_item=solid, second_items=[half_space], operator="DIFFERENCE")
```

---

## Example 11: Arbitrary Profile Extrusion (Custom Window Frame)

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api

# Custom U-shaped frame profile (coordinates in meters)
frame_outline = [
    (0.0, 0.0), (0.06, 0.0), (0.06, 0.08),
    (0.0, 0.08), (0.0, 0.0)  # Must close
]

profile = ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile=frame_outline, name="Window Frame U")

# Extrude along depth
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=1.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=frame_element, representation=representation)
```

---

## Example 12: Multiple Contexts (Body + Axis)

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.representation

# Assumes: model, model3d root context already created

# Get or create Body and Axis sub-contexts
body = ifcopenshell.util.representation.get_context(model, "Model", "Body", "MODEL_VIEW")
if not body:
    body = ifcopenshell.api.run("context.add_context", model,
        context_type="Model", context_identifier="Body",
        target_view="MODEL_VIEW", parent=model3d)

axis_ctx = ifcopenshell.util.representation.get_context(model, "Model", "Axis", "GRAPH_VIEW")
if not axis_ctx:
    axis_ctx = ifcopenshell.api.run("context.add_context", model,
        context_type="Model", context_identifier="Axis",
        target_view="GRAPH_VIEW", parent=model3d)

# Create wall with body AND axis representations
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall with Axis")

# Body representation
body_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=body_repr)

# Axis representation (2D centerline)
axis_repr = ifcopenshell.api.run("geometry.add_axis_representation", model,
    context=axis_ctx, axis=((0.0, 0.0), (5.0, 0.0)))
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=axis_repr)
```

---

## Example 13: Type Geometry with MappedItems

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

# Assumes: model, body context, storey already created

# Create type with geometry
col_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="Square Column 400x400")

profile = ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class="IfcRectangleProfileDef")
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=profile, attributes={"XDim": 0.4, "YDim": 0.4})

type_repr = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)
# Assign to TYPE (not occurrence) — geometry is shared via MappedItem
ifcopenshell.api.run("geometry.assign_representation", model,
    product=col_type, representation=type_repr)

# Create occurrences that inherit geometry
for i, x_pos in enumerate([0.0, 5.0, 10.0]):
    col = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcColumn", name=f"Column C{i+1}")
    ifcopenshell.api.run("type.assign_type", model,
        related_objects=[col], relating_type=col_type)

    matrix = numpy.eye(4)
    matrix[0][3] = x_pos
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=col, matrix=matrix)
    ifcopenshell.api.run("spatial.assign_container", model,
        products=[col], relating_structure=storey)
```

---

## Example 14: Bonsai — Update Representation from Blender Mesh

```python
# Bonsai context (inside Blender) — v0.8.x
import bpy
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

obj = bpy.context.active_object

# After editing mesh in Blender, sync to IFC:
bpy.ops.bim.update_representation(obj=obj.name)

# For parametric (force rectangular extrusion detection):
bpy.ops.bim.update_representation(
    obj=obj.name,
    ifc_representation_class="IfcExtrudedAreaSolid/IfcRectangleProfileDef")
```

---

## Example 15: Bonsai — Edit Mode Workflow

```python
# Bonsai context (inside Blender) — v0.8.x
import bpy

obj = bpy.context.active_object

# Enter edit mode (IFC-aware: loads current representation for editing)
bpy.ops.bim.override_mode_set_edit()

# ... user edits mesh in Blender edit mode ...

# Return to object mode (auto-saves mesh changes back to IFC)
bpy.ops.bim.override_mode_set_object()
```

---

## Example 16: ShapeBuilder — Sphere and Block Primitives

```python
# IfcOpenShell standalone — IFC4
from ifcopenshell.util.shape_builder import ShapeBuilder

builder = ShapeBuilder(model)

# Create a sphere (radius 0.5m at position 0,0,1)
sphere = builder.sphere(radius=0.5, center=(0.0, 0.0, 1.0))

# Create a block (1m x 2m x 0.5m)
block = builder.block(position=(0.0, 0.0, 0.0),
    x_length=1.0, y_length=2.0, z_length=0.5)

# Combine in single representation
representation = builder.get_representation(body, [sphere, block])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=representation)
```

---

## Example 17: Placement with Rotation

```python
# IfcOpenShell standalone — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy
import math

# Create placement matrix: translate (3,2,0) + rotate 45° around Z
angle = math.radians(45)
matrix = numpy.array([
    [math.cos(angle), -math.sin(angle), 0.0, 3.0],
    [math.sin(angle),  math.cos(angle), 0.0, 2.0],
    [0.0,              0.0,             1.0, 0.0],
    [0.0,              0.0,             0.0, 1.0],
])

ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix, is_si=True)
```
