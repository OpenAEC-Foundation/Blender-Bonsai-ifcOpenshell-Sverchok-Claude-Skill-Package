# IFC Model Creation — Anti-Patterns

Common mistakes when creating IFC models with IfcOpenShell. Each entry explains WHAT is wrong and WHY it fails.

---

## Anti-Pattern 1: Using `model.create_entity()` Instead of `ifcopenshell.api.run()`

```python
# WRONG — bypasses validation, GlobalId, and OwnerHistory
wall = model.create_entity("IfcWall", Name="Wall 001")
```

**WHY:** `model.create_entity()` creates a raw entity without generating a GlobalId, without attaching OwnerHistory (required in IFC2X3), and without validating predefined types against the schema. The resulting entity may be malformed and rejected by BIM viewers or IFC validators. `ifcopenshell.api.run("root.create_entity", ...)` handles all of these automatically.

```python
# CORRECT — use the API for safe entity creation
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
```

---

## Anti-Pattern 2: Creating Elements Without Spatial Containment

```python
# WRONG — element has no spatial container
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Floating Wall")
# Missing: spatial.assign_container
model.write("output.ifc")
```

**WHY:** Physical elements (walls, slabs, columns, doors) without spatial containment via `IfcRelContainedInSpatialStructure` are invisible in most BIM viewers. Viewers like BIMcollab, Solibri, and Navisworks render elements based on their spatial context. Without containment, the element exists in the file but has no spatial reference, making it unfindable in model trees and often hidden from 3D views.

```python
# CORRECT — assign container after creating element
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

---

## Anti-Pattern 3: Skipping `geometry.edit_object_placement` After Element Creation

```python
# WRONG — element has no IfcLocalPlacement
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=rep)
# Missing: geometry.edit_object_placement
```

**WHY:** Without a placement, the element has no defined position in 3D space. Some viewers may place it at the world origin; others may fail to display it entirely. The IFC schema expects every product to have an `ObjectPlacement`. Calling `geometry.edit_object_placement` (even without a matrix argument, which defaults to identity/origin) creates the required `IfcLocalPlacement` entity.

```python
# CORRECT — always set placement (identity if origin is intended)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
```

---

## Anti-Pattern 4: Creating Geometry Before Setting Up Contexts

```python
# WRONG — no context exists yet
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Trying to create geometry without a Body context
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None, length=5.0, height=3.0, thickness=0.2)  # Error or invalid
```

**WHY:** All geometry in IFC requires an `IfcGeometricRepresentationSubContext` (like "Body" within "Model"). The `context` parameter tells the geometry function which representation context to create the shape in. Without first creating the Model context and Body subcontext, there is no valid target. Passing `None` either raises an error or creates orphaned geometry that viewers cannot display.

```python
# CORRECT — create contexts before geometry
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)
# Now create geometry with valid context
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

---

## Anti-Pattern 5: Skipping `owner.set_user` in IFC2X3

```python
# WRONG — IFC2X3 requires OwnerHistory on all rooted entities
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")  # OwnerHistory is NULL → schema violation
```

**WHY:** The IFC2X3 schema mandates OwnerHistory as a REQUIRED attribute on every rooted entity (IfcRoot subclass). The `root.create_entity` API attaches OwnerHistory from the currently active user, but only if a user has been set via `owner.set_user`. Without it, OwnerHistory is NULL, producing a file that violates the IFC2X3 schema and fails validation. IFC4 makes OwnerHistory optional, so this issue is specific to IFC2X3.

```python
# CORRECT — set up owner before creating any entities in IFC2X3
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
person = ifcopenshell.api.run("owner.add_person", model,
    identification="jdoe", family_name="Doe", given_name="John")
org = ifcopenshell.api.run("owner.add_organisation", model,
    identification="ACME", name="ACME Engineering")
user = ifcopenshell.api.run("owner.add_person_and_organisation", model,
    person=person, organisation=org)
ifcopenshell.api.run("owner.add_application", model)
ifcopenshell.api.run("owner.set_user", model, user=user)
# Now entities get valid OwnerHistory
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
```

---

## Anti-Pattern 6: Skipping Unit Assignment

```python
# WRONG — no units assigned
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
# Missing: unit.assign_unit
```

**WHY:** Without unit assignment, all numeric values in the model have no defined scale. A wall with `length=5.0` could be 5 meters, 5 feet, or 5 millimeters — viewers and downstream tools cannot determine which. Most BIM viewers default to meters, but this assumption is unreliable. Calling `unit.assign_unit` without arguments assigns SI defaults (meters, square meters, cubic meters, radians), making all dimensions unambiguous.

```python
# CORRECT — always assign units immediately after project creation
ifcopenshell.api.run("unit.assign_unit", model)
```

---

## Anti-Pattern 7: Mismatching Type and Occurrence Classes

```python
# WRONG — IfcWall occurrence cannot use IfcSlabType
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
slab_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlabType", name="Floor Slab Type")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=slab_type)  # Schema violation
```

**WHY:** IFC requires type-occurrence class consistency. An IfcWall must be typed by an IfcWallType, an IfcSlab by an IfcSlabType, and so on. Assigning a mismatched type creates an invalid model that fails schema validation. The mapping is defined in the IFC schema: each element class has a specific corresponding type class.

```python
# CORRECT — match type class to occurrence class
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

---

## Anti-Pattern 8: Using `ifcopenshell.file()` for Production Models

```python
# WRONG — bare file has no header, no project, no units
model = ifcopenshell.file(schema="IFC4")
```

**WHY:** `ifcopenshell.file()` creates a bare IFC file without header metadata (timestamps, author, application), without an IfcProject entity, and without unit definitions. While useful for unit tests or minimal experiments, production files created this way lack required metadata and fail most validation checks. `project.create_file` pre-populates the file header with timestamps, application info, and MVD settings.

```python
# CORRECT — use project.create_file for production models
model = ifcopenshell.api.run("project.create_file", version="IFC4")
```

---

## Anti-Pattern 9: Building Spatial Hierarchy in Wrong Order

```python
# WRONG — elements created before spatial hierarchy
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
# Cannot assign container — no storey exists yet!
```

**WHY:** Physical elements require a spatial container (IfcBuildingStorey, IfcSpace, etc.) to be visible and properly structured. If you create elements before the spatial hierarchy (Project → Site → Building → Storey), there is nothing to contain them. The correct creation order is: file → project → units → contexts → spatial hierarchy → types → physical elements. Following this order ensures that containers exist before elements are assigned to them.

```python
# CORRECT — build spatial hierarchy first, then create elements
# 1. file  2. project  3. units  4. contexts
# 5. site → building → storey
# 6. types (optional)  7. elements  8. properties
```
