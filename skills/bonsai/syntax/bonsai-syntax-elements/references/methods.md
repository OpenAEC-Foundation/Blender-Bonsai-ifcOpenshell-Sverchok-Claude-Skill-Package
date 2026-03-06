# Bonsai Element Access — Method Reference

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11

## Import Paths

```python
# Core IFC store (element retrieval, mapping tables)
from bonsai.bim.ifc import IfcStore

# Tool classes (high-level element access)
import bonsai.tool as tool

# IfcOpenShell utilities (relationship navigation)
import ifcopenshell.util.element
import ifcopenshell.util.selector
import ifcopenshell.util.representation
```

---

## IfcStore — Central State Manager (`bonsai.bim.ifc`)

### Static Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `IfcStore.path` | `str` | Absolute path to the `.ifc` file on disk |
| `IfcStore.file` | `ifcopenshell.file \| None` | Live in-memory IFC model instance |
| `IfcStore.schema` | `str` | Schema string: `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |
| `IfcStore.id_map` | `dict[int, bpy.types.Object]` | IFC step ID → Blender object |
| `IfcStore.guid_map` | `dict[str, bpy.types.Object]` | GlobalId → Blender object |
| `IfcStore.edited_objs` | `set` | Blender objects with pending IFC changes |

### Static Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_file()` | `() -> ifcopenshell.file \| None` | IFC file or `None` | Returns the live IFC model. Returns `None` if no project is loaded. |
| `get_schema()` | `() -> str` | Schema string | Returns IFC schema version string. |
| `get_element()` | `(ifc_id: int) -> bpy.types.Object \| None` | Blender object or `None` | Looks up a Blender object by IFC step ID. Equivalent to `IfcStore.id_map.get(ifc_id)`. |
| `execute_ifc_operator()` | `(operator, context) -> set[str]` | `{"FINISHED"}` or `{"CANCELLED"}` | Wraps IFC operator execution with undo/redo support. |

---

## tool.Ifc — High-Level IFC Access

> Import: `import bonsai.tool as tool` → use `tool.Ifc`
> Implementation: `src/bonsai/bonsai/tool/ifc.py`
> Interface: `src/bonsai/bonsai/core/tool.py` (`@interface class Ifc`)

### Element Access Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get()` | `() -> ifcopenshell.file \| None` | IFC file instance | Returns the live `ifcopenshell.file`. Returns `None` if no file is loaded. Equivalent to `IfcStore.get_file()`. |
| `set(ifc)` | `(ifc: ifcopenshell.file) -> None` | `None` | Sets the active IFC file and triggers post-load hooks (rebuilds id_map, guid_map, scene). |
| `get_entity(obj)` | `(obj: bpy.types.Object) -> ifcopenshell.entity_instance \| None` | IFC entity or `None` | Maps a Blender object to its linked IFC entity. Returns `None` if the object is not linked to IFC (ifc_definition_id == 0). |
| `get_object(element)` | `(element: ifcopenshell.entity_instance) -> bpy.types.Object \| None` | Blender object or `None` | Maps an IFC entity to its linked Blender object. Returns `None` if the entity has no scene representation. |
| `link(element, obj)` | `(element: ifcopenshell.entity_instance, obj: bpy.types.Object) -> None` | `None` | Establishes bidirectional link: sets `obj.BIMProperties.ifc_definition_id`, populates `IfcStore.id_map` and `IfcStore.guid_map`. |
| `unlink(element, obj)` | `(element: ifcopenshell.entity_instance, obj: bpy.types.Object) -> None` | `None` | Removes bidirectional link. Clears `ifc_definition_id` and removes from id_map/guid_map. |
| `run(command, **kwargs)` | `(command: str, **kwargs) -> Any` | Varies | Executes `ifcopenshell.api.run(command, model, **kwargs)` against the active IFC file. |
| `get_schema()` | `() -> str` | Schema string | Returns the schema version of the active IFC file. |

### tool.Ifc.get_entity() — Internal Behavior

```python
# Simplified internal logic (Bonsai v0.8.x)
def get_entity(self, obj):
    ifc_file = IfcStore.get_file()
    if ifc_file is None:
        return None
    ifc_id = obj.BIMProperties.ifc_definition_id
    if ifc_id == 0:
        return None
    try:
        return ifc_file.by_id(ifc_id)
    except RuntimeError:
        return None
```

### tool.Ifc.get_object() — Internal Behavior

```python
# Simplified internal logic (Bonsai v0.8.x)
def get_object(self, element):
    return IfcStore.id_map.get(element.id())
```

---

## ifcopenshell.file — Query Methods

These methods are available on the `ifcopenshell.file` instance returned by `tool.Ifc.get()`.

| Method | Signature | Returns | Raises | Description |
|--------|-----------|---------|--------|-------------|
| `by_type(type)` | `(type: str, include_subtypes: bool = True) -> list[entity_instance]` | List of entities | — | Returns all entities of a given IFC class. Empty list if none found. |
| `by_id(id)` | `(id: int) -> entity_instance` | Single entity | `RuntimeError` if not found | Returns entity by step ID. |
| `by_guid(guid)` | `(guid: str) -> entity_instance` | Single entity | `RuntimeError` if not found | Returns entity by GlobalId. |

### Entity Instance Attributes

Every `ifcopenshell.entity_instance` exposes:

| Attribute/Method | Type | Description |
|-----------------|------|-------------|
| `.id()` | `int` | IFC step ID (unique within file) |
| `.is_a()` | `str` | IFC class name (e.g., `"IfcWall"`) |
| `.is_a(type)` | `bool` | Check if entity is of a given type (includes inheritance) |
| `.GlobalId` | `str` | 22-character IFC GlobalId (on rooted entities) |
| `.Name` | `str \| None` | Entity name (on named entities) |
| `.Description` | `str \| None` | Entity description |
| `.get_info()` | `dict` | All attributes as a dictionary |

---

## ifcopenshell.util.element — Relationship Navigation

> Import: `import ifcopenshell.util.element`

| Function | Signature | Returns | Description |
|----------|-----------|---------|-------------|
| `get_container(element)` | `(element: entity_instance) -> entity_instance \| None` | Spatial container | Returns the `IfcBuildingStorey`, `IfcSpace`, etc. that contains this element via `IfcRelContainedInSpatialStructure`. |
| `get_type(element)` | `(element: entity_instance) -> entity_instance \| None` | Type entity | Returns the `IfcTypeObject` (e.g., `IfcWallType`) via `IfcRelDefinesByType`. |
| `get_types(type_element)` | `(type_element: entity_instance) -> list[entity_instance]` | Occurrences | Returns all element instances that use this type definition. |
| `get_psets(element)` | `(element: entity_instance, psets_only: bool = False, qtos_only: bool = False, should_inherit: bool = True) -> dict` | Nested dict | Returns all psets/qtos as `{pset_name: {prop_name: value}}`. With `should_inherit=True`, includes type-level psets. |
| `get_material(element)` | `(element: entity_instance) -> entity_instance \| None` | Material entity | Returns `IfcMaterial`, `IfcMaterialLayerSet`, `IfcMaterialProfileSet`, or `IfcMaterialConstituentSet`. |
| `get_decomposition(element)` | `(element: entity_instance) -> list[entity_instance]` | Children | Returns all elements decomposed from this element (via `IfcRelAggregates` and `IfcRelContainedInSpatialStructure`). |
| `get_aggregate(element)` | `(element: entity_instance) -> entity_instance \| None` | Parent | Returns the aggregating parent element (via `IfcRelAggregates` inverse). |
| `get_grouped_by(element)` | `(element: entity_instance) -> list[entity_instance]` | Groups | Returns all `IfcGroup` entities that this element belongs to. |

---

## ifcopenshell.util.selector — CSS-Like Queries

> Import: `import ifcopenshell.util.selector`

| Function | Signature | Returns | Description |
|----------|-----------|---------|-------------|
| `filter_elements(ifc_file, query)` | `(ifc_file: ifcopenshell.file, query: str) -> set[entity_instance]` | Set of entities | Filter elements using CSS-like query syntax. |

### Query Syntax Examples

```python
import ifcopenshell.util.selector

model = tool.Ifc.get()

# All IfcWall entities
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")

# Walls with specific property
external_walls = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, Pset_WallCommon.IsExternal=True')

# Elements by attribute
named = ifcopenshell.util.selector.filter_elements(
    model, 'IfcElement, Name="Wall-001"')
```

---

## BIMObjectProperties — Blender-Side IFC Link

> Registered on every `bpy.types.Object` by Bonsai

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `obj.BIMProperties.ifc_definition_id` | `int` | Read | IFC entity step ID. `0` means object is NOT linked to IFC. |

### Usage

```python
import bpy

obj = bpy.context.active_object

# Check if IFC-linked
if obj.BIMProperties.ifc_definition_id == 0:
    print("Not an IFC object")
else:
    ifc_id = obj.BIMProperties.ifc_definition_id
    entity = IfcStore.get_file().by_id(ifc_id)
    print(f"IFC class: {entity.is_a()}")
```

**NEVER** set `ifc_definition_id` directly. ALWAYS use `tool.Ifc.link()` to establish the mapping.

---

## tool.Blender — Blender Scene Utilities

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_obj_ifc_definition_id(obj)` | `(obj: bpy.types.Object) -> int` | IFC step ID | Returns `obj.BIMProperties.ifc_definition_id`. Returns `0` if not linked. |
| `set_active_object(obj)` | `(obj: bpy.types.Object) -> None` | `None` | Sets the active object in the Blender scene. |

---

## tool.Spatial — Spatial Container Access

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_container(element)` | `(element: entity_instance) -> entity_instance \| None` | Spatial container | Get the IfcBuildingStorey/IfcSpace containing this element. |
| `get_decomposed_elements(container)` | `(container: entity_instance) -> list[entity_instance]` | Elements | Get all elements contained in a spatial structure. |

---

## tool.Pset — Property Set Access

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_element_pset(element, pset_name)` | `(element: entity_instance, pset_name: str) -> dict \| None` | Property dict | Get a single pset as `{prop_name: value}`. |
| `get_pset_props(obj, pset_name)` | `(obj: bpy.types.Object, pset_name: str) -> PropertyGroup` | Blender PropertyGroup | Get Blender-side cached pset properties. |

---

## tool.Type — Type Definition Access

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_type(element)` | `(element: entity_instance) -> entity_instance \| None` | Type entity | Get the IfcTypeObject for an element occurrence. |
