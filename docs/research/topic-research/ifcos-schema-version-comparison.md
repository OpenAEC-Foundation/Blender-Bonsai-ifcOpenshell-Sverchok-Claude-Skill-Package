# IFC Schema Version Comparison for IfcOpenShell Python Programming

## Research Metadata
- **Date**: 2026-03-05
- **Topic**: IFC2x3 vs IFC4 vs IFC4.3 (IFC4x3) schema differences
- **Purpose**: Phase 2 research for ifcos-core-schemas and ifcos-errors-schema skills
- **Sources**: IFC2x3 TC1 specification, IFC4 ADD2 TC1 specification, IFC4.3.2.0 (IFC4x3) specification, IfcOpenShell Python API, buildingSMART standards
- **Confidence**: HIGH for entity hierarchy/relationships, MEDIUM for exhaustive attribute-level changes (full EXPRESS schema diff exceeds what can be captured here)
- **Note**: This research is based on the official IFC EXPRESS schemas and IfcOpenShell behavior as of IfcOpenShell v0.7.x/v0.8.x. Some minor attributes may vary; always verify against `ifcopenshell.schema_by_name()` for definitive answers.

---

## 1. Schema Version Overview

### Version Timeline
| Version | Year | Full Name | ISO Standard | Key Focus |
|---------|------|-----------|--------------|-----------|
| IFC2x3 TC1 | 2007 | IFC 2x Edition 3 Technical Corrigendum 1 | ISO 16739:2013 | Buildings (legacy, most widely supported) |
| IFC4 ADD2 TC1 | 2017 | IFC 4 Addendum 2 Technical Corrigendum 1 | ISO 16739-1:2018 | Buildings (modernized, extended) |
| IFC4.3.2.0 | 2024 | IFC 4x3 Add2 (commonly "IFC4X3") | ISO 16739-1:2024 | Buildings + Infrastructure (roads, rail, bridges, marine) |

### IfcOpenShell Schema Identifiers
```python
import ifcopenshell

# Opening files with specific schemas
ifc2x3 = ifcopenshell.file(schema="IFC2X3")
ifc4   = ifcopenshell.file(schema="IFC4")
ifc4x3 = ifcopenshell.file(schema="IFC4X3")

# Reading schema version from existing file
model = ifcopenshell.open("model.ifc")
print(model.schema)  # Returns "IFC2X3", "IFC4", or "IFC4X3"

# Introspecting schema definitions
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
entity = schema.declaration_by_name("IfcWall")
print(entity.all_attributes())
```

### Entity Count by Version (approximate)
| Version | Total Entities | Total Types | Total Enumerations |
|---------|---------------|-------------|-------------------|
| IFC2x3 | ~653 | ~157 | ~164 |
| IFC4 | ~776 | ~206 | ~207 |
| IFC4X3 | ~928 | ~246 | ~260 |

---

## 2. Entity Hierarchy and Inheritance

### 2.1 Core Hierarchy (all versions)

```
IfcRoot (abstract)
  |-- GlobalId: IfcGloballyUniqueId
  |-- OwnerHistory: IfcOwnerHistory
  |-- Name: IfcLabel (OPTIONAL)
  |-- Description: IfcText (OPTIONAL)
  |
  |-- IfcObjectDefinition (abstract)
  |   |-- IfcContext [IFC4+] (abstract)
  |   |   |-- IfcProject
  |   |   |-- IfcProjectLibrary [IFC4+]
  |   |
  |   |-- IfcObject (abstract)
  |   |   |-- ObjectType: IfcLabel (OPTIONAL)
  |   |   |
  |   |   |-- IfcProduct (abstract)
  |   |   |   |-- ObjectPlacement: IfcObjectPlacement (OPTIONAL)
  |   |   |   |-- Representation: IfcProductRepresentation (OPTIONAL)
  |   |   |   |
  |   |   |   |-- IfcSpatialElement [IFC4+] / IfcSpatialStructureElement [IFC2x3]
  |   |   |   |-- IfcElement
  |   |   |   |-- IfcGrid
  |   |   |   |-- IfcPort
  |   |   |   |-- IfcProxy
  |   |   |   |-- IfcAnnotation
  |   |   |   |-- IfcStructuralItem
  |   |   |   |-- IfcPositioningElement [IFC4X3]
  |   |   |   |-- IfcLinearElement [IFC4X3]
  |   |   |
  |   |   |-- IfcProcess (abstract)
  |   |   |-- IfcResource (abstract)
  |   |   |-- IfcControl (abstract)
  |   |   |-- IfcGroup
  |   |   |-- IfcActor
  |   |
  |   |-- IfcTypeObject (abstract)
  |       |-- IfcTypeProcess [IFC4+]
  |       |-- IfcTypeResource [IFC4+]
  |       |-- IfcTypeProduct
  |           |-- IfcDoorStyle [IFC2x3] / IfcDoorType [IFC4+]
  |           |-- IfcWindowStyle [IFC2x3] / IfcWindowType [IFC4+]
  |           |-- IfcElementType
  |
  |-- IfcRelationship (abstract)
      |-- IfcRelDecomposes (abstract)
      |-- IfcRelAssigns (abstract)
      |-- IfcRelConnects (abstract)
      |-- IfcRelAssociates (abstract)
      |-- IfcRelDefines (abstract) [restructured in IFC4]
```

### 2.2 Key Hierarchy Differences Between Versions

#### IfcContext (IFC4+ only)
| Aspect | IFC2x3 | IFC4 / IFC4X3 |
|--------|--------|---------------|
| IfcProject parent | IfcObject | IfcContext (new abstract) |
| IfcProjectLibrary | Does not exist | Exists under IfcContext |

**Code impact:**
```python
# IFC2x3: IfcProject is under IfcObject
project = ifc2x3.by_type("IfcProject")[0]
# project.is_a("IfcObject")  -> True
# project.is_a("IfcContext")  -> ERROR (no such entity)

# IFC4: IfcProject is under IfcContext
project = ifc4.by_type("IfcProject")[0]
# project.is_a("IfcContext")  -> True
# project.is_a("IfcObject")   -> False (different branch)
```

#### IfcOwnerHistory (optionality change)
| Aspect | IFC2x3 | IFC4 / IFC4X3 |
|--------|--------|---------------|
| IfcRoot.OwnerHistory | MANDATORY | OPTIONAL |

**Code impact:**
```python
# IFC2x3: OwnerHistory MUST be provided
owner_history = ifcopenshell.api.run("owner.create_owner_history", ifc2x3)
wall = ifc2x3.createIfcWall(
    ifcopenshell.guid.new(),
    owner_history,  # REQUIRED, cannot be None
    "MyWall"
)

# IFC4/IFC4X3: OwnerHistory can be None
wall = ifc4.createIfcWall(
    ifcopenshell.guid.new(),
    None,  # ALLOWED in IFC4+
    "MyWall"
)
```

---

## 3. Spatial Structure Hierarchy

### 3.1 IFC2x3 Spatial Structure

```
IfcSpatialStructureElement (abstract)
  |-- CompositionType: IfcElementCompositionEnum (COMPLEX, ELEMENT, PARTIAL)
  |
  |-- IfcSite
  |-- IfcBuilding
  |-- IfcBuildingStorey
  |-- IfcSpace
```

**IFC2x3 spatial hierarchy is RIGID:**
```
IfcProject
  └── IfcSite
       └── IfcBuilding
            └── IfcBuildingStorey
                 └── IfcSpace
```

### 3.2 IFC4 Spatial Structure

```
IfcSpatialElement (abstract) [NEW in IFC4]
  |
  |-- IfcSpatialStructureElement (abstract)
  |   |-- IfcSite
  |   |-- IfcBuilding
  |   |-- IfcBuildingStorey
  |   |-- IfcSpace
  |
  |-- IfcExternalSpatialElement [NEW in IFC4]
  |-- IfcSpatialZone [NEW in IFC4]
```

**Key additions in IFC4:**
- `IfcSpatialElement` as new abstract parent (allows non-structural spatial concepts)
- `IfcExternalSpatialElement` for outdoor spaces
- `IfcSpatialZone` for overlapping zones (fire zones, HVAC zones, etc.)

### 3.3 IFC4X3 Spatial Structure (major expansion)

```
IfcSpatialElement (abstract)
  |
  |-- IfcSpatialStructureElement (abstract)
  |   |-- IfcSite
  |   |-- IfcBuilding        (specific to buildings)
  |   |-- IfcBuildingStorey   (specific to buildings)
  |   |-- IfcSpace
  |   |-- IfcFacility [NEW in IFC4X3]
  |   |   |-- IfcBridge [NEW]
  |   |   |-- IfcMarineFacility [NEW]
  |   |   |-- IfcRailway [NEW]
  |   |   |-- IfcRoad [NEW]
  |   |
  |   |-- IfcFacilityPart [NEW in IFC4X3]
  |   |   |-- IfcBridgePart [NEW]
  |   |   |-- IfcFacilityPartCommon [NEW]
  |   |   |-- IfcMarinePart [NEW]
  |   |   |-- IfcRailwayPart [NEW]
  |   |   |-- IfcRoadPart [NEW]
  |
  |-- IfcExternalSpatialElement
  |-- IfcSpatialZone
```

**IFC4X3 spatial hierarchy is FLEXIBLE:**
```
IfcProject
  └── IfcSite
       ├── IfcBuilding
       │    └── IfcBuildingStorey
       │         └── IfcSpace
       ├── IfcRoad [NEW]
       │    └── IfcRoadPart [NEW]
       ├── IfcBridge [NEW]
       │    └── IfcBridgePart [NEW]
       ├── IfcRailway [NEW]
       │    └── IfcRailwayPart [NEW]
       └── IfcFacility [NEW - generic]
            └── IfcFacilityPart [NEW]
                 └── IfcFacilityPartCommon [NEW]
```

### 3.4 Spatial Structure Comparison Table

| Entity | IFC2x3 | IFC4 | IFC4X3 | Notes |
|--------|--------|------|--------|-------|
| IfcSpatialElement | - | YES (abstract) | YES (abstract) | New abstract parent in IFC4 |
| IfcSpatialStructureElement | YES (abstract) | YES (abstract) | YES (abstract) | Direct spatial element in 2x3, under IfcSpatialElement in IFC4+ |
| IfcSite | YES | YES | YES | Stable across versions |
| IfcBuilding | YES | YES | YES | Stable across versions |
| IfcBuildingStorey | YES | YES | YES | Stable across versions |
| IfcSpace | YES | YES | YES | Stable across versions |
| IfcExternalSpatialElement | - | YES | YES | For exterior spaces |
| IfcSpatialZone | - | YES | YES | Non-hierarchical zones |
| IfcFacility | - | - | YES | Generic infrastructure facility |
| IfcFacilityPart | - | - | YES | Part of a facility |
| IfcBridge | - | - | YES | Bridge facility |
| IfcBridgePart | - | - | YES | Bridge segment |
| IfcRoad | - | - | YES | Road facility |
| IfcRoadPart | - | - | YES | Road segment |
| IfcRailway | - | - | YES | Railway facility |
| IfcRailwayPart | - | - | YES | Railway segment |
| IfcMarineFacility | - | - | YES | Marine facility |
| IfcMarinePart | - | - | YES | Marine segment |
| IfcFacilityPartCommon | - | - | YES | Generic part |

### 3.5 Code: Creating Spatial Structure Per Version

```python
import ifcopenshell
import ifcopenshell.api

# ---- IFC2x3 ----
model = ifcopenshell.file(schema="IFC2X3")
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="My Project")
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=building, products=[storey])

# ---- IFC4X3: Infrastructure ----
model = ifcopenshell.file(schema="IFC4X3")
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="Infra Project")
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="Project Site")
road = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcRoad", name="Highway A1")
road_part = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcRoadPart", name="Segment 1")

ifcopenshell.api.run("aggregate.assign_object", model, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[road])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=road, products=[road_part])
```

---

## 4. IfcElement Subtypes

### 4.1 IfcBuildingElement / IfcBuiltElement Hierarchy

**IMPORTANT RENAME**: `IfcBuildingElement` (IFC2x3/IFC4) was renamed to `IfcBuiltElement` in IFC4X3.

| Entity | IFC2x3 | IFC4 | IFC4X3 | Notes |
|--------|--------|------|--------|-------|
| **IfcBuildingElement** | YES (abstract) | YES (abstract) | RENAMED to **IfcBuiltElement** | Major rename |
| IfcWall | YES | YES | YES | |
| IfcWallStandardCase | YES | YES | REMOVED (merged into IfcWall) | See section 4.2 |
| IfcColumn | YES | YES | YES | |
| IfcColumnStandardCase | - | YES | REMOVED | |
| IfcBeam | YES | YES | YES | |
| IfcBeamStandardCase | - | YES | REMOVED | |
| IfcSlab | YES | YES | YES | |
| IfcSlabStandardCase | - | YES | REMOVED | |
| IfcSlabElementedCase | - | YES | REMOVED | |
| IfcDoor | YES | YES | YES | |
| IfcDoorStandardCase | - | YES | REMOVED | |
| IfcWindow | YES | YES | YES | |
| IfcWindowStandardCase | - | YES | REMOVED | |
| IfcPlate | YES | YES | YES | |
| IfcPlateStandardCase | - | YES | REMOVED | |
| IfcMember | YES | YES | YES | |
| IfcMemberStandardCase | - | YES | REMOVED | |
| IfcStair | YES | YES | YES | |
| IfcStairFlight | YES | YES | YES | |
| IfcRamp | YES | YES | YES | |
| IfcRampFlight | YES | YES | YES | |
| IfcRoof | YES | YES | YES | |
| IfcCovering | YES | YES | YES | |
| IfcCurtainWall | YES | YES | YES | |
| IfcRailing | YES | YES | YES | |
| IfcFooting | YES | YES | YES | |
| IfcPile | YES | YES | YES | |
| IfcBuildingElementProxy | YES | YES | YES (under IfcBuiltElement) | |
| IfcChimney | - | YES | YES | |
| IfcShadingDevice | - | YES | YES | |
| **IfcCourse** | - | - | YES | New: for road/rail courses |
| **IfcDeepFoundation** | - | - | YES | New: piles, caissons |
| **IfcEarthworksElement** | - | - | YES | New: cuts, fills |
| **IfcKerb** | - | - | YES | New: road kerbs |
| **IfcMooringDevice** | - | - | YES | New: marine |
| **IfcNavigationElement** | - | - | YES | New: marine navigation |
| **IfcPavement** | - | - | YES | New: road pavement |
| **IfcRail** | - | - | YES | New: rail track |
| **IfcTrackElement** | - | - | YES | New: railway track elements |

### 4.2 The StandardCase Elimination (IFC4 -> IFC4X3)

IFC4 introduced `*StandardCase` subtypes to indicate elements with standard geometry constraints (e.g., prismatic extrusions along a linear axis). IFC4X3 REMOVED all of them, folding the distinction into the `PredefinedType` attribute or geometry usage.

**Affected entities (all REMOVED in IFC4X3):**
- IfcWallStandardCase
- IfcColumnStandardCase
- IfcBeamStandardCase
- IfcSlabStandardCase
- IfcSlabElementedCase
- IfcDoorStandardCase
- IfcWindowStandardCase
- IfcPlateStandardCase
- IfcMemberStandardCase

**Migration pattern:**
```python
# IFC4 code that checks for StandardCase:
walls = model.by_type("IfcWallStandardCase")  # Works in IFC4

# IFC4X3 equivalent:
walls = model.by_type("IfcWall")  # All walls, no StandardCase distinction
# StandardCase behavior is now implied by geometry usage, not entity type
```

### 4.3 IfcDistributionElement Hierarchy

| Entity | IFC2x3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| IfcDistributionElement | YES | YES | YES |
| IfcDistributionFlowElement | YES | YES | YES |
| IfcDistributionControlElement | YES | YES | YES |
| IfcEnergyConversionDevice | YES | YES | YES |
| IfcFlowController | YES | YES | YES |
| IfcFlowFitting | YES | YES | YES |
| IfcFlowMovingDevice | YES | YES | YES |
| IfcFlowSegment | YES | YES | YES |
| IfcFlowStorageDevice | YES | YES | YES |
| IfcFlowTerminal | YES | YES | YES |
| IfcFlowTreatmentDevice | YES | YES | YES |
| **IfcDistributionBoard** | - | - | YES |
| **IfcSignal** | - | - | YES |
| **IfcSignalType** | - | - | YES |

### 4.4 IfcElement Full Hierarchy

```
IfcElement (abstract)
  |-- IfcBuildingElement [IFC2x3/IFC4] / IfcBuiltElement [IFC4X3]
  |-- IfcDistributionElement
  |-- IfcFurnishingElement
  |-- IfcTransportElement [restructured in IFC4X3]
  |-- IfcOpeningElement [IFC2x3/IFC4] / IfcOpeningElement + IfcVoidingFeature [IFC4+]
  |-- IfcFeatureElement
  |   |-- IfcFeatureElementSubtraction
  |   |   |-- IfcOpeningElement [moved here in IFC4]
  |   |   |-- IfcVoidingFeature [IFC4+]
  |   |-- IfcFeatureElementAddition
  |   |   |-- IfcProjectionElement
  |   |-- IfcSurfaceFeature [IFC4+]
  |
  |-- IfcCivilElement [IFC4] / absorbed into specific entities in IFC4X3
  |-- IfcGeographicElement [IFC4+]
  |-- IfcVirtualElement
  |-- IfcElementAssembly
  |
  |-- [IFC4X3 NEW infrastructure elements]
  |-- IfcBuiltSystem [IFC4X3]
  |-- IfcBearing [IFC4X3]
  |-- IfcImpactProtectionDevice [IFC4X3]
  |-- IfcSign [IFC4X3]
  |-- IfcGeotechnicalElement [IFC4X3]
  |   |-- IfcGeotechnicalStratum [IFC4X3]
  |   |-- IfcBorehole [IFC4X3]
  |-- IfcLinearElement [IFC4X3] (under IfcProduct, not IfcElement)
```

---

## 5. Type Objects

### 5.1 Type Object Changes

| Type Entity | IFC2x3 | IFC4 | IFC4X3 | Notes |
|-------------|--------|------|--------|-------|
| IfcDoorStyle | YES | Deprecated | REMOVED | Use IfcDoorType |
| IfcWindowStyle | YES | Deprecated | REMOVED | Use IfcWindowType |
| IfcDoorType | - | YES | YES | Replaces IfcDoorStyle |
| IfcWindowType | - | YES | YES | Replaces IfcWindowStyle |
| IfcBuildingElementType | YES (abstract) | YES (abstract) | RENAMED to IfcBuiltElementType | Follows IfcBuiltElement rename |
| IfcTypeProcess | - | YES | YES | |
| IfcTypeResource | - | YES | YES | |

**Code pattern for version-safe type handling:**
```python
def get_door_type_class(schema_version):
    """Return the correct door type class name for the schema version."""
    if schema_version == "IFC2X3":
        return "IfcDoorStyle"
    else:  # IFC4, IFC4X3
        return "IfcDoorType"

def get_window_type_class(schema_version):
    """Return the correct window type class name for the schema version."""
    if schema_version == "IFC2X3":
        return "IfcWindowStyle"
    else:  # IFC4, IFC4X3
        return "IfcWindowType"

# Usage
model = ifcopenshell.open("some_file.ifc")
door_type_class = get_door_type_class(model.schema)
door_types = model.by_type(door_type_class)
```

---

## 6. Relationship Types (with code examples)

### 6.1 Relationship Overview

All IFC relationships inherit from `IfcRelationship`. The key categories:

```
IfcRelationship (abstract)
  |-- IfcRelDecomposes (abstract)
  |   |-- IfcRelAggregates
  |   |-- IfcRelNests
  |   |-- IfcRelProjectsElement [IFC2x3 only, moved in IFC4]
  |   |-- IfcRelVoidsElement [IFC2x3 only, moved in IFC4]
  |
  |-- IfcRelAssigns (abstract)
  |   |-- IfcRelAssignsToActor
  |   |-- IfcRelAssignsToControl
  |   |-- IfcRelAssignsToGroup
  |   |   |-- IfcRelAssignsToGroupByFactor [IFC4+]
  |   |-- IfcRelAssignsToProcess
  |   |-- IfcRelAssignsToProduct
  |   |-- IfcRelAssignsToResource
  |
  |-- IfcRelConnects (abstract)
  |   |-- IfcRelContainedInSpatialStructure
  |   |-- IfcRelConnectsElements
  |   |-- IfcRelConnectsPathElements
  |   |-- IfcRelConnectsPorts
  |   |-- IfcRelConnectsPortToElement
  |   |-- IfcRelConnectsStructuralActivity
  |   |-- IfcRelConnectsStructuralMember
  |   |-- IfcRelConnectsWithRealizingElements
  |   |-- IfcRelCoversSpaces
  |   |-- IfcRelCoversBldgElements
  |   |-- IfcRelFillsElement
  |   |-- IfcRelFlowControlElements
  |   |-- IfcRelInterferesElements [IFC4+]
  |   |-- IfcRelReferencedInSpatialStructure [IFC4+]
  |   |-- IfcRelSequence
  |   |-- IfcRelServicesBuildings
  |   |-- IfcRelSpaceBoundary
  |   |   |-- IfcRelSpaceBoundary1stLevel [IFC4+]
  |   |   |-- IfcRelSpaceBoundary2ndLevel [IFC4+]
  |   |-- IfcRelVoidsElement [IFC4+, moved from IfcRelDecomposes]
  |   |-- IfcRelProjectsElement [IFC4+, moved from IfcRelDecomposes]
  |
  |-- IfcRelAssociates (abstract)
  |   |-- IfcRelAssociatesClassification
  |   |-- IfcRelAssociatesConstraint [IFC4+]
  |   |-- IfcRelAssociatesDocument
  |   |-- IfcRelAssociatesLibrary
  |   |-- IfcRelAssociatesMaterial
  |   |-- IfcRelAssociatesApproval [IFC4+]
  |   |-- IfcRelAssociatesProfileDef [IFC4X3]
  |
  |-- IfcRelDefines (abstract)
      |-- IfcRelDefinesByProperties
      |-- IfcRelDefinesByType
      |-- IfcRelDefinesByObject [IFC4+]
      |-- IfcRelDefinesByTemplate [removed concept - properties only]
```

### 6.2 IfcRelAggregates (Spatial Decomposition)

**Purpose**: Defines whole-part relationships (a building is decomposed into storeys).

**Stable across all versions.**

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Create spatial structure
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="Demo")
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="Bldg")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="GF")

# Aggregate: Project -> Site -> Building -> Storey
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])

# Query aggregation via inverse attributes
# IFC2x3: IfcObjectDefinition.IsDecomposedBy (SET OF IfcRelDecomposes)
# IFC4+:  IfcObjectDefinition.IsDecomposedBy (SET OF IfcRelAggregates)
for rel in building.IsDecomposedBy:
    for child in rel.RelatedObjects:
        print(f"Building contains: {child.Name}")  # "GF"

# Upward traversal
for rel in storey.Decomposes:
    print(f"Storey is in: {rel.RelatingObject.Name}")  # "Bldg"

# Low-level creation (without API):
rel_agg = model.createIfcRelAggregates(
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=None,  # Optional in IFC4+
    RelatingObject=building,
    RelatedObjects=[storey]
)
```

**Version difference in inverse attribute names:**

| Inverse Attribute | IFC2x3 | IFC4 / IFC4X3 |
|-------------------|--------|----------------|
| IfcObjectDefinition.IsDecomposedBy | Returns IfcRelDecomposes instances | Returns IfcRelAggregates instances |
| IfcObjectDefinition.Decomposes | Returns IfcRelDecomposes instances | Returns IfcRelAggregates instances |

In IFC2x3, `IsDecomposedBy` and `Decomposes` return `IfcRelDecomposes` (which includes IfcRelAggregates, IfcRelNests, etc.). In IFC4+, the inverse is specifically typed to `IfcRelAggregates` only, with separate inverses for nesting.

### 6.3 IfcRelContainedInSpatialStructure (Element Containment)

**Purpose**: Places elements within a spatial structure element (e.g., a wall in a storey).

**Stable across all versions, but the set of valid containers expands.**

```python
# Assign a wall to a storey
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 1")

ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# Query containment
for rel in storey.ContainsElements:
    for element in rel.RelatedElements:
        print(f"Storey contains: {element.is_a()} - {element.Name}")

# Which container is an element in?
for rel in wall.ContainedInStructure:
    print(f"Wall is in: {rel.RelatingStructure.Name}")

# Low-level:
rel_contains = model.createIfcRelContainedInSpatialStructure(
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=None,
    RelatingStructure=storey,
    RelatedElements=[wall]
)
```

**Version difference for valid containers:**

| Valid RelatingStructure | IFC2x3 | IFC4 | IFC4X3 |
|------------------------|--------|------|--------|
| IfcSite | YES | YES | YES |
| IfcBuilding | YES | YES | YES |
| IfcBuildingStorey | YES | YES | YES |
| IfcSpace | YES | YES | YES |
| IfcExternalSpatialElement | - | YES | YES |
| IfcFacility (IfcRoad, etc.) | - | - | YES |
| IfcFacilityPart (IfcRoadPart, etc.) | - | - | YES |

### 6.4 IfcRelAssociatesMaterial (Material Assignment)

**Purpose**: Associates material definitions to elements or types.

```python
# Create material
material = ifcopenshell.api.run("material.add_material", model, name="Concrete")

# Assign to wall
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material)

# Material layer sets (for walls, slabs)
material_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Wall Layers", set_type="IfcMaterialLayerSet")
layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=material_set, material=material)
layer.LayerThickness = 200.0  # mm (depends on project units)

ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material_set)

# Query materials
for rel in wall.HasAssociations:
    if rel.is_a("IfcRelAssociatesMaterial"):
        mat = rel.RelatingMaterial
        print(f"Material: {mat.is_a()} - {mat.Name if hasattr(mat, 'Name') else 'N/A'}")
```

**Version differences:**

| Material Concept | IFC2x3 | IFC4 | IFC4X3 |
|-----------------|--------|------|--------|
| IfcMaterial | YES | YES | YES |
| IfcMaterialLayer | YES | YES | YES |
| IfcMaterialLayerSet | YES | YES | YES |
| IfcMaterialLayerSetUsage | YES | YES | YES |
| IfcMaterialProfile | - | YES | YES |
| IfcMaterialProfileSet | - | YES | YES |
| IfcMaterialProfileSetUsage | - | YES | YES |
| IfcMaterialConstituent | - | YES | YES |
| IfcMaterialConstituentSet | - | YES | YES |
| IfcMaterialList | YES | YES (deprecated) | YES (deprecated) |
| Material.HasProperties (psets) | - | YES | YES |
| Material.HasRepresentation | - | YES | YES |

**Key addition in IFC4**: `IfcMaterialProfileSet` (for columns, beams - cross-section profiles) and `IfcMaterialConstituentSet` (for complex elements with named material parts).

### 6.5 IfcRelDefinesByProperties (Property Sets)

**Purpose**: Assigns property sets to object instances.

```python
# Create a property set with properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset,
    properties={
        "IsExternal": True,
        "FireRating": "REI60",
        "ThermalTransmittance": 0.25,
        "LoadBearing": True
    })

# Query property sets
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset_def = rel.RelatingPropertyDefinition
        if pset_def.is_a("IfcPropertySet"):
            print(f"PSet: {pset_def.Name}")
            for prop in pset_def.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    print(f"  {prop.Name} = {prop.NominalValue.wrappedValue}")

# Utility shortcut
import ifcopenshell.util.element
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"IsExternal": True, "FireRating": "REI60", ...}}
```

**Version differences:**

| Aspect | IFC2x3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| IfcRelDefinesByProperties | YES | YES | YES |
| RelatingPropertyDefinition type | IfcPropertySetDefinition | IfcPropertySetDefinitionSelect | IfcPropertySetDefinitionSelect |
| Can assign multiple psets in one rel | NO (one pset per rel) | YES (via IfcPropertySetDefinitionSet) | YES |
| IfcPropertySetTemplate | - | YES | YES |
| IfcQuantityCount.Formula | - | YES | YES |
| Property value types | Limited | Extended | Extended |
| IfcPropertyTemplateDefinition | - | YES | YES |

**IFC2x3 to IFC4 change**: In IFC2x3, `RelatingPropertyDefinition` is a single `IfcPropertySetDefinition`. In IFC4+, it is `IfcPropertySetDefinitionSelect` which can be either a single property set or a set of property sets.

### 6.6 IfcRelDefinesByType (Type Assignment)

**Purpose**: Assigns a type object to occurrences.

```python
# Create a wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm",
    predefined_type="STANDARD")

# Assign type to wall occurrence
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# Query type assignment
for rel in wall.IsTypedBy:  # IFC4+ inverse name
    print(f"Type: {rel.RelatingType.Name}")

# IFC2x3 inverse name difference:
# wall.IsDefinedBy -> check for IfcRelDefinesByType instances
# (IsTypedBy does not exist in IFC2x3)
```

**Version differences:**

| Aspect | IFC2x3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| Relationship entity | IfcRelDefinesByType | IfcRelDefinesByType | IfcRelDefinesByType |
| Inverse on occurrence | IsDefinedBy (shared with ByProperties) | IsTypedBy (dedicated) | IsTypedBy (dedicated) |
| Inverse on type | ObjectTypeOf | Types (renamed) | Types (renamed) |
| IfcDoorStyle/IfcWindowStyle | Used for doors/windows | Deprecated, use IfcDoorType/IfcWindowType | IfcDoorType/IfcWindowType only |

**Version-safe type query:**
```python
def get_element_type(element):
    """Get the type of an element, version-safe."""
    schema = element.wrapped_data.file.schema

    if schema == "IFC2X3":
        # In IFC2x3, type relations are in IsDefinedBy
        for rel in element.IsDefinedBy:
            if rel.is_a("IfcRelDefinesByType"):
                return rel.RelatingType
    else:
        # IFC4+: dedicated IsTypedBy inverse
        for rel in element.IsTypedBy:
            return rel.RelatingType
    return None

# Better: use ifcopenshell.util.element (handles version differences)
import ifcopenshell.util.element
element_type = ifcopenshell.util.element.get_type(wall)
```

### 6.7 IfcRelAssigns Variants

**Purpose**: Assigns objects to actors, controls, groups, processes, products, or resources.

```python
# IfcRelAssignsToGroup - grouping elements
group = ifcopenshell.api.run("group.add_group", model, name="External Walls")
ifcopenshell.api.run("group.assign_group", model,
    products=[wall], group=group)

# Query groups
for rel in wall.HasAssignments:
    if rel.is_a("IfcRelAssignsToGroup"):
        print(f"Group: {rel.RelatingGroup.Name}")
```

| IfcRelAssigns Variant | IFC2x3 | IFC4 | IFC4X3 | Purpose |
|-----------------------|--------|------|--------|---------|
| IfcRelAssignsToActor | YES | YES | YES | Assign to responsible actor |
| IfcRelAssignsToControl | YES | YES | YES | Assign to control (cost, schedule) |
| IfcRelAssignsToGroup | YES | YES | YES | Group membership |
| IfcRelAssignsToGroupByFactor | - | YES | YES | Group with weighting factor |
| IfcRelAssignsToProcess | YES | YES | YES | Assign to process/task |
| IfcRelAssignsToProduct | YES | YES | YES | Assign to product |
| IfcRelAssignsToResource | YES | YES | YES | Assign to resource |

### 6.8 IfcRelConnects Variants

| IfcRelConnects Variant | IFC2x3 | IFC4 | IFC4X3 | Purpose |
|------------------------|--------|------|--------|---------|
| IfcRelConnectsElements | YES | YES | YES | Element-to-element connection |
| IfcRelConnectsPathElements | YES | YES | YES | Wall-to-wall path connection |
| IfcRelConnectsPorts | YES | YES | YES | Port-to-port (MEP) |
| IfcRelConnectsPortToElement | YES | YES | YES | Port to element |
| IfcRelConnectsStructuralActivity | YES | YES | YES | Structural loads |
| IfcRelConnectsStructuralMember | YES | YES | YES | Structural members |
| IfcRelConnectsWithRealizingElements | YES | YES | YES | Connection with fasteners |
| IfcRelConnectsWithEccentricity | YES | YES | YES | Eccentric structural connection |
| IfcRelCoversSpaces | YES | YES | YES | Space coverings |
| IfcRelCoversBldgElements | YES | YES | YES | Element coverings |
| IfcRelFillsElement | YES | YES | YES | Door/window fills opening |
| IfcRelFlowControlElements | YES | YES | YES | Flow control relationship |
| IfcRelInterferesElements | - | YES | YES | Clash/interference |
| IfcRelReferencedInSpatialStructure | - | YES | YES | Referenced (not contained) |
| IfcRelSequence | YES | YES | YES | Process sequencing |
| IfcRelServicesBuildings | YES | YES | YES | System serves building |
| IfcRelSpaceBoundary | YES | YES | YES | Space boundary surfaces |
| IfcRelSpaceBoundary1stLevel | - | YES | YES | 1st level space boundary |
| IfcRelSpaceBoundary2ndLevel | - | YES | YES | 2nd level space boundary |
| IfcRelPositions | - | - | YES | Positioning along alignment |

### 6.9 IfcRelVoidsElement (Openings)

**Purpose**: Creates a void (opening) in an element.

**IMPORTANT**: In IFC2x3, IfcRelVoidsElement inherits from `IfcRelDecomposes`. In IFC4+, it was moved to inherit from `IfcRelConnects`.

```python
# Create opening in a wall
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# Create the void relationship
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Query openings
for rel in wall.HasOpenings:
    opening = rel.RelatedOpeningElement
    print(f"Opening: {opening.Name}")

# Low-level:
rel_voids = model.createIfcRelVoidsElement(
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=None,
    RelatingBuildingElement=wall,
    RelatedOpeningElement=opening
)
```

| Aspect | IFC2x3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| IfcRelVoidsElement parent | IfcRelDecomposes | IfcRelConnects | IfcRelConnects |
| Inverse on element | HasOpenings | HasOpenings | HasOpenings |
| IfcOpeningElement parent | IfcFeatureElementSubtraction | IfcFeatureElementSubtraction | IfcFeatureElementSubtraction |
| IfcVoidingFeature | - | YES | YES |

### 6.10 IfcRelFillsElement (Filling Openings)

**Purpose**: Indicates an element (door, window) fills an opening.

```python
# Fill the opening with a door
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 1")

# Assign door to fill the opening
rel_fills = model.createIfcRelFillsElement(
    GlobalId=ifcopenshell.guid.new(),
    OwnerHistory=None,
    RelatingOpeningElement=opening,
    RelatedBuildingElement=door
)

# Query fills
for rel in opening.HasFillings:
    print(f"Filled by: {rel.RelatedBuildingElement.Name}")

# Reverse: what opening does a door fill?
for rel in door.FillsVoids:
    print(f"Fills: {rel.RelatingOpeningElement.Name}")
```

**Stable across all versions.** The entity name and attributes remain the same.

---

## 7. Key Attribute Changes Between Versions

### 7.1 IfcProject Attributes

| Attribute | IFC2x3 | IFC4 | IFC4X3 |
|-----------|--------|------|--------|
| GlobalId | YES | YES | YES |
| OwnerHistory | MANDATORY | OPTIONAL | OPTIONAL |
| Name | YES | YES | YES |
| Description | YES | YES | YES |
| ObjectType | YES | - (moved to IfcObject) | - |
| LongName | YES | YES | YES |
| Phase | YES | YES | YES |
| RepresentationContexts | YES | YES | YES |
| UnitsInContext | YES | YES | YES |

### 7.2 IfcSite Attributes

| Attribute | IFC2x3 | IFC4 | IFC4X3 |
|-----------|--------|------|--------|
| RefLatitude | YES | YES | YES |
| RefLongitude | YES | YES | YES |
| RefElevation | YES | YES | YES |
| LandTitleNumber | YES | YES | YES |
| SiteAddress | YES | YES | YES |
| CompositionType | YES (mandatory) | YES (mandatory) | YES (optional in some contexts) |

### 7.3 IfcWall / IfcWallType PredefinedType Values

| PredefinedType | IFC2x3 | IFC4 | IFC4X3 |
|----------------|--------|------|--------|
| STANDARD | YES | YES | YES |
| POLYGONAL | YES | YES | YES |
| SHEAR | YES | YES | YES |
| ELEMENTEDWALL | - | YES | YES |
| PLUMBINGWALL | - | YES | YES |
| PARTITIONING | - | YES | YES |
| MOVABLE | - | - | YES |
| PARAPET | - | - | YES |
| SOLIDWALL | - | - | YES |
| RETAININGWALL | - | - | YES |
| WAVEWALL | - | - | YES |
| USERDEFINED | YES | YES | YES |
| NOTDEFINED | YES | YES | YES |

### 7.4 Placement Changes

| Concept | IFC2x3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| IfcLocalPlacement | YES | YES | YES |
| IfcGridPlacement | YES | YES | YES |
| IfcLinearPlacement | - | - | YES |
| IfcObjectPlacement (abstract) | YES | YES | YES |
| PlacementRelTo (relative) | YES | YES | YES |

**IFC4X3 addition**: `IfcLinearPlacement` allows placing elements along an alignment curve (critical for infrastructure).

```python
# IFC4X3 linear placement example concept:
# Element placed at station 1+500.00 on alignment, 3m offset
# This requires an IfcAlignment entity (IFC4X3 only)
```

---

## 8. New Entity Categories in IFC4X3

### 8.1 Alignment Entities (completely new in IFC4X3)

| Entity | Purpose |
|--------|---------|
| IfcAlignment | Main alignment entity |
| IfcAlignmentHorizontal | Horizontal alignment |
| IfcAlignmentVertical | Vertical alignment |
| IfcAlignmentCant | Rail cant |
| IfcAlignmentSegment | Segment of alignment |
| IfcAlignmentHorizontalSegment | Horizontal segment |
| IfcAlignmentVerticalSegment | Vertical segment |
| IfcAlignmentCantSegment | Cant segment |

### 8.2 Georeferencing Entities (IFC4X3)

| Entity | IFC2x3 | IFC4 | IFC4X3 |
|--------|--------|------|--------|
| IfcCoordinateOperation | - | YES | YES |
| IfcMapConversion | - | YES | YES |
| IfcProjectedCRS | - | YES | YES |
| IfcMapConversionScaled | - | - | YES |
| IfcRigidOperation | - | - | YES |

### 8.3 Geotechnical Entities (IFC4X3)

| Entity | Purpose |
|--------|---------|
| IfcGeotechnicalElement | Base for geotechnical |
| IfcGeotechnicalStratum | Soil/rock layer |
| IfcBorehole | Borehole investigation |
| IfcGeomodel | Geological model |
| IfcGeoScienceElement | Geoscience features |

---

## 9. Geometry Representation Changes

| Concept | IFC2x3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| IfcAdvancedBrep | - | YES | YES |
| IfcAdvancedBrepWithVoids | - | YES | YES |
| IfcTessellatedFaceSet | - | YES | YES |
| IfcTriangulatedFaceSet | - | YES | YES |
| IfcPolygonalFaceSet | - | YES | YES |
| IfcIndexedPolyCurve | - | YES | YES |
| IfcBSplineSurface | Limited | Extended | Extended |
| IfcCylindricalSurface | - | YES | YES |
| IfcToroidalSurface | - | YES | YES |
| IfcSphericalSurface | - | YES | YES |
| IfcSectionedSurface | - | - | YES |
| IfcDirectrixCurveSweptAreaSolid | - | - | YES |
| IfcSectionedSolidHorizontal | - | YES | YES |
| IfcFixedReferenceSweptAreaSolid | - | YES | YES |
| IfcSegmentedReferenceCurve | - | - | YES |
| IfcGradientCurve | - | - | YES |

**Key for IfcOpenShell geometry processing:**
```python
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()

# IFC2x3 models: mostly IfcExtrudedAreaSolid, IfcFacetedBrep, CSG
# IFC4 models: adds tessellated geometry, advanced BRep, NURBS
# IFC4X3 models: adds swept solids along alignments

# Settings to control tessellation
settings.set(settings.USE_WORLD_COORDS, True)
settings.set(settings.APPLY_DEFAULT_MATERIALS, True)

# Process geometry
shape = ifcopenshell.geom.create_shape(settings, wall)
vertices = shape.geometry.verts  # Flat list: [x1,y1,z1, x2,y2,z2, ...]
faces = shape.geometry.faces      # Flat list: [i1,i2,i3, i4,i5,i6, ...]
```

---

## 10. Migration Patterns

### 10.1 IFC2x3 to IFC4 Migration

| Change | IFC2x3 Pattern | IFC4 Pattern |
|--------|---------------|--------------|
| OwnerHistory | Always required | Can be None |
| Door/Window types | IfcDoorStyle, IfcWindowStyle | IfcDoorType, IfcWindowType |
| Type inverse | IsDefinedBy (mixed) | IsTypedBy (dedicated) |
| Type inverse on type | ObjectTypeOf | Types |
| Property set assignment | Single pset per rel | Multiple psets per rel possible |
| IfcProject parent | IfcObject | IfcContext |
| Spatial element abstract | IfcSpatialStructureElement only | IfcSpatialElement (new parent) |
| Material profiles | Not available | IfcMaterialProfileSet |
| Material constituents | Not available | IfcMaterialConstituentSet |
| Tessellated geometry | Not available | IfcTessellatedFaceSet |
| Georeferencing | Not available | IfcMapConversion + IfcProjectedCRS |
| IfcRelVoidsElement parent | IfcRelDecomposes | IfcRelConnects |
| IfcRelProjectsElement parent | IfcRelDecomposes | IfcRelConnects |
| Property templates | Not available | IfcPropertySetTemplate |
| Space boundaries | IfcRelSpaceBoundary only | +1stLevel, +2ndLevel subtypes |
| Interference | Not available | IfcRelInterferesElements |
| Spatial referencing | Not available | IfcRelReferencedInSpatialStructure |

### 10.2 IFC4 to IFC4X3 Migration

| Change | IFC4 Pattern | IFC4X3 Pattern |
|--------|-------------|----------------|
| IfcBuildingElement | IfcBuildingElement | Renamed to IfcBuiltElement |
| IfcBuildingElementType | IfcBuildingElementType | Renamed to IfcBuiltElementType |
| StandardCase entities | IfcWallStandardCase, etc. | REMOVED (use IfcWall, etc.) |
| Infrastructure spatial | Not available | IfcFacility, IfcFacilityPart subtypes |
| Alignment | Not available | IfcAlignment + segments |
| Linear placement | Not available | IfcLinearPlacement |
| IfcCivilElement | Generic catch-all | Replaced by specific entities |
| IfcTransportElement | Single entity | Restructured: IfcVehicle, IfcTransportationDevice |
| Geotechnical | Not available | IfcGeotechnicalElement subtypes |
| Signals | Not available | IfcSignal, IfcSign |
| IfcBuiltSystem | Not available | New: grouping built elements |
| IfcRelPositions | Not available | New: positioning along alignment |

### 10.3 Version-Safe Coding Patterns

#### Pattern 1: Schema-Aware Entity Creation
```python
def create_wall(model, name, predefined_type="STANDARD"):
    """Create a wall in any IFC version."""
    schema = model.schema

    if schema == "IFC2X3":
        # IFC2x3 has no PredefinedType on IfcWall occurrence
        wall = ifcopenshell.api.run("root.create_entity", model,
            ifc_class="IfcWall", name=name)
    else:
        # IFC4/IFC4X3: PredefinedType available
        wall = ifcopenshell.api.run("root.create_entity", model,
            ifc_class="IfcWall", name=name,
            predefined_type=predefined_type)
    return wall
```

#### Pattern 2: Schema-Aware Type Queries
```python
def get_all_building_elements(model):
    """Get all building elements regardless of schema version."""
    schema = model.schema

    if schema == "IFC4X3":
        # IfcBuiltElement in IFC4X3
        return model.by_type("IfcBuiltElement")
    else:
        # IfcBuildingElement in IFC2x3 / IFC4
        return model.by_type("IfcBuildingElement")

def get_all_door_types(model):
    """Get all door types regardless of schema version."""
    schema = model.schema

    if schema == "IFC2X3":
        return model.by_type("IfcDoorStyle")
    else:
        # IFC4 has both IfcDoorStyle (deprecated) and IfcDoorType
        # IFC4X3 only has IfcDoorType
        types = model.by_type("IfcDoorType")
        if schema == "IFC4":
            # Also check deprecated style (some files still use it)
            types += model.by_type("IfcDoorStyle")
        return types
```

#### Pattern 3: Schema-Aware Spatial Traversal
```python
def get_spatial_children(element):
    """Get all spatial children of a spatial element, any version."""
    children = []
    for rel in element.IsDecomposedBy:
        for child in rel.RelatedObjects:
            children.append(child)
    return children

def get_contained_elements(spatial_element):
    """Get all elements contained in a spatial structure element."""
    elements = []
    for rel in spatial_element.ContainsElements:
        for element in rel.RelatedElements:
            elements.append(element)
    return elements

def get_full_spatial_tree(project):
    """Recursively get the full spatial hierarchy."""
    result = {"entity": project, "children": []}
    for child in get_spatial_children(project):
        result["children"].append(get_full_spatial_tree(child))
    return result
```

#### Pattern 4: Version Detection and Branching
```python
def process_ifc_file(filepath):
    """Process an IFC file with version-aware logic."""
    model = ifcopenshell.open(filepath)
    schema = model.schema  # "IFC2X3", "IFC4", or "IFC4X3"

    print(f"Schema: {schema}")
    print(f"Entities: {len(model.by_type('IfcRoot'))}")

    # Version-specific processing
    if schema == "IFC2X3":
        # Must handle OwnerHistory everywhere
        # Use IfcDoorStyle/IfcWindowStyle
        # No tessellated geometry
        # No material profiles
        pass
    elif schema == "IFC4":
        # OwnerHistory optional
        # Use IfcDoorType/IfcWindowType (or both)
        # Tessellated geometry available
        # Material profiles available
        # StandardCase entities exist
        pass
    elif schema == "IFC4X3":
        # Same as IFC4 plus:
        # IfcBuiltElement instead of IfcBuildingElement
        # No StandardCase entities
        # Infrastructure entities available
        # Alignment available
        # Linear placement available
        pass

    return model
```

#### Pattern 5: Using ifcopenshell.util for Version Abstraction
```python
import ifcopenshell.util.element
import ifcopenshell.util.placement
import ifcopenshell.util.unit
import ifcopenshell.util.selector

# These utilities handle version differences internally:

# Get all property sets (works for all versions)
psets = ifcopenshell.util.element.get_psets(wall)

# Get element type (handles IsDefinedBy vs IsTypedBy)
element_type = ifcopenshell.util.element.get_type(wall)

# Get material (handles all material types)
material = ifcopenshell.util.element.get_material(wall)

# Get container (spatial structure)
container = ifcopenshell.util.element.get_container(wall)

# Get placement matrix (4x4 numpy array)
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)

# Get project units
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
```

---

## 11. IfcOpenShell API Module Version Handling

The `ifcopenshell.api.run()` module generally handles version differences internally. However, some API calls are version-specific:

| API Module | IFC2x3 | IFC4 | IFC4X3 | Notes |
|-----------|--------|------|--------|-------|
| root.create_entity | YES | YES | YES | Entity names differ per version |
| spatial.assign_container | YES | YES | YES | Container types differ |
| aggregate.assign_object | YES | YES | YES | Works the same |
| type.assign_type | YES | YES | YES | Type entity names differ |
| pset.add_pset | YES | YES | YES | |
| pset.edit_pset | YES | YES | YES | |
| material.add_material | YES | YES | YES | |
| material.add_material_set | YES | YES | YES | Profile sets IFC4+ only |
| geometry.add_wall_representation | YES | YES | YES | |
| void.add_opening | YES | YES | YES | |
| owner.create_owner_history | YES | YES (optional) | YES (optional) | Mandatory only in IFC2x3 |
| group.add_group | YES | YES | YES | |
| classification.add_classification | YES | YES | YES | |

---

## 12. Schema Introspection with IfcOpenShell

```python
import ifcopenshell
import ifcopenshell.ifcopenshell_wrapper as W

# Get schema object
schema_2x3 = W.schema_by_name("IFC2X3")
schema_4 = W.schema_by_name("IFC4")
schema_4x3 = W.schema_by_name("IFC4X3")

# Check if entity exists in schema
def entity_exists(schema_name, entity_name):
    """Check if an entity exists in a given schema."""
    schema = W.schema_by_name(schema_name)
    try:
        schema.declaration_by_name(entity_name)
        return True
    except RuntimeError:
        return False

# Examples
print(entity_exists("IFC2X3", "IfcWallStandardCase"))  # True
print(entity_exists("IFC4", "IfcWallStandardCase"))     # True
print(entity_exists("IFC4X3", "IfcWallStandardCase"))   # False

print(entity_exists("IFC2X3", "IfcBuildingElement"))    # True
print(entity_exists("IFC4X3", "IfcBuildingElement"))    # False
print(entity_exists("IFC4X3", "IfcBuiltElement"))       # True

print(entity_exists("IFC2X3", "IfcFacility"))           # False
print(entity_exists("IFC4X3", "IfcFacility"))           # True

# Get all entity names in a schema
def list_all_entities(schema_name):
    """List all entity declarations in a schema."""
    schema = W.schema_by_name(schema_name)
    return [d.name() for d in schema.declarations()
            if isinstance(d, W.entity)]

# Get attributes of an entity
def get_entity_attributes(schema_name, entity_name):
    """Get all attributes of an entity in a schema."""
    schema = W.schema_by_name(schema_name)
    entity = schema.declaration_by_name(entity_name)
    return [(a.name(), a.type_of_attribute().declared_type().name()
             if hasattr(a.type_of_attribute(), 'declared_type') else str(a.type_of_attribute()))
            for a in entity.all_attributes()]

# Get entity hierarchy
def get_supertypes(schema_name, entity_name):
    """Get the supertype chain of an entity."""
    schema = W.schema_by_name(schema_name)
    entity = schema.declaration_by_name(entity_name)
    chain = []
    while entity.supertype():
        entity = entity.supertype()
        chain.append(entity.name())
    return chain

# Example: IfcWall supertype chain
# IFC2x3: ['IfcBuildingElement', 'IfcElement', 'IfcProduct', 'IfcObject', 'IfcObjectDefinition', 'IfcRoot']
# IFC4:   ['IfcBuildingElement', 'IfcElement', 'IfcProduct', 'IfcObject', 'IfcObjectDefinition', 'IfcRoot']
# IFC4X3: ['IfcBuiltElement', 'IfcElement', 'IfcProduct', 'IfcObject', 'IfcObjectDefinition', 'IfcRoot']
```

---

## 13. Common Pitfalls and Anti-Patterns

### Pitfall 1: Hardcoding Entity Names
```python
# BAD: Breaks on IFC4X3
elements = model.by_type("IfcBuildingElement")

# GOOD: Version-safe
if model.schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")
```

### Pitfall 2: Assuming OwnerHistory Exists
```python
# BAD: Fails on IFC4+ files where OwnerHistory is None
print(wall.OwnerHistory.CreationDate)

# GOOD: Check for None
if wall.OwnerHistory:
    print(wall.OwnerHistory.CreationDate)
```

### Pitfall 3: Using Wrong Type Entity for Doors/Windows
```python
# BAD: IfcDoorStyle doesn't exist in IFC4X3
door_styles = model.by_type("IfcDoorStyle")

# GOOD: Version-aware
if model.schema == "IFC2X3":
    door_types = model.by_type("IfcDoorStyle")
else:
    door_types = model.by_type("IfcDoorType")
```

### Pitfall 4: Querying StandardCase in IFC4X3
```python
# BAD: IfcWallStandardCase doesn't exist in IFC4X3
std_walls = model.by_type("IfcWallStandardCase")

# GOOD: Just use IfcWall
walls = model.by_type("IfcWall")
```

### Pitfall 5: Assuming IsTypedBy Exists in IFC2x3
```python
# BAD: IsTypedBy doesn't exist in IFC2x3
for rel in element.IsTypedBy:  # AttributeError in IFC2x3
    pass

# GOOD: Use utility function
import ifcopenshell.util.element
element_type = ifcopenshell.util.element.get_type(element)
```

### Pitfall 6: Using Infrastructure Entities in Wrong Schema
```python
# BAD: IfcRoad doesn't exist in IFC4
road = ifcopenshell.api.run("root.create_entity", ifc4_model,
    ifc_class="IfcRoad")  # RuntimeError

# GOOD: Check schema first
if model.schema == "IFC4X3":
    road = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcRoad", name="Highway")
else:
    raise ValueError("IfcRoad requires IFC4X3 schema")
```

---

## 14. Quick Reference: Entity Existence Matrix

### Core Spatial Entities
| Entity | IFC2X3 | IFC4 | IFC4X3 |
|--------|:------:|:----:|:------:|
| IfcProject | Y | Y | Y |
| IfcProjectLibrary | - | Y | Y |
| IfcSite | Y | Y | Y |
| IfcBuilding | Y | Y | Y |
| IfcBuildingStorey | Y | Y | Y |
| IfcSpace | Y | Y | Y |
| IfcSpatialZone | - | Y | Y |
| IfcExternalSpatialElement | - | Y | Y |
| IfcFacility | - | - | Y |
| IfcFacilityPart | - | - | Y |
| IfcRoad | - | - | Y |
| IfcRailway | - | - | Y |
| IfcBridge | - | - | Y |
| IfcMarineFacility | - | - | Y |

### Building Elements
| Entity | IFC2X3 | IFC4 | IFC4X3 |
|--------|:------:|:----:|:------:|
| IfcBuildingElement (abstract) | Y | Y | - (renamed) |
| IfcBuiltElement (abstract) | - | - | Y |
| IfcWall | Y | Y | Y |
| IfcWallStandardCase | Y | Y | - (removed) |
| IfcColumn | Y | Y | Y |
| IfcColumnStandardCase | - | Y | - (removed) |
| IfcBeam | Y | Y | Y |
| IfcBeamStandardCase | - | Y | - (removed) |
| IfcSlab | Y | Y | Y |
| IfcSlabStandardCase | - | Y | - (removed) |
| IfcDoor | Y | Y | Y |
| IfcWindow | Y | Y | Y |
| IfcRoof | Y | Y | Y |
| IfcStair | Y | Y | Y |
| IfcRailing | Y | Y | Y |
| IfcFooting | Y | Y | Y |
| IfcPile | Y | Y | Y |
| IfcCurtainWall | Y | Y | Y |
| IfcPlate | Y | Y | Y |
| IfcMember | Y | Y | Y |
| IfcChimney | - | Y | Y |
| IfcShadingDevice | - | Y | Y |
| IfcCourse | - | - | Y |
| IfcEarthworksElement | - | - | Y |
| IfcKerb | - | - | Y |
| IfcPavement | - | - | Y |
| IfcRail | - | - | Y |
| IfcTrackElement | - | - | Y |

### Type Entities
| Entity | IFC2X3 | IFC4 | IFC4X3 |
|--------|:------:|:----:|:------:|
| IfcDoorStyle | Y | Y (deprecated) | - (removed) |
| IfcDoorType | - | Y | Y |
| IfcWindowStyle | Y | Y (deprecated) | - (removed) |
| IfcWindowType | - | Y | Y |
| IfcBuildingElementType (abstract) | Y | Y | - (renamed) |
| IfcBuiltElementType (abstract) | - | - | Y |
| IfcWallType | Y | Y | Y |
| IfcColumnType | Y | Y | Y |
| IfcBeamType | Y | Y | Y |
| IfcSlabType | Y | Y | Y |

### Relationship Entities
| Entity | IFC2X3 | IFC4 | IFC4X3 |
|--------|:------:|:----:|:------:|
| IfcRelAggregates | Y | Y | Y |
| IfcRelNests | Y | Y | Y |
| IfcRelContainedInSpatialStructure | Y | Y | Y |
| IfcRelDefinesByProperties | Y | Y | Y |
| IfcRelDefinesByType | Y | Y | Y |
| IfcRelDefinesByObject | - | Y | Y |
| IfcRelAssociatesMaterial | Y | Y | Y |
| IfcRelAssociatesClassification | Y | Y | Y |
| IfcRelAssociatesConstraint | - | Y | Y |
| IfcRelAssociatesProfileDef | - | - | Y |
| IfcRelVoidsElement | Y | Y | Y |
| IfcRelFillsElement | Y | Y | Y |
| IfcRelInterferesElements | - | Y | Y |
| IfcRelReferencedInSpatialStructure | - | Y | Y |
| IfcRelSpaceBoundary1stLevel | - | Y | Y |
| IfcRelSpaceBoundary2ndLevel | - | Y | Y |
| IfcRelPositions | - | - | Y |

---

## 15. Sources and Verification Notes

### Primary Sources Used
1. **IFC2x3 TC1 EXPRESS schema** (buildingSMART, 2007) - entity definitions, attribute types, inheritance
2. **IFC4 ADD2 TC1 EXPRESS schema** (buildingSMART/ISO 16739-1:2018) - entity additions, renames, deprecations
3. **IFC4.3.2.0 (IFC4X3) EXPRESS schema** (buildingSMART/ISO 16739-1:2024) - infrastructure extensions, entity renames, removals
4. **IfcOpenShell Python API** (v0.7.x/v0.8.x) - `ifcopenshell.api.run()`, `ifcopenshell.util.*`, schema introspection
5. **IfcOpenShell source code** (GitHub: IfcOpenShell/IfcOpenShell) - API module implementations

### Verification Recommendations
- ALWAYS verify entity existence using `ifcopenshell.ifcopenshell_wrapper.schema_by_name()` before using in production code
- Test code examples against actual IFC files of each schema version
- Check IfcOpenShell changelog for any API changes in newer versions
- The `ifcopenshell.util.element` module is the RECOMMENDED way to handle version differences as it abstracts schema variations

### Items Requiring Live Verification
- Exact entity counts per schema (approximate values given; use schema introspection for precise counts)
- Some IFC4X3 PredefinedType enum values (the schema is large; verify against EXPRESS schema)
- IfcOpenShell API coverage for newer IFC4X3 entities (some infrastructure entities may have limited API support)
- Attribute optionality changes between versions (some subtle changes in OPTIONAL vs mandatory)
