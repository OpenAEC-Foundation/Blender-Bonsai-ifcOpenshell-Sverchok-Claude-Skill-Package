# Profile Definition Working Code Examples

All examples verified against the [IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/profile/index.html) and research in [supplementary-ifcos-gaps.md](../../../../../docs/research/supplementary-ifcos-gaps.md).

---

## 1. Steel I-Beam with Profile and Geometry

### Complete Beam with HEA 200 Profile

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

# === File Setup ===
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Structural Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Geometry context
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Spatial structure
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

# === Profile Definition ===
i_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=i_profile,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010,
        "FilletRadius": 0.018
    })

# === Material Profile Set ===
steel = ifcopenshell.api.material.add_material(model,
    name="S355 J2", category="steel")
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=i_profile)

# === Beam Type ===
beam_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeamType", name="HEA 200 Beam")
ifcopenshell.api.material.assign_material(model,
    products=[beam_type], type="IfcMaterialProfileSet", material=profile_set)

# === Beam Occurrence with Geometry ===
beam = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="B-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[beam], relating_type=beam_type)

# Create extruded geometry from profile
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=i_profile, depth=6.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=beam, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=beam, matrix=numpy.eye(4))

# Place in spatial structure
ifcopenshell.api.run("spatial.assign_container", model,
    products=[beam], relating_structure=storey)

model.write("steel_beam_hea200.ifc")
```

---

## 2. Steel Column with Circular Hollow Section

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# CHS profile (circular hollow section)
chs_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcCircleHollowProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=chs_profile,
    attributes={
        "ProfileName": "CHS 219.1x8.0",
        "Radius": 0.10955,       # outer radius = 219.1/2 mm in meters
        "WallThickness": 0.008
    })

# Material profile set
steel = ifcopenshell.api.material.add_material(model,
    name="S275", category="steel")
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="CHS 219.1x8.0", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=chs_profile)

# Column type
column_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="CHS 219.1x8.0 Column")
ifcopenshell.api.material.assign_material(model,
    products=[column_type], type="IfcMaterialProfileSet", material=profile_set)

# Column occurrence with geometry
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="C-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[column], relating_type=column_type)

representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=chs_profile, depth=3.5)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=column, matrix=numpy.eye(4))

model.write("steel_column_chs.ifc")
```

---

## 3. Concrete Rectangular Column

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Rectangular profile for concrete column
rect_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcRectangleProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=rect_profile,
    attributes={
        "ProfileName": "RC 400x600",
        "XDim": 0.400,
        "YDim": 0.600
    })

# Material profile set
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete C35/45", category="concrete")
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="RC 400x600", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=concrete, profile=rect_profile)

# Column type and occurrence
column_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="RC-COL-400x600")
ifcopenshell.api.material.assign_material(model,
    products=[column_type], type="IfcMaterialProfileSet", material=profile_set)

column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="C-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[column], relating_type=column_type)

representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=rect_profile, depth=3.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=column, matrix=numpy.eye(4))

model.write("concrete_column_rect.ifc")
```

---

## 4. Custom Arbitrary Profile (Non-Standard Shape)

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Custom T-shaped profile from polyline coordinates (all in meters)
# This is useful when standard IfcTShapeProfileDef doesn't match
custom_t = ifcopenshell.api.profile.add_arbitrary_profile(model,
    profile=[
        (-0.150, 0.0),      # bottom-left of web
        (0.150, 0.0),       # bottom-right of web
        (0.150, 0.250),     # right side, web meets flange
        (0.300, 0.250),     # right extent of flange
        (0.300, 0.300),     # top-right of flange
        (-0.300, 0.300),    # top-left of flange
        (-0.300, 0.250),    # left extent of flange
        (-0.150, 0.250),    # left side, web meets flange
    ],
    name="Custom T 600x300")

# Use in extrusion
member = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcMember", name="M-001")
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=custom_t, depth=4.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=member, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=member, matrix=numpy.eye(4))

model.write("custom_profile_member.ifc")
```

---

## 5. Hollow Box Profile with Voids

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Hollow box section via arbitrary profile with void
# Outer: 500x500mm, Wall: 50mm
hollow_box = ifcopenshell.api.profile.add_arbitrary_profile_with_voids(model,
    outer_profile=[
        (0.0, 0.0),
        (0.5, 0.0),
        (0.5, 0.5),
        (0.0, 0.5)
    ],
    inner_profiles=[
        [
            (0.05, 0.05),
            (0.45, 0.05),
            (0.45, 0.45),
            (0.05, 0.45)
        ]
    ],
    name="Hollow Box 500x500x50")

# Material and profile set
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete C40/50", category="concrete")
ps = ifcopenshell.api.material.add_material_set(model,
    name="Hollow Box 500", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=ps, material=concrete, profile=hollow_box)

# Column with hollow box
col_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="COL-HB500")
ifcopenshell.api.material.assign_material(model,
    products=[col_type], type="IfcMaterialProfileSet", material=ps)

column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="C-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[column], relating_type=col_type)

representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=hollow_box, depth=4.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=column, matrix=numpy.eye(4))

model.write("hollow_box_column.ifc")
```

---

## 6. Multiple Profile Types in One Project

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Mixed Steel Structure")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

steel = ifcopenshell.api.material.add_material(model,
    name="S355", category="steel")

# --- IPE 300 Beam Profile ---
ipe_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=ipe_profile,
    attributes={
        "ProfileName": "IPE 300",
        "OverallWidth": 0.150,
        "OverallDepth": 0.300,
        "WebThickness": 0.0071,
        "FlangeThickness": 0.0107,
        "FilletRadius": 0.015
    })

ipe_ps = ifcopenshell.api.material.add_material_set(model,
    name="IPE 300", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=ipe_ps, material=steel, profile=ipe_profile)

beam_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeamType", name="BEAM-IPE300")
ifcopenshell.api.material.assign_material(model,
    products=[beam_type], type="IfcMaterialProfileSet", material=ipe_ps)

# --- L-Shape Angle Brace ---
angle_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcLShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=angle_profile,
    attributes={
        "ProfileName": "L 100x100x10",
        "Depth": 0.100,
        "Width": 0.100,
        "Thickness": 0.010,
        "FilletRadius": 0.012
    })

angle_ps = ifcopenshell.api.material.add_material_set(model,
    name="L 100x100x10", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=angle_ps, material=steel, profile=angle_profile)

brace_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcMemberType", name="BRACE-L100")
ifcopenshell.api.material.assign_material(model,
    products=[brace_type], type="IfcMaterialProfileSet", material=angle_ps)

# --- U-Channel Purlin ---
channel_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcUShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=channel_profile,
    attributes={
        "ProfileName": "UPN 200",
        "Depth": 0.200,
        "FlangeWidth": 0.075,
        "WebThickness": 0.0085,
        "FlangeThickness": 0.0115,
        "FilletRadius": 0.0115
    })

channel_ps = ifcopenshell.api.material.add_material_set(model,
    name="UPN 200", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=channel_ps, material=steel, profile=channel_profile)

purlin_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcMemberType", name="PURLIN-UPN200")
ifcopenshell.api.material.assign_material(model,
    products=[purlin_type], type="IfcMaterialProfileSet", material=channel_ps)

# --- Create Occurrences ---
beam = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeam", name="B-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[beam], relating_type=beam_type)

brace = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcMember", name="BR-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[brace], relating_type=brace_type)

purlin = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcMember", name="P-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[purlin], relating_type=purlin_type)

# Create geometry for each
for element, profile, depth in [
    (beam, ipe_profile, 8.0),
    (brace, angle_profile, 5.0),
    (purlin, channel_profile, 6.0)
]:
    rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
        context=body, profile=profile, depth=depth)
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=element, representation=rep)
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=element, matrix=numpy.eye(4))

model.write("mixed_steel_structure.ifc")
```

---

## 7. Copy and Modify a Profile

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# Start with an existing HEA 200 profile
original = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=original,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010,
        "FilletRadius": 0.018
    })

# Copy and modify to create HEA 300
hea300 = ifcopenshell.api.profile.copy_profile(model, profile=original)
ifcopenshell.api.profile.edit_profile(model,
    profile=hea300,
    attributes={
        "ProfileName": "HEA 300",
        "OverallWidth": 0.300,
        "OverallDepth": 0.290,
        "WebThickness": 0.0085,
        "FlangeThickness": 0.014,
        "FilletRadius": 0.027
    })
# original remains HEA 200; hea300 is fully independent
```

---

## 8. Query Profiles from Existing Model

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("existing_model.ifc")

# Find all beams and their profiles
for beam in model.by_type("IfcBeam"):
    mat = ifcopenshell.util.element.get_material(beam, should_skip_usage=True)

    if mat is None:
        print(f"{beam.Name}: No material")
        continue

    if mat.is_a("IfcMaterialProfileSet"):
        print(f"{beam.Name}: {mat.Name}")
        for mp in mat.MaterialProfiles:
            profile = mp.Profile
            material = mp.Material
            mat_name = material.Name if material else "None"

            if profile.is_a("IfcIShapeProfileDef"):
                print(f"  I-Profile: {profile.ProfileName}")
                print(f"  Material: {mat_name}")
                print(f"  Width: {profile.OverallWidth * 1000:.0f}mm")
                print(f"  Depth: {profile.OverallDepth * 1000:.0f}mm")
                print(f"  Web: {profile.WebThickness * 1000:.1f}mm")
                print(f"  Flange: {profile.FlangeThickness * 1000:.1f}mm")
            elif profile.is_a("IfcCircleHollowProfileDef"):
                print(f"  CHS: {profile.ProfileName}")
                print(f"  Material: {mat_name}")
                print(f"  Diameter: {profile.Radius * 2000:.1f}mm")
                print(f"  Wall: {profile.WallThickness * 1000:.1f}mm")
            elif profile.is_a("IfcRectangleProfileDef"):
                print(f"  Rect: {profile.ProfileName}")
                print(f"  Material: {mat_name}")
                print(f"  Size: {profile.XDim * 1000:.0f}x{profile.YDim * 1000:.0f}mm")
            else:
                print(f"  Profile: {profile.is_a()} - {profile.ProfileName}")
                print(f"  Material: {mat_name}")
    elif mat.is_a("IfcMaterialProfileSetUsage"):
        ps = mat.ForProfileSet
        print(f"{beam.Name}: {ps.Name} (via usage)")
        for mp in ps.MaterialProfiles:
            print(f"  Profile: {mp.Profile.is_a()}")
```

---

## 9. Rectangular Hollow Section (RHS) Column

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# RHS (Rectangular Hollow Section) using IfcRectangleHollowProfileDef
rhs_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcRectangleHollowProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=rhs_profile,
    attributes={
        "ProfileName": "RHS 200x100x8",
        "XDim": 0.200,
        "YDim": 0.100,
        "WallThickness": 0.008,
        "InnerFilletRadius": 0.008,
        "OuterFilletRadius": 0.012
    })

# Use in material profile set
steel = ifcopenshell.api.material.add_material(model,
    name="S355", category="steel")
rhs_ps = ifcopenshell.api.material.add_material_set(model,
    name="RHS 200x100x8", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=rhs_ps, material=steel, profile=rhs_profile)
```

---

## 10. Circular Concrete Column

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# Solid circular profile for concrete column
circle_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcCircleProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=circle_profile,
    attributes={
        "ProfileName": "RC Circle 450",
        "Radius": 0.225   # 450mm diameter = 225mm radius
    })

# Material profile set
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete C30/37", category="concrete")
circle_ps = ifcopenshell.api.material.add_material_set(model,
    name="RC Circle 450", set_type="IfcMaterialProfileSet")
ifcopenshell.api.material.add_profile(model,
    profile_set=circle_ps, material=concrete, profile=circle_profile)

# Column type
col_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="COL-CIR-450")
ifcopenshell.api.material.assign_material(model,
    products=[col_type], type="IfcMaterialProfileSet", material=circle_ps)
```
