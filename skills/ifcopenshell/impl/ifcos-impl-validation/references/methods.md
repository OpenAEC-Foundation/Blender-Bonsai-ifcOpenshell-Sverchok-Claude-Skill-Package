# Validation API Signatures

## ifcopenshell.validate Module

### validate()

```python
ifcopenshell.validate.validate(
    f,                      # ifcopenshell.file instance or str filepath
    logger,                 # logging.Logger instance or json_logger instance
    express_rules=False     # bool — enable EXPRESS WHERE rule checking (10-100x slower)
) -> None
```

Validates an IFC model against the EXPRESS schema definition. Checks entity attributes for type correctness, cardinality, enumerations, aggregations, GUID format, file header structure, and application references. Results are emitted via the logger, NOT returned.

**Validation checks performed:**
- Entity attribute type correctness (string, integer, float, entity reference)
- Inverse attribute cardinality (min/max counts of referencing entities)
- Simple type validation (IfcLabel, IfcLengthMeasure, etc.)
- Select type validation (value is a valid member of the select)
- Enumeration validation (value is a valid enum member)
- Aggregation validation (list/set size constraints)
- GUID format validation (22-character base64 format)
- File header structure validation
- Application reference validation
- EXPRESS WHERE rules (only when `express_rules=True`)

---

### json_logger

```python
ifcopenshell.validate.json_logger() -> json_logger
```

Returns a logger instance that collects validation results as structured data.

**Attributes:**
- `statements` — `list[dict]` — Collected validation log entries
- `state` — `dict` — Current contextual state

**Methods:**
- `log(level, message, *args)` — Record a validation message
- `set_state(key, value)` — Store contextual state (e.g., current entity)

**Statement dict structure:**
```python
{
    "level": str,       # "WARNING", "ERROR", etc.
    "message": str,     # Formatted validation message
    "instance": Any,    # Entity instance reference (if applicable)
    "attribute": str    # Affected attribute name (if applicable)
}
```

---

### LogDetectionHandler

```python
ifcopenshell.validate.LogDetectionHandler() -> LogDetectionHandler
```

A `logging.Handler` subclass that sets a flag when any message is logged.

**Attributes:**
- `message_logged` — `bool` — `True` if any log record was emitted through this handler

**Methods:**
- `emit(record)` — Process a log record (sets `message_logged = True`)

**Usage:** Add to a `logging.Logger` to detect if `validate()` produced any output.

---

### validate_guid()

```python
ifcopenshell.validate.validate_guid(
    guid: str           # 22-character IFC GlobalId string
) -> str | None
```

Returns `None` if the GUID is valid (22-character base64). Returns an error description string if invalid.

---

### validate_ifc_header()

```python
ifcopenshell.validate.validate_ifc_header(
    f,                  # ifcopenshell.file instance
    logger              # logging.Logger instance
) -> None
```

Validates the IFC file header structure (FILE_DESCRIPTION, FILE_NAME, FILE_SCHEMA).

---

### validate_ifc_applications()

```python
ifcopenshell.validate.validate_ifc_applications(
    f,                  # ifcopenshell.file instance
    logger              # logging.Logger instance
) -> None
```

Validates application entity references in the IFC file.

---

### assert_valid()

```python
ifcopenshell.validate.assert_valid(
    attr_type,          # EXPRESS attribute type definition
    val,                # Attribute value to validate
    schema,             # Schema module (e.g., ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4"))
    no_throw=False,     # bool — if True, return error instead of raising
    attr=None           # str — attribute name for error context
) -> None | str
```

Validates a single attribute value against its EXPRESS schema type definition. When `no_throw=False` (default), raises `ValidationError` on failure. When `no_throw=True`, returns an error string or `None`.

---

### assert_valid_inverse()

```python
ifcopenshell.validate.assert_valid_inverse(
    attr,               # EXPRESS inverse attribute definition
    val,                # Inverse attribute value (set of entities)
    schema              # Schema module
) -> None
```

Validates inverse attribute cardinality constraints. Raises `ValidationError` if the count of referencing entities violates the schema.

---

### log_internal_cpp_errors()

```python
ifcopenshell.validate.log_internal_cpp_errors(
    f,                  # ifcopenshell.file instance
    logger              # logging.Logger instance
) -> None
```

Captures and logs errors from the C++ STEP parser that occurred during file loading.

---

## ifctester Module

### ifctester.open()

```python
ifctester.open(
    filepath: str,      # Path to IDS XML file
    validate: bool = False  # Validate IDS XML against schema on load
) -> ifctester.ids.Ids
```

Loads an IDS (Information Delivery Specification) file from disk.

---

### ifctester.ids.from_string()

```python
ifctester.ids.from_string(
    xml: str,           # IDS XML content as string
    validate: bool = False  # Validate against IDS XML schema
) -> ifctester.ids.Ids
```

Parses an IDS specification from an XML string.

---

### ifctester.ids.Ids

```python
ifctester.ids.Ids(
    title: str | None = "Untitled",
    copyright: str | None = None,
    version: str | None = None,
    description: str | None = None,
    author: str | None = None,
    date: str | None = None,
    purpose: str | None = None,
    milestone: str | None = None
)
```

**Key attributes:**
- `specifications_` — `list[Specification]` — List of validation specifications
- `info_` — `dict` — Document metadata
- `filepath_` — `str` — Source file path
- `filename_` — `str` — Source file name

**Key methods:**

| Method | Signature | Returns | Purpose |
|--------|-----------|---------|---------|
| `validate` | `(ifc_file, should_filter_version=False, filepath=None)` | `None` | Run all specifications against IFC file |
| `to_xml` | `(filepath="output.xml")` | `None` | Write IDS to XML file |
| `to_string` | `()` | `str` | Serialize IDS to XML string |
| `asdict` | `()` | `dict` | Convert to dictionary |
| `parse` | `(data)` | `None` | Parse IDS data into object |

---

### ifctester.ids.Specification

```python
ifctester.ids.Specification(
    name: str = "Unnamed",
    minOccurs: int = 0,
    maxOccurs: int | str = "unbounded",
    ifcVersion: list[str] | None = None,
    identifier: str | None = None,
    description: str | None = None,
    instructions: str | None = None
)
```

**Key attributes:**
- `applicability_` — `list` — Facets defining which entities this spec applies to
- `requirements_` — `list` — Facets defining what data entities must have
- `applicable_entities_` — `list` — Entities matching applicability (populated after validate)
- `passed_entities_` — `list` — Entities passing all requirements
- `failed_entities_` — `list` — Entities failing requirements
- `status` — `bool | None` — Overall pass/fail status

**Key methods:**

| Method | Signature | Returns | Purpose |
|--------|-----------|---------|---------|
| `validate` | `(ifc_file, should_filter_version=False)` | `None` | Validate this specification |
| `check_ifc_version` | `(ifc_file)` | `bool` | Check IFC version compatibility |
| `get_usage` | `()` | `Cardinality` | Get occurrence constraints |
| `set_usage` | `(usage)` | `None` | Set required/optional/prohibited |
| `reset_status` | `()` | `None` | Clear validation results |

---

## IDS Facet Types

### Entity Facet

```python
ifctester.facet.Entity(
    name: str,                  # IFC class name (e.g., "IfcWall")
    predefinedType: str = None, # Optional predefined type filter
    instructions: str = None    # Human-readable instructions
)
```

### Attribute Facet

```python
ifctester.facet.Attribute(
    name: str,                  # Attribute name (e.g., "Name")
    value: str = None,          # Expected value or pattern
    cardinality: str = "required",  # "required", "optional", "prohibited"
    instructions: str = None
)
```

### Classification Facet

```python
ifctester.facet.Classification(
    value: str = None,          # Classification reference value
    system: str = None,         # Classification system name
    uri: str = None,            # Classification system URI
    cardinality: str = "required",
    instructions: str = None
)
```

### Property Facet

```python
ifctester.facet.Property(
    propertySet: str,           # Property set name (e.g., "Pset_WallCommon")
    baseName: str,              # Property name (e.g., "FireRating")
    value: str = None,          # Expected value
    dataType: str = None,       # IFC data type (e.g., "IfcLabel")
    uri: str = None,            # Property set template URI
    cardinality: str = "required",
    instructions: str = None
)
```

### Material Facet

```python
ifctester.facet.Material(
    value: str = None,          # Material name or pattern
    uri: str = None,            # Material URI
    cardinality: str = "required",
    instructions: str = None
)
```

### PartOf Facet

```python
ifctester.facet.PartOf(
    name: str = None,           # Parent entity IFC class
    predefinedType: str = None, # Parent predefined type
    relation: str = None,       # Relationship type (e.g., "IfcRelContainedInSpatialStructure")
    cardinality: str = "required",
    instructions: str = None
)
```

All facets implement:
- `filter(ifc_file, elements) -> list` — Filter/validate elements against facet criteria

---

## IDS Reporter Classes

### Console Reporter

```python
ifctester.reporter.Console(
    ids: ifctester.ids.Ids,
    use_colour: bool = True
)
```

Methods: `report()`, `to_string()`, `write()`

### Json Reporter

```python
ifctester.reporter.Json(
    ids: ifctester.ids.Ids,
    hide_skipped: bool = False
)
```

Methods: `report()`, `to_string()`, `to_file(filepath)`

### Html Reporter

```python
ifctester.reporter.Html(
    ids: ifctester.ids.Ids,
    hide_skipped: bool = False
)
```

Methods: `report()`, `to_string()`, `to_file(filepath)`

### Bcf Reporter

```python
ifctester.reporter.Bcf(
    ids: ifctester.ids.Ids,
    hide_skipped: bool = False
)
```

Methods: `report()`, `to_file(filepath)`

### Ods Reporter

```python
ifctester.reporter.Ods(
    ids: ifctester.ids.Ids,
    excel_safe: bool = False
)
```

Methods: `report()`, `to_file(filepath)`

### Txt Reporter

```python
ifctester.reporter.Txt(
    ids: ifctester.ids.Ids
)
```

Methods: `report()`, `to_string()`, `to_file(filepath)`

---

## Georeference API Functions

### add_georeferencing()

```python
ifcopenshell.api.run("georeference.add_georeferencing", model,
    ifc_class: str = "IfcMapConversion",   # "IfcMapConversion" or "IfcMapConversionScaled" (IFC4X3 only)
    name: str = "EPSG:3857"                # Default CRS name
) -> None
```

Creates empty `IfcMapConversion` (or `IfcMapConversionScaled`) and `IfcProjectedCRS` entities. MUST be called before `edit_georeferencing`.

---

### edit_georeferencing()

```python
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    coordinate_operation: dict[str, Any] | None = None,
    projected_crs: dict[str, Any] | None = None
) -> None
```

**coordinate_operation dict keys:**
- `Eastings` — `float` — False origin easting
- `Northings` — `float` — False origin northing
- `OrthogonalHeight` — `float` — Height above reference
- `XAxisAbscissa` — `float` — cos(rotation angle) for Project North
- `XAxisOrdinate` — `float` — sin(rotation angle) for Project North
- `Scale` — `float` — Scale factor (1.0 = no scaling)

**projected_crs dict keys:**
- `Name` — `str` — EPSG code (e.g., `"EPSG:28992"`)
- `Description` — `str` — CRS description
- `GeodeticDatum` — `str` — Datum name
- `VerticalDatum` — `str` — Vertical datum name
- `MapProjection` — `str` — Projection method
- `MapZone` — `str` — Map zone identifier
- `MapUnit` — IFC unit reference (IFC4+) or string (IFC2X3)

---

### edit_true_north()

```python
ifcopenshell.api.run("georeference.edit_true_north", model,
    true_north: tuple[float, float] | float | None = 0.0
) -> None
```

Accepts a rotation angle in decimal degrees (anticlockwise from Y-axis positive) or a 2D direction vector `(x, y)`.

---

### edit_wcs()

```python
ifcopenshell.api.run("georeference.edit_wcs", model,
    x: float = 0.0,        # WCS X offset
    y: float = 0.0,        # WCS Y offset
    z: float = 0.0,        # WCS Z offset
    rotation: float = 0.0, # WCS rotation in radians
    is_si: bool = True      # True = meters, False = file native units
) -> None
```

Adjusts the World Coordinate System origin. **NEVER** move WCS from (0,0,0) without a specific surveying reason.

---

### remove_georeferencing()

```python
ifcopenshell.api.run("georeference.remove_georeferencing", model) -> None
```

Removes all georeferencing data (IfcMapConversion, IfcProjectedCRS) from the model. In IFC2X3, removes the corresponding property sets.
