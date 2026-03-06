# Geometry API Method Signatures

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Blender 4.2.0+
> **Source-verified** against installed Bonsai v0.8.3 and IfcOpenShell source

---

## 1. IfcOpenShell Geometry API (`ifcopenshell.api.run("geometry.*", ...)`)

### geometry.add_wall_representation

```python
ifcopenshell.api.run("geometry.add_wall_representation", model,
    context: ifcopenshell.entity_instance,         # IfcGeometricRepresentationSubContext (Body)
    length: float = 1.0,                           # Wall length in SI meters
    height: float = 3.0,                           # Wall height in SI meters
    direction_sense: str = "POSITIVE",             # "POSITIVE" or "NEGATIVE"
    offset: float = 0.0,                           # Base offset from origin
    thickness: float = 0.2,                        # Wall thickness in SI meters
    x_angle: float = 0.0,                          # Slope along X-axis (radians)
    clippings: list[dict | Clipping] | None = None,  # Clipping planes
    booleans: list[ifcopenshell.entity_instance] | None = None,  # Existing boolean operands
) -> ifcopenshell.entity_instance  # Returns IfcShapeRepresentation
```

- **RepresentationType**: `"SweptSolid"` (no clippings) or `"Clipping"` (with clippings)
- **Items**: `[IfcExtrudedAreaSolid]` wrapping `IfcArbitraryClosedProfileDef` from `IfcIndexedPolyCurve` (IFC4) or `IfcPolyline` (IFC2X3)
- **Clipping format**: `{"location": (x, y, z), "normal": (nx, ny, nz)}` or `Clipping(location=..., normal=...)`

### geometry.add_slab_representation

```python
ifcopenshell.api.run("geometry.add_slab_representation", model,
    context: ifcopenshell.entity_instance,
    depth: float = 0.2,                            # Slab thickness in SI meters
    direction_sense: str = "POSITIVE",
    offset: float = 0.0,
    x_angle: float = 0.0,                          # Slope in radians
    clippings: list[dict | Clipping | ifcopenshell.entity_instance] | None = None,
    polyline: list[tuple[float, float]] | None = None,  # Custom footprint (SI meters)
) -> ifcopenshell.entity_instance  # Returns IfcShapeRepresentation
```

- Default footprint: 1m x 1m square (project units). Custom `polyline` replaces it.
- Produces `IfcExtrudedAreaSolid` wrapping `IfcArbitraryClosedProfileDef`.

### geometry.add_profile_representation

```python
ifcopenshell.api.run("geometry.add_profile_representation", model,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,              # Any IfcProfileDef subclass
    depth: float = 1.0,                                 # Extrusion depth in SI meters
    cardinal_point: int | None = 5,                     # 1-19 (default 5 = "mid-depth centre")
    clippings: list[dict | Clipping] | None = None,
    placement_zx_axes: tuple[tuple | None, tuple | None] = (None, None),
) -> ifcopenshell.entity_instance  # Returns IfcShapeRepresentation
```

Cardinal points (IFC standard):
| Value | Name |
|---|---|
| 1 | bottom left |
| 2 | bottom centre |
| 3 | bottom right |
| 4 | mid-depth left |
| 5 | mid-depth centre (default) |
| 6 | mid-depth right |
| 7 | top left |
| 8 | top centre |
| 9 | top right |
| 10 | geometric centroid |

Supported profile types for automatic extent calculation: `IfcAsymmetricIShapeProfileDef`, `IfcCShapeProfileDef`, `IfcCircleProfileDef`, `IfcEllipseProfileDef`, `IfcIShapeProfileDef`, `IfcLShapeProfileDef`, `IfcRectangleProfileDef`, `IfcTShapeProfileDef`, `IfcUShapeProfileDef`, `IfcZShapeProfileDef`.

### geometry.add_mesh_representation

```python
ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context: ifcopenshell.entity_instance,
    vertices: list[list[tuple[float, float, float]]],  # [[item1_verts], [item2_verts], ...]
    edges: list | None = None,                          # Not yet supported
    faces: list[list[list[int]]] = None,               # [[item1_faces], [item2_faces], ...]
    coordinate_offset: tuple | None = None,
    unit_scale: float | None = None,                   # None = assume SI meters
    force_faceted_brep: bool = False,                  # Force IfcFacetedBrep
) -> ifcopenshell.entity_instance  # Returns IfcShapeRepresentation
```

- **IFC4+**: Returns `RepresentationType = "Tessellation"` with `IfcPolygonalFaceSet`
- **IFC2X3** (or `force_faceted_brep=True`): Returns `"Brep"` with `IfcFacetedBrep`
- `vertices` and `faces` are **parallel lists**: each outer element is one `IfcRepresentationItem`

### geometry.add_axis_representation

```python
ifcopenshell.api.run("geometry.add_axis_representation", model,
    context: ifcopenshell.entity_instance,  # Model/Axis/GRAPH_VIEW or Plan/Axis/GRAPH_VIEW
    axis: tuple[tuple, tuple],              # ((x1,y1), (x2,y2)) for 2D; ((x1,y1,z1),...) for 3D
) -> ifcopenshell.entity_instance  # Returns IfcShapeRepresentation
```

- **RepresentationType**: `"Curve2D"` (2D points) or `"Curve3D"` (3D points)

### geometry.add_representation (Blender-dependent)

```python
ifcopenshell.api.run("geometry.add_representation", model,
    context: ifcopenshell.entity_instance,
    blender_object: bpy.types.Object,
    geometry: bpy.types.Mesh | bpy.types.Curve,
    coordinate_offset: numpy.ndarray | None = None,
    total_items: int = 1,
    unit_scale: float | None = None,
    should_force_faceted_brep: bool = False,
    should_force_triangulation: bool = False,
    should_generate_uvs: bool = False,
    ifc_representation_class: str | None = None,
    profile_set_usage: ifcopenshell.entity_instance | None = None,
    text_literal: ifcopenshell.entity_instance | None = None,
) -> ifcopenshell.entity_instance | None
```

Valid `ifc_representation_class` values:
- `"IfcExtrudedAreaSolid/IfcRectangleProfileDef"`
- `"IfcExtrudedAreaSolid/IfcCircleProfileDef"`
- `"IfcExtrudedAreaSolid/IfcArbitraryClosedProfileDef"`
- `"IfcExtrudedAreaSolid/IfcArbitraryProfileDefWithVoids"`
- `"IfcExtrudedAreaSolid/IfcMaterialProfileSetUsage"`
- `"IfcGeometricCurveSet/IfcTextLiteral"`
- `"IfcTextLiteral"`

### geometry.assign_representation

```python
ifcopenshell.api.run("geometry.assign_representation", model,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance,
) -> None
```

- For `IfcProduct`: appends to `product.Representation.Representations`
- For `IfcTypeProduct`: creates `IfcRepresentationMap` and `IfcMappedItem` references on all instances

### geometry.unassign_representation

```python
ifcopenshell.api.run("geometry.unassign_representation", model,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance,
) -> None
```

- Removes representation from product. Removes `IfcProductDefinitionShape` if last representation.
- For type products: removes `IfcRepresentationMap` and all mapped items on instances.

### geometry.remove_representation

```python
ifcopenshell.api.run("geometry.remove_representation", model,
    representation: ifcopenshell.entity_instance,
    should_keep_named_profiles: bool = True,
) -> None
```

- Deep-removes representation and its items (styles, textures, layer assignments).
- Named profiles are preserved by default (assumed to be library profiles).
- **ALWAYS** call `unassign_representation` first.

### geometry.edit_object_placement

```python
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product: ifcopenshell.entity_instance,
    matrix: numpy.ndarray | None = None,       # 4x4 matrix; defaults to identity
    is_si: bool = True,                        # True = matrix translation in meters
    should_transform_children: bool = False,   # Move child elements with parent
) -> ifcopenshell.entity_instance  # Returns new IfcLocalPlacement
```

- Only supports `IfcLocalPlacement`
- Children include: openings, fillings, decomposed parts, nested parts
- `IfcFeatureElement` children are always moved with parent regardless of `should_transform_children`

### geometry.add_boolean

```python
ifcopenshell.api.run("geometry.add_boolean", model,
    first_item: ifcopenshell.entity_instance,       # IfcBooleanOperand (solid/mesh)
    second_items: list[ifcopenshell.entity_instance],
    operator: str = "DIFFERENCE",                    # "DIFFERENCE", "INTERSECTION", "UNION"
) -> list[ifcopenshell.entity_instance]             # List of IfcBooleanResult
```

- Creates `IfcBooleanClippingResult` when operator=DIFFERENCE + first is SweptSolid and second is `IfcHalfSpaceSolid`
- Otherwise creates `IfcBooleanResult`
- Removes second items from their parent `IfcShapeRepresentation.Items`

### geometry.remove_boolean

```python
ifcopenshell.api.run("geometry.remove_boolean", model,
    item: ifcopenshell.entity_instance,  # IfcBooleanResult or item in boolean chain
) -> None
```

- Restores first operand to parent representation
- Restores second operand as top-level item

---

## 2. Context API (`ifcopenshell.api.run("context.*", ...)`)

### context.add_context

```python
ifcopenshell.api.run("context.add_context", model,
    context_type: str | None = None,            # "Model" or "Plan"
    context_identifier: str | None = None,      # "Body", "Axis", "Box", "Profile", etc.
    target_view: str | None = None,             # "MODEL_VIEW", "PLAN_VIEW", "GRAPH_VIEW", etc.
    target_scale: float | None = None,
    parent: ifcopenshell.entity_instance | None = None,  # Parent for subcontexts
) -> ifcopenshell.entity_instance
```

- No `parent` → returns `IfcGeometricRepresentationContext`
- With `parent` → returns `IfcGeometricRepresentationSubContext`

---

## 3. Profile API (`ifcopenshell.api.run("profile.*", ...)`)

### profile.add_parameterized_profile

```python
ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class: str,                # IfcParameterizedProfileDef subclass name
    profile_type: str = "AREA",    # "AREA" or "CURVE"
) -> ifcopenshell.entity_instance
```

Set dimensions via `attribute.edit_attributes` after creation.

### profile.add_arbitrary_profile

```python
ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile: list[tuple[float, float]],  # 2D coordinates (SI meters), closed curve
    name: str | None = None,
) -> ifcopenshell.entity_instance  # IfcArbitraryClosedProfileDef
```

- First and last point MUST be identical (closed curve).

### profile.add_arbitrary_profile_with_voids

```python
ifcopenshell.api.run("profile.add_arbitrary_profile_with_voids", model,
    outer_profile: list[tuple[float, float]],
    inner_profiles: list[list[tuple[float, float]]],
    name: str | None = None,
) -> ifcopenshell.entity_instance  # IfcArbitraryProfileDefWithVoids
```

### profile.edit_profile

```python
ifcopenshell.api.run("profile.edit_profile", model,
    profile: ifcopenshell.entity_instance,
    attributes: dict,
) -> None
```

### profile.remove_profile / profile.copy_profile

```python
ifcopenshell.api.run("profile.remove_profile", model, profile=profile) -> None
ifcopenshell.api.run("profile.copy_profile", model, profile=profile) -> ifcopenshell.entity_instance
```

---

## 4. Feature API (`ifcopenshell.api.run("feature.*", ...)`)

### feature.add_feature

```python
ifcopenshell.api.run("feature.add_feature", model,
    feature: ifcopenshell.entity_instance,  # IfcFeatureElement subclass
    element: ifcopenshell.entity_instance,  # IfcElement to apply feature to
) -> ifcopenshell.entity_instance
```

- `IfcFeatureElementSubtraction` (e.g., `IfcOpeningElement`) → creates `IfcRelVoidsElement`
- `IfcFeatureElementAddition` (e.g., `IfcProjectionElement`) → creates `IfcRelProjectsElement`
- `IfcSurfaceFeature` → creates `IfcRelAdheresToElement` (IFC4X3)

### feature.add_filling

```python
ifcopenshell.api.run("feature.add_filling", model,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance,
) -> None  # Creates IfcRelFillsElement
```

### feature.remove_feature / feature.remove_filling

```python
ifcopenshell.api.run("feature.remove_feature", model, feature=feature) -> None
ifcopenshell.api.run("feature.remove_filling", model, filling=filling) -> None
```

---

## 5. ShapeBuilder (`ifcopenshell.util.shape_builder.ShapeBuilder`)

```python
from ifcopenshell.util.shape_builder import ShapeBuilder, V

builder = ShapeBuilder(ifc_file)
```

### Curve Creation

```python
builder.polyline(
    points: list[tuple],                # 2D or 3D points
    closed: bool = False,
    position_offset: tuple | None = None,
    arc_points: tuple[int, ...] = (),   # Indices of arc midpoints
) -> ifcopenshell.entity_instance  # IfcIndexedPolyCurve (IFC4+) or IfcPolyline (IFC2X3)

builder.rectangle(
    size: tuple = (1.0, 1.0),
    position: tuple | None = None,
) -> ifcopenshell.entity_instance  # IfcIndexedPolyCurve

builder.circle(
    center: tuple = (0.0, 0.0),
    radius: float = 1.0,
) -> ifcopenshell.entity_instance  # IfcCircle
```

### Profile Creation

```python
builder.profile(
    outer_curve: ifcopenshell.entity_instance,  # Must be 2D IfcCurve
    name: str | None = None,
    inner_curves: tuple = (),
    profile_type: str = "AREA",
) -> ifcopenshell.entity_instance  # IfcArbitraryClosedProfileDef or IfcArbitraryProfileDefWithVoids
```

### Solid Creation

```python
builder.extrude(
    profile_or_curve: ifcopenshell.entity_instance,
    magnitude: float = 1.0,
    position: tuple = (0.0, 0.0, 0.0),
    extrusion_vector: tuple = (0.0, 0.0, 1.0),
    position_z_axis: tuple = (0.0, 0.0, 1.0),
    position_x_axis: tuple = (1.0, 0.0, 0.0),
    position_y_axis: tuple | None = None,
) -> ifcopenshell.entity_instance  # IfcExtrudedAreaSolid

builder.sphere(
    radius: float = 1.0,
    center: tuple = (0.0, 0.0, 0.0),
) -> ifcopenshell.entity_instance  # IfcSphere

builder.block(
    position: tuple = (0.0, 0.0, 0.0),
    x_length: float = 1.0,
    y_length: float = 1.0,
    z_length: float = 1.0,
) -> ifcopenshell.entity_instance  # IfcBlock

builder.half_space_solid(
    plane: ifcopenshell.entity_instance,  # IfcPlane
    agreement_flag: bool = False,         # False: +Z side is void
) -> ifcopenshell.entity_instance  # IfcHalfSpaceSolid

builder.create_swept_disk_solid(
    path_curve: ifcopenshell.entity_instance,  # Must be 3D IfcCurve
    radius: float,
) -> ifcopenshell.entity_instance  # IfcSweptDiskSolid
```

### Mesh Creation

```python
builder.mesh(
    points: list[tuple],
    faces: list[list[int]],
) -> ifcopenshell.entity_instance  # IfcPolygonalFaceSet (IFC4+) or IfcFacetedBrep (IFC2X3)

builder.faceted_brep(
    points: list[tuple],
    faces: list[list[int]],
) -> ifcopenshell.entity_instance  # IfcFacetedBrep

builder.triangulated_face_set(
    points: list[tuple],
    faces: list[list[int]],   # Triangles only, 0-indexed
) -> ifcopenshell.entity_instance  # IfcTriangulatedFaceSet (not available in IFC2X3)

builder.polygonal_face_set(
    points: list[tuple],
    faces: list[list[int] | list[list[int]]],  # Outer + inner loops
) -> ifcopenshell.entity_instance  # IfcPolygonalFaceSet (not available in IFC2X3)

builder.extrude_face_set(
    points: list[tuple],          # Consecutive closed polyline points
    magnitude: float,
    extrusion_vector: tuple = (0, 0, 1),
    offset: tuple | None = None,
    start_cap: bool = True,
    end_cap: bool = True,
) -> ifcopenshell.entity_instance  # IfcPolygonalFaceSet
```

### Representation Assembly

```python
builder.get_representation(
    context: ifcopenshell.entity_instance,       # IfcGeometricRepresentationSubContext
    items: ifcopenshell.entity_instance | list,  # Single or list of items
    representation_type: str | None = None,      # Auto-guessed if None
) -> ifcopenshell.entity_instance  # IfcShapeRepresentation or IfcTopologyRepresentation
```

### Transform Utilities

```python
builder.translate(curve_or_item, translation: tuple, create_copy: bool = False)
builder.rotate(curve_or_item, angle: float, axis: str, center: tuple = (0,0,0), create_copy: bool = False)
builder.mirror(curve_or_item, mirror_axes: tuple, mirror_point: tuple = (0,0,0), create_copy: bool = False)

builder.plane(
    location: tuple = (0.0, 0.0, 0.0),
    normal: tuple = (0.0, 0.0, 1.0),
) -> ifcopenshell.entity_instance  # IfcPlane

builder.create_axis2_placement_3d(
    position: tuple = (0,0,0),
    z_axis: tuple = (0,0,1),
    x_axis: tuple = (1,0,0),
) -> ifcopenshell.entity_instance  # IfcAxis2Placement3D

builder.create_axis2_placement_2d(
    position: tuple,
    forward: tuple = (1, 0),
) -> ifcopenshell.entity_instance  # IfcAxis2Placement2D
```

### Extrusion Axis Helpers

```python
builder.extrude_kwargs(axis: str) -> dict
# axis="Y" | "X" | "Z"
# Returns dict with position_x_axis, position_z_axis, extrusion_vector

builder.rotate_extrusion_kwargs_by_z(kwargs: dict, angle: float, counter_clockwise: bool = False) -> dict
```

### Utility Methods

```python
builder.get_polyline_coords(polyline) -> numpy.ndarray
builder.set_polyline_coords(polyline, coords) -> None
builder.get_rectangle_coords(size, position=None) -> numpy.ndarray
builder.deep_copy(element) -> ifcopenshell.entity_instance
```

---

## 6. Context Utility (`ifcopenshell.util.representation`)

```python
import ifcopenshell.util.representation

# Get existing context (returns None if not found)
body = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

# Guess RepresentationType from items
rep_type = ifcopenshell.util.representation.guess_type(items)
```

---

## 7. Bonsai Tool Geometry Methods (`tool.Geometry`)

Key methods on `bonsai.tool.geometry.Geometry`:

```python
@classmethod
def get_active_representation(cls, obj: bpy.types.Object) -> ifcopenshell.entity_instance | None

@classmethod
def get_representation_by_context(cls, element, context) -> ifcopenshell.entity_instance | None

@classmethod
def get_ifc_representation_class(cls, product, representation) -> str | None
# Returns e.g. "IfcExtrudedAreaSolid/IfcRectangleProfileDef"

@classmethod
def get_profile_set_usage(cls, element) -> ifcopenshell.entity_instance | None

@classmethod
def is_body_representation(cls, representation) -> bool

@classmethod
def is_meshlike(cls, representation) -> bool
# True for: Brep, AdvancedBrep, Tessellation, SurfaceModel, Solid, SolidModel

@classmethod
def is_swept_profile(cls, representation) -> bool
# True if representation has IfcExtrudedAreaSolid items

@classmethod
def is_mapped_representation(cls, representation) -> bool

@classmethod
def is_boolean_operand(cls, obj) -> bool

@classmethod
def resolve_mapped_representation(cls, representation) -> ifcopenshell.entity_instance
# Follows MappedItem chain to get root representation

@classmethod
def has_openings(cls, element) -> bool

@classmethod
def get_openings(cls, element) -> Generator[ifcopenshell.entity_instance, None, None]

@classmethod
def should_force_faceted_brep(cls) -> bool

@classmethod
def should_force_triangulation(cls) -> bool

@classmethod
def should_generate_uvs(cls, obj) -> bool

@classmethod
def reload_representation(cls, obj_or_objs) -> None

@classmethod
def clear_cache(cls, element) -> None
```

---

## 8. Bonsai Core Geometry Functions (`bonsai.core.geometry`)

```python
def edit_object_placement(
    ifc, geometry, surveyor,
    obj: bpy.types.Object | None = None,
    apply_scale: bool = True,
) -> None

def add_representation(
    ifc, geometry, style, surveyor,
    obj: bpy.types.Object,
    context: ifcopenshell.entity_instance,
    ifc_representation_class: str | None = None,
    profile_set_usage: ifcopenshell.entity_instance | None = None,
) -> ifcopenshell.entity_instance | None

def switch_representation(
    ifc, geometry,
    obj: bpy.types.Object,
    representation: ifcopenshell.entity_instance,
    should_reload: bool = True,
    is_global: bool = True,
    should_sync_changes_first: bool = False,
    apply_openings: bool = True,
) -> None

def remove_representation(
    ifc, geometry,
    obj: bpy.types.Object,
    representation: ifcopenshell.entity_instance,
) -> None

def purge_unused_representations(ifc, geometry) -> int

def get_representation_ifc_parameters(
    geometry,
    obj: bpy.types.Object,
    should_sync_changes_first: bool = False,
) -> None
```
