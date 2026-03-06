# IFC Validation Workflow Examples

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Python 3.11
> **IFC Schemas**: IFC2X3, IFC4, IFC4X3

---

## Example 1: Complete Six-Phase Validation Pipeline

Executes all six validation phases in the prescribed order: Schema → Spatial → Properties → Geometry → Classification → IDS.

### Before (unvalidated model, unknown quality)

```python
import ifcopenshell

model = ifcopenshell.open("project.ifc")
# No validation performed. Model quality unknown.
# Handing this off to a contractor is risky.
```

### After (full validation pipeline with structured report)

```python
import ifcopenshell
import ifcopenshell.validate
import ifcopenshell.util.element
import ifcopenshell.util.classification
import ifctester
import ifctester.reporter
import json

def run_full_validation(ifc_path, ids_path=None, requirements=None):
    """
    Six-phase IFC validation pipeline.
    Returns structured report with severity-classified findings.
    """
    report = {
        "file": ifc_path,
        "schema": None,
        "phases": [],
        "summary": {"BLOCKER": 0, "WARNING": 0, "INFO": 0}
    }

    # Load model
    model = ifcopenshell.open(ifc_path)
    report["schema"] = model.schema

    # --- Phase 1: Schema Validation ---
    phase1 = {"name": "Schema Validation", "findings": []}
    json_log = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, json_log)

    for stmt in json_log.statements:
        severity = "BLOCKER" if stmt["level"] == "ERROR" else "WARNING"
        phase1["findings"].append({"severity": severity, "message": stmt["message"]})
        report["summary"][severity] += 1

    # If schema has BLOCKERs, stop here
    report["phases"].append(phase1)
    blocker_count = sum(1 for f in phase1["findings"] if f["severity"] == "BLOCKER")
    if blocker_count > 0:
        report["phases"].append({"name": "Remaining phases skipped", "findings": [
            {"severity": "BLOCKER", "message": f"{blocker_count} schema errors found. Fix before proceeding."}
        ]})
        return report

    # --- Phase 2: Spatial Hierarchy ---
    phase2 = {"name": "Spatial Hierarchy", "findings": []}

    projects = model.by_type("IfcProject")
    if len(projects) != 1:
        phase2["findings"].append({"severity": "BLOCKER",
            "message": f"Expected 1 IfcProject, found {len(projects)}"})
        report["summary"]["BLOCKER"] += 1

    sites = model.by_type("IfcSite")
    if len(sites) == 0:
        phase2["findings"].append({"severity": "BLOCKER", "message": "No IfcSite found"})
        report["summary"]["BLOCKER"] += 1

    buildings = model.by_type("IfcBuilding")
    if model.schema == "IFC4X3":
        facilities = model.by_type("IfcFacility")
        if len(buildings) == 0 and len(facilities) == 0:
            phase2["findings"].append({"severity": "BLOCKER",
                "message": "No IfcBuilding or IfcFacility found"})
            report["summary"]["BLOCKER"] += 1
    else:
        if len(buildings) == 0:
            phase2["findings"].append({"severity": "BLOCKER", "message": "No IfcBuilding found"})
            report["summary"]["BLOCKER"] += 1

    storeys = model.by_type("IfcBuildingStorey")
    if len(storeys) == 0 and len(buildings) > 0:
        phase2["findings"].append({"severity": "WARNING", "message": "No IfcBuildingStorey found"})
        report["summary"]["WARNING"] += 1

    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            phase2["findings"].append({"severity": "WARNING",
                "message": f"#{element.id()} {element.is_a()} '{element.Name}': Not spatially contained"})
            report["summary"]["WARNING"] += 1

    report["phases"].append(phase2)

    # --- Phase 3: Property Set Compliance ---
    phase3 = {"name": "Property Compliance", "findings": []}

    if requirements is None:
        requirements = {
            "IfcWall": {"Pset_WallCommon": ["IsExternal", "LoadBearing", "FireRating"]},
            "IfcSlab": {"Pset_SlabCommon": ["IsExternal", "LoadBearing"]},
            "IfcDoor": {"Pset_DoorCommon": ["IsExternal", "FireRating"]},
            "IfcWindow": {"Pset_WindowCommon": ["IsExternal", "ThermalTransmittance"]},
        }

    for ifc_class, pset_reqs in requirements.items():
        for element in model.by_type(ifc_class):
            psets = ifcopenshell.util.element.get_psets(element)
            for pset_name, props in pset_reqs.items():
                if pset_name not in psets:
                    phase3["findings"].append({"severity": "WARNING",
                        "message": f"#{element.id()} {element.Name}: Missing pset '{pset_name}'"})
                    report["summary"]["WARNING"] += 1
                    continue
                for prop in props:
                    if psets[pset_name].get(prop) is None:
                        phase3["findings"].append({"severity": "WARNING",
                            "message": f"#{element.id()} {element.Name}: Missing {pset_name}.{prop}"})
                        report["summary"]["WARNING"] += 1

    report["phases"].append(phase3)

    # --- Phase 4: Geometry Validation ---
    phase4 = {"name": "Geometry Validation", "findings": []}

    for element in model.by_type("IfcElement"):
        if not element.Representation:
            phase4["findings"].append({"severity": "WARNING",
                "message": f"#{element.id()} {element.is_a()} '{element.Name}': No geometry"})
            report["summary"]["WARNING"] += 1
            continue
        for rep in element.Representation.Representations:
            if not rep.Items or len(rep.Items) == 0:
                phase4["findings"].append({"severity": "WARNING",
                    "message": f"#{element.id()} {element.Name}: Empty representation '{rep.RepresentationIdentifier}'"})
                report["summary"]["WARNING"] += 1

    report["phases"].append(phase4)

    # --- Phase 5: Classification Validation ---
    phase5 = {"name": "Classification Validation", "findings": []}

    systems = model.by_type("IfcClassification")
    if len(systems) == 0:
        phase5["findings"].append({"severity": "INFO", "message": "No classification system in model"})
        report["summary"]["INFO"] += 1
    else:
        for ref in model.by_type("IfcClassificationReference"):
            if not ref.Identification and not getattr(ref, "ItemReference", None):
                phase5["findings"].append({"severity": "WARNING",
                    "message": f"#{ref.id()} IfcClassificationReference: Missing Identification"})
                report["summary"]["WARNING"] += 1

    report["phases"].append(phase5)

    # --- Phase 6: IDS Validation ---
    phase6 = {"name": "IDS Validation", "findings": []}

    if ids_path:
        ids = ifctester.open(ids_path)
        ids.validate(model)
        for spec in ids.specifications:
            if spec.total_fail > 0:
                severity = "BLOCKER" if spec.minOccurs > 0 else "WARNING"
                phase6["findings"].append({"severity": severity,
                    "message": f"IDS '{spec.name}': {spec.total_fail}/{spec.total_applicable} elements failed"})
                report["summary"][severity] += 1
    else:
        phase6["findings"].append({"severity": "INFO", "message": "No IDS specification provided, skipping"})
        report["summary"]["INFO"] += 1

    report["phases"].append(phase6)

    return report


# --- Run the pipeline ---
report = run_full_validation("project.ifc", ids_path="requirements.ids")

print(f"Schema: {report['schema']}")
print(f"Summary: {report['summary']}")
for phase in report["phases"]:
    print(f"\n--- {phase['name']} ---")
    for finding in phase["findings"]:
        print(f"  [{finding['severity']}] {finding['message']}")
```

---

## Example 2: Schema Validation with Pass/Fail Gate

Use in CI/CD pipelines to block delivery of schema-invalid IFC files.

### Before (no validation gate)

```python
# CI pipeline just checks if the file exists
import os
if os.path.exists("deliverable.ifc"):
    print("File present. Shipping.")
# Result: Broken IFC files reach the client.
```

### After (schema validation gate)

```python
import sys
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("deliverable.ifc")

# Phase 1: Basic schema validation
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

errors = [s for s in json_log.statements if s["level"] == "ERROR"]
warnings = [s for s in json_log.statements if s["level"] == "WARNING"]

if errors:
    print(f"BLOCKED: {len(errors)} schema error(s) found")
    for e in errors:
        print(f"  ERROR: {e['message']}")
    sys.exit(1)

# Phase 2: EXPRESS WHERE rules (only if basic passes)
json_log_express = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log_express, express_rules=True)

express_issues = json_log_express.statements
if express_issues:
    print(f"WARNING: {len(express_issues)} EXPRESS rule violation(s)")
    for w in express_issues:
        print(f"  {w['level']}: {w['message']}")

print(f"PASSED: Schema validation OK ({len(warnings)} warning(s))")
sys.exit(0)
```

---

## Example 3: IDS Validation with Multiple Report Formats

Generate IDS validation reports for different audiences.

### Before (manual requirement checking)

```python
# Manually checking each wall for fire rating
model = ifcopenshell.open("project.ifc")
for wall in model.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    if "Pset_WallCommon" in psets:
        if "FireRating" in psets["Pset_WallCommon"]:
            print(f"OK: {wall.Name}")
        else:
            print(f"MISSING: {wall.Name} lacks FireRating")
# Result: Only one requirement checked. Not scalable. Not standardized.
```

### After (IDS-based validation with multi-format reporting)

```python
import ifcopenshell
import ifctester
import ifctester.reporter

# Load IDS specification and IFC model
ids = ifctester.open("project_requirements.ids")
model = ifcopenshell.open("project.ifc")

# Run validation
ids.validate(model)

# Console output for developer
console_reporter = ifctester.reporter.Console(ids)
console_reporter.report()

# HTML report for project manager
html_reporter = ifctester.reporter.Html(ids)
html_reporter.report()
html_reporter.to_file("validation_report.html")

# JSON report for CI/CD pipeline
json_reporter = ifctester.reporter.Json(ids)
json_reporter.report()
json_reporter.to_file("validation_results.json")

# BCF report for BIM coordination
bcf_reporter = ifctester.reporter.Bcf(ids)
bcf_reporter.report()
bcf_reporter.to_file("validation_issues.bcf")

# Summary
for spec in ids.specifications:
    status = "PASS" if spec.total_fail == 0 else "FAIL"
    print(f"[{status}] {spec.name}: {spec.total_pass}/{spec.total_applicable} passed")
```

---

## Example 4: Spatial Hierarchy Validation with IFC4X3 Support

Validate spatial structure across all three IFC schema versions.

### Before (hard-coded for IFC4 only)

```python
model = ifcopenshell.open("infrastructure.ifc")
# Assumes IFC4 structure — breaks on IFC4X3 models with IfcFacility
buildings = model.by_type("IfcBuilding")
if len(buildings) == 0:
    print("ERROR: No building found")
# Result: False positive on valid IFC4X3 infrastructure models.
```

### After (schema-aware spatial validation)

```python
import ifcopenshell
import ifcopenshell.util.element

def validate_spatial_hierarchy(model):
    """Schema-aware spatial hierarchy validation for IFC2X3, IFC4, and IFC4X3."""
    schema = model.schema
    findings = []

    # IfcProject: exactly one required in all schemas
    projects = model.by_type("IfcProject")
    if len(projects) != 1:
        findings.append(("BLOCKER", f"Expected 1 IfcProject, found {len(projects)}"))
        return findings  # Cannot proceed without valid project

    # IfcSite: required in all schemas
    sites = model.by_type("IfcSite")
    if len(sites) == 0:
        findings.append(("BLOCKER", "No IfcSite found"))

    # Building or Facility: schema-dependent
    buildings = model.by_type("IfcBuilding")

    if schema == "IFC4X3":
        facilities = model.by_type("IfcFacility")
        if len(buildings) == 0 and len(facilities) == 0:
            findings.append(("BLOCKER", "No IfcBuilding or IfcFacility found (IFC4X3)"))
        elif len(buildings) == 0 and len(facilities) > 0:
            findings.append(("INFO",
                f"Using IfcFacility ({len(facilities)} found) instead of IfcBuilding (IFC4X3 infrastructure model)"))
    else:
        if len(buildings) == 0:
            findings.append(("BLOCKER", f"No IfcBuilding found ({schema})"))

    # IfcBuildingStorey: expected but not mandatory
    storeys = model.by_type("IfcBuildingStorey")
    if len(storeys) == 0 and len(buildings) > 0:
        findings.append(("WARNING", "No IfcBuildingStorey found"))

    # Spatial containment: all IfcElement instances must be contained
    uncontained = []
    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            uncontained.append(element)

    if uncontained:
        findings.append(("WARNING",
            f"{len(uncontained)} element(s) not spatially contained"))
        for elem in uncontained[:10]:  # Report first 10
            findings.append(("WARNING",
                f"  #{elem.id()} {elem.is_a()} '{elem.Name}'"))
        if len(uncontained) > 10:
            findings.append(("WARNING",
                f"  ... and {len(uncontained) - 10} more"))

    if not findings:
        findings.append(("INFO", f"Spatial hierarchy is complete ({schema})"))

    return findings


model = ifcopenshell.open("project.ifc")
for severity, message in validate_spatial_hierarchy(model):
    print(f"[{severity}] {message}")
```

---

## Example 5: Creating IDS Specifications Programmatically

Build IDS specifications in Python for automated requirement generation.

### Before (requirements documented in PDF, checked manually)

```
Project Requirements Document (PDF):
- All walls must have IsExternal property
- All walls must have FireRating property
- All doors must have a Uniclass 2015 classification
```

### After (machine-readable IDS specification)

```python
import ifctester.ids

# Create IDS document
ids = ifctester.ids.Ids(
    title="Building Handover Requirements",
    description="Minimum information requirements for design-stage delivery",
    author="QA Team",
    version="1.0"
)

# Specification 1: All walls must have FireRating
spec_wall_fire = ifctester.ids.Specification(
    name="Wall Fire Rating",
    ifcVersion=["IFC4", "IFC4X3"],
    description="All walls must declare fire rating for code compliance",
    minOccurs=1
)
# (Add applicability and requirement facets per IDS schema)
ids.specifications_.append(spec_wall_fire)

# Specification 2: All doors must have classification
spec_door_class = ifctester.ids.Specification(
    name="Door Classification",
    ifcVersion=["IFC4", "IFC4X3"],
    description="All doors must have Uniclass 2015 classification reference",
    minOccurs=1
)
ids.specifications_.append(spec_door_class)

# Export to IDS XML file
ids.to_xml("handover_requirements.ids")
print("IDS specification created: handover_requirements.ids")

# Validate a model against this specification
import ifcopenshell
import ifctester

model = ifcopenshell.open("project.ifc")
ids_loaded = ifctester.open("handover_requirements.ids")
ids_loaded.validate(model)

for spec in ids_loaded.specifications:
    status = "PASS" if spec.total_fail == 0 else "FAIL"
    print(f"[{status}] {spec.name}: {spec.total_pass}/{spec.total_applicable}")
```

---

## Example 6: Bonsai-Integrated Validation (Inside Blender)

Run validation from within a Bonsai/Blender session.

### Before (exporting and validating externally)

```python
# Export IFC, open terminal, run validation separately
bpy.ops.export_ifc.bim(filepath="/tmp/export.ifc")
# Then manually: python -m ifcopenshell.validate /tmp/export.ifc
# Result: Extra step, breaks native IFC workflow, loses Bonsai context.
```

### After (validate directly from Bonsai)

```python
import ifcopenshell
import ifcopenshell.validate
import ifcopenshell.util.element
from bonsai.bim.ifc import IfcStore

# Access the live IFC model (no export needed)
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open or create an IFC project first.")

schema = model.schema
print(f"Validating {schema} model...")

# Phase 1: Schema validation
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

errors = [s for s in json_log.statements if s["level"] == "ERROR"]
if errors:
    for e in errors:
        print(f"SCHEMA ERROR: {e['message']}")
    raise RuntimeError(f"Schema validation failed with {len(errors)} error(s). Fix before proceeding.")

# Phase 2: Spatial containment check
uncontained = []
for element in model.by_type("IfcElement"):
    container = ifcopenshell.util.element.get_container(element)
    if container is None:
        uncontained.append(element)

if uncontained:
    print(f"WARNING: {len(uncontained)} element(s) lack spatial containment:")
    for elem in uncontained[:5]:
        print(f"  #{elem.id()} {elem.is_a()} '{elem.Name}'")
else:
    print("All elements are spatially contained.")

# Phase 3: Property spot check
for wall in model.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    common = psets.get("Pset_WallCommon", {})
    if "IsExternal" not in common:
        print(f"WARNING: #{wall.id()} '{wall.Name}' missing Pset_WallCommon.IsExternal")

print("Validation complete.")
```

---

## Example 7: Version-Specific Classification Validation

Handle the classification reference identifier difference between IFC2X3 and IFC4/IFC4X3.

### Before (breaks on IFC2X3)

```python
for ref in model.by_type("IfcClassificationReference"):
    # IFC4 attribute — crashes on IFC2X3 where it is called ItemReference
    print(ref.Identification)
```

### After (schema-aware classification check)

```python
import ifcopenshell
import ifcopenshell.util.classification

model = ifcopenshell.open("project.ifc")
schema = model.schema

# Check classification references with correct attribute per schema
for ref in model.by_type("IfcClassificationReference"):
    if schema == "IFC2X3":
        identifier = ref.ItemReference
    else:
        identifier = ref.Identification

    if not identifier:
        print(f"WARNING: #{ref.id()} IfcClassificationReference has no identifier ({schema})")
    else:
        print(f"OK: #{ref.id()} classified as '{identifier}'")

# Check element-level classification coverage
total_elements = len(model.by_type("IfcElement"))
classified = 0
for element in model.by_type("IfcElement"):
    refs = ifcopenshell.util.classification.get_references(element)
    if refs:
        classified += 1

print(f"Classification coverage: {classified}/{total_elements} elements "
      f"({100 * classified // max(total_elements, 1)}%)")
```
