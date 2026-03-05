# IfcOpenShell: Error Patterns, Performance & Installation Guide

> Research document for IfcOpenShell pre-study. Covers common error patterns, performance
> optimization for large models, AI code generation pitfalls, installation, and common operations.

---

## Table of Contents

1. [Common Error Patterns](#1-common-error-patterns)
2. [Performance for Large Models](#2-performance-for-large-models)
3. [AI Common Mistakes in IfcOpenShell Code Generation](#3-ai-common-mistakes-in-ifcopenshell-code-generation)
4. [Installation](#4-installation)
5. [Common Operations](#5-common-operations)

---

## 1. Common Error Patterns

### 1.1 Schema Mismatch Errors

**Problem**: Using entities or attributes from one IFC schema version in a file of another version.
IFC2X3, IFC4, and IFC4X3 have different entity definitions, attributes, and type hierarchies.

```python
# ERROR: Using IFC4 entity in IFC2X3 file
import ifcopenshell

model = ifcopenshell.file(schema="IFC2X3")
# IfcBuildingElementProxy exists in IFC2X3, but IfcGeographicElement does NOT
try:
    geo = model.create_entity("IfcGeographicElement")  # Only exists in IFC4+
except RuntimeError as e:
    print(f"Schema error: {e}")
    # RuntimeError: IfcGeographicElement is not a valid IFC class for IFC2X3
```

```python
# FIX: Check schema and use correct entity classes
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")  # Use IFC4 for IfcGeographicElement
geo = model.create_entity("IfcGeographicElement")

# Or check schema at runtime:
schema = model.schema
if schema == "IFC2X3":
    # Use IFC2X3-compatible entities
    element = model.create_entity("IfcBuildingElementProxy")
elif schema in ("IFC4", "IFC4X3"):
    element = model.create_entity("IfcGeographicElement")
```

**Key differences between schemas:**
- `IfcSIUnit.Dimensions` is derived in IFC4 but explicit in IFC2X3
- `IfcOwnerHistory` is mandatory in IFC2X3, optional in IFC4
- IFC4X3 adds alignment and infrastructure entities not in IFC4
- Some entities like `IfcBoiler` exist only in IFC4+

---

### 1.2 Missing Required Attributes (Owner History in IFC2X3)

**Problem**: In IFC2X3, `OwnerHistory` is **mandatory** on all rooted entities. The high-level API
(`ifcopenshell.api.root.create_entity`) will try to auto-create it but fails if user/application
is not set up first.

```python
# ERROR: Creating entities in IFC2X3 without setting up ownership
import ifcopenshell
import ifcopenshell.api.root
import ifcopenshell.api.project

model = ifcopenshell.api.project.create_file(version="IFC2X3")
try:
    project = ifcopenshell.api.root.create_entity(
        model, ifc_class="IfcProject", name="My Project")
except RuntimeError as e:
    print(f"Error: {e}")
    # RuntimeError: Attribute not set - OwnerHistory requires Person/Organization/Application
```

```python
# FIX: Set up ownership data BEFORE creating any entities in IFC2X3
import ifcopenshell
import ifcopenshell.api.project
import ifcopenshell.api.owner
import ifcopenshell.api.root

model = ifcopenshell.api.project.create_file(version="IFC2X3")

# Step 1: Create person and organization
person = ifcopenshell.api.owner.add_person(model,
    identification="JDoe", family_name="Doe", given_name="John")
org = ifcopenshell.api.owner.add_organisation(model,
    identification="MyOrg", name="My Organization")
ifcopenshell.api.owner.add_person_and_organisation(model,
    person=person, organisation=org)

# Step 2: Create application
application = ifcopenshell.api.owner.add_application(model)

# Step 3: NOW you can create entities
project = ifcopenshell.api.root.create_entity(
    model, ifc_class="IfcProject", name="My Project")
```

**Note**: In IFC4, owner history is **optional** and not recommended unless legally required.

---

### 1.3 Incorrect Relationship Creation

**Problem**: Manually creating relationship entities without proper inverse attributes, or using
wrong relationship types.

```python
# ERROR: Manually creating IfcRelContainedInSpatialStructure incorrectly
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]
storey = model.by_type("IfcBuildingStorey")[0]

# WRONG: Manually creating relationship without proper setup
rel = model.create_entity("IfcRelContainedInSpatialStructure")
rel.RelatingStructure = storey
rel.RelatedElements = (wall,)
# Missing GlobalId, OwnerHistory (IFC2X3), and may create duplicate relationships
```

```python
# FIX: Use the high-level API for relationship management
import ifcopenshell
import ifcopenshell.api.spatial

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]
storey = model.by_type("IfcBuildingStorey")[0]

# Correct: API handles GlobalId, deduplication, and inverse relationships
ifcopenshell.api.spatial.assign_container(
    model, relating_structure=storey, products=[wall])

# For aggregation (e.g., site->building->storey):
import ifcopenshell.api.aggregate
ifcopenshell.api.aggregate.assign_object(
    model, relating_object=building, products=[storey])
```

**Common relationship mistakes:**
- Using `IfcRelContainedInSpatialStructure` when `IfcRelAggregates` is needed (and vice versa)
- An element can only be in ONE spatial container but LLMs often forget to check/remove existing
- Forgetting that aggregation is for spatial hierarchy (Project→Site→Building→Storey)
- Forgetting that containment is for elements within a storey

---

### 1.4 Geometry Processing Failures

**Problem**: Geometry processing can fail due to invalid geometry, missing representations, or
incorrect settings.

```python
# ERROR: Processing geometry without proper settings
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

settings = ifcopenshell.geom.settings()
try:
    shape = ifcopenshell.geom.create_shape(settings, wall)
except RuntimeError as e:
    print(f"Geometry error: {e}")
    # Can fail if: wall has no representation, invalid boolean operations,
    # missing placement, or corrupted geometry definitions
```

```python
# FIX: Handle geometry processing robustly
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")

settings = ifcopenshell.geom.settings()
# Optional: disable booleans for faster but less accurate geometry
# settings.set("disable-boolean-result", True)

for wall in model.by_type("IfcWall"):
    # Check if element has geometry before processing
    if not wall.Representation:
        print(f"Skipping {wall.Name} - no geometry representation")
        continue

    try:
        shape = ifcopenshell.geom.create_shape(settings, wall)
        verts = shape.geometry.verts  # Flat list: [x1,y1,z1, x2,y2,z2, ...]
        faces = shape.geometry.faces  # Flat list of triangle indices
    except RuntimeError as e:
        print(f"Failed to process {wall.GlobalId}: {e}")
        continue
```

**Common geometry failures:**
- `IfcFaceBasedSurfaceModel` with millions of triangles consuming excessive memory
- Boolean operations failing on malformed solids
- Missing `ObjectPlacement` on elements with representations
- OpenCASCADE vs CGAL geometry engine producing different results

---

### 1.5 Unit Conversion Issues

**Problem**: IFC files can use different unit systems. Coordinates are stored in **project units**,
not necessarily SI meters. Ignoring unit scale leads to wrong dimensions.

```python
# ERROR: Assuming coordinates are always in meters
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# WRONG: Raw coordinates may be in millimeters, feet, inches, etc.
import ifcopenshell.util.placement
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
x, y, z = matrix[:, 3][:3]
print(f"Position: ({x}, {y}, {z})")  # Could be in mm, not meters!
```

```python
# FIX: Always calculate and apply unit scale
import ifcopenshell
import ifcopenshell.util.unit
import ifcopenshell.util.placement

model = ifcopenshell.open("model.ifc")

# Calculate the conversion factor to SI meters
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

wall = model.by_type("IfcWall")[0]
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
x, y, z = matrix[:, 3][:3]

# Convert to SI meters
x_meters = x * unit_scale
y_meters = y * unit_scale
z_meters = z * unit_scale
print(f"Position in meters: ({x_meters}, {y_meters}, {z_meters})")

# To convert FROM SI meters TO project units:
project_length = si_meters / unit_scale
```

**Common unit pitfalls:**
- Imperial files using feet/inches while code assumes metric
- Millimeter-based files (common in some regions) treated as meter values
- Area and volume units need squared/cubed scale factors
- Angular units may be degrees or radians

---

### 1.6 GUID-Related Errors

**Problem**: IFC uses a special 22-character base64-encoded GUID format (not standard UUID).
Incorrect GUID generation causes validation failures and entity duplication.

```python
# ERROR: Using standard UUID or invalid GUID format
import uuid
import ifcopenshell

model = ifcopenshell.file()
# WRONG: Standard UUID is not valid IFC GUID format
wall = model.create_entity("IfcWall", GlobalId=str(uuid.uuid4()))
# IFC GUIDs must be exactly 22 characters in base64 encoding

# WRONG: Duplicate GUIDs
guid = ifcopenshell.guid.new()
wall1 = model.create_entity("IfcWall", GlobalId=guid)
wall2 = model.create_entity("IfcWall", GlobalId=guid)  # Duplicate!
```

```python
# FIX: Use IfcOpenShell's built-in GUID generation
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file()

# Correct: Generate proper IFC-format GUID
guid = ifcopenshell.guid.new()
print(f"IFC GUID: {guid}")  # e.g., "3Oe$oOoSz4MQF2KwHObnqN"
print(f"Length: {len(guid)}")  # Always 22

# Best: Use high-level API which auto-generates GUIDs
import ifcopenshell.api.root
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
# GlobalId is automatically generated and guaranteed unique

# Convert between UUID and IFC GUID if needed:
standard_uuid = ifcopenshell.guid.expand(guid)     # To standard UUID
ifc_guid = ifcopenshell.guid.compress(standard_uuid)  # To IFC GUID
```

---

### 1.7 Property Set Errors

**Problem**: Incorrectly creating or editing property sets, using wrong property value types,
or creating orphaned property sets not linked to any element.

```python
# ERROR: Creating property set manually with wrong structure
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# WRONG: Creating property set without linking it to the element
pset = model.create_entity("IfcPropertySet",
    Name="Pset_WallCommon")
prop = model.create_entity("IfcPropertySingleValue",
    Name="IsExternal",
    NominalValue=model.create_entity("IfcBoolean", True))
pset.HasProperties = [prop]
# Property set exists but is NOT connected to the wall!
```

```python
# FIX: Use the high-level API for property set management
import ifcopenshell
import ifcopenshell.api.pset

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Create and link property set to element in one step
pset = ifcopenshell.api.pset.add_pset(model, product=wall,
    name="Pset_WallCommon")

# Edit properties - API handles value type inference
ifcopenshell.api.pset.edit_pset(model, pset=pset, properties={
    "IsExternal": True,           # Automatically creates IfcBoolean
    "ThermalTransmittance": 0.35, # Automatically creates IfcReal
    "Reference": "W-001",         # Automatically creates IfcLabel
})

# Reading properties back:
import ifcopenshell.util.element
psets = ifcopenshell.util.element.get_psets(wall)
print(psets["Pset_WallCommon"]["IsExternal"])  # True
```

**Common property set mistakes:**
- Forgetting to create `IfcRelDefinesByProperties` to link pset to element
- Using wrong nominal value types (e.g., `IfcLabel` where `IfcIdentifier` is expected)
- CamelCase sensitivity: `quantity.LengthValue` not `quantity.lengthValue`
- Trying to add properties to type vs instance (use `get_psets(element, psets_only=True)`)

---

### 1.8 Type Assignment Errors

**Problem**: Incorrectly assigning or querying element types, confusing type vs occurrence.

```python
# ERROR: Trying to assign type incorrectly
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]
wall_type = model.by_type("IfcWallType")[0]

# WRONG: Direct attribute assignment doesn't create the relationship
wall.IsTypedBy = wall_type  # This is an inverse attribute, not settable!
```

```python
# FIX: Use API for type assignment
import ifcopenshell
import ifcopenshell.api.type

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]
wall_type = model.by_type("IfcWallType")[0]

# Correct: Use API to create IfcRelDefinesByType relationship
ifcopenshell.api.type.assign_type(model, related_objects=[wall],
    relating_type=wall_type)

# Query type correctly:
import ifcopenshell.util.element
element_type = ifcopenshell.util.element.get_type(wall)
print(element_type.Name if element_type else "No type assigned")

# Get all occurrences of a type:
occurrences = ifcopenshell.util.element.get_types(wall_type)
```

---

### 1.9 File Corruption Patterns

**Problem**: Files can become corrupted through improper entity deletion, dangling references,
or incomplete writes.

```python
# ERROR: Deleting entities leaving dangling references
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# WRONG: Low-level removal leaves dangling references in relationships
model.remove(wall)
# Any IfcRelContainedInSpatialStructure, IfcRelDefinesByProperties, etc.
# still reference this deleted entity -> corrupted file
```

```python
# FIX: Use high-level API for safe entity removal
import ifcopenshell
import ifcopenshell.api.root

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Correct: API cleans up all relationships and references
ifcopenshell.api.root.remove_product(model, product=wall)

# For batch removal, collect all elements first:
walls_to_remove = model.by_type("IfcWall")
for wall in list(walls_to_remove):  # list() to avoid modification during iteration
    ifcopenshell.api.root.remove_product(model, product=wall)

# Always validate after significant modifications:
model.write("output.ifc")
```

**File corruption causes:**
- Removing entities with `model.remove()` instead of API methods
- Modifying entity IDs after creation
- Writing to file during iteration
- Not properly closing/flushing file handles
- Circular references in entity graphs

---

## 2. Performance for Large Models

### 2.1 Memory Usage Patterns

IfcOpenShell loads the **entire IFC file into memory** when using `ifcopenshell.open()`. For large
files (100MB+), this can consume significant RAM:

| File Size | Approximate RAM Usage | Notes |
|-----------|----------------------|-------|
| 10 MB     | ~100-200 MB          | Comfortable on most systems |
| 100 MB    | ~1-2 GB              | Watch memory on 8GB systems |
| 500 MB    | ~5-10 GB             | Requires 16GB+ RAM |
| 1+ GB     | ~10-20+ GB           | May need specialized strategies |
| 2+ GB     | 30+ GB               | Risk of OOM on standard hardware |

**Factors affecting memory:**
- Duplicate entity instances (common in poorly optimized files)
- Geometry processing adds 2-5x memory on top of file loading
- Multi-threaded geometry iterator multiplies memory per thread
- OpenCASCADE geometry engine uses more memory than CGAL

### 2.2 Efficient Querying Strategies

```python
import ifcopenshell

model = ifcopenshell.open("large_model.ifc")

# FAST: by_type() uses internal class index - O(1) lookup
walls = model.by_type("IfcWall")

# FAST: by_id() uses internal ID map - O(1) lookup
entity = model.by_id(12345)

# FAST: by_guid() uses GUID index
entity = model.by_guid("3Oe$oOoSz4MQF2KwHObnqN")

# SLOW: Iterating all entities and filtering
# DON'T DO THIS for large files:
walls_slow = [e for e in model if e.is_a("IfcWall")]  # Iterates ALL entities

# EFFICIENT: Use utility functions instead of manual traversal
import ifcopenshell.util.element

# Get all elements in a storey (uses inverse relationships)
storey = model.by_type("IfcBuildingStorey")[0]
elements = ifcopenshell.util.element.get_decomposition(storey)

# Get properties without traversing all relationships manually
psets = ifcopenshell.util.element.get_psets(wall)
```

### 2.3 Batch vs Individual Operations

```python
import ifcopenshell
import ifcopenshell.api.pset
import ifcopenshell.api.spatial

model = ifcopenshell.open("large_model.ifc")

# INEFFICIENT: Individual container assignment creates separate relationships
storey = model.by_type("IfcBuildingStorey")[0]
walls = model.by_type("IfcWall")
for wall in walls:
    ifcopenshell.api.spatial.assign_container(
        model, relating_structure=storey, products=[wall])
    # Creates N separate IfcRelContainedInSpatialStructure entities!

# EFFICIENT: Batch assignment uses a single relationship entity
ifcopenshell.api.spatial.assign_container(
    model, relating_structure=storey, products=list(walls))
# Creates ONE IfcRelContainedInSpatialStructure with all walls

# EFFICIENT: Batch property editing
for wall in model.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    if "Pset_WallCommon" in psets:
        # Get the actual pset entity for editing
        for rel in wall.IsDefinedBy:
            if hasattr(rel, "RelatingPropertyDefinition"):
                pdef = rel.RelatingPropertyDefinition
                if pdef.is_a("IfcPropertySet") and pdef.Name == "Pset_WallCommon":
                    ifcopenshell.api.pset.edit_pset(model, pset=pdef,
                        properties={"IsExternal": True})
```

### 2.4 Geometry Processing Optimization

```python
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")

# SETTINGS FOR PERFORMANCE:
settings = ifcopenshell.geom.settings()

# Disable features you don't need:
# settings.set("disable-boolean-result", True)  # Skip booleans (faster)
# settings.set("apply-default-materials", False)

# USE ITERATOR for bulk geometry processing (much faster than create_shape loop)
iterator = ifcopenshell.geom.iterator(
    settings,
    model,
    multiprocessing.cpu_count()  # Use all CPU cores
)

# Optional: filter to specific types only
# iterator = ifcopenshell.geom.iterator(
#     settings, model, multiprocessing.cpu_count(),
#     include=model.by_type("IfcWall")
# )

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        verts = shape.geometry.verts
        faces = shape.geometry.faces
        # Process geometry...
        if not iterator.next():
            break
```

### 2.5 `ifcopenshell.geom.iterator` vs `create_shape`

| Feature | `create_shape()` | `geom.iterator` |
|---------|-----------------|-----------------|
| **Use case** | Single element | Bulk processing |
| **Multi-threading** | No | Yes (multi-core) |
| **Geometry caching** | No | Yes (reuses identical geometry) |
| **Memory efficiency** | Lower overhead per call | Better for many elements |
| **Filtering** | Process any entity | `include` parameter for type filtering |
| **Error handling** | Exception per element | Skips failed elements automatically |
| **Speed for 1000+ elements** | Slow (sequential) | 5-10x faster (parallel + caching) |

```python
# create_shape: Good for single/few elements
shape = ifcopenshell.geom.create_shape(settings, single_wall)

# iterator: Required for large files
iterator = ifcopenshell.geom.iterator(settings, model, num_threads)
```

### 2.6 Memory Management for 100MB+ Files

```python
import ifcopenshell
import ifcopenshell.geom
import gc

# Strategy 1: Process in chunks using iterator with type filtering
model = ifcopenshell.open("huge_model.ifc")

element_types = ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam"]
for etype in element_types:
    elements = model.by_type(etype)
    if not elements:
        continue

    settings = ifcopenshell.geom.settings()
    iterator = ifcopenshell.geom.iterator(settings, model, 4,
        include=elements)

    if iterator.initialize():
        while True:
            shape = iterator.get()
            # Process and store results, release shape data
            process_and_store(shape)
            if not iterator.next():
                break

    # Force garbage collection between types
    gc.collect()

# Strategy 2: Reduce thread count for memory-constrained systems
# More threads = faster but uses MORE memory (up to 2-3x with 32 threads)
# Use fewer threads on memory-limited machines:
iterator = ifcopenshell.geom.iterator(settings, model, 2)  # 2 threads

# Strategy 3: Use CGAL geometry engine (less memory than OpenCASCADE)
# Requires CGAL-enabled build of IfcOpenShell

# Strategy 4: For data extraction only (no geometry), skip geometry entirely
model = ifcopenshell.open("huge_model.ifc")
for wall in model.by_type("IfcWall"):
    # Extract properties without processing geometry
    psets = ifcopenshell.util.element.get_psets(wall)
    # Much lower memory than geometry processing

# Strategy 5: Process-level isolation for very large files
# Fork a subprocess for geometry processing, kill it when done
# to reclaim ALL memory (Python GC may not release everything)
import subprocess
import json

result = subprocess.run(
    ["python", "process_geometry.py", "huge_model.ifc"],
    capture_output=True, text=True
)
data = json.loads(result.stdout)
```

**Advanced strategies from community:**
- **SQLite conversion**: Convert IFC to SQLite for ~25% memory usage at cost of speed
- **Lazy loading**: Stream/seek instead of loading everything into memory (experimental)
- **File optimization**: Remove duplicate entities to reduce file/memory size (use IfcPatch)
- **Disable inverse maps**: When not querying inverse relationships

---

## 3. AI Common Mistakes in IfcOpenShell Code Generation

### 3.1 Hallucinated API Methods

LLMs frequently generate plausible-looking but **non-existent** methods:

```python
# HALLUCINATED (these methods DO NOT EXIST):
model.get_all_walls()                    # No such method
model.add_wall(name="W1")               # No such shortcut
ifcopenshell.create_project()            # Not a top-level function
wall.get_properties()                    # Not a method on entity_instance
wall.set_property("Name", "Wall 1")     # Not how you set properties
ifcopenshell.api.run("wall.create", ...) # Not a valid API domain
ifcopenshell.api.geometry.create_wall()  # Not the actual method name
model.get_spatial_structure()            # Not a method

# CORRECT equivalents:
model.by_type("IfcWall")                                              # Get walls
ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")      # Create wall
ifcopenshell.api.project.create_file()                                 # Create project
ifcopenshell.util.element.get_psets(wall)                             # Get properties
ifcopenshell.api.attribute.edit_attributes(model, product=wall,
    attributes={"Name": "Wall 1"})                                    # Set attributes
ifcopenshell.api.geometry.add_wall_representation(model, ...)         # Wall geometry
```

### 3.2 Wrong Parameter Orders

```python
# WRONG: Parameters in incorrect order or wrong names
ifcopenshell.api.spatial.assign_container(
    model, products=[wall], relating_structure=storey)
# This may work with kwargs but positional ordering matters

# WRONG: Old-style api.run() with incorrect keyword arguments
ifcopenshell.api.run("spatial.assign_container", model,
    element=wall, container=storey)  # Wrong parameter names!

# CORRECT (modern API style):
ifcopenshell.api.spatial.assign_container(
    model, relating_structure=storey, products=[wall])

# CORRECT (deprecated api.run style, still seen in old examples):
# ifcopenshell.api.run("spatial.assign_container", model,
#     relating_structure=storey, products=[wall])
```

### 3.3 Confusing `api.run()` with Direct Entity Creation

```python
# WRONG: Mixing high-level API and low-level creation incorrectly
import ifcopenshell

model = ifcopenshell.file()

# Low-level: creates entity but NO GlobalId, NO OwnerHistory, NO relationships
wall = model.create_entity("IfcWall")
wall.Name = "My Wall"
# This wall has no GlobalId! Not a valid IFC entity.

# WRONG: Using api.run (deprecated)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall")

# CORRECT: Use module-level API functions directly
import ifcopenshell.api.root
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall",
    name="My Wall")
# Automatically generates GlobalId and handles OwnerHistory
```

**Key rule**: `ifcopenshell.api.run()` is **deprecated**. Use direct module calls:
- Old: `ifcopenshell.api.run("root.create_entity", model, ...)`
- New: `ifcopenshell.api.root.create_entity(model, ...)`

### 3.4 Schema Version Assumptions

```python
# WRONG: LLMs often default to IFC4 without checking
model = ifcopenshell.file()  # Defaults to IFC4
# But user's existing file might be IFC2X3!

# WRONG: Using IFC4-only features without checking
existing_model = ifcopenshell.open("legacy_file.ifc")
# LLM assumes IFC4, but file is IFC2X3
ifcopenshell.api.root.create_entity(existing_model, ifc_class="IfcWall",
    predefined_type="SOLIDWALL")  # PredefinedType enum may differ

# CORRECT: Always check and respect the file's schema
existing_model = ifcopenshell.open("some_file.ifc")
print(f"Schema: {existing_model.schema}")  # "IFC2X3", "IFC4", or "IFC4X3"

# Create new file matching an existing file's schema:
new_model = ifcopenshell.api.project.create_file(
    version=existing_model.schema)
```

### 3.5 Missing Owner History (IFC2X3)

Already covered in detail in Section 1.2. Key points LLMs miss:
- IFC2X3 requires `IfcOwnerHistory` on ALL rooted entities
- Must create `IfcPerson`, `IfcOrganization`, `IfcPersonAndOrganization`, and `IfcApplication` first
- IFC4 makes `OwnerHistory` optional — LLMs trained on IFC4 examples forget IFC2X3 requirements
- The high-level API handles this IF ownership is set up first

### 3.6 Incorrect Geometry Creation

```python
# WRONG: LLMs often try to create geometry entities directly
import ifcopenshell

model = ifcopenshell.file()
# Attempting to manually build geometry (error-prone and verbose)
point1 = model.create_entity("IfcCartesianPoint", Coordinates=(0.0, 0.0, 0.0))
point2 = model.create_entity("IfcCartesianPoint", Coordinates=(5.0, 0.0, 0.0))
line = model.create_entity("IfcPolyline", Points=[point1, point2])
# ... 30+ more lines of manual geometry entity creation
# Easy to get wrong: missing contexts, wrong representation types, etc.

# CORRECT: Use high-level geometry API
import ifcopenshell.api.geometry
import ifcopenshell.api.context
import ifcopenshell.api.root

model = ifcopenshell.api.project.create_file()
project = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")
ifcopenshell.api.unit.assign_unit(model)

# Create representation context FIRST (often forgotten by LLMs!)
context = ifcopenshell.api.context.add_context(model, context_type="Model")
body = ifcopenshell.api.context.add_context(model, context_type="Model",
    context_identifier="Body", target_view="MODEL_VIEW", parent=context)

wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")

# Set placement
ifcopenshell.api.geometry.edit_object_placement(model, product=wall)

# Create wall geometry (5m long, 3m high, 0.2m thick)
representation = ifcopenshell.api.geometry.add_wall_representation(
    model, context=body, length=5, height=3, thickness=0.2)

# Assign representation to wall
ifcopenshell.api.geometry.assign_representation(
    model, product=wall, representation=representation)
```

### 3.7 Other Common AI Mistakes

| Mistake | Example | Fix |
|---------|---------|-----|
| Wrong import paths | `from ifcopenshell import api` | `import ifcopenshell.api.root` |
| CamelCase errors | `wall.name`, `quantity.lengthValue` | `wall.Name`, `quantity.LengthValue` |
| Wrong util module | `ifcopenshell.util.get_psets()` | `ifcopenshell.util.element.get_psets()` |
| Forgetting unit setup | Creating geometry without `assign_unit` | Always call `ifcopenshell.api.unit.assign_unit(model)` |
| Forgetting context | Creating representations without context | Create `Model/Body/MODEL_VIEW` context first |
| Inverse attribute writes | `wall.ContainedInStructure = storey` | Use `spatial.assign_container()` API |
| String vs entity | `pset.HasProperties = ["prop1"]` | Must be entity instances, not strings |
| Missing file model arg | `ifcopenshell.api.root.create_entity(ifc_class="IfcWall")` | First arg must be `model` |

---

## 4. Installation

### 4.1 pip (Recommended)

```bash
# Install from PyPI (wheels available for Windows, Linux, macOS)
pip install ifcopenshell

# Specific version
pip install ifcopenshell==0.8.4

# Verify installation
python -c "import ifcopenshell; print(ifcopenshell.version)"
```

**Supported platforms (as of 0.8.4):**
- Python 3.9 - 3.14
- Linux (x86_64, aarch64 via conda)
- Windows (x86_64)
- macOS (Intel and Apple Silicon)

### 4.2 Conda

```bash
# From conda-forge (recommended, also installs IfcConvert)
conda install -c conda-forge ifcopenshell

# Create dedicated environment (recommended for OpenCASCADE compatibility)
conda create -n ifc -c conda-forge ifcopenshell pythonocc-core
conda activate ifc

# From IfcOpenShell channel (daily builds)
conda install -c ifcopenshell ifcopenshell
```

**Note**: If using other packages that depend on OpenCASCADE (occt), install them
simultaneously to ensure version compatibility.

### 4.3 Source Build

```bash
# Clone repository
git clone https://github.com/IfcOpenShell/IfcOpenShell.git
cd IfcOpenShell

# The Python package is in src/ifcopenshell-python/
# Add to Python path or copy to site-packages:
cp -r src/ifcopenshell-python/ifcopenshell $(python -c "import site; print(site.getsitepackages()[0])")/

# Download precompiled C++ binaries from IfcOpenShell Build Service
# for your platform, or build from source with CMake
```

### 4.4 Docker

```bash
docker run -it aecgeeks/ifcopenshell python3 -c 'import ifcopenshell; print(ifcopenshell.version)'
```

### 4.5 Other Platforms

- **Google Colab**: Pre-configured notebook available
- **WebAssembly**: Experimental pyodide support via `wasm-wheels` repository
- **Linux Distros**: Available in Arch Linux (AUR), Fedora (COPR), Ubuntu PPAs
- **Bonsai/BlenderBIM**: Includes IfcOpenShell with Blender-based GUI

### 4.6 Dependencies & Troubleshooting

**Core dependencies:**
- OpenCASCADE (occt) - geometry kernel, bundled in pip/conda packages
- Python C extensions - platform-specific compiled binaries

**Common installation issues:**
- `ModuleNotFoundError: No module named 'ifcopenshell.api'` → Reinstall matching your Python version
- `ImportError: libpython3.x.so` → Missing shared library (common in Docker/containers)
- Python version mismatch → Ensure pip/conda installs match your Python version
- OpenCASCADE conflicts → Use dedicated conda environment

---

## 5. Common Operations

### 5.1 Create a Minimal Valid IFC File

```python
import ifcopenshell
import ifcopenshell.api.root
import ifcopenshell.api.unit
import ifcopenshell.api.context
import ifcopenshell.api.project
import ifcopenshell.api.spatial
import ifcopenshell.api.geometry
import ifcopenshell.api.aggregate

# 1. Create file and project
model = ifcopenshell.api.project.create_file()
project = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcProject", name="My Project")

# 2. Assign units (metric by default)
ifcopenshell.api.unit.assign_unit(model)

# 3. Create geometry context
context = ifcopenshell.api.context.add_context(model, context_type="Model")
body = ifcopenshell.api.context.add_context(model, context_type="Model",
    context_identifier="Body", target_view="MODEL_VIEW", parent=context)

# 4. Create spatial hierarchy
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey])

# 5. Create a wall with geometry
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall",
    name="My Wall")
ifcopenshell.api.geometry.edit_object_placement(model, product=wall)
representation = ifcopenshell.api.geometry.add_wall_representation(
    model, context=body, length=5, height=3, thickness=0.2)
ifcopenshell.api.geometry.assign_representation(model,
    product=wall, representation=representation)

# 6. Place wall in storey
ifcopenshell.api.spatial.assign_container(model,
    relating_structure=storey, products=[wall])

# 7. Write file
model.write("minimal_model.ifc")
```

### 5.2 Extract Data from an Existing IFC File

```python
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.placement
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# Basic file info
print(f"Schema: {model.schema}")
print(f"Total entities: {len(list(model))}")

# Get unit scale for coordinate conversion
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

# Extract all walls with properties and location
for wall in model.by_type("IfcWall"):
    print(f"\n--- {wall.Name} (GUID: {wall.GlobalId}) ---")

    # Get type
    wall_type = ifcopenshell.util.element.get_type(wall)
    if wall_type:
        print(f"  Type: {wall_type.Name}")

    # Get spatial container
    container = ifcopenshell.util.element.get_container(wall)
    if container:
        print(f"  In: {container.Name}")

    # Get placement (convert to meters)
    if wall.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            wall.ObjectPlacement)
        x, y, z = matrix[:, 3][:3] * unit_scale
        print(f"  Position: ({x:.2f}, {y:.2f}, {z:.2f}) m")

    # Get all properties
    psets = ifcopenshell.util.element.get_psets(wall)
    for pset_name, props in psets.items():
        print(f"  {pset_name}:")
        for prop_name, value in props.items():
            if prop_name != "id":
                print(f"    {prop_name}: {value}")
```

### 5.3 Modify Properties

```python
import ifcopenshell
import ifcopenshell.api.pset
import ifcopenshell.api.attribute
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")

for wall in model.by_type("IfcWall"):
    # Modify element attributes
    ifcopenshell.api.attribute.edit_attributes(model,
        product=wall, attributes={"Name": f"Wall-{wall.GlobalId[:8]}"})

    # Add/modify property set
    psets = ifcopenshell.util.element.get_psets(wall)
    if "Pset_WallCommon" in psets:
        # Find existing pset entity
        for rel in wall.IsDefinedBy:
            if hasattr(rel, "RelatingPropertyDefinition"):
                pdef = rel.RelatingPropertyDefinition
                if pdef.is_a("IfcPropertySet") and pdef.Name == "Pset_WallCommon":
                    ifcopenshell.api.pset.edit_pset(model, pset=pdef,
                        properties={"IsExternal": True, "Reference": "EXT-01"})
    else:
        # Create new property set
        pset = ifcopenshell.api.pset.add_pset(model, product=wall,
            name="Pset_WallCommon")
        ifcopenshell.api.pset.edit_pset(model, pset=pset,
            properties={"IsExternal": False, "Reference": "INT-01"})

model.write("modified_model.ifc")
```

### 5.4 Geometry Processing (Extract Mesh Data)

```python
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("model.ifc")

settings = ifcopenshell.geom.settings()

# For bulk processing, use iterator:
iterator = ifcopenshell.geom.iterator(settings, model,
    multiprocessing.cpu_count())

results = []
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)

        verts = shape.geometry.verts  # [x1,y1,z1, x2,y2,z2, ...]
        faces = shape.geometry.faces  # [i1,i2,i3, i4,i5,i6, ...]

        # Convert flat lists to coordinate tuples
        vertices = [(verts[i], verts[i+1], verts[i+2])
                     for i in range(0, len(verts), 3)]
        triangles = [(faces[i], faces[i+1], faces[i+2])
                      for i in range(0, len(faces), 3)]

        results.append({
            "guid": element.GlobalId,
            "name": element.Name,
            "type": element.is_a(),
            "num_vertices": len(vertices),
            "num_triangles": len(triangles),
        })

        if not iterator.next():
            break

print(f"Processed {len(results)} elements")
```

---

## Sources

- [IfcOpenShell Documentation](https://docs.ifcopenshell.org/)
- [IfcOpenShell Academy](https://academy.ifcopenshell.org/)
- [IfcOpenShell GitHub](https://github.com/IfcOpenShell/IfcOpenShell)
- [OSArch Community](https://community.osarch.org/)
- [IfcOpenShell PyPI](https://pypi.org/project/ifcopenshell/)
- [GitHub Issue #4340 - IFC2x3 API compatibility](https://github.com/IfcOpenShell/IfcOpenShell/issues/4340)
- [GitHub Issue #6905 - Memory usage with geometry iterator](https://github.com/IfcOpenShell/IfcOpenShell/issues/6905)
- [GitHub Issue #2025 - Large IFC dataset strategies](https://github.com/IfcOpenShell/IfcOpenShell/issues/2025)
- [GitHub Issue #5026 - Slow file opening](https://github.com/IfcOpenShell/IfcOpenShell/issues/5026)
