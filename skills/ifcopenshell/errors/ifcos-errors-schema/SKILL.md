---
name: ifcos-errors-schema
description: "Documents IFC schema-related errors and pitfalls across IFC2X3, IFC4, and IFC4X3 including entity availability differences, attribute type changes, removed/added entities, IfcOpenShell ifcpatch for schema migration, and common SchemaError debugging. Activates when encountering IFC schema errors, migrating between IFC versions, or debugging entity not found errors."
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Schema Error Reference

## Quick Reference

### Decision Tree: Schema Error Diagnosis

```
Error when using IFC entity or attribute?
├── RuntimeError: entity "XYZ" not found
│   ├── Entity exists in a DIFFERENT schema version?
│   │   ├── YES → Check model.schema, use version-aware class name
│   │   │   ├── IfcBuildingElement → IFC4X3 uses IfcBuiltElement
│   │   │   ├── IfcWallStandardCase → IFC4X3 removed, use IfcWall
│   │   │   ├── IfcDoorStyle → IFC4+ uses IfcDoorType
│   │   │   └── IfcWindowStyle → IFC4+ uses IfcWindowType
│   │   └── NO → Entity does not exist in ANY schema
│   │       └── Check spelling and IFC specification
│   │
│   └── Entity is IFC4X3-only (infrastructure)?
│       └── IfcRoad, IfcBridge, IfcRailway, IfcAlignment, etc.
│           → ONLY available in schema="IFC4X3"
│
├── AttributeError: entity has no attribute "XYZ"
│   ├── PredefinedType on IFC2X3 elements?
│   │   └── Most elements lack PredefinedType in IFC2X3
│   ├── OwnerHistory is None in IFC2X3?
│   │   └── OwnerHistory is REQUIRED in IFC2X3, OPTIONAL in IFC4+
│   └── Attribute renamed between versions?
│       └── Check attribute change table below
│
├── Need to migrate between schema versions?
│   └── Use ifcpatch Migrate recipe
│       ├── IFC2X3 → IFC4: ifcpatch.execute({...recipe: "Migrate", arguments: ["IFC4"]})
│       └── IFC4 → IFC4X3: ifcpatch.execute({...recipe: "Migrate", arguments: ["IFC4X3"]})
│
└── Schema mismatch when combining files?
    └── ALWAYS check model.schema before transferring entities
        └── Use model.add(entity) ONLY between same-schema files
```

### Critical Warnings

- **ALWAYS** check `model.schema` before using schema-specific entities. NEVER assume the schema.
- **NEVER** use `IfcBuildingElement` in IFC4X3 code — it was renamed to `IfcBuiltElement`.
- **NEVER** use `IfcWallStandardCase` or any `*StandardCase` entity in IFC4X3 — they were all removed.
- **NEVER** use `IfcDoorStyle` or `IfcWindowStyle` in IFC4+ code — use `IfcDoorType` and `IfcWindowType`.
- **ALWAYS** provide `OwnerHistory` when creating entities in IFC2X3 files (it is REQUIRED).
- **NEVER** transfer entities between files of different schemas without migration.
- **ALWAYS** use `ifcopenshell.api.run()` for entity creation — it handles schema differences automatically.
- **ALWAYS** validate IFC files after schema migration with ifcpatch.

---

## Essential Error Patterns

### Error 1: Entity Not Found — Schema Version Mismatch

**Symptom**: `RuntimeError` when calling `model.by_type()` or `model.create_entity()` with an entity name that does not exist in the file's schema.

```python
# WRONG: IfcBuiltElement does not exist in IFC2X3 or IFC4
model = ifcopenshell.open("legacy_building.ifc")  # schema = "IFC2X3"
elements = model.by_type("IfcBuiltElement")  # RuntimeError!

# WRONG: IfcBuildingElement does not exist in IFC4X3
model = ifcopenshell.open("new_infra.ifc")  # schema = "IFC4X3"
elements = model.by_type("IfcBuildingElement")  # RuntimeError!
```

**Fix**: ALWAYS check `model.schema` and use the correct entity name.

```python
# CORRECT: Version-aware entity class selection
schema = model.schema
if schema == "IFC4X3":
    building_element_class = "IfcBuiltElement"
else:  # IFC2X3, IFC4
    building_element_class = "IfcBuildingElement"
elements = model.by_type(building_element_class)
```

### Error 2: StandardCase Entity Not Found in IFC4X3

**Symptom**: `RuntimeError` when querying `IfcWallStandardCase`, `IfcBeamStandardCase`, etc. in an IFC4X3 file.

IFC4 introduced 11 `*StandardCase` subtypes. IFC4X3 **removed all of them**, merging behavior back into parent classes.

**Removed in IFC4X3:**
- IfcBeamStandardCase → use IfcBeam
- IfcColumnStandardCase → use IfcColumn
- IfcDoorStandardCase → use IfcDoor
- IfcMemberStandardCase → use IfcMember
- IfcOpeningStandardCase → use IfcOpeningElement
- IfcPlateStandardCase → use IfcPlate
- IfcSlabStandardCase → use IfcSlab
- IfcSlabElementedCase → use IfcSlab
- IfcWallElementedCase → use IfcWall
- IfcWindowStandardCase → use IfcWindow

```python
# CORRECT: Version-safe wall query
schema = model.schema
if schema == "IFC2X3":
    walls = model.by_type("IfcWallStandardCase")
elif schema == "IFC4":
    walls = model.by_type("IfcWallStandardCase")  # exists in IFC4
else:  # IFC4X3
    walls = model.by_type("IfcWall")  # StandardCase removed
```

### Error 3: IFC4X3-Only Entities on Older Schemas

**Symptom**: `RuntimeError` when creating infrastructure entities on IFC2X3 or IFC4 files.

```python
# WRONG: IfcRoad only exists in IFC4X3
model = ifcopenshell.file(schema="IFC4")
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway")  # RuntimeError!
```

**IFC4X3-only entities** (partial list):
- Facilities: IfcFacility, IfcBridge, IfcRoad, IfcRailway, IfcMarineFacility
- Facility parts: IfcFacilityPart, IfcBridgePart, IfcRoadPart, IfcRailwayPart, IfcMarinePart
- Infrastructure: IfcAlignment, IfcBearing, IfcCourse, IfcKerb, IfcPavement, IfcRail, IfcTrackElement
- Geotechnical: IfcGeotechnicalElement, IfcBorehole, IfcGeomodel
- Other: IfcDeepFoundation, IfcLinearElement, IfcTransportationDevice

**Fix**: ALWAYS create IFC4X3 files for infrastructure projects.

```python
# CORRECT: Use IFC4X3 for infrastructure
model = ifcopenshell.file(schema="IFC4X3")
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway")  # Works
```

### Error 4: OwnerHistory Required in IFC2X3

**Symptom**: Invalid IFC2X3 file — `OwnerHistory` is `$` (null) but IFC2X3 requires it on all `IfcRoot` subclasses.

```python
# WRONG: OwnerHistory is REQUIRED in IFC2X3
model = ifcopenshell.file(schema="IFC2X3")
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    None,  # OwnerHistory = None → INVALID in IFC2X3!
    "MyWall"
)
```

**Fix**: Use `ifcopenshell.api.run()` which handles OwnerHistory automatically, or provide it explicitly.

```python
# CORRECT: api.run() handles OwnerHistory for all schemas
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="MyWall")

# CORRECT: Manual creation with explicit OwnerHistory
owner_history = ifcopenshell.api.run("owner.create_owner_history", model)
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    owner_history,  # REQUIRED in IFC2X3
    "MyWall"
)
```

### Error 5: IfcDoorStyle / IfcWindowStyle vs IfcDoorType / IfcWindowType

**Symptom**: `RuntimeError` when querying `IfcDoorType` in IFC2X3, or `IfcDoorStyle` in IFC4X3.

| Type Entity | IFC2X3 | IFC4 | IFC4X3 |
|-------------|--------|------|--------|
| IfcDoorStyle | YES | Deprecated | REMOVED |
| IfcWindowStyle | YES | Deprecated | REMOVED |
| IfcDoorType | NO | YES | YES |
| IfcWindowType | NO | YES | YES |

```python
# CORRECT: Version-safe type class selection
def get_door_type_class(schema):
    if schema == "IFC2X3":
        return "IfcDoorStyle"
    return "IfcDoorType"  # IFC4, IFC4X3

def get_window_type_class(schema):
    if schema == "IFC2X3":
        return "IfcWindowStyle"
    return "IfcWindowType"  # IFC4, IFC4X3
```

### Error 6: PredefinedType Not Available in IFC2X3

**Symptom**: `AttributeError` when accessing `element.PredefinedType` on IFC2X3 entities that lack this attribute.

Many entities gained `PredefinedType` in IFC4 that did not have it in IFC2X3: IfcBeam, IfcColumn, IfcDoor, IfcWall, IfcWindow, IfcSlab, and others.

```python
# WRONG: Assumes PredefinedType exists (fails on IFC2X3 IfcBeam)
for beam in model.by_type("IfcBeam"):
    print(beam.PredefinedType)  # AttributeError in IFC2X3!

# CORRECT: Check schema or use hasattr
for beam in model.by_type("IfcBeam"):
    if hasattr(beam, "PredefinedType"):
        print(beam.PredefinedType)
    else:
        print("No PredefinedType (IFC2X3)")
```

### Error 7: IfcContext Does Not Exist in IFC2X3

**Symptom**: `RuntimeError` querying `IfcContext` in IFC2X3. In IFC2X3, `IfcProject` inherits from `IfcObject`. In IFC4+, `IfcProject` inherits from `IfcContext` (new abstract class).

```python
# WRONG: IfcContext does not exist in IFC2X3
project = model.by_type("IfcContext")  # RuntimeError in IFC2X3!

# CORRECT: Use IfcProject directly (works in all versions)
project = model.by_type("IfcProject")[0]
```

---

## Schema Migration with ifcpatch

### Migration Recipe

The `Migrate` recipe in ifcpatch converts IFC files between schema versions. Upgrading is more reliable than downgrading.

```python
import ifcopenshell
import ifcpatch

# Upgrade IFC2X3 → IFC4
model = ifcopenshell.open("input_2x3.ifc")
output = ifcpatch.execute({
    "input": "input_2x3.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4"]
})
ifcpatch.write(output, "output_ifc4.ifc")

# Upgrade IFC4 → IFC4X3
model = ifcopenshell.open("input_ifc4.ifc")
output = ifcpatch.execute({
    "input": "input_ifc4.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4X3"]
})
ifcpatch.write(output, "output_ifc4x3.ifc")
```

### Migration Limitations

- The Migrate recipe is **experimental** — ALWAYS validate output.
- Upgrading (IFC2X3 → IFC4 → IFC4X3) is more stable than downgrading.
- Entity renames are handled automatically (IfcBuildingElement → IfcBuiltElement).
- StandardCase merging is handled automatically.
- Custom property sets are preserved.
- Geometry may require manual review after migration.
- ALWAYS compare entity counts before and after migration.

### Post-Migration Validation

```python
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("migrated.ifc")
logger = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, logger)

for error in logger.statements:
    print(f"{error['severity']}: {error['message']}")
```

---

## Entity Availability Quick Reference

### Entities That Changed Name

| IFC2X3 | IFC4 | IFC4X3 |
|--------|------|--------|
| IfcBuildingElement | IfcBuildingElement | **IfcBuiltElement** |
| IfcBuildingElementType | IfcBuildingElementType | **IfcBuiltElementType** |
| IfcDoorStyle | IfcDoorType (new) | IfcDoorType |
| IfcWindowStyle | IfcWindowType (new) | IfcWindowType |

### Entities Removed in IFC4 (from IFC2X3)

- IfcBezierCurve → use IfcBSplineCurve
- IfcRationalBezierCurve → use IfcBSplineCurve
- Ifc2DCompositeCurve → use IfcCompositeCurve
- IfcCalendarDate, IfcLocalTime, IfcDateAndTime → use IfcDateTime (string)
- IfcMove → use IfcTask
- IfcOrderRequest → use IfcTask
- IfcRelAssignsTasks → use IfcRelAssignsToProcess
- IfcElectricDistributionPoint → use IfcElectricDistributionBoard

### Entities Removed in IFC4X3 (from IFC4)

All StandardCase subtypes:
- IfcBeamStandardCase, IfcColumnStandardCase, IfcDoorStandardCase
- IfcMemberStandardCase, IfcOpeningStandardCase, IfcPlateStandardCase
- IfcSlabStandardCase, IfcSlabElementedCase, IfcWallElementedCase
- IfcWindowStandardCase
- IfcPresentationStyleAssignment → use IfcStyledItem directly

### Attribute Changes Across Versions

| Entity | Attribute | IFC2X3 | IFC4+ |
|--------|-----------|--------|-------|
| IfcRoot | OwnerHistory | REQUIRED | OPTIONAL |
| IfcMaterial | Description, Category | Not available | OPTIONAL |
| IfcStairFlight | NumberOfRiser | Singular | NumberOfRisers (plural) |
| IfcBuildingElementProxy | CompositionType | Present | Replaced by PredefinedType |
| IfcRelSequence | LagTime | float | TimeLag (IfcLagTime object) |
| Most elements | PredefinedType | Not available | OPTIONAL |
| Date/Time fields | — | IfcDateAndTime objects | IfcDateTime strings |

---

## Schema Introspection

### Checking Entity Existence at Runtime

```python
import ifcopenshell

def entity_exists_in_schema(schema_name, entity_name):
    """Check if an entity exists in a specific IFC schema."""
    schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
    try:
        schema.declaration_by_name(entity_name)
        return True
    except RuntimeError:
        return False

# Usage
print(entity_exists_in_schema("IFC2X3", "IfcBuiltElement"))  # False
print(entity_exists_in_schema("IFC4X3", "IfcBuiltElement"))  # True
print(entity_exists_in_schema("IFC4X3", "IfcBuildingElement"))  # False
```

### Listing Entity Attributes Per Schema

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
entity = schema.declaration_by_name("IfcWall")
for attr in entity.all_attributes():
    print(f"{attr.name()}: {attr.type_of_attribute()}")
```

---

## Version-Safe Coding Patterns

### Pattern: Universal Building Element Query

```python
def get_building_elements(model):
    """Get all building/built elements regardless of schema version."""
    schema = model.schema
    if schema == "IFC4X3":
        return model.by_type("IfcBuiltElement")
    return model.by_type("IfcBuildingElement")
```

### Pattern: Safe Entity Creation with Schema Check

```python
def create_entity_safe(model, ifc_class, **kwargs):
    """Create entity with schema validation."""
    schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(model.schema)
    try:
        schema.declaration_by_name(ifc_class)
    except RuntimeError:
        raise ValueError(
            f"Entity '{ifc_class}' does not exist in {model.schema}. "
            f"Check the schema version compatibility."
        )
    return ifcopenshell.api.run("root.create_entity", model,
        ifc_class=ifc_class, **kwargs)
```

### Pattern: Schema-Agnostic Utilities

ALWAYS prefer `ifcopenshell.util` functions over manual traversal — they handle schema differences internally:

```python
import ifcopenshell.util.element

# These work identically across IFC2X3, IFC4, and IFC4X3:
container = ifcopenshell.util.element.get_container(element)
psets = ifcopenshell.util.element.get_psets(element)
material = ifcopenshell.util.element.get_material(element)
element_type = ifcopenshell.util.element.get_type(element)
```

---

## Reference Links

- [API Method Signatures](references/methods.md) — Schema introspection and migration methods
- [Working Code Examples](references/examples.md) — Version-safe coding patterns
- [Anti-Patterns](references/anti-patterns.md) — Schema-related mistakes and fixes
