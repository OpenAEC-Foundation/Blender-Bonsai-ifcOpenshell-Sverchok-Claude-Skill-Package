---
name: ifcos-agents-code-validator
description: "Provides a systematic validation checklist for reviewing IfcOpenShell Python code, checking for schema compatibility errors, incorrect API usage, entity reference invalidation, performance anti-patterns, missing error handling, and IFC standard compliance. Activates when Claude reviews, validates, audits, or checks IfcOpenShell code quality."
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Code Validator Agent

## Quick Reference

### When to Activate This Validator

Activate this validation checklist when:
- Reviewing IfcOpenShell Python code for correctness
- Auditing IFC file manipulation scripts before production use
- Checking code that creates, modifies, or reads IFC files
- Validating code that processes IFC geometry
- Reviewing code that handles multiple IFC schema versions
- Assessing performance of scripts processing large IFC files (100MB+)

### Severity Levels

| Severity | Meaning | Action |
|----------|---------|--------|
| **BLOCKER** | Code will crash, corrupt data, or produce invalid IFC files | MUST fix before code is accepted |
| **WARNING** | Code has potential bugs, performance issues, or fragile patterns | SHOULD fix; document reason if deferred |
| **INFO** | Code works but does not follow best practices | MAY fix; note for future improvement |

---

## Validation Checklist

Execute these checks in order. Each check references the dependency skill that defines the rule.

### Step 1: Schema Awareness (BLOCKER)

**Source: ifcos-errors-schema**

```
Code uses IFC entities?
├── Does code check model.schema before using schema-specific entities?
│   ├── NO → BLOCKER: Add schema check
│   └── YES → Pass
│
├── Does code use any *StandardCase entity (IfcWallStandardCase, etc.)?
│   ├── YES + no IFC4X3 guard → BLOCKER: StandardCase removed in IFC4X3
│   └── NO or guarded → Pass
│
├── Does code use IfcBuiltElement or IfcBuildingElement?
│   ├── YES + no schema branch → BLOCKER: Name differs by schema
│   └── Handled → Pass
│
├── Does code use IfcDoorStyle/IfcWindowStyle?
│   ├── YES + targets IFC4+ → BLOCKER: Use IfcDoorType/IfcWindowType
│   └── Handled → Pass
│
├── Does code use IFC4X3-only entities (IfcRoad, IfcBridge, IfcAlignment)?
│   ├── YES + schema != "IFC4X3" → BLOCKER: Entity does not exist
│   └── Correct schema → Pass
│
└── Does code access PredefinedType on IFC2X3 entities?
    ├── YES + no hasattr guard → BLOCKER: Most IFC2X3 entities lack PredefinedType
    └── Guarded → Pass
```

### Step 2: API Usage Correctness (BLOCKER)

**Source: ifcos-syntax-api, ifcos-errors-patterns**

```
Code creates or modifies IFC entities?
├── Uses model.create_entity() for production code?
│   └── BLOCKER: Use ifcopenshell.api.run("root.create_entity", ...) instead
│
├── Uses ifcopenshell.api.run() or direct module calls?
│   ├── Invents non-existent API modules/functions?
│   │   └── BLOCKER: Verify against the 35 API modules table
│   ├── Uses positional arguments after model?
│   │   └── BLOCKER: ALWAYS use keyword arguments
│   └── Passes single element where list is required (v0.8+)?
│       └── BLOCKER: products=, related_objects= require lists
│
├── Modifies entity attributes directly (wall.Name = "X")?
│   ├── For Name/Description → WARNING: Use api.run("attribute.edit_attributes", ...)
│   └── For relationships (ContainedInStructure, IsTypedBy) → BLOCKER: Use API
│
├── Creates relationships manually (model.create_entity("IfcRelContained..."))?
│   └── BLOCKER: Use spatial.assign_container, aggregate.assign_object, etc.
│
└── Creates IFC file with ifcopenshell.file() for production use?
    └── WARNING: Use ifcopenshell.api.run("project.create_file", ...) instead
```

### Step 3: File I/O Correctness (BLOCKER/WARNING)

**Source: ifcos-syntax-fileio**

```
Code opens or creates IFC files?
├── Uses ifcopenshell.file() without explicit schema= parameter?
│   └── WARNING: ALWAYS specify schema explicitly
│
├── Uses project.create_file() with schema= instead of version=?
│   └── BLOCKER: Parameter name is version=, not schema=
│
├── Creates file without IfcProject + units + context + spatial hierarchy?
│   └── WARNING: Incomplete IFC files cause downstream failures
│
├── Calls model.remove() directly on products?
│   └── BLOCKER: Use api.run("root.remove_product", ...) for safe removal
│
├── Uses transactions?
│   ├── Missing end_transaction() or discard_transaction()?
│   │   └── BLOCKER: Open transactions cause undefined behavior
│   └── Calls undo()/redo() inside active transaction?
│       └── BLOCKER: NEVER call undo/redo during active transaction
│
└── Writes to disk?
    └── Pass (model.write() is safe)
```

### Step 4: Entity Reference Safety (BLOCKER)

**Source: ifcos-errors-patterns**

```
Code removes entities or uses undo?
├── Accesses entity attributes after model.remove()?
│   └── BLOCKER: Entity wrapper is invalid after removal
│
├── Iterates and removes simultaneously?
│   └── BLOCKER: Collect to list first, then remove
│
├── Caches entity references across file modifications?
│   └── BLOCKER: References invalidate after remove/undo
│
└── Extracts data before removal?
    └── Pass
```

### Step 5: None Return Value Handling (BLOCKER)

**Source: ifcos-errors-patterns**

```
Code calls utility functions that return None?
├── Uses get_container(element).Name without None check?
│   └── BLOCKER: get_container returns None if no containment
│
├── Uses get_type(element).Name without None check?
│   └── BLOCKER: get_type returns None if no type assigned
│
├── Uses get_material(element) without None check?
│   └── BLOCKER: get_material returns None if no material
│
├── Uses geom.create_shape() without try/except RuntimeError?
│   └── BLOCKER: Geometry processing fails for many elements
│
└── Uses model.by_guid() without None check?
    └── WARNING: Returns None if GUID not found
```

### Step 6: GUID Handling (BLOCKER)

**Source: ifcos-errors-patterns**

```
Code creates entities with GlobalId?
├── Uses uuid.uuid4() or str(uuid) for GlobalId?
│   └── BLOCKER: IFC requires 22-char base64 GUID; use ifcopenshell.guid.new()
│
├── Reuses same GUID for multiple entities?
│   └── BLOCKER: Every entity MUST have a unique GlobalId
│
└── Uses ifcopenshell.api.run("root.create_entity") (auto-generates)?
    └── Pass
```

### Step 7: Unit Handling (WARNING)

**Source: ifcos-errors-patterns**

```
Code reads coordinates or dimensions from IFC files?
├── Assumes coordinates are in meters?
│   └── WARNING: ALWAYS call ifcopenshell.util.unit.calculate_unit_scale(model)
│
├── Applies unit_scale to all extracted coordinates?
│   └── Pass
│
└── Creates geometry with hardcoded values?
    └── INFO: Document assumed unit system in comments
```

### Step 8: Property Set Handling (WARNING)

**Source: ifcos-errors-patterns**

```
Code creates property sets?
├── Creates pset without checking for existing one?
│   └── WARNING: Produces duplicate property sets; check get_psets() first
│
├── Creates pset manually without IfcRelDefinesByProperties?
│   └── BLOCKER: Orphaned pset; use api.run("pset.add_pset", ...)
│
└── Uses api.run("pset.add_pset") + api.run("pset.edit_pset")?
    └── Pass
```

### Step 9: Performance Patterns (WARNING)

**Source: ifcos-errors-performance**

```
Code processes geometry for multiple elements?
├── Calls create_shape() in a loop for 10+ elements?
│   └── WARNING: Use ifcopenshell.geom.iterator instead (5-10x faster)
│
├── Uses geom.iterator without multiprocessing.cpu_count()?
│   └── WARNING: Pass cpu_count() for optimal parallelism
│
├── Stores all geometry shapes in memory simultaneously?
│   └── WARNING: Process each shape and discard immediately
│
├── Uses get_info(recursive=True) on large files?
│   └── WARNING: Materializes entire entity graph; extract specific attributes
│
├── Opens same large file multiple times?
│   └── WARNING: Open once, pass model reference
│
├── Iterates all entities with `for e in model` instead of by_type()?
│   └── WARNING: by_type() is O(1), full iteration is O(n)
│
├── Calls spatial.assign_container per-element instead of batching?
│   └── WARNING: Pass all products in one call
│
└── Processes 200MB+ file without gc.collect() between batches?
    └── INFO: Add gc.collect() between processing phases
```

### Step 10: IFC2X3 Compatibility (BLOCKER)

**Source: ifcos-errors-schema**

```
Code targets or handles IFC2X3 files?
├── Creates entities without OwnerHistory?
│   └── BLOCKER in IFC2X3: OwnerHistory is REQUIRED on all IfcRoot subclasses
│
├── Uses ifcopenshell.api.run() for entity creation?
│   └── Pass (API handles OwnerHistory automatically)
│
├── Queries IfcContext (does not exist in IFC2X3)?
│   └── BLOCKER: Use IfcProject directly
│
└── Uses IfcDoorType/IfcWindowType on IFC2X3?
    └── BLOCKER: Use IfcDoorStyle/IfcWindowStyle for IFC2X3
```

---

## Decision Trees

### Decision Tree: Severity Classification

```
Is the issue...
├── A crash, data corruption, or invalid IFC output?
│   └── BLOCKER
│       Examples:
│       - Schema mismatch (entity not found)
│       - Invalid entity reference after removal
│       - Missing OwnerHistory in IFC2X3
│       - Dangling references from model.remove()
│       - Wrong GUID format
│       - Open transaction never closed
│
├── A potential bug, performance problem, or fragile pattern?
│   └── WARNING
│       Examples:
│       - Missing None check on get_container()
│       - create_shape() loop instead of iterator
│       - No unit scale applied
│       - Duplicate property sets
│       - Per-element API calls instead of batch
│
└── A best-practice deviation with no functional impact?
    └── INFO
        Examples:
        - Implicit schema parameter
        - Missing gc.collect() call
        - Hardcoded unit assumptions (documented)
```

### Decision Tree: Auto-Fix Applicability

```
Can this issue be auto-fixed?
├── Schema entity name substitution?
│   └── YES: Replace with version-aware branching
│
├── model.remove() → api.run("root.remove_product")?
│   └── YES: Direct substitution
│
├── model.create_entity() → api.run("root.create_entity")?
│   └── YES: Add ifc_class= and name= parameters
│
├── Missing None check on utility return value?
│   └── YES: Wrap in `if result is not None:` guard
│
├── create_shape() loop → iterator?
│   └── PARTIAL: Requires restructuring; provide template
│
├── Single element → list parameter?
│   └── YES: Wrap in list brackets [element]
│
└── Missing schema check?
    └── PARTIAL: Insert model.schema check; pattern depends on context
```

---

## Auto-Fix Patterns

### Fix 1: Single Element to List Parameter (v0.8+)

```python
# BEFORE (BLOCKER):
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)

# AFTER:
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

### Fix 2: Direct Removal to Safe API Removal

```python
# BEFORE (BLOCKER):
model.remove(wall)

# AFTER:
ifcopenshell.api.run("root.remove_product", model, product=wall)
```

### Fix 3: Low-Level Entity Creation to API

```python
# BEFORE (BLOCKER):
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Wall 001")

# AFTER:
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
```

### Fix 4: Missing None Guard

```python
# BEFORE (BLOCKER):
container = ifcopenshell.util.element.get_container(wall)
print(container.Name)

# AFTER:
container = ifcopenshell.util.element.get_container(wall)
if container is not None:
    print(container.Name)
```

### Fix 5: UUID to IFC GUID

```python
# BEFORE (BLOCKER):
import uuid
wall = model.create_entity("IfcWall", GlobalId=str(uuid.uuid4()))

# AFTER:
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
# Or if manual creation is required:
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Wall")
```

### Fix 6: Iterate-and-Remove to Collect-Then-Remove

```python
# BEFORE (BLOCKER):
for wall in model.by_type("IfcWall"):
    model.remove(wall)

# AFTER:
walls_to_remove = list(model.by_type("IfcWall"))
for wall in walls_to_remove:
    ifcopenshell.api.run("root.remove_product", model, product=wall)
```

### Fix 7: create_shape Loop to Iterator

```python
# BEFORE (WARNING):
# IfcOpenShell — all schema versions
for wall in model.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)
    process(shape)

# AFTER:
# IfcOpenShell — all schema versions
import multiprocessing
walls = model.by_type("IfcWall")
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(), include=walls)
if iterator.initialize():
    while True:
        shape = iterator.get()
        process(shape)
        if not iterator.next():
            break
```

### Fix 8: Schema-Aware Entity Selection

```python
# BEFORE (BLOCKER):
# IfcOpenShell — assumes single schema
elements = model.by_type("IfcBuildingElement")

# AFTER:
# IfcOpenShell — all schema versions
schema = model.schema
if schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")
```

---

## Validation Report Format

After running all checks, produce a report in this format:

```
## IfcOpenShell Code Validation Report

### Summary
- BLOCKERS: [count]
- WARNINGS: [count]
- INFO: [count]

### BLOCKER Issues
1. [Line X]: [Description] — [Fix reference]

### WARNING Issues
1. [Line X]: [Description] — [Fix reference]

### INFO Issues
1. [Line X]: [Description] — [Fix reference]

### Verdict
[PASS / FAIL (if any BLOCKERS exist)]
```

---

## Reference Links

- [Validation Rules and Detection Patterns](references/methods.md)
- [Before/After Validation Fix Examples](references/examples.md)
- [Anti-Patterns Catalog](references/anti-patterns.md)

### Dependency Skills (define the rules this validator checks)
- **ifcos-syntax-fileio** — File I/O patterns and transaction management
- **ifcos-syntax-api** — API module system, invocation patterns, parameter conventions
- **ifcos-errors-patterns** — Error categories, debugging strategies, None handling
- **ifcos-errors-schema** — Schema version differences, entity availability, migration
- **ifcos-errors-performance** — Geometry iterator, caching, memory management

### External References
- IfcOpenShell Documentation: https://docs.ifcopenshell.org/
- IFC Schema Specifications: https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/
