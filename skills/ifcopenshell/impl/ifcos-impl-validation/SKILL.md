---
name: ifcos-impl-validation
description: >
  Use when validating IFC files for schema compliance, IDS conformance, or custom quality
  rules. Prevents the common mistake of only checking schema validity without verifying
  property set completeness or spatial hierarchy correctness. Covers ifcopenshell.validate,
  ifctester for IDS validation, georeference validation, and custom validation pipelines.
  Keywords: validation, IDS, ifctester, ifcopenshell.validate, schema compliance, quality assurance, property set check, spatial hierarchy check, IFC validation, check model quality, find missing properties.
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Validation Workflows

## Quick Reference

### Critical Warnings

- **ALWAYS** run `ifcopenshell.validate.validate()` with basic mode first (default `express_rules=False`). NEVER enable EXPRESS WHERE rules on a first pass — they are 10-100x slower on large models.
- **ALWAYS** pass a configured `logging.Logger` instance to `validate()`. The function logs results; it does NOT return them.
- **NEVER** assume a model is valid because `validate()` completes without raising an exception. Validation issues are logged, not raised. Use `LogDetectionHandler` or `json_logger` to detect issues programmatically.
- **ALWAYS** use `ifctester` for IDS (Information Delivery Specification) validation. `ifcopenshell.validate` checks schema compliance only — it does NOT check project-specific requirements.
- **NEVER** confuse schema validation with IDS validation. Schema validation verifies the IFC file structure is correct per EXPRESS schema. IDS validation verifies the IFC data meets project information requirements.
- **ALWAYS** call `add_georeferencing` before `edit_georeferencing`. The IfcMapConversion and IfcProjectedCRS entities MUST exist before editing.
- **NEVER** use `IfcMapConversionScaled` in IFC4 models. The scaled variant is only available in IFC4X3.
- **ALWAYS** check `model.schema` before applying version-specific validation logic. Georeferencing differs between IFC2X3 (property sets) and IFC4+ (dedicated entities).

### Decision Tree: Which Validation Approach

```
What are you validating?
├── IFC file structure and schema compliance?
│   ├── Basic type/attribute checking?
│   │   └── ifcopenshell.validate.validate(model, logger)
│   ├── Full EXPRESS WHERE rules?
│   │   └── ifcopenshell.validate.validate(model, logger, express_rules=True)
│   ├── GUID format only?
│   │   └── ifcopenshell.validate.validate_guid(guid_string)
│   └── File header only?
│       └── ifcopenshell.validate.validate_ifc_header(model, logger)
│
├── Project information requirements (IDS)?
│   ├── Load IDS specification?
│   │   └── ifctester.open("spec.ids")
│   ├── Validate IFC against IDS?
│   │   └── ids.validate(ifc_file)
│   └── Generate validation report?
│       ├── Console → ifctester.reporter.Console(ids)
│       ├── JSON → ifctester.reporter.Json(ids)
│       ├── HTML → ifctester.reporter.Html(ids)
│       ├── BCF → ifctester.reporter.Bcf(ids)
│       └── ODS → ifctester.reporter.Ods(ids)
│
├── Georeferencing correctness?
│   ├── IFC2X3 → Check property sets on IfcProject
│   ├── IFC4 → Check IfcMapConversion + IfcProjectedCRS entities
│   └── IFC4X3 → Check IfcMapConversion or IfcMapConversionScaled
│
└── Custom business rules?
    └── Write Python functions using ifcopenshell entity traversal
        (see Custom Validation Rules section)
```

---

## Essential Patterns

### Pattern 1: Basic Schema Validation

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.validate
import logging

model = ifcopenshell.open("building.ifc")

# Set up logger to capture validation output
logger = logging.getLogger("ifcopenshell.validate")
logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))
logger.addHandler(handler)

# Run basic validation (no EXPRESS WHERE rules)
ifcopenshell.validate.validate(model, logger)
```

### Pattern 2: Programmatic Validation with json_logger

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("building.ifc")

# Use json_logger for structured, programmatic access to results
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

# Iterate validation results
for statement in json_log.statements:
    print(f"Level: {statement['level']}, Message: {statement['message']}")

# Check if any issues were found
if json_log.statements:
    print(f"Total issues: {len(json_log.statements)}")
else:
    print("Model is valid")
```

### Pattern 3: Detect Validation Issues (Pass/Fail Gate)

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.validate
import logging

model = ifcopenshell.open("building.ifc")

logger = logging.getLogger("ifcopenshell.validate")
logger.setLevel(logging.WARNING)

# Add detection handler to check if ANY issues exist
detection_handler = ifcopenshell.validate.LogDetectionHandler()
logger.addHandler(detection_handler)

ifcopenshell.validate.validate(model, logger)

if detection_handler.message_logged:
    print("FAIL: Validation issues found")
else:
    print("PASS: Model is valid")
```

### Pattern 4: IDS Validation with ifctester

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifctester
import ifctester.ids
import ifctester.reporter

# Step 1: Load the IDS specification
ids = ifctester.open("requirements.ids")

# Step 2: Load the IFC file
ifc_file = ifcopenshell.open("building.ifc")

# Step 3: Validate IFC against IDS
ids.validate(ifc_file)

# Step 4: Report results
reporter = ifctester.reporter.Console(ids)
reporter.report()

# Or generate HTML report
html_reporter = ifctester.reporter.Html(ids)
html_reporter.report()
html_reporter.to_file("validation_report.html")
```

### Pattern 5: Georeferencing Validation

```python
# IfcOpenShell: IFC4 / IFC4X3
import ifcopenshell

model = ifcopenshell.open("building.ifc")

# Check schema version for georeferencing method
if model.schema == "IFC2X3":
    # IFC2X3: Georeferencing stored as property sets on IfcProject
    project = model.by_type("IfcProject")[0]
    # Check for ePSet_MapConversion and ePSet_ProjectedCRS property sets
    psets = ifcopenshell.util.element.get_psets(project)
    has_georef = "ePSet_MapConversion" in psets
else:
    # IFC4 / IFC4X3: Dedicated entities
    map_conversions = model.by_type("IfcMapConversion")
    projected_crs = model.by_type("IfcProjectedCRS")
    has_georef = len(map_conversions) > 0 and len(projected_crs) > 0

    if model.schema == "IFC4X3":
        # Also check for IfcMapConversionScaled (IFC4X3 only)
        scaled = model.by_type("IfcMapConversionScaled")
        has_georef = has_georef or len(scaled) > 0

if has_georef:
    print("Georeferencing is present")
else:
    print("WARNING: No georeferencing found")
```

---

## IDS Validation (Information Delivery Specification)

### What is IDS?

IDS is a buildingSMART standard (ISO 7817-3) that defines machine-readable information requirements for IFC models. An IDS file specifies:

- **Applicability**: Which IFC entities a requirement applies to (e.g., all IfcWall instances)
- **Requirements**: What data those entities must contain (e.g., must have a FireRating property)

### IDS Facet Types

| Facet | Purpose | Example |
|-------|---------|---------|
| `Entity` | Filter by IFC class and predefined type | All IfcWall with PredefinedType=SOLIDWALL |
| `Attribute` | Check entity attribute values | Name must match pattern "W-*" |
| `Classification` | Check classification references | Must have Uniclass 2015 reference |
| `Property` | Check property set values | Pset_WallCommon.FireRating must exist |
| `Material` | Check material assignments | Must have material assigned |
| `PartOf` | Check spatial/aggregation relationships | Must be contained in IfcBuildingStorey |

### Creating IDS Programmatically

```python
# IfcOpenShell: all schema versions
import ifctester.ids

# Create new IDS document
ids = ifctester.ids.Ids(
    title="Project Requirements",
    description="Minimum information requirements for structural review",
    author="QA Team",
    version="1.0"
)

# Create a specification: All walls must have fire rating
spec = ifctester.ids.Specification(
    name="Wall Fire Rating Required",
    ifcVersion=["IFC4"],
    description="All load-bearing walls must declare fire rating"
)

# Add to IDS
ids.specifications_.append(spec)

# Export to IDS XML
ids.to_xml("project_requirements.ids")
```

### IDS Reporter Types

| Reporter | Output | Use Case |
|----------|--------|----------|
| `Console` | Terminal text with color | Quick interactive checks |
| `Json` | Structured JSON data | CI/CD pipelines, API integration |
| `Html` | Formatted HTML page | Stakeholder reports |
| `Bcf` | BCF issue file | BIM coordination (links to model elements) |
| `Ods` | Spreadsheet (ODS) | Data analysis, Excel-compatible review |
| `Txt` | Plain text | Log files, archival |

---

## Georeferencing: IFC2X3 vs IFC4+ Differences

### Version Comparison

| Feature | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| Map conversion | Property set on IfcProject | `IfcMapConversion` entity | `IfcMapConversion` or `IfcMapConversionScaled` |
| CRS definition | Property set | `IfcProjectedCRS` entity | `IfcProjectedCRS` entity |
| MapUnit attribute | String (full unit name) | `IfcNamedUnit` object | `IfcNamedUnit` object |
| Removal method | Remove property sets | Remove entities | Remove entities |
| API support | Manual property sets | `georeference.*` API | `georeference.*` API |

### Georeferencing API (IFC4+)

| Function | Purpose |
|----------|---------|
| `georeference.add_georeferencing` | Create empty IfcMapConversion + IfcProjectedCRS |
| `georeference.edit_georeferencing` | Set coordinate operation and CRS parameters |
| `georeference.edit_true_north` | Set true north orientation (degrees or vector) |
| `georeference.edit_wcs` | Adjust world coordinate system origin |
| `georeference.remove_georeferencing` | Remove all georeferencing data |

### Georeferencing Anti-Patterns

- **NEVER** set georeferencing without consulting the project surveyor. Incorrect Eastings/Northings place the building at the wrong location on Earth.
- **NEVER** confuse Project North with True North. `XAxisAbscissa`/`XAxisOrdinate` in the coordinate operation define **Project North** rotation. True North is set separately via `edit_true_north`.
- **NEVER** use `IfcMapConversionScaled` in IFC4 models. It exists only in IFC4X3.
- **ALWAYS** use EPSG codes in the format `"EPSG:XXXXX"` for the CRS Name attribute.
- **NEVER** move the WCS from origin (0,0,0) without a specific surveying reason. This affects all geometry in the model.

---

## Custom Validation Rules

### Pattern: Property Set Completeness Check

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.element

def validate_required_psets(model, ifc_class, required_psets):
    """Validate that all entities of a type have required property sets."""
    errors = []
    elements = model.by_type(ifc_class)
    for element in elements:
        psets = ifcopenshell.util.element.get_psets(element)
        for pset_name, required_props in required_psets.items():
            if pset_name not in psets:
                errors.append(f"#{element.id()} {element.Name}: Missing {pset_name}")
                continue
            for prop in required_props:
                if prop not in psets[pset_name]:
                    errors.append(
                        f"#{element.id()} {element.Name}: "
                        f"Missing {pset_name}.{prop}"
                    )
    return errors

# Usage
model = ifcopenshell.open("building.ifc")
errors = validate_required_psets(model, "IfcWall", {
    "Pset_WallCommon": ["IsExternal", "LoadBearing", "FireRating"],
})
for error in errors:
    print(f"ERROR: {error}")
```

### Pattern: Spatial Structure Validation

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.element

def validate_spatial_containment(model):
    """Check that all physical elements are spatially contained."""
    errors = []
    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            errors.append(
                f"#{element.id()} {element.is_a()} '{element.Name}': "
                f"Not contained in any spatial element"
            )
    return errors
```

---

## Validation Pipeline Pattern

### Automated QA Pipeline

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.validate
import ifctester
import ifctester.reporter
import logging
import sys

def run_validation_pipeline(ifc_path, ids_path=None):
    """Run complete validation pipeline: schema + IDS + custom rules."""
    results = {"schema": None, "ids": None, "custom": []}

    # Phase 1: Schema validation
    model = ifcopenshell.open(ifc_path)
    json_log = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, json_log)
    results["schema"] = {
        "passed": len(json_log.statements) == 0,
        "issues": len(json_log.statements),
        "details": json_log.statements
    }

    # Phase 2: IDS validation (if specification provided)
    if ids_path:
        ids = ifctester.open(ids_path)
        ids.validate(model)
        json_reporter = ifctester.reporter.Json(ids)
        json_reporter.report()
        results["ids"] = json_reporter.to_string()

    # Phase 3: Custom rules
    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            results["custom"].append(
                f"#{element.id()} {element.is_a()}: No spatial container"
            )

    return results

# Usage
results = run_validation_pipeline("building.ifc", "requirements.ids")
if not results["schema"]["passed"]:
    print(f"Schema issues: {results['schema']['issues']}")
    sys.exit(1)
```

---

## Command-Line Validation

### ifcopenshell.validate CLI

```bash
# Basic validation
python -m ifcopenshell.validate model.ifc

# With EXPRESS WHERE rules
python -m ifcopenshell.validate model.ifc --rules

# JSON output
python -m ifcopenshell.validate model.ifc --json

# Show attribute field positions in error messages
python -m ifcopenshell.validate model.ifc --fields
```

### ifctester CLI

```bash
# Validate IFC against IDS
python -m ifctester model.ifc requirements.ids

# Generate HTML report
python -m ifctester model.ifc requirements.ids --reporter Html --output report.html
```

---

## Dependencies

- **ifcos-syntax-api** — For `ifcopenshell.api.run()` invocation patterns and `georeference.*` API functions
- **ifcos-syntax-fileio** — For `ifcopenshell.open()`, `ifcopenshell.file()`, and file I/O patterns

---

## Reference Links

- [Validation API Signatures](references/methods.md) — Complete function signatures for validate, ifctester, and georeference modules
- [Working Code Examples](references/examples.md) — End-to-end validation workflow examples
- [Anti-Patterns](references/anti-patterns.md) — Common validation mistakes and how to avoid them
