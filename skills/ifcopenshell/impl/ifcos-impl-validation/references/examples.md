# Validation Workflow Code Examples

## Example 1: Complete Schema Validation with Logging

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.validate
import logging

def validate_schema(ifc_path):
    """Run schema validation and return structured results."""
    model = ifcopenshell.open(ifc_path)
    print(f"Schema: {model.schema}, Entities: {len(model)}")

    # Option A: Use json_logger for programmatic access
    json_log = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, json_log)

    warnings = [s for s in json_log.statements if s["level"] == "WARNING"]
    errors = [s for s in json_log.statements if s["level"] == "ERROR"]

    print(f"Warnings: {len(warnings)}, Errors: {len(errors)}")
    for error in errors:
        print(f"  ERROR: {error['message']}")

    return {"warnings": warnings, "errors": errors, "valid": len(errors) == 0}

result = validate_schema("building.ifc")
```

## Example 2: Schema Validation with Pass/Fail Gate

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.validate
import logging
import sys

def validate_or_fail(ifc_path):
    """Validate and exit with code 1 if issues found."""
    model = ifcopenshell.open(ifc_path)

    logger = logging.getLogger("ifcopenshell.validate")
    logger.setLevel(logging.WARNING)
    stream_handler = logging.StreamHandler()
    stream_handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))
    logger.addHandler(stream_handler)

    detection = ifcopenshell.validate.LogDetectionHandler()
    logger.addHandler(detection)

    ifcopenshell.validate.validate(model, logger)

    if detection.message_logged:
        print("VALIDATION FAILED")
        sys.exit(1)
    else:
        print("VALIDATION PASSED")

validate_or_fail("building.ifc")
```

## Example 3: EXPRESS WHERE Rule Validation (Two-Phase)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.validate

def two_phase_validation(ifc_path):
    """Run basic validation first, then EXPRESS rules if basic passes."""
    model = ifcopenshell.open(ifc_path)

    # Phase 1: Basic schema validation (fast)
    basic_log = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, basic_log)

    if basic_log.statements:
        print(f"Phase 1 FAILED: {len(basic_log.statements)} basic issues found")
        print("Fix basic issues before running EXPRESS WHERE rules")
        for s in basic_log.statements[:10]:  # Show first 10
            print(f"  {s['level']}: {s['message']}")
        return False

    print("Phase 1 PASSED: No basic issues")

    # Phase 2: EXPRESS WHERE rules (slow, 10-100x longer)
    print("Running EXPRESS WHERE rules...")
    express_log = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, express_log, express_rules=True)

    if express_log.statements:
        print(f"Phase 2 FAILED: {len(express_log.statements)} WHERE rule violations")
        for s in express_log.statements[:10]:
            print(f"  {s['level']}: {s['message']}")
        return False

    print("Phase 2 PASSED: All EXPRESS WHERE rules satisfied")
    return True

two_phase_validation("building.ifc")
```

## Example 4: GUID Validation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.validate

def validate_all_guids(ifc_path):
    """Check that all GlobalIds in the model are valid."""
    model = ifcopenshell.open(ifc_path)
    invalid_guids = []

    for entity in model.by_type("IfcRoot"):
        guid = entity.GlobalId
        error = ifcopenshell.validate.validate_guid(guid)
        if error is not None:
            invalid_guids.append({
                "entity": f"#{entity.id()} {entity.is_a()}",
                "guid": guid,
                "error": error
            })

    if invalid_guids:
        print(f"Found {len(invalid_guids)} invalid GUIDs:")
        for item in invalid_guids:
            print(f"  {item['entity']}: '{item['guid']}' — {item['error']}")
    else:
        print("All GUIDs are valid")

    return invalid_guids

validate_all_guids("building.ifc")
```

## Example 5: IDS Validation — Load and Validate

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifctester
import ifctester.reporter

def validate_against_ids(ifc_path, ids_path):
    """Validate IFC model against IDS specification."""
    # Load IDS specification
    ids = ifctester.open(ids_path)

    # Load IFC model
    ifc_file = ifcopenshell.open(ifc_path)

    # Run validation
    ids.validate(ifc_file)

    # Console report
    console = ifctester.reporter.Console(ids)
    console.report()

    # Check overall results
    for spec in ids.specifications_:
        status = "PASS" if spec.status else "FAIL"
        print(f"  [{status}] {spec.name}")
        if not spec.status:
            print(f"    Failed entities: {len(spec.failed_entities_)}")
            print(f"    Passed entities: {len(spec.passed_entities_)}")

validate_against_ids("building.ifc", "requirements.ids")
```

## Example 6: IDS Validation with Multiple Report Formats

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifctester
import ifctester.reporter

def validate_and_report(ifc_path, ids_path, output_dir="."):
    """Validate and generate reports in multiple formats."""
    ids = ifctester.open(ids_path)
    ifc_file = ifcopenshell.open(ifc_path)
    ids.validate(ifc_file)

    # HTML report for stakeholders
    html = ifctester.reporter.Html(ids)
    html.report()
    html.to_file(f"{output_dir}/validation_report.html")

    # JSON report for CI/CD
    json_rep = ifctester.reporter.Json(ids)
    json_rep.report()
    json_rep.to_file(f"{output_dir}/validation_report.json")

    # BCF report for BIM coordination
    bcf = ifctester.reporter.Bcf(ids)
    bcf.report()
    bcf.to_file(f"{output_dir}/validation_issues.bcf")

    # ODS report for data review
    ods = ifctester.reporter.Ods(ids, excel_safe=True)
    ods.report()
    ods.to_file(f"{output_dir}/validation_report.ods")

    print(f"Reports generated in {output_dir}/")

validate_and_report("building.ifc", "requirements.ids", "./reports")
```

## Example 7: Create IDS Specification Programmatically

```python
# IfcOpenShell — all schema versions
import ifctester.ids

# Create IDS document
ids = ifctester.ids.Ids(
    title="Structural Review Requirements",
    description="Minimum information requirements for structural BIM review",
    author="QA Department",
    version="2.0",
    purpose="Design review",
    milestone="LOD 300"
)

# Specification 1: All walls must have fire rating
spec1 = ifctester.ids.Specification(
    name="Wall Fire Rating",
    ifcVersion=["IFC4", "IFC4X3"],
    description="All load-bearing walls must declare a fire rating property"
)
ids.specifications_.append(spec1)

# Specification 2: All spaces must have names
spec2 = ifctester.ids.Specification(
    name="Space Naming",
    ifcVersion=["IFC2X3", "IFC4", "IFC4X3"],
    description="All spaces must have a meaningful Name attribute"
)
ids.specifications_.append(spec2)

# Export to IDS XML file
ids.to_xml("structural_review.ids")
print(f"Created IDS with {len(ids.specifications_)} specifications")

# Also get as string
xml_str = ids.to_string()
```

## Example 8: Georeferencing Setup and Validation (IFC4)

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
from math import cos, sin, radians

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Example Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Set up georeferencing
ifcopenshell.api.run("georeference.add_georeferencing", model)

ifcopenshell.api.run("georeference.edit_georeferencing", model,
    projected_crs={"Name": "EPSG:28992"},   # Amersfoort / RD New
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": cos(radians(-5.0)),
        "XAxisOrdinate": sin(radians(-5.0)),
        "Scale": 1.0
    })

ifcopenshell.api.run("georeference.edit_true_north", model,
    true_north=5.0)

# Validate georeferencing
map_conv = model.by_type("IfcMapConversion")
proj_crs = model.by_type("IfcProjectedCRS")
assert len(map_conv) == 1, "Expected exactly one IfcMapConversion"
assert len(proj_crs) == 1, "Expected exactly one IfcProjectedCRS"
assert proj_crs[0].Name == "EPSG:28992", "CRS name mismatch"
print("Georeferencing validated successfully")
```

## Example 9: Georeferencing Validation Across Schema Versions

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

def validate_georeferencing(ifc_path):
    """Validate georeferencing based on schema version."""
    model = ifcopenshell.open(ifc_path)
    schema = model.schema
    errors = []

    if schema == "IFC2X3":
        # IFC2X3: Check property sets on IfcProject
        projects = model.by_type("IfcProject")
        if not projects:
            errors.append("No IfcProject found")
            return errors

        psets = ifcopenshell.util.element.get_psets(projects[0])
        if "ePSet_MapConversion" not in psets:
            errors.append("Missing ePSet_MapConversion property set on IfcProject")
        if "ePSet_ProjectedCRS" not in psets:
            errors.append("Missing ePSet_ProjectedCRS property set on IfcProject")

        if "ePSet_MapConversion" in psets:
            mc = psets["ePSet_MapConversion"]
            required_keys = ["Eastings", "Northings", "OrthogonalHeight"]
            for key in required_keys:
                if key not in mc:
                    errors.append(f"ePSet_MapConversion missing '{key}'")

    elif schema in ("IFC4", "IFC4X3"):
        # IFC4/IFC4X3: Check dedicated entities
        map_conversions = model.by_type("IfcMapConversion")
        projected_crs = model.by_type("IfcProjectedCRS")

        if schema == "IFC4X3":
            scaled = model.by_type("IfcMapConversionScaled")
            map_conversions = list(map_conversions) + list(scaled)

        if not map_conversions:
            errors.append("No IfcMapConversion entity found")
        elif len(map_conversions) > 1:
            errors.append(f"Multiple IfcMapConversion entities ({len(map_conversions)})")

        if not projected_crs:
            errors.append("No IfcProjectedCRS entity found")
        else:
            crs = projected_crs[0]
            if not crs.Name or not crs.Name.startswith("EPSG:"):
                errors.append(f"IfcProjectedCRS.Name should use EPSG format, got: '{crs.Name}'")

        # Check for IFC4X3-only entities in IFC4
        if schema == "IFC4":
            scaled = model.by_type("IfcMapConversionScaled")
            if scaled:
                errors.append("IfcMapConversionScaled found in IFC4 model (only valid in IFC4X3)")

    if errors:
        print(f"Georeferencing validation FAILED ({len(errors)} issues):")
        for error in errors:
            print(f"  - {error}")
    else:
        print("Georeferencing validation PASSED")

    return errors

validate_georeferencing("building.ifc")
```

## Example 10: Property Set Completeness Validation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

def validate_pset_completeness(ifc_path, rules):
    """
    Validate property set completeness based on rules.

    Args:
        ifc_path: Path to IFC file
        rules: Dict mapping IFC class to required {pset_name: [property_names]}
    """
    model = ifcopenshell.open(ifc_path)
    all_errors = []

    for ifc_class, required_psets in rules.items():
        elements = model.by_type(ifc_class)
        for element in elements:
            psets = ifcopenshell.util.element.get_psets(element)
            for pset_name, required_props in required_psets.items():
                if pset_name not in psets:
                    all_errors.append({
                        "entity": f"#{element.id()} {element.is_a()} '{element.Name}'",
                        "issue": f"Missing property set: {pset_name}"
                    })
                    continue
                for prop_name in required_props:
                    if prop_name not in psets[pset_name]:
                        all_errors.append({
                            "entity": f"#{element.id()} {element.is_a()} '{element.Name}'",
                            "issue": f"Missing property: {pset_name}.{prop_name}"
                        })
                    elif psets[pset_name][prop_name] is None:
                        all_errors.append({
                            "entity": f"#{element.id()} {element.is_a()} '{element.Name}'",
                            "issue": f"Null value: {pset_name}.{prop_name}"
                        })

    return all_errors

# Define validation rules
rules = {
    "IfcWall": {
        "Pset_WallCommon": ["IsExternal", "LoadBearing", "FireRating"],
    },
    "IfcSlab": {
        "Pset_SlabCommon": ["IsExternal", "LoadBearing"],
    },
    "IfcDoor": {
        "Pset_DoorCommon": ["IsExternal", "FireRating"],
    },
}

errors = validate_pset_completeness("building.ifc", rules)
if errors:
    print(f"Found {len(errors)} property completeness issues:")
    for e in errors:
        print(f"  {e['entity']}: {e['issue']}")
```

## Example 11: Spatial Containment Validation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

def validate_spatial_structure(ifc_path):
    """Validate that spatial hierarchy and containment are correct."""
    model = ifcopenshell.open(ifc_path)
    errors = []

    # Check 1: IfcProject exists
    projects = model.by_type("IfcProject")
    if len(projects) != 1:
        errors.append(f"Expected 1 IfcProject, found {len(projects)}")
        return errors

    # Check 2: All physical elements are spatially contained
    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            errors.append(
                f"#{element.id()} {element.is_a()} '{element.Name}': "
                f"Not spatially contained"
            )

    # Check 3: All IfcBuildingStorey are inside an IfcBuilding
    for storey in model.by_type("IfcBuildingStorey"):
        parent = ifcopenshell.util.element.get_aggregate(storey)
        if parent is None or not parent.is_a("IfcBuilding"):
            errors.append(
                f"#{storey.id()} IfcBuildingStorey '{storey.Name}': "
                f"Not aggregated in an IfcBuilding"
            )

    # Check 4: All IfcBuilding are inside an IfcSite
    for building in model.by_type("IfcBuilding"):
        parent = ifcopenshell.util.element.get_aggregate(building)
        if parent is None or not parent.is_a("IfcSite"):
            errors.append(
                f"#{building.id()} IfcBuilding '{building.Name}': "
                f"Not aggregated in an IfcSite"
            )

    return errors

errors = validate_spatial_structure("building.ifc")
if errors:
    print(f"Spatial structure issues ({len(errors)}):")
    for e in errors:
        print(f"  {e}")
else:
    print("Spatial structure is valid")
```

## Example 12: Full QA Pipeline with Combined Reporting

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.validate
import ifcopenshell.util.element
import ifctester
import ifctester.reporter
import json
import sys

def run_full_qa_pipeline(ifc_path, ids_path=None, output_dir="."):
    """Complete QA pipeline: schema + IDS + custom checks."""
    report = {
        "file": ifc_path,
        "schema_validation": {"status": "not_run", "issues": []},
        "ids_validation": {"status": "not_run", "specs": []},
        "custom_checks": {"status": "not_run", "issues": []},
        "overall": "unknown"
    }

    model = ifcopenshell.open(ifc_path)

    # --- Phase 1: Schema Validation ---
    json_log = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, json_log)

    report["schema_validation"]["issues"] = [
        {"level": s["level"], "message": s["message"]}
        for s in json_log.statements
    ]
    report["schema_validation"]["status"] = (
        "pass" if not json_log.statements else "fail"
    )

    # --- Phase 2: IDS Validation ---
    if ids_path:
        ids = ifctester.open(ids_path)
        ids.validate(model)

        for spec in ids.specifications_:
            report["ids_validation"]["specs"].append({
                "name": spec.name,
                "status": "pass" if spec.status else "fail",
                "applicable": len(spec.applicable_entities_),
                "passed": len(spec.passed_entities_),
                "failed": len(spec.failed_entities_)
            })

        all_passed = all(s["status"] == "pass" for s in report["ids_validation"]["specs"])
        report["ids_validation"]["status"] = "pass" if all_passed else "fail"

        # Generate HTML report
        html = ifctester.reporter.Html(ids)
        html.report()
        html.to_file(f"{output_dir}/ids_report.html")

    # --- Phase 3: Custom Checks ---
    custom_issues = []

    # Check: All elements spatially contained
    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            custom_issues.append(
                f"#{element.id()} {element.is_a()}: No spatial container"
            )

    # Check: No duplicate GUIDs
    guids = {}
    for entity in model.by_type("IfcRoot"):
        guid = entity.GlobalId
        if guid in guids:
            custom_issues.append(
                f"Duplicate GUID '{guid}': #{guids[guid]} and #{entity.id()}"
            )
        guids[guid] = entity.id()

    report["custom_checks"]["issues"] = custom_issues
    report["custom_checks"]["status"] = "pass" if not custom_issues else "fail"

    # --- Overall Status ---
    statuses = [
        report["schema_validation"]["status"],
        report["ids_validation"]["status"],
        report["custom_checks"]["status"],
    ]
    report["overall"] = "pass" if all(s in ("pass", "not_run") for s in statuses) else "fail"

    # Write JSON report
    with open(f"{output_dir}/qa_report.json", "w") as f:
        json.dump(report, f, indent=2)

    print(f"QA Pipeline: {report['overall'].upper()}")
    print(f"  Schema: {report['schema_validation']['status']} ({len(report['schema_validation']['issues'])} issues)")
    if ids_path:
        print(f"  IDS: {report['ids_validation']['status']} ({len(report['ids_validation']['specs'])} specs)")
    print(f"  Custom: {report['custom_checks']['status']} ({len(report['custom_checks']['issues'])} issues)")

    return report

# Usage
report = run_full_qa_pipeline("building.ifc", "requirements.ids", "./qa_output")
if report["overall"] != "pass":
    sys.exit(1)
```

## Example 13: Command-Line Validation

```bash
# Basic schema validation
python -m ifcopenshell.validate model.ifc

# Schema validation with EXPRESS WHERE rules
python -m ifcopenshell.validate model.ifc --rules

# Schema validation with JSON output
python -m ifcopenshell.validate model.ifc --json

# IDS validation with console output
python -m ifctester model.ifc requirements.ids

# IDS validation with HTML report
python -m ifctester model.ifc requirements.ids --reporter Html --output report.html
```
