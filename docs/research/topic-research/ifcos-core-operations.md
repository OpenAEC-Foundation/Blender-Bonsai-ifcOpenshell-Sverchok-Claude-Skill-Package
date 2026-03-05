# IfcOpenShell Python Library - Core Operations Research

## Date: 2026-03-05
## Status: RESEARCH (Phase 2 - Deep Research)
## Scope: File I/O, Element Traversal, util modules, geom module
## Sources: docs.ifcopenshell.org, academy.ifcopenshell.org, GitHub IfcOpenShell/IfcOpenShell

---

## 1. FILE I/O OPERATIONS

### 1.1 ifcopenshell.open() - Opening IFC Files

Opens an existing IFC file (STEP Physical File format, .ifc) and returns an `ifcopenshell.file` object.

**Signature:**
```python
ifcopenshell.open(path: str, should_stream: bool = False) -> ifcopenshell.file
```

**Parameters:**
- `path` (str): File path to the .ifc file. MUST be a string, not a Path object (older versions).
- `should_stream` (bool): If True, uses streaming mode for large files (lower memory, sequential access). Default: False.

**Working Code Examples:**
```python
import ifcopenshell

# Basic file opening
model = ifcopenshell.open("/path/to/model.ifc")

# Check schema version after opening
print(model.schema)       # "IFC2X3", "IFC4", or "IFC4X3"
print(model.schema_identifier)  # Same as above, canonical identifier

# Get header information
print(model.header.file_name.name)
print(model.header.file_description.description)

# Count entities
print(f"Total entities: {len(model)}")

# Streaming mode for very large files (100MB+)
# Only supports sequential iteration, NOT random access
model_stream = ifcopenshell.open("/path/to/huge_model.ifc", should_stream=True)
for entity in model_stream:
    if entity.is_a("IfcWall"):
        print(entity.Name)
```

**Anti-patterns:**
- NEVER pass `pathlib.Path` directly in older IfcOpenShell versions; convert to `str()` first.
- NEVER assume the schema -- ALWAYS check `model.schema` before using schema-specific attributes.
- NEVER keep files open indefinitely in loops; the entire file is loaded into memory (unless streaming).

**Error Handling:**
```python
import ifcopenshell

try:
    model = ifcopenshell.open("nonexistent.ifc")
except FileNotFoundError:
    print("File not found")
except ifcopenshell.Error:
    print("Invalid or corrupt IFC file")
```

---

### 1.2 file.write() - Saving IFC Files

Writes the in-memory IFC model to disk as STEP Physical File (.ifc).

**Signature:**
```python
file.write(path: str) -> None
```

**Working Code Examples:**
```python
import ifcopenshell

model = ifcopenshell.open("input.ifc")

# Modify something
wall = model.by_type("IfcWall")[0]
wall.Name = "Renamed Wall"

# Save to new file (preserves original)
model.write("output.ifc")

# Overwrite original (destructive)
model.write("input.ifc")
```

**Anti-patterns:**
- NEVER write to the same file you are reading from in a streaming scenario.
- ALWAYS write to a new path first, then rename if you want to replace the original (safe pattern).

**Safe Overwrite Pattern:**
```python
import ifcopenshell
import shutil

model = ifcopenshell.open("model.ifc")
# ... modifications ...
model.write("model_temp.ifc")
shutil.move("model_temp.ifc", "model.ifc")
```

---

### 1.3 ifcopenshell.file() - Creating New IFC Files

Creates a new, empty IFC file in memory with a specified schema.

**Signature:**
```python
ifcopenshell.file(schema: str = "IFC4") -> ifcopenshell.file
```

**Parameters:**
- `schema` (str): IFC schema identifier. Valid values: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"`. Default: `"IFC4"`.

**Working Code Examples:**
```python
import ifcopenshell

# Create new IFC4 file (recommended for most cases)
model = ifcopenshell.file(schema="IFC4")

# Create IFC2X3 file (legacy, for older software compatibility)
model_legacy = ifcopenshell.file(schema="IFC2X3")

# Create IFC4X3 file (latest, for infrastructure projects)
model_infra = ifcopenshell.file(schema="IFC4X3")

# Check what you created
print(model.schema)  # "IFC4"
```

**CRITICAL: A new file is empty. For a valid IFC, you need at minimum:**
1. IfcOwnerHistory (IFC2X3/IFC4) -- optional in IFC4X3
2. IfcProject
3. IfcUnitAssignment (units)
4. IfcGeometricRepresentationContext

**Minimal Valid IFC4 File (Manual Approach):**
```python
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file(schema="IFC4")

# Owner history (simplified)
person = model.create_entity("IfcPerson", FamilyName="Doe")
org = model.create_entity("IfcOrganization", Name="MyOrg")
person_org = model.create_entity("IfcPersonAndOrganization",
    ThePerson=person, TheOrganization=org)
app = model.create_entity("IfcApplication",
    ApplicationDeveloper=org,
    Version="1.0",
    ApplicationFullName="MyApp",
    ApplicationIdentifier="MyApp")
owner_history = model.create_entity("IfcOwnerHistory",
    OwningUser=person_org,
    OwningApplication=app,
    ChangeAction="NOCHANGE",
    CreationDate=0)

# Units
length_unit = model.create_entity("IfcSIUnit",
    UnitType="LENGTHUNIT", Name="METRE")
area_unit = model.create_entity("IfcSIUnit",
    UnitType="AREAUNIT", Name="SQUARE_METRE")
volume_unit = model.create_entity("IfcSIUnit",
    UnitType="VOLUMEUNIT", Name="CUBIC_METRE")
units = model.create_entity("IfcUnitAssignment",
    Units=[length_unit, area_unit, volume_unit])

# Context
context = model.create_entity("IfcGeometricRepresentationContext",
    ContextType="Model",
    CoordinateSpaceDimension=3,
    Precision=1e-5,
    WorldCoordinateSystem=model.create_entity("IfcAxis2Placement3D",
        Location=model.create_entity("IfcCartesianPoint", Coordinates=(0.0, 0.0, 0.0))
    ))

# Project
project = model.create_entity("IfcProject",
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=owner_history,
    Name="My Project",
    UnitsInContext=units,
    RepresentationContexts=[context])

model.write("minimal.ifc")
```

**RECOMMENDED: Use ifcopenshell.api instead of manual entity creation (see Section 1.4 note):**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)
context = ifcopenshell.api.run("context.add_context", model, context_type="Model",
    context_identifier="Body", target_view="MODEL_VIEW")
```

---

### 1.4 file.create_entity() - Creating IFC Entities Manually

Low-level method to create individual IFC entities. For high-level workflows, prefer `ifcopenshell.api.run()`.

**Signature:**
```python
file.create_entity(type: str, *args, **kwargs) -> ifcopenshell.entity_instance
```

**Parameters:**
- `type` (str): IFC entity class name (e.g., `"IfcWall"`, `"IfcCartesianPoint"`)
- `*args`: Positional arguments matching the entity's attribute order in the schema
- `**kwargs`: Named arguments matching entity attribute names

**Working Code Examples:**
```python
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file(schema="IFC4")

# Using keyword arguments (RECOMMENDED - explicit and readable)
point = model.create_entity("IfcCartesianPoint", Coordinates=(0.0, 0.0, 0.0))

# Using positional arguments (matches schema attribute order)
point2 = model.create_entity("IfcCartesianPoint", (1.0, 2.0, 3.0))

# Creating a wall with keyword arguments
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),
    Name="My Wall",
    Description="External wall",
    ObjectType="STANDARD")

# Entity attributes are accessible as properties
print(wall.GlobalId)         # "2hK7x..."
print(wall.Name)             # "My Wall"
print(wall.is_a())           # "IfcWall"
print(wall.is_a("IfcRoot"))  # True (IfcWall inherits from IfcRoot)
print(wall.id())             # STEP file ID (e.g., 42)

# Modify attributes after creation
wall.Name = "Updated Wall Name"
wall.Description = "Updated description"

# Get all attribute names and values
print(wall.get_info())
# Returns dict: {"id": 42, "type": "IfcWall", "GlobalId": "2hK7x...", "Name": "My Wall", ...}

# get_info() with include_identifier=False for clean data
info = wall.get_info(include_identifier=False)
```

**Entity Attribute Access Patterns:**
```python
# Access by name
name = wall.Name

# Access by index (matches schema order)
name = wall[2]  # Name is typically the 3rd attribute (index 2) for IfcRoot-derived

# Check if attribute is set (not None/$)
if wall.Description is not None:
    print(wall.Description)

# Get attribute info (name, type, whether optional)
for i, attr in enumerate(wall.wrapped_data.get_attribute_names()):
    print(f"  {i}: {attr} = {wall[i]}")
```

**Anti-patterns:**
- NEVER use `create_entity()` for complex operations (spatial containment, property sets, geometry) -- use `ifcopenshell.api.run()` instead, which handles all relationships automatically.
- NEVER forget GlobalId for IfcRoot-derived entities. Use `ifcopenshell.guid.new()` to generate.
- NEVER reuse GlobalIds between entities.

---

## 2. ELEMENT TRAVERSAL

### 2.1 file.by_type() - Query by IFC Type

Returns all entities of a given IFC class, optionally including subtypes.

**Signature:**
```python
file.by_type(type: str, include_subtypes: bool = True) -> list[ifcopenshell.entity_instance]
```

**Parameters:**
- `type` (str): IFC entity class name (e.g., `"IfcWall"`, `"IfcProduct"`)
- `include_subtypes` (bool): If True (default), includes subclasses. If False, exact type only.

**Working Code Examples:**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")

# Get ALL walls (including IfcWallStandardCase in IFC2X3)
walls = model.by_type("IfcWall")
print(f"Found {len(walls)} walls")

# Get ONLY IfcWall, NOT IfcWallStandardCase
walls_exact = model.by_type("IfcWall", include_subtypes=False)

# Get all products (everything with geometry potential)
products = model.by_type("IfcProduct")

# Get all building elements
elements = model.by_type("IfcBuildingElement")

# Get spatial structure elements
spatials = model.by_type("IfcSpatialStructureElement")  # IFC2X3/IFC4
# In IFC4X3: use IfcSpatialElement

# Iterate with filtering
for wall in model.by_type("IfcWall"):
    print(f"Wall: {wall.Name or 'Unnamed'} (GlobalId: {wall.GlobalId})")

# Common queries
projects = model.by_type("IfcProject")          # Always exactly 1
sites = model.by_type("IfcSite")
buildings = model.by_type("IfcBuilding")
storeys = model.by_type("IfcBuildingStorey")
spaces = model.by_type("IfcSpace")
doors = model.by_type("IfcDoor")
windows = model.by_type("IfcWindow")
slabs = model.by_type("IfcSlab")
columns = model.by_type("IfcColumn")
beams = model.by_type("IfcBeam")
roofs = model.by_type("IfcRoof")
stairs = model.by_type("IfcStair")
curtain_walls = model.by_type("IfcCurtainWall")
members = model.by_type("IfcMember")
plates = model.by_type("IfcPlate")
```

**Performance Notes:**
- `by_type()` is cached internally after first call per type. Subsequent calls are fast.
- For very large models, prefer targeted type queries over `by_type("IfcProduct")`.
- The include_subtypes=True behavior uses the IFC inheritance tree, which is powerful but can return unexpected types if you're not aware of the hierarchy.

**IFC Type Hierarchy (key branches):**
```
IfcRoot
  IfcObjectDefinition
    IfcObject
      IfcProduct
        IfcElement
          IfcBuildingElement
            IfcWall, IfcSlab, IfcColumn, IfcBeam, IfcDoor, IfcWindow, ...
          IfcDistributionElement
            IfcDistributionFlowElement
              IfcFlowSegment, IfcFlowTerminal, IfcFlowFitting, ...
          IfcOpeningElement
          IfcFurnishingElement
        IfcSpatialStructureElement (IFC2X3/IFC4)
          IfcSite, IfcBuilding, IfcBuildingStorey, IfcSpace
      IfcGroup
      IfcProcess
    IfcTypeObject
      IfcTypeProduct
        IfcElementType
          IfcWallType, IfcSlabType, IfcColumnType, ...
  IfcRelationship
    IfcRelDecomposes
      IfcRelAggregates
    IfcRelAssigns
    IfcRelConnects
      IfcRelContainedInSpatialStructure
      IfcRelFillsElement
      IfcRelVoidsElement
    IfcRelDefines
      IfcRelDefinesByType
      IfcRelDefinesByProperties
    IfcRelAssociates
      IfcRelAssociatesMaterial
      IfcRelAssociatesClassification
  IfcPropertyDefinition
    IfcPropertySet
    IfcPropertySetDefinition
```

---

### 2.2 file.by_id() - Query by STEP ID

Returns a single entity by its numeric STEP file ID (the #123 identifier in .ifc files).

**Signature:**
```python
file.by_id(id: int) -> ifcopenshell.entity_instance
```

**Working Code Examples:**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")

# Get entity by STEP ID
entity = model.by_id(42)
print(entity)               # #42=IfcWall('2hK7x...',...)
print(entity.is_a())        # "IfcWall"
print(entity.id())          # 42

# Get the STEP ID of any entity
wall = model.by_type("IfcWall")[0]
step_id = wall.id()
same_wall = model.by_id(step_id)
assert wall == same_wall  # True

# Error handling - invalid ID
try:
    entity = model.by_id(999999)
except RuntimeError:
    print("Entity with that ID does not exist")
```

**Anti-patterns:**
- NEVER use STEP IDs as permanent identifiers. They change when files are re-exported.
- ALWAYS use GlobalId (GUID) for persistent identification across file versions.
- STEP IDs are session-local identifiers, useful for debugging and internal references only.

---

### 2.3 file.by_guid() - Query by GlobalId

Returns a single entity by its IFC GlobalId (22-character base64 encoded GUID).

**Signature:**
```python
file.by_guid(guid: str) -> ifcopenshell.entity_instance
```

**Working Code Examples:**
```python
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.open("model.ifc")

# Get entity by GlobalId
wall = model.by_guid("2hK7xADfT1OBm_2bAddSRO")
print(wall.Name)

# GlobalId is only on IfcRoot-derived entities
# IfcCartesianPoint, IfcDirection, etc. do NOT have GlobalIds

# Generate a new GlobalId
new_guid = ifcopenshell.guid.new()
print(new_guid)  # e.g., "3hK7xADfT1OBm_2bAddSRO" (22 chars)

# Convert between IFC GUID and standard UUID
import uuid
standard_uuid = ifcopenshell.guid.expand(new_guid)   # Returns UUID string
ifc_guid = ifcopenshell.guid.compress(standard_uuid)  # Returns 22-char GUID

# Error handling
try:
    entity = model.by_guid("nonexistent_guid_12345")
except RuntimeError:
    print("No entity found with that GlobalId")
```

**GUID Format:**
- IFC uses a 22-character base64 encoding of a 128-bit UUID
- Characters: `0-9`, `A-Z`, `a-z`, `_`, `$`
- ALWAYS use `ifcopenshell.guid.new()` to generate; NEVER create manually

---

### 2.4 Inverse References

Inverse references find entities that REFERENCE a given entity. This is how you traverse relationships "upward" or "sideways" in IFC.

**Signature:**
```python
file.get_inverse(entity: ifcopenshell.entity_instance) -> set[ifcopenshell.entity_instance]
```

**Working Code Examples:**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")

wall = model.by_type("IfcWall")[0]

# Get ALL entities that reference this wall
inverse = model.get_inverse(wall)
for ref in inverse:
    print(f"  {ref.is_a()} #{ref.id()}")

# Common inverse patterns:

# 1. Find which storey contains an element
def get_storey(model, element):
    """Find the building storey that contains an element."""
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelContainedInSpatialStructure"):
            return rel.RelatingStructure
    return None

wall = model.by_type("IfcWall")[0]
storey = get_storey(model, wall)
if storey:
    print(f"Wall is on storey: {storey.Name}")

# 2. Find property sets attached to an element
def get_psets(model, element):
    """Get all property sets for an element."""
    psets = {}
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelDefinesByProperties"):
            pset = rel.RelatingPropertyDefinition
            if pset.is_a("IfcPropertySet"):
                psets[pset.Name] = {
                    prop.Name: prop.NominalValue.wrappedValue
                    for prop in pset.HasProperties
                    if prop.is_a("IfcPropertySingleValue") and prop.NominalValue
                }
    return psets

psets = get_psets(model, wall)
for pset_name, props in psets.items():
    print(f"\n{pset_name}:")
    for name, value in props.items():
        print(f"  {name}: {value}")

# 3. Find the type of an element
def get_element_type(model, element):
    """Get the type object for an element."""
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelDefinesByType"):
            return rel.RelatingType
    return None

wall_type = get_element_type(model, wall)
if wall_type:
    print(f"Wall type: {wall_type.Name}")

# 4. Find material of an element
def get_material(model, element):
    """Get material association for an element."""
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelAssociatesMaterial"):
            return rel.RelatingMaterial
    return None

# 5. Find openings in a wall
def get_openings(model, wall):
    """Get opening elements that void a wall."""
    openings = []
    for rel in model.get_inverse(wall):
        if rel.is_a("IfcRelVoidsElement"):
            openings.append(rel.RelatedOpeningElement)
    return openings
```

**IMPORTANT: Prefer ifcopenshell.util.element helpers over manual inverse traversal (see Section 3.1).**

---

### 2.5 Iterating Over All Entities

**Working Code Examples:**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")

# Iterate over ALL entities in the file
for entity in model:
    pass  # entity is an ifcopenshell.entity_instance

# Count entities by type
from collections import Counter
type_counts = Counter(entity.is_a() for entity in model)
for ifc_type, count in type_counts.most_common(20):
    print(f"  {ifc_type}: {count}")

# Total number of entities
total = len(model)
print(f"Total entities: {total}")

# Filter during iteration (more memory efficient than by_type for one-pass)
walls_with_name = [e for e in model if e.is_a("IfcWall") and e.Name]

# Check entity existence
entity = model.by_type("IfcWall")[0]
print(entity in model)  # True (not removed)

# Remove an entity
model.remove(entity)
print(entity in model)  # False (removed -- but the Python object still exists, don't use it)
```

**Removing Entities:**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Simple remove (does NOT remove references to this entity)
model.remove(wall)

# RECOMMENDED: Use ifcopenshell.api for safe removal with relationship cleanup
import ifcopenshell.api
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run("root.remove_product", model, product=wall)
```

---

## 3. ifcopenshell.util.* MODULES

### Complete Module Listing

| Module | Purpose | Key Functions |
|--------|---------|---------------|
| `ifcopenshell.util.element` | Element information helpers | `get_psets()`, `get_type()`, `get_container()`, `get_materials()`, `get_decomposition()` |
| `ifcopenshell.util.unit` | Unit conversion and parsing | `get_project_unit()`, `convert()`, `calculate_unit_scale()` |
| `ifcopenshell.util.placement` | Object placement calculations | `get_local_placement()`, `get_storey_elevation()` |
| `ifcopenshell.util.selector` | CSS-like element selection | `filter_elements()` (previously `Selector().parse()`) |
| `ifcopenshell.util.date` | IFC date format handling | `ifc2datetime()`, `datetime2ifc()` |
| `ifcopenshell.util.shape_builder` | Geometry construction helper | `ShapeBuilder` class for profiles, extrusions, booleans |
| `ifcopenshell.util.attribute` | Attribute type resolution | `get_primitive_type()` |
| `ifcopenshell.util.classification` | Classification reference helpers | `get_classification()`, `get_references()` |
| `ifcopenshell.util.cost` | Cost item calculations | Cost schedule utilities |
| `ifcopenshell.util.constraint` | Constraint utilities | Constraint management helpers |
| `ifcopenshell.util.doc` | Documentation/description helpers | Entity documentation lookups |
| `ifcopenshell.util.geolocation` | Geographic coordinates | `get_true_north()`, coordinate transformations |
| `ifcopenshell.util.pset` | Property set template utilities | PSet template handling |
| `ifcopenshell.util.representation` | Representation helpers | `get_representation()`, `resolve_representation()` |
| `ifcopenshell.util.schema` | Schema introspection | `get_entity_attributes()`, schema metadata |
| `ifcopenshell.util.sequence` | Work schedule/sequence | Task scheduling utilities |
| `ifcopenshell.util.system` | System membership | MEP system utilities |
| `ifcopenshell.util.type` | Type utilities | Type-instance relationship helpers |

---

### 3.1 ifcopenshell.util.element - Element Information Helpers

The MOST commonly used utility module. Provides clean access to element properties, types, spatial containment, and materials without manual inverse traversal.

```python
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# ---- GET PROPERTY SETS ----
# Returns dict of {pset_name: {prop_name: value}}
psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, props in psets.items():
    print(f"\n{pset_name}:")
    for prop_name, prop_value in props.items():
        print(f"  {prop_name} = {prop_value}")

# Get psets including quantity sets (IfcElementQuantity)
psets_and_qsets = ifcopenshell.util.element.get_psets(wall, psets_only=False)

# Get only quantity sets
qsets = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# ---- GET ELEMENT TYPE ----
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    print(f"Type: {wall_type.Name}")
    # Get type-level psets too
    type_psets = ifcopenshell.util.element.get_psets(wall_type)

# ---- GET SPATIAL CONTAINER ----
container = ifcopenshell.util.element.get_container(wall)
if container:
    print(f"Contained in: {container.is_a()} - {container.Name}")
    # Typically returns IfcBuildingStorey

# ---- GET MATERIAL ----
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material type: {material.is_a()}")
    # Could be IfcMaterial, IfcMaterialLayerSet, IfcMaterialLayerSetUsage,
    # IfcMaterialProfileSet, IfcMaterialConstituentSet, etc.
    if material.is_a("IfcMaterialLayerSetUsage"):
        for layer in material.ForLayerSet.MaterialLayers:
            print(f"  Layer: {layer.Material.Name} ({layer.LayerThickness}mm)")
    elif material.is_a("IfcMaterial"):
        print(f"  Single material: {material.Name}")

# ---- GET MATERIALS (all, as list) ----
materials = ifcopenshell.util.element.get_materials(wall)
# Returns list of IfcMaterial entities

# ---- GET DECOMPOSITION ----
# Get child elements (e.g., storeys in a building, elements in an assembly)
building = model.by_type("IfcBuilding")[0]
children = ifcopenshell.util.element.get_decomposition(building)
for child in children:
    print(f"  {child.is_a()}: {child.Name}")

# ---- GET AGGREGATE ----
# Get the parent aggregate (inverse of decomposition)
storey = model.by_type("IfcBuildingStorey")[0]
parent = ifcopenshell.util.element.get_aggregate(storey)
if parent:
    print(f"Parent: {parent.is_a()} - {parent.Name}")  # IfcBuilding

# ---- GET GROUPED ELEMENTS ----
# Elements in an IfcGroup
groups = model.by_type("IfcGroup")
for group in groups:
    members = ifcopenshell.util.element.get_grouped_by(group)
```

---

### 3.2 ifcopenshell.util.unit - Unit Conversion

Handles IFC unit systems (SI, Imperial) and provides conversion between units.

```python
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# ---- GET PROJECT UNITS ----
# Get the length unit defined in the project
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
print(f"Length unit: {length_unit}")  # e.g., IfcSIUnit METRE or MILLI METRE

area_unit = ifcopenshell.util.unit.get_project_unit(model, "AREAUNIT")
volume_unit = ifcopenshell.util.unit.get_project_unit(model, "VOLUMEUNIT")
angle_unit = ifcopenshell.util.unit.get_project_unit(model, "PLANEANGLEUNIT")

# ---- CALCULATE UNIT SCALE ----
# Get the scale factor to convert from file units to metres
# This is CRITICAL for correct geometry interpretation
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Unit scale (to metres): {unit_scale}")
# If file uses millimetres: unit_scale = 0.001
# If file uses metres: unit_scale = 1.0
# If file uses feet: unit_scale = 0.3048

# Convert a length value from file units to metres
length_in_file_units = 5000.0  # e.g., 5000 mm
length_in_metres = length_in_file_units * unit_scale

# ---- CONVERT BETWEEN UNITS ----
# Convert value from one unit to another
value_metres = ifcopenshell.util.unit.convert(
    value=1.0,
    from_prefix=None,        # No prefix = base unit
    from_unit="METRE",
    to_prefix="MILLI",       # Target: millimetres
    to_unit="METRE"
)
print(f"1 metre = {value_metres} millimetres")  # 1000.0

# Convert with unit entities
value = ifcopenshell.util.unit.convert(
    value=12.0,
    from_prefix=None,
    from_unit="FOOT",
    to_prefix=None,
    to_unit="METRE"
)
print(f"12 feet = {value} metres")  # 3.6576
```

---

### 3.3 ifcopenshell.util.placement - Object Placement

Calculates absolute and relative placements of IFC objects.

```python
import ifcopenshell
import ifcopenshell.util.placement
import numpy as np

model = ifcopenshell.open("model.ifc")

# ---- GET LOCAL PLACEMENT AS 4x4 MATRIX ----
wall = model.by_type("IfcWall")[0]
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
# Returns a 4x4 numpy matrix (homogeneous transformation)
print(matrix)
# [[Rx  Ry  Rz  Tx]
#  [Rx  Ry  Rz  Ty]
#  [Rx  Ry  Rz  Tz]
#  [0   0   0   1 ]]

# Extract position (translation component)
x, y, z = matrix[0][3], matrix[1][3], matrix[2][3]
print(f"Position: ({x}, {y}, {z})")

# ---- PLACEMENT CHAIN ----
# IFC uses relative placements (each element relative to its container)
# get_local_placement resolves the FULL chain to give absolute coordinates

# For an element contained in a storey:
# Element placement -> Storey placement -> Building placement -> Site placement
# get_local_placement resolves ALL of these automatically

# ---- GET STOREY ELEVATION ----
storey = model.by_type("IfcBuildingStorey")[0]
elevation = storey.Elevation  # Direct attribute, in file units
print(f"Storey elevation: {elevation}")
```

---

### 3.4 ifcopenshell.util.selector - CSS-like Element Selection

Provides a powerful query language for selecting IFC elements using a CSS-inspired syntax.

```python
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# ---- FILTER ELEMENTS (modern API) ----
# Select all walls
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")

# Select walls with a specific name
named_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Name="External Wall"')

# Select by property set value
# Syntax: IfcType, /PsetName/.PropertyName = "value"
fire_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')

# Select by type name
typed = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, type="WT01"')

# Select elements on a specific storey
storey_elements = ifcopenshell.util.selector.filter_elements(
    model, 'IfcBuildingElement, container="Ground Floor"')

# Select by material
concrete = ifcopenshell.util.selector.filter_elements(
    model, 'IfcElement, material="Concrete"')

# Complex queries with logical operators
complex_query = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True, Name *= "EXT"')

# ---- SELECTOR QUERY SYNTAX REFERENCE ----
# IfcType                         -> Select by IFC class
# , Name="X"                      -> Filter by Name attribute
# , Name *= "X"                   -> Name contains "X"
# , /PsetName/.PropName = value   -> Filter by property value
# , type="TypeName"               -> Filter by type object name
# , container="ContainerName"     -> Filter by spatial container
# , material="MaterialName"       -> Filter by material name
# , Description != None           -> Attribute is not null
```

**Note on API evolution:** Older versions used `Selector().parse(model, query)`. The modern API uses `filter_elements(model, query)`. ALWAYS use the modern form.

---

### 3.5 ifcopenshell.util.date - IFC Date Handling

Converts between IFC date/time representations and Python datetime objects.

```python
import ifcopenshell
import ifcopenshell.util.date
from datetime import datetime, date

# ---- IFC DATE FORMATS ----
# IFC uses multiple date representations:
# 1. IfcTimeStamp (integer, Unix epoch seconds)
# 2. IfcDate (string "YYYY-MM-DD")
# 3. IfcDateTime (string "YYYY-MM-DDTHH:MM:SS")
# 4. IfcDuration (string "P1Y2M3DT4H5M6S")
# 5. IfcCalendarDate (entity, IFC2X3 only)

# ---- CONVERT IFC TO PYTHON ----
# From timestamp (e.g., OwnerHistory.CreationDate)
model = ifcopenshell.open("model.ifc")
owner_history = model.by_type("IfcOwnerHistory")[0]
creation_timestamp = owner_history.CreationDate
if creation_timestamp:
    dt = ifcopenshell.util.date.ifc2datetime(creation_timestamp)
    print(f"Created: {dt}")  # datetime object

# From IFC date string
py_date = ifcopenshell.util.date.ifc2datetime("2024-06-15")

# From IFC datetime string
py_datetime = ifcopenshell.util.date.ifc2datetime("2024-06-15T10:30:00")

# From IFC duration string
py_duration = ifcopenshell.util.date.ifc2datetime("P30D")  # 30 days

# ---- CONVERT PYTHON TO IFC ----
# To IFC date string
ifc_date = ifcopenshell.util.date.datetime2ifc("2024-06-15", "IfcDate")

# To IFC datetime string
ifc_dt = ifcopenshell.util.date.datetime2ifc(
    datetime(2024, 6, 15, 10, 30, 0), "IfcDateTime")

# To Unix timestamp (for IfcTimeStamp / OwnerHistory)
import time
timestamp = int(time.time())
```

---

### 3.6 ifcopenshell.util.shape_builder - Geometry Construction

High-level helper for creating IFC geometry (profiles, extrusions, polylines, etc.).

```python
import ifcopenshell
import ifcopenshell.util.shape_builder

model = ifcopenshell.file(schema="IFC4")
builder = ifcopenshell.util.shape_builder.ShapeBuilder(model)

# ---- CREATE PROFILES ----

# Rectangle profile
rectangle = builder.rectangle(size=(0.3, 0.2))  # 300mm x 200mm
# Returns IfcRectangleProfileDef

# Circle profile
circle = builder.circle(radius=0.15)
# Returns IfcCircleProfileDef

# Arbitrary polyline profile
polyline_points = [(0.0, 0.0), (0.3, 0.0), (0.3, 0.2), (0.15, 0.3), (0.0, 0.2)]
polyline_profile = builder.polyline(polyline_points, closed=True)

# ---- CREATE EXTRUSIONS ----

# Extrude a profile along Z axis
extrusion = builder.extrude(
    rectangle,
    magnitude=3.0,               # Height: 3m
    position=None,               # Default: origin
    extrusion_vector=(0.0, 0.0, 1.0)  # Direction: up
)
# Returns IfcExtrudedAreaSolid

# ---- CREATE POLYLINES ----

# 2D polyline
polyline_2d = builder.polyline([(0.0, 0.0), (1.0, 0.0), (1.0, 1.0)])

# 3D polyline
polyline_3d = builder.polyline([(0.0, 0.0, 0.0), (1.0, 0.0, 0.0), (1.0, 0.0, 3.0)])

# ---- COMPLETE EXAMPLE: Create a simple wall geometry ----
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="Demo")
ifcopenshell.api.run("unit.assign_unit", model)
ctx = ifcopenshell.api.run("context.add_context", model, context_type="Model",
    context_identifier="Body", target_view="MODEL_VIEW")

site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=building, products=[storey])

wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 001")
ifcopenshell.api.run("spatial.assign_container", model, relating_structure=storey, products=[wall])

builder = ifcopenshell.util.shape_builder.ShapeBuilder(model)
profile = builder.rectangle(size=(5.0, 0.2))  # 5m long, 200mm thick
extrusion = builder.extrude(profile, magnitude=3.0)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall,
    representation=model.create_entity("IfcShapeRepresentation",
        ContextOfItems=ctx,
        RepresentationIdentifier="Body",
        RepresentationType="SweptSolid",
        Items=[extrusion]))

model.write("wall_demo.ifc")
```

---

### 3.7 Other Utility Modules (Brief Descriptions + Examples)

#### ifcopenshell.util.attribute
```python
import ifcopenshell.util.attribute

# Get the primitive Python type for an IFC attribute
# Useful for form generation or validation
attr_type = ifcopenshell.util.attribute.get_primitive_type(
    ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
        .declaration_by_name("IfcWall")
        .all_attributes()[2]  # Name attribute
)
```

#### ifcopenshell.util.classification
```python
import ifcopenshell.util.classification

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get classification references for an element
refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    print(f"Classification: {ref.Identification} - {ref.Name}")
```

#### ifcopenshell.util.geolocation
```python
import ifcopenshell.util.geolocation

model = ifcopenshell.open("model.ifc")

# Get true north angle
true_north = ifcopenshell.util.geolocation.get_true_north(model)
print(f"True north angle: {true_north} degrees")

# Get map conversion (coordinate reference system info)
# Useful for GIS integration
```

#### ifcopenshell.util.representation
```python
import ifcopenshell.util.representation

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get a specific representation by context and identifier
body_rep = ifcopenshell.util.representation.get_representation(
    wall, context="Model", subcontext="Body")
if body_rep:
    print(f"Representation type: {body_rep.RepresentationType}")
    print(f"Items: {len(body_rep.Items)}")

# Resolve mapped representations (IfcMappedItem -> actual geometry)
resolved = ifcopenshell.util.representation.resolve_representation(body_rep)
```

#### ifcopenshell.util.schema
```python
import ifcopenshell.util.schema

# Introspect IFC schema
# Get all attributes of an entity class
attrs = ifcopenshell.util.schema.get_entity_attributes("IFC4", "IfcWall")
```

---

## 4. ifcopenshell.geom MODULE

### 4.1 Overview

The `ifcopenshell.geom` module processes IFC geometry definitions into renderable/usable geometry (meshes, BRep shapes). It wraps the C++ geometry kernel (OpenCASCADE) for performance.

### 4.2 Settings Object

**Creating and Configuring Settings:**
```python
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")

# Create settings object
settings = ifcopenshell.geom.settings()

# ---- KEY PARAMETERS ----

# USE_BREP_DATA: Get BRep (boundary representation) as serialized OpenCASCADE shape
# Default: False
# Use for: CAD operations, boolean operations, precise geometry
settings.set(settings.USE_BREP_DATA, True)

# USE_WORLD_COORDS: Apply object placement to geometry (absolute coordinates)
# Default: False
# Use for: When you need geometry in world space, not local space
settings.set(settings.USE_WORLD_COORDS, True)

# WELD_VERTICES: Merge duplicate vertices in tessellated output
# Default: True
# Use for: Clean meshes; disable for per-face vertices (flat shading)
settings.set(settings.WELD_VERTICES, True)

# APPLY_DEFAULT_MATERIALS: Assign default material when none specified
# Default: True
settings.set(settings.APPLY_DEFAULT_MATERIALS, True)

# DISABLE_TRIANGULATION: Skip tessellation, only return BRep
# Default: False
# Use for: When you only need BRep data, saves processing time
settings.set(settings.DISABLE_TRIANGULATION, True)

# APPLY_LAYERSETS: Apply material layer set geometry offsets
# Default: False
settings.set(settings.APPLY_LAYERSETS, False)

# SEW_SHELLS: Attempt to sew open shells into closed solids
# Default: False (for performance; enable for quality)
settings.set(settings.SEW_SHELLS, True)

# DISABLE_OPENING_SUBTRACTIONS: Skip boolean subtraction of openings
# Default: False
# Use for: Performance when openings not needed, or debugging
settings.set(settings.DISABLE_OPENING_SUBTRACTIONS, False)

# ELEMENT_HIERARCHY: Include decomposition children in parent geometry
# Default: False
settings.set(settings.ELEMENT_HIERARCHY, False)
```

### 4.3 create_shape() - Process Single Element Geometry

**Signature:**
```python
ifcopenshell.geom.create_shape(settings, element) -> ShapeType
```

**Working Code Examples:**

#### Tessellated Output (Default - Triangulated Mesh)
```python
import ifcopenshell
import ifcopenshell.geom
import numpy as np

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

wall = model.by_type("IfcWall")[0]

# Process geometry
shape = ifcopenshell.geom.create_shape(settings, wall)

# Access geometry data
geometry = shape.geometry

# Vertices: flat array of [x1,y1,z1, x2,y2,z2, ...]
verts = geometry.verts
vertices = np.array(verts).reshape(-1, 3)
print(f"Vertices: {len(vertices)}")

# Faces: flat array of triangle indices [i1,i2,i3, i4,i5,i6, ...]
faces_flat = geometry.faces
faces = np.array(faces_flat).reshape(-1, 3)
print(f"Triangles: {len(faces)}")

# Edges: flat array of edge indices [i1,i2, i3,i4, ...]
edges_flat = geometry.edges
edges = np.array(edges_flat).reshape(-1, 2)
print(f"Edges: {len(edges)}")

# Normals: flat array like vertices
normals = np.array(geometry.normals).reshape(-1, 3) if geometry.normals else None

# Material IDs per face
material_ids = geometry.material_ids

# Materials (list of IfcSurfaceStyle info)
materials = geometry.materials

# Transformation matrix (4x4, column-major)
matrix = np.array(shape.transformation.matrix.data).reshape(4, 3)
# Note: This is a 4x3 matrix (rotation + translation)
# Convert to 4x4:
mat4x4 = np.eye(4)
mat4x4[:4, :3] = matrix

# ---- COMPLETE EXAMPLE: Extract all geometry to OBJ-like format ----
def extract_mesh(model, element):
    """Extract triangulated mesh from an IFC element."""
    settings = ifcopenshell.geom.settings()
    settings.set(settings.USE_WORLD_COORDS, True)

    try:
        shape = ifcopenshell.geom.create_shape(settings, element)
    except RuntimeError:
        return None  # No geometry or processing failed

    verts = np.array(shape.geometry.verts).reshape(-1, 3)
    faces = np.array(shape.geometry.faces).reshape(-1, 3)
    return {"vertices": verts, "faces": faces}

# Extract geometry for all walls
for wall in model.by_type("IfcWall"):
    mesh = extract_mesh(model, wall)
    if mesh:
        print(f"{wall.Name}: {len(mesh['vertices'])} verts, {len(mesh['faces'])} tris")
```

#### BRep Output (Precise Boundary Representation)
```python
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")

# BRep settings
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_BREP_DATA, True)
settings.set(settings.USE_WORLD_COORDS, True)

wall = model.by_type("IfcWall")[0]
shape = ifcopenshell.geom.create_shape(settings, wall)

# BRep data is serialized OpenCASCADE TopoDS_Shape
brep_data = shape.geometry.brep_data
# This is a binary string that can be deserialized with OCC

# To use with pythonOCC (PythonOCC-Core):
# from OCC.Core.BRepTools import BRepTools
# from OCC.Core.TopoDS import TopoDS_Shape
# from OCC.Core.BRep import BRep_Builder
# shape_occ = TopoDS_Shape()
# builder = BRep_Builder()
# BRepTools.Read(shape_occ, brep_data, builder)
```

### 4.4 Iterator - Batch Processing All Elements

For processing geometry of ALL elements in a file, use the iterator for much better performance than calling `create_shape()` in a loop.

```python
import ifcopenshell
import ifcopenshell.geom
import multiprocessing
import numpy as np

model = ifcopenshell.open("model.ifc")

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

# Create iterator for parallel processing
iterator = ifcopenshell.geom.iterator(
    settings,
    model,
    multiprocessing.cpu_count()  # Number of parallel threads
)

# Process all elements
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)

        verts = np.array(shape.geometry.verts).reshape(-1, 3)
        faces = np.array(shape.geometry.faces).reshape(-1, 3)

        print(f"{element.is_a()} '{element.Name}': "
              f"{len(verts)} vertices, {len(faces)} triangles")

        if not iterator.next():
            break

# ---- FILTERED ITERATOR ----
# Process only specific element types
iterator = ifcopenshell.geom.iterator(
    settings,
    model,
    multiprocessing.cpu_count(),
    include=model.by_type("IfcWall")  # Only process walls
)

# OR exclude certain types
iterator = ifcopenshell.geom.iterator(
    settings,
    model,
    multiprocessing.cpu_count(),
    exclude=model.by_type("IfcSpace")  # Skip spaces (usually invisible)
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        # ... process shape ...
        if not iterator.next():
            break
```

### 4.5 BRep vs Tessellation Decision Matrix

| Criterion | Tessellation (Default) | BRep |
|-----------|----------------------|------|
| **Use case** | Visualization, rendering, game engines | CAD operations, analysis, measurements |
| **Data type** | Triangles (vertices + face indices) | OpenCASCADE TopoDS_Shape |
| **Precision** | Approximated (faceted) | Exact (curves, surfaces preserved) |
| **Performance** | Faster to process, smaller data | Slower, larger data |
| **Dependencies** | None (built-in) | Needs PythonOCC for advanced ops |
| **Boolean ops** | Not possible | Full boolean support |
| **Measurements** | Approximate (from mesh) | Exact (from curves/surfaces) |
| **Settings** | Default settings | `USE_BREP_DATA=True` |

### 4.6 Common Geometry Processing Errors

```python
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()

for element in model.by_type("IfcProduct"):
    try:
        shape = ifcopenshell.geom.create_shape(settings, element)
    except RuntimeError as e:
        # Common causes:
        # - Element has no geometry representation
        # - Invalid geometry definition (degenerate shapes)
        # - Boolean operation failure (opening subtraction)
        # - Unsupported geometry type
        print(f"Failed: {element.is_a()} #{element.id()} - {e}")
    except Exception as e:
        print(f"Unexpected error: {element.is_a()} #{element.id()} - {e}")
```

**IMPORTANT: Not all IfcProduct subtypes have geometry.** These commonly lack geometry:
- `IfcProject`
- `IfcSite` (sometimes has terrain geometry)
- `IfcBuilding` (rarely has geometry)
- `IfcBuildingStorey` (almost never)
- Some `IfcAnnotation` types

---

## 5. SUPPLEMENTARY: ifcopenshell.api MODULE (Brief Reference)

While not in the original research scope, the `ifcopenshell.api` module is the RECOMMENDED high-level interface and is referenced throughout this document. Key API namespaces:

| Namespace | Purpose | Example |
|-----------|---------|---------|
| `root` | Create/remove entities | `api.run("root.create_entity", model, ifc_class="IfcWall")` |
| `spatial` | Spatial containment | `api.run("spatial.assign_container", model, ...)` |
| `aggregate` | Decomposition | `api.run("aggregate.assign_object", model, ...)` |
| `geometry` | Geometry management | `api.run("geometry.assign_representation", model, ...)` |
| `type` | Type assignment | `api.run("type.assign_type", model, ...)` |
| `pset` | Property sets | `api.run("pset.add_pset", model, ...)` |
| `material` | Material assignment | `api.run("material.assign_material", model, ...)` |
| `context` | Representation contexts | `api.run("context.add_context", model, ...)` |
| `unit` | Unit assignment | `api.run("unit.assign_unit", model)` |
| `owner` | Ownership/history | `api.run("owner.create_owner_history", model, ...)` |
| `classification` | Classification systems | `api.run("classification.add_classification", model, ...)` |
| `cost` | Cost items | `api.run("cost.add_cost_schedule", model, ...)` |
| `sequence` | Work schedules | `api.run("sequence.add_work_schedule", model, ...)` |
| `system` | MEP systems | `api.run("system.add_system", model, ...)` |
| `group` | Grouping | `api.run("group.add_group", model, ...)` |
| `void` | Openings | `api.run("void.add_opening", model, ...)` |
| `boundary` | Space boundaries | `api.run("boundary.assign_connection_geometry", model, ...)` |
| `structural` | Structural analysis | `api.run("structural.add_structural_member", model, ...)` |
| `document` | Document references | `api.run("document.add_reference", model, ...)` |
| `drawing` | Drawing generation | Bonsai-specific drawing tools |
| `nest` | Nesting relationships | `api.run("nest.assign_object", model, ...)` |
| `layer` | Presentation layers | `api.run("layer.add_layer", model, ...)` |
| `profile` | Profile definitions | `api.run("profile.add_parameterised_profile", model, ...)` |
| `style` | Visual styles | `api.run("style.add_style", model, ...)` |

**Standard api.run() Pattern:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# ALWAYS: api.run("namespace.function", model, **kwargs)
result = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall",
    name="My Wall",
    predefined_type="STANDARD")
```

---

## 6. CRITICAL VERSION DIFFERENCES

### IFC2X3 vs IFC4 vs IFC4X3

| Feature | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| Wall subtypes | IfcWall, IfcWallStandardCase | IfcWall only (PredefinedType) | IfcWall only |
| Slab subtypes | IfcSlab | IfcSlab, IfcSlabElementedCase | IfcSlab |
| OwnerHistory | REQUIRED on all IfcRoot | REQUIRED | OPTIONAL |
| Spatial elements | IfcSpatialStructureElement | IfcSpatialStructureElement | IfcSpatialElement (broader) |
| Infrastructure | Not supported | Not supported | IfcRoad, IfcBridge, IfcRailway, etc. |
| Property templates | Limited | Full PSD support | Full PSD support |
| Material profiles | IfcMaterialLayerSet only | + IfcMaterialProfileSet, IfcMaterialConstituentSet | Same as IFC4 |
| Alignment | Not available | Not available | IfcAlignment (linear referencing) |

### IfcOpenShell Python API Version Notes

- `file.schema` returns `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` (canonical strings)
- `by_type("IfcWallStandardCase")` works in IFC2X3 files but returns empty in IFC4
- ALWAYS check `model.schema` before using schema-specific entity types
- `ifcopenshell.api.run()` handles schema differences internally (preferred)

---

## 7. RESEARCH METADATA

### Sources Used
1. **docs.ifcopenshell.org** - Official API documentation
2. **academy.ifcopenshell.org** - Tutorial and learning resources
3. **blenderbim.org/docs-python/** - Python API docs
4. **github.com/IfcOpenShell/IfcOpenShell** - Source code (src/ifcopenshell-python/)
5. **buildingsmart.org** - IFC schema specifications

### Confidence Assessment
| Section | Confidence | Notes |
|---------|------------|-------|
| File I/O (1) | HIGH | Core, stable API; well documented |
| Element Traversal (2) | HIGH | Core, stable API; well documented |
| util.element (3.1) | HIGH | Most used utility; extensive real-world usage |
| util.unit (3.2) | HIGH | Critical for correct data; well tested |
| util.placement (3.3) | HIGH | Stable; numpy dependency noted |
| util.selector (3.4) | MEDIUM | API evolved (Selector -> filter_elements); verify current syntax |
| util.date (3.5) | HIGH | Simple conversion utilities |
| util.shape_builder (3.6) | MEDIUM | Relatively newer; API may have evolved |
| geom module (4) | HIGH | Core geometry engine; well documented |
| api module (5) | HIGH | Standard high-level interface |
| Version differences (6) | HIGH | Well-documented schema differences |

### Verification Needed
- [ ] Confirm `filter_elements()` is the current selector API (vs `Selector().parse()`)
- [ ] Verify `shape_builder` current method signatures
- [ ] Test `should_stream` parameter availability in current release
- [ ] Confirm `settings` parameter names (some may have been renamed)
- [ ] Verify `ifcopenshell.util.geolocation` function names

### Cross-References to Other Planned Skills
- `ifcos-syntax-fileio` -> Section 1 (File I/O)
- `ifcos-syntax-elements` -> Section 2 (Element Traversal)
- `ifcos-syntax-util` -> Section 3 (util modules)
- `ifcos-syntax-geometry` -> Section 4 (geom module)
- `ifcos-syntax-api` -> Section 5 (api module)
- `ifcos-core-schemas` -> Section 6 (version differences)
