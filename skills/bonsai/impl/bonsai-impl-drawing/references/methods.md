# Drawing API — Method Signatures Reference

> **Version**: Bonsai v0.8.x | Module: `bonsai.bim.module.drawing`

## 1. Core Drawing Functions (`bonsai/core/drawing.py`)

### add_drawing

```python
def add_drawing(
    ifc: type[tool.Ifc],
    collector: type[tool.Collector],
    drawing: type[tool.Drawing],
    target_view: ifcopenshell.util.representation.TARGET_VIEW,
    location_hint: Union[tool.Drawing.LocationHintLiteral, int],
) -> None:
```

**Parameters**:
- `ifc` — Ifc tool interface (provides `ifc.get()` for IfcStore access)
- `collector` — Collector tool interface (manages Blender collection assignments)
- `drawing` — Drawing tool interface (provides drawing-specific operations)
- `target_view` — One of: `"PLAN_VIEW"`, `"SECTION_VIEW"`, `"ELEVATION_VIEW"`, `"REFLECTED_PLAN_VIEW"`, `"MODEL_VIEW"`
- `location_hint` — Storey index (plan views) or directional orientation (section/elevation views), `0` for auto-detect

**Creates**:
1. Unique drawing name based on `target_view`
2. Camera matrix from `target_view` and `location_hint`
3. `IfcAnnotation` via `ifcopenshell.api.run("root.create_entity")`
4. `IfcGroup` for associated annotations
5. `IfcDocumentReference` pointing to SVG output path
6. `EPset_Drawing` property set with TargetView, Scale, HumanScale, HasUnderlay, HasLinework, HasAnnotation
7. Blender camera object assigned to drawing collection

### add_annotation

```python
def add_annotation(
    ifc: type[tool.Ifc],
    collector: type[tool.Collector],
    drawing: type[tool.Drawing],
    object_type: str,
    relating_type_id: Optional[int] = None,
) -> None:
```

**Parameters**:
- `object_type` — One of: `"TEXT"`, `"TEXT_LEADER"`, `"DIMENSION"`, `"ANGLE"`, `"SECTION_LEVEL"`, `"PLAN_LEVEL"`, `"ELEVATION"`, `"GRID"`, `"FILL_AREA"`, `"STAIR_ARROW"`, `"HIDDEN_LINE"`, `"BREAKLINE"`
- `relating_type_id` — Optional IFC entity ID of the annotation type template

## 2. Operators (`bonsai/bim/module/drawing/operator.py`)

### AddDrawing (`bim.add_drawing`)

```python
class AddDrawing(bpy.types.Operator):
    bl_idname = "bim.add_drawing"
    bl_label = "Add Drawing"

    target_view: bpy.props.EnumProperty(
        items=[
            ("PLAN_VIEW", "Plan View", ""),
            ("SECTION_VIEW", "Section View", ""),
            ("ELEVATION_VIEW", "Elevation View", ""),
            ("REFLECTED_PLAN_VIEW", "Reflected Plan View", ""),
            ("MODEL_VIEW", "Model View", ""),
        ]
    )
    location_hint: bpy.props.IntProperty(default=0)
```

### CreateDrawing (`bim.create_drawing`)

```python
class CreateDrawing(bpy.types.Operator):
    bl_idname = "bim.create_drawing"
    bl_label = "Create Drawing"

    print_all: bpy.props.BoolProperty(default=False)
    # True = generate all checked drawings; False = active drawing only
    open_viewer: bpy.props.BoolProperty(default=False)
    # True = launch viewer after generation
    sync: bpy.props.BoolProperty(default=False)
    # True = pre-sync IFC data before drawing generation
```

### DuplicateDrawing (`bim.duplicate_drawing`)

```python
class DuplicateDrawing(bpy.types.Operator):
    bl_idname = "bim.duplicate_drawing"
    bl_label = "Duplicate Drawing"

    should_duplicate_annotations: bpy.props.BoolProperty(default=False)
    # True = also duplicate all annotations in the drawing group
```

### AddAnnotation (`bim.add_annotation`)

```python
class AddAnnotation(bpy.types.Operator):
    bl_idname = "bim.add_annotation"
    bl_label = "Add Annotation"

    object_type: bpy.props.StringProperty()
    # Annotation type name (TEXT, DIMENSION, etc.)
    relating_type_id: bpy.props.IntProperty(default=0)
    # IFC entity ID of annotation type template (0 = no type)
```

### AddAnnotationType (`bim.add_annotation_type`)

```python
class AddAnnotationType(bpy.types.Operator):
    bl_idname = "bim.add_annotation_type"
    bl_label = "Add Annotation Type"
    # Creates a reusable annotation type template in the IFC model
```

### AddSheet (`bim.add_sheet`)

```python
class AddSheet(bpy.types.Operator):
    bl_idname = "bim.add_sheet"
    bl_label = "Add Sheet"
    # Creates IfcDocumentInformation + IfcDocumentReference
    # Generates base SVG layout from titleblock template
```

### CreateSheets (`bim.create_sheets`)

```python
class CreateSheets(bpy.types.Operator):
    bl_idname = "bim.create_sheets"
    bl_label = "Create Sheets"
    # Composes all drawings into sheet SVG
    # Exports to PDF/DXF (requires InkScape)
```

### AddDrawingToSheet (`bim.add_drawing_to_sheet`)

```python
class AddDrawingToSheet(bpy.types.Operator):
    bl_idname = "bim.add_drawing_to_sheet"
    bl_label = "Add Drawing To Sheet"
    # Links a drawing IfcDocumentReference to the active sheet
```

### OpenSheet (`bim.open_sheet`)

```python
class OpenSheet(bpy.types.Operator):
    bl_idname = "bim.open_sheet"
    bl_label = "Open Sheet"
    # Opens the generated sheet with system default viewer
```

### OpenDrawing (`bim.open_drawing`)

```python
class OpenDrawing(bpy.types.Operator):
    bl_idname = "bim.open_drawing"
    bl_label = "Open Drawing"
    # Opens the generated SVG with system default viewer
```

### OpenLayout (`bim.open_layout`)

```python
class OpenLayout(bpy.types.Operator):
    bl_idname = "bim.open_layout"
    bl_label = "Open Layout"
    # Opens the SVG layout file with system default viewer
```

### ActivateModel (`bim.activate_model`)

```python
class ActivateModel(bpy.types.Operator):
    bl_idname = "bim.activate_model"
    bl_label = "Activate Model"
    # Switches from drawing view to 3D model view
    # Shows all objects, hides annotation objects
```

### SelectAllDrawings (`bim.select_all_drawings`)

```python
class SelectAllDrawings(bpy.types.Operator):
    bl_idname = "bim.select_all_drawings"
    bl_label = "Select All Drawings"
    # Batch-selects drawings (Shift modifier toggles selection)
```

### SelectAllSheets (`bim.select_all_sheets`)

```python
class SelectAllSheets(bpy.types.Operator):
    bl_idname = "bim.select_all_sheets"
    bl_label = "Select All Sheets"
    # Batch-selects sheets (Shift modifier toggles selection)
```

## 3. Property Groups (`bonsai/bim/module/drawing/prop.py`)

### BIMCameraProperties

```python
class BIMCameraProperties(bpy.types.PropertyGroup):
    # Attached to camera data blocks

    # Target view type
    target_view: EnumProperty  # PLAN_VIEW, SECTION_VIEW, etc.

    # Scale
    diagram_scale: EnumProperty
    # Metric: "1:5000" through "1:1", plus "CUSTOM"
    # Imperial: equivalent range with ft/in notation
    custom_scale_numerator: IntProperty
    custom_scale_denominator: IntProperty
    is_nts: BoolProperty  # Not To Scale flag

    # Camera type
    camera_type: EnumProperty  # ORTHO | PERSP

    # Raster settings
    raster_x: IntProperty(default=1000)
    raster_y: IntProperty(default=1000)
    dpi: IntProperty(default=75)

    # View dimensions
    width: FloatProperty(default=50.0)  # meters
    height: FloatProperty(default=50.0)  # meters

    # Layer toggles
    has_underlay: BoolProperty
    has_linework: BoolProperty
    has_annotation: BoolProperty

    # Generation modes
    linework_mode: EnumProperty  # OPENCASCADE | FREESTYLE
    fill_mode: EnumProperty  # NONE | SHAPELY | SVGFILL
    cut_mode: EnumProperty  # BISECT | OPENCASCADE

    # Filters
    filter_mode: EnumProperty  # NONE | include/exclude
    filter_groups: CollectionProperty
```

### BIMAnnotationProperties

```python
class BIMAnnotationProperties(bpy.types.PropertyGroup):
    object_type: EnumProperty
    # TEXT, TEXT_LEADER, DIMENSION, ANGLE, SECTION_LEVEL,
    # PLAN_LEVEL, ELEVATION, GRID, FILL_AREA, STAIR_ARROW,
    # HIDDEN_LINE, BREAKLINE

    relating_type_id: IntProperty  # IFC entity ID of type
```

### BIMTextProperties

```python
class BIMTextProperties(bpy.types.PropertyGroup):
    font_size: EnumProperty
    # Options: "1.8", "2.5", "3.5", "5.0", "7.0" (millimeters)

    # Alignment
    # Horizontal: left, middle, right
    # Vertical: top, middle, bottom

    # Symbol
    symbol: EnumProperty
    # NO SYMBOL, CUSTOM SYMBOL, or predefined symbols

    # Literals (CollectionProperty)
    # 16 categories: Basic, Attributes, Property Sets,
    # Material, Task, Cost, etc.
```

### DrawingStyle

```python
class DrawingStyle(bpy.types.PropertyGroup):
    render_type: EnumProperty  # DEFAULT | VIEWPORT
    raster_style: StringProperty  # JSON-formatted render config
```

## 4. SVG Engine (`bonsai/bim/module/drawing/svgwriter.py`)

### SvgWriter

```python
class SvgWriter:
    """Central SVG generation engine for Bonsai drawings."""

    def project_point_onto_camera(self, point: Vector) -> Vector:
        """Projects 3D world coordinate to camera-space via plane intersection."""

    # Coordinate pipeline:
    # 1. 3D world → camera-space (project_point_onto_camera)
    # 2. Camera-space → SVG canvas (offset + scale by diagram_scale × 1000)
    # 3. Units: IFC meters → SVG millimeters
```

## 5. Sheet Engine (`bonsai/bim/module/drawing/sheeter.py`)

### SheetBuilder

```python
class SheetBuilder:
    """Composes multiple drawings into printable sheet layouts."""

    def build(self) -> None:
        """Three-phase composition:
        1. Titleblock Processing — embed titleblock SVG, apply geolocation
        2. Drawing Integration — embed SVGs with style isolation
           (ensure_drawing_unique_styles prefixes CSS classes)
        3. Document Assembly — incorporate schedules, resolve templates
        """

    def next_drawing_location(self) -> tuple[float, float]:
        """Auto-layout positioning.
        Start: (30, 30) mm
        Padding: 10 mm between drawings
        Horizontal rows; new row at titleblock width (default 840 mm)
        """

    def ensure_drawing_unique_styles(self, svg_element, drawing_id: str) -> None:
        """Prefixes all CSS class names with drawing_id to prevent
        style conflicts when embedding multiple drawings."""

    def convert_to_mm(self, value: str) -> float:
        """Converts CSS length units (cm, in, pt, px, Q, pc) to mm."""
```

## 6. Annotation Engine (`bonsai/bim/module/drawing/annotation.py`)

### Annotator

```python
class Annotator:
    """Creates Blender geometry for IFC annotations."""

    # Data type mapping:
    # Mesh (bpy.types.Mesh)  → vertex-based annotations (points, lines)
    # Curve (bpy.types.Curve) → leader lines, polylines (resolution_u=2, 3D curves)
    # Empty (bpy.types.Object EMPTY) → lightweight reference markers
```

## 7. Helper Utilities (`bonsai/bim/module/drawing/helper.py`)

```python
# Key utility functions:
def format_distance(value: float, precision: int = 4) -> str:
    """Formats metric/imperial distance for display."""

class BoundingBox:
    """Axis-aligned bounding box calculations for drawing elements."""
```
