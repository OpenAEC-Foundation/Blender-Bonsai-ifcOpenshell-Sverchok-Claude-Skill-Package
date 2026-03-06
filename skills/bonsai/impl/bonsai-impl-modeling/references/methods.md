# Bonsai Modeling — API Method Signatures

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x
> **Calling convention**: `ifcopenshell.api.run("module.function", model, **kwargs)`
> **Source**: Verified against IfcOpenShell source and docs.ifcopenshell.org

---

## 1. Entity Creation

### root.create_entity

```python
ifcopenshell.api.run("root.create_entity", model,
    ifc_class: str = "IfcBuildingElementProxy",
    predefined_type: Optional[str] = None,
    name: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_class` | `str` | `"IfcBuildingElementProxy"` | Any rooted IFC class |
| `predefined_type` | `str \| None` | `None` | Built-in enum value or `"USERDEFINED"` |
| `name` | `str \| None` | `None` | Element name |

**Returns**: New entity with auto-generated `GlobalId` and `OwnerHistory`.

**Notes**:
- If `predefined_type` is not a recognized enum value, it is treated as `USERDEFINED` with `ObjectType` set to the given string.
- Works for both element types (`IfcWallType`) and occurrences (`IfcWall`).

---

## 2. Type Assignment

### type.assign_type

```python
ifcopenshell.api.run("type.assign_type", model,
    related_objects: list[ifcopenshell.entity_instance],
    relating_type: ifcopenshell.entity_instance,
    should_map_representations: bool = True,
) -> Union[ifcopenshell.entity_instance, None]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `related_objects` | `list` | required | List of element occurrences |
| `relating_type` | `entity` | required | The `IfcElementType` to assign |
| `should_map_representations` | `bool` | `True` | Map type geometry to occurrences via `IfcMappedItem` |

**Returns**: `IfcRelDefinesByType` or `None` if list is empty.

**CRITICAL**: `related_objects` MUST be a list. NEVER pass a single entity.

### type.unassign_type

```python
ifcopenshell.api.run("type.unassign_type", model,
    related_objects: list[ifcopenshell.entity_instance],
) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `related_objects` | `list` | required | List of element occurrences to unassign |

**Note**: Does NOT remove mapped representations or material usages from the previously assigned type.

---

## 3. Geometry Creation

### geometry.add_wall_representation

```python
ifcopenshell.api.run("geometry.add_wall_representation", model,
    context: ifcopenshell.entity_instance,
    length: float = 1.0,
    height: float = 3.0,
    thickness: float = 0.2,
    direction_sense: str = "POSITIVE",
    offset: float = 0.0,
    x_angle: float = 0.0,
    clippings: Optional[list] = None,
    booleans: Optional[list] = None,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context` | `entity` | required | `IfcGeometricRepresentationContext` (Model/Body/MODEL_VIEW) |
| `length` | `float` | `1.0` | Wall length in meters |
| `height` | `float` | `3.0` | Wall height in meters |
| `thickness` | `float` | `0.2` | Wall thickness in meters |
| `direction_sense` | `str` | `"POSITIVE"` | `"POSITIVE"` or `"NEGATIVE"` |
| `offset` | `float` | `0.0` | Base offset from origin in meters |
| `x_angle` | `float` | `0.0` | Slope angle along X-axis in radians |
| `clippings` | `list \| None` | `None` | List of `Clipping` objects or dicts |
| `booleans` | `list \| None` | `None` | List of existing `IfcBooleanResult` entities |

**Returns**: `IfcShapeRepresentation` (type `"SweptSolid"` or `"Clipping"` if clippings present).

### geometry.add_slab_representation

```python
ifcopenshell.api.run("geometry.add_slab_representation", model,
    context: ifcopenshell.entity_instance,
    depth: float = 0.2,
    direction_sense: str = "POSITIVE",
    offset: float = 0.0,
    x_angle: float = 0.0,
    clippings: Optional[list] = None,
    polyline: Optional[list[tuple[float, float]]] = None,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context` | `entity` | required | `IfcGeometricRepresentationContext` |
| `depth` | `float` | `0.2` | Slab depth in meters |
| `direction_sense` | `str` | `"POSITIVE"` | Extrusion direction |
| `offset` | `float` | `0.0` | Offset from origin |
| `x_angle` | `float` | `0.0` | Slope angle in radians |
| `clippings` | `list \| None` | `None` | List of `Clipping` objects |
| `polyline` | `list \| None` | `None` | List of 2D `(x, y)` tuples defining outline. Default: 1m x 1m square |

**Returns**: `IfcShapeRepresentation`.

### geometry.add_profile_representation

```python
ifcopenshell.api.run("geometry.add_profile_representation", model,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,
    depth: float = 1.0,
    cardinal_point: int = 5,
    clippings: Optional[list] = None,
    placement_zx_axes: tuple = (None, None),
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context` | `entity` | required | `IfcGeometricRepresentationContext` |
| `profile` | `entity` | required | `IfcProfileDef` to extrude |
| `depth` | `float` | `1.0` | Extrusion depth in meters |
| `cardinal_point` | `int` | `5` | Cardinal point (1-10). 5 = mid-depth centre |
| `clippings` | `list \| None` | `None` | List of `Clipping` objects |
| `placement_zx_axes` | `tuple` | `(None, None)` | `(Z_axis, X_axis)` vectors for orientation |

**Returns**: `IfcShapeRepresentation`. Used for columns, beams, members, footings.

### geometry.assign_representation

```python
ifcopenshell.api.run("geometry.assign_representation", model,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance,
) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `product` | `entity` | required | `IfcProduct` or `IfcTypeProduct` |
| `representation` | `entity` | required | `IfcShapeRepresentation` to assign |

**Notes**:
- For `IfcProduct`: adds to `IfcProductDefinitionShape`.
- For `IfcTypeProduct`: creates `IfcRepresentationMap`.

### geometry.remove_representation

```python
ifcopenshell.api.run("geometry.remove_representation", model,
    representation: ifcopenshell.entity_instance,
) -> None
```

### geometry.edit_object_placement

```python
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product: ifcopenshell.entity_instance,
    matrix: Optional[np.ndarray] = None,
    is_si: bool = True,
    should_transform_children: bool = False,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `product` | `entity` | required | Element to place |
| `matrix` | `np.ndarray \| None` | identity | 4x4 transformation matrix |
| `is_si` | `bool` | `True` | Whether matrix uses SI units |
| `should_transform_children` | `bool` | `False` | Move child elements with parent |

**Returns**: `IfcLocalPlacement`.

---

## 4. Material Operations

### material.add_material

```python
ifcopenshell.api.run("material.add_material", model,
    name: Optional[str] = None,
    category: Optional[str] = None,
    description: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str \| None` | `None` | Material name |
| `category` | `str \| None` | `None` | Standard values: `"concrete"`, `"steel"`, `"aluminium"`, `"block"`, `"brick"`, `"stone"`, `"wood"`, `"glass"`, `"gypsum"`, `"plastic"`, `"earth"` |
| `description` | `str \| None` | `None` | Description text |

**Returns**: `IfcMaterial`.

### material.add_material_set

```python
ifcopenshell.api.run("material.add_material_set", model,
    name: str = "Unnamed",
    set_type: str = "IfcMaterialConstituentSet",
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | `"Unnamed"` | Set name |
| `set_type` | `str` | `"IfcMaterialConstituentSet"` | One of: `"IfcMaterialLayerSet"`, `"IfcMaterialProfileSet"`, `"IfcMaterialConstituentSet"`, `"IfcMaterialList"` |

**Returns**: The material set entity.

### material.add_layer

```python
ifcopenshell.api.run("material.add_layer", model,
    layer_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

**Returns**: `IfcMaterialLayer`. Thickness MUST be set separately via `material.edit_layer`.

### material.edit_layer

```python
ifcopenshell.api.run("material.edit_layer", model,
    layer: ifcopenshell.entity_instance,
    attributes: Optional[dict] = None,
    material: Optional[ifcopenshell.entity_instance] = None,
) -> None
```

| Attribute Key | Type | Description |
|---------------|------|-------------|
| `LayerThickness` | `float` | Layer thickness in meters |
| `IsVentilated` | `bool` | Whether layer has ventilation |
| `Name` | `str` | Layer name |
| `Category` | `str` | Layer category |
| `Priority` | `int` | Layer priority |

### material.add_profile

```python
ifcopenshell.api.run("material.add_profile", model,
    profile_set: ifcopenshell.entity_instance,
    material: Optional[ifcopenshell.entity_instance] = None,
    profile: Optional[ifcopenshell.entity_instance] = None,
    name: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

**Returns**: `IfcMaterialProfile`.

### material.add_constituent

```python
ifcopenshell.api.run("material.add_constituent", model,
    constituent_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

**Returns**: `IfcMaterialConstituent`. NOT available in IFC2X3.

### material.assign_material

```python
ifcopenshell.api.run("material.assign_material", model,
    products: list[ifcopenshell.entity_instance],
    type: str = "IfcMaterial",
    material: Optional[ifcopenshell.entity_instance] = None,
) -> Union[ifcopenshell.entity_instance, list, None]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `products` | `list` | required | List of `IfcProduct` or `IfcTypeProduct` |
| `type` | `str` | `"IfcMaterial"` | Material type string |
| `material` | `entity \| None` | `None` | Material or material set to assign |

**Returns**: `IfcRelAssociatesMaterial` or list thereof.

**CRITICAL**: `products` MUST be a list. Unassigns any previous material.

---

## 5. Spatial Assignment

### spatial.assign_container

```python
ifcopenshell.api.run("spatial.assign_container", model,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance,
) -> Union[ifcopenshell.entity_instance, None]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `products` | `list` | required | List of `IfcElement` entities |
| `relating_structure` | `entity` | required | `IfcSpatialStructureElement` (storey, building, space) |

**Returns**: `IfcRelContainedInSpatialStructure` or `None`.

**CRITICAL**: NEVER use `aggregate.assign_object` for placing physical elements. That is ONLY for spatial hierarchy (Site → Building → Storey).

---

## 6. Feature Operations

### feature.add_feature

```python
ifcopenshell.api.run("feature.add_feature", model,
    feature: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `feature` | `entity` | required | `IfcFeatureElement` (e.g., `IfcOpeningElement`) |
| `element` | `entity` | required | Host `IfcElement` |

**Returns**: `IfcRelVoidsElement` (for openings), `IfcRelProjectsElement` (for projections), or `IfcRelAdheresToElement` (for surface features).

**CRITICAL**: NEVER use `void.add_opening` — this function does NOT exist in IfcOpenShell v0.8.0+.

### void.add_filling

```python
ifcopenshell.api.run("void.add_filling", model,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `opening` | `entity` | required | `IfcOpeningElement` |
| `element` | `entity` | required | `IfcDoor`, `IfcWindow`, etc. |

**Returns**: `IfcRelFillsElement`.

---

## 7. Context Operations

### context.add_context

```python
ifcopenshell.api.run("context.add_context", model,
    context_type: Optional[str] = None,
    context_identifier: Optional[str] = None,
    target_view: Optional[str] = None,
    target_scale: Optional[float] = None,
    parent: Optional[ifcopenshell.entity_instance] = None,
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context_type` | `str \| None` | `None` | `"Model"` or `"Plan"` |
| `context_identifier` | `str \| None` | `None` | `"Body"`, `"Axis"`, `"Box"`, `"FootPrint"`, `"Annotation"` |
| `target_view` | `str \| None` | `None` | `"MODEL_VIEW"`, `"PLAN_VIEW"`, `"GRAPH_VIEW"` |
| `parent` | `entity \| None` | `None` | Parent context (required for sub-contexts) |

**Returns**: `IfcGeometricRepresentationContext` or `IfcGeometricRepresentationSubContext`.

---

## 8. Profile Operations

### profile.add_parameterized_profile

```python
ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class: str,
    profile_type: str = "AREA",
) -> ifcopenshell.entity_instance
```

Common `ifc_class` values: `"IfcRectangleProfileDef"`, `"IfcCircleProfileDef"`, `"IfcIShapeProfileDef"`, `"IfcLShapeProfileDef"`, `"IfcTShapeProfileDef"`, `"IfcCShapeProfileDef"`, `"IfcUShapeProfileDef"`, `"IfcCircleHollowProfileDef"`, `"IfcRectangleHollowProfileDef"`, `"IfcEllipseProfileDef"`.

### profile.add_arbitrary_profile

```python
ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile: list[tuple[float, float]],
    name: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

**Returns**: `IfcArbitraryClosedProfileDef` from 2D coordinate list.

---

## 9. Utility Functions

### ifcopenshell.util.element

```python
import ifcopenshell.util.element

# Get type of an occurrence
element_type = ifcopenshell.util.element.get_type(element)

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(element_type)

# Get material of element (including inherited from type)
material = ifcopenshell.util.element.get_material(element)

# Get spatial container
container = ifcopenshell.util.element.get_container(element)

# Get all property sets
psets = ifcopenshell.util.element.get_psets(element)
```

### ifcopenshell.util.representation

```python
import ifcopenshell.util.representation

# Get existing context
body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

# Get representation of specific context
repr = ifcopenshell.util.representation.get_representation(
    element, "Model", "Body", "MODEL_VIEW")
```

### ifcopenshell.util.placement

```python
import ifcopenshell.util.placement

# Get element's 4x4 placement matrix
matrix = ifcopenshell.util.placement.get_local_placement(element.ObjectPlacement)
```
