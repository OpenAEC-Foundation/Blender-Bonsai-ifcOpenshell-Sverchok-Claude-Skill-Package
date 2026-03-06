# Profile API Method Signatures

Complete API reference for IfcOpenShell profile operations. All signatures verified against the [official IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/profile/index.html).

---

## ifcopenshell.api.profile.add_parameterised_profile()

Creates a standard parametric profile entity (IfcParameterizedProfileDef subclass).

```python
ifcopenshell.api.profile.add_parameterised_profile(
    file: ifcopenshell.file,
    ifc_class: str,
    profile_type: str = "AREA"
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `ifc_class` | `str` | *required* | Profile class name (e.g., `"IfcIShapeProfileDef"`, `"IfcCircleProfileDef"`) |
| `profile_type` | `str` | `"AREA"` | Profile type: `"AREA"` (solid extrusion) or `"CURVE"` (centerline) |

### Return Value

`ifcopenshell.entity_instance` — The created profile entity matching the specified `ifc_class`.

### Notes

- **ALWAYS** use British spelling: `add_parameterised_profile`. The American spelling `add_parameterized_profile` does NOT exist.
- The profile is created with dimension attributes at their default values (typically zero or null). ALWAYS call `edit_profile()` afterwards to set dimensions.
- Valid `ifc_class` values are any subclass of `IfcParameterizedProfileDef` — see the profile types table below.

### Supported Profile Classes

| ifc_class | IFC Entity Created | Shape |
|-----------|-------------------|-------|
| `"IfcRectangleProfileDef"` | Rectangle | Solid rectangle |
| `"IfcRectangleHollowProfileDef"` | Hollow rectangle | Rectangle with uniform wall |
| `"IfcRoundedRectangleProfileDef"` | Rounded rectangle | Rectangle with rounded corners |
| `"IfcCircleProfileDef"` | Circle | Solid circle |
| `"IfcCircleHollowProfileDef"` | Hollow circle | Circle with wall thickness (tube) |
| `"IfcEllipseProfileDef"` | Ellipse | Solid ellipse |
| `"IfcIShapeProfileDef"` | I/H-beam | Symmetric I-shape |
| `"IfcAsymmetricIShapeProfileDef"` | Asymmetric I-beam | I-shape with different top/bottom flanges |
| `"IfcTShapeProfileDef"` | T-shape | T cross-section |
| `"IfcLShapeProfileDef"` | L-shape / angle | Equal or unequal angle |
| `"IfcUShapeProfileDef"` | U-shape / channel | Channel section |
| `"IfcCShapeProfileDef"` | C-shape | C cross-section |
| `"IfcZShapeProfileDef"` | Z-shape | Z cross-section |
| `"IfcTrapeziumProfileDef"` | Trapezoid | Trapezoidal section |

---

## ifcopenshell.api.profile.add_arbitrary_profile()

Creates a polyline-based profile from a list of 2D coordinate tuples.

```python
ifcopenshell.api.profile.add_arbitrary_profile(
    file: ifcopenshell.file,
    profile: list[tuple[float, float]],
    name: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `profile` | `list[tuple[float, float]]` | *required* | List of 2D coordinate tuples defining the polyline. Coordinates MUST be in SI meters. |
| `name` | `str \| None` | `None` | Optional semantic identifier for the profile |

### Return Value

`ifcopenshell.entity_instance` — An IfcArbitraryClosedProfileDef entity.

### Notes

- Coordinates MUST be in SI meters. A 500mm dimension is `0.5`, NOT `500`.
- The API auto-closes the polyline (connects last point to first). Providing an explicitly closed polygon (first point == last point) is acceptable but not required.
- The coordinate list defines a closed curve; the interior is the profile area.
- Winding order matters for voids but not for simple profiles.

---

## ifcopenshell.api.profile.add_arbitrary_profile_with_voids()

Creates a polyline-based profile with one or more internal voids (holes).

```python
ifcopenshell.api.profile.add_arbitrary_profile_with_voids(
    file: ifcopenshell.file,
    outer_profile: list[tuple[float, float]],
    inner_profiles: list[list[tuple[float, float]]],
    name: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `outer_profile` | `list[tuple[float, float]]` | *required* | Outer boundary coordinate list (SI meters) |
| `inner_profiles` | `list[list[tuple[float, float]]]` | *required* | List of void coordinate lists (each is a closed polyline) |
| `name` | `str \| None` | `None` | Optional semantic identifier |

### Return Value

`ifcopenshell.entity_instance` — An IfcArbitraryProfileDefWithVoids entity.

### Notes

- `inner_profiles` is a list of lists. Each inner list defines one void.
- Voids MUST be fully contained within the outer profile boundary.
- Voids MUST NOT overlap each other.
- All coordinates in SI meters.

---

## ifcopenshell.api.profile.edit_profile()

Modifies attributes of an existing IfcProfileDef entity.

```python
ifcopenshell.api.profile.edit_profile(
    file: ifcopenshell.file,
    profile: ifcopenshell.entity_instance,
    attributes: dict
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `profile` | `entity_instance` | *required* | The IfcProfileDef entity to modify |
| `attributes` | `dict` | *required* | Dictionary of attribute names and values |

### Return Value

`None`

### Common Attributes by Profile Type

#### IfcIShapeProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier (e.g., "HEA 200") |
| `OverallWidth` | `float` | Yes | Total flange width (meters) |
| `OverallDepth` | `float` | Yes | Total section depth (meters) |
| `WebThickness` | `float` | Yes | Web thickness (meters) |
| `FlangeThickness` | `float` | Yes | Flange thickness (meters) |
| `FilletRadius` | `float` | No | Fillet radius at web-flange junction (meters) |
| `FlangeEdgeRadius` | `float` | No | Edge radius of flange (meters) |
| `FlangeSlope` | `float` | No | Flange slope angle (radians) |

#### IfcRectangleProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `XDim` | `float` | Yes | Width (meters) |
| `YDim` | `float` | Yes | Height (meters) |

#### IfcRectangleHollowProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `XDim` | `float` | Yes | Outer width (meters) |
| `YDim` | `float` | Yes | Outer height (meters) |
| `WallThickness` | `float` | Yes | Uniform wall thickness (meters) |
| `InnerFilletRadius` | `float` | No | Inner corner radius (meters) |
| `OuterFilletRadius` | `float` | No | Outer corner radius (meters) |

#### IfcCircleProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Radius` | `float` | Yes | Circle radius (meters) |

#### IfcCircleHollowProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Radius` | `float` | Yes | Outer radius (meters) |
| `WallThickness` | `float` | Yes | Wall thickness (meters) |

#### IfcEllipseProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `SemiAxis1` | `float` | Yes | Semi-axis in X direction (meters) |
| `SemiAxis2` | `float` | Yes | Semi-axis in Y direction (meters) |

#### IfcTShapeProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Depth` | `float` | Yes | Total depth (meters) |
| `FlangeWidth` | `float` | Yes | Flange width (meters) |
| `WebThickness` | `float` | Yes | Web thickness (meters) |
| `FlangeThickness` | `float` | Yes | Flange thickness (meters) |
| `FilletRadius` | `float` | No | Web-flange fillet radius (meters) |
| `FlangeEdgeRadius` | `float` | No | Flange edge radius (meters) |
| `WebEdgeRadius` | `float` | No | Web edge radius (meters) |
| `WebSlope` | `float` | No | Web slope angle (radians) |
| `FlangeSlope` | `float` | No | Flange slope angle (radians) |

#### IfcLShapeProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Depth` | `float` | Yes | Vertical leg length (meters) |
| `Width` | `float` | No | Horizontal leg length (meters, defaults to Depth if omitted for equal angle) |
| `Thickness` | `float` | Yes | Leg thickness (meters) |
| `FilletRadius` | `float` | No | Internal fillet radius (meters) |
| `EdgeRadius` | `float` | No | Leg edge radius (meters) |
| `LegSlope` | `float` | No | Leg slope angle (radians) |

#### IfcUShapeProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Depth` | `float` | Yes | Total depth (meters) |
| `FlangeWidth` | `float` | Yes | Flange width (meters) |
| `WebThickness` | `float` | Yes | Web thickness (meters) |
| `FlangeThickness` | `float` | Yes | Flange thickness (meters) |
| `FilletRadius` | `float` | No | Internal fillet radius (meters) |
| `EdgeRadius` | `float` | No | Flange edge radius (meters) |
| `FlangeSlope` | `float` | No | Flange slope angle (radians) |

#### IfcCShapeProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Depth` | `float` | Yes | Total depth (meters) |
| `Width` | `float` | Yes | Total width (meters) |
| `WallThickness` | `float` | Yes | Wall thickness (meters) |
| `Girth` | `float` | Yes | Lip/return length (meters) |
| `InternalFilletRadius` | `float` | No | Internal fillet radius (meters) |

#### IfcZShapeProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `Depth` | `float` | Yes | Total depth (meters) |
| `FlangeWidth` | `float` | Yes | Flange width (meters) |
| `WebThickness` | `float` | Yes | Web thickness (meters) |
| `FlangeThickness` | `float` | Yes | Flange thickness (meters) |
| `FilletRadius` | `float` | No | Fillet radius (meters) |
| `EdgeRadius` | `float` | No | Edge radius (meters) |

#### IfcTrapeziumProfileDef

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `ProfileName` | `str` | No | Semantic identifier |
| `BottomXDim` | `float` | Yes | Bottom edge width (meters) |
| `TopXDim` | `float` | Yes | Top edge width (meters) |
| `YDim` | `float` | Yes | Height (meters) |
| `TopXOffset` | `float` | Yes | Horizontal offset of top edge from bottom-left (meters) |

---

## ifcopenshell.api.profile.copy_profile()

Duplicates a profile entity including all dimension attributes, disconnected from original element references.

```python
ifcopenshell.api.profile.copy_profile(
    file: ifcopenshell.file,
    profile: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `profile` | `entity_instance` | *required* | The IfcProfileDef to duplicate |

### Return Value

`ifcopenshell.entity_instance` — A new profile entity with identical attributes but no element associations.

### Notes

- The copy is fully independent. Modifying the copy does not affect the original.
- Property sets associated with the profile are also copied.

---

## ifcopenshell.api.profile.remove_profile()

Deletes a profile entity from the model.

```python
ifcopenshell.api.profile.remove_profile(
    file: ifcopenshell.file,
    profile: ifcopenshell.entity_instance
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `profile` | `entity_instance` | *required* | The IfcProfileDef to remove |

### Return Value

`None`

### Notes

- **WARNING:** Removing a profile that is still referenced by geometry representations or material profile sets creates dangling references. ALWAYS ensure the profile is unused before removing.
- To check usage: look for references in `IfcMaterialProfile.Profile` and `IfcSweptAreaSolid.SweptArea`.

---

## Related Functions (from other modules)

### geometry.add_profile_representation()

Creates extruded solid geometry from a profile definition.

```python
ifcopenshell.api.geometry.add_profile_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,
    depth: float,
    cardinal_point: int = 5
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `context` | `entity_instance` | *required* | Body subcontext for representation |
| `profile` | `entity_instance` | *required* | IfcProfileDef defining the cross-section |
| `depth` | `float` | *required* | Extrusion length in meters |
| `cardinal_point` | `int` | `5` | Insertion point (1-9, where 5 = centroid) |

### material.add_profile()

Adds a material profile to an IfcMaterialProfileSet. See ifcos-impl-materials for full documentation.

```python
ifcopenshell.api.material.add_profile(
    file: ifcopenshell.file,
    profile_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance | None = None,
    profile: ifcopenshell.entity_instance | None = None,
    name: str | None = None
) -> ifcopenshell.entity_instance
```
