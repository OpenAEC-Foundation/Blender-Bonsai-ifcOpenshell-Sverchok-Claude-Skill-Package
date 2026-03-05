# IFC Schema Versions & Entity Hierarchy — IfcOpenShell Research

> Research document covering IFC2x3, IFC4, and IFC4.3 (IFC4x3) schema differences,
> entity hierarchy, relationship types, and migration patterns.

---

## Table of Contents

1. [Schema Version Overview](#1-schema-version-overview)
2. [Entity Hierarchy and Inheritance](#2-entity-hierarchy-and-inheritance)
3. [IFC2x3 vs IFC4 vs IFC4.3 Comparison](#3-ifc2x3-vs-ifc4-vs-ifc43-comparison)
4. [Relationship Types](#4-relationship-types)
5. [Migration Patterns](#5-migration-patterns)
6. [Version Comparison Tables](#6-version-comparison-tables)
7. [IfcOpenShell Code Examples](#7-ifcopenshell-code-examples)

---

## 1. Schema Version Overview

### IFC2x3 (2006)
- **Status**: Legacy, still most widely used in practice
- **Focus**: Building-centric (architecture, structural, MEP)
- **Entity count**: ~600 entities
- **ISO**: ISO/PAS 16739:2005
- **Key limitation**: No infrastructure support, limited spatial structure

### IFC4 (2013)
- **Status**: Current standard for buildings
- **Focus**: Enhanced building support, improved geometry, property sets
- **Entity count**: ~750 entities
- **ISO**: ISO 16739:2013, revised as ISO 16739-1:2018
- **Key additions**: IfcSpatialElement, enhanced parametric geometry, PredefinedType on elements, StandardCase subtypes

### IFC4.3 / IFC4x3 (2024)
- **Status**: Latest — ISO 16739-1:2024 (published April 2024)
- **Focus**: Infrastructure expansion (roads, bridges, railways, marine, tunnels)
- **Entity count**: ~900+ entities
- **Key additions**: IfcFacility subtypes, IfcAlignment, infrastructure elements, geotechnical elements
- **Note**: "IFC4.3" and "IFC4x3" refer to the same schema. BuildingSMART changed the naming convention from "x" notation to dot notation.

---

## 2. Entity Hierarchy and Inheritance

### Core Hierarchy (all versions)

```
IfcRoot (abstract)
├── IfcObjectDefinition (abstract)
│   ├── IfcObject (abstract)
│   │   ├── IfcProduct (abstract)
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
│   │   │   │   ├── IfcBuiltElement (was: IfcBuildingElement in IFC2x3)
│   │   │   │   ├── IfcCivilElement
│   │   │   │   ├── IfcDistributionElement
│   │   │   │   ├── IfcElementAssembly
│   │   │   │   ├── IfcElementComponent
│   │   │   │   ├── IfcFeatureElement
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
├── IfcPropertyDefinition
│   ├── IfcPropertySetDefinition
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

**IFC2x3 Spatial Hierarchy:**
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
└── IfcSpatialElement (abstract) ← NEW abstract supertype
    ├── IfcSpatialStructureElement (abstract)
    │   ├── IfcSite
    │   ├── IfcBuilding
    │   ├── IfcBuildingStorey
    │   └── IfcSpace
    ├── IfcExternalSpatialStructureElement ← NEW
    │   └── IfcExternalSpatialElement ← NEW
    └── IfcSpatialZone ← NEW
```

**IFC4.3 Spatial Hierarchy:**
```
IfcProduct
└── IfcSpatialElement (abstract)
    ├── IfcSpatialStructureElement (abstract)
    │   ├── IfcSite
    │   ├── IfcFacility (abstract) ← NEW supertype above IfcBuilding
    │   │   ├── IfcBuilding (moved under IfcFacility)
    │   │   ├── IfcBridge ← NEW
    │   │   ├── IfcRoad ← NEW
    │   │   ├── IfcRailway ← NEW
    │   │   └── IfcMarineFacility ← NEW
    │   ├── IfcFacilityPart (abstract) ← NEW
    │   │   ├── IfcBridgePart ← NEW
    │   │   ├── IfcFacilityPartCommon ← NEW
    │   │   ├── IfcRoadPart ← NEW
    │   │   ├── IfcRailwayPart ← NEW
    │   │   └── IfcMarinePart ← NEW
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

### IfcElement Direct Subtypes (IFC4.3)

| Subtype | Description |
|---------|-------------|
| IfcBuiltElement | Physical building/infrastructure components |
| IfcCivilElement | Generic civil engineering elements |
| IfcDistributionElement | MEP elements (HVAC, plumbing, electrical) |
| IfcElementAssembly | Composed element assemblies |
| IfcElementComponent | Small components (fasteners, reinforcement) |
| IfcFeatureElement | Openings, projections, voids |
| IfcFurnishingElement | Furniture and furnishings |
| IfcGeographicElement | Geographic features (terrain, vegetation) |
| IfcGeotechnicalElement | Geotechnical elements (IFC4.3) |
| IfcLinearElement | Linear infrastructure elements (IFC4.3) |
| IfcPositioningElement | Alignment, referent, linear positioning (IFC4.3) |
| IfcTransportationDevice | Vehicles, transport devices (IFC4.3) |
| IfcVirtualElement | Non-physical boundary elements |

---

## 3. IFC2x3 vs IFC4 vs IFC4.3 Comparison

### 3.1 Entity Renames

| IFC2x3 Name | IFC4 Name | IFC4.3 Name |
|-------------|-----------|-------------|
| IfcBuildingElement | IfcBuildingElement | **IfcBuiltElement** |
| IfcBuildingElementType | IfcBuildingElementType | **IfcBuiltElementType** |
| — | IfcBeamStandardCase | **Removed** (merged into IfcBeam) |
| — | IfcColumnStandardCase | **Removed** (merged into IfcColumn) |
| — | IfcDoorStandardCase | **Removed** (merged into IfcDoor) |
| — | IfcMemberStandardCase | **Removed** (merged into IfcMember) |
| — | IfcOpeningStandardCase | **Removed** (merged into IfcOpeningElement) |
| — | IfcPlateStandardCase | **Removed** (merged into IfcPlate) |
| — | IfcSlabStandardCase | **Removed** (merged into IfcSlab) |
| — | IfcSlabElementedCase | **Removed** (merged into IfcSlab) |
| — | IfcWallElementedCase | **Removed** (merged into IfcWall) |
| — | IfcWindowStandardCase | **Removed** (merged into IfcWindow) |

### 3.2 Attribute Changes

| Entity | IFC2x3 | IFC4 / IFC4.3 | Change |
|--------|--------|---------------|--------|
| IfcMaterial | Name only | Name, Description, Category | New optional attributes |
| IfcStairFlight | NumberOfRiser | NumberOfRisers | Typo fix (plural) |
| IfcBuildingElementProxy | CompositionType | PredefinedType | Attribute replaced |
| IfcCostSchedule | Actor-based attributes | DateTime/Label fields | Simplified |
| IfcConstructionResource | ResourceIdentifier, etc. | Usage, BaseCosts | Restructured |
| IfcRelSequence | LagTime (float) | TimeLag (IfcLagTime) | Changed to object reference |
| Many elements | No PredefinedType | PredefinedType added | IfcBeam, IfcColumn, IfcDoor, IfcWall, IfcWindow, etc. |
| Date/Time entities | IfcDateAndTime objects | IfcDateTime (string) | Simplified in IFC4 |
| IfcObjectPlacement | (in IfcLocalPlacement) | PlacementRelTo at supertype | Attribute moved up (IFC4.3) |

### 3.3 New Entity Types per Version

#### New in IFC4 (not in IFC2x3)
- **Spatial**: IfcSpatialElement, IfcSpatialZone, IfcExternalSpatialElement
- **Elements**: IfcChimney, IfcShadingDevice, IfcGeographicElement
- **StandardCase variants**: IfcBeamStandardCase, IfcColumnStandardCase, IfcDoorStandardCase, etc.
- **Geometry**: IfcTriangulatedFaceSet, IfcAdvancedBrep, IfcBSplineSurfaceWithKnots
- **Properties**: Enhanced property set structure, IfcPropertyTemplateDefinition
- **Context**: IfcProjectLibrary, IfcContext (abstract supertype of IfcProject)

#### New in IFC4.3 (not in IFC4)
- **Facilities**: IfcFacility, IfcBridge, IfcRoad, IfcRailway, IfcMarineFacility
- **Facility Parts**: IfcFacilityPart, IfcBridgePart, IfcRoadPart, IfcRailwayPart, IfcMarinePart, IfcFacilityPartCommon
- **Infrastructure elements**: IfcCourse, IfcEarthworksElement, IfcEarthworksCut, IfcEarthworksFill, IfcKerb, IfcPavement, IfcRail, IfcTrackElement, IfcMooringDevice, IfcNavigationElement, IfcBearing
- **Geotechnical**: IfcGeotechnicalElement, IfcBorehole, IfcGeomodel, IfcGeoSlice
- **Linear/Positioning**: IfcLinearElement, IfcPositioningElement, IfcAlignment, IfcAlignmentCant, IfcAlignmentHorizontal, IfcAlignmentVertical, IfcAlignmentSegment, IfcReferent
- **Transportation**: IfcTransportationDevice, IfcVehicle
- **Distribution**: IfcLiquidTerminal, IfcSignal, IfcSign, IfcConveyorSegment
- **Deep foundations**: IfcDeepFoundation, IfcPile (moved under IfcDeepFoundation), IfcCaissonFoundation
- **Other**: IfcVibrationDamper, IfcTendonConduit, IfcImpactProtectionDevice

### 3.4 Deprecated/Removed Entities

#### Removed in IFC4 (from IFC2x3)
- IfcBezierCurve → use IfcBSplineCurve
- IfcRationalBezierCurve → use IfcBSplineCurve
- Ifc2DCompositeCurve → use IfcCompositeCurve
- IfcCalendarDate, IfcLocalTime, IfcDateAndTime → use IfcDateTime (string)
- IfcDimensionCurve, IfcDimensionCurveTerminator
- IfcAnnotationSurface
- IfcMaterialDefinitionRepresentation
- IfcTimeSeriesSchedule
- IfcMove → use IfcTask
- IfcOrderRequest → use IfcTask
- IfcRelAssignsTasks → removed
- IfcElectricDistributionPoint → retained read-only for legacy
- 100+ other entities related to annotation, dimension callouts, material property types, constraint relationships

#### Removed in IFC4.3 (from IFC4)
- All StandardCase subtypes: IfcBeamStandardCase, IfcColumnStandardCase, IfcDoorStandardCase, IfcMemberStandardCase, IfcOpeningStandardCase, IfcPlateStandardCase, IfcSlabStandardCase, IfcSlabElementedCase, IfcWallElementedCase, IfcWindowStandardCase
- IfcPresentationStyleAssignment → IfcStyledItem.Styles simplified

---

## 4. Relationship Types

### 4.1 IfcRelationship Hierarchy (IFC4.3)

```
IfcRelationship (abstract)
├── IfcRelAssigns (abstract) — assignment relationships
│   ├── IfcRelAssignsToActor
│   ├── IfcRelAssignsToControl
│   ├── IfcRelAssignsToGroup
│   │   └── IfcRelAssignsToGroupByFactor
│   ├── IfcRelAssignsToProcess
│   ├── IfcRelAssignsToProduct
│   └── IfcRelAssignsToResource
├── IfcRelAssociates (abstract) — external reference associations
│   ├── IfcRelAssociatesApproval
│   ├── IfcRelAssociatesClassification
│   ├── IfcRelAssociatesConstraint
│   ├── IfcRelAssociatesDocument
│   ├── IfcRelAssociatesLibrary
│   ├── IfcRelAssociatesMaterial
│   └── IfcRelAssociatesProfileDef
├── IfcRelConnects (abstract) — connectivity relationships
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
├── IfcRelDeclares — project/library declarations
├── IfcRelDecomposes (abstract) — decomposition relationships
│   ├── IfcRelAggregates
│   ├── IfcRelNests
│   ├── IfcRelProjectsElement
│   └── IfcRelVoidsElement
└── IfcRelDefines (abstract) — definition relationships
    ├── IfcRelDefinesByObject
    ├── IfcRelDefinesByProperties
    ├── IfcRelDefinesByTemplate
    └── IfcRelDefinesByType
```

### 4.2 Key Relationship Details

#### IfcRelAggregates (Spatial Decomposition)
- **Purpose**: Whole/part composition — relates a parent to its child parts
- **Inheritance**: IfcRelationship → IfcRelDecomposes → IfcRelAggregates
- **Key attributes**: `RelatingObject` (the whole), `RelatedObjects` (set of parts)
- **Use**: Spatial hierarchy (Project → Site → Building → Storey), element decomposition (Roof → Slabs + Rafters)
- **Rule**: RelatingObject cannot be in RelatedObjects (no self-reference)

#### IfcRelContainedInSpatialStructure (Element Containment)
- **Purpose**: Assigns elements to a spatial structure element
- **Inheritance**: IfcRelationship → IfcRelConnects → IfcRelContainedInSpatialStructure
- **Key attributes**: `RelatingStructure` (spatial container), `RelatedElements` (set of elements)
- **Use**: Wall contained in Storey, Equipment contained in Space

#### IfcRelAssociatesMaterial (Material Association)
- **Purpose**: Links material definitions to elements or types
- **Inheritance**: IfcRelationship → IfcRelAssociates → IfcRelAssociatesMaterial
- **Key attributes**: `RelatedObjects` (elements/types), `RelatingMaterial` (IfcMaterialSelect)
- **Material types**: IfcMaterial, IfcMaterialList, IfcMaterialLayerSet, IfcMaterialLayerSetUsage, IfcMaterialProfileSet, IfcMaterialConstituentSet (IFC4+)
- **Rule**: Maximum one material association per building element

#### IfcRelDefinesByProperties (Property Assignment)
- **Purpose**: Assigns property sets to object instances
- **Inheritance**: IfcRelationship → IfcRelDefines → IfcRelDefinesByProperties
- **Key attributes**: `RelatedObjects` (object instances), `RelatingPropertyDefinition` (property set)
- **Use**: Attaching Pset_WallCommon to an IfcWall instance

#### IfcRelDefinesByType (Type Assignment)
- **Purpose**: Assigns an object type to object occurrences
- **Inheritance**: IfcRelationship → IfcRelDefines → IfcRelDefinesByType
- **Key attributes**: `RelatedObjects` (occurrences), `RelatingType` (IfcTypeObject)
- **Use**: Multiple IfcWindow instances sharing one IfcWindowType

#### IfcRelVoidsElement (Opening Elements)
- **Purpose**: Creates a void (opening) in an element
- **Inheritance**: IfcRelationship → IfcRelDecomposes → IfcRelVoidsElement
- **Key attributes**: `RelatingBuildingElement` (host element), `RelatedOpeningElement` (IfcOpeningElement)
- **Use**: Door/window opening in a wall

#### IfcRelFillsElement (Filling Openings)
- **Purpose**: Places an element into an opening
- **Inheritance**: IfcRelationship → IfcRelConnects → IfcRelFillsElement
- **Key attributes**: `RelatingOpeningElement` (the opening), `RelatedBuildingElement` (the fill, e.g. door/window)
- **Use**: IfcDoor filling an IfcOpeningElement in an IfcWall

### 4.3 IfcRelAssigns Variants

| Variant | Purpose | RelatingObject |
|---------|---------|---------------|
| IfcRelAssignsToActor | Assign responsibility to a person/organization | IfcActor |
| IfcRelAssignsToControl | Assign to a control (budget, schedule) | IfcControl |
| IfcRelAssignsToGroup | Logical grouping of objects | IfcGroup |
| IfcRelAssignsToProcess | Assign to a process/task | IfcProcess |
| IfcRelAssignsToProduct | Assign to a product | IfcProduct |
| IfcRelAssignsToResource | Assign to a resource | IfcResource |

### 4.4 IfcRelConnects Variants

| Variant | Purpose |
|---------|---------|
| IfcRelConnectsElements | Physical connection between elements |
| IfcRelConnectsPathElements | Wall-to-wall connections with path info |
| IfcRelConnectsWithRealizingElements | Connection realized by physical elements (fasteners) |
| IfcRelConnectsPortToElement | Port-to-element connection (MEP) |
| IfcRelConnectsPorts | Port-to-port connection (flow networks) |
| IfcRelConnectsStructuralActivity | Structural load/action to structural item |
| IfcRelConnectsStructuralMember | Structural member connection (joints) |
| IfcRelContainedInSpatialStructure | Element containment in spatial structure |
| IfcRelCoversBldgElements | Covering applied to building element |
| IfcRelCoversSpaces | Covering applied to space |
| IfcRelFillsElement | Element filling an opening |
| IfcRelFlowControlElements | Flow control element assignment |
| IfcRelInterferesElements | Clash/interference between elements |
| IfcRelPositions | Linear positioning relationship (IFC4.3) |
| IfcRelReferencedInSpatialStructure | Referencing (not containment) in spatial structure |
| IfcRelSequence | Process sequencing with time lag |
| IfcRelServicesBuildings | System servicing a facility |
| IfcRelSpaceBoundary | Space boundary definition |

---

## 5. Migration Patterns

### 5.1 IFC2x3 → IFC4 Migration

#### Entity Mapping
```
IFC2x3                          IFC4
─────────────────────────────────────────────────
IfcBuildingElement            → IfcBuildingElement (unchanged)
IfcBezierCurve                → IfcBSplineCurve
IfcRationalBezierCurve        → IfcBSplineCurve
Ifc2DCompositeCurve           → IfcCompositeCurve
IfcCalendarDate               → IfcDate (string)
IfcLocalTime                  → IfcTime (string)
IfcDateAndTime                → IfcDateTime (string)
IfcMove                       → IfcTask
IfcOrderRequest               → IfcTask
IfcRelAssignsTasks            → (removed, use IfcRelAssignsToProcess)
IfcElectricDistributionPoint  → IfcElectricDistributionBoard
IfcMaterial.Name              → IfcMaterial.Name + .Description + .Category
IfcStairFlight.NumberOfRiser  → IfcStairFlight.NumberOfRisers
```

#### Key Migration Considerations
1. **Date/time conversion**: All date/time entity references must be converted to ISO 8601 strings
2. **PredefinedType**: Many elements gain optional PredefinedType — existing data gets NOTDEFINED
3. **Geometry**: BSpline curves/surfaces get richer parameterization
4. **Property sets**: Enhanced with new standard properties; custom property sets remain compatible
5. **Upward compatibility**: Most widely-used IFC2x3 data can be read as IFC4 with minor adjustments

### 5.2 IFC4 → IFC4.3 Migration

#### Entity Mapping
```
IFC4                            IFC4.3
─────────────────────────────────────────────────
IfcBuildingElement            → IfcBuiltElement (RENAMED)
IfcBuildingElementType        → IfcBuiltElementType (RENAMED)
IfcBeamStandardCase           → IfcBeam (MERGED)
IfcColumnStandardCase         → IfcColumn (MERGED)
IfcDoorStandardCase           → IfcDoor (MERGED)
IfcMemberStandardCase         → IfcMember (MERGED)
IfcOpeningStandardCase        → IfcOpeningElement (MERGED)
IfcPlateStandardCase          → IfcPlate (MERGED)
IfcSlabStandardCase           → IfcSlab (MERGED)
IfcSlabElementedCase          → IfcSlab (MERGED)
IfcWallElementedCase          → IfcWall (MERGED)
IfcWindowStandardCase         → IfcWindow (MERGED)
IfcBuilding                   → IfcBuilding (now subtype of IfcFacility)
IfcPile                       → IfcPile (now subtype of IfcDeepFoundation)
IfcPresentationStyleAssignment → (removed, use IfcStyledItem directly)
```

#### Key Migration Considerations
1. **StandardCase removal**: All StandardCase/ElementedCase entities merged into parent — behavior determined by property sets
2. **Entity rename**: IfcBuildingElement → IfcBuiltElement is the most impactful rename
3. **Hierarchy insertion**: IfcFacility inserted between IfcBuilding and IfcSpatialStructureElement
4. **Geometry changes**: IfcCompositeCurve.Segments type changed from ListOfIfcCompositeCurveSegment to ListOfIfcSegment
5. **Axis placement**: Location changed from IfcCartesianPoint to IfcPoint (supertype)

### 5.3 IfcOpenShell Migration Tool

```python
import ifcopenshell
import ifcpatch

# Upgrade IFC2x3 to IFC4
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

> **Note**: Upgrading is more stable than downgrading. The Migrate recipe is marked as experimental.

---

## 6. Version Comparison Tables

### 6.1 Schema Overview

| Feature | IFC2x3 | IFC4 | IFC4.3 |
|---------|--------|------|--------|
| ISO Standard | ISO/PAS 16739:2005 | ISO 16739-1:2018 | ISO 16739-1:2024 |
| Entity Count | ~600 | ~750 | ~900+ |
| Infrastructure | No | Limited | Full |
| Spatial Hierarchy | 4 levels | 6 levels | 12+ levels |
| Geometry | Basic B-rep, CSG | +Tessellation, AdvancedBrep | +Alignment sweeps, cant |
| PredefinedType | Limited | On most elements | On all elements |
| StandardCase entities | No | Yes (11 types) | Removed (merged back) |
| Alignment support | No | No | Full (horizontal, vertical, cant) |
| Linear referencing | No | No | Yes (IfcLinearPlacement) |

### 6.2 Spatial Structure Entity Availability

| Entity | IFC2x3 | IFC4 | IFC4.3 |
|--------|--------|------|--------|
| IfcProject | ✅ | ✅ | ✅ |
| IfcSite | ✅ | ✅ | ✅ |
| IfcBuilding | ✅ | ✅ | ✅ |
| IfcBuildingStorey | ✅ | ✅ | ✅ |
| IfcSpace | ✅ | ✅ | ✅ |
| IfcSpatialElement | ❌ | ✅ | ✅ |
| IfcSpatialZone | ❌ | ✅ | ✅ |
| IfcExternalSpatialElement | ❌ | ✅ | ✅ |
| IfcFacility | ❌ | ❌ | ✅ |
| IfcFacilityPart | ❌ | ❌ | ✅ |
| IfcBridge | ❌ | ❌ | ✅ |
| IfcRoad | ❌ | ❌ | ✅ |
| IfcRailway | ❌ | ❌ | ✅ |
| IfcMarineFacility | ❌ | ❌ | ✅ |
| IfcBridgePart | ❌ | ❌ | ✅ |
| IfcRoadPart | ❌ | ❌ | ✅ |
| IfcRailwayPart | ❌ | ❌ | ✅ |
| IfcMarinePart | ❌ | ❌ | ✅ |
| IfcFacilityPartCommon | ❌ | ❌ | ✅ |

### 6.3 Key Element Entity Availability

| Entity | IFC2x3 | IFC4 | IFC4.3 |
|--------|--------|------|--------|
| IfcWall | ✅ | ✅ | ✅ |
| IfcBeam | ✅ | ✅ | ✅ |
| IfcColumn | ✅ | ✅ | ✅ |
| IfcSlab | ✅ | ✅ | ✅ |
| IfcDoor | ✅ | ✅ | ✅ |
| IfcWindow | ✅ | ✅ | ✅ |
| IfcRoof | ✅ | ✅ | ✅ |
| IfcStair | ✅ | ✅ | ✅ |
| IfcBuildingElement | ✅ | ✅ | ❌ (→ IfcBuiltElement) |
| IfcBuiltElement | ❌ | ❌ | ✅ |
| IfcChimney | ❌ | ✅ | ✅ |
| IfcShadingDevice | ❌ | ✅ | ✅ |
| IfcGeographicElement | ❌ | ✅ | ✅ |
| IfcBeamStandardCase | ❌ | ✅ | ❌ (merged) |
| IfcColumnStandardCase | ❌ | ✅ | ❌ (merged) |
| IfcWallStandardCase | ❌ | ✅ | ✅ (retained) |
| IfcBearing | ❌ | ❌ | ✅ |
| IfcCourse | ❌ | ❌ | ✅ |
| IfcEarthworksElement | ❌ | ❌ | ✅ |
| IfcKerb | ❌ | ❌ | ✅ |
| IfcPavement | ❌ | ❌ | ✅ |
| IfcRail | ❌ | ❌ | ✅ |
| IfcTrackElement | ❌ | ❌ | ✅ |
| IfcDeepFoundation | ❌ | ❌ | ✅ |
| IfcGeotechnicalElement | ❌ | ❌ | ✅ |
| IfcLinearElement | ❌ | ❌ | ✅ |
| IfcAlignment | ❌ | ❌ | ✅ |
| IfcTransportationDevice | ❌ | ❌ | ✅ |

### 6.4 Relationship Entity Availability

| Relationship | IFC2x3 | IFC4 | IFC4.3 |
|-------------|--------|------|--------|
| IfcRelAggregates | ✅ | ✅ | ✅ |
| IfcRelContainedInSpatialStructure | ✅ | ✅ | ✅ |
| IfcRelAssociatesMaterial | ✅ | ✅ | ✅ |
| IfcRelDefinesByProperties | ✅ | ✅ | ✅ |
| IfcRelDefinesByType | ✅ | ✅ | ✅ |
| IfcRelVoidsElement | ✅ | ✅ | ✅ |
| IfcRelFillsElement | ✅ | ✅ | ✅ |
| IfcRelAssignsToActor | ✅ | ✅ | ✅ |
| IfcRelAssignsToGroup | ✅ | ✅ | ✅ |
| IfcRelAssignsToProcess | ✅ | ✅ | ✅ |
| IfcRelAssignsToProduct | ✅ | ✅ | ✅ |
| IfcRelAssignsToResource | ✅ | ✅ | ✅ |
| IfcRelAssignsTasks | ✅ | ❌ | ❌ |
| IfcRelConnectsElements | ✅ | ✅ | ✅ |
| IfcRelConnectsPorts | ✅ | ✅ | ✅ |
| IfcRelSequence | ✅ | ✅ | ✅ |
| IfcRelSpaceBoundary | ✅ | ✅ | ✅ |
| IfcRelDeclares | ❌ | ✅ | ✅ |
| IfcRelDefinesByTemplate | ❌ | ✅ | ✅ |
| IfcRelDefinesByObject | ❌ | ✅ | ✅ |
| IfcRelInterferesElements | ❌ | ✅ | ✅ |
| IfcRelAssociatesProfileDef | ❌ | ❌ | ✅ |
| IfcRelPositions | ❌ | ❌ | ✅ |

---

## 7. IfcOpenShell Code Examples

### 7.1 Schema Version Detection and File Creation

```python
import ifcopenshell

# Open a file and check its schema
model = ifcopenshell.open("example.ifc")
print(f"Schema: {model.schema}")  # 'IFC2X3', 'IFC4', or 'IFC4X3'

# Create new file with specific schema
model_2x3 = ifcopenshell.file(schema="IFC2X3")
model_4 = ifcopenshell.file(schema="IFC4")
model_4x3 = ifcopenshell.file(schema="IFC4X3")

# Using the API (recommended)
import ifcopenshell.api
model = ifcopenshell.api.run("project.create_file", schema="IFC4")
```

### 7.2 Spatial Hierarchy with IfcRelAggregates

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", schema="IFC4")

# Create spatial hierarchy
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="My Project")
site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="My Building")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="Ground Floor")

# Aggregate: Project → Site → Building → Storey
# Each call creates an IfcRelAggregates instance
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model, relating_object=building, products=[storey])

# IFC4.3 infrastructure example
model43 = ifcopenshell.api.run("project.create_file", schema="IFC4X3")
project = ifcopenshell.api.run("root.create_entity", model43, ifc_class="IfcProject", name="Bridge Project")
site = ifcopenshell.api.run("root.create_entity", model43, ifc_class="IfcSite", name="Site")
bridge = ifcopenshell.api.run("root.create_entity", model43, ifc_class="IfcBridge", name="Main Bridge")

ifcopenshell.api.run("aggregate.assign_object", model43, relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model43, relating_object=site, products=[bridge])
```

### 7.3 IfcRelContainedInSpatialStructure (Element Containment)

```python
# Create a wall and contain it in a storey
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 001")

# Assign the wall to the storey — creates IfcRelContainedInSpatialStructure
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# Query: get all elements in a storey
import ifcopenshell.util.element
elements = ifcopenshell.util.element.get_decomposition(storey)

# Direct access to containment relationship
for rel in storey.ContainsElements:  # inverse attribute
    for element in rel.RelatedElements:
        print(f"  {element.is_a()}: {element.Name}")
```

### 7.4 IfcRelAssociatesMaterial (Material Association)

```python
# Create material and assign to wall
material = ifcopenshell.api.run("material.add_material", model, name="Concrete")

# Single material assignment — creates IfcRelAssociatesMaterial
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=material)

# Layer set for walls
material_set = ifcopenshell.api.run("material.add_material_set", model,
    name="Wall Layers", set_type="IfcMaterialLayerSet")
layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=material_set, material=material)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer, attributes={"LayerThickness": 200.0})

# Query material of an element
import ifcopenshell.util.element
mat = ifcopenshell.util.element.get_material(wall)
print(f"Material: {mat.Name if mat else 'None'}")
```

### 7.5 IfcRelDefinesByProperties (Property Assignment)

```python
# Create and assign property set
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={
        "IsExternal": True,
        "FireRating": "REI120",
        "ThermalTransmittance": 0.24
    })

# Query properties
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {'Pset_WallCommon': {'IsExternal': True, 'FireRating': 'REI120', ...}}
```

### 7.6 IfcRelDefinesByType (Type Assignment)

```python
# Create a wall type
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")

# Assign type to wall occurrence — creates IfcRelDefinesByType
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# Query type of an element
element_type = ifcopenshell.util.element.get_type(wall)
print(f"Type: {element_type.Name if element_type else 'None'}")

# Get all occurrences of a type
occurrences = ifcopenshell.util.element.get_types(wall_type)
```

### 7.7 IfcRelVoidsElement and IfcRelFillsElement (Openings)

```python
# Create an opening in a wall — creates IfcRelVoidsElement
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# The void relationship
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)

# Create a door and fill the opening — creates IfcRelFillsElement
door = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoor", name="Door 001")
ifcopenshell.api.run("void.add_filling", model,
    opening=opening, element=door)

# Query openings of an element
for rel in wall.HasOpenings:  # inverse: IfcRelVoidsElement
    opening = rel.RelatedOpeningElement
    # Check if opening is filled
    for fill_rel in opening.HasFillings:  # inverse: IfcRelFillsElement
        filling = fill_rel.RelatedBuildingElement
        print(f"Opening filled by: {filling.is_a()} - {filling.Name}")
```

### 7.8 Version-Aware Code Pattern

```python
import ifcopenshell

model = ifcopenshell.open("some_file.ifc")
schema = model.schema  # 'IFC2X3', 'IFC4', or 'IFC4X3'

# Version-aware element class name
if schema == "IFC4X3":
    building_element_class = "IfcBuiltElement"
else:
    building_element_class = "IfcBuildingElement"

# Get all building elements regardless of version
elements = model.by_type(building_element_class)

# Version-aware spatial hierarchy query
if schema == "IFC4X3":
    facilities = model.by_type("IfcFacility")  # includes IfcBuilding, IfcBridge, etc.
else:
    facilities = model.by_type("IfcBuilding")

# Schema-agnostic approach using ifcopenshell.util
import ifcopenshell.util.element
for element in model.by_type("IfcElement"):
    container = ifcopenshell.util.element.get_container(element)
    psets = ifcopenshell.util.element.get_psets(element)
    material = ifcopenshell.util.element.get_material(element)
    element_type = ifcopenshell.util.element.get_type(element)
```

---

## References

- [IFC 4.3.2 Documentation](https://ifc43-docs.standards.buildingsmart.org/)
- [buildingSMART IFC Schema Specifications](https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/)
- [IFC Release Notes](https://technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/ifc-release-notes/)
- [IfcOpenShell Documentation](https://docs.ifcopenshell.org/)
- [IfcOpenShell GitHub](https://github.com/IfcOpenShell/IfcOpenShell)
- [IFC Schema Version Notes (STEP Tools)](https://steptools.com/docs/ifcbim/notes_schema.html)
- [buildingSMART Forums: IFC version changes](https://forums.buildingsmart.org/t/list-of-changes-between-ifc-versions/4604)
- [buildingSMART Forums: IFC2x3 to IFC4 mappings](https://forums.buildingsmart.org/t/ifc2x3-to-ifc4-mappings/3093)
