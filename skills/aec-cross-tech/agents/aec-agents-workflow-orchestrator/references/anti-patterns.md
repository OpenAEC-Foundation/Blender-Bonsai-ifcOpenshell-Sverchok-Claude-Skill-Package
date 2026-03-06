# Cross-Technology Workflow Anti-Patterns

> Common mistakes when orchestrating multi-technology AEC workflows.
> Each entry includes the anti-pattern, why it fails, and the correct approach.

---

## AP-1: Dual File Handle — Opening IFC in Both Bonsai and Standalone IfcOpenShell

### The Anti-Pattern

```python
# WRONG — two independent handles to the same IFC data
from bonsai.bim.ifc import IfcStore
import ifcopenshell

# Bonsai already has the file loaded
bonsai_model = IfcStore.get_file()

# Opening again creates a SEPARATE in-memory copy
standalone_model = ifcopenshell.open(IfcStore.path)  # WRONG

# Changes to standalone_model are INVISIBLE to Bonsai
ifcopenshell.api.run("root.create_entity", standalone_model,
    ifc_class="IfcWall", name="Ghost Wall")  # NOT in Bonsai
```

### Why It Fails

`IfcStore.get_file()` returns Bonsai's live in-memory IFC graph. `ifcopenshell.open()` creates a completely independent copy. Changes to one copy are invisible to the other. Saving from either handle overwrites the other's changes. The Blender viewport only reflects Bonsai's in-memory graph, so standalone changes never appear.

### Correct Approach

```python
# In Bonsai context: ALWAYS use IfcStore
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# ALL mutations go through the same model instance
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Visible Wall")  # Bonsai sees this

# For standalone post-processing: save from Bonsai FIRST, THEN open separately
# in a DIFFERENT Python process
```

---

## AP-2: Missing Placement Sync — Moving Blender Objects Without Updating IFC

### The Anti-Pattern

```python
# WRONG — moving Blender object but forgetting IFC sync
import bpy

obj = bpy.context.active_object
obj.location = (5.0, 3.0, 0.0)  # Moves in Blender viewport

# IFC placement is NOT updated!
# The IFC file still records the OLD position.
# Saving the project writes WRONG coordinates to .ifc file.
```

### Why It Fails

Bonsai maintains a bidirectional link between Blender objects and IFC entities, but transform changes are NOT automatically synced from Blender to IFC. The Blender viewport shows the new position, but `IfcLocalPlacement` in the IFC graph still holds the original transform. When saving, the IFC file contains stale placement data. Reopening the file resets the object to its IFC-recorded position.

### Correct Approach

```python
# CORRECT — sync placement after any transform change
import bpy

obj = bpy.context.active_object
obj.location = (5.0, 3.0, 0.0)

# ALWAYS call this after moving/rotating/scaling
bpy.ops.bim.edit_object_placement()
# This updates IfcLocalPlacement to match the Blender transform
```

---

## AP-3: Direct Entity Attribute Modification — Bypassing the API

### The Anti-Pattern

```python
# WRONG — modifying IFC attributes directly
import ifcopenshell

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Direct attribute modification bypasses relationship management
wall.Name = "Renamed Wall"                    # WRONG
wall.Description = "A modified wall"          # WRONG
wall.ObjectPlacement = some_placement         # WRONG — breaks placement tree
wall.Representation = some_representation     # WRONG — orphans old representation
```

### Why It Fails

`ifcopenshell.api.run()` manages the full IFC relationship graph. Direct attribute modification bypasses:
- Relationship cleanup (old relationships become orphaned)
- Inverse relationship updates (bi-directional links break)
- OwnerHistory updates (modification timestamps are not recorded)
- Bonsai sync hooks (IfcStore maps become stale)
- Schema validation (invalid attribute values are not caught)

### Correct Approach

```python
# CORRECT — use ifcopenshell.api for ALL mutations
import ifcopenshell.api

# Rename via attribute API (or just set Name for simple text attributes)
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "Renamed Wall"})

# Placement via geometry API
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=new_matrix)

# Representation via geometry API
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=new_representation)
```

---

## AP-4: Wrong Execution Context — Using Viewport Operators in Headless Mode

### The Anti-Pattern

```python
# WRONG — using viewport-dependent operators in background mode
# Running: blender --background --python script.py

import bpy

# These operators REQUIRE an active 3D viewport
bpy.ops.view3d.snap_cursor_to_center()        # FAILS — no viewport
bpy.ops.view3d.camera_to_view()               # FAILS — no viewport
bpy.ops.screen.screen_full_area()             # FAILS — no screen

# These operators need specific context that background mode lacks
bpy.ops.mesh.select_all(action='SELECT')      # FAILS if not in edit mode
bpy.ops.object.mode_set(mode='EDIT')          # May FAIL without active object
```

### Why It Fails

Background mode (`blender --background`) does not create a window, screen, or 3D viewport. Operators in the `view3d`, `screen`, and other UI-dependent namespaces require these structures. The operator poll function returns False, and the call raises a RuntimeError.

### Correct Approach

```python
# CORRECT — use data-level API in background mode
import bpy

# Direct data access (works in ALL modes)
obj = bpy.data.objects.get("Cube")
if obj:
    obj.location = (0.0, 0.0, 0.0)  # Direct property, no operator needed

# For mesh editing in background: use bmesh
import bmesh
mesh = obj.data
bm = bmesh.new()
bm.from_mesh(mesh)
# ... edit operations ...
bm.to_mesh(mesh)
bm.free()

# For IFC operations in background: use IfcOpenShell directly
import ifcopenshell
model = ifcopenshell.open("model.ifc")
# All ifcopenshell.api.run() calls work without a viewport
```

---

## AP-5: Schema Mismatch — Using IFC4X3 Entities in IFC4 Files

### The Anti-Pattern

```python
# WRONG — using IFC4X3-only entities in an IFC4 file
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")

# IfcRoad does NOT exist in IFC4
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway A1")  # RUNTIME ERROR

# IfcBuiltElement does NOT exist in IFC4 (it's IFC4X3)
elements = model.by_type("IfcBuiltElement")  # Returns empty tuple (no error, but wrong)

# IfcFacility does NOT exist in IFC4
facility = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcFacility", name="Facility")  # RUNTIME ERROR
```

### Why It Fails

IFC schema versions define different entity sets. Infrastructure entities (IfcRoad, IfcBridge, IfcRailway, IfcFacility) exist only in IFC4X3. IfcBuildingElement was renamed to IfcBuiltElement in IFC4X3. Using the wrong entity class for the schema version causes runtime errors or silent incorrect results.

### Correct Approach

```python
# CORRECT — schema-aware entity selection
import ifcopenshell
import ifcopenshell.api

schema = "IFC4X3"  # Choose schema based on project requirements
model = ifcopenshell.api.run("project.create_file", version=schema)

# Schema-aware element class
if model.schema == "IFC4X3":
    elements = model.by_type("IfcBuiltElement")
else:
    elements = model.by_type("IfcBuildingElement")

# Verify entity existence before creation
import ifcopenshell.ifcopenshell_wrapper
schema_obj = ifcopenshell.ifcopenshell_wrapper.schema_by_name(model.schema)
try:
    schema_obj.declaration_by_name("IfcRoad")
    road = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcRoad", name="Highway A1")
except RuntimeError:
    print(f"IfcRoad not available in {model.schema}")
```

---

## AP-6: Missing Geometry Prerequisites — Creating Elements Before Context/Units

### The Anti-Pattern

```python
# WRONG — creating geometry before setting up context and units
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")

# SKIPPING unit.assign_unit — geometry dimensions are ambiguous
# SKIPPING context.add_context — no representation context exists

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")

# This fails or produces invalid geometry because no context exists
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None, length=5.0, height=3.0, thickness=0.2)  # FAILS — context is None
```

### Why It Fails

IFC geometry requires:
1. **Units** — without units, numeric values (5.0 meters vs 5.0 millimeters) are ambiguous
2. **Geometric representation context** — every `IfcShapeRepresentation` references a context that defines the coordinate space and representation type

Viewers and downstream tools cannot interpret geometry without these prerequisites.

### Correct Approach

```python
# CORRECT — follow the initialization sequence
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")

# Step 1: Units FIRST
ifcopenshell.api.run("unit.assign_unit", model)

# Step 2: Contexts SECOND
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Step 3: NOW create geometry
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=rep)
```

---

## AP-7: Blender Version Assumptions — Using Removed APIs

### The Anti-Pattern

```python
# WRONG — using Blender 3.x API in 4.0+
import bpy

# Dict context override — REMOVED in Blender 4.0
override = bpy.context.copy()
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="Subsurf")  # CRASHES in 4.0+

# bgl module — REMOVED in Blender 5.0
import bgl  # ImportError in 5.0+
bgl.glEnable(bgl.GL_BLEND)

# Old node group socket API — REMOVED in Blender 4.0
node_group.inputs.new("NodeSocketFloat", "My Input")  # AttributeError in 4.0+
```

### Why It Fails

Blender introduces breaking changes across major versions:
- **4.0**: Dict context overrides removed (use `context.temp_override()`), node group `inputs/outputs` replaced by `interface`
- **4.3**: Grease Pencil API completely rewritten
- **5.0**: `bgl` module removed (use `gpu` module), compositor access changed

### Correct Approach

```python
# CORRECT — version-aware code
import bpy

# Context override (3.2+, works in 4.x and 5.x)
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Subsurf")

# Drawing (5.0+)
import gpu
from gpu_extras.batch import batch_for_shader

# Node group sockets (4.0+)
node_group.interface.new_socket(
    name="My Input", in_out='INPUT', socket_type='NodeSocketFloat')

# Version branching when backward compatibility is needed
if bpy.app.version >= (4, 0, 0):
    # 4.0+ path
    pass
else:
    # 3.x fallback
    pass
```

---

## AP-8: Orphaned IFC Elements — Creating Without Spatial Containment

### The Anti-Pattern

```python
# WRONG — creating elements without assigning a spatial container
import ifcopenshell.api

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Floating Wall")

# Wall exists in the IFC file but belongs to NO storey/space
# Most IFC viewers will either hide it or show it at the origin
# Validation tools flag it as an error
```

### Why It Fails

IFC elements without spatial containment (`IfcRelContainedInSpatialStructure`) are:
- Invisible in most BIM viewers (they filter by spatial hierarchy)
- Flagged as validation errors by IFC checkers
- Missing from quantity takeoffs and reports that traverse the spatial tree
- Not associated with any building storey elevation

### Correct Approach

```python
# CORRECT — ALWAYS assign spatial container after element creation
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Contained Wall")

# Assign to storey IMMEDIATELY after creation
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# Verify containment
container = ifcopenshell.util.element.get_container(wall)
assert container is not None, f"Wall {wall.Name} has no spatial container"
```

---

## AP-9: IFC2X3 OwnerHistory Omission

### The Anti-Pattern

```python
# WRONG — creating IFC2X3 entities without OwnerHistory
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")

# In IFC2X3, OwnerHistory is MANDATORY on all IfcRoot entities
# If using low-level creation:
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    None,  # WRONG — OwnerHistory cannot be None in IFC2X3
    "Wall"
)
```

### Why It Fails

IFC2X3 requires `OwnerHistory` as a MANDATORY attribute on all entities inheriting from `IfcRoot`. Unlike IFC4/IFC4X3 where it is OPTIONAL, omitting it in IFC2X3 produces a schema-invalid file that fails validation and may crash viewers.

### Correct Approach

```python
# CORRECT — use ifcopenshell.api.run() which handles OwnerHistory automatically
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
# API automatically creates and assigns OwnerHistory for IFC2X3

# If manual creation is needed (rare):
owner_history = ifcopenshell.api.run("owner.create_owner_history", model)
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    owner_history,  # REQUIRED in IFC2X3
    "Wall"
)
```
