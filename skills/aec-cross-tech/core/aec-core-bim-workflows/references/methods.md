# Cross-Technology API Integration Points

> Complete method signatures for cross-technology BIM workflow integration.
> Covers: IfcOpenShell, Bonsai (IfcStore + tool.*), and Blender (bpy) integration surfaces.

---

## 1. IfcOpenShell Core API (Standalone Context)

### File Operations

```python
# Open existing IFC file
# Returns: ifcopenshell.file
model = ifcopenshell.open(filepath: str) -> ifcopenshell.file

# Create new IFC file with proper header
# version: "IFC2X3" | "IFC4" | "IFC4X3"
# Returns: ifcopenshell.file
model = ifcopenshell.api.run("project.create_file", version: str = "IFC4") -> ifcopenshell.file

# Write IFC file to disk
model.write(filepath: str) -> None
```

### Entity Creation (ifcopenshell.api.run)

```python
# Create a rooted IFC entity (generates GlobalId automatically)
# ifc_class: e.g., "IfcProject", "IfcSite", "IfcWall", "IfcColumn"
# name: optional display name
# predefined_type: optional, e.g., "FLOOR" for IfcSlab
# Returns: entity_instance
entity = ifcopenshell.api.run("root.create_entity", model,
    ifc_class: str,
    name: str = None,
    predefined_type: str = None
) -> entity_instance

# Remove an entity and clean up relationships
ifcopenshell.api.run("root.remove_product", model,
    product: entity_instance
) -> None
```

### Spatial Structure

```python
# Assign objects to a parent container (aggregation)
# products: list of child entities
# relating_object: parent entity (e.g., IfcProject, IfcSite, IfcBuilding)
ifcopenshell.api.run("aggregate.assign_object", model,
    products: list[entity_instance],
    relating_object: entity_instance
) -> entity_instance  # Returns IfcRelAggregates

# Assign elements to a spatial container
# relating_structure: e.g., IfcBuildingStorey
# products: list of physical elements
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure: entity_instance,
    products: list[entity_instance]
) -> entity_instance  # Returns IfcRelContainedInSpatialStructure
```

### Units and Contexts

```python
# Assign default SI units (meters, radians, seconds)
# length: dict with "is_metric": True, "raw": "METRE" (optional overrides)
ifcopenshell.api.run("unit.assign_unit", model,
    length: dict = None,
    area: dict = None,
    volume: dict = None
) -> None

# Add geometric representation context
# context_type: "Model" or "Plan"
# context_identifier: "Body", "Axis", "Box", "Annotation" (for subcontexts)
# target_view: "MODEL_VIEW", "PLAN_VIEW", "GRAPH_VIEW"
# parent: parent context (required for subcontexts)
context = ifcopenshell.api.run("context.add_context", model,
    context_type: str,
    context_identifier: str = None,
    target_view: str = None,
    parent: entity_instance = None
) -> entity_instance
```

### Geometry

```python
# Add wall representation (extruded rectangle)
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context: entity_instance,    # Body subcontext
    length: float,
    height: float,
    thickness: float
) -> entity_instance  # Returns IfcShapeRepresentation

# Add slab representation (extruded polyline footprint)
rep = ifcopenshell.api.run("geometry.add_slab_representation", model,
    context: entity_instance,
    depth: float,                # slab thickness
    polyline: list[tuple[float, float]]  # 2D footprint points
) -> entity_instance

# Add profile-based representation (columns, beams)
rep = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context: entity_instance,
    profile: entity_instance,    # IfcProfileDef
    depth: float                 # extrusion length
) -> entity_instance

# Assign representation to product
ifcopenshell.api.run("geometry.assign_representation", model,
    product: entity_instance,
    representation: entity_instance
) -> None

# Set object placement (position/rotation via 4x4 matrix)
# matrix: numpy 4x4 array (identity = origin)
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product: entity_instance,
    matrix: numpy.ndarray = numpy.eye(4)
) -> None
```

### Property Sets

```python
# Add a new property set to a product
# name: "Pset_WallCommon", "Qto_WallBaseQuantities", or custom "MyOrg_WallData"
pset = ifcopenshell.api.run("pset.add_pset", model,
    product: entity_instance,
    name: str
) -> entity_instance  # Returns IfcPropertySet

# Edit property values in a pset
ifcopenshell.api.run("pset.edit_pset", model,
    pset: entity_instance,
    properties: dict[str, Any]  # {"IsExternal": True, "FireRating": "REI60"}
) -> None

# Add quantity set
qto = ifcopenshell.api.run("pset.add_qto", model,
    product: entity_instance,
    name: str
) -> entity_instance  # Returns IfcElementQuantity

# Edit quantities
ifcopenshell.api.run("pset.edit_qto", model,
    qto: entity_instance,
    properties: dict[str, float]
) -> None
```

### Type Assignment

```python
# Assign type to occurrences
# related_objects: list of occurrence entities (IfcWall, IfcColumn, etc.)
# relating_type: type entity (IfcWallType, IfcColumnType, etc.)
ifcopenshell.api.run("type.assign_type", model,
    related_objects: list[entity_instance],
    relating_type: entity_instance
) -> entity_instance  # Returns IfcRelDefinesByType
```

### Materials

```python
# Add a material
material = ifcopenshell.api.run("material.add_material", model,
    name: str,
    category: str = None  # "concrete", "steel", "brick", etc.
) -> entity_instance

# Add material set (layers, profiles, or constituents)
material_set = ifcopenshell.api.run("material.add_material_set", model,
    name: str,
    set_type: str  # "IfcMaterialLayerSet" | "IfcMaterialProfileSet" | "IfcMaterialConstituentSet"
) -> entity_instance

# Add layer to layer set
layer = ifcopenshell.api.run("material.add_layer", model,
    layer_set: entity_instance,
    material: entity_instance
) -> entity_instance

# Edit layer attributes
ifcopenshell.api.run("material.edit_layer", model,
    layer: entity_instance,
    attributes: dict  # {"LayerThickness": 0.2}
) -> None

# Assign material to products
ifcopenshell.api.run("material.assign_material", model,
    products: list[entity_instance],
    type: str = "IfcMaterial",  # or "IfcMaterialLayerSet", etc.
    material: entity_instance = None
) -> entity_instance
```

### Classification

```python
# Add classification system
classification = ifcopenshell.api.run("classification.add_classification", model,
    classification: str  # e.g., "Uniclass", "NL-SfB", "OmniClass"
) -> entity_instance

# Add classification reference to elements
ifcopenshell.api.run("classification.add_reference", model,
    products: list[entity_instance],
    classification: entity_instance,
    identification: str,  # e.g., "21.22"
    name: str = None      # e.g., "Exterior Walls"
) -> entity_instance
```

### Openings (Voids)

```python
# Add opening to wall (IfcOpenShell standalone)
ifcopenshell.api.run("void.add_opening", model,
    opening: entity_instance,    # IfcOpeningElement
    element: entity_instance     # IfcWall
) -> entity_instance

# Fill opening with door/window
ifcopenshell.api.run("void.add_filling", model,
    opening: entity_instance,
    element: entity_instance     # IfcDoor or IfcWindow
) -> entity_instance
```

### IFC2X3 Owner History (REQUIRED for IFC2X3 only)

```python
# Set up owner (MUST call before any root.create_entity in IFC2X3)
person = ifcopenshell.api.run("owner.add_person", model,
    identification: str, family_name: str, given_name: str
) -> entity_instance

org = ifcopenshell.api.run("owner.add_organisation", model,
    identification: str, name: str
) -> entity_instance

user = ifcopenshell.api.run("owner.add_person_and_organisation", model,
    person: entity_instance, organisation: entity_instance
) -> entity_instance

application = ifcopenshell.api.run("owner.add_application", model
) -> entity_instance

ifcopenshell.api.run("owner.set_user", model,
    user: entity_instance
) -> None
```

---

## 2. IfcOpenShell Utility Functions

```python
import ifcopenshell.util.element

# Get all property sets for an element (returns dict of dicts)
psets = ifcopenshell.util.element.get_psets(
    element: entity_instance,
    psets_only: bool = False,
    qtos_only: bool = False
) -> dict[str, dict[str, Any]]
# Example return: {"Pset_WallCommon": {"IsExternal": True, "FireRating": "REI60"}}

# Get spatial container of an element
container = ifcopenshell.util.element.get_container(
    element: entity_instance
) -> entity_instance | None
# Returns: IfcBuildingStorey, IfcSpace, IfcFacilityPart, or None

# Get decomposition children
children = ifcopenshell.util.element.get_decomposition(
    element: entity_instance
) -> list[entity_instance]

# Get type of an element
element_type = ifcopenshell.util.element.get_type(
    element: entity_instance
) -> entity_instance | None
```

### Geometry Processing

```python
import ifcopenshell.geom

# Configure geometry settings
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)   # Apply transformations
settings.set(settings.SEW_SHELLS, True)          # Fix open shells
settings.set(settings.WELD_VERTICES, True)       # Merge duplicate vertices

# Create shape from IFC element
shape = ifcopenshell.geom.create_shape(
    settings: ifcopenshell.geom.settings,
    element: entity_instance
) -> ShapeType
# shape.geometry.verts: flat float list [x0,y0,z0, x1,y1,z1, ...]
# shape.geometry.faces: flat int list [i0,i1,i2, i3,i4,i5, ...]
# shape.geometry.normals: flat float list (per-vertex normals)
# shape.transformation.matrix.data: flat 4x4 transformation matrix
```

### Element Selector (CSS-like queries)

```python
import ifcopenshell.util.selector

# Select elements using CSS-like syntax
elements = ifcopenshell.util.selector.filter_elements(
    model: ifcopenshell.file,
    query: str
) -> list[entity_instance]

# Query examples:
# "IfcWall"                              → All walls
# "IfcWall, IfcSlab"                     → All walls and slabs
# ".IfcWall[Name='Wall 001']"            → Wall with specific name
# ".IfcWall[PredefinedType='SOLIDWALL']" → Walls of specific type
```

---

## 3. Bonsai Integration API

### IfcStore (Central State Manager)

```python
from bonsai.bim.ifc import IfcStore

# Static attributes
IfcStore.path: str                    # Absolute path to .ifc file
IfcStore.file: ifcopenshell.file      # Live in-memory IFC (or None)
IfcStore.schema: str                  # "IFC2X3" | "IFC4" | "IFC4X3"
IfcStore.id_map: dict[int, bpy.types.Object]  # IFC id → Blender object
IfcStore.guid_map: dict[str, bpy.types.Object] # GlobalId → Blender object
IfcStore.edited_objs: set             # Objects with pending IFC changes

# Static methods
model = IfcStore.get_file() -> ifcopenshell.file | None
schema = IfcStore.get_schema() -> str | None
obj = IfcStore.get_element(ifc_id: int) -> bpy.types.Object | None
```

### tool.Ifc (High-Level IFC Access)

```python
# These are called as static methods on the tool.Ifc class

# Get live IFC file
model = tool.Ifc.get() -> ifcopenshell.file | None

# Set active IFC file (triggers post-load hooks)
tool.Ifc.set(ifc: ifcopenshell.file) -> None

# Execute ifcopenshell.api command (model is implicit)
result = tool.Ifc.run(command: str, **kwargs) -> Any

# Blender object → IFC entity
entity = tool.Ifc.get_entity(obj: bpy.types.Object) -> entity_instance | None

# IFC entity → Blender object
obj = tool.Ifc.get_object(element: entity_instance) -> bpy.types.Object | None

# Create bidirectional link between IFC entity and Blender object
tool.Ifc.link(element: entity_instance, obj: bpy.types.Object) -> None
```

### Key Bonsai Operators (bpy.ops.bim.*)

```python
# Project management
bpy.ops.bim.create_project()           # Create new IFC project
bpy.ops.bim.load_project(filepath=str) # Load existing IFC file

# Element creation
bpy.ops.bim.add_wall()                 # Add wall at cursor
bpy.ops.bim.add_slab()                 # Add slab
bpy.ops.bim.add_column()               # Add column
bpy.ops.bim.add_beam()                 # Add beam
bpy.ops.bim.add_opening()              # Add opening to selected wall

# Placement sync (CRITICAL after moving objects)
bpy.ops.bim.edit_object_placement()    # Sync Blender transform → IFC placement

# Spatial operations
bpy.ops.bim.assign_container()         # Assign selected to spatial container
bpy.ops.bim.enable_editing_container() # Enter container editing mode

# Property operations
bpy.ops.bim.add_pset(obj=str, obj_type=str, pset_name=str)
bpy.ops.bim.edit_pset()
bpy.ops.bim.remove_pset()

# Type operations
bpy.ops.bim.assign_type()
bpy.ops.bim.unassign_type()

# Save
bpy.ops.bim.save_project()             # Write IFC to disk
bpy.ops.bim.save_project_as(filepath=str)
```

### Blender Object ↔ IFC Entity Properties

```python
# Check if object is linked to IFC
ifc_id = obj.BIMProperties.ifc_definition_id  # int: 0 = not linked

# Get IFC class of linked entity
entity = tool.Ifc.get_entity(obj)
if entity:
    ifc_class = entity.is_a()      # e.g., "IfcWall"
    global_id = entity.GlobalId    # e.g., "2O2Fr$t4X7Zf8NOew3FLOH"
    name = entity.Name             # e.g., "Wall 001"
```

---

## 4. Blender Geometry API (for IFC mesh import)

```python
import bpy

# Create new mesh data block
mesh = bpy.data.meshes.new(name: str) -> bpy.types.Mesh

# Populate mesh from vertex/face data
# vertices: list of (x, y, z) tuples
# edges: list of (v1, v2) tuples (usually empty for IFC)
# faces: list of (v1, v2, v3) or (v1, v2, v3, v4) tuples
mesh.from_pydata(
    vertices: list[tuple[float, float, float]],
    edges: list[tuple[int, int]],
    faces: list[tuple[int, ...]]
) -> None
mesh.update() -> None  # ALWAYS call after from_pydata

# Create object and link to scene
obj = bpy.data.objects.new(name: str, object_data: bpy.types.Mesh) -> bpy.types.Object
bpy.context.collection.objects.link(obj) -> None

# Set object transform
import mathutils
obj.matrix_world = mathutils.Matrix(matrix_4x4)

# Create material and assign
mat = bpy.data.materials.new(name: str) -> bpy.types.Material
obj.data.materials.append(mat)
```

---

## Sources

- IfcOpenShell API: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html
- IfcOpenShell Geometry: https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html
- Bonsai Source: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
- Blender Python API: https://docs.blender.org/api/current/bpy.types.Mesh.html
