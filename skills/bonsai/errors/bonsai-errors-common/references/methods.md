# Bonsai Error Signatures and Diagnostic Commands

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11

## Error Signature Reference

### Import Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| `ModuleNotFoundError: No module named 'blenderbim'` | Using pre-2024 import paths | §1.1 |
| `ModuleNotFoundError: No module named 'bpy'` | Running outside Blender Python | §1.2 |
| `ModuleNotFoundError: No module named 'bonsai'` | Addon not installed or wrong Blender version | §1.3 |
| `ImportError: cannot import name 'X' from 'bonsai'` | Stale bytecode cache after upgrade | §1.4 |

### IFC Runtime Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| `AttributeError: 'NoneType' object has no attribute 'by_type'` | No IFC file loaded | §2.1 |
| `AttributeError: 'NoneType' object has no attribute 'by_id'` | No IFC file loaded | §2.1 |
| `RuntimeError: No entity with id X` | Stale entity reference | §2.2 |
| `RuntimeError: No structural analysis view context found` | Missing representation context | §2.3 |
| `ModuleNotFoundError: No module named 'ifcopenshell.api.void'` | Deprecated API module | §2.4 |
| `AttributeError: module 'ifcopenshell.api' has no attribute 'void'` | Deprecated API module | §2.4 |

### Schema Validation Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| `Missing mandatory attribute: IfcProject.UnitsInContext` | No unit assignment | §3.1 |
| `Invalid IfcRelAggregates for physical elements` | Wrong relationship type | §3.2 |
| `No IfcGeometricRepresentationContext found` | Missing context | §3.3 |
| `KeyError: 'product'` / `TypeError: unexpected keyword argument 'product'` | Singular vs list parameter | §3.4 |

### Operator Poll Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| `RuntimeError: Operator bim.X failed, context is incorrect` | Wrong Blender context | §6.x |
| `Operator bim.X is not available` (greyed out in UI) | `poll()` returned False | §6.x |

### Drawing Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| Empty SVG file (0 KB or header only) | Camera/sync/frustum issue | §7.1 |
| `FileNotFoundError: inkscape not found` | InkScape not installed | §7.4 |
| Missing annotation layer in SVG | Perspective camera used | §7.2 |

### BCF Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| `AttributeError: 'NoneType' object has no attribute 'topics'` | BcfStore not initialized | §8.1 |
| Viewpoint shows wrong location | Georeference mismatch | §8.2 |
| `BCF version mismatch` | Mixing v2.1 and v3.0 | §8.3 |

### Clash Detection Errors

| Error Message | Root Cause | Section |
|--------------|------------|---------|
| 0 clashes found (expected results) | IFC file not saved to disk | §9.2 |
| `ValueError: No sources in group` | Missing Group A or B | §9.3 |

---

## Diagnostic Commands

### Environment Verification

```python
# Bonsai v0.8.x — Verify Bonsai installation and version
import bpy

# Check Bonsai addon is enabled
addon = bpy.context.preferences.addons.get("bonsai")
if addon is None:
    print("ERROR: Bonsai addon is not enabled")
else:
    print("Bonsai addon is active")

# Check Blender version
print(f"Blender version: {bpy.app.version_string}")
assert bpy.app.version >= (4, 2, 0), "Bonsai v0.8.x requires Blender 4.2.0+"

# Check Python version
import sys
print(f"Python version: {sys.version}")
```

### IFC State Diagnostics

```python
# Bonsai v0.8.x — Diagnose IFC state
from bonsai.bim.ifc import IfcStore

# Check if IFC file is loaded
model = IfcStore.get_file()
print(f"IFC file loaded: {model is not None}")

if model is not None:
    print(f"Schema: {IfcStore.schema}")
    print(f"File path: {IfcStore.path}")
    print(f"Entity count: {len(model.wrapped_data.entity_names())}")
    print(f"IfcProject count: {len(model.by_type('IfcProject'))}")
    print(f"id_map entries: {len(IfcStore.id_map)}")
    print(f"guid_map entries: {len(IfcStore.guid_map)}")
    print(f"Edited objects: {len(IfcStore.edited_objs)}")
```

### Spatial Hierarchy Diagnostics

```python
# Bonsai v0.8.x — Verify spatial hierarchy
import bonsai.tool as tool
import ifcopenshell.util.element

model = tool.Ifc.get()
if model is None:
    print("ERROR: No IFC file loaded")
else:
    # Check for IfcProject
    projects = model.by_type("IfcProject")
    print(f"IfcProject count: {len(projects)} (MUST be exactly 1)")

    # Check for IfcSite
    sites = model.by_type("IfcSite")
    print(f"IfcSite count: {len(sites)}")

    # Check for IfcBuilding
    buildings = model.by_type("IfcBuilding")
    print(f"IfcBuilding count: {len(buildings)}")

    # Check for IfcBuildingStorey
    storeys = model.by_type("IfcBuildingStorey")
    print(f"IfcBuildingStorey count: {len(storeys)}")

    # Verify hierarchy chain
    for storey in storeys:
        container = ifcopenshell.util.element.get_aggregate(storey)
        if container is None:
            print(f"WARNING: Storey '{storey.Name}' has no parent aggregate")
        elif not container.is_a("IfcBuilding"):
            print(f"WARNING: Storey '{storey.Name}' parent is {container.is_a()}, expected IfcBuilding")
```

### Representation Context Diagnostics

```python
# Bonsai v0.8.x — Verify representation contexts
import ifcopenshell.util.representation

model = tool.Ifc.get()
if model:
    body = ifcopenshell.util.representation.get_context(
        model, "Model", "Body", "MODEL_VIEW")
    print(f"Model/Body/MODEL_VIEW context: {'FOUND' if body else 'MISSING'}")

    plan = ifcopenshell.util.representation.get_context(
        model, "Plan", "Annotation", "PLAN_VIEW")
    print(f"Plan/Annotation/PLAN_VIEW context: {'FOUND' if plan else 'MISSING'}")

    axis = ifcopenshell.util.representation.get_context(
        model, "Model", "Axis", "GRAPH_VIEW")
    print(f"Model/Axis/GRAPH_VIEW context: {'FOUND' if axis else 'MISSING'}")
```

### Unit Diagnostics

```python
# Bonsai v0.8.x — Verify unit assignment
import ifcopenshell.util.unit

model = tool.Ifc.get()
if model:
    project = model.by_type("IfcProject")[0]
    units = project.UnitsInContext
    if units is None:
        print("ERROR: No IfcUnitAssignment on project")
    else:
        for unit in units.Units:
            if hasattr(unit, "UnitType"):
                prefix = getattr(unit, "Prefix", None) or ""
                name = getattr(unit, "Name", "") or ""
                print(f"  {unit.UnitType}: {prefix}{name}")
```

### Object-IFC Link Diagnostics

```python
# Bonsai v0.8.x — Verify object-entity links
import bpy
import bonsai.tool as tool

for obj in bpy.context.selected_objects:
    ifc_id = obj.BIMObjectProperties.ifc_definition_id
    entity = tool.Ifc.get_entity(obj)
    blender_obj = tool.Ifc.get_object(entity) if entity else None

    print(f"Object: {obj.name}")
    print(f"  ifc_definition_id: {ifc_id}")
    print(f"  Entity found: {entity is not None}")
    if entity:
        print(f"  IFC class: {entity.is_a()}")
        print(f"  GlobalId: {entity.GlobalId}")
    print(f"  Reverse lookup matches: {blender_obj == obj if blender_obj else 'N/A'}")
```

### Drawing State Diagnostics

```python
# Bonsai v0.8.x — Verify drawing state
import bpy

scene = bpy.context.scene
camera = scene.camera

print(f"Active camera: {camera.name if camera else 'NONE'}")
if camera:
    print(f"  Camera type: {camera.data.type}")
    cam_props = camera.data.BIMCameraProperties
    print(f"  Target view: {cam_props.target_view}")
    print(f"  Has linework: {cam_props.has_linework}")
    print(f"  Has underlay: {cam_props.has_underlay}")
    print(f"  Has annotation: {cam_props.has_annotation}")
    print(f"  Diagram scale: {cam_props.diagram_scale}")
```

### Operator Poll Diagnostics

```python
# Bonsai v0.8.x — Check operator prerequisites
import bpy

# Test if an operator can execute
operator_idname = "bim.create_drawing"
can_execute = bpy.ops.bim.create_drawing.poll()
print(f"{operator_idname} poll(): {can_execute}")

# Common prerequisite checks
print(f"Active object: {bpy.context.active_object is not None}")
print(f"Object mode: {bpy.context.mode == 'OBJECT'}")
print(f"IFC loaded: {IfcStore.get_file() is not None}")
print(f"Active camera: {bpy.context.scene.camera is not None}")
```

### Linux GLIBC Diagnostics

```bash
# Check GLIBC version (Bonsai v0.8.x requires 2.34+)
ldd --version | head -1

# Check Ubuntu version
lsb_release -a 2>/dev/null || cat /etc/os-release
```

---

## Cache Purge Reference

After direct `ifcopenshell.api` modifications, purge the relevant module data cache:

| Modified Domain | Cache Class | Purge Command |
|----------------|-------------|---------------|
| Property sets | `PsetData` | `from bonsai.bim.module.pset.data import PsetData; PsetData.purge()` |
| Drawing | `DrawingData` | `from bonsai.bim.module.drawing.data import DrawingData; DrawingData.purge()` |
| Spatial | `SpatialData` | `from bonsai.bim.module.spatial.data import SpatialData; SpatialData.purge()` |
| QTO | `QtoData` | `from bonsai.bim.module.qto.data import QtoData; QtoData.purge()` |
| Type | `TypeData` | `from bonsai.bim.module.type.data import TypeData; TypeData.purge()` |
| Material | `MaterialData` | `from bonsai.bim.module.material.data import MaterialData; MaterialData.purge()` |
