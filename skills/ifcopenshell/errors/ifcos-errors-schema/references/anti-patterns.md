# Schema Error Anti-Patterns

Common mistakes when working with IFC schema versions in IfcOpenShell. Each anti-pattern explains WHAT is wrong and WHY it causes problems.

---

## AP-1: Hardcoding Entity Names Without Schema Check

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
model = ifcopenshell.open("model.ifc")
elements = model.by_type("IfcBuildingElement")  # RuntimeError if IFC4X3!
```

### Correct

```python
# IfcOpenShell — all schema versions
model = ifcopenshell.open("model.ifc")
if model.schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")
```

### WHY

`IfcBuildingElement` was renamed to `IfcBuiltElement` in IFC4X3. Hardcoding the entity name without checking the schema causes a `RuntimeError` on IFC4X3 files. ALWAYS check `model.schema` before using entities that changed between versions.

---

## AP-2: Assuming StandardCase Entities Exist

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
walls = model.by_type("IfcWallStandardCase")  # Fails on IFC4X3
beams = model.by_type("IfcBeamStandardCase")  # Fails on IFC4X3
```

### Correct

```python
# IfcOpenShell — all schema versions
schema = model.schema
if schema == "IFC4X3":
    walls = model.by_type("IfcWall")  # StandardCase merged into parent
    beams = model.by_type("IfcBeam")
else:
    walls = model.by_type("IfcWall")  # Includes StandardCase via subtypes
    beams = model.by_type("IfcBeam")  # Includes StandardCase via subtypes
```

### WHY

IFC4 introduced 11 `*StandardCase` and `*ElementedCase` subtypes. IFC4X3 **removed all of them**, merging the distinction back into the parent entities. Querying `IfcWallStandardCase` on an IFC4X3 file raises `RuntimeError`. Since `by_type()` with `include_subtypes=True` (default) already includes subtypes, using the parent class works correctly in all versions.

---

## AP-3: Using IfcDoorStyle / IfcWindowStyle in IFC4+ Code

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
door_types = model.by_type("IfcDoorStyle")  # Fails on IFC4X3 (removed)
```

### Correct

```python
# IfcOpenShell — all schema versions
if model.schema == "IFC2X3":
    door_types = model.by_type("IfcDoorStyle")
else:
    door_types = model.by_type("IfcDoorType")
```

### WHY

`IfcDoorStyle` and `IfcWindowStyle` are IFC2X3 entities. IFC4 introduced `IfcDoorType` and `IfcWindowType` as replacements (both styles were deprecated). In IFC4X3, `IfcDoorStyle` and `IfcWindowStyle` may be fully removed. ALWAYS use the version-appropriate type entity.

---

## AP-4: Omitting OwnerHistory in IFC2X3

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
model = ifcopenshell.file(schema="IFC2X3")
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    None,  # OwnerHistory = None is INVALID in IFC2X3
    "MyWall"
)
```

### Correct

```python
# IfcOpenShell — IFC2X3
# Option 1: Use api.run() (handles OwnerHistory automatically)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="MyWall")

# Option 2: Provide OwnerHistory explicitly
owner_history = ifcopenshell.api.run("owner.create_owner_history", model)
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    owner_history,
    "MyWall"
)
```

### WHY

In IFC2X3, `OwnerHistory` is a REQUIRED attribute on all `IfcRoot` subclasses. Setting it to `None` creates an invalid IFC file that will fail validation. In IFC4+, `OwnerHistory` became OPTIONAL, so `None` is acceptable. Using `ifcopenshell.api.run("root.create_entity", ...)` handles this difference automatically.

---

## AP-5: Accessing PredefinedType on IFC2X3 Entities

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
for beam in model.by_type("IfcBeam"):
    print(beam.PredefinedType)  # AttributeError on IFC2X3 IfcBeam!
```

### Correct

```python
# IfcOpenShell — all schema versions
for beam in model.by_type("IfcBeam"):
    if hasattr(beam, "PredefinedType"):
        print(f"{beam.Name}: {beam.PredefinedType}")
    else:
        print(f"{beam.Name}: No PredefinedType (IFC2X3)")
```

### WHY

Many IFC entities gained `PredefinedType` in IFC4 that did not have it in IFC2X3. Accessing `PredefinedType` on an IFC2X3 entity raises `AttributeError`. ALWAYS check with `hasattr()` or verify `model.schema` before accessing version-specific attributes.

Affected entities include: IfcBeam, IfcColumn, IfcDoor, IfcWall, IfcWindow, IfcSlab, IfcRoof, IfcStair, IfcFooting, IfcMember, IfcPlate, IfcRailing, IfcRamp, IfcCovering, and others.

---

## AP-6: Creating Infrastructure Entities on Wrong Schema

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
model = ifcopenshell.file(schema="IFC4")
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway")  # RuntimeError: IfcRoad not in IFC4!
```

### Correct

```python
# IfcOpenShell — IFC4X3
model = ifcopenshell.file(schema="IFC4X3")
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway")  # Works
```

### WHY

Infrastructure entities (IfcRoad, IfcBridge, IfcRailway, IfcAlignment, IfcFacility, etc.) are ONLY available in IFC4X3. Attempting to create them in IFC2X3 or IFC4 files raises `RuntimeError`. ALWAYS use `schema="IFC4X3"` for infrastructure projects.

---

## AP-7: Transferring Entities Between Different Schema Files

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
source = ifcopenshell.open("building_2x3.ifc")  # IFC2X3
target = ifcopenshell.file(schema="IFC4X3")

wall = source.by_type("IfcWall")[0]
target.add(wall)  # Schema mismatch — undefined behavior!
```

### Correct

```python
# IfcOpenShell — same schema transfer
source = ifcopenshell.open("building_2x3.ifc")
target = ifcopenshell.file(schema=source.schema)  # Match schema

wall = source.by_type("IfcWall")[0]
target.add(wall)  # Safe — same schema

# OR: Migrate first, then transfer
import ifcpatch
output = ifcpatch.execute({
    "input": "building_2x3.ifc",
    "file": source,
    "recipe": "Migrate",
    "arguments": ["IFC4X3"]
})
ifcpatch.write(output, "building_4x3.ifc")
```

### WHY

Transferring entities between files with different schemas leads to undefined behavior. Entity attributes, types, and relationships may not be compatible. ALWAYS ensure source and target files use the same schema, or migrate the source file first using ifcpatch.

---

## AP-8: Not Validating After Schema Migration

### Wrong

```python
# IfcOpenShell + ifcpatch — BAD PRACTICE
import ifcpatch

model = ifcopenshell.open("input_2x3.ifc")
output = ifcpatch.execute({
    "input": "input_2x3.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4"]
})
ifcpatch.write(output, "output_ifc4.ifc")
# Done! Ship it! (NO — migration can produce invalid files)
```

### Correct

```python
# IfcOpenShell + ifcpatch — CORRECT
import ifcpatch
import ifcopenshell.validate

model = ifcopenshell.open("input_2x3.ifc")
source_count = len(model)

output = ifcpatch.execute({
    "input": "input_2x3.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4"]
})
ifcpatch.write(output, "output_ifc4.ifc")

# Validate the result
migrated = ifcopenshell.open("output_ifc4.ifc")
print(f"Schema: {migrated.schema}")
print(f"Entities: {source_count} -> {len(migrated)}")

logger = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(migrated, logger)

errors = [s for s in logger.statements if s["severity"] == "Error"]
if errors:
    print(f"VALIDATION FAILED: {len(errors)} errors")
    for e in errors[:5]:
        print(f"  {e['message']}")
else:
    print("Validation passed")
```

### WHY

The ifcpatch `Migrate` recipe is **experimental**. Migration can produce invalid files due to unmapped entities, changed attribute types, or missing required data. ALWAYS validate the output and compare entity counts. Schema migration is not lossless.

---

## AP-9: Using IfcContext in IFC2X3 Code

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
model = ifcopenshell.open("model.ifc")  # Could be IFC2X3
contexts = model.by_type("IfcContext")  # RuntimeError in IFC2X3!
```

### Correct

```python
# IfcOpenShell — all schema versions
model = ifcopenshell.open("model.ifc")
# IfcProject works in ALL versions
projects = model.by_type("IfcProject")
```

### WHY

`IfcContext` is an abstract supertype introduced in IFC4 as the parent of `IfcProject`. It does not exist in IFC2X3, where `IfcProject` inherits from `IfcObject` instead. Querying `IfcContext` on an IFC2X3 file raises `RuntimeError`. Use `IfcProject` directly — it works across all versions.

---

## AP-10: Assuming IfcSpatialElement Exists in IFC2X3

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
spatial = model.by_type("IfcSpatialElement")  # RuntimeError in IFC2X3!
```

### Correct

```python
# IfcOpenShell — all schema versions
schema = model.schema
if schema == "IFC2X3":
    spatial = model.by_type("IfcSpatialStructureElement")
else:  # IFC4, IFC4X3
    spatial = model.by_type("IfcSpatialElement")
```

### WHY

`IfcSpatialElement` was introduced in IFC4 as a new abstract supertype above `IfcSpatialStructureElement`. In IFC2X3, `IfcSpatialStructureElement` is the top-level spatial abstract class. Always check the schema version before querying abstract types that differ between versions.

---

## AP-11: Ignoring IfcStairFlight Attribute Rename

### Wrong

```python
# IfcOpenShell — BAD PRACTICE (works only on IFC2X3)
for stair_flight in model.by_type("IfcStairFlight"):
    risers = stair_flight.NumberOfRiser  # AttributeError in IFC4+!
```

### Correct

```python
# IfcOpenShell — all schema versions
for stair_flight in model.by_type("IfcStairFlight"):
    if model.schema == "IFC2X3":
        risers = stair_flight.NumberOfRiser  # Singular in IFC2X3
    else:
        risers = stair_flight.NumberOfRisers  # Plural in IFC4+
```

### WHY

`IfcStairFlight.NumberOfRiser` (singular) was renamed to `NumberOfRisers` (plural) in IFC4 to fix a typo. Accessing the old attribute name on IFC4+ files raises `AttributeError`.

---

## AP-12: Using create_entity() for IfcRoot Subtypes Instead of api.run()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),
    Name="MyWall")
# Missing: OwnerHistory (required in IFC2X3), no relationships created
```

### Correct

```python
# IfcOpenShell — all schema versions
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="MyWall")
# api.run handles: GlobalId, OwnerHistory (schema-aware), and proper setup
```

### WHY

`model.create_entity()` is a low-level method that does NOT handle schema-specific requirements. It will not create `OwnerHistory` (required in IFC2X3), does not handle attribute differences between versions, and does not create required relationships. ALWAYS use `ifcopenshell.api.run("root.create_entity", ...)` which is schema-aware and handles these differences automatically.
