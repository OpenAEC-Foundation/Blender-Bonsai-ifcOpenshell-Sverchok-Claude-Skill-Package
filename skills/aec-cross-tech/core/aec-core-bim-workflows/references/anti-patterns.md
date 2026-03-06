# Cross-Technology Integration Anti-Patterns

> Common mistakes when integrating IfcOpenShell, Bonsai, and Blender.
> Each anti-pattern includes: what goes wrong, why it fails, and the correct approach.

---

## Anti-Pattern 1: Opening IFC Files Separately While Bonsai Is Active

### What Goes Wrong

```python
# WRONG: Opening a second handle to the same IFC file
from bonsai.bim.ifc import IfcStore
import ifcopenshell

# Bonsai already has the file loaded in memory
bonsai_model = IfcStore.get_file()

# Developer opens a separate handle — creates DIVERGENT STATE
standalone_model = ifcopenshell.open(IfcStore.path)

# Changes to standalone_model are invisible to Bonsai
wall = ifcopenshell.api.run("root.create_entity", standalone_model,
    ifc_class="IfcWall", name="Ghost Wall")

# This wall exists in standalone_model but NOT in Bonsai's IfcStore
# Bonsai UI shows nothing. Saving from Bonsai overwrites the standalone changes.
```

### Why It Fails

Bonsai maintains a single in-memory `ifcopenshell.file` instance via `IfcStore`. Opening the same `.ifc` file with `ifcopenshell.open()` creates an independent copy. Changes to one copy are invisible to the other. When Bonsai saves via `model.write()`, it writes its own copy — overwriting any changes made to the standalone copy.

### Correct Approach

```python
# CORRECT: Always use IfcStore's file when Bonsai is active
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded")

# All mutations go through the same in-memory model
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Visible Wall")
# Bonsai sees the change immediately
```

---

## Anti-Pattern 2: Direct IFC Entity Attribute Modification

### What Goes Wrong

```python
# WRONG: Modifying IFC entity attributes directly
model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Direct attribute modification bypasses relationship management
wall.Name = "Renamed Wall"                    # Seems to work but...
wall.ObjectPlacement = None                   # Breaks spatial references
wall.Representation = None                    # Orphans geometry entities

# Even "simple" property changes can break the graph:
for rel in wall.IsDefinedBy:
    pset = rel.RelatingPropertyDefinition
    for prop in pset.HasProperties:
        prop.NominalValue.wrappedValue = "new_value"  # May corrupt type info
```

### Why It Fails

The IFC data model is a directed graph with complex relationships. Direct attribute modification:
- Bypasses `OwnerHistory` updates (REQUIRED in IFC2X3)
- Does not clean up orphaned inverse relationships
- Does not trigger validation of attribute types
- Can corrupt the relationship graph (e.g., removing ObjectPlacement without updating IfcRelContainedInSpatialStructure)
- Skips Bonsai's undo/redo tracking

### Correct Approach

```python
# CORRECT: Use ifcopenshell.api.run() for ALL mutations
import ifcopenshell.api

# Rename entity
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={"Name": "Renamed Wall"})

# Edit placement
import numpy as np
matrix = np.eye(4)
matrix[0][3] = 5.0  # Move to X=5
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=wall, matrix=matrix)

# Edit properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset,
    properties={"IsExternal": True, "FireRating": "REI90"})

# Remove entity (properly cleans up all relationships)
ifcopenshell.api.run("root.remove_product", model, product=wall)
```

---

## Anti-Pattern 3: Moving Blender Objects Without Syncing to IFC (Bonsai Context)

### What Goes Wrong

```python
# WRONG: Moving object in Blender without IFC sync
import bpy

obj = bpy.context.active_object
obj.location = (5.0, 3.0, 0.0)  # Blender object moves visually

# But the IFC entity's ObjectPlacement is UNCHANGED
# Saving the project writes the OLD position to disk
# Next time you open the file, the object snaps back to its IFC position
```

### Why It Fails

Bonsai uses a "native IFC" workflow where the IFC file is the authoritative document. Blender objects are visualizations of IFC entities. When you modify a Blender object's location/rotation/scale directly, the IFC entity's `IfcLocalPlacement` is NOT updated. On next file load, Bonsai regenerates the Blender mesh from IFC data — using the unchanged placement.

### Correct Approach

```python
# CORRECT: Sync placement after moving
import bpy

obj = bpy.context.active_object
obj.location = (5.0, 3.0, 0.0)

# CRITICAL: Sync Blender transform → IFC placement
bpy.ops.bim.edit_object_placement()

# OR: Set placement via IFC API directly (automatically syncs both ways)
import ifcopenshell.api
import numpy as np
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
entity = tool.Ifc.get_entity(obj)

matrix = np.eye(4)
matrix[0][3] = 5.0
matrix[1][3] = 3.0
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=entity, matrix=matrix)
```

---

## Anti-Pattern 4: Using `blenderbim.*` Imports (Renamed to `bonsai.*`)

### What Goes Wrong

```python
# WRONG: Legacy imports (pre-2024)
from blenderbim.bim.ifc import IfcStore  # ModuleNotFoundError in v0.8.0+
import blenderbim.tool as tool            # ModuleNotFoundError in v0.8.0+
```

### Why It Fails

BlenderBIM was renamed to Bonsai in 2024 (v0.8.0). All Python module paths changed from `blenderbim.*` to `bonsai.*`. Code using old imports fails with `ModuleNotFoundError`.

### Correct Approach

```python
# CORRECT: Current imports (v0.8.0+)
from bonsai.bim.ifc import IfcStore
import bonsai.tool as tool
```

---

## Anti-Pattern 5: Missing Spatial Containment for Physical Elements

### What Goes Wrong

```python
# WRONG: Creating elements without assigning to spatial container
model = ifcopenshell.api.run("project.create_file", version="IFC4")
# ... project, units, context, spatial hierarchy created ...

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Floating Wall")
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)

# Missing: spatial.assign_container
# Wall exists in the file but has no spatial context
model.write("model.ifc")
```

### Why It Fails

Most BIM viewers filter elements by spatial hierarchy. An element without `IfcRelContainedInSpatialStructure` (spatial containment):
- Is invisible in Bonsai's spatial browser
- Does not appear in storey-based views in BIM viewers (Solibri, BIMcollab, Navisworks)
- Fails spatial coordination checks
- Creates orphaned data that is difficult to discover and manage

### Correct Approach

```python
# CORRECT: Always assign spatial containment
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Properly Contained Wall")
# ... geometry assignment ...

# ALWAYS assign to a spatial container
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

---

## Anti-Pattern 6: Skipping Units and Contexts Before Creating Geometry

### What Goes Wrong

```python
# WRONG: Creating geometry before setting up units and contexts
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")

# Jump straight to element creation — no units, no context
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall")

# This may fail or produce ambiguous geometry:
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None,  # No context available!
    length=5.0, height=3.0, thickness=0.2)
# TypeError or invalid IFC: geometry has no representation context
```

### Why It Fails

IFC geometry requires two prerequisites:
1. **Units**: Without `unit.assign_unit`, numeric dimensions (5.0, 3.0, 0.2) have no meaning — meters? millimeters? inches?
2. **Geometric context**: Every `IfcShapeRepresentation` must belong to an `IfcGeometricRepresentationContext`. Without it, viewers cannot interpret or display the geometry.

### Correct Approach

```python
# CORRECT: Follow the mandatory order
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")

# 1. Units FIRST
ifcopenshell.api.run("unit.assign_unit", model)

# 2. Context SECOND
model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# 3. NOW create geometry
rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

---

## Anti-Pattern 7: Copying Entity IDs Between IFC Files

### What Goes Wrong

```python
# WRONG: Using entity IDs from one file in another
source = ifcopenshell.open("source.ifc")
target = ifcopenshell.open("target.ifc")

source_wall = source.by_type("IfcWall")[0]
source_id = source_wall.id()  # e.g., #142

# This does NOT get the same wall — IDs are file-scoped
target_entity = target.by_id(source_id)  # Gets a completely different entity!

# Also WRONG: Copying GlobalIds without checking uniqueness
wall = ifcopenshell.api.run("root.create_entity", target,
    ifc_class="IfcWall", name=source_wall.Name)
# The API generates a new GlobalId — but if you force-set the source's GlobalId:
wall.GlobalId = source_wall.GlobalId  # May conflict if both files are merged
```

### Why It Fails

- **Entity IDs** (`#142`) are file-local step numbers. Entity `#142` in file A is unrelated to `#142` in file B.
- **GlobalIds** must be unique within a file. Copying a GlobalId from one file to another creates a duplicate if the files are ever merged or linked.

### Correct Approach

```python
# CORRECT: Match entities by GlobalId for cross-file lookups
source = ifcopenshell.open("source.ifc")
target = ifcopenshell.open("target.ifc")

source_wall = source.by_type("IfcWall")[0]
source_guid = source_wall.GlobalId

# Look up by GlobalId in target
target_wall = target.by_guid(source_guid)  # Returns entity or None

# For data transfer between files: copy properties, not entities
if target_wall:
    import ifcopenshell.util.element
    source_psets = ifcopenshell.util.element.get_psets(source_wall)
    for pset_name, properties in source_psets.items():
        if pset_name.startswith("id"):  # Skip internal "id" key
            continue
        target_pset = ifcopenshell.api.run("pset.add_pset", target,
            product=target_wall, name=pset_name)
        # Remove the "id" key from properties dict
        props = {k: v for k, v in properties.items() if k != "id"}
        ifcopenshell.api.run("pset.edit_pset", target,
            pset=target_pset, properties=props)
```

---

## Anti-Pattern 8: Using `void.add_opening()` in Bonsai v0.8.0+

### What Goes Wrong

```python
# WRONG: Old API for openings in Bonsai context
import ifcopenshell.api
from bonsai.bim.ifc import IfcStore

model = IfcStore.get_file()
wall = model.by_type("IfcWall")[0]

opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# This API call may fail or produce incorrect results in Bonsai v0.8.0+
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)
```

### Why It Fails

Bonsai v0.8.0+ changed the opening/void workflow from `void.add_opening()` to `feature.add_feature()`. The old API may still work in standalone IfcOpenShell, but in a Bonsai context it can fail to properly update the Blender mesh representation and IfcStore state.

### Correct Approach

```python
# CORRECT: Use the current API for Bonsai v0.8.0+

# In Bonsai context: use Bonsai operators
bpy.ops.bim.add_opening()  # Interactive, handles mesh + IFC sync

# In standalone IfcOpenShell: void.add_opening() still works
ifcopenshell.api.run("void.add_opening", model,
    opening=opening, element=wall)
# Note: This is fine for standalone scripts, just not in Bonsai context
```

---

## Anti-Pattern 9: Mixed API Styles (run() vs create_entity())

### What Goes Wrong

```python
# WRONG: Mixing ifcopenshell.api.run() with model.create_entity()
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")

# API-style: proper GlobalId, validation
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")

# Direct-style: manual GlobalId, no validation, no OwnerHistory
site = model.create_entity("IfcSite",
    GlobalId=ifcopenshell.guid.new(),
    Name="Site",
    OwnerHistory=None)  # WRONG for IFC2X3

# Now manually create the aggregation relationship
model.create_entity("IfcRelAggregates",
    GlobalId=ifcopenshell.guid.new(),
    RelatingObject=project,
    RelatedObjects=[site])
# Missing: OwnerHistory on IfcRelAggregates (REQUIRED for IFC2X3)
```

### Why It Fails

Mixing `ifcopenshell.api.run()` with `model.create_entity()`:
- Creates inconsistent entity handling (some with OwnerHistory, some without)
- Bypasses API-level validation and relationship management
- Doubles the code complexity (API calls handle boilerplate automatically)
- Creates maintenance headaches when switching IFC schema versions

This anti-pattern was observed in the building.py project (see vooronderzoek-ecosystem-sources.md §3.2).

### Correct Approach

```python
# CORRECT: Use ifcopenshell.api.run() consistently for ALL entity creation
model = ifcopenshell.api.run("project.create_file", version="IFC4")

project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Project")
ifcopenshell.api.run("unit.assign_unit", model)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
# API handles GlobalId, OwnerHistory, and relationship creation automatically
```

---

## Summary Table

| # | Anti-Pattern | Severity | Context |
|---|-------------|----------|---------|
| 1 | Opening IFC separately while Bonsai active | CRITICAL | Bonsai |
| 2 | Direct IFC entity attribute modification | CRITICAL | All |
| 3 | Moving Blender objects without IFC sync | HIGH | Bonsai |
| 4 | Using `blenderbim.*` imports | HIGH | Bonsai |
| 5 | Missing spatial containment | HIGH | All |
| 6 | Skipping units and contexts | HIGH | All |
| 7 | Copying entity IDs between files | MEDIUM | Standalone |
| 8 | Using `void.add_opening()` in Bonsai v0.8.0+ | MEDIUM | Bonsai |
| 9 | Mixed API styles (run() vs create_entity()) | MEDIUM | All |

---

## Sources

- IfcOpenShell Documentation: https://docs.ifcopenshell.org
- Bonsai Documentation: https://docs.bonsaibim.org
- OSArch Community: https://community.osarch.org
- Vooronderzoek Ecosystem Sources §3: Cross-Technology Patterns
- building.py IFC exchange: https://github.com/OpenAEC-Foundation/building-py
