# End-to-End Workflow Examples

> Complete orchestrated workflow examples spanning multiple AEC technologies.
> Each example specifies the execution context, required technologies, and version constraints.

---

## Example 1: Create IFC Building Model from Scratch (Standalone)

**Context**: Standalone Python (no Blender)
**Technologies**: IfcOpenShell 0.8.x
**Schema**: IFC4
**Pipeline**: IFC-First Workflow (Pattern A)

```python
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element
import numpy as np

# ============================================================
# Step 1: Initialize IFC file with project, units, and contexts
# ============================================================
model = ifcopenshell.api.run("project.create_file", version="IFC4")

project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Residential Building")

ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# ============================================================
# Step 2: Build spatial hierarchy
# ============================================================
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Construction Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
ground_floor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
first_floor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[ground_floor, first_floor], relating_object=building)

# Set storey elevations via placement
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=ground_floor, matrix=np.eye(4))

first_floor_matrix = np.eye(4)
first_floor_matrix[2, 3] = 3.0  # Z offset = 3.0m
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=first_floor, matrix=first_floor_matrix)

# ============================================================
# Step 3: Create element types
# ============================================================
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Exterior Wall 200mm")
slab_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlabType", name="Floor Slab 250mm")

# ============================================================
# Step 4: Create walls on ground floor
# ============================================================
wall_data = [
    ("Wall North", 0.0, 0.0, 0.0, 10.0),    # name, x, y, rotation_deg, length
    ("Wall East",  10.0, 0.0, 90.0, 8.0),
    ("Wall South", 10.0, 8.0, 180.0, 10.0),
    ("Wall West",  0.0, 8.0, 270.0, 8.0),
]

for name, x, y, rot_deg, length in wall_data:
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=name)

    rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
        context=body, length=length, height=3.0, thickness=0.2)
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=wall, representation=rep)

    rot_rad = np.radians(rot_deg)
    matrix = np.array([
        [np.cos(rot_rad), -np.sin(rot_rad), 0.0, x],
        [np.sin(rot_rad),  np.cos(rot_rad), 0.0, y],
        [0.0,              0.0,             1.0, 0.0],
        [0.0,              0.0,             0.0, 1.0]
    ])
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=wall, matrix=matrix)

    ifcopenshell.api.run("spatial.assign_container", model,
        relating_structure=ground_floor, products=[wall])
    ifcopenshell.api.run("type.assign_type", model,
        related_objects=[wall], relating_type=wall_type)

# ============================================================
# Step 5: Add property sets
# ============================================================
for wall in model.by_type("IfcWall"):
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")
    ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
        "IsExternal": True,
        "LoadBearing": True,
        "FireRating": "REI60"
    })

# ============================================================
# Step 6: Write output
# ============================================================
model.write("residential_building.ifc")
print(f"Created IFC file with {len(model.by_type('IfcWall'))} walls")
```

---

## Example 2: IFC Geometry to Blender Visualization (No Bonsai)

**Context**: Blender Python (bpy available, Bonsai NOT loaded)
**Technologies**: IfcOpenShell 0.8.x + Blender 3.x-5.x
**Pipeline**: Bridge Pattern (IfcOpenShell -> Blender)

```python
# Run inside Blender: blender --python this_script.py
# OR from Blender's scripting workspace
import bpy
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.element

# ============================================================
# Step 1: Open IFC file with IfcOpenShell (standalone, no Bonsai)
# ============================================================
ifc_file = ifcopenshell.open("building.ifc")
schema = ifc_file.schema
print(f"Loaded IFC file with schema: {schema}")

# ============================================================
# Step 2: Configure geometry extraction settings
# ============================================================
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

# ============================================================
# Step 3: Create Blender collections matching spatial hierarchy
# ============================================================
root_collection = bpy.context.scene.collection

for storey in ifc_file.by_type("IfcBuildingStorey"):
    storey_col = bpy.data.collections.new(storey.Name or f"Storey_{storey.id()}")
    root_collection.children.link(storey_col)

    # Get elements contained in this storey
    for rel in getattr(storey, "ContainsElements", []):
        for element in rel.RelatedElements:
            if element.Representation is None:
                continue

            try:
                shape = ifcopenshell.geom.create_shape(settings, element)
            except RuntimeError:
                print(f"Skipping {element.is_a()} #{element.id()}: geometry error")
                continue

            verts = shape.geometry.verts
            faces = shape.geometry.faces

            vertices = [(verts[i], verts[i+1], verts[i+2])
                        for i in range(0, len(verts), 3)]
            triangles = [(faces[i], faces[i+1], faces[i+2])
                         for i in range(0, len(faces), 3)]

            mesh_name = f"{element.is_a()}_{element.Name or element.id()}"
            mesh = bpy.data.meshes.new(name=mesh_name)
            mesh.from_pydata(vertices, [], triangles)
            mesh.update()

            obj = bpy.data.objects.new(mesh_name, mesh)
            storey_col.objects.link(obj)

            # Store IFC metadata as custom properties
            obj["ifc_global_id"] = element.GlobalId
            obj["ifc_class"] = element.is_a()

# ============================================================
# Step 4: Apply materials based on IFC class
# ============================================================
class_colors = {
    "IfcWall": (0.85, 0.8, 0.7, 1.0),
    "IfcSlab": (0.6, 0.6, 0.6, 1.0),
    "IfcColumn": (0.5, 0.5, 0.55, 1.0),
    "IfcBeam": (0.55, 0.5, 0.45, 1.0),
    "IfcDoor": (0.5, 0.3, 0.15, 1.0),
    "IfcWindow": (0.7, 0.85, 0.95, 0.5),
}

for obj in bpy.data.objects:
    ifc_class = obj.get("ifc_class")
    if ifc_class and ifc_class in class_colors:
        mat_name = f"MAT_{ifc_class}"
        mat = bpy.data.materials.get(mat_name)
        if mat is None:
            mat = bpy.data.materials.new(name=mat_name)
            mat.use_nodes = True
            bsdf = mat.node_tree.nodes.get("Principled BSDF")
            if bsdf:
                bsdf.inputs["Base Color"].default_value = class_colors[ifc_class]
        obj.data.materials.append(mat)

print("IFC model loaded into Blender viewport")
```

---

## Example 3: Bonsai BIM Authoring with Property Sets

**Context**: Blender with Bonsai addon loaded
**Technologies**: Bonsai v0.8.x + IfcOpenShell 0.8.x + Blender 4.2+
**Pipeline**: Bonsai-Native Workflow (Pattern C)

```python
# Run inside Blender with Bonsai addon enabled
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

# ============================================================
# Step 1: Create or verify IFC project
# ============================================================
model = IfcStore.get_file()
if model is None:
    bpy.ops.bim.create_project(schema="IFC4")
    model = IfcStore.get_file()
    if model is None:
        raise RuntimeError("Failed to create IFC project")

print(f"Working with schema: {model.schema}")

# ============================================================
# Step 2: Verify spatial structure exists
# ============================================================
storeys = model.by_type("IfcBuildingStorey")
if not storeys:
    raise RuntimeError("No building storey found. Create spatial structure first.")
target_storey = storeys[0]

# ============================================================
# Step 3: Create wall via IfcOpenShell API (with Bonsai sync)
# ============================================================
# Get body context
contexts = model.by_type("IfcGeometricRepresentationSubContext")
body_context = None
for ctx in contexts:
    if ctx.ContextIdentifier == "Body":
        body_context = ctx
        break

if body_context is None:
    raise RuntimeError("No Body sub-context found")

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Interior Wall 001")

rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body_context, length=4.0, height=2.8, thickness=0.1)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=rep)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall)

ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=target_storey, products=[wall])

# ============================================================
# Step 4: Add property sets
# ============================================================
pset_common = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset_common, properties={
    "IsExternal": False,
    "LoadBearing": False,
    "FireRating": "EI30",
    "AcousticRating": "Rw 45 dB"
})

pset_custom = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Custom_WallProperties")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset_custom, properties={
    "ProjectPhase": "Phase 1",
    "CostCenter": "CC-2024-001"
})

# ============================================================
# Step 5: Assign type
# ============================================================
wall_types = model.by_type("IfcWallType")
interior_type = None
for wt in wall_types:
    if "Interior" in (wt.Name or ""):
        interior_type = wt
        break

if interior_type is None:
    interior_type = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWallType", name="Interior Partition 100mm")

ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=interior_type)

# ============================================================
# Step 6: Verify result
# ============================================================
psets = ifcopenshell.util.element.get_psets(wall)
container = ifcopenshell.util.element.get_container(wall)
assigned_type = ifcopenshell.util.element.get_type(wall)

print(f"Wall: {wall.Name}")
print(f"Container: {container.Name if container else 'NONE'}")
print(f"Type: {assigned_type.Name if assigned_type else 'NONE'}")
print(f"Property sets: {list(psets.keys())}")
```

---

## Example 4: Batch IFC Validation and Report (Headless)

**Context**: Standalone Python
**Technologies**: IfcOpenShell 0.8.x
**Pipeline**: Pipeline 4 (IFC Analysis + Report)

```python
import ifcopenshell
import ifcopenshell.util.element
import json
from pathlib import Path

def validate_ifc_file(filepath: str) -> dict:
    """Validate an IFC file and return structured report."""
    report = {
        "file": filepath,
        "schema": None,
        "errors": [],
        "warnings": [],
        "stats": {}
    }

    model = ifcopenshell.open(filepath)
    report["schema"] = model.schema

    # === Check 1: Project exists ===
    projects = model.by_type("IfcProject")
    if not projects:
        report["errors"].append("No IfcProject entity found")
        return report
    report["stats"]["project_name"] = projects[0].Name

    # === Check 2: Spatial hierarchy ===
    sites = model.by_type("IfcSite")
    buildings = model.by_type("IfcBuilding")
    storeys = model.by_type("IfcBuildingStorey")

    if not sites:
        report["errors"].append("No IfcSite found")
    if not buildings:
        report["errors"].append("No IfcBuilding found")
    if not storeys:
        report["warnings"].append("No IfcBuildingStorey found")

    report["stats"]["sites"] = len(sites)
    report["stats"]["buildings"] = len(buildings)
    report["stats"]["storeys"] = len(storeys)

    # === Check 3: Elements without spatial container ===
    orphan_count = 0
    element_classes = ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam",
                       "IfcDoor", "IfcWindow", "IfcStair", "IfcRoof"]
    for cls in element_classes:
        for elem in model.by_type(cls):
            container = ifcopenshell.util.element.get_container(elem)
            if container is None:
                orphan_count += 1
                report["warnings"].append(
                    f"{cls} '{elem.Name}' (#{elem.id()}) has no spatial container")

    report["stats"]["orphan_elements"] = orphan_count

    # === Check 4: Elements without geometry ===
    no_geom_count = 0
    skip_classes = {"IfcSite", "IfcBuilding", "IfcBuildingStorey",
                    "IfcSpace", "IfcProject"}
    for product in model.by_type("IfcProduct"):
        if product.is_a() in skip_classes:
            continue
        if hasattr(product, "Representation") and product.Representation is None:
            no_geom_count += 1
            report["warnings"].append(
                f"{product.is_a()} '{product.Name}' (#{product.id()}) has no geometry")

    report["stats"]["elements_without_geometry"] = no_geom_count

    # === Check 5: Units assigned ===
    project = projects[0]
    if not project.UnitsInContext:
        report["errors"].append("No units assigned to IfcProject")

    # === Check 6: IFC2X3 OwnerHistory ===
    if model.schema == "IFC2X3":
        for entity in model.by_type("IfcRoot"):
            if entity.OwnerHistory is None:
                report["errors"].append(
                    f"IFC2X3 requires OwnerHistory on {entity.is_a()} "
                    f"'{entity.Name}' (#{entity.id()})")
                break  # Report first occurrence only

    # === Element counts ===
    report["stats"]["total_elements"] = len(model.by_type("IfcElement"))
    report["stats"]["walls"] = len(model.by_type("IfcWall"))
    report["stats"]["slabs"] = len(model.by_type("IfcSlab"))
    report["stats"]["columns"] = len(model.by_type("IfcColumn"))

    return report


def batch_validate(input_dir: str, output_file: str):
    """Validate all IFC files in a directory."""
    input_path = Path(input_dir)
    reports = []

    for ifc_file in sorted(input_path.glob("*.ifc")):
        print(f"Validating: {ifc_file.name}")
        report = validate_ifc_file(str(ifc_file))
        reports.append(report)

    with open(output_file, "w") as f:
        json.dump(reports, f, indent=2)

    # Summary
    total_errors = sum(len(r["errors"]) for r in reports)
    total_warnings = sum(len(r["warnings"]) for r in reports)
    print(f"\nValidated {len(reports)} files: {total_errors} errors, {total_warnings} warnings")
    print(f"Report written to: {output_file}")


# Usage:
# batch_validate("/path/to/ifc_files/", "validation_report.json")
```

---

## Example 5: Multi-Technology Pipeline — Author in Bonsai, Post-Process Standalone

**Context**: Two-phase workflow (Bonsai + standalone IfcOpenShell)
**Technologies**: Bonsai v0.8.x (phase 1) + IfcOpenShell standalone (phase 2)
**Pipeline**: Pipeline 5 (Bonsai Authoring + Standalone Post-Processing)

### Phase 1: Bonsai authoring (inside Blender)

```python
# Run inside Blender with Bonsai loaded
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Author walls using Bonsai operators
bpy.ops.bim.add_wall()

# Save to disk — MUST save before standalone post-processing
bpy.ops.bim.save_project()
saved_path = IfcStore.path
print(f"Saved IFC to: {saved_path}")
# Output: saved_path = "/path/to/project.ifc"
```

### Phase 2: Standalone post-processing (separate Python process)

```python
# Run as standalone Python script — NOT inside Blender
# NEVER run this while Bonsai still has the file open
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.open("project.ifc")

# Add classification to all walls
classification = ifcopenshell.api.run("classification.add_classification", model,
    classification="Uniclass2015")

for wall in model.by_type("IfcWall"):
    # Add classification reference
    ifcopenshell.api.run("classification.add_reference", model,
        products=[wall],
        classification=classification,
        identification="EF_25_10",
        name="Walls")

    # Add quantity set if missing
    psets = ifcopenshell.util.element.get_psets(wall, psets_only=False)
    if "Qto_WallBaseQuantities" not in psets:
        qto = ifcopenshell.api.run("pset.add_qto", model,
            product=wall, name="Qto_WallBaseQuantities")
        ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
            "Length": 5.0,
            "Height": 3.0,
            "Width": 0.2,
            "GrossVolume": 3.0,
            "NetVolume": 2.8,
            "GrossSideArea": 15.0,
            "NetSideArea": 14.0,
        })

model.write("project_enriched.ifc")
print(f"Post-processed {len(model.by_type('IfcWall'))} walls")
```

---

## Example 6: Context-Aware Script (Runs in Any Environment)

**Context**: Auto-detecting (standalone, Blender, or Bonsai)
**Technologies**: Adapts to available environment

```python
"""AEC script that adapts to the available execution context."""

def detect_context():
    ctx = {"blender": False, "bonsai": False, "ifcopenshell": False}
    try:
        import bpy
        ctx["blender"] = True
        ctx["blender_version"] = tuple(bpy.app.version)
    except ImportError:
        pass
    try:
        import ifcopenshell
        ctx["ifcopenshell"] = True
    except ImportError:
        pass
    if ctx["blender"]:
        try:
            from bonsai.bim.ifc import IfcStore
            ctx["bonsai"] = True
            ctx["bonsai_has_file"] = IfcStore.get_file() is not None
        except ImportError:
            pass
    return ctx


def get_ifc_model(filepath=None):
    """Get IFC model from best available source."""
    ctx = detect_context()

    if ctx.get("bonsai") and ctx.get("bonsai_has_file"):
        from bonsai.bim.ifc import IfcStore
        return IfcStore.get_file(), "bonsai"

    if ctx["ifcopenshell"] and filepath:
        import ifcopenshell
        return ifcopenshell.open(filepath), "standalone"

    raise RuntimeError(
        "No IFC model available. Provide a filepath or load a Bonsai project.")


def count_elements(model):
    """Count elements by IFC class."""
    import ifcopenshell.util.element
    counts = {}
    schema = model.schema

    # Schema-aware element class
    base_class = "IfcBuiltElement" if schema == "IFC4X3" else "IfcBuildingElement"
    for element in model.by_type(base_class):
        cls = element.is_a()
        counts[cls] = counts.get(cls, 0) + 1

    return counts


# Usage — works in any context:
# model, source = get_ifc_model("building.ifc")
# print(f"Loaded from: {source}")
# print(count_elements(model))
```
