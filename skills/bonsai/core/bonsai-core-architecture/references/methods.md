# Bonsai Core Architecture — Method Reference

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11

## Import Paths

```python
# Core IFC store
from bonsai.bim.ifc import IfcStore

# Tool classes (dependency injection targets)
import bonsai.tool as tool

# Core functions (abstract use-case logic)
import bonsai.core.spatial
import bonsai.core.geometry
import bonsai.core.type
import bonsai.core.pset

# IfcOpenShell (data layer)
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element
import ifcopenshell.util.placement
import ifcopenshell.util.representation
import ifcopenshell.util.shape_builder
```

---

## IfcStore Class (`bonsai.bim.ifc`)

### Static Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `IfcStore.path` | `str` | Absolute path to the `.ifc` file on disk |
| `IfcStore.file` | `ifcopenshell.file \| None` | Live in-memory IFC model instance |
| `IfcStore.schema` | `str` | Schema string: `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |
| `IfcStore.id_map` | `dict[int, bpy.types.Object]` | IFC step ID -> Blender object |
| `IfcStore.guid_map` | `dict[str, bpy.types.Object]` | GlobalId -> Blender object |
| `IfcStore.edited_objs` | `set` | Objects with pending IFC changes |

### Static Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_file()` | `() -> ifcopenshell.file \| None` | IFC file or None | Get the live IFC model |
| `get_schema()` | `() -> str` | Schema string | Get IFC schema version |
| `get_element()` | `(ifc_id: int) -> bpy.types.Object \| None` | Blender object | Get Blender object by IFC ID |
| `execute_ifc_operator()` | `(operator, context) -> set` | `{"FINISHED"}` or `{"CANCELLED"}` | Wrap operator execution with undo support |

---

## tool.Ifc — IFC Model Access

> Import: `import bonsai.tool as tool` then use `tool.Ifc`

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get()` | `() -> ifcopenshell.file \| None` | IFC file instance | Returns live ifcopenshell.file. Returns None if no file loaded. |
| `set(ifc)` | `(ifc: ifcopenshell.file) -> None` | None | Sets active IFC file and triggers post-load hooks |
| `run(command, **kwargs)` | `(command: str, **kwargs) -> Any` | Varies | Execute ifcopenshell.api command against active file |
| `get_entity(obj)` | `(obj: bpy.types.Object) -> entity_instance \| None` | IFC entity | Blender object -> IFC entity. None if not linked. |
| `get_object(element)` | `(element: entity_instance) -> bpy.types.Object \| None` | Blender object | IFC entity -> Blender object. None if no representation. |
| `link(element, obj)` | `(element: entity_instance, obj: bpy.types.Object) -> None` | None | Establish bidirectional IFC <-> Blender link |

### tool.Ifc.run() — Common Commands

| Command | Key Parameters | Description |
|---------|---------------|-------------|
| `"root.create_entity"` | `ifc_class: str, name: str` | Create new IFC entity |
| `"spatial.assign_container"` | `relating_structure, product` | Place element in spatial zone |
| `"aggregate.assign_object"` | `relating_object, products: list` | Spatial decomposition hierarchy |
| `"geometry.assign_representation"` | `product, representation` | Assign geometry to element |
| `"geometry.add_wall_representation"` | `context, length, height, thickness` | Create wall geometry |
| `"pset.add_pset"` | `product, name: str` | Create property set |
| `"pset.edit_pset"` | `pset, properties: dict` | Edit property values |
| `"type.assign_type"` | `relating_type, related_objects: list` | Assign IFC type |
| `"feature.add_feature"` | `feature, element` | Add opening void to element |
| `"material.assign_material"` | `products: list, material` | Assign material to elements |

---

## tool.Spatial — Spatial Container Hierarchy

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_container(element)` | `(element: entity_instance) -> entity_instance \| None` | Spatial container | Get the spatial container of an element |
| `get_decomposed_elements(container)` | `(container: entity_instance) -> list` | List of child elements | Get all elements in a container |
| `assign_container(element, structure)` | `(element, structure) -> None` | None | Place element in spatial zone |

---

## tool.Geometry — Mesh/Representation Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_active_representation(obj)` | `(obj: bpy.types.Object) -> entity_instance \| None` | Representation | Get active geometry representation |
| `edit_object_placement(obj)` | `(obj: bpy.types.Object) -> None` | None | Sync Blender transform to IFC placement |

---

## tool.Blender — Blender Scene Utilities

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_obj_ifc_definition_id(obj)` | `(obj: bpy.types.Object) -> int` | IFC step ID | Get the IFC ID of a Blender object (0 = not linked) |
| `set_active_object(obj)` | `(obj: bpy.types.Object) -> None` | None | Set active object in Blender scene |

---

## tool.Pset — Property Sets Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_element_pset(element, pset_name)` | `(element, pset_name: str) -> dict \| None` | Property dict | Get property set values for element |
| `get_pset_props(obj, pset_name)` | `(obj, pset_name: str) -> PropertyGroup` | Blender props | Get Blender property group for pset |

---

## tool.Type — IFC Type Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_type(element)` | `(element: entity_instance) -> entity_instance \| None` | Type element | Get the IfcTypeObject for an element |
| `assign_type(relating_type, related_objects)` | `(relating_type, related_objects: list) -> None` | None | Assign type to occurrences |

---

## tool.Material — Material/Layer Set Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_material(element)` | `(element: entity_instance) -> entity_instance \| None` | Material | Get assigned material/material set |
| `get_style(material)` | `(material: entity_instance) -> entity_instance \| None` | Surface style | Get visual style for material |

---

## tool.Context — Representation Context Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_context(model, type, id, target_view)` | `(model, type: str, id: str, target_view: str) -> entity_instance \| None` | Context | Get representation context |

---

## tool.Style — Visual Style Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_style(obj)` | `(obj: bpy.types.Object) -> entity_instance \| None` | Style | Get IFC style for object |
| `get_surface_style(obj)` | `(obj: bpy.types.Object) -> entity_instance \| None` | Surface style | Get surface rendering style |

---

## tool.Drawing — 2D Drawing/Documentation

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_active_drawing()` | `() -> entity_instance \| None` | Drawing | Get currently active drawing |
| `get_drawing_elements(drawing)` | `(drawing: entity_instance) -> list` | Elements | Get all elements in a drawing |

---

## Core Function Signatures

Core functions define abstract use-case logic. They receive `tool.*` classes via dependency injection.

### core.spatial

```python
def assign_container(ifc, spatial, structure_obj, element_obj):
    """Place an element in a spatial container.

    Args:
        ifc: tool.Ifc instance (injected)
        spatial: tool.Spatial instance (injected)
        structure_obj: Blender object representing spatial container
        element_obj: Blender object representing element to contain
    """
```

### core.geometry

```python
def edit_object_placement(ifc, geometry, obj):
    """Sync Blender object transform to IFC placement.

    Args:
        ifc: tool.Ifc instance
        geometry: tool.Geometry instance
        obj: Blender object with modified transform
    """
```

---

## IfcOpenShell Utility Functions

### ifcopenshell.util.element

| Function | Signature | Returns | Description |
|----------|-----------|---------|-------------|
| `get_psets(element)` | `(element, psets_only=False, qtos_only=False, should_inherit=True) -> dict` | Pset dict | Get all property/quantity sets |
| `get_type(element)` | `(element: entity_instance) -> entity_instance \| None` | Type | Get element's type definition |
| `get_types(type_element)` | `(type_element: entity_instance) -> list` | Occurrences | Get all instances of a type |

### ifcopenshell.util.representation

| Function | Signature | Returns | Description |
|----------|-----------|---------|-------------|
| `get_context(ifc_file, context_type, context_identifier, target_view)` | `(ifc_file, type: str, id: str, view: str) -> entity_instance \| None` | Context | Find existing representation context |

### ifcopenshell.util.shape_builder

| Class/Method | Signature | Returns | Description |
|-------------|-----------|---------|-------------|
| `ShapeBuilder(ifc_file)` | Constructor | Builder instance | Create parametric geometry builder |
| `builder.extrude(profile, magnitude, extrusion_vector)` | `(profile, magnitude: float, vector) -> entity_instance` | Extrusion | Create extruded solid |
| `builder.get_representation(context, items)` | `(context, items: list) -> entity_instance` | Representation | Wrap items in shape representation |

---

## BIM Object Properties

### BIMObjectProperties (on every Blender object)

| Property | Type | Description |
|----------|------|-------------|
| `obj.BIMProperties.ifc_definition_id` | `int` | IFC entity step ID. `0` = not linked to IFC. |

### Checking IFC Link Status

```python
obj = bpy.context.active_object
ifc_id = obj.BIMProperties.ifc_definition_id
if ifc_id == 0:
    print("Object is not linked to IFC")
else:
    entity = IfcStore.get_file().by_id(ifc_id)
    print(f"IFC class: {entity.is_a()}, Name: {entity.Name}")
```

---

## Operator Namespace: `bpy.ops.bim.*`

### Key Operators

| Operator | Description | Key Parameters |
|----------|-------------|---------------|
| `bpy.ops.bim.assign_class()` | Assign IFC class to Blender object | `ifc_class`, `predefined_type` |
| `bpy.ops.bim.edit_object_placement()` | Sync Blender transform to IFC | None |
| `bpy.ops.bim.save_project()` | Save IFC file to disk | `filepath` |
| `bpy.ops.bim.load_project()` | Load IFC file into Bonsai | `filepath` |
| `bpy.ops.bim.create_project()` | Create new IFC project | None |

### Operator Polling Rule

ALWAYS check poll() before calling any `bpy.ops.bim.*` operator:

```python
if bpy.ops.bim.assign_class.poll():
    bpy.ops.bim.assign_class(ifc_class="IfcWall")
else:
    print("Cannot assign class in current context")
```
