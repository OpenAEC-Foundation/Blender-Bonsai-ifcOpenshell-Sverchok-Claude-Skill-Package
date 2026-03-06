# Project Management Anti-Patterns

> **Version**: IfcOpenShell v0.8.x | Bonsai v0.8.x | IFC2X3 / IFC4 / IFC4X3

## Anti-Pattern 1: Using ifcopenshell.file() Instead of create_file()

**Severity**: CRITICAL — produces invalid IFC files

**Wrong**:
```python
import ifcopenshell

# Low-level constructor — creates EMPTY file with NO IfcProject, NO units, NO contexts
model = ifcopenshell.file(schema="IFC4")

# All subsequent operations fail or produce invalid IFC:
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
# No IfcProject → validator error
# No IfcUnitAssignment → dimensions are meaningless
# No representation context → geometry cannot be assigned
```

**Correct**:
```python
import ifcopenshell
import ifcopenshell.api

# High-level API — creates IfcProject, default units, representation contexts
model = ifcopenshell.api.project.create_file(version="IFC4")
```

**Why**: `ifcopenshell.file()` is a low-level constructor intended for parsing existing files. `create_file()` bootstraps all required project entities. Every IFC schema requires IfcProject with IfcUnitAssignment as a minimum.

---

## Anti-Pattern 2: Skipping Unit Assignment

**Severity**: CRITICAL — all dimensions become ambiguous

**Wrong**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")
# Skip units, go straight to creating elements
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
# Is wall height 3.0 metres? 3.0 millimetres? 3.0 feet? Nobody knows.
```

**Correct**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")

# ALWAYS assign units immediately after project creation
ifcopenshell.api.unit.assign_unit(model)  # Default: mm, m², m³

# Now create elements — dimensions are unambiguous
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
```

**Why**: Without an explicit IfcUnitAssignment, receiving applications cannot interpret numeric values. Some viewers assume metres, others assume millimetres. This causes buildings to appear 1000x too large or too small.

---

## Anti-Pattern 3: Editing Georeferencing Before Adding It

**Severity**: HIGH — raises runtime error

**Wrong**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")

# Trying to edit georeferencing that doesn't exist yet
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={"Eastings": 155000.0, "Northings": 463000.0})
# ERROR: No IfcMapConversion or IfcProjectedCRS entities found
```

**Correct**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")

# Step 1: Create the georeferencing entities FIRST
ifcopenshell.api.georeference.add_georeferencing(model)

# Step 2: THEN edit them
ifcopenshell.api.georeference.edit_georeferencing(model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={"Eastings": 155000.0, "Northings": 463000.0})
```

**Why**: `edit_georeferencing` expects existing `IfcMapConversion` and `IfcProjectedCRS` entities. The `add_georeferencing` call creates the empty entities that `edit_georeferencing` then populates.

---

## Anti-Pattern 4: Using IfcMapConversionScaled in IFC4

**Severity**: HIGH — creates schema-invalid file

**Wrong**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")

# IfcMapConversionScaled is IFC4X3 ONLY
ifcopenshell.api.georeference.add_georeferencing(model,
    ifc_class="IfcMapConversionScaled")
# ERROR or silently invalid: IfcMapConversionScaled does not exist in IFC4 schema
```

**Correct**:
```python
# For IFC4: use IfcMapConversion (default)
model_ifc4 = ifcopenshell.api.project.create_file(version="IFC4")
ifcopenshell.api.georeference.add_georeferencing(model_ifc4)  # default = IfcMapConversion

# For IFC4X3: IfcMapConversionScaled is allowed
model_4x3 = ifcopenshell.api.project.create_file(version="IFC4X3")
ifcopenshell.api.georeference.add_georeferencing(model_4x3,
    ifc_class="IfcMapConversionScaled")
```

**Why**: `IfcMapConversionScaled` was introduced in IFC4X3 to support grid scale factors for infrastructure. It does not exist in IFC4 or IFC2X3 schemas.

---

## Anti-Pattern 5: Confusing Project North with True North

**Severity**: MEDIUM — causes incorrect solar analysis and site orientation

**Wrong**:
```python
from math import cos, sin, radians

# Setting Project North rotation in edit_georeferencing
# AND setting the same angle as true_north
ifcopenshell.api.georeference.edit_georeferencing(model,
    coordinate_operation={
        "XAxisAbscissa": cos(radians(-5.0)),   # 5° Project North rotation
        "XAxisOrdinate": sin(radians(-5.0)),
    })

ifcopenshell.api.georeference.edit_true_north(model,
    true_north=-5.0)  # WRONG: this doubles the rotation
```

**Correct**:
```python
from math import cos, sin, radians

# Project North: rotation from Grid North to Project North
# (how the building grid relates to the survey grid)
ifcopenshell.api.georeference.edit_georeferencing(model,
    coordinate_operation={
        "XAxisAbscissa": cos(radians(-5.0)),   # 5° rotation
        "XAxisOrdinate": sin(radians(-5.0)),
    })

# True North: rotation from Project North to Geographic North
# (for solar analysis — separate from Project North)
ifcopenshell.api.georeference.edit_true_north(model,
    true_north=3.0)  # Different value — this is the magnetic/true declination
```

**Why**: Project North (in IfcMapConversion) defines how the local model grid relates to the survey coordinate system. True North (on IfcGeometricRepresentationContext) defines geographic north for solar/wind analysis. These are independent values.

---

## Anti-Pattern 6: Describing Bonsai Workflow as Import/Export

**Severity**: MEDIUM — leads to incorrect workflow assumptions

**Wrong**:
```python
# "Export to IFC" mindset — WRONG for Bonsai
bpy.ops.export_scene.ifc(filepath="output.ifc")  # Does NOT exist in Bonsai
# or
bpy.ops.import_scene.ifc(filepath="input.ifc")    # Does NOT exist in Bonsai
```

**Correct**:
```python
# Bonsai uses NATIVE IFC — the .ifc file IS the project document
bpy.ops.bim.load_project(filepath="project.ifc")   # Open (not import)
bpy.ops.bim.save_project(filepath="project.ifc")   # Save (not export)
```

**Why**: Bonsai is a native IFC authoring tool. The IFC file is the primary document, not a derivative format. There is no translation/conversion step. Thinking in import/export terms leads to workflows where the `.blend` file is treated as primary and IFC as secondary — this is the opposite of how Bonsai works.

---

## Anti-Pattern 7: Missing Spatial Hierarchy

**Severity**: HIGH — most IFC validators reject files without spatial structure

**Wrong**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")
ifcopenshell.api.unit.assign_unit(model)

# Skip spatial hierarchy, directly create elements
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
# Wall has no spatial container — invalid for most use cases
```

**Correct**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")
ifcopenshell.api.unit.assign_unit(model)

# Create required spatial hierarchy
project = model.by_type("IfcProject")[0]
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey])

# NOW create elements in a spatial container
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
ifcopenshell.api.spatial.assign_container(model,
    relating_structure=storey, products=[wall])
```

**Why**: IFC requires: IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey. Elements without spatial containment are "orphans" that most viewers hide or reject.

---

## Anti-Pattern 8: Missing Representation Context Before Adding Geometry

**Severity**: HIGH — geometry assignment fails

**Wrong**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")

# Try to add geometry without verifying/creating context
ifcopenshell.api.geometry.add_wall_representation(model,
    context=None, ...)
# ERROR: No valid IfcGeometricRepresentationContext
```

**Correct**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")

# Create or verify representation contexts BEFORE adding geometry
model_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model")
body_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=model_ctx)

# NOW geometry can be added with a valid context reference
```

**Why**: Every IfcShapeRepresentation requires a reference to an IfcGeometricRepresentationContext or IfcGeometricRepresentationSubContext. Without it, geometry has no coordinate system or display purpose.

---

## Anti-Pattern 9: Using blenderbim.* Module Path

**Severity**: HIGH — ModuleNotFoundError in Bonsai v0.8.0+

**Wrong**:
```python
from blenderbim.bim.ifc import IfcStore       # ModuleNotFoundError
import blenderbim.tool as tool                  # ModuleNotFoundError
from blenderbim.core import project             # ModuleNotFoundError
```

**Correct**:
```python
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
from bonsai.core import project
```

**Why**: Bonsai was renamed from BlenderBIM in 2024 (v0.8.0). All module paths changed from `blenderbim.*` to `bonsai.*`. Old code and tutorials using the old path will fail.

---

## Anti-Pattern 10: Moving WCS Without Survey Justification

**Severity**: MEDIUM — causes coordinate confusion across all model elements

**Wrong**:
```python
# Arbitrarily moving WCS for "convenience"
ifcopenshell.api.georeference.edit_wcs(model,
    x=100.0, y=200.0, z=0.0)
# ALL local placements are now offset — coordinates become confusing
```

**Correct**:
```python
# Keep WCS at origin unless survey data explicitly requires an offset
ifcopenshell.api.georeference.edit_wcs(model,
    x=0.0, y=0.0, z=0.0, rotation=0.0, is_si=True)

# Use IfcMapConversion to handle the real-world coordinate offset instead
ifcopenshell.api.georeference.edit_georeferencing(model,
    coordinate_operation={
        "Eastings": 155000.0,    # Real-world offset goes HERE
        "Northings": 463000.0,
    })
```

**Why**: The WCS origin affects every `IfcLocalPlacement` in the model. Moving WCS shifts the entire local coordinate system, making element coordinates non-intuitive. Use `IfcMapConversion` for real-world offset — that is its purpose.

---

## Anti-Pattern 11: Running Bonsai Scripts Outside Blender

**Severity**: CRITICAL — immediate ImportError

**Wrong**:
```bash
$ python3 my_bonsai_script.py
# ImportError: No module named 'bpy'
```

**Correct**:
```bash
$ blender --background --python my_bonsai_script.py
```

**Why**: Bonsai depends on `bpy` (Blender Python API) which is only available inside Blender's bundled Python. System Python cannot import `bpy` or `bonsai.*`.

**Note**: Standalone IfcOpenShell scripts (without `bpy` or `bonsai.*` imports) CAN run in system Python. Only use `blender --python` when your script needs Bonsai or Blender APIs.
