# QTO Workflow Examples

> **Bonsai v0.8.x** | All examples verified against `bonsai.bim.module.qto` source

## Example 1: Batch QTO for All Walls

Calculate standard base quantities for all wall elements in the project.

```python
# Bonsai v0.8.x — Batch QTO for walls
import bpy

# Step 1: Deselect all objects to trigger "all elements" mode
bpy.ops.object.select_all(action='DESELECT')

# Step 2: Set the QTO rule
# The operator uses BIMQtoProperties.qto_rule if no parameter override
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"
)

# Result: All IfcWall elements now have IfcElementQuantity sets with:
# Length, Width, Height, GrossFootprintArea, NetFootprintArea,
# GrossSideArea, NetSideArea, GrossVolume, NetVolume
```

## Example 2: QTO for Selected Objects Only

Calculate quantities for specific selected elements.

```python
# Bonsai v0.8.x — QTO for selected elements
import bpy

# Step 1: Select target objects
bpy.ops.object.select_all(action='DESELECT')
wall_obj = bpy.data.objects.get("IfcWall/Wall_001")
if wall_obj:
    wall_obj.select_set(True)
    bpy.context.view_layer.objects.active = wall_obj

# Step 2: Run QTO on selection
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"
)

# Result: Only Wall_001 gets IfcElementQuantity updated
```

## Example 3: Single Quantity Calculation

Calculate one specific quantity for the active element.

```python
# Bonsai v0.8.x — Calculate gross volume for active element
import bpy

# Ensure an IFC element object is active
obj = bpy.context.active_object
if obj and obj.BIMObjectProperties.ifc_definition_id:
    bpy.ops.bim.calculate_single_quantity(
        calculator="Blender",
        qto_name="Qto_WallBaseQuantities",
        prop_name="GrossVolume",
        calculator_function="get_gross_volume"
    )
```

## Example 4: Slab Quantities with Perimeter

Calculate all base quantities for slab elements including perimeter.

```python
# Bonsai v0.8.x — Slab base quantities
import bpy

# Select all slab objects
bpy.ops.object.select_all(action='DESELECT')
for obj in bpy.data.objects:
    element = tool.Ifc.get_entity(obj)
    if element and element.is_a("IfcSlab"):
        obj.select_set(True)

# Run QTO with slab rule
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_SlabBaseQuantities"
)

# Result: Width, Length, Depth, GrossArea, NetArea,
# GrossVolume, NetVolume, Perimeter
```

## Example 5: Column Quantities Including Weight

Calculate column quantities. Weight requires material density to be set first.

```python
# Bonsai v0.8.x — Column quantities with weight
import bpy
import ifcopenshell
import ifcopenshell.api

# Step 1: ALWAYS set material density BEFORE weight calculation
ifc_file = tool.Ifc.get()
column = tool.Ifc.get_entity(bpy.context.active_object)

# Get the column's material
material = ifcopenshell.util.element.get_material(column)
if material:
    # Assign MassDensity (concrete = ~2400 kg/m3)
    ifcopenshell.api.run("pset.edit_pset", ifc_file,
        pset=ifcopenshell.util.element.get_psets(material).get("Pset_MaterialCommon"),
        properties={"MassDensity": 2400.0}
    )

# Step 2: Now calculate quantities (weight will work)
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_ColumnBaseQuantities"
)

# Result: Length, CrossSectionArea, OuterSurfaceArea,
# GrossVolume, NetVolume, GrossWeight, NetWeight
```

## Example 6: Edit Mode — Edge Length Measurement

Measure total length of selected edges (useful for rebar, trim, etc.).

```python
# Bonsai v0.8.x — Edge length measurement
import bpy

# Step 1: Ensure object is selected and in Edit Mode
obj = bpy.context.active_object
bpy.ops.object.mode_set(mode='EDIT')

# Step 2: Select edges of interest (example: select all edges)
bpy.ops.mesh.select_all(action='SELECT')

# Step 3: Calculate edge lengths
bpy.ops.bim.calculate_edge_lengths()

# Result: Total edge length displayed in operator result
```

## Example 7: Edit Mode — Face Area Measurement

Measure total area of selected faces.

```python
# Bonsai v0.8.x — Face area measurement
import bpy

obj = bpy.context.active_object
bpy.ops.object.mode_set(mode='EDIT')

# Select specific faces (top faces for roofing area, etc.)
bpy.ops.mesh.select_all(action='DESELECT')
# Use mesh selection tools to pick desired faces

bpy.ops.bim.calculate_face_areas()

# Result: Total face area displayed in operator result
```

## Example 8: Formwork Area Calculation

Calculate formwork area for concrete elements (excludes top faces).

```python
# Bonsai v0.8.x — Formwork area
import bpy

# Select concrete elements
bpy.ops.object.select_all(action='DESELECT')
for obj in bpy.data.objects:
    element = tool.Ifc.get_entity(obj)
    if element and element.is_a("IfcColumn"):
        obj.select_set(True)

# Total formwork (all sides except top)
bpy.ops.bim.calculate_formwork_area()

# Side-only formwork (lateral surfaces, excludes top AND bottom)
bpy.ops.bim.calculate_side_formwork_area()
```

## Example 9: Gross vs. Net Volume Comparison

Compare gross and net volumes to determine opening impact.

```python
# Bonsai v0.8.x — Gross vs Net volume
import bpy

wall = bpy.context.active_object

# Calculate gross volume (wall without door/window openings)
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="GrossVolume",
    calculator_function="get_gross_volume"
)

# Calculate net volume (wall with door/window openings subtracted)
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="NetVolume",
    calculator_function="get_net_volume"
)

# Opening volume = GrossVolume - NetVolume
```

## Example 10: Space Quantities

Calculate space-level quantities (floor area, volume, heights).

```python
# Bonsai v0.8.x — Space quantities
import bpy

# Select space objects
bpy.ops.object.select_all(action='DESELECT')
for obj in bpy.data.objects:
    element = tool.Ifc.get_entity(obj)
    if element and element.is_a("IfcSpace"):
        obj.select_set(True)

bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_SpaceBaseQuantities"
)

# Result: Height, FinishCeilingHeight, FinishFloorHeight,
# GrossFloorArea, NetFloorArea, GrossVolume, NetVolume
```

## Example 11: Reading Calculated Quantities via IfcOpenShell

After QTO calculation, read the stored quantities from the IFC model.

```python
# Bonsai v0.8.x — Read QTO results from IFC
import ifcopenshell
import ifcopenshell.util.element

ifc_file = tool.Ifc.get()
wall = tool.Ifc.get_entity(bpy.context.active_object)

# Get all quantity sets for this element
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# Access specific quantity set
wall_qto = qtos.get("Qto_WallBaseQuantities", {})
gross_volume = wall_qto.get("GrossVolume", 0.0)
net_volume = wall_qto.get("NetVolume", 0.0)
length = wall_qto.get("Length", 0.0)

print(f"Wall gross volume: {gross_volume:.3f} m3")
print(f"Wall net volume: {net_volume:.3f} m3")
print(f"Wall length: {length:.3f} m")
```

## Example 12: QTO with Fallback Calculator

Enable fallback to try alternative calculators when primary lacks support.

```python
# Bonsai v0.8.x — QTO with fallback
import bpy

# Enable fallback mode
bpy.context.scene.BIMQtoProperties.fallback = True

# Run QTO — if Blender calculator lacks a function for a specific
# class/quantity combination, it tries alternative calculators
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"
)

# Disable fallback after use
bpy.context.scene.BIMQtoProperties.fallback = False
```

## Example 13: QTO-Cost Integration Query

Query cost items linked to element quantities.

```python
# Bonsai v0.8.x — QTO-Cost integration
import bpy

element = tool.Ifc.get_entity(bpy.context.active_object)

# Get cost items related to this element's quantities
related = tool.Qto.get_related_cost_item_quantities(element)

for entry in related:
    cost_item = entry["cost_item"]
    quantities = entry["quantities"]
    print(f"Cost item: {cost_item.Name}")
    for q in quantities:
        print(f"  Quantity: {q.Name} = {q.NominalValue.wrappedValue}")
```
