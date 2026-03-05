# IfcOpenShell Error Patterns, Performance, and AI Pitfalls - Research Document

## Date: 2026-03-05
## Status: RESEARCH (Phase 2 / Phase 4 input)
## Scope: Error patterns, performance optimization, AI common mistakes, installation

---

## 1. Common Error Patterns

### 1.1 Schema Mismatch Errors

IFC has multiple schema versions (IFC2x3, IFC4, IFC4x3). Entities, attributes, and relationships differ between schemas. Using the wrong entity for a schema version is one of the most common errors.

**ERROR: Using IFC4-only entity in IFC2x3 file**
```python
import ifcopenshell

# WRONG: IfcBuildingElementProxy exists in both, but IfcBuiltElement is IFC4x3 only
model = ifcopenshell.file(schema="IFC2X3")
# This will raise RuntimeError because IfcBuiltElement does not exist in IFC2x3
element = model.create_entity("IfcBuiltElement", GlobalId=ifcopenshell.guid.new())
# RuntimeError: entity 'IfcBuiltElement' not found in schema 'IFC2X3'
```

**FIX: Check schema before entity creation**
```python
import ifcopenshell

model = ifcopenshell.file(schema="IFC2X3")
schema = model.schema

# Use schema-appropriate entities
if schema == "IFC2X3":
    # IFC2x3 uses IfcBuildingElement subtypes
    element = model.create_entity("IfcBuildingElementProxy",
        GlobalId=ifcopenshell.guid.new(),
        OwnerHistory=owner_history,  # REQUIRED in IFC2x3
        Name="Generic Element"
    )
elif schema == "IFC4":
    element = model.create_entity("IfcBuildingElementProxy",
        GlobalId=ifcopenshell.guid.new(),
        Name="Generic Element"
    )
elif schema == "IFC4X3":
    # IFC4x3 renamed many entities, e.g. IfcBuiltElement
    element = model.create_entity("IfcBuiltElement",
        GlobalId=ifcopenshell.guid.new(),
        Name="Generic Element"
    )
```

**Key schema differences to watch:**
| Entity/Concept | IFC2x3 | IFC4 | IFC4x3 |
|---------------|--------|------|--------|
| Building element base | IfcBuildingElement | IfcBuildingElement | IfcBuiltElement |
| Distribution element | IfcDistributionElement | IfcDistributionElement | IfcDistributionElement |
| Spatial container | IfcBuildingStorey only | IfcBuildingStorey, IfcSpace | IfcBuildingStorey, IfcSpace, IfcFacility |
| OwnerHistory | REQUIRED on most entities | OPTIONAL | OPTIONAL |
| Predefined type | attribute on subtype | PredefinedType attribute | PredefinedType attribute |
| Alignment | Not available | Not available | IfcAlignment (new) |
| Bridge/Road | Not available | Not available | IfcBridge, IfcRoad (new) |

---

### 1.2 Missing Required Attributes

Different IFC schemas have different required attributes. IFC2x3 is notably stricter about OwnerHistory.

**ERROR: Missing OwnerHistory in IFC2x3**
```python
import ifcopenshell

model = ifcopenshell.file(schema="IFC2X3")

# WRONG: IfcWallStandardCase REQUIRES OwnerHistory in IFC2x3
wall = model.create_entity("IfcWallStandardCase",
    GlobalId=ifcopenshell.guid.new(),
    Name="Wall 001"
)
# This may not raise immediately, but produces INVALID IFC
# Validators will reject this file. Many IFC viewers will fail to load it.
```

**FIX: Always create OwnerHistory for IFC2x3**
```python
import ifcopenshell
import ifcopenshell.guid
import time

model = ifcopenshell.file(schema="IFC2X3")

# Step 1: Create required ownership chain
person = model.create_entity("IfcPerson",
    FamilyName="Doe",
    GivenName="John"
)
organization = model.create_entity("IfcOrganization",
    Name="MyCompany"
)
person_org = model.create_entity("IfcPersonAndOrganization",
    ThePerson=person,
    TheOrganization=organization
)
application = model.create_entity("IfcApplication",
    ApplicationDeveloper=organization,
    Version="1.0",
    ApplicationFullName="MyApp",
    ApplicationIdentifier="MyApp"
)
owner_history = model.create_entity("IfcOwnerHistory",
    OwningUser=person_org,
    OwningApplication=application,
    ChangeAction="NOCHANGE",
    CreationDate=int(time.time())
)

# Step 2: NOW create the wall with OwnerHistory
wall = model.create_entity("IfcWallStandardCase",
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=owner_history,
    Name="Wall 001"
)
```

**FIX (better): Use ifcopenshell.api which handles this automatically**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC2X3")

# ifcopenshell.api.run handles OwnerHistory automatically
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject",
    name="My Project"
)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallStandardCase",
    name="Wall 001"
)
# The API layer creates OwnerHistory and assigns it
```

---

### 1.3 Incorrect Relationship Creation

IFC uses objectified relationships (IfcRelAggregates, IfcRelContainedInSpatialStructure, etc.). Creating them incorrectly is a common source of invalid files.

**ERROR: Direct assignment instead of relationship entity**
```python
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")

building = model.create_entity("IfcBuilding",
    GlobalId=ifcopenshell.guid.new(), Name="Building")
storey = model.create_entity("IfcBuildingStorey",
    GlobalId=ifcopenshell.guid.new(), Name="Ground Floor")

# WRONG: There is no "parent" attribute. IFC uses relationship entities.
# storey.parent = building  # AttributeError: entity has no attribute 'parent'

# WRONG: Manually creating relationship with wrong cardinality
rel = model.create_entity("IfcRelAggregates",
    GlobalId=ifcopenshell.guid.new(),
    RelatingObject=storey,        # WRONG: storey should be the child
    RelatedObjects=[building]     # WRONG: building should be the parent
)
```

**FIX: Correct relationship direction and use API**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Using API (recommended)
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="My Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# Correct spatial hierarchy: Project > Site > Building > Storey
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# For elements IN a storey, use spatial containment (not aggregation)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

**Common relationship confusion:**
| Relationship | Purpose | Example |
|-------------|---------|---------|
| IfcRelAggregates | Whole-part decomposition | Building contains Storeys |
| IfcRelContainedInSpatialStructure | Element in spatial container | Wall in Storey |
| IfcRelDefinesByType | Element to Type assignment | Wall to WallType |
| IfcRelDefinesByProperties | Property set assignment | Wall has Pset_WallCommon |
| IfcRelAssociatesMaterial | Material assignment | Wall has material |
| IfcRelVoidsElement | Opening in element | Opening in Wall |
| IfcRelFillsElement | Element filling opening | Window fills Opening |
| IfcRelNests | Ordered composition | Distribution ports in element |

---

### 1.4 Geometry Processing Failures

Geometry processing with `ifcopenshell.geom` depends on OpenCASCADE and can fail for various reasons.

**ERROR: Shape creation fails due to missing settings**
```python
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# WRONG: No settings object, or wrong settings
try:
    shape = ifcopenshell.geom.create_shape(wall)  # Missing settings argument
except TypeError:
    pass

# WRONG: Settings not configured for desired output
settings = ifcopenshell.geom.settings()
shape = ifcopenshell.geom.create_shape(settings, wall)
# May produce unexpected results if settings are not configured properly
```

**FIX: Properly configure geometry settings**
```python
import ifcopenshell
import ifcopenshell.geom
import numpy as np

model = ifcopenshell.open("model.ifc")

# Configure settings based on what you need
settings = ifcopenshell.geom.settings()

# For mesh/triangulated output (most common for visualization):
settings.set(settings.USE_WORLD_COORDS, True)  # Apply transformations
settings.set(settings.WELD_VERTICES, True)      # Merge duplicate vertices

# Process a single element
wall = model.by_type("IfcWall")[0]
try:
    shape = ifcopenshell.geom.create_shape(settings, wall)
    # Access geometry data
    verts = shape.geometry.verts          # Flat list: [x1,y1,z1, x2,y2,z2, ...]
    faces = shape.geometry.faces          # Flat list: [i1,i2,i3, i4,i5,i6, ...]

    # Reshape for use
    vertices = np.array(verts).reshape(-1, 3)
    triangles = np.array(faces).reshape(-1, 3)
except RuntimeError as e:
    # Some elements have no geometry or unsupported geometry types
    print(f"Cannot process geometry for {wall.Name}: {e}")
```

**ERROR: Processing elements without geometry representation**
```python
# WRONG: Trying to create shape for spatial elements without geometry
settings = ifcopenshell.geom.settings()
site = model.by_type("IfcSite")[0]
shape = ifcopenshell.geom.create_shape(settings, site)
# RuntimeError: Failed to process shape - IfcSite often has no 3D geometry
```

**FIX: Check for geometry representation before processing**
```python
def has_geometry(element):
    """Check if an IFC element has a 3D geometry representation."""
    if not hasattr(element, "Representation") or element.Representation is None:
        return False
    for rep in element.Representation.Representations:
        if rep.RepresentationIdentifier in ("Body", "Facetation", "Tessellation"):
            return True
    return False

# Only process elements with geometry
for wall in model.by_type("IfcWall"):
    if has_geometry(wall):
        try:
            shape = ifcopenshell.geom.create_shape(settings, wall)
        except RuntimeError:
            print(f"Geometry processing failed for {wall.GlobalId}")
```

---

### 1.5 Unit Conversion Issues

IFC files can use different unit systems. Failing to account for units leads to models that are 1000x too large or small.

**ERROR: Assuming meters when file uses millimeters**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# WRONG: Reading coordinates without checking units
# If the file is in millimeters, you get values like 3500 instead of 3.5
placement = wall.ObjectPlacement
# ... extracting coordinates directly without conversion
```

**FIX: Use ifcopenshell.util.unit for conversions**
```python
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# Method 1: Get the length unit scale factor
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# Returns factor to convert from file units to SI (meters)
# If file is in mm: unit_scale = 0.001
# If file is in m:  unit_scale = 1.0
# If file is in ft: unit_scale = 0.3048

# Apply to coordinates
raw_x = 3500.0  # From the IFC file
x_meters = raw_x * unit_scale  # 3.5 meters

# Method 2: Get specific unit type
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
print(f"Length unit: {length_unit}")  # e.g., IfcSIUnit MILLI METRE

# Method 3: Convert a value explicitly
value_in_si = ifcopenshell.util.unit.convert(
    value=3500.0,
    from_prefix=None,      # or "MILLI", "CENTI", etc.
    from_unit="METRE",
    to_prefix=None,
    to_unit="METRE"
)
```

**Common unit pitfalls:**
- Architectural models from the US use feet/inches (1 foot = 0.3048 meters)
- European structural models often use millimeters
- MEP models may use millimeters or centimeters
- Area units and volume units need separate conversion factors
- Angular units may be degrees or radians (IFC default is radians)

---

### 1.6 GUID-Related Errors

IFC uses 22-character base64-encoded GUIDs (GloballyUniqueId). Using standard UUIDs or duplicate GUIDs causes problems.

**ERROR: Using wrong GUID format**
```python
import ifcopenshell
import uuid

model = ifcopenshell.file(schema="IFC4")

# WRONG: Standard UUID string (36 chars with hyphens)
wall = model.create_entity("IfcWall",
    GlobalId=str(uuid.uuid4()),  # "550e8400-e29b-41d4-a716-446655440000" - TOO LONG
    Name="Wall"
)
# May cause: RuntimeError or truncation errors. IFC GlobalId is exactly 22 characters.

# WRONG: Reusing the same GUID
guid = ifcopenshell.guid.new()
wall1 = model.create_entity("IfcWall", GlobalId=guid, Name="Wall 1")
wall2 = model.create_entity("IfcWall", GlobalId=guid, Name="Wall 2")
# Duplicate GUIDs! File will be invalid. Viewers may show only one entity.
```

**FIX: Always use ifcopenshell.guid for GUID generation**
```python
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file(schema="IFC4")

# CORRECT: Generate proper IFC GUID (22-char base64)
guid1 = ifcopenshell.guid.new()  # e.g., "3Ks0WO3qP2xhJ0wBKg$u5H"
guid2 = ifcopenshell.guid.new()  # Always unique

wall1 = model.create_entity("IfcWall", GlobalId=guid1, Name="Wall 1")
wall2 = model.create_entity("IfcWall", GlobalId=guid2, Name="Wall 2")

# Convert between standard UUID and IFC GUID
standard_uuid = ifcopenshell.guid.expand(guid1)  # Returns UUID string
ifc_guid = ifcopenshell.guid.compress(standard_uuid)  # Back to IFC GUID

# When using ifcopenshell.api, GUIDs are auto-generated
wall3 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 3")
# wall3.GlobalId is automatically set to a unique IFC GUID
```

---

### 1.7 File Corruption Patterns

IFC files can become corrupted through incorrect modification patterns.

**ERROR: Removing entities without cleaning up references**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")

wall = model.by_type("IfcWall")[0]

# WRONG: Direct removal leaves dangling references
model.remove(wall)
# Now any IfcRelContainedInSpatialStructure, IfcRelDefinesByProperties,
# IfcRelAssociatesMaterial that referenced this wall has a dangling reference.
# The file is CORRUPTED.
```

**FIX: Use the API to remove elements (handles cleanup)**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")

wall = model.by_type("IfcWall")[0]

# CORRECT: API removes element AND cleans up all relationships
ifcopenshell.api.run("root.remove_product", model, product=wall)
# This removes:
# - The wall entity itself
# - Its placement (IfcLocalPlacement) if not shared
# - Its representation (IfcProductDefinitionShape) if not shared
# - References in IfcRelContainedInSpatialStructure
# - References in IfcRelDefinesByProperties
# - References in IfcRelAssociatesMaterial
# - Any IfcRelVoidsElement relationships
# etc.
```

**ERROR: Modifying entity attributes with wrong data types**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# WRONG: Setting string where entity reference is expected
wall.ObjectPlacement = "at origin"
# RuntimeError: Expected IfcObjectPlacement, got str

# WRONG: Setting single value where aggregate is expected
rel = model.by_type("IfcRelContainedInSpatialStructure")[0]
rel.RelatedElements = wall  # WRONG: expects a tuple/set, not single entity
```

**FIX: Use correct data types**
```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# CORRECT: Use entity references
origin = model.create_entity("IfcCartesianPoint", Coordinates=(0.0, 0.0, 0.0))
axis = model.create_entity("IfcAxis2Placement3D", Location=origin)
placement = model.create_entity("IfcLocalPlacement", RelativePlacement=axis)
wall.ObjectPlacement = placement  # Entity reference, not string

# CORRECT: Use tuple for aggregates
rel = model.by_type("IfcRelContainedInSpatialStructure")[0]
existing = list(rel.RelatedElements)
existing.append(wall)
rel.RelatedElements = tuple(existing)  # Must be tuple, not list
```

---

### 1.8 Property Set Errors

Property sets (Psets) are the primary mechanism for attaching data to IFC elements. Errors here are extremely common.

**ERROR: Creating property set without proper structure**
```python
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Wall 001")

# WRONG: Trying to set properties directly on the wall
# wall.FireRating = "REI60"  # AttributeError: not a direct attribute

# WRONG: Creating property set without linking to element
pset = model.create_entity("IfcPropertySet",
    GlobalId=ifcopenshell.guid.new(),
    Name="Pset_WallCommon",
    HasProperties=[
        model.create_entity("IfcPropertySingleValue",
            Name="FireRating",
            NominalValue=model.create_entity("IfcLabel", wrappedValue="REI60")
        )
    ]
)
# pset exists but is NOT connected to the wall - orphaned entity
```

**FIX: Use API or create complete relationship chain**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Method 1: Using API (recommended)
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset,
    properties={
        "FireRating": "REI60",
        "IsExternal": True,
        "ThermalTransmittance": 0.24
    }
)
# API automatically creates IfcRelDefinesByProperties to link pset to wall

# Method 2: Manual creation (when API is not available)
prop1 = model.create_entity("IfcPropertySingleValue",
    Name="FireRating",
    NominalValue=model.create_entity("IfcLabel", wrappedValue="REI60")
)
prop2 = model.create_entity("IfcPropertySingleValue",
    Name="IsExternal",
    NominalValue=model.create_entity("IfcBoolean", wrappedValue=True)
)
pset = model.create_entity("IfcPropertySet",
    GlobalId=ifcopenshell.guid.new(),
    Name="Pset_WallCommon",
    HasProperties=[prop1, prop2]
)
# CRITICAL: Must create relationship to connect pset to wall
model.create_entity("IfcRelDefinesByProperties",
    GlobalId=ifcopenshell.guid.new(),
    RelatingPropertyDefinition=pset,
    RelatedObjects=[wall]
)
```

**Reading properties correctly:**
```python
import ifcopenshell.util.element

# CORRECT: Use utility function to read properties
props = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"FireRating": "REI60", "IsExternal": True, ...}}

fire_rating = props.get("Pset_WallCommon", {}).get("FireRating")

# Get quantity sets too
qsets = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
```

---

### 1.9 Type Assignment Errors

IFC separates element occurrences (IfcWall) from element types (IfcWallType). Incorrect type assignment is common.

**ERROR: Wrong type class for element**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# WRONG: Creating wrong type class
slab_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlabType", name="Concrete Slab Type")

# WRONG: Assigning slab type to a wall
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=slab_type)
# This may succeed technically but produces semantically invalid IFC
# Validators will flag this as a type mismatch
```

**FIX: Match element and type classes**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# CORRECT: Create matching element and type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Basic Wall 200mm",
    predefined_type="SOLIDWALL")
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001",
    predefined_type="SOLIDWALL")

# Assign matching type
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# Element-to-Type mapping:
# IfcWall          -> IfcWallType
# IfcSlab          -> IfcSlabType
# IfcColumn        -> IfcColumnType
# IfcBeam          -> IfcBeamType
# IfcDoor          -> IfcDoorType (IFC4) / IfcDoorStyle (IFC2x3)
# IfcWindow        -> IfcWindowType (IFC4) / IfcWindowStyle (IFC2x3)
# IfcMember        -> IfcMemberType
# IfcPlate         -> IfcPlateType
# IfcCovering      -> IfcCoveringType
# IfcFurnishingElement -> IfcFurnishingElementType (IFC4) / IfcFurnitureType (IFC2x3)
```

---

## 2. Performance Considerations for Large Models

### 2.1 Memory Usage Patterns

**Problem: Loading entire large model into memory**
```python
import ifcopenshell

# For a 500MB IFC file, this loads EVERYTHING into memory
# Memory usage can be 3-5x the file size (1.5-2.5 GB RAM)
model = ifcopenshell.open("large_model.ifc")

# Then iterating all elements compounds the issue
all_walls = model.by_type("IfcWall")
for wall in all_walls:
    psets = ifcopenshell.util.element.get_psets(wall)
    # Each pset lookup traverses relationships
```

**Optimization: Use selective loading and generators**
```python
import ifcopenshell

# Strategy 1: Only extract what you need
model = ifcopenshell.open("large_model.ifc")

# Use by_type with include_subtypes=False when you know the exact type
walls = model.by_type("IfcWall", include_subtypes=False)
# Faster than include_subtypes=True (default) which also checks IfcWallStandardCase etc.

# Strategy 2: Use by_id for specific elements (O(1) lookup)
element = model.by_id(12345)  # Direct lookup, very fast

# Strategy 3: Process and release
# Extract data, then let Python GC clean up references
wall_data = []
for wall in model.by_type("IfcWall"):
    wall_data.append({
        "id": wall.id(),
        "guid": wall.GlobalId,
        "name": wall.Name
    })
# Now work with wall_data (plain dicts), not IFC entities
# The entity references are not held, reducing memory pressure

# Strategy 4: Close file when done
del model  # Release the file object and all entities
```

### 2.2 Efficient Querying Strategies

```python
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# SLOW: Iterating all elements and checking conditions
external_walls = []
for wall in model.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    if psets.get("Pset_WallCommon", {}).get("IsExternal", False):
        external_walls.append(wall)

# FASTER: Use ifcopenshell.util.selector (IFC query language)
external_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal=True'
)

# FASTEST for repeated lookups: Build index once
from collections import defaultdict
walls_by_storey = defaultdict(list)
for rel in model.by_type("IfcRelContainedInSpatialStructure"):
    container = rel.RelatingStructure
    for element in rel.RelatedElements:
        if element.is_a("IfcWall"):
            walls_by_storey[container.Name].append(element)
# Now O(1) lookup by storey name
```

### 2.3 Batch Operations vs Individual Operations

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# SLOW: Individual API calls in a loop
wall_names = [f"Wall {i}" for i in range(1000)]
walls = []
for name in wall_names:
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=name)
    walls.append(wall)
# Each API call has overhead: validation, owner history lookup, etc.

# FASTER: Use batch mode (disable undo tracking)
# ifcopenshell.api has internal transaction management
# For bulk operations, create entities directly when possible
walls_fast = []
for i in range(1000):
    wall = model.create_entity("IfcWall",
        GlobalId=ifcopenshell.guid.new(),
        Name=f"Wall {i}")
    walls_fast.append(wall)
# Direct entity creation is 5-10x faster than API calls
# BUT: You lose automatic OwnerHistory, validation, and relationship management

# RECOMMENDED for bulk: API with transaction batching
# Assign psets in bulk rather than one-by-one
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall")
for wall in walls:
    ifcopenshell.api.run("type.assign_type", model,
        related_objects=[wall], relating_type=wall_type)
# Better: assign all at once
ifcopenshell.api.run("type.assign_type", model,
    related_objects=walls, relating_type=wall_type)
# Single relationship entity with multiple related objects
```

### 2.4 Geometry Processing Optimization

**Use iterator for batch geometry processing (large files)**
```python
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")

# SLOW: create_shape one element at a time
settings = ifcopenshell.geom.settings()
for wall in model.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)
    # Sequential processing, no parallelism

# FAST: Use iterator for batch processing with parallelism
settings = ifcopenshell.geom.settings()
iterator = ifcopenshell.geom.iterator(
    settings,
    model,
    multiprocessing.cpu_count(),  # Number of parallel threads
    include=model.by_type("IfcWall")  # Optional: filter entities
)

# Process shapes in batches
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)

        verts = shape.geometry.verts
        faces = shape.geometry.faces

        # Process geometry here...

        if not iterator.next():
            break
```

**Key differences: iterator vs create_shape**
| Aspect | `create_shape()` | `ifcopenshell.geom.iterator` |
|--------|-----------------|------------------------------|
| Processing | Single element | Batch processing |
| Parallelism | None | Multi-threaded (OpenMP) |
| Memory | Loads one at a time | Can stream results |
| Use case | Few elements, interactive | Large files, batch export |
| Speed (1000 elements) | ~60 seconds | ~8 seconds (8 cores) |
| Error handling | Try/except per element | Skips failed elements |

### 2.5 File Loading Strategies

```python
import ifcopenshell

# Eager loading (default) - loads entire file
model = ifcopenshell.open("model.ifc")
# All entities parsed and loaded into memory immediately

# For very large files (500MB+), consider:

# Strategy 1: Process file in streaming fashion
# Read raw STEP file and extract only needed data
def extract_wall_names_streaming(filepath):
    """Extract wall names without full model load."""
    names = []
    with open(filepath, 'r') as f:
        for line in f:
            if "IFCWALL(" in line.upper() or "IFCWALLSTANDARDCASE(" in line.upper():
                # Parse the STEP line for the Name attribute
                # This is a simplified example; real STEP parsing is more complex
                parts = line.split("'")
                if len(parts) >= 4:
                    names.append(parts[3])  # Name is typically 4th quoted string
    return names

# Strategy 2: Use ifcopenshell with targeted queries
model = ifcopenshell.open("large_model.ifc")
# Get only the entity IDs you need
wall_ids = [w.id() for w in model.by_type("IfcWall")]
# Process in chunks
chunk_size = 100
for i in range(0, len(wall_ids), chunk_size):
    chunk = wall_ids[i:i + chunk_size]
    for wid in chunk:
        wall = model.by_id(wid)
        # Process wall...

# Strategy 3: HDF5 cache for repeated access
# IfcOpenShell can create HDF5 cache files for faster re-loading
# This is used internally by Bonsai/BlenderBIM
import ifcopenshell.ifcopenshell_wrapper as W
# HDF5 caching is primarily used via Bonsai's internal mechanisms
```

### 2.6 Memory Management for Large Files (100MB+)

```python
import ifcopenshell
import gc

# Problem: 100MB+ IFC files can use 500MB-1GB+ RAM when loaded

# Strategy 1: Load, extract, release
def extract_data_from_large_file(filepath):
    """Extract needed data and release the model."""
    model = ifcopenshell.open(filepath)

    # Extract only what you need into plain Python objects
    data = {
        "project_name": model.by_type("IfcProject")[0].Name,
        "wall_count": len(model.by_type("IfcWall")),
        "walls": []
    }

    for wall in model.by_type("IfcWall"):
        psets = ifcopenshell.util.element.get_psets(wall)
        data["walls"].append({
            "guid": wall.GlobalId,
            "name": wall.Name,
            "properties": psets
        })

    # Release the model
    del model
    gc.collect()

    return data  # Work with plain dicts from here

# Strategy 2: Process geometry in batches to limit peak memory
def process_geometry_batched(filepath, batch_size=50):
    """Process geometry in controlled batches."""
    model = ifcopenshell.open(filepath)
    settings = ifcopenshell.geom.settings()

    elements = model.by_type("IfcProduct")
    results = []

    for i in range(0, len(elements), batch_size):
        batch = elements[i:i + batch_size]
        for element in batch:
            if not hasattr(element, "Representation") or element.Representation is None:
                continue
            try:
                shape = ifcopenshell.geom.create_shape(settings, element)
                # Extract data you need
                results.append({
                    "id": element.id(),
                    "volume": calculate_volume(shape),
                })
            except RuntimeError:
                pass

        # Force garbage collection between batches
        gc.collect()

    del model
    gc.collect()
    return results

# Strategy 3: Use iterator with include filter for geometry
def process_only_walls_geometry(filepath):
    """Only process wall geometry, ignore everything else."""
    model = ifcopenshell.open(filepath)
    settings = ifcopenshell.geom.settings()

    walls = model.by_type("IfcWall")
    iterator = ifcopenshell.geom.iterator(settings, model, 4, include=walls)

    results = []
    if iterator.initialize():
        while True:
            shape = iterator.get()
            results.append(shape.geometry.verts)
            if not iterator.next():
                break

    del model
    gc.collect()
    return results
```

---

## 3. AI Common Mistakes When Generating IfcOpenShell Code

### 3.1 Hallucinated API Methods That Don't Exist

```python
# WRONG: These methods DO NOT EXIST in ifcopenshell
model = ifcopenshell.open("model.ifc")

# Hallucination 1: .get_properties() does not exist on entities
wall.get_properties()           # AttributeError
wall.properties                 # AttributeError
wall.get_property("FireRating") # AttributeError

# CORRECT: Use ifcopenshell.util.element
import ifcopenshell.util.element
psets = ifcopenshell.util.element.get_psets(wall)
# Returns dict of dicts: {"PsetName": {"PropName": value, ...}}

# Hallucination 2: .get_geometry() does not exist on entities
wall.get_geometry()     # AttributeError
wall.geometry           # AttributeError (no such attribute)

# CORRECT: Use ifcopenshell.geom.create_shape()
settings = ifcopenshell.geom.settings()
shape = ifcopenshell.geom.create_shape(settings, wall)

# Hallucination 3: .children or .get_children() does not exist
building.children               # AttributeError
building.get_children()         # AttributeError
building.get_contained_elements()  # AttributeError

# CORRECT: Use inverse references or utility functions
import ifcopenshell.util.element
children = ifcopenshell.util.element.get_decomposition(building)
# Or for spatial containment:
contained = ifcopenshell.util.element.get_container(wall)

# Hallucination 4: model.get_all() or model.entities does not exist
model.get_all()         # AttributeError
model.entities          # AttributeError

# CORRECT:
all_entities = model.by_type("IfcProduct")  # All products
# Or iterate: for entity in model: ...

# Hallucination 5: model.add_wall(), model.add_slab() etc. do not exist
model.add_wall("Wall 1")    # AttributeError
model.create_wall()         # AttributeError

# CORRECT:
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 1")
# Or:
wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())

# Hallucination 6: ifcopenshell.read() does not exist
model = ifcopenshell.read("file.ifc")  # AttributeError

# CORRECT:
model = ifcopenshell.open("file.ifc")

# Hallucination 7: model.save() does not exist
model.save("output.ifc")    # AttributeError

# CORRECT:
model.write("output.ifc")

# Hallucination 8: ifcopenshell.create() does not exist
model = ifcopenshell.create(schema="IFC4")  # AttributeError

# CORRECT:
model = ifcopenshell.file(schema="IFC4")
```

### 3.2 Wrong Parameter Orders and Names

```python
# WRONG: Parameter name confusion in API calls
# The API uses specific parameter names that must match exactly

# Wrong parameter name: "element" vs "product"
ifcopenshell.api.run("root.remove_product", model, element=wall)
# CORRECT:
ifcopenshell.api.run("root.remove_product", model, product=wall)

# Wrong parameter name: "parent" vs "relating_object"
ifcopenshell.api.run("aggregate.assign_object", model,
    product=storey, parent=building)
# CORRECT:
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
# Note: "products" is a list, not singular "product"

# Wrong parameter name: "container" vs "relating_structure"
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, container=storey)
# CORRECT:
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# Wrong parameter name in pset operations
ifcopenshell.api.run("pset.add_pset", model,
    element=wall, name="Pset_WallCommon")
# CORRECT:
ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

# Wrong parameter for type assignment
ifcopenshell.api.run("type.assign_type", model,
    element=wall, type=wall_type)
# CORRECT:
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

### 3.3 Confusing ifcopenshell.api.run() with Direct Entity Creation

```python
# These are TWO DIFFERENT approaches with different trade-offs

# Approach 1: Direct entity creation (low-level)
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),
    Name="Wall 001"
)
# Pros: Fast, no overhead
# Cons: No OwnerHistory, no validation, no automatic GUID,
#        no relationship management, easy to create invalid IFC

# Approach 2: API calls (high-level, recommended)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall",
    name="Wall 001"
)
# Pros: Handles OwnerHistory (IFC2x3), generates GUID, validates schema,
#        provides undo support, proper relationship management
# Cons: Slower (5-10x), more verbose for bulk operations

# COMMON MISTAKE: Mixing approaches inconsistently
# Creating entity with API but modifying with direct attribute access
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
wall.Name = "Changed Name"  # This works but bypasses undo tracking

# Better: Use API for modifications too
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "Changed Name"})

# RULE OF THUMB:
# - Use API for: creating/removing entities, managing relationships, psets
# - Use direct access for: reading attributes, quick queries
# - Use direct creation for: bulk operations where speed matters
#   (but ensure you handle OwnerHistory, GUIDs, relationships yourself)
```

### 3.4 Schema Version Assumptions

```python
# MISTAKE: Assuming the schema version without checking

model = ifcopenshell.open("unknown_file.ifc")

# WRONG: Assuming IFC4 and using IFC4-only features
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Wall Type")
# In IFC2x3, IfcWallType exists but some attributes differ

# WRONG: Using IfcWallStandardCase (deprecated in IFC4)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallStandardCase", name="Wall")
# IfcWallStandardCase exists in IFC2x3 and IFC4, but is
# DEPRECATED in IFC4 (use IfcWall with PredefinedType instead)
# Does NOT exist in IFC4x3

# CORRECT: Always check schema first
schema = model.schema
print(f"Schema: {schema}")  # "IFC2X3", "IFC4", "IFC4X3"

if schema == "IFC2X3":
    # Use IFC2x3-appropriate entities and patterns
    # OwnerHistory is REQUIRED
    # IfcDoorStyle/IfcWindowStyle (not IfcDoorType/IfcWindowType)
    pass
elif schema == "IFC4":
    # OwnerHistory is optional
    # IfcDoorType/IfcWindowType (not Style)
    # IfcWallStandardCase is deprecated, use IfcWall
    pass
elif schema.startswith("IFC4X3"):
    # IfcBuiltElement replaces IfcBuildingElement
    # New entities: IfcAlignment, IfcRoad, IfcBridge, IfcFacility
    # IfcWallStandardCase does NOT exist
    pass

# CORRECT: Schema-aware entity selection
def get_wall_class(schema):
    if schema == "IFC2X3":
        return "IfcWallStandardCase"  # Common in IFC2x3
    else:
        return "IfcWall"  # Preferred in IFC4+

def get_door_type_class(schema):
    if schema == "IFC2X3":
        return "IfcDoorStyle"
    else:
        return "IfcDoorType"
```

### 3.5 Missing Owner History (IFC2x3 Requirement)

```python
# In IFC2x3, OwnerHistory is REQUIRED on virtually all rooted entities
# In IFC4/IFC4x3, it is OPTIONAL

# MISTAKE: Creating IFC2x3 entities without OwnerHistory
model = ifcopenshell.file(schema="IFC2X3")

# This creates an INVALID IFC2x3 file
project = model.create_entity("IfcProject",
    GlobalId=ifcopenshell.guid.new(),
    Name="My Project"
)
# Missing: OwnerHistory parameter

# CORRECT for IFC2x3: Always include OwnerHistory
# Use ifcopenshell.api.run which handles this automatically:
model = ifcopenshell.file(schema="IFC2X3")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
# The API creates OwnerHistory automatically for IFC2x3

# OR manually if you must use create_entity:
owner_history = create_owner_history(model)  # See section 1.2 for full code
project = model.create_entity("IfcProject",
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=owner_history,
    Name="My Project"
)

# CHECKING if OwnerHistory is needed:
def needs_owner_history(model):
    return model.schema == "IFC2X3"
```

### 3.6 Incorrect Geometry Creation

```python
# MISTAKE: Creating geometry without proper representation context

model = ifcopenshell.file(schema="IFC4")
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")

# WRONG: Creating geometry without a representation context
# The model needs a geometric representation context first
point1 = model.create_entity("IfcCartesianPoint", Coordinates=(0.0, 0.0, 0.0))
point2 = model.create_entity("IfcCartesianPoint", Coordinates=(5.0, 0.0, 0.0))
polyline = model.create_entity("IfcPolyline", Points=[point1, point2])
# This geometry is orphaned - not connected to any context or element

# CORRECT: Full geometry creation workflow
# Step 1: Create project with units and context
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")

# Create units
ifcopenshell.api.run("unit.assign_unit", model,
    length={"is_metric": True, "raw": "METRES"},
    area={"is_metric": True, "raw": "SQUARE_METRE"},
    volume={"is_metric": True, "raw": "CUBIC_METRE"}
)

# Create geometric representation context
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body_context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=context)

# Step 2: Create geometry using API
# Example: Extruded rectangle for a wall
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Create an extruded area solid (rectangle extruded along Z)
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context,
    length=5.0,    # 5 meters
    height=3.0,    # 3 meters
    thickness=0.2  # 200mm
)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)

# Add placement
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall)

# COMMON GEOMETRY MISTAKES:
# 1. Forgetting to create SubContext (Body, Axis, etc.)
# 2. Using wrong coordinate system (IFC uses right-hand, Z-up)
# 3. Not assigning representation to element
# 4. Missing units definition (geometry values interpreted incorrectly)
# 5. Creating 2D geometry when 3D is expected (or vice versa)
```

---

## 4. Installation

### 4.1 pip Installation (Recommended)

```bash
# Standard installation
pip install ifcopenshell

# NOTE: The pip package may lag behind the latest development version.
# For the latest version, use the development builds or build from source.

# Verify installation
python -c "import ifcopenshell; print(ifcopenshell.version)"
```

**Important pip notes:**
- The pip package bundles the C++ core and OpenCASCADE dependencies
- Available for Python 3.8-3.12 on Linux, macOS, and Windows
- Wheels are pre-built, no compilation needed
- Some platforms may not have pre-built wheels (e.g., ARM Linux)
- The pip package name is `ifcopenshell` (all lowercase)

### 4.2 conda Installation

```bash
# Using conda-forge channel
conda install -c conda-forge ifcopenshell

# Or in a new environment
conda create -n ifc python=3.11 ifcopenshell -c conda-forge
conda activate ifc

# conda-forge typically has more recent builds than pip
# Also handles OpenCASCADE dependency management better

# Verify
python -c "import ifcopenshell; print(ifcopenshell.version)"
```

**conda advantages:**
- Better dependency resolution (OpenCASCADE, OCE)
- Cross-platform consistency
- Often more up-to-date than pip
- Can install alongside Blender's Python if managed carefully

### 4.3 Building from Source

```bash
# Prerequisites
# Linux (Ubuntu/Debian):
sudo apt-get install git cmake gcc g++ libboost-all-dev \
    libocct-foundation-dev libocct-modeling-data-dev \
    libocct-modeling-algorithms-dev libocct-ocaf-dev \
    libocct-visualization-dev libocct-data-exchange-dev \
    swig python3-dev

# macOS:
brew install cmake boost opencascade swig

# Windows:
# Install Visual Studio Build Tools, CMake, and download OpenCASCADE from
# https://dev.opencascade.org/release

# Clone the repository
git clone --recursive https://github.com/IfcOpenShell/IfcOpenShell.git
cd IfcOpenShell

# Build with CMake
mkdir build && cd build
cmake ../cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DOCC_INCLUDE_DIR=/usr/include/opencascade \
    -DOCC_LIBRARY_DIR=/usr/lib/x86_64-linux-gnu \
    -DPYTHON_EXECUTABLE=$(which python3) \
    -DCOLLADA_SUPPORT=OFF \
    -DBUILD_IFCPYTHON=ON
make -j$(nproc)
make install

# Alternative: Use the provided build scripts
cd IfcOpenShell
# Linux/macOS:
python nix/build-all.py

# The build process is complex. For most users, pip or conda is recommended.
```

### 4.4 Dependencies

| Dependency | Purpose | Required? |
|-----------|---------|-----------|
| OpenCASCADE (OCCT) 7.5+ | Geometry kernel (BRep operations) | YES (for geometry) |
| Boost 1.67+ | C++ utilities | YES (build from source) |
| SWIG 4.0+ | Python bindings generation | YES (build from source) |
| Python 3.8+ | Python runtime | YES |
| CMake 3.16+ | Build system | YES (build from source) |
| lxml (Python) | XML processing | Optional (for some utilities) |
| numpy (Python) | Array operations | Optional (for geometry data) |

### 4.5 Platform-Specific Notes

**Windows:**
- pip install works out-of-the-box with pre-built wheels
- When using with Blender: Blender bundles its own Python; install ifcopenshell into Blender's Python: `<blender_path>/python/bin/python -m pip install ifcopenshell`
- PATH conflicts: If both system Python and Blender Python have ifcopenshell, ensure the correct one is loaded
- Long path issues: Windows has 260-char path limit. Use `\\?\` prefix or enable long paths in Group Policy

**macOS:**
- pip install works for Intel and Apple Silicon (M1/M2/M3)
- OpenCASCADE installed via brew may conflict with bundled version
- On Apple Silicon: Ensure you use the arm64 Python, not Rosetta-emulated x86_64

**Linux:**
- pip install works on most distributions
- For headless servers: ifcopenshell works without display/GPU (unlike Blender)
- Docker: Use `python:3.11-slim` base image, pip install ifcopenshell works directly
- HPC environments: conda is often preferred for consistent dependency management

**Blender Integration:**
```python
# Check if ifcopenshell is available in Blender's Python
import sys
print(sys.executable)  # Shows Blender's Python path

# Install into Blender's Python
# From command line (not from within Blender):
# /path/to/blender/python/bin/python -m pip install ifcopenshell

# Bonsai (formerly BlenderBIM) bundles ifcopenshell
# If Bonsai is installed, ifcopenshell is already available inside Blender
```

---

## 5. Quick Reference: API Method Names (Correct vs Hallucinated)

### Correct API Module Categories:
```
ifcopenshell.api.run("root.*")        - Create/remove entities
ifcopenshell.api.run("spatial.*")     - Spatial containment
ifcopenshell.api.run("aggregate.*")   - Aggregation (decomposition)
ifcopenshell.api.run("type.*")        - Type assignment
ifcopenshell.api.run("pset.*")        - Property sets
ifcopenshell.api.run("geometry.*")    - Geometry operations
ifcopenshell.api.run("material.*")    - Material assignment
ifcopenshell.api.run("context.*")     - Representation contexts
ifcopenshell.api.run("unit.*")        - Unit assignment
ifcopenshell.api.run("owner.*")       - Owner history
ifcopenshell.api.run("attribute.*")   - Attribute editing
ifcopenshell.api.run("void.*")        - Openings/voids
ifcopenshell.api.run("classification.*") - Classification
ifcopenshell.api.run("group.*")       - Grouping
ifcopenshell.api.run("layer.*")       - Presentation layers
ifcopenshell.api.run("style.*")       - Visual styles
ifcopenshell.api.run("structural.*")  - Structural analysis
ifcopenshell.api.run("system.*")      - Building systems
ifcopenshell.api.run("document.*")    - Document references
ifcopenshell.api.run("constraint.*")  - Constraints
ifcopenshell.api.run("cost.*")        - Cost items
ifcopenshell.api.run("sequence.*")    - Scheduling
ifcopenshell.api.run("profile.*")     - Cross-section profiles
ifcopenshell.api.run("nest.*")        - Nesting (ordered composition)
```

### Most Common API Calls with Correct Parameters:
```python
# Create entity
ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001", predefined_type="SOLIDWALL")

# Remove entity
ifcopenshell.api.run("root.remove_product", model, product=wall)

# Spatial containment
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# Aggregation
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Type assignment
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# Property sets
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"IsExternal": True, "FireRating": "REI60"})

# Material
material = ifcopenshell.api.run("material.add_material", model,
    name="Concrete")
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material)

# Geometry context
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=context)

# Units
ifcopenshell.api.run("unit.assign_unit", model)  # Defaults to SI

# Edit attributes
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "New Name", "Description": "A wall"})
```

### Utility Functions (ifcopenshell.util):
```python
import ifcopenshell.util.element
import ifcopenshell.util.unit
import ifcopenshell.util.placement
import ifcopenshell.util.selector
import ifcopenshell.util.date
import ifcopenshell.util.schema
import ifcopenshell.util.cost
import ifcopenshell.util.sequence

# Element utilities
ifcopenshell.util.element.get_psets(element)          # Get property sets
ifcopenshell.util.element.get_type(element)           # Get element type
ifcopenshell.util.element.get_container(element)      # Get spatial container
ifcopenshell.util.element.get_decomposition(element)  # Get children
ifcopenshell.util.element.get_material(element)       # Get material

# Unit utilities
ifcopenshell.util.unit.calculate_unit_scale(model)    # File-to-SI factor
ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")

# Placement utilities
ifcopenshell.util.placement.get_local_placement(placement)  # 4x4 matrix

# Selector (query language)
ifcopenshell.util.selector.filter_elements(model, query_string)
```

---

## 6. Summary of Critical Rules

1. **ALWAYS check model.schema before using schema-specific entities**
2. **ALWAYS use ifcopenshell.guid.new() for GUIDs, NEVER uuid.uuid4()**
3. **ALWAYS create OwnerHistory for IFC2x3 (use API to automate this)**
4. **ALWAYS create relationships via API, not direct attribute assignment**
5. **ALWAYS use ifcopenshell.util.element.get_psets() to read properties**
6. **ALWAYS use ifcopenshell.geom.iterator for batch geometry processing**
7. **ALWAYS check for Representation before calling create_shape()**
8. **ALWAYS account for unit scale (calculate_unit_scale) when reading coordinates**
9. **NEVER use model.save() (use model.write()), model.read() (use ifcopenshell.open())**
10. **NEVER remove entities with model.remove() without cleaning up relationships (use API)**
11. **NEVER assume attributes exist across schema versions without checking**
12. **NEVER mix IfcDoorStyle (IFC2x3) with IfcDoorType (IFC4+) or vice versa**
13. **PREFER ifcopenshell.api.run() over model.create_entity() for correctness**
14. **PREFER ifcopenshell.geom.iterator over create_shape() for large files**
15. **PREFER extracting data to plain dicts and releasing model for memory management**
