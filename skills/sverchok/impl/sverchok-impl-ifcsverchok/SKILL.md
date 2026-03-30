---
name: sverchok-impl-ifcsverchok
description: >
  Use when generating IFC files from Sverchok geometry using IfcSverchok nodes. Prevents
  the critical mistake of not understanding SvIfcStore transient file management (data is
  purged on every full tree update -- use use_bonsai_file to persist). Covers the 31
  specialized nodes, two geometry conversion modes, the 6-step IFC generation workflow,
  and Bonsai integration.
  Keywords: IfcSverchok, SvIfcStore, IFC from Sverchok, geometry to IFC, Bonsai integration, transient file, use_bonsai_file, IFC generation, node-based BIM, parametric IFC, generate IFC from nodes.
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+, Bonsai addon, and IfcOpenShell 0.8.x.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-impl-ifcsverchok

## Quick Reference

### What Is IfcSverchok

IfcSverchok is an extension that bridges Sverchok parametric geometry with the IFC (Industry Foundation Classes) open BIM standard. It enables creation of native IFC models using visual node-based programming in Blender.

- **31 specialized nodes** (24 IFC + 7 Shape Builder) for BIM model creation
- **Two geometry modes**: Blender mesh objects or raw Sverchok geometry to IFC
- **Transient IFC file**: single shared `SvIfcStore.file` managed across all nodes
- **Bonsai integration**: toggle to read/write Bonsai's active IFC file
- **Status**: Alpha (version 0.0.0) — expect stability issues

#### Dependency Chain

```
IfcSverchok
  ├── Bonsai (BlenderBIM) — must be enabled first
  ├── Sverchok v1.2+ — node tree framework
  └── ifcopenshell 0.8.x — IFC file manipulation
```

All three dependencies must be installed and enabled. IfcSverchok registers its nodes only when all dependencies are available.

### Critical Warnings

**NEVER** use Blender undo inside an IfcSverchok node tree — this causes crashes due to the transient `SvIfcStore.file` becoming out of sync with Blender's undo state.

**NEVER** assume multiple IFC files can coexist — `SvIfcStore` manages exactly ONE transient file shared by ALL nodes in ALL trees. Running "Re-run all nodes" calls `purge()` which resets the entire file.

**NEVER** mix `use_bonsai_file=True` with `SvIfcWriteFile` export — writing overwrites Bonsai's active file state. Use one mode or the other.

**NEVER** expect IfcSverchok's `process()` to follow standard Sverchok conventions — `SvIfcCore` applies `zip_long_repeat` TWICE (double-nested), which is different from regular Sverchok nodes that apply it once.

**ALWAYS** include a geometry-to-representation conversion node (`SvIfcBMeshToIfcRepr` or `SvIfcSverchokToIfcRepr`) before `SvIfcCreateEntity` — entities without representations produce empty IFC files.

**ALWAYS** let `ensure_hirarchy()` handle missing spatial structure on export rather than manually creating IfcProject/IfcSite/IfcBuilding — the write node auto-completes the hierarchy.

### Decision Tree

```
Need to create IFC from Sverchok geometry?
├── From Blender mesh objects → SvIfcBMeshToIfcRepr node
├── From Sverchok vertices/faces → SvIfcSverchokToIfcRepr node
├── From IFC ShapeBuilder profiles → SvIfcSbRectangle → SvIfcSbExtrude → SvIfcSbRepresentation
└── From existing IFC file → SvIfcReadFile node

Need to build IFC hierarchy?
├── Quick setup → SvIfcQuickProjectSetup (creates project + site + building + storey)
├── Manual hierarchy → SvIfcAddSpatialElement for each level
└── Minimal → Just use SvIfcCreateEntity + SvIfcWriteFile (ensure_hirarchy fills gaps)

Need to query IFC data?
├── By STEP ID → SvIfcById node
├── By GlobalId → SvIfcByGuid node
├── By IFC class → SvIfcByType node
├── By selector query → SvIfcByQuery node
└── Read attributes → SvIfcReadEntity node

Need Bonsai integration?
├── Write to Bonsai's file → Toggle "Use Bonsai File" button, then run nodes
├── Read Bonsai's file → Toggle "Use Bonsai File", use query nodes
└── Export standalone → Keep use_bonsai_file=False (default), use SvIfcWriteFile
```

---

## Essential Patterns

### Pattern 1: SvIfcStore: Transient File Management

```python
# IfcOpenShell 0.8.x / IfcSverchok
# SvIfcStore manages a SINGLE transient IFC file across the node tree.

class SvIfcStore:
    file: Union[ifcopenshell.file, None] = None
    id_map: dict[str, Any] = {}        # node_id -> created entity data
    schema_identifiers = ["IFC4", "IFC2X3"]
    use_bonsai_file = False

    @staticmethod
    def purge() -> None:
        """Reset all state. Called before each full node tree re-run."""
        SvIfcStore.file = None
        SvIfcStore.id_map = {}

    @staticmethod
    def get_file() -> ifcopenshell.file:
        """Return the current transient file.
        Creates boilerplate IFC4 file if none exists.
        Returns Bonsai's file when use_bonsai_file is True."""
        if SvIfcStore.use_bonsai_file:
            return tool.Ifc.get()
        if SvIfcStore.file is None:
            SvIfcStore.create_boilerplate()
        return SvIfcStore.file
```

Key behaviors:
- `purge()` runs at the start of every full tree re-evaluation
- `id_map` tracks which node created which IFC entities (keyed by `node_id`)
- `get_file()` lazily creates a boilerplate IFC4 file on first access
- Only ONE file exists at a time — no multi-file workflows

### Pattern 2: SvIfcCore: Double-Nested Input Processing

```python
# IfcOpenShell 0.8.x / IfcSverchok
# SvIfcCore applies zip_long_repeat TWICE: different from standard Sverchok.

from sverchok.data_structure import zip_long_repeat

class SvIfcCore:
    sv_input_names: list[str]

    def process(self) -> None:
        # Gather all inputs
        sv_inputs_nested = []
        for name in self.sv_input_names:
            sv_inputs_nested.append(self.inputs[name].sv_get())

        # FIRST zip_long_repeat: across objects (outer nesting level)
        for sv_input_nested in zip_long_repeat(*sv_inputs_nested):
            # SECOND zip_long_repeat: within each object (inner nesting level)
            for sv_input in zip_long_repeat(*sv_input_nested):
                sv_input = list(sv_input)
                self.process_ifc(*sv_input)
```

Why double-nested: Sverchok data is `[[obj1_data], [obj2_data]]`. The first zip matches across inputs at the object level. The second zip iterates within each object, processing individual items. This allows a single `SvIfcCreateEntity` node to create multiple entities across multiple objects.

### Pattern 3: Sverchok Geometry to IFC Representation

```python
# IfcOpenShell 0.8.x / IfcSverchok
# Converting Sverchok geometry (vertices, edges, faces) to IFC representation.
# Node: SvIfcSverchokToIfcRepr

# In a Sverchok node tree, connect:
# [Plane Generator] → Vertices → [SvIfcSverchokToIfcRepr] → Representation → [SvIfcCreateEntity]
#                   → Faces    → /

# SvIfcSverchokToIfcRepr internally:
# 1. Normalizes vertex/edge/face data to 4-deep nesting
# 2. Zips geometry objects together
# 3. For each geometry object:
#    - Creates IfcCartesianPointList3D from vertices
#    - Creates IfcTriangulatedFaceSet or IfcPolygonalFaceSet from faces
#    - Wraps in IfcShapeRepresentation with context_type/context_identifier
# 4. Outputs IFC representation entity IDs
```

### Pattern 4: Blender Mesh to IFC Representation

```python
# IfcOpenShell 0.8.x / IfcSverchok
# Converting Blender mesh objects to IFC representations.
# Node: SvIfcBMeshToIfcRepr

# In a Sverchok node tree, connect:
# [Objects In] → Objects → [SvIfcBMeshToIfcRepr] → Representations → [SvIfcCreateEntity]
#                                                 → Locations      → /

# SvIfcBMeshToIfcRepr internally:
# 1. Takes Blender mesh objects as input
# 2. Separates loose parts into individual geometry sets
# 3. Generates IFC representation per object
# 4. Preserves world matrices as Locations output
# 5. Outputs: Representations (IFC repr IDs) + Locations (4x4 matrices)

# Inputs:
#   context_type: "Model" or "Plan"
#   context_identifier: "Body", "Annotation", "Box", or "Axis"
#   target_view: geometric target view
#   blender_objects: list of Blender mesh objects
```

### Pattern 5: 6-Step IFC Generation Workflow

```python
# IfcOpenShell 0.8.x / IfcSverchok
# Complete workflow for generating a valid IFC file from Sverchok.

# Step 1: Generate Geometry
# Use any Sverchok generator nodes (Box, Plane, Cylinder, etc.)
# OR use ShapeBuilder nodes (SvIfcSbRectangle → SvIfcSbExtrude)
# OR use Blender Objects In node

# Step 2: Convert to IFC Representation
# [SvIfcSverchokToIfcRepr]: from Sverchok vertices/edges/faces
# [SvIfcBMeshToIfcRepr]   : from Blender mesh objects
# [SvIfcSbRepresentation] : from ShapeBuilder geometry

# Step 3: Create IFC Entities
# [SvIfcCreateEntity]
#   IfcClass = "IfcWall"  (or IfcSlab, IfcColumn, etc.)
#   Names = "Wall_001"
#   Representation = (from Step 2)

# Step 4: Build Spatial Hierarchy
# [SvIfcAddSpatialElement]
#   IfcClass = "IfcBuildingStorey"
#   Elements = (from Step 3)
# Or skip: ensure_hirarchy() auto-completes on export

# Step 5: Add Properties (Optional)
# [SvIfcAddPset]
#   Name = "Pset_WallCommon"
#   Properties = '{"IsExternal": true, "LoadBearing": false}'

# Step 6: Export
# [SvIfcWriteFile]
#   path = "/path/to/output.ifc"
#   Triggers ensure_hirarchy() automatically
```

### Pattern 6: ShapeBuilder Workflow

```python
# IfcOpenShell 0.8.x / IfcSverchok
# Using ShapeBuilder nodes to create IFC geometry natively.

# ShapeBuilder creates geometry directly as IFC entities,
# bypassing Blender/Sverchok mesh conversion entirely.

# Node chain:
# [SvIfcSbRectangle] → profile → [SvIfcSbExtrude] → shape → [SvIfcSbRepresentation] → repr
#   XDim = 0.3                     Magnitude = 3.0
#   YDim = 0.2

# SvIfcSbRectangle creates an IfcRectangleProfileDef
# SvIfcSbExtrude extrudes the profile into an IfcExtrudedAreaSolid
# SvIfcSbRepresentation wraps into IfcShapeRepresentation

# Available ShapeBuilder nodes:
# SvIfcSbRectangle: rectangle profile
# SvIfcSbExtrude  : extrusion along axis
# SvIfcSbRepresentation: wraps to shape representation
# SvSbMesh        : creates IFC mesh from vertices/polygons
# SvSbPolyline    : creates IFC polyline from vertices
# SvSbShapeOutput : converts IFC shape back to Sverchok geometry
# SvIfcSbTest     : debug/testing node
```

### Pattern 7: Automatic Hierarchy Completion (ensure_hirarchy)

```python
# IfcOpenShell 0.8.x / IfcSverchok
# Called automatically by SvIfcWriteFile before export.

# ensure_hirarchy() performs these steps:
# 1. Creates a default IfcBuilding if none exists in the file
# 2. Assigns orphaned IfcBuildingStorey elements to the building
# 3. Assigns uncontained IfcElement instances to the building
# 4. Creates a default IfcSite if none exists
# 5. Links Building → Site → Project

# This means a minimal workflow can skip manual hierarchy:
# [Geometry] → [Repr] → [CreateEntity] → [WriteFile]
# ensure_hirarchy() fills in Project/Site/Building automatically.

# NOTE: The function is named "ensure_hirarchy" (typo preserved from source).
# Do NOT attempt to call "ensure_hierarchy": the correct name has no second 'e'.
```

---

## Common Operations

### Complete Node Catalog (31 Nodes)

#### IFC Nodes (24)

| Node Class | Label | Purpose |
|------------|-------|---------|
| `SvIfcCreateFile` | IFC Create File | Creates new empty IFC file with specified schema |
| `SvIfcReadFile` | IFC Read File | Opens existing IFC file from disk |
| `SvIfcWriteFile` | IFC Write File | Exports transient file to disk (calls ensure_hirarchy) |
| `SvIfcCreateEntity` | IFC Create Entity | Creates IFC entities with geometry and placement |
| `SvIfcCreateShape` | IFC Create Blender Shape | Converts IFC entities to Blender mesh objects |
| `SvIfcReadEntity` | IFC Read Entity | Reads entity, exposes all attributes as outputs |
| `SvIfcPickIfcClass` | IFC Class Picker | UI picker for IFC classes by product category |
| `SvIfcById` | IFC By Id | Retrieves entities by STEP file ID |
| `SvIfcByGuid` | IFC By Guid | Retrieves entities by GlobalId |
| `SvIfcByType` | IFC By Type | Queries all entities of a given type |
| `SvIfcByQuery` | IFC By Query | Queries using IfcOpenShell selector syntax |
| `SvIfcAdd` | IFC Add | Adds an entity to the IFC file |
| `SvIfcAddPset` | IFC Add Pset | Creates or edits property sets on elements |
| `SvIfcAddSpatialElement` | IFC Add Spatial Element | Creates spatial elements, assigns contained elements |
| `SvIfcRemove` | IFC Remove | Removes an entity from the IFC file |
| `SvIfcGenerateGuid` | IFC Generate Guid | Generates IFC-compliant GUID |
| `SvIfcGetProperty` | IFC Get Property | Retrieves property value from a property set |
| `SvIfcGetAttribute` | IFC Get Attribute | Retrieves a direct attribute value |
| `SvIfcSelectBlenderObjects` | IFC Select Blender Objects | Selects Blender objects by matching GlobalId |
| `SvIfcApi` | IFC API | Generic node calling any ifcopenshell.api function |
| `SvIfcBMeshToIfcRepr` | IFC BMesh to IFC Repr | Converts Blender objects to IFC representations |
| `SvIfcSverchokToIfcRepr` | IFC Sverchok to IFC Repr | Converts Sverchok geometry to IFC representations |
| `SvIfcCreateProject` | IFC Create Project | Adds IfcProject, units, and representation context |
| `SvIfcQuickProjectSetup` | IFC Quick Project Setup | Creates complete IFC file with project metadata |

#### IFC Shape Builder Nodes (7)

| Node Class | Label | Purpose |
|------------|-------|---------|
| `SvIfcSbRectangle` | IFC Rectangle | Creates IFC rectangle profile via ShapeBuilder |
| `SvIfcSbExtrude` | IFC Extrude | Extrudes IFC profile along axis |
| `SvIfcSbRepresentation` | IFC Representation | Wraps items into IfcShapeRepresentation |
| `SvIfcSbTest` | IFC SB Test | Test/debug node |
| `SvSbShapeOutput` | IFC Shape Output | Converts IFC shape to Sverchok geometry |
| `SvSbMesh` | IFC Mesh | Creates IFC mesh from vertices and polygons |
| `SvSbPolyline` | IFC Polyline | Creates IFC polyline from vertices |

### SvIfcStore Static Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `purge` | `() -> None` | Resets file, id_map. Called before full re-run |
| `get_file` | `() -> ifcopenshell.file` | Returns transient file (creates if needed) |
| `create_boilerplate` | `() -> None` | Creates minimal IFC4 file with header data |

### SvIfcStore Class Attributes

| Attribute | Type | Purpose |
|-----------|------|---------|
| `file` | `ifcopenshell.file \| None` | The single transient IFC file |
| `id_map` | `dict[str, Any]` | Maps node_id to created IFC entity data |
| `schema_identifiers` | `list[str]` | Supported schemas: `["IFC4", "IFC2X3"]` |
| `use_bonsai_file` | `bool` | When True, `get_file()` returns Bonsai's file |

### Geometry Mode Comparison

| Aspect | SvIfcBMeshToIfcRepr | SvIfcSverchokToIfcRepr |
|--------|---------------------|------------------------|
| Input source | Blender mesh objects | Sverchok vertices/edges/faces |
| Loose parts | Auto-separated | N/A (already separate objects) |
| World matrix | Preserved as Locations output | Not applicable |
| Outputs | Representations + Locations | Representations only |
| Use when | Working with existing Blender geometry | Working with procedural Sverchok geometry |

### Known Issues and Limitations

| Issue | Impact | Workaround |
|-------|--------|------------|
| Blender crashes during undo | Data loss | Save frequently, avoid undo in IFC trees |
| Single transient file | No multi-file workflows | Process one IFC file at a time |
| No type/material library | Cannot manage IfcTypeProduct or IfcMaterial | Use ifcopenshell.api directly via SvIfcApi node |
| Limited geometry types | Only basic shapes via ShapeBuilder | Use SvIfcSverchokToIfcRepr for complex geometry |
| No infrastructure support | IfcBridge, IfcRoad unavailable | Not supported in current version |
| No MVD validation | No compliance checking | Validate externally after export |
| No quantity takeoff | No IfcElementQuantity nodes | Use ifcopenshell.api via SvIfcApi node |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for SvIfcStore, SvIfcCore, all 31 nodes, and ensure_hirarchy
- [references/examples.md](references/examples.md) — Working code examples for IFC generation workflows, ShapeBuilder, Bonsai integration
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with IfcSverchok, with explanations

### Cross-References

- [ifcos-core-concepts](../../../ifcopenshell/core/ifcos-core-concepts/SKILL.md) — IfcOpenShell fundamentals
- [ifcos-syntax-elements](../../../ifcopenshell/syntax/ifcos-syntax-elements/SKILL.md) — IFC element types and attributes
- [ifcos-impl-creation](../../../ifcopenshell/impl/ifcos-impl-creation/SKILL.md) — Creating IFC entities with ifcopenshell.api
- [bonsai-core-architecture](../../../bonsai/core/bonsai-core-architecture/SKILL.md) — Bonsai addon architecture

### Official Sources

- https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
- https://community.osarch.org/discussion/284/sverchok-ifc
