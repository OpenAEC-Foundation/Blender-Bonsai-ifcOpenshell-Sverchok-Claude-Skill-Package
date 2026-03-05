# IfcOpenShell Pre-Research Document (Vooronderzoek)

**Date:** 2026-03-05
**Subject:** Comprehensive IfcOpenShell library research for Claude Skill Package development

---

## Table of Contents

1. [IfcOpenShell Overview](#1-ifcopenshell-overview)
2. [File I/O Operations](#2-file-io-operations)
3. [ifcopenshell.api.run() Complete Category Overview](#3-ifcopenshellapirun-complete-category-overview)
4. [Element Traversal](#4-element-traversal)
5. [ifcopenshell.util.* Modules](#5-ifcopenshellutil-modules)
6. [ifcopenshell.geom Module](#6-ifcopenshellgeom-module)
7. [IFC Schema Versions](#7-ifc-schema-versions)
8. [Entity Hierarchy and Inheritance](#8-entity-hierarchy-and-inheritance)
9. [Relationship Types](#9-relationship-types)
10. [Common Operations](#10-common-operations)
11. [Common Error Patterns](#11-common-error-patterns)
12. [Performance Considerations](#12-performance-considerations)
13. [AI Common Mistakes](#13-ai-common-mistakes)
14. [Sources](#14-sources)

---

## 1. IfcOpenShell Overview

### What is IfcOpenShell?

IfcOpenShell is an open-source library for reading, writing, and manipulating IFC (Industry Foundation Classes) files. IFC is the ISO 16739 standard for Building Information Modeling (BIM) data exchange. IfcOpenShell provides both a C++ core engine (with OpenCASCADE geometry backend) and Python bindings, enabling developers to work with IFC models programmatically.

**Key Statistics:**
- **35 API modules** covering all major BIM domains
- **29 utility submodules** for common data operations
- **Multiple IFC versions supported**: IFC2X3, IFC4, IFC4X3
- **Extensive geometry support** including parametric shapes, boolean operations, and tessellation
- **Production-ready** with active maintenance (v0.8.4 current)
- **LGPL-3.0-or-later license** for open-source use

### Library Components

| Component | Purpose |
|---|---|
| `ifcopenshell` | Core Python module -- file I/O, entity access, schema introspection |
| `ifcopenshell.api` | High-level API with `run()` function for safe IFC modifications |
| `ifcopenshell.util` | Utility modules for element info, units, placement, selectors, dates |
| `ifcopenshell.geom` | Geometry processing engine (OpenCASCADE-based) |
| `ifcopenshell.express` | EXPRESS schema parser and introspection |
| `ifcopenshell.validate` | IFC file validation |
| `ifcopenshell.template` | Minimal IFC file templates |

### Primary Use Cases

- **BIM Model Analysis**: Extract properties, geometry, and relationships
- **Automated Model Generation**: Create IFC models programmatically
- **Geometric Processing**: Convert geometry for visualization or analysis
- **Data Migration**: Transform between IFC versions and external formats
- **Quality Assurance**: Validate models against standards
- **Integration**: Connect BIM data with external systems

### Installation

```bash
# Preferred: pip install
pip install ifcopenshell

# Alternative: conda (includes OpenCASCADE dependencies)
conda install -c conda-forge ifcopenshell

# Verify installation
python -c "import ifcopenshell; print(ifcopenshell.version)"
```

**IMPORTANT:** The pip package includes pre-built binaries. If geometry processing fails, ALWAYS verify that the OpenCASCADE backend is properly linked. On Linux, `ldd` the `.so` file to check.

### System Requirements

- **Python 3.8+** (3.10+ recommended)
- **Operating Systems**: Windows, macOS, Linux
- **Memory**: 4GB minimum (8GB+ for large models)
- **Disk Space**: ~500MB for full installation

---

## 2. File I/O Operations

### Opening an IFC File

```python
import ifcopenshell

# Open an existing file
model = ifcopenshell.open("building.ifc")

# The model object is an ifcopenshell.file instance
print(type(model))  # <class 'ifcopenshell.file'>
print(model.schema)  # 'IFC4', 'IFC2X3', or 'IFC4X3'
```

### Creating a New IFC File

```python
# Method 1: Create empty file with specific schema
model = ifcopenshell.file(schema="IFC4")

# Method 2: Create from template (includes required header and IfcProject)
model = ifcopenshell.template.create(schema="IFC4")

# Method 3: Use the API (recommended - sets up project structure)
model = ifcopenshell.api.run("project.create_file", version="IFC4")
```

**NEVER** use `ifcopenshell.file()` without a schema argument -- it defaults to IFC4 but this is implicit and error-prone. ALWAYS specify the schema explicitly.

### Writing an IFC File

```python
# Write to disk
model.write("output.ifc")

# Get IFC string content
ifc_string = model.to_string()
```

### Creating Entities Directly (Low-Level)

```python
# Low-level entity creation -- use ifcopenshell.api.run() instead when possible
wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new(), Name="My Wall")

# Alternative: createIfcWall shorthand
wall = model.createIfcWall(
    GlobalId=ifcopenshell.guid.new(),
    Name="My Wall",
    OwnerHistory=owner_history
)
```

**IMPORTANT:** Direct `create_entity()` does NOT create required relationships (spatial containment, type assignment, property sets). ALWAYS prefer `ifcopenshell.api.run()` for creating elements because it handles relationships automatically.

### The file() Object Key Methods

| Method | Purpose |
|---|---|
| `model.by_type("IfcWall")` | Get all entities of a type (returns tuple) |
| `model.by_id(42)` | Get entity by integer step ID |
| `model.by_guid("2O2Fr$...")` | Get entity by GlobalId |
| `model.create_entity(type, **kwargs)` | Create new entity (low-level) |
| `model.remove(entity)` | Remove entity from model (low-level, no cleanup) |
| `model.traverse(entity)` | Get all referenced entities recursively |
| `model.get_inverse(entity)` | Get entities referencing this entity |
| `model.schema` | Return schema identifier string |
| `model.write(path)` | Write to file |
| `model.to_string()` | Get IFC content as string |

---

## 3. ifcopenshell.api.run() Complete Category Overview

The `ifcopenshell.api.run()` function is the **primary** way to modify IFC files. It ensures proper relationship creation, GUID generation, and schema compliance.

### Syntax

```python
result = ifcopenshell.api.run("category.operation", model, **kwargs)

# Equivalent direct call (alternative syntax):
result = ifcopenshell.api.category.operation(model, **kwargs)
```

### Complete API Module Table (35 Modules)

| Module | Category | Primary Purpose | Core Functions |
|--------|----------|-----------------|----------------|
| **aggregate** | Spatial | Build hierarchies within projects | `assign_object`, `unassign_object` |
| **alignment** | Infrastructure | Road/rail alignment design | `add_alignment`, edit |
| **attribute** | Data | Direct attribute manipulation | `edit_attributes` |
| **boundary** | Spatial | Spatial boundaries and connections | `assign_connection_geometry` |
| **classification** | Organization | Classification systems | `add_classification`, `add_reference` |
| **cogo** | Geometry | Coordinate geometry operations | calculate, transform |
| **constraint** | Design | Design constraints and metrics | `add_objective`, `add_metric`, `assign_constraint` |
| **context** | Representation | Representation contexts | `add_context`, `edit_context`, `remove_context` |
| **control** | Relations | Control relationships | assign, establish |
| **cost** | Management | Cost schedules and calculations | `add_cost_schedule`, `add_cost_item` |
| **document** | References | Documentation and references | `add_information`, `assign_document` |
| **drawing** | Visualization | Drawing representations | `add_drawing`, `add_annotation` |
| **feature** | Geometry | Geometric features and infills | `add_feature` |
| **geometry** | Core | Representations and operations | `add_wall_representation`, `add_mesh_representation`, `add_profile_representation`, `assign_representation`, `edit_object_placement` |
| **georeference** | Location | Map coordinates and true north | set_coordinates, define |
| **grid** | Planning | Architectural and structural grids | create, manage, add_axis |
| **group** | Organization | Element groupings and collections | `add_group`, `assign_group`, `unassign_group` |
| **layer** | Visualization | Drawing layers | `add_layer`, `assign_layer` |
| **library** | Resources | External library references | `add_library`, `add_reference`, `assign_reference` |
| **material** | Construction | Material definitions and layers | `add_material`, `assign_material`, `add_material_set`, `add_layer` |
| **nest** | Organization | Nesting relationships | `assign_object`, `unassign_object` |
| **owner** | Organization | Actor information and roles | `add_person`, `add_organisation`, `set_user` |
| **profile** | Geometry | Profile definitions for structures | `add_parameterised_profile`, `add_arbitrary_profile` |
| **project** | Setup | Project file creation | `create_file`, `append_asset` |
| **pset** | Properties | Property and quantity sets | `add_pset`, `edit_pset`, `add_qto`, `edit_qto` |
| **pset_template** | Standards | Property set templates | create, assign, standardize |
| **resource** | Management | Resource allocation and tracking | `add_resource`, `assign_resource` |
| **root** | Entities | Base entity creation | `create_entity`, `remove_product`, `copy_class`, `reassign_class` |
| **sequence** | Scheduling | Task scheduling and dependencies | `add_work_schedule`, `add_task`, `edit_task_time` |
| **spatial** | Organization | Element spatial placement | `assign_container`, `unassign_container`, `reference_structure` |
| **structural** | Engineering | Structural models and analysis | `add_structural_analysis_model`, `add_structural_member` |
| **style** | Visualization | Presentation styles and textures | `add_style`, `add_surface_style`, `assign_representation_styles` |
| **system** | Building Systems | MEP systems and connections | `add_system`, `assign_system` |
| **type** | Templates | Type definitions and representations | `assign_type`, `unassign_type`, `map_type_representations` |
| **unit** | Measurement | Measurement units and conversion | `assign_unit`, `add_si_unit`, `add_conversion_based_unit` |
| **void** | Geometry | Void and opening definitions | `add_opening`, `add_filling` |

**Complete source:** https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html

### Detailed Examples per Key Category

#### root -- Create, Copy, and Remove

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Create a wall
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 001")

# Copy an existing element (preserves placement, psets, materials, NOT geometry)
wall_copy = ifcopenshell.api.run("root.copy_class", model, product=wall)

# Reassign class (change type while retaining geometry and relationships)
slab = ifcopenshell.api.run("root.reassign_class", model, product=wall_copy, ifc_class="IfcSlab")

# Remove an element (ALSO removes relationships)
ifcopenshell.api.run("root.remove_product", model, product=slab)
```

#### spatial -- Spatial Containment

```python
# Create spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="Ground Floor")

# Build hierarchy via aggregation
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=building, products=[storey])

# Assign wall to storey (containment -- one container only)
ifcopenshell.api.run("spatial.assign_container", model, relating_structure=storey, products=[wall])

# Referencing (multiple spaces allowed, for multi-storey elements)
ifcopenshell.api.run("spatial.reference_structure", model, products=[column], relating_structure=storey2)
```

#### geometry -- Representations

```python
import numpy

# Add geometric representation context first (root context, then subcontext)
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body_context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Add wall representation (extruded rectangle)
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)

# Set placement (4x4 transformation matrix, defaults to identity/origin)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.eye(4))
```

#### pset -- Property Sets

```python
# Add property set to wall
pset = ifcopenshell.api.run("pset.add_pset", model, product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "FireRating": "REI120",
    "ThermalTransmittance": 0.24,
    "LoadBearing": True
})

# Add quantity set
qto = ifcopenshell.api.run("pset.add_qto", model, product=wall, name="Qto_WallBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 5.0,
    "Height": 3.0,
    "Width": 0.2,
    "NetSideArea": 14.4,
    "GrossVolume": 3.0
})
```

#### material -- Materials

```python
# Create material
material = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30/37", category="concrete")

# Single material assignment
ifcopenshell.api.run("material.assign_material", model, products=[wall], material=material)

# Layered material for composite walls
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Wall Layers", set_type="IfcMaterialLayerSet")

concrete = ifcopenshell.api.run("material.add_material", model, name="Concrete")
layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
ifcopenshell.api.run("material.edit_layer", model, layer=layer,
    attributes={"LayerThickness": 0.2})

insulation = ifcopenshell.api.run("material.add_material", model, name="Mineral Wool")
layer2 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.run("material.edit_layer", model, layer=layer2,
    attributes={"LayerThickness": 0.1})

# Assign layer set to wall type (best practice: assign to types, not occurrences)
ifcopenshell.api.run("material.assign_material", model, products=[wall_type],
    type="IfcMaterialLayerSet", material=layer_set)
```

#### type -- Type Assignment

```python
# Create a wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")

# Assign type to wall occurrences
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)

# Query type of an element
import ifcopenshell.util.element
element_type = ifcopenshell.util.element.get_type(wall)
```

---

## 4. Element Traversal

### by_type -- Query by IFC Class

```python
# Get all walls (includes subtypes like IfcWallStandardCase in IFC4)
walls = model.by_type("IfcWall")

# Get ONLY IfcWall, NOT subtypes
walls_exact = model.by_type("IfcWall", include_subtypes=False)

# Get all products (everything with geometry)
products = model.by_type("IfcProduct")

# Get all spatial elements
spaces = model.by_type("IfcSpatialStructureElement")
```

**IMPORTANT:** `by_type()` returns a **tuple**, not a list. It includes subtypes by default.

### by_id and by_guid

```python
# Get entity by integer step ID (the #123 number in .ifc files)
entity = model.by_id(123)

# Get entity by GlobalId (22-character IFC GUID)
entity = model.by_guid("2O2Fr$t4X7Zf8NOew3FLOH")
```

### Inverse References

```python
# Get all entities that REFERENCE this entity
inverse = model.get_inverse(wall)

# Example: find which storey contains this wall
for rel in model.get_inverse(wall):
    if rel.is_a("IfcRelContainedInSpatialStructure"):
        print(f"Wall is in: {rel.RelatingStructure.Name}")

# Traverse all entities referenced BY this entity (forward references, recursive)
referenced = model.traverse(wall)
```

### Entity Attribute Access

```python
# Access attributes directly
print(wall.Name)            # "Wall 001"
print(wall.GlobalId)        # "2O2Fr$t4X7Zf8NOew3FLOH"
print(wall.is_a())          # "IfcWall"
print(wall.is_a("IfcRoot")) # True (inheritance check)
print(wall.id())            # 123 (step file ID)

# Get all attributes as dict
info = wall.get_info()      # {"id": 123, "type": "IfcWall", "GlobalId": "...", ...}

# Get info recursively (follows references -- EXPENSIVE, avoid on large models)
info_recursive = wall.get_info(recursive=True)
```

---

## 5. ifcopenshell.util.* Modules

The `ifcopenshell.util` package contains 29 utility submodules that simplify complex IFC relationship traversal.

### ifcopenshell.util.element

The most frequently used utility module. Provides shortcut functions for common element queries.

```python
from ifcopenshell.util.element import get_psets, get_type, get_container, get_material

# Get ALL property sets as nested dict
psets = get_psets(wall)
# Returns: {"Pset_WallCommon": {"id": 42, "IsExternal": True, "FireRating": "REI120", ...}, ...}

# Get the type element
wall_type = get_type(wall)  # Returns IfcWallType or None

# Get spatial container
container = get_container(wall)  # Returns IfcBuildingStorey or None

# Get material
material = get_material(wall)  # Returns material entity or None

# Get components/decomposition
from ifcopenshell.util.element import get_decomposition
components = get_decomposition(storey)  # All elements in storey
```

### ifcopenshell.util.unit

```python
from ifcopenshell.util.unit import calculate_unit_scale

# Get conversion factor to project SI units
scale = calculate_unit_scale(model)
# If project uses millimeters: scale = 0.001
# ALWAYS multiply raw coordinate values by this scale to get meters
```

### ifcopenshell.util.placement

```python
from ifcopenshell.util.placement import get_local_placement

# Get 4x4 transformation matrix for entity placement
matrix = get_local_placement(wall.ObjectPlacement)
# Returns numpy 4x4 array
# matrix[:3, 3] gives the XYZ translation
# matrix[:3, :3] gives the rotation
```

### ifcopenshell.util.selector

```python
from ifcopenshell.util.selector import Selector

# CSS-like query syntax for IFC elements
selector = Selector()
walls = selector.parse(model, ".IfcWall")
external_walls = selector.parse(model, '.IfcWall[Pset_WallCommon.IsExternal = "True"]')
```

### ifcopenshell.util.date

```python
from ifcopenshell.util.date import ifc2datetime, datetime2ifc
from datetime import datetime

# Convert IFC date string to Python datetime
dt = ifc2datetime("2024-01-15T10:30:00")

# Convert Python datetime to IFC format
ifc_date = datetime2ifc(datetime.now(), "IfcDateTime")
```

### ifcopenshell.util.shape_builder

```python
from ifcopenshell.util.shape_builder import ShapeBuilder

builder = ShapeBuilder(model)

# Create a rectangle profile
rectangle = builder.rectangle(size=(5.0, 0.2))

# Create circle profile
circle = builder.circle(radius=0.1)

# Extrude a profile into a 3D shape
extrusion = builder.extrude(rectangle, magnitude=3.0)
```

### Complete util Submodule Reference

**Data Access Utilities:**

| Submodule | Purpose | Key Functions |
|-----------|---------|---------------|
| **element** | Common element relationships | `get_psets`, `get_type`, `get_container`, `get_material`, `get_decomposition`, `copy` |
| **placement** | Coordinate transformations | `get_local_placement`, `set_placement`, `get_matrix` |
| **shape** | Geometry-based calculations | `get_surface_area`, `get_volume`, `simplify` |
| **unit** | Unit conversion | `calculate_unit_scale`, `get_unit_assignment` |
| **attribute** | Attribute access | `get_attribute`, `set_attribute` |
| **representation** | Representation handling | `get_representation`, `assign_representation` |

**Domain-Specific Utilities:**

| Submodule | Purpose |
|-----------|---------|
| **alignment** | Alignment data extraction |
| **classification** | Classification handling |
| **cost** | Cost data extraction |
| **sequence** | Scheduling data |
| **system** | Building system tracing |
| **constraint** | Design constraints |
| **boundary** | Spatial boundary calculation |

**Supporting Utilities:**

| Submodule | Purpose |
|-----------|---------|
| **schema** | Schema information and querying |
| **selector** | CSS-like query syntax |
| **brick** | Brick ontology mapping |
| **pset** | Property set templates |
| **data** | Generic data operations |
| **file** | File-level operations |
| **type** | Type definition utilities |
| **profiler** | Performance profiling |
| **doc** | Documentation extraction |
| **geolocation** | Geographic data |
| **date** | Temporal information |
| **fm** | Facilities management |
| **resource** | Resource utilities |
| **shape_builder** | Programmatic geometry creation |

**Complete source:** https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html

---

## 6. ifcopenshell.geom Module

### Geometry Processing Engine

The `ifcopenshell.geom` module uses OpenCASCADE Technology (OCCT) to convert IFC geometric representations into processable geometry (meshes or BRep solids).

### Basic Geometry Extraction

```python
import ifcopenshell.geom

# Configure geometry settings
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)       # Apply transformations
settings.set(settings.USE_BREP_DATA, False)          # Tessellated output
settings.set(settings.WELD_VERTICES, True)           # Merge duplicate vertices

# Process a single element
shape = ifcopenshell.geom.create_shape(settings, wall)

# Access geometry data
if shape is not None:
    verts = shape.geometry.verts   # Flat list: [x1,y1,z1, x2,y2,z2, ...]
    faces = shape.geometry.faces   # Flat list: [i1,i2,i3, i4,i5,i6, ...]
    matrix = shape.transformation.matrix.data  # 4x4 transformation matrix

    print(f"Vertices: {len(verts) // 3}")
    print(f"Faces: {len(faces) // 3}")
```

### Settings Reference Table

| Setting | Default | Description |
|---|---|---|
| `USE_WORLD_COORDS` | False | Apply full transformation chain to get world-space coordinates |
| `USE_BREP_DATA` | False | Return OpenCASCADE BRep instead of tessellated mesh |
| `WELD_VERTICES` | False | Merge coincident vertices |
| `APPLY_DEFAULT_MATERIALS` | True | Include material color information |
| `DISABLE_OPENING_SUBTRACTIONS` | False | Skip boolean operations for openings (faster) |
| `SEW_SHELLS` | False | Repair non-manifold geometry |

Additional settings via string-based API:

```python
settings = ifcopenshell.geom.settings()

# Precision and Quality
settings.set("precision", 0.001)    # 1mm (0.0001-0.01 range)
settings.set("deflection", 0.001)   # Triangle quality

# Include/Exclude Filters
settings.set("include-elements", ["IfcWall", "IfcColumn"])
settings.set("exclude-elements", ["IfcSpace", "IfcOpeningElement"])

# Representation Context
settings.set("context-identifiers", ["Body", "Axis"])

# Dimensionality
settings.set("dimensionality", 3)  # 3D output (2 for 2D)
```

### Iterator for Batch Processing

```python
import multiprocessing

# Process ALL geometry elements efficiently
iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        verts = shape.geometry.verts
        faces = shape.geometry.faces
        # ... process geometry ...
        if not iterator.next():
            break
```

**ALWAYS** use the iterator for processing 100+ elements. It is 5-10x faster than calling `create_shape()` in a loop.

### BRep vs Tessellation

| Feature | Tessellation (default) | BRep (`USE_BREP_DATA=True`) |
|---|---|---|
| Output | Triangle mesh (verts + faces) | OpenCASCADE TopoDS_Shape |
| Precision | Approximate | Exact |
| Use case | Visualization, export to mesh formats | CAD operations, boolean operations |
| Performance | Fast | Slower |
| Dependencies | None extra | OpenCASCADE Python bindings (OCC) |

### Geometry Creation Examples

#### Wall Representation (Uniform Thickness)

```python
# Create body context first (root context, then subcontext)
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body_context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create wall representation (length x height x thickness)
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=5.0, height=3.0, thickness=0.2)

# Assign representation to wall
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)
```

#### Mesh Representation (Complex Shapes)

```python
# Define vertices and faces
vertices = [[0,0,0],[1,0,0],[1,1,0],[0,1,0],[0,0,1],[1,0,1],[1,1,1],[0,1,1]]
faces = [[0,1,2,3],[4,7,6,5],[0,4,5,1],[1,5,6,2],[2,6,7,3],[3,7,4,0]]

# Create mesh representation
mesh_rep = ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body_context, vertices=vertices, faces=faces)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=mesh_rep)
```

#### Profile Extrusion (Structural Elements)

```python
# Create profile and extrude for beams/columns
profile_rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body_context, profile=ibeam_profile, depth=5.0)

ifcopenshell.api.run("geometry.assign_representation", model,
    product=beam, representation=profile_rep)
```

---

## 7. IFC Schema Versions

### Schema Overview

| Feature | IFC2X3 | IFC4 (Add2) | IFC4X3 (IFC4.3) |
|---------|--------|-------------|------------------|
| **ISO Standard** | ISO/PAS 16739:2005 | ISO 16739-1:2018 | ISO 16739-1:2024 |
| **Status** | Legacy (widely used) | Current standard | Latest version |
| **Entity Count** | ~600 | ~750 | ~900+ |
| **Infrastructure** | Not supported | Limited | Full (roads, rail, bridges) |
| **Spatial Hierarchy** | 4 levels | 6 levels | 12+ levels |
| **Geometry** | Basic B-rep, CSG | +Tessellation, AdvancedBrep | +Alignment sweeps, cant |
| **PredefinedType** | Limited | On most elements | On all elements |
| **StandardCase entities** | No | Yes (11 types) | Removed (merged back) |
| **Alignment support** | No | No | Full (horizontal, vertical, cant) |
| **Linear referencing** | No | No | Yes (IfcLinearPlacement) |

### Schema Selection Guidelines

**Use IFC2X3 if:**
- Legacy software compatibility is critical
- Project standards explicitly require it
- Minimal geometric complexity needed

**Use IFC4 if (recommended for most projects):**
- Good geometric support needed
- Excellent software compatibility
- General BIM workflows

**Use IFC4X3 if:**
- Infrastructure/alignment required (roads, bridges, railways)
- Latest features needed
- Complex MEP systems
- New project standards apply

### Key Entity Changes Between Versions

#### Entity Renames

| IFC2X3 Name | IFC4 Name | IFC4X3 Name |
|-------------|-----------|-------------|
| IfcBuildingElement | IfcBuildingElement | **IfcBuiltElement** (renamed) |
| IfcBuildingElementType | IfcBuildingElementType | **IfcBuiltElementType** (renamed) |
| -- | IfcBeamStandardCase | **Removed** (merged into IfcBeam) |
| -- | IfcColumnStandardCase | **Removed** (merged into IfcColumn) |
| -- | IfcDoorStandardCase | **Removed** (merged into IfcDoor) |
| -- | IfcMemberStandardCase | **Removed** (merged into IfcMember) |
| -- | IfcSlabStandardCase | **Removed** (merged into IfcSlab) |
| -- | IfcWallElementedCase | **Removed** (merged into IfcWall) |
| -- | IfcWindowStandardCase | **Removed** (merged into IfcWindow) |

#### Attribute Changes

| Entity | IFC2X3 | IFC4 / IFC4X3 | Change |
|--------|--------|---------------|--------|
| IfcMaterial | Name only | Name, Description, Category | New optional attributes |
| IfcOwnerHistory | REQUIRED | OPTIONAL in IFC4+ | Relaxed requirement |
| IfcBuildingElementProxy | CompositionType | PredefinedType | Attribute replaced |
| Many elements | No PredefinedType | PredefinedType added | IfcBeam, IfcColumn, IfcDoor, IfcWall, IfcWindow, etc. |
| Date/Time entities | IfcDateAndTime objects | IfcDateTime (string) | Simplified in IFC4 |

#### New Entities in IFC4.3 (Infrastructure)

| Entity | Description |
|--------|-------------|
| IfcFacility | Abstract supertype for all facility types |
| IfcBridge, IfcRoad, IfcRailway, IfcMarineFacility | Infrastructure facility types |
| IfcBridgePart, IfcRoadPart, IfcRailwayPart, IfcMarinePart | Facility part types |
| IfcAlignment, IfcAlignmentHorizontal, IfcAlignmentVertical | Alignment geometry |
| IfcBearing, IfcCourse, IfcKerb, IfcPavement | Infrastructure elements |
| IfcRail, IfcTrackElement | Railway elements |
| IfcEarthworksElement, IfcGeotechnicalElement | Geotechnical elements |
| IfcLinearElement, IfcPositioningElement | Linear referencing elements |
| IfcDeepFoundation | Foundation elements (IfcPile moved here) |
| IfcTransportationDevice | Vehicles and transport devices |

### Key Migration Notes: IFC2X3 to IFC4

1. **IfcWallStandardCase** -- use `IfcWall` with `PredefinedType=STANDARD`
2. **IfcRelDecomposes** -- split into `IfcRelAggregates` and `IfcRelNests`
3. **IfcOwnerHistory** -- OPTIONAL in IFC4 (was REQUIRED in IFC2X3)
4. **IfcMaterialLayerSetUsage** -- still supported but `IfcMaterialConstituentSet` added
5. **Date/time conversion** -- all date/time entity references must be converted to ISO 8601 strings

### Key Migration Notes: IFC4 to IFC4.3

1. **IfcBuildingStorey** -- still valid but `IfcFacilityPart` is the new generic spatial element
2. **IfcBuilding** -- still valid but `IfcFacility` is the new base class
3. **Infrastructure entities** -- `IfcRoad`, `IfcRailway`, `IfcBridge`, `IfcMarineFacility` added
4. **IfcAlignment** -- new spatial element for linear infrastructure
5. **All StandardCase classes** -- REMOVED entirely, merged back into parent classes
6. **IfcBuildingElement** -- renamed to `IfcBuiltElement`

### Spatial Structure Entity Availability

| Entity | IFC2X3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| IfcProject | Yes | Yes | Yes |
| IfcSite | Yes | Yes | Yes |
| IfcBuilding | Yes | Yes | Yes |
| IfcBuildingStorey | Yes | Yes | Yes |
| IfcSpace | Yes | Yes | Yes |
| IfcSpatialElement | No | Yes | Yes |
| IfcSpatialZone | No | Yes | Yes |
| IfcFacility | No | No | Yes |
| IfcFacilityPart | No | No | Yes |
| IfcBridge | No | No | Yes |
| IfcRoad | No | No | Yes |
| IfcRailway | No | No | Yes |
| IfcMarineFacility | No | No | Yes |

### Creating Files with Specific Schema

```python
# Explicit schema specification
model = ifcopenshell.file(schema="IFC4X3")

# Using the API
model = ifcopenshell.api.run("project.create_file", version="IFC4X3")

# Verify schema
print(model.schema)  # "IFC4X3"

# Supported values: "IFC2X3", "IFC4", "IFC4X3"
```

### Schema-Aware Code Pattern

```python
model = ifcopenshell.open("some_file.ifc")
schema = model.schema  # 'IFC2X3', 'IFC4', or 'IFC4X3'

# Version-aware element class name
if schema == "IFC4X3":
    building_element_class = "IfcBuiltElement"
else:
    building_element_class = "IfcBuildingElement"

elements = model.by_type(building_element_class)

# Version-aware StandardCase handling
if schema == "IFC2X3":
    walls = model.by_type("IfcWallStandardCase")
else:
    walls = [w for w in model.by_type("IfcWall") if w.PredefinedType == "STANDARD"]

# Schema-agnostic approach using ifcopenshell.util (recommended)
import ifcopenshell.util.element
for element in model.by_type("IfcElement"):
    container = ifcopenshell.util.element.get_container(element)
    psets = ifcopenshell.util.element.get_psets(element)
    material = ifcopenshell.util.element.get_material(element)
    element_type = ifcopenshell.util.element.get_type(element)
```

### Migration Tool (ifcpatch)

```python
import ifcopenshell
import ifcpatch

# Upgrade IFC2X3 to IFC4
model = ifcopenshell.open("input_2x3.ifc")
output = ifcpatch.execute({
    "input": "input_2x3.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4"]
})
ifcpatch.write(output, "output_ifc4.ifc")

# Upgrade IFC4 to IFC4X3
model = ifcopenshell.open("input_ifc4.ifc")
output = ifcpatch.execute({
    "input": "input_ifc4.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4X3"]
})
ifcpatch.write(output, "output_ifc4x3.ifc")
```

**Note:** Upgrading is more stable than downgrading. The Migrate recipe is marked as experimental. ALWAYS validate after conversion.

---

## 8. Entity Hierarchy and Inheritance

### Core Inheritance Chain

```
IfcRoot (abstract) -- GlobalId, OwnerHistory, Name, Description
├── IfcObjectDefinition (abstract)
│   ├── IfcObject (abstract)
│   │   ├── IfcProduct (abstract) -- has ObjectPlacement, Representation
│   │   │   ├── IfcSpatialElement (abstract, IFC4+)
│   │   │   │   ├── IfcSpatialStructureElement (abstract)
│   │   │   │   │   ├── IfcSite
│   │   │   │   │   ├── IfcFacility (abstract, IFC4.3)
│   │   │   │   │   │   ├── IfcBuilding
│   │   │   │   │   │   ├── IfcBridge (IFC4.3)
│   │   │   │   │   │   ├── IfcRoad (IFC4.3)
│   │   │   │   │   │   ├── IfcRailway (IFC4.3)
│   │   │   │   │   │   └── IfcMarineFacility (IFC4.3)
│   │   │   │   │   ├── IfcFacilityPart (IFC4.3)
│   │   │   │   │   │   ├── IfcBridgePart (IFC4.3)
│   │   │   │   │   │   ├── IfcFacilityPartCommon (IFC4.3)
│   │   │   │   │   │   ├── IfcRoadPart (IFC4.3)
│   │   │   │   │   │   ├── IfcRailwayPart (IFC4.3)
│   │   │   │   │   │   └── IfcMarinePart (IFC4.3)
│   │   │   │   │   ├── IfcBuildingStorey
│   │   │   │   │   └── IfcSpace
│   │   │   │   ├── IfcExternalSpatialStructureElement
│   │   │   │   │   └── IfcExternalSpatialElement
│   │   │   │   └── IfcSpatialZone
│   │   │   ├── IfcElement (abstract)
│   │   │   │   ├── IfcBuiltElement (was: IfcBuildingElement in IFC2X3/IFC4)
│   │   │   │   │   ├── IfcWall
│   │   │   │   │   ├── IfcSlab
│   │   │   │   │   ├── IfcBeam
│   │   │   │   │   ├── IfcColumn
│   │   │   │   │   ├── IfcDoor
│   │   │   │   │   ├── IfcWindow
│   │   │   │   │   ├── IfcRoof
│   │   │   │   │   ├── IfcStair
│   │   │   │   │   ├── IfcBearing (IFC4.3)
│   │   │   │   │   ├── IfcCourse (IFC4.3)
│   │   │   │   │   ├── IfcEarthworksElement (IFC4.3)
│   │   │   │   │   ├── IfcKerb (IFC4.3)
│   │   │   │   │   ├── IfcPavement (IFC4.3)
│   │   │   │   │   ├── IfcRail (IFC4.3)
│   │   │   │   │   ├── IfcTrackElement (IFC4.3)
│   │   │   │   │   └── ...
│   │   │   │   ├── IfcDistributionElement
│   │   │   │   ├── IfcOpeningElement (via IfcFeatureElement)
│   │   │   │   ├── IfcFurnishingElement
│   │   │   │   ├── IfcGeographicElement (IFC4+)
│   │   │   │   ├── IfcGeotechnicalElement (IFC4.3)
│   │   │   │   ├── IfcLinearElement (IFC4.3)
│   │   │   │   ├── IfcPositioningElement (IFC4.3)
│   │   │   │   ├── IfcTransportationDevice (IFC4.3)
│   │   │   │   └── IfcVirtualElement
│   │   │   ├── IfcAnnotation
│   │   │   ├── IfcPort
│   │   │   └── IfcStructuralItem / IfcStructuralActivity
│   │   ├── IfcProcess
│   │   │   └── IfcTask
│   │   ├── IfcResource
│   │   ├── IfcActor
│   │   ├── IfcControl
│   │   └── IfcGroup
│   ├── IfcContext
│   │   ├── IfcProject
│   │   └── IfcProjectLibrary
│   └── IfcTypeObject
│       └── IfcTypeProduct
│           └── IfcElementType (abstract)
│               ├── IfcWallType
│               ├── IfcSlabType
│               ├── IfcBeamType
│               └── ...
├── IfcPropertyDefinition
│   ├── IfcPropertySetDefinition
│   │   ├── IfcPropertySet
│   │   └── IfcElementQuantity
│   └── IfcPropertyTemplateDefinition
└── IfcRelationship (abstract)
    ├── IfcRelAssigns
    ├── IfcRelAssociates
    ├── IfcRelConnects
    ├── IfcRelDeclares
    ├── IfcRelDecomposes
    └── IfcRelDefines
```

### Spatial Structure Hierarchy Changes Across Versions

**IFC2X3 Spatial Hierarchy:**
```
IfcProduct
└── IfcSpatialStructureElement (abstract)
    ├── IfcSite
    ├── IfcBuilding
    ├── IfcBuildingStorey
    └── IfcSpace
```

**IFC4 Spatial Hierarchy:**
```
IfcProduct
└── IfcSpatialElement (abstract) <-- NEW abstract supertype
    ├── IfcSpatialStructureElement (abstract)
    │   ├── IfcSite
    │   ├── IfcBuilding
    │   ├── IfcBuildingStorey
    │   └── IfcSpace
    ├── IfcExternalSpatialStructureElement <-- NEW
    │   └── IfcExternalSpatialElement <-- NEW
    └── IfcSpatialZone <-- NEW
```

**IFC4.3 Spatial Hierarchy:**
```
IfcProduct
└── IfcSpatialElement (abstract)
    ├── IfcSpatialStructureElement (abstract)
    │   ├── IfcSite
    │   ├── IfcFacility (abstract) <-- NEW supertype above IfcBuilding
    │   │   ├── IfcBuilding (moved under IfcFacility)
    │   │   ├── IfcBridge <-- NEW
    │   │   ├── IfcRoad <-- NEW
    │   │   ├── IfcRailway <-- NEW
    │   │   └── IfcMarineFacility <-- NEW
    │   ├── IfcFacilityPart (abstract) <-- NEW
    │   │   ├── IfcBridgePart <-- NEW
    │   │   ├── IfcFacilityPartCommon <-- NEW
    │   │   ├── IfcRoadPart <-- NEW
    │   │   ├── IfcRailwayPart <-- NEW
    │   │   └── IfcMarinePart <-- NEW
    │   ├── IfcBuildingStorey
    │   └── IfcSpace
    ├── IfcExternalSpatialStructureElement
    │   └── IfcExternalSpatialElement
    └── IfcSpatialZone
```

### IfcBuiltElement Subtypes (IFC4.3)

IfcBuiltElement (renamed from IfcBuildingElement) has ~30 direct subtypes:

| Subtype | Notes |
|---------|-------|
| IfcBeam | All versions |
| IfcBearing | IFC4.3 (bridges) |
| IfcBuildingElementProxy | All versions |
| IfcChimney | IFC4+ |
| IfcColumn | All versions |
| IfcCourse | IFC4.3 (infrastructure) |
| IfcCovering | All versions |
| IfcCurtainWall | All versions |
| IfcDeepFoundation | IFC4.3 (replaces IfcPile hierarchy) |
| IfcDoor | All versions |
| IfcEarthworksElement | IFC4.3 (geotechnics) |
| IfcFooting | All versions |
| IfcKerb | IFC4.3 (roads) |
| IfcMember | All versions |
| IfcMooringDevice | IFC4.3 (marine) |
| IfcNavigationElement | IFC4.3 (marine) |
| IfcPavement | IFC4.3 (roads) |
| IfcPlate | All versions |
| IfcRail | IFC4.3 (railway) |
| IfcRailing | All versions |
| IfcRamp | All versions |
| IfcRampFlight | All versions |
| IfcRoof | All versions |
| IfcShadingDevice | IFC4+ |
| IfcSlab | All versions |
| IfcStair | All versions |
| IfcStairFlight | All versions |
| IfcTrackElement | IFC4.3 (railway) |
| IfcWall | All versions |
| IfcWindow | All versions |

### is_a() for Type Checking

```python
wall = model.by_type("IfcWall")[0]
wall.is_a()              # "IfcWall"
wall.is_a("IfcWall")     # True
wall.is_a("IfcElement")  # True (parent class)
wall.is_a("IfcProduct")  # True (grandparent)
wall.is_a("IfcRoot")     # True (ancestor)
wall.is_a("IfcSlab")     # False
```

---

## 9. Relationship Types

IFC uses **objectified relationships** -- relationships are first-class entities with their own attributes. NEVER try to set relationships by assigning attributes directly.

### Relationship Hierarchy (IFC4.3)

```
IfcRelationship (abstract)
├── IfcRelAssigns (abstract) -- assignment relationships
│   ├── IfcRelAssignsToActor
│   ├── IfcRelAssignsToControl
│   ├── IfcRelAssignsToGroup
│   │   └── IfcRelAssignsToGroupByFactor
│   ├── IfcRelAssignsToProcess
│   ├── IfcRelAssignsToProduct
│   └── IfcRelAssignsToResource
├── IfcRelAssociates (abstract) -- external reference associations
│   ├── IfcRelAssociatesApproval
│   ├── IfcRelAssociatesClassification
│   ├── IfcRelAssociatesConstraint
│   ├── IfcRelAssociatesDocument
│   ├── IfcRelAssociatesLibrary
│   ├── IfcRelAssociatesMaterial
│   └── IfcRelAssociatesProfileDef
├── IfcRelConnects (abstract) -- connectivity relationships
│   ├── IfcRelConnectsElements
│   │   ├── IfcRelConnectsPathElements
│   │   └── IfcRelConnectsWithRealizingElements
│   ├── IfcRelConnectsPortToElement
│   ├── IfcRelConnectsPorts
│   ├── IfcRelConnectsStructuralActivity
│   ├── IfcRelConnectsStructuralMember
│   │   └── IfcRelConnectsWithEccentricity
│   ├── IfcRelContainedInSpatialStructure
│   ├── IfcRelCoversBldgElements
│   ├── IfcRelCoversSpaces
│   ├── IfcRelFillsElement
│   ├── IfcRelFlowControlElements
│   ├── IfcRelInterferesElements
│   ├── IfcRelPositions (IFC4.3)
│   ├── IfcRelReferencedInSpatialStructure
│   ├── IfcRelSequence
│   ├── IfcRelServicesBuildings
│   └── IfcRelSpaceBoundary
│       ├── IfcRelSpaceBoundary1stLevel
│       └── IfcRelSpaceBoundary2ndLevel
├── IfcRelDeclares -- project/library declarations
├── IfcRelDecomposes (abstract) -- decomposition relationships
│   ├── IfcRelAggregates
│   ├── IfcRelNests
│   ├── IfcRelProjectsElement
│   └── IfcRelVoidsElement
└── IfcRelDefines (abstract) -- definition relationships
    ├── IfcRelDefinesByObject
    ├── IfcRelDefinesByProperties
    ├── IfcRelDefinesByTemplate
    └── IfcRelDefinesByType
```

### Key Relationship Types with Code

#### IfcRelContainedInSpatialStructure (Spatial Containment)

**Purpose:** Assigns elements to a spatial structure element. Each element can be contained in only ONE spatial structure.

```python
# Assign element to spatial structure
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# Query: find container of element
for rel in model.by_type("IfcRelContainedInSpatialStructure"):
    if wall in rel.RelatedElements:
        print(f"Container: {rel.RelatingStructure.Name}")

# Or use the utility function (recommended)
container = ifcopenshell.util.element.get_container(wall)

# Get all elements in a storey via inverse attribute
for rel in storey.ContainsElements:
    for element in rel.RelatedElements:
        print(f"  {element.is_a()}: {element.Name}")
```

#### IfcRelAggregates (Decomposition)

**Purpose:** Whole/part composition. Relates a parent to its child parts (Project to Site to Building to Storey, or physical assemblies).

```python
# Project -> Site -> Building -> Storey hierarchy
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])

# Query: find parts of building
for rel in model.by_type("IfcRelAggregates"):
    if rel.RelatingObject == building:
        for part in rel.RelatedObjects:
            print(f"Part: {part.Name}")

# Physical assembly: stair from parts
stair = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcStair")
flight = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcStairFlight")
landing = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSlab",
    predefined_type="LANDING")
railing = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcRailing")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[flight, landing, railing], relating_object=stair)
```

#### IfcRelDefinesByType (Type Assignment)

**Purpose:** Assigns an object type to object occurrences. Multiple occurrences share one type definition.

```python
# Assign type to element
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# Query type
element_type = ifcopenshell.util.element.get_type(wall)
print(f"Type: {element_type.Name if element_type else 'None'}")

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(wall_type)
```

#### IfcRelDefinesByProperties (Property Set Assignment)

**Purpose:** Assigns property sets to object instances. Handled internally by `pset.add_pset`.

```python
# This is handled internally by pset.add_pset
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
# The IfcRelDefinesByProperties is created automatically

# Query properties
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {'Pset_WallCommon': {'id': 42, 'IsExternal': True, 'FireRating': 'REI120', ...}}
```

#### IfcRelAssociatesMaterial (Material Association)

**Purpose:** Links material definitions to elements or types. Maximum one material association per building element.

```python
material = ifcopenshell.api.run("material.add_material", model, name="Concrete")
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material)

# Query material
mat = ifcopenshell.util.element.get_material(wall)
print(f"Material: {mat.Name if mat else 'None'}")
```

#### IfcRelVoidsElement (Openings)

**Purpose:** Creates a void (opening) in an element.

```python
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Query openings of an element
for rel in wall.HasOpenings:  # inverse: IfcRelVoidsElement
    opening = rel.RelatedOpeningElement
    print(f"Opening: {opening.Name}")
```

#### IfcRelFillsElement (Door/Window in Opening)

**Purpose:** Places an element into an opening.

```python
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)

# Query what fills an opening
for fill_rel in opening.HasFillings:
    filling = fill_rel.RelatedBuildingElement
    print(f"Filled by: {filling.is_a()} - {filling.Name}")
```

### IfcRelAssigns Variants

| Variant | Purpose | RelatingObject |
|---------|---------|---------------|
| IfcRelAssignsToActor | Assign responsibility to person/organization | IfcActor |
| IfcRelAssignsToControl | Assign to a control (budget, schedule) | IfcControl |
| IfcRelAssignsToGroup | Logical grouping of objects | IfcGroup |
| IfcRelAssignsToProcess | Assign to a process/task | IfcProcess |
| IfcRelAssignsToProduct | Assign to a product | IfcProduct |
| IfcRelAssignsToResource | Assign to a resource | IfcResource |

### Relationship Entity Availability Across Versions

| Relationship | IFC2X3 | IFC4 | IFC4X3 |
|-------------|--------|------|--------|
| IfcRelAggregates | Yes | Yes | Yes |
| IfcRelContainedInSpatialStructure | Yes | Yes | Yes |
| IfcRelAssociatesMaterial | Yes | Yes | Yes |
| IfcRelDefinesByProperties | Yes | Yes | Yes |
| IfcRelDefinesByType | Yes | Yes | Yes |
| IfcRelVoidsElement | Yes | Yes | Yes |
| IfcRelFillsElement | Yes | Yes | Yes |
| IfcRelAssignsTasks | Yes | No | No |
| IfcRelDeclares | No | Yes | Yes |
| IfcRelDefinesByTemplate | No | Yes | Yes |
| IfcRelInterferesElements | No | Yes | Yes |
| IfcRelAssociatesProfileDef | No | No | Yes |
| IfcRelPositions | No | No | Yes |

---

## 10. Common Operations

### Create a Minimal Valid IFC File

```python
import ifcopenshell
import ifcopenshell.api

# 1. Create model
model = ifcopenshell.file(schema="IFC4")

# 2. Setup project
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model,
    length={"is_metric": True, "raw": "METRES"})

# 3. Create geometric context
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body_context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# 4. Create spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])

# 5. Create wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# 6. Add properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90"
})

# 7. Write
model.write("minimal.ifc")
```

### Extract Data from Existing File

```python
import ifcopenshell
from ifcopenshell.util.element import get_psets, get_container, get_type

model = ifcopenshell.open("building.ifc")

for wall in model.by_type("IfcWall"):
    psets = get_psets(wall)
    container = get_container(wall)
    wall_type = get_type(wall)

    print(f"Wall: {wall.Name}")
    print(f"  Container: {container.Name if container else 'None'}")
    print(f"  Type: {wall_type.Name if wall_type else 'None'}")

    if "Pset_WallCommon" in psets:
        print(f"  External: {psets['Pset_WallCommon'].get('IsExternal', 'N/A')}")
        print(f"  Fire Rating: {psets['Pset_WallCommon'].get('FireRating', 'N/A')}")
```

### Modify Properties in Existing File

```python
model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Modify name
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "New Name"})

# Modify/add properties
from ifcopenshell.util.element import get_psets
psets = get_psets(wall)
if "Pset_WallCommon" in psets:
    pset_entity = model.by_id(psets["Pset_WallCommon"]["id"])
    ifcopenshell.api.run("pset.edit_pset", model, pset=pset_entity, properties={
        "IsExternal": False,
        "FireRating": "REI60"
    })

model.write("modified.ifc")
```

### Create Opening with Door

```python
# Create opening element
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# Add opening geometry
opening_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=0.9, height=2.1, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_rep)

# Create void relationship (boolean subtraction in wall)
ifcopenshell.api.run("void.add_opening", model, opening=opening, element=wall)

# Create door and fill the opening
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
ifcopenshell.api.run("void.add_filling", model, opening=opening, element=door)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[door])
```

### IFC4.3 Infrastructure Example

```python
# Create infrastructure project with bridge
model43 = ifcopenshell.api.run("project.create_file", version="IFC4X3")
project = ifcopenshell.api.run("root.create_entity", model43,
    ifc_class="IfcProject", name="Bridge Project")
site = ifcopenshell.api.run("root.create_entity", model43,
    ifc_class="IfcSite", name="Site")
bridge = ifcopenshell.api.run("root.create_entity", model43,
    ifc_class="IfcBridge", name="Main Bridge")

ifcopenshell.api.run("aggregate.assign_object", model43,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model43,
    relating_object=site, products=[bridge])
```

---

## 11. Common Error Patterns

### Error 1: AttributeError on IfcWallStandardCase in IFC4.3

```python
# WRONG: IfcWallStandardCase does not exist in IFC4.3
walls = model.by_type("IfcWallStandardCase")  # RuntimeError in IFC4.3!

# CORRECT: Check schema version first
if model.schema == "IFC2X3":
    walls = model.by_type("IfcWallStandardCase")
else:
    walls = model.by_type("IfcWall")
```

### Error 2: Missing OwnerHistory in IFC2X3

```python
# WRONG: OwnerHistory is REQUIRED in IFC2X3
wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new(), Name="W1")
# This creates an invalid IFC2X3 file!

# CORRECT: ALWAYS use api.run() which handles this automatically
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="W1")
```

### Error 3: Forgetting Spatial Containment

```python
# WRONG: Element is floating -- not contained in any storey
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="W1")
model.write("output.ifc")  # Wall exists but has no spatial location!

# CORRECT: ALWAYS assign spatial containment
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="W1")
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

### Error 4: Raw Coordinates Without Unit Conversion

```python
# WRONG: Assuming coordinates are in meters
from ifcopenshell.util.placement import get_local_placement
matrix = get_local_placement(wall.ObjectPlacement)
x, y, z = matrix[:3, 3]  # Could be in millimeters!

# CORRECT: ALWAYS apply unit scale
from ifcopenshell.util.unit import calculate_unit_scale
scale = calculate_unit_scale(model)
x_meters = x * scale
y_meters = y * scale
z_meters = z * scale
```

### Error 5: Direct Attribute Assignment for Relationships

```python
# WRONG: NEVER set relationships by attribute
wall.ContainedInStructure = storey  # This does NOT work!

# CORRECT: Use api.run() for relationships
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

### Error 6: Geometry Processing Without World Coordinates

```python
# WRONG: Not specifying world coordinates
shape = ifcopenshell.geom.create_shape(ifcopenshell.geom.settings(), wall)
# Coordinates are in LOCAL space -- not world position!

# CORRECT: Enable world coordinates
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)
shape = ifcopenshell.geom.create_shape(settings, wall)
```

### Error 7: Using remove() Without Cleaning Up Relationships

```python
# WRONG: Direct removal leaves dangling references
model.remove(wall)  # Relationships still reference this wall!

# CORRECT: Use api.run() which cleans up relationships
ifcopenshell.api.run("root.remove_product", model, product=wall)
```

### Error 8: Not Handling None Returns from Utility Functions

```python
# WRONG: Assuming get_container always returns a value
container = ifcopenshell.util.element.get_container(wall)
print(container.Name)  # AttributeError if container is None!

# CORRECT: ALWAYS check for None
container = ifcopenshell.util.element.get_container(wall)
if container:
    print(container.Name)
else:
    print("No container assigned")
```

### Error 9: Creating Duplicate Property Sets

```python
# WRONG: Creating a new pset without checking if it exists
pset = ifcopenshell.api.run("pset.add_pset", model, product=wall, name="Pset_WallCommon")
# If Pset_WallCommon already exists, you now have TWO!

# CORRECT: Check first
from ifcopenshell.util.element import get_psets
existing_psets = get_psets(wall)
if "Pset_WallCommon" in existing_psets:
    pset = model.by_id(existing_psets["Pset_WallCommon"]["id"])
else:
    pset = ifcopenshell.api.run("pset.add_pset", model, product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={"IsExternal": True})
```

### Error 10: Not Checking Shape for None Before Accessing Geometry

```python
# WRONG: Assuming geometry always succeeds
shape = ifcopenshell.geom.create_shape(settings, element)
vertices = shape.geometry.verts  # Crashes if shape is None

# CORRECT: ALWAYS check for None
shape = ifcopenshell.geom.create_shape(settings, element)
if shape is not None:
    vertices = shape.geometry.verts
else:
    print(f"No geometry for {element.Name}")
```

---

## 12. Performance Considerations

### File Loading

- `ifcopenshell.open()` loads the **entire file into memory**. For files > 500MB, this takes significant RAM.
- NEVER open a large file multiple times. Open once and pass the `model` reference.
- Use `model.by_type()` with `include_subtypes=False` when you know the exact type -- this avoids traversing the class hierarchy.

### File Size Reference

| Model Size | Elements | File Size | RAM Usage | Load Time |
|------------|----------|-----------|-----------|-----------|
| Small | < 1,000 | < 10 MB | < 100 MB | < 1s |
| Medium | 1,000 - 50,000 | 10 - 200 MB | 200 MB - 2 GB | 1-10s |
| Large | 50,000 - 500,000 | 200 MB - 2 GB | 2 - 16 GB | 10-60s |
| Very Large | > 500,000 | > 2 GB | 16+ GB | 60s+ |

### Geometry Processing

- `ifcopenshell.geom.create_shape()` is **expensive** -- it invokes OpenCASCADE boolean operations.
- ALWAYS use the **iterator** pattern for batch processing instead of calling `create_shape()` in a loop:

```python
import multiprocessing

# SLOW: O(n) individual calls with overhead per call
for wall in model.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)  # NEVER do this for 100+ elements

# FAST: Iterator with multiprocessing (5-10x faster)
iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        if not iterator.next():
            break
```

- Set `DISABLE_OPENING_SUBTRACTIONS=True` if you do NOT need boolean subtractions -- this skips expensive CSG operations.

### Property Access

- `get_psets(element)` traverses relationships every call. For bulk queries, cache results:

```python
# SLOW: Repeated relationship traversal
for wall in walls:
    psets = get_psets(wall)  # Re-traverses every time

# FAST: Cache results
psets_cache = {w.id(): get_psets(w) for w in walls}
for wall in walls:
    psets = psets_cache[wall.id()]  # Instant access

# FASTEST: Single relationship query
pset_rels = model.by_type("IfcRelDefinesByProperties")
pset_map = {}
for rel in pset_rels:
    for obj in rel.RelatedObjects:
        if obj.id() not in pset_map:
            pset_map[obj.id()] = []
        pset_map[obj.id()].append(rel.RelatingPropertyDefinition)
```

### Entity Creation

- ALWAYS batch spatial containment assignments when creating multiple elements:

```python
# SLOW: One assignment per element
for wall in walls:
    ifcopenshell.api.run("spatial.assign_container", model,
        relating_structure=storey, products=[wall])

# FAST: Batch assignment
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=walls)
```

### Memory Management

- Call `model.garbage_collect()` after removing many entities to reclaim memory.
- For read-only operations on large files, avoid `get_info(recursive=True)` -- it materializes the entire entity graph into Python dicts.
- NEVER store all geometry shapes in memory. Process and discard immediately.

---

## 13. AI Common Mistakes

These are mistakes that AI code generators (including LLMs) frequently make when generating IfcOpenShell code:

1. **Inventing API functions that do not exist.** AI models fabricate function names like `ifcopenshell.api.run("wall.create", ...)` or `ifcopenshell.create_wall()`. ALWAYS use the documented `root.create_entity` with `ifc_class` parameter.

2. **Using wrong parameter names.** The `api.run()` function uses specific keyword arguments. AI models frequently guess wrong names. Example: `relating_structure` (correct) vs `structure` (wrong) for spatial containment. ALWAYS check the API signature.

3. **Forgetting the spatial hierarchy.** A valid IFC file requires: Project -> Site -> Building -> Storey -> Elements. AI models frequently create elements without this hierarchy, producing invalid files.

4. **Mixing schema versions.** AI generates IFC2X3-specific code (like `IfcWallStandardCase`) for IFC4 files, or uses IFC4.3 features on IFC4 schemas. ALWAYS check `model.schema` first.

5. **Confusing `by_type()` return values.** `model.by_type("IfcWall")` returns a **tuple**, not a list. It includes subtypes by default. AI often treats it as a single entity instead of a collection.

6. **Setting relationships via attributes.** IFC relationships are separate entities. AI models try `wall.material = "Concrete"` instead of using `ifcopenshell.api.run("material.assign_material", ...)`.

7. **Generating invalid GlobalIds.** IFC GlobalIds are exactly 22 characters in a specific base64 encoding. ALWAYS use `ifcopenshell.guid.new()` -- NEVER generate GUIDs manually or use Python's `uuid` module.

8. **Not creating representation contexts.** Before assigning geometry, a `context` MUST exist. AI models skip `context.add_context` and get errors on geometry operations.

9. **Assuming imperial units.** IFC files can use any unit system. ALWAYS call `calculate_unit_scale()` before interpreting coordinate values.

10. **Using `model.remove()` instead of `api.run("root.remove_product", ...)`.** Direct removal leaves orphaned relationships. ALWAYS use the API function for clean removal.

11. **Forgetting to set object placement.** After assigning geometry, elements MUST have an `ObjectPlacement`. ALWAYS call `geometry.edit_object_placement` after geometry assignment.

12. **Using create_shape in loops for bulk processing.** The geometry iterator is 5-10x faster. NEVER call `create_shape()` in a loop for more than a few elements.

---

## 14. Sources

### Primary Documentation
- [IfcOpenShell Official Documentation](https://docs.ifcopenshell.org/)
- [IfcOpenShell Python API Reference](https://docs.ifcopenshell.org/ifcopenshell-python.html)
- [IfcOpenShell API Modules](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html)
- [IfcOpenShell Util Module](https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html)
- [IfcOpenShell Geometry Settings](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_settings.html)
- [IfcOpenShell Geometry Processing](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html)
- [IfcOpenShell Geometry Creation](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_creation.html)
- [IfcOpenShell Code Examples](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)

### Learning Resources
- [IfcOpenShell Academy](https://academy.ifcopenshell.org/)
- [OSArch Wiki - IfcOpenShell Code Examples](https://wiki.osarch.org/index.php?title=IfcOpenShell_code_examples)

### IFC Schema Standards
- [buildingSMART IFC Schema Specifications](https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/)
- [buildingSMART IFC4 Documentation](https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/)
- [buildingSMART IFC4.3 Documentation](https://ifc43-docs.standards.buildingsmart.org/)

### Repository and Packages
- [IfcOpenShell GitHub Repository](https://github.com/IfcOpenShell/IfcOpenShell)
- [IfcOpenShell PyPI Package](https://pypi.org/project/ifcopenshell/)
- [IfcOpenShell Conda Package](https://anaconda.org/ifcopenshell/ifcopenshell)
