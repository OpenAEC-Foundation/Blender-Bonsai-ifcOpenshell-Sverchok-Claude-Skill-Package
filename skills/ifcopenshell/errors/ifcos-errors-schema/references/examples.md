# Schema Error Examples

Working code examples for handling IFC schema differences, migration, and version-safe programming with IfcOpenShell.

---

## Example 1: Version-Safe Building Element Query

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema

# IfcBuildingElement renamed to IfcBuiltElement in IFC4X3
if schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:  # IFC2X3 or IFC4
    elements = model.by_type("IfcBuildingElement")

print(f"Schema: {schema}, Building elements: {len(elements)}")
for elem in elements[:5]:
    print(f"  {elem.is_a()}: {elem.Name}")
```

---

## Example 2: Version-Safe Wall Query (StandardCase Handling)

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema

# IfcWallStandardCase exists in IFC2X3 and IFC4, removed in IFC4X3
if schema in ("IFC2X3", "IFC4"):
    # Query both IfcWall and IfcWallStandardCase
    walls = list(model.by_type("IfcWall"))
    # Note: IfcWallStandardCase is a subtype of IfcWall,
    # so by_type("IfcWall") already includes StandardCase walls
    # when include_subtypes=True (default)
else:  # IFC4X3
    walls = list(model.by_type("IfcWall"))

print(f"Total walls: {len(walls)}")
```

---

## Example 3: Version-Safe Door/Window Type Query

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema

# Door types: IfcDoorStyle (IFC2X3) vs IfcDoorType (IFC4+)
if schema == "IFC2X3":
    door_types = model.by_type("IfcDoorStyle")
    window_types = model.by_type("IfcWindowStyle")
else:  # IFC4 or IFC4X3
    door_types = model.by_type("IfcDoorType")
    window_types = model.by_type("IfcWindowType")

print(f"Door types: {len(door_types)}")
print(f"Window types: {len(window_types)}")
```

---

## Example 4: Safe PredefinedType Access

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")

for wall in model.by_type("IfcWall"):
    # PredefinedType may not exist in IFC2X3
    if hasattr(wall, "PredefinedType") and wall.PredefinedType:
        print(f"{wall.Name}: PredefinedType={wall.PredefinedType}")
    else:
        print(f"{wall.Name}: No PredefinedType")
```

---

## Example 5: Schema Introspection — Check Entity Existence

```python
# IfcOpenShell — all schema versions
import ifcopenshell

def entity_exists(schema_name, entity_name):
    """Check if an entity exists in a given IFC schema."""
    schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
    try:
        schema.declaration_by_name(entity_name)
        return True
    except RuntimeError:
        return False

# Check entity availability across versions
entities_to_check = [
    "IfcBuildingElement", "IfcBuiltElement",
    "IfcWallStandardCase", "IfcDoorStyle", "IfcDoorType",
    "IfcFacility", "IfcRoad", "IfcAlignment",
    "IfcSpatialElement", "IfcContext"
]

for entity in entities_to_check:
    ifc2x3 = entity_exists("IFC2X3", entity)
    ifc4 = entity_exists("IFC4", entity)
    ifc4x3 = entity_exists("IFC4X3", entity)
    print(f"{entity:30s}  2X3={ifc2x3}  4={ifc4}  4X3={ifc4x3}")
```

Expected output:
```
IfcBuildingElement              2X3=True   4=True   4X3=False
IfcBuiltElement                 2X3=False  4=False  4X3=True
IfcWallStandardCase             2X3=True   4=True   4X3=False
IfcDoorStyle                    2X3=True   4=True   4X3=False
IfcDoorType                     2X3=False  4=True   4X3=True
IfcFacility                     2X3=False  4=False  4X3=True
IfcRoad                         2X3=False  4=False  4X3=True
IfcAlignment                    2X3=False  4=False  4X3=True
IfcSpatialElement               2X3=False  4=True   4X3=True
IfcContext                      2X3=False  4=True   4X3=True
```

---

## Example 6: Migrate IFC2X3 to IFC4

```python
# IfcOpenShell + ifcpatch
import ifcopenshell
import ifcpatch

# Step 1: Open the source file
model = ifcopenshell.open("building_2x3.ifc")
print(f"Source schema: {model.schema}")  # "IFC2X3"
print(f"Source entities: {len(model)}")

# Step 2: Execute migration
output = ifcpatch.execute({
    "input": "building_2x3.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4"]
})

# Step 3: Write output
ifcpatch.write(output, "building_ifc4.ifc")

# Step 4: Validate result
migrated = ifcopenshell.open("building_ifc4.ifc")
print(f"Output schema: {migrated.schema}")  # "IFC4"
print(f"Output entities: {len(migrated)}")

# Step 5: Compare entity counts
for entity_type in ["IfcWall", "IfcDoor", "IfcWindow", "IfcSlab"]:
    try:
        original_count = len(model.by_type(entity_type))
        migrated_count = len(migrated.by_type(entity_type))
        match = "OK" if original_count == migrated_count else "MISMATCH"
        print(f"  {entity_type}: {original_count} -> {migrated_count} [{match}]")
    except RuntimeError:
        print(f"  {entity_type}: entity name changed between versions")
```

---

## Example 7: Migrate IFC4 to IFC4X3

```python
# IfcOpenShell + ifcpatch
import ifcopenshell
import ifcpatch

model = ifcopenshell.open("building_ifc4.ifc")
print(f"Source schema: {model.schema}")  # "IFC4"

output = ifcpatch.execute({
    "input": "building_ifc4.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4X3"]
})
ifcpatch.write(output, "building_ifc4x3.ifc")

migrated = ifcopenshell.open("building_ifc4x3.ifc")
print(f"Output schema: {migrated.schema}")  # "IFC4X3"

# After migration: IfcBuildingElement is now IfcBuiltElement
# StandardCase entities are merged into parent types
try:
    built_elements = migrated.by_type("IfcBuiltElement")
    print(f"IfcBuiltElement count: {len(built_elements)}")
except RuntimeError:
    print("ERROR: IfcBuiltElement not found — migration may have failed")
```

---

## Example 8: Post-Migration Validation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.validate

model = ifcopenshell.open("migrated_model.ifc")

# Run validation
logger = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, logger)

# Report results
errors = [s for s in logger.statements if s["severity"] == "Error"]
warnings = [s for s in logger.statements if s["severity"] == "Warning"]

print(f"Errors: {len(errors)}, Warnings: {len(warnings)}")
for error in errors[:10]:
    print(f"  ERROR: {error['message']}")
for warning in warnings[:10]:
    print(f"  WARNING: {warning['message']}")
```

---

## Example 9: IFC4X3 Infrastructure Project Setup

```python
# IfcOpenShell — IFC4X3 only
import ifcopenshell
import ifcopenshell.api

# Infrastructure entities require IFC4X3
model = ifcopenshell.api.run("project.create_file", version="IFC4X3")

project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Bridge Project")
ifcopenshell.api.run("unit.assign_unit", model)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Project Site")
bridge = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBridge", name="Main Bridge")
bridge_part = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBridgePart", name="Deck Section 1")

ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[bridge])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=bridge, products=[bridge_part])

model.write("bridge_project.ifc")
print(f"Schema: {model.schema}")  # "IFC4X3"
```

---

## Example 10: Compare Entity Attributes Across Schemas

```python
# IfcOpenShell — schema introspection
import ifcopenshell

def compare_entity_attributes(entity_name, schemas=("IFC2X3", "IFC4", "IFC4X3")):
    """Compare attributes of an entity across schema versions."""
    results = {}
    for schema_name in schemas:
        schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
        try:
            entity = schema.declaration_by_name(entity_name)
            attrs = [attr.name() for attr in entity.all_attributes()]
            results[schema_name] = attrs
        except RuntimeError:
            results[schema_name] = None  # Entity does not exist

    print(f"\n=== {entity_name} ===")
    for schema_name, attrs in results.items():
        if attrs is None:
            print(f"  {schema_name}: DOES NOT EXIST")
        else:
            print(f"  {schema_name}: {', '.join(attrs)}")

# Compare IfcWall across versions
compare_entity_attributes("IfcWall")
# IFC2X3: GlobalId, OwnerHistory, Name, Description, ObjectType, ObjectPlacement, Representation, Tag
# IFC4:   + PredefinedType
# IFC4X3: + PredefinedType

# Check renamed entities
compare_entity_attributes("IfcBuildingElement")
# IFC2X3: exists
# IFC4:   exists
# IFC4X3: DOES NOT EXIST (renamed to IfcBuiltElement)

compare_entity_attributes("IfcBuiltElement")
# IFC2X3: DOES NOT EXIST
# IFC4:   DOES NOT EXIST
# IFC4X3: exists
```

---

## Example 11: Schema-Aware Element Summary

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

def summarize_model(filepath):
    """Print a schema-aware summary of an IFC model."""
    model = ifcopenshell.open(filepath)
    schema = model.schema
    print(f"File: {filepath}")
    print(f"Schema: {schema}")
    print(f"Total entities: {len(model)}")

    # Version-aware queries
    if schema == "IFC4X3":
        building_elements = model.by_type("IfcBuiltElement")
    else:
        building_elements = model.by_type("IfcBuildingElement")

    print(f"Building elements: {len(building_elements)}")

    # Spatial structure (works across all versions)
    projects = model.by_type("IfcProject")
    sites = model.by_type("IfcSite")
    buildings = model.by_type("IfcBuilding")
    storeys = model.by_type("IfcBuildingStorey")

    print(f"Projects: {len(projects)}")
    print(f"Sites: {len(sites)}")
    print(f"Buildings: {len(buildings)}")
    print(f"Storeys: {len(storeys)}")

    # IFC4X3 infrastructure (only query if schema supports it)
    if schema == "IFC4X3":
        for infra_type in ["IfcRoad", "IfcBridge", "IfcRailway", "IfcAlignment"]:
            count = len(model.by_type(infra_type))
            if count > 0:
                print(f"{infra_type}: {count}")

summarize_model("model.ifc")
```

---

## Example 12: OwnerHistory Handling Across Versions

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

def create_wall_safe(model, name):
    """Create a wall that is valid in any schema version.
    Uses api.run() which handles OwnerHistory automatically."""
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=name)
    return wall

# IFC2X3: api.run creates OwnerHistory automatically
model_2x3 = ifcopenshell.api.run("project.create_file", version="IFC2X3")
project = ifcopenshell.api.run("root.create_entity", model_2x3,
    ifc_class="IfcProject", name="Test")
wall = create_wall_safe(model_2x3, "Wall IFC2X3")
print(f"OwnerHistory: {wall.OwnerHistory}")  # Not None in IFC2X3

# IFC4: OwnerHistory is optional, api.run may set it to None
model_4 = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model_4,
    ifc_class="IfcProject", name="Test")
wall = create_wall_safe(model_4, "Wall IFC4")
print(f"OwnerHistory: {wall.OwnerHistory}")  # May be None in IFC4
```
