# Performance Method Signatures

Complete API reference for IfcOpenShell performance-critical methods. All signatures verified against the [official IfcOpenShell documentation](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html).

---

## ifcopenshell.geom.iterator

Creates a multi-threaded geometry iterator for batch processing of IFC elements.

```python
ifcopenshell.geom.iterator(
    settings: ifcopenshell.geom.settings,
    file_or_filename: Union[ifcopenshell.file, str],
    num_threads: int = 1,
    include: Optional[Union[list, tuple]] = None,
    exclude: Optional[Union[list, tuple]] = None,
    geometry_library: str = "opencascade"
) -> ifcopenshell.geom.iterator
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `settings` | `ifcopenshell.geom.settings` | *required* | Geometry processing settings |
| `file_or_filename` | `ifcopenshell.file \| str` | *required* | Loaded model or path to IFC file |
| `num_threads` | `int` | `1` | Number of parallel threads. Use `multiprocessing.cpu_count()` for max parallelism |
| `include` | `list \| tuple \| None` | `None` | Whitelist of elements to process. Pass result of `by_type()` to filter |
| `exclude` | `list \| tuple \| None` | `None` | Blacklist of elements to skip |
| `geometry_library` | `str` | `"opencascade"` | Geometry engine: `"opencascade"` or `"cgal"` |

### Return Value

`ifcopenshell.geom.iterator` — Iterator object. Call `.initialize()` before use.

### Iterator Methods

| Method | Return | Description |
|--------|--------|-------------|
| `.initialize()` | `bool` | Prepare iterator. Returns `False` if no processable elements |
| `.get()` | shape object | Get current shape. Access `.geometry.verts`, `.geometry.faces`, `.id` |
| `.next()` | `bool` | Advance to next shape. Returns `False` when exhausted |

### Shape Object Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `shape.id` | `int` | STEP ID of the source element. Use `model.by_id(shape.id)` to get entity |
| `shape.guid` | `str` | GlobalId of the source element |
| `shape.geometry.verts` | `tuple[float]` | Flat vertex coordinates: `[x1,y1,z1, x2,y2,z2, ...]` |
| `shape.geometry.faces` | `tuple[int]` | Flat triangle indices: `[i1,i2,i3, i4,i5,i6, ...]` |
| `shape.geometry.materials` | `tuple` | Material definitions for the shape |
| `shape.geometry.material_ids` | `tuple[int]` | Material index per face |
| `shape.geometry.edges` | `tuple[int]` | Edge vertex index pairs |
| `shape.transformation.matrix.data` | `tuple[float]` | 4x4 transformation matrix (column-major) |

---

## ifcopenshell.geom.create_shape

Creates geometry for a single IFC element.

```python
ifcopenshell.geom.create_shape(
    settings: ifcopenshell.geom.settings,
    inst: ifcopenshell.entity_instance,
    representation: Optional[ifcopenshell.entity_instance] = None
) -> shape_object
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `settings` | `ifcopenshell.geom.settings` | *required* | Geometry processing settings |
| `inst` | `ifcopenshell.entity_instance` | *required* | The IFC element to process |
| `representation` | `ifcopenshell.entity_instance \| None` | `None` | Specific representation to use. When `None`, uses default Body representation |

### Return Value

Shape object with `.geometry.verts`, `.geometry.faces`, and `.transformation.matrix.data`.

### Exceptions

| Exception | Condition |
|-----------|-----------|
| `RuntimeError` | Element has no geometry, unsupported geometry type, or boolean operation failure |
| `TypeError` | Missing or wrong settings argument |

---

## ifcopenshell.geom.settings

Configures geometry processing behavior.

```python
settings = ifcopenshell.geom.settings()
settings.set(setting_name, value)
```

### Available Settings

| Setting Name | Type | Default | Description |
|-------------|------|---------|-------------|
| `"use-world-coords"` | `bool` | `False` | Apply global placement transforms to output coordinates |
| `"weld-vertices"` | `bool` | `False` | Merge duplicate vertices (reduces output size) |
| `"disable-opening-subtractions"` | `bool` | `False` | Skip boolean CSG operations. Major speedup, less accurate |
| `"apply-default-materials"` | `bool` | `False` | Include material information in output |
| `"use-python-opencascade"` | `bool` | `False` | Return BRep (non-triangulated) via pythonOCC |
| `"dimensionality"` | constant | `CURVES_SURFACES_AND_SOLIDS` | Output type: curves, surfaces, solids, or combinations |

### Setting via Constants (Alternative)

```python
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)
settings.set(settings.WELD_VERTICES, True)
settings.set(settings.DISABLE_OPENING_SUBTRACTIONS, True)
```

---

## model.by_type()

Retrieves all entities of a given IFC type from the model.

```python
model.by_type(
    type: str,
    include_subtypes: bool = True
) -> tuple[ifcopenshell.entity_instance]
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `type` | `str` | *required* | IFC entity type name (e.g., `"IfcWall"`, `"IfcProduct"`) |
| `include_subtypes` | `bool` | `True` | Include entities of subtypes. Set `False` for exact type match |

### Return Value

`tuple` of entity instances. Returns empty tuple if no entities of that type exist.

### Performance Notes

- Uses internal class index — O(1) lookup regardless of model size.
- `include_subtypes=False` is slightly faster when you know the exact type.
- Returns a tuple (immutable). Convert to list only when modification is needed.

---

## model.by_id()

Retrieves a single entity by its STEP file integer ID.

```python
model.by_id(id: int) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | `int` | *required* | STEP file entity ID (the `#NNN` number) |

### Return Value

`ifcopenshell.entity_instance` — The entity with the given ID.

### Exceptions

| Exception | Condition |
|-----------|-----------|
| `RuntimeError` | No entity with the given ID exists |

### Performance Notes

- O(1) lookup via internal ID map. Use for direct access when you have the ID.

---

## model.by_guid()

Retrieves a single entity by its IFC GlobalId.

```python
model.by_guid(guid: str) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `guid` | `str` | *required* | 22-character IFC GlobalId |

### Return Value

`ifcopenshell.entity_instance` — The entity with the given GlobalId.

### Performance Notes

- O(1) lookup via GUID index. Use when correlating entities across files or sessions.

---

## ifcopenshell.util.element.get_psets()

Retrieves all property sets and/or quantity sets for an element.

```python
ifcopenshell.util.element.get_psets(
    element: ifcopenshell.entity_instance,
    psets_only: bool = False,
    qtos_only: bool = False,
    should_inherit: bool = True
) -> dict[str, dict[str, Any]]
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `ifcopenshell.entity_instance` | *required* | The IFC element to query |
| `psets_only` | `bool` | `False` | Return only property sets, not quantity sets |
| `qtos_only` | `bool` | `False` | Return only quantity sets, not property sets |
| `should_inherit` | `bool` | `True` | Include properties inherited from element type |

### Return Value

`dict` — Nested dict: `{"PsetName": {"PropName": value, "id": int}, ...}`.

### Performance Notes

- Traverses `IsDefinedBy` relationships on EVERY call. For bulk queries, cache results:

```python
# Build cache once for all walls
cache = {w.id(): ifcopenshell.util.element.get_psets(w)
         for w in model.by_type("IfcWall")}
# O(1) lookup per element from here
```

---

## ifcopenshell.util.selector.filter_elements()

Queries elements using IfcOpenShell's selector syntax.

```python
ifcopenshell.util.selector.filter_elements(
    ifc_file: ifcopenshell.file,
    query: str
) -> list[ifcopenshell.entity_instance]
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_file` | `ifcopenshell.file` | *required* | The IFC model to query |
| `query` | `str` | *required* | Selector query string |

### Query Syntax Examples

| Query | Selects |
|-------|---------|
| `"IfcWall"` | All walls (including subtypes) |
| `"IfcWall, /Pset_WallCommon/.IsExternal=True"` | External walls only |
| `"IfcSlab[PredefinedType=FLOOR]"` | Floor slabs only |

### Performance Notes

- Performs a full scan of matching entities. Slower than building a manual index for repeated lookups.
- Useful for one-off complex queries. For repeated access, build your own index from relationship entities.

---

## Sources

- [IfcOpenShell Geometry Processing](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html)
- [IfcOpenShell Geometry Settings](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_settings.html)
- [IfcOpenShell file API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html)
- [IfcOpenShell util.element API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/element/index.html)
- [IfcOpenShell util.selector API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/selector/index.html)
