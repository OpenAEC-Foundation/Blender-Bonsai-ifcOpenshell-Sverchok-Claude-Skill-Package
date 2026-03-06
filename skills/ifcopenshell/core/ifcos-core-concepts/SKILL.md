---
name: ifcos-core-concepts
description: "Explains IFC data model fundamentals including entity hierarchy (IfcRoot to IfcElement), spatial structure, ownership model, placement system, representation system, and relationship model across IFC2X3, IFC4, and IFC4X3 schemas. Activates when working with IFC files, understanding BIM data structures, or navigating IFC entity relationships."
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Core Concepts

## Quick Reference

### Entity Hierarchy Overview

```
IfcRoot (abstract) — GlobalId, OwnerHistory, Name, Description
├── IfcObjectDefinition (abstract)
│   ├── IfcObject (abstract)
│   │   ├── IfcProduct (abstract) — ObjectPlacement, Representation
│   │   │   ├── IfcSpatialElement [IFC4+] / IfcSpatialStructureElement [IFC2X3]
│   │   │   ├── IfcElement (abstract)
│   │   │   │   ├── IfcBuiltElement [IFC4X3] / IfcBuildingElement [IFC2X3/IFC4]
│   │   │   │   ├── IfcDistributionElement
│   │   │   │   ├── IfcFurnishingElement
│   │   │   │   └── ... (see references/methods.md)
│   │   │   ├── IfcAnnotation
│   │   │   └── IfcPort
│   │   ├── IfcProcess — IfcTask
│   │   ├── IfcResource
│   │   ├── IfcControl
│   │   └── IfcGroup
│   ├── IfcContext [IFC4+] / IfcObject [IFC2X3]
│   │   ├── IfcProject
│   │   └── IfcProjectLibrary [IFC4+]
│   └── IfcTypeObject
│       └── IfcTypeProduct → IfcElementType
├── IfcPropertyDefinition
│   ├── IfcPropertySet
│   └── IfcElementQuantity
└── IfcRelationship (abstract)
    ├── IfcRelDecomposes — IfcRelAggregates, IfcRelVoidsElement
    ├── IfcRelAssigns — assignments to actors, groups, processes
    ├── IfcRelConnects — IfcRelContainedInSpatialStructure, IfcRelFillsElement
    ├── IfcRelAssociates — IfcRelAssociatesMaterial, IfcRelAssociatesClassification
    ├── IfcRelDeclares — project/library declarations [IFC4+]
    └── IfcRelDefines — IfcRelDefinesByType, IfcRelDefinesByProperties
```

### Critical Version Differences

| Concept | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| IfcProject parent | IfcObject | IfcContext | IfcContext |
| OwnerHistory | MANDATORY | OPTIONAL | OPTIONAL |
| Spatial abstract parent | IfcSpatialStructureElement | IfcSpatialElement | IfcSpatialElement |
| Building element class | IfcBuildingElement | IfcBuildingElement | **IfcBuiltElement** (renamed) |
| StandardCase subtypes | Not present | Present (IfcWallStandardCase, etc.) | **REMOVED** |
| Infrastructure entities | Not present | Not present | IfcRoad, IfcBridge, IfcRailway, etc. |
| IfcFacility | Not present | Not present | Abstract parent of IfcBuilding |
| Door/Window type | IfcDoorStyle/IfcWindowStyle | IfcDoorType/IfcWindowType | IfcDoorType/IfcWindowType |
| IfcSpatialZone | Not present | Present | Present |
| IfcProjectLibrary | Not present | Present | Present |

### IfcRoot Attributes (Present on ALL Named Entities)

Every entity inheriting from IfcRoot has these four attributes:

| Attribute | Type | IFC2X3 | IFC4+ |
|-----------|------|--------|-------|
| GlobalId | IfcGloballyUniqueId | REQUIRED | REQUIRED |
| OwnerHistory | IfcOwnerHistory | **REQUIRED** | **OPTIONAL** |
| Name | IfcLabel | OPTIONAL | OPTIONAL |
| Description | IfcText | OPTIONAL | OPTIONAL |

---

## Essential Patterns

### 1. Spatial Structure Hierarchy

The spatial structure defines the physical organization of a project. ALWAYS build it top-down using `aggregate.assign_object`.

**IFC2X3 — Building-only hierarchy (RIGID):**
```
IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace
```

**IFC4 — Extended with zones:**
```
IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace
                                    + IfcSpatialZone (non-hierarchical overlays)
                                    + IfcExternalSpatialElement (outdoor)
```

**IFC4X3 — Infrastructure support (FLEXIBLE):**
```
IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace
                      → IfcRoad → IfcRoadPart
                      → IfcBridge → IfcBridgePart
                      → IfcRailway → IfcRailwayPart
                      → IfcFacility → IfcFacilityPart
```

```python
# IFC2X3/IFC4: Standard building hierarchy
# Schema: IFC2X3, IFC4
model = ifcopenshell.file(schema="IFC4")
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="My Project")
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=building, products=[storey])
```

```python
# IFC4X3: Infrastructure hierarchy
# Schema: IFC4X3
model = ifcopenshell.file(schema="IFC4X3")
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="Infra Project")
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="Project Site")
road = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcRoad", name="Highway A1")
road_part = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcRoadPart", name="Segment 1")

ifcopenshell.api.run("aggregate.assign_object", model, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[road])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=road, products=[road_part])
```

### 2. Entity Inheritance and Type Checking

ALWAYS use `is_a()` for type checking. It traverses the full inheritance chain.

```python
# Schema: IFC2X3, IFC4, IFC4X3
wall = model.by_type("IfcWall")[0]
wall.is_a()              # Returns "IfcWall"
wall.is_a("IfcWall")     # True
wall.is_a("IfcElement")  # True (parent class)
wall.is_a("IfcProduct")  # True (grandparent)
wall.is_a("IfcRoot")     # True (ancestor)
wall.is_a("IfcSlab")     # False (different subtype)
```

NEVER compare `is_a()` return values as strings for inheritance checks:
```python
# WRONG — misses subtypes
if wall.is_a() == "IfcBuildingElement":  # Fails: wall.is_a() returns "IfcWall"

# CORRECT — checks full inheritance chain
if wall.is_a("IfcBuildingElement"):  # True for all building element subtypes
```

### 3. Relationship Model (Objectified Relationships)

IFC uses objectified relationships — relationships are first-class IfcRoot entities with their own GlobalId. NEVER set relationships by assigning attributes directly.

**Key relationship types:**

| Relationship | Purpose | API Call |
|-------------|---------|----------|
| IfcRelAggregates | Whole/part decomposition (spatial hierarchy) | `aggregate.assign_object` |
| IfcRelContainedInSpatialStructure | Element-to-storey containment | `spatial.assign_container` |
| IfcRelDefinesByType | Type assignment (occurrence ↔ type) | `type.assign_type` |
| IfcRelDefinesByProperties | Property set assignment | `pset.add_pset` (auto-creates) |
| IfcRelAssociatesMaterial | Material assignment | `material.assign_material` |
| IfcRelVoidsElement | Opening in element | `void.add_opening` |
| IfcRelFillsElement | Door/window in opening | `void.add_filling` |
| IfcRelAssociatesClassification | Classification reference | `classification.add_reference` |

```python
# Schema: IFC2X3, IFC4, IFC4X3
# CORRECT: Use API to create relationships
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# CORRECT: Query relationships via utility functions
container = ifcopenshell.util.element.get_container(wall)
element_type = ifcopenshell.util.element.get_type(wall)
material = ifcopenshell.util.element.get_material(wall)
psets = ifcopenshell.util.element.get_psets(wall)
```

### 4. Ownership Model

Every IfcRoot entity references an IfcOwnerHistory that records who created/modified it.

**IFC2X3:** OwnerHistory is MANDATORY on all IfcRoot entities.
**IFC4/IFC4X3:** OwnerHistory is OPTIONAL.

```python
# Schema: IFC2X3 — OwnerHistory REQUIRED
owner_history = ifcopenshell.api.run("owner.create_owner_history", model)
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    owner_history,  # REQUIRED — cannot be None
    "MyWall"
)

# Schema: IFC4, IFC4X3 — OwnerHistory OPTIONAL
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    None,  # ALLOWED in IFC4+
    "MyWall"
)
```

ALWAYS use `ifcopenshell.api.run("root.create_entity", ...)` instead of direct `createIfc*` calls. The API handles OwnerHistory automatically based on schema version.

### 5. Placement System

Every IfcProduct has an optional ObjectPlacement. Placements are relative to a parent placement, forming a placement tree.

```python
# Schema: IFC2X3, IFC4, IFC4X3
import numpy

# Set placement using 4x4 transformation matrix (identity = origin)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.eye(4))

# Set placement relative to a container
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=numpy.array([
        [1.0, 0.0, 0.0, 5.0],   # X offset = 5.0
        [0.0, 1.0, 0.0, 0.0],
        [0.0, 0.0, 1.0, 0.0],
        [0.0, 0.0, 0.0, 1.0]
    ]))
```

### 6. Representation System

Geometric representation is assigned through representation contexts. ALWAYS create a context before assigning geometry.

```python
# Schema: IFC2X3, IFC4, IFC4X3
# Step 1: Create representation context
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body_context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Step 2: Create representation
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=5.0, height=3.0, thickness=0.2)

# Step 3: Assign representation to product
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
```

### 7. Type Objects (Occurrence/Type Pattern)

IFC separates type definitions from occurrences. A type (e.g., IfcWallType) defines shared properties; occurrences (e.g., IfcWall) reference the type.

```python
# Schema: IFC2X3, IFC4, IFC4X3
# Create a type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")

# Assign type to occurrences
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)

# Query type of an occurrence
element_type = ifcopenshell.util.element.get_type(wall1)

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(wall_type)
```

**Version-specific type entity names:**

| Element | IFC2X3 Type | IFC4/IFC4X3 Type |
|---------|-------------|------------------|
| IfcDoor | IfcDoorStyle | IfcDoorType |
| IfcWindow | IfcWindowStyle | IfcWindowType |
| All others | IfcWallType, IfcSlabType, etc. | Same |

---

## Common Operations

### Query All Elements by Type

```python
# Schema: IFC2X3, IFC4, IFC4X3
walls = model.by_type("IfcWall")           # All walls (tuple)
elements = model.by_type("IfcElement")     # ALL elements (includes subtypes)
products = model.by_type("IfcProduct")     # ALL products (elements + spatial + annotations)
```

### Navigate Spatial Hierarchy

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell.util.element

# Get the spatial container of an element
container = ifcopenshell.util.element.get_container(wall)  # Returns IfcBuildingStorey etc.

# Get all elements contained in a spatial structure
for rel in storey.ContainsElements:
    for element in rel.RelatedElements:
        print(f"{element.is_a()}: {element.Name}")

# Walk the spatial decomposition tree
for rel in building.IsDecomposedBy:
    for part in rel.RelatedObjects:
        print(f"{part.is_a()}: {part.Name}")  # Storeys
```

### Read Properties

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell.util.element

# Get all property sets as dict
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {'Pset_WallCommon': {'id': 42, 'IsExternal': True, 'FireRating': 'REI120'}}

# Get a specific property value
is_external = psets.get("Pset_WallCommon", {}).get("IsExternal")
```

### Schema Version Detection

```python
# Schema: IFC2X3, IFC4, IFC4X3
model = ifcopenshell.open("building.ifc")
schema = model.schema  # Returns "IFC2X3", "IFC4", or "IFC4X3"

# Version-aware entity class selection
if schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")
```

### Detect Schema and Handle IfcBuiltElement/IfcBuildingElement

```python
# Schema: IFC2X3, IFC4, IFC4X3
def get_building_elements(model):
    """Get all building/built elements regardless of schema version."""
    schema = model.schema
    if schema == "IFC4X3":
        return model.by_type("IfcBuiltElement")
    return model.by_type("IfcBuildingElement")
```

### Check Entity Existence in Schema

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell.ifcopenshell_wrapper

schema_obj = ifcopenshell.ifcopenshell_wrapper.schema_by_name(model.schema)
try:
    entity = schema_obj.declaration_by_name("IfcRoad")
    print("IfcRoad exists in this schema")
except RuntimeError:
    print("IfcRoad does NOT exist in this schema")
```

---

## Reference Links

- **[references/methods.md](references/methods.md)** — Complete entity class reference with attributes per schema version
- **[references/examples.md](references/examples.md)** — Working code examples for entity traversal, spatial structure creation, property access, and relationship navigation
- **[references/anti-patterns.md](references/anti-patterns.md)** — Common IFC conceptual mistakes with explanations

### Official Documentation

- IFC4X3 Documentation: https://ifc43-docs.standards.buildingsmart.org/
- buildingSMART IFC Standards: https://technical.buildingsmart.org/standards/ifc/
- IFC Schema Specifications: https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/
