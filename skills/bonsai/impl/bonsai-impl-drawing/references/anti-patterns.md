# Drawing Anti-Patterns

> **Version**: Bonsai v0.8.x | Module: `bonsai.bim.module.drawing`

## Anti-Pattern 1: Generating Drawings Without an Active Camera

**Wrong**:
```python
# WRONG — Calling create_drawing without an active orthographic camera
bpy.ops.bim.create_drawing(sync=True)
# Result: Fails or produces empty/corrupt SVG
```

**Correct**:
```python
# CORRECT — ALWAYS activate the drawing view first
# Ensure the drawing camera is selected and active in viewport
bpy.ops.bim.open_drawing()  # Activates the drawing's camera view
bpy.ops.bim.create_drawing(sync=True)
```

**Why**: `CreateDrawing` reads camera properties (scale, target_view, dimensions) from the active camera object. Without an active orthographic camera, the operator has no context for SVG generation and produces invalid output.

---

## Anti-Pattern 2: Using FREESTYLE Mode Without the Add-on

**Wrong**:
```python
# WRONG — Setting FREESTYLE linework mode without checking add-on
camera = bpy.context.scene.camera
camera.data.BIMCameraProperties.linework_mode = "FREESTYLE"
bpy.ops.bim.create_drawing(sync=True)
# Result: Linework generation fails silently or produces empty vector layer
```

**Correct**:
```python
# CORRECT — ALWAYS verify the Freestyle SVG Exporter add-on is active
import addon_utils

# Check if Freestyle SVG Exporter is enabled
is_enabled = addon_utils.check("render_freestyle_svg")[1]
if not is_enabled:
    addon_utils.enable("render_freestyle_svg")

camera = bpy.context.scene.camera
camera.data.BIMCameraProperties.linework_mode = "FREESTYLE"
bpy.ops.bim.create_drawing(sync=True)
```

**Why**: The FREESTYLE linework mode delegates SVG line generation to Blender's Freestyle SVG Exporter add-on. If the add-on is not enabled in Blender preferences, the linework layer will be empty or the operator will fail.

---

## Anti-Pattern 3: Manually Editing Generated SVG Files

**Wrong**:
```python
# WRONG — Editing SVG files directly with text manipulation
import xml.etree.ElementTree as ET

tree = ET.parse("/path/to/drawing.svg")
root = tree.getroot()
# Manually adding elements to the SVG...
text_element = ET.SubElement(root, "text")
text_element.text = "Manual Label"
tree.write("/path/to/drawing.svg")
# Result: Changes are LOST on next bim.create_drawing call
```

**Correct**:
```python
# CORRECT — ALWAYS modify annotations in Blender, then regenerate
bpy.ops.bim.add_annotation(object_type="TEXT")
# Position and edit the text annotation in Blender's viewport
bpy.ops.bim.create_drawing(sync=True)
# SVG is regenerated with all annotations from the IFC model
```

**Why**: `bim.create_drawing` regenerates the entire SVG from scratch each time. Any manual edits to the SVG file are overwritten. ALWAYS make changes through Blender annotations (which are stored as IFC entities) and regenerate the SVG.

---

## Anti-Pattern 4: Expecting Annotations in Perspective Views

**Wrong**:
```python
# WRONG — Adding annotations to a perspective (MODEL_VIEW) drawing
bpy.ops.bim.add_drawing(target_view="MODEL_VIEW", location_hint=0)
bpy.ops.bim.add_annotation(object_type="DIMENSION")
bpy.ops.bim.create_drawing(sync=True)
# Result: Annotations are NOT generated in the SVG output
```

**Correct**:
```python
# CORRECT — Use orthographic views for annotated drawings
bpy.ops.bim.add_drawing(target_view="PLAN_VIEW", location_hint=0)
bpy.ops.bim.add_annotation(object_type="DIMENSION")
bpy.ops.bim.create_drawing(sync=True)
# Result: Annotations correctly projected and rendered in SVG
```

**Why**: The SvgWriter annotation pipeline is designed for orthographic projections only. Perspective views (`MODEL_VIEW` with `camera_type=PERSP`) do not support annotation rendering because 2D annotation projection requires a parallel projection matrix. Use `PLAN_VIEW`, `SECTION_VIEW`, `ELEVATION_VIEW`, or `REFLECTED_PLAN_VIEW` for annotated drawings.

---

## Anti-Pattern 5: Not Syncing IFC Data Before Drawing Generation

**Wrong**:
```python
# WRONG — Generating drawing without syncing IFC changes
# (User has made changes to the model but not saved/synced)
bpy.ops.bim.create_drawing(sync=False)
# Result: Drawing shows stale/outdated model state
```

**Correct**:
```python
# CORRECT — ALWAYS sync IFC data before generating drawings
bpy.ops.bim.create_drawing(sync=True)
# Or save the IFC file first:
# bpy.ops.bim.save_project()
# bpy.ops.bim.create_drawing()
```

**Why**: The drawing pipeline reads geometry and properties from the IFC model. If Blender-side changes have not been synced to the IFC store, the generated SVG reflects the last-synced state, not the current model. ALWAYS use `sync=True` or save the project before generating drawings.

---

## Anti-Pattern 6: Creating Sheets Without InkScape Installed

**Wrong**:
```python
# WRONG — Attempting sheet composition/PDF export without InkScape
bpy.ops.bim.add_sheet()
bpy.ops.bim.add_drawing_to_sheet()
bpy.ops.bim.create_sheets()
# Result: Sheet composition fails; PDF/DXF export is impossible
```

**Correct**:
```python
# CORRECT — ALWAYS ensure InkScape is installed before sheet operations
import shutil
inkscape_path = shutil.which("inkscape")
if inkscape_path is None:
    raise RuntimeError(
        "InkScape is required for sheet composition and PDF export. "
        "Install from https://inkscape.org/"
    )

bpy.ops.bim.add_sheet()
bpy.ops.bim.add_drawing_to_sheet()
bpy.ops.bim.create_sheets()
```

**Why**: The `SheetBuilder` (`sheeter.py`) relies on InkScape for SVG composition and PDF/DXF export. Without InkScape on the system PATH, `create_sheets` either fails or produces incomplete output. ALWAYS install InkScape as a project dependency.

---

## Anti-Pattern 7: Using Absolute Paths for Titleblock Templates

**Wrong**:
```python
# WRONG — Hardcoded absolute path to titleblock SVG
titleblock_path = "C:/Users/architect/templates/titleblock_A1.svg"
# Or: "/home/architect/templates/titleblock_A1.svg"
# Result: Breaks on other machines, fails in collaborative environments
```

**Correct**:
```python
# CORRECT — ALWAYS use project-relative paths for titleblock resources
# Titleblock SVGs should be stored within the project directory
# Bonsai resolves paths relative to the IFC project location
```

**Why**: Absolute paths break portability. When the project is shared with collaborators or moved to a different machine, absolute paths to titleblock SVGs become invalid. Bonsai's sheet system expects titleblock resources to be locatable relative to the project. ALWAYS use project-relative paths.

---

## Anti-Pattern 8: Using Legacy `blenderbim` Module Paths

**Wrong**:
```python
# WRONG — Using the legacy BlenderBIM module path
from blenderbim.bim.module.drawing import operator
# Or:
import blenderbim.bim.module.drawing.svgwriter as svgwriter
# Result: ImportError in Bonsai v0.8.x
```

**Correct**:
```python
# CORRECT — ALWAYS use the bonsai module path
from bonsai.bim.module.drawing import operator
# Or:
import bonsai.bim.module.drawing.svgwriter as svgwriter
```

**Why**: Bonsai was renamed from BlenderBIM. Since Bonsai v0.8.0, the module path is `bonsai.bim.module.*`. Using the legacy `blenderbim.bim.module.*` path causes `ImportError` on current versions. ALWAYS use `bonsai.bim.module.drawing`.

---

## Summary Table

| # | Anti-Pattern | Rule |
|---|-------------|------|
| 1 | Generate drawing without active camera | ALWAYS activate drawing camera first |
| 2 | FREESTYLE mode without add-on enabled | ALWAYS verify Freestyle SVG Exporter is active |
| 3 | Manually edit generated SVG files | NEVER edit SVGs; ALWAYS modify annotations and regenerate |
| 4 | Annotations in perspective views | ALWAYS use orthographic views for annotated drawings |
| 5 | Skip IFC sync before drawing | ALWAYS use `sync=True` or save project first |
| 6 | Sheet creation without InkScape | ALWAYS install InkScape for sheet composition |
| 7 | Absolute paths for titleblocks | ALWAYS use project-relative paths |
| 8 | Legacy `blenderbim` module path | ALWAYS use `bonsai.bim.module.drawing` |
