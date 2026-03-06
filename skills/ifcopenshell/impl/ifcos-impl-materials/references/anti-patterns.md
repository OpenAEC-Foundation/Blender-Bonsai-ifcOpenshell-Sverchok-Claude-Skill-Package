# Material Assignment Anti-Patterns

Common mistakes when working with IfcOpenShell material operations. Each anti-pattern explains WHAT is wrong and WHY it causes problems.

---

## AP-1: Creating Material Entities Directly with create_entity()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")
material = model.create_entity("IfcMaterial", Name="Concrete")
rel = model.create_entity("IfcRelAssociatesMaterial",
    GlobalId=ifcopenshell.guid.new(),
    RelatingMaterial=material,
    RelatedObjects=[wall])
```

### Correct

```python
# IfcOpenShell — IFC4
import ifcopenshell.api

concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete", category="concrete")
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)
```

### WHY

Using `create_entity()` bypasses the API's validation and relationship management. The API handles OwnerHistory, proper relationship setup, automatic unassignment of previous materials, and schema-specific attribute handling. Direct entity creation leads to incomplete relationships that break IFC validators and downstream tools.

---

## AP-2: Setting Material via Direct Attribute Assignment

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
wall.HasAssociations = [material_rel]
# OR
wall.Material = concrete
```

### Correct

```python
# IfcOpenShell — all schema versions
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)
```

### WHY

IFC relationships are separate entities (IfcRelAssociatesMaterial), not direct attributes on elements. Setting attributes directly does not create the required relationship entity. The `Material` attribute does not exist on IfcWall. `HasAssociations` is an inverse attribute (read-only, computed from relationships). ALWAYS use the API for material assignment.

---

## AP-3: Assigning Material to Occurrences Instead of Types

### Wrong

```python
# IfcOpenShell — BAD PRACTICE (unnecessarily verbose)
for wall in model.by_type("IfcWall"):
    ifcopenshell.api.material.assign_material(model,
        products=[wall], material=layer_set)
```

### Correct

```python
# IfcOpenShell — all schema versions
# Assign to the TYPE once — all occurrences inherit
ifcopenshell.api.material.assign_material(model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)
```

### WHY

In IFC, material assignment follows the type-occurrence pattern. Assigning material to the element type (IfcWallType, IfcSlabType, etc.) means all occurrences of that type automatically inherit the material. Assigning to individual occurrences creates redundant IfcRelAssociatesMaterial entities, increases file size, and makes bulk changes difficult. Assign to occurrences only when an individual element needs a material different from its type.

---

## AP-4: Using IfcMaterialConstituentSet in IFC2X3

### Wrong

```python
# IfcOpenShell — IFC2X3 — BAD PRACTICE
model = ifcopenshell.file(schema="IFC2X3")
constituent_set = ifcopenshell.api.material.add_material_set(model,
    name="Window", set_type="IfcMaterialConstituentSet")
# ERROR: IfcMaterialConstituentSet does not exist in IFC2X3
```

### Correct

```python
# IfcOpenShell — all schema versions
if model.schema == "IFC2X3":
    mat_list = ifcopenshell.api.material.add_material_set(model,
        name="Window", set_type="IfcMaterialList")
    ifcopenshell.api.material.add_list_item(model,
        material_list=mat_list, material=aluminium)
    ifcopenshell.api.material.add_list_item(model,
        material_list=mat_list, material=glass)
else:
    constituent_set = ifcopenshell.api.material.add_material_set(model,
        name="Window", set_type="IfcMaterialConstituentSet")
    ifcopenshell.api.material.add_constituent(model,
        constituent_set=constituent_set, material=aluminium, name="Frame")
    ifcopenshell.api.material.add_constituent(model,
        constituent_set=constituent_set, material=glass, name="Glazing")
```

### WHY

`IfcMaterialConstituentSet` was introduced in IFC4. It does not exist in the IFC2X3 schema. Attempting to create it in an IFC2X3 file raises a schema validation error. For IFC2X3, use `IfcMaterialList` as the fallback (it provides an unordered material collection without named parts). ALWAYS check `model.schema` before using IFC4-specific material types.

---

## AP-5: Forgetting to Set LayerThickness After add_layer()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
layer = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=brick)
# Layer has thickness = 0.0 — invalid for wall geometry
```

### Correct

```python
# IfcOpenShell — all schema versions
layer = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=brick)
ifcopenshell.api.material.edit_layer(model,
    layer=layer, attributes={"LayerThickness": 0.102})
```

### WHY

`add_layer()` creates a layer with `LayerThickness = 0.0`. A zero-thickness layer is geometrically meaningless and causes wall/slab representations to have no visible geometry for that layer. IFC validators flag zero-thickness layers as errors. ALWAYS call `edit_layer()` immediately after `add_layer()` to set the correct thickness in model units (meters for SI).

---

## AP-6: Not Specifying set_type in add_material_set()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
layer_set = ifcopenshell.api.material.add_material_set(model,
    name="Wall Layers")
# Default set_type is "IfcMaterialConstituentSet", NOT "IfcMaterialLayerSet"
```

### Correct

```python
# IfcOpenShell — all schema versions
layer_set = ifcopenshell.api.material.add_material_set(model,
    name="Wall Layers", set_type="IfcMaterialLayerSet")
```

### WHY

The default `set_type` parameter for `add_material_set()` is `"IfcMaterialConstituentSet"`, NOT `"IfcMaterialLayerSet"`. Relying on the default creates the wrong type of material set, which causes `add_layer()` calls to fail because you cannot add layers to a constituent set. ALWAYS specify `set_type` explicitly.

---

## AP-7: Using Hallucinated API Functions

### Wrong

```python
# IfcOpenShell — DOES NOT EXIST
ifcopenshell.api.run("material.create_material", model, name="Steel")
ifcopenshell.api.run("material.set_material", model, element=wall, material=steel)
ifcopenshell.api.run("material.add_material_layer", model, ...)
wall.set_material(concrete)
```

### Correct

```python
# IfcOpenShell — all schema versions
steel = ifcopenshell.api.material.add_material(model, name="Steel")
ifcopenshell.api.material.assign_material(model, products=[wall], material=steel)
layer = ifcopenshell.api.material.add_layer(model, layer_set=layer_set, material=steel)
```

### WHY

AI models frequently hallucinate material API function names. The correct function names are:
- `add_material` (NOT `create_material`)
- `assign_material` (NOT `set_material`)
- `add_layer` (NOT `add_material_layer`)
- There is no `set_material()` method on elements

ALWAYS use the documented function names from `ifcopenshell.api.material.*`.

---

## AP-8: Adding Profile Without Profile Definition

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel)
# No profile definition — cross-section shape is undefined
```

### Correct

```python
# IfcOpenShell — all schema versions
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")
profile_def = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef",
    OverallWidth=0.200, OverallDepth=0.190,
    WebThickness=0.0065, FlangeThickness=0.010)
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=profile_def)
```

### WHY

An IfcMaterialProfile without an associated IfcProfileDef has no cross-section geometry. Geometry engines cannot generate extrusions for beams/columns without a profile definition. IFC validators flag profiles without profile definitions as incomplete. ALWAYS create an IfcProfileDef and pass it to `add_profile()`.

---

## AP-9: Removing Material Without Unassigning First

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
ifcopenshell.api.material.remove_material(model, material=concrete)
# Elements still reference this material via IfcRelAssociatesMaterial → broken references
```

### Correct

```python
# IfcOpenShell — all schema versions
# First unassign from all products that use this material
for rel in model.by_type("IfcRelAssociatesMaterial"):
    if rel.RelatingMaterial == concrete:
        ifcopenshell.api.material.unassign_material(model,
            products=list(rel.RelatedObjects))

# Then safely remove the material
ifcopenshell.api.material.remove_material(model, material=concrete)
```

### WHY

Removing a material while it is still referenced by IfcRelAssociatesMaterial relationships creates null references in the IFC file. These broken references cause errors in IFC viewers and validators. ALWAYS unassign materials from all products before deleting them.

---

## AP-10: Assigning Material Set with Wrong type Parameter

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
layer_set = ifcopenshell.api.material.add_material_set(model,
    name="Wall", set_type="IfcMaterialLayerSet")

# type= does not match the actual set type
ifcopenshell.api.material.assign_material(model,
    products=[wall_type], type="IfcMaterial", material=layer_set)
```

### Correct

```python
# IfcOpenShell — all schema versions
ifcopenshell.api.material.assign_material(model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)
```

### WHY

The `type` parameter in `assign_material()` must match the actual type of the material entity being assigned. Passing `type="IfcMaterial"` when assigning an `IfcMaterialLayerSet` causes the API to create incorrect relationship entities. The `type` parameter values must be one of: `"IfcMaterial"`, `"IfcMaterialLayerSet"`, `"IfcMaterialProfileSet"`, `"IfcMaterialConstituentSet"`, `"IfcMaterialList"`.

---

## AP-11: Confusing Material (Physical) with Style (Visual)

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
# Expecting material to automatically provide visual appearance
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete", category="concrete")
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)
# Wall has material but NO color/appearance in viewer
```

### Correct

```python
# IfcOpenShell — all schema versions
# Step 1: Physical material
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete", category="concrete")
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)

# Step 2: Visual style (separate)
style = ifcopenshell.api.style.add_style(model, name="Concrete Grey")
ifcopenshell.api.style.add_surface_style(model, style=style,
    ifc_class="IfcSurfaceStyleShading",
    attributes={"SurfaceColour": {"Name": None, "Red": 0.7, "Green": 0.7, "Blue": 0.7}})
ifcopenshell.api.style.assign_representation_styles(model,
    shape_representation=body_rep, styles=[style])
```

### WHY

In IFC, physical materials (IfcMaterial) and visual styles (IfcSurfaceStyle) are separate concepts. Assigning a material named "Concrete" does NOT automatically make the element appear grey in a viewer. Visual appearance requires a separate IfcSurfaceStyle assigned to the element's representation. If visual appearance is needed, ALWAYS create and assign both material and style.

---

## AP-12: Using api.run() String Syntax Inconsistently

### Wrong

```python
# IfcOpenShell — BAD PRACTICE (mixing old and new syntax inconsistently)
mat = ifcopenshell.api.run("material.add_material", model, name="Steel")
ifcopenshell.api.material.assign_material(model, products=[wall], material=mat)
```

### Correct (pick one style and be consistent)

```python
# Style A: Direct module call (recommended)
mat = ifcopenshell.api.material.add_material(model, name="Steel")
ifcopenshell.api.material.assign_material(model, products=[wall], material=mat)

# Style B: api.run() string syntax (also correct)
mat = ifcopenshell.api.run("material.add_material", model, name="Steel")
ifcopenshell.api.run("material.assign_material", model, products=[wall], material=mat)
```

### WHY

Both syntaxes are functionally equivalent and both are correct. However, mixing them inconsistently within the same codebase reduces readability. Choose one style and use it consistently. The direct module call syntax is recommended for new code because it provides better IDE autocompletion and type checking.
