---
name: aec-core-bim-workflows
description: "Orchestrates end-to-end BIM workflows combining IfcOpenShell for IFC manipulation, Bonsai for native BIM authoring in Blender, and Blender for 3D operations. Covers common AEC automation patterns: IFC file creation and enrichment, model validation pipelines, geometry extraction workflows, property set management across tools, and batch processing of building models."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Cross-Technology BIM Workflows

> **Scope**: End-to-end workflows combining IfcOpenShell, Bonsai, and Blender
> **Dependencies**: `ifcos-impl-creation`, `bonsai-core-architecture`
> **Version coverage**: Blender 3.x-5.x | IFC2X3/IFC4/IFC4X3 | Bonsai v0.8.x

## Critical Warnings

1. **ALWAYS** determine the execution context first: standalone Python (IfcOpenShell only) vs Blender Python (bpy available) vs Bonsai-loaded (IfcStore available). The API surface differs per context.
2. **ALWAYS** use `ifcopenshell.api.run()` for IFC mutations in ALL contexts. NEVER modify IFC entity attributes directly — this bypasses relationship management and breaks graph integrity.
3. **ALWAYS** call `bpy.ops.bim.edit_object_placement()` after modifying a Blender object's location/rotation/scale when Bonsai is active. Blender transforms do NOT auto-sync to IFC.
4. **NEVER** mix `IfcStore.get_file()` (Bonsai context) with `ifcopenshell.open()` on the same file simultaneously. Bonsai owns the in-memory IFC graph — opening a second handle creates divergent state.
5. **NEVER** use `blenderbim.*` imports. The package was renamed to `bonsai.*` in v0.8.0 (2024).
6. **ALWAYS** set up units (`unit.assign_unit`) and geometric contexts (`context.add_context`) before creating any geometry. Geometry without context or units is ambiguous and may render incorrectly.
7. **NEVER** use `void.add_opening()` in Bonsai v0.8.0+. Use `feature.add_feature()` instead.

---

## Decision Tree: Choose Your Execution Context

```
What is your workflow?
|
+-- Creating/modifying IFC files WITHOUT Blender?
|   +-- Use: Standalone IfcOpenShell (headless Python)
|   +-- Import: ifcopenshell, ifcopenshell.api
|   +-- File access: ifcopenshell.open(path) or ifcopenshell.api.run("project.create_file")
|   +-- Best for: batch processing, CI/CD validation, server-side operations
|
+-- Creating/modifying IFC files IN Blender WITH Bonsai?
|   +-- Use: Bonsai-loaded context
|   +-- File access: IfcStore.get_file() — NEVER ifcopenshell.open()
|   +-- IFC mutations: tool.Ifc.run("command", **kwargs) OR ifcopenshell.api.run()
|   +-- Blender sync: bpy.ops.bim.* operators handle bidirectional sync
|   +-- Best for: interactive BIM authoring, visual verification
|
+-- Extracting geometry from IFC for Blender visualization (no Bonsai)?
|   +-- Use: IfcOpenShell geometry + Blender bpy
|   +-- Import: ifcopenshell, ifcopenshell.geom, bpy
|   +-- Geometry: ifcopenshell.geom.create_shape(settings, element)
|   +-- Mesh: bpy.data.meshes.new() + mesh.from_pydata(verts, [], faces)
|   +-- Best for: lightweight IFC viewers, geometry analysis
|
+-- Batch processing multiple IFC files?
    +-- Use: Standalone IfcOpenShell (headless) or blender --background --python
    +-- If Blender ops needed: blender --background --python script.py
    +-- If pure IFC: python script.py (no Blender dependency)
    +-- Best for: model validation, property extraction, report generation
```

---

## Decision Tree: IFC Creation vs Enrichment

```
Starting from scratch or enriching existing?
|
+-- New IFC model from scratch?
|   +-- Follow the 9-step creation pipeline:
|   |   1. project.create_file → Create file with schema version
|   |   2. root.create_entity (IfcProject) → Create project
|   |   3. unit.assign_unit → Set measurement units
|   |   4. context.add_context → Set up Model + Body subcontext
|   |   5. root.create_entity + aggregate.assign_object → Build spatial hierarchy
|   |   6. root.create_entity + type assignment → Create element types
|   |   7. root.create_entity + geometry + placement → Create elements
|   |   8. spatial.assign_container → Place elements in hierarchy
|   |   9. pset.add_pset + pset.edit_pset → Add properties
|   +-- Reference: ifcos-impl-creation (Pattern 1)
|
+-- Enriching an existing IFC file?
|   +-- Open: model = ifcopenshell.open("existing.ifc")
|   +-- Query: elements = model.by_type("IfcWall")
|   +-- Add properties: pset.add_pset + pset.edit_pset
|   +-- Add classifications: classification.add_reference
|   +-- Modify geometry: geometry operations
|   +-- Write: model.write("enriched.ifc")
|   +-- WARNING: Preserve existing GlobalIds. NEVER recreate entities that already exist.
|
+-- Merging data from multiple IFC files?
    +-- Open source: source = ifcopenshell.open("source.ifc")
    +-- Open target: target = ifcopenshell.open("target.ifc")
    +-- Extract data from source (properties, classifications, types)
    +-- Apply to target using ifcopenshell.api.run()
    +-- WARNING: Entity IDs are file-scoped. NEVER copy raw IDs between files.
    +-- WARNING: GlobalIds MUST be unique within a file. Generate new GUIDs for copied entities.
```

---

## Sverchok Parametric Design Step

When workflows involve parametric, generative, or data-driven geometry (arrays, facades, repetitive elements), insert a **Sverchok step** before BIM authoring:

```
[Sverchok: Parametric Geometry] --> [IfcSverchok: Generate IFC] --> [Bonsai: Review/Enrich]
```

- **Sverchok** generates geometry via visual node trees (`SverchCustomTreeType`)
- **IfcSverchok** (31 nodes) converts parametric geometry to IFC entities within the node tree
- Enable `use_bonsai_file` on `SvIfcCreateProject` to write directly into Bonsai's active IFC file
- **Warning**: `SvIfcStore` is transient — purged on every full tree update. Persist via Bonsai or `model.write()`
- Refer to: `sverchok-impl-parametric`, `sverchok-impl-ifcsverchok`

---

## Essential Patterns

### Pattern 1: Standalone IFC Creation (No Blender)

```python
# Context: Standalone Python — IfcOpenShell only
# Version: IFC4 | IfcOpenShell 0.8.x
import ifcopenshell
import ifcopenshell.api
import numpy as np

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

model.write("output.ifc")
```

### Pattern 2: IFC Geometry Extraction to Blender Mesh (No Bonsai)

```python
# Context: Blender Python (bpy available), NO Bonsai
# Version: Blender 3.x-5.x | IfcOpenShell 0.8.x
import bpy
import ifcopenshell
import ifcopenshell.geom

ifc_file = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

for wall in ifc_file.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)
    verts = shape.geometry.verts     # flat list: [x0,y0,z0, x1,y1,z1, ...]
    faces = shape.geometry.faces     # flat list: [i0,i1,i2, i3,i4,i5, ...]

    # Reshape into Blender format
    vertices = [(verts[i], verts[i+1], verts[i+2])
                for i in range(0, len(verts), 3)]
    triangles = [(faces[i], faces[i+1], faces[i+2])
                 for i in range(0, len(faces), 3)]

    mesh = bpy.data.meshes.new(name=wall.Name or "Wall")
    mesh.from_pydata(vertices, [], triangles)
    mesh.update()

    obj = bpy.data.objects.new(wall.Name or "Wall", mesh)
    bpy.context.collection.objects.link(obj)
```

### Pattern 3: Bonsai-Context BIM Authoring

```python
# Context: Blender with Bonsai loaded
# Version: Bonsai v0.8.x | Blender 4.2+
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

# ALWAYS check for active IFC project
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Create or open a project first.")

# Create element via Bonsai's tool layer
# Option A: Use ifcopenshell.api.run() directly
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="New Wall")

# Option B: Use Bonsai operators (handles Blender sync automatically)
bpy.ops.bim.add_wall()

# After creating elements, ensure spatial containment
storey = model.by_type("IfcBuildingStorey")[0]
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# CRITICAL: If you moved the Blender object, sync placement to IFC
bpy.ops.bim.edit_object_placement(context_override)

# Add property set
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI60"
})
```

### Pattern 4: Batch Property Extraction (Headless)

```python
# Context: Standalone Python or blender --background
# Version: IfcOpenShell 0.8.x | IFC4
import ifcopenshell
import ifcopenshell.util.element
import csv

model = ifcopenshell.open("building.ifc")

with open("wall_report.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["GlobalId", "Name", "IsExternal", "FireRating", "LoadBearing"])

    for wall in model.by_type("IfcWall"):
        psets = ifcopenshell.util.element.get_psets(wall)
        common = psets.get("Pset_WallCommon", {})
        writer.writerow([
            wall.GlobalId,
            wall.Name,
            common.get("IsExternal", ""),
            common.get("FireRating", ""),
            common.get("LoadBearing", ""),
        ])
```

### Pattern 5: Model Validation Pipeline

```python
# Context: Standalone Python
# Version: IfcOpenShell 0.8.x | IFC4/IFC2X3
import ifcopenshell
import ifcopenshell.util.element

def validate_model(filepath: str) -> list[str]:
    errors = []
    model = ifcopenshell.open(filepath)

    # Check 1: Spatial hierarchy exists
    if not model.by_type("IfcProject"):
        errors.append("CRITICAL: No IfcProject found")
    if not model.by_type("IfcSite"):
        errors.append("CRITICAL: No IfcSite found")
    if not model.by_type("IfcBuilding"):
        errors.append("CRITICAL: No IfcBuilding found")
    if not model.by_type("IfcBuildingStorey"):
        errors.append("WARNING: No IfcBuildingStorey found")

    # Check 2: All physical elements have spatial containment
    physical_types = ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam",
                      "IfcDoor", "IfcWindow"]
    for ifc_type in physical_types:
        for element in model.by_type(ifc_type):
            container = ifcopenshell.util.element.get_container(element)
            if container is None:
                errors.append(
                    f"WARNING: {element.is_a()} '{element.Name}' "
                    f"(#{element.id()}) has no spatial container")

    # Check 3: All elements have geometry
    for element in model.by_type("IfcProduct"):
        if hasattr(element, "Representation") and element.Representation is None:
            if element.is_a() not in ("IfcSite", "IfcBuilding",
                                       "IfcBuildingStorey", "IfcSpace",
                                       "IfcProject"):
                errors.append(
                    f"WARNING: {element.is_a()} '{element.Name}' "
                    f"(#{element.id()}) has no geometry representation")

    # Check 4: Units are assigned
    project = model.by_type("IfcProject")[0] if model.by_type("IfcProject") else None
    if project and not project.UnitsInContext:
        errors.append("CRITICAL: No units assigned to project")

    # Check 5: IFC2X3 OwnerHistory requirement
    if model.schema == "IFC2X3":
        for entity in model.by_type("IfcRoot"):
            if entity.OwnerHistory is None:
                errors.append(
                    f"ERROR: IFC2X3 requires OwnerHistory on "
                    f"{entity.is_a()} '{entity.Name}' (#{entity.id()})")

    return errors
```

### Pattern 6: Multi-File Batch Processing

```python
# Context: Standalone Python
# Version: IfcOpenShell 0.8.x
import ifcopenshell
import ifcopenshell.api
from pathlib import Path

def enrich_models(input_dir: str, output_dir: str, classification_system: str):
    """Add classification references to all IFC files in a directory."""
    input_path = Path(input_dir)
    output_path = Path(output_dir)
    output_path.mkdir(parents=True, exist_ok=True)

    for ifc_file in input_path.glob("*.ifc"):
        model = ifcopenshell.open(str(ifc_file))

        # Add classification system if not present
        classifications = model.by_type("IfcClassification")
        if not any(c.Name == classification_system for c in classifications):
            classification = ifcopenshell.api.run(
                "classification.add_classification", model,
                classification=classification_system)
        else:
            classification = next(
                c for c in classifications if c.Name == classification_system)

        # Process each element
        for wall in model.by_type("IfcWall"):
            ifcopenshell.api.run("classification.add_reference", model,
                products=[wall],
                classification=classification,
                identification="21.22",
                name="Exterior Walls")

        output_file = output_path / ifc_file.name
        model.write(str(output_file))
```

---

## Cross-Technology API Integration Points

### IfcOpenShell <-> Blender (Without Bonsai)

| IfcOpenShell Operation | Blender Equivalent | Integration Point |
|----------------------|-------------------|-------------------|
| `ifcopenshell.geom.create_shape()` | `mesh.from_pydata(verts, [], faces)` | Geometry extraction: shape.geometry.verts/faces |
| `element.ObjectPlacement` | `obj.matrix_world` | 4x4 transformation matrix |
| `model.by_type("IfcWall")` | `bpy.data.objects` filtering | Element enumeration |
| IFC materials | `bpy.data.materials` | Manual mapping required |

### IfcOpenShell <-> Bonsai

| IfcOpenShell Operation | Bonsai Equivalent | Notes |
|----------------------|------------------|-------|
| `ifcopenshell.api.run(cmd, model, **kw)` | `tool.Ifc.run(cmd, **kw)` | Bonsai wraps the API; model is implicit |
| `ifcopenshell.open(path)` | `IfcStore.get_file()` | NEVER open separately when Bonsai is active |
| `element.id()` | `obj.BIMProperties.ifc_definition_id` | Bidirectional mapping via IfcStore.id_map |
| `element.GlobalId` | `IfcStore.guid_map` | Bidirectional GUID-to-object lookup |

### Bonsai <-> Blender

| Bonsai Operation | Blender Equivalent | Sync Rule |
|-----------------|-------------------|-----------|
| `bpy.ops.bim.add_wall()` | Creates mesh + IFC entity | Automatic bidirectional sync |
| IFC placement change | `obj.location` / `obj.matrix_world` | MUST call `bpy.ops.bim.edit_object_placement()` |
| `pset.edit_pset()` | BIM property panels | Panel shows live IFC data |
| `IfcStore.edited_objs` | Modified Blender objects | Pending IFC changes queue |

---

## Version Compatibility Matrix

| Feature | IFC2X3 | IFC4 | IFC4X3 | Notes |
|---------|--------|------|--------|-------|
| OwnerHistory | REQUIRED | OPTIONAL | OPTIONAL | Set via `owner.set_user` before entity creation |
| IfcWallStandardCase | Available | DEPRECATED | DEPRECATED | Use IfcWall with PredefinedType instead |
| IfcFacility | N/A | N/A | Available | Generalization of IfcBuilding for infrastructure |
| IfcAlignment | N/A | N/A | Available | Linear infrastructure alignment |
| IfcDoorType | N/A (use IfcDoorStyle) | Available | Available | |
| IfcWindowType | N/A (use IfcWindowStyle) | Available | Available | |
| Property set types | IfcPropertySingleValue | + Bounded, Table, List | + Bounded, Table, List | |
| Material sets | IfcMaterialList | + IfcMaterialConstituentSet | + IfcMaterialConstituentSet | |

| Feature | Blender 3.x | Blender 4.x | Blender 5.x | Notes |
|---------|-------------|-------------|-------------|-------|
| bgl module | Available | Deprecated | REMOVED | Use `gpu` module for all drawing |
| Extension system | N/A | Available (4.2+) | Required | bl_info replaced by blender_manifest.toml |
| Bonsai compatibility | Pre-rename (BlenderBIM) | Bonsai v0.8.x | Bonsai v0.8.x+ | ALWAYS use `bonsai.*` imports |
| Grease Pencil API | Legacy | Rewritten (4.3) | New API | Complete API break in 4.3 |

---

## Common Operations Quick Reference

### Property Set Management Across Contexts

```python
# Standalone IfcOpenShell: Read properties
import ifcopenshell.util.element
psets = ifcopenshell.util.element.get_psets(element)
value = psets.get("Pset_WallCommon", {}).get("IsExternal")

# Standalone IfcOpenShell: Write properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=element, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset,
    properties={"IsExternal": True})

# Bonsai context: Read properties (same IfcOpenShell util)
model = IfcStore.get_file()
entity = tool.Ifc.get_entity(bpy.context.active_object)
psets = ifcopenshell.util.element.get_psets(entity)

# Bonsai context: Write properties (same API)
ifcopenshell.api.run("pset.edit_pset", model, pset=pset,
    properties={"IsExternal": True})
```

### Spatial Hierarchy Traversal

```python
# Get container of an element
import ifcopenshell.util.element
container = ifcopenshell.util.element.get_container(element)
# Returns: IfcBuildingStorey, IfcSpace, or None

# Get all elements in a storey
elements = ifcopenshell.util.element.get_decomposition(storey)

# Walk full spatial tree
def walk_spatial(element, depth=0):
    print("  " * depth + f"{element.is_a()}: {element.Name}")
    for rel in getattr(element, "IsDecomposedBy", []):
        for child in rel.RelatedObjects:
            walk_spatial(child, depth + 1)
    for rel in getattr(element, "ContainsElements", []):
        for child in rel.RelatedElements:
            walk_spatial(child, depth + 1)

project = model.by_type("IfcProject")[0]
walk_spatial(project)
```

---

## Reference Links

- [Cross-Technology API Methods](references/methods.md) — Complete integration point signatures
- [End-to-End Workflow Examples](references/examples.md) — Full working examples for common BIM workflows
- [Cross-Tech Anti-Patterns](references/anti-patterns.md) — Common integration mistakes and how to avoid them

## Sources

- IfcOpenShell Documentation: https://docs.ifcopenshell.org
- IfcOpenShell API Reference: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- Bonsai Documentation: https://docs.bonsaibim.org
- Blender Python API: https://docs.blender.org/api/current/index.html
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell
- OSArch Community: https://community.osarch.org
