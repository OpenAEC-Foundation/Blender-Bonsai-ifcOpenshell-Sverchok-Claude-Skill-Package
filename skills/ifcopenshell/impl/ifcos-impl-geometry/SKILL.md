---
name: ifcos-impl-geometry
description: 'Covers IfcOpenShell geometry processing including geometry settings, create_shape() for mesh extraction, geometry iterator for batch processing, representation creation, extrusion/CSG/BRep geometry, and coordinate transformations. Activates when extracting 3D geometry from IFC files, creating geometric representations, or processing IFC geometry for visualization.'
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Geometry Processing & Creation

## Quick Reference

### Decision Tree: Reading vs Creating Geometry

```
Need geometry from IFC elements?
├── Extract existing geometry? (read/process)
│   ├── Single element → ifcopenshell.geom.create_shape(settings, element)
│   ├── Multiple elements (100+) → ifcopenshell.geom.iterator(settings, model, cpu_count)
│   └── Specific entity attributes → Manual traversal (element.Representation)
│
└── Create new geometry? (write/author)
    ├── Simple wall block → geometry.add_wall_representation()
    ├── Extruded profile (beam/column) → geometry.add_profile_representation()
    ├── Arbitrary mesh (furniture/equipment) → geometry.add_mesh_representation()
    ├── Boolean operation (opening/cut) → geometry.add_boolean()
    ├── Custom parametric shape → ShapeBuilder + builder.get_representation()
    └── Slab/door/window/railing → geometry.add_{type}_representation()
```

### Decision Tree: Tessellation vs BRep

```
What do you need the geometry for?
├── Visualization / rendering / game engine / export to mesh format
│   └── Tessellation (default) — triangulated mesh output
│       settings.set(settings.USE_BREP_DATA, False)  # default
│
├── CAD operations / boolean operations / precise measurements
│   └── BRep — exact OpenCASCADE TopoDS_Shape
│       settings.set(settings.USE_BREP_DATA, True)
│       Requires: pythonOCC (PythonOCC-Core) for advanced operations
│
└── Geometry export to glTF/OBJ
    └── Use serializer (ifcopenshell.geom.serializers)
```

### Decision Tree: Local vs World Coordinates

```
What coordinate space?
├── Need absolute position in the model
│   └── USE_WORLD_COORDS = True
│       Vertices include full placement chain (element → storey → building → site)
│
├── Need position relative to element origin
│   └── USE_WORLD_COORDS = False (default)
│       Apply shape.transformation.matrix manually if needed
│
└── Need to compare positions across elements
    └── USE_WORLD_COORDS = True (ALWAYS for cross-element comparison)
```

### Critical Warnings

- **ALWAYS** wrap `create_shape()` in try/except RuntimeError. Not all IfcProduct subtypes have geometry (IfcProject, IfcBuildingStorey, some IfcSite).
- **ALWAYS** use the iterator for processing 100+ elements. It is 5-10x faster than calling `create_shape()` in a loop due to internal caching and multithreading.
- **ALWAYS** set up a representation context before creating geometry. Call `context.add_context()` for the root 3D context, then a subcontext for Body/MODEL_VIEW.
- **ALWAYS** call `geometry.edit_object_placement()` on elements that have geometry. Elements without a placement are positioned at the global origin.
- **ALWAYS** apply unit scale when reading raw coordinate values from IFC entities. Use `ifcopenshell.util.unit.calculate_unit_scale(model)` to get the conversion factor to meters.
- **NEVER** assume geometry coordinates are in meters. Check the project units first.
- **NEVER** access `shape.geometry.verts` without checking that `create_shape()` did not raise an exception.
- **NEVER** create geometry representations without first having a context (IfcGeometricRepresentationSubContext with identifier="Body").
- **NEVER** use `model.create_entity("IfcExtrudedAreaSolid", ...)` for production code when `geometry.add_profile_representation()` or `geometry.add_wall_representation()` is available. The API functions handle context assignment, representation types, and validation automatically.

---

## Essential Patterns

### Pattern 1: Configure Geometry Settings

```python
# IfcOpenShell — all schema versions
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()

# Most common configuration for mesh extraction:
settings.set(settings.USE_WORLD_COORDS, True)   # Absolute coordinates
settings.set(settings.WELD_VERTICES, True)       # Clean mesh
settings.set(settings.USE_BREP_DATA, False)       # Triangulated output

# For BRep/CAD operations:
settings_brep = ifcopenshell.geom.settings()
settings_brep.set(settings_brep.USE_BREP_DATA, True)
settings_brep.set(settings_brep.USE_WORLD_COORDS, True)

# For fast preview (skip openings):
settings_fast = ifcopenshell.geom.settings()
settings_fast.set(settings_fast.DISABLE_OPENING_SUBTRACTIONS, True)
settings_fast.set(settings_fast.USE_WORLD_COORDS, True)
```

### Pattern 2: Extract Single Element Geometry

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import numpy as np

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

wall = model.by_type("IfcWall")[0]

try:
    shape = ifcopenshell.geom.create_shape(settings, wall)
except RuntimeError:
    print("No geometry or processing failed")
    shape = None

if shape:
    verts = np.array(shape.geometry.verts).reshape(-1, 3)
    faces = np.array(shape.geometry.faces).reshape(-1, 3)
    edges = np.array(shape.geometry.edges).reshape(-1, 2)
    print(f"Vertices: {len(verts)}, Triangles: {len(faces)}")
```

### Pattern 3: Batch Process with Iterator

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing
import numpy as np

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count())

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        verts = np.array(shape.geometry.verts).reshape(-1, 3)
        faces = np.array(shape.geometry.faces).reshape(-1, 3)
        # ... process geometry ...
        if not iterator.next():
            break
```

### Pattern 4: Filtered Iterator

```python
# IfcOpenShell — all schema versions
# Process only walls
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(),
    include=model.by_type("IfcWall"))

# OR exclude spaces (invisible elements)
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(),
    exclude=model.by_type("IfcSpace"))

if iterator.initialize():
    while True:
        shape = iterator.get()
        # ... process ...
        if not iterator.next():
            break
```

### Pattern 5: Create Representation Context (Required Before Any Geometry Creation)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Step 1: Root 3D context (REQUIRED)
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")

# Step 2: Body subcontext (REQUIRED for geometry)
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)
```

### Pattern 6: Create Wall Geometry

```python
# IfcOpenShell — all schema versions
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="W-01")

# Parametric wall: length=5m, height=3m, thickness=0.2m
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)

# Set placement (identity = origin, or use a 4x4 matrix)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall)
```

### Pattern 7: Create Profile Extrusion (Beam/Column)

```python
# IfcOpenShell — all schema versions
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="C-01")

# Create a rectangle profile
profile = model.create_entity("IfcRectangleProfileDef",
    ProfileType="AREA", XDim=0.3, YDim=0.3)

# Extrude the profile
col_rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=col_rep)

ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=column)
```

### Pattern 8: Create Mesh Geometry (Arbitrary Shape)

```python
# IfcOpenShell — all schema versions
element = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcFurniture", name="Table")

vertices = [[0,0,0],[1,0,0],[1,1,0],[0,1,0],
            [0,0,0.8],[1,0,0.8],[1,1,0.8],[0,1,0.8]]
faces = [[0,1,2,3],[4,7,6,5],[0,4,5,1],[1,5,6,2],[2,6,7,3],[3,7,4,0]]

mesh_rep = ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body, vertices=vertices, faces=faces)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=mesh_rep)

ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=element)
```

### Pattern 9: Object Placement with Transformation Matrix

```python
# IfcOpenShell — all schema versions
import numpy as np
import ifcopenshell.util.placement

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Rotated Wall")

# Start with identity matrix
matrix = np.eye(4)

# Rotate 90 degrees around Z axis
matrix = ifcopenshell.util.placement.rotation(90, "Z") @ matrix

# Set position: X=2, Y=3, Z=0
matrix[:, 3][0:3] = (2.0, 3.0, 0.0)

ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix, is_si=True)
```

### Pattern 10: Boolean Operations (CSG)

```python
# IfcOpenShell — all schema versions
# Create a wall with an opening (boolean subtraction)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall with Opening")

wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)

# Create an opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

opening_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=0.9, height=2.1, thickness=0.2)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_rep)

# Create void relationship (boolean subtraction)
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)
```

---

## Common Operations

For detailed code examples of these operations, see [Working Code Examples](references/examples.md):

- **Transformation matrix** — Extract and convert 4x3 column-major matrix from processed shapes
- **Material/style access** — Read diffuse color, transparency from shape.geometry.materials
- **ShapeBuilder** — Create custom geometry with rectangle/circle/polyline profiles + extrusion
- **glTF serialization** — Export IFC geometry to glTF/GLB using iterator + serializer
- **Shape helpers** — Use `ifcopenshell.util.shape` for grouped vertex/edge/face arrays

For complete settings and data structure tables, see [API Method Signatures](references/methods.md).

---

## Version Notes

- Geometry processing via `ifcopenshell.geom` is schema-agnostic. The same `create_shape()` and `iterator` calls work for IFC2X3, IFC4, and IFC4X3 files.
- Geometry creation via `ifcopenshell.api.geometry.*` is also schema-agnostic — the API handles schema differences internally.
- The `ShapeBuilder` utility (`ifcopenshell.util.shape_builder`) works across all schema versions.
- IFC4X3 adds alignment-based geometry (IfcLinearPlacement, IfcAlignment) not covered by the standard geometry API functions.
- BRep serialization to IFC depends on schema capabilities: IFC4+ supports more complex curved surfaces than IFC2X3.

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all geometry methods
- [Working Code Examples](references/examples.md) — End-to-end examples for geometry operations
- [Anti-Patterns](references/anti-patterns.md) — Common geometry mistakes and how to avoid them
