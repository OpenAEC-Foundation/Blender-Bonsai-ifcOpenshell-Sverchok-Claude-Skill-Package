# ifcos-syntax-api — Complete API Method Signatures

All signatures verified against IfcOpenShell v0.8.4 documentation.
Source: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html

---

## Invocation Pattern

```python
# Both patterns are equivalent:
result = ifcopenshell.api.run("module.function", file, **kwargs)
result = ifcopenshell.api.module.function(file, **kwargs)
```

---

## 1. root — Entity Creation and Manipulation

```python
root.create_entity(
    file: ifcopenshell.file,
    ifc_class: str = "IfcBuildingElementProxy",
    predefined_type: str | None = None,
    name: str | None = None
) → ifcopenshell.entity_instance
```
Creates a new rooted IFC product. Auto-generates GlobalId and ownership history.

```python
root.copy_class(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```
Duplicates product preserving: placement, psets, ports, aggregation, containment, type associations, voids, materials, group memberships. Does NOT copy representations.

```python
root.reassign_class(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    ifc_class: str = "IfcBuildingElementProxy",
    predefined_type: str | None = None,
    occurrence_class: str | None = None
) → ifcopenshell.entity_instance
```
Changes IFC class while retaining geometry and relationships.

```python
root.remove_product(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance
) → None
```
Smart delete: removes product and ALL relationships (geometry, placement, properties, materials, containment, aggregation, nesting).

---

## 2. spatial — Spatial Containment

```python
spatial.assign_container(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Places products in a spatial structure. Each product can only be contained in ONE structure. Returns IfcRelContainedInSpatialStructure.

```python
spatial.unassign_container(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance]
) → None
```

```python
spatial.reference_structure(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Non-hierarchical reference — a product can be referenced in multiple spaces (e.g., multi-storey columns).

```python
spatial.dereference_structure(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) → None
```

---

## 3. aggregate — Hierarchical Decomposition

```python
aggregate.assign_object(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    relating_object: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance | None
```
Creates IfcRelAggregate. Automatically removes previous aggregation/containment/nesting. Recalculates placements relative to new parent.

```python
aggregate.unassign_object(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance]
) → None
```

---

## 4. geometry — Representations and Placement

```python
geometry.edit_object_placement(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    matrix: numpy.ndarray | None = None,
    is_si: bool = True,
    should_transform_children: bool = False
) → ifcopenshell.entity_instance
```
Sets 3D position/orientation via 4x4 transformation matrix. Defaults to identity (origin).

```python
geometry.assign_representation(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    representation: ifcopenshell.entity_instance
) → None
```

```python
geometry.add_wall_representation(
    file: ifcopenshell.file,
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

```python
geometry.add_slab_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    depth: float = 0.2,
    direction_sense: str = "POSITIVE",
    offset: float = 0.0,
    x_angle: float = 0.0,
    clippings: list | None = None,
    polyline: list | None = None
) → ifcopenshell.entity_instance
```

```python
geometry.add_profile_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance,
    depth: float = 1.0,
    cardinal_point: int = 5,
    clippings: list | None = None,
    placement_zx_axes: tuple = (None, None)
) → ifcopenshell.entity_instance
```
Cardinal points: 1=bottom-left, 5=center (default), 9=top-right. Used for beams/columns.

```python
geometry.add_mesh_representation(
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

```python
geometry.add_door_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    operation_type: str = "SINGLE_SWING_LEFT",
    lining_properties: dict | None = None,
    panel_properties: dict | None = None,
    part_of_product: ifcopenshell.entity_instance | None = None,
    unit_scale: float | None = None
) → ifcopenshell.entity_instance | None
```

```python
geometry.add_window_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    overall_height: float | None = None,
    overall_width: float | None = None,
    partition_type: str = "SINGLE_PANEL",
    lining_properties: dict | None = None,
    panel_properties: list | None = None,
    part_of_product: ifcopenshell.entity_instance | None = None,
    unit_scale: float | None = None
) → ifcopenshell.entity_instance | None
```

```python
geometry.add_axis_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    axis: list
) → ifcopenshell.entity_instance
```

```python
geometry.add_footprint_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    ...
) → ifcopenshell.entity_instance
```

```python
geometry.add_railing_representation(
    file: ifcopenshell.file,
    context: ifcopenshell.entity_instance,
    ...
) → ifcopenshell.entity_instance
```

```python
geometry.add_boolean(
    file: ifcopenshell.file,
    first_item: ifcopenshell.entity_instance,
    second_items: list[ifcopenshell.entity_instance],
    operator: str = "DIFFERENCE"
) → list[ifcopenshell.entity_instance]
```
Operators: `DIFFERENCE`, `INTERSECTION`, `UNION`.

```python
geometry.remove_representation(file, representation) → None
geometry.unassign_representation(file, product, representation) → None
geometry.map_representation(file, ...) → ifcopenshell.entity_instance
geometry.remove_boolean(file, item) → None
geometry.connect_element(file, ...) → None
geometry.connect_path(file, ...) → None
geometry.connect_wall(file, ...) → None
geometry.disconnect_element(file, ...) → None
geometry.disconnect_path(file, ...) → None
geometry.create_2pt_wall(file, ...) → ifcopenshell.entity_instance
geometry.regenerate_wall_representation(file, ...) → None
geometry.add_shape_aspect(file, ...) → ifcopenshell.entity_instance
geometry.validate_type(file, ...) → None
```

---

## 5. context — Representation Contexts

```python
context.add_context(
    file: ifcopenshell.file,
    context_type: str | None = None,
    context_identifier: str | None = None,
    target_view: str | None = None,
    target_scale: float | None = None,
    parent: ifcopenshell.entity_instance | None = None
) → ifcopenshell.entity_instance
```
Root context: only `context_type` ("Model" or "Plan"). Subcontext: requires `context_identifier`, `target_view`, and `parent`.

Context identifiers: `Body`, `Box`, `Axis`, `Profile`, `Footprint`, `Clearance`, `Annotation`.
Target views: `MODEL_VIEW`, `PLAN_VIEW`, `ELEVATION_VIEW`, `SECTION_VIEW`, `GRAPH_VIEW`, `SKETCH_VIEW`.

```python
context.edit_context(file, context, attributes: dict) → None
context.remove_context(file, context) → None
```

---

## 6. type — Type Definitions

```python
type.assign_type(
    file: ifcopenshell.file,
    related_objects: list[ifcopenshell.entity_instance],
    relating_type: ifcopenshell.entity_instance,
    should_map_representations: bool = True
) → ifcopenshell.entity_instance | None
```

```python
type.unassign_type(
    file: ifcopenshell.file,
    related_objects: list[ifcopenshell.entity_instance]
) → None
```

```python
type.map_type_representations(
    file: ifcopenshell.file,
    related_object: ifcopenshell.entity_instance,
    relating_type: ifcopenshell.entity_instance
) → None
```

---

## 7. pset — Property and Quantity Sets

```python
pset.add_pset(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    name: str,
    ifc2x3_subclass: str | None = None
) → ifcopenshell.entity_instance
```

```python
pset.edit_pset(
    file: ifcopenshell.file,
    pset: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict[str, Any] | None = None,
    pset_template: ifcopenshell.entity_instance | None = None,
    should_purge: bool = True
) → None
```
Setting a property to `None` deletes it (when `should_purge=True`).

```python
pset.add_qto(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    name: str
) → ifcopenshell.entity_instance
```

```python
pset.edit_qto(
    file: ifcopenshell.file,
    qto: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict[str, Any] | None = None,
    pset_template: ifcopenshell.entity_instance | None = None
) → None
```

```python
pset.assign_pset(file, products: list, pset) → ifcopenshell.entity_instance | None
pset.remove_pset(file, product, pset) → None
pset.unassign_pset(file, products: list, pset) → None
pset.unshare_pset(file, products: list, pset) → list[ifcopenshell.entity_instance]
```

---

## 8. material — Material Definitions

```python
material.add_material(
    file: ifcopenshell.file,
    name: str | None = None,
    category: str | None = None,
    description: str | None = None
) → ifcopenshell.entity_instance
```
Standard categories: concrete, steel, aluminium, block, brick, stone, wood, glass, gypsum, plastic, earth.

```python
material.add_material_set(
    file: ifcopenshell.file,
    name: str = "Unnamed",
    set_type: str = "IfcMaterialConstituentSet"
) → ifcopenshell.entity_instance
```
Set types: `IfcMaterialLayerSet`, `IfcMaterialProfileSet`, `IfcMaterialConstituentSet`, `IfcMaterialList`.

```python
material.assign_material(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    type: str = "IfcMaterial",
    material: ifcopenshell.entity_instance | None = None
) → ifcopenshell.entity_instance | list | None
```

```python
material.add_layer(file, layer_set, material, name=None) → entity_instance
material.edit_layer(file, layer, attributes=None, material=None) → None
material.remove_layer(file, layer) → None
material.edit_layer_usage(file, usage, attributes) → None
```

```python
material.add_profile(file, profile_set, material=None, profile=None, name=None) → entity_instance
material.edit_profile(file, profile, attributes=None, material=None) → None
material.remove_profile(file, profile) → None
material.edit_profile_usage(file, usage, attributes) → None
material.assign_profile(file, ...) → None
```

```python
material.add_constituent(file, constituent_set, material, name=None) → entity_instance
material.edit_constituent(file, constituent, attributes=None, material=None) → None
material.remove_constituent(file, constituent) → None
```

```python
material.copy_material(file, material) → entity_instance
material.edit_material(file, material, attributes) → None
material.remove_material(file, material) → None
material.remove_material_set(file, material) → None
material.unassign_material(file, products: list) → None
material.edit_assigned_material(file, ...) → None
material.add_list_item(file, ...) → entity_instance
material.remove_list_item(file, ...) → None
material.reorder_set_item(file, ...) → None
material.set_shape_aspect_constituents(file, ...) → None
```

---

## 9. attribute — Direct Attribute Editing

```python
attribute.edit_attributes(
    file: ifcopenshell.file,
    product: ifcopenshell.entity_instance,
    attributes: dict[str, Any]
) → None
```

---

## 10. unit — Unit Assignment

```python
unit.assign_unit(
    file: ifcopenshell.file,
    units: list | None = None,
    length: dict | None = None,
    area: dict | None = None,
    volume: dict | None = None
) → ifcopenshell.entity_instance
```
Called without arguments: assigns default SI units (meters, square meters, cubic meters, radians).

```python
unit.add_si_unit(file, unit_type, prefix=None, name=None) → entity_instance
unit.add_conversion_based_unit(file, name, conversion_factor, ...) → entity_instance
unit.add_monetary_unit(file, currency) → entity_instance
unit.remove_unit(file, unit) → None
```

---

## 11. void — Openings and Voids

```python
void.add_opening(
    file: ifcopenshell.file,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

```python
void.add_filling(
    file: ifcopenshell.file,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance
) → ifcopenshell.entity_instance
```

```python
void.remove_opening(file, opening) → None
void.remove_filling(file, element) → None
```

---

## 12. owner — Ownership and Actors

```python
owner.add_person(file, identification=None, family_name=None, given_name=None) → entity_instance
owner.add_organisation(file, identification=None, name=None) → entity_instance
owner.add_person_and_organisation(file, person, organisation) → entity_instance
owner.set_user(file, user) → None
owner.add_application(file, ...) → entity_instance
owner.add_role(file, ...) → entity_instance
owner.add_address(file, ...) → entity_instance
owner.edit_person(file, person, attributes) → None
owner.edit_organisation(file, organisation, attributes) → None
owner.remove_person(file, person) → None
owner.remove_organisation(file, organisation) → None
owner.remove_person_and_organisation(file, person_and_organisation) → None
owner.remove_role(file, role) → None
owner.remove_address(file, address) → None
owner.unassign_actor(file, ...) → None
```

---

## 13. classification — Classification Systems

```python
classification.add_classification(file, classification) → entity_instance
classification.add_reference(file, products: list, identification, name=None, classification=None) → entity_instance
classification.edit_classification(file, classification, attributes) → None
classification.edit_reference(file, reference, attributes) → None
classification.remove_classification(file, classification) → None
classification.remove_reference(file, reference, products: list) → None
```

---

## 14. group — Element Grouping

```python
group.add_group(file, name=None, description=None, ifc_class="IfcGroup") → entity_instance
group.assign_group(file, products: list, group) → entity_instance | None
group.edit_group(file, group, attributes) → None
group.unassign_group(file, products: list, group) → None
group.remove_group(file, group) → None
group.update_group_products(file, group, products: list) → None
```

---

## 15. style — Presentation Styles

```python
style.add_style(file, name=None, ifc_class="IfcSurfaceStyle") → entity_instance
style.add_surface_style(file, style, ifc_class="IfcSurfaceStyleShading", attributes=None) → entity_instance
style.assign_representation_styles(file, shape_representation, styles: list, ...) → None
style.edit_surface_style(file, style, attributes) → None
style.remove_style(file, style) → None
style.unassign_representation_styles(file, ...) → None
style.assign_material_style(file, ...) → None
style.unassign_material_style(file, ...) → None
```

---

## 16. project — Project File Management

```python
project.create_file(version: str = "IFC4") → ifcopenshell.file
```
Creates a new IFC file with proper headers. Parameter is `version=`, NOT `schema=`.

```python
project.append_asset(file, library, element) → entity_instance
project.assign_declaration(file, definitions: list, relating_context) → entity_instance | None
project.unassign_declaration(file, definitions: list, relating_context) → None
```

---

## 17. cost — Cost Management

```python
cost.add_cost_schedule(file, name=None, predefined_type="NOTDEFINED") → entity_instance
cost.add_cost_item(file, cost_schedule=None, cost_item=None) → entity_instance
cost.add_cost_item_quantity(file, cost_item, ifc_class="IfcQuantityCount") → entity_instance
cost.add_cost_value(file, parent) → entity_instance
cost.edit_cost_item(file, cost_item, attributes) → None
cost.edit_cost_item_quantity(file, physical_quantity, attributes) → None
cost.edit_cost_schedule(file, cost_schedule, attributes) → None
cost.edit_cost_value(file, cost_value, attributes) → None
cost.edit_cost_value_formula(file, cost_value, formula) → None
cost.assign_cost_item_quantity(file, cost_item, products: list, ...) → None
cost.copy_cost_item(file, cost_item) → entity_instance
cost.remove_cost_item(file, cost_item) → None
cost.remove_cost_item_quantity(file, ...) → None
cost.remove_cost_schedule(file, cost_schedule) → None
cost.remove_cost_value(file, parent, cost_value) → None
cost.unassign_cost_item_quantity(file, ...) → None
```

---

## 18. sequence — Task Scheduling (4D BIM)

```python
sequence.add_work_schedule(file, name=None, predefined_type="NOTDEFINED", ...) → entity_instance
sequence.add_task(file, work_schedule=None, parent_task=None, name=None, ...) → entity_instance
sequence.edit_task_time(file, task_time=None, task=None, attributes=None) → None
sequence.add_time_period(file, recurrence_pattern) → entity_instance
sequence.add_work_calendar(file, name=None) → entity_instance
sequence.add_work_time(file, work_calendar, time_type="WorkingTimes") → entity_instance
sequence.assign_lag_time(file, rel_sequence, lag_value, duration_type="ELAPSEDTIME") → entity_instance
sequence.assign_process(file, relating_process, related_object) → entity_instance
sequence.assign_product(file, relating_product, related_object) → entity_instance
sequence.assign_recurrence_pattern(file, ...) → entity_instance
sequence.assign_sequence(file, relating_process, related_process, ...) → entity_instance
sequence.cascade_schedule(file, task) → None
sequence.edit_sequence(file, rel_sequence, attributes) → None
sequence.edit_task(file, task, attributes) → None
sequence.edit_work_calendar(file, work_calendar, attributes) → None
sequence.edit_work_schedule(file, work_schedule, attributes) → None
sequence.recalculate_schedule(file, work_schedule) → None
sequence.remove_task(file, task) → None
sequence.remove_time_period(file, time_period) → None
sequence.remove_work_calendar(file, work_calendar) → None
sequence.remove_work_schedule(file, work_schedule) → None
sequence.unassign_lag_time(file, rel_sequence) → None
sequence.unassign_process(file, relating_process, related_object) → None
sequence.unassign_product(file, relating_product, related_object) → None
sequence.unassign_recurrence_pattern(file, ...) → None
sequence.unassign_sequence(file, relating_process, related_process) → None
```

---

## 19. system — MEP Systems

```python
system.add_system(file, ifc_class="IfcDistributionSystem") → entity_instance
system.assign_system(file, products: list, system) → entity_instance | None
system.edit_system(file, system, attributes) → None
system.unassign_system(file, products: list, system) → None
system.remove_system(file, system) → None
system.add_port(file, element, ...) → entity_instance
system.connect_port(file, port1, port2, ...) → None
system.disconnect_port(file, port1, port2) → None
system.remove_port(file, port) → None
```

---

## 20. structural — Structural Analysis

```python
structural.add_structural_analysis_model(file) → entity_instance
structural.add_structural_member(file, structural_analysis_model, ...) → entity_instance
structural.add_structural_connection(file, ...) → entity_instance
structural.add_structural_load(file, ...) → entity_instance
structural.add_structural_load_case(file, ...) → entity_instance
structural.add_structural_load_group(file, ...) → entity_instance
structural.add_structural_boundary_condition(file, ...) → entity_instance
structural.edit_structural_analysis_model(file, ...) → None
structural.edit_structural_boundary_condition(file, ...) → None
structural.edit_structural_connection_cs(file, ...) → None
structural.edit_structural_item_axis(file, ...) → None
structural.edit_structural_load(file, ...) → None
structural.edit_structural_load_case(file, ...) → None
structural.assign_structural_analysis_model(file, ...) → entity_instance
structural.unassign_structural_analysis_model(file, ...) → None
structural.remove_structural_analysis_model(file, ...) → None
structural.remove_structural_connection_condition(file, ...) → None
structural.remove_structural_load(file, ...) → None
structural.remove_structural_load_case(file, ...) → None
structural.remove_structural_load_group(file, ...) → None
```

---

## 21-35. Additional Modules (Summary Signatures)

### nest
```python
nest.assign_object(file, related_objects: list, relating_object) → entity_instance | None
nest.unassign_object(file, related_objects: list) → None
```

### boundary
```python
boundary.assign_connection_geometry(file, rel_space_boundary, ...) → None
boundary.edit_boundary(file, boundary, ...) → None
boundary.remove_boundary(file, boundary) → None
boundary.copy_boundary(file, boundary) → entity_instance
```

### profile
```python
profile.add_parameterised_profile(file, ifc_class, ...) → entity_instance
profile.add_arbitrary_profile(file, profile, name=None) → entity_instance
profile.add_arbitrary_profile_with_voids(file, outer_profile, inner_profiles, ...) → entity_instance
profile.edit_profile(file, profile, attributes) → None
profile.remove_profile(file, profile) → None
```

### library
```python
library.add_library(file, name=None) → entity_instance
library.add_reference(file, library) → entity_instance
library.assign_reference(file, reference, products: list) → entity_instance | None
library.edit_library(file, library, attributes) → None
library.edit_reference(file, reference, attributes) → None
library.remove_library(file, library) → None
library.remove_reference(file, reference) → None
library.unassign_reference(file, reference, products: list) → None
```

### document
```python
document.add_information(file, parent=None) → entity_instance
document.add_reference(file, information) → entity_instance
document.assign_document(file, products: list, document) → entity_instance | None
document.edit_information(file, information, attributes) → None
document.edit_reference(file, reference, attributes) → None
document.remove_information(file, information) → None
document.remove_reference(file, reference) → None
document.unassign_document(file, products: list, document) → None
```

### constraint
```python
constraint.add_objective(file, name=None) → entity_instance
constraint.add_metric(file, objective) → entity_instance
constraint.add_metric_reference(file, metric, reference_path) → None
constraint.assign_constraint(file, products: list, constraint) → entity_instance | None
constraint.edit_metric(file, metric, attributes) → None
constraint.edit_objective(file, objective, attributes) → None
constraint.remove_constraint(file, constraint) → None
constraint.remove_metric(file, metric) → None
constraint.unassign_constraint(file, products: list, constraint) → None
```

### resource
```python
resource.add_resource(file, parent_resource=None, ifc_class="IfcCrewResource", name=None) → entity_instance
resource.add_resource_quantity(file, resource, ifc_class) → entity_instance
resource.add_resource_time(file, resource) → entity_instance
resource.assign_resource(file, relating_resource, related_object) → entity_instance
resource.calculate_resource_work(file, resource) → None
resource.edit_resource(file, resource, attributes) → None
resource.edit_resource_quantity(file, physical_quantity, attributes) → None
resource.edit_resource_time(file, resource_time, attributes) → None
resource.remove_resource(file, resource) → None
resource.remove_resource_quantity(file, resource) → None
resource.unassign_resource(file, relating_resource, related_object) → None
```

### drawing
```python
drawing.add_drawing(file, target_view, name=None) → entity_instance
drawing.add_annotation(file, drawing, ...) → entity_instance
drawing.edit_text_literal(file, ...) → None
drawing.remove_drawing(file, drawing) → None
```

### layer
```python
layer.add_layer(file, name=None) → entity_instance
layer.assign_layer(file, items: list, layer) → None
layer.edit_layer(file, layer, attributes) → None
layer.remove_layer(file, layer) → None
layer.unassign_layer(file, items: list, layer) → None
```

### control
```python
control.assign_control(file, relating_control, related_object) → entity_instance
control.unassign_control(file, relating_control, related_object) → None
```

### georeference
```python
georeference.add_georeferencing(file) → None
georeference.edit_georeferencing(file, ...) → None
georeference.edit_true_north(file, true_north) → None
georeference.remove_georeferencing(file) → None
georeference.set_wcs(file, x, y, z, e, n, h) → None
```

### grid
```python
grid.create_grid_axis(file, axis_tag, ...) → entity_instance
grid.create_axis_system(file, ...) → entity_instance
```

### feature
```python
feature.add_feature(file, element, feature) → entity_instance
feature.edit_feature(file, feature, attributes) → None
feature.remove_feature(file, feature) → None
```

### alignment
```python
alignment.add_alignment(file) → entity_instance
```

### cogo
```python
cogo.add_cogo_point(file, ...) → entity_instance
```

### pset_template
```python
pset_template.add_pset_template(file, ...) → entity_instance
pset_template.add_prop_template(file, ...) → entity_instance
pset_template.edit_pset_template(file, ...) → None
pset_template.edit_prop_template(file, ...) → None
pset_template.remove_pset_template(file, ...) → None
pset_template.remove_prop_template(file, ...) → None
```

---

## Sources

- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/root/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/spatial/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/geometry/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/type/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/pset/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/material/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/aggregate/index.html
- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/context/index.html
