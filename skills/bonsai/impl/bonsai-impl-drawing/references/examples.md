# Drawing Workflow Examples

> **Version**: Bonsai v0.8.x | Module: `bonsai.bim.module.drawing`

## Example 1: Complete Drawing Pipeline — Floor Plan to PDF Sheet

```python
# Bonsai v0.8.x — Full pipeline: create plan, annotate, compose sheet, export PDF
import bpy

# ─── Step 1: Create a plan view drawing ───
bpy.ops.bim.add_drawing(
    target_view="PLAN_VIEW",
    location_hint=0  # Auto-detect from active building storey
)
# Result: New camera object created, IfcAnnotation + IfcGroup in IFC model,
# EPset_Drawing property set with TargetView="PLAN_VIEW"

# ─── Step 2: Add annotations to the drawing ───
# PREREQUISITE: The drawing camera MUST be active (selected in viewport)

# Add a text label
bpy.ops.bim.add_annotation(object_type="TEXT")

# Add a linear dimension
bpy.ops.bim.add_annotation(object_type="DIMENSION")

# Add a level indicator
bpy.ops.bim.add_annotation(object_type="PLAN_LEVEL")

# ─── Step 3: Generate the SVG drawing ───
bpy.ops.bim.create_drawing(
    sync=True,          # ALWAYS sync IFC data before generation
    print_all=False,    # Only active drawing
    open_viewer=False   # Do not launch viewer
)
# Result: SVG file generated at the IfcDocumentReference path

# ─── Step 4: Create a sheet and add the drawing ───
bpy.ops.bim.add_sheet()
# Result: IfcDocumentInformation created, titleblock SVG template applied

bpy.ops.bim.add_drawing_to_sheet()
# Result: Drawing linked to the active sheet

# ─── Step 5: Build the sheet and export ───
bpy.ops.bim.create_sheets()
# Result: Final composed SVG with titleblock + drawing + annotations
# PDF/DXF export if InkScape is installed

# ─── Step 6: View the result ───
bpy.ops.bim.open_sheet()
```

## Example 2: Create Multiple Drawing Types

```python
# Bonsai v0.8.x — Create plan, section, and elevation views
import bpy

# Plan view of ground floor (storey index 0)
bpy.ops.bim.add_drawing(
    target_view="PLAN_VIEW",
    location_hint=0
)

# Plan view of first floor (storey index 1)
bpy.ops.bim.add_drawing(
    target_view="PLAN_VIEW",
    location_hint=1
)

# Section view (vertical cut)
bpy.ops.bim.add_drawing(
    target_view="SECTION_VIEW",
    location_hint=0  # Directional: N/S/E/W mapped to index
)

# Elevation view (facade)
bpy.ops.bim.add_drawing(
    target_view="ELEVATION_VIEW",
    location_hint=0
)

# Reflected ceiling plan
bpy.ops.bim.add_drawing(
    target_view="REFLECTED_PLAN_VIEW",
    location_hint=0
)

# 3D model view (perspective/axonometric)
bpy.ops.bim.add_drawing(
    target_view="MODEL_VIEW",
    location_hint=0  # Ignored for MODEL_VIEW
)
```

## Example 3: Batch Generate All Drawings

```python
# Bonsai v0.8.x — Generate SVGs for all checked drawings at once
import bpy

bpy.ops.bim.create_drawing(
    print_all=True,     # Generate ALL checked drawings
    sync=True,          # Sync IFC data first
    open_viewer=False
)
```

## Example 4: Add Various Annotation Types

```python
# Bonsai v0.8.x — Annotate a drawing with multiple annotation types
import bpy

# PREREQUISITE: Active drawing camera MUST be selected

# Text annotation
bpy.ops.bim.add_annotation(object_type="TEXT")

# Text with leader line
bpy.ops.bim.add_annotation(object_type="TEXT_LEADER")

# Linear dimension
bpy.ops.bim.add_annotation(object_type="DIMENSION")

# Angular dimension
bpy.ops.bim.add_annotation(object_type="ANGLE")

# Section cut marker
bpy.ops.bim.add_annotation(object_type="SECTION_LEVEL")

# Grid line reference
bpy.ops.bim.add_annotation(object_type="GRID")

# Hatching/fill pattern
bpy.ops.bim.add_annotation(object_type="FILL_AREA")

# Stair direction indicator
bpy.ops.bim.add_annotation(object_type="STAIR_ARROW")

# Break line symbol
bpy.ops.bim.add_annotation(object_type="BREAKLINE")
```

## Example 5: Annotation with Type Reference

```python
# Bonsai v0.8.x — Create annotation type and use it
import bpy

# Step 1: Create a reusable annotation type
bpy.ops.bim.add_annotation_type()
# Result: Creates annotation type template in IFC model

# Step 2: Get the type's IFC entity ID
# (access via Bonsai property or IfcStore query)
# type_id = <retrieved IFC entity ID>

# Step 3: Add annotation referencing the type
bpy.ops.bim.add_annotation(
    object_type="DIMENSION",
    relating_type_id=42  # Replace with actual IFC entity ID
)
```

## Example 6: Duplicate a Drawing

```python
# Bonsai v0.8.x — Duplicate an existing drawing
import bpy

# Duplicate drawing WITHOUT annotations
bpy.ops.bim.duplicate_drawing(
    should_duplicate_annotations=False
)

# Duplicate drawing WITH all annotations copied
bpy.ops.bim.duplicate_drawing(
    should_duplicate_annotations=True
)
```

## Example 7: Multi-Sheet Document Set

```python
# Bonsai v0.8.x — Create a document set with multiple sheets
import bpy

# ─── Generate all drawings first ───
bpy.ops.bim.create_drawing(print_all=True, sync=True, open_viewer=False)

# ─── Sheet 1: Floor Plans ───
bpy.ops.bim.add_sheet()
# Select ground floor plan drawing, then:
bpy.ops.bim.add_drawing_to_sheet()
# Select first floor plan drawing, then:
bpy.ops.bim.add_drawing_to_sheet()

# ─── Sheet 2: Sections and Elevations ───
bpy.ops.bim.add_sheet()
# Select section drawing, then:
bpy.ops.bim.add_drawing_to_sheet()
# Select elevation drawing, then:
bpy.ops.bim.add_drawing_to_sheet()

# ─── Build all sheets ───
bpy.ops.bim.select_all_sheets()
bpy.ops.bim.create_sheets()
# Result: All selected sheets composed with titleblocks, exported to PDF/DXF
```

## Example 8: Switch Between Model and Drawing Views

```python
# Bonsai v0.8.x — Toggle between 3D model and 2D drawing views
import bpy

# Switch to 3D model view (shows all objects, hides annotations)
bpy.ops.bim.activate_model()

# Open a specific drawing (switches to camera view, shows annotations)
bpy.ops.bim.open_drawing()
```

## Example 9: Drawing with Filter Groups

```python
# Bonsai v0.8.x — Configure drawing element filtering
import bpy

# Access the active camera's BIMCameraProperties
camera = bpy.context.scene.camera
props = camera.data.BIMCameraProperties

# Set filter mode (controls which elements appear in the drawing)
# NONE = show all elements
# Other modes use filter_groups to include/exclude elements
```

## Example 10: SVG Coordinate Understanding

```python
# Bonsai v0.8.x — Understanding the SVG coordinate pipeline
# (Reference only — SvgWriter handles this internally)

# The SvgWriter converts coordinates through this pipeline:
# 1. 3D world coordinates (meters)
#    ↓ project_point_onto_camera() — plane intersection
# 2. Camera-space coordinates
#    ↓ offset to canvas center
#    ↓ scale by (diagram_scale × 1000)
# 3. SVG canvas coordinates (millimeters)

# Example: At 1:100 scale
# diagram_scale = 0.01 (1/100)
# A 10m wall → 10 × 0.01 × 1000 = 100mm in SVG

# Sheet auto-layout positions drawings:
# - Start position: (30, 30) mm from top-left
# - Padding: 10 mm between drawings
# - Row breaks when exceeding titleblock width (default 840 mm)
```
