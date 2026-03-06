# Element Traversal — Working Code Examples

## Example 1: Basic Element Queries

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")
print(f"Schema: {model.schema}")
print(f"Total entities: {len(model)}")

# Query by type
walls = model.by_type("IfcWall")
print(f"Found {len(walls)} walls")

# Iterate with attribute access
for wall in walls:
    print(f"  #{wall.id()} {wall.Name or 'Unnamed'} (GUID: {wall.GlobalId})")

# Query spatial structure
project = model.by_type("IfcProject")[0]  # ALWAYS exactly 1
sites = model.by_type("IfcSite")
buildings = model.by_type("IfcBuilding")
storeys = model.by_type("IfcBuildingStorey")
spaces = model.by_type("IfcSpace")

print(f"Project: {project.Name}")
print(f"Sites: {len(sites)}, Buildings: {len(buildings)}")
print(f"Storeys: {len(storeys)}, Spaces: {len(spaces)}")
```

---

## Example 2: Entity Information Extraction

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Get all attributes as dict
info = wall.get_info()
print(f"Type: {info['type']}")
print(f"STEP ID: {info['id']}")
print(f"Name: {info.get('Name')}")
print(f"GlobalId: {info.get('GlobalId')}")

# List all attribute names and values
for key, value in info.items():
    print(f"  {key}: {value}")

# Get only primitive values (skip entity references)
scalar_info = wall.get_info(scalar_only=True)
for key, value in scalar_info.items():
    if value is not None:
        print(f"  {key}: {value}")
```

---

## Example 3: Type Checking with is_a()

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")

for entity in model.by_type("IfcElement"):
    # Classify elements by type hierarchy
    if entity.is_a("IfcWall"):
        category = "Wall"
    elif entity.is_a("IfcSlab"):
        category = "Slab"
    elif entity.is_a("IfcColumn"):
        category = "Column"
    elif entity.is_a("IfcBeam"):
        category = "Beam"
    elif entity.is_a("IfcDoor"):
        category = "Door"
    elif entity.is_a("IfcWindow"):
        category = "Window"
    else:
        category = entity.is_a()  # Get exact type name

    print(f"  {category}: {entity.Name or 'Unnamed'}")
```

---

## Example 4: Universal Property Extraction Pattern

```python
# IfcOpenShell — all schema versions
# This is the FUNDAMENTAL pattern for reading IFC properties.
# Path: element.IsDefinedBy → IfcRelDefinesByProperties
#       → RelatingPropertyDefinition (IfcPropertySet)
#       → HasProperties → NominalValue.wrappedValue
import ifcopenshell

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Manual property extraction (for understanding the IFC structure)
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset_or_qset = rel.RelatingPropertyDefinition

        if pset_or_qset.is_a("IfcPropertySet"):
            print(f"\nPropertySet: {pset_or_qset.Name}")
            for prop in pset_or_qset.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    if prop.NominalValue:
                        print(f"  {prop.Name}: {prop.NominalValue.wrappedValue}")
                    else:
                        print(f"  {prop.Name}: (no value)")
                elif prop.is_a("IfcPropertyEnumeratedValue"):
                    values = [v.wrappedValue for v in prop.EnumerationValues]
                    print(f"  {prop.Name}: {values}")
                elif prop.is_a("IfcPropertyListValue"):
                    values = [v.wrappedValue for v in prop.ListValues]
                    print(f"  {prop.Name}: {values}")

        elif pset_or_qset.is_a("IfcElementQuantity"):
            print(f"\nQuantitySet: {pset_or_qset.Name}")
            for qty in pset_or_qset.Quantities:
                if qty.is_a("IfcQuantityLength"):
                    print(f"  {qty.Name}: {qty.LengthValue}")
                elif qty.is_a("IfcQuantityArea"):
                    print(f"  {qty.Name}: {qty.AreaValue}")
                elif qty.is_a("IfcQuantityVolume"):
                    print(f"  {qty.Name}: {qty.VolumeValue}")
                elif qty.is_a("IfcQuantityWeight"):
                    print(f"  {qty.Name}: {qty.WeightValue}")
                elif qty.is_a("IfcQuantityCount"):
                    print(f"  {qty.Name}: {qty.CountValue}")
```

---

## Example 5: Property Extraction Using util.element (Recommended)

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Get ALL property sets as a clean dict
psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, props in psets.items():
    print(f"\n{pset_name}:")
    for prop_name, prop_value in props.items():
        print(f"  {prop_name} = {prop_value}")

# Get quantity sets only
qsets = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
for qset_name, quantities in qsets.items():
    print(f"\n{qset_name}:")
    for qty_name, qty_value in quantities.items():
        print(f"  {qty_name} = {qty_value}")

# Check a specific property value
pset_common = psets.get("Pset_WallCommon", {})
is_external = pset_common.get("IsExternal", False)
is_loadbearing = pset_common.get("LoadBearing", False)
print(f"External: {is_external}, Load-bearing: {is_loadbearing}")

# Get type-level properties too
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    type_psets = ifcopenshell.util.element.get_psets(wall_type)
    print(f"\nType properties for {wall_type.Name}:")
    for pset_name, props in type_psets.items():
        for prop_name, prop_value in props.items():
            print(f"  {pset_name}/{prop_name} = {prop_value}")
```

---

## Example 6: Inverse Reference Traversal

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# List ALL inverse references
print(f"Entities referencing wall #{wall.id()}:")
for ref in model.get_inverse(wall):
    print(f"  {ref.is_a()} #{ref.id()}")

# Find spatial container via inverse
def get_container_manual(model, element):
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelContainedInSpatialStructure"):
            return rel.RelatingStructure
    return None

# Find element type via inverse
def get_type_manual(model, element):
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelDefinesByType"):
            return rel.RelatingType
    return None

# Find material via inverse
def get_material_manual(model, element):
    for rel in model.get_inverse(element):
        if rel.is_a("IfcRelAssociatesMaterial"):
            return rel.RelatingMaterial
    return None

# Find openings in a wall via inverse
def get_openings_manual(model, wall):
    openings = []
    for rel in model.get_inverse(wall):
        if rel.is_a("IfcRelVoidsElement"):
            openings.append(rel.RelatedOpeningElement)
    return openings

container = get_container_manual(model, wall)
wall_type = get_type_manual(model, wall)
material = get_material_manual(model, wall)
openings = get_openings_manual(model, wall)

print(f"Container: {container.Name if container else 'None'}")
print(f"Type: {wall_type.Name if wall_type else 'None'}")
print(f"Material: {material.is_a() if material else 'None'}")
print(f"Openings: {len(openings)}")
```

---

## Example 7: GUID Operations

```python
# IfcOpenShell — all schema versions (GUID module is schema-independent)
import ifcopenshell
import ifcopenshell.guid

# Generate a new IFC GlobalId
new_guid = ifcopenshell.guid.new()
print(f"New GUID: {new_guid}")       # 22-character string
print(f"Length: {len(new_guid)}")     # 22

# Convert IFC GUID → standard UUID
uuid_str = ifcopenshell.guid.expand(new_guid)
print(f"UUID: {uuid_str}")           # "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Convert standard UUID → IFC GUID
ifc_guid = ifcopenshell.guid.compress(uuid_str)
print(f"IFC GUID: {ifc_guid}")       # Same as new_guid
assert ifc_guid == new_guid

# Round-trip: look up entity, get GUID, look up again
model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]
guid = wall.GlobalId
same_wall = model.by_guid(guid)
assert wall == same_wall
```

---

## Example 8: Element Statistics Report

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element
from collections import Counter

model = ifcopenshell.open("building.ifc")

# Entity type distribution
type_counts = Counter(entity.is_a() for entity in model)
print("=== Entity Distribution (Top 20) ===")
for ifc_type, count in type_counts.most_common(20):
    print(f"  {ifc_type}: {count}")

# Building element summary
print("\n=== Building Elements ===")
for element_type in ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam",
                      "IfcDoor", "IfcWindow", "IfcRoof", "IfcStair"]:
    elements = model.by_type(element_type)
    if elements:
        print(f"  {element_type}: {len(elements)}")

# Spatial structure
print("\n=== Spatial Structure ===")
for storey in model.by_type("IfcBuildingStorey"):
    container = ifcopenshell.util.element.get_decomposition(storey)
    # Also get contained elements
    contained = []
    for rel in model.get_inverse(storey):
        if rel.is_a("IfcRelContainedInSpatialStructure"):
            contained.extend(rel.RelatedElements)
    print(f"  {storey.Name}: {len(contained)} elements")
```

---

## Example 9: CSS-like Selector Queries

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("building.ifc")

# Select all walls
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")
print(f"All walls: {len(walls)}")

# Select external walls by property
ext_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')
print(f"External walls: {len(ext_walls)}")

# Select walls containing "EXT" in name
named = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Name *= "EXT"')
print(f"Walls with 'EXT' in name: {len(named)}")

# Select elements on a specific storey
ground = ifcopenshell.util.selector.filter_elements(
    model, 'IfcBuildingElement, container="Ground Floor"')
print(f"Ground floor elements: {len(ground)}")

# Select by material
concrete = ifcopenshell.util.selector.filter_elements(
    model, 'IfcElement, material="Concrete"')
print(f"Concrete elements: {len(concrete)}")
```

---

## Example 10: Schema-Aware Element Queries

```python
# IfcOpenShell — demonstrates schema-specific queries
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema

# Schema-aware building element query
if schema == "IFC2X3":
    # IFC2X3 has IfcWallStandardCase as a separate type
    all_walls = model.by_type("IfcWall")  # includes IfcWallStandardCase
    standard_walls = model.by_type("IfcWallStandardCase")
    print(f"Total walls: {len(all_walls)}, Standard: {len(standard_walls)}")
elif schema in ("IFC4", "IFC4X3"):
    # IFC4+ removed IfcWallStandardCase — all walls are IfcWall
    all_walls = model.by_type("IfcWall")
    print(f"Total walls: {len(all_walls)}")

# Schema-aware spatial query
if schema == "IFC2X3":
    spatials = model.by_type("IfcSpatialStructureElement")
elif schema in ("IFC4", "IFC4X3"):
    spatials = model.by_type("IfcSpatialElement")
print(f"Spatial elements: {len(spatials)}")

# Schema-aware building elements parent
if schema in ("IFC2X3", "IFC4"):
    elements = model.by_type("IfcBuildingElement")
elif schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
print(f"Building elements: {len(elements)}")

# IFC4X3 infrastructure types
if schema == "IFC4X3":
    bridges = model.by_type("IfcBridge")
    roads = model.by_type("IfcRoad")
    railways = model.by_type("IfcRailway")
    print(f"Bridges: {len(bridges)}, Roads: {len(roads)}, Railways: {len(railways)}")
```
