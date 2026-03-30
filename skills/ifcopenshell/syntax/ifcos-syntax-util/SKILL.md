---
name: ifcos-syntax-util
description: >
  Use when extracting data from IFC models using utility functions -- element properties,
  selector syntax, placement calculations, unit conversion, or cost/schedule data.
  Prevents the common mistake of manually parsing IFC relationships instead of using
  ifcopenshell.util helpers. Covers element utilities, selector syntax, placement
  helpers, date/unit conversion, and shape extraction.
  Keywords: ifcopenshell.util, selector, placement, unit conversion, element utility,
  shape extraction, ifcopenshell.util.element, ifcopenshell.util.selector.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Utility Modules (`ifcopenshell.util.*`)

## Quick Reference

### Decision Tree: Choosing the Right Utility Module

```
What data do you need from an IFC element?
├── Property sets, types, containers, materials?
│   └── ifcopenshell.util.element
│       ├── get_psets()         → all property sets as dict
│       ├── get_pset()          → single property set or property
│       ├── get_type()          → type element (e.g., IfcWallType)
│       ├── get_container()     → spatial container (e.g., IfcBuildingStorey)
│       ├── get_material()      → material assignment
│       ├── get_materials()     → list of individual materials
│       ├── get_decomposition() → child elements
│       └── get_aggregate()     → parent aggregate
│
├── Query/filter elements by criteria?
│   └── ifcopenshell.util.selector
│       └── filter_elements()   → CSS-like query syntax
│
├── Position/coordinates?
│   └── ifcopenshell.util.placement
│       ├── get_local_placement() → 4x4 transformation matrix
│       └── get_storey_elevation() → storey Z elevation
│
├── Unit conversion?
│   └── ifcopenshell.util.unit
│       ├── calculate_unit_scale() → scale factor to SI metres
│       ├── convert()              → value between unit systems
│       └── get_project_unit()     → project's default unit
│
├── Dates, durations, timestamps?
│   └── ifcopenshell.util.date
│       ├── ifc2datetime()  → IFC date → Python datetime
│       └── datetime2ifc()  → Python datetime → IFC format
│
├── Geometry metrics (area, volume, bbox)?
│   └── ifcopenshell.util.shape (requires processed geometry)
│       ├── get_volume()    → element volume
│       ├── get_area()      → surface area
│       ├── get_bbox()      → bounding box
│       └── get_vertices()  → vertex coordinates
│
├── Classification references?
│   └── ifcopenshell.util.classification
│       ├── get_references() → classification refs for element
│       └── get_classification() → parent classification system
│
├── Cost data?
│   └── ifcopenshell.util.cost
│       ├── get_cost_items_for_product() → cost items linked to product
│       └── get_cost_values()            → cost item values
│
├── Schedule/sequence data?
│   └── ifcopenshell.util.sequence
│       ├── get_tasks_for_product() → tasks linked to product
│       └── count_working_days()    → working days between dates
│
└── Schema introspection (attribute types, enums)?
    └── ifcopenshell.util.attribute
        ├── get_primitive_type() → Python type for IFC attribute
        └── get_enum_items()    → enum options for attribute
```

### Critical Warnings

- **ALWAYS** import utility modules explicitly: `import ifcopenshell.util.element`, NOT `from ifcopenshell.util import *`.
- **ALWAYS** multiply raw coordinate values by `calculate_unit_scale(model)` to get SI metres. IFC files store coordinates in project units (millimetres, feet, etc.).
- **ALWAYS** use `filter_elements()` (modern API) for selector queries. NEVER use the deprecated `Selector().parse()` class.
- **NEVER** traverse IFC inverse relationships manually when a `ifcopenshell.util.element` helper exists. Manual traversal is error-prone and schema-dependent.
- **NEVER** assume property set names are standardized. Custom property sets vary per project. ALWAYS check for `None` returns.
- **ALWAYS** check `model.schema` before using schema-specific property set names (e.g., `Pset_WallCommon` exists in IFC4 but attribute names differ from IFC2X3).
- **NEVER** call `ifcopenshell.util.shape` functions without first processing geometry via `ifcopenshell.geom`. Shape utilities operate on processed geometry objects, NOT raw IFC entities.

---

## Essential Patterns

### Pattern 1: Extract All Properties from an Element

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# All property sets (excludes quantity sets by default)
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"id": 42, "IsExternal": True, ...}, ...}

# Include quantity sets
all_props = ifcopenshell.util.element.get_psets(wall, psets_only=False)

# Only quantity sets
qsets = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# Single property set by name
wall_common = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon")
# Returns dict or None

# Single property value
is_external = ifcopenshell.util.element.get_pset(
    wall, "Pset_WallCommon", "IsExternal")
# Returns value or None
```

### Pattern 2: Navigate Element Relationships

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Type element (IfcWallType, IfcDoorType, etc.)
wall_type = ifcopenshell.util.element.get_type(wall)

# Spatial container (IfcBuildingStorey, IfcSpace, etc.)
container = ifcopenshell.util.element.get_container(wall)

# Material assignment
material = ifcopenshell.util.element.get_material(wall)

# Individual materials as flat list
materials = ifcopenshell.util.element.get_materials(wall)

# Parent aggregate
parent = ifcopenshell.util.element.get_aggregate(wall)

# Child elements (decomposition)
building = model.by_type("IfcBuilding")[0]
children = ifcopenshell.util.element.get_decomposition(building)
```

### Pattern 3: Filter Elements with Selector Queries

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# By IFC class
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")

# Multiple classes
elements = ifcopenshell.util.selector.filter_elements(
    model, "IfcWall, IfcSlab")

# By attribute value
named = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Name="External Wall"')

# By property set value
external = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')

# By spatial container
ground = ifcopenshell.util.selector.filter_elements(
    model, 'IfcBuildingElement, container="Ground Floor"')

# By material
concrete = ifcopenshell.util.selector.filter_elements(
    model, 'IfcElement, material="Concrete"')

# By type name
typed = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, type="WT01"')

# Partial match (contains)
ext_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Name *= "EXT"')

# Exclusion
non_slabs = ifcopenshell.util.selector.filter_elements(
    model, "! IfcSlab")
```

### Pattern 4: Unit Conversion

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# CRITICAL: Get scale factor from project units to SI metres
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# Millimetres → 0.001, Metres → 1.0, Feet → 0.3048

# Convert raw coordinate to metres
raw_length = 5000.0  # e.g., value from IFC file in mm
length_metres = raw_length * unit_scale

# Get project's length unit entity
length_unit = ifcopenshell.util.unit.get_project_unit(model, "LENGTHUNIT")

# Convert between specific units
value_mm = ifcopenshell.util.unit.convert(
    value=1.0,
    from_prefix=None, from_unit="METRE",
    to_prefix="MILLI", to_unit="METRE"
)  # Returns 1000.0
```

### Pattern 5: Get Element Placement

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.placement

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get absolute 4x4 transformation matrix
matrix = ifcopenshell.util.placement.get_local_placement(
    wall.ObjectPlacement)
# Returns numpy 4x4 ndarray

# Extract position
x, y, z = matrix[0][3], matrix[1][3], matrix[2][3]

# Extract rotation (3x3 submatrix)
rotation = matrix[:3, :3]

# Storey elevation
storey = model.by_type("IfcBuildingStorey")[0]
elevation = ifcopenshell.util.placement.get_storey_elevation(storey)
```

### Pattern 6: Date Conversion

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.date
from datetime import datetime

model = ifcopenshell.open("model.ifc")

# IFC timestamp → Python datetime
owner = model.by_type("IfcOwnerHistory")[0]
if owner.CreationDate:
    dt = ifcopenshell.util.date.ifc2datetime(owner.CreationDate)

# IFC date string → Python date
py_date = ifcopenshell.util.date.ifc2datetime("2024-06-15")

# IFC duration → Python timedelta
duration = ifcopenshell.util.date.ifc2datetime("P30D")

# Python datetime → IFC format
ifc_dt = ifcopenshell.util.date.datetime2ifc(
    datetime(2024, 6, 15, 10, 30), "IfcDateTime")
# Returns "2024-06-15T10:30:00"

ifc_date = ifcopenshell.util.date.datetime2ifc(
    datetime(2024, 6, 15), "IfcDate")
# Returns "2024-06-15"
```

---

## Common Operations

### Classification Lookup

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.classification

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    classification = ifcopenshell.util.classification.get_classification(ref)
    print(f"{classification.Name}: {ref.Identification} - {ref.Name}")
```

### Cost Item Lookup

```python
# IfcOpenShell: IFC4/IFC4X3 (cost entities not in IFC2X3)
import ifcopenshell
import ifcopenshell.util.cost

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

cost_items = ifcopenshell.util.cost.get_cost_items_for_product(wall)
for item in cost_items:
    values = ifcopenshell.util.cost.get_cost_values(item)
    schedule = ifcopenshell.util.cost.get_cost_schedule(item)
```

### Schedule Task Lookup

```python
# IfcOpenShell: IFC4/IFC4X3
import ifcopenshell
import ifcopenshell.util.sequence

model = ifcopenshell.open("model.ifc")

schedules = model.by_type("IfcWorkSchedule")
for schedule in schedules:
    for task in ifcopenshell.util.sequence.get_root_tasks(schedule):
        print(f"Task: {task.Name}")
        calendar = ifcopenshell.util.sequence.get_calendar(task)
        nested = ifcopenshell.util.sequence.get_nested_tasks(task)
```

### Attribute Schema Introspection

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.attribute

# Get primitive Python type for an IFC attribute
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
entity_decl = schema.declaration_by_name("IfcWall")
attr = entity_decl.all_attributes()[1]  # e.g., Name attribute
ptype = ifcopenshell.util.attribute.get_primitive_type(attr)

# Get enum options
enum_attr = entity_decl.all_attributes()[7]  # PredefinedType
items = ifcopenshell.util.attribute.get_enum_items(enum_attr)
```

### Geometry Extraction (Requires Processed Shapes)

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.shape

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

wall = model.by_type("IfcWall")[0]

# REQUIRED: Process geometry first
shape = ifcopenshell.geom.create_shape(settings, wall)

# Now use shape utilities
volume = ifcopenshell.util.shape.get_volume(shape.geometry)
area = ifcopenshell.util.shape.get_area(shape.geometry)
vertices = ifcopenshell.util.shape.get_vertices(shape.geometry)
bbox = ifcopenshell.util.shape.get_bbox(shape.geometry)
bottom_z = ifcopenshell.util.shape.get_bottom_elevation(shape.geometry)
top_z = ifcopenshell.util.shape.get_top_elevation(shape.geometry)
```

### Copy/Duplicate Elements

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Shallow copy (new GlobalId, same relationships)
new_wall = ifcopenshell.util.element.copy(model, wall)

# Deep copy (copies element + directly related subelements)
deep_wall = ifcopenshell.util.element.copy_deep(model, wall)
```

---

## Selector Query Syntax Reference

| Syntax | Meaning | Example |
|--------|---------|---------|
| `IfcType` | Select by IFC class | `"IfcWall"` |
| `IfcType, IfcType` | Multiple classes | `"IfcWall, IfcSlab"` |
| `, Name="X"` | Filter by Name equals | `'IfcWall, Name="W01"'` |
| `, Name *= "X"` | Name contains | `'IfcWall, Name *= "EXT"'` |
| `, Name != None` | Attribute is not null | `'IfcWall, Name != None'` |
| `, /Pset/.Prop = val` | Filter by property value | `'IfcWall, /Pset_WallCommon/.IsExternal = True'` |
| `, type="X"` | Filter by type name | `'IfcWall, type="WT01"'` |
| `, container="X"` | Filter by spatial container | `'IfcElement, container="Level 1"'` |
| `, material="X"` | Filter by material | `'IfcElement, material="Concrete"'` |
| `! IfcType` | Exclude class | `"! IfcSlab"` |

---

## Module Summary Table

| Module | Import | Primary Functions | Schema Sensitivity |
|--------|--------|-------------------|--------------------|
| `element` | `ifcopenshell.util.element` | `get_psets`, `get_type`, `get_container`, `get_material` | Low — works across all schemas |
| `selector` | `ifcopenshell.util.selector` | `filter_elements`, `get_element_value` | Low — query syntax is schema-agnostic |
| `placement` | `ifcopenshell.util.placement` | `get_local_placement`, `get_storey_elevation` | Low — placement structure is consistent |
| `unit` | `ifcopenshell.util.unit` | `calculate_unit_scale`, `convert`, `get_project_unit` | Low — unit types are consistent |
| `date` | `ifcopenshell.util.date` | `ifc2datetime`, `datetime2ifc` | Medium — IfcCalendarDate is IFC2X3 only |
| `shape` | `ifcopenshell.util.shape` | `get_volume`, `get_area`, `get_bbox`, `get_vertices` | Low — operates on processed geometry |
| `classification` | `ifcopenshell.util.classification` | `get_references`, `get_classification` | Low |
| `cost` | `ifcopenshell.util.cost` | `get_cost_items_for_product`, `get_cost_values` | High — IFC4+ only |
| `sequence` | `ifcopenshell.util.sequence` | `get_root_tasks`, `count_working_days` | High — IFC4+ only |
| `attribute` | `ifcopenshell.util.attribute` | `get_primitive_type`, `get_enum_items` | Low — schema introspection |

---

## Version Notes

### IFC2X3 Limitations
- `IfcCalendarDate` entity used instead of ISO 8601 date strings. Use `ifc2datetime()` which handles both.
- Cost scheduling entities (`IfcCostSchedule`, `IfcCostItem`) exist but with fewer attributes.
- `IfcTask` has a different attribute structure (no `TaskTime` in IFC2X3).

### IFC4 vs IFC4X3
- IFC4X3 adds `IfcAlignment` and related entities supported by `ifcopenshell.util.alignment`.
- Property set names are consistent between IFC4 and IFC4X3 for building elements.

### IfcOpenShell Version Notes
- `filter_elements()` replaced `Selector().parse()`. The `Selector` class is deprecated.
- `get_pset()` (singular) is a convenience wrapper added in recent versions. Falls back to `get_psets()` if unavailable.

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all utility functions
- [Working Code Examples](references/examples.md) — End-to-end examples for common scenarios
- [Anti-Patterns](references/anti-patterns.md) — Common utility module mistakes and how to avoid them
