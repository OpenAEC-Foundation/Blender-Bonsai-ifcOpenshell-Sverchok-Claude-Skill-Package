---
name: bonsai-impl-qto
description: "Implements Bonsai quantity takeoff (QTO) workflows including calculating quantities from IFC element geometry, using QtoCalculator for automated base quantity computation, managing IfcElementQuantity sets (area, length, volume, weight), custom quantity set definitions, and bulk quantity operations across building elements. Covers the complete QTO pipeline from geometry analysis to quantity export."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai QTO (Quantity Takeoff) — Implementation Skill

> **Bonsai v0.8.x** | Module: `bonsai.bim.module.qto` | NEVER use `blenderbim.bim.module.qto`

## 1. Quick Reference

### Critical Warnings

- **ALWAYS** verify objects have mesh geometry before running QTO calculators. Empties, curves, and lights cause failures.
- **ALWAYS** ensure objects have an IFC class assigned via Bonsai before running QTO. Calculator functions query IFC properties.
- **ALWAYS** assign `Pset_MaterialCommon.MassDensity` BEFORE running weight calculations (`get_net_weight`, `get_gross_weight`).
- **ALWAYS** re-run QTO calculations after modifying geometry. Quantities are NOT live-linked to geometry.
- **NEVER** assume `get_gross_volume()` accounts for openings. Gross = original geometry WITHOUT openings. Net = WITH openings subtracted.
- **NEVER** use the legacy `blenderbim` module path. ALWAYS use `bonsai.bim.module.qto`.

### Decision Tree: Which QTO Approach?

```
Need quantities for IFC elements?
├── Single element, single quantity?
│   └── Use bim.calculate_single_quantity operator
│       Parameters: calculator, qto_name, prop_name, calculator_function
│
├── Single element, all base quantities?
│   └── Use bim.perform_quantity_take_off with element selected
│       Select object → Set qto_rule → Execute operator
│
├── Multiple elements, standard base quantities?
│   └── Use bim.perform_quantity_take_off in batch mode
│       Select all target objects → Set qto_rule → Execute operator
│
├── All elements of a class?
│   └── Use bim.perform_quantity_take_off with no selection
│       Operator processes all IfcElement instances matching the rule
│
├── Custom quantity (edge lengths, face areas)?
│   └── Use mesh-level operators in Edit Mode:
│       - bim.calculate_edge_lengths (selected edges)
│       - bim.calculate_face_areas (selected faces)
│       - bim.calculate_circle_radius (selected vertices)
│
└── Need gross vs net distinction?
    ├── With openings (doors/windows in walls) → Use net variants
    │   get_net_volume(), get_net_surface_area(), get_net_footprint_area()
    └── Without openings (original geometry) → Use gross variants
        get_gross_volume(), get_gross_surface_area(), get_gross_footprint_area()
```

## 2. Module Architecture

### File Structure

| File | Purpose |
|------|---------|
| `__init__.py` | Module registration |
| `operator.py` | 8 QTO operators |
| `prop.py` | `BIMQtoProperties` with calculator/rule settings |
| `ui.py` | Panel UI for quantity takeoff |
| `data.py` | Data loading and caching |
| `calculator.py` | Geometry calculation engine (30+ functions) |
| `helper.py` | Mesh quantity calculation helpers |

### Two-Layer System

1. **Calculator functions** (`calculator.py`) — Extract numeric values from Blender geometry (`bpy.types.Object`)
2. **QTO rules** (JSON configuration) — Map IFC classes to standard quantity sets and calculator functions

### Core/Tool/Operator Pattern

```
bonsai.core.qto        → Pure logic functions (accept tool interfaces)
bonsai.tool.Qto        → Blender-specific implementations
bonsai.bim.module.qto  → Operators that wire core to tool
```

## 3. Calculator Functions by Category

### Linear Quantities

| Function | Returns | Notes |
|----------|---------|-------|
| `get_linear_length()` | Longest bounding box edge | Simple bbox calculation |
| `get_length()` | Parametric axis length | AXIS2/AXIS3 aware |
| `get_width()` | Width dimension | From bbox or parametric data |
| `get_height()` | Height (Z dimension) | From bounding box |
| `get_depth()` | Depth dimension | For slabs, footings |
| `get_perimeter()` | Net footprint perimeter | Perimeter of lowest polygon |
| `get_gross_perimeter()` | Gross footprint perimeter | Without opening subtractions |

### Area Quantities

| Function | Returns | Notes |
|----------|---------|-------|
| `get_net_footprint_area()` | Lowest polygon face area | Net of openings |
| `get_gross_footprint_area()` | Footprint without openings | Gross geometry |
| `get_roofprint_area()` | Highest polygon face area | For roofs |
| `get_side_area()` | Lateral face areas | Directional filter support |
| `get_lateral_area()` | Side surfaces | Lateral faces only |
| `get_top_area()` | +Z normal faces | Configurable angle threshold (default 45 deg) |
| `get_net_surface_area()` | Total surface (net) | With opening subtractions |
| `get_gross_surface_area()` | Total surface (gross) | Without opening subtractions |
| `get_cross_section_area()` | Area at cut plane | For columns, beams |
| `get_projected_area()` | Projection onto x/y/z | Axis-specific projection |
| `get_opening_area()` | Opening areas | Normal vector angle filtering |
| `get_formwork_area()` | Surface minus top faces | For concrete formwork |
| `get_side_formwork_area()` | Lateral surfaces only | Excludes Z-normal faces |

### Volume Quantities

| Function | Returns | Notes |
|----------|---------|-------|
| `get_net_volume()` | Volume with openings | BMesh-calculated |
| `get_gross_volume()` | Volume without openings | Uses `disable-opening-subtractions` flag |
| `get_space_net_volume()` | Space volume minus walls/columns | Decomposition-aware |

### Weight Quantities

| Function | Returns | Notes |
|----------|---------|-------|
| `get_net_weight()` | net_volume x mass_density | Requires `Pset_MaterialCommon.MassDensity` |
| `get_gross_weight()` | gross_volume x mass_density | Requires `Pset_MaterialCommon.MassDensity` |
| Profile-based weight | `Pset_ProfileMechanical.MassPerLength` x length | For steel profiles |

### Specialized Quantities

| Function | Returns | Notes |
|----------|---------|-------|
| `get_stair_length()` | Hypotenuse of (length^2 + height^2) | For stairs |
| `get_finish_floor_height()` | Floor finish height | Decomposition-aware |
| `get_finish_ceiling_height()` | Ceiling height | Decomposition-aware |
| `get_contact_area()` | Polygon intersection between objects | Requires Shapely |

## 4. QTO Rules (JSON Configuration)

Rules map IFC classes to standard quantity sets and calculator functions. Stored as JSON in `ifc5d.qto.rules`.

### Rule Structure

```json
{
    "Name": "Qto_WallBaseQuantities",
    "Description": "Standard wall base quantities per IFC4",
    "Calculator": "Blender",
    "Mappings": {
        "IfcWall": {
            "Qto_WallBaseQuantities": {
                "Length": "get_length",
                "Width": "get_width",
                "Height": "get_height",
                "GrossFootprintArea": "get_gross_footprint_area",
                "NetFootprintArea": "get_net_footprint_area",
                "GrossSideArea": "get_gross_surface_area",
                "NetSideArea": "get_net_surface_area",
                "GrossVolume": "get_gross_volume",
                "NetVolume": "get_net_volume"
            }
        }
    }
}
```

### Rule Selection

- `BIMQtoProperties.qto_rule` — Enum populated from `tool.Qto.get_qto_rules()`
- Rules are filtered by IFC schema version (IFC4 vs IFC4X3)
- `BIMQtoProperties.fallback=True` — Tries alternative calculators if primary lacks support

## 5. Standard Quantity Sets

| IFC Class | Quantity Set | Key Quantities |
|-----------|-------------|----------------|
| `IfcWall` | `Qto_WallBaseQuantities` | Length, Width, Height, GrossVolume, NetVolume, GrossSideArea, NetSideArea |
| `IfcSlab` | `Qto_SlabBaseQuantities` | Width, Length, Depth, GrossArea, NetArea, GrossVolume, NetVolume, Perimeter |
| `IfcColumn` | `Qto_ColumnBaseQuantities` | Length, CrossSectionArea, OuterSurfaceArea, GrossVolume, NetVolume, GrossWeight |
| `IfcBeam` | `Qto_BeamBaseQuantities` | Length, CrossSectionArea, OuterSurfaceArea, GrossVolume, NetVolume, GrossWeight |
| `IfcDoor` | `Qto_DoorBaseQuantities` | Height, Width, Area |
| `IfcWindow` | `Qto_WindowBaseQuantities` | Height, Width, Area |
| `IfcSpace` | `Qto_SpaceBaseQuantities` | Height, FinishCeilingHeight, FinishFloorHeight, GrossFloorArea, NetFloorArea |
| `IfcRoof` | `Qto_RoofBaseQuantities` | GrossArea, NetArea, ProjectedArea |
| `IfcStair` | `Qto_StairBaseQuantities` | Length, GrossVolume, NetVolume |

## 6. Operators Reference

| Operator | bl_idname | Use Case |
|----------|-----------|----------|
| `CalculateCircleRadius` | `bim.calculate_circle_radius` | Selected vertices in Edit Mode |
| `CalculateEdgeLengths` | `bim.calculate_edge_lengths` | Selected edges in Edit Mode |
| `CalculateFaceAreas` | `bim.calculate_face_areas` | Selected faces in Edit Mode |
| `CalculateObjectVolumes` | `bim.calculate_object_volumes` | Mesh volumes in Object Mode |
| `CalculateFormworkArea` | `bim.calculate_formwork_area` | Surface minus top faces |
| `CalculateSideFormworkArea` | `bim.calculate_side_formwork_area` | Lateral surfaces only |
| `CalculateSingleQuantity` | `bim.calculate_single_quantity` | One quantity via specific calculator function |
| `PerformQuantityTakeOff` | `bim.perform_quantity_take_off` | Batch QTO using rules |

### PerformQuantityTakeOff — Primary Batch Operator

```python
# Bonsai v0.8.x — Batch quantity takeoff
import bpy

bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"  # Rule name from JSON config
)
```

**Behavior**: Processes selected objects. If no objects selected, processes ALL `IfcElement` instances matching the rule's IFC class filter.

### CalculateSingleQuantity — Targeted Calculation

```python
# Bonsai v0.8.x — Single quantity calculation
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="GrossVolume",
    calculator_function="get_gross_volume"
)
```

## 7. Cost Integration

QTO integrates with Bonsai cost management:

```python
# Bonsai v0.8.x — QTO-Cost linkage
# tool.Qto.get_related_cost_item_quantities(product) returns:
# [{"cost_item": <IfcCostItem>, "quantities": [<matched quantity properties>]}]
```

Unit conversion: `tool.Qto.convert_to_project_units()` handles measurement system conversion using IFC schema unit definitions.

## 8. External Dependencies

| Dependency | Used By | Purpose |
|------------|---------|---------|
| `BMesh` | Volume calculations | Mesh volume computation |
| `mathutils.BVHTree` | Proximity detection | Spatial queries |
| `Shapely` | `get_contact_area()` | 2D polygon intersection |
| `ifcopenshell.api.geometry` | Gross calculations | Shape generation with opening control |

## 9. Gross vs. Net — Critical Distinction

```
Gross quantities:
  - Original element geometry WITHOUT openings
  - Wall with door opening → gross volume = solid wall volume
  - Use when: material ordering, structural analysis

Net quantities:
  - Element geometry WITH openings subtracted
  - Wall with door opening → net volume = wall volume minus door void
  - Use when: actual material in place, cost calculation
```

**ALWAYS** choose the correct variant. Using gross when net is required (or vice versa) produces incorrect quantity takeoffs.

## 10. Reference Files

- [references/methods.md](references/methods.md) — Complete API signatures for all calculator functions and operators
- [references/examples.md](references/examples.md) — Working QTO workflow examples
- [references/anti-patterns.md](references/anti-patterns.md) — Common QTO mistakes and correct approaches
