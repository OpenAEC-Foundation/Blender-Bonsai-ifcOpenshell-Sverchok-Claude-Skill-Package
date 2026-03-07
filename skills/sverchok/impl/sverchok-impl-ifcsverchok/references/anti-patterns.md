# IfcSverchok — Anti-Patterns

> IfcOpenShell 0.8.x / Sverchok v1.4.0+ / Blender 4.0+/5.x

## AP-001: Using Undo in IfcSverchok Node Trees

### Bad

```python
# User presses Ctrl+Z after running IfcSverchok nodes
# Blender reverts node tree state, but SvIfcStore.file is NOT reverted
# Result: SvIfcStore.file references entities that no longer match node state
# Consequence: Crash or corrupted IFC output
```

### Good

```python
# Save the .blend file before running IfcSverchok nodes.
# If something goes wrong, revert to saved file.
# To re-run: click "Re-run all nodes" which calls SvIfcStore.purge()
# and rebuilds the IFC file from scratch.
```

### Why

`SvIfcStore.file` is a Python-side object not tracked by Blender's undo system. Undo restores the node tree but leaves the transient IFC file in its previous state, causing entity reference mismatches and crashes.

---

## AP-002: Expecting Multiple IFC Files

### Bad

```python
# Attempting to work with two IFC files simultaneously
tree1 = bpy.data.node_groups["BuildingA"]  # Has SvIfcCreateFile → IFC4
tree2 = bpy.data.node_groups["BuildingB"]  # Has SvIfcCreateFile → IFC4
# Running both trees overwrites SvIfcStore.file — only the last one survives
```

### Good

```python
# Process one IFC file at a time.
# Run tree1, export with SvIfcWriteFile.
# Then run tree2, export separately.
# Or combine everything into a single node tree.
```

### Why

`SvIfcStore` is a singleton with exactly one `file` attribute. All nodes across all trees share this single file. Creating a new file in any tree replaces the file for all trees.

---

## AP-003: Skipping Representation Conversion

### Bad

```python
# Connecting Sverchok geometry directly to SvIfcCreateEntity
# [Box Generator] → Vertices → [SvIfcCreateEntity]
# Result: Entity is created but has NO geometry in the IFC file
# The exported IFC file contains an IfcWall with no shape representation
```

### Good

```python
# ALWAYS convert geometry to IFC representation first
# [Box Generator] → Vertices → [SvIfcSverchokToIfcRepr] → Repr → [SvIfcCreateEntity]
#                 → Faces    → /
```

### Why

IFC entities require `IfcShapeRepresentation` objects, not raw vertex data. The representation converter nodes (`SvIfcBMeshToIfcRepr`, `SvIfcSverchokToIfcRepr`, `SvIfcSbRepresentation`) create the required IFC geometric representation entities.

---

## AP-004: Manually Building Full Hierarchy When Not Needed

### Bad

```python
# Creating every spatial element manually for a simple model
project = tree.nodes.new('SvIfcCreateProject')
site = tree.nodes.new('SvIfcAddSpatialElement')  # IfcSite
building = tree.nodes.new('SvIfcAddSpatialElement')  # IfcBuilding
storey = tree.nodes.new('SvIfcAddSpatialElement')  # IfcBuildingStorey
# ... connecting all of them in the correct hierarchy
# This is verbose and error-prone for simple models
```

### Good

```python
# For simple models, let ensure_hirarchy() handle it automatically.
# Just create the entity and write the file:
# [Geometry] → [Repr Converter] → [SvIfcCreateEntity] → [SvIfcWriteFile]
# ensure_hirarchy() auto-creates Project, Site, Building and links them.

# Only build manual hierarchy when you need:
# - Multiple buildings or storeys
# - Custom naming for spatial elements
# - Specific spatial assignments
```

### Why

`SvIfcWriteFile` calls `ensure_hirarchy()` before export, which creates default `IfcProject`, `IfcSite`, and `IfcBuilding` if they do not exist, and assigns orphaned elements. Manual hierarchy is only needed for multi-storey or multi-building models.

---

## AP-005: Mixing Bonsai File Mode with Export

### Bad

```python
# Toggle "Use Bonsai File" then try to export
SvIfcStore.use_bonsai_file = True
# ... run nodes that modify Bonsai's active file ...
# Then SvIfcWriteFile exports — this writes Bonsai's file to a new path
# but ensure_hirarchy() may modify Bonsai's spatial structure unexpectedly
```

### Good

```python
# Choose ONE mode:

# Mode A: Standalone (default)
SvIfcStore.use_bonsai_file = False
# Create geometry → representation → entities → SvIfcWriteFile
# Clean, predictable, no side effects on Bonsai

# Mode B: Bonsai integration
SvIfcStore.use_bonsai_file = True
# Create entities that modify Bonsai's file in-place
# Do NOT use SvIfcWriteFile — let Bonsai handle file management
# Use Bonsai's own export functionality instead
```

### Why

When `use_bonsai_file=True`, `SvIfcStore.get_file()` returns Bonsai's live file. Running `ensure_hirarchy()` or `SvIfcWriteFile` on this file modifies Bonsai's internal state, potentially creating duplicate spatial elements or corrupting Bonsai's hierarchy tracking.

---

## AP-006: Assuming Standard Sverchok Input Processing

### Bad

```python
# Writing a custom IfcSverchok node using standard Sverchok process() pattern
class SvMyIfcNode(SverchCustomTreeNode, bpy.types.Node):
    def process(self):
        data = self.inputs['Data'].sv_get()
        for item in zip_long_repeat(*data):  # Only ONE level of zip
            self.do_ifc_work(*item)
# This processes at the wrong nesting level — misses the double-nested structure
```

### Good

```python
# Inherit from SvIfcCore and implement process_ifc() instead
class SvMyIfcNode(SvIfcCore, SverchCustomTreeNode, bpy.types.Node):
    sv_input_names = ['Data', 'Name']

    def process_ifc(self, data, name):
        # SvIfcCore.process() handles the double-nested zip_long_repeat
        # Each call receives individual scalar values, not lists
        file = SvIfcStore.get_file()
        # ... create IFC entity with data and name ...
```

### Why

`SvIfcCore.process()` applies `zip_long_repeat` twice — once at the object level (outer list) and once at the item level (inner list). Standard Sverchok nodes apply it once. Using the wrong pattern causes either processing at the wrong granularity or index errors.

---

## AP-007: Forgetting to Purge After Partial Runs

### Bad

```python
# Running individual nodes or partial tree sections
# without purging SvIfcStore first
# Result: id_map contains stale entity references from previous runs
# New entities may conflict with old references
```

### Good

```python
# Always use "Re-run all nodes" to trigger SvIfcStore.purge()
# before a clean generation pass.
# Or purge manually before scripted runs:
from ifcsverchok.helper import SvIfcStore
SvIfcStore.purge()
tree.force_update()
```

### Why

`SvIfcStore.id_map` accumulates entity references across partial runs. Without purging, nodes may find stale entries in `id_map` and skip entity creation or reference deleted entities. `purge()` ensures a clean slate.

---

## AP-008: Using SvIfcCreateShape for Round-Trip Editing

### Bad

```python
# Creating Blender shapes from IFC, modifying them, then expecting IFC to update
# [SvIfcReadFile] → [SvIfcByType] → [SvIfcCreateShape] → Blender objects
# User edits Blender objects manually
# Expectation: IFC file is updated with the edits
# Reality: SvIfcCreateShape is ONE-WAY — Blender objects are disconnected copies
```

### Good

```python
# For round-trip IFC editing, use Bonsai directly.
# IfcSverchok is designed for IFC GENERATION, not round-trip editing.
# To modify existing IFC: read → query → use SvIfcApi to call ifcopenshell.api
```

### Why

`SvIfcCreateShape` creates Blender mesh objects from IFC entities for visualization purposes only. These objects have no live connection back to the IFC file. Edits to these Blender objects are not reflected in the IFC data.

---

## AP-009: Connecting Incompatible Geometry Modes

### Bad

```python
# Mixing ShapeBuilder output with SvIfcSverchokToIfcRepr
# [SvIfcSbRectangle] → Profile → [SvIfcSverchokToIfcRepr]
# ShapeBuilder outputs IFC profile entities, not Sverchok vertex/face data
# Result: Type error or malformed representation
```

### Good

```python
# ShapeBuilder nodes have their own pipeline:
# [SvIfcSbRectangle] → [SvIfcSbExtrude] → [SvIfcSbRepresentation] → Repr

# Sverchok geometry has its own pipeline:
# [Box Generator] → Vertices/Faces → [SvIfcSverchokToIfcRepr] → Repr

# Blender objects have their own pipeline:
# [Objects In] → Objects → [SvIfcBMeshToIfcRepr] → Repr

# All three output compatible Representation that feeds into SvIfcCreateEntity
```

### Why

The three geometry modes (ShapeBuilder, Sverchok geometry, Blender mesh) operate on different data types. ShapeBuilder works with IFC profile/solid entities. `SvIfcSverchokToIfcRepr` expects Sverchok nested vertex/face lists. `SvIfcBMeshToIfcRepr` expects Blender objects. Mixing inputs between modes causes type mismatches.

---

## AP-010: Not Checking Bonsai Active File Existence

### Bad

```python
# Enabling use_bonsai_file when no Bonsai project is loaded
SvIfcStore.use_bonsai_file = True
file = SvIfcStore.get_file()  # Returns None if Bonsai has no active file
# Subsequent node operations fail with AttributeError on None
```

### Good

```python
# Verify Bonsai has an active file before toggling
from bonsai.bim.ifc import IfcStore as BonsaiIfcStore

if BonsaiIfcStore.get_file() is not None:
    SvIfcStore.use_bonsai_file = True
else:
    print("No active Bonsai IFC file — using standalone mode")
    SvIfcStore.use_bonsai_file = False
```

### Why

When `use_bonsai_file=True`, `SvIfcStore.get_file()` delegates to `tool.Ifc.get()`. If Bonsai has no loaded IFC project, this returns `None`. All downstream nodes assume a valid `ifcopenshell.file` object and crash on `None`.
