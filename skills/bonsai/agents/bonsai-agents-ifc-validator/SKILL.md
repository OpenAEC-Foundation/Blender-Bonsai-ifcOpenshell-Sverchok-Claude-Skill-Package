---
name: bonsai-agents-ifc-validator
description: >
  Use when validating, auditing, or verifying IFC model quality in Bonsai projects.
  Provides systematic checks for spatial hierarchy completeness, property set compliance,
  geometry validity, classification correctness, and IDS (Information Delivery Specification)
  conformance using ifctester. Prevents shipping models with missing spatial containment
  or incomplete property sets.
  Keywords: IFC validation, audit, quality check, spatial hierarchy, property set, IDS, ifctester, model quality, Bonsai validation, compliance, check my IFC file, is my model correct, find errors.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender with Bonsai addon."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai IFC Validator Agent

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Blender 4.2.0+ | Python 3.11
> **IFC Schemas**: IFC2X3, IFC4, IFC4X3
> **Dependencies**: bonsai-core-architecture, bonsai-syntax-properties, ifcos-impl-validation

## Quick Reference: When to Activate

Activate this validator when the user:

- Requests validation, auditing, checking, or verification of an IFC model
- Asks to run quality assurance (QA) on a Bonsai project
- Needs to verify IFC compliance before model handover or delivery
- Wants to check an IFC model against an IDS specification
- Asks to diagnose structural or data issues in an IFC file
- Requests a pre-submission or pre-export quality check
- Needs to verify spatial hierarchy, property completeness, or classification correctness

## Critical Warnings

1. **ALWAYS** run validation in the prescribed order: Schema → Spatial → Properties → Geometry → Classification → IDS. Schema failures invalidate all subsequent checks.
2. **ALWAYS** check `model.schema` before applying version-specific validation logic. IFC2X3, IFC4, and IFC4X3 have different entity sets and rules.
3. **NEVER** assume a model is valid because `ifcopenshell.validate.validate()` completes without exception. Validation issues are logged, NOT raised. Use `json_logger` or `LogDetectionHandler` to detect issues.
4. **ALWAYS** use `ifcopenshell.util.element.get_psets()` for property reading. NEVER traverse `IsDefinedBy` relationships manually.
5. **ALWAYS** use `ifctester` for IDS validation. `ifcopenshell.validate` checks schema compliance only — it does NOT check project-specific requirements.
6. **NEVER** confuse schema validation with IDS validation. Schema validation verifies IFC file structure per EXPRESS schema. IDS validation verifies IFC data meets project information requirements.
7. **ALWAYS** access the IFC model via `tool.Ifc.get()` inside Bonsai or `IfcStore.get_file()`. NEVER access `IfcStore.file` directly.
8. **ALWAYS** guard against `None` return from `IfcStore.get_file()` before running any validation.

---

## Validation Process: Execution Order

Execute these phases sequentially. If Phase 1 (Schema) produces BLOCKER-level issues, STOP and report before continuing.

### Phase 1: Schema Validation

Validates IFC file structure against the EXPRESS schema definition.

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("project.ifc")  # or tool.Ifc.get() inside Bonsai

# Step 1: Basic schema validation
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

schema_errors = [s for s in json_log.statements if s["level"] == "ERROR"]
schema_warnings = [s for s in json_log.statements if s["level"] == "WARNING"]

# Step 2: If basic passes, run EXPRESS WHERE rules (10-100x slower)
if not schema_errors:
    json_log_express = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, json_log_express, express_rules=True)
```

**Severity mapping:**
| Condition | Severity |
|-----------|----------|
| Any ERROR in json_log.statements | BLOCKER |
| Any WARNING in json_log.statements | WARNING |
| EXPRESS WHERE rule violation | WARNING |
| Clean schema validation pass | INFO (proceed) |

### Phase 2: Spatial Hierarchy Validation

Verifies the required IFC spatial structure exists and all elements are contained.

```python
import ifcopenshell
import ifcopenshell.util.element

def validate_spatial_hierarchy(model):
    errors = []
    schema = model.schema

    # 2a: Verify IfcProject exists (exactly one)
    projects = model.by_type("IfcProject")
    if len(projects) != 1:
        errors.append(("BLOCKER", f"Expected 1 IfcProject, found {len(projects)}"))
        return errors

    # 2b: Verify IfcSite exists
    sites = model.by_type("IfcSite")
    if len(sites) == 0:
        errors.append(("BLOCKER", "No IfcSite found"))

    # 2c: Verify IfcBuilding exists
    buildings = model.by_type("IfcBuilding")
    if len(buildings) == 0:
        errors.append(("BLOCKER", "No IfcBuilding found"))

    # 2d: Verify IfcBuildingStorey exists (standard buildings)
    storeys = model.by_type("IfcBuildingStorey")
    if len(storeys) == 0 and len(buildings) > 0:
        errors.append(("WARNING", "No IfcBuildingStorey found"))

    # 2e: Verify spatial decomposition chain
    # IFC4X3 allows IfcFacility / IfcFacilityPart as alternatives
    if schema == "IFC4X3":
        facilities = model.by_type("IfcFacility")
        if len(buildings) == 0 and len(facilities) == 0:
            errors.append(("BLOCKER", "No IfcBuilding or IfcFacility found"))

    # 2f: Check all physical elements have spatial containment
    for element in model.by_type("IfcElement"):
        container = ifcopenshell.util.element.get_container(element)
        if container is None:
            errors.append((
                "WARNING",
                f"#{element.id()} {element.is_a()} '{element.Name}': "
                f"Not contained in any spatial element"
            ))

    return errors
```

**Severity mapping:**
| Condition | Severity |
|-----------|----------|
| Missing IfcProject or count != 1 | BLOCKER |
| Missing IfcSite | BLOCKER |
| Missing IfcBuilding (IFC2X3/IFC4) | BLOCKER |
| Missing IfcBuilding AND IfcFacility (IFC4X3) | BLOCKER |
| Missing IfcBuildingStorey | WARNING |
| IfcElement without spatial containment | WARNING |
| Spatial hierarchy complete | INFO |

### Phase 3: Property Set Compliance

Validates property sets exist and contain required properties.

```python
import ifcopenshell.util.element

def validate_property_compliance(model, requirements):
    """
    requirements: dict of {ifc_class: {pset_name: [prop_names]}}
    Example: {"IfcWall": {"Pset_WallCommon": ["IsExternal", "FireRating"]}}
    """
    errors = []
    for ifc_class, pset_reqs in requirements.items():
        elements = model.by_type(ifc_class)
        for element in elements:
            psets = ifcopenshell.util.element.get_psets(element)
            for pset_name, required_props in pset_reqs.items():
                if pset_name not in psets:
                    errors.append((
                        "WARNING",
                        f"#{element.id()} {element.is_a()} '{element.Name}': "
                        f"Missing pset '{pset_name}'"
                    ))
                    continue
                for prop in required_props:
                    val = psets[pset_name].get(prop)
                    if val is None:
                        errors.append((
                            "WARNING",
                            f"#{element.id()} {element.Name}: "
                            f"Missing {pset_name}.{prop}"
                        ))
    return errors
```

**Standard property requirements by element type:**

| IFC Class | Standard Pset | Key Properties |
|-----------|--------------|----------------|
| `IfcWall` | `Pset_WallCommon` | IsExternal, LoadBearing, FireRating |
| `IfcSlab` | `Pset_SlabCommon` | IsExternal, LoadBearing |
| `IfcDoor` | `Pset_DoorCommon` | IsExternal, FireRating |
| `IfcWindow` | `Pset_WindowCommon` | IsExternal, ThermalTransmittance |
| `IfcBeam` | `Pset_BeamCommon` | LoadBearing |
| `IfcColumn` | `Pset_ColumnCommon` | LoadBearing |
| `IfcSpace` | `Pset_SpaceCommon` | IsExternal, GrossPlannedArea |

**Severity mapping:**
| Condition | Severity |
|-----------|----------|
| Standard pset completely missing on element | WARNING |
| Required property missing from existing pset | WARNING |
| Property value is empty/null | INFO |
| All required properties present | INFO |

### Phase 4: Geometry Validation

Checks geometry representations exist and are valid.

```python
def validate_geometry(model):
    errors = []

    for element in model.by_type("IfcElement"):
        # 4a: Check element has a representation
        if not element.Representation:
            errors.append((
                "WARNING",
                f"#{element.id()} {element.is_a()} '{element.Name}': "
                f"No geometry representation"
            ))
            continue

        # 4b: Check representation has items
        for rep in element.Representation.Representations:
            if not rep.Items or len(rep.Items) == 0:
                errors.append((
                    "WARNING",
                    f"#{element.id()} {element.Name}: "
                    f"Empty representation '{rep.RepresentationIdentifier}'"
                ))

    # 4c: Check for placement consistency
    for element in model.by_type("IfcElement"):
        if element.ObjectPlacement is None:
            errors.append((
                "INFO",
                f"#{element.id()} {element.is_a()} '{element.Name}': "
                f"No IfcLocalPlacement"
            ))

    return errors
```

**Severity mapping:**
| Condition | Severity |
|-----------|----------|
| IfcElement with no Representation at all | WARNING |
| Empty representation (no Items) | WARNING |
| Missing IfcLocalPlacement | INFO |
| Valid geometry present | INFO |

### Phase 5: Classification Validation

Verifies classification references (Uniclass, OmniClass, NL-SfB, etc.) are present and correctly structured.

```python
import ifcopenshell.util.classification

def validate_classifications(model, required_system=None):
    errors = []

    # 5a: Check classification systems exist
    systems = model.by_type("IfcClassification")
    if len(systems) == 0:
        errors.append(("INFO", "No classification system referenced in model"))
        return errors

    if required_system:
        system_names = [s.Name for s in systems]
        if required_system not in system_names:
            errors.append((
                "WARNING",
                f"Required classification system '{required_system}' not found. "
                f"Available: {system_names}"
            ))

    # 5b: Check elements have classification references
    classified_count = 0
    unclassified = []
    for element in model.by_type("IfcElement"):
        refs = ifcopenshell.util.classification.get_references(element)
        if refs:
            classified_count += 1
        else:
            unclassified.append(element)

    total = len(model.by_type("IfcElement"))
    if unclassified:
        errors.append((
            "INFO",
            f"{len(unclassified)}/{total} elements lack classification references"
        ))

    # 5c: Validate classification reference format
    for ref in model.by_type("IfcClassificationReference"):
        if not ref.Identification and not ref.ItemReference:
            errors.append((
                "WARNING",
                f"#{ref.id()} IfcClassificationReference: "
                f"Missing Identification/ItemReference"
            ))

    return errors
```

**Severity mapping:**
| Condition | Severity |
|-----------|----------|
| No classification system in model | INFO |
| Required classification system missing | WARNING |
| Element lacks classification reference | INFO |
| IfcClassificationReference with empty ID | WARNING |

### Phase 6: IDS Validation (Information Delivery Specification)

Validates the IFC model against an IDS specification file using ifctester.

```python
import ifctester
import ifctester.ids
import ifctester.reporter

def validate_ids(model, ids_path):
    """Validate model against IDS specification."""
    errors = []

    # 6a: Load IDS specification
    ids = ifctester.open(ids_path)

    # 6b: Run validation
    ids.validate(model)

    # 6c: Collect results per specification
    for spec in ids.specifications:
        total = spec.total_applicable
        passed = spec.total_pass
        failed = spec.total_fail

        if failed > 0:
            severity = "BLOCKER" if spec.minOccurs > 0 else "WARNING"
            errors.append((
                severity,
                f"IDS '{spec.name}': {failed}/{total} elements failed"
            ))

    # 6d: Generate report
    reporter = ifctester.reporter.Json(ids)
    reporter.report()
    report_data = reporter.to_string()

    return errors, report_data
```

**Bonsai UI access**: `BIM > IFC Tester` panel provides integrated IDS validation.

**Severity mapping:**
| Condition | Severity |
|-----------|----------|
| Required IDS specification failed | BLOCKER |
| Optional IDS specification failed | WARNING |
| All IDS specifications passed | INFO |
| No IDS specification provided | INFO (skip phase) |

---

## Decision Tree: Validation Severity Classification

```
Issue found during validation
|
+--> Does it violate IFC schema rules?
|    +--> YES --> BLOCKER (invalid IFC file, model cannot be trusted)
|    +--> NO --> continue
|
+--> Does it break spatial hierarchy completeness?
|    +--> Missing IfcProject/IfcSite/IfcBuilding? --> BLOCKER
|    +--> Missing IfcBuildingStorey? --> WARNING
|    +--> Element not spatially contained? --> WARNING
|    +--> NO --> continue
|
+--> Does it fail a required IDS specification?
|    +--> YES (minOccurs > 0) --> BLOCKER
|    +--> YES (optional) --> WARNING
|    +--> NO --> continue
|
+--> Does it involve missing required properties?
|    +--> Missing standard pset entirely? --> WARNING
|    +--> Missing individual property? --> WARNING
|    +--> Empty property value? --> INFO
|    +--> NO --> continue
|
+--> Does it involve geometry issues?
|    +--> No representation at all? --> WARNING
|    +--> Empty representation? --> WARNING
|    +--> Missing placement? --> INFO
|    +--> NO --> continue
|
+--> Does it involve classification?
     +--> Required system missing? --> WARNING
     +--> Element unclassified? --> INFO
     +--> Malformed reference? --> WARNING
```

**Severity Definitions:**

| Severity | Meaning | Action |
|----------|---------|--------|
| BLOCKER | Model is invalid or non-compliant. Delivery MUST NOT proceed. | Fix immediately. Re-validate after fix. |
| WARNING | Data quality issue. Model is technically valid but incomplete. | Fix before delivery if requirements demand it. |
| INFO | Observation. No action required unless project rules specify otherwise. | Document for awareness. |

---

## IDS Integration: ifctester Usage Patterns

### Core Workflow

```python
import ifctester
import ifctester.ids
import ifctester.reporter
import ifcopenshell

# Load IDS and IFC
ids = ifctester.open("specification.ids")
model = ifcopenshell.open("project.ifc")

# Validate and report
ids.validate(model)
ifctester.reporter.Console(ids).report()
```

### Report Formats

| Format | Reporter Class | Use Case |
|--------|---------------|----------|
| Console | `ifctester.reporter.Console(ids)` | Quick interactive checks |
| JSON | `ifctester.reporter.Json(ids)` | CI/CD pipelines |
| HTML | `ifctester.reporter.Html(ids)` | Stakeholder reports |
| BCF | `ifctester.reporter.Bcf(ids)` | BIM coordination |
| ODS | `ifctester.reporter.Ods(ids)` | Spreadsheet review |

### IDS Facet Types

| Facet | Checks |
|-------|--------|
| Entity | IFC class and predefined type |
| Attribute | Entity attribute values |
| Classification | Classification references present |
| Property | Property set values |
| Material | Material assignments |
| PartOf | Spatial/aggregation relationships |

---

## Model Access

| Context | Access Method |
|---------|--------------|
| Inside Bonsai | `model = tool.Ifc.get()` — ALWAYS check for None |
| Inside Bonsai (legacy) | `model = IfcStore.get_file()` |
| Standalone/headless | `model = ifcopenshell.open("project.ifc")` |
| Bonsai UI IDS | `BIM > IFC Tester` panel |

---

## Version-Specific Validation Notes

| Aspect | IFC2X3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| Spatial root | IfcProject → IfcSite → IfcBuilding | Same as IFC2X3 | IfcProject → IfcSite → IfcBuilding OR IfcFacility |
| Georeferencing | Property sets on IfcProject | IfcMapConversion + IfcProjectedCRS entities | Same as IFC4 + IfcMapConversionScaled |
| Classification | IfcClassificationReference.ItemReference | IfcClassificationReference.Identification | Same as IFC4 |
| Property templates | Limited | Full IfcPropertySetTemplate support | Full support |
| IDS support | Yes (via ifctester) | Yes (via ifctester) | Yes (via ifctester) |

---

## References

- [Validation Methods](references/methods.md) — Validation rules, ifctester API, and function signatures
- [Validation Examples](references/examples.md) — Complete validation workflow examples
- [Anti-Patterns](references/anti-patterns.md) — Common validation mistakes and how to avoid them

## Dependencies

- **bonsai-core-architecture** — IfcStore access, tool.Ifc, three-layer architecture
- **bonsai-syntax-properties** — Property set reading and validation patterns
- **ifcos-impl-validation** — ifcopenshell.validate, ifctester, georeference validation

## Sources

- Bonsai Documentation: https://docs.bonsaibim.org
- IfcOpenShell Validation: https://docs.ifcopenshell.org
- buildingSMART IDS: https://technical.buildingsmart.org/projects/information-delivery-specification-ids/
- IFC Schema Specifications: https://standards.buildingsmart.org/IFC/
