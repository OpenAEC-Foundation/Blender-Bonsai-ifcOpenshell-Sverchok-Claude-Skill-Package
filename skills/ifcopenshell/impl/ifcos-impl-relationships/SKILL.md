---
name: ifcos-impl-relationships
description: >
  Use when managing IFC element relationships -- spatial containment, aggregation, type
  assignment, property association, material association, or void relationships. Prevents
  the common mistake of creating elements without establishing their spatial containment
  (orphaned elements). Covers relationship differences between IFC2X3 and IFC4.
  Keywords: relationship, containment, aggregation, type assignment, IfcRelContainedInSpatialStructure, IfcRelAggregates, IfcRelDefinesByType, void, nesting, assign to storey, connect elements, add opening.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Relationship Management with IfcOpenShell

## Quick Reference

### Decision Tree: Which Relationship Do I Need?

```
What are you connecting?
│
├── Spatial hierarchy (Project → Site → Building → Storey)?
│   └── Use aggregate.assign_object
│       └── Creates IfcRelAggregates
│
├── Element inside a spatial container (Wall in Storey)?
│   └── Use spatial.assign_container
│       └── Creates IfcRelContainedInSpatialStructure
│
├── Element referenced in (but not contained by) a spatial structure?
│   └── Use spatial.reference_structure
│       └── Creates IfcRelReferencedInSpatialStructure (IFC4+ only)
│
├── Type assignment (WallType → Wall occurrences)?
│   └── Use type.assign_type
│       └── Creates IfcRelDefinesByType
│
├── Property set on element?
│   └── Use pset.add_pset (creates IfcRelDefinesByProperties automatically)
│
├── Material on element?
│   └── Use material.assign_material
│       └── Creates IfcRelAssociatesMaterial
│
├── Opening/void in element (hole in wall)?
│   └── Use void.add_opening
│       └── Creates IfcRelVoidsElement
│
├── Filling an opening (door in hole)?
│   └── Use void.add_filling
│       └── Creates IfcRelFillsElement
│
├── Nesting (component attached to host at connection point)?
│   └── Use nest.assign_object
│       └── Creates IfcRelNests
│
├── Physical assembly (stair = flights + landings + railings)?
│   └── Use aggregate.assign_object
│       └── Creates IfcRelAggregates
│
└── Grouping (logical set, e.g. "External Walls")?
    └── Use group.assign_group
        └── Creates IfcRelAssignsToGroup
```

### Critical Warnings

- **ALWAYS** use `ifcopenshell.api.run()` to create relationships. NEVER create relationship entities directly with `model.create_entity("IfcRelAggregates", ...)` — the API handles GlobalId generation, OwnerHistory, placement recalculation, and cleanup of prior relationships.
- **NEVER** assign attributes directly to set relationship properties. IFC uses objectified relationships — relationships are first-class entities.
- **ALWAYS** check `model.schema` before using schema-specific inverse attributes. `IsTypedBy` exists only in IFC4+; in IFC2X3, type relations are in `IsDefinedBy`.
- **NEVER** contain an element in multiple spatial structures. Each element has exactly ONE spatial container via `IfcRelContainedInSpatialStructure`. Use `spatial.reference_structure` for secondary references.
- **ALWAYS** use `ifcopenshell.util.element` for querying relationships. It handles version differences internally.

---

## Relationship Entity Hierarchy

```
IfcRelationship (abstract)
├── IfcRelDecomposes (abstract)
│   ├── IfcRelAggregates          ← aggregate.assign_object
│   ├── IfcRelNests               ← nest.assign_object
│   ├── IfcRelProjectsElement
│   └── IfcRelVoidsElement        ← void.add_opening
├── IfcRelAssigns (abstract)
│   ├── IfcRelAssignsToGroup      ← group.assign_group
│   └── ... (Actor, Control, Process, Product, Resource)
├── IfcRelAssociates (abstract)
│   ├── IfcRelAssociatesMaterial  ← material.assign_material
│   └── ... (Classification, Document, Library, etc.)
├── IfcRelConnects (abstract)
│   ├── IfcRelContainedInSpatialStructure  ← spatial.assign_container
│   ├── IfcRelFillsElement        ← void.add_filling
│   └── ... (Ports, Structural, Space Boundary, etc.)
├── IfcRelDeclares (IFC4+)
└── IfcRelDefines (abstract)
    ├── IfcRelDefinesByProperties  ← pset.add_pset (automatic)
    └── IfcRelDefinesByType        ← type.assign_type
```

---

## Version Differences: Relationship Splits Between IFC2X3 and IFC4

### Inverse Attribute Changes

| Query | IFC2X3 | IFC4 / IFC4X3 |
|-------|--------|----------------|
| Type of element | `element.IsDefinedBy` → filter for `IfcRelDefinesByType` | `element.IsTypedBy` (dedicated inverse) |
| Type's inverse | `type.ObjectTypeOf` | `type.Types` (renamed) |
| Aggregation children | `obj.IsDecomposedBy` → returns `IfcRelDecomposes` | `obj.IsDecomposedBy` → returns `IfcRelAggregates` only |
| Nesting children | `obj.IsDecomposedBy` → filter for `IfcRelNests` | `obj.IsNestedBy` (dedicated inverse, IFC4+) |

### Entity Classification Changes

| Aspect | IFC2X3 | IFC4+ |
|--------|--------|-------|
| `IfcRelVoidsElement` parent | `IfcRelDecomposes` | `IfcRelDecomposes` (unchanged) |
| `IfcRelProjectsElement` parent | `IfcRelDecomposes` | `IfcRelDecomposes` (unchanged) |
| `IfcRelReferencedInSpatialStructure` | Does not exist | Available |
| `IfcRelDeclares` | Does not exist | Available |
| `IfcRelDefinesByObject` | Does not exist | Available |
| `IfcRelAssignsToGroupByFactor` | Does not exist | Available |
| `IfcRelInterferesElements` | Does not exist | Available |
| `IfcRelPositions` | Does not exist | IFC4X3 only |

### Valid Spatial Containers Per Version

| Container Entity | IFC2X3 | IFC4 | IFC4X3 |
|------------------|--------|------|--------|
| IfcSite | YES | YES | YES |
| IfcBuilding | YES | YES | YES |
| IfcBuildingStorey | YES | YES | YES |
| IfcSpace | YES | YES | YES |
| IfcExternalSpatialElement | — | YES | YES |
| IfcFacility (IfcBridge, IfcRoad, etc.) | — | — | YES |
| IfcFacilityPart (IfcRoadPart, etc.) | — | — | YES |

### Material Types Per Version

| Material Concept | IFC2X3 | IFC4+ |
|-----------------|--------|-------|
| IfcMaterial | YES | YES |
| IfcMaterialLayerSet | YES | YES |
| IfcMaterialLayerSetUsage | YES | YES |
| IfcMaterialProfileSet | — | YES |
| IfcMaterialProfileSetUsage | — | YES |
| IfcMaterialConstituentSet | — | YES |
| IfcMaterialList | YES | YES (deprecated) |

### Type Entity Changes

| IFC2X3 | IFC4+ Replacement |
|--------|-------------------|
| IfcDoorStyle | IfcDoorType |
| IfcWindowStyle | IfcWindowType |

---

## Essential Patterns

### Pattern 1: Build Spatial Hierarchy (Aggregation)

```python
# IfcOpenShell: IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# Each call creates an IfcRelAggregates
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])
```

### Pattern 2: Contain Elements in Spatial Structure

```python
# IfcOpenShell: all schema versions
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Creates IfcRelContainedInSpatialStructure
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

### Pattern 3: Assign Type to Occurrences

```python
# IfcOpenShell: IFC4
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")

# Creates IfcRelDefinesByType
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

### Pattern 4: Add Property Set

```python
# IfcOpenShell: all schema versions
# pset.add_pset creates IfcRelDefinesByProperties automatically
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={
        "IsExternal": True,
        "FireRating": "REI120",
        "ThermalTransmittance": 0.24
    })
```

### Pattern 5: Assign Material

```python
# IfcOpenShell: all schema versions
material = ifcopenshell.api.run("material.add_material", model,
    name="Concrete")

# Creates IfcRelAssociatesMaterial
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material)
```

### Pattern 6: Create Opening and Fill It

```python
# IfcOpenShell: all schema versions
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# Creates IfcRelVoidsElement
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")

# Creates IfcRelFillsElement
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)
```

### Pattern 7: Nest Components

```python
# IfcOpenShell: all schema versions
sink = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSanitaryTerminal", name="Kitchen Sink")
faucet = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcValve", name="Faucet")

# Creates IfcRelNests: faucet is nested in sink
ifcopenshell.api.run("nest.assign_object", model,
    related_objects=[faucet], relating_object=sink)
```

---

## Querying Relationships

### Version-Safe Queries Using ifcopenshell.util.element

```python
# IfcOpenShell: all schema versions
import ifcopenshell.util.element

# Get spatial container of element
container = ifcopenshell.util.element.get_container(wall)

# Get type of element
element_type = ifcopenshell.util.element.get_type(wall)

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(wall_type)

# Get property sets
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"IsExternal": True, ...}}

# Get material
material = ifcopenshell.util.element.get_material(wall)

# Get full decomposition tree
children = ifcopenshell.util.element.get_decomposition(building)

# Get aggregate parent
parent = ifcopenshell.util.element.get_aggregate(storey)
```

### Direct Inverse Attribute Queries

```python
# IfcOpenShell: IFC4+
# Aggregation children
for rel in building.IsDecomposedBy:
    for child in rel.RelatedObjects:
        print(f"Child: {child.Name}")

# Aggregation parent
for rel in storey.Decomposes:
    print(f"Parent: {rel.RelatingObject.Name}")

# Spatial containment
for rel in storey.ContainsElements:
    for elem in rel.RelatedElements:
        print(f"Contains: {elem.is_a()} - {elem.Name}")

# Element's container
for rel in wall.ContainedInStructure:
    print(f"In: {rel.RelatingStructure.Name}")

# Type assignment (IFC4+)
for rel in wall.IsTypedBy:
    print(f"Type: {rel.RelatingType.Name}")

# Openings
for rel in wall.HasOpenings:
    opening = rel.RelatedOpeningElement
    for fill in opening.HasFillings:
        print(f"Filled by: {fill.RelatedBuildingElement.Name}")

# Property sets
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        print(f"PSet: {pset.Name}")

# Material
for rel in wall.HasAssociations:
    if rel.is_a("IfcRelAssociatesMaterial"):
        print(f"Material: {rel.RelatingMaterial.is_a()}")

# Nesting (IFC4+)
for rel in sink.IsNestedBy:
    for child in rel.RelatedObjects:
        print(f"Nested: {child.Name}")
```

### IFC2X3 Inverse Attribute Differences

```python
# IfcOpenShell: IFC2X3 ONLY
# Type assignment: NO IsTypedBy inverse in IFC2X3
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByType"):
        print(f"Type: {rel.RelatingType.Name}")

# Aggregation/Nesting: shared inverse in IFC2X3
for rel in building.IsDecomposedBy:
    # Returns IfcRelDecomposes instances (includes both aggregation and nesting)
    if rel.is_a("IfcRelAggregates"):
        for child in rel.RelatedObjects:
            print(f"Aggregated: {child.Name}")
    elif rel.is_a("IfcRelNests"):
        for child in rel.RelatedObjects:
            print(f"Nested: {child.Name}")
```

---

## Removing Relationships

```python
# IfcOpenShell: all schema versions

# Remove spatial containment
ifcopenshell.api.run("spatial.unassign_container", model, products=[wall])

# Remove aggregation
ifcopenshell.api.run("aggregate.unassign_object", model, products=[storey])

# Remove type assignment
ifcopenshell.api.run("type.unassign_type", model, related_objects=[wall])

# Remove material assignment
ifcopenshell.api.run("material.unassign_material", model, products=[wall])

# Remove nesting
ifcopenshell.api.run("nest.unassign_object", model, related_objects=[faucet])

# Remove property set
ifcopenshell.api.run("pset.remove_pset", model, product=wall, pset=pset)

# Remove spatial reference
ifcopenshell.api.run("spatial.dereference_structure", model,
    products=[wall], relating_structure=other_storey)
```

---

## Version-Safe Type Query Pattern

```python
# IfcOpenShell: all schema versions
def get_element_type_safe(element):
    """Get element type across all IFC versions."""
    schema = element.wrapped_data.file.schema
    if schema == "IFC2X3":
        for rel in element.IsDefinedBy:
            if rel.is_a("IfcRelDefinesByType"):
                return rel.RelatingType
    else:
        for rel in element.IsTypedBy:
            return rel.RelatingType
    return None

# PREFERRED: use ifcopenshell.util.element.get_type() which handles this internally
import ifcopenshell.util.element
element_type = ifcopenshell.util.element.get_type(wall)
```

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all relationship API methods
- [Working Code Examples](references/examples.md) — End-to-end examples for all relationship operations
- [Anti-Patterns](references/anti-patterns.md) — Common relationship mistakes and how to avoid them
