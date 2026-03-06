# IFC Model Creation — API Method Signatures

Complete method signatures for all `ifcopenshell.api.run()` functions used in IFC model creation workflows.

---

## project — File Creation

### `project.create_file`

```python
ifcopenshell.api.run("project.create_file",
    version: str = "IFC4"           # "IFC2X3", "IFC4", or "IFC4X3"
) → ifcopenshell.file
```

Creates a new IFC file with pre-populated header metadata, timestamps, and MVD settings. ALWAYS use this for production files instead of `ifcopenshell.file()`.

**Returns:** New `ifcopenshell.file` instance with proper header.

---

## root — Entity Creation and Management

### `root.create_entity`

```python
ifcopenshell.api.run("root.create_entity", model,
    ifc_class: str = "IfcBuildingElementProxy",
    predefined_type: str | None = None,
    name: str | None = None
) → ifcopenshell.entity_instance
```

Creates any rooted IFC entity. Automatically generates GlobalId and attaches OwnerHistory (when set). Validates predefined types against schema.

**Parameters:**
- `ifc_class` — Any rooted IFC class: `IfcProject`, `IfcSite`, `IfcBuilding`, `IfcBuildingStorey`, `IfcWall`, `IfcWallType`, `IfcSlab`, `IfcColumn`, `IfcDoor`, `IfcWindow`, `IfcOpeningElement`, `IfcSpace`, etc.
- `predefined_type` — Built-in or user-defined subtype (e.g., `"FLOOR"` for IfcSlab, `"STANDARD"` for IfcWall)
- `name` — Human-readable name

**Returns:** Newly created entity instance.

### `root.copy_class`

```python
ifcopenshell.api.run("root.copy_class", model,
    product: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

Duplicates a product preserving: placement, property sets, nested ports, aggregation, containment, type associations, voids, materials, and group memberships. Does NOT copy representations.

### `root.reassign_class`

```python
ifcopenshell.api.run("root.reassign_class", model,
    product: ifcopenshell.entity_instance,
    ifc_class: str = "IfcBuildingElementProxy",
    predefined_type: str | None = None,
    occurrence_class: str | None = None
) → ifcopenshell.entity_instance
```

Changes the IFC class of an existing product while retaining geometry and relationships.

### `root.remove_product`

```python
ifcopenshell.api.run("root.remove_product", model,
    product: ifcopenshell.entity_instance
) → None
```

Removes a product and all its relationships (geometry, placement, properties, materials, containment, aggregation, nesting). Use this instead of `model.remove()` for safe deletion.

---

## aggregate — Hierarchical Decomposition

### `aggregate.assign_object`

```python
ifcopenshell.api.run("aggregate.assign_object", model,
    products: list[ifcopenshell.entity_instance],
    relating_object: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```

Assigns products as parts of a parent/whole. Creates IfcRelAggregate. Automatically removes previous aggregation, containment, or nesting relationships. Recalculates placements relative to new parent.

**Parameters:**
- `products` — Component parts (spatial or physical elements)
- `relating_object` — Parent/whole element

**Returns:** IfcRelAggregate or None if list is empty.

### `aggregate.unassign_object`

```python
ifcopenshell.api.run("aggregate.unassign_object", model,
    products: list[ifcopenshell.entity_instance]
) → None
```

Removes products from their aggregate parent.

---

## spatial — Spatial Containment

### `spatial.assign_container`

```python
ifcopenshell.api.run("spatial.assign_container", model,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```

Places physical products inside a spatial structure element. Creates IfcRelContainedInSpatialStructure. Each product can only be contained in ONE structure at a time.

**Parameters:**
- `products` — List of IfcElement instances (walls, doors, columns, etc.)
- `relating_structure` — IfcSpatialStructureElement (IfcBuilding, IfcBuildingStorey, or IfcSpace)

**Returns:** IfcRelContainedInSpatialStructure or None.

### `spatial.unassign_container`

```python
ifcopenshell.api.run("spatial.unassign_container", model,
    products: list[ifcopenshell.entity_instance]
) → None
```

### `spatial.reference_structure`

```python
ifcopenshell.api.run("spatial.reference_structure", model,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```

Creates non-hierarchical references (IfcRelReferencedInSpatialStructure). Unlike containment, a product CAN be referenced in multiple spaces. Use for elements spanning multiple floors.

### `spatial.dereference_structure`

```python
ifcopenshell.api.run("spatial.dereference_structure", model,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → None
```

---

## context — Geometric Representation Contexts

### `context.add_context`

```python
ifcopenshell.api.run("context.add_context", model,
    context_type: str | None = None,
    context_identifier: str | None = None,
    target_view: str | None = None,
    target_scale: float | None = None,
    parent: ifcopenshell.entity_instance | None = None
) → ifcopenshell.entity_instance
```

Creates root contexts or subcontexts. Root contexts use only `context_type`. Subcontexts require `context_identifier`, `target_view`, and `parent`.

**Parameters:**
- `context_type` — `"Model"` (3D) or `"Plan"` (2D)
- `context_identifier` — `"Body"`, `"Box"`, `"Axis"`, `"Profile"`, `"Footprint"`, `"Clearance"`, `"Annotation"`
- `target_view` — `"MODEL_VIEW"`, `"PLAN_VIEW"`, `"ELEVATION_VIEW"`, `"SECTION_VIEW"`, `"GRAPH_VIEW"`, `"SKETCH_VIEW"`
- `parent` — Parent context (None for root contexts)

**Returns:** IfcGeometricRepresentationContext or IfcGeometricRepresentationSubContext.

### `context.remove_context`

```python
ifcopenshell.api.run("context.remove_context", model,
    context: ifcopenshell.entity_instance
) → None
```

Removes context and all associated representations and child subcontexts.

---

## unit — Unit System

### `unit.assign_unit`

```python
ifcopenshell.api.run("unit.assign_unit", model,
    length: dict | None = None,
    area: dict | None = None,
    volume: dict | None = None
) → None
```

Assigns units to the model. Without arguments, assigns default SI units (meters, square meters, cubic meters, radians).

**Parameters (optional):**
- `length` — `{"is_metric": True, "raw": "METRES"}` or `{"is_metric": False, "raw": "FEET"}`
- `area` — `{"is_metric": True, "raw": "SQUARE_METRE"}`
- `volume` — `{"is_metric": True, "raw": "CUBIC_METRE"}`

---

## geometry — Geometry Creation and Placement

### `geometry.add_wall_representation`

```python
ifcopenshell.api.run("geometry.add_wall_representation", model,
    context: ifcopenshell.entity_instance,
    length: float = 1.0,
    height: float = 3.0,
    direction_sense: str = "POSITIVE",
    offset: float = 0.0,
    thickness: float = 0.2,
    x_angle: float = 0.0,
    clippings: list | None = None,
    booleans: list | None = None
) → ifcopenshell.entity_instance
```

Creates parametric wall body geometry. Returns IfcShapeRepresentation.

### `geometry.add_slab_representation`

```python
ifcopenshell.api.run("geometry.add_slab_representation", model,
    context: ifcopenshell.entity_instance,
    depth: float = 0.2,
    direction_sense: str = "POSITIVE",
    offset: float = 0.0,
    x_angle: float = 0.0,
    clippings: list | None = None,
    polyline: list | None = None
) → ifcopenshell.entity_instance
```

Creates slab geometry. `polyline` defines the footprint as list of (x, y) tuples.

### `geometry.add_profile_representation`

```python
ifcopenshell.api.run("geometry.add_profile_representation", model,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,
    depth: float = 1.0,
    cardinal_point: int = 5,
    clippings: list | None = None,
    placement_zx_axes: tuple = (None, None)
) → ifcopenshell.entity_instance
```

Creates profiled extrusion geometry for beams, columns. `cardinal_point` controls the alignment point (5 = centroid).

### `geometry.add_mesh_representation`

```python
ifcopenshell.api.run("geometry.add_mesh_representation", model,
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

### `geometry.add_door_representation`

```python
ifcopenshell.api.run("geometry.add_door_representation", model,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    operation_type: str = "SINGLE_SWING_LEFT",
    lining_properties: dict | None = None,
    panel_properties: dict | None = None
) → ifcopenshell.entity_instance | None
```

### `geometry.add_window_representation`

```python
ifcopenshell.api.run("geometry.add_window_representation", model,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    partition_type: str = "SINGLE_PANEL",
    lining_properties: dict | None = None,
    panel_properties: list | None = None
) → ifcopenshell.entity_instance | None
```

### `geometry.assign_representation`

```python
ifcopenshell.api.run("geometry.assign_representation", model,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance
) → None
```

Associates a geometric representation with a product. Call AFTER creating the representation.

### `geometry.edit_object_placement`

```python
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product: ifcopenshell.entity_instance,
    matrix: numpy.ndarray | None = None,
    is_si: bool = True,
    should_transform_children: bool = False
) → ifcopenshell.entity_instance
```

Sets 3D position and orientation using a 4x4 transformation matrix. Defaults to identity matrix (origin). Returns IfcLocalPlacement.

**Parameters:**
- `matrix` — 4x4 numpy array. Translation in column 3, rotation in top-left 3x3. None = identity (origin).
- `is_si` — If True, matrix values are in SI units (meters). If False, uses project units.
- `should_transform_children` — If True, nested elements move with parent.

---

## type — Type Assignment

### `type.assign_type`

```python
ifcopenshell.api.run("type.assign_type", model,
    related_objects: list[ifcopenshell.entity_instance],
    relating_type: ifcopenshell.entity_instance,
    should_map_representations: bool = True
) → ifcopenshell.entity_instance | None
```

Assigns a type to occurrences. Occurrences inherit properties and materials from the type. Creates IfcRelDefinesByType.

**Parameters:**
- `related_objects` — List of IfcElement occurrences
- `relating_type` — The IfcElementType to assign
- `should_map_representations` — Whether to map type geometry to occurrences

### `type.unassign_type`

```python
ifcopenshell.api.run("type.unassign_type", model,
    related_objects: list[ifcopenshell.entity_instance]
) → None
```

### `type.map_type_representations`

```python
ifcopenshell.api.run("type.map_type_representations", model,
    related_object: ifcopenshell.entity_instance,
    relating_type: ifcopenshell.entity_instance
) → None
```

Ensures occurrence has same representation as its type. Called when type representations change.

---

## pset — Property Sets and Quantities

### `pset.add_pset`

```python
ifcopenshell.api.run("pset.add_pset", model,
    product: ifcopenshell.entity_instance,
    name: str
) → ifcopenshell.entity_instance
```

Creates a new IfcPropertySet for a product. Standard sets use `"Pset_"` prefix. Custom sets MUST use a different prefix.

### `pset.edit_pset`

```python
ifcopenshell.api.run("pset.edit_pset", model,
    pset: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict[str, Any] | None = None,
    pset_template: ifcopenshell.entity_instance | None = None,
    should_purge: bool = True
) → None
```

Sets property values. Python type mapping:
- `str` → IfcLabel
- `float` → IfcReal
- `bool` → IfcBoolean
- `int` → IfcInteger
- `None` → deletes property (when `should_purge=True`)

### `pset.add_qto`

```python
ifcopenshell.api.run("pset.add_qto", model,
    product: ifcopenshell.entity_instance,
    name: str
) → ifcopenshell.entity_instance
```

Creates IfcElementQuantity. Standard sets use `"Qto_"` prefix.

### `pset.edit_qto`

```python
ifcopenshell.api.run("pset.edit_qto", model,
    qto: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict[str, Any] | None = None,
    pset_template: ifcopenshell.entity_instance | None = None
) → None
```

Auto-detects quantity types from property names ("area" → IfcAreaMeasure, "length" → IfcLengthMeasure, "volume" → IfcVolumeMeasure, "count" → IfcCountMeasure, "weight" → IfcMassMeasure).

### `pset.assign_pset`

```python
ifcopenshell.api.run("pset.assign_pset", model,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```

Shares a pset across multiple elements. Returns IfcRelDefinesByProperties.

### `pset.remove_pset`

```python
ifcopenshell.api.run("pset.remove_pset", model,
    product: ifcopenshell.entity_instance,
    pset: ifcopenshell.entity_instance
) → None
```

---

## void — Openings and Voids

### `void.add_opening`

```python
ifcopenshell.api.run("void.add_opening", model,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

Creates an IfcRelVoidsElement (boolean subtraction). The opening geometry is subtracted from the element geometry.

**Parameters:**
- `opening` — IfcOpeningElement with geometry
- `element` — IfcElement (wall, slab, etc.) to cut

### `void.add_filling`

```python
ifcopenshell.api.run("void.add_filling", model,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

Fills an opening with an element (door, window). Creates IfcRelFillsElement.

**Parameters:**
- `opening` — IfcOpeningElement that was created via `void.add_opening`
- `element` — IfcDoor, IfcWindow, or other filling element

### `void.remove_opening`

```python
ifcopenshell.api.run("void.remove_opening", model,
    opening: ifcopenshell.entity_instance
) → None
```

### `void.remove_filling`

```python
ifcopenshell.api.run("void.remove_filling", model,
    element: ifcopenshell.entity_instance
) → None
```

---

## owner — Ownership (Required for IFC2X3)

### `owner.add_person`

```python
ifcopenshell.api.run("owner.add_person", model,
    identification: str = "HSeldon",
    family_name: str = "Seldon",
    given_name: str = "Hari"
) → ifcopenshell.entity_instance
```

### `owner.add_organisation`

```python
ifcopenshell.api.run("owner.add_organisation", model,
    identification: str = "ORG",
    name: str = "Organisation"
) → ifcopenshell.entity_instance
```

### `owner.add_person_and_organisation`

```python
ifcopenshell.api.run("owner.add_person_and_organisation", model,
    person: ifcopenshell.entity_instance,
    organisation: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

### `owner.add_application`

```python
ifcopenshell.api.run("owner.add_application", model,
    application_developer: ifcopenshell.entity_instance | None = None,
    version: str | None = None,
    application_full_name: str = "IfcOpenShell",
    application_identifier: str = "IfcOpenShell"
) → ifcopenshell.entity_instance
```

### `owner.set_user`

```python
ifcopenshell.api.run("owner.set_user", model,
    user: ifcopenshell.entity_instance
) → None
```

Sets the active user for subsequent entity creation. In IFC2X3, this MUST be called before `root.create_entity` — the OwnerHistory is required and uses the set user.

---

## material — Material Assignment (Summary)

### `material.add_material`

```python
ifcopenshell.api.run("material.add_material", model,
    name: str | None = None,
    category: str | None = None,
    description: str | None = None
) → ifcopenshell.entity_instance
```

### `material.add_material_set`

```python
ifcopenshell.api.run("material.add_material_set", model,
    name: str = "Unnamed",
    set_type: str = "IfcMaterialConstituentSet"
) → ifcopenshell.entity_instance
```

Set types: `"IfcMaterialLayerSet"` (walls/slabs), `"IfcMaterialProfileSet"` (beams/columns), `"IfcMaterialConstituentSet"` (composite elements).

### `material.add_layer`

```python
ifcopenshell.api.run("material.add_layer", model,
    layer_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: str | None = None
) → ifcopenshell.entity_instance
```

### `material.edit_layer`

```python
ifcopenshell.api.run("material.edit_layer", model,
    layer: ifcopenshell.entity_instance,
    attributes: dict | None = None,
    material: ifcopenshell.entity_instance | None = None
) → None
```

Key attribute: `{"LayerThickness": 0.2}` (in SI meters).

### `material.assign_material`

```python
ifcopenshell.api.run("material.assign_material", model,
    products: list[ifcopenshell.entity_instance],
    type: str = "IfcMaterial",
    material: ifcopenshell.entity_instance | None = None
) → ifcopenshell.entity_instance | list | None
```

Associates materials with products. Typically applied to types, not individual occurrences.

---

## profile — Cross-Section Profiles

### `profile.add_parameterised_profile`

```python
ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class: str,
    **kwargs
) → ifcopenshell.entity_instance
```

Common profile classes and their parameters:
- `IfcRectangleProfileDef` — `XDim`, `YDim`
- `IfcCircleProfileDef` — `Radius`
- `IfcCircleHollowProfileDef` — `Radius`, `WallThickness`
- `IfcIShapeProfileDef` — `OverallWidth`, `OverallDepth`, `WebThickness`, `FlangeThickness`

### `profile.add_arbitrary_profile`

```python
ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile: list,
    name: str | None = None
) → ifcopenshell.entity_instance
```

Creates a profile from a list of 2D coordinate points defining a closed polyline.
