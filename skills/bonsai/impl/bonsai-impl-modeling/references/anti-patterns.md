# Bonsai Modeling — Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x
> Every anti-pattern includes: what goes wrong, why it fails, and the correct approach.

---

## Anti-Pattern 1: Using `void.add_opening` Instead of `feature.add_feature`

### Wrong

```python
ifcopenshell.api.run("void.add_opening", model,
    opening=opening_element,
    element=wall_element)
# ModuleNotFoundError: No module named 'ifcopenshell.api.void.add_opening'
```

### Why It Fails

The `void.add_opening` function does NOT exist in IfcOpenShell v0.8.0+. The `void` module was restructured. Opening creation is now handled by the `feature` module, which supports all feature types (voids, projections, surface features).

### Correct

```python
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening_element,
    element=wall_element)
```

**Rule**: ALWAYS use `feature.add_feature()` for voids. NEVER use `void.add_opening`.

---

## Anti-Pattern 2: Passing Single Entity Instead of List to `type.assign_type`

### Wrong

```python
ifcopenshell.api.run("type.assign_type", model,
    relating_type=wall_type,
    related_object=wall_instance)   # WRONG: single entity, wrong parameter name
```

### Why It Fails

The parameter is named `related_objects` (plural) and expects a Python list. Passing a single entity causes a `TypeError` or silently fails because the API iterates over the entity's attributes instead of treating it as a product list.

### Correct

```python
ifcopenshell.api.run("type.assign_type", model,
    relating_type=wall_type,
    related_objects=[wall_instance])  # ALWAYS a list, even for one element
```

**Rule**: ALWAYS wrap elements in a list for `related_objects`, `products`, and similar parameters.

---

## Anti-Pattern 3: Using `aggregate.assign_object` for Element Containment

### Wrong

```python
# Trying to place a wall inside a storey using aggregation
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=storey,
    product=wall)
```

### Why It Fails

`IfcRelAggregates` is for spatial decomposition hierarchy (Project → Site → Building → Storey). Physical elements like walls, columns, and slabs are placed inside spatial elements via `IfcRelContainedInSpatialStructure`. Using aggregation creates an invalid IFC structure that fails schema validation.

### Correct

```python
# For physical elements in a spatial container:
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall],
    relating_structure=storey)

# For spatial hierarchy (storey in building):
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey],
    relating_object=building)
```

**Rule**: ALWAYS use `spatial.assign_container` for placing physical elements. Use `aggregate.assign_object` ONLY for spatial hierarchy decomposition.

---

## Anti-Pattern 4: Assigning Materials to Occurrences Instead of Types

### Wrong

```python
# Assigning layer set directly to the wall occurrence
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_occurrence],
    material=layer_set)
```

### Why It Fails

In IFC, material composition (layer sets, profile sets, constituent sets) is defined on the TYPE, not on individual occurrences. Occurrences inherit materials from their type. When materials are assigned directly to occurrences instead of types:
- `IfcMaterialLayerSetUsage` and `IfcMaterialProfileSetUsage` are not generated
- Type-based material inheritance breaks
- Parametric geometry regeneration fails in Bonsai

### Correct

```python
# Assign to TYPE (occurrences inherit automatically)
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type],        # TYPE, not occurrence
    material=layer_set)

# Occurrences get IfcMaterialLayerSetUsage via type.assign_type
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall_occurrence],
    relating_type=wall_type,
    should_map_representations=True)
```

**Rule**: ALWAYS assign material sets to types. Occurrences inherit via `type.assign_type`.

---

## Anti-Pattern 5: Creating Geometry Without Verifying Representation Context

### Wrong

```python
# Directly creating geometry without checking context
wall_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None,  # or missing context parameter
    length=5.0, height=3.0, thickness=0.2)
# TypeError or RuntimeError
```

### Why It Fails

Every geometric representation in IFC requires a valid `IfcGeometricRepresentationContext` (or sub-context). A new IFC file may not have a Body sub-context created yet. Attempting to create geometry without a valid context raises an error or produces orphaned geometry.

### Correct

```python
import ifcopenshell.util.representation

# ALWAYS verify or create context first
body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

if not body_context:
    model_ctx = ifcopenshell.api.run("context.add_context", model,
        context_type="Model")
    body_context = ifcopenshell.api.run("context.add_context", model,
        context_type="Model",
        context_identifier="Body",
        target_view="MODEL_VIEW",
        parent=model_ctx)

# Now safe to create geometry
wall_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context,
    length=5.0, height=3.0, thickness=0.2)
```

**Rule**: ALWAYS verify representation context exists before creating geometry.

---

## Anti-Pattern 6: Using `blenderbim.*` Import Paths

### Wrong

```python
from blenderbim.bim.ifc import IfcStore
import blenderbim.tool as tool
import blenderbim.core.spatial
# ModuleNotFoundError: No module named 'blenderbim'
```

### Why It Fails

Bonsai was renamed from BlenderBIM in 2024 (v0.8.0). All module paths changed from `blenderbim.*` to `bonsai.*`. Code using the old import paths fails with `ModuleNotFoundError` in Bonsai v0.8.0+.

### Correct

```python
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
import bonsai.core.spatial
```

| Old (pre-v0.8.0) | New (v0.8.0+) |
|-------------------|---------------|
| `blenderbim.bim.ifc` | `bonsai.bim.ifc` |
| `blenderbim.tool` | `bonsai.tool` |
| `blenderbim.core` | `bonsai.core` |
| `blenderbim.bim.module.*` | `bonsai.bim.module.*` |

**Rule**: ALWAYS use `bonsai.*`. NEVER use `blenderbim.*`.

---

## Anti-Pattern 7: Forgetting `IfcStore.get_file()` None Check

### Wrong

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.file  # Direct attribute access
walls = model.by_type("IfcWall")
# AttributeError: 'NoneType' object has no attribute 'by_type'
```

### Why It Fails

If no IFC project is loaded in Blender, `IfcStore.file` is `None`. Accessing `.by_type()` on `None` raises `AttributeError`. This is the most common runtime error in Bonsai scripts.

### Correct

```python
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()  # Use the method, not the attribute
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")
walls = model.by_type("IfcWall")
```

**Rule**: ALWAYS use `IfcStore.get_file()` and check for `None`.

---

## Anti-Pattern 8: Moving Blender Objects Without Syncing IFC Placement

### Wrong

```python
import bpy

obj = bpy.context.active_object
obj.location = (5.0, 0.0, 0.0)
# Blender object moved, but IFC IfcLocalPlacement is NOT updated
```

### Why It Fails

Bonsai maintains IFC placement data separately from Blender's transform. Moving a Blender object via `obj.location`, `obj.matrix_world`, or the G key only changes the Blender scene. The IFC `IfcLocalPlacement` remains at the old position. When the file is saved, the element appears at its original IFC position.

### Correct

```python
import bpy

obj = bpy.context.active_object
obj.location = (5.0, 0.0, 0.0)
bpy.ops.bim.edit_object_placement()  # Sync to IFC

# Or use the ifcopenshell API directly:
import numpy as np
import ifcopenshell.api
matrix = np.eye(4)
matrix[0, 3] = 5.0
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=element, matrix=matrix, is_si=True)
```

**Rule**: ALWAYS sync IFC placement after any Blender transform change.

---

## Anti-Pattern 9: Creating Material Layer Without Setting Thickness

### Wrong

```python
layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)
# Layer created with LayerThickness = None or 0.0
# Geometry regeneration produces zero-thickness layer
```

### Why It Fails

`material.add_layer` creates an `IfcMaterialLayer` with no thickness by default. Without calling `material.edit_layer` to set `LayerThickness`, the layer has zero thickness, producing degenerate geometry that causes rendering errors and invalid IFC output.

### Correct

```python
layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=concrete)

# ALWAYS set thickness immediately after adding a layer
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer,
    attributes={"LayerThickness": 0.200})  # 200mm
```

**Rule**: ALWAYS call `material.edit_layer` with `LayerThickness` immediately after `material.add_layer`.

---

## Anti-Pattern 10: Using `bpy.ops.bim.*` Instead of `ifcopenshell.api.run()` in Scripts

### Wrong

```python
import bpy
# Using operators in scripts — fragile, context-dependent
bpy.ops.bim.assign_class(ifc_class="IfcWall")
bpy.ops.bim.assign_container(structure=storey_id)
```

### Why It Fails

Bonsai operators (`bpy.ops.bim.*`) depend on Blender's context state (active object, selected objects, mode). They are designed for UI interaction, not scripted workflows. They:
- Require specific context overrides
- May change names between versions
- Have implicit side effects (scene updates, undo stack)
- Are harder to test and debug

### Correct

```python
import ifcopenshell.api

# Direct API calls — stable, explicit, context-independent
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

**Rule**: ALWAYS prefer `ifcopenshell.api.run()` over `bpy.ops.bim.*` in scripts. Use operators only for operations that have no API equivalent (e.g., `bpy.ops.bim.edit_object_placement` for Blender-side sync).
