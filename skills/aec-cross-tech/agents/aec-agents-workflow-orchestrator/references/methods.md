# Workflow Methods Reference

> Technology-specific entry points and method signatures for AEC workflow orchestration.
> Version coverage: Blender 3.x-5.x | IfcOpenShell 0.8.x | IFC2X3/IFC4/IFC4X3 | Bonsai v0.8.x

---

## 1. IfcOpenShell Entry Points (Standalone Context)

### File Operations

```python
# Open existing IFC file
# Version: IfcOpenShell 0.8.x | Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell
model = ifcopenshell.open("path/to/file.ifc")
schema = model.schema  # "IFC2X3", "IFC4", or "IFC4X3"

# Create new IFC file
# Version: IfcOpenShell 0.8.x
import ifcopenshell.api
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Write IFC file
model.write("output.ifc")
```

### Project Initialization Sequence

ALWAYS execute these steps in order when creating a new IFC file:

```python
# Step 1: Create file
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Step 2: Create project entity
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project Name")

# Step 3: Assign units (REQUIRED before geometry)
ifcopenshell.api.run("unit.assign_unit", model)

# Step 4: Create geometric contexts (REQUIRED before representations)
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)
```

### Spatial Hierarchy Construction

```python
# Build spatial hierarchy (ALWAYS top-down)
# Schema: IFC2X3, IFC4
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site Name")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building Name")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Floor Name")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Schema: IFC4X3 (infrastructure)
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Road Name")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[road], relating_object=site)
```

### Element Creation

```python
# Create element + geometry + placement + containment
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Geometry (requires body context from initialization)
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=rep)

# Placement (identity matrix = origin)
import numpy as np
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=np.eye(4))

# Spatial containment
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

### Property Management

```python
# Add property set
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=element, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI60"
})

# Read properties
import ifcopenshell.util.element
psets = ifcopenshell.util.element.get_psets(element)
# Returns: {"Pset_WallCommon": {"id": 42, "IsExternal": True, ...}}
```

### Element Query Methods

```python
# Query by type
walls = model.by_type("IfcWall")           # All walls (tuple)
elements = model.by_type("IfcElement")     # All elements (includes subtypes)

# Query by ID
element = model.by_id(42)                  # By step file ID

# Query by GlobalId
element = model.by_guid("2TxLOH2Df5OBt...")

# Type check with inheritance
element.is_a("IfcWall")      # True for IfcWall instances
element.is_a("IfcElement")   # True for ANY element subtype

# Schema-aware element class
if model.schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")
```

### Utility Functions

```python
import ifcopenshell.util.element

# Get spatial container
container = ifcopenshell.util.element.get_container(element)

# Get element type
element_type = ifcopenshell.util.element.get_type(element)

# Get material
material = ifcopenshell.util.element.get_material(element)

# Get all property sets as dict
psets = ifcopenshell.util.element.get_psets(element)

# Get decomposition (children of spatial element)
children = ifcopenshell.util.element.get_decomposition(storey)
```

---

## 2. Blender Entry Points (Blender Python Context)

### Geometry Creation

```python
# Create mesh from vertices and faces
# Version: Blender 3.x/4.x/5.x
import bpy

mesh = bpy.data.meshes.new("MeshName")
verts = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
faces = [(0, 1, 2, 3)]
mesh.from_pydata(verts, [], faces)
mesh.update()  # ALWAYS call after from_pydata

obj = bpy.data.objects.new("ObjectName", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED — links to scene
```

### Evaluated Mesh Access (Post-Modifier)

```python
# Get geometry after modifiers are applied
# Version: Blender 3.x/4.x/5.x
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()

verts = [tuple(v.co) for v in mesh_eval.vertices]
faces = [tuple(p.vertices) for p in mesh_eval.polygons]

obj_eval.to_mesh_clear()  # ALWAYS clean up
```

### Object Selection and Activation

```python
# Version: Blender 3.x/4.x/5.x
obj.select_set(True)
bpy.context.view_layer.objects.active = obj
```

### Context Override (Version-Critical)

```python
# Blender 3.2+ and 4.x/5.x — CORRECT
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Subsurf")

# Blender 3.x ONLY (BROKEN in 4.0+)
# override = bpy.context.copy()  # DO NOT USE in 4.0+
```

### Version Detection

```python
# Version: Blender 3.x/4.x/5.x
major, minor, patch = bpy.app.version

if bpy.app.version >= (4, 0, 0):
    # Use temp_override, NodeTree.interface
    pass
if bpy.app.version >= (5, 0, 0):
    # Use gpu module exclusively (bgl REMOVED)
    pass
```

---

## 3. Bonsai Entry Points (Bonsai-Loaded Context)

### IfcStore Access

```python
# Version: Bonsai v0.8.x | Blender 4.2+
from bonsai.bim.ifc import IfcStore

# ALWAYS guard against None
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# Key IfcStore attributes
IfcStore.path       # str: path to .ifc file
IfcStore.file       # ifcopenshell.file or None
IfcStore.schema     # "IFC2X3", "IFC4", or "IFC4X3"
IfcStore.id_map     # dict: {ifc_id -> blender_object}
IfcStore.guid_map   # dict: {GlobalId -> blender_object}
```

### Entity-Object Mapping

```python
# Blender object -> IFC entity
obj = bpy.context.active_object
ifc_id = obj.BIMObjectProperties.ifc_definition_id
if ifc_id == 0:
    # Object is not linked to IFC
    pass
else:
    entity = model.by_id(ifc_id)

# IFC entity -> Blender object
blender_obj = IfcStore.id_map.get(entity.id())
```

### Bonsai Operators (bpy.ops.bim.*)

```python
# Project management
bpy.ops.bim.create_project(schema="IFC4")
bpy.ops.bim.save_project()
bpy.ops.bim.load_project(filepath="model.ifc")

# Element creation
bpy.ops.bim.add_wall()
bpy.ops.bim.add_slab()
bpy.ops.bim.add_column()

# Spatial structure
bpy.ops.bim.add_building()
bpy.ops.bim.add_storey()
bpy.ops.bim.assign_container()

# Placement sync (REQUIRED after manual transform changes)
bpy.ops.bim.edit_object_placement()
```

---

## 4. Geometry Bridge Methods

### IfcOpenShell Geometry Processing

```python
# Extract geometry from IFC element
# Version: IfcOpenShell 0.8.x
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)   # World coordinates
# settings.set(settings.USE_BREP_DATA, True)    # BRep instead of triangulated

shape = ifcopenshell.geom.create_shape(settings, element)
verts = shape.geometry.verts   # Flat: [x0,y0,z0, x1,y1,z1, ...]
faces = shape.geometry.faces   # Flat: [i0,i1,i2, i3,i4,i5, ...]

# Reshape for Blender
vertices = [(verts[i], verts[i+1], verts[i+2]) for i in range(0, len(verts), 3)]
triangles = [(faces[i], faces[i+1], faces[i+2]) for i in range(0, len(faces), 3)]
```

### Batch Geometry Processing with Iterator

```python
# Process all elements efficiently
# Version: IfcOpenShell 0.8.x
import ifcopenshell.geom
import multiprocessing

settings = ifcopenshell.geom.settings()
iterator = ifcopenshell.geom.iterator(
    settings, ifc_file, multiprocessing.cpu_count())

if iterator.initialize():
    while True:
        shape = iterator.get()
        # Process shape.geometry.verts / shape.geometry.faces
        if not iterator.next():
            break
```

---

## 5. Schema Version Detection Methods

```python
# IfcOpenShell: detect schema
model = ifcopenshell.open("file.ifc")
schema = model.schema  # "IFC2X3", "IFC4", "IFC4X3"

# Check entity existence in schema
import ifcopenshell.ifcopenshell_wrapper
schema_obj = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema)
try:
    schema_obj.declaration_by_name("IfcRoad")
    # Entity exists in this schema
except RuntimeError:
    # Entity does NOT exist
    pass

# Blender: detect version
import bpy
major, minor, patch = bpy.app.version

# Bonsai: detect version and schema
from bonsai.bim.ifc import IfcStore
bonsai_schema = IfcStore.get_schema()  # "IFC2X3", "IFC4", "IFC4X3"
```

---

## 6. Context Detection Method

```python
def detect_aec_context() -> dict:
    """Probe runtime environment to determine available AEC technologies.

    Returns dict with keys: blender, bonsai, ifcopenshell, and version info.
    """
    ctx = {
        "blender": False, "blender_version": None,
        "bonsai": False, "bonsai_has_project": False,
        "ifcopenshell": False, "ifc_schema": None,
    }

    try:
        import ifcopenshell
        ctx["ifcopenshell"] = True
    except ImportError:
        pass

    try:
        import bpy
        ctx["blender"] = True
        ctx["blender_version"] = tuple(bpy.app.version)
    except ImportError:
        return ctx  # No Blender = standalone Python

    try:
        from bonsai.bim.ifc import IfcStore
        ctx["bonsai"] = True
        model = IfcStore.get_file()
        if model is not None:
            ctx["bonsai_has_project"] = True
            ctx["ifc_schema"] = model.schema
    except ImportError:
        pass

    return ctx
```
