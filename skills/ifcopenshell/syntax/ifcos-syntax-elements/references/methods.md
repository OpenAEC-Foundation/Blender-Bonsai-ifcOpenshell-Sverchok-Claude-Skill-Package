# Element Traversal — API Method Signatures

## File-Level Query Methods

### model.by_type()

```python
file.by_type(type: str, include_subtypes: bool = True) -> tuple[ifcopenshell.entity_instance, ...]
```

Returns all entities of a given IFC class.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `type` | `str` | required | IFC entity class name (e.g., `"IfcWall"`, `"IfcProduct"`) |
| `include_subtypes` | `bool` | `True` | If True, includes subclasses in the IFC inheritance tree |

**Returns:** Tuple of `entity_instance` objects. Returns empty tuple `()` if no matches.

**Notes:**
- Results are cached internally after the first call per type. Subsequent calls are fast.
- Uses the IFC inheritance tree when `include_subtypes=True`.
- The `type` parameter is case-sensitive and MUST match the IFC class name exactly.

---

### model.by_id()

```python
file.by_id(id: int) -> ifcopenshell.entity_instance
```

Returns a single entity by its numeric STEP file ID.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | `int` | required | STEP file ID (the `#123` number in .ifc files) |

**Returns:** Single `entity_instance`.

**Raises:** `RuntimeError` if no entity with that ID exists.

---

### model.by_guid()

```python
file.by_guid(guid: str) -> ifcopenshell.entity_instance
```

Returns a single entity by its IFC GlobalId.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `guid` | `str` | required | 22-character IFC GUID (base64-encoded UUID) |

**Returns:** Single `entity_instance`.

**Raises:** `RuntimeError` if no entity with that GlobalId exists.

**Notes:**
- Only `IfcRoot`-derived entities have GlobalIds.
- IFC GUIDs use characters: `0-9`, `A-Z`, `a-z`, `_`, `$`.

---

### model.get_inverse()

```python
file.get_inverse(entity: ifcopenshell.entity_instance) -> set[ifcopenshell.entity_instance]
```

Returns all entities that REFERENCE the given entity (inverse references).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `entity` | `entity_instance` | required | The entity to find references to |

**Returns:** Set of `entity_instance` objects that hold a reference to the given entity.

**Notes:**
- This traverses "upward" or "sideways" in the IFC relationship graph.
- Common relationship types found via inverse: `IfcRelContainedInSpatialStructure`, `IfcRelDefinesByProperties`, `IfcRelDefinesByType`, `IfcRelAssociatesMaterial`, `IfcRelVoidsElement`, `IfcRelAggregates`.

---

### model.traverse()

```python
file.traverse(entity: ifcopenshell.entity_instance, max_levels: int = -1) -> list[ifcopenshell.entity_instance]
```

Recursively traverses all entities referenced BY the given entity (forward references).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `entity` | `entity_instance` | required | The starting entity |
| `max_levels` | `int` | `-1` | Maximum recursion depth. `-1` means unlimited. |

**Returns:** List of all referenced entities (including the starting entity).

**Notes:**
- This is the opposite of `get_inverse()` — it follows forward references, not backward.
- Can return very large result sets for complex entities. Use `max_levels` to limit depth.

---

## Entity Instance Methods

### entity.is_a()

```python
entity_instance.is_a(type: str = None, with_schema: bool = False) -> str | bool
```

Returns the IFC class name or checks inheritance.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `type` | `str` | `None` | If provided, checks if entity is this type or a subtype |
| `with_schema` | `bool` | `False` | If True, includes schema prefix (e.g., `"IFC4.IfcWall"`) |

**Returns:**
- No arguments: `str` — the IFC class name (e.g., `"IfcWall"`)
- With `type` argument: `bool` — True if entity is the given type or inherits from it

**Notes:**
- Class name comparison is case-insensitive.
- Inheritance check follows the full IFC class hierarchy.

---

### entity.id()

```python
entity_instance.id() -> int
```

Returns the STEP file numerical identifier.

**Returns:** `int` — the `#123` number from the .ifc file.

**Notes:**
- STEP IDs are NOT persistent across file re-exports.
- ALWAYS use GlobalId for persistent identification.

---

### entity.get_info()

```python
entity_instance.get_info(
    include_identifier: bool = True,
    recursive: bool = False,
    return_type: type = dict,
    ignore: tuple = (),
    scalar_only: bool = False
) -> dict
```

Returns all entity attributes as a dictionary.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `include_identifier` | `bool` | `True` | Include `"id"` and `"type"` keys in result |
| `recursive` | `bool` | `False` | Recursively expand referenced entities into dicts |
| `return_type` | `type` | `dict` | Container type for the result |
| `ignore` | `tuple` | `()` | Attribute names to exclude |
| `scalar_only` | `bool` | `False` | Only return primitive values, skip entity references |

**Returns:** Dictionary with keys `"id"`, `"type"`, and all entity attribute names.

**Notes:**
- `recursive=True` is extremely expensive on complex entities. NEVER use in loops over many elements.
- `get_info_2()` is a faster variant but only supports `recursive=True`, `return_type=dict`, and empty `ignore`.

---

### entity.attribute_name()

```python
entity_instance.attribute_name(attr_idx: int) -> str
```

Returns the name of a positional attribute by index.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `attr_idx` | `int` | required | Zero-based attribute index |

**Returns:** `str` — attribute name.

---

### entity.attribute_type()

```python
entity_instance.attribute_type(attr: int | str) -> str
```

Returns the data type of an attribute.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `attr` | `int \| str` | required | Attribute index or name |

**Returns:** `str` — the IFC data type name.

---

### entity.to_string()

```python
entity_instance.to_string(valid_spf: bool = True) -> str
```

Returns STEP Physical File notation string representation.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `valid_spf` | `bool` | `True` | Conform to STEP Physical File format |

**Returns:** `str` — e.g., `"#42=IFCWALL('2hK7x...',...)"`.

---

### entity.unwrap_value() (static)

```python
entity_instance.unwrap_value(v) -> Any
```

Extracts the underlying Python value from a wrapped IFC value.

**Notes:**
- This is the programmatic equivalent of accessing `.wrappedValue` on `IfcPropertySingleValue.NominalValue`.

---

## GUID Module Functions

### ifcopenshell.guid.new()

```python
ifcopenshell.guid.new() -> str
```

Generates a new random IFC GlobalId (22-character base64-encoded UUID).

**Returns:** `str` — 22-character IFC GUID.

---

### ifcopenshell.guid.expand()

```python
ifcopenshell.guid.expand(ifc_guid: str) -> str
```

Converts an IFC GUID to a standard UUID string.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_guid` | `str` | required | 22-character IFC GUID |

**Returns:** `str` — standard UUID string (e.g., `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"`).

---

### ifcopenshell.guid.compress()

```python
ifcopenshell.guid.compress(uuid_str: str) -> str
```

Converts a standard UUID string to an IFC GUID.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `uuid_str` | `str` | required | Standard UUID string |

**Returns:** `str` — 22-character IFC GUID.

---

## Utility Module: ifcopenshell.util.element

### get_psets()

```python
ifcopenshell.util.element.get_psets(
    element: ifcopenshell.entity_instance,
    psets_only: bool = False,
    qtos_only: bool = False
) -> dict[str, dict[str, Any]]
```

Returns all property sets (and optionally quantity sets) for an element.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | required | The IFC element |
| `psets_only` | `bool` | `False` | Only return IfcPropertySet, not IfcElementQuantity |
| `qtos_only` | `bool` | `False` | Only return IfcElementQuantity |

**Returns:** `dict` — `{pset_name: {prop_name: value, ...}, ...}`

---

### get_type()

```python
ifcopenshell.util.element.get_type(element: ifcopenshell.entity_instance) -> ifcopenshell.entity_instance | None
```

Returns the type object for an element, or None.

---

### get_container()

```python
ifcopenshell.util.element.get_container(element: ifcopenshell.entity_instance) -> ifcopenshell.entity_instance | None
```

Returns the spatial container (typically IfcBuildingStorey) for an element, or None.

---

### get_material()

```python
ifcopenshell.util.element.get_material(element: ifcopenshell.entity_instance) -> ifcopenshell.entity_instance | None
```

Returns the material association for an element, or None. Result may be IfcMaterial, IfcMaterialLayerSet, IfcMaterialLayerSetUsage, IfcMaterialProfileSet, or IfcMaterialConstituentSet.

---

### get_materials()

```python
ifcopenshell.util.element.get_materials(element: ifcopenshell.entity_instance) -> list[ifcopenshell.entity_instance]
```

Returns all IfcMaterial entities associated with an element.

---

### get_decomposition()

```python
ifcopenshell.util.element.get_decomposition(element: ifcopenshell.entity_instance) -> list[ifcopenshell.entity_instance]
```

Returns child elements of an aggregate (e.g., storeys in a building).

---

### get_aggregate()

```python
ifcopenshell.util.element.get_aggregate(element: ifcopenshell.entity_instance) -> ifcopenshell.entity_instance | None
```

Returns the parent aggregate of an element (inverse of decomposition), or None.

---

### get_grouped_by()

```python
ifcopenshell.util.element.get_grouped_by(group: ifcopenshell.entity_instance) -> list[ifcopenshell.entity_instance]
```

Returns all elements that are members of an IfcGroup.

---

## Utility Module: ifcopenshell.util.selector

### filter_elements()

```python
ifcopenshell.util.selector.filter_elements(
    ifc_file: ifcopenshell.file,
    query: str
) -> list[ifcopenshell.entity_instance]
```

Selects elements using a CSS-like query syntax.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_file` | `ifcopenshell.file` | required | The IFC file to query |
| `query` | `str` | required | CSS-like selector string |

**Query Syntax:**

| Pattern | Meaning |
|---------|---------|
| `IfcWall` | Select by IFC class |
| `Name="X"` | Filter by Name attribute |
| `Name *= "X"` | Name contains "X" |
| `/PsetName/.PropName = value` | Filter by property value |
| `type="TypeName"` | Filter by type object name |
| `container="ContainerName"` | Filter by spatial container |
| `material="MaterialName"` | Filter by material name |

**Notes:**
- ALWAYS use `filter_elements()`. The older `Selector().parse()` API is deprecated.
