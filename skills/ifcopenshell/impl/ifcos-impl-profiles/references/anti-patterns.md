# Profile Definition Anti-Patterns

Common mistakes when working with IfcOpenShell profile operations. Each anti-pattern explains WHAT is wrong and WHY it causes problems.

---

## AP-1: Using American Spelling for add_parameterised_profile

### Wrong

```python
# IfcOpenShell — DOES NOT EXIST
profile = ifcopenshell.api.profile.add_parameterized_profile(model,
    ifc_class="IfcIShapeProfileDef")
# AttributeError: module has no attribute 'add_parameterized_profile'
```

### Correct

```python
# IfcOpenShell — all schema versions
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
```

### WHY

The IfcOpenShell API uses British spelling: `parameterised` (not `parameterized`). This is one of the most common AI hallucination errors. The function simply does not exist with American spelling. ALWAYS use `add_parameterised_profile`.

---

## AP-2: Using Millimeters Instead of Meters for Dimensions

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
ifcopenshell.api.profile.edit_profile(model,
    profile=i_profile,
    attributes={
        "OverallWidth": 200,        # 200 METERS, not 200mm!
        "OverallDepth": 190,        # 190 METERS, not 190mm!
        "WebThickness": 6.5,        # 6.5 METERS!
        "FlangeThickness": 10       # 10 METERS!
    })
```

### Correct

```python
# IfcOpenShell — all schema versions
ifcopenshell.api.profile.edit_profile(model,
    profile=i_profile,
    attributes={
        "OverallWidth": 0.200,      # 200mm = 0.200 meters
        "OverallDepth": 0.190,      # 190mm = 0.190 meters
        "WebThickness": 0.0065,     # 6.5mm = 0.0065 meters
        "FlangeThickness": 0.010    # 10mm = 0.010 meters
    })
```

### WHY

IfcOpenShell uses SI meters internally for all length measurements. Passing millimeters results in profiles that are 1000x too large. A beam profile that should be 200mm wide becomes 200 meters wide. This produces invalid geometry that may crash viewers or produce absurd models. ALWAYS convert to meters: divide mm values by 1000.

---

## AP-3: Creating Profile Entities Directly with create_entity()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
profile = model.create_entity("IfcIShapeProfileDef",
    ProfileType="AREA",
    ProfileName="HEA 200",
    OverallWidth=0.200,
    OverallDepth=0.190,
    WebThickness=0.0065,
    FlangeThickness=0.010)
```

### Correct

```python
# IfcOpenShell — all schema versions
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010
    })
```

### WHY

Using `model.create_entity()` bypasses the API's internal validation, position handling, and proper attribute setup. The `add_parameterised_profile()` function ensures the `ProfileType` attribute is set correctly, creates any required sub-entities (like position coordinates), and handles schema differences. Direct entity creation may produce profiles that appear valid but lack required sub-entities, causing failures in geometry processing.

---

## AP-4: Forgetting to Set Dimensions After add_parameterised_profile()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
# Profile dimensions are all zero/null — no visible geometry

# Using directly in geometry
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=6.0)
# Result: degenerate geometry (zero-width beam)
```

### Correct

```python
# IfcOpenShell — all schema versions
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010
    })
```

### WHY

`add_parameterised_profile()` creates the profile entity with default (zero/null) dimension values. It does NOT accept dimension parameters directly. ALWAYS call `edit_profile()` immediately after creation to set all required dimensions. Without dimensions, geometry extrusions produce degenerate (zero-area) shapes.

---

## AP-5: Confusing profile.edit_profile() with material.edit_profile()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
# Trying to set profile dimensions via material.edit_profile
ifcopenshell.api.material.edit_profile(model,
    profile=material_profile,
    attributes={"OverallWidth": 0.200})
# This edits the IfcMaterialProfile, NOT the IfcProfileDef
```

### Correct

```python
# IfcOpenShell — all schema versions
# profile.edit_profile → edits IfcProfileDef (dimensions)
ifcopenshell.api.profile.edit_profile(model,
    profile=profile_def,
    attributes={"OverallWidth": 0.200, "OverallDepth": 0.190})

# material.edit_profile → edits IfcMaterialProfile (name, description, priority)
ifcopenshell.api.material.edit_profile(model,
    profile=material_profile_item,
    attributes={"Name": "Main Profile"})
```

### WHY

There are two different `edit_profile` functions in IfcOpenShell:
- `ifcopenshell.api.profile.edit_profile()` — edits the **geometric shape** (IfcProfileDef dimensions like width, depth, thickness)
- `ifcopenshell.api.material.edit_profile()` — edits the **material profile item** (IfcMaterialProfile properties like name, description, priority)

Confusing these two functions causes silent failures where dimension changes don't take effect because you modified the wrong entity.

---

## AP-6: Incorrect ifc_class String (Case or Spelling Errors)

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIshapeProfileDef")    # Wrong: lowercase 's' in shape
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IFCIShapeProfileDef")    # Wrong: uppercase 'FC'
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIProfileDef")         # Wrong: missing 'Shape'
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcHShapeProfileDef")    # Wrong: no such class (use IfcIShapeProfileDef)
```

### Correct

```python
# IfcOpenShell — all schema versions
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")    # Exact IFC class name
```

### WHY

The `ifc_class` parameter must exactly match an IFC schema class name. IFC class names are case-sensitive and follow strict naming conventions. Common mistakes:
- `IfcIshapeProfileDef` (lowercase 's') — WRONG
- `IfcHShapeProfileDef` — does NOT exist (I-beams and H-beams both use `IfcIShapeProfileDef`)
- `IfcChannelProfileDef` — does NOT exist (use `IfcUShapeProfileDef` for channels)
- `IfcAngleProfileDef` — does NOT exist (use `IfcLShapeProfileDef` for angles)

ALWAYS copy the exact class name from the IFC specification.

---

## AP-7: Not Setting ProfileName for Identification

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile,
    attributes={
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010
    })
# Profile has no name — appears as "Unnamed" or blank in BIM viewers
```

### Correct

```python
# IfcOpenShell — all schema versions
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile,
    attributes={
        "ProfileName": "HEA 200",     # ALWAYS set for identification
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010
    })
```

### WHY

Without a `ProfileName`, profiles appear unnamed in BIM viewers, schedulers, and quantity takeoff tools. When a model has dozens of beams and columns, unnamed profiles make it impossible to identify which section is used where. `ProfileName` is optional in the schema but SHOULD always be set for practical use. Use standard steel section names (HEA 200, IPE 300, CHS 168.3x8) or descriptive names (RC 400x600).

---

## AP-8: Using Hallucinated Profile API Functions

### Wrong

```python
# IfcOpenShell — NONE OF THESE EXIST
ifcopenshell.api.profile.create_profile(model, ...)
ifcopenshell.api.profile.add_i_profile(model, ...)
ifcopenshell.api.profile.add_profile(model, ...)
ifcopenshell.api.run("profile.create_i_beam", model, ...)
ifcopenshell.api.run("profile.add_standard_profile", model, ...)
```

### Correct

```python
# IfcOpenShell — all schema versions
# The ONLY profile creation functions:
ifcopenshell.api.profile.add_parameterised_profile(model, ifc_class="...")
ifcopenshell.api.profile.add_arbitrary_profile(model, profile=[...])
ifcopenshell.api.profile.add_arbitrary_profile_with_voids(model, ...)
ifcopenshell.api.profile.edit_profile(model, profile=..., attributes={...})
ifcopenshell.api.profile.copy_profile(model, profile=...)
ifcopenshell.api.profile.remove_profile(model, profile=...)
```

### WHY

AI models frequently hallucinate profile API function names. The `ifcopenshell.api.profile` module has exactly 6 functions. There is no `create_profile`, no `add_i_profile`, no `add_profile` (that's in the material module), no `add_standard_profile`. ALWAYS use the documented function names.

---

## AP-9: Removing a Profile Still Referenced by Geometry or Materials

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
ifcopenshell.api.profile.remove_profile(model, profile=i_profile)
# If a beam's geometry or material profile set still references this profile,
# the model now has broken references
```

### Correct

```python
# IfcOpenShell — all schema versions
# First check for references
is_used = False

# Check material profile references
for mp in model.by_type("IfcMaterialProfile"):
    if mp.Profile == profile_to_remove:
        is_used = True
        break

# Check geometry references (swept area solids)
if not is_used:
    for solid in model.by_type("IfcSweptAreaSolid"):
        if solid.SweptArea == profile_to_remove:
            is_used = True
            break

if is_used:
    print("Profile is still in use — remove references first")
else:
    ifcopenshell.api.profile.remove_profile(model, profile=profile_to_remove)
```

### WHY

Removing a profile that is still referenced by `IfcMaterialProfile.Profile` or `IfcSweptAreaSolid.SweptArea` creates null references. These broken references cause geometry processing errors and IFC validation failures. ALWAYS verify that a profile is unreferenced before removing it.

---

## AP-10: Passing Dimension Parameters to add_parameterised_profile()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
# add_parameterised_profile does NOT accept dimension kwargs
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef",
    OverallWidth=0.200,
    OverallDepth=0.190,
    WebThickness=0.0065,
    FlangeThickness=0.010)
# These keyword arguments are silently ignored!
```

### Correct

```python
# IfcOpenShell — all schema versions
# Step 1: Create profile (only accepts ifc_class and profile_type)
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")

# Step 2: Set dimensions via edit_profile
ifcopenshell.api.profile.edit_profile(model,
    profile=profile,
    attributes={
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010
    })
```

### WHY

The `add_parameterised_profile()` function accepts only `ifc_class` and `profile_type` as parameters. Dimension attributes like `OverallWidth`, `OverallDepth`, etc. are NOT valid parameters for this function. They may be silently ignored or cause unexpected errors. ALWAYS use `edit_profile()` as a separate step to set dimensions.

---

## AP-11: Creating Material Profile Without Profile Definition

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel)
# Missing profile=... parameter — no cross-section geometry defined
```

### Correct

```python
# IfcOpenShell — all schema versions
# ALWAYS create an IfcProfileDef first
profile_def = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile_def,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010
    })

profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=profile_def)
```

### WHY

An `IfcMaterialProfile` without an associated `IfcProfileDef` has no cross-section geometry. Geometry engines cannot generate extrusions for beams/columns without a profile definition. IFC validators flag profiles without profile definitions as incomplete. ALWAYS create an IfcProfileDef first and pass it to `material.add_profile()`.

---

## AP-12: Using api.run() with Wrong Function Paths

### Wrong

```python
# IfcOpenShell — THESE ARE WRONG
ifcopenshell.api.run("profile.add_profile", model, ...)         # Wrong function name
ifcopenshell.api.run("profile.create_parameterised_profile", model, ...)  # Wrong function name
ifcopenshell.api.run("geometry.add_profile", model, ...)        # Wrong module
```

### Correct

```python
# IfcOpenShell — all schema versions
# Profile creation via api.run()
profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.run("profile.edit_profile", model,
    profile=profile, attributes={"OverallWidth": 0.200})

# OR direct module call (equivalent)
profile = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.profile.edit_profile(model,
    profile=profile, attributes={"OverallWidth": 0.200})
```

### WHY

The exact function paths for profile operations are:
- `profile.add_parameterised_profile` (NOT `add_profile`, NOT `create_parameterised_profile`)
- `profile.add_arbitrary_profile`
- `profile.add_arbitrary_profile_with_voids`
- `profile.edit_profile`
- `profile.copy_profile`
- `profile.remove_profile`

Profile creation is in the `profile` module, NOT in `geometry`. Geometry uses profiles but the creation functions are separate.
