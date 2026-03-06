# IFC Validation Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Python 3.11
> **IFC Schemas**: IFC2X3, IFC4, IFC4X3

---

## Anti-Pattern 1: Assuming `validate()` Raises Exceptions on Invalid Models

### Wrong

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("project.ifc")

try:
    ifcopenshell.validate.validate(model, logger)
    print("Model is valid!")  # WRONG: validate() never raises on invalid data
except Exception:
    print("Model is invalid!")
```

### Why This Is Wrong

`ifcopenshell.validate.validate()` **logs** validation issues to the provided logger. It does NOT raise exceptions when the model contains schema violations. The function completes normally regardless of whether the model is valid or invalid. Wrapping it in `try/except` gives a false sense of security — every model appears to pass.

### Correct

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("project.ifc")

json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

if json_log.statements:
    errors = [s for s in json_log.statements if s["level"] == "ERROR"]
    print(f"Model has {len(errors)} error(s) and {len(json_log.statements)} total issue(s)")
else:
    print("Model is valid")
```

---

## Anti-Pattern 2: Running EXPRESS WHERE Rules on First Pass

### Wrong

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("large_hospital.ifc")  # 500MB model

# First validation pass with EXPRESS WHERE rules enabled
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log, express_rules=True)
# Result: Takes 30+ minutes on large models. May time out in CI/CD.
```

### Why This Is Wrong

EXPRESS WHERE rules are 10-100x slower than basic schema validation. Running them on the first pass wastes time because basic schema errors (missing attributes, wrong types) invalidate WHERE rule results anyway. If the model has fundamental schema errors, the EXPRESS rule output is unreliable.

### Correct

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("large_hospital.ifc")

# Step 1: Basic schema validation (fast)
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

errors = [s for s in json_log.statements if s["level"] == "ERROR"]
if errors:
    print(f"BLOCKED: {len(errors)} schema error(s). Fix before running EXPRESS rules.")
else:
    # Step 2: EXPRESS WHERE rules (slow, only after basic passes)
    json_log_express = ifcopenshell.validate.json_logger()
    ifcopenshell.validate.validate(model, json_log_express, express_rules=True)
    print(f"EXPRESS rule issues: {len(json_log_express.statements)}")
```

---

## Anti-Pattern 3: Confusing Schema Validation with IDS Validation

### Wrong

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("project.ifc")

json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

if not json_log.statements:
    print("Model meets all project requirements!")  # WRONG
```

### Why This Is Wrong

`ifcopenshell.validate` checks that the IFC file conforms to the EXPRESS schema definition (correct entity types, valid attribute values, proper relationships). It does NOT check whether the model contains the information your project requires (e.g., "all walls must have a fire rating"). A schema-valid model can be completely empty of useful data. Schema validation and IDS validation answer different questions:

- **Schema validation**: "Is this a valid IFC file?"
- **IDS validation**: "Does this IFC file contain the information we need?"

### Correct

```python
import ifcopenshell
import ifcopenshell.validate
import ifctester
import ifctester.reporter

model = ifcopenshell.open("project.ifc")

# Step 1: Schema validation (structural correctness)
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

schema_errors = [s for s in json_log.statements if s["level"] == "ERROR"]
if schema_errors:
    print(f"Schema invalid: {len(schema_errors)} error(s). Cannot check requirements.")
else:
    print("Schema valid. Checking project requirements...")

    # Step 2: IDS validation (project requirement conformance)
    ids = ifctester.open("project_requirements.ids")
    ids.validate(model)

    for spec in ids.specifications:
        if spec.total_fail > 0:
            print(f"FAIL: {spec.name} — {spec.total_fail}/{spec.total_applicable} elements non-conformant")
        else:
            print(f"PASS: {spec.name}")
```

---

## Anti-Pattern 4: Traversing `IsDefinedBy` Manually Instead of Using `get_psets()`

### Wrong

```python
import ifcopenshell

model = ifcopenshell.open("project.ifc")

for wall in model.by_type("IfcWall"):
    for rel in wall.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            pset = rel.RelatingPropertyDefinition
            if pset.is_a("IfcPropertySet") and pset.Name == "Pset_WallCommon":
                for prop in pset.HasProperties:
                    if prop.Name == "FireRating":
                        print(f"Wall {wall.Name}: FireRating = {prop.NominalValue.wrappedValue}")
```

### Why This Is Wrong

Manual traversal of `IsDefinedBy` relationships is verbose, error-prone, and fragile. It misses property sets inherited through `IfcRelDefinesByType` (type-level psets), does not handle `IfcPropertySetTemplate`, and breaks when the relationship structure varies between IFC versions. The `ifcopenshell.util.element.get_psets()` utility handles all these cases and returns a clean dictionary.

### Correct

```python
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("project.ifc")

for wall in model.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    fire_rating = psets.get("Pset_WallCommon", {}).get("FireRating")
    if fire_rating is not None:
        print(f"Wall {wall.Name}: FireRating = {fire_rating}")
    else:
        print(f"Wall {wall.Name}: FireRating missing")
```

---

## Anti-Pattern 5: Accessing `IfcStore.file` Directly Without None Check

### Wrong

```python
from bonsai.bim.ifc import IfcStore

# Direct attribute access — crashes if no project is loaded
model = IfcStore.file
walls = model.by_type("IfcWall")  # AttributeError: 'NoneType' has no attribute 'by_type'
```

### Why This Is Wrong

`IfcStore.file` is `None` when no IFC project is loaded in Bonsai. Accessing it directly without a guard causes an `AttributeError` crash. In a validation context, this is especially dangerous because the validator may run before the user has opened a project, or after a project has been closed. The correct accessor method is `IfcStore.get_file()` (or `tool.Ifc.get()` in the three-layer architecture), followed by a `None` check.

### Correct

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file before running validation.")

walls = model.by_type("IfcWall")
```

Or using `tool.Ifc` in the three-layer architecture:

```python
model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")
```

---

## Anti-Pattern 6: Ignoring Schema Version When Validating Classification References

### Wrong

```python
import ifcopenshell

model = ifcopenshell.open("legacy_project.ifc")  # IFC2X3 file

for ref in model.by_type("IfcClassificationReference"):
    code = ref.Identification  # AttributeError on IFC2X3
    print(f"Classification: {code}")
```

### Why This Is Wrong

The classification reference identifier attribute changed names between IFC schema versions:
- **IFC2X3**: `IfcClassificationReference.ItemReference`
- **IFC4 / IFC4X3**: `IfcClassificationReference.Identification`

Accessing the wrong attribute causes an `AttributeError` on IFC2X3 models. Validation code must check `model.schema` and use the correct attribute name.

### Correct

```python
import ifcopenshell

model = ifcopenshell.open("project.ifc")
schema = model.schema

for ref in model.by_type("IfcClassificationReference"):
    if schema == "IFC2X3":
        code = ref.ItemReference
    else:
        code = ref.Identification

    if not code:
        print(f"WARNING: #{ref.id()} IfcClassificationReference has no identifier")
    else:
        print(f"OK: #{ref.id()} classified as '{code}'")
```

---

## Anti-Pattern 7: Skipping Validation Phase Order

### Wrong

```python
import ifcopenshell
import ifctester

model = ifcopenshell.open("project.ifc")

# Jump straight to IDS validation without checking schema first
ids = ifctester.open("requirements.ids")
ids.validate(model)
# Result: IDS reports may be unreliable if the model has schema-level corruption.
# Missing entities or broken relationships cause misleading IDS failures.
```

### Why This Is Wrong

The six validation phases must execute sequentially: Schema → Spatial → Properties → Geometry → Classification → IDS. Each phase depends on the integrity verified by earlier phases. If the IFC file has schema-level corruption (Phase 1), spatial hierarchy checks (Phase 2) will produce misleading results. If the spatial hierarchy is broken, property checks (Phase 3) on mis-contained elements are unreliable. Running IDS validation (Phase 6) on a schema-invalid model may report false failures because the entities or relationships the IDS specification targets may be corrupted.

### Correct

```python
import ifcopenshell
import ifcopenshell.validate
import ifctester

model = ifcopenshell.open("project.ifc")

# Phase 1: Schema validation FIRST
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

errors = [s for s in json_log.statements if s["level"] == "ERROR"]
if errors:
    print(f"BLOCKED: {len(errors)} schema error(s). Fix before running further validation.")
    for e in errors:
        print(f"  {e['message']}")
    # STOP. Do not proceed to IDS.
else:
    print("Schema valid. Proceeding to IDS validation...")
    ids = ifctester.open("requirements.ids")
    ids.validate(model)
    for spec in ids.specifications:
        status = "PASS" if spec.total_fail == 0 else "FAIL"
        print(f"[{status}] {spec.name}")
```

---

## Anti-Pattern 8: Hard-Coding IFC4-Only Spatial Structure for IFC4X3 Models

### Wrong

```python
import ifcopenshell

model = ifcopenshell.open("rail_project.ifc")  # IFC4X3 infrastructure model

buildings = model.by_type("IfcBuilding")
if len(buildings) == 0:
    print("BLOCKER: No IfcBuilding found!")
    # False alarm: IFC4X3 infrastructure models use IfcFacility instead of IfcBuilding
```

### Why This Is Wrong

IFC4X3 introduced `IfcFacility` and `IfcFacilityPart` as alternatives to `IfcBuilding` for infrastructure projects (railways, roads, bridges, tunnels). An IFC4X3 rail model may have zero `IfcBuilding` entities and use `IfcFacility` instead. Hard-coding the IFC2X3/IFC4 spatial structure requirement produces false BLOCKER errors on valid IFC4X3 models.

### Correct

```python
import ifcopenshell

model = ifcopenshell.open("rail_project.ifc")
schema = model.schema

buildings = model.by_type("IfcBuilding")

if schema == "IFC4X3":
    facilities = model.by_type("IfcFacility")
    if len(buildings) == 0 and len(facilities) == 0:
        print("BLOCKER: No IfcBuilding or IfcFacility found (IFC4X3)")
    elif len(buildings) == 0:
        print(f"INFO: Using {len(facilities)} IfcFacility instance(s) (IFC4X3 infrastructure)")
else:
    if len(buildings) == 0:
        print(f"BLOCKER: No IfcBuilding found ({schema})")
```

---

## Summary Table

| # | Anti-Pattern | Root Cause | Severity |
|---|-------------|-----------|----------|
| 1 | Assuming `validate()` raises exceptions | Misunderstanding API contract | Critical — false pass on every model |
| 2 | Running EXPRESS WHERE rules first | Performance unawareness | High — CI/CD timeouts, wasted compute |
| 3 | Confusing schema vs. IDS validation | Conceptual confusion | Critical — "valid" file with no useful data |
| 4 | Manual `IsDefinedBy` traversal | Ignoring utility functions | Medium — misses type-level psets, fragile code |
| 5 | Accessing `IfcStore.file` without None check | Missing guard clause | High — runtime crash in Bonsai |
| 6 | Ignoring schema version for classifications | Hard-coded IFC4 assumptions | High — crash on IFC2X3 models |
| 7 | Skipping validation phase order | Ignoring dependencies between phases | High — misleading IDS results |
| 8 | Hard-coding IFC4 spatial structure for IFC4X3 | Not supporting infrastructure models | Medium — false BLOCKERs on valid IFC4X3 |
