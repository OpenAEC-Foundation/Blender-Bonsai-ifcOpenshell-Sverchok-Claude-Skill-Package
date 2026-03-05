# IfcOpenShell API Categories - Complete Reference

**Source:** docs.ifcopenshell.org (v0.8.4), bonsaibim.org docs-python
**Date researched:** 2026-03-05
**Scope:** All `ifcopenshell.api.run()` categories and their functions

---

## Overview: ifcopenshell.api.run()

`ifcopenshell.api.run()` is the primary entry point for all API operations in IfcOpenShell. It accepts a dot-notation string identifying the module and function, followed by keyword arguments.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()

# Pattern: ifcopenshell.api.run("module.function", model, **kwargs)
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject")

# Equivalent direct call:
project = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")
```

---

## Complete List of API Modules

The following modules are available under `ifcopenshell.api`:

| Module | Purpose |
|--------|---------|
| aggregate | Hierarchical decomposition of spatial/physical elements |
| alignment | Alignment and curve management |
| attribute | Entity attribute editing |
| boundary | Boundary condition management |
| classification | Classification reference handling |
| cogo | Coordinate geometry operations |
| constraint | Constraint and metric management |
| context | Geometric representation context definition |
| control | Control relationship assignment |
| cost | Cost scheduling and budgeting |
| document | Document reference management |
| drawing | Drawing and annotation tools |
| feature | Feature and opening creation |
| geometry | Geometric representation and placement |
| georeference | Geographic coordinate systems |
| grid | Grid axis definition |
| group | Element grouping operations |
| layer | Layer management and assignment |
| library | Library reference handling |
| material | Material and layer composition |
| nest | Nesting hierarchy management |
| owner | Ownership and actor information |
| profile | Profile definition and management |
| project | Project file creation |
| pset | Property and quantity set handling |
| pset_template | Property set template definition |
| resource | Resource and labor management |
| root | Entity creation and class manipulation |
| sequence | Task scheduling and workflows |
| spatial | Spatial containment relationships |
| structural | Structural analysis and loads |
| style | Presentation styling |
| system | MEP system and port connectivity |
| type | Type definition and mapping |
| unit | Unit system definition |
| void | Void and opening management |
| nest | Nesting relationships |

---

## Detailed Module Documentation

---

### 1. root

**Description:** The foundational module for creating, copying, reassigning, and removing physical and spatial elements. One of the most frequently used API modules. Handles any "rooted" IFC product (walls, doors, slabs, spaces, types, etc.).

**Key Functions:**

#### `create_entity()`
```python
create_entity(
    file: ifcopenshell.file,
    ifc_class: str = 'IfcBuildingElementProxy',
    predefined_type: str | None = None,
    name: str | None = None
) → ifcopenshell.entity_instance
```
Creates a new rooted IFC product. Automatically generates a valid GlobalId and stores ownership history. Validates predefined types (built-in and user-defined).

**Parameters:**
- `ifc_class`: Any rooted IFC class name (e.g., "IfcWall", "IfcProject", "IfcWallType")
- `predefined_type`: Built-in or user-defined type for the class
- `name`: Human-readable name for the new element

**Returns:** Newly created entity instance

#### `copy_class()`
```python
copy_class(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```
Duplicates a product preserving: placement, property sets, nested ports, aggregation, containment, type associations, voids, materials, and group memberships. Does NOT copy representations (must be handled manually).

#### `reassign_class()`
```python
reassign_class(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    ifc_class: str = 'IfcBuildingElementProxy',
    predefined_type: str | None = None,
    occurrence_class: str | None = None
) → ifcopenshell.entity_instance
```
Changes the IFC class of an existing product while retaining geometry and relationships. In IFC4+ automatically reassigns occurrence classes when a type is reassigned.

#### `remove_product()`
```python
remove_product(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance
) → None
```
Smart delete: removes a product and all its relationships (geometry, placement, properties, materials, containment, aggregation, nesting). Applicable to IfcAnnotation, IfcElement, IfcElementType, IfcSpatialElement, IfcSpatialElementType.

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()

# Create basic spatial hierarchy
project = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")
site    = ifcopenshell.api.root.create_entity(model, ifc_class="IfcSite")
building = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuilding")
storey  = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuildingStorey")

# Create physical elements
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
wall_type = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWallType", name="WT01")

# Copy and reassign
wall_copy = ifcopenshell.api.root.copy_class(model, product=wall)
slab = ifcopenshell.api.root.reassign_class(model, product=wall_copy, ifc_class="IfcSlab")

# Remove
ifcopenshell.api.root.remove_product(model, product=slab)
```

---

### 2. spatial

**Description:** Manages spatial containment and referencing relationships. Physical elements (walls, doors, furniture) must be placed within spatial structure elements (spaces, storeys, buildings) using containment or referencing.

**Key Functions:**

#### `assign_container()`
```python
assign_container(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Places physical products inside a spatial structure element. Establishes an `IfcRelContainedInSpatialStructure` relationship. Each product may only be contained in one structure at a time.

**Parameters:**
- `products`: List of IfcElement instances to place in the space
- `relating_structure`: IfcSpatialStructureElement (IfcBuilding, IfcBuildingStorey, or IfcSpace)

**Returns:** IfcRelContainedInSpatialStructure or None if list is empty

#### `unassign_container()`
```python
unassign_container(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance]
) → None
```
Removes containment relationships from products.

#### `reference_structure()`
```python
reference_structure(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Creates non-hierarchical references (IfcRelReferencedInSpatialStructure). Unlike containment, a product can be referenced in multiple spaces simultaneously. Used for elements spanning multiple floors (stairs, multi-storey columns).

#### `dereference_structure()`
```python
dereference_structure(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → None
```
Removes a referencing relationship between products and a spatial structure.

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()
project  = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")
site     = ifcopenshell.api.root.create_entity(model, ifc_class="IfcSite")
building = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuilding")
storey   = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuildingStorey")
storey2  = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuildingStorey")

# Build spatial hierarchy
ifcopenshell.api.aggregate.assign_object(model, products=[site], relating_object=project)
ifcopenshell.api.aggregate.assign_object(model, products=[building], relating_object=site)
ifcopenshell.api.aggregate.assign_object(model, products=[storey, storey2], relating_object=building)

# Create elements
wall   = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
column = ifcopenshell.api.root.create_entity(model, ifc_class="IfcColumn")

# Containment (one space only)
ifcopenshell.api.spatial.assign_container(model, products=[wall], relating_structure=storey)

# Referencing (multiple spaces allowed)
ifcopenshell.api.spatial.assign_container(model, products=[column], relating_structure=storey)
ifcopenshell.api.spatial.reference_structure(model, products=[column], relating_structure=storey2)
```

---

### 3. geometry

**Description:** Creates geometric representations and assigns them to elements. Supports arbitrary geometry (mesh, boolean operations) and parametric geometry following IFC construction rules (layered walls, profiled beams, etc.).

**Key Functions:**

#### `edit_object_placement()`
```python
edit_object_placement(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    matrix: numpy.ndarray | None = None,
    is_si: bool = True,
    should_transform_children: bool = False
) → ifcopenshell.entity_instance
```
Sets the 3D position and orientation of an element using a 4x4 transformation matrix. Defaults to identity matrix (origin). Returns IfcLocalPlacement.

#### `assign_representation()`
```python
assign_representation(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance
) → None
```
Associates a geometric representation with a product.

#### `add_wall_representation()`
```python
add_wall_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    length: float = 1.0,
    height: float = 3.0,
    direction_sense: str = 'POSITIVE',
    offset: float = 0.0,
    thickness: float = 0.2,
    x_angle: float = 0.0,
    clippings: list | None = None,
    booleans: list | None = None
) → ifcopenshell.entity_instance
```
Creates a parametric wall body geometry. Returns IfcShapeRepresentation.

#### `add_mesh_representation()`
```python
add_mesh_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    vertices: list,
    edges: list | None = None,
    faces: list | None = None,
    coordinate_offset: list | None = None,
    unit_scale: float | None = None,
    force_faceted_brep: bool = False
) → ifcopenshell.entity_instance
```
Creates arbitrary polygon mesh geometry from vertices and face indices.

#### `add_profile_representation()`
```python
add_profile_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,
    depth: float = 1.0,
    cardinal_point: int = 5,
    clippings: list | None = None,
    placement_zx_axes: tuple = (None, None)
) → ifcopenshell.entity_instance
```
Creates profiled extrusion geometry for beams, columns, etc.

#### `add_axis_representation()`
```python
add_axis_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    axis: list
) → ifcopenshell.entity_instance
```
Creates a simplified axis line representation for walls, beams, columns.

#### `add_slab_representation()`
```python
add_slab_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    depth: float = 0.2,
    direction_sense: str = 'POSITIVE',
    offset: float = 0.0,
    x_angle: float = 0.0,
    clippings: list | None = None,
    polyline: list | None = None
) → ifcopenshell.entity_instance
```

#### `add_door_representation()`
```python
add_door_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    operation_type: str = 'SINGLE_SWING_LEFT',
    lining_properties: dict | None = None,
    panel_properties: dict | None = None,
    part_of_product: ifcopenshell.entity_instance | None = None,
    unit_scale: float | None = None
) → ifcopenshell.entity_instance | None
```

#### `add_window_representation()`
```python
add_window_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    partition_type: str = 'SINGLE_PANEL',
    lining_properties: dict | None = None,
    panel_properties: list | None = None,
    part_of_product: ifcopenshell.entity_instance | None = None,
    unit_scale: float | None = None
) → ifcopenshell.entity_instance | None
```

#### `add_boolean()`
```python
add_boolean(
    file: ifcopenshell.file,
    first_item: ifcopenshell.entity_instance,
    second_items: list[ifcopenshell.entity_instance],
    operator: str = 'DIFFERENCE'
) → list[ifcopenshell.entity_instance]
```
Performs boolean operations (DIFFERENCE, INTERSECTION, UNION) on representation items.

#### Other geometry functions:
- `remove_representation()` - Removes a shape representation
- `unassign_representation()` - Disconnects representation from product
- `map_representation()` - Creates IfcRepresentationMap for type sharing
- `connect_element()` - Connects two elements (IfcRelConnectsElements)
- `connect_path()` - Connects path elements (walls, etc.)
- `connect_wall()` - Specialized wall-to-wall connection
- `disconnect_element()` / `disconnect_path()` - Remove connections
- `create_2pt_wall()` - Creates wall from two points
- `regenerate_wall_representation()` - Rebuilds wall geometry after connections change
- `add_railing_representation()` - Creates railing geometry along a path
- `add_footprint_representation()` - Creates 2D footprint curves
- `add_shape_aspect()` - Associates shape aspects to representation items
- `remove_boolean()` - Removes a boolean operation
- `validate_type()` - Validates and corrects RepresentationType

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api
import numpy

model = ifcopenshell.file()
ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject")

# Set up contexts
model3d = ifcopenshell.api.context.add_context(model, context_type="Model")
body = ifcopenshell.api.context.add_context(model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create wall with parametric geometry
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
representation = ifcopenshell.api.geometry.add_wall_representation(
    model, context=body, length=5, height=3, thickness=0.2)
ifcopenshell.api.geometry.assign_representation(model, product=wall, representation=representation)
ifcopenshell.api.geometry.edit_object_placement(model, product=wall)

# Custom mesh geometry
mesh_rep = ifcopenshell.api.geometry.add_mesh_representation(
    model,
    context=body,
    vertices=[[0,0,0],[1,0,0],[1,1,0],[0,1,0],[0,0,1],[1,0,1],[1,1,1],[0,1,1]],
    faces=[[0,1,2,3],[4,7,6,5],[0,4,5,1],[1,5,6,2],[2,6,7,3],[3,7,4,0]]
)
```

---

### 4. type

**Description:** Manages construction types for physical elements. Types group occurrences sharing the same shape, properties, and materials (e.g., wall types, window types, column types). Using types is described as "critical to the success of any project."

**Key Functions:**

#### `assign_type()`
```python
assign_type(
    file: ifcopenshell.file,
    related_objects: list[ifcopenshell.entity_instance],
    relating_type: ifcopenshell.entity_instance,
    should_map_representations: bool = True
) → ifcopenshell.entity_instance | None
```
Assigns a type to occurrences. Occurrences inherit all properties and materials from the type. If the type has geometry, occurrences share it via mapped representations.

**Parameters:**
- `related_objects`: List of IfcElement occurrences
- `relating_type`: The IfcElementType to assign
- `should_map_representations`: Whether to automatically map type geometry to occurrences

**Returns:** IfcRelDefinesByType or None if list is empty

#### `map_type_representations()`
```python
map_type_representations(
    file: ifcopenshell.file,
    related_object: ifcopenshell.entity_instance,
    relating_type: ifcopenshell.entity_instance
) → None
```
Ensures occurrence has the same representation as its type. Called when type representations change to maintain consistency.

#### `unassign_type()`
```python
unassign_type(
    file: ifcopenshell.file,
    related_objects: list[ifcopenshell.entity_instance]
) → None
```
Removes type association from occurrences. Note: does not automatically remove mapped representations or material usages.

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()
project = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")

# Create a wall type with material layer set
wall_type = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWallType", name="WT-200")
material_set = ifcopenshell.api.material.add_material_set(
    model, name="WT-200", set_type="IfcMaterialLayerSet")

gypsum = ifcopenshell.api.material.add_material(model, name="Gypsum Board")
insulation = ifcopenshell.api.material.add_material(model, name="Insulation")
steel = ifcopenshell.api.material.add_material(model, name="Steel Stud")

layer1 = ifcopenshell.api.material.add_layer(model, layer_set=material_set, material=gypsum)
layer2 = ifcopenshell.api.material.add_layer(model, layer_set=material_set, material=steel)
layer3 = ifcopenshell.api.material.add_layer(model, layer_set=material_set, material=insulation)
layer4 = ifcopenshell.api.material.add_layer(model, layer_set=material_set, material=gypsum)

ifcopenshell.api.material.edit_layer(model, layer=layer1, attributes={"LayerThickness": 0.012})
ifcopenshell.api.material.edit_layer(model, layer=layer2, attributes={"LayerThickness": 0.090})
ifcopenshell.api.material.edit_layer(model, layer=layer3, attributes={"LayerThickness": 0.012})
ifcopenshell.api.material.assign_material(model, products=[wall_type], type="IfcMaterialLayerSet", material=material_set)

# Create occurrences and assign type
wall1 = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
wall2 = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
ifcopenshell.api.type.assign_type(model, related_objects=[wall1, wall2], relating_type=wall_type)
```

---

### 5. pset

**Description:** Manages property sets (psets) and quantity sets (qtos) for IFC elements. Property sets store key-value metadata; quantity sets store measurable data. Essential for encoding element specifications, performance data, and compliance information.

**Key Functions:**

#### `add_pset()`
```python
add_pset(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    name: str,
    ifc2x3_subclass: str | None = None
) → ifcopenshell.entity_instance
```
Adds a new property set to a product. Standard sets use "Pset_" prefix; custom sets should use alternative prefixes. Returns IfcPropertySet.

#### `edit_pset()`
```python
edit_pset(
    file: ifcopenshell.file,
    pset: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict[str, Any] | None = None,
    pset_template: ifcopenshell.entity_instance | None = None,
    should_purge: bool = True
) → None
```
Edits property set name and properties. Python types map automatically:
- `str` → IfcLabel
- `float` → IfcReal
- `bool` → IfcBoolean
- `int` → IfcInteger
- `None` → deletes property (when `should_purge=True`)

#### `add_qto()`
```python
add_qto(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    name: str
) → ifcopenshell.entity_instance
```
Adds a quantity set for measurable data. Standard sets use "Qto_" prefix. Returns IfcElementQuantity.

#### `edit_qto()`
```python
edit_qto(
    file: ifcopenshell.file,
    qto: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict[str, Any] | None = None,
    pset_template: ifcopenshell.entity_instance | None = None
) → None
```
Auto-detects quantity types from property names ("area" → IfcAreaMeasure, etc.).

#### `assign_pset()`
```python
assign_pset(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Assigns a pset to multiple elements (shared pset). Returns IfcRelDefinesByProperties.

#### `remove_pset()`
```python
remove_pset(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    pset: ifcopenshell.entity_instance
) → None
```

#### `unassign_pset()`
```python
unassign_pset(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance
) → None
```
Removes assignment without deleting the pset itself.

#### `unshare_pset()`
```python
unshare_pset(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance
) → list[ifcopenshell.entity_instance]
```
Creates individual pset copies from a shared pset. Returns list of new psets.

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")

# Add standard property set
pset = ifcopenshell.api.pset.add_pset(model, product=wall, name="Pset_WallCommon")
ifcopenshell.api.pset.edit_pset(model, pset=pset, properties={
    "Reference": "WT-200",
    "IsExternal": True,
    "FireRating": "REI60",
    "ThermalTransmittance": 0.28,
    "LoadBearing": True,
})

# Add quantity set
qto = ifcopenshell.api.pset.add_qto(model, product=wall, name="Qto_WallBaseQuantities")
ifcopenshell.api.pset.edit_qto(model, qto=qto, properties={
    "Length": 5.0,
    "Height": 3.0,
    "Width": 0.2,
    "NetSideArea": 14.4,
    "GrossVolume": 3.0,
})

# Shared pset across multiple elements
wall2 = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")
shared_pset = ifcopenshell.api.pset.add_pset(model, product=wall, name="Pset_BuildingElement")
ifcopenshell.api.pset.assign_pset(model, products=[wall, wall2], pset=shared_pset)
```

---

### 6. material

**Description:** Manages physical material definitions and their associations to building elements. Covers simple single materials, layered constructions (walls), profiled constructions (beams), and composite materials (windows). Visual styling is handled separately via the style API.

**Key Functions:**

#### `add_material()`
```python
add_material(
    file: ifcopenshell.file,
    name: str | None = None,
    category: str | None = None,
    description: str | None = None
) → ifcopenshell.entity_instance
```
Creates an IfcMaterial. Category standardizes classification: concrete, steel, aluminium, block, brick, stone, wood, glass, gypsum, plastic, earth.

#### `add_material_set()`
```python
add_material_set(
    file: ifcopenshell.file,
    name: str = 'Unnamed',
    set_type: str = 'IfcMaterialConstituentSet'
) → ifcopenshell.entity_instance
```
Creates a material container. Types:
- `IfcMaterialLayerSet` - for layered walls, slabs
- `IfcMaterialProfileSet` - for profiled beams, columns
- `IfcMaterialConstituentSet` - for composite elements (windows, doors)
- `IfcMaterialList` - legacy IFC2X3 approach

#### `assign_material()`
```python
assign_material(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    type: str = 'IfcMaterial',
    material: ifcopenshell.entity_instance | None = None
) → ifcopenshell.entity_instance | list | None
```
Associates materials with products. Typically applied to types rather than individual occurrences.

#### `add_layer()` / `edit_layer()` / `remove_layer()`
```python
add_layer(file, layer_set, material, name=None) → entity_instance
edit_layer(file, layer, attributes=None, material=None) → None
remove_layer(file, layer) → None
```
Manage individual layers within IfcMaterialLayerSet. Key attribute: `LayerThickness`.

#### `add_constituent()` / `edit_constituent()` / `remove_constituent()`
```python
add_constituent(file, constituent_set, material, name=None) → entity_instance
edit_constituent(file, constituent, attributes=None, material=None) → None
remove_constituent(file, constituent) → None
```
Manage materials within IfcMaterialConstituentSet (e.g., frame + glazing for windows).

#### `add_profile()` / `edit_profile()` / `remove_profile()`
```python
add_profile(file, profile_set, material=None, profile=None, name=None) → entity_instance
edit_profile(file, profile, attributes=None, material=None) → None
remove_profile(file, profile) → None
```
Manage profile items within IfcMaterialProfileSet for extruded structural elements.

#### `edit_layer_usage()` / `edit_profile_usage()`
```python
edit_layer_usage(file, usage, attributes) → None
edit_profile_usage(file, usage, attributes) → None
```
Control offset from reference line/axis placement.

#### Other functions:
- `copy_material()` - Duplicates material with all properties and styles
- `edit_material()` - Updates material name, category, description
- `remove_material()` / `remove_material_set()` - Delete material definitions
- `add_list_item()` / `remove_list_item()` - Manage legacy IfcMaterialList
- `assign_profile()` - Changes profile curve and updates geometry
- `reorder_set_item()` - Reorders layers or profiles within sets
- `set_shape_aspect_constituents()` - Associates geometry to specific constituents
- `unassign_material()` - Removes material associations from products
- `edit_assigned_material()` - Modifies assigned material properties

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()

# Simple material
concrete = ifcopenshell.api.material.add_material(model, name="C30/37", category="concrete")
steel = ifcopenshell.api.material.add_material(model, name="S355", category="steel")

# Layered wall material
wall_type = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWallType")
layer_set = ifcopenshell.api.material.add_material_set(
    model, name="Ext Wall 300", set_type="IfcMaterialLayerSet")
layer1 = ifcopenshell.api.material.add_layer(model, layer_set=layer_set,
    material=ifcopenshell.api.material.add_material(model, name="Brick", category="brick"))
ifcopenshell.api.material.edit_layer(model, layer=layer1, attributes={"LayerThickness": 0.102})
ifcopenshell.api.material.assign_material(model, products=[wall_type],
    type="IfcMaterialLayerSet", material=layer_set)

# Profiled beam material
column_type = ifcopenshell.api.root.create_entity(model, ifc_class="IfcColumnType")
profile_set = ifcopenshell.api.material.add_material_set(
    model, name="HEA200", set_type="IfcMaterialProfileSet")
profile_item = ifcopenshell.api.material.add_profile(
    model, profile_set=profile_set, material=steel)
ifcopenshell.api.material.assign_material(model, products=[column_type],
    type="IfcMaterialProfileSet", material=profile_set)
```

---

### 7. aggregate

**Description:** Manages hierarchical decomposition relationships in IFC. All physical and spatial elements form a tree starting at IfcProject. Aggregation breaks larger wholes into smaller parts (IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace; or physical assemblies like stairs comprising flights and landings).

**Key Functions:**

#### `assign_object()`
```python
assign_object(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_object: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Assigns products as parts of a parent/whole. Creates IfcRelAggregate relationship. Automatically removes previous aggregation, containment, or nesting relationships. Recalculates IFC placements relative to new parent.

**Parameters:**
- `products`: Component parts (IfcElement or IfcSpatialStructureElement subclasses)
- `relating_object`: Parent/whole element

**Returns:** IfcRelAggregate or None if list is empty

#### `unassign_object()`
```python
unassign_object(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance]
) → None
```
Removes products from their aggregate. Warning: may make products invisible in some IFC applications.

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()

# Build standard spatial hierarchy
project  = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")
site     = ifcopenshell.api.root.create_entity(model, ifc_class="IfcSite")
building = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuilding")
storey   = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuildingStorey")

ifcopenshell.api.aggregate.assign_object(model, products=[site], relating_object=project)
ifcopenshell.api.aggregate.assign_object(model, products=[building], relating_object=site)
ifcopenshell.api.aggregate.assign_object(model, products=[storey], relating_object=building)

# Physical assembly: stair from parts
stair = ifcopenshell.api.root.create_entity(model, ifc_class="IfcStair")
flight = ifcopenshell.api.root.create_entity(model, ifc_class="IfcStairFlight")
landing = ifcopenshell.api.root.create_entity(model, ifc_class="IfcSlab",
    predefined_type="LANDING")
railing = ifcopenshell.api.root.create_entity(model, ifc_class="IfcRailing")
ifcopenshell.api.aggregate.assign_object(model,
    products=[flight, landing, railing], relating_object=stair)
```

---

### 8. context

**Description:** Manages geometric representation contexts that classify when and how geometry is used. Contexts distinguish between 3D body models, 2D plans, clearance zones, structural axes, and annotations. Required before any geometry can be assigned.

**Key Functions:**

#### `add_context()`
```python
add_context(
    file: ifcopenshell.file,
    context_type: str | None = None,
    context_identifier: str | None = None,
    target_view: str | None = None,
    target_scale: float | None = None,
    parent: ifcopenshell.entity_instance | None = None
) → ifcopenshell.entity_instance
```
Creates root contexts or subcontexts. Root contexts use only `context_type`; subcontexts also require `context_identifier`, `target_view`, and `parent`.

**Parameters:**
- `context_type`: "Model" (3D) or "Plan" (2D)
- `context_identifier`: Purpose of the representation:
  - `Body` - main 3D solid/surface geometry
  - `Box` - bounding box
  - `Axis` - structural centerline/axis
  - `Profile` - cross-section profile
  - `Footprint` - plan footprint
  - `Clearance` - clearance zone
  - `Annotation` - annotations and dimensions
- `target_view`: How geometry is viewed:
  - `MODEL_VIEW` - general 3D view
  - `PLAN_VIEW` - top-down 2D plan
  - `ELEVATION_VIEW` - vertical elevation
  - `SECTION_VIEW` - cross-section
  - `GRAPH_VIEW` - schematic/axis
  - `SKETCH_VIEW` - sketch
- `target_scale`: Zoom level detail
- `parent`: Parent context (None for root contexts)

**Returns:** IfcGeometricRepresentationContext or IfcGeometricRepresentationSubContext

#### `edit_context()`
```python
edit_context(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    attributes: dict[str, Any]
) → None
```
Modifies context attributes (e.g., rename ContextIdentifier).

#### `remove_context()`
```python
remove_context(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance
) → None
```
Removes context and all associated representations and child subcontexts.

**Code Example:**
```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file()
ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject")

# Root contexts (required first)
model3d = ifcopenshell.api.context.add_context(model, context_type="Model")
plan    = ifcopenshell.api.context.add_context(model, context_type="Plan")

# 3D subcontexts
body = ifcopenshell.api.context.add_context(model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

axis = ifcopenshell.api.context.add_context(model,
    context_type="Model", context_identifier="Axis",
    target_view="GRAPH_VIEW", parent=model3d)

box = ifcopenshell.api.context.add_context(model,
    context_type="Model", context_identifier="Box",
    target_view="MODEL_VIEW", parent=model3d)

# 2D subcontexts
plan_axis = ifcopenshell.api.context.add_context(model,
    context_type="Plan", context_identifier="Axis",
    target_view="GRAPH_VIEW", parent=plan)

annotation = ifcopenshell.api.context.add_context(model,
    context_type="Plan", context_identifier="Annotation",
    target_view="PLAN_VIEW", parent=plan)

section_anno = ifcopenshell.api.context.add_context(model,
    context_type="Plan", context_identifier="Annotation",
    target_view="SECTION_VIEW", parent=plan)
```

---

## Additional API Categories (Summary)

### 9. unit
Manages unit systems (SI, imperial). Required for any model with geometry.

```python
ifcopenshell.api.run("unit.assign_unit", model)  # Default SI units
```

### 10. owner
Manages users, organizations, and ownership history.

```python
person = ifcopenshell.api.run("owner.add_person", model, identification="jdoe", family_name="Doe")
org    = ifcopenshell.api.run("owner.add_organisation", model, identification="ACME", name="ACME Corp")
user   = ifcopenshell.api.run("owner.add_person_and_organisation", model, person=person, organisation=org)
ifcopenshell.api.run("owner.set_user", model, user=user)
```

### 11. classification
Links elements to classification systems (Uniclass, OmniClass, etc.).

```python
system = ifcopenshell.api.run("classification.add_classification", model, classification="Uniclass 2015")
ref    = ifcopenshell.api.run("classification.add_reference", model,
    products=[wall], identification="Ss_20_10_30", name="Walls", classification=system)
```

### 12. cost
Manages cost schedules, cost items, and cost assignments.

```python
schedule  = ifcopenshell.api.run("cost.add_cost_schedule", model, name="BOQ")
cost_item = ifcopenshell.api.run("cost.add_cost_item", model, cost_schedule=schedule)
ifcopenshell.api.run("cost.add_cost_item_quantity", model, cost_item=cost_item, ifc_class="IfcQuantityArea")
```

### 13. sequence
Manages work schedules, tasks, and time-based planning (4D BIM).

```python
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model, name="Construction Program")
task = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Foundation", identification="A")
ifcopenshell.api.run("sequence.edit_task_time", model, task=task,
    attributes={"ScheduleStart": "2026-01-01", "ScheduleFinish": "2026-03-01"})
```

### 14. void
Manages openings and voids in elements (e.g., door/window openings in walls).

```python
opening = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcOpeningElement")
ifcopenshell.api.run("void.add_opening", model, opening=opening, element=wall)
```

### 15. nest
Manages nesting relationships (distinct from aggregation; for ports and distribution elements).

```python
ifcopenshell.api.run("nest.assign_object", model, related_objects=[port], relating_object=element)
```

### 16. group
Groups elements for organizational or functional purposes.

```python
group = ifcopenshell.api.run("group.add_group", model, name="Fire Zone A", ifc_class="IfcGroup")
ifcopenshell.api.run("group.assign_group", model, products=[wall, door], group=group)
```

### 17. layer
Manages presentation layers (CAD layers) for elements.

```python
layer = ifcopenshell.api.run("layer.add_layer", model, name="A-WALL")
ifcopenshell.api.run("layer.assign_layer", model, items=[representation], layer=layer)
```

### 18. style
Manages visual styles (colors, materials, surface textures) for representations.

```python
style = ifcopenshell.api.run("style.add_style", model, name="Concrete")
ifcopenshell.api.run("style.add_surface_style", model, style=style,
    attributes={"SurfaceColour": {"Red": 0.8, "Green": 0.8, "Blue": 0.8}})
ifcopenshell.api.run("style.assign_representation_styles", model,
    representation=body_rep, styles=[style])
```

### 19. structural
Manages structural analysis elements, loads, and boundary conditions.

```python
analysis_model = ifcopenshell.api.run("structural.add_structural_analysis_model", model)
member = ifcopenshell.api.run("structural.add_structural_member", model,
    structural_analysis_model=analysis_model)
```

### 20. system
Manages MEP systems (HVAC, plumbing, electrical) and distribution networks.

```python
system = ifcopenshell.api.run("system.add_system", model, ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.assign_system", model, products=[duct], system=system)
```

### 21. boundary
Manages space boundary conditions for energy analysis.

```python
boundary = ifcopenshell.api.run("boundary.assign_connection_geometry", model, rel=rel_boundary)
```

### 22. profile
Manages cross-section profile definitions (I-beams, channels, circles, etc.).

```python
profile = ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile=points, name="Custom Profile")
```

### 23. library
Manages external library references and data libraries.

```python
library = ifcopenshell.api.run("library.add_library", model, name="Manufacturer Catalog")
ref = ifcopenshell.api.run("library.add_reference", model, library=library)
ifcopenshell.api.run("library.assign_reference", model, reference=ref, products=[element])
```

### 24. document
Manages document references attached to elements.

```python
info = ifcopenshell.api.run("document.add_information", model)
ifcopenshell.api.run("document.edit_information", model, information=info,
    attributes={"Identification": "SPEC-001", "Name": "Technical Specification"})
ifcopenshell.api.run("document.assign_document", model, products=[element], document=info)
```

### 25. constraint
Manages metric constraints and object constraints.

```python
objective = ifcopenshell.api.run("constraint.add_objective", model, name="FireRating")
metric    = ifcopenshell.api.run("constraint.add_metric", model, objective=objective)
ifcopenshell.api.run("constraint.assign_constraint", model,
    products=[element], constraint=objective)
```

### 26. resource
Manages construction resources (labour, equipment, materials) and their use in work tasks.

```python
resource = ifcopenshell.api.run("resource.add_resource", model,
    ifc_class="IfcLaborResource", name="Concrete crew")
ifcopenshell.api.run("resource.assign_resource", model, relating_resource=resource, related_object=task)
```

### 27. drawing
Manages drawing sheets, views, and annotations for documentation.

```python
drawing = ifcopenshell.api.run("drawing.add_drawing", model, target_view="PLAN_VIEW",
    name="Ground Floor Plan")
```

### 28. project
Manages project file creation (distinct from root.create_entity).

```python
model = ifcopenshell.api.project.create_file(version="IFC4")
```

### 29. feature
Manages feature elements and opening operations.

```python
feature = ifcopenshell.api.run("feature.add_feature", model, element=wall,
    feature=opening)
```

### 30. alignment
Manages linear alignment geometry for infrastructure (roads, railways).

```python
alignment = ifcopenshell.api.run("alignment.add_alignment", model)
```

---

## Complete Minimal IFC File Example

```python
import ifcopenshell
import ifcopenshell.api

# Create model
model = ifcopenshell.file()

# Project setup
project = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Contexts
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body    = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Spatial hierarchy
site     = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite", name="Default Site")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding", name="Building A")
storey   = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model, products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model, products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model, products=[storey], relating_object=building)

# Wall with geometry
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 1")
rep  = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model, product=wall, representation=rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model, products=[wall], relating_structure=storey)

# Properties
pset = ifcopenshell.api.run("pset.add_pset", model, product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90",
})

# Save
model.write("my_project.ifc")
```

---

## Sources

- [IfcOpenShell API Index](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html)
- [ifcopenshell.api.root](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/root/index.html)
- [ifcopenshell.api.spatial](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/spatial/index.html)
- [ifcopenshell.api.geometry](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/geometry/index.html)
- [ifcopenshell.api.type](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/type/index.html)
- [ifcopenshell.api.pset](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/pset/index.html)
- [ifcopenshell.api.material](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/material/index.html)
- [ifcopenshell.api.aggregate](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/aggregate/index.html)
- [ifcopenshell.api.context](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/context/index.html)
- [IfcOpenShell Code Examples](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)
