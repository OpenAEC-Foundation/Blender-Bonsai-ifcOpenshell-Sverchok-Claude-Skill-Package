---
name: ifcos-syntax-elements
description: >
  Use when querying, traversing, or extracting data from IFC elements -- by_type, by_id,
  by_guid, inverse references, or property extraction. Prevents the common mistake of
  manually traversing relationships instead of using the universal property extraction
  pattern (IsDefinedBy -> HasProperties). Covers get_info(), is_a(), GUID utilities,
  and attribute access patterns.
  Keywords: by_type, by_id, by_guid, get_info, is_a, inverse, IsDefinedBy, HasProperties,
  IFC query, element traversal, property extraction.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Element Traversal and Querying

## Quick Reference

### Decision Tree: Finding Elements

```
Need to find IFC elements?
├── Know the IFC class? (IfcWall, IfcDoor, etc.)
│   └── model.by_type("IfcWall")
│       ├── Need subtypes included? → include_subtypes=True (DEFAULT)
│       └── Need exact type only? → include_subtypes=False
│
├── Know the STEP ID? (#123 in .ifc file)
│   └── model.by_id(123)
│       └── WARNING: STEP IDs are NOT persistent across re-exports
│
├── Know the GlobalId? (22-char IFC GUID)
│   └── model.by_guid("2O2Fr$t4X7Zf8NOew3FLOH")
│       └── GlobalId is ONLY on IfcRoot-derived entities
│
├── Need complex filtering? (by property, container, material)
│   └── ifcopenshell.util.selector.filter_elements(model, query)
│
└── Need ALL entities in the file?
    └── for entity in model: ...
```

### Decision Tree: Reading Element Data

```
Need data from an element?
├── Single attribute? → element.Name, element.GlobalId, etc.
│
├── All attributes as dict? → element.get_info()
│   ├── Need referenced entities expanded? → get_info(recursive=True)
│   │   └── WARNING: EXPENSIVE on large models — avoid in loops
│   └── Need only primitive values? → get_info(scalar_only=True)
│
├── Type checking?
│   ├── Get type name → element.is_a()  # returns "IfcWall"
│   └── Check inheritance → element.is_a("IfcElement")  # returns True/False
│
├── Property sets?
│   ├── RECOMMENDED → ifcopenshell.util.element.get_psets(element)
│   └── Manual pattern → element.IsDefinedBy traversal (see below)
│
├── Spatial container? → ifcopenshell.util.element.get_container(element)
├── Element type? → ifcopenshell.util.element.get_type(element)
├── Material? → ifcopenshell.util.element.get_material(element)
└── STEP ID? → element.id()
```

### Critical Warnings

- **ALWAYS** use `model.by_type()` for type-based queries — it is internally cached after the first call per type.
- **ALWAYS** use `ifcopenshell.guid.new()` to generate GlobalIds. NEVER construct GUID strings manually.
- **ALWAYS** use `ifcopenshell.util.element.get_psets()` for property extraction in production code. Use the manual `IsDefinedBy` traversal only when you need fine-grained control.
- **NEVER** use STEP IDs (`entity.id()`) as persistent identifiers. They change when files are re-exported. Use `GlobalId` for cross-session identification.
- **NEVER** call `get_info(recursive=True)` in a loop over many elements — it recursively expands all references and is extremely expensive.
- **NEVER** assume an entity has a `GlobalId`. Only `IfcRoot`-derived entities (IfcWall, IfcProject, etc.) have GlobalIds. Low-level entities (IfcCartesianPoint, IfcDirection) do NOT.
- **ALWAYS** handle `None` values when accessing optional attributes. Many IFC attributes are optional and return `None`.
- **ALWAYS** check `prop.NominalValue` is not `None` before accessing `.wrappedValue` in the property extraction pattern.

---

## Essential Patterns

### Pattern 1: Query Elements by IFC Class

```python
# IfcOpenShell: all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")

# Get all walls (includes subtypes like IfcWallStandardCase in IFC2X3)
walls = model.by_type("IfcWall")

# Get ONLY exact IfcWall, NOT subtypes
walls_exact = model.by_type("IfcWall", include_subtypes=False)

# Common element queries
storeys = model.by_type("IfcBuildingStorey")
spaces = model.by_type("IfcSpace")
doors = model.by_type("IfcDoor")
windows = model.by_type("IfcWindow")
slabs = model.by_type("IfcSlab")
columns = model.by_type("IfcColumn")
beams = model.by_type("IfcBeam")
```

### Pattern 2: Query by ID and GUID

```python
# IfcOpenShell: all schema versions
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.open("model.ifc")

# By STEP ID (the #123 number in .ifc files)
entity = model.by_id(123)

# By GlobalId (22-character IFC GUID)
entity = model.by_guid("2O2Fr$t4X7Zf8NOew3FLOH")

# Generate a new GlobalId
new_guid = ifcopenshell.guid.new()

# Convert between IFC GUID and standard UUID
standard_uuid = ifcopenshell.guid.expand(new_guid)   # → UUID string
ifc_guid = ifcopenshell.guid.compress(standard_uuid)  # → 22-char GUID
```

### Pattern 3: Entity Attribute Access and Type Checking

```python
# IfcOpenShell: all schema versions
wall = model.by_type("IfcWall")[0]

# Direct attribute access
print(wall.Name)           # "Wall 001" or None
print(wall.GlobalId)       # "2O2Fr$t4X7Zf8NOew3FLOH"
print(wall.Description)    # Optional, may be None

# Type checking
wall.is_a()                # "IfcWall" (returns type name)
wall.is_a("IfcWall")       # True (exact match)
wall.is_a("IfcElement")    # True (parent class)
wall.is_a("IfcRoot")       # True (ancestor)
wall.is_a("IfcSlab")       # False

# STEP ID
wall.id()                  # 123 (integer)

# All attributes as dict
info = wall.get_info()
# {"id": 123, "type": "IfcWall", "GlobalId": "...", "Name": "...", ...}
```

### Pattern 4: Inverse References

```python
# IfcOpenShell: all schema versions
wall = model.by_type("IfcWall")[0]

# Get ALL entities that REFERENCE this wall
inverse = model.get_inverse(wall)  # returns set

# Find which storey contains this wall
for rel in model.get_inverse(wall):
    if rel.is_a("IfcRelContainedInSpatialStructure"):
        print(f"Wall is in: {rel.RelatingStructure.Name}")

# Find property sets attached to an element
for rel in model.get_inverse(wall):
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            print(f"PSet: {pset.Name}")

# Find the type object of an element
for rel in model.get_inverse(wall):
    if rel.is_a("IfcRelDefinesByType"):
        print(f"Type: {rel.RelatingType.Name}")
```

### Pattern 5: Universal IFC Property Extraction

This is the fundamental pattern for extracting properties from ANY IFC element. It traverses: `IsDefinedBy` → `IfcRelDefinesByProperties` → `IfcPropertySet` → `HasProperties` → `wrappedValue`.

```python
# IfcOpenShell: all schema versions
# UNIVERSAL PATTERN: Works with IfcOpenShell, Bonsai, web-ifc
def extract_properties(element):
    """Extract all property sets and their values from an IFC element."""
    result = {}
    for rel in element.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            pset = rel.RelatingPropertyDefinition
            if pset.is_a("IfcPropertySet"):
                props = {}
                for prop in pset.HasProperties:
                    if prop.is_a("IfcPropertySingleValue") and prop.NominalValue:
                        props[prop.Name] = prop.NominalValue.wrappedValue
                result[pset.Name] = props
    return result

wall = model.by_type("IfcWall")[0]
all_props = extract_properties(wall)
for pset_name, props in all_props.items():
    print(f"\n{pset_name}:")
    for name, value in props.items():
        print(f"  {name}: {value}")
```

### Pattern 6: Recommended: Use util.element Helpers

```python
# IfcOpenShell: all schema versions
import ifcopenshell.util.element

wall = model.by_type("IfcWall")[0]

# Get ALL property sets as {pset_name: {prop_name: value}}
psets = ifcopenshell.util.element.get_psets(wall)

# Include quantity sets (IfcElementQuantity)
psets_and_qsets = ifcopenshell.util.element.get_psets(wall, psets_only=False)

# Get only quantity sets
qsets = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# Get element type
wall_type = ifcopenshell.util.element.get_type(wall)

# Get spatial container (typically IfcBuildingStorey)
container = ifcopenshell.util.element.get_container(wall)

# Get material
material = ifcopenshell.util.element.get_material(wall)

# Get all materials as list
materials = ifcopenshell.util.element.get_materials(wall)

# Get parent aggregate
parent = ifcopenshell.util.element.get_aggregate(wall)

# Get decomposition (children)
building = model.by_type("IfcBuilding")[0]
children = ifcopenshell.util.element.get_decomposition(building)
```

---

## Common Operations

### Iterating Over All Entities

```python
# IfcOpenShell: all schema versions
# Iterate over ALL entities
for entity in model:
    pass

# Total entity count
total = len(model)

# Count entities by type
from collections import Counter
type_counts = Counter(entity.is_a() for entity in model)
for ifc_type, count in type_counts.most_common(20):
    print(f"  {ifc_type}: {count}")
```

### Finding Openings in a Wall

```python
# IfcOpenShell: all schema versions
def get_openings(model, wall):
    """Get opening elements that void a wall."""
    openings = []
    for rel in model.get_inverse(wall):
        if rel.is_a("IfcRelVoidsElement"):
            openings.append(rel.RelatedOpeningElement)
    return openings
```

### CSS-like Element Selection

```python
# IfcOpenShell: all schema versions
import ifcopenshell.util.selector

# Select walls by class
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")

# Select by property value
ext_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')

# Select by container
ground_elements = ifcopenshell.util.selector.filter_elements(
    model, 'IfcBuildingElement, container="Ground Floor"')
```

### Error Handling for Lookups

```python
# IfcOpenShell: all schema versions
# by_id and by_guid raise RuntimeError if not found
try:
    entity = model.by_id(999999)
except RuntimeError:
    print("Entity not found")

try:
    entity = model.by_guid("nonexistent_guid_12345")
except RuntimeError:
    print("Entity not found")

# by_type returns empty tuple if no matches
walls = model.by_type("IfcWall")  # () if none exist
```

---

## Version Notes

### Schema-Specific Entity Names

| Concept | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| Building elements parent | `IfcBuildingElement` | `IfcBuildingElement` | `IfcBuiltElement` |
| Spatial elements parent | `IfcSpatialStructureElement` | `IfcSpatialElement` | `IfcSpatialElement` |
| Wall subtype | `IfcWallStandardCase` | `IfcWall` (subtype removed) | `IfcWall` |
| Facility types | N/A | N/A | `IfcBridge`, `IfcRoad`, `IfcRailway`, `IfcMarineFacility` |

### Query API Compatibility

The query methods (`by_type`, `by_id`, `by_guid`, `get_inverse`, `get_info`, `is_a`) are **schema-agnostic** — they work identically across IFC2X3, IFC4, and IFC4X3. Only the entity class names passed to these methods differ per schema.

### GUID Module

`ifcopenshell.guid` is schema-independent. The same GUID format (22-character base64) is used across all IFC versions.

| Function | Purpose |
|----------|---------|
| `ifcopenshell.guid.new()` | Generate a new IFC GlobalId |
| `ifcopenshell.guid.expand(ifc_guid)` | Convert IFC GUID → standard UUID string |
| `ifcopenshell.guid.compress(uuid_str)` | Convert UUID string → 22-char IFC GUID |

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all element traversal methods
- [Working Code Examples](references/examples.md) — End-to-end examples for element queries and property extraction
- [Anti-Patterns](references/anti-patterns.md) — Common element traversal mistakes and how to avoid them
