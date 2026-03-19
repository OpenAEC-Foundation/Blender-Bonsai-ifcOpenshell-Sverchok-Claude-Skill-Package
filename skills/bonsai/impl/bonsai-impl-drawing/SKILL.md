---
name: bonsai-impl-drawing
description: >
  Use when generating 2D drawings from IFC models in Bonsai -- plans, sections, elevations,
  annotations, or sheet layouts. Prevents the common mistake of creating views without proper
  drawing style configuration (getting empty or unfiltered output). Covers the complete
  documentation pipeline from model views through annotations to printable SVG sheet output.
  Keywords: drawing, plan, section, elevation, annotation, sheet, SVG, titleblock,
  documentation, 2D drawing, Bonsai drawing.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Drawing and Documentation — Implementation Guide

> **Version**: Bonsai v0.8.x | Blender 4.2+ | Python 3.11
> **Module path**: `bonsai.bim.module.drawing` — NEVER `blenderbim.bim.module.drawing`
> **Dependencies**: bonsai-core-architecture (Bonsai tool/core/operator pattern)

## 1. Critical Warnings

- ALWAYS use `bonsai.bim.module.drawing` — NEVER use `blenderbim.bim.module.drawing` (legacy name)
- ALWAYS activate a drawing camera before calling `bim.create_drawing`
- NEVER edit generated SVG files directly — ALWAYS modify annotations in Blender and regenerate
- ALWAYS install InkScape before using sheet composition or PDF export
- Annotations are ONLY generated for orthographic camera views — NEVER for perspective views
- ALWAYS sync IFC data before generating drawings (`sync=True` or save first)
- ALWAYS use project-relative paths for titleblock SVG templates

## 2. Architecture Overview

The drawing module follows Bonsai's core/tool/operator pattern:

```
bonsai/core/drawing.py        → Pure logic (no Blender imports)
bonsai/tool/drawing.py         → Blender-specific implementation
bonsai/bim/module/drawing/
├── operator.py                → 40+ Blender operators
├── prop.py                    → Property definitions
├── ui.py                      → Panel UI
├── annotation.py              → Annotator class (geometry creation)
├── svgwriter.py               → SvgWriter class (SVG engine)
├── sheeter.py                 → SheetBuilder class (sheet composition)
├── decoration.py              → Viewport decorations
├── gizmos.py                  → Interactive annotation gizmos
├── handler.py                 → Event handlers
├── helper.py                  → Utilities (format_distance, BoundingBox)
├── scheduler.py               → Schedule/table generation
├── shaders.py                 → GPU shaders for viewport
├── workspace.py               → AnnotationTool workspace + keymaps
└── data.py                    → Data loading and caching
```

## 3. Drawing Types Decision Tree

```
What type of drawing do you need?
│
├─ Floor plan (top-down view)        → target_view="PLAN_VIEW"
├─ Vertical section (cut through)    → target_view="SECTION_VIEW"
├─ Facade/external view              → target_view="ELEVATION_VIEW"
├─ Reflected ceiling plan            → target_view="REFLECTED_PLAN_VIEW"
└─ 3D perspective/axonometric        → target_view="MODEL_VIEW"
```

Each drawing is backed by:
- An `IfcAnnotation` entity in an `IfcGroup`
- Properties in `EPset_Drawing` property set:
  - `TargetView` — one of the five view types
  - `Scale` — diagram scale (e.g., "1:100")
  - `HumanScale` — display string for scale
  - `HasUnderlay` — render-based background image toggle
  - `HasLinework` — vector line drawing toggle
  - `HasAnnotation` — annotation overlay toggle

## 4. Complete Drawing Pipeline

### 4.1 Step 1: Create a Drawing

```python
# Bonsai v0.8.x — Add a plan view drawing
import bpy

bpy.ops.bim.add_drawing(
    target_view="PLAN_VIEW",
    location_hint=0  # 0 = auto-detect from active storey
)
```

**Location hint rules**:
- `PLAN_VIEW` / `REFLECTED_PLAN_VIEW`: `location_hint` maps to building storey index
- `SECTION_VIEW` / `ELEVATION_VIEW`: `location_hint` represents direction (N/S/E/W)
- `MODEL_VIEW`: `location_hint` is ignored

**Core function** (`bonsai/core/drawing.py`):

```python
# Bonsai v0.8.x — add_drawing creates:
# 1. Unique drawing name from target_view
# 2. Camera matrix from target_view + location_hint
# 3. IfcAnnotation via ifcopenshell.api.run("root.create_entity")
# 4. IfcGroup for annotations
# 5. IfcDocumentReference to SVG output path
# 6. EPset_Drawing properties (TargetView, Scale, etc.)
# 7. Blender camera object assigned to collection
```

### 4.2 Step 2: Configure Camera Properties

```python
# Bonsai v0.8.x — Camera/drawing configuration
# Access via camera object's BIMCameraProperties

# Scale options (metric):
#   1:5000, 1:2000, 1:1000, 1:500, 1:200, 1:100, 1:50,
#   1:20, 1:10, 1:5, 1:2, 1:1, CUSTOM
# Custom scale: custom_scale_numerator / custom_scale_denominator

# Camera type: ORTHO (orthographic) | PERSP (perspective)
# Raster resolution: raster_x, raster_y (default 1000 px each)
# DPI: default 75
# View dimensions: width, height (default 50 m each)
# NTS flag: is_nts (Not To Scale)
```

**Layer toggles** (control what the drawing contains):
- `has_underlay` — render-based background image
- `has_linework` — vector line drawing
- `has_annotation` — annotation overlays

**Filter mode**: `NONE` or include/exclude filter groups for element visibility control.

### 4.3 Step 3: Add Annotations

```python
# Bonsai v0.8.x — Add annotations to the active drawing
import bpy

# PREREQUISITE: MUST have active drawing (camera view)

# Add a text annotation
bpy.ops.bim.add_annotation(object_type="TEXT")

# Add a dimension with type reference
bpy.ops.bim.add_annotation(
    object_type="DIMENSION",
    relating_type_id=42  # IFC ID of annotation type
)
```

**Supported annotation types**:

| Type | Description | Blender Data |
|------|-------------|-------------|
| `TEXT` | Text labels | Mesh |
| `TEXT_LEADER` | Text with leader line | Curve |
| `DIMENSION` | Linear dimensions | Mesh |
| `ANGLE` | Angular dimensions | Mesh |
| `SECTION_LEVEL` | Section cut markers | Mesh |
| `PLAN_LEVEL` | Level indicators | Mesh |
| `ELEVATION` | Elevation markers (SVG symbols) | Mesh |
| `GRID` | Grid line references | Mesh |
| `FILL_AREA` | Hatching and fills | Mesh |
| `STAIR_ARROW` | Stair direction indicators | Mesh |
| `HIDDEN_LINE` | Hidden/dashed lines | Mesh |
| `BREAKLINE` | Break line symbols | Mesh |

**Text properties** (`BIMTextProperties`):
- Font sizes (mm): 1.8, 2.5, 3.5, 5.0, 7.0
- Horizontal alignment: left, middle, right
- Vertical alignment: top, middle, bottom
- Literals support 16 categories: Basic, Attributes, Property Sets, Material, Task, Cost, etc.
- SVG output supports markdown: bold, italic, links, bullet points

### 4.4 Step 4: Generate SVG Drawing

```python
# Bonsai v0.8.x — Generate SVG from active drawing
bpy.ops.bim.create_drawing(
    sync=True,         # Pre-sync IFC data (ALWAYS recommended)
    print_all=False,   # False = active drawing only; True = all checked drawings
    open_viewer=False  # Launch viewer after generation
)
```

**SVG generation pipeline**:

```
CreateDrawing Pipeline (bim.create_drawing):
1. Initialize camera and drawing metadata
2. Sync IFC file (if sync=True)
3. Generate UNDERLAY (raster render → PNG → embedded in SVG)
4. Generate LINEWORK (vector):
   ├─ OPENCASCADE mode → HLR via OCC library (precise, slower)
   └─ FREESTYLE mode → Blender Freestyle SVG Exporter (artistic, faster)
5. Generate ANNOTATIONS (via SvgWriter, orthographic only)
6. Combine all SVG layers into final output
```

**Linework mode decision**:

| Situation | Use |
|-----------|-----|
| Precise hidden-line removal needed | `OPENCASCADE` mode |
| Artistic line styles preferred | `FREESTYLE` mode |
| FREESTYLE mode selected | ALWAYS verify Freestyle SVG Exporter add-on is enabled |

**Fill modes** (`fill_mode`): `NONE`, `SHAPELY` (polygon fills), `SVGFILL` (SVG-native patterns)

**Cut modes** (`cut_mode`): `BISECT` (Blender mesh bisect), `OPENCASCADE` (OCC section cutting)

### 4.5 Step 5: Compose Sheets

```python
# Bonsai v0.8.x — Sheet composition pipeline

# Step 1: Create sheet with titleblock
bpy.ops.bim.add_sheet()
# Creates IfcDocumentInformation + IfcDocumentReference
# Generates base SVG layout from titleblock template

# Step 2: Add drawings to sheet
bpy.ops.bim.add_drawing_to_sheet()
# Links drawing (IfcDocumentReference) to the sheet

# Step 3: Build final sheet
bpy.ops.bim.create_sheets()
# Composes all drawings into sheet SVG
# Optionally exports to PDF/DXF (requires InkScape)
```

**SheetBuilder** (`sheeter.py`) composition process:
1. **Titleblock Processing**: embeds titleblock SVG, applies geolocation data
2. **Drawing Integration**: embeds drawing SVGs with style isolation via `ensure_drawing_unique_styles()` (prefixes CSS classes), applies clipping paths
3. **Document Assembly**: incorporates schedules, resolves mustache-syntax template variables

**Auto-layout** (`next_drawing_location()`):
- Default start: (30, 30) mm
- Padding: 10 mm between drawings
- Horizontal placement within rows, new row when exceeding titleblock width
- Default titleblock width: 840 mm

### 4.6 Step 6: Export and Open

```python
# Bonsai v0.8.x — Open generated outputs
bpy.ops.bim.open_drawing()   # Open SVG with system viewer
bpy.ops.bim.open_sheet()     # Open sheet with system viewer
bpy.ops.bim.open_layout()    # Open SVG layout file
```

## 5. SVG Coordinate Pipeline

The `SvgWriter` converts 3D model coordinates to SVG canvas:

```
3D world coordinates
    → project_point_onto_camera() (plane intersection)
    → camera-space coordinates
    → offset to canvas center, scale by (diagram_scale × 1000)
    → SVG canvas coordinates (millimeters)
```

Unit conversion: IFC meters → SVG millimeters.

`convert_to_mm()` normalizes CSS length units (cm, in, pt, px, Q, pc) to millimeters for positioning.

## 6. Key Operator Reference

| Operator | bl_idname | Purpose |
|----------|-----------|---------|
| AddDrawing | `bim.add_drawing` | Create new drawing view with camera |
| CreateDrawing | `bim.create_drawing` | Generate/refresh SVG |
| DuplicateDrawing | `bim.duplicate_drawing` | Copy drawing (optional annotation copy) |
| AddAnnotation | `bim.add_annotation` | Add annotation to active drawing |
| AddAnnotationType | `bim.add_annotation_type` | Create annotation type template |
| AddSheet | `bim.add_sheet` | Create new sheet document |
| CreateSheets | `bim.create_sheets` | Build sheets with PDF/DXF export |
| AddDrawingToSheet | `bim.add_drawing_to_sheet` | Link drawing to sheet |
| OpenSheet | `bim.open_sheet` | Open sheet with viewer |
| OpenDrawing | `bim.open_drawing` | Open SVG with viewer |
| OpenLayout | `bim.open_layout` | Open SVG layout file |
| ActivateModel | `bim.activate_model` | Switch to 3D model view |
| SelectAllDrawings | `bim.select_all_drawings` | Batch select drawings |
| SelectAllSheets | `bim.select_all_sheets` | Batch select sheets |

## 7. Annotation Workspace

The `AnnotationTool` workspace provides dedicated annotation editing:

| Shortcut | Action |
|----------|--------|
| `Shift+A` | Add new annotation |
| `Shift+T` | Bulk tag (auto-adjust to selected objects) |
| `Shift+G` | Readjust tags based on assigned products |
| `Shift+E` | Edit text annotations |

## 8. Drawing Style Configuration

`DrawingStyle` PropertyGroup controls rendering appearance:
- `render_type`: `DEFAULT` (Blender render settings) or `VIEWPORT` (viewport shading)
- `raster_style`: JSON-formatted render configuration string

## 9. Reference Files

- **[references/methods.md](references/methods.md)** — Complete drawing API signatures
- **[references/examples.md](references/examples.md)** — Working drawing workflow examples
- **[references/anti-patterns.md](references/anti-patterns.md)** — Drawing anti-patterns and mistakes to avoid

## 10. Dependencies

- **InkScape**: REQUIRED for sheet composition and PDF/DXF export
- **OpenCASCADE** (via ifcopenshell): REQUIRED for OPENCASCADE linework/cut modes
- **Shapely**: REQUIRED for SHAPELY fill mode and polygon operations
- **Freestyle SVG Exporter**: REQUIRED when using FREESTYLE linework mode (Blender add-on)
