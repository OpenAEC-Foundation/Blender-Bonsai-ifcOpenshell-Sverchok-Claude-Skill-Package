---
name: bonsai-syntax-elements
description: "Provides Bonsai IFC element access syntax including tool.Ifc.get() for file access, element-to-Blender-object mapping, IfcStore element retrieval, relating IFC entities to scene objects, and the Bonsai data bridge between bpy.types.Object and IFC elements. Covers core patterns for reading and navigating IFC data within the Bonsai addon context."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai IFC Element Access Syntax

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.*` — NEVER `blenderbim.*`
> **Dependencies**: bonsai-core-architecture, ifcos-syntax-api

## Critical Warnings

1. **ALWAYS** use `bonsai.*` imports. NEVER use `blenderbim.*` — renamed in 2024 (v0.8.0+).
2. **ALWAYS** check `IfcStore.get_file()` and `tool.Ifc.get()` for `None` before any IFC operation.
3. **ALWAYS** use `tool.Ifc.get_entity(obj)` to get an IFC entity from a Blender object. NEVER read `obj.BIMProperties.ifc_definition_id` and call `model.by_id()` manually — the tool method handles edge cases.
4. **NEVER** assume every Blender object has an IFC entity. `get_entity()` returns `None` for non-IFC objects.
5. **NEVER** assume every IFC entity has a Blender object. `get_object()` returns `None` for entities without scene representation.
6. **ALWAYS** run element access code inside Blender (`blender --python`). `bonsai.tool.*` depends on `bpy` and CANNOT run in standalone Python.
7. **NEVER** cache IFC entity references across file reloads. Entity instances become stale when the IFC file is reloaded.
8. **ALWAYS** use `tool.Ifc.link(element, obj)` to create bidirectional mappings. NEVER manually populate `IfcStore.id_map` or `IfcStore.guid_map`.

---

## Decision Tree: How to Access IFC Elements

```
What do you have?
│
├── A Blender object (bpy.types.Object)?
│   │
│   ├── Need the IFC entity?
│   │   └── tool.Ifc.get_entity(obj)
│   │       ├── Returns entity_instance → access .GlobalId, .Name, .is_a(), attributes
│   │       └── Returns None → object is NOT linked to IFC
│   │
│   ├── Need just the IFC step ID?
│   │   └── obj.BIMProperties.ifc_definition_id
│   │       ├── Returns int > 0 → linked to IFC entity with that step ID
│   │       └── Returns 0 → NOT linked to IFC
│   │
│   └── Need IFC class name?
│       └── entity = tool.Ifc.get_entity(obj)
│           └── entity.is_a()  # "IfcWall", "IfcSlab", etc.
│
├── An IFC entity (ifcopenshell.entity_instance)?
│   │
│   ├── Need the Blender object?
│   │   └── tool.Ifc.get_object(entity)
│   │       ├── Returns bpy.types.Object → manipulate via Blender API
│   │       └── Returns None → entity has no Blender scene representation
│   │
│   └── Need another entity related to this one?
│       ├── Type definition?
│       │   └── ifcopenshell.util.element.get_type(entity)
│       ├── Spatial container?
│       │   └── ifcopenshell.util.element.get_container(entity)
│       ├── Property sets?
│       │   └── ifcopenshell.util.element.get_psets(entity)
│       └── Material?
│           └── ifcopenshell.util.element.get_material(entity)
│
├── An IFC class name (string)?
│   │
│   └── Need all entities of that class?
│       └── model = tool.Ifc.get()
│           └── model.by_type("IfcWall")  # Returns list of entity_instance
│
├── An IFC GlobalId (string)?
│   │
│   └── model = tool.Ifc.get()
│       └── model.by_guid("2O2Fr$t4X7Zf8...")  # Returns entity_instance
│
├── An IFC step ID (int)?
│   │
│   ├── Need the IFC entity?
│   │   └── model = tool.Ifc.get()
│   │       └── model.by_id(step_id)  # Returns entity_instance
│   │
│   └── Need the Blender object directly?
│       └── IfcStore.id_map.get(step_id)  # Returns bpy.types.Object or None
│
└── Nothing — need to discover elements?
    │
    ├── All elements in the model?
    │   └── model.by_type("IfcElement")  # All physical elements
    │
    ├── All elements in a specific storey?
    │   └── ifcopenshell.util.element.get_decomposition(storey)
    │
    └── All elements matching a query?
        └── Use ifcopenshell.util.selector (CSS-like syntax)
```

---

## Essential Patterns

### Pattern 1: Get the Live IFC File

```python
# Bonsai v0.8.x — inside Blender context

# Method A: via tool.Ifc (preferred in Bonsai tool/core code)
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Method B: via IfcStore (equivalent, used in operator code)
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")
```

`tool.Ifc.get()` and `IfcStore.get_file()` return the same `ifcopenshell.file` instance. Use `tool.Ifc.get()` in tool/core code; use `IfcStore.get_file()` in operator/UI code.

### Pattern 2: Blender Object → IFC Entity

```python
# Bonsai v0.8.x — inside Blender context
import bpy
import bonsai.tool as tool

obj = bpy.context.active_object
entity = tool.Ifc.get_entity(obj)

if entity is None:
    print(f"{obj.name} is not linked to IFC")
else:
    print(f"IFC class: {entity.is_a()}")
    print(f"GlobalId:  {entity.GlobalId}")
    print(f"Name:      {entity.Name}")
    print(f"Step ID:   {entity.id()}")
```

### Pattern 3: IFC Entity → Blender Object

```python
# Bonsai v0.8.x — inside Blender context
import bonsai.tool as tool

model = tool.Ifc.get()
wall = model.by_type("IfcWall")[0]
obj = tool.Ifc.get_object(wall)

if obj is None:
    print(f"Wall {wall.Name} has no Blender representation")
else:
    print(f"Blender object: {obj.name}")
    obj.select_set(True)  # Select in viewport
```

### Pattern 4: Query Elements by Type

```python
# Bonsai v0.8.x — inside Blender context
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# All walls
walls = model.by_type("IfcWall")

# All physical elements (walls, slabs, columns, doors, etc.)
elements = model.by_type("IfcElement")

# All spatial elements (sites, buildings, storeys, spaces)
spatial = model.by_type("IfcSpatialElement")  # IFC4+
# For IFC2X3: model.by_type("IfcSpatialStructureElement")

# Specific type with subtypes excluded
walls_only = model.by_type("IfcWall", include_subtypes=False)
# Excludes IfcWallStandardCase (IFC2X3) and subtypes
```

### Pattern 5: Look Up by GlobalId or Step ID

```python
# Bonsai v0.8.x — inside Blender context
import bonsai.tool as tool
from bonsai.bim.ifc import IfcStore

model = tool.Ifc.get()

# By GlobalId (22-character IFC identifier)
entity = model.by_guid("2O2Fr$t4X7Zf8NOew3FLOH")

# By step ID (integer)
entity = model.by_id(42)

# Step ID → Blender object (direct lookup, bypasses entity)
obj = IfcStore.id_map.get(42)

# GlobalId → Blender object (direct lookup)
obj = IfcStore.guid_map.get("2O2Fr$t4X7Zf8NOew3FLOH")
```

### Pattern 6: Navigate Entity Relationships

```python
# Bonsai v0.8.x — inside Blender context
import bonsai.tool as tool
import ifcopenshell.util.element

model = tool.Ifc.get()
wall = model.by_type("IfcWall")[0]

# Get the spatial container (storey/space) of an element
container = ifcopenshell.util.element.get_container(wall)
if container:
    print(f"Wall is in: {container.Name} ({container.is_a()})")

# Get the type definition
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    print(f"Wall type: {wall_type.Name}")

# Get all property sets
psets = ifcopenshell.util.element.get_psets(wall)
for pset_name, props in psets.items():
    print(f"  {pset_name}: {props}")

# Get material
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material: {material.is_a()} — {material.Name}")
```

### Pattern 7: Iterate Over All Elements with Blender Objects

```python
# Bonsai v0.8.x — inside Blender context
import bonsai.tool as tool
from bonsai.bim.ifc import IfcStore

model = tool.Ifc.get()

# Method A: Iterate IfcStore.id_map (fastest for Blender-linked elements)
for ifc_id, obj in IfcStore.id_map.items():
    entity = model.by_id(ifc_id)
    print(f"{obj.name} → {entity.is_a()} ({entity.Name})")

# Method B: Iterate IFC entities and look up Blender objects
for wall in model.by_type("IfcWall"):
    obj = tool.Ifc.get_object(wall)
    if obj:
        print(f"{wall.Name} → {obj.name}")
```

### Pattern 8: Check if Object is IFC-Linked

```python
# Bonsai v0.8.x — inside Blender context
import bpy

obj = bpy.context.active_object

# Quick check via property (no tool import needed)
ifc_id = obj.BIMProperties.ifc_definition_id
if ifc_id == 0:
    print("Not linked to IFC")
else:
    print(f"Linked to IFC entity #{ifc_id}")

# Full check with entity retrieval
import bonsai.tool as tool
entity = tool.Ifc.get_entity(obj)
is_ifc_linked = entity is not None
```

---

## The Blender-IFC Data Bridge

### Mapping Architecture

```
Blender Side                          IFC Side
───────────────────────                ────────────────────────
bpy.types.Object                      ifcopenshell.entity_instance
  .name                    ←→          .Name
  .BIMProperties
    .ifc_definition_id     ────→       .id()  (step ID)
                           ←────       IfcStore.id_map[id]
  .matrix_world            ←→          IfcLocalPlacement
  .data (Mesh)             ←→          IfcShapeRepresentation

           tool.Ifc.get_entity(obj)  ──→
           tool.Ifc.get_object(ent)  ←──
           tool.Ifc.link(ent, obj)   ←→  bidirectional registration
```

### Key Properties on Blender Objects

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `obj.BIMProperties.ifc_definition_id` | `int` | Read | IFC step ID. `0` = not linked. |
| `obj.name` | `str` | Read/Write | Blender object name (may differ from IFC Name) |
| `obj.matrix_world` | `Matrix` | Read/Write | World-space transform (sync to IFC via `edit_object_placement`) |
| `obj.data` | `bpy.types.Mesh` | Read | Geometry data (regenerated from IFC representations) |

### Mapping Lookup Tables

| Lookup | Data Structure | Direction | Key → Value |
|--------|---------------|-----------|-------------|
| `IfcStore.id_map` | `dict` | IFC → Blender | `int` (step ID) → `bpy.types.Object` |
| `IfcStore.guid_map` | `dict` | IFC → Blender | `str` (GlobalId) → `bpy.types.Object` |
| `obj.BIMProperties.ifc_definition_id` | Property | Blender → IFC | `bpy.types.Object` → `int` (step ID) |

---

## IfcStore Element Access Methods

### Quick Reference

| Method | Input | Returns | Use When |
|--------|-------|---------|----------|
| `IfcStore.get_file()` | — | `ifcopenshell.file \| None` | Getting the live IFC model |
| `IfcStore.get_element(ifc_id)` | `int` | `bpy.types.Object \| None` | Looking up Blender object by step ID |
| `IfcStore.id_map` | — | `dict[int, Object]` | Iterating all linked objects |
| `IfcStore.guid_map` | — | `dict[str, Object]` | Looking up by GlobalId |
| `tool.Ifc.get()` | — | `ifcopenshell.file \| None` | Same as `get_file()` (tool layer) |
| `tool.Ifc.get_entity(obj)` | `Object` | `entity_instance \| None` | Object → IFC entity |
| `tool.Ifc.get_object(element)` | `entity_instance` | `Object \| None` | IFC entity → Object |
| `tool.Ifc.link(element, obj)` | Both | `None` | Register bidirectional mapping |

---

## ifcopenshell.file Query Methods

These methods are called on the `ifcopenshell.file` instance returned by `tool.Ifc.get()` or `IfcStore.get_file()`.

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `by_type(type)` | `(type: str, include_subtypes=True) -> list` | List of entities | All entities of a given IFC class |
| `by_id(id)` | `(id: int) -> entity_instance` | Single entity | Entity by step ID. Raises `RuntimeError` if not found. |
| `by_guid(guid)` | `(guid: str) -> entity_instance` | Single entity | Entity by GlobalId. Raises `RuntimeError` if not found. |

### ALWAYS Wrap Lookups in Try/Except When ID Source is External

```python
model = tool.Ifc.get()
try:
    entity = model.by_id(user_provided_id)
except RuntimeError:
    print(f"No entity with step ID {user_provided_id}")
```

---

## ifcopenshell.util.element — Relationship Navigation

| Function | Signature | Returns | Description |
|----------|-----------|---------|-------------|
| `get_container(element)` | `(element) -> entity \| None` | Spatial container | IfcBuildingStorey, IfcSpace, etc. |
| `get_type(element)` | `(element) -> entity \| None` | Type definition | IfcWallType, IfcSlabType, etc. |
| `get_types(type_element)` | `(type_element) -> list` | Occurrences | All instances of a type |
| `get_psets(element)` | `(element, psets_only=False, qtos_only=False, should_inherit=True) -> dict` | Property dict | All psets/qtos as nested dict |
| `get_material(element)` | `(element) -> entity \| None` | Material | IfcMaterial, IfcMaterialLayerSet, etc. |
| `get_decomposition(element)` | `(element) -> list` | Children | All decomposed child elements |
| `get_aggregate(element)` | `(element) -> entity \| None` | Parent | Aggregating parent element |

---

## Common Operations

### Select All Elements of a Type in Viewport

```python
# Bonsai v0.8.x — select all IfcDoor objects in Blender viewport
import bpy
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

bpy.ops.object.select_all(action='DESELECT')
for door in model.by_type("IfcDoor"):
    obj = tool.Ifc.get_object(door)
    if obj:
        obj.select_set(True)
```

### Get Property Value from Active Object

```python
# Bonsai v0.8.x — read FireRating from active object
import bpy
import bonsai.tool as tool
import ifcopenshell.util.element

obj = bpy.context.active_object
entity = tool.Ifc.get_entity(obj)
if entity:
    psets = ifcopenshell.util.element.get_psets(entity)
    wall_common = psets.get("Pset_WallCommon", {})
    fire_rating = wall_common.get("FireRating")
    print(f"FireRating: {fire_rating}")
```

### List All Elements in a Storey

```python
# Bonsai v0.8.x — list all elements contained in first storey
import bonsai.tool as tool
import ifcopenshell.util.element

model = tool.Ifc.get()
storey = model.by_type("IfcBuildingStorey")[0]
elements = ifcopenshell.util.element.get_decomposition(storey)
for el in elements:
    print(f"  {el.is_a()}: {el.Name} (#{el.id()})")
```

---

## Version Notes

### Bonsai v0.8.0+ (2024 Rename)

| Change | Before (BlenderBIM) | After (Bonsai v0.8.0+) |
|--------|---------------------|------------------------|
| Module path | `blenderbim.*` | `bonsai.*` |
| IfcStore import | `from blenderbim.bim.ifc import IfcStore` | `from bonsai.bim.ifc import IfcStore` |
| Tool import | `import blenderbim.tool as tool` | `import bonsai.tool as tool` |
| Core import | `import blenderbim.core.spatial` | `import bonsai.core.spatial` |

### IfcOpenShell v0.8+ List Parameters

Since IfcOpenShell v0.8, most relationship API functions use **list** parameters (`products`, `related_objects`) instead of singular. This affects `tool.Ifc.run()` calls:

```python
# CORRECT (v0.8+)
tool.Ifc.run("spatial.assign_container",
    products=[wall], relating_structure=storey)

# WRONG (pre-v0.8 style)
tool.Ifc.run("spatial.assign_container",
    product=wall, relating_structure=storey)  # KeyError
```

---

## Reference Links

- [Element Access Methods](references/methods.md) — Complete API signatures for element access
- [Working Code Examples](references/examples.md) — End-to-end element access examples
- [Anti-Patterns](references/anti-patterns.md) — Common element access mistakes and how to avoid them

## Dependencies

- **bonsai-core-architecture** — Three-layer architecture, IfcStore overview, operator pattern
- **ifcos-syntax-api** — `ifcopenshell.api.run()` invocation patterns, module table

## Sources

- Bonsai source: `src/bonsai/bonsai/bim/ifc.py` (IfcStore)
- Bonsai source: `src/bonsai/bonsai/tool/ifc.py` (tool.Ifc implementation)
- Bonsai source: `src/bonsai/bonsai/core/tool.py` (@interface definitions)
- IfcOpenShell API: https://docs.ifcopenshell.org
- Bonsai Documentation: https://docs.bonsaibim.org
