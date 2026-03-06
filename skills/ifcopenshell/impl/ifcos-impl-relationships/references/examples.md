# IFC Relationship Management — Working Code Examples

All examples use `ifcopenshell.api.run()` for relationship creation. Version-safe query patterns use `ifcopenshell.util.element`.

---

## Example 1: Complete Spatial Hierarchy with Aggregation

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Campus Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Create spatial elements
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Main Campus")
bldg_a = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
bldg_b = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building B")
storey_a1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="A-Ground")
storey_a2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="A-First")
storey_b1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="B-Ground")

# Build hierarchy via aggregation (each call creates IfcRelAggregates)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[bldg_a, bldg_b], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey_a1, storey_a2], relating_object=bldg_a)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey_b1], relating_object=bldg_b)
```

---

## Example 2: Spatial Containment — Elements in Storeys

```python
# IfcOpenShell — IFC4
# Assumes model and storey exist (see Example 1)

# Create elements
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Floor Slab")

# Assign all to ground floor (creates IfcRelContainedInSpatialStructure)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_a1, products=[wall, column, slab])

# Move wall to a different storey (previous containment is removed automatically)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_a2, products=[wall])
```

---

## Example 3: Spatial Reference for Multi-Storey Elements

```python
# IfcOpenShell — IFC4+ only
# Use spatial references for elements spanning multiple storeys

# Tall column contained in ground floor, referenced in first floor
tall_column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Double-Height Column")

# Primary containment (one storey only)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_a1, products=[tall_column])

# Secondary reference (IfcRelReferencedInSpatialStructure)
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[tall_column], relating_structure=storey_a2)
```

---

## Example 4: Type Assignment with Multiple Occurrences

```python
# IfcOpenShell — IFC4

# Create wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="WT-200 Standard")

# Create multiple wall occurrences
walls = []
for i in range(5):
    w = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=f"Wall {i+1:03d}")
    walls.append(w)

# Assign type to all occurrences (creates IfcRelDefinesByType)
ifcopenshell.api.run("type.assign_type", model,
    related_objects=walls, relating_type=wall_type)

# Later: unassign type from one wall
ifcopenshell.api.run("type.unassign_type", model,
    related_objects=[walls[2]])
```

---

## Example 5: Property Set Relationships

```python
# IfcOpenShell — IFC4

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Create property set (automatically creates IfcRelDefinesByProperties)
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI120"
})

# Share the same pset with additional walls
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 002")
wall3 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 003")
ifcopenshell.api.run("pset.assign_pset", model,
    products=[wall2, wall3], pset=pset)

# Remove property set from one wall
ifcopenshell.api.run("pset.remove_pset", model, product=wall3, pset=pset)
```

---

## Example 6: Material Association

```python
# IfcOpenShell — IFC4

# Single material assignment
concrete = ifcopenshell.api.run("material.add_material", model,
    name="C30/37 Concrete", category="concrete")
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")

# Creates IfcRelAssociatesMaterial
ifcopenshell.api.run("material.assign_material", model,
    products=[column], material=concrete)

# Material layer set on wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Cavity Wall")
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Cavity Wall Layers", set_type="IfcMaterialLayerSet")

brick = ifcopenshell.api.run("material.add_material", model,
    name="Brick", category="brick")
insulation = ifcopenshell.api.run("material.add_material", model,
    name="Insulation", category="mineral wool")

for mat, thickness in [(brick, 0.102), (insulation, 0.080), (brick, 0.102)]:
    layer = ifcopenshell.api.run("material.add_layer", model,
        layer_set=layer_set, material=mat)
    ifcopenshell.api.run("material.edit_layer", model,
        layer=layer, attributes={"LayerThickness": thickness})

ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)
```

---

## Example 7: Void Relationships — Openings and Fillings

```python
# IfcOpenShell — IFC4
# Assumes wall, body context, and storey exist

# Create opening (IfcRelVoidsElement)
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Window Opening")
opening_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=1.2, height=1.5, thickness=0.3)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=opening)

# Cut wall (creates IfcRelVoidsElement)
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Fill with window (creates IfcRelFillsElement)
window = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWindow", name="Window W1")
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=window)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_a1, products=[window])
```

---

## Example 8: Nesting — Component Attached to Host

```python
# IfcOpenShell — IFC4

# Sink with nested faucet (IfcRelNests)
sink = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSanitaryTerminal", name="Kitchen Sink")
faucet = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcValve", name="Kitchen Faucet")
drain = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="Drain Pipe")

# Nest both components in the sink
ifcopenshell.api.run("nest.assign_object", model,
    related_objects=[faucet, drain], relating_object=sink)

# Contain the host (sink) — nested children follow
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_a1, products=[sink])
```

---

## Example 9: Grouping Elements

```python
# IfcOpenShell — IFC4

# Create a logical group (IfcRelAssignsToGroup)
ext_walls_group = ifcopenshell.api.run("group.add_group", model,
    name="External Walls", description="All external load-bearing walls")

# Add walls to the group
wall_a = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall A")
wall_b = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall B")
wall_c = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Ext Wall C")

ifcopenshell.api.run("group.assign_group", model,
    products=[wall_a, wall_b, wall_c], group=ext_walls_group)

# Remove one wall from group
ifcopenshell.api.run("group.unassign_group", model,
    products=[wall_c], group=ext_walls_group)
```

---

## Example 10: Querying Relationships with ifcopenshell.util.element

```python
# IfcOpenShell — all schema versions (IFC2X3/IFC4/IFC4X3)
import ifcopenshell.util.element

# Get spatial container
container = ifcopenshell.util.element.get_container(wall)
if container:
    print(f"Wall is in: {container.Name}")

# Get element type
elem_type = ifcopenshell.util.element.get_type(wall)
if elem_type:
    print(f"Wall type: {elem_type.Name}")

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(wall_type)
print(f"Type has {len(occurrences)} occurrences")

# Get all property sets as nested dict
psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, props in psets.items():
    print(f"  {pset_name}:")
    for prop_name, value in props.items():
        if prop_name != "id":
            print(f"    {prop_name} = {value}")

# Get material
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material: {material.is_a()} - {getattr(material, 'Name', 'N/A')}")

# Get decomposition tree
children = ifcopenshell.util.element.get_decomposition(building)
print(f"Building has {len(children)} decomposed children")

# Get aggregate parent
parent = ifcopenshell.util.element.get_aggregate(storey_a1)
if parent:
    print(f"Storey parent: {parent.Name}")
```

---

## Example 11: Direct Inverse Attribute Queries (IFC4+)

```python
# IfcOpenShell — IFC4+ only (inverse attribute names differ in IFC2X3)

# Aggregation children
for rel in building.IsDecomposedBy:
    for child in rel.RelatedObjects:
        print(f"Building contains: {child.is_a()} — {child.Name}")

# Elements in a storey
for rel in storey_a1.ContainsElements:
    for elem in rel.RelatedElements:
        print(f"Storey contains: {elem.is_a()} — {elem.Name}")

# Type assignment (IFC4+ uses IsTypedBy)
for rel in wall.IsTypedBy:
    print(f"Wall type: {rel.RelatingType.Name}")

# Openings in wall
for rel in wall.HasOpenings:
    opening = rel.RelatedOpeningElement
    print(f"Opening: {opening.Name}")
    for fill_rel in opening.HasFillings:
        print(f"  Filled by: {fill_rel.RelatedBuildingElement.Name}")

# Property sets
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        print(f"PSet: {pset.Name}")
```

---

## Example 12: Removing Relationships

```python
# IfcOpenShell — all schema versions

# Remove spatial containment
ifcopenshell.api.run("spatial.unassign_container", model, products=[wall])

# Remove aggregation
ifcopenshell.api.run("aggregate.unassign_object", model, products=[storey_a1])

# Remove type assignment
ifcopenshell.api.run("type.unassign_type", model, related_objects=[wall])

# Remove material
ifcopenshell.api.run("material.unassign_material", model, products=[wall])

# Remove nesting
ifcopenshell.api.run("nest.unassign_object", model, related_objects=[faucet])

# Remove property set from element
ifcopenshell.api.run("pset.remove_pset", model, product=wall, pset=pset)

# Remove spatial reference
ifcopenshell.api.run("spatial.dereference_structure", model,
    products=[tall_column], relating_structure=storey_a2)
```
