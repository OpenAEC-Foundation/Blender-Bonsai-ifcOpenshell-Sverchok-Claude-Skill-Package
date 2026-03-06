# Bonsai Spatial Structure — API Method Signatures

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+
> **Source**: `core/spatial.py`, `tool/spatial.py`, `bim/module/spatial/operator.py`

---

## 1. IfcOpenShell API Functions (Spatial Operations)

These are the `ifcopenshell.api` functions used for spatial structure management. Called via `ifcopenshell.api.run()` or direct module calls.

### aggregate.assign_object

```python
ifcopenshell.api.run("aggregate.assign_object", model,
    products: list[ifcopenshell.entity_instance],  # Child elements to aggregate
    relating_object: ifcopenshell.entity_instance,  # Parent element
) -> ifcopenshell.entity_instance  # Returns IfcRelAggregates
```

Connects spatial elements in a parent-child hierarchy via `IfcRelAggregates`. Use for: Project→Site, Site→Building, Building→Storey, Storey→Space.

### aggregate.unassign_object

```python
ifcopenshell.api.run("aggregate.unassign_object", model,
    products: list[ifcopenshell.entity_instance],  # Elements to detach
) -> None
```

Removes elements from their aggregate parent.

### spatial.assign_container

```python
ifcopenshell.api.run("spatial.assign_container", model,
    products: list[ifcopenshell.entity_instance],       # Physical elements to contain
    relating_structure: ifcopenshell.entity_instance,    # Spatial container (storey, space, etc.)
) -> ifcopenshell.entity_instance  # Returns IfcRelContainedInSpatialStructure
```

Places physical elements (walls, doors, columns) inside a spatial container. An element can only be contained in ONE container at a time. Reassigning moves the element.

### spatial.unassign_container

```python
ifcopenshell.api.run("spatial.unassign_container", model,
    products: list[ifcopenshell.entity_instance],  # Elements to remove from container
) -> None
```

Removes elements from their spatial container.

### spatial.reference_structure

```python
ifcopenshell.api.run("spatial.reference_structure", model,
    products: list[ifcopenshell.entity_instance],       # Elements to reference
    relating_structure: ifcopenshell.entity_instance,    # Spatial structure to reference from
) -> ifcopenshell.entity_instance  # Returns IfcRelReferencedInSpatialStructure
```

Creates a non-exclusive spatial reference. Use for multi-storey elements (columns, curtain walls) that span multiple storeys.

### spatial.dereference_structure

```python
ifcopenshell.api.run("spatial.dereference_structure", model,
    products: list[ifcopenshell.entity_instance],       # Elements to dereference
    relating_structure: ifcopenshell.entity_instance,    # Structure to remove reference from
) -> None
```

Removes a spatial reference relationship.

### root.create_entity (for spatial elements)

```python
ifcopenshell.api.run("root.create_entity", model,
    ifc_class: str,    # "IfcSite", "IfcBuilding", "IfcBuildingStorey", "IfcSpace"
    name: str = None,  # Element name
    predefined_type: str = None,  # Optional predefined type
) -> ifcopenshell.entity_instance
```

Creates a new spatial element with auto-generated GlobalId and ownership.

### root.remove_product

```python
ifcopenshell.api.run("root.remove_product", model,
    product: ifcopenshell.entity_instance,  # Element to remove
) -> None
```

Safely removes an element and cleans up ALL relationships. ALWAYS use this instead of `model.remove()`.

---

## 2. IfcOpenShell Utility Functions

### ifcopenshell.util.element.get_container

```python
ifcopenshell.util.element.get_container(
    element: ifcopenshell.entity_instance,
) -> Union[ifcopenshell.entity_instance, None]
```

Returns the spatial container of an element (via `IfcRelContainedInSpatialStructure`), or `None` if uncontained.

### ifcopenshell.util.element.get_decomposition

```python
ifcopenshell.util.element.get_decomposition(
    element: ifcopenshell.entity_instance,
) -> list[ifcopenshell.entity_instance]
```

Returns all elements in the decomposition tree below the given element (recursive).

### ifcopenshell.util.element.get_parts

```python
ifcopenshell.util.element.get_parts(
    element: ifcopenshell.entity_instance,
) -> list[ifcopenshell.entity_instance]
```

Returns direct children via `IfcRelAggregates` (non-recursive, one level only).

---

## 3. Bonsai Core Functions (`core/spatial.py`)

These functions implement the business logic layer. Called by operators with dependency-injected tool classes.

### core.spatial.assign_container

```python
def assign_container(
    ifc: type[tool.Ifc],
    collector: type[tool.Collector],
    spatial: type[tool.Spatial],
    container: ifcopenshell.entity_instance,  # Target spatial container
    objs: Optional[list[bpy.types.Object]] = None,  # Blender objects to assign
) -> Union[ifcopenshell.entity_instance, None]
```

Assigns Blender objects to a spatial container. Handles aggregate hierarchies by finding root elements first. Validates via `spatial.can_contain()`.

### core.spatial.remove_container

```python
def remove_container(
    ifc: type[tool.Ifc],
    collector: type[tool.Collector],
    obj: bpy.types.Object,
) -> None
```

Removes an element from its spatial container via `spatial.unassign_container`.

### core.spatial.reference_structure

```python
def reference_structure(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
    structure: Optional[ifcopenshell.entity_instance] = None,
    element: Optional[ifcopenshell.entity_instance] = None,
) -> Union[ifcopenshell.entity_instance, None]
```

Creates an `IfcRelReferencedInSpatialStructure` relationship. Validates via `spatial.can_reference()`.

### core.spatial.dereference_structure

```python
def dereference_structure(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
    structure: Optional[ifcopenshell.entity_instance] = None,
    element: Optional[ifcopenshell.entity_instance] = None,
) -> None
```

Removes an `IfcRelReferencedInSpatialStructure` relationship.

### core.spatial.copy_to_container

```python
def copy_to_container(
    ifc: type[tool.Ifc],
    collector: type[tool.Collector],
    spatial: type[tool.Spatial],
    obj: bpy.types.Object,
    containers: list[ifcopenshell.entity_instance],
) -> list[ifcopenshell.entity_instance]
```

Copies an element to one or more destination containers. Maintains relative positioning.

### core.spatial.delete_container

```python
def delete_container(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
    geometry: type[tool.Geometry],
    container: ifcopenshell.entity_instance,
) -> None
```

Deletes a spatial container and its Blender object. IfcProject cannot be deleted (enforced at operator level).

### core.spatial.select_container

```python
def select_container(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
    container: ifcopenshell.entity_instance,
    selection_mode: str = "ADD",  # "ADD", "REMOVE", "SINGLE"
) -> None
```

Selects the Blender object corresponding to a spatial container.

### core.spatial.select_similar_container

```python
def select_similar_container(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
    obj: bpy.types.Object,
    is_recursive: bool = True,
) -> None
```

Selects all elements in the same container as the given object. `is_recursive=True` includes sub-containers.

### core.spatial.generate_space

```python
def generate_space(
    ifc: type[tool.Ifc],
    model: type[tool.Model],
    root: type[tool.Root],
    spatial: type[tool.Spatial],
    type: type[tool.Type],
) -> Union[None, str]
```

Generates an `IfcSpace` from surrounding wall/column geometry. Requires a default container to be set. Raises `SpaceGenerationError` on failure.

### core.spatial.generate_spaces_from_walls

```python
def generate_spaces_from_walls(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
    collector: type[tool.Collector],
) -> None
```

Generates multiple `IfcSpace` elements from selected wall objects by computing wall union and creating spaces from interior holes.

### core.spatial.set_default_container

```python
def set_default_container(
    spatial: type[tool.Spatial],
    container: ifcopenshell.entity_instance,
) -> None
```

Sets the default container for new element creation.

### core.spatial.import_spatial_decomposition

```python
def import_spatial_decomposition(
    spatial: type[tool.Spatial],
) -> None
```

Refreshes the spatial decomposition tree in the UI.

### core.spatial.toggle_space_visibility

```python
def toggle_space_visibility(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
) -> None
```

Toggles IfcSpace objects between WIRE and TEXTURED display modes.

### core.spatial.toggle_hide_spaces

```python
def toggle_hide_spaces(
    ifc: type[tool.Ifc],
    spatial: type[tool.Spatial],
) -> None
```

Toggles visibility (show/hide) of all IfcSpace objects.

### core.spatial.enable_editing_container / disable_editing_container

```python
def enable_editing_container(spatial: type[tool.Spatial], obj: bpy.types.Object) -> None
def disable_editing_container(spatial: type[tool.Spatial], obj: bpy.types.Object) -> None
```

Enter/exit container editing mode for an object.

### core.spatial.contract_container / expand_container

```python
def contract_container(
    spatial: type[tool.Spatial],
    container: ifcopenshell.entity_instance,
    is_recursive: bool,
) -> None

def expand_container(
    spatial: type[tool.Spatial],
    container: ifcopenshell.entity_instance,
    is_recursive: bool,
) -> None
```

Collapse/expand a container in the spatial decomposition UI tree.

---

## 4. Bonsai Tool Methods (`tool/spatial.py`)

The `Spatial` tool class implements the `bonsai.core.tool.Spatial` interface. All methods are `@classmethod`.

### Query Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `Spatial.get_container(element)` | `entity_instance \| None` | Spatial container of element |
| `Spatial.get_decomposition(element)` | `list[entity_instance]` | All decomposed elements (recursive) |
| `Spatial.get_decomposed_elements(container, is_recursive=True)` | `list[entity_instance]` | Elements inside a container |
| `Spatial.get_root_element(element)` | `entity_instance` | Topmost parent (walks up aggregates/nests/voids) |
| `Spatial.get_active_container()` | `entity_instance \| None` | Currently active container from UI |
| `Spatial.get_selected_containers()` | `list[entity_instance]` | IFC spatial elements from selection |
| `Spatial.get_selected_objects_without_containers()` | `list[bpy.types.Object]` | Selected objects that are NOT spatial elements |
| `Spatial.guess_default_container()` | `entity_instance \| None` | First IfcBuildingStorey via Project>Site>Building>Storey |

### Validation Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `Spatial.can_contain(container, element)` | `bool` | Validates containment compatibility |
| `Spatial.can_reference(structure, element)` | `bool` | Validates reference compatibility |
| `Spatial.is_bounding_class(visible_element)` | `bool` | True for IfcWall, IfcColumn, IfcMember, IfcVirtualElement, IfcPlate |

### Container Management Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `Spatial.set_default_container(container)` | `None` | Sets default container + updates active layer collection |
| `Spatial.set_target_container_as_default()` | `None` | Sets editing target container as default |
| `Spatial.edit_container_name(container, name)` | `None` | Edits Name attribute via API |

### Space Generation Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `Spatial.get_space_polygon_from_context_visible_objects(x, y)` | `Polygon \| str` | Space polygon enclosing point (x,y) |
| `Spatial.get_boundary_lines_from_context_visible_objects()` | `list[LineString]` | Boundary lines from visible walls/columns |
| `Spatial.get_union_shape_from_selected_objects()` | `Polygon` | Union of selected wall/column footprints |
| `Spatial.get_boundary_elements(selected_objects)` | `list` | Filters to IfcWall and IfcColumn entities |
| `Spatial.get_polygons(boundary_elements)` | `list[Polygon]` | 2D footprint polygons from elements |
| `Spatial.get_bmesh_from_polygon(poly, h)` | `bmesh.types.BMesh` | Extruded bmesh from shapely polygon |
| `Spatial.get_x_y_z_h_mat_from_obj(obj)` | `tuple[float, float, float, float, Matrix]` | Center XY, bottom Z, height, matrix from object |
| `Spatial.get_x_y_z_h_mat_from_cursor()` | `tuple[float, float, float, float, Matrix]` | Cursor XY, container Z, default height 3m, identity matrix |

### UI Tree Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `Spatial.import_spatial_decomposition()` | `None` | Rebuilds full spatial tree from IfcProject |
| `Spatial.import_spatial_element(element, level_index)` | `None` | Recursively adds element and children to UI tree |
| `Spatial.load_contained_elements()` | `None` | Loads contained elements (by type/decomposition/classification) |
| `Spatial.contract_container(container, is_recursive)` | `None` | Collapses container in UI tree |
| `Spatial.expand_container(container, is_recursive)` | `None` | Expands container in UI tree |
| `Spatial.toggle_container_element(element_index, is_recursive)` | `None` | Toggles element expansion in UI |

### Selection/Visibility Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `Spatial.select_products(products, unhide=False)` | `None` | Selects Blender objects for IFC products |
| `Spatial.filter_products(products, action)` | `None` | select/isolate/unhide/hide products |
| `Spatial.deselect_objects()` | `None` | Deselects all selected objects |
| `Spatial.set_space_visibility(is_visible)` | `None` | Shows/hides all spatial elements |
| `Spatial.toggle_spaces_visibility_wired_and_textured(spaces)` | `None` | Toggles WIRE/TEXTURED display |
| `Spatial.toggle_hide_spaces(spaces)` | `None` | Toggles visibility of IfcSpace objects |

---

## 5. Bonsai Operators (`bim/module/spatial/operator.py`)

All operators use `bl_idname` prefix `bim.` and are called via `bpy.ops.bim.*`.

| Operator | bl_idname | Properties | Purpose |
|----------|-----------|------------|---------|
| `AssignContainer` | `bim.assign_container` | `container: IntProperty` | Assign selected objects to container |
| `RemoveContainer` | `bim.remove_container` | — | Remove container assignment |
| `CopyToContainer` | `bim.copy_to_container` | `container: IntProperty` | Copy objects to container |
| `ReferenceStructure` | `bim.reference_structure` | — | Reference selected objects from structures |
| `DereferenceStructure` | `bim.dereference_structure` | — | Remove spatial references |
| `ReferenceFromProvidedStructure` | `bim.reference_from_provided_structure` | `structure: IntProperty`, `dereference: BoolProperty` | Reference/dereference from specific structure |
| `SelectContainer` | `bim.select_container` | `container: IntProperty`, `selection_mode: EnumProperty` | Select container object |
| `SelectSimilarContainer` | `bim.select_similar_container` | `is_recursive: BoolProperty(True)` | Select all objects in same container |
| `SelectProduct` | `bim.select_product` | `product: IntProperty` | Select a single product |
| `SelectDecomposedElement` | `bim.select_decomposed_element` | `element: IntProperty` | Select decomposed element |
| `SelectDecomposedElements` | `bim.select_decomposed_elements` | `should_filter: BoolProperty`, `container: IntProperty`, `is_recursive: BoolProperty` | Select filtered elements |
| `SetDefaultContainer` | `bim.set_default_container` | `container: IntProperty` | Set default container (NOT IfcProject) |
| `DeleteContainer` | `bim.delete_container` | `container: IntProperty` | Delete spatial container (NOT IfcProject) |
| `ImportSpatialDecomposition` | `bim.import_spatial_decomposition` | — | Load spatial tree in UI |
| `ContractContainer` | `bim.contract_container` | `container: IntProperty`, `is_recursive: BoolProperty` | Collapse container in UI |
| `ExpandContainer` | `bim.expand_container` | `container: IntProperty`, `is_recursive: BoolProperty` | Expand container in UI |
| `ToggleContainerElement` | `bim.toggle_container_element` | `element_index: IntProperty`, `is_recursive: BoolProperty` | Toggle element in contained list |
| `SetContainerVisibility` | `bim.set_container_visibility` | `container: IntProperty`, `should_include_children: BoolProperty`, `mode: StringProperty` | HIDE/SHOW/ISOLATE container |
| `SetElementVisibility` | `bim.set_element_visibility` | `container: IntProperty`, `should_filter: BoolProperty`, `mode: StringProperty` | HIDE/SHOW/ISOLATE elements |
| `ToggleGrids` | `bim.toggle_grids` | `is_visible: BoolProperty` | Toggle grid visibility |
| `ToggleSpatialElements` | `bim.toggle_spatial_elements` | `is_visible: BoolProperty` | Toggle spatial element visibility |
| `EnableEditingContainer` | `bim.enable_editing_container` | — | Enter container editing mode |
| `DisableEditingContainer` | `bim.disable_editing_container` | — | Exit container editing mode |
| `GenerateSpace` | `bim.generate_space` | — | Generate IfcSpace from walls |
| `GenerateSpacesFromWalls` | `bim.generate_spaces_from_walls` | — | Generate spaces from selected walls |
