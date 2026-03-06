# QTO API Signatures Reference

> **Bonsai v0.8.x** | Module: `bonsai.bim.module.qto`

## 1. Operators

### PerformQuantityTakeOff

```python
# bl_idname: bim.perform_quantity_take_off
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.perform_quantity_take_off(
    qto_rule: str  # Rule name from JSON config (e.g., "Qto_WallBaseQuantities")
)
```

**Parameters**:
- `qto_rule` (str) — Name of the QTO rule set to apply. Must match a rule defined in `ifc5d.qto.rules` JSON files.

**Behavior**:
- If objects are selected: processes only selected objects
- If no objects selected: processes ALL `IfcElement` instances matching the rule's IFC class filter
- Creates or updates `IfcElementQuantity` property sets on matched elements
- Respects `BIMQtoProperties.fallback` for alternative calculator fallback

**Returns**: `{'FINISHED'}` on success

---

### CalculateSingleQuantity

```python
# bl_idname: bim.calculate_single_quantity
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_single_quantity(
    calculator: str,           # Calculator module name (e.g., "Blender")
    qto_name: str,             # Target quantity set name (e.g., "Qto_WallBaseQuantities")
    prop_name: str,            # Specific property name (e.g., "GrossVolume")
    calculator_function: str   # Function name (e.g., "get_gross_volume")
)
```

**Parameters**:
- `calculator` (str) — Calculator backend identifier. Use `"Blender"` for Bonsai's built-in calculator.
- `qto_name` (str) — IFC quantity set name to target.
- `prop_name` (str) — Individual quantity property name within the set.
- `calculator_function` (str) — Calculator function to invoke.

**Returns**: `{'FINISHED'}` on success

---

### CalculateCircleRadius

```python
# bl_idname: bim.calculate_circle_radius
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_circle_radius()
```

**Context**: Edit Mode with vertices selected.
**Behavior**: Calculates the circle radius defined by selected vertices.
**Returns**: `{'FINISHED'}` on success

---

### CalculateEdgeLengths

```python
# bl_idname: bim.calculate_edge_lengths
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_edge_lengths()
```

**Context**: Edit Mode with edges selected.
**Behavior**: Sums the lengths of all selected edges.
**Returns**: `{'FINISHED'}` on success

---

### CalculateFaceAreas

```python
# bl_idname: bim.calculate_face_areas
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_face_areas()
```

**Context**: Edit Mode with faces selected.
**Behavior**: Sums the areas of all selected faces.
**Returns**: `{'FINISHED'}` on success

---

### CalculateObjectVolumes

```python
# bl_idname: bim.calculate_object_volumes
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_object_volumes()
```

**Context**: Object Mode with mesh objects selected.
**Behavior**: Calculates BMesh-based volume for each selected mesh object.
**Returns**: `{'FINISHED'}` on success

---

### CalculateFormworkArea

```python
# bl_idname: bim.calculate_formwork_area
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_formwork_area()
```

**Context**: Object Mode with mesh objects selected.
**Behavior**: Calculates total surface area excluding top-facing faces. Used for concrete formwork estimation.
**Returns**: `{'FINISHED'}` on success

---

### CalculateSideFormworkArea

```python
# bl_idname: bim.calculate_side_formwork_area
# Location: bonsai.bim.module.qto.operator
bpy.ops.bim.calculate_side_formwork_area()
```

**Context**: Object Mode with mesh objects selected.
**Behavior**: Calculates lateral surface area only, excluding faces with Z-direction normals. Used for side-only formwork estimation.
**Returns**: `{'FINISHED'}` on success

---

## 2. Calculator Functions

All functions are in `bonsai.bim.module.qto.calculator`. They operate on `bpy.types.Object` instances.

### Linear Quantity Functions

```python
def get_linear_length(o: bpy.types.Object) -> float:
    """Returns the longest bounding box edge dimension.
    Uses Blender bounding box coordinates.
    Returns: length in project units (meters)."""

def get_length(o: bpy.types.Object) -> float:
    """Returns parametric axis length.
    AXIS2/AXIS3 aware — checks IFC representation axis.
    Falls back to bounding box if no parametric data.
    Returns: length in project units (meters)."""

def get_width(o: bpy.types.Object) -> float:
    """Returns width from bounding box or parametric data.
    Returns: width in project units (meters)."""

def get_height(o: bpy.types.Object) -> float:
    """Returns height from bounding box Z dimension.
    Returns: height in project units (meters)."""

def get_depth(o: bpy.types.Object) -> float:
    """Returns depth dimension.
    Used for slabs, footings.
    Returns: depth in project units (meters)."""

def get_perimeter(o: bpy.types.Object) -> float:
    """Returns perimeter of net footprint polygon.
    Net = with opening subtractions.
    Returns: perimeter in project units (meters)."""

def get_gross_perimeter(o: bpy.types.Object) -> float:
    """Returns perimeter of gross footprint polygon.
    Gross = without opening subtractions.
    Returns: perimeter in project units (meters)."""
```

### Area Quantity Functions

```python
def get_net_footprint_area(o: bpy.types.Object) -> float:
    """Returns area of lowest polygon faces (net of openings).
    Returns: area in project units (square meters)."""

def get_gross_footprint_area(o: bpy.types.Object) -> float:
    """Returns footprint area without opening subtractions.
    Returns: area in project units (square meters)."""

def get_roofprint_area(o: bpy.types.Object) -> float:
    """Returns area of highest polygon faces.
    Used for roof elements.
    Returns: area in project units (square meters)."""

def get_side_area(o: bpy.types.Object) -> float:
    """Returns lateral face areas with directional filter.
    Returns: area in project units (square meters)."""

def get_lateral_area(o: bpy.types.Object) -> float:
    """Returns side surface areas.
    Returns: area in project units (square meters)."""

def get_top_area(o: bpy.types.Object) -> float:
    """Returns area of faces with +Z normal.
    Configurable angle threshold (default 45 degrees).
    Returns: area in project units (square meters)."""

def get_net_surface_area(o: bpy.types.Object) -> float:
    """Returns total surface area with opening subtractions.
    Returns: area in project units (square meters)."""

def get_gross_surface_area(o: bpy.types.Object) -> float:
    """Returns total surface area without opening subtractions.
    Returns: area in project units (square meters)."""

def get_cross_section_area(o: bpy.types.Object) -> float:
    """Returns area at cut plane.
    Used for columns and beams.
    Returns: area in project units (square meters)."""

def get_projected_area(o: bpy.types.Object) -> float:
    """Returns projection area onto x/y/z axes.
    Returns: area in project units (square meters)."""

def get_opening_area(o: bpy.types.Object) -> float:
    """Returns area of openings.
    Uses normal vector angle filtering.
    Returns: area in project units (square meters)."""

def get_formwork_area(o: bpy.types.Object) -> float:
    """Returns surface area excluding top faces.
    For concrete formwork estimation.
    Returns: area in project units (square meters)."""

def get_side_formwork_area(o: bpy.types.Object) -> float:
    """Returns lateral surfaces only (excluding Z-normal faces).
    Returns: area in project units (square meters)."""
```

### Volume Quantity Functions

```python
def get_net_volume(o: bpy.types.Object) -> float:
    """Returns BMesh-calculated volume with openings.
    Net = opening voids are subtracted.
    Returns: volume in project units (cubic meters)."""

def get_gross_volume(o: bpy.types.Object) -> float:
    """Returns volume without opening subtractions.
    Uses 'disable-opening-subtractions' flag in
    ifcopenshell.api.geometry.create_shape_settings.
    Returns: volume in project units (cubic meters)."""

def get_space_net_volume(o: bpy.types.Object) -> float:
    """Returns space volume minus walls/columns.
    Decomposition-aware: accounts for building elements within the space.
    Returns: volume in project units (cubic meters)."""
```

### Weight Quantity Functions

```python
def get_net_weight(o: bpy.types.Object) -> float:
    """Returns net_volume * mass_density.
    REQUIRES: Pset_MaterialCommon.MassDensity assigned to element material.
    Returns: weight in project units (kilograms)."""

def get_gross_weight(o: bpy.types.Object) -> float:
    """Returns gross_volume * mass_density.
    REQUIRES: Pset_MaterialCommon.MassDensity assigned to element material.
    Returns: weight in project units (kilograms)."""

# Profile-based weight (for steel profiles):
# Uses Pset_ProfileMechanical.MassPerLength * length
# Applies to elements with profile-based representations.
```

### Specialized Quantity Functions

```python
def get_stair_length(o: bpy.types.Object) -> float:
    """Returns hypotenuse of (length^2 + height^2).
    For stair elements.
    Returns: length in project units (meters)."""

def get_finish_floor_height(o: bpy.types.Object) -> float:
    """Returns floor finish height.
    Decomposition-aware: considers space containment.
    Returns: height in project units (meters)."""

def get_finish_ceiling_height(o: bpy.types.Object) -> float:
    """Returns ceiling height.
    Decomposition-aware: considers space containment.
    Returns: height in project units (meters)."""

def get_contact_area(o: bpy.types.Object) -> float:
    """Returns polygon intersection area between objects.
    REQUIRES: Shapely library for 2D polygon intersection.
    Returns: area in project units (square meters)."""
```

## 3. Tool Functions

### tool.Qto

```python
# bonsai.tool.Qto

def get_qto_rules() -> list[tuple[str, str, str]]:
    """Returns available QTO rules as Blender enum items.
    Filters by current IFC schema version (IFC4 vs IFC4X3).
    Returns: list of (identifier, name, description) tuples."""

def get_related_cost_item_quantities(product: ifcopenshell.entity_instance) -> list[dict]:
    """Returns cost items linked to the product's quantities.
    Returns: [{"cost_item": <IfcCostItem>, "quantities": [<matched props>]}]"""

def convert_to_project_units(value: float, unit_type: str) -> float:
    """Converts a value to project units using IFC schema unit definitions.
    Parameters:
        value: numeric value to convert
        unit_type: IFC unit type identifier
    Returns: converted value in project units."""
```

## 4. Properties

### BIMQtoProperties

```python
# bonsai.bim.module.qto.prop.BIMQtoProperties
# Registered on bpy.types.Scene as scene.BIMQtoProperties

class BIMQtoProperties:
    qto_rule: bpy.props.EnumProperty
    # Populated from tool.Qto.get_qto_rules()
    # Filtered by current IFC schema version

    fallback: bpy.props.BoolProperty
    # Default: False
    # When True: tries alternative calculators if primary lacks support
```

## 5. QTO Rule JSON Schema

```json
{
    "Name": "string — Rule set name (e.g., 'Qto_WallBaseQuantities')",
    "Description": "string — Human-readable description",
    "Calculator": "string — Calculator backend ('Blender')",
    "Mappings": {
        "<IfcClassName>": {
            "<Qto_SetName>": {
                "<QuantityName>": "<calculator_function_name>"
            }
        }
    }
}
```

**Rule location**: `ifc5d.qto.rules` module (JSON files).
**Rule selection**: `BIMQtoProperties.qto_rule` enum, filtered by IFC schema version.
