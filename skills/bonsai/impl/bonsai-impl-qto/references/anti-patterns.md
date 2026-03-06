# QTO Anti-Patterns

> **Bonsai v0.8.x** | Common mistakes when implementing quantity takeoff workflows

---

## Anti-Pattern 1: Running QTO on Non-Mesh Objects

### Wrong

```python
# WRONG — Empties, curves, and lights have no mesh data
import bpy

empty_obj = bpy.data.objects.new("MyEmpty", None)
bpy.context.collection.objects.link(empty_obj)
empty_obj.select_set(True)
bpy.context.view_layer.objects.active = empty_obj

bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"
)
# FAILS: Calculator functions require bpy.types.Mesh data
```

### Correct

```python
# CORRECT — ALWAYS verify objects have mesh geometry before QTO
import bpy

obj = bpy.context.active_object
if obj and obj.type == 'MESH' and obj.data:
    bpy.ops.bim.perform_quantity_take_off(
        qto_rule="Qto_WallBaseQuantities"
    )
```

### Why

Calculator functions operate on `bpy.types.Object` instances and access mesh vertices, edges, and faces via BMesh. Objects without mesh data (empties, cameras, lights, curves) cause attribute errors or return zero values.

---

## Anti-Pattern 2: Using Gross Volume When Openings Matter

### Wrong

```python
# WRONG — Using gross volume for a wall with windows and doors
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="GrossVolume",             # Includes void space where door is
    calculator_function="get_gross_volume"
)
# Overestimates material by ignoring door/window openings
```

### Correct

```python
# CORRECT — Use net volume when element has openings
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="NetVolume",               # Subtracts opening voids
    calculator_function="get_net_volume"
)
```

### Why

- **Gross** = original element geometry WITHOUT opening subtractions
- **Net** = element geometry WITH opening voids subtracted

Using gross volume for material cost calculation on a wall with doors/windows overestimates material quantity. ALWAYS use net variants when the element has openings and you need actual material quantities.

---

## Anti-Pattern 3: Weight Calculation Without Material Density

### Wrong

```python
# WRONG — Running weight calculation without setting MassDensity first
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_ColumnBaseQuantities",
    prop_name="GrossWeight",
    calculator_function="get_gross_weight"
)
# Returns 0 or fails: no MassDensity defined
```

### Correct

```python
# CORRECT — ALWAYS assign MassDensity BEFORE weight calculation
import ifcopenshell
import ifcopenshell.api

ifc_file = tool.Ifc.get()
element = tool.Ifc.get_entity(bpy.context.active_object)
material = ifcopenshell.util.element.get_material(element)

if material:
    psets = ifcopenshell.util.element.get_psets(material)
    pset = psets.get("Pset_MaterialCommon")
    if pset:
        ifcopenshell.api.run("pset.edit_pset", ifc_file,
            pset=ifc_file.by_id(pset["id"]),
            properties={"MassDensity": 2400.0}  # kg/m3 for concrete
        )
    else:
        pset = ifcopenshell.api.run("pset.add_pset", ifc_file,
            product=material,
            name="Pset_MaterialCommon"
        )
        ifcopenshell.api.run("pset.edit_pset", ifc_file,
            pset=pset,
            properties={"MassDensity": 2400.0}
        )

# NOW weight calculation works
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_ColumnBaseQuantities",
    prop_name="GrossWeight",
    calculator_function="get_gross_weight"
)
```

### Why

Weight functions compute `volume * mass_density`. The mass density is read from `Pset_MaterialCommon.MassDensity` on the element's material. If this property is not set, the calculation returns zero or fails silently.

---

## Anti-Pattern 4: Assuming QTO Results Auto-Update

### Wrong

```python
# WRONG — Modifying geometry and assuming quantities update automatically
import bpy

# Calculate initial quantities
bpy.ops.bim.perform_quantity_take_off(qto_rule="Qto_WallBaseQuantities")

# Modify wall geometry (extend height)
wall = bpy.context.active_object
wall.dimensions.z = 4.0  # Changed from 3.0 to 4.0

# Read quantities — STILL shows old values!
# Quantities are NOT live-linked to geometry
```

### Correct

```python
# CORRECT — ALWAYS re-run QTO after geometry changes
import bpy

# Calculate initial quantities
bpy.ops.bim.perform_quantity_take_off(qto_rule="Qto_WallBaseQuantities")

# Modify wall geometry
wall = bpy.context.active_object
wall.dimensions.z = 4.0

# Re-run QTO to update quantities
bpy.ops.bim.perform_quantity_take_off(qto_rule="Qto_WallBaseQuantities")
```

### Why

IFC quantities (`IfcElementQuantity`) are stored as static values in the IFC model. They are NOT dynamically linked to Blender mesh geometry. After ANY geometry modification, you MUST re-run the QTO calculation to update the stored quantities.

---

## Anti-Pattern 5: QTO on Objects Without IFC Class Assignment

### Wrong

```python
# WRONG — Running QTO on a plain Blender mesh without IFC class
import bpy

# Create a plain cube (no IFC class)
bpy.ops.mesh.primitive_cube_add()
cube = bpy.context.active_object

cube.select_set(True)
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"
)
# Fails or produces no result: object has no IFC entity
```

### Correct

```python
# CORRECT — Object MUST have an IFC class assigned via Bonsai
import bpy

obj = bpy.context.active_object
element = tool.Ifc.get_entity(obj)

if element and element.is_a("IfcWall"):
    bpy.ops.bim.perform_quantity_take_off(
        qto_rule="Qto_WallBaseQuantities"
    )
else:
    # Assign IFC class first, or skip this object
    pass
```

### Why

Calculator functions query IFC properties (material, representations, decomposition relationships) through `tool.Ifc`. A plain Blender object without an IFC entity association has no `ifc_definition_id` and cannot participate in IFC quantity takeoff.

---

## Anti-Pattern 6: Using Legacy blenderbim Module Path

### Wrong

```python
# WRONG — Legacy module path (pre-rename)
from blenderbim.bim.module.qto import calculator
result = calculator.get_net_volume(obj)
```

### Correct

```python
# CORRECT — Current Bonsai module path (v0.8.x)
from bonsai.bim.module.qto import calculator
result = calculator.get_net_volume(obj)
```

### Why

Bonsai was renamed from BlenderBIM. The `blenderbim` module path is deprecated and will not resolve in current Bonsai versions (v0.8.x+). ALWAYS use the `bonsai` module path.

---

## Anti-Pattern 7: Mixing Up Quantity Set Names and IFC Classes

### Wrong

```python
# WRONG — Using wall quantity set for a column
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",  # Wall set on a column!
    prop_name="Width",
    calculator_function="get_width"
)
# Creates incorrect quantity set association on the column entity
```

### Correct

```python
# CORRECT — Use the matching quantity set for the element class
element = tool.Ifc.get_entity(bpy.context.active_object)

if element.is_a("IfcColumn"):
    bpy.ops.bim.calculate_single_quantity(
        calculator="Blender",
        qto_name="Qto_ColumnBaseQuantities",  # Correct set for columns
        prop_name="CrossSectionArea",
        calculator_function="get_cross_section_area"
    )
elif element.is_a("IfcWall"):
    bpy.ops.bim.calculate_single_quantity(
        calculator="Blender",
        qto_name="Qto_WallBaseQuantities",  # Correct set for walls
        prop_name="Width",
        calculator_function="get_width"
    )
```

### Why

IFC defines specific quantity sets for each element class (`Qto_WallBaseQuantities` for `IfcWall`, `Qto_ColumnBaseQuantities` for `IfcColumn`, etc.). Assigning a wall quantity set to a column entity creates non-standard IFC data that violates schema conventions and may cause issues in downstream applications (cost estimation, model checking).

---

## Summary Table

| # | Anti-Pattern | Rule |
|---|-------------|------|
| 1 | QTO on non-mesh objects | ALWAYS verify `obj.type == 'MESH'` before running calculators |
| 2 | Gross volume when openings matter | ALWAYS use net variants for elements with openings |
| 3 | Weight without MassDensity | ALWAYS set `Pset_MaterialCommon.MassDensity` before weight calculations |
| 4 | Assuming auto-update | ALWAYS re-run QTO after geometry changes |
| 5 | QTO without IFC class | ALWAYS verify IFC class assignment before QTO |
| 6 | Legacy blenderbim path | ALWAYS use `bonsai.bim.module.qto`, NEVER `blenderbim` |
| 7 | Wrong quantity set for class | ALWAYS match quantity set name to element IFC class |
