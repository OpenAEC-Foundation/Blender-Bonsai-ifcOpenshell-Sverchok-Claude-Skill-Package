# End-to-End Workflow Examples

> Complete working examples for common cross-technology BIM workflows.
> Each example specifies its execution context, version requirements, and expected output.

---

## Example 1: Create IFC Model and Visualize in Blender (Two-Phase Workflow)

**Use case**: Generate an IFC building model programmatically, then load it into Blender for visualization.
**Context**: Phase 1 = standalone Python, Phase 2 = Blender Python (no Bonsai)
**Versions**: IfcOpenShell 0.8.x | IFC4 | Blender 4.x-5.x

### Phase 1: Create IFC Model (standalone)

```python
# File: create_model.py
# Run: python create_model.py
import ifcopenshell
import ifcopenshell.api
import numpy as np

# --- Step 1: Create file and project ---
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Office Building")
ifcopenshell.api.run("unit.assign_unit", model)

# --- Step 2: Geometric context ---
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# --- Step 3: Spatial hierarchy ---
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Main Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Block A")
ground = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
first = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[ground, first], relating_object=building)

# Set storey elevations
matrix_ground = np.eye(4)
matrix_ground[2][3] = 0.0
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=ground, matrix=matrix_ground)

matrix_first = np.eye(4)
matrix_first[2][3] = 3.5
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=first, matrix=matrix_first)

# --- Step 4: Create wall type with material layers ---
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="WT-200 Exterior")
layer_set = ifcopenshell.api.run("material.add_material_set", model,
    name="WT-200", set_type="IfcMaterialLayerSet")
brick = ifcopenshell.api.run("material.add_material", model,
    name="Brick", category="brick")
insulation = ifcopenshell.api.run("material.add_material", model,
    name="Insulation", category="mineral wool")

layer1 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=brick)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer1, attributes={"LayerThickness": 0.1})
layer2 = ifcopenshell.api.run("material.add_layer", model,
    layer_set=layer_set, material=insulation)
ifcopenshell.api.run("material.edit_layer", model,
    layer=layer2, attributes={"LayerThickness": 0.1})

ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], type="IfcMaterialLayerSet", material=layer_set)

# --- Step 5: Create walls on ground floor ---
wall_data = [
    ("Wall North", 0.0, 10.0, 0.0, 0.0, 0.0),     # x, length, y, z, rotation
    ("Wall East",  10.0, 8.0, 0.0, 0.0, 90.0),
    ("Wall South", 0.0, 10.0, 8.0, 0.0, 0.0),
    ("Wall West",  0.0, 8.0, 0.0, 0.0, 90.0),
]

import math
for name, x, length, y, z, angle_deg in wall_data:
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=name)
    rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
        context=body, length=length, height=3.0, thickness=0.2)
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=wall, representation=rep)

    angle = math.radians(angle_deg)
    matrix = np.array([
        [math.cos(angle), -math.sin(angle), 0, x],
        [math.sin(angle),  math.cos(angle), 0, y],
        [0,                0,               1, z],
        [0,                0,               0, 1]
    ])
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=wall, matrix=matrix)
    ifcopenshell.api.run("spatial.assign_container", model,
        relating_structure=ground, products=[wall])
    ifcopenshell.api.run("type.assign_type", model,
        related_objects=[wall], relating_type=wall_type)

    # Add standard property set
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")
    ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
        "IsExternal": True,
        "LoadBearing": True,
        "FireRating": "REI60"
    })

# --- Step 6: Add floor slab ---
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Floor Slab", predefined_type="FLOOR")
slab_rep = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context=body, depth=0.25,
    polyline=[(0.0, 0.0), (10.0, 0.0), (10.0, 8.0), (0.0, 8.0)])
ifcopenshell.api.run("geometry.assign_representation", model,
    product=slab, representation=slab_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=slab)
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=ground, products=[slab])

model.write("office_building.ifc")
print(f"Written: office_building.ifc ({len(model.by_type('IfcProduct'))} products)")
```

### Phase 2: Load into Blender (Blender Python, no Bonsai)

```python
# File: import_to_blender.py
# Run: blender --background --python import_to_blender.py
import bpy
import ifcopenshell
import ifcopenshell.geom

# Clear default scene
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()

# Open IFC file
ifc_file = ifcopenshell.open("office_building.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

# Create collection per IFC class
collections = {}

for product in ifc_file.by_type("IfcProduct"):
    if product.is_a() in ("IfcProject", "IfcSite", "IfcBuilding",
                           "IfcBuildingStorey"):
        continue  # Skip non-geometric spatial elements

    try:
        shape = ifcopenshell.geom.create_shape(settings, product)
    except Exception:
        continue  # Skip elements without geometry

    verts = shape.geometry.verts
    faces = shape.geometry.faces

    vertices = [(verts[i], verts[i+1], verts[i+2])
                for i in range(0, len(verts), 3)]
    triangles = [(faces[i], faces[i+1], faces[i+2])
                 for i in range(0, len(faces), 3)]

    if not vertices:
        continue

    # Get or create collection for this IFC class
    ifc_class = product.is_a()
    if ifc_class not in collections:
        col = bpy.data.collections.new(ifc_class)
        bpy.context.scene.collection.children.link(col)
        collections[ifc_class] = col

    mesh = bpy.data.meshes.new(name=product.Name or ifc_class)
    mesh.from_pydata(vertices, [], triangles)
    mesh.update()

    obj = bpy.data.objects.new(product.Name or ifc_class, mesh)
    collections[ifc_class].objects.link(obj)

    # Store IFC metadata as custom properties
    obj["ifc_global_id"] = product.GlobalId
    obj["ifc_class"] = ifc_class

# Save blend file
bpy.ops.wm.save_as_mainfile(filepath="office_building.blend")
print(f"Saved: office_building.blend ({len(bpy.data.objects)} objects)")
```

---

## Example 2: Property Set Audit Pipeline (Headless Batch)

**Use case**: Scan all IFC files in a directory, check for required property sets, generate CSV report.
**Context**: Standalone Python (no Blender)
**Versions**: IfcOpenShell 0.8.x | IFC4

```python
# File: audit_psets.py
# Run: python audit_psets.py /path/to/models
import ifcopenshell
import ifcopenshell.util.element
import csv
import sys
from pathlib import Path

# Define required property sets per IFC class
REQUIRED_PSETS = {
    "IfcWall": {
        "Pset_WallCommon": ["IsExternal", "LoadBearing", "FireRating"],
    },
    "IfcSlab": {
        "Pset_SlabCommon": ["IsExternal", "LoadBearing"],
    },
    "IfcColumn": {
        "Pset_ColumnCommon": ["LoadBearing"],
    },
    "IfcDoor": {
        "Pset_DoorCommon": ["IsExternal", "FireRating"],
    },
    "IfcWindow": {
        "Pset_WindowCommon": ["IsExternal", "ThermalTransmittance"],
    },
}

def audit_file(filepath: str) -> list[dict]:
    """Audit a single IFC file for missing property sets."""
    findings = []
    model = ifcopenshell.open(filepath)

    for ifc_class, pset_reqs in REQUIRED_PSETS.items():
        for element in model.by_type(ifc_class):
            psets = ifcopenshell.util.element.get_psets(element)

            for pset_name, required_props in pset_reqs.items():
                pset_data = psets.get(pset_name, {})

                if not pset_data:
                    findings.append({
                        "file": Path(filepath).name,
                        "element_id": element.id(),
                        "global_id": element.GlobalId,
                        "element_name": element.Name or "(unnamed)",
                        "ifc_class": ifc_class,
                        "issue": f"Missing pset: {pset_name}",
                        "severity": "ERROR",
                    })
                    continue

                for prop_name in required_props:
                    if prop_name not in pset_data or pset_data[prop_name] is None:
                        findings.append({
                            "file": Path(filepath).name,
                            "element_id": element.id(),
                            "global_id": element.GlobalId,
                            "element_name": element.Name or "(unnamed)",
                            "ifc_class": ifc_class,
                            "issue": f"Missing property: {pset_name}.{prop_name}",
                            "severity": "WARNING",
                        })

    return findings

def main():
    input_dir = Path(sys.argv[1]) if len(sys.argv) > 1 else Path(".")
    all_findings = []

    for ifc_path in sorted(input_dir.glob("*.ifc")):
        print(f"Auditing: {ifc_path.name}")
        findings = audit_file(str(ifc_path))
        all_findings.extend(findings)
        print(f"  → {len(findings)} issues found")

    # Write CSV report
    output_path = input_dir / "pset_audit_report.csv"
    with open(output_path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=[
            "file", "severity", "ifc_class", "element_name",
            "global_id", "element_id", "issue"
        ])
        writer.writeheader()
        writer.writerows(all_findings)

    print(f"\nTotal: {len(all_findings)} issues across {len(list(input_dir.glob('*.ifc')))} files")
    print(f"Report: {output_path}")

if __name__ == "__main__":
    main()
```

---

## Example 3: IFC Enrichment — Add Classification to Existing Model

**Use case**: Read an existing IFC file, add NL-SfB classification references to all walls and slabs.
**Context**: Standalone Python
**Versions**: IfcOpenShell 0.8.x | IFC4

```python
# File: add_classifications.py
# Run: python add_classifications.py model.ifc
import ifcopenshell
import ifcopenshell.api
import sys

def add_nlsfb_classifications(filepath: str, output_path: str):
    model = ifcopenshell.open(filepath)

    # Create NL-SfB classification system
    classification = ifcopenshell.api.run(
        "classification.add_classification", model,
        classification="NL-SfB")

    # Define NL-SfB mappings per IFC class
    nlsfb_map = {
        "IfcWall":    ("21", "Buitenwanden"),
        "IfcSlab":    ("23", "Vloeren"),
        "IfcColumn":  ("22", "Binnenkolommen"),
        "IfcBeam":    ("22", "Liggers"),
        "IfcRoof":    ("27", "Daken"),
        "IfcDoor":    ("31", "Buitendeuren"),
        "IfcWindow":  ("31", "Vensters"),
        "IfcStair":   ("24", "Trappen"),
    }

    enriched_count = 0
    for ifc_class, (code, description) in nlsfb_map.items():
        elements = model.by_type(ifc_class)
        if elements:
            ifcopenshell.api.run("classification.add_reference", model,
                products=elements,
                classification=classification,
                identification=code,
                name=description)
            enriched_count += len(elements)

    model.write(output_path)
    print(f"Enriched {enriched_count} elements with NL-SfB classification")
    print(f"Written: {output_path}")

if __name__ == "__main__":
    input_file = sys.argv[1]
    output_file = input_file.replace(".ifc", "_classified.ifc")
    add_nlsfb_classifications(input_file, output_file)
```

---

## Example 4: Bonsai Script — Automated Wall Layout from CSV

**Use case**: Read wall definitions from CSV, create walls in Bonsai with proper IFC properties.
**Context**: Blender with Bonsai loaded
**Versions**: Bonsai v0.8.x | Blender 4.2+ | IFC4
**Run**: blender --python create_walls_from_csv.py

```python
# File: create_walls_from_csv.py
# Run: blender --python create_walls_from_csv.py
import bpy
import csv
import math
import numpy as np
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

# --- Step 1: Create new project if none exists ---
if IfcStore.get_file() is None:
    bpy.ops.bim.create_project()

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("Failed to create IFC project")

# Get first storey (or create one)
storeys = model.by_type("IfcBuildingStorey")
if not storeys:
    raise RuntimeError("No IfcBuildingStorey found in project")
storey = storeys[0]

# Get Body context
contexts = model.by_type("IfcGeometricRepresentationSubContext")
body = next((c for c in contexts if c.ContextIdentifier == "Body"), None)
if body is None:
    raise RuntimeError("No Body subcontext found")

# --- Step 2: Read wall data from CSV ---
# CSV format: name,x,y,length,height,thickness,angle_deg,is_external,fire_rating
wall_definitions = [
    # Example data (replace with csv.reader for file input)
    {"name": "Wall-N-001", "x": 0.0, "y": 0.0, "length": 10.0,
     "height": 3.0, "thickness": 0.2, "angle": 0.0,
     "is_external": True, "fire_rating": "REI60"},
    {"name": "Wall-E-001", "x": 10.0, "y": 0.0, "length": 8.0,
     "height": 3.0, "thickness": 0.2, "angle": 90.0,
     "is_external": True, "fire_rating": "REI60"},
    {"name": "Wall-S-001", "x": 0.0, "y": 8.0, "length": 10.0,
     "height": 3.0, "thickness": 0.2, "angle": 0.0,
     "is_external": True, "fire_rating": "REI60"},
    {"name": "Wall-INT-001", "x": 5.0, "y": 0.0, "length": 8.0,
     "height": 3.0, "thickness": 0.15, "angle": 90.0,
     "is_external": False, "fire_rating": "EI30"},
]

# --- Step 3: Create walls ---
for wall_def in wall_definitions:
    # Create wall entity
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=wall_def["name"])

    # Add geometry
    rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
        context=body,
        length=wall_def["length"],
        height=wall_def["height"],
        thickness=wall_def["thickness"])
    ifcopenshell.api.run("geometry.assign_representation", model,
        product=wall, representation=rep)

    # Set placement
    angle = math.radians(wall_def["angle"])
    matrix = np.array([
        [math.cos(angle), -math.sin(angle), 0, wall_def["x"]],
        [math.sin(angle),  math.cos(angle), 0, wall_def["y"]],
        [0, 0, 1, 0],
        [0, 0, 0, 1]
    ])
    ifcopenshell.api.run("geometry.edit_object_placement", model,
        product=wall, matrix=matrix)

    # Assign to storey
    ifcopenshell.api.run("spatial.assign_container", model,
        relating_structure=storey, products=[wall])

    # Add property set
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")
    ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
        "IsExternal": wall_def["is_external"],
        "LoadBearing": True,
        "FireRating": wall_def["fire_rating"]
    })

print(f"Created {len(wall_definitions)} walls")

# --- Step 4: Save ---
bpy.ops.bim.save_project()
```

---

## Example 5: Geometry Extraction with Material Mapping

**Use case**: Extract IFC geometry and map IFC materials to Blender materials with colors.
**Context**: Blender Python (no Bonsai)
**Versions**: Blender 4.x-5.x | IfcOpenShell 0.8.x

```python
# File: ifc_to_blender_materials.py
# Run: blender --background --python ifc_to_blender_materials.py
import bpy
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.element

# Material color mapping (IFC material category → RGBA)
MATERIAL_COLORS = {
    "concrete": (0.7, 0.7, 0.7, 1.0),
    "steel":    (0.5, 0.5, 0.6, 1.0),
    "brick":    (0.7, 0.3, 0.2, 1.0),
    "glass":    (0.4, 0.6, 0.8, 0.3),
    "wood":     (0.6, 0.4, 0.2, 1.0),
    "default":  (0.8, 0.8, 0.8, 1.0),
}

def get_material_category(element) -> str:
    """Extract material category from IFC element."""
    material = ifcopenshell.util.element.get_material(element)
    if material is None:
        return "default"
    if material.is_a("IfcMaterial"):
        return (material.Category or "default").lower()
    if material.is_a("IfcMaterialLayerSet"):
        # Use first layer's material category
        if material.MaterialLayers:
            first_mat = material.MaterialLayers[0].Material
            return (first_mat.Category or "default").lower() if first_mat else "default"
    return "default"

def get_or_create_blender_material(category: str) -> bpy.types.Material:
    """Get or create a Blender material for the given IFC material category."""
    mat_name = f"IFC_{category}"
    mat = bpy.data.materials.get(mat_name)
    if mat:
        return mat

    mat = bpy.data.materials.new(name=mat_name)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes.get("Principled BSDF")
    color = MATERIAL_COLORS.get(category, MATERIAL_COLORS["default"])
    if bsdf:
        bsdf.inputs["Base Color"].default_value = color
        if color[3] < 1.0:
            mat.blend_method = 'BLEND'  # Blender 3.x-4.x
            bsdf.inputs["Alpha"].default_value = color[3]
    return mat

# Clear scene
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()

# Load IFC
ifc_file = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

for product in ifc_file.by_type("IfcBuildingElement"):
    try:
        shape = ifcopenshell.geom.create_shape(settings, product)
    except Exception:
        continue

    verts = shape.geometry.verts
    faces = shape.geometry.faces
    vertices = [(verts[i], verts[i+1], verts[i+2])
                for i in range(0, len(verts), 3)]
    triangles = [(faces[i], faces[i+1], faces[i+2])
                 for i in range(0, len(faces), 3)]

    if not vertices:
        continue

    mesh = bpy.data.meshes.new(name=product.Name or product.is_a())
    mesh.from_pydata(vertices, [], triangles)
    mesh.update()

    obj = bpy.data.objects.new(product.Name or product.is_a(), mesh)
    bpy.context.collection.objects.link(obj)

    # Map IFC material to Blender material
    category = get_material_category(product)
    blender_mat = get_or_create_blender_material(category)
    obj.data.materials.append(blender_mat)

bpy.ops.wm.save_as_mainfile(filepath="model_with_materials.blend")
```

---

## Example 6: Spatial Tree Report (Cross-Context)

**Use case**: Generate a spatial hierarchy report from an IFC file, showing containment and element counts.
**Context**: Standalone Python
**Versions**: IfcOpenShell 0.8.x | IFC2X3/IFC4/IFC4X3

```python
# File: spatial_report.py
# Run: python spatial_report.py model.ifc
import ifcopenshell
import ifcopenshell.util.element
import sys

def count_elements_by_type(container, model) -> dict[str, int]:
    """Count elements by IFC class within a spatial container."""
    counts = {}
    for rel in getattr(container, "ContainsElements", []):
        for element in rel.RelatedElements:
            ifc_class = element.is_a()
            counts[ifc_class] = counts.get(ifc_class, 0) + 1
    return counts

def print_spatial_tree(element, model, depth=0):
    """Recursively print spatial hierarchy with element counts."""
    prefix = "  " * depth
    name = element.Name or "(unnamed)"
    print(f"{prefix}{element.is_a()}: {name}")

    # Count contained elements
    counts = count_elements_by_type(element, model)
    if counts:
        for ifc_class, count in sorted(counts.items()):
            print(f"{prefix}  └ {ifc_class}: {count}")

    # Recurse into decomposition (Site → Building → Storey)
    for rel in getattr(element, "IsDecomposedBy", []):
        for child in rel.RelatedObjects:
            print_spatial_tree(child, model, depth + 1)

def main():
    filepath = sys.argv[1] if len(sys.argv) > 1 else "model.ifc"
    model = ifcopenshell.open(filepath)

    print(f"=== Spatial Report: {filepath} ===")
    print(f"Schema: {model.schema}")
    print(f"Total entities: {len(list(model))}")
    print()

    for project in model.by_type("IfcProject"):
        print_spatial_tree(project, model)

if __name__ == "__main__":
    main()
```

**Example output**:
```
=== Spatial Report: office_building.ifc ===
Schema: IFC4
Total entities: 142

IfcProject: Office Building
  IfcSite: Main Site
    IfcBuilding: Office Block A
      IfcBuildingStorey: Ground Floor
        └ IfcSlab: 1
        └ IfcWall: 4
      IfcBuildingStorey: First Floor
```

---

## Sources

- IfcOpenShell Documentation: https://docs.ifcopenshell.org
- IfcOpenShell API: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- Bonsai Documentation: https://docs.bonsaibim.org
- Blender Python API: https://docs.blender.org/api/current/index.html
