# Utility Module Method Signatures

Complete API reference for `ifcopenshell.util.*` modules. All signatures verified against the [official IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html).

---

## ifcopenshell.util.element

### get_psets()

Retrieves all property sets (and optionally quantity sets) assigned to an element as a nested dictionary.

```python
ifcopenshell.util.element.get_psets(
    element: ifcopenshell.entity_instance,
    psets_only: bool = False,
    qtos_only: bool = False,
    should_inherit: bool = True,
    verbose: bool = False
) -> dict[str, dict[str, Any]]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | *required* | Any IFC element (IfcWall, IfcDoor, IfcBuildingStorey, etc.) |
| `psets_only` | `bool` | `False` | If `True`, exclude quantity sets (IfcElementQuantity) |
| `qtos_only` | `bool` | `False` | If `True`, return only quantity sets |
| `should_inherit` | `bool` | `True` | If `True`, include properties inherited from the element's type |
| `verbose` | `bool` | `False` | If `True`, include additional metadata in output |

**Returns:** `dict` — `{"PsetName": {"PropName": value, "id": int, ...}, ...}`. Each property set dict includes an `"id"` key with the entity's STEP ID.

---

### get_pset()

Retrieves a single property set or individual property value.

```python
ifcopenshell.util.element.get_pset(
    element: ifcopenshell.entity_instance,
    name: str,
    prop: str | None = None,
    psets_only: bool = False,
    qtos_only: bool = False,
    should_inherit: bool = True,
    verbose: bool = False
) -> Any | dict | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | *required* | Any IFC element |
| `name` | `str` | *required* | Property set name (e.g., `"Pset_WallCommon"`) |
| `prop` | `str \| None` | `None` | Specific property name within the set. Returns entire pset dict if `None` |
| `psets_only` | `bool` | `False` | Exclude quantity sets |
| `qtos_only` | `bool` | `False` | Only quantity sets |
| `should_inherit` | `bool` | `True` | Include type-inherited properties |
| `verbose` | `bool` | `False` | Include metadata |

**Returns:** Property value (if `prop` specified), property set dict (if only `name`), or `None` if not found.

---

### get_type()

Returns the type element (e.g., IfcWallType) associated with an occurrence.

```python
ifcopenshell.util.element.get_type(
    element: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

**Returns:** The type entity, or `None` if untyped.

---

### get_container()

Returns the spatial structure container (IfcBuildingStorey, IfcSpace, IfcBuilding, etc.).

```python
ifcopenshell.util.element.get_container(
    element: ifcopenshell.entity_instance,
    should_get_direct: bool = False,
    ifc_class: str | None = None
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | *required* | The element to query |
| `should_get_direct` | `bool` | `False` | If `True`, return only the directly containing structure |
| `ifc_class` | `str \| None` | `None` | Filter by specific class (e.g., `"IfcBuildingStorey"`) |

**Returns:** The container entity, or `None` if uncontained.

---

### get_material()

Returns the material assignment for an element.

```python
ifcopenshell.util.element.get_material(
    element: ifcopenshell.entity_instance,
    should_skip_usage: bool = False,
    should_inherit: bool = True
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | *required* | The element to query |
| `should_skip_usage` | `bool` | `False` | If `True`, return the set directly instead of the usage entity |
| `should_inherit` | `bool` | `True` | If `True`, fall back to type material when element has none |

**Returns:** Material entity (IfcMaterial, IfcMaterialLayerSetUsage, IfcMaterialProfileSet, etc.) or `None`.

---

### get_materials()

Returns a flat list of individual IfcMaterial entities for an element.

```python
ifcopenshell.util.element.get_materials(
    element: ifcopenshell.entity_instance,
    should_inherit: bool = True
) -> list[ifcopenshell.entity_instance]
```

**Returns:** List of `IfcMaterial` entities. Empty list if no material assigned.

---

### get_decomposition()

Returns all child elements via decomposition (IfcRelAggregates) and containment (IfcRelContainedInSpatialStructure).

```python
ifcopenshell.util.element.get_decomposition(
    element: ifcopenshell.entity_instance,
    is_recursive: bool = True
) -> set[ifcopenshell.entity_instance]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | *required* | Parent element |
| `is_recursive` | `bool` | `True` | If `True`, traverse full hierarchy recursively |

**Returns:** Set of child entities.

---

### get_aggregate()

Returns the parent aggregate of an element.

```python
ifcopenshell.util.element.get_aggregate(
    element: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

**Returns:** Parent entity (e.g., IfcBuilding for a storey), or `None`.

---

### get_grouped_by()

Returns elements that belong to a group.

```python
ifcopenshell.util.element.get_grouped_by(
    element: ifcopenshell.entity_instance,
    is_recursive: bool = True
) -> list[ifcopenshell.entity_instance]
```

**Returns:** List of group member entities.

---

### copy()

Creates a shallow copy of an element with a regenerated GlobalId.

```python
ifcopenshell.util.element.copy(
    ifc_file: ifcopenshell.file,
    element: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```

**Returns:** The new entity instance.

---

### copy_deep()

Creates a deep copy of an element and all directly related subelements.

```python
ifcopenshell.util.element.copy_deep(
    ifc_file: ifcopenshell.file,
    element: ifcopenshell.entity_instance,
    exclude: list[str] | None = None,
    exclude_callback: Callable | None = None,
    copied_entities: dict | None = None
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_file` | `ifcopenshell.file` | *required* | The IFC file |
| `element` | `entity_instance` | *required* | Element to deep-copy |
| `exclude` | `list[str] \| None` | `None` | IFC class names to skip during copy |
| `exclude_callback` | `Callable \| None` | `None` | Custom function to decide exclusions |
| `copied_entities` | `dict \| None` | `None` | Internal tracking of already-copied entities |

**Returns:** The new deep-copied entity.

---

## ifcopenshell.util.selector

### filter_elements()

Filters IFC elements using a CSS-inspired query syntax.

```python
ifcopenshell.util.selector.filter_elements(
    ifc_file: ifcopenshell.file,
    query: str,
    elements: set[ifcopenshell.entity_instance] | None = None,
    edit_in_place: bool = False
) -> set[ifcopenshell.entity_instance]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_file` | `ifcopenshell.file` | *required* | The IFC model |
| `query` | `str` | *required* | Selector query string |
| `elements` | `set \| None` | `None` | Pre-filtered element set to further narrow |
| `edit_in_place` | `bool` | `False` | If `True`, modify the `elements` set in place |

**Returns:** Set of matching elements.

---

### get_element_value()

Retrieves a value from an element using query syntax.

```python
ifcopenshell.util.selector.get_element_value(
    element: ifcopenshell.entity_instance,
    query: str
) -> Any
```

**Returns:** The queried value, or `None`.

---

### set_element_value()

Sets a value on an element using query syntax.

```python
ifcopenshell.util.selector.set_element_value(
    ifc_file: ifcopenshell.file,
    element: ifcopenshell.entity_instance | dict | Iterable | None,
    query: str | list[str],
    value: Any,
    *,
    concat: str = ", "
) -> None
```

---

## ifcopenshell.util.placement

### get_local_placement()

Converts an IfcLocalPlacement into an absolute 4x4 transformation matrix by resolving the full placement chain.

```python
ifcopenshell.util.placement.get_local_placement(
    placement: ifcopenshell.entity_instance | None = None
) -> npt.NDArray[np.float64]
```

**Returns:** 4x4 numpy array. `matrix[:3, 3]` = XYZ translation. `matrix[:3, :3]` = rotation.

---

### get_axis2placement()

Parses an IfcAxis2Placement (2D or 3D) into a 4x4 matrix.

```python
ifcopenshell.util.placement.get_axis2placement(
    placement: ifcopenshell.entity_instance
) -> npt.NDArray[np.float64]
```

**Returns:** 4x4 numpy array.

---

### get_storey_elevation()

Returns the Z elevation of a building storey in project units.

```python
ifcopenshell.util.placement.get_storey_elevation(
    storey: ifcopenshell.entity_instance
) -> float
```

**Returns:** Elevation in project units. Checks placement Z first, falls back to `Elevation` attribute.

---

### rotation()

Creates a 4x4 rotation matrix around a specified axis.

```python
ifcopenshell.util.placement.rotation(
    angle: float,
    axis: Literal["X", "Y", "Z"],
    is_degrees: bool = True
) -> npt.NDArray[np.float64]
```

---

## ifcopenshell.util.unit

### calculate_unit_scale()

Returns a scale factor to convert from IFC project units to SI units.

```python
ifcopenshell.util.unit.calculate_unit_scale(
    ifc_file: ifcopenshell.file,
    unit_type: str = "LENGTHUNIT"
) -> float
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_file` | `ifcopenshell.file` | *required* | The IFC model |
| `unit_type` | `str` | `"LENGTHUNIT"` | IFC unit type (e.g., `"LENGTHUNIT"`, `"AREAUNIT"`, `"VOLUMEUNIT"`) |

**Returns:** `float` — Multiply raw values by this factor to get SI units. Example: millimetres → `0.001`.

---

### convert()

Converts a value between unit systems.

```python
ifcopenshell.util.unit.convert(
    value: float,
    from_prefix: str | None,
    from_unit: str,
    to_prefix: str | None,
    to_unit: str
) -> float
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | `float` | The value to convert |
| `from_prefix` | `str \| None` | SI prefix of source unit (`"MILLI"`, `"CENTI"`, `None` for base) |
| `from_unit` | `str` | Source unit name (`"METRE"`, `"FOOT"`, etc.) |
| `to_prefix` | `str \| None` | SI prefix of target unit |
| `to_unit` | `str` | Target unit name |

**Returns:** Converted value.

---

### get_project_unit()

Returns the default project unit for a given unit type.

```python
ifcopenshell.util.unit.get_project_unit(
    ifc_file: ifcopenshell.file,
    unit_type: str,
    use_cache: bool = False
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_file` | `ifcopenshell.file` | *required* | The IFC model |
| `unit_type` | `str` | *required* | `"LENGTHUNIT"`, `"AREAUNIT"`, `"VOLUMEUNIT"`, `"PLANEANGLEUNIT"`, etc. |
| `use_cache` | `bool` | `False` | Cache result for repeated calls |

**Returns:** IfcNamedUnit entity or `None`.

---

### get_full_unit_name()

Returns human-readable unit name from unit entity.

```python
ifcopenshell.util.unit.get_full_unit_name(
    unit: ifcopenshell.entity_instance
) -> str
```

**Returns:** e.g., `"MILLI METRE"`, `"SQUARE METRE"`.

---

## ifcopenshell.util.date

### ifc2datetime()

Converts any IFC date/time representation to a Python datetime/date/timedelta.

```python
ifcopenshell.util.date.ifc2datetime(
    element: str | int | ifcopenshell.entity_instance
) -> datetime.datetime | datetime.date | datetime.timedelta
```

| Input Type | Example | Python Output |
|------------|---------|---------------|
| `int` (IfcTimeStamp) | `1718438400` | `datetime.datetime` |
| `str` (IfcDate) | `"2024-06-15"` | `datetime.date` |
| `str` (IfcDateTime) | `"2024-06-15T10:30:00"` | `datetime.datetime` |
| `str` (IfcDuration) | `"P30D"` | `datetime.timedelta` |
| entity (IfcCalendarDate) | IFC2X3 entity | `datetime.date` |

---

### datetime2ifc()

Converts Python datetime to IFC format.

```python
ifcopenshell.util.date.datetime2ifc(
    dt: datetime.date | str | None,
    ifc_type: Literal[
        "IfcDuration", "IfcTimeStamp", "IfcDateTime",
        "IfcDate", "IfcTime", "IfcCalendarDate", "IfcLocalTime"
    ]
) -> int | str | dict[str, Any] | None
```

| `ifc_type` | Output Type | Example Output |
|------------|-------------|----------------|
| `"IfcTimeStamp"` | `int` | `1718438400` |
| `"IfcDate"` | `str` | `"2024-06-15"` |
| `"IfcDateTime"` | `str` | `"2024-06-15T10:30:00"` |
| `"IfcTime"` | `str` | `"10:30:00"` |
| `"IfcDuration"` | `str` | `"P30D"` |

---

## ifcopenshell.util.shape

All shape utility functions operate on **processed geometry** objects from `ifcopenshell.geom.create_shape()`. They do NOT accept raw IFC entity instances.

### Core Functions

```python
ifcopenshell.util.shape.get_volume(geometry) -> float
ifcopenshell.util.shape.get_area(geometry) -> float
ifcopenshell.util.shape.get_footprint_area(geometry) -> float
ifcopenshell.util.shape.get_side_area(geometry) -> float
ifcopenshell.util.shape.get_outer_surface_area(geometry) -> float
ifcopenshell.util.shape.get_vertices(geometry) -> npt.NDArray
ifcopenshell.util.shape.get_faces(geometry) -> npt.NDArray
ifcopenshell.util.shape.get_edges(geometry) -> npt.NDArray
ifcopenshell.util.shape.get_normals(geometry) -> npt.NDArray
ifcopenshell.util.shape.get_bbox(geometry) -> tuple[npt.NDArray, npt.NDArray]
ifcopenshell.util.shape.get_bbox_centroid(geometry) -> tuple[float, float, float]
ifcopenshell.util.shape.get_bottom_elevation(geometry) -> float
ifcopenshell.util.shape.get_top_elevation(geometry) -> float
ifcopenshell.util.shape.get_x(geometry) -> float
ifcopenshell.util.shape.get_y(geometry) -> float
ifcopenshell.util.shape.get_z(geometry) -> float
```

---

## ifcopenshell.util.classification

### get_references()

Returns classification references assigned to an element.

```python
ifcopenshell.util.classification.get_references(
    element: ifcopenshell.entity_instance,
    should_inherit: bool = True
) -> set[ifcopenshell.entity_instance]
```

**Returns:** Set of IfcClassificationReference entities.

---

### get_classification()

Returns the IfcClassification entity from a classification reference.

```python
ifcopenshell.util.classification.get_classification(
    reference: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```

---

## ifcopenshell.util.cost

### Key Functions

```python
ifcopenshell.util.cost.get_cost_items_for_product(
    product: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]

ifcopenshell.util.cost.get_cost_values(
    cost_item: ifcopenshell.entity_instance
) -> list[dict]

ifcopenshell.util.cost.get_cost_schedule(
    cost_item: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance

ifcopenshell.util.cost.get_root_cost_items(
    cost_schedule: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]

ifcopenshell.util.cost.get_nested_cost_items(
    cost_item: ifcopenshell.entity_instance,
    is_deep: bool = False
) -> list[ifcopenshell.entity_instance]

ifcopenshell.util.cost.get_total_quantity(
    root_element: ifcopenshell.entity_instance
) -> float | None
```

---

## ifcopenshell.util.sequence

### Key Functions

```python
ifcopenshell.util.sequence.get_root_tasks(
    work_schedule: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]

ifcopenshell.util.sequence.get_nested_tasks(
    task: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]

ifcopenshell.util.sequence.get_parent_task(
    task: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None

ifcopenshell.util.sequence.get_calendar(
    task: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None

ifcopenshell.util.sequence.get_tasks_for_product(
    product: ifcopenshell.entity_instance,
    schedule: ifcopenshell.entity_instance
) -> tuple[list, list]

ifcopenshell.util.sequence.count_working_days(
    start: datetime.date,
    finish: datetime.date,
    calendar: ifcopenshell.entity_instance
) -> int

ifcopenshell.util.sequence.is_working_day(
    day: datetime.date,
    calendar: ifcopenshell.entity_instance
) -> bool

ifcopenshell.util.sequence.offset_date(
    start: datetime.date,
    duration: datetime.timedelta,
    duration_type: str,
    calendar: ifcopenshell.entity_instance
) -> datetime.date
```

---

## ifcopenshell.util.attribute

### get_primitive_type()

Returns the Python primitive type for an IFC attribute definition.

```python
ifcopenshell.util.attribute.get_primitive_type(
    attribute_or_data_type: Any
) -> PrimitiveTypeOutput
```

---

### get_enum_items()

Returns all valid enum values for an enumerated attribute.

```python
ifcopenshell.util.attribute.get_enum_items(
    attribute: Any
) -> tuple[str, ...]
```

---

### get_select_items()

Returns all valid declarations for a select-type attribute.

```python
ifcopenshell.util.attribute.get_select_items(
    attribute: Any
) -> tuple[ifcopenshell.ifcopenshell_wrapper.declaration, ...]
```
