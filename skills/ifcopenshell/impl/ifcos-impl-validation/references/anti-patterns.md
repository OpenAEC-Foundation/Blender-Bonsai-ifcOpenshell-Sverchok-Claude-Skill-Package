# Validation Anti-Patterns

## Anti-Pattern 1: Enabling EXPRESS WHERE Rules on First Pass

**WRONG:**
```python
# Running full EXPRESS WHERE rules immediately on an untested model
ifcopenshell.validate.validate(model, logger, express_rules=True)
# This is 10-100x slower than basic validation and may take hours on large models
```

**CORRECT:**
```python
# Phase 1: Run basic validation first (fast)
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

if not json_log.statements:
    # Phase 2: Only run EXPRESS rules after basic validation passes
    ifcopenshell.validate.validate(model, json_log, express_rules=True)
else:
    print("Fix basic issues first before running EXPRESS WHERE rules")
```

**Why:** EXPRESS WHERE rule evaluation requires the C++ backend to evaluate EXPRESS expressions. On models with thousands of entities, this adds 10-100x processing time. Running basic validation first catches type errors, cardinality violations, and format issues at minimal cost. Only invest in WHERE rule checking after the cheap checks pass.

---

## Anti-Pattern 2: Assuming validate() Raises Exceptions on Invalid Models

**WRONG:**
```python
try:
    ifcopenshell.validate.validate(model, logger)
    print("Model is valid!")  # WRONG: validate() does not raise on invalid models
except Exception:
    print("Model is invalid!")
```

**CORRECT:**
```python
# Option A: Use LogDetectionHandler
detection = ifcopenshell.validate.LogDetectionHandler()
logger.addHandler(detection)
ifcopenshell.validate.validate(model, logger)

if detection.message_logged:
    print("Model has issues")
else:
    print("Model is valid")

# Option B: Use json_logger
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

if json_log.statements:
    print(f"Model has {len(json_log.statements)} issues")
else:
    print("Model is valid")
```

**Why:** `validate()` emits validation results via the logger. It does NOT raise exceptions for validation failures. An exception-based approach will always report the model as valid, even when it has schema violations. ALWAYS use `LogDetectionHandler` or `json_logger` to detect issues programmatically.

---

## Anti-Pattern 3: Confusing Schema Validation with IDS Validation

**WRONG:**
```python
# Checking project-specific requirements with ifcopenshell.validate
ifcopenshell.validate.validate(model, logger)
# This does NOT check if walls have FireRating properties
# This does NOT check if elements have correct classification references
# This does NOT check if naming conventions are followed
```

**CORRECT:**
```python
# Schema compliance → ifcopenshell.validate
ifcopenshell.validate.validate(model, logger)

# Project information requirements → ifctester with IDS
ids = ifctester.open("project_requirements.ids")
ids.validate(model)
reporter = ifctester.reporter.Console(ids)
reporter.report()
```

**Why:** `ifcopenshell.validate` checks whether the IFC file conforms to the EXPRESS schema (correct types, cardinality, valid enumerations). It does NOT verify project-specific information requirements like "all walls must have a fire rating" or "all elements must be classified with Uniclass 2015". For project requirements, ALWAYS use `ifctester` with an IDS specification.

---

## Anti-Pattern 4: Using IfcMapConversionScaled in IFC4 Models

**WRONG:**
```python
# IFC4 model — IfcMapConversionScaled does NOT exist in IFC4
model = ifcopenshell.file(schema="IFC4")
ifcopenshell.api.run("georeference.add_georeferencing", model,
    ifc_class="IfcMapConversionScaled")  # ERROR: Not available in IFC4
```

**CORRECT:**
```python
# IFC4 — use IfcMapConversion only
model = ifcopenshell.file(schema="IFC4")
ifcopenshell.api.run("georeference.add_georeferencing", model,
    ifc_class="IfcMapConversion")

# IFC4X3 — IfcMapConversionScaled IS available
model_infra = ifcopenshell.file(schema="IFC4X3")
ifcopenshell.api.run("georeference.add_georeferencing", model_infra,
    ifc_class="IfcMapConversionScaled")
```

**Why:** `IfcMapConversionScaled` is an IFC4X3 addition for infrastructure projects that need a scale factor in the map conversion. It does NOT exist in the IFC4 schema. Attempting to create it in an IFC4 model causes a schema error. ALWAYS check the target schema version before using version-specific entity classes.

---

## Anti-Pattern 5: Editing Georeferencing Before Adding It

**WRONG:**
```python
# Trying to edit georeferencing that does not exist yet
model = ifcopenshell.file(schema="IFC4")
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={"Eastings": 155000.0, "Northings": 463000.0})
# ERROR: No IfcMapConversion or IfcProjectedCRS entity exists to edit
```

**CORRECT:**
```python
# Step 1: Create the entities first
ifcopenshell.api.run("georeference.add_georeferencing", model)

# Step 2: Then edit them
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={"Eastings": 155000.0, "Northings": 463000.0})
```

**Why:** `edit_georeferencing` modifies existing `IfcMapConversion` and `IfcProjectedCRS` entities. If these entities do not exist, the edit call fails. ALWAYS call `add_georeferencing` first to create the empty entities, then `edit_georeferencing` to populate them.

---

## Anti-Pattern 6: Confusing Project North with True North

**WRONG:**
```python
# Setting true north via coordinate_operation (WRONG: this is Project North)
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    coordinate_operation={
        "XAxisAbscissa": cos(radians(true_north_angle)),  # WRONG
        "XAxisOrdinate": sin(radians(true_north_angle)),  # WRONG
    })
```

**CORRECT:**
```python
from math import cos, sin, radians

# Project North rotation → coordinate_operation (XAxisAbscissa/XAxisOrdinate)
project_north_rotation = -5.0  # degrees from Grid North to Project North
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    coordinate_operation={
        "XAxisAbscissa": cos(radians(project_north_rotation)),
        "XAxisOrdinate": sin(radians(project_north_rotation)),
    })

# True North → separate API call
true_north_angle = 5.0  # degrees anticlockwise from Y-axis
ifcopenshell.api.run("georeference.edit_true_north", model,
    true_north=true_north_angle)
```

**Why:** Project North (grid rotation in the coordinate operation) and True North (geographic north for solar analysis) are distinct concepts in IFC. `XAxisAbscissa`/`XAxisOrdinate` define the rotation from Grid North to Project North. True North is set independently via `edit_true_north`. Confusing them results in incorrect solar studies, incorrect map alignment, or both.

---

## Anti-Pattern 7: Moving WCS Without Surveying Reason

**WRONG:**
```python
# Moving WCS to "center the model" — this is almost never correct
ifcopenshell.api.run("georeference.edit_wcs", model,
    x=500.0, y=300.0, z=0.0)
# ALL local placements are now offset by (500, 300, 0)
```

**CORRECT:**
```python
# Keep WCS at origin (default) unless surveyor explicitly requires it
ifcopenshell.api.run("georeference.edit_wcs", model,
    x=0.0, y=0.0, z=0.0, rotation=0.0, is_si=True)
```

**Why:** The World Coordinate System (WCS) origin affects ALL geometric placements in the model. Moving it without a specific surveying requirement causes all local placements to shift by the WCS offset. This creates subtle coordinate errors that are difficult to diagnose. NEVER move the WCS from (0,0,0) unless the project surveyor provides explicit instructions.

---

## Anti-Pattern 8: Not Checking Schema Before Version-Specific Validation

**WRONG:**
```python
# Assuming IFC4+ entities exist in every model
model = ifcopenshell.open("model.ifc")
map_conv = model.by_type("IfcMapConversion")  # Fails silently on IFC2X3
if not map_conv:
    print("No georeferencing")  # WRONG: IFC2X3 uses property sets, not entities
```

**CORRECT:**
```python
model = ifcopenshell.open("model.ifc")

if model.schema == "IFC2X3":
    # IFC2X3: georeferencing is in property sets
    psets = ifcopenshell.util.element.get_psets(model.by_type("IfcProject")[0])
    has_georef = "ePSet_MapConversion" in psets
elif model.schema in ("IFC4", "IFC4X3"):
    # IFC4+: georeferencing uses dedicated entities
    has_georef = len(model.by_type("IfcMapConversion")) > 0
```

**Why:** IFC2X3, IFC4, and IFC4X3 represent georeferencing differently. IFC2X3 uses property sets (`ePSet_MapConversion`, `ePSet_ProjectedCRS`) on `IfcProject`. IFC4+ uses dedicated entities (`IfcMapConversion`, `IfcProjectedCRS`). ALWAYS check `model.schema` before applying version-specific validation logic.

---

## Anti-Pattern 9: Validating Without Logging Configuration

**WRONG:**
```python
import ifcopenshell.validate
import logging

logger = logging.getLogger("ifcopenshell.validate")
# No handler configured — validation output is silently discarded
ifcopenshell.validate.validate(model, logger)
# No output visible, no way to detect issues
```

**CORRECT:**
```python
import ifcopenshell.validate
import logging

logger = logging.getLogger("ifcopenshell.validate")
logger.setLevel(logging.DEBUG)

# Add at least one handler to capture output
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))
logger.addHandler(handler)

ifcopenshell.validate.validate(model, logger)
```

**Why:** Python's logging module discards messages if no handler is configured. If you pass a bare logger to `validate()` without adding a handler and setting the level, all validation output is silently lost. ALWAYS configure at least one handler and set the appropriate log level before calling `validate()`. Alternatively, use `json_logger()` which collects results internally without requiring handler configuration.

---

## Anti-Pattern 10: Running IDS Validation Without Checking IFC Version Compatibility

**WRONG:**
```python
ids = ifctester.open("requirements.ids")  # Written for IFC4
ifc_file = ifcopenshell.open("old_model.ifc")  # IFC2X3 file
ids.validate(ifc_file)
# May produce misleading results if IDS specs target IFC4-only entities
```

**CORRECT:**
```python
ids = ifctester.open("requirements.ids")
ifc_file = ifcopenshell.open("old_model.ifc")

# Check version compatibility per specification
for spec in ids.specifications_:
    if spec.ifcVersion and ifc_file.schema not in spec.ifcVersion:
        print(f"WARNING: Spec '{spec.name}' targets {spec.ifcVersion}, "
              f"but model is {ifc_file.schema}")

# Run validation with version filtering
ids.validate(ifc_file, should_filter_version=True)
```

**Why:** IDS specifications can target specific IFC versions. Running an IFC4-targeted specification against an IFC2X3 model produces misleading results — entities may not exist, property sets may differ, and relationships may be structured differently. ALWAYS check version compatibility and use `should_filter_version=True` to skip specifications that do not match the model's schema version.
