# Material Assignment Working Code Examples

All examples verified against the [IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/material/index.html) and research in [vooronderzoek-ifcopenshell.md](../../../docs/research/vooronderzoek-ifcopenshell.md).

---

## 1. Single Material Assignment

### Create and Assign a Simple Material

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Create wall
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Create material with category
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete C30/37", category="concrete")

# Assign single material to wall
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)

# Verify assignment
import ifcopenshell.util.element
mat = ifcopenshell.util.element.get_material(wall)
print(f"Material: {mat.Name}")  # "Concrete C30/37"
```

### Assign Material to Multiple Elements at Once

```python
# IfcOpenShell — all schema versions
steel = ifcopenshell.api.material.add_material(model,
    name="S355", category="steel")

beam1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="Beam 001")
beam2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="Beam 002")

# Assign to multiple products in one call
ifcopenshell.api.material.assign_material(model,
    products=[beam1, beam2], material=steel)
```

---

## 2. Layered Material (IfcMaterialLayerSet)

### External Cavity Wall with Three Layers

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Create materials
facing_brick = ifcopenshell.api.material.add_material(model,
    name="Facing Brick", category="brick")
mineral_wool = ifcopenshell.api.material.add_material(model,
    name="Mineral Wool 100mm", category="glass")
concrete_block = ifcopenshell.api.material.add_material(model,
    name="Concrete Block 100mm", category="block")
plaster = ifcopenshell.api.material.add_material(model,
    name="Gypsum Plaster", category="gypsum")

# Create layer set
layer_set = ifcopenshell.api.material.add_material_set(model,
    name="External Cavity Wall", set_type="IfcMaterialLayerSet")

# Add layers from OUTSIDE to INSIDE
# Layer 1: Facing brick (102mm)
layer1 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=facing_brick)
ifcopenshell.api.material.edit_layer(model,
    layer=layer1, attributes={"LayerThickness": 0.102})

# Layer 2: Insulation (100mm)
layer2 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=mineral_wool)
ifcopenshell.api.material.edit_layer(model,
    layer=layer2, attributes={"LayerThickness": 0.100})

# Layer 3: Concrete block (100mm)
layer3 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=concrete_block)
ifcopenshell.api.material.edit_layer(model,
    layer=layer3, attributes={"LayerThickness": 0.100})

# Layer 4: Plaster finish (13mm)
layer4 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=plaster)
ifcopenshell.api.material.edit_layer(model,
    layer=layer4, attributes={"LayerThickness": 0.013})

# Create wall type and assign layer set to the TYPE
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="External Cavity Wall 315mm")
ifcopenshell.api.material.assign_material(model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)

# Create wall occurrences and assign type
wall1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="W-001")
wall2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="W-002")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)

# Verify: walls inherit material from type
import ifcopenshell.util.element
mat = ifcopenshell.util.element.get_material(wall1)
print(f"Material type: {mat.is_a()}")  # IfcMaterialLayerSetUsage
```

### Slab with Layered Construction

```python
# IfcOpenShell — all schema versions
# Floor slab: screed + concrete + insulation
screed = ifcopenshell.api.material.add_material(model,
    name="Cement Screed", category="concrete")
reinforced_concrete = ifcopenshell.api.material.add_material(model,
    name="Reinforced Concrete C35/45", category="concrete")
eps = ifcopenshell.api.material.add_material(model,
    name="EPS Insulation", category="plastic")

slab_layers = ifcopenshell.api.material.add_material_set(model,
    name="Floor Slab 350mm", set_type="IfcMaterialLayerSet")

# Top to bottom
s1 = ifcopenshell.api.material.add_layer(model,
    layer_set=slab_layers, material=screed)
ifcopenshell.api.material.edit_layer(model,
    layer=s1, attributes={"LayerThickness": 0.065})

s2 = ifcopenshell.api.material.add_layer(model,
    layer_set=slab_layers, material=reinforced_concrete)
ifcopenshell.api.material.edit_layer(model,
    layer=s2, attributes={"LayerThickness": 0.200})

s3 = ifcopenshell.api.material.add_layer(model,
    layer_set=slab_layers, material=eps)
ifcopenshell.api.material.edit_layer(model,
    layer=s3, attributes={"LayerThickness": 0.085})

slab_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlabType", name="Floor Slab 350mm")
ifcopenshell.api.material.assign_material(model,
    products=[slab_type], type="IfcMaterialLayerSet", material=slab_layers)
```

---

## 3. Profiled Material (IfcMaterialProfileSet)

### Steel I-Beam with Profile

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Structural Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Create steel material
steel = ifcopenshell.api.material.add_material(model,
    name="S355 J2", category="steel")

# Create profile set
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")

# Create I-shape profile definition
profile_def = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef",
    OverallWidth=0.200, OverallDepth=0.190,
    WebThickness=0.0065, FlangeThickness=0.010)

# Add profile to set with material
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=profile_def)

# Create beam type and assign
beam_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeamType", name="HEA 200 Beam")
ifcopenshell.api.material.assign_material(model,
    products=[beam_type], type="IfcMaterialProfileSet", material=profile_set)

# Create beam occurrence
beam = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="B-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[beam], relating_type=beam_type)
```

### Circular Column with Profile

```python
# IfcOpenShell — all schema versions
# Steel circular hollow section column
steel = ifcopenshell.api.material.add_material(model,
    name="S275", category="steel")

chs_set = ifcopenshell.api.material.add_material_set(model,
    name="CHS 219.1x8.0", set_type="IfcMaterialProfileSet")

# Circular hollow section profile
chs_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcCircleHollowProfileDef",
    Radius=0.10955, WallThickness=0.008)

ifcopenshell.api.material.add_profile(model,
    profile_set=chs_set, material=steel, profile=chs_profile)

column_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="CHS 219.1x8.0 Column")
ifcopenshell.api.material.assign_material(model,
    products=[column_type], type="IfcMaterialProfileSet", material=chs_set)
```

---

## 4. Constituent Material (IFC4+ Only)

### Window with Frame and Glazing Constituents

```python
# IfcOpenShell — IFC4 and IFC4X3 ONLY
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Create materials
aluminium = ifcopenshell.api.material.add_material(model,
    name="Aluminium 6063-T5", category="aluminium")
double_glazing = ifcopenshell.api.material.add_material(model,
    name="Double Glazing 4-16-4", category="glass")
seal = ifcopenshell.api.material.add_material(model,
    name="EPDM Rubber Seal", category="plastic")

# Create constituent set
window_mat = ifcopenshell.api.material.add_material_set(model,
    name="Standard Window Assembly", set_type="IfcMaterialConstituentSet")

# Add named constituents
frame_const = ifcopenshell.api.material.add_constituent(model,
    constituent_set=window_mat, material=aluminium, name="Frame")
glazing_const = ifcopenshell.api.material.add_constituent(model,
    constituent_set=window_mat, material=double_glazing, name="Glazing")
seal_const = ifcopenshell.api.material.add_constituent(model,
    constituent_set=window_mat, material=seal, name="Seal")

# Assign to window type
window_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWindowType", name="Standard Window 1200x1500")
ifcopenshell.api.material.assign_material(model,
    products=[window_type], type="IfcMaterialConstituentSet",
    material=window_mat)

# Create window occurrence
window = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWindow", name="WIN-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[window], relating_type=window_type)

# Verify constituents
import ifcopenshell.util.element
mat = ifcopenshell.util.element.get_material(window)
if mat and mat.is_a("IfcMaterialConstituentSet"):
    for c in mat.MaterialConstituents:
        print(f"  {c.Name}: {c.Material.Name}")
```

---

## 5. Schema-Aware Material Assignment

### Handling IFC2X3 vs IFC4+ Differences

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

def assign_composite_material(model, element_type, materials_dict):
    """Assign composite material in a schema-aware manner.

    Args:
        model: The IFC file
        element_type: The element type to assign to
        materials_dict: dict of {name: IfcMaterial} pairs
    """
    if model.schema == "IFC2X3":
        # IFC2X3: Use IfcMaterialList (no named parts)
        mat_list = ifcopenshell.api.material.add_material_set(model,
            name="Composite", set_type="IfcMaterialList")
        for name, material in materials_dict.items():
            ifcopenshell.api.material.add_list_item(model,
                material_list=mat_list, material=material)
        ifcopenshell.api.material.assign_material(model,
            products=[element_type], type="IfcMaterialList", material=mat_list)
    else:
        # IFC4+: Use IfcMaterialConstituentSet (named parts)
        constituent_set = ifcopenshell.api.material.add_material_set(model,
            name="Composite", set_type="IfcMaterialConstituentSet")
        for name, material in materials_dict.items():
            ifcopenshell.api.material.add_constituent(model,
                constituent_set=constituent_set, material=material, name=name)
        ifcopenshell.api.material.assign_material(model,
            products=[element_type], type="IfcMaterialConstituentSet",
            material=constituent_set)

# Usage
materials = {
    "Frame": aluminium,
    "Glazing": glass,
}
assign_composite_material(model, window_type, materials)
```

---

## 6. Querying and Inspecting Materials

### Get Material from Element (with Type Inheritance)

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

def describe_material(element):
    """Print material information for any IFC element."""
    mat = ifcopenshell.util.element.get_material(element)

    if mat is None:
        print(f"{element.Name}: No material assigned")
        return

    mat_type = mat.is_a()
    print(f"{element.Name}: {mat_type}")

    if mat_type == "IfcMaterial":
        print(f"  Material: {mat.Name}")

    elif mat_type == "IfcMaterialLayerSetUsage":
        layer_set = mat.ForLayerSet
        print(f"  Layer Set: {layer_set.LayerSetName}")
        total = 0.0
        for layer in layer_set.MaterialLayers:
            t = layer.LayerThickness
            total += t
            print(f"  Layer: {layer.Material.Name} ({t*1000:.0f}mm)")
        print(f"  Total thickness: {total*1000:.0f}mm")

    elif mat_type == "IfcMaterialLayerSet":
        print(f"  Layer Set: {mat.LayerSetName}")
        for layer in mat.MaterialLayers:
            print(f"  Layer: {layer.Material.Name} ({layer.LayerThickness*1000:.0f}mm)")

    elif mat_type == "IfcMaterialProfileSet":
        print(f"  Profile Set: {mat.Name}")
        for profile in mat.MaterialProfiles:
            print(f"  Profile: {profile.Material.Name if profile.Material else 'None'}")

    elif mat_type == "IfcMaterialProfileSetUsage":
        ps = mat.ForProfileSet
        print(f"  Profile Set: {ps.Name}")
        for profile in ps.MaterialProfiles:
            print(f"  Profile: {profile.Material.Name if profile.Material else 'None'}")

    elif mat_type == "IfcMaterialConstituentSet":
        print(f"  Constituent Set: {mat.Name}")
        for c in mat.MaterialConstituents:
            print(f"  {c.Name}: {c.Material.Name}")

    elif mat_type == "IfcMaterialList":
        for m in mat.Materials:
            print(f"  Material: {m.Name}")

# Usage
for wall in model.by_type("IfcWall"):
    describe_material(wall)
```

### Find All Elements Using a Specific Material

```python
# IfcOpenShell — all schema versions
def find_elements_by_material_name(model, material_name):
    """Find all elements that use a material with the given name."""
    results = []
    for rel in model.by_type("IfcRelAssociatesMaterial"):
        mat = rel.RelatingMaterial
        if mat.is_a("IfcMaterial") and mat.Name == material_name:
            results.extend(rel.RelatedObjects)
        elif mat.is_a("IfcMaterialLayerSet"):
            for layer in mat.MaterialLayers:
                if layer.Material and layer.Material.Name == material_name:
                    results.extend(rel.RelatedObjects)
                    break
        elif mat.is_a("IfcMaterialConstituentSet"):
            if hasattr(mat, "MaterialConstituents") and mat.MaterialConstituents:
                for c in mat.MaterialConstituents:
                    if c.Material and c.Material.Name == material_name:
                        results.extend(rel.RelatedObjects)
                        break
    return results

elements = find_elements_by_material_name(model, "Concrete C30/37")
for el in elements:
    print(f"  {el.is_a()}: {el.Name}")
```

---

## 7. Material Modification Operations

### Replace Material in a Layer

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

# Get existing layer set from wall type
mat = ifcopenshell.util.element.get_material(wall_type, should_skip_usage=True)
if mat and mat.is_a("IfcMaterialLayerSet"):
    for layer in mat.MaterialLayers:
        if layer.Material.Name == "Mineral Wool":
            # Create replacement material
            pir = ifcopenshell.api.material.add_material(model,
                name="PIR Board", category="plastic")
            # Update the layer's material
            ifcopenshell.api.material.edit_layer(model,
                layer=layer, material=pir)
            break
```

### Add a Layer to an Existing Layer Set

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

mat = ifcopenshell.util.element.get_material(wall_type, should_skip_usage=True)
if mat and mat.is_a("IfcMaterialLayerSet"):
    # Add a new interior finish layer
    paint = ifcopenshell.api.material.add_material(model,
        name="Interior Paint", category="plastic")
    new_layer = ifcopenshell.api.material.add_layer(model,
        layer_set=mat, material=paint)
    ifcopenshell.api.material.edit_layer(model,
        layer=new_layer, attributes={"LayerThickness": 0.001})
```

### Copy and Modify a Material

```python
# IfcOpenShell — all schema versions
# Create a variant of an existing material
original = ifcopenshell.api.material.add_material(model,
    name="Concrete C30/37", category="concrete")

copied = ifcopenshell.api.material.copy_material(model, material=original)
ifcopenshell.api.material.edit_material(model,
    material=copied, attributes={"Name": "Concrete C40/50"})
```

---

## 8. Material with Visual Style

### Apply Color to Material Representation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Set up geometry context
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Painted Wall")
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.eye(4))

# Assign physical material
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete", category="concrete")
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)

# Create visual style (separate from physical material)
style = ifcopenshell.api.style.add_style(model, name="Concrete Grey")
ifcopenshell.api.style.add_surface_style(model, style=style,
    ifc_class="IfcSurfaceStyleShading",
    attributes={
        "SurfaceColour": {"Name": None, "Red": 0.7, "Green": 0.7, "Blue": 0.7}
    })

# Assign style to the wall's body representation
ifcopenshell.api.style.assign_representation_styles(model,
    shape_representation=representation, styles=[style])

model.write("wall_with_material_and_style.ifc")
```

---

## 9. Complete Workflow: Building with Full Material Setup

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

# === File Setup ===
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Office Building")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# === Spatial Structure ===
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Block A")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])

# === Materials ===
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete C30/37", category="concrete")
brick = ifcopenshell.api.material.add_material(model,
    name="Facing Brick", category="brick")
insulation = ifcopenshell.api.material.add_material(model,
    name="Rock Wool 120mm", category="glass")
steel = ifcopenshell.api.material.add_material(model,
    name="S355", category="steel")
aluminium = ifcopenshell.api.material.add_material(model,
    name="Aluminium Frame", category="aluminium")
glass_mat = ifcopenshell.api.material.add_material(model,
    name="Double Glazing", category="glass")

# === Wall Type: Layered ===
wall_layers = ifcopenshell.api.material.add_material_set(model,
    name="External Wall 322mm", set_type="IfcMaterialLayerSet")
l1 = ifcopenshell.api.material.add_layer(model,
    layer_set=wall_layers, material=brick)
ifcopenshell.api.material.edit_layer(model, layer=l1,
    attributes={"LayerThickness": 0.102})
l2 = ifcopenshell.api.material.add_layer(model,
    layer_set=wall_layers, material=insulation)
ifcopenshell.api.material.edit_layer(model, layer=l2,
    attributes={"LayerThickness": 0.120})
l3 = ifcopenshell.api.material.add_layer(model,
    layer_set=wall_layers, material=concrete)
ifcopenshell.api.material.edit_layer(model, layer=l3,
    attributes={"LayerThickness": 0.100})

wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="EXT-WALL-322")
ifcopenshell.api.material.assign_material(model,
    products=[wall_type], type="IfcMaterialLayerSet", material=wall_layers)

# === Column Type: Profiled ===
col_profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEB 200", set_type="IfcMaterialProfileSet")
i_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef",
    OverallWidth=0.200, OverallDepth=0.200,
    WebThickness=0.009, FlangeThickness=0.015)
ifcopenshell.api.material.add_profile(model,
    profile_set=col_profile_set, material=steel, profile=i_profile)

column_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="COL-HEB200")
ifcopenshell.api.material.assign_material(model,
    products=[column_type], type="IfcMaterialProfileSet", material=col_profile_set)

# === Window Type: Constituent ===
win_constituents = ifcopenshell.api.material.add_material_set(model,
    name="Standard Window", set_type="IfcMaterialConstituentSet")
ifcopenshell.api.material.add_constituent(model,
    constituent_set=win_constituents, material=aluminium, name="Frame")
ifcopenshell.api.material.add_constituent(model,
    constituent_set=win_constituents, material=glass_mat, name="Glazing")

window_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWindowType", name="WIN-1200x1500")
ifcopenshell.api.material.assign_material(model,
    products=[window_type], type="IfcMaterialConstituentSet",
    material=win_constituents)

# === Slab: Single Material ===
slab_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlabType", name="SLAB-200")
ifcopenshell.api.material.assign_material(model,
    products=[slab_type], material=concrete)

# === Create Occurrences ===
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="W-001")
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="C-001")
window = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWindow", name="WIN-001")
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="SL-001")

# Assign types
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[column], relating_type=column_type)
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[window], relating_type=window_type)
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[slab], relating_type=slab_type)

# Assign to storey
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall, column, window, slab])

# === Save ===
model.write("office_building_materials.ifc")
print("Model saved with full material setup.")
```
