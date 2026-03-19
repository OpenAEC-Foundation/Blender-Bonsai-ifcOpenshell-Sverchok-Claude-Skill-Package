---
name: ifcos-impl-profiles
description: >
  Use when creating structural cross-sections or profile-based extrusions in IFC -- I-beams,
  C-channels, rectangles, circles, or arbitrary polyline profiles. Prevents the common mistake
  of defining profiles without proper placement (profile origin misalignment). Covers
  parametric profiles, arbitrary profiles, profile-based extrusions, and material profile sets.
  Keywords: profile, IfcProfileDef, I-beam, C-channel, rectangle, circle, extrusion,
  cross-section, material profile set, structural profile.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Profile Definitions Implementation Guide

## Quick Reference

### Decision Tree: Which Profile Type to Use

```
What cross-section shape do you need?
├── Standard structural section (I-beam, channel, angle, tube)?
│   └── Parametric profile → profile.add_parameterised_profile()
│       ├── I-beam / H-beam → ifc_class="IfcIShapeProfileDef"
│       ├── Asymmetric I-beam → ifc_class="IfcAsymmetricIShapeProfileDef"
│       ├── Channel (U-shape) → ifc_class="IfcUShapeProfileDef"
│       ├── C-shape → ifc_class="IfcCShapeProfileDef"
│       ├── T-shape → ifc_class="IfcTShapeProfileDef"
│       ├── Angle (L-shape) → ifc_class="IfcLShapeProfileDef"
│       ├── Z-shape → ifc_class="IfcZShapeProfileDef"
│       ├── Rectangle → ifc_class="IfcRectangleProfileDef"
│       ├── Hollow rectangle → ifc_class="IfcRectangleHollowProfileDef"
│       ├── Rounded rectangle → ifc_class="IfcRoundedRectangleProfileDef"
│       ├── Circle → ifc_class="IfcCircleProfileDef"
│       ├── Hollow circle / tube → ifc_class="IfcCircleHollowProfileDef"
│       ├── Ellipse → ifc_class="IfcEllipseProfileDef"
│       └── Trapezoid → ifc_class="IfcTrapeziumProfileDef"
│
├── Custom non-standard solid shape?
│   └── Arbitrary profile → profile.add_arbitrary_profile()
│       └── Provide list of 2D coordinate tuples (SI meters)
│
├── Custom shape with holes / voids?
│   └── Arbitrary profile with voids → profile.add_arbitrary_profile_with_voids()
│       └── Provide outer_profile + inner_profiles coordinate lists
│
└── Combination of multiple profiles?
    └── Create individual profiles, then combine via
        IfcCompositeProfileDef (manual entity creation required)
```

### Decision Tree: How to Use a Profile

```
What do you want to do with the profile?
├── Create extruded geometry (3D shape from 2D profile)?
│   └── geometry.add_profile_representation(context, profile, depth)
│       └── Then assign to element with geometry.assign_representation()
│
├── Define material cross-section for beam/column?
│   └── material.add_material_set(set_type="IfcMaterialProfileSet")
│       └── material.add_profile(profile_set, material, profile)
│           └── Assign to element TYPE with material.assign_material()
│
└── Both (geometry + material definition)?
    └── Create profile ONCE, reference it in BOTH places
        └── Same IfcProfileDef is shared between representation and material
```

### Critical Warnings

- **ALWAYS** use `add_parameterised_profile` (British spelling). `add_parameterized_profile` (American spelling) does NOT exist and raises an error.
- **ALWAYS** specify profile dimensions in SI meters. A 200mm flange width MUST be `0.200`, NOT `200`.
- **ALWAYS** set dimension attributes via `profile.edit_profile()` after creating a parametric profile. The `add_parameterised_profile()` function creates the profile entity but dimension attributes default to zero/null.
- **ALWAYS** set `ProfileName` via `edit_profile()` for identification in BIM viewers and schedules.
- **NEVER** create profile entities directly with `model.create_entity("IfcIShapeProfileDef", ...)`. ALWAYS use `profile.add_parameterised_profile()` which handles proper schema setup.
- **NEVER** leave profile polylines open in `add_arbitrary_profile()`. The API auto-closes the loop, but relying on this is fragile. Provide explicit closure or ensure coordinates define a proper closed polygon.
- **NEVER** confuse `profile.edit_profile()` (edits IfcProfileDef dimensions) with `material.edit_profile()` (edits IfcMaterialProfile properties like name, description, priority).
- **ALWAYS** verify that the `ifc_class` string matches an IFC schema class name exactly (case-sensitive). `"IfcIshapeProfileDef"` fails silently.

---

## Essential Patterns

### Pattern 1: Create a Parametric I-Beam Profile

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Step 1: Create profile entity
i_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")

# Step 2: Set dimensions (ALWAYS in SI meters)
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
```

### Pattern 2: Create a Rectangular Profile

```python
# IfcOpenShell — all schema versions
rect_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcRectangleProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=rect_profile,
    attributes={
        "ProfileName": "RC 400x600",
        "XDim": 0.400,
        "YDim": 0.600
    })
```

### Pattern 3: Create a Circular Hollow Section

```python
# IfcOpenShell — all schema versions
chs_profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcCircleHollowProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=chs_profile,
    attributes={
        "ProfileName": "CHS 168.3x8",
        "Radius": 0.08415,
        "WallThickness": 0.008
    })
```

### Pattern 4: Create an Arbitrary Profile from Coordinates

```python
# IfcOpenShell — all schema versions
# Custom L-shaped profile (non-standard dimensions)
custom_profile = ifcopenshell.api.profile.add_arbitrary_profile(model,
    profile=[
        (0.0, 0.0),
        (0.3, 0.0),
        (0.3, 0.05),
        (0.05, 0.05),
        (0.05, 0.2),
        (0.0, 0.2)
    ],
    name="Custom L 300x200x50")
```

### Pattern 5: Create a Profile with Voids

```python
# IfcOpenShell — all schema versions
hollow_profile = ifcopenshell.api.profile.add_arbitrary_profile_with_voids(model,
    outer_profile=[
        (0.0, 0.0), (0.5, 0.0), (0.5, 0.5), (0.0, 0.5)
    ],
    inner_profiles=[
        [(0.05, 0.05), (0.45, 0.05), (0.45, 0.45), (0.05, 0.45)]
    ],
    name="Hollow Box 500x500")
```

### Pattern 6: Profile-Based Extrusion (3D Geometry)

```python
# IfcOpenShell — all schema versions
# Requires representation context from project bootstrap
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create column with I-profile extrusion
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C-01")
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body,
    profile=i_profile,
    depth=3.5)  # extrusion height in meters
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=column)
```

### Pattern 7: Material Profile Set (Profile + Material for Structural Elements)

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# Step 1: Create material
steel = ifcopenshell.api.material.add_material(model,
    name="S355 J2", category="steel")

# Step 2: Create profile set container
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")

# Step 3: Create profile definition
profile_def = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile_def,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010,
        "FilletRadius": 0.018
    })

# Step 4: Add profile to material set
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=profile_def)

# Step 5: Assign to element TYPE (best practice)
beam_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBeamType", name="HEA 200 Beam")
ifcopenshell.api.material.assign_material(model,
    products=[beam_type], type="IfcMaterialProfileSet", material=profile_set)
```

---

## Parametric Profile Types Reference

### Dimension Parameters by Profile Type

| IFC Class | Shape | Key Parameters (all in meters) |
|-----------|-------|-------------------------------|
| `IfcRectangleProfileDef` | Rectangle | `XDim`, `YDim` |
| `IfcRectangleHollowProfileDef` | Hollow rectangle | `XDim`, `YDim`, `WallThickness`, `InnerFilletRadius`*, `OuterFilletRadius`* |
| `IfcRoundedRectangleProfileDef` | Rounded rectangle | `XDim`, `YDim`, `RoundingRadius` |
| `IfcCircleProfileDef` | Circle | `Radius` |
| `IfcCircleHollowProfileDef` | Hollow circle / tube | `Radius`, `WallThickness` |
| `IfcEllipseProfileDef` | Ellipse | `SemiAxis1`, `SemiAxis2` |
| `IfcIShapeProfileDef` | I/H-beam | `OverallWidth`, `OverallDepth`, `WebThickness`, `FlangeThickness`, `FilletRadius`* |
| `IfcAsymmetricIShapeProfileDef` | Asymmetric I | `BottomFlangeWidth`, `OverallDepth`, `WebThickness`, `BottomFlangeThickness`, `TopFlangeWidth`*, `TopFlangeThickness`* |
| `IfcTShapeProfileDef` | T-shape | `Depth`, `FlangeWidth`, `WebThickness`, `FlangeThickness`, `FilletRadius`*, `WebEdgeRadius`* |
| `IfcLShapeProfileDef` | L-shape / angle | `Depth`, `Width`*, `Thickness`, `FilletRadius`*, `EdgeRadius`* |
| `IfcUShapeProfileDef` | U-shape / channel | `Depth`, `FlangeWidth`, `WebThickness`, `FlangeThickness`, `FilletRadius`* |
| `IfcCShapeProfileDef` | C-shape | `Depth`, `Width`, `WallThickness`, `Girth`, `InternalFilletRadius`* |
| `IfcZShapeProfileDef` | Z-shape | `Depth`, `FlangeWidth`, `WebThickness`, `FlangeThickness`, `FilletRadius`* |
| `IfcTrapeziumProfileDef` | Trapezoid | `BottomXDim`, `TopXDim`, `YDim`, `TopXOffset` |

*Parameters marked with `*` are optional.*

### Profile Type Parameter

The `profile_type` parameter in `add_parameterised_profile` accepts:

| Value | Meaning |
|-------|---------|
| `AREA` | Used for solid extrusions (default, most common) |
| `CURVE` | Used for curve-based/centerline representations |

**ALWAYS** use `AREA` (the default) for standard structural profiles. Use `CURVE` only for centerline-based representations.

---

## Common Operations

### Copy a Profile

```python
# IfcOpenShell — all schema versions
copied_profile = ifcopenshell.api.profile.copy_profile(model, profile=i_profile)
# Returns a new profile with identical dimensions, disconnected from original elements
```

### Modify Profile Dimensions

```python
# IfcOpenShell — all schema versions
# Change I-beam dimensions (e.g., upgrade from HEA 200 to HEA 300)
ifcopenshell.api.profile.edit_profile(model,
    profile=i_profile,
    attributes={
        "ProfileName": "HEA 300",
        "OverallWidth": 0.300,
        "OverallDepth": 0.290,
        "WebThickness": 0.0085,
        "FlangeThickness": 0.014,
        "FilletRadius": 0.027
    })
```

### Remove a Profile

```python
# IfcOpenShell — all schema versions
# ALWAYS ensure profile is not referenced by geometry or material sets first
ifcopenshell.api.profile.remove_profile(model, profile=unused_profile)
```

### Query Profile from a Structural Element

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

mat = ifcopenshell.util.element.get_material(beam, should_skip_usage=True)
if mat and mat.is_a("IfcMaterialProfileSet"):
    for mp in mat.MaterialProfiles:
        profile = mp.Profile
        if profile.is_a("IfcIShapeProfileDef"):
            print(f"Profile: {profile.ProfileName}")
            print(f"  Width: {profile.OverallWidth * 1000:.0f}mm")
            print(f"  Depth: {profile.OverallDepth * 1000:.0f}mm")
            print(f"  Web: {profile.WebThickness * 1000:.1f}mm")
            print(f"  Flange: {profile.FlangeThickness * 1000:.1f}mm")
```

---

## Integration Points

### Profile + Geometry

Profiles are used with `geometry.add_profile_representation()` to generate extruded solid geometry. The profile defines the 2D cross-section; the `depth` parameter defines the extrusion length.

```python
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=my_profile, depth=6.0)
```

### Profile + Material

Profiles are used with `material.add_profile()` to define material cross-sections for structural analysis and BIM data exchange. A single IfcProfileDef can be shared between geometry and material definitions.

### Profile + Type

ALWAYS assign material profile sets to element types (IfcBeamType, IfcColumnType, IfcMemberType), not individual occurrences. Occurrences inherit from their type.

---

## Version Notes

### Schema Compatibility

All `ifcopenshell.api.profile` functions work identically across IFC2X3, IFC4, and IFC4X3. Profile definitions are schema-stable entities — the same IfcProfileDef classes and attributes are available in all schema versions.

### IfcOpenShell Version Notes

- Since IfcOpenShell v0.8+, use keyword arguments for all parameters after the model.
- Both `api.run("profile.add_parameterised_profile", model, ...)` and `ifcopenshell.api.profile.add_parameterised_profile(model, ...)` are valid. Use one style consistently.

---

## Dependencies

- **ifcos-syntax-api** — For `api.run()` invocation patterns and module overview
- **ifcos-impl-materials** — For material profile set creation and assignment
- **ifcos-impl-geometry** — For profile-based representation creation

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all profile API functions
- [Working Code Examples](references/examples.md) — End-to-end profile workflows
- [Anti-Patterns](references/anti-patterns.md) — Common profile mistakes and how to avoid them
