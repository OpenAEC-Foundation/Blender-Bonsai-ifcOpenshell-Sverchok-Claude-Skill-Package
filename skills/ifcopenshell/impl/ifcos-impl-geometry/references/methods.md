# ifcos-impl-geometry — API Method Signatures

## ifcopenshell.geom Module

### settings()

```python
ifcopenshell.geom.settings() → Settings
```

Creates a geometry settings object for configuring how geometry is processed.

**Setting Methods:**

```python
settings.set(setting_constant, value) → None
settings.set(setting_name_string, value) → None
```

**Setting Constants (boolean):**

| Constant | Default | Description |
|---|---|---|
| `settings.USE_WORLD_COORDS` | False | Apply full placement chain to geometry |
| `settings.USE_BREP_DATA` | False | Return BRep instead of tessellated mesh |
| `settings.WELD_VERTICES` | True | Merge coincident vertices |
| `settings.APPLY_DEFAULT_MATERIALS` | True | Include material/style data |
| `settings.DISABLE_TRIANGULATION` | False | Skip tessellation |
| `settings.APPLY_LAYERSETS` | False | Apply material layer offsets |
| `settings.SEW_SHELLS` | False | Repair open shells |
| `settings.DISABLE_OPENING_SUBTRACTIONS` | False | Skip opening booleans |
| `settings.ELEMENT_HIERARCHY` | False | Include children in parent geometry |

**String-Based Settings:**

| Name | Type | Default | Description |
|---|---|---|---|
| `"precision"` | float | 0.001 | Geometry precision (0.0001-0.01) |
| `"deflection"` | float | 0.001 | Triangle quality for tessellation |
| `"dimensionality"` | int | 3 | Output dimensionality (2 or 3) |
| `"context-identifiers"` | list[str] | all | Filter by context identifier |
| `"include-elements"` | list[str] | none | IFC class names to include |
| `"exclude-elements"` | list[str] | none | IFC class names to exclude |
| `"apply-default-materials"` | bool | True | Alias for APPLY_DEFAULT_MATERIALS |
| `"use-python-opencascade"` | bool | False | Return pythonOCC shapes |

---

### create_shape()

```python
ifcopenshell.geom.create_shape(
    settings: Settings,
    element: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance | None = None
) → ShapeType
```

Processes geometry for a single IFC element. Returns a shape object containing tessellated mesh or BRep data depending on settings.

**Parameters:**
- `settings` — Settings object configured via `ifcopenshell.geom.settings()`
- `element` — IFC product entity (IfcWall, IfcColumn, etc.), shape representation, representation item, or profile definition
- `representation` — Optional explicit representation to process (when element has multiple representations)

**Returns:** Shape object with the following properties:

| Property | Type | Description |
|---|---|---|
| `.id` | int | IFC entity step ID |
| `.guid` | str | IFC GlobalId |
| `.geometry.verts` | tuple[float] | Flat vertex coordinates [x1,y1,z1, x2,y2,z2, ...] |
| `.geometry.faces` | tuple[int] | Flat triangle indices [i1,i2,i3, ...] |
| `.geometry.edges` | tuple[int] | Flat edge indices [i1,i2, ...] |
| `.geometry.normals` | tuple[float] | Flat per-vertex normals [nx1,ny1,nz1, ...] |
| `.geometry.materials` | list | Style objects (see Material/Style section) |
| `.geometry.material_ids` | tuple[int] | Material index per face |
| `.geometry.item_ids` | tuple[int] | Representation item index per face |
| `.geometry.id` | str | Unique geometry identifier |
| `.geometry.brep_data` | bytes | BRep data (only when USE_BREP_DATA=True) |
| `.transformation.matrix.data` | tuple[float] | 4x3 column-major transformation |

**Raises:** `RuntimeError` when element has no geometry or processing fails.

**Material/Style Object Properties:**

| Property | Type | Description |
|---|---|---|
| `.name` | str | Human-readable identifier |
| `.original_name()` | str | Entity class name |
| `.has_diffuse` | bool | Whether diffuse color is available |
| `.diffuse` | tuple[float] | RGB color values (0-1 range) |
| `.has_transparency` | bool | Whether transparency is available |
| `.transparency` | float | Transparency value (0=opaque, 1=transparent) |

---

### iterator()

```python
ifcopenshell.geom.iterator(
    settings: Settings,
    file: ifcopenshell.file,
    num_threads: int = 1,
    include: list[ifcopenshell.entity_instance] | None = None,
    exclude: list[ifcopenshell.entity_instance] | None = None
) → Iterator
```

Creates an iterator for batch processing all elements with geometry.

**Parameters:**
- `settings` — Settings object
- `file` — IFC model
- `num_threads` — Number of parallel processing threads (use `multiprocessing.cpu_count()`)
- `include` — If provided, only process these elements
- `exclude` — If provided, skip these elements

**Iterator Methods:**

| Method | Returns | Description |
|---|---|---|
| `.initialize()` | bool | Initialize iteration. Returns False if no elements to process. |
| `.get()` | ShapeType | Get current shape (same structure as create_shape return) |
| `.next()` | bool | Advance to next element. Returns False when done. |

**Usage Pattern:**
```python
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        # ... process ...
        if not iterator.next():
            break
```

---

### serializer_settings()

```python
ifcopenshell.geom.serializer_settings() → SerializerSettings
```

Creates settings for geometry serializers.

**Settings:**

| Name | Type | Description |
|---|---|---|
| `"use-element-guids"` | bool | Use IFC GUIDs as element identifiers in output |

---

### Serializers

```python
ifcopenshell.geom.serializers.gltf(
    filename: str,
    settings: Settings,
    serializer_settings: SerializerSettings
) → GltfSerializer

ifcopenshell.geom.serializers.obj(
    filename: str,
    settings: Settings,
    serializer_settings: SerializerSettings
) → ObjSerializer
```

**Serializer Methods:**

| Method | Description |
|---|---|
| `.setFile(model)` | Associate IFC model |
| `.setUnitNameAndMagnitude(name, magnitude)` | Set output units |
| `.writeHeader()` | Initialize output file |
| `.write(shape)` | Write a single shape |
| `.finalize()` | Complete and close output file |

---

## ifcopenshell.api.geometry Module

### edit_object_placement()

```python
ifcopenshell.api.geometry.edit_object_placement(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    matrix: numpy.ndarray | None = None,
    is_si: bool = True,
    should_transform_children: bool = False
) → ifcopenshell.entity_instance
```

Sets the 3D position and orientation of an element using a 4x4 transformation matrix.

**Parameters:**
- `product` — The IFC product to position
- `matrix` — 4x4 numpy transformation matrix (defaults to identity = origin)
- `is_si` — If True, matrix values are in SI units (meters). If False, values are in project units.
- `should_transform_children` — If True, recursively update child placements

**Returns:** IfcLocalPlacement entity

---

### assign_representation()

```python
ifcopenshell.api.geometry.assign_representation(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance
) → None
```

Associates a geometric representation (IfcShapeRepresentation) with a product. Creates an IfcProductDefinitionShape if the product does not have one.

---

### add_wall_representation()

```python
ifcopenshell.api.geometry.add_wall_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    length: float = 1.0,
    height: float = 3.0,
    direction_sense: str = 'POSITIVE',
    offset: float = 0.0,
    thickness: float = 0.2,
    x_angle: float = 0.0,
    clippings: list | None = None,
    booleans: list | None = None
) → ifcopenshell.entity_instance
```

Creates a parametric wall body geometry as an extruded rectangle.

**Parameters:**
- `context` — IfcGeometricRepresentationSubContext (Body context)
- `length` — Wall length along local X axis (meters if is_si=True)
- `height` — Wall height along local Z axis
- `direction_sense` — "POSITIVE" or "NEGATIVE" for thickness direction
- `offset` — Offset from axis line
- `thickness` — Wall thickness along local Y axis
- `x_angle` — Rotation angle around X axis (radians)
- `clippings` — List of clipping plane definitions
- `booleans` — List of boolean operation definitions

**Returns:** IfcShapeRepresentation

---

### add_mesh_representation()

```python
ifcopenshell.api.geometry.add_mesh_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    vertices: list,
    edges: list | None = None,
    faces: list | None = None,
    coordinate_offset: list | None = None,
    unit_scale: float | None = None,
    force_faceted_brep: bool = False
) → ifcopenshell.entity_instance
```

Creates arbitrary polygon mesh geometry from vertices and face indices.

**Parameters:**
- `context` — Body context
- `vertices` — List of [x,y,z] coordinate lists
- `edges` — Optional edge index pairs
- `faces` — List of face index lists (triangles, quads, or n-gons)
- `coordinate_offset` — Optional [x,y,z] offset applied to all vertices
- `unit_scale` — Optional scale factor for coordinates
- `force_faceted_brep` — If True, force IfcFacetedBrep output

**Returns:** IfcShapeRepresentation

---

### add_profile_representation()

```python
ifcopenshell.api.geometry.add_profile_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,
    depth: float = 1.0,
    cardinal_point: int = 5,
    clippings: list | None = None,
    placement_zx_axes: tuple = (None, None)
) → ifcopenshell.entity_instance
```

Creates profiled extrusion geometry (beams, columns, etc.).

**Parameters:**
- `context` — Body context
- `profile` — IfcProfileDef entity (IfcRectangleProfileDef, IfcCircleProfileDef, IfcIShapeProfileDef, etc.)
- `depth` — Extrusion depth along local Z axis
- `cardinal_point` — Insertion point (1-9, like numpad; 5=center)
- `clippings` — Optional clipping planes
- `placement_zx_axes` — Optional (z_axis, x_axis) tuple for custom extrusion direction

**Returns:** IfcShapeRepresentation

---

### add_slab_representation()

```python
ifcopenshell.api.geometry.add_slab_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    depth: float = 0.2,
    direction_sense: str = 'POSITIVE',
    offset: float = 0.0,
    x_angle: float = 0.0,
    clippings: list | None = None,
    polyline: list | None = None
) → ifcopenshell.entity_instance
```

---

### add_door_representation()

```python
ifcopenshell.api.geometry.add_door_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    operation_type: str = 'SINGLE_SWING_LEFT',
    lining_properties: dict | None = None,
    panel_properties: dict | None = None,
    part_of_product: ifcopenshell.entity_instance | None = None,
    unit_scale: float | None = None
) → ifcopenshell.entity_instance | None
```

---

### add_window_representation()

```python
ifcopenshell.api.geometry.add_window_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    partition_type: str = 'SINGLE_PANEL',
    lining_properties: dict | None = None,
    panel_properties: list | None = None,
    part_of_product: ifcopenshell.entity_instance | None = None,
    unit_scale: float | None = None
) → ifcopenshell.entity_instance | None
```

---

### add_boolean()

```python
ifcopenshell.api.geometry.add_boolean(
    file: ifcopenshell.file,
    first_item: ifcopenshell.entity_instance,
    second_items: list[ifcopenshell.entity_instance],
    operator: str = 'DIFFERENCE'
) → list[ifcopenshell.entity_instance]
```

Performs CSG boolean operations on representation items.

**Parameters:**
- `first_item` — Primary representation item
- `second_items` — List of items to apply boolean operation with
- `operator` — "DIFFERENCE", "INTERSECTION", or "UNION"

**Returns:** List of created IfcBooleanResult entities

---

### add_railing_representation()

```python
ifcopenshell.api.geometry.add_railing_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    ...
) → ifcopenshell.entity_instance
```

Creates railing geometry along a path.

---

### add_axis_representation()

```python
ifcopenshell.api.geometry.add_axis_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    axis: list
) → ifcopenshell.entity_instance
```

Creates an axis line representation for walls, beams, columns.

---

### add_footprint_representation()

```python
ifcopenshell.api.geometry.add_footprint_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    ...
) → ifcopenshell.entity_instance
```

Creates 2D footprint/plan view representation.

---

### create_2pt_wall()

```python
ifcopenshell.api.geometry.create_2pt_wall(
    file: ifcopenshell.file,
    element: ifcopenshell.entity_instance,
    context: ifcopenshell.entity_instance,
    p1: tuple[float, float],
    p2: tuple[float, float],
    elevation: float,
    height: float,
    thickness: float,
    ...
) → ifcopenshell.entity_instance
```

Creates wall geometry defined by two 2D endpoints.

---

### remove_representation()

```python
ifcopenshell.api.geometry.remove_representation(
    file: ifcopenshell.file,
    representation: ifcopenshell.entity_instance
) → None
```

Removes a shape representation and its items from the model.

---

### unassign_representation()

```python
ifcopenshell.api.geometry.unassign_representation(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance
) → None
```

Disconnects a representation from a product without deleting it.

---

### map_representation()

```python
ifcopenshell.api.geometry.map_representation(
    file: ifcopenshell.file,
    representation: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

Creates an IfcRepresentationMap for sharing geometry across type occurrences.

---

### remove_boolean()

```python
ifcopenshell.api.geometry.remove_boolean(
    file: ifcopenshell.file,
    item: ifcopenshell.entity_instance
) → None
```

Removes a boolean operation from a representation.

---

## ifcopenshell.util.shape Module

### get_shape_matrix()

```python
ifcopenshell.util.shape.get_shape_matrix(
    shape: ShapeType
) → numpy.ndarray
```

Returns the 4x4 transformation matrix as a nested numpy array.

---

### get_vertices()

```python
ifcopenshell.util.shape.get_vertices(
    geometry: GeometryType
) → list[list[float]]
```

Groups flat vertex array into [[x1,y1,z1], [x2,y2,z2], ...].

---

### get_edges()

```python
ifcopenshell.util.shape.get_edges(
    geometry: GeometryType
) → list[list[int]]
```

Groups flat edge array into [[i1,i2], [i3,i4], ...].

---

### get_faces()

```python
ifcopenshell.util.shape.get_faces(
    geometry: GeometryType
) → list[list[int]]
```

Groups flat face array into [[i1,i2,i3], [i4,i5,i6], ...].

---

## ifcopenshell.util.placement Module

### rotation()

```python
ifcopenshell.util.placement.rotation(
    angle: float,
    axis: str
) → numpy.ndarray
```

Creates a 4x4 rotation matrix.

**Parameters:**
- `angle` — Rotation angle in degrees
- `axis` — Rotation axis: "X", "Y", or "Z"

**Returns:** 4x4 numpy rotation matrix

---

### get_local_placement()

```python
ifcopenshell.util.placement.get_local_placement(
    placement: ifcopenshell.entity_instance
) → numpy.ndarray
```

Resolves the full placement chain (element → storey → building → site) and returns the absolute 4x4 transformation matrix.

---

## ifcopenshell.util.shape_builder Module

### ShapeBuilder

```python
from ifcopenshell.util.shape_builder import ShapeBuilder, V

builder = ShapeBuilder(model)
```

**Key Methods:**

| Method | Description |
|---|---|
| `builder.rectangle(size=V(w, h))` | Create rectangle curve |
| `builder.circle(center, radius)` | Create circle curve |
| `builder.polyline(points, arc_points=[], closed=False)` | Create polyline/polycurve |
| `builder.profile(outer, inner_curves=[], name="")` | Create IfcArbitraryProfileDefWithVoids |
| `builder.extrude(profile, magnitude, position=None)` | Create extruded area solid |
| `builder.mirror(curve, mirror_axes, mirror_point, create_copy)` | Mirror geometry |
| `builder.translate(items, vector)` | Translate items |
| `builder.create_swept_disk_solid(curve, radius)` | Swept disk (pipes, rebar) |
| `builder.get_representation(context, items)` | Create IfcShapeRepresentation |

**V() Helper:**

```python
from ifcopenshell.util.shape_builder import V
v2d = V(1.0, 2.0)       # 2D vector
v3d = V(1.0, 2.0, 3.0)  # 3D vector
v3d = v2d.to_3d()        # Convert 2D to 3D (z=0)
```

---

## ifcopenshell.util.representation Module

### get_context()

```python
ifcopenshell.util.representation.get_context(
    model: ifcopenshell.file,
    context_type: str,
    context_identifier: str | None = None,
    target_view: str | None = None
) → ifcopenshell.entity_instance | None
```

Finds an existing representation context in the model.

**Parameters:**
- `context_type` — "Model" or "Plan"
- `context_identifier` — "Body", "Axis", "Box", "Profile", "Footprint", etc.
- `target_view` — "MODEL_VIEW", "PLAN_VIEW", "ELEVATION_VIEW", etc.

**Returns:** IfcGeometricRepresentationSubContext or None

---

## Common IFC Profile Entities (for add_profile_representation)

| Entity | Key Parameters |
|---|---|
| `IfcRectangleProfileDef` | ProfileType, XDim, YDim |
| `IfcRoundedRectangleProfileDef` | ProfileType, XDim, YDim, RoundingRadius |
| `IfcRectangleHollowProfileDef` | ProfileType, XDim, YDim, WallThickness |
| `IfcCircleProfileDef` | ProfileType, Radius |
| `IfcCircleHollowProfileDef` | ProfileType, Radius, WallThickness |
| `IfcIShapeProfileDef` | ProfileType, OverallWidth, OverallDepth, WebThickness, FlangeThickness |
| `IfcLShapeProfileDef` | ProfileType, Depth, Width, Thickness |
| `IfcTShapeProfileDef` | ProfileType, Depth, FlangeWidth, WebThickness, FlangeThickness |
| `IfcArbitraryClosedProfileDef` | ProfileType, OuterCurve |
| `IfcArbitraryProfileDefWithVoids` | ProfileType, OuterCurve, InnerCurves |
