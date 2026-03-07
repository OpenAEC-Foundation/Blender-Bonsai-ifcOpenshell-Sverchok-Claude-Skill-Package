# IfcSverchok — Working Examples

> IfcOpenShell 0.8.x / Sverchok v1.4.0+ / Blender 4.0+/5.x

## Example 1: Minimal IFC Wall via ShapeBuilder

Create a simple IFC wall using ShapeBuilder nodes (no Blender geometry needed).

### Node Tree Layout

```
[Number (0.3)] ─→ XDim ─→ [SvIfcSbRectangle] ─→ Profile ─→ [SvIfcSbExtrude] ─→ Shape ─→ [SvIfcSbRepresentation] ─→ Repr ─→ [SvIfcCreateEntity] ─→ [SvIfcWriteFile]
[Number (0.2)] ─→ YDim ─→ /                                  Magnitude=3.0                                          IfcClass="IfcWall"       path="wall.ifc"
                                                                                                                      Names="Simple Wall"
```

### Programmatic Equivalent

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
import bpy

tree = bpy.data.node_groups.new("IFC_Wall", 'SverchCustomTreeType')

with tree.init_tree():
    # Number nodes for dimensions
    xdim = tree.nodes.new('SvNumberNode')
    xdim.location = (0, 100)
    xdim.float_ = 0.3

    ydim = tree.nodes.new('SvNumberNode')
    ydim.location = (0, 0)
    ydim.float_ = 0.2

    # ShapeBuilder: Rectangle profile
    rect = tree.nodes.new('SvIfcSbRectangle')
    rect.location = (200, 50)

    # ShapeBuilder: Extrude profile
    extrude = tree.nodes.new('SvIfcSbExtrude')
    extrude.location = (400, 50)
    extrude.Magnitude = 3.0

    # ShapeBuilder: Wrap as representation
    repr_node = tree.nodes.new('SvIfcSbRepresentation')
    repr_node.location = (600, 50)

    # Create IFC entity
    entity = tree.nodes.new('SvIfcCreateEntity')
    entity.location = (800, 50)
    entity.IfcClass = "IfcWall"
    entity.Names = "Simple Wall"

    # Write file
    write = tree.nodes.new('SvIfcWriteFile')
    write.location = (1000, 50)
    write.Path = "//wall.ifc"

    # Connect nodes
    tree.links.new(xdim.outputs[0], rect.inputs['XDim'])
    tree.links.new(ydim.outputs[0], rect.inputs['YDim'])
    tree.links.new(rect.outputs['Profile'], extrude.inputs['Profile'])
    tree.links.new(extrude.outputs['Shape'], repr_node.inputs['Items'])
    tree.links.new(repr_node.outputs['Representation'], entity.inputs['Representation'])
    tree.links.new(entity.outputs['Entity'], write.inputs[0])

tree.force_update()
# Result: wall.ifc with IfcWall, auto-generated hierarchy via ensure_hirarchy()
```

---

## Example 2: Sverchok Geometry to IFC

Convert procedural Sverchok geometry (a box) into an IFC column.

### Node Tree Layout

```
[Box Generator] ─→ Vertices ─→ [SvIfcSverchokToIfcRepr] ─→ Representations ─→ [SvIfcCreateEntity] ─→ [SvIfcWriteFile]
                ─→ Faces    ─→ /                             context_type="Model"  IfcClass="IfcColumn"    path="column.ifc"
                                                              context_id="Body"     Names="Col_001"
```

### Programmatic Equivalent

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
import bpy

tree = bpy.data.node_groups.new("IFC_Column", 'SverchCustomTreeType')

with tree.init_tree():
    # Box generator for column geometry
    box = tree.nodes.new('SvBoxNodeMk2')
    box.location = (0, 0)
    box.Size = 0.3
    box.Divx = 1
    box.Divy = 1
    box.Divz = 1

    # Convert Sverchok geometry to IFC representation
    to_repr = tree.nodes.new('SvIfcSverchokToIfcRepr')
    to_repr.location = (250, 0)

    # Create IFC entity
    entity = tree.nodes.new('SvIfcCreateEntity')
    entity.location = (500, 0)
    entity.IfcClass = "IfcColumn"
    entity.Names = "Col_001"

    # Write file
    write = tree.nodes.new('SvIfcWriteFile')
    write.location = (750, 0)
    write.Path = "//column.ifc"

    # Connect
    tree.links.new(box.outputs['Vertices'], to_repr.inputs['Vertices'])
    tree.links.new(box.outputs['Faces'], to_repr.inputs['Faces'])
    tree.links.new(to_repr.outputs['Representations'], entity.inputs['Representation'])
    tree.links.new(entity.outputs['Entity'], write.inputs[0])

tree.force_update()
```

---

## Example 3: Blender Objects to IFC

Convert existing Blender mesh objects into IFC elements.

### Node Tree Layout

```
[Objects In] ─→ Objects ─→ [SvIfcBMeshToIfcRepr] ─→ Representations ─→ [SvIfcCreateEntity] ─→ [SvIfcWriteFile]
                                                  ─→ Locations       ─→ /Location              path="model.ifc"
                                                                        IfcClass="IfcSlab"
                                                                        Names="Floor_001"
```

### Programmatic Equivalent

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
import bpy

tree = bpy.data.node_groups.new("IFC_Slab", 'SverchCustomTreeType')

with tree.init_tree():
    # Get Blender objects
    obj_in = tree.nodes.new('SvObjectsNodeMK3')
    obj_in.location = (0, 0)
    # Configure to pick specific objects via UI

    # Convert Blender mesh to IFC representation
    bmesh_repr = tree.nodes.new('SvIfcBMeshToIfcRepr')
    bmesh_repr.location = (250, 0)

    # Create IFC entity with location from world matrix
    entity = tree.nodes.new('SvIfcCreateEntity')
    entity.location = (500, 0)
    entity.IfcClass = "IfcSlab"
    entity.Names = "Floor_001"

    # Write file
    write = tree.nodes.new('SvIfcWriteFile')
    write.location = (750, 0)
    write.Path = "//model.ifc"

    # Connect: representation + location
    tree.links.new(obj_in.outputs['Objects'], bmesh_repr.inputs['blender_objects'])
    tree.links.new(bmesh_repr.outputs['Representations'], entity.inputs['Representation'])
    tree.links.new(bmesh_repr.outputs['Locations'], entity.inputs['Location'])
    tree.links.new(entity.outputs['Entity'], write.inputs[0])

tree.force_update()
```

---

## Example 4: Full Workflow with Spatial Hierarchy and Properties

Complete 6-step workflow including manual spatial hierarchy and property sets.

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
import bpy

tree = bpy.data.node_groups.new("IFC_Full", 'SverchCustomTreeType')

with tree.init_tree():
    # Step 1: Geometry — Rectangle profile
    rect = tree.nodes.new('SvIfcSbRectangle')
    rect.location = (0, 0)

    # Step 2a: Extrude
    extrude = tree.nodes.new('SvIfcSbExtrude')
    extrude.location = (200, 0)
    extrude.Magnitude = 3.0

    # Step 2b: Wrap as representation
    repr_node = tree.nodes.new('SvIfcSbRepresentation')
    repr_node.location = (400, 0)

    # Step 3: Create entity
    entity = tree.nodes.new('SvIfcCreateEntity')
    entity.location = (600, 0)
    entity.IfcClass = "IfcWall"
    entity.Names = "Exterior Wall"

    # Step 4: Spatial hierarchy — assign to storey
    storey = tree.nodes.new('SvIfcAddSpatialElement')
    storey.location = (800, 0)
    storey.IfcClass = "IfcBuildingStorey"

    # Step 5: Add properties
    pset = tree.nodes.new('SvIfcAddPset')
    pset.location = (800, -200)
    pset.Name = "Pset_WallCommon"
    pset.Properties = '{"IsExternal": true, "LoadBearing": false}'

    # Step 6: Write file
    write = tree.nodes.new('SvIfcWriteFile')
    write.location = (1000, 0)
    write.Path = "//full_model.ifc"

    # Connect chain
    tree.links.new(rect.outputs['Profile'], extrude.inputs['Profile'])
    tree.links.new(extrude.outputs['Shape'], repr_node.inputs['Items'])
    tree.links.new(repr_node.outputs['Representation'], entity.inputs['Representation'])
    tree.links.new(entity.outputs['Entity'], storey.inputs['Elements'])
    tree.links.new(entity.outputs['Entity'], pset.inputs['Element'])
    tree.links.new(storey.outputs['SpatialElement'], write.inputs[0])

tree.force_update()
# Result: IFC file with wall in storey, property set, auto-completed hierarchy
```

---

## Example 5: Reading and Querying an Existing IFC File

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
import bpy

tree = bpy.data.node_groups.new("IFC_Query", 'SverchCustomTreeType')

with tree.init_tree():
    # Read existing file
    read = tree.nodes.new('SvIfcReadFile')
    read.location = (0, 0)
    read.Path = "//input.ifc"

    # Query all walls
    by_type = tree.nodes.new('SvIfcByType')
    by_type.location = (250, 0)
    by_type.IfcClass = "IfcWall"

    # Read attributes of each wall
    read_entity = tree.nodes.new('SvIfcReadEntity')
    read_entity.location = (500, 0)

    # Convert to Blender shapes for visualization
    create_shape = tree.nodes.new('SvIfcCreateShape')
    create_shape.location = (500, -200)

    # Connect
    tree.links.new(by_type.outputs['Entities'], read_entity.inputs['Entity'])
    tree.links.new(by_type.outputs['Entities'], create_shape.inputs['Entity'])

tree.force_update()
```

---

## Example 6: Using Bonsai Integration

Toggle `use_bonsai_file` to modify Bonsai's active IFC file from Sverchok.

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok + Bonsai
# IMPORTANT: Bonsai must have an active IFC file loaded

from ifcsverchok.helper import SvIfcStore

# Enable Bonsai file mode
SvIfcStore.use_bonsai_file = True

# Now SvIfcStore.get_file() returns Bonsai's active file
# Any IfcSverchok node operations affect Bonsai's file directly

# To switch back to standalone mode:
SvIfcStore.use_bonsai_file = False
SvIfcStore.purge()  # Clear the transient file

# In the Sverchok UI: click "Use Bonsai File" toggle button
# This sets SvIfcStore.use_bonsai_file = True
```

**Edge cases with Bonsai integration**:
- If Bonsai has no active file, `tool.Ifc.get()` returns `None` — nodes fail silently
- Undo is even more dangerous: both Bonsai and IfcSverchok state can desynchronize
- Writing to Bonsai's file does not trigger Bonsai's UI refresh — manual refresh needed

---

## Example 7: Using SvIfcApi for Advanced Operations

Use the generic API node for operations not covered by specialized nodes.

### Node Tree Layout

```
[SvIfcApi] with Function="root.create_entity"
               Arguments='{"ifc_class": "IfcMaterial", "name": "Concrete"}'
```

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
import bpy

tree = bpy.data.node_groups.new("IFC_API", 'SverchCustomTreeType')

with tree.init_tree():
    # Use generic API node for material creation
    # (no dedicated material node exists in IfcSverchok)
    api_node = tree.nodes.new('SvIfcApi')
    api_node.location = (0, 0)
    api_node.Function = "material.add_material"
    api_node.Arguments = '{"name": "Concrete C30/37"}'

tree.force_update()
```

---

## Example 8: Multiple Entities with Double-Nested Processing

Demonstrates how SvIfcCore's double-nested zip handles multiple objects.

```python
# Conceptual: how double-nested processing works with multiple inputs

# If SvIfcCreateEntity receives:
#   IfcClass = [["IfcWall", "IfcWall"]]        → 1 object group, 2 items
#   Names   = [["Wall_A", "Wall_B"]]            → 1 object group, 2 items
#   Repr    = [[repr_1, repr_2]]                → 1 object group, 2 items

# First zip_long_repeat (object level):
#   → (["IfcWall", "IfcWall"], ["Wall_A", "Wall_B"], [repr_1, repr_2])

# Second zip_long_repeat (item level):
#   → ("IfcWall", "Wall_A", repr_1)  → process_ifc("IfcWall", "Wall_A", repr_1)
#   → ("IfcWall", "Wall_B", repr_2)  → process_ifc("IfcWall", "Wall_B", repr_2)

# If inputs have mismatched lengths, zip_long_repeat repeats the shorter:
#   IfcClass = [["IfcWall"]]                    → 1 object group, 1 item
#   Names   = [["Wall_A", "Wall_B", "Wall_C"]]  → 1 object group, 3 items
#   → "IfcWall" is repeated to match 3 names
```

---

## Example 9: Parametric Wall Array

Use Sverchok's list processing to create multiple walls at different positions.

```
[Number Range (0..10..2)] ─→ [Vector In (x=range, y=0, z=0)] ─→ [Matrix In] ─→ Location
                                                                                    ↓
[SvIfcSbRectangle] ─→ [SvIfcSbExtrude] ─→ [SvIfcSbRepresentation] ─→ Repr ─→ [SvIfcCreateEntity] ─→ [SvIfcWriteFile]
  XDim=0.3              Magnitude=3.0                                  IfcClass="IfcWall"             path="walls.ifc"
  YDim=0.2                                                             Names="Wall"
```

The double-nested processing in `SvIfcCreateEntity` automatically creates one wall per position from the number range, with the single representation repeated for all.
