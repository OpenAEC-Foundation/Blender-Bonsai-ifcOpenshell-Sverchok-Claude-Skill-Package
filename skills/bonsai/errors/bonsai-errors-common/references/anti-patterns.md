# Anti-Patterns That Cause Bonsai Errors

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11
> Each entry describes a pattern that causes errors, explains why, and provides the correct approach.

---

## AP-01: Using `blenderbim.*` Import Paths

**Anti-Pattern**:
```python
from blenderbim.bim.ifc import IfcStore
import blenderbim.tool as tool
import blenderbim.core.spatial
```

**Why It Fails**: Bonsai was renamed from BlenderBIM in September 2024 (v0.8.0). All Python module paths changed. The old `blenderbim.*` paths produce `ModuleNotFoundError` on any Bonsai v0.8.0+ installation.

**Correct Approach**:
```python
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
import bonsai.core.spatial
```

**Rule**: ALWAYS use `bonsai.*`. NEVER use `blenderbim.*`. Operator `bl_idname` values (`bim.*`) did NOT change in the rename.

---

## AP-02: Running Bonsai Scripts From System Python

**Anti-Pattern**:
```bash
python3 my_bonsai_script.py
pip install bonsai  # This is NOT the Bonsai BIM addon
```

**Why It Fails**: `bonsai.*` and `bpy` are ONLY available inside Blender's bundled Python environment. System Python cannot import them. The `bonsai` package on PyPI is unrelated to Bonsai BIM.

**Correct Approach**:
```bash
blender --background --python my_bonsai_script.py
```

For headless IFC operations without Blender, use `ifcopenshell` directly (it IS installable via pip).

**Rule**: ALWAYS run Bonsai scripts via `blender --python`. NEVER import `bonsai.*` from system Python.

---

## AP-03: Not Guarding `IfcStore.get_file()` for None

**Anti-Pattern**:
```python
model = IfcStore.get_file()
walls = model.by_type("IfcWall")  # Crashes if no project loaded
```

**Why It Fails**: `IfcStore.get_file()` returns `None` when no IFC project is loaded. Calling methods on `None` raises `AttributeError`.

**Correct Approach**:
```python
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded.")
walls = model.by_type("IfcWall")
```

**Rule**: ALWAYS check for `None` before any IFC operation. This applies to both `IfcStore.get_file()` and `tool.Ifc.get()`.

---

## AP-04: Using `void.add_opening` Instead of `feature.add_feature`

**Anti-Pattern**:
```python
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)
```

**Why It Fails**: The `void` API module was renamed to `feature` in IfcOpenShell v0.8.0. The function signature also changed: `opening` became `feature`.

**Correct Approach**:
```python
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)
```

**Rule**: ALWAYS use `feature.add_feature()`. NEVER use `void.add_opening()`.

---

## AP-05: Confusing Aggregation With Spatial Containment

**Anti-Pattern**:
```python
# Using aggregate to place a wall in a storey
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=storey, products=[wall])
```

**Why It Fails**: `aggregate.assign_object` creates `IfcRelAggregates`, which is for spatial decomposition (Project→Site→Building→Storey). Physical elements (walls, columns, slabs) MUST use `IfcRelContainedInSpatialStructure` via `spatial.assign_container`.

Using aggregation for physical elements creates an invalid IFC hierarchy that fails validation and causes downstream issues in BIM tools.

**Correct Approach**:
```python
# For physical elements → spatial containment
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# For spatial hierarchy → aggregation
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])
```

**Rule**: ALWAYS use `spatial.assign_container` for physical elements. ALWAYS use `aggregate.assign_object` for spatial hierarchy only.

---

## AP-06: Passing Singular Entities Instead of Lists (v0.8+)

**Anti-Pattern**:
```python
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)

ifcopenshell.api.run("type.assign_type", model,
    related_object=wall, relating_type=wall_type)
```

**Why It Fails**: IfcOpenShell v0.8+ changed relationship API parameters from singular (`product`, `related_object`) to plural lists (`products`, `related_objects`). The old parameter names raise `KeyError` or `TypeError`.

**Correct Approach**:
```python
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

**Rule**: ALWAYS wrap entities in lists for `products` and `related_objects` parameters, even for single entities.

---

## AP-07: Moving Objects Without Syncing IFC Placement

**Anti-Pattern**:
```python
obj.location = (5.0, 0.0, 3.0)
# Done — IFC placement is NOT updated
```

**Why It Fails**: Blender's transform system and Bonsai's IFC placement system are independent. Moving a Blender object with `obj.location`, `obj.matrix_world`, or the `G` key does NOT update `IfcLocalPlacement` in the IFC model. The saved IFC file retains the original coordinates.

**Correct Approach**:
```python
obj.location = (5.0, 0.0, 3.0)
bpy.ops.bim.edit_object_placement()
```

**Rule**: ALWAYS call `bpy.ops.bim.edit_object_placement()` after any direct matrix/location change.

---

## AP-08: Creating IFC Files With `ifcopenshell.file()` Directly

**Anti-Pattern**:
```python
model = ifcopenshell.file(schema="IFC4")
# No IfcProject, no units, no contexts
model.write("project.ifc")
```

**Why It Fails**: `ifcopenshell.file()` creates a bare, empty IFC file. It lacks the mandatory `IfcProject`, `IfcUnitAssignment`, and `IfcGeometricRepresentationContext`. This file is invalid per ALL IFC schemas and causes errors in BIM tools.

**Correct Approach**:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")
ifcopenshell.api.unit.assign_unit(model)
```

**Rule**: ALWAYS use `ifcopenshell.api.project.create_file()`. NEVER use `ifcopenshell.file()` for new projects.

---

## AP-09: Assuming UI Property Changes Write to IFC

**Anti-Pattern**:
```python
# Changing a Blender property and expecting IFC to update
obj.BIMObjectProperties.some_property = "new_value"
# Or editing a property panel field in the UI
```

**Why It Fails**: Blender properties and IFC data are separate systems. Modifying a Blender property only changes the UI display. The IFC model is only updated when changes go through `ifcopenshell.api.run("pset.edit_pset", ...)` or via the corresponding Bonsai operator that commits the change.

**Correct Approach**:
```python
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=entity, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"FireRating": "REI60"})
```

**Rule**: ALWAYS use `ifcopenshell.api.run("pset.edit_pset", ...)` to write properties to IFC.

---

## AP-10: Modifying IFC Via API Without Purging Caches

**Anti-Pattern**:
```python
# Direct API modification without cache management
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"Status": "Demolished"})
# Bonsai UI panels still show old values
```

**Why It Fails**: Each Bonsai module maintains a data cache (e.g., `PsetData`, `DrawingData`, `SpatialData`). Direct `ifcopenshell.api` calls bypass the cache update mechanism. The UI displays stale cached data until the cache is manually purged.

**Correct Approach**:
```python
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"Status": "Demolished"})

# Purge the relevant cache
from bonsai.bim.module.pset.data import PsetData
PsetData.purge()
```

**Rule**: ALWAYS purge the relevant module data cache after direct `ifcopenshell.api` modifications.

---

## AP-11: Caching Entity References Across Operations

**Anti-Pattern**:
```python
# Store entity reference for later use
wall_entity = model.by_type("IfcWall")[0]

# ... much later, possibly after undo/redo or file reload ...
print(wall_entity.Name)  # May crash or return wrong data
```

**Why It Fails**: Entity references become stale after:
- File reload (`bpy.ops.bim.load_project`)
- Undo/redo operations
- External file modifications
- Model regeneration

Stale references produce `RuntimeError` or silently return incorrect data.

**Correct Approach**:
```python
# Store the stable identifier (step ID or GlobalId)
wall_id = wall_entity.id()
wall_guid = wall_entity.GlobalId

# Re-query when needed
model = IfcStore.get_file()
wall_entity = model.by_id(wall_id)
```

**Rule**: NEVER cache entity references across operations. ALWAYS re-query by ID or GlobalId.

---

## AP-12: Running QTO on Non-Mesh Objects

**Anti-Pattern**:
```python
# Selecting empties, curves, or lights and running QTO
bpy.ops.bim.perform_quantity_take_off(qto_rule="Qto_WallBaseQuantities")
```

**Why It Fails**: QTO calculator functions require `bpy.types.Mesh` data. Empties, curves, cameras, and lights have no mesh geometry. The calculator functions either fail silently or raise errors.

**Correct Approach**: ALWAYS verify target objects have mesh geometry before running QTO:
```python
for obj in bpy.context.selected_objects:
    if obj.type != 'MESH':
        obj.select_set(False)  # Deselect non-mesh objects
```

**Rule**: ALWAYS ensure target objects have mesh geometry before running QTO calculators.

---

## AP-13: Running Clash Detection on Unsaved Files

**Anti-Pattern**:
```python
# Modify geometry, then immediately run clash detection
bpy.ops.bim.execute_ifc_clash()
```

**Why It Fails**: Clash detection uses the IfcClash engine, which reads IFC files FROM DISK. It does NOT operate on the live in-memory model. Unsaved modifications are invisible to clash detection.

**Correct Approach**:
```python
bpy.ops.bim.save_project()  # Save to disk first
bpy.ops.bim.execute_ifc_clash()  # Now detects current geometry
```

**Rule**: ALWAYS save the IFC file before running clash detection.

---

## AP-14: Creating Drawings Without Orthographic Camera

**Anti-Pattern**:
```python
# Using a perspective camera for annotated drawings
camera = bpy.data.cameras.new("DrawingCam")
camera.type = 'PERSP'  # WRONG for annotated drawings
```

**Why It Fails**: Bonsai annotations are ONLY generated for orthographic camera views. Perspective cameras produce SVG files without annotation layers. Dimensions, text labels, and other annotations are missing from the output.

**Correct Approach**: ALWAYS use orthographic cameras for annotated drawings:
```python
bpy.ops.bim.add_drawing(target_view="PLAN_VIEW", location_hint=0)
# This creates an orthographic camera automatically
```

**Rule**: ALWAYS use orthographic cameras for drawings that require annotations.

---

## AP-15: Using Threading With Bonsai/IfcStore

**Anti-Pattern**:
```python
import threading

def background_ifc_work():
    model = IfcStore.get_file()
    walls = model.by_type("IfcWall")
    # Modify IFC data from background thread

thread = threading.Thread(target=background_ifc_work)
thread.start()
```

**Why It Fails**: `IfcStore` is a static singleton with NO thread-safety guarantees. Concurrent access corrupts internal state (`id_map`, `guid_map`, `edited_objs`). Blender operators also require the main thread.

**Correct Approach**: ALWAYS run Bonsai operations on the main thread. For deferred execution:
```python
import bpy

def deferred_work():
    model = IfcStore.get_file()
    # Safe — runs on main thread
    return None  # Return None to run once

bpy.app.timers.register(deferred_work)
```

**Rule**: NEVER use threading with Bonsai operations. Use `bpy.app.timers` for deferred execution.

---

## AP-16: Editing .ifc Files Externally While Blender Is Open

**Anti-Pattern**:
```bash
# In a separate terminal while Blender has the file open
nano project.ifc
# Or: python3 -c "import ifcopenshell; m = ifcopenshell.open('project.ifc'); m.write('project.ifc')"
```

**Why It Fails**: Bonsai maintains an in-memory copy of the IFC data via `IfcStore`. External edits create a desync between the on-disk file and the in-memory model. The next save from Blender overwrites external changes. Entity references may become invalid.

**Correct Approach**: NEVER edit IFC files externally while Blender has them open. If external editing is required:
1. Close the project in Blender
2. Edit externally
3. Reopen in Blender

If already edited externally: File > Reload IFC Project.

**Rule**: NEVER edit `.ifc` files on disk while Blender has the project open.

---

## AP-17: Using `get_gross_volume()` When Openings Matter

**Anti-Pattern**:
```python
# Using gross volume for cost estimation on walls with windows
volume = calculator.get_gross_volume(wall_obj)
cost = volume * price_per_m3  # Overestimates — includes window voids
```

**Why It Fails**: Gross volume represents the ORIGINAL geometry WITHOUT opening subtractions. For a wall with a window opening, gross volume includes the void space. This overestimates material quantities and costs.

**Correct Approach**:
```python
# Use net volume for actual material in place
volume = calculator.get_net_volume(wall_obj)  # Excludes opening voids
cost = volume * price_per_m3
```

**Rule**: ALWAYS use net variants (`get_net_volume`, `get_net_surface_area`) for elements with openings when calculating actual material quantities. Use gross variants only for material ordering or structural analysis of the original geometry.

---

## AP-18: Calling Operators Without Checking Poll Prerequisites

**Anti-Pattern**:
```python
# Calling operator without verifying prerequisites
bpy.ops.bim.create_drawing()  # Fails if no camera or no IFC file
bpy.ops.bim.edit_object_placement()  # Fails if no active IFC object
```

**Why It Fails**: Bonsai operators implement `poll()` classmethods. When prerequisites are not met, `poll()` returns `False` and operators raise `RuntimeError` from Python (or appear greyed out in UI).

**Correct Approach**:
```python
# Check prerequisites before calling
if bpy.ops.bim.create_drawing.poll():
    bpy.ops.bim.create_drawing(sync=True)
else:
    print("Prerequisites not met — check camera, IFC file, drawing state")
```

**Rule**: ALWAYS verify operator prerequisites before calling. Check `poll()` or verify state manually (active IFC file, active object, correct mode, active camera).
