# Utility Module Working Code Examples

All examples verified against the [IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html) and [IfcOpenShell Academy](https://academy.ifcopenshell.org/).

---

## 1. Property Set Extraction

### Get All Property Sets for an Element

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get all property sets (excludes quantity sets)
psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, props in psets.items():
    print(f"\n{pset_name}:")
    for prop_name, prop_value in props.items():
        if prop_name != "id":  # Skip internal STEP ID
            print(f"  {prop_name} = {prop_value}")
```

### Get Properties Including Inherited from Type

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# should_inherit=True (default) includes type-level psets
all_psets = ifcopenshell.util.element.get_psets(wall, should_inherit=True)

# Occurrence-only psets (no type inheritance)
own_psets = ifcopenshell.util.element.get_psets(wall, should_inherit=False)
```

### Get a Specific Property Value

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get single property value — returns None if pset or property missing
is_external = ifcopenshell.util.element.get_pset(
    wall, "Pset_WallCommon", "IsExternal")
if is_external is not None:
    print(f"IsExternal: {is_external}")

# Get entire property set as dict
wall_common = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon")
if wall_common:
    print(f"Fire rating: {wall_common.get('FireRating', 'N/A')}")
```

### Get Quantity Sets

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Only quantity sets
qsets = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
for qset_name, quantities in qsets.items():
    print(f"\n{qset_name}:")
    for q_name, q_value in quantities.items():
        if q_name != "id":
            print(f"  {q_name} = {q_value}")

# Both property sets and quantity sets
all_data = ifcopenshell.util.element.get_psets(wall, psets_only=False)
```

---

## 2. Element Relationship Navigation

### Get Type, Container, and Material

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Type
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    print(f"Type: {wall_type.Name} ({wall_type.is_a()})")
    # Types have their own property sets
    type_psets = ifcopenshell.util.element.get_psets(wall_type)

# Spatial container
container = ifcopenshell.util.element.get_container(wall)
if container:
    print(f"Container: {container.Name} ({container.is_a()})")

# Material
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material: {material.is_a()}")
    if material.is_a("IfcMaterialLayerSetUsage"):
        for layer in material.ForLayerSet.MaterialLayers:
            print(f"  Layer: {layer.Material.Name} - {layer.LayerThickness}")
    elif material.is_a("IfcMaterial"):
        print(f"  Name: {material.Name}")

# Flat list of individual materials
materials = ifcopenshell.util.element.get_materials(wall)
for mat in materials:
    print(f"  Material: {mat.Name}")
```

### Traverse Spatial Hierarchy

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")

# Get all elements in a building storey
storey = model.by_type("IfcBuildingStorey")[0]
children = ifcopenshell.util.element.get_decomposition(storey)
print(f"Storey '{storey.Name}' contains {len(children)} elements:")
for child in children:
    print(f"  {child.is_a()}: {child.Name}")

# Get parent aggregate
parent = ifcopenshell.util.element.get_aggregate(storey)
if parent:
    print(f"Parent: {parent.is_a()} - {parent.Name}")

# Navigate upward from element to project
element = model.by_type("IfcWall")[0]
container = ifcopenshell.util.element.get_container(element)
building = ifcopenshell.util.element.get_aggregate(container) if container else None
site = ifcopenshell.util.element.get_aggregate(building) if building else None
project = ifcopenshell.util.element.get_aggregate(site) if site else None
```

### Work with Groups

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")

groups = model.by_type("IfcGroup")
for group in groups:
    members = ifcopenshell.util.element.get_grouped_by(group)
    print(f"Group '{group.Name}': {len(members)} members")
    for member in members:
        print(f"  {member.is_a()}: {member.Name}")
```

---

## 3. Selector Query Filtering

### Basic Element Selection

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# All walls
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")
print(f"Found {len(walls)} walls")

# All walls and slabs
elements = ifcopenshell.util.selector.filter_elements(
    model, "IfcWall, IfcSlab")

# Exclude a class
non_slabs = ifcopenshell.util.selector.filter_elements(model, "! IfcSlab")
```

### Filter by Attributes and Properties

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# By exact name
named = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Name="External Wall 001"')

# By name containing substring
ext = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Name *= "External"')

# By property set value
external_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')

# By type name
typed = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, type="200mm Concrete"')

# By spatial container
ground_floor = ifcopenshell.util.selector.filter_elements(
    model, 'IfcBuildingElement, container="Ground Floor"')

# By material
concrete_elements = ifcopenshell.util.selector.filter_elements(
    model, 'IfcElement, material="Concrete"')

# Non-null attribute check
with_description = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Description != None')
```

### Subset Filtering

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# First get all walls
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")

# Then filter that subset further
external_walls = ifcopenshell.util.selector.filter_elements(
    model, '/Pset_WallCommon/.IsExternal = True', elements=walls)
```

### Get and Set Element Values

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get a value using query syntax
name = ifcopenshell.util.selector.get_element_value(wall, "Name")
is_ext = ifcopenshell.util.selector.get_element_value(
    wall, "/Pset_WallCommon/.IsExternal")
```

---

## 4. Unit Conversion

### Get Unit Scale for Coordinate Conversion

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# Get scale to convert from file units to SI metres
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Unit scale: {unit_scale}")
# millimetres: 0.001
# metres:      1.0
# feet:        0.3048

# Convert a raw coordinate value
raw_value = 5000.0  # from IFC file
metres = raw_value * unit_scale
print(f"{raw_value} file units = {metres} metres")
```

### Query Project Units

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# Get length unit
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
if length_unit:
    name = ifcopenshell.util.unit.get_full_unit_name(length_unit)
    print(f"Length unit: {name}")  # e.g., "MILLI METRE"

# Get area and volume units
area_unit = ifcopenshell.util.unit.get_project_unit(model, "AREAUNIT")
volume_unit = ifcopenshell.util.unit.get_project_unit(model, "VOLUMEUNIT")
angle_unit = ifcopenshell.util.unit.get_project_unit(model, "PLANEANGLEUNIT")
```

### Convert Between Unit Systems

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.unit

# Metres to millimetres
mm = ifcopenshell.util.unit.convert(
    value=1.0,
    from_prefix=None, from_unit="METRE",
    to_prefix="MILLI", to_unit="METRE")
print(f"1 m = {mm} mm")  # 1000.0

# Feet to metres
m = ifcopenshell.util.unit.convert(
    value=12.0,
    from_prefix=None, from_unit="FOOT",
    to_prefix=None, to_unit="METRE")
print(f"12 ft = {m} m")  # 3.6576

# Millimetres to metres
m2 = ifcopenshell.util.unit.convert(
    value=5000.0,
    from_prefix="MILLI", from_unit="METRE",
    to_prefix=None, to_unit="METRE")
print(f"5000 mm = {m2} m")  # 5.0
```

---

## 5. Placement and Coordinates

### Get Element Position

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.placement
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

wall = model.by_type("IfcWall")[0]

# Get absolute 4x4 transformation matrix
matrix = ifcopenshell.util.placement.get_local_placement(
    wall.ObjectPlacement)

# Extract position (in project units)
x = matrix[0][3]
y = matrix[1][3]
z = matrix[2][3]

# Convert to metres
x_m, y_m, z_m = x * unit_scale, y * unit_scale, z * unit_scale
print(f"Position (metres): ({x_m:.3f}, {y_m:.3f}, {z_m:.3f})")
```

### Get All Element Positions

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.placement
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

for wall in model.by_type("IfcWall"):
    if wall.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            wall.ObjectPlacement)
        pos = matrix[:3, 3] * unit_scale
        print(f"{wall.Name}: ({pos[0]:.3f}, {pos[1]:.3f}, {pos[2]:.3f}) m")
```

### Get Storey Elevations

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.placement
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

for storey in model.by_type("IfcBuildingStorey"):
    elevation = ifcopenshell.util.placement.get_storey_elevation(storey)
    print(f"{storey.Name}: {elevation * unit_scale:.3f} m")
```

---

## 6. Date and Time Conversion

### Convert IFC Dates to Python

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.date

model = ifcopenshell.open("model.ifc")

# From IfcTimeStamp (integer, e.g., OwnerHistory.CreationDate)
owner = model.by_type("IfcOwnerHistory")[0]
if owner.CreationDate:
    dt = ifcopenshell.util.date.ifc2datetime(owner.CreationDate)
    print(f"Created: {dt}")

# From IfcDate string
py_date = ifcopenshell.util.date.ifc2datetime("2024-06-15")
print(f"Date: {py_date}")  # datetime.date(2024, 6, 15)

# From IfcDateTime string
py_dt = ifcopenshell.util.date.ifc2datetime("2024-06-15T10:30:00")
print(f"DateTime: {py_dt}")

# From IfcDuration string (ISO 8601)
delta = ifcopenshell.util.date.ifc2datetime("P30D")
print(f"Duration: {delta}")  # timedelta(days=30)
```

### Convert Python to IFC Formats

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.date
from datetime import datetime

now = datetime.now()

# To IfcDateTime
ifc_dt = ifcopenshell.util.date.datetime2ifc(now, "IfcDateTime")
print(f"IfcDateTime: {ifc_dt}")  # "2024-06-15T10:30:00"

# To IfcDate
ifc_date = ifcopenshell.util.date.datetime2ifc(now, "IfcDate")
print(f"IfcDate: {ifc_date}")  # "2024-06-15"

# To IfcTimeStamp (Unix epoch integer)
ifc_ts = ifcopenshell.util.date.datetime2ifc(now, "IfcTimeStamp")
print(f"IfcTimeStamp: {ifc_ts}")  # e.g., 1718438400
```

---

## 7. Classification References

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.classification

model = ifcopenshell.open("model.ifc")

# Get classification references for a specific element
wall = model.by_type("IfcWall")[0]
refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    classification = ifcopenshell.util.classification.get_classification(ref)
    print(f"System: {classification.Name}")
    print(f"  Code: {ref.Identification}")
    print(f"  Name: {ref.Name}")

# Check all elements with classifications
for element in model.by_type("IfcBuildingElement"):
    refs = ifcopenshell.util.classification.get_references(element)
    if refs:
        print(f"\n{element.is_a()} '{element.Name}':")
        for ref in refs:
            print(f"  {ref.Identification}: {ref.Name}")
```

---

## 8. Geometry Extraction

### Extract Volume and Area

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.shape
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

for wall in model.by_type("IfcWall"):
    try:
        shape = ifcopenshell.geom.create_shape(settings, wall)
        volume = ifcopenshell.util.shape.get_volume(shape.geometry)
        area = ifcopenshell.util.shape.get_area(shape.geometry)
        print(f"{wall.Name}: volume={volume:.4f} m³, area={area:.4f} m²")
    except RuntimeError:
        print(f"{wall.Name}: geometry processing failed")
```

### Extract Bounding Box and Elevation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.shape

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

wall = model.by_type("IfcWall")[0]
shape = ifcopenshell.geom.create_shape(settings, wall)

bbox = ifcopenshell.util.shape.get_bbox(shape.geometry)
print(f"Min corner: {bbox[0]}")
print(f"Max corner: {bbox[1]}")

centroid = ifcopenshell.util.shape.get_bbox_centroid(shape.geometry)
print(f"Centroid: {centroid}")

bottom = ifcopenshell.util.shape.get_bottom_elevation(shape.geometry)
top = ifcopenshell.util.shape.get_top_elevation(shape.geometry)
print(f"Bottom Z: {bottom}, Top Z: {top}, Height: {top - bottom}")
```

---

## 9. Cost and Schedule Data

### Extract Cost Schedule

```python
# IfcOpenShell — IFC4/IFC4X3
import ifcopenshell
import ifcopenshell.util.cost

model = ifcopenshell.open("model.ifc")

schedules = model.by_type("IfcCostSchedule")
for schedule in schedules:
    print(f"\nCost Schedule: {schedule.Name}")
    root_items = ifcopenshell.util.cost.get_root_cost_items(schedule)
    for item in root_items:
        values = ifcopenshell.util.cost.get_cost_values(item)
        nested = ifcopenshell.util.cost.get_nested_cost_items(item)
        print(f"  {item.Name}: {len(values)} values, {len(nested)} sub-items")
```

### Extract Work Schedule

```python
# IfcOpenShell — IFC4/IFC4X3
import ifcopenshell
import ifcopenshell.util.sequence
import ifcopenshell.util.date

model = ifcopenshell.open("model.ifc")

for schedule in model.by_type("IfcWorkSchedule"):
    print(f"\nWork Schedule: {schedule.Name}")
    for task in ifcopenshell.util.sequence.get_root_tasks(schedule):
        print(f"  Task: {task.Name}")
        if hasattr(task, "TaskTime") and task.TaskTime:
            start = task.TaskTime.ScheduleStart
            finish = task.TaskTime.ScheduleFinish
            if start:
                print(f"    Start: {ifcopenshell.util.date.ifc2datetime(start)}")
            if finish:
                print(f"    Finish: {ifcopenshell.util.date.ifc2datetime(finish)}")

        nested = ifcopenshell.util.sequence.get_nested_tasks(task)
        for sub in nested:
            print(f"    Sub-task: {sub.Name}")
```

---

## 10. Complete Workflow: Building Element Report

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit
import ifcopenshell.util.placement
import ifcopenshell.util.classification

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

# Get project unit info
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")
unit_name = ifcopenshell.util.unit.get_full_unit_name(length_unit) if length_unit else "unknown"
print(f"Project length unit: {unit_name}")
print(f"Schema: {model.schema}")
print(f"Unit scale to metres: {unit_scale}")
print("=" * 60)

for wall in model.by_type("IfcWall"):
    print(f"\nWall: {wall.Name} (#{wall.id()})")

    # Type
    wall_type = ifcopenshell.util.element.get_type(wall)
    if wall_type:
        print(f"  Type: {wall_type.Name}")

    # Container
    container = ifcopenshell.util.element.get_container(wall)
    if container:
        print(f"  Location: {container.Name}")

    # Position
    if wall.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            wall.ObjectPlacement)
        pos = matrix[:3, 3] * unit_scale
        print(f"  Position: ({pos[0]:.3f}, {pos[1]:.3f}, {pos[2]:.3f}) m")

    # Materials
    materials = ifcopenshell.util.element.get_materials(wall)
    if materials:
        mat_names = [m.Name for m in materials]
        print(f"  Materials: {', '.join(mat_names)}")

    # Key properties
    is_ext = ifcopenshell.util.element.get_pset(
        wall, "Pset_WallCommon", "IsExternal")
    fire = ifcopenshell.util.element.get_pset(
        wall, "Pset_WallCommon", "FireRating")
    if is_ext is not None:
        print(f"  External: {is_ext}")
    if fire:
        print(f"  Fire Rating: {fire}")

    # Classifications
    refs = ifcopenshell.util.classification.get_references(wall)
    for ref in refs:
        print(f"  Classification: {ref.Identification} - {ref.Name}")
```

---

## 11. Copy and Duplicate Elements

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Shallow copy — new entity with new GlobalId, same property links
new_wall = ifcopenshell.util.element.copy(model, wall)
print(f"Original: #{wall.id()}, Copy: #{new_wall.id()}")
print(f"Original GlobalId: {wall.GlobalId}")
print(f"Copy GlobalId: {new_wall.GlobalId}")  # Different

# Deep copy — copies element + directly related geometry, psets, etc.
deep_wall = ifcopenshell.util.element.copy_deep(model, wall)

# Deep copy excluding specific entity types
deep_wall_no_psets = ifcopenshell.util.element.copy_deep(
    model, wall, exclude=["IfcPropertySet"])
```
