# Geometry Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+
> These anti-patterns are verified against Bonsai/IfcOpenShell source code and real error scenarios.

---

## Anti-Pattern 1: Creating IfcExtrudedAreaSolid via model.create_entity()

### Wrong

```python
# WRONG: Bypasses unit conversion, context assignment, and RepresentationType setup
solid = model.create_entity("IfcExtrudedAreaSolid",
    SweptArea=profile,
    ExtrudedDirection=model.createIfcDirection((0., 0., 1.)),
    Depth=3.0)
shape_rep = model.create_entity("IfcShapeRepresentation",
    ContextOfItems=body,
    RepresentationIdentifier="Body",
    RepresentationType="SweptSolid",
    Items=[solid])
```

### Why It Fails

- No unit conversion: `Depth=3.0` is ambiguous (meters? millimeters?) — API functions handle this automatically.
- No `RepresentationType` validation: manually setting the wrong type produces invalid IFC files.
- No profile normalization: `IfcIndexedPolyCurve` vs `IfcPolyline` selection depends on schema version — API handles this.
- No ownership/history tracking: `ifcopenshell.api` functions manage `IfcOwnerHistory` automatically.

### Correct

```python
# Option A: Use geometry API
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)

# Option B: Use ShapeBuilder
from ifcopenshell.util.shape_builder import ShapeBuilder
builder = ShapeBuilder(model)
solid = builder.extrude(profile, magnitude=3.0)
representation = builder.get_representation(body, [solid])
```

---

## Anti-Pattern 2: Moving Blender Objects Without Syncing IFC Placement

### Wrong

```python
# WRONG: Blender transform changes, but IfcLocalPlacement is NOT updated
import bpy
obj = bpy.context.active_object
obj.location = (5.0, 3.0, 0.0)
# IFC entity still thinks the element is at its original position!
```

### Why It Fails

- Bonsai maintains a bidirectional mapping between Blender objects and IFC entities.
- `obj.location` and `obj.matrix_world` modify the Blender scene only.
- `IfcLocalPlacement` (the IFC positioning entity) is NOT automatically synced.
- On next file save or reload, the Blender position is overwritten by the (stale) IFC placement.

### Correct

```python
# Inside Bonsai (Blender context)
import bpy
obj = bpy.context.active_object
obj.location = (5.0, 3.0, 0.0)
bpy.ops.bim.edit_object_placement()  # Syncs Blender transform → IFC

# Standalone IfcOpenShell
import numpy
matrix = numpy.eye(4)
matrix[0][3] = 5.0  # X
matrix[1][3] = 3.0  # Y
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix, is_si=True)
```

---

## Anti-Pattern 3: Removing Representation Without Unassigning First

### Wrong

```python
# WRONG: Representation is still assigned to product
ifcopenshell.api.run("geometry.remove_representation", model,
    representation=representation)
# Leaves dangling reference in IfcProductDefinitionShape.Representations
```

### Why It Fails

- `remove_representation()` deep-removes the entity and its children (items, styles, textures).
- If the representation is still referenced in `IfcProductDefinitionShape.Representations` or an `IfcRepresentationMap`, the IFC file has dangling references.
- Bonsai's core layer (`core.geometry.remove_representation`) handles this correctly, but direct API use must follow the order manually.

### Correct

```python
# Step 1: Unassign from product
ifcopenshell.api.run("geometry.unassign_representation", model,
    product=element, representation=representation)
# Step 2: Remove the representation
ifcopenshell.api.run("geometry.remove_representation", model,
    representation=representation)
```

---

## Anti-Pattern 4: Using void.add_opening() Instead of feature.add_feature()

### Wrong

```python
# WRONG for Bonsai v0.8.0+: void.add_opening() is superseded
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)
```

### Why It Fails

- In Bonsai v0.8.0+, `feature.add_feature()` is the unified API for all feature types: subtractions (openings), additions (projections), and surface features.
- `void.add_opening()` still works in standalone IfcOpenShell but does not trigger Bonsai's internal update hooks (mesh regeneration, opening visualization).
- Using the old API in Bonsai context may cause desynchronization between the IFC graph and the Blender viewport.

### Correct

```python
# Bonsai v0.8.0+ and standalone IfcOpenShell
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Window Opening")
# Add geometry to opening first (see examples.md Example 7)
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)
```

---

## Anti-Pattern 5: Flat Vertex List in add_mesh_representation()

### Wrong

```python
# WRONG: vertices as flat list of tuples instead of list-of-lists
vertices = [(0.,0.,0.), (1.,0.,0.), (1.,1.,0.), (0.,1.,0.)]
faces = [(0,1,2,3)]
ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body, vertices=vertices, faces=faces)
# TypeError or produces malformed geometry
```

### Why It Fails

- `vertices` and `faces` expect **parallel lists-of-lists**: each outer element is one `IfcRepresentationItem`.
- A flat list of tuples is interpreted as multiple items with single vertices, producing empty or degenerate geometry.

### Correct

```python
# Wrap in outer list: one item with all vertices
vertices = [[(0.,0.,0.), (1.,0.,0.), (1.,1.,0.), (0.,1.,0.),
             (0.,0.,1.), (1.,0.,1.), (1.,1.,1.), (0.,1.,1.)]]
faces = [[(0,1,2,3), (4,5,6,7), (0,1,5,4), (1,2,6,5), (2,3,7,6), (3,0,4,7)]]
representation = ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body, vertices=vertices, faces=faces)
```

---

## Anti-Pattern 6: Missing Body Context Before Adding Geometry

### Wrong

```python
# WRONG: No context exists, pass None
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None, length=5.0, height=3.0, thickness=0.2)
# AttributeError: 'NoneType' object has no attribute ...
```

### Why It Fails

- Every `IfcShapeRepresentation` requires a `ContextOfItems` reference to an `IfcGeometricRepresentationSubContext`.
- Without a context, the API function cannot create a valid representation.
- This is the most common error for new users who skip the project bootstrap sequence.

### Correct

```python
import ifcopenshell.util.representation

# ALWAYS check and create context before using geometry functions
body = ifcopenshell.util.representation.get_context(model, "Model", "Body", "MODEL_VIEW")
if not body:
    model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
    body = ifcopenshell.api.run("context.add_context", model,
        context_type="Model", context_identifier="Body",
        target_view="MODEL_VIEW", parent=model3d)

representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

---

## Anti-Pattern 7: Assigning Geometry to Occurrences Instead of Types

### Wrong

```python
# WRONG: Adding geometry to each occurrence individually
for col in columns:
    repr = ifcopenshell.api.run("geometry.add_profile_representation", model,
        context=body, profile=profile, depth=3.0)
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=col, representation=repr)
# Creates N duplicate representations instead of sharing via MappedItem
```

### Why It Fails

- IFC uses `IfcMappedItem` to share geometry efficiently across type instances.
- Assigning geometry to each occurrence separately creates redundant data, bloats file size, and breaks the type-occurrence relationship.
- Updating geometry requires changing every occurrence individually instead of once on the type.

### Correct

```python
# Assign geometry to TYPE once
col_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumnType", name="Standard Column")
repr = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=col_type, representation=repr)

# Assign type to occurrences — geometry is automatically shared via MappedItem
for col in columns:
    ifcopenshell.api.run("type.assign_type", model,
        related_objects=[col], relating_type=col_type)
```

---

## Anti-Pattern 8: Setting product.Representation Directly

### Wrong

```python
# WRONG: Direct attribute assignment bypasses relationship management
wall.Representation = model.create_entity("IfcProductDefinitionShape",
    Representations=[representation])
```

### Why It Fails

- `geometry.assign_representation` handles `IfcProductDefinitionShape` creation, `IfcRepresentationMap` setup (for types), and `IfcMappedItem` propagation to instances.
- Direct assignment skips all of this, leaving orphaned entities and broken type-instance geometry sharing.
- For `IfcTypeProduct`, direct assignment does NOT create `IfcMappedItem` references on instances.

### Correct

```python
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
```

---

## Anti-Pattern 9: Forgetting to Close Arbitrary Profiles

### Wrong

```python
# WRONG: Profile outline is not closed (first point != last point)
outline = [(0.0, 0.0), (1.0, 0.0), (1.0, 0.5), (0.0, 0.5)]
profile = ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile=outline)
# Produces open curve — invalid for IfcArbitraryClosedProfileDef
```

### Why It Fails

- `IfcArbitraryClosedProfileDef` requires a closed curve as its `OuterCurve`.
- An open polyline produces an invalid profile that may cause geometry processing failures in IFC viewers or downstream analysis tools.

### Correct

```python
# Close the profile: first and last points MUST be identical
outline = [(0.0, 0.0), (1.0, 0.0), (1.0, 0.5), (0.0, 0.5), (0.0, 0.0)]
profile = ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile=outline)
```
