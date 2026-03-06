# IFC Core Concepts — Entity Classes and Attributes Reference

## IfcRoot (Abstract Base)

All named IFC entities inherit from IfcRoot. ALWAYS expect these four attributes on any IfcRoot-derived entity.

| Attribute | Type | IFC2X3 | IFC4 | IFC4X3 |
|-----------|------|--------|------|--------|
| GlobalId | IfcGloballyUniqueId | REQUIRED | REQUIRED | REQUIRED |
| OwnerHistory | IfcOwnerHistory | **REQUIRED** | **OPTIONAL** | **OPTIONAL** |
| Name | IfcLabel | OPTIONAL | OPTIONAL | OPTIONAL |
| Description | IfcText | OPTIONAL | OPTIONAL | OPTIONAL |

---

## IfcObjectDefinition Branch

### IfcObject (Abstract)

Adds `ObjectType` (IfcLabel, OPTIONAL) — a free-form text classification.

### IfcProduct (Abstract)

Adds placement and representation to IfcObject:

| Attribute | Type | Notes |
|-----------|------|-------|
| ObjectPlacement | IfcObjectPlacement | OPTIONAL — position in 3D space |
| Representation | IfcProductRepresentation | OPTIONAL — geometric shape |

---

## Spatial Structure Entities

### IfcSpatialStructureElement (Abstract)

| Attribute | Type | Notes |
|-----------|------|-------|
| LongName | IfcLabel | OPTIONAL — extended name |
| CompositionType | IfcElementCompositionEnum | COMPLEX, ELEMENT, or PARTIAL |

**Inverse attributes:**
- `ContainsElements` → IfcRelContainedInSpatialStructure (elements in this space)
- `IsDecomposedBy` → IfcRelAggregates (child spatial elements)

### IfcSite

| Attribute | Type | Notes |
|-----------|------|-------|
| RefLatitude | IfcCompoundPlaneAngleMeasure | OPTIONAL — WGS84 latitude |
| RefLongitude | IfcCompoundPlaneAngleMeasure | OPTIONAL — WGS84 longitude |
| RefElevation | IfcLengthMeasure | OPTIONAL — elevation above sea level |
| LandTitleNumber | IfcLabel | OPTIONAL |
| SiteAddress | IfcPostalAddress | OPTIONAL |

**Available in:** IFC2X3, IFC4, IFC4X3

### IfcBuilding

| Attribute | Type | Notes |
|-----------|------|-------|
| ElevationOfRefHeight | IfcLengthMeasure | OPTIONAL |
| ElevationOfTerrain | IfcLengthMeasure | OPTIONAL |
| BuildingAddress | IfcPostalAddress | OPTIONAL |

**Available in:** IFC2X3, IFC4, IFC4X3
**IFC4X3 change:** Now inherits from IfcFacility (abstract) instead of directly from IfcSpatialStructureElement.

### IfcBuildingStorey

| Attribute | Type | Notes |
|-----------|------|-------|
| Elevation | IfcLengthMeasure | OPTIONAL — storey elevation |

**Available in:** IFC2X3, IFC4, IFC4X3

### IfcSpace

| Attribute | Type | Notes |
|-----------|------|-------|
| PredefinedType | IfcSpaceTypeEnum | OPTIONAL [IFC4+] |
| ElevationWithFlooring | IfcLengthMeasure | OPTIONAL |

**Available in:** IFC2X3, IFC4, IFC4X3

### IfcFacility (Abstract) — IFC4X3 ONLY

Abstract supertype for facility types. IfcBuilding moved under this entity in IFC4X3.

**Subtypes:**
| Entity | Domain | Available |
|--------|--------|-----------|
| IfcBuilding | Buildings | IFC4X3 (moved here) |
| IfcBridge | Bridge infrastructure | IFC4X3 |
| IfcRoad | Road infrastructure | IFC4X3 |
| IfcRailway | Railway infrastructure | IFC4X3 |
| IfcMarineFacility | Marine infrastructure | IFC4X3 |

### IfcFacilityPart (Abstract) — IFC4X3 ONLY

Part/segment of a facility.

**Subtypes:**
| Entity | Domain | Available |
|--------|--------|-----------|
| IfcBridgePart | Bridge segment | IFC4X3 |
| IfcRoadPart | Road segment | IFC4X3 |
| IfcRailwayPart | Railway segment | IFC4X3 |
| IfcMarinePart | Marine segment | IFC4X3 |
| IfcFacilityPartCommon | Generic part | IFC4X3 |

### IfcSpatialZone — IFC4+ ONLY

Non-hierarchical spatial zone that overlaps with the main spatial structure (fire zones, HVAC zones, lighting zones).

### IfcExternalSpatialElement — IFC4+ ONLY

Represents exterior/outdoor spatial elements.

---

## IfcContext Branch

### IfcProject

| Attribute | Type | Notes |
|-----------|------|-------|
| UnitsInContext | IfcUnitAssignment | OPTIONAL (but ALWAYS set units) |
| RepresentationContexts | SET OF IfcRepresentationContext | OPTIONAL |
| Phase | IfcLabel | OPTIONAL |

**IFC2X3:** IfcProject inherits from IfcObject.
**IFC4/IFC4X3:** IfcProject inherits from IfcContext (new abstract class).

```python
# IFC2X3: IfcProject is under IfcObject
project = model.by_type("IfcProject")[0]
project.is_a("IfcObject")   # True in IFC2X3
# project.is_a("IfcContext") -> ERROR — IfcContext does not exist in IFC2X3

# IFC4/IFC4X3: IfcProject is under IfcContext
project.is_a("IfcContext")   # True in IFC4+
project.is_a("IfcObject")    # False in IFC4+ (different branch)
```

### IfcProjectLibrary — IFC4+ ONLY

Library of reusable definitions. Does NOT exist in IFC2X3.

---

## IfcElement Branch

### IfcElement (Abstract)

| Attribute | Type | Notes |
|-----------|------|-------|
| Tag | IfcIdentifier | OPTIONAL — element identifier/tag |

**Key inverse attributes:**
- `ContainedInStructure` → IfcRelContainedInSpatialStructure
- `HasOpenings` → IfcRelVoidsElement
- `IsTypedBy` → IfcRelDefinesByType [IFC4+]
- `IsDefinedBy` → IfcRelDefinesByProperties

### IfcBuiltElement / IfcBuildingElement

**CRITICAL RENAME:** `IfcBuildingElement` (IFC2X3/IFC4) was renamed to `IfcBuiltElement` in IFC4X3.

**Common subtypes (all versions unless noted):**

| Entity | IFC2X3 | IFC4 | IFC4X3 | Notes |
|--------|--------|------|--------|-------|
| IfcWall | YES | YES | YES | |
| IfcWallStandardCase | YES | YES | **REMOVED** | Merged into IfcWall in IFC4X3 |
| IfcColumn | YES | YES | YES | |
| IfcColumnStandardCase | — | YES | **REMOVED** | |
| IfcBeam | YES | YES | YES | |
| IfcBeamStandardCase | — | YES | **REMOVED** | |
| IfcSlab | YES | YES | YES | |
| IfcSlabStandardCase | — | YES | **REMOVED** | |
| IfcDoor | YES | YES | YES | |
| IfcDoorStandardCase | — | YES | **REMOVED** | |
| IfcWindow | YES | YES | YES | |
| IfcWindowStandardCase | — | YES | **REMOVED** | |
| IfcPlate | YES | YES | YES | |
| IfcMember | YES | YES | YES | |
| IfcRoof | YES | YES | YES | |
| IfcStair | YES | YES | YES | |
| IfcStairFlight | YES | YES | YES | |
| IfcRamp | YES | YES | YES | |
| IfcRampFlight | YES | YES | YES | |
| IfcFooting | YES | YES | YES | |
| IfcCovering | YES | YES | YES | |
| IfcCurtainWall | YES | YES | YES | |
| IfcRailing | YES | YES | YES | |
| IfcBuildingElementProxy | YES | YES | YES | |
| IfcChimney | — | YES | YES | |
| IfcShadingDevice | — | YES | YES | |
| IfcCourse | — | — | YES | Infrastructure |
| IfcDeepFoundation | — | — | YES | Replaces IfcPile hierarchy |
| IfcEarthworksElement | — | — | YES | Geotechnics |
| IfcKerb | — | — | YES | Roads |
| IfcPavement | — | — | YES | Roads |
| IfcRail | — | — | YES | Railway |
| IfcTrackElement | — | — | YES | Railway |
| IfcBearing | — | — | YES | Bridges |
| IfcMooringDevice | — | — | YES | Marine |
| IfcNavigationElement | — | — | YES | Marine |

### IfcElement Direct Subtypes (IFC4X3)

| Entity | Description |
|--------|-------------|
| IfcBuiltElement | Physical building/infrastructure components |
| IfcCivilElement | Generic civil engineering elements |
| IfcDistributionElement | MEP elements (HVAC, plumbing, electrical) |
| IfcElementAssembly | Composed element assemblies |
| IfcElementComponent | Small components (fasteners, reinforcement) |
| IfcFeatureElement | Openings, projections, voids |
| IfcFurnishingElement | Furniture and furnishings |
| IfcGeographicElement | Geographic features (terrain, vegetation) [IFC4+] |
| IfcGeotechnicalElement | Geotechnical elements [IFC4X3] |
| IfcLinearElement | Linear infrastructure elements [IFC4X3] |
| IfcPositioningElement | Alignment, referent, linear positioning [IFC4X3] |
| IfcTransportationDevice | Vehicles, transport devices [IFC4X3] |
| IfcVirtualElement | Non-physical boundary elements |

---

## IfcTypeObject Branch

### IfcTypeObject

| Attribute | Type | Notes |
|-----------|------|-------|
| ApplicableOccurrence | IfcIdentifier | OPTIONAL |
| HasPropertySets | SET OF IfcPropertySetDefinition | OPTIONAL |

### IfcTypeProduct

Adds `RepresentationMaps` (SET OF IfcRepresentationMap) and `Tag` (IfcLabel).

### IfcElementType (Abstract)

| Attribute | Type | Notes |
|-----------|------|-------|
| ElementType | IfcLabel | OPTIONAL — free-form element type designation |

**Version-specific type names:**

| Type | IFC2X3 | IFC4/IFC4X3 |
|------|--------|-------------|
| Door type | IfcDoorStyle | IfcDoorType |
| Window type | IfcWindowStyle | IfcWindowType |
| Wall type | IfcWallType | IfcWallType |
| Slab type | IfcSlabType | IfcSlabType |

---

## IfcRelationship Branch

### Decomposition Relationships (IfcRelDecomposes)

| Entity | Purpose | Key Attributes |
|--------|---------|---------------|
| IfcRelAggregates | Whole/part (spatial hierarchy) | RelatingObject, RelatedObjects |
| IfcRelNests | Ordered nesting | RelatingObject, RelatedObjects |
| IfcRelVoidsElement | Opening in element | RelatingBuildingElement, RelatedOpeningElement |
| IfcRelProjectsElement | Projection on element | RelatingElement, RelatedFeatureElement |

### Connectivity Relationships (IfcRelConnects)

| Entity | Purpose | Key Attributes |
|--------|---------|---------------|
| IfcRelContainedInSpatialStructure | Element in space | RelatingStructure, RelatedElements |
| IfcRelFillsElement | Element fills opening | RelatingOpeningElement, RelatedBuildingElement |
| IfcRelConnectsElements | Element-to-element connection | RelatingElement, RelatedElement |
| IfcRelSpaceBoundary | Space boundary | RelatingSpace, RelatedBuildingElement |
| IfcRelReferencedInSpatialStructure | Multi-storey reference | RelatingStructure, RelatedElements |
| IfcRelPositions | Linear positioning [IFC4X3] | RelatingPositioningElement, RelatedProducts |

### Definition Relationships (IfcRelDefines)

| Entity | Purpose | Key Attributes |
|--------|---------|---------------|
| IfcRelDefinesByType | Type assignment | RelatingType, RelatedObjects |
| IfcRelDefinesByProperties | Property set assignment | RelatingPropertyDefinition, RelatedObjects |
| IfcRelDefinesByObject | Object-to-object definition | RelatingObject, RelatedObjects |

### Association Relationships (IfcRelAssociates)

| Entity | Purpose | Key Attributes |
|--------|---------|---------------|
| IfcRelAssociatesMaterial | Material assignment | RelatingMaterial, RelatedObjects |
| IfcRelAssociatesClassification | Classification reference | RelatingClassification, RelatedObjects |
| IfcRelAssociatesDocument | Document reference | RelatingDocument, RelatedObjects |
| IfcRelAssociatesConstraint | Constraint assignment | RelatingConstraint, RelatedObjects |
| IfcRelAssociatesLibrary | Library reference | RelatingLibrary, RelatedObjects |
| IfcRelAssociatesProfileDef | Profile definition [IFC4X3] | RelatingProfileDef, RelatedObjects |

### Assignment Relationships (IfcRelAssigns)

| Entity | Purpose | Key Attributes |
|--------|---------|---------------|
| IfcRelAssignsToGroup | Group membership | RelatingGroup, RelatedObjects |
| IfcRelAssignsToActor | Actor assignment | RelatingActor, RelatedObjects |
| IfcRelAssignsToControl | Control assignment | RelatingControl, RelatedObjects |
| IfcRelAssignsToProcess | Process assignment | RelatingProcess, RelatedObjects |
| IfcRelAssignsToProduct | Product assignment | RelatingProduct, RelatedObjects |
| IfcRelAssignsToResource | Resource assignment | RelatingResource, RelatedObjects |

### IfcRelDeclares — IFC4+ ONLY

Declares objects within a project or library context. Does NOT exist in IFC2X3.

---

## IfcPropertyDefinition Branch

### IfcPropertySet

| Attribute | Type | Notes |
|-----------|------|-------|
| HasProperties | SET OF IfcProperty | REQUIRED — the properties in this set |

### IfcElementQuantity

| Attribute | Type | Notes |
|-----------|------|-------|
| MethodOfMeasurement | IfcLabel | OPTIONAL |
| Quantities | SET OF IfcPhysicalQuantity | REQUIRED |

---

## Schema Introspection Methods

```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell.ifcopenshell_wrapper

# Get schema object
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")

# Get entity declaration
entity = schema.declaration_by_name("IfcWall")

# List all attributes
for attr in entity.all_attributes():
    print(f"{attr.name()}: {attr.type_of_attribute()}")

# Check if entity exists in schema
try:
    schema.declaration_by_name("IfcRoad")
except RuntimeError:
    print("Entity does not exist in this schema")

# Get supertype chain
parent = entity.supertype()
while parent:
    print(parent.name())
    parent = parent.supertype()
```
