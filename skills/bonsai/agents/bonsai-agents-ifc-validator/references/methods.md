# Validation Methods Reference

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Python 3.11
> **IFC Schemas**: IFC2X3, IFC4, IFC4X3

---

## Schema Validation — `ifcopenshell.validate`

### `ifcopenshell.validate.validate(f, logger, express_rules=False)`

Validates an IFC model against its EXPRESS schema definition.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `f` | `ifcopenshell.file` | required | The IFC model to validate |
| `logger` | `logging.Logger` or `json_logger` | required | Logger instance that receives validation results |
| `express_rules` | `bool` | `False` | When `True`, also evaluates EXPRESS WHERE rules (10-100x slower) |

**Returns**: `None`. All results are sent to `logger`. Does NOT raise exceptions on invalid models.

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("project.ifc")
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

errors = [s for s in json_log.statements if s["level"] == "ERROR"]
warnings = [s for s in json_log.statements if s["level"] == "WARNING"]
```

### `ifcopenshell.validate.json_logger()`

Creates a structured logger that stores validation results as a list of dictionaries.

**Returns**: Logger-compatible object with `.statements` attribute (list of dicts).

Each statement dict contains:

| Key | Type | Description |
|-----|------|-------------|
| `"level"` | `str` | `"ERROR"` or `"WARNING"` |
| `"message"` | `str` | Human-readable validation message |
| `"instance"` | `entity_instance` or `None` | The IFC entity that triggered the issue |

### `ifcopenshell.validate.LogDetectionHandler()`

A `logging.Handler` subclass that sets a boolean flag when any message is logged.

| Attribute | Type | Description |
|-----------|------|-------------|
| `.message_logged` | `bool` | `True` if any validation issue was logged, `False` otherwise |

```python
import logging
import ifcopenshell.validate

logger = logging.getLogger("ifcopenshell.validate")
detection_handler = ifcopenshell.validate.LogDetectionHandler()
logger.addHandler(detection_handler)

ifcopenshell.validate.validate(model, logger)

if detection_handler.message_logged:
    print("Validation issues detected")
```

### `ifcopenshell.validate.validate_guid(guid_string)`

Validates the format of an IFC GlobalId (22-character base64 encoded string).

| Parameter | Type | Description |
|-----------|------|-------------|
| `guid_string` | `str` | The GlobalId string to validate |

**Returns**: `bool` — `True` if the GUID format is valid.

### `ifcopenshell.validate.validate_ifc_header(f, logger)`

Validates the IFC file header section (FILE_DESCRIPTION, FILE_NAME, FILE_SCHEMA).

| Parameter | Type | Description |
|-----------|------|-------------|
| `f` | `ifcopenshell.file` | The IFC model to validate |
| `logger` | `logging.Logger` | Logger instance for results |

**Returns**: `None`. Results sent to `logger`.

---

## IDS Validation — `ifctester`

### `ifctester.open(ids_path)`

Loads an IDS (Information Delivery Specification) file from disk.

| Parameter | Type | Description |
|-----------|------|-------------|
| `ids_path` | `str` | Path to the `.ids` XML file |

**Returns**: `ifctester.ids.Ids` instance.

### `ifctester.ids.Ids`

Root container for an IDS document.

| Constructor Parameter | Type | Default | Description |
|----------------------|------|---------|-------------|
| `title` | `str` | `""` | IDS document title |
| `description` | `str` | `""` | IDS document description |
| `author` | `str` | `""` | Author identifier |
| `version` | `str` | `""` | IDS version string |

| Attribute | Type | Description |
|-----------|------|-------------|
| `.specifications_` | `list[Specification]` | List of all specifications in this IDS |

| Method | Description |
|--------|-------------|
| `.validate(ifc_file)` | Validate an IFC model against all specifications |
| `.to_xml(filepath)` | Export the IDS to an XML file |

### `ifctester.ids.Specification`

A single requirement specification within an IDS document.

| Constructor Parameter | Type | Default | Description |
|----------------------|------|---------|-------------|
| `name` | `str` | required | Specification name |
| `ifcVersion` | `list[str]` | `[]` | Applicable IFC versions (e.g., `["IFC4"]`) |
| `description` | `str` | `""` | Human-readable description |
| `minOccurs` | `int` | `0` | Minimum required matches. `0` = optional, `>0` = required |
| `maxOccurs` | `str` or `int` | `"unbounded"` | Maximum allowed matches |

| Attribute (post-validation) | Type | Description |
|-----------------------------|------|-------------|
| `.total_applicable` | `int` | Number of IFC entities the specification applied to |
| `.total_pass` | `int` | Number of entities that passed all requirements |
| `.total_fail` | `int` | Number of entities that failed one or more requirements |

### IDS Facet Classes

All facets are in `ifctester.ids`:

| Facet Class | Purpose | Key Parameters |
|-------------|---------|----------------|
| `Entity(name, predefinedType)` | Filter/require IFC class | `name`: IFC class string, `predefinedType`: optional |
| `Attribute(name, value)` | Check entity attribute | `name`: attribute name, `value`: expected value or pattern |
| `Classification(system, value)` | Check classification reference | `system`: classification system name, `value`: code |
| `Property(propertySet, baseName, value, dataType)` | Check property value | `propertySet`: pset name, `baseName`: property name |
| `Material(value)` | Check material assignment | `value`: material name or pattern |
| `PartOf(relation, name)` | Check containment/aggregation | `relation`: relationship type |

---

## Reporter Classes — `ifctester.reporter`

All reporters follow the same interface:

```python
reporter = ifctester.reporter.<ReporterClass>(ids)
reporter.report()
```

| Reporter Class | Output Format | Key Methods |
|---------------|--------------|-------------|
| `Console(ids)` | Colored terminal text | `.report()` |
| `Json(ids)` | JSON string | `.report()`, `.to_string()`, `.to_file(path)` |
| `Html(ids)` | HTML page | `.report()`, `.to_file(path)` |
| `Bcf(ids)` | BCF XML file | `.report()`, `.to_file(path)` |
| `Ods(ids)` | ODS spreadsheet | `.report()`, `.to_file(path)` |
| `Txt(ids)` | Plain text | `.report()`, `.to_file(path)` |

---

## Spatial Hierarchy Validation Rules

### Required Entities by Schema

| Schema | Required Spatial Chain |
|--------|----------------------|
| IFC2X3 | `IfcProject` (1) → `IfcSite` (≥1) → `IfcBuilding` (≥1) |
| IFC4 | `IfcProject` (1) → `IfcSite` (≥1) → `IfcBuilding` (≥1) |
| IFC4X3 | `IfcProject` (1) → `IfcSite` (≥1) → `IfcBuilding` (≥1) OR `IfcFacility` (≥1) |

### Spatial Containment Check

```python
import ifcopenshell.util.element

container = ifcopenshell.util.element.get_container(element)
# Returns: IfcSpatialStructureElement or None
```

---

## Property Set Validation Rules

### Standard Property Sets and Required Properties

| IFC Class | Standard Pset | Required Properties |
|-----------|--------------|---------------------|
| `IfcWall` | `Pset_WallCommon` | `IsExternal`, `LoadBearing`, `FireRating` |
| `IfcSlab` | `Pset_SlabCommon` | `IsExternal`, `LoadBearing` |
| `IfcDoor` | `Pset_DoorCommon` | `IsExternal`, `FireRating` |
| `IfcWindow` | `Pset_WindowCommon` | `IsExternal`, `ThermalTransmittance` |
| `IfcBeam` | `Pset_BeamCommon` | `LoadBearing` |
| `IfcColumn` | `Pset_ColumnCommon` | `LoadBearing` |
| `IfcSpace` | `Pset_SpaceCommon` | `IsExternal`, `GrossPlannedArea` |

### Property Retrieval

```python
import ifcopenshell.util.element

psets = ifcopenshell.util.element.get_psets(element)
# Returns: dict of {pset_name: {prop_name: value, ...}, ...}

# Access a specific property value:
fire_rating = psets.get("Pset_WallCommon", {}).get("FireRating")
```

---

## Classification Validation Rules

### `ifcopenshell.util.classification.get_references(element)`

Retrieves all classification references assigned to an IFC element.

| Parameter | Type | Description |
|-----------|------|-------------|
| `element` | `entity_instance` | The IFC element to query |

**Returns**: List of `IfcClassificationReference` entities (may be empty).

### Classification Reference Identifier Field

| Schema | Identifier Attribute |
|--------|---------------------|
| IFC2X3 | `IfcClassificationReference.ItemReference` |
| IFC4 | `IfcClassificationReference.Identification` |
| IFC4X3 | `IfcClassificationReference.Identification` |

---

## Geometry Validation Rules

### Representation Check

```python
# Check if an element has geometry
if element.Representation is None:
    # Element has no geometry representation

# Check if representation has items
for rep in element.Representation.Representations:
    if not rep.Items or len(rep.Items) == 0:
        # Empty representation context
```

### Placement Check

```python
if element.ObjectPlacement is None:
    # Element has no IfcLocalPlacement
```

---

## Model Access Methods

| Context | Method | Returns |
|---------|--------|---------|
| Inside Bonsai (v0.8.x) | `tool.Ifc.get()` | `ifcopenshell.file` or `None` |
| Inside Bonsai (legacy) | `IfcStore.get_file()` | `ifcopenshell.file` or `None` |
| Standalone script | `ifcopenshell.open(path)` | `ifcopenshell.file` |
| Bonsai UI IDS panel | `BIM > IFC Tester` | GUI-based validation |

### Schema Detection

```python
model = ifcopenshell.open("project.ifc")
schema = model.schema  # "IFC2X3", "IFC4", or "IFC4X3"
```

---

## Severity Classification Rules

| Severity | Trigger Conditions |
|----------|-------------------|
| **BLOCKER** | Schema ERROR; missing IfcProject/IfcSite/IfcBuilding; required IDS spec failed (`minOccurs > 0`) |
| **WARNING** | Schema WARNING; EXPRESS WHERE violation; missing IfcBuildingStorey; element without spatial containment; missing standard pset; missing property; required classification system absent; malformed classification reference; empty representation; optional IDS spec failed |
| **INFO** | Empty property value; missing IfcLocalPlacement; no classification system in model; element unclassified; all checks passed |

---

## CLI Commands

### Schema Validation

```bash
python -m ifcopenshell.validate model.ifc              # Basic validation
python -m ifcopenshell.validate model.ifc --rules       # Include EXPRESS WHERE rules
python -m ifcopenshell.validate model.ifc --json        # JSON output
python -m ifcopenshell.validate model.ifc --fields      # Show attribute field positions
```

### IDS Validation

```bash
python -m ifctester model.ifc requirements.ids                           # Console output
python -m ifctester model.ifc requirements.ids --reporter Html --output report.html  # HTML report
```
