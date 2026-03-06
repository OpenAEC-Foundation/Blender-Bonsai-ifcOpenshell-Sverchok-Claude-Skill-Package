# Bonsai Error Reproduction and Fix Examples

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11
> All examples run via `blender --background --python script.py` or from Blender's Python console.

---

## Example 1: Import Path Migration (§1.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — This FAILS with ModuleNotFoundError
from blenderbim.bim.ifc import IfcStore  # ModuleNotFoundError
import blenderbim.tool as tool           # ModuleNotFoundError
```

### Correct Fix

```python
# Bonsai v0.8.x — Correct import paths
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool

model = IfcStore.get_file()
if model is not None:
    print(f"Schema: {IfcStore.schema}")
```

### Version-Safe Import (Supporting Both Legacy and Current)

```python
# Bonsai v0.8.x — Version detection pattern
try:
    from bonsai.bim.ifc import IfcStore
    import bonsai.tool as tool
except ImportError:
    try:
        from blenderbim.bim.ifc import IfcStore
        import blenderbim.tool as tool
    except ImportError:
        raise RuntimeError("Neither Bonsai nor BlenderBIM is installed")
```

---

## Example 2: Guarding Against None IFC File (§2.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — This FAILS when no project is loaded
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
walls = model.by_type("IfcWall")  # AttributeError: 'NoneType' has no attribute 'by_type'
```

### Correct Fix

```python
# Bonsai v0.8.x — ALWAYS guard against None
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open or create an IFC project first.")

walls = model.by_type("IfcWall")
print(f"Found {len(walls)} walls")
```

---

## Example 3: Wrong Spatial Relationship (§3.2)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: using aggregate for physical elements
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
storey = model.by_type("IfcBuildingStorey")[0]
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# WRONG — aggregate is for spatial hierarchy (site->building->storey)
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=storey, products=[wall])
# This creates IfcRelAggregates instead of IfcRelContainedInSpatialStructure
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: use spatial.assign_container for physical elements
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
storey = model.by_type("IfcBuildingStorey")[0]
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# CORRECT — spatial containment for physical elements
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

---

## Example 4: Singular vs List Parameter (§3.4)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: passing single entity instead of list
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
storey = model.by_type("IfcBuildingStorey")[0]
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# WRONG — pre-v0.8 singular parameter style
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)  # KeyError: 'product'

# WRONG — singular related_object
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Type-001")
ifcopenshell.api.run("type.assign_type", model,
    related_object=wall, relating_type=wall_type)  # KeyError
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: ALWAYS use list parameters
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
storey = model.by_type("IfcBuildingStorey")[0]
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# CORRECT — list parameter
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# CORRECT — list parameter for type assignment
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Type-001")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

---

## Example 5: Deprecated void.add_opening (§2.4)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: void module does not exist
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
wall = model.by_type("IfcWall")[0]
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Opening-001")

# WRONG — void.add_opening does not exist in v0.8+
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)  # AttributeError
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: use feature.add_feature
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
wall = model.by_type("IfcWall")[0]
opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Opening-001")

# CORRECT — feature.add_feature replaces void.add_opening
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)
```

---

## Example 6: Placement Not Synced After Move (§4.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: IFC placement not updated
import bpy
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool

obj = bpy.context.active_object
entity = tool.Ifc.get_entity(obj)

# Move the object in Blender
obj.location = (5.0, 0.0, 3.0)

# IFC placement is STILL at the old location
# The .ifc file will save the OLD coordinates
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: sync placement after move
import bpy
from bonsai.bim.ifc import IfcStore

obj = bpy.context.active_object

# Move the object
obj.location = (5.0, 0.0, 3.0)

# ALWAYS sync IFC placement after transform changes
bpy.ops.bim.edit_object_placement()
```

---

## Example 7: Properties Not Written to IFC (§5.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: UI property change does not write to IFC
import bpy

obj = bpy.context.active_object
# Modifying a Blender UI property — this does NOT update IFC
# obj.PsetProperties["FireRating"] = "REI60"  # Only changes UI, not IFC
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: use ifcopenshell.api to write properties
import ifcopenshell.api
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded")

obj = bpy.context.active_object
entity = tool.Ifc.get_entity(obj)
if entity is None:
    raise RuntimeError("Object has no IFC assignment")

# Create or get pset
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=entity, name="Pset_WallCommon")

# Write property to IFC
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset,
    properties={"FireRating": "REI60"})

# Purge UI cache so Bonsai panels reflect the change
from bonsai.bim.module.pset.data import PsetData
PsetData.purge()
```

---

## Example 8: Missing Representation Context (§2.3)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: adding geometry without context
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# WRONG — no context parameter, or context does not exist
representation = ifcopenshell.api.run(
    "geometry.add_wall_representation", model,
    context=None,  # Fails — context is required
    length=5.0, height=3.0, thickness=0.2)
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: verify/create context before geometry
import ifcopenshell.api
import ifcopenshell.util.representation
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Step 1: Verify or create representation context
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

# Step 2: Create geometry with valid context
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")
representation = ifcopenshell.api.run(
    "geometry.add_wall_representation", model,
    context=body_context,
    length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
```

---

## Example 9: Creating Project Without API (§3.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: creating IFC file without proper setup
import ifcopenshell

# WRONG — creates empty file without IfcProject, units, or contexts
model = ifcopenshell.file(schema="IFC4")
model.write("project.ifc")
# This file is INVALID per IFC specification
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: use project creation API
import ifcopenshell
import ifcopenshell.api

# Step 1: Create file with proper project structure
model = ifcopenshell.api.project.create_file(version="IFC4")

# Step 2: Assign units (REQUIRED)
ifcopenshell.api.unit.assign_unit(model)

# Step 3: Create spatial hierarchy
project = model.by_type("IfcProject")[0]
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="Default Site")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Default Building")
storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey])

# Step 4: Create representation contexts
model_ctx = ifcopenshell.api.context.add_context(model,
    context_type="Model")
ifcopenshell.api.context.add_context(model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model_ctx)

# Step 5: Save
model.write("project.ifc")
```

---

## Example 10: Drawing Generation With Missing Prerequisites (§7.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: creating drawing without prerequisites
import bpy

# WRONG — no active drawing camera
bpy.ops.bim.create_drawing()
# RuntimeError: Operator bim.create_drawing failed, context is incorrect
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: verify all drawing prerequisites
import bpy
from bonsai.bim.ifc import IfcStore

# Step 1: Verify IFC file is loaded
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Step 2: Create a drawing (sets up camera)
bpy.ops.bim.add_drawing(target_view="PLAN_VIEW", location_hint=0)

# Step 3: Generate SVG with sync enabled
bpy.ops.bim.create_drawing(sync=True, print_all=False)
```

---

## Example 11: Cache Purge After Direct API Modification (§10.1)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: modifying IFC without cache purge
import ifcopenshell.api
import bonsai.tool as tool

model = tool.Ifc.get()
entity = tool.Ifc.get_entity(bpy.context.active_object)

# Direct API modification
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset_entity,
    properties={"Status": "Demolished"})

# Bonsai UI panels still show OLD property values
# because module data cache is stale
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: purge cache after direct API modification
import ifcopenshell.api
import bonsai.tool as tool

model = tool.Ifc.get()
entity = tool.Ifc.get_entity(bpy.context.active_object)

# Direct API modification
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset_entity,
    properties={"Status": "Demolished"})

# Purge relevant data cache
from bonsai.bim.module.pset.data import PsetData
PsetData.purge()

# Now UI panels reflect the updated property
```

---

## Example 12: QTO Weight Calculation Missing Density (§5.3)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: weight calculation without material density
import bpy

# Select a wall object
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="GrossWeight",
    calculator_function="get_gross_weight")
# Returns None — no MassDensity assigned
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: assign material density first
import ifcopenshell.api
import bonsai.tool as tool

model = tool.Ifc.get()
entity = tool.Ifc.get_entity(bpy.context.active_object)

# Step 1: Get the material
import ifcopenshell.util.element
material = ifcopenshell.util.element.get_material(entity)

if material and material.is_a("IfcMaterial"):
    # Step 2: Create/edit Pset_MaterialCommon with MassDensity
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=material, name="Pset_MaterialCommon")
    ifcopenshell.api.run("pset.edit_pset", model,
        pset=pset,
        properties={"MassDensity": 2400.0})  # kg/m³ for concrete

# Step 3: Now weight calculation works
bpy.ops.bim.calculate_single_quantity(
    calculator="Blender",
    qto_name="Qto_WallBaseQuantities",
    prop_name="GrossWeight",
    calculator_function="get_gross_weight")
```

---

## Example 13: Clash Detection on Unsaved File (§9.2)

### Reproducing the Error

```python
# Bonsai v0.8.x — WRONG: running clash detection without saving
import bpy

# Modify geometry in Blender...
# Then immediately run clash detection
bpy.ops.bim.execute_ifc_clash()
# Clash detection reads from DISK — returns stale results
```

### Correct Fix

```python
# Bonsai v0.8.x — CORRECT: save before clash detection
import bpy

# Step 1: ALWAYS save IFC file first
bpy.ops.bim.save_project()

# Step 2: Run clash detection on saved file
bpy.ops.bim.execute_ifc_clash()
```
