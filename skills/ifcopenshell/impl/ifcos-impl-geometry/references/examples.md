# ifcos-impl-geometry — Working Code Examples

## Access Transformation Matrix from Processed Shape

```python
# IfcOpenShell — all schema versions
import ifcopenshell.geom
import numpy as np

settings = ifcopenshell.geom.settings()
shape = ifcopenshell.geom.create_shape(settings, element)
matrix_data = shape.transformation.matrix.data
# matrix_data is a flat list of 12 values (4x3 column-major)
mat4x3 = np.array(matrix_data).reshape(4, 3)
# Convert 4x3 to 4x4:
full_matrix = np.eye(4)
full_matrix[:4, :3] = mat4x3
```

---

## Access Material/Style Information

```python
# IfcOpenShell — all schema versions
shape = ifcopenshell.geom.create_shape(settings, element)
materials = shape.geometry.materials
material_ids = shape.geometry.material_ids

for i, mat in enumerate(materials):
    print(f"Material {i}: {mat.name}")
    if mat.has_diffuse:
        r, g, b = mat.diffuse
        print(f"  Color: ({r}, {g}, {b})")
    if mat.has_transparency:
        print(f"  Transparency: {mat.transparency}")
```

---

## Use ShapeBuilder for Custom Geometry

```python
# IfcOpenShell — all schema versions
from ifcopenshell.util.shape_builder import V, ShapeBuilder
import ifcopenshell.util.representation

builder = ShapeBuilder(model)

# Create a rectangle profile and extrude it
rect = builder.rectangle(size=V(1.0, 0.5))
profile = builder.profile(rect)
extrusion = builder.extrude(profile, 2.0)

# Get representation context
body = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

representation = builder.get_representation(context=body, items=[extrusion])

ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=representation)
```

---

## Serialize Geometry to glTF

```python
# IfcOpenShell — all schema versions
import ifcopenshell.geom
import multiprocessing

settings = ifcopenshell.geom.settings()
settings.set("dimensionality",
    ifcopenshell.ifcopenshell_wrapper.CURVES_SURFACES_AND_SOLIDS)
settings.set("apply-default-materials", True)

serialiser_settings = ifcopenshell.geom.serializer_settings()
serialiser_settings.set("use-element-guids", True)

serialiser = ifcopenshell.geom.serializers.gltf(
    "output.glb", settings, serialiser_settings)

iterator = ifcopenshell.geom.iterator(settings, model,
    multiprocessing.cpu_count())

serialiser.setFile(model)
serialiser.setUnitNameAndMagnitude("METER", 1.0)
serialiser.writeHeader()

if iterator.initialize():
    while True:
        serialiser.write(iterator.get())
        if not iterator.next():
            break

serialiser.finalize()
```

---

## Get Shape Helper Utilities

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.shape

shape = ifcopenshell.geom.create_shape(settings, element)

# Get transformation as nested numpy array
matrix = ifcopenshell.util.shape.get_shape_matrix(shape)

# Get grouped geometry data (not flat arrays)
vertices = ifcopenshell.util.shape.get_vertices(shape.geometry)
# Returns: [[x1,y1,z1], [x2,y2,z2], ...]

edges = ifcopenshell.util.shape.get_edges(shape.geometry)
# Returns: [[i1,i2], [i3,i4], ...]

faces = ifcopenshell.util.shape.get_faces(shape.geometry)
# Returns: [[i1,i2,i3], [i4,i5,i6], ...]
```

---

## Settings Reference

| Setting Constant | Default | Description |
|---|---|---|
| `USE_WORLD_COORDS` | False | Apply full placement chain for world-space coordinates |
| `USE_BREP_DATA` | False | Return OpenCASCADE BRep instead of tessellated mesh |
| `WELD_VERTICES` | True | Merge coincident vertices |
| `APPLY_DEFAULT_MATERIALS` | True | Include material/style color information |
| `DISABLE_TRIANGULATION` | False | Skip tessellation (use with USE_BREP_DATA) |
| `APPLY_LAYERSETS` | False | Apply material layer set geometry offsets |
| `SEW_SHELLS` | False | Repair non-manifold geometry (slower) |
| `DISABLE_OPENING_SUBTRACTIONS` | False | Skip boolean subtraction of openings (faster) |
| `ELEMENT_HIERARCHY` | False | Include decomposition children in parent geometry |

### String-Based Settings

```python
settings.set("precision", 0.001)        # Geometry precision (0.0001-0.01)
settings.set("deflection", 0.001)       # Triangle quality for tessellation
settings.set("dimensionality", 3)       # 3D or 2D output
settings.set("context-identifiers", ["Body"])  # Filter by context
```

---

## Geometry Data Structure Reference

### Shape Object (from create_shape / iterator)

| Property | Type | Description |
|---|---|---|
| `shape.id` | int | IFC entity step ID |
| `shape.guid` | str | IFC GlobalId |
| `shape.geometry.verts` | flat list | [x1,y1,z1, x2,y2,z2, ...] |
| `shape.geometry.faces` | flat list | [i1,i2,i3, ...] triangle indices |
| `shape.geometry.edges` | flat list | [i1,i2, ...] edge vertex indices |
| `shape.geometry.normals` | flat list | [nx1,ny1,nz1, ...] per-vertex normals |
| `shape.geometry.materials` | list | Style objects with color/transparency |
| `shape.geometry.material_ids` | flat list | Material index per face |
| `shape.geometry.item_ids` | flat list | Representation item index per face |
| `shape.transformation.matrix.data` | flat list | 4x3 column-major transform |

### Representation Context Identifiers

| Identifier | Purpose | Typical Use |
|---|---|---|
| `"Body"` | 3D solid geometry | Walls, columns, beams, furniture |
| `"Axis"` | Parametric axis line | Wall center line, beam axis |
| `"Box"` | Bounding box | Clash detection, quick preview |
| `"Profile"` | 2D cross-section profile | Structural sections |
| `"Footprint"` | Plan view outline | Floor plan generation |
| `"Clearance"` | Clearance zone | Space requirements |
| `"Annotation"` | Drawing annotations | Dimensions, labels |
