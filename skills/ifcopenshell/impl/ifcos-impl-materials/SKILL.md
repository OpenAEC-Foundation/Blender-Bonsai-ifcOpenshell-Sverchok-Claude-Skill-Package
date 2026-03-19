---
name: ifcos-impl-materials
description: >
  Use when assigning materials to IFC elements -- single materials, layer sets (walls),
  profile sets (beams/columns), or constituent sets (IFC4+). Prevents the common mistake
  of using IfcMaterialConstituentSet in IFC2X3 (not available). Covers IfcMaterial,
  IfcMaterialLayerSet, IfcMaterialProfileSet, material properties, and presentation.
  Keywords: material, IfcMaterial, layer set, profile set, constituent set, material
  assignment, IfcMaterialLayerSet, IfcMaterialProfileSet, material properties.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Material Assignment Implementation Guide

## Quick Reference

### Decision Tree: Which Material Type to Use

```
What kind of element needs material?
├── Single homogeneous material (e.g., steel column, concrete beam)?
│   └── IfcMaterial → assign_material(type="IfcMaterial")
│
├── Layered construction (wall, slab, roof with defined layers)?
│   └── IfcMaterialLayerSet → add_material_set(set_type="IfcMaterialLayerSet")
│       └── Each layer has a thickness (LayerThickness in meters)
│
├── Profiled structural element (beam, column with cross-section)?
│   └── IfcMaterialProfileSet → add_material_set(set_type="IfcMaterialProfileSet")
│       └── Each profile has an IfcProfileDef defining the cross-section
│
├── Composite element with named parts (window: frame+glazing)?
│   ├── IFC4+ → IfcMaterialConstituentSet
│   │   └── add_material_set(set_type="IfcMaterialConstituentSet")
│   └── IFC2X3 → IfcMaterialList (legacy, no named parts)
│       └── add_material_set(set_type="IfcMaterialList")
│
└── Legacy unordered material list (IFC2X3 only)?
    └── IfcMaterialList → add_material_set(set_type="IfcMaterialList")
```

### Decision Tree: Assign Material to Type or Occurrence?

```
Where to assign the material?
├── Element type (IfcWallType, IfcSlabType, etc.)? [RECOMMENDED]
│   └── Assign to type → all occurrences inherit the material
│       └── ifcopenshell.api.material.assign_material(model,
│           products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)
│
└── Individual occurrence (IfcWall, IfcSlab, etc.)?
    └── Assign to occurrence → overrides type material for this element only
        └── ifcopenshell.api.material.assign_material(model,
            products=[wall], material=concrete)
```

### Critical Warnings

- **ALWAYS** assign materials to element types (IfcWallType, IfcSlabType), not individual occurrences. Occurrences inherit from their type. Assign to occurrences only when overriding type material.
- **ALWAYS** use `ifcopenshell.api.material.*` functions for material operations. NEVER create IfcMaterial or IfcRelAssociatesMaterial entities directly with `create_entity()`.
- **NEVER** use `IfcMaterialConstituentSet` with IFC2X3 schema. It does not exist in IFC2X3. Use `IfcMaterialList` instead.
- **ALWAYS** check `model.schema` before using schema-specific material types.
- **NEVER** assign multiple materials directly to the same product. Use material sets (layer, profile, constituent) for composite materials. `assign_material()` automatically unassigns previous materials.
- **ALWAYS** set `LayerThickness` on layers after creation using `edit_layer()`. Layers without thickness are invalid.
- **ALWAYS** provide a profile definition (`IfcProfileDef`) when adding profiles to a profile set. A material profile without a profile curve is incomplete.
- **NEVER** set material relationships via direct attribute assignment (e.g., `wall.HasAssociations = ...`). ALWAYS use the API.

---

## Essential Patterns

### Pattern 1: Create and Assign a Single Material

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

# Create a material with category
concrete = ifcopenshell.api.material.add_material(model,
    name="Concrete C30/37", category="concrete")

# Assign to a wall (single material, no layers)
ifcopenshell.api.material.assign_material(model,
    products=[wall], material=concrete)
```

### Pattern 2: Create a Layered Wall Material (IfcMaterialLayerSet)

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# Step 1: Create individual materials
brick = ifcopenshell.api.material.add_material(model,
    name="Facing Brick", category="brick")
insulation = ifcopenshell.api.material.add_material(model,
    name="Mineral Wool", category="glass")
block = ifcopenshell.api.material.add_material(model,
    name="Concrete Block", category="block")

# Step 2: Create layer set container
layer_set = ifcopenshell.api.material.add_material_set(model,
    name="External Wall 300mm", set_type="IfcMaterialLayerSet")

# Step 3: Add layers (order = outside to inside)
layer1 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=brick)
ifcopenshell.api.material.edit_layer(model,
    layer=layer1, attributes={"LayerThickness": 0.102})

layer2 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.material.edit_layer(model,
    layer=layer2, attributes={"LayerThickness": 0.100})

layer3 = ifcopenshell.api.material.add_layer(model,
    layer_set=layer_set, material=block)
ifcopenshell.api.material.edit_layer(model,
    layer=layer3, attributes={"LayerThickness": 0.100})

# Step 4: Assign to wall TYPE (best practice)
ifcopenshell.api.material.assign_material(model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)
```

### Pattern 3: Create a Profiled Beam Material (IfcMaterialProfileSet)

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# Step 1: Create material
steel = ifcopenshell.api.material.add_material(model,
    name="S355", category="steel")

# Step 2: Create profile set
profile_set = ifcopenshell.api.material.add_material_set(model,
    name="HEA 200", set_type="IfcMaterialProfileSet")

# Step 3: Create profile definition (cross-section shape)
profile_def = ifcopenshell.api.profile.add_parameterised_profile(model,
    ifc_class="IfcIShapeProfileDef",
    OverallWidth=0.200, OverallDepth=0.190,
    WebThickness=0.0065, FlangeThickness=0.010)

# Step 4: Add profile item to set
profile_item = ifcopenshell.api.material.add_profile(model,
    profile_set=profile_set, material=steel, profile=profile_def)

# Step 5: Assign to column TYPE
ifcopenshell.api.material.assign_material(model,
    products=[column_type], type="IfcMaterialProfileSet", material=profile_set)
```

### Pattern 4: Constituent Set for Composite Elements (IFC4+ Only)

```python
# IfcOpenShell — IFC4 and IFC4X3 ONLY (NOT IFC2X3)
import ifcopenshell.api

# Step 1: Create materials
aluminium = ifcopenshell.api.material.add_material(model,
    name="Aluminium Frame", category="aluminium")
glass = ifcopenshell.api.material.add_material(model,
    name="Double Glazing", category="glass")

# Step 2: Create constituent set
constituent_set = ifcopenshell.api.material.add_material_set(model,
    name="Window Assembly", set_type="IfcMaterialConstituentSet")

# Step 3: Add named constituents
frame = ifcopenshell.api.material.add_constituent(model,
    constituent_set=constituent_set, material=aluminium, name="Frame")
glazing = ifcopenshell.api.material.add_constituent(model,
    constituent_set=constituent_set, material=glass, name="Glazing")

# Step 4: Assign to window TYPE
ifcopenshell.api.material.assign_material(model,
    products=[window_type], type="IfcMaterialConstituentSet",
    material=constituent_set)
```

### Pattern 5: Query Material from an Element

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

# Get material (returns IfcMaterial, IfcMaterialLayerSet, etc.)
material = ifcopenshell.util.element.get_material(wall)

if material is None:
    print("No material assigned")
elif material.is_a("IfcMaterial"):
    print(f"Single material: {material.Name}")
elif material.is_a("IfcMaterialLayerSet"):
    for layer in material.MaterialLayers:
        print(f"Layer: {layer.Material.Name}, Thickness: {layer.LayerThickness}")
elif material.is_a("IfcMaterialLayerSetUsage"):
    layer_set = material.ForLayerSet
    for layer in layer_set.MaterialLayers:
        print(f"Layer: {layer.Material.Name}, Thickness: {layer.LayerThickness}")
elif material.is_a("IfcMaterialProfileSet"):
    for profile in material.MaterialProfiles:
        print(f"Profile: {profile.Material.Name}")
elif material.is_a("IfcMaterialConstituentSet"):
    for constituent in material.MaterialConstituents:
        print(f"Constituent: {constituent.Name} - {constituent.Material.Name}")
```

### Pattern 6: Add Visual Style to a Material

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

# Create a surface style (visual appearance)
style = ifcopenshell.api.style.add_style(model, name="Concrete Grey")
ifcopenshell.api.style.add_surface_style(model, style=style,
    ifc_class="IfcSurfaceStyleShading",
    attributes={
        "SurfaceColour": {"Name": None, "Red": 0.7, "Green": 0.7, "Blue": 0.7}
    })

# Assign style to a representation
ifcopenshell.api.style.assign_representation_styles(model,
    shape_representation=body_rep, styles=[style])
```

---

## Common Operations

### Material Categories (Standard Values)

| Category | Use For |
|----------|---------|
| `"concrete"` | Concrete elements, precast |
| `"steel"` | Structural steel, reinforcement |
| `"aluminium"` | Window frames, curtain walls |
| `"brick"` | Masonry walls |
| `"block"` | Concrete blocks |
| `"stone"` | Natural stone elements |
| `"wood"` | Timber construction |
| `"glass"` | Glazing, insulation (glass wool) |
| `"gypsum"` | Plasterboard, gypsum finishes |
| `"plastic"` | PVC pipes, synthetic membranes |
| `"earth"` | Foundation fill, landscaping |

### Material Set Type Selection Guide

| Element Type | Material Set | Schema |
|-------------|-------------|--------|
| Walls (layered) | `IfcMaterialLayerSet` | All |
| Slabs (layered) | `IfcMaterialLayerSet` | All |
| Roofs (layered) | `IfcMaterialLayerSet` | All |
| Beams | `IfcMaterialProfileSet` | All |
| Columns | `IfcMaterialProfileSet` | All |
| Members | `IfcMaterialProfileSet` | All |
| Windows | `IfcMaterialConstituentSet` | IFC4+ |
| Doors | `IfcMaterialConstituentSet` | IFC4+ |
| Curtain walls | `IfcMaterialConstituentSet` | IFC4+ |
| Furniture | `IfcMaterial` (single) | All |
| Equipment | `IfcMaterial` (single) | All |
| Windows (legacy) | `IfcMaterialList` | IFC2X3 |

### Remove Material from Element

```python
# IfcOpenShell — all schema versions
ifcopenshell.api.material.unassign_material(model, products=[wall])
```

### Copy a Material with All Properties

```python
# IfcOpenShell — all schema versions
new_material = ifcopenshell.api.material.copy_material(model,
    material=existing_material)
# Copies psets and styles. Set items are copied but underlying materials reused.
```

### Edit Material Properties

```python
# IfcOpenShell — all schema versions
ifcopenshell.api.material.edit_material(model,
    material=concrete, attributes={"Name": "Concrete C35/45", "Category": "concrete"})
```

### Reorder Layers in a Layer Set

```python
# IfcOpenShell — all schema versions
ifcopenshell.api.material.reorder_set_item(model,
    material_set=layer_set, old_index=2, new_index=0)
```

### Edit Layer Usage (Offset from Reference Line)

```python
# IfcOpenShell — all schema versions
# Get the usage from the element
material = ifcopenshell.util.element.get_material(wall)
if material.is_a("IfcMaterialLayerSetUsage"):
    ifcopenshell.api.material.edit_layer_usage(model,
        usage=material, attributes={"OffsetFromReferenceLine": -0.1})
```

### Delete Material and Material Set

```python
# IfcOpenShell — all schema versions
# First unassign from all products
ifcopenshell.api.material.unassign_material(model, products=[wall_type])

# Then remove the material set
ifcopenshell.api.material.remove_material_set(model, material_set=layer_set)

# Remove individual materials (only if not used elsewhere)
ifcopenshell.api.material.remove_material(model, material=brick)
```

---

## Version Notes

### IFC2X3 vs IFC4+ Material Differences

| Feature | IFC2X3 | IFC4 / IFC4X3 |
|---------|--------|----------------|
| IfcMaterial | Yes | Yes |
| IfcMaterialLayerSet | Yes | Yes |
| IfcMaterialProfileSet | Yes | Yes |
| IfcMaterialConstituentSet | **No** | Yes |
| IfcMaterialList | Yes (primary) | Yes (legacy) |
| Material category attribute | **No** | Yes |
| Material description attribute | **No** | Yes |

### Schema-Aware Material Assignment

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

if model.schema == "IFC2X3":
    # IFC2X3: Use IfcMaterialList for composite elements
    material_list = ifcopenshell.api.material.add_material_set(model,
        name="Window Materials", set_type="IfcMaterialList")
    ifcopenshell.api.material.add_list_item(model,
        material_list=material_list, material=aluminium)
    ifcopenshell.api.material.add_list_item(model,
        material_list=material_list, material=glass)
else:
    # IFC4+: Use IfcMaterialConstituentSet for named parts
    constituent_set = ifcopenshell.api.material.add_material_set(model,
        name="Window Assembly", set_type="IfcMaterialConstituentSet")
    ifcopenshell.api.material.add_constituent(model,
        constituent_set=constituent_set, material=aluminium, name="Frame")
    ifcopenshell.api.material.add_constituent(model,
        constituent_set=constituent_set, material=glass, name="Glazing")
```

---

## Material + Style Integration

Materials define physical properties. Styles define visual appearance. These are separate concepts in IFC. ALWAYS apply both together for visual BIM models.

### Workflow: Material + Visual Style

```
1. Create IfcMaterial → physical definition
2. Create IfcSurfaceStyle → visual appearance (color, texture)
3. Assign style to representation → visual display
4. Assign material to product → physical association
```

Materials and styles are NOT linked automatically. A material named "Concrete" does not automatically render as grey. The style must be explicitly created and assigned to the element's representation.

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all material API methods
- [Working Code Examples](references/examples.md) — End-to-end examples for all material operations
- [Anti-Patterns](references/anti-patterns.md) — Common material assignment mistakes and how to avoid them
