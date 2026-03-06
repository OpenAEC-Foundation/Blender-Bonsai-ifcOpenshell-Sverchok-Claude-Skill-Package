# Bonsai BIM — Supplementary Research: Gap Domains

> **Target version**: Bonsai v0.8.4 | Minimum Blender: 4.2.0 | Python 3.11
> **Date**: 2026-03-05
> **Status**: COMPLETE
> **Scope**: Gap domains NOT covered in vooronderzoek-bonsai.md
> **Module path**: `bonsai.bim.module.*` — NEVER `blenderbim.bim.module.*`

---

## Table of Contents

1. [Drawing Module](#1-drawing-module)
   - 1.1 [Overview and Architecture](#11-overview-and-architecture)
   - 1.2 [Drawing Types and Views](#12-drawing-types-and-views)
   - 1.3 [Creating Drawings](#13-creating-drawings)
   - 1.4 [SVG Generation Pipeline](#14-svg-generation-pipeline)
   - 1.5 [Annotations](#15-annotations)
   - 1.6 [Sheets and Documents](#16-sheets-and-documents)
   - 1.7 [Camera and View Configuration](#17-camera-and-view-configuration)
   - 1.8 [Drawing Operators Reference](#18-drawing-operators-reference)
   - 1.9 [Drawing Anti-Patterns](#19-drawing-anti-patterns)
2. [Quantity Takeoff (QTO) Module](#2-quantity-takeoff-qto-module)
   - 2.1 [Overview and Architecture](#21-overview-and-architecture)
   - 2.2 [Calculator System](#22-calculator-system)
   - 2.3 [Quantity Rules and JSON Configuration](#23-quantity-rules-and-json-configuration)
   - 2.4 [Standard Quantity Sets](#24-standard-quantity-sets)
   - 2.5 [QTO Operators Reference](#25-qto-operators-reference)
   - 2.6 [Cost Integration](#26-cost-integration)
   - 2.7 [QTO Anti-Patterns](#27-qto-anti-patterns)
3. [BCF Module — BIM Collaboration Format](#3-bcf-module--bim-collaboration-format)
   - 3.1 [Overview and Architecture](#31-overview-and-architecture)
   - 3.2 [BCF Project Lifecycle](#32-bcf-project-lifecycle)
   - 3.3 [Topics](#33-topics)
   - 3.4 [Viewpoints](#34-viewpoints)
   - 3.5 [Comments and Issue Tracking](#35-comments-and-issue-tracking)
   - 3.6 [BCF File Management](#36-bcf-file-management)
   - 3.7 [BCF Operators Reference](#37-bcf-operators-reference)
   - 3.8 [BCF Anti-Patterns](#38-bcf-anti-patterns)
4. [Clash Detection Module](#4-clash-detection-module)
   - 4.1 [Overview and Architecture](#41-overview-and-architecture)
   - 4.2 [Clash Sets](#42-clash-sets)
   - 4.3 [Running Clash Detection](#43-running-clash-detection)
   - 4.4 [Result Interpretation and Visualization](#44-result-interpretation-and-visualization)
   - 4.5 [Smart Grouping](#45-smart-grouping)
   - 4.6 [BCF Integration](#46-bcf-integration)
   - 4.7 [Clash Operators Reference](#47-clash-operators-reference)
   - 4.8 [Clash Anti-Patterns](#48-clash-anti-patterns)
5. [Python Runtime Quirks](#5-python-runtime-quirks)
   - 5.1 [Tool/Core/UI Pattern for Scripts](#51-toolcoreui-pattern-for-scripts)
   - 5.2 [IfcStore Singleton — Deep Dive](#52-ifcstore-singleton--deep-dive)
   - 5.3 [Operators vs ifcopenshell.api Directly](#53-operators-vs-ifcopenshellapi-directly)
   - 5.4 [Operator Context Requirements](#54-operator-context-requirements)
   - 5.5 [BlenderBIM to Bonsai Migration](#55-blenderbim-to-bonsai-migration)
   - 5.6 [Bonsai Property Access](#56-bonsai-property-access)
   - 5.7 [Data Synchronization: Blender Objects and IFC Entities](#57-data-synchronization-blender-objects-and-ifc-entities)
   - 5.8 [Runtime Anti-Patterns](#58-runtime-anti-patterns)
6. [Sources](#sources)

---

## 1. Drawing Module

### 1.1 Overview and Architecture

The drawing module (`bonsai.bim.module.drawing`) provides 2D drawing generation from 3D BIM models. It produces SVG-based architectural drawings (plans, sections, elevations, details) directly from IFC data stored in the Bonsai model.

**Module structure** (16 files at `src/bonsai/bonsai/bim/module/drawing/`):

| File | Purpose |
|------|---------|
| `__init__.py` | Module registration |
| `operator.py` | All drawing operators (40+ operators) |
| `prop.py` | Property definitions (camera, annotation, sheet, text) |
| `ui.py` | Panel UI for Drawings & Documents |
| `annotation.py` | Annotation geometry creation (Annotator class) |
| `data.py` | Data loading and caching for drawings, sheets, schedules |
| `decoration.py` | Viewport decorations for annotation display |
| `gizmos.py` | Interactive gizmos for annotation editing |
| `handler.py` | Event handlers for drawing state management |
| `helper.py` | Utilities: format_distance, coordinate math, BoundingBox |
| `scheduler.py` | Schedule/table generation |
| `shaders.py` | GPU shaders for viewport rendering |
| `sheeter.py` | Sheet composition (SheetBuilder class) |
| `svgwriter.py` | SVG generation engine (SvgWriter class) |
| `workspace.py` | AnnotationTool workspace with keymaps |

The module follows Bonsai's core/tool/operator pattern:
- **Core** (`bonsai.core.drawing`): Pure logic functions that accept tool interfaces
- **Tool** (`bonsai.tool.drawing`): Blender-specific implementations
- **Operator** (`bonsai.bim.module.drawing.operator`): Blender operators that wire core to tool

> **Bonsai v0.8.0+**: The drawing module is marked as "still in early development" in official documentation. SVG generation relies on InkScape for final rendering and sheet composition.

### 1.2 Drawing Types and Views

Bonsai supports five `target_view` types, defined as enum values in `BIMCameraProperties`:

| target_view | Description | Camera Orientation |
|-------------|-------------|--------------------|
| `PLAN_VIEW` | Floor plan (top-down view) | Looking down (-Z) |
| `SECTION_VIEW` | Vertical section (cut through building) | Horizontal cut plane |
| `ELEVATION_VIEW` | External elevation (facade view) | Horizontal, facing building |
| `REFLECTED_PLAN_VIEW` | Reflected ceiling plan | Looking up (+Z) |
| `MODEL_VIEW` | 3D perspective or axonometric view | Any orientation |

Each drawing is backed by an IFC `IfcAnnotation` entity in a group (`IfcGroup`), with properties stored in the `EPset_Drawing` property set:

```python
# Bonsai v0.8.x — Drawing target view is stored in EPset_Drawing
# The property set contains:
#   TargetView  — one of the five view types above
#   Scale       — diagram scale (e.g., "1:100")
#   HumanScale  — display string for the scale
#   HasUnderlay — whether to render underlay image
#   HasLinework — whether to render linework
#   HasAnnotation — whether to render annotations
```

### 1.3 Creating Drawings

**Core function signature** (from `bonsai/core/drawing.py`):

```python
# Bonsai v0.8.x — core drawing creation
def add_drawing(
    ifc: type[tool.Ifc],
    collector: type[tool.Collector],
    drawing: type[tool.Drawing],
    target_view: ifcopenshell.util.representation.TARGET_VIEW,
    location_hint: Union[tool.Drawing.LocationHintLiteral, int],
) -> None:
    """Creates a new drawing with camera, annotation group, and document reference.

    Steps performed:
    1. Generate unique drawing name based on target_view
    2. Compute camera matrix from target_view and location_hint
    3. Create IfcAnnotation via ifcopenshell.api.run("root.create_entity")
    4. Create IfcGroup for annotations
    5. Create IfcDocumentReference pointing to SVG output
    6. Set EPset_Drawing properties (TargetView, Scale, etc.)
    7. Create Blender camera object and assign to collection
    """
```

**Operator usage from Python**:

```python
# Bonsai v0.8.x — Add a plan view drawing via operator
import bpy

bpy.ops.bim.add_drawing(
    target_view="PLAN_VIEW",
    location_hint=0  # 0 = auto-detect from active storey
)
```

**Location hints**: For plan views, the `location_hint` maps to a building storey. For sections/elevations, it represents directional orientation (North, South, East, West).

### 1.4 SVG Generation Pipeline

The `CreateDrawing` operator (`bim.create_drawing`) generates SVG output through a multi-stage pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│                    CreateDrawing Pipeline                     │
├──────────────────────────────────────────────────────────────┤
│ 1. Initialize camera and drawing metadata                    │
│ 2. Sync IFC file (optional, via sync parameter)              │
│ 3. Generate UNDERLAY (raster render image)                   │
│    └─ Blender render → PNG → embedded in SVG                 │
│ 4. Generate LINEWORK (vector line drawing)                   │
│    ├─ Mode: OPENCASCADE → HLR via OCC library               │
│    └─ Mode: FREESTYLE → Blender Freestyle SVG Exporter       │
│ 5. Generate ANNOTATIONS (via SvgWriter)                      │
│    └─ Only for orthographic views                            │
│ 6. Combine all SVG layers into final output                  │
└──────────────────────────────────────────────────────────────┘
```

**SvgWriter** (`svgwriter.py`) is the central SVG engine:

```python
# Bonsai v0.8.x — SvgWriter coordinate pipeline
# 1. 3D world coordinates → camera-space projection
#    project_point_onto_camera() uses plane intersection
# 2. Camera-space → SVG canvas coordinates
#    Offset to canvas center, scale by (diagram_scale × 1000)
# 3. Unit conversion: IFC meters → SVG millimeters
```

**Linework modes** (configured in `BIMCameraProperties.linework_mode`):

| Mode | Engine | Quality | Speed | Requirements |
|------|--------|---------|-------|-------------|
| `OPENCASCADE` | OpenCASCADE HLR | Precise hidden-line removal | Slower | `ifcopenshell` with OCC |
| `FREESTYLE` | Blender Freestyle | Artistic line styles | Faster | Freestyle SVG Exporter add-on |

**Fill modes** (`BIMCameraProperties.fill_mode`):

| Mode | Description |
|------|-------------|
| `NONE` | No fills |
| `SHAPELY` | Polygon fills via Shapely library |
| `SVGFILL` | SVG-native fill patterns |

**Cut modes** (`BIMCameraProperties.cut_mode`):

| Mode | Description |
|------|-------------|
| `BISECT` | Blender mesh bisect for cut planes |
| `OPENCASCADE` | OCC-based section cutting |

### 1.5 Annotations

Annotations are IFC entities (`IfcAnnotation`) added to drawing groups. The `Annotator` class (`annotation.py`) creates Blender geometry for annotations.

**Supported annotation data types**:

| Data Type | Blender Type | Usage |
|-----------|-------------|-------|
| Mesh | `bpy.types.Mesh` | Vertex-based annotations (points, lines) |
| Curve | `bpy.types.Curve` | Leader lines, polylines (`resolution_u=2`, 3D curves) |
| Empty | `bpy.types.Object(EMPTY)` | Lightweight reference markers |

**Annotation object types** (selected via `BIMAnnotationProperties.object_type`):

- **TEXT** — Text labels with font size, alignment, and markdown support
- **TEXT_LEADER** — Text with a leader line (curve-based)
- **DIMENSION** — Linear dimensions with start/end points
- **ANGLE** — Angular dimension annotations
- **SECTION_LEVEL** — Section cut markers
- **PLAN_LEVEL** — Level indicators on plans
- **ELEVATION** — Elevation markers (predefined SVG symbols)
- **GRID** — Grid line references
- **FILL_AREA** — Hatching and fill patterns
- **STAIR_ARROW** — Stair direction indicators
- **HIDDEN_LINE** — Hidden/dashed line annotations
- **BREAKLINE** — Break line symbols

**Annotation creation from Python**:

```python
# Bonsai v0.8.x — Add a text annotation to the active drawing
import bpy

# MUST have an active drawing (camera view)
bpy.ops.bim.add_annotation(object_type="TEXT")

# For annotations with a type:
bpy.ops.bim.add_annotation(
    object_type="DIMENSION",
    relating_type_id=42  # IFC ID of the annotation type
)
```

**Text annotation properties** (`BIMTextProperties`):

```python
# Bonsai v0.8.x — Text properties structure
# Font sizes (mm): 1.8, 2.5, 3.5, 5.0, 7.0
# Horizontal alignment: left, middle, right
# Vertical alignment: top, middle, bottom
# Symbol options: NO SYMBOL, CUSTOM SYMBOL, or predefined symbols
# Literals support 16 categories:
#   Basic, Attributes, Property Sets, Material, Task, Cost, etc.
# Markdown support in SVG: bold, italic, links, bullet points
```

**Annotation workspace** (`workspace.py`): The `AnnotationTool` provides a dedicated workspace with keymaps:

| Shortcut | Action |
|----------|--------|
| `Shift+A` | Add new annotation |
| `Shift+T` | Bulk tag (auto-adjust annotations to selected objects) |
| `Shift+G` | Readjust tags based on assigned products |
| `Shift+E` | Edit text annotations |

### 1.6 Sheets and Documents

Sheets are IFC `IfcDocumentInformation` entities that compose multiple drawings into a single page layout.

**Sheet creation workflow**:

```python
# Bonsai v0.8.x — Sheet creation pipeline
# 1. Create sheet with titleblock
bpy.ops.bim.add_sheet()
# Creates IfcDocumentInformation + IfcDocumentReference
# Generates base SVG layout from titleblock template

# 2. Add drawings to sheet
bpy.ops.bim.add_drawing_to_sheet()
# Links a drawing (IfcDocumentReference) to the sheet

# 3. Build final sheet (generates PDF/DXF)
bpy.ops.bim.create_sheets()
# Composes all drawings into sheet SVG, optionally exports to PDF/DXF
```

**SheetBuilder** (`sheeter.py`) handles SVG composition:

```python
# Bonsai v0.8.x — SheetBuilder composition process
# build() performs three phases:
# 1. Titleblock Processing
#    - Embeds titleblock SVG
#    - Applies geolocation data (grid north, true north rotations)
# 2. Drawing Integration
#    - Embeds drawing SVGs with style isolation
#    - ensure_drawing_unique_styles() prefixes all CSS class names
#      with drawing identifier to prevent conflicts
#    - Applies clipping paths via UUID-based identifiers
# 3. Document Assembly
#    - Incorporates schedules and reference documents
#    - Resolves mustache-syntax template variables
```

**Drawing placement**: `next_drawing_location()` implements automatic layout:
- Default start position: (30, 30) mm
- Padding: 10 mm between drawings
- Horizontal placement within rows, new rows when width exceeds titleblock width
- Default titleblock width: 840 mm

**Unit handling**: `convert_to_mm()` normalizes CSS length units (cm, in, pt, px, Q, pc) to millimeters for positioning.

### 1.7 Camera and View Configuration

Drawing cameras are configured via `BIMCameraProperties`:

```python
# Bonsai v0.8.x — Camera/drawing configuration properties

# Scale options (metric): 1:5000, 1:2000, 1:1000, 1:500, 1:200,
#   1:100, 1:50, 1:20, 1:10, 1:5, 1:2, 1:1, CUSTOM
# Scale options (imperial): similar range with ft/in notation
# Custom scale: custom_scale_numerator / custom_scale_denominator

# Camera type: ORTHO (orthographic) | PERSP (perspective)
# Raster resolution: raster_x, raster_y (default 1000 px each)
# DPI: default 75
# View dimensions: width, height (default 50 m each)
# NTS flag: is_nts (Not To Scale)

# Layer toggles:
#   has_underlay  — render-based background image
#   has_linework  — vector line drawing
#   has_annotation — annotation overlays

# Filter mode: NONE | include/exclude filter groups
# Filter groups: CollectionProperty for element filtering
```

**Drawing style** (`DrawingStyle` PropertyGroup):
- `render_type`: `DEFAULT` (Blender render settings) or `VIEWPORT` (viewport shading)
- `raster_style`: JSON-formatted render configuration string

### 1.8 Drawing Operators Reference

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| AddDrawing | `bim.add_drawing` | Create a new drawing view with camera |
| CreateDrawing | `bim.create_drawing` | Generate/refresh SVG from active camera |
| DuplicateDrawing | `bim.duplicate_drawing` | Copy drawing with optional annotation duplication |
| AddAnnotation | `bim.add_annotation` | Add annotation to active drawing |
| AddAnnotationType | `bim.add_annotation_type` | Create annotation type template |
| AddSheet | `bim.add_sheet` | Create a new sheet document |
| CreateSheets | `bim.create_sheets` | Build sheets with PDF/DXF export |
| AddDrawingToSheet | `bim.add_drawing_to_sheet` | Link drawing to active sheet |
| OpenSheet | `bim.open_sheet` | Open sheet with system viewer |
| OpenDrawing | `bim.open_drawing` | Open SVG with viewer |
| OpenLayout | `bim.open_layout` | Open SVG layout file |
| ActivateModel | `bim.activate_model` | Switch to 3D model view (show all, hide annotations) |
| SelectAllDrawings | `bim.select_all_drawings` | Batch select drawings (Shift toggles) |
| SelectAllSheets | `bim.select_all_sheets` | Batch select sheets |

**CreateDrawing parameters**:
- `print_all` (bool): Generate all checked drawings
- `open_viewer` (bool): Launch viewer after generation
- `sync` (bool): Pre-sync IFC data before drawing

### 1.9 Drawing Anti-Patterns

| Anti-Pattern | Correct Approach |
|-------------|-----------------|
| Calling `bim.create_drawing` without an active orthographic camera | ALWAYS activate a drawing view first via `bim.activate_model` then select the drawing camera |
| Using FREESTYLE mode without the Freestyle SVG Exporter add-on enabled | ALWAYS verify the Freestyle SVG Exporter add-on is active in Blender preferences |
| Manually editing generated SVG files | NEVER edit SVGs directly; ALWAYS modify annotations in Blender and regenerate |
| Assuming annotations work in perspective views | Annotations are ONLY generated for orthographic camera views |
| Not syncing IFC before drawing generation | ALWAYS use `sync=True` or save the IFC file before calling `bim.create_drawing` |
| Creating sheets without InkScape installed | Sheet composition and PDF export require InkScape; ALWAYS install it as a dependency |
| Using absolute paths for titleblock templates | ALWAYS use project-relative paths for titleblock SVG resources |

---

## 2. Quantity Takeoff (QTO) Module

### 2.1 Overview and Architecture

The QTO module (`bonsai.bim.module.qto`) calculates geometric quantities from Blender mesh data and assigns them as IFC `IfcElementQuantity` entities.

**Module structure** (7 files at `src/bonsai/bonsai/bim/module/qto/`):

| File | Purpose |
|------|---------|
| `__init__.py` | Module registration |
| `operator.py` | QTO operators (8 operators) |
| `prop.py` | `BIMQtoProperties` with calculator/rule settings |
| `ui.py` | Panel UI for quantity takeoff |
| `data.py` | Data loading and caching |
| `calculator.py` | Geometry calculation engine (30+ functions) |
| `helper.py` | Mesh quantity calculation helpers |

The QTO system uses a two-layer approach:
1. **Calculator functions** — Extract numeric values from Blender geometry
2. **QTO rules** — Map IFC classes to standard quantity sets and calculator functions via JSON configuration

### 2.2 Calculator System

The calculator (`calculator.py`) provides 30+ functions for extracting quantities from Blender mesh geometry. Functions operate on `bpy.types.Object` instances and use Blender's mesh API, BMesh, and matrix transformations.

**Quantity categories and key functions**:

#### Linear Quantities

```python
# Bonsai v0.8.x — Linear quantity calculator functions
# get_linear_length()    — longest bounding box edge
# get_length()           — parametric axis length (AXIS2/AXIS3 aware)
# get_width()            — width from bounding box or parametric data
# get_height()           — height from bounding box Z dimension
# get_depth()            — depth dimension
# get_perimeter()        — perimeter of net footprint
# get_gross_perimeter()  — perimeter of gross footprint
```

#### Area Quantities

```python
# Bonsai v0.8.x — Area quantity calculator functions
# get_net_footprint_area()    — area of lowest polygon faces
# get_gross_footprint_area()  — area without opening subtractions
# get_roofprint_area()        — area of highest polygon faces
# get_side_area()             — lateral face areas with directional filter
# get_lateral_area()          — side surfaces
# get_top_area()              — faces with +Z normal (configurable angle threshold, default 45°)
# get_net_surface_area()      — total surface area (net of openings)
# get_gross_surface_area()    — total surface area (gross)
# get_cross_section_area()    — area at cut plane
# get_projected_area()        — projection onto x/y/z axes
# get_opening_area()          — area of openings (normal vector angle filtering)
# get_formwork_area()         — surface excluding top faces
# get_side_formwork_area()    — lateral surfaces only (excluding Z-normal faces)
```

#### Volume Quantities

```python
# Bonsai v0.8.x — Volume quantity calculator functions
# get_net_volume()       — BMesh-calculated volume (with openings)
# get_gross_volume()     — volume without opening subtractions
#                          Uses "disable-opening-subtractions" flag
#                          in ifcopenshell.api.geometry.create_shape_settings
# get_space_net_volume() — space volume minus walls/columns
#                          (decomposition-aware)
```

#### Weight Quantities

```python
# Bonsai v0.8.x — Weight quantity calculator functions
# get_net_weight()   — net_volume × mass_density
#                      Mass density from Pset_MaterialCommon.MassDensity
# get_gross_weight() — gross_volume × mass_density
# Profile-based:     — Uses Pset_ProfileMechanical.MassPerLength × length
```

#### Specialized Quantities

```python
# Bonsai v0.8.x — Specialized calculator functions
# get_stair_length()         — hypotenuse of (length² + height²)
# get_finish_floor_height()  — floor finish height (decomposition-aware)
# get_finish_ceiling_height() — ceiling height (decomposition-aware)
# get_contact_area()         — polygon intersection between objects (Shapely)
```

**External dependencies**:
- `BMesh` — Volume calculations
- `mathutils.BVHTree` — Proximity detection
- `Shapely` — 2D polygon intersection for contact area
- `ifcopenshell.api.geometry` — Shape generation with opening control

**Gross vs. Net distinction**: Gross functions calculate measures for the original element geometry WITHOUT openings. Net functions INCLUDE opening subtractions. This distinction is critical for accurate quantity takeoff.

### 2.3 Quantity Rules and JSON Configuration

QTO rules map IFC element classes to standard quantity sets and calculator functions. Rules are stored as JSON files in the `ifc5d.qto.rules` module.

**Rule JSON structure**:

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

**Rule selection in UI**: `BIMQtoProperties.qto_rule` enum is populated from `tool.Qto.get_qto_rules()`, which filters rules by IFC schema version (IFC4 vs IFC4X3).

**Fallback mechanism**: When `BIMQtoProperties.fallback=True`, the system tries alternative calculators if the primary calculator lacks support for a specific class or quantity set.

### 2.4 Standard Quantity Sets

The IFC schema defines standard quantity sets for each element class. Key AEC-relevant quantity sets:

| IFC Class | Quantity Set | Key Quantities |
|-----------|-------------|----------------|
| `IfcWall` | `Qto_WallBaseQuantities` | Length, Width, Height, GrossVolume, NetVolume, GrossSideArea, NetSideArea, GrossFootprintArea, NetFootprintArea |
| `IfcSlab` | `Qto_SlabBaseQuantities` | Width, Length, Depth, GrossArea, NetArea, GrossVolume, NetVolume, Perimeter |
| `IfcColumn` | `Qto_ColumnBaseQuantities` | Length, CrossSectionArea, OuterSurfaceArea, GrossVolume, NetVolume, GrossWeight, NetWeight |
| `IfcBeam` | `Qto_BeamBaseQuantities` | Length, CrossSectionArea, OuterSurfaceArea, GrossVolume, NetVolume, GrossWeight, NetWeight |
| `IfcDoor` | `Qto_DoorBaseQuantities` | Height, Width, Area |
| `IfcWindow` | `Qto_WindowBaseQuantities` | Height, Width, Area |
| `IfcSpace` | `Qto_SpaceBaseQuantities` | Height, FinishCeilingHeight, FinishFloorHeight, GrossFloorArea, NetFloorArea, GrossVolume, NetVolume |
| `IfcRoof` | `Qto_RoofBaseQuantities` | GrossArea, NetArea, ProjectedArea |
| `IfcStair` | `Qto_StairBaseQuantities` | Length, GrossVolume, NetVolume |

### 2.5 QTO Operators Reference

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| CalculateCircleRadius | `bim.calculate_circle_radius` | Calculate circle radius for selected vertices |
| CalculateEdgeLengths | `bim.calculate_edge_lengths` | Sum selected edge lengths |
| CalculateFaceAreas | `bim.calculate_face_areas` | Sum selected face areas |
| CalculateObjectVolumes | `bim.calculate_object_volumes` | Calculate mesh volumes |
| CalculateFormworkArea | `bim.calculate_formwork_area` | Surface area excluding top faces |
| CalculateSideFormworkArea | `bim.calculate_side_formwork_area` | Lateral surfaces only |
| CalculateSingleQuantity | `bim.calculate_single_quantity` | Calculate one quantity using a specific calculator function |
| PerformQuantityTakeOff | `bim.perform_quantity_take_off` | Batch QTO for all selected/all elements using rules |

**PerformQuantityTakeOff workflow**:

```python
# Bonsai v0.8.x — Batch quantity takeoff
import bpy

# Perform QTO using a specific rule set
bpy.ops.bim.perform_quantity_take_off(
    qto_rule="Qto_WallBaseQuantities"  # Rule name from JSON config
)

# With fallback enabled (tries alternative calculators):
# Set in BIMQtoProperties.fallback = True before calling
```

**CalculateSingleQuantity parameters**:
- `calculator` — Calculator module name
- `qto_name` — Target quantity set name (e.g., "Qto_WallBaseQuantities")
- `prop_name` — Specific property name (e.g., "GrossVolume")
- `calculator_function` — Function name (e.g., "get_gross_volume")

### 2.6 Cost Integration

The QTO module integrates with cost management via `tool.Qto.get_related_cost_item_quantities()`:

```python
# Bonsai v0.8.x — QTO-Cost integration
# get_related_cost_item_quantities(product) returns:
# [
#     {
#         "cost_item": <IfcCostItem>,
#         "quantities": [<matched quantity properties>]
#     }
# ]
# Matching links cost items to base QTO quantities
# on the same product, enabling automatic cost calculation
```

Unit conversion is handled by `tool.Qto.convert_to_project_units()`, which converts between measurement systems using the IFC schema's unit definitions.

### 2.7 QTO Anti-Patterns

| Anti-Pattern | Correct Approach |
|-------------|-----------------|
| Running QTO on non-mesh objects (empties, curves) | ALWAYS ensure target objects have mesh geometry; calculators require `bpy.types.Mesh` data |
| Using `get_gross_volume()` when openings matter | ALWAYS use `get_net_volume()` for elements with openings (doors, windows in walls) |
| Forgetting to set `Pset_MaterialCommon.MassDensity` for weight calculations | ALWAYS assign material density BEFORE running weight calculations |
| Running `perform_quantity_take_off` without selecting objects | Either select specific objects OR the operator processes all `IfcElement` instances |
| Assuming QTO results auto-update when geometry changes | ALWAYS re-run QTO calculations after modifying geometry; quantities are NOT live-linked |
| Ignoring the Gross/Net distinction | Gross = original geometry without openings; Net = with openings subtracted. ALWAYS choose the correct variant for your use case |
| Using QTO on objects without IFC class assignment | Calculator functions query IFC properties; the object MUST have an IFC class assigned via Bonsai |

---

## 3. BCF Module — BIM Collaboration Format

### 3.1 Overview and Architecture

The BCF module (`bonsai.bim.module.bcf`) implements BIM Collaboration Format support for issue tracking and coordination. It supports both BCF v2.1 and BCF v3.0 formats.

**Module structure** (5 files at `src/bonsai/bonsai/bim/module/bcf/`):

| File | Purpose |
|------|---------|
| `__init__.py` | Module registration |
| `operator.py` | BCF operators (30+ operators) |
| `prop.py` | `BCFProperties`, `BcfTopic`, `BcfComment` property groups |
| `ui.py` | Panel UI for BCF management |
| `bcfstore.py` | `BcfStore` singleton for BCF data management |

**Dependencies**: The module uses the `bcf` Python library (`bcf.bcfxml`) bundled with IfcOpenShell for BCF-XML file handling.

### 3.2 BCF Project Lifecycle

The `BcfStore` class manages BCF data as a **class-level singleton** (not an instance singleton):

```python
# Bonsai v0.8.x — BcfStore singleton pattern
class BcfStore:
    bcfxml: Union[bcf.bcfxml.BcfXml, None] = None  # Class variable

    @staticmethod
    def get_bcfxml():
        """Lazy initialization — loads BCF file on first access.
        Returns cached instance on subsequent calls.
        Resolves relative paths against .blend file directory.
        Silently handles load exceptions (e.g., permission denied)."""

    @staticmethod
    def set(bcfxml, filepath):
        """Store BCF data and update properties.
        Auto-detects BCF version (2 or 3) on load."""

    @staticmethod
    def set_by_filepath(filepath):
        """Convenience: load BCF by filename."""

    @staticmethod
    def unload_bcfxml():
        """Clear all stored BCF data."""
```

**Project lifecycle**:

```python
# Bonsai v0.8.x — BCF project lifecycle
# 1. Create new project
bpy.ops.bim.new_bcf_project()  # Creates empty BCF (v2 or v3)

# 2. OR load existing BCF file
bpy.ops.bim.load_bcf_project(filepath="/path/to/issues.bcf")

# 3. Work with topics, comments, viewpoints...

# 4. Save project
bpy.ops.bim.save_bcf_project(filepath="/path/to/output.bcf")

# 5. Unload when done
bpy.ops.bim.unload_bcf_project()
```

**Drag-and-drop support**: `BCFFileHandlerOperator` (`bim.load_bcf_project_file_handler`) registers `.bcf` as a Blender file handler, enabling drag-and-drop BCF import.

### 3.3 Topics

Topics represent coordination issues. Each topic has a GUID, title, metadata, and associated viewpoints/comments.

**Topic properties** (`BcfTopic`):

| Property | Type | Description |
|----------|------|-------------|
| `name` (GUID) | String | Unique topic identifier |
| `title` | String | Topic title/summary |
| `type` | String | Issue type (e.g., "Error", "Warning", "Info") |
| `status` | String | Workflow status (e.g., "Open", "Closed", "ReOpened") |
| `priority` | String | Priority level (e.g., "Critical", "Normal", "Minor") |
| `stage` | String | Project stage reference |
| `assigned_to` | String | Responsible person email |
| `due_date` | String | Due date |
| `description` | String | Detailed description |
| `creation_date` | String | ISO timestamp |
| `creation_author` | String | Author email |
| `modified_date` | String | Last modification timestamp |
| `modified_author` | String | Last modifier email |

**Topic operations from Python**:

```python
# Bonsai v0.8.x — BCF topic management
import bpy

# Create a new topic (requires author to be set)
bpy.ops.bim.add_bcf_topic()

# Edit topic metadata
bpy.ops.bim.edit_bcf_topic()  # Updates priority, status, type, stage, etc.

# View a topic
bpy.ops.bim.view_bcf_topic(topic_guid="abc-123-def")

# Remove a topic
bpy.ops.bim.remove_bcf_topic(topic_guid="abc-123-def")

# Labels
bpy.ops.bim.add_bcf_label()
bpy.ops.bim.remove_bcf_label(label_index=0)

# Reference links
bpy.ops.bim.add_bcf_reference_link()
bpy.ops.bim.remove_bcf_reference_link(link_index=0)
bpy.ops.bim.open_bcf_reference_link()  # Opens URL in browser

# Related topics
bpy.ops.bim.add_bcf_related_topic()
bpy.ops.bim.remove_bcf_related_topic(related_topic_index=0)
```

### 3.4 Viewpoints

Viewpoints capture camera state, element visibility, selection, and clipping planes. They provide visual context for each topic.

**Viewpoint creation**:

```python
# Bonsai v0.8.x — BCF viewpoint operations
import bpy

# Capture current camera as viewpoint (requires active camera)
bpy.ops.bim.add_bcf_viewpoint()
# This operation:
# 1. Captures camera position and orientation
# 2. Records component visibility and selection state
# 3. Renders a PNG snapshot for the viewpoint thumbnail
# 4. Stores clipping planes if present
# 5. Embeds bitmap annotations if applicable

# Activate (restore) a viewpoint
bpy.ops.bim.activate_bcf_viewpoint()
# Restores:
# - Camera position and orientation
# - Component visibility/selection
# - Clipping planes
# - Supports georeference offset correction

# Remove a viewpoint
bpy.ops.bim.remove_bcf_viewpoint()
```

**Georeference support**: When activating viewpoints, the operator accounts for coordinate offsets between the BCF viewpoint's coordinate system and the loaded IFC model's georeference origin. This is critical for federated models with different coordinate origins.

### 3.5 Comments and Issue Tracking

Comments form threaded discussions within each topic:

```python
# Bonsai v0.8.x — BCF comment management
import bpy

# Add comment (requires author and comment text)
bpy.ops.bim.add_bcf_comment()
# Optional: link comment to a specific viewpoint

# Edit comment text
bpy.ops.bim.edit_bcf_comment()
# Updates: comment text, modified_date, modified_author

# Remove comment
bpy.ops.bim.remove_bcf_comment(comment_guid="abc-123")

# Load comments for active topic
bpy.ops.bim.load_bcf_comments()
```

**Comment properties** (`BcfComment`):

| Property | Type | Description |
|----------|------|-------------|
| `name` (GUID) | String | Comment identifier |
| `comment` | String | Comment text |
| `author` | String | Author email |
| `date` | String | Creation timestamp |
| `viewpoint` | String | Linked viewpoint GUID (optional) |
| `modified_date` | String | Last edit timestamp |
| `modified_author` | String | Last editor email |
| `is_editable` | Bool | UI edit state flag |

### 3.6 BCF File Management

**Document references and BIM snippets**:

```python
# Bonsai v0.8.x — BCF document management
import bpy

# Add document reference (internal or external)
bpy.ops.bim.add_bcf_document_reference()
# BCF v3: stores in bcfxml.documents registry

# Add BIM snippet (IFC fragment for issue context)
bpy.ops.bim.add_bcf_bim_snippet()

# Add header file (IFC file reference with spatial metadata)
bpy.ops.bim.add_bcf_header_file()

# Extract embedded files to disk
bpy.ops.bim.extract_bcf_file()  # Exports header, snippet, or document

# Load IFC from BCF header
bpy.ops.bim.load_bcf_header_ifc_file()
```

**BCF version differences**:

| Feature | BCF v2.1 | BCF v3.0 |
|---------|----------|----------|
| Document storage | Per-topic | Global `bcfxml.documents` registry |
| API support | No | Yes (OpenCDE compliant) |
| Viewpoint format | BCFv2 schema | BCFv3 schema |
| Project metadata | Basic | Extended |

### 3.7 BCF Operators Reference

**Project management**:

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| NewBcfProject | `bim.new_bcf_project` | Create new BCF project (v2 or v3) |
| LoadBcfProject | `bim.load_bcf_project` | Load BCF file from disk |
| UnloadBcfProject | `bim.unload_bcf_project` | Unload current project |
| SaveBcfProject | `bim.save_bcf_project` | Export BCF to file |
| EditBcfProjectName | `bim.edit_bcf_project_name` | Rename project |

**Topic management**:

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| LoadBcfTopics | `bim.load_bcf_topics` | Load all topics from project |
| LoadBcfTopic | `bim.load_bcf_topic` | Load single topic (by GUID) |
| AddBcfTopic | `bim.add_bcf_topic` | Create new topic |
| RemoveBcfTopic | `bim.remove_bcf_topic` | Delete topic |
| EditBcfTopic | `bim.edit_bcf_topic` | Update topic fields |
| ViewBcfTopic | `bim.view_bcf_topic` | Set active topic |

**Viewpoint management**:

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| AddBcfViewpoint | `bim.add_bcf_viewpoint` | Capture camera + visibility state |
| RemoveBcfViewpoint | `bim.remove_bcf_viewpoint` | Remove viewpoint |
| ActivateBcfViewpoint | `bim.activate_bcf_viewpoint` | Restore camera + visibility + clipping |

**Comment management**:

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| LoadBcfComments | `bim.load_bcf_comments` | Populate comments for active topic |
| AddBcfComment | `bim.add_bcf_comment` | Create new comment |
| EditBcfComment | `bim.edit_bcf_comment` | Update comment text |
| RemoveBcfComment | `bim.remove_bcf_comment` | Delete comment |

### 3.8 BCF Anti-Patterns

| Anti-Pattern | Correct Approach |
|-------------|-----------------|
| Accessing `BcfStore.bcfxml` directly without calling `get_bcfxml()` | ALWAYS use `BcfStore.get_bcfxml()` for lazy initialization and path resolution |
| Creating a viewpoint without an active camera | `bim.add_bcf_viewpoint` polls for active camera; ALWAYS set a camera before calling |
| Not setting author email before creating topics/comments | ALWAYS configure `BCFProperties.author` before creating topics or comments |
| Mixing BCF v2 and v3 operations on the same file | NEVER mix versions; the version is detected on load and MUST be consistent |
| Concurrent access to BCF files from multiple processes | BcfStore's singleton pattern does NOT handle concurrent access; permission denied errors occur silently |
| Forgetting to save after modifications | BCF changes are in-memory; ALWAYS call `bim.save_bcf_project` to persist |
| Assuming viewpoint coordinates match model coordinates | Viewpoint activation applies georeference offsets; coordinate systems may differ between BCF source and loaded model |

---

## 4. Clash Detection Module

### 4.1 Overview and Architecture

The clash detection module (`bonsai.bim.module.clash`) identifies geometric conflicts between IFC elements using the IfcClash engine (a frontend for the FCL — Flexible Collision Library).

**Module structure** (6 files at `src/bonsai/bonsai/bim/module/clash/`):

| File | Purpose |
|------|---------|
| `__init__.py` | Module registration |
| `operator.py` | Clash operators (12 operators) |
| `prop.py` | `BIMClashProperties`, `ClashSet`, `ClashSource`, `Clash` |
| `ui.py` | Panel UI for clash detection |
| `data.py` | Data loading and caching |
| `decorator.py` | 3D viewport visualization of clash points |

### 4.2 Clash Sets

A **clash set** defines two groups of elements (Group A and Group B) to check against each other. Each group contains one or more IFC file sources with optional filtering.

**ClashSet properties**:

| Property | Type | Description |
|----------|------|-------------|
| `mode` | Enum | Detection type: `intersection`, `collision`, `clearance` |
| `tolerance` | Float | Precision threshold for detection |
| `clearance` | Float | Proximity distance for clearance checks |
| `allow_touching` | Bool | Whether surface contact is acceptable |
| `check_all` | Bool | Check all combinations vs. optimized subset |
| `a` | Collection[ClashSource] | Group A sources |
| `b` | Collection[ClashSource] | Group B sources |

**ClashSource properties**:

| Property | Type | Description |
|----------|------|-------------|
| `name` | String | Filepath to .ifc file |
| `filter_groups` | Collection[BIMFilterGroup] | Element filter criteria |
| `mode` | Enum | `"a"` (all), `"i"` (include filtered), `"e"` (exclude filtered) |

**Clash mode descriptions**:

| Mode | Description | Use Case |
|------|-------------|----------|
| `intersection` | Elements physically overlap | Hard clashes (pipe through wall) |
| `collision` | Elements collide (touching or overlapping) | Physical conflicts |
| `clearance` | Elements within specified distance | Maintenance access, fire safety clearances |

**Setting up clash sets from Python**:

```python
# Bonsai v0.8.x — Clash set configuration
import bpy

# Add a new clash set
bpy.ops.bim.add_clash_set()

# Add sources to Group A
bpy.ops.bim.add_clash_source(group="a")
bpy.ops.bim.select_clash_source()  # File picker for IFC source

# Add sources to Group B
bpy.ops.bim.add_clash_source(group="b")

# Configure detection mode via properties
props = bpy.context.scene.BIMClashProperties
clash_set = props.clash_sets[props.active_clash_set_index]
clash_set.mode = "intersection"
clash_set.tolerance = 0.001  # 1mm tolerance
```

### 4.3 Running Clash Detection

**Primary execution operator**: `ExecuteIfcClash` (`bim.execute_ifc_clash`)

```python
# Bonsai v0.8.x — Execute clash detection
import bpy

# Standard execution — prompts for output file path (.bcf or .json)
bpy.ops.bim.execute_ifc_clash()

# Quick clash mode (ALT+click) — runs without file dialog
# Uses default output path
```

**Execution workflow**:
1. Export clash set configuration to temporary JSON
2. IfcClash engine loads IFC sources from each group
3. Apply element filters (include/exclude by class, property, etc.)
4. Run geometric clash detection using FCL
5. Write results to BCF or JSON output file
6. Optionally create clash visualization snapshots

**Important**: Clash detection operates on IFC files on disk, NOT on the live Blender scene. The IFC sources in `ClashSource.name` MUST point to saved .ifc files.

### 4.4 Result Interpretation and Visualization

**Clash result properties** (`Clash`):

| Property | Type | Description |
|----------|------|-------------|
| `a_global_id` | String | GlobalId of element in Group A |
| `b_global_id` | String | GlobalId of element in Group B |
| `a_name` | String | Name of element A |
| `b_name` | String | Name of element B |
| `clash_type` | String | Classification (intersection, collision, clearance) |
| `status` | Bool | Tracking flag for resolution |

**Loading and viewing results**:

```python
# Bonsai v0.8.x — Clash result interaction
import bpy

# Load results from JSON
bpy.ops.bim.select_ifc_clash_results()
# Reads JSON file, selects conflicting geometry in viewport

# Navigate to individual clash
bpy.ops.bim.select_clash(index=0)
# Highlights clash pair, positions camera at clash point
# Shows two colored points (P1, P2) and connecting line

# Hide clash visualization
bpy.ops.bim.hide_clash()
```

**ClashDecorator** (`decorator.py`) renders clash visualization in the 3D viewport:

```python
# Bonsai v0.8.x — ClashDecorator visualization
# Uses GPU shaders (UNIFORM_COLOR, POLYLINE_UNIFORM_COLOR)
# Renders in POST_PIXEL space (overlay on 3D viewport)
#
# Visual elements:
# - Two colored points at P1, P2 (clash endpoints)
# - Connecting line between P1 and P2 (if different)
# - Text label at midpoint with clash info
#
# Colors from addon preferences:
# - Selected clash color
# - Unselected clash color
# - Special element color
```

### 4.5 Smart Grouping

Smart grouping clusters nearby clashes to reduce noise and identify systemic issues:

```python
# Bonsai v0.8.x — Smart clash grouping
import bpy

# Group clashes by spatial proximity
bpy.ops.bim.smart_clash_group()
# Uses configurable max_distance parameter
# Groups stored in SmartClashGroup collection

# Load smart groups for active clash set
bpy.ops.bim.load_smart_groups_for_active_clash_set()

# Select all elements in a smart group
bpy.ops.bim.select_smart_group(group_index=0)
```

**SmartClashGroup properties**:

| Property | Type | Description |
|----------|------|-------------|
| `number` | Int | Group identifier |
| `global_ids` | Collection | GlobalIds of related elements |

### 4.6 BCF Integration

Clash results can be exported directly as BCF issues:

```python
# Bonsai v0.8.x — Clash-to-BCF workflow
import bpy

# Execute clash detection with BCF output
bpy.ops.bim.execute_ifc_clash()
# When prompted for output, save as .bcf file

# Load BCF results into BCF panel
bpy.ops.bim.load_bcf_project(filepath="/path/to/clashes.bcf")

# Each clash becomes a BCF topic with:
# - Viewpoint showing the clash location
# - Component references for both clashing elements
# - Metadata about clash type and severity
```

**Export/Import for external tools**:

```python
# Bonsai v0.8.x — Clash set import/export
bpy.ops.bim.export_clash_sets()  # Export clash configuration to JSON
bpy.ops.bim.import_clash_sets()  # Import clash configuration from JSON
```

### 4.7 Clash Operators Reference

| Operator | bl_idname | Description |
|----------|-----------|-------------|
| AddClashSet | `bim.add_clash_set` | Create new clash set |
| RemoveClashSet | `bim.remove_clash_set` | Delete clash set |
| AddClashSource | `bim.add_clash_source` | Add IFC source to group |
| RemoveClashSource | `bim.remove_clash_source` | Remove source from group |
| SelectClashSource | `bim.select_clash_source` | File picker for IFC source |
| ExportClashSets | `bim.export_clash_sets` | Export configuration to JSON |
| ImportClashSets | `bim.import_clash_sets` | Import configuration from JSON |
| ExecuteIfcClash | `bim.execute_ifc_clash` | Run clash detection (ALT = quick mode) |
| SelectIfcClashResults | `bim.select_ifc_clash_results` | Load and select clash results |
| SelectClash | `bim.select_clash` | Navigate to individual clash |
| HideClash | `bim.hide_clash` | Remove clash visualization |
| SmartClashGroup | `bim.smart_clash_group` | Group clashes by proximity |
| LoadSmartGroupsForActiveClashSet | `bim.load_smart_groups_for_active_clash_set` | Load saved smart groups |
| SelectSmartGroup | `bim.select_smart_group` | Select elements in smart group |

### 4.8 Clash Anti-Patterns

| Anti-Pattern | Correct Approach |
|-------------|-----------------|
| Running clash detection on unsaved IFC files | Clash detection reads IFC files from DISK; ALWAYS save the IFC file first |
| Using `collision` mode with `allow_touching=True` for detecting overlaps | This generates false positives where elements share surfaces; use `intersection` mode for hard clash detection |
| Checking all elements against all elements without filtering | ALWAYS use `filter_groups` to limit scope; unfiltered clash detection on large models causes excessive runtime and noise |
| Ignoring smart grouping for large result sets | ALWAYS use `bim.smart_clash_group` to cluster related clashes; individual review of hundreds of clashes is impractical |
| Expecting real-time clash checking | Clash detection is a batch operation; there is NO live clash monitoring in Bonsai |
| Running clash detection within the same model without two groups | ALWAYS define BOTH Group A and Group B, even for intra-model clashes (e.g., structural vs. MEP) |
| Not specifying an output file before execution | `bim.execute_ifc_clash` requires an output path (.bcf or .json); create the output file/directory BEFORE running |

---

## 5. Python Runtime Quirks

### 5.1 Tool/Core/UI Pattern for Scripts

Bonsai separates concerns into three layers. Understanding this pattern is essential for writing scripts that interact with Bonsai functionality.

**Layer architecture**:

```
┌────────────────────────────────────────────────────────┐
│  UI Layer (bonsai.bim.module.<module>.operator.py)      │
│  - Blender operators (bpy.types.Operator)              │
│  - Collects context and parameters                     │
│  - Calls core functions with tool implementations      │
├────────────────────────────────────────────────────────┤
│  Core Layer (bonsai.core.<module>.py)                   │
│  - Pure logic functions                                │
│  - Accepts tool interfaces as type parameters          │
│  - NO Blender imports, NO side effects                 │
│  - Testable outside Blender                            │
├────────────────────────────────────────────────────────┤
│  Tool Layer (bonsai.tool.<module>.py)                   │
│  - Blender-specific implementations                   │
│  - Implements interfaces defined in core/tool.py       │
│  - Accesses bpy, Blender scene, objects, etc.          │
└────────────────────────────────────────────────────────┘
```

**How an operator calls through the layers**:

```python
# Bonsai v0.8.x — Example: How bim.add_drawing flows through layers

# 1. OPERATOR (bonsai.bim.module.drawing.operator.AddDrawing)
class AddDrawing(bpy.types.Operator):
    bl_idname = "bim.add_drawing"

    def _execute(self, context):
        # Collects parameters from UI
        target_view = context.scene.DocProperties.target_view
        location_hint = ...
        # Calls core function with tool implementations
        core.drawing.add_drawing(
            ifc=tool.Ifc,           # Tool implementation
            collector=tool.Collector,
            drawing=tool.Drawing,
            target_view=target_view,
            location_hint=location_hint,
        )

# 2. CORE (bonsai.core.drawing.add_drawing)
def add_drawing(ifc, collector, drawing, target_view, location_hint):
    # Pure logic — no Blender imports
    name = drawing.generate_drawing_name(target_view)
    matrix = drawing.generate_drawing_matrix(target_view, location_hint)
    element = ifc.run("root.create_entity", ifc_class="IfcAnnotation")
    # ... more logic using tool interfaces

# 3. TOOL (bonsai.tool.drawing.Drawing)
class Drawing(bonsai.core.tool.Drawing):
    @classmethod
    def generate_drawing_name(cls, target_view):
        # Blender-specific: accesses bpy.data, scene, etc.
        return f"{target_view}_{len(bpy.data.cameras)}"
```

**For external scripts**, the recommended approach is to call operators directly:

```python
# Bonsai v0.8.x — Recommended scripting approach
import bpy

# Option 1: Call operators (simplest)
bpy.ops.bim.add_drawing(target_view="PLAN_VIEW", location_hint=0)

# Option 2: Use tool.Ifc for IFC access (recommended for data queries)
import bonsai.tool as tool
model = tool.Ifc.get()  # Returns ifcopenshell.file instance
walls = model.by_type("IfcWall")
```

### 5.2 IfcStore Singleton — Deep Dive

`IfcStore` (`bonsai.bim.ifc`) is a static class that maintains global state for the active IFC file and Blender-IFC mappings.

**Key class-level attributes**:

```python
# Bonsai v0.8.x — IfcStore internal state
class IfcStore:
    file: ifcopenshell.file = None       # Active IFC document
    path: str = ""                        # File path on disk
    schema: str = None                    # IFC schema identifier

    # Bidirectional Blender ↔ IFC mappings
    id_map: dict = {}      # IFC element ID → Blender object/material
    guid_map: dict = {}    # GlobalId → Blender object/material

    # Transaction/undo history
    history: list = []     # TransactionStep objects
    future: list = []      # Redo stack

    # Change tracking
    edited_objs: set = set()  # Objects modified during session

    # Geometry cache (HDF5-based)
    # Cache path derived from file hash + timestamp
```

**File access**:

```python
# Bonsai v0.8.x — Accessing the IFC file

# Modern approach (RECOMMENDED):
import bonsai.tool as tool
model = tool.Ifc.get()  # Returns ifcopenshell.file or None

# Legacy approach (still functional but DEPRECATED for new code):
from bonsai.bim.ifc import IfcStore
model = IfcStore.get_file()  # Lazy-loads if needed

# NEVER access IfcStore.file directly — it may be None before lazy init
```

**Element linking**:

```python
# Bonsai v0.8.x — Blender ↔ IFC element mapping
# IfcStore.link_element(element, obj) establishes:
#   id_map[element.id()] = obj
#   guid_map[element.GlobalId] = obj
#   obj.BIMObjectProperties.ifc_definition_id = element.id()

# IfcStore.unlink_element(element) removes all associations

# Lookup examples:
blender_obj = IfcStore.id_map.get(element_id)
blender_obj = IfcStore.guid_map.get(global_id)
```

**Transaction system**: IfcStore mirrors Blender's undo system with its own transaction history. `TransactionStep` objects contain rollback/commit callbacks.

```python
# Bonsai v0.8.x — Transaction quirks
# - Nested modal operators are NOT supported by Blender
#   Bonsai emulates nesting through manual transaction management
# - Undo failures are captured in bonsai.last_error
#   (persistent callbacks don't display UI errors)
# - Cancelled modals force bpy.ops.ed.undo_push()
#   to synchronize Blender's undo stack with Bonsai's internal state
```

### 5.3 Operators vs ifcopenshell.api Directly

There are two ways to modify IFC data in Bonsai — through Bonsai operators or through `ifcopenshell.api` directly. Each has trade-offs:

**Bonsai operators** (`bpy.ops.bim.*`):

```python
# Bonsai v0.8.x — Using operators
import bpy

# Advantages:
# - Handles Blender ↔ IFC synchronization automatically
# - Updates viewport, UI panels, and caches
# - Integrates with undo/redo system
# - Validates context and prerequisites

# Disadvantages:
# - Requires correct Blender context (active object, mode, area)
# - Slower due to UI overhead
# - Limited parametrization compared to API

bpy.ops.bim.add_wall()  # Creates wall + Blender object + IFC entity
```

**ifcopenshell.api directly**:

```python
# Bonsai v0.8.x — Using ifcopenshell.api directly
import ifcopenshell.api
import bonsai.tool as tool

model = tool.Ifc.get()

# Advantages:
# - Full control over IFC operations
# - No context requirements
# - Faster execution
# - More flexible parametrization

# Disadvantages:
# - Does NOT update Blender viewport automatically
# - Does NOT sync with IfcStore mappings
# - Does NOT update Bonsai UI caches
# - MUST manually handle data synchronization

wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="My Wall Type")

# CRITICAL: After direct API calls, MUST refresh Bonsai state
# The exact refresh mechanism depends on what was modified:
# - For new entities: link via IfcStore or reload model
# - For property changes: purge relevant data caches
```

**When to use which**:

| Scenario | Recommended Approach |
|----------|---------------------|
| Interactive modeling (add walls, slabs) | ALWAYS use `bpy.ops.bim.*` operators |
| Batch property modifications | Use `ifcopenshell.api` directly + cache purge |
| Querying IFC data (read-only) | Use `tool.Ifc.get()` + ifcopenshell queries |
| Creating IFC entities with custom parameters | Use `ifcopenshell.api` + manual sync |
| Scripts running headless (no UI) | MUST use `ifcopenshell.api` directly |

### 5.4 Operator Context Requirements

Bonsai operators have strict context requirements. Calling an operator in the wrong context raises `RuntimeError` or produces incorrect results.

**Common context requirements**:

```python
# Bonsai v0.8.x — Operator context pitfalls

# Many operators require an active IFC file
# ALWAYS check before calling:
import bonsai.tool as tool
if tool.Ifc.get() is None:
    raise RuntimeError("No IFC file loaded")

# Some operators require a selected object with IFC data
# ALWAYS verify:
obj = bpy.context.active_object
if obj is None or not obj.BIMObjectProperties.ifc_definition_id:
    raise RuntimeError("No IFC object selected")

# Drawing operators require camera context
# MUST be in camera view for CreateDrawing:
if bpy.context.scene.camera is None:
    raise RuntimeError("No active camera")

# Some operators only work in OBJECT mode
# ALWAYS check mode:
if bpy.context.mode != 'OBJECT':
    bpy.ops.object.mode_set(mode='OBJECT')
```

**Poll functions**: Most Bonsai operators implement `poll()` classmethods that check prerequisites. If `poll()` returns False, the operator is greyed out in UI and raises an error if called from Python.

### 5.5 BlenderBIM to Bonsai Migration

The rename from BlenderBIM to Bonsai (v0.8.0, September 2024) changed all module import paths:

**Import path migration**:

| Old (BlenderBIM) | New (Bonsai v0.8.0+) |
|-------------------|-----------------------|
| `from blenderbim.bim.ifc import IfcStore` | `from bonsai.bim.ifc import IfcStore` |
| `import blenderbim.tool as tool` | `import bonsai.tool as tool` |
| `blenderbim.bim.module.drawing` | `bonsai.bim.module.drawing` |
| `blenderbim.core.drawing` | `bonsai.core.drawing` |
| `bpy.ops.bim.*` | `bpy.ops.bim.*` (UNCHANGED) |

**Critical notes**:
- Operator `bl_idname` values (`bim.*`) did NOT change — only Python module paths changed
- Many online examples and tutorials still use `blenderbim.*` imports — these ALWAYS fail on Bonsai v0.8.0+
- The old `blenderbim` add-on files may remain installed but inactive after upgrade — they MUST be manually removed to avoid conflicts
- Property classes (e.g., `BIMProperties`, `DocProperties`) retained their names

**Version detection**:

```python
# Bonsai v0.8.x — Detect Bonsai vs BlenderBIM
try:
    import bonsai.tool as tool  # Bonsai v0.8.0+
    IS_BONSAI = True
except ImportError:
    try:
        import blenderbim.tool as tool  # Legacy BlenderBIM
        IS_BONSAI = False
    except ImportError:
        raise RuntimeError("Neither Bonsai nor BlenderBIM is installed")
```

### 5.6 Bonsai Property Access

Bonsai stores its state in Blender scene and object properties. Key property access paths:

```python
# Bonsai v0.8.x — Property access patterns
import bpy

# Scene-level properties:
bim_props = bpy.context.scene.BIMProperties
# .ifc_file — path to active IFC file
# .schema_dir — IFC schema directory
# .data_dir — Bonsai data directory

doc_props = bpy.context.scene.DocProperties
# .drawings — Collection of Drawing items
# .sheets — Collection of Sheet items
# .schedules — Collection of Document items
# .target_view — active drawing target view
# .titleblock — selected titleblock enum

qto_props = bpy.context.scene.BIMQtoProperties
# .calculator — active calculator name
# .calculator_function — active calculation function
# .qto_rule — active rule set
# .qto_result — last calculation result string
# .qto_name — quantity set name
# .prop_name — property name

clash_props = bpy.context.scene.BIMClashProperties
# .clash_sets — Collection of ClashSet items
# .smart_clash_groups — Collection of SmartClashGroup items

bcf_props = bpy.context.scene.BCFProperties
# .topics — Collection of BcfTopic items
# .author — author email for BCF operations

# Object-level properties:
obj = bpy.context.active_object
ifc_id = obj.BIMObjectProperties.ifc_definition_id  # IFC entity ID
# If ifc_id == 0, the object has NO IFC assignment

# Camera-level properties (for drawings):
camera = bpy.context.scene.camera
cam_props = camera.data.BIMCameraProperties
# .target_view — PLAN_VIEW, SECTION_VIEW, etc.
# .diagram_scale — scale enum
# .linework_mode — OPENCASCADE or FREESTYLE
# .has_underlay / .has_linework / .has_annotation
```

### 5.7 Data Synchronization: Blender Objects and IFC Entities

Maintaining consistency between Blender's scene graph and the IFC model is the central challenge of Bonsai scripting.

**How synchronization works**:

```python
# Bonsai v0.8.x — Data synchronization mechanisms

# 1. IfcStore.id_map / guid_map — bidirectional lookups
#    Updated by IfcStore.link_element() / unlink_element()

# 2. BIMObjectProperties.ifc_definition_id — stored on each Blender object
#    Links object → IFC entity

# 3. edited_objs tracking — IfcStore.edited_objs set
#    Tracks objects modified during session for sync on save

# 4. Message bus listeners — tool.Ifc.setup_listeners(obj)
#    Registers bpy.msgbus callbacks to detect Blender property changes

# 5. Data caches — Module-specific data classes
#    Each module (drawing, qto, etc.) has a data.py with caching
#    Caches MUST be purged after direct API modifications
```

**Manual synchronization after direct API calls**:

```python
# Bonsai v0.8.x — Manual sync patterns

# After modifying properties via ifcopenshell.api:
# Purge the relevant data cache
from bonsai.bim.module.pset.data import PsetData
PsetData.purge()

# After creating new IFC entities:
# Option 1: Use tool.Collector to create Blender representation
import bonsai.tool as tool
tool.Collector.assign(obj)

# Option 2: Reload the entire model (heavy, last resort)
bpy.ops.bim.load_project(filepath=tool.Ifc.get_path())

# After geometry changes:
# Trigger geometric rebuild
bpy.ops.bim.update_representation(obj=obj.name)
```

**Cache system**: Each Bonsai module maintains a data cache (e.g., `DrawingData`, `PsetData`). These caches are loaded lazily and MUST be manually purged when data changes outside the normal operator flow.

### 5.8 Runtime Anti-Patterns

| Anti-Pattern | Correct Approach |
|-------------|-----------------|
| Importing `from blenderbim.bim.ifc import IfcStore` | ALWAYS use `from bonsai.bim.ifc import IfcStore` or preferably `import bonsai.tool as tool; model = tool.Ifc.get()` |
| Accessing `IfcStore.file` directly | ALWAYS use `IfcStore.get_file()` or `tool.Ifc.get()` for lazy initialization |
| Modifying IFC data via `ifcopenshell.api` without purging caches | ALWAYS purge relevant module data caches after direct API modifications |
| Calling operators without checking `poll()` prerequisites | ALWAYS verify context: active object, IFC file loaded, correct mode |
| Assuming Blender objects auto-update after IFC changes | Blender viewport does NOT auto-refresh; ALWAYS trigger representation update or reload |
| Using threading with Bonsai/IfcStore | IfcStore is NOT thread-safe; ALWAYS run Bonsai operations on the main thread |
| Storing references to IFC entities across undo operations | Undo invalidates IfcStore.file and all entity references; ALWAYS re-query after undo |
| Writing scripts that work in both BlenderBIM and Bonsai without version detection | ALWAYS implement version detection (see section 5.5) or target a single version |
| Calling `bpy.ops.bim.*` operators from a non-main thread or modal context | Operators require the main thread and appropriate Blender context; use `bpy.app.timers` for deferred execution |
| Assuming `BIMObjectProperties.ifc_definition_id` is always valid | The ID becomes invalid after file reload or undo; ALWAYS verify against `IfcStore.id_map` |

---

## Sources

### Bonsai Source Code (v0.8.0)
- Drawing module: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/bim/module/drawing
- QTO module: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/bim/module/qto
- BCF module: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/bim/module/bcf
- Clash module: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/bim/module/clash
- Core drawing: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/core/
- Tool implementations: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/tool/
- IfcStore (ifc.py): https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai/bonsai/bim/ifc.py

### Official Documentation
- Bonsai Documentation: https://docs.bonsaibim.org/
- Bonsai Drawing Guide: https://docs.bonsaibim.org/guides/drawings/index.html
- IfcOpenShell BCF Documentation: https://docs.ifcopenshell.org/bcf.html
- IfcOpenShell Bonsai Reference: https://docs.ifcopenshell.org/bonsai.html

### Community Resources
- OSArch Community (Bonsai tag): https://community.osarch.org/discussions/tagged/bonsai-bim/p1
- IfcOpenShell scripting discussion: https://community.osarch.org/discussion/504/ifcopenshell-scripting-on-ifc-file-loaded-in-blenderbim
- BlenderBIM Python library documentation: https://community.osarch.org/discussion/2201/
- Clash detection workflow: https://community.osarch.org/discussion/2736/clash-detection-how-to
- BlenderBIM to Bonsai migration: https://github.com/IfcOpenShell/IfcOpenShell/issues/5422
