# Validation Rules and Detection Patterns

Complete catalog of validation rules with code patterns to detect each issue.

---

## Rule V-001: Schema Check Before Entity Usage

**Severity**: BLOCKER
**Source**: ifcos-errors-schema

**Detection Pattern**: Code uses `model.by_type()` or `model.create_entity()` with schema-specific entity names without first reading `model.schema`.

```python
# DETECT: Any of these entity names used without schema guard
SCHEMA_SPECIFIC_ENTITIES = {
    "IFC4X3_only": [
        "IfcBuiltElement", "IfcBuiltElementType",
        "IfcFacility", "IfcFacilityPart",
        "IfcRoad", "IfcBridge", "IfcRailway",
        "IfcAlignment", "IfcBearing", "IfcCourse",
        "IfcKerb", "IfcPavement", "IfcRail",
    ],
    "IFC2X3_only": [
        "IfcDoorStyle", "IfcWindowStyle",
        "IfcBezierCurve", "IfcRationalBezierCurve",
    ],
    "removed_IFC4X3": [
        "IfcWallStandardCase", "IfcBeamStandardCase",
        "IfcColumnStandardCase", "IfcDoorStandardCase",
        "IfcMemberStandardCase", "IfcOpeningStandardCase",
        "IfcPlateStandardCase", "IfcSlabStandardCase",
        "IfcSlabElementedCase", "IfcWallElementedCase",
        "IfcWindowStandardCase", "IfcBuildingElement",
    ],
    "IFC4_plus_only": [
        "IfcDoorType", "IfcWindowType",
    ],
}
```

**Validation Logic**: Scan for string literals matching schema-specific entity names. If found, verify that `model.schema` is checked in the same function scope or a calling function. Report BLOCKER if no guard exists.

---

## Rule V-002: API Module Existence

**Severity**: BLOCKER
**Source**: ifcos-syntax-api

**Detection Pattern**: `ifcopenshell.api.run()` calls with non-existent module.function strings.

```python
# DETECT: First argument to api.run() must match one of 35 modules
VALID_API_MODULES = [
    "aggregate", "alignment", "attribute", "boundary", "classification",
    "cogo", "constraint", "context", "control", "cost", "document",
    "drawing", "feature", "geometry", "georeference", "grid", "group",
    "layer", "library", "material", "nest", "owner", "profile",
    "project", "pset", "pset_template", "resource", "root", "sequence",
    "spatial", "structural", "style", "system", "type", "unit", "void",
]

# Common hallucinated calls to reject:
HALLUCINATED_CALLS = [
    "element.create", "wall.create", "wall.add",
    "entity.create", "ifc.open", "file.create",
    "element.remove", "building.create",
]
```

**Validation Logic**: Extract the first string argument from every `api.run()` call. Split on `"."` and verify the module name exists in `VALID_API_MODULES`. Report BLOCKER for any match against `HALLUCINATED_CALLS` or unknown modules.

---

## Rule V-003: List Parameters for Relationship Functions

**Severity**: BLOCKER
**Source**: ifcos-syntax-api

**Detection Pattern**: Relationship API calls using singular parameter names (`product=`, `related_object=`) instead of plural list parameters (`products=`, `related_objects=`).

```python
# DETECT: These functions require list parameters since v0.8+
LIST_PARAM_FUNCTIONS = {
    "spatial.assign_container": "products",
    "spatial.unassign_container": "products",
    "aggregate.assign_object": "products",
    "aggregate.unassign_object": "products",
    "type.assign_type": "related_objects",
    "type.unassign_type": "related_objects",
    "material.assign_material": "products",
    "group.assign_group": "products",
    "classification.add_reference": "products",
    "nest.assign_object": "products",
}

# Singular parameter names that indicate the error:
WRONG_SINGULAR_PARAMS = ["product=", "related_object=", "element="]
```

**Validation Logic**: For each `api.run()` call matching a function in `LIST_PARAM_FUNCTIONS`, check that the corresponding parameter is passed as a list (enclosed in `[...]`). Report BLOCKER if a singular parameter name is used.

---

## Rule V-004: Direct Entity Removal

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns, ifcos-syntax-fileio

**Detection Pattern**: `model.remove(entity)` called on products (IfcWall, IfcSlab, etc.).

```python
# DETECT: model.remove() called on IfcProduct subclasses
# Pattern: model.remove(variable) where variable is an IFC product

# Safe alternative:
# ifcopenshell.api.run("root.remove_product", model, product=entity)
```

**Validation Logic**: Find all calls to `model.remove()`. If the removed entity is known to be an `IfcProduct` subclass (from context: `by_type("IfcWall")`, etc.), report BLOCKER. If entity type is unknown, report WARNING.

---

## Rule V-005: Entity Reference After Removal

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns

**Detection Pattern**: Variable referencing an IFC entity is used after `model.remove()` or `api.run("root.remove_product")` is called with that variable.

```python
# DETECT pattern:
# wall = model.by_type("IfcWall")[0]
# model.remove(wall)
# wall.Name  ← BLOCKER: wall is invalidated

# Also detect:
# wall = model.by_type("IfcWall")[0]
# ifcopenshell.api.run("root.remove_product", model, product=wall)
# wall.GlobalId  ← BLOCKER: wall is invalidated
```

**Validation Logic**: Track entity variable assignments and removal calls. If a variable is accessed after its entity is removed (in the same scope), report BLOCKER.

---

## Rule V-006: Iterate-and-Remove

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns

**Detection Pattern**: `for entity in model.by_type(...)` loop containing `model.remove()` or `api.run("root.remove_product")`.

```python
# DETECT: Removal inside iteration over the same collection
# for wall in model.by_type("IfcWall"):
#     model.remove(wall)  ← BLOCKER

# CORRECT pattern to verify:
# walls = list(model.by_type("IfcWall"))
# for wall in walls:
#     ifcopenshell.api.run("root.remove_product", model, product=wall)
```

**Validation Logic**: Find `for` loops iterating over `model.by_type()`. If any removal call exists inside the loop body and the iteration is NOT over a materialized list, report BLOCKER.

---

## Rule V-007: None Return Value Check

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns

**Detection Pattern**: Return value of utility functions accessed without None guard.

```python
# DETECT: Functions that return None and require guards
NONE_RETURNING_FUNCTIONS = [
    "ifcopenshell.util.element.get_container",
    "ifcopenshell.util.element.get_type",
    "ifcopenshell.util.element.get_material",
    "get_container",  # imported form
    "get_type",       # imported form
    "get_material",   # imported form
]

# Pattern: result = get_container(x); result.Name  ← BLOCKER if no None check
```

**Validation Logic**: For each call to a None-returning function, verify that the result is checked for None (via `if result is not None`, `if result:`, or similar) before attribute access. Report BLOCKER if attribute is accessed without guard.

---

## Rule V-008: GUID Format

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns

**Detection Pattern**: `uuid.uuid4()` or `str(uuid.uuid4())` used as GlobalId value.

```python
# DETECT: Standard UUID used for IFC GlobalId
# GlobalId=str(uuid.uuid4())  ← BLOCKER
# GlobalId=uuid.uuid4().hex   ← BLOCKER

# CORRECT:
# GlobalId=ifcopenshell.guid.new()
# Or use api.run("root.create_entity") which auto-generates
```

**Validation Logic**: Scan for `uuid.uuid4()` used in context of `GlobalId=` assignment or `create_entity()` calls. Report BLOCKER.

---

## Rule V-009: Unit Scale Application

**Severity**: WARNING
**Source**: ifcos-errors-patterns

**Detection Pattern**: Coordinates extracted from IFC entities without calling `ifcopenshell.util.unit.calculate_unit_scale()`.

```python
# DETECT: Coordinate extraction without unit conversion
# matrix = get_local_placement(wall.ObjectPlacement)
# x, y, z = matrix[:, 3][:3]  ← WARNING if no unit_scale applied

# CORRECT:
# unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# x, y, z = matrix[:, 3][:3] * unit_scale
```

**Validation Logic**: If code extracts placement matrices or coordinates from IFC entities, verify that `calculate_unit_scale()` is called in the same module. Report WARNING if missing.

---

## Rule V-010: Geometry Processing Without Error Handling

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns

**Detection Pattern**: `ifcopenshell.geom.create_shape()` called without `try/except RuntimeError`.

```python
# DETECT:
# shape = ifcopenshell.geom.create_shape(settings, element)  ← BLOCKER if no try/except

# CORRECT:
# try:
#     shape = ifcopenshell.geom.create_shape(settings, element)
# except RuntimeError:
#     shape = None
```

**Validation Logic**: Find all `create_shape()` calls. Verify each is inside a `try/except` block that catches `RuntimeError`. Report BLOCKER if unguarded.

---

## Rule V-011: Geometry Iterator for Batch Processing

**Severity**: WARNING
**Source**: ifcos-errors-performance

**Detection Pattern**: `create_shape()` called inside a loop iterating 10+ elements.

```python
# DETECT:
# for wall in model.by_type("IfcWall"):
#     shape = ifcopenshell.geom.create_shape(settings, wall)  ← WARNING

# CORRECT:
# iterator = ifcopenshell.geom.iterator(settings, model, cpu_count(), include=walls)
```

**Validation Logic**: Find `create_shape()` inside `for` loops that iterate over `model.by_type()` results. Report WARNING and suggest `geom.iterator`.

---

## Rule V-012: Duplicate Property Set Creation

**Severity**: WARNING
**Source**: ifcos-errors-patterns

**Detection Pattern**: `api.run("pset.add_pset", ...)` called without first checking `get_psets()` for existing pset with the same name.

```python
# DETECT:
# pset = ifcopenshell.api.run("pset.add_pset", model,
#     product=wall, name="Pset_WallCommon")  ← WARNING if no prior check

# CORRECT:
# existing = get_psets(wall)
# if "Pset_WallCommon" not in existing:
#     pset = ifcopenshell.api.run("pset.add_pset", ...)
```

**Validation Logic**: For each `pset.add_pset` call, check if a `get_psets()` call precedes it with a membership test for the pset name. Report WARNING if missing.

---

## Rule V-013: OwnerHistory in IFC2X3

**Severity**: BLOCKER
**Source**: ifcos-errors-schema

**Detection Pattern**: Entities created in IFC2X3 files without OwnerHistory, using low-level `model.create_entity()`.

```python
# DETECT:
# model = ifcopenshell.file(schema="IFC2X3")
# wall = model.create_entity("IfcWall", GlobalId=..., OwnerHistory=None, Name="...")
# ← BLOCKER: OwnerHistory is REQUIRED in IFC2X3

# SAFE: Using api.run() handles this automatically
```

**Validation Logic**: If file schema is IFC2X3 and `model.create_entity()` is used, verify OwnerHistory is provided and non-None. Report BLOCKER if missing. No issue if `api.run("root.create_entity")` is used.

---

## Rule V-014: Transaction Safety

**Severity**: BLOCKER
**Source**: ifcos-syntax-fileio

**Detection Pattern**: `begin_transaction()` without matching `end_transaction()` or `discard_transaction()`.

```python
# DETECT: Unbalanced transaction calls
# model.begin_transaction()
# ... code that may raise exceptions ...
# model.end_transaction()  ← Must be guaranteed to execute

# Also detect:
# model.undo() inside begin_transaction()/end_transaction() block ← BLOCKER
```

**Validation Logic**: Track `begin_transaction()` and `end_transaction()`/`discard_transaction()` calls. Verify they are balanced. Verify no `undo()`/`redo()` calls between them. Report BLOCKER for imbalance.

---

## Rule V-015: Per-Element API Calls Instead of Batch

**Severity**: WARNING
**Source**: ifcos-errors-performance

**Detection Pattern**: `spatial.assign_container` or `type.assign_type` called inside a loop with a single-element list.

```python
# DETECT:
# for wall in walls:
#     ifcopenshell.api.run("spatial.assign_container", model,
#         products=[wall], relating_structure=storey)
# ← WARNING: Creates N relationship entities instead of 1

# CORRECT:
# ifcopenshell.api.run("spatial.assign_container", model,
#     products=list(walls), relating_structure=storey)
```

**Validation Logic**: Find `spatial.assign_container`, `aggregate.assign_object`, `type.assign_type`, or `material.assign_material` inside loops. If the list parameter contains a single variable from the loop iterator, report WARNING.

---

## Rule V-016: Full Model Iteration Instead of by_type

**Severity**: WARNING
**Source**: ifcos-errors-performance

**Detection Pattern**: `for entity in model` or list comprehension filtering `model` directly.

```python
# DETECT:
# walls = [e for e in model if e.is_a("IfcWall")]  ← WARNING: O(n) scan
# for entity in model:
#     if entity.is_a("IfcWall"):  ← WARNING: O(n) scan

# CORRECT:
# walls = model.by_type("IfcWall")  ← O(1) indexed lookup
```

**Validation Logic**: Find iterations over `model` (the file object) where `.is_a()` filtering is applied inside the loop. Report WARNING and suggest `model.by_type()`.

---

## Rule V-017: get_info(recursive=True) on Large Files

**Severity**: WARNING
**Source**: ifcos-errors-performance

**Detection Pattern**: `get_info(recursive=True)` called in loops or on files known to be large.

```python
# DETECT:
# for element in model.by_type("IfcWall"):
#     info = element.get_info(recursive=True)  ← WARNING: Materializes entity graph

# CORRECT:
# for element in model.by_type("IfcWall"):
#     info = element.get_info()  # Non-recursive, returns immediate attributes only
```

**Validation Logic**: Find `get_info(recursive=True)` calls. Report WARNING, especially if inside loops.

---

## Rule V-018: project.create_file Parameter Name

**Severity**: BLOCKER
**Source**: ifcos-syntax-fileio, ifcos-syntax-api

**Detection Pattern**: `api.run("project.create_file", schema=...)` using wrong parameter name.

```python
# DETECT:
# ifcopenshell.api.run("project.create_file", schema="IFC4")  ← BLOCKER
# Correct parameter is version=, not schema=

# CORRECT:
# ifcopenshell.api.run("project.create_file", version="IFC4")
```

**Validation Logic**: Find `project.create_file` calls. Verify parameter is `version=`, not `schema=`. Report BLOCKER if `schema=` is used.

---

## Rule V-019: Attribute Case Sensitivity

**Severity**: BLOCKER
**Source**: ifcos-errors-patterns

**Detection Pattern**: Entity attributes accessed with wrong case (snake_case or lowercase instead of PascalCase).

```python
# DETECT: Wrong attribute case
WRONG_CASE_PATTERNS = {
    "global_id": "GlobalId",
    "globalid": "GlobalId",
    "name": "Name",  # when used as wall.name
    "object_type": "ObjectType",
    "object_placement": "ObjectPlacement",
    "representation": "Representation",  # lowercase
    "predefined_type": "PredefinedType",
    "owner_history": "OwnerHistory",
}
```

**Validation Logic**: Scan for attribute access on IFC entity variables using lowercase or snake_case. Verify against PascalCase IFC attribute names. Report BLOCKER for definite mismatches.

---

## Rule V-020: Missing Geometry Context Setup

**Severity**: WARNING
**Source**: ifcos-syntax-api

**Detection Pattern**: Code creates geometry representations without first setting up representation contexts (Model → Body → MODEL_VIEW).

```python
# DETECT: geometry.add_wall_representation (or similar) called without context.add_context
# representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
#     context=body, ...)  ← WARNING if context.add_context is never called

# CORRECT sequence:
# model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
# body = ifcopenshell.api.run("context.add_context", model,
#     context_type="Model", context_identifier="Body",
#     target_view="MODEL_VIEW", parent=model3d)
```

**Validation Logic**: If any `geometry.add_*_representation` call exists, verify that `context.add_context` was called earlier in the code. Report WARNING if missing.
