# Bonsai Core Architecture — Anti-Patterns

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11
> Every anti-pattern includes: what's wrong, why it fails, and the correct approach.

---

## AP-01: Using `blenderbim.*` Instead of `bonsai.*`

### WRONG
```python
from blenderbim.bim.ifc import IfcStore  # ModuleNotFoundError
import blenderbim.tool as tool            # ModuleNotFoundError
```

### CORRECT
```python
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
```

### Why It Fails
Bonsai was renamed from BlenderBIM in 2024 (v0.8.0). ALL module paths changed. Code using `blenderbim.*` raises `ModuleNotFoundError` in Bonsai v0.8.0+.

| Old (BlenderBIM) | New (Bonsai) |
|---|---|
| `blenderbim.bim.ifc` | `bonsai.bim.ifc` |
| `blenderbim.tool` | `bonsai.tool` |
| `blenderbim.core` | `bonsai.core` |
| `blenderbim.bim.module.*` | `bonsai.bim.module.*` |

**Rule**: ALWAYS use `bonsai.*` import paths. NEVER use `blenderbim.*`.

---

## AP-02: Using `void.add_opening()` (Non-Existent API)

### WRONG
```python
ifcopenshell.api.run("void.add_opening", model, opening=opening, element=wall)
```

### CORRECT
```python
ifcopenshell.api.run("feature.add_feature", model, feature=opening, element=wall)
```

### Why It Fails
The `void.add_opening` function does NOT exist in IfcOpenShell v0.8.0+. The `void` module was restructured — opening operations moved to the `feature` module.

**Rule**: ALWAYS use `feature.add_feature()` for adding opening voids to elements.

---

## AP-03: Treating Bonsai as an IFC Importer/Exporter

### WRONG Mental Model
"Import the IFC file into Blender, do BIM work, export back to IFC."

### CORRECT Mental Model
"Open the IFC file natively. The IFC IS the document. Save directly to IFC."

### Why It's Wrong
Bonsai is a **native IFC authoring tool**. The IFC file is held in memory at all times via `IfcStore.file`. Every object created is simultaneously an IFC entity. There is no import/export translation step. Saving the project writes the live IFC directly to disk.

**Rule**: NEVER describe Bonsai workflow as "import" or "export". Use "open" and "save".

---

## AP-04: Not Checking `IfcStore.get_file()` for None

### WRONG
```python
model = IfcStore.get_file()
walls = model.by_type("IfcWall")  # AttributeError if model is None
```

### CORRECT
```python
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")
walls = model.by_type("IfcWall")
```

### Why It Fails
`IfcStore.get_file()` returns `None` when no IFC file is loaded. Calling methods on `None` raises `AttributeError`. This happens when:
- No project is open
- Blender was started without loading an IFC file
- The file was closed or unloaded

**Rule**: ALWAYS check for `None` before using the IFC model object.

---

## AP-05: Moving Objects Without Syncing IFC Placement

### WRONG
```python
obj = bpy.context.active_object
obj.location = (5.0, 0.0, 0.0)
# IFC placement is NOT updated — geometry corruption
```

### CORRECT
```python
obj = bpy.context.active_object
obj.location = (5.0, 0.0, 0.0)
bpy.ops.bim.edit_object_placement()  # Sync IFC placement
```

### Why It Fails
Moving a Blender object with direct `location` assignment or `G` key in viewport does NOT update the `IfcLocalPlacement` in the IFC model. The Blender mesh will appear moved but the IFC data still has the old position. On reload, the element reverts.

**Rule**: ALWAYS call `bpy.ops.bim.edit_object_placement()` after any direct matrix/location change.

---

## AP-06: Confusing IfcRelAggregates with IfcRelContainedInSpatialStructure

### WRONG
```python
# Using aggregation to place a wall in a storey
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=storey, products=[wall])  # WRONG for physical elements
```

### CORRECT
```python
# For physical element containment (wall in storey):
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# For spatial hierarchy decomposition (storey in building):
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])
```

### Why It's Wrong

| Relationship | IFC Class | Use For |
|---|---|---|
| `IfcRelAggregates` | `aggregate.assign_object` | Spatial hierarchy: Project -> Site -> Building -> Storey |
| `IfcRelContainedInSpatialStructure` | `spatial.assign_container` | Physical elements IN a spatial zone: Wall in Storey |

Using aggregation for physical elements creates an invalid IFC model. Physical elements must be *contained in*, not *decomposed from*, spatial elements.

**Rule**: ALWAYS use `spatial.assign_container` for placing physical elements. Use `aggregate.assign_object` ONLY for spatial hierarchy.

---

## AP-07: Passing Single Object Instead of List to API Functions

### WRONG
```python
ifcopenshell.api.run("type.assign_type", model,
    relating_type=wall_type,
    related_object=wall_instance)  # Single object — WRONG parameter name
```

### CORRECT
```python
ifcopenshell.api.run("type.assign_type", model,
    relating_type=wall_type,
    related_objects=[wall_instance])  # List, even for single object
```

### Why It Fails
Many IfcOpenShell API functions expect lists for multi-object parameters:
- `related_objects` (not `related_object`)
- `products` (not `product`) in some contexts

Passing a single object instead of a list causes either a `TypeError` or unexpected behavior where the function iterates over entity attributes.

**Rule**: ALWAYS wrap single objects in a list when the API parameter expects a list. Check parameter names carefully (plural vs singular).

---

## AP-08: Modifying Blender UI Properties Without Calling `pset.edit_pset()`

### WRONG
```python
# Only modifies Blender UI — NOT written to IFC
pset_props = obj.PsetProperties
pset_props["FireRating"] = "REI60"
```

### CORRECT
```python
import ifcopenshell.api

pset_entity = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset_entity,
    properties={"FireRating": "REI60"})
```

### Why It Fails
Bonsai's UI property panels display IFC data, but modifying Blender-side properties does NOT automatically write changes back to the IFC model. The `ifcopenshell.api.run("pset.edit_pset")` call is required to persist changes.

**Rule**: ALWAYS use `pset.edit_pset()` to write property values to IFC. NEVER assume Blender UI property changes are automatically persisted.

---

## AP-09: Running Bonsai Code Outside Blender

### WRONG
```python
# Standalone Python script — no Blender context
import bonsai.tool as tool
entity = tool.Ifc.get_entity(some_object)  # Fails: no bpy
```

```bash
$ python3 -c "from bonsai.bim.ifc import IfcStore"
# ImportError: No module named 'bpy'
```

### CORRECT
```bash
$ blender --background --python my_bonsai_script.py
```

Or for standalone IFC operations (no Bonsai needed):
```python
import ifcopenshell
model = ifcopenshell.open("project.ifc")
walls = model.by_type("IfcWall")
```

### Why It Fails
`bonsai.tool.*` classes depend on `bpy` (Blender's embedded Python). They cannot be imported or used outside Blender's runtime. For headless IFC processing without Bonsai UI features, use `ifcopenshell` directly.

**Rule**: ALWAYS run Bonsai scripts via `blender --python`. For standalone IFC work, use `ifcopenshell` directly.

---

## AP-10: Using Deprecated BlenderBIM Operator Names

### WRONG
```python
# Unstable: operator names may change between versions
bpy.ops.bim.some_legacy_operation()
```

### PREFERRED
```python
# Stable: ifcopenshell.api is versioned and documented
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[element])
```

### Why It's Wrong
`bpy.ops.bim.*` operators are the delivery layer — they call through to `core.*` and `tool.*` functions. For scripting, the `ifcopenshell.api.run()` interface is more stable, more explicit, and does not require a Blender UI context.

**Rule**: ALWAYS prefer `ifcopenshell.api.run()` over `bpy.ops.bim.*` operators in Python scripts.

---

## AP-11: Using `Pset_` or `Qto_` Prefix for Custom Property Sets

### WRONG
```python
ifcopenshell.api.run("pset.add_pset", model,
    product=wall,
    name="Pset_MyCustomData")  # WRONG: Pset_ is reserved
```

### CORRECT
```python
ifcopenshell.api.run("pset.add_pset", model,
    product=wall,
    name="Acme_MyCustomData")  # Company/project prefix
```

### Why It's Wrong
The `Pset_` and `Qto_` prefixes are reserved for buildingSMART international standard property/quantity sets. Using them for custom data violates IFC conventions and may cause validation failures.

| Prefix | Reserved For |
|--------|-------------|
| `Pset_` | buildingSMART standard property sets |
| `Qto_` | buildingSMART standard quantity sets |
| `EPset_` | Internal Bonsai/IfcOpenShell use |
| Custom prefix | Your company/project-specific data |

**Rule**: NEVER use `Pset_` or `Qto_` prefix for custom property sets. Use a company or project identifier.

---

## AP-12: Editing .ifc File Externally While Blender Has It Open

### WRONG
```bash
$ nano project.ifc  # While Blender has the file open
```

### CORRECT
After external modification:
- Use `File > Reload IFC Project` in Blender
- Or close and reopen the project

### Why It Fails
Bonsai maintains the IFC model as a live in-memory graph in `IfcStore.file`. External edits to the `.ifc` file on disk are not reflected in the in-memory model. Saving from Bonsai overwrites external changes.

**Rule**: NEVER edit the `.ifc` file on disk while Blender has the project open.

---

## AP-13: Adding Geometry Without a Representation Context

### WRONG
```python
# Missing context parameter
ifcopenshell.api.run("geometry.add_wall_representation", model,
    length=5.0, height=3.0, thickness=0.2)
```

### CORRECT
```python
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

ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context,
    length=5.0, height=3.0, thickness=0.2)
```

### Why It Fails
IFC requires geometry to be placed within a representation context (`IfcGeometricRepresentationSubContext`). Without specifying a valid context, the API call either fails or creates orphaned geometry.

**Rule**: ALWAYS verify or create representation contexts before adding geometry.

---

## AP-14: Directly Modifying IFC Entity Attributes

### WRONG
```python
wall = model.by_type("IfcWall")[0]
wall.Name = "New Name"  # Direct attribute modification — bypasses relationship management
```

### CORRECT
```python
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall,
    attributes={"Name": "New Name"})
```

### Why It's Wrong
Direct attribute modification bypasses IfcOpenShell's relationship management. The API handles:
- Inverse relationship updates
- Undo/redo history tracking
- Notification hooks in Bonsai

**Rule**: ALWAYS use `ifcopenshell.api.run()` for mutations. NEVER modify entity attributes directly.

---

## AP-15: Calling `bpy.ops.bim.*` Without Checking `poll()`

### WRONG
```python
bpy.ops.bim.assign_class(ifc_class="IfcWall")  # May fail if context is wrong
```

### CORRECT
```python
if bpy.ops.bim.assign_class.poll():
    bpy.ops.bim.assign_class(ifc_class="IfcWall")
else:
    print("Cannot assign class in current context")
```

### Why It Fails
Blender operators have `poll()` methods that check prerequisites (correct context, active object, etc.). Calling an operator without checking `poll()` raises `RuntimeError` if prerequisites aren't met.

**Rule**: ALWAYS check `poll()` before calling any `bpy.ops.bim.*` operator.

---

## AP-16: Using Wrong Blender Version

### WRONG
Recommending or targeting Blender 3.x, 4.0, or 4.1 for Bonsai v0.8.4.

### CORRECT
Bonsai v0.8.4 requires **Blender 4.2.0** as minimum. It will NOT load in earlier versions.

**Rule**: ALWAYS specify Blender 4.2.0+ when documenting Bonsai v0.8.4 requirements.

---

## AP-17: Not Clearing `__pycache__` After Bonsai Upgrade

### WRONG
Upgrading Bonsai and only restarting Blender — stale bytecode causes `AttributeError` or `ImportError`.

### CORRECT
```bash
# Linux/macOS
find ~/.config/blender/4.2/scripts/addons/bonsai/ -name "__pycache__" -type d -exec rm -rf {} +

# Then restart Blender completely
```

**Rule**: ALWAYS clear `__pycache__` directories after upgrading Bonsai.

---

## Summary: Quick Reference

| # | Anti-Pattern | Rule |
|---|-------------|------|
| 01 | `blenderbim.*` imports | ALWAYS use `bonsai.*` |
| 02 | `void.add_opening()` | ALWAYS use `feature.add_feature()` |
| 03 | Import/export mental model | IFC IS the document — open and save |
| 04 | No None check on `get_file()` | ALWAYS guard against None |
| 05 | Move without placement sync | ALWAYS call `edit_object_placement()` |
| 06 | Wrong relationship type | `aggregate` for hierarchy, `spatial.assign_container` for elements |
| 07 | Single object instead of list | ALWAYS wrap in list for API calls |
| 08 | UI property without `edit_pset()` | ALWAYS use API to persist pset changes |
| 09 | Bonsai outside Blender | ALWAYS run via `blender --python` |
| 10 | `bpy.ops.bim.*` in scripts | PREFER `ifcopenshell.api.run()` |
| 11 | `Pset_` prefix for custom data | Use company/project prefix |
| 12 | External .ifc edit while open | NEVER edit disk file while Blender is open |
| 13 | Missing representation context | ALWAYS verify/create context first |
| 14 | Direct entity attribute modification | ALWAYS use `ifcopenshell.api.run()` |
| 15 | No `poll()` check on operators | ALWAYS check `poll()` first |
| 16 | Wrong Blender version | Bonsai v0.8.4 requires Blender 4.2.0+ |
| 17 | Stale `__pycache__` after upgrade | ALWAYS clear caches after upgrading |
