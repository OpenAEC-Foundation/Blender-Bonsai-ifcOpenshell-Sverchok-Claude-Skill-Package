---
name: bonsai-errors-common
description: >
  Use when debugging Bonsai errors -- IFC schema violations, spatial hierarchy errors,
  property set failures, geometry representation issues, or operator poll failures.
  Prevents wasting time on symptoms instead of root causes (e.g., missing IfcOwnerHistory
  in IFC2X3 causing cryptic errors). Provides diagnostic decision trees and recovery
  strategies for all common Bonsai failure modes.
  Keywords: Bonsai error, schema violation, spatial hierarchy error, poll failure,
  drawing error, BCF error, IFC error, troubleshooting, debug.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender with Bonsai addon."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Common Errors — Diagnostic & Recovery Guide

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.*` — NEVER `blenderbim.*`
> **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements

## Quick Reference — Error Decision Tree

```
Bonsai error encountered?
│
├── Python ImportError / ModuleNotFoundError?
│   ├── "No module named 'blenderbim'" → §1.1 (rename migration)
│   ├── "No module named 'bpy'" → §1.2 (wrong Python environment)
│   └── "No module named 'bonsai'" → §1.3 (addon not installed)
│
├── AttributeError / RuntimeError on IFC operation?
│   ├── "NoneType has no attribute 'by_type'" → §2.1 (no IFC file loaded)
│   ├── "No entity with id X" → §2.2 (stale entity reference)
│   ├── "RuntimeError: No structural analysis view context" → §2.3 (missing context)
│   └── "'void' module" or "add_opening" → §2.4 (deprecated API)
│
├── IFC schema violation or validation failure?
│   ├── Missing IfcProject / IfcUnitAssignment → §3.1 (invalid project)
│   ├── Wrong spatial hierarchy (aggregate vs contain) → §3.2
│   ├── Missing representation context → §3.3
│   └── Singular vs list parameter error → §3.4 (v0.8+ API change)
│
├── Geometry / placement issue?
│   ├── Object moved but IFC placement unchanged → §4.1
│   ├── Geometry not visible after creation → §4.2
│   └── Geometry corruption after external edit → §4.3
│
├── Property set error?
│   ├── Properties not written to IFC → §5.1
│   ├── Pset values not appearing in UI → §5.2
│   └── Weight calculation returns None → §5.3
│
├── Operator poll failure (greyed out / RuntimeError)?
│   ├── No active IFC file → §6.1
│   ├── No active object / wrong selection → §6.2
│   ├── Wrong Blender mode (Edit vs Object) → §6.3
│   └── No active camera / drawing context → §6.4
│
├── Drawing generation error?
│   ├── Empty SVG output → §7.1
│   ├── Missing annotations → §7.2
│   ├── Sheet composition failure → §7.3
│   └── InkScape dependency error → §7.4
│
├── BCF workflow error?
│   ├── BcfStore initialization failure → §8.1
│   ├── Viewpoint coordinate mismatch → §8.2
│   └── Version conflict (v2 vs v3) → §8.3
│
└── Clash detection error?
    ├── Empty results on known conflicts → §9.1
    ├── Unsaved IFC file → §9.2
    └── Missing Group A/B definition → §9.3
```

## 1. Import and Environment Errors

### §1.1 ModuleNotFoundError: blenderbim

**Error**: `ModuleNotFoundError: No module named 'blenderbim'`

**Cause**: Using pre-2024 import paths. Bonsai was renamed from BlenderBIM in v0.8.0 (September 2024).

**Fix**: Replace ALL `blenderbim.*` imports with `bonsai.*`:

| Old (WRONG) | New (CORRECT) |
|-------------|---------------|
| `from blenderbim.bim.ifc import IfcStore` | `from bonsai.bim.ifc import IfcStore` |
| `import blenderbim.tool as tool` | `import bonsai.tool as tool` |
| `import blenderbim.core.spatial` | `import bonsai.core.spatial` |

**Rule**: ALWAYS use `bonsai.*`. Operator `bl_idname` values (`bim.*`) did NOT change.

### §1.2 ModuleNotFoundError: bpy

**Error**: `ModuleNotFoundError: No module named 'bpy'`

**Cause**: Running Bonsai scripts from system Python instead of Blender's bundled Python.

**Fix**: ALWAYS execute via Blender:
```bash
blender --background --python my_script.py
```

NEVER run `python3 my_script.py` for scripts importing `bonsai.*` or `bpy`.

### §1.3 Bonsai Not Installed

**Error**: `ModuleNotFoundError: No module named 'bonsai'`

**Diagnosis**:
1. Verify Blender version is 4.2.0+ (Bonsai v0.8.x minimum)
2. Check addon is enabled: Edit > Preferences > Add-ons > search "Bonsai"
3. On Linux: verify GLIBC 2.34+ (`ldd --version | head -1`)

### §1.4 Module Cache Corruption After Upgrade

**Error**: `AttributeError` or `ImportError` after upgrading Bonsai

**Cause**: Stale `.pyc` bytecode from previous version.

**Fix**:
```bash
find ~/.config/blender/4.2/scripts/addons/bonsai/ -name "__pycache__" -type d -exec rm -rf {} +
```
Then restart Blender completely.

## 2. IFC Runtime Errors

### §2.1 NoneType Error on IFC Operations

**Error**: `AttributeError: 'NoneType' object has no attribute 'by_type'`

**Cause**: `IfcStore.get_file()` returns `None` — no IFC project is loaded.

**Fix**: ALWAYS guard IFC access:
```python
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")
```

### §2.2 Stale Entity Reference

**Error**: `RuntimeError: No entity with id X` or entity attributes return unexpected values.

**Cause**: Entity references become invalid after file reload, undo operations, or external file modification.

**Fix**: NEVER cache entity references across operations. ALWAYS re-query:
```python
entity = model.by_id(step_id)  # Re-query each time
```

### §2.3 Missing Representation Context

**Error**: `RuntimeError: No structural analysis view context found` or geometry creation fails silently.

**Cause**: Adding geometry or representations without required `IfcGeometricRepresentationContext`.

**Fix**: ALWAYS verify context exists before creating geometry:
```python
import ifcopenshell.util.representation

body_context = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")
if not body_context:
    model_ctx = ifcopenshell.api.run("context.add_context", model,
        context_type="Model")
    body_context = ifcopenshell.api.run("context.add_context", model,
        context_type="Model", context_identifier="Body",
        target_view="MODEL_VIEW", parent=model_ctx)
```

### §2.4 Deprecated void.add_opening

**Error**: Module `void` not found or `add_opening` does not exist.

**Cause**: `void.add_opening()` was replaced by `feature.add_feature()` in IfcOpenShell v0.8.0+.

**Fix**:
```python
# WRONG
ifcopenshell.api.run("void.add_opening", model, opening=opening, element=wall)

# CORRECT
ifcopenshell.api.run("feature.add_feature", model, feature=opening, element=wall)
```

## 3. IFC Schema Violations

### §3.1 Invalid Project Structure

**Error**: IFC validation failure — missing IfcProject or IfcUnitAssignment.

**Cause**: Creating IFC files with `ifcopenshell.file()` directly instead of the project API.

**Fix**: ALWAYS use the project creation API:
```python
model = ifcopenshell.api.project.create_file(version="IFC4")
ifcopenshell.api.unit.assign_unit(model)
```

NEVER use `ifcopenshell.file()` directly — it skips IfcProject, unit assignment, and context creation.

### §3.2 Wrong Spatial Relationship Type

**Error**: Elements appear in wrong hierarchy level or validation reports spatial errors.

**Cause**: Confusing `IfcRelAggregates` (spatial decomposition) with `IfcRelContainedInSpatialStructure` (element containment).

**Decision tree**:
```
Placing an element?
├── Physical element (wall, column, slab) into a storey?
│   └── spatial.assign_container (creates IfcRelContainedInSpatialStructure)
│
├── Spatial element (storey) into building?
│   └── aggregate.assign_object (creates IfcRelAggregates)
│
└── Building into site?
    └── aggregate.assign_object (creates IfcRelAggregates)
```

**Rule**: ALWAYS use `spatial.assign_container` for physical elements. ALWAYS use `aggregate.assign_object` for spatial hierarchy.

### §3.3 Missing Representation Context

See §2.3. Every IFC project MUST have at least a `Model/Body/MODEL_VIEW` context before assigning geometry.

### §3.4 Singular vs List Parameter (v0.8+ Breaking Change)

**Error**: `KeyError` or `TypeError` when calling relationship API functions.

**Cause**: IfcOpenShell v0.8+ changed singular parameters to lists.

**Fix**:
```python
# WRONG (pre-v0.8 style)
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)

# CORRECT (v0.8+ style)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# WRONG
ifcopenshell.api.run("type.assign_type", model,
    related_object=wall, relating_type=wall_type)

# CORRECT
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

**Rule**: ALWAYS pass lists to `products`, `related_objects` parameters, even for single entities.

## 4. Geometry and Placement Errors

### §4.1 IFC Placement Not Synced After Move

**Error**: Object appears moved in Blender viewport but IFC `IfcLocalPlacement` retains old coordinates.

**Cause**: Blender transform operations (`G` key, `obj.location = ...`) do NOT auto-update IFC placement.

**Fix**: ALWAYS call after any transform change:
```python
bpy.ops.bim.edit_object_placement()
```

### §4.2 Geometry Not Visible After Creation

**Diagnostic checklist**:
1. Verify representation context exists (§2.3)
2. Verify `geometry.assign_representation` was called
3. Verify element is assigned to a spatial container
4. Verify Blender object was linked via `tool.Ifc.link(entity, obj)`

### §4.3 Geometry Corruption From External Edit

**Error**: IFC data inconsistent with Blender viewport after editing `.ifc` file externally.

**Cause**: Editing the IFC file on disk while Blender has the project open.

**Fix**: NEVER edit `.ifc` files externally while Blender has them open. After external modification: File > Reload IFC Project.

## 5. Property Set Errors

### §5.1 Properties Not Written to IFC

**Error**: Property values set in Blender UI or via `obj.PsetProperties` do not persist in IFC file.

**Cause**: Modifying Blender UI properties does NOT automatically write to IFC.

**Fix**: ALWAYS use `ifcopenshell.api`:
```python
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model,
    pset=pset, properties={"FireRating": "REI60"})
```

### §5.2 Pset Values Not Appearing in UI

**Cause**: Bonsai data cache not refreshed after direct API modifications.

**Fix**: Purge the relevant data cache:
```python
from bonsai.bim.module.pset.data import PsetData
PsetData.purge()
```

### §5.3 Weight Calculation Returns None

**Error**: `get_net_weight()` or `get_gross_weight()` returns `None` or zero.

**Cause**: `Pset_MaterialCommon.MassDensity` not assigned to the element's material.

**Fix**: ALWAYS assign material density BEFORE running weight calculations.

## 6. Operator Poll Failures

Bonsai operators implement `poll()` classmethods that check prerequisites. When `poll()` returns `False`, operators are greyed out in UI and raise `RuntimeError` from Python.

### §6.1 No Active IFC File

**Symptom**: Most `bpy.ops.bim.*` operators fail or are greyed out.

**Fix**: Load or create an IFC project first:
```python
bpy.ops.bim.create_project(template="metric_m")
# OR
bpy.ops.bim.load_project(filepath="project.ifc")
```

### §6.2 No Active Object or Wrong Selection

**Symptom**: Element-specific operators (edit placement, assign class) fail.

**Fix**: ALWAYS verify selection:
```python
obj = bpy.context.active_object
if obj is None:
    raise RuntimeError("No active object selected")
if obj.BIMObjectProperties.ifc_definition_id == 0:
    raise RuntimeError("Active object has no IFC assignment")
```

### §6.3 Wrong Blender Mode

**Symptom**: Operators that require Object mode fail in Edit mode (or vice versa).

**Fix**: Switch mode before calling:
```python
if bpy.context.mode != 'OBJECT':
    bpy.ops.object.mode_set(mode='OBJECT')
```

### §6.4 No Active Camera / Drawing Context

**Symptom**: Drawing operators (`bim.create_drawing`, `bim.add_annotation`) fail.

**Fix**: ALWAYS activate a drawing camera before calling drawing operators:
```python
if bpy.context.scene.camera is None:
    raise RuntimeError("No active camera. Activate a drawing view first.")
```

## 7. Drawing Generation Errors

### §7.1 Empty SVG Output

**Diagnostic checklist**:
1. Active camera MUST be orthographic (not perspective)
2. IFC data MUST be synced (`sync=True` in `bim.create_drawing`)
3. Drawing elements MUST be within camera frustum
4. `has_linework` and/or `has_underlay` MUST be enabled

### §7.2 Missing Annotations

**Cause**: Annotations are ONLY generated for orthographic camera views.

**Fix**: NEVER use perspective cameras for annotated drawings. Verify camera type is `ORTHO`.

### §7.3 Sheet Composition Failure

**Cause**: InkScape not installed or not on system PATH.

**Fix**: Install InkScape. Sheet composition and PDF/DXF export REQUIRE InkScape.

### §7.4 Freestyle SVG Exporter Not Active

**Error**: FREESTYLE linework mode produces no output.

**Fix**: Enable the Freestyle SVG Exporter add-on in Blender preferences before using FREESTYLE mode.

## 8. BCF Workflow Errors

### §8.1 BcfStore Initialization Failure

**Cause**: Accessing `BcfStore.bcfxml` directly instead of using the lazy initializer.

**Fix**: ALWAYS use `BcfStore.get_bcfxml()` for proper initialization and path resolution.

### §8.2 Viewpoint Coordinate Mismatch

**Cause**: Viewpoint activation applies georeference offsets. Coordinate systems may differ between BCF source and loaded model.

**Fix**: Verify georeference settings match between the model that created the BCF and the model loading it.

### §8.3 BCF Version Conflict

**Error**: Operations fail when mixing BCF v2.1 and v3.0 data.

**Fix**: NEVER mix BCF versions within a single file. The version is detected on load and MUST remain consistent.

**Rule**: ALWAYS configure `BCFProperties.author` before creating topics or comments. ALWAYS call `bim.save_bcf_project` to persist changes.

## 9. Clash Detection Errors

### §9.1 Empty Results on Known Conflicts

**Diagnostic checklist**:
1. Verify IFC files are SAVED to disk (clash detection reads from disk, not memory)
2. Verify both Group A and Group B are defined with sources
3. Verify element filters are not excluding the conflicting elements
4. Verify `clash_set.mode` matches the conflict type (intersection vs clearance)

### §9.2 Unsaved IFC File

**Error**: Clash detection runs but finds no clashes in recently modified geometry.

**Cause**: Clash detection operates on IFC files on disk, NOT the live Blender scene.

**Fix**: ALWAYS save the IFC file before running clash detection:
```python
bpy.ops.bim.save_project()
bpy.ops.bim.execute_ifc_clash()
```

### §9.3 Missing Group Definition

**Error**: Clash detection requires both Group A and Group B.

**Fix**: ALWAYS define both groups, even for intra-model clashes (e.g., structural vs MEP from the same file).

## 10. Data Synchronization Errors

### §10.1 Viewport Not Updating After API Changes

**Cause**: Direct `ifcopenshell.api` calls do NOT trigger Blender viewport refresh.

**Fix**: After direct API modifications:
```python
# For property changes — purge data cache
from bonsai.bim.module.pset.data import PsetData
PsetData.purge()

# For geometry changes — trigger representation update
bpy.ops.bim.update_representation(obj=obj.name)
```

### §10.2 Threading Errors

**Cause**: `IfcStore` is NOT thread-safe. Bonsai operators require the main thread.

**Fix**: NEVER use threading with Bonsai operations. For deferred execution, use:
```python
bpy.app.timers.register(my_deferred_function)
```

### §10.3 Invalid References After Undo

**Cause**: Undo invalidates `IfcStore.file` and all cached entity references.

**Fix**: NEVER store entity references across undo operations. ALWAYS re-query after undo.

## References

- [Error Signatures and Diagnostic Commands](references/methods.md)
- [Error Reproduction and Fix Examples](references/examples.md)
- [Anti-Patterns That Cause Bonsai Errors](references/anti-patterns.md)

## Related Skills

- **bonsai-core-architecture** — Three-layer architecture, IfcStore, operator patterns
- **bonsai-syntax-elements** — Element access patterns, entity-object mapping
- **bonsai-impl-project** — Project creation, schema selection, units
- **bonsai-impl-modeling** — Element creation pipeline, type system, materials
- **bonsai-impl-drawing** — Drawing generation, annotations, sheets
- **bonsai-impl-qto** — Quantity takeoff, calculator functions
- **bonsai-impl-bcf** — BCF issue tracking workflows
- **bonsai-impl-clash** — Clash detection workflows

## Sources

- Bonsai Documentation: https://docs.bonsaibim.org
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell
- OSArch Community: https://community.osarch.org
- IfcOpenShell API Docs: https://docs.ifcopenshell.org
