# Sverchok Extensions and IfcSverchok: Comprehensive Research

> Research document for the Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package.
> Research date: 2026-03-07
> Scope: IfcSverchok (60%), TopologicSverchok, Sverchok-Extra, other extensions, extension development.
> Sources verified against IfcOpenShell v0.8.0 source code and community documentation.
> Status: Complete

---

## Table of Contents

1. [IfcSverchok](#1-ifcsverchok)
   - [1.1 Overview and Purpose](#11-overview-and-purpose)
   - [1.2 Architecture](#12-architecture)
   - [1.3 Complete Node Catalog](#13-complete-node-catalog)
   - [1.4 IFC File Generation Workflow](#14-ifc-file-generation-workflow)
   - [1.5 Two Geometry Modes](#15-two-geometry-modes)
   - [1.6 Integration with Bonsai](#16-integration-with-bonsai)
   - [1.7 Installation and Dependencies](#17-installation-and-dependencies)
   - [1.8 GSoC 2022 Expansion](#18-gsoc-2022-expansion)
   - [1.9 Status: Alpha](#19-status-alpha)
   - [1.10 Working Examples](#110-working-examples)
2. [TopologicSverchok](#2-topologicsverchok)
3. [Sverchok-Extra](#3-sverchok-extra)
4. [Other Extensions](#4-other-extensions)
5. [Extension Development](#5-extension-development)

---

## 1. IfcSverchok

### 1.1 Overview and Purpose

IfcSverchok is a Blender addon that extends Sverchok (the visual programming environment for Blender) with nodes for creating, reading, and manipulating IFC (Industry Foundation Classes) data. It provides the open-source equivalent of Grasshopper's GeometryGym plugin for Rhinoceros, but within Blender's node-based workflow.

**Source location**: `https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok`

**Primary goals**:

- Enable parametric generation of IFC-compliant building models using visual programming nodes.
- Allow both creation of new IFC files from scratch and manipulation of existing IFC models.
- Provide two geometry input paths: from Blender mesh objects and from Sverchok procedural geometry (vertices, edges, faces).
- Bridge the gap between parametric design in Sverchok and BIM data output in IFC format.

**bl_info from source** (`__init__.py`):

```python
bl_info = {
    "name": "IFC for Sverchok",
    "author": "Martina Jakubowska, Dion Moult",
    "version": (0, 0, 0),
    "blender": (3, 1, 0),
    "location": "Node Editor",
    "category": "Node",
    "description": "An extension to Sverchok to work with IFC data",
    "tracker_url": "https://github.com/IfcOpenShell/IfcOpenShell/issues",
}
```

The addon was initiated by Dion Moult (creator of IfcOpenShell/Bonsai) in 2020 and expanded significantly during Google Summer of Code 2022 by Martina Jakubowska (mdjska).

### 1.2 Architecture

#### Addon Structure

IfcSverchok is structured as a standard Blender addon that registers itself as a Sverchok extension. The directory layout in `src/ifcsverchok/` is:

```
ifcsverchok/
  __init__.py           # Addon registration, node indexing, operators, panel
  helper.py             # SvIfcCore base class, SvIfcStore, socket utilities
  nodes/
    __init__.py
    ifc/
      __init__.py
      add.py                    # SvIfcAdd
      add_pset.py               # SvIfcAddPset
      add_spatial_element.py    # SvIfcAddSpatialElement
      api.py                    # SvIfcApi
      bmesh_to_ifc.py           # SvIfcBMeshToIfcRepr
      by_guid.py                # SvIfcByGuid
      by_id.py                  # SvIfcById
      by_query.py               # SvIfcByQuery
      by_type.py                # SvIfcByType
      create_entity.py          # SvIfcCreateEntity
      create_file.py            # SvIfcCreateFile
      create_project.py         # SvIfcCreateProject
      create_shape.py           # SvIfcCreateShape
      generate_guid.py          # SvIfcGenerateGuid
      get_attribute.py          # SvIfcGetAttribute
      get_property.py           # SvIfcGetProperty
      pick_ifc_class.py         # SvIfcPickIfcClass
      quick_project_setup.py    # SvIfcQuickProjectSetup
      read_entity.py            # SvIfcReadEntity
      read_file.py              # SvIfcReadFile
      remove.py                 # SvIfcRemove
      select_blender_objects.py # SvIfcSelectBlenderObjects
      sverchok_to_ifc.py        # SvIfcSverchokToIfcRepr
      write_file.py             # SvIfcWriteFile
      shape_builder/
        extrude.py              # SvIfcSbExtrude
        mesh.py                 # SvSbMesh
        polyline.py             # SvSbPolyline
        rectangle.py            # SvIfcSbRectangle
        representation.py       # SvIfcSbRepresentation
        shape_output.py         # SvSbShapeOutput
        test.py                 # Test utilities
```

#### Class Hierarchy

Every IfcSverchok node inherits from three base classes:

```python
class SvIfcSomeNode(bpy.types.Node, SverchCustomTreeNode, ifcsverchok.helper.SvIfcCore):
    ...
```

- `bpy.types.Node` -- Blender's node API.
- `SverchCustomTreeNode` -- Sverchok's custom tree node base providing `sv_init`, `process`, and socket utilities.
- `SvIfcCore` -- IfcSverchok's base class providing `process_ifc()` as an abstract method, plus data flattening and iteration logic for nested socket data.

#### SvIfcCore Processing Model

The `SvIfcCore` class in `helper.py` provides:

- `process()` -- Handles nested input data from `self.sv_input_names`, iterates through combinations, and calls `process_ifc()` for each.
- `process_ifc(*args, **kwargs)` -- Abstract method that subclasses implement with their IFC logic.
- Inputs and outputs are "double nested" (list-of-lists) for proper data flow between nodes.

#### SvIfcStore

The `SvIfcStore` class manages a transient IFC file in memory:

- Maintains a module-level dictionary mapping GUIDs to `ifcopenshell.file` objects.
- Provides `get_file()` to access the current working IFC file.
- Tracks entity-to-node mappings via `id_map` for edit/delete operations.
- Enables the panel's "Re-run all nodes" and "Write File" buttons.

#### Helper Functions

- `get_socket_value(socket, name, value_type)` -- Retrieves values from sockets; supports "SINGLE", "CONTAINER", and "FLATTEN" modes.
- `set_socket_value(socket, name, value)` -- Assigns values to output sockets with proper nesting.
- `create_socket(collection, name, description, data_type)` -- Factory function for creating typed sockets.
- `get_file()` -- Returns the active IFC file from SvIfcStore.

#### Node Categories

The `__init__.py` organizes nodes into two Sverchok menu categories:

1. **IFC** -- 24 nodes for file operations, entity management, queries, and geometry conversion.
2. **IFC Shape Builder** -- 6 user-facing nodes for creating IFC geometry primitives (extrusions, rectangles, polylines, meshes).

#### Panel and Operators

Two operators are registered on a custom panel in the Sverchok node editor:

- **IFC_Sv_UpdateCurrent** ("Re-run all nodes") -- Forces recalculation of the entire node tree and resets the transient IFC file.
- **IFC_Sv_write_file** ("Write File") -- Exports the transient IFC file with automatic hierarchy validation via `ensure_hirarchy()`.

### 1.3 Complete Node Catalog

The following lists every IfcSverchok node in the v0.8.0 codebase, organized by category.

#### IFC Category (24 nodes)

| # | bl_idname | bl_label | Inputs | Outputs | Purpose |
|---|-----------|----------|--------|---------|---------|
| 1 | `SvIfcCreateFile` | IFC Create File | `schema` (str, default "IFC4") | `file` (ifcopenshell.file) | Creates a new empty IFC file with the specified schema version |
| 2 | `SvIfcReadFile` | IFC Read File | `path` (str) | `file` (ifcopenshell.file) | Opens an existing IFC file from disk |
| 3 | `SvIfcWriteFile` | IFC Write File | `path` (str) | `output` (str, status message) | Writes the transient IFC file to disk; calls `ensure_hirarchy()` to create missing IfcBuilding/IfcSite/IfcProject structure |
| 4 | `SvIfcCreateProject` | IFC Create Project | `file` (ifcopenshell.file), `project_name` (str) | `file` (ifcopenshell.file) | Creates an IfcProject with length units and Model/Body representation contexts |
| 5 | `SvIfcQuickProjectSetup` | IFC Quick Project Setup | `filename`, `timestring`, `organization`, `creator`, `schema_identifier` (default "IFC4"), `application_version`, `timestamp`, `application` (default "IfcOpenShell"), `project_globalid`, `project_name` | `file` (ifcopenshell.file) | One-node project creation using `ifcopenshell.template.create()` with customizable metadata |
| 6 | `SvIfcCreateEntity` | IFC Create Entity | `Names` (str), `Descriptions` (str), `IfcClass` (str, mandatory), `Representations` (IFC repr IDs), `Locations` (matrices) | `Entities` (entity IDs) | Creates IFC entities with name, description, representation, and placement. Uses `ifcopenshell.api.root.create_entity()` |
| 7 | `SvIfcReadEntity` | IFC Read Entity | `entity` (entity ID) | `id`, `is_a`, + dynamic outputs per entity attribute | Decomposes an IFC entity into its schema attributes. Dynamically generates output sockets based on entity type |
| 8 | `SvIfcPickIfcClass` | IFC Pick Class | `ifc_product` (enum: IfcElement, IfcElementType, IfcSpatialElement, etc.), `ifc_class` (enum, dynamically populated) | `IfcClass` (str) | Dropdown-based IFC class selector using `ifcopenshell.schema_wrapper` to traverse class hierarchy |
| 9 | `SvIfcAdd` | IFC Add | `file` (ifcopenshell.file), `entity` (entity_instance) | `file` (ifcopenshell.file), `entity` (entity_instance) | Adds an entity to an IFC file via `file.add(entity)` |
| 10 | `SvIfcRemove` | IFC Remove | `file` (ifcopenshell.file), `entity` (entity_instance) | `file` (ifcopenshell.file) | Removes an entity from an IFC file recursively |
| 11 | `SvIfcAddPset` | IFC Add Pset | `Name` (str, e.g. "Pset_WallCommon"), `Properties` (JSON str, e.g. `{"IsExternal":"True"}`), `Elements` (entity IDs) | `Entity` (pset IDs) | Adds or edits property sets on IFC elements using `ifcopenshell.api.pset.add_pset()` and `edit_pset()` |
| 12 | `SvIfcAddSpatialElement` | IFC Add Spatial Element | `Names` (str), `IfcClass` (str, default "IfcSpace"), `Elements` (entity IDs) | `Entities` (spatial element IDs) | Creates spatial elements and assigns containment via `ifcopenshell.api.spatial.assign_container()` or aggregation via `ifcopenshell.api.aggregate.assign_object()` |
| 13 | `SvIfcByGuid` | IFC By Guid | `guid` (str) | `Entities` (entity_instances) | Retrieves IFC entities by their GlobalId attribute |
| 14 | `SvIfcById` | IFC By Id | `id` (str) | `Entities` (entity_instances) | Retrieves IFC entities by their STEP file ID |
| 15 | `SvIfcByType` | IFC By Type | `ifc_product` (enum), `ifc_class` (enum), `custom_ifc_class` (str) | `Entities` (entity_instances), `Entity Ids` (ints) | Retrieves all entities of a given IFC type. Custom class name overrides dropdown selection |
| 16 | `SvIfcByQuery` | IFC By Query | `query` (str) | `Entity` (entity_instances) | Queries entities using `ifcopenshell.util.selector.filter_elements()` with IFC query syntax |
| 17 | `SvIfcGetAttribute` | IFC Get Attribute | `entity` (entity ID), `attribute_name` (str) | `value` (str) | Retrieves a single attribute from an entity, supports both name-based and index-based access |
| 18 | `SvIfcGetProperty` | IFC Get Property | `entity_ids` (entity IDs), `pset_name` (str), `prop_name` (str) | `value` (str) | Retrieves property values from property sets using `ifcopenshell.util.element.get_psets()` |
| 19 | `SvIfcGenerateGuid` | IFC Generate Guid | (none) | `guid` (str) | Generates a new IFC GUID using `ifcopenshell.guid.new()` |
| 20 | `SvIfcBMeshToIfcRepr` | IFC BMesh to Repr | `context_type` (enum: Model/Plan), `context_identifier` (enum: Body/Annotation/Box/Axis), `target_view` (enum), `blender_objects` (Blender objects) | `Representations` (repr IDs), `Locations` (world matrices) | Converts Blender mesh objects to IFC ShapeRepresentations using `ifcopenshell.api.geometry.add_representation()` |
| 21 | `SvIfcSverchokToIfcRepr` | IFC Sverchok to Repr | `context_type` (enum), `context_identifier` (enum), `target_view` (enum), `Vertices` (verts), `Edges` (edges), `Faces` (faces) | `Representation(s)` (repr IDs) | Converts Sverchok procedural geometry (verts/edges/faces) to IFC mesh representations using `ifcopenshell.api.geometry.add_mesh_representation()` |
| 22 | `SvIfcCreateShape` | IFC Create Shape | `Entities` (entity IDs) | `Object(s)` (Blender objects) | Creates Blender 3D mesh objects from IFC entities for viewport visualization using IfcImporter |
| 23 | `SvIfcSelectBlenderObjects` | IFC Select Blender Objects | `entities` (entity_instances) | (none, performs selection) | Selects Blender objects in the viewport that correspond to given IFC entities by matching GlobalId |
| 24 | `SvIfcApi` | IFC API | `usecase` (str), + dynamic inputs per API function | `file` (result) | Generic node exposing any `ifcopenshell.api.run()` usecase. Dynamically generates input sockets from API docstrings via `ifcopenshell.api.extract_docs()` |

#### IFC Shape Builder Category (6 user-facing nodes)

| # | bl_idname | bl_label | Inputs | Outputs | Purpose |
|---|-----------|----------|--------|---------|---------|
| 1 | `SvIfcSbRectangle` | IFC Rectangle | `Size (2D)` (tuple[float, float]) | `Rectangle` (entity_instance) | Creates an IFC rectangle profile using `ShapeBuilder.rectangle()` |
| 2 | `SvSbPolyline` | IFC Polyline | `Vers` (vertices), `Closed` (bool) | `Representation Item` (entity_instance) | Creates an IFC polyline from vertex data using `ShapeBuilder` |
| 3 | `SvIfcSbExtrude` | IFC Extrude | `Curve` (entity_instance), `Magnitude` (float), `Position` (XYZ vertex) | `Extruded Profile` (entity_instance) | Extrudes a profile curve along X, Y, or Z axis using `ShapeBuilder.extrude()` |
| 4 | `SvSbMesh` | IFC Mesh | `Vers` (vertices), `Pols` (polygons) | `Representation Item` (entity_instance) | Creates an IFC mesh from vertices and polygons via `ShapeBuilder` |
| 5 | `SvIfcSbRepresentation` | IFC Representation | `Representation Item` (entity_instance) | `Shape Representation` (entity_instance) | Wraps representation items into an IfcShapeRepresentation with Model/Body/MODEL_VIEW context |
| 6 | `SvSbShapeOutput` | IFC Shape Output | `Representation` (entity_instance) | `Vers` (vertices), `Edgs` (edges), `Pols` (polygons) | Converts IfcShapeRepresentation back to Sverchok geometry data using `ifcopenshell.geom.create_shape()` |

**Total: 30 user-facing nodes** (24 IFC + 6 Shape Builder).

### 1.4 IFC File Generation Workflow

The fundamental workflow for generating an IFC file from a Sverchok node tree follows these steps:

#### Step 1: Create or Read an IFC File

Use `SvIfcCreateFile` (specifying "IFC4" or "IFC2X3") or `SvIfcQuickProjectSetup` (which provides metadata fields) to create a new file. Alternatively, use `SvIfcReadFile` to open an existing IFC file.

#### Step 2: Set Up Project Structure

If using `SvIfcCreateFile`, connect it to `SvIfcCreateProject` to establish:
- An IfcProject entity with a name
- Length units (SI LENGTHUNIT)
- A Model representation context
- A Body sub-context for MODEL_VIEW

If using `SvIfcQuickProjectSetup`, this is handled automatically via `ifcopenshell.template.create()`.

#### Step 3: Define Geometry

Choose one of three paths:

**Path A -- From Blender Objects:** Connect Blender mesh objects to `SvIfcBMeshToIfcRepr`.

**Path B -- From Sverchok Geometry:** Connect Sverchok vertex/edge/face data to `SvIfcSverchokToIfcRepr`.

**Path C -- From Shape Builder:** Use Shape Builder nodes (Rectangle/Polyline -> Extrude -> Representation).

#### Step 4: Create IFC Entities

Connect the IFC class from `SvIfcPickIfcClass` and the representations to `SvIfcCreateEntity`.

#### Step 5: Organize Spatial Structure

Use `SvIfcAddSpatialElement` to create spatial containers and assign elements.

#### Step 6: Add Properties

Use `SvIfcAddPset` to attach property sets with JSON-formatted key-value pairs.

#### Step 7: Write the File

Connect the file path to `SvIfcWriteFile` or use the panel button "Write File".

#### Workflow Diagram (Conceptual)

```
[Create File] -> [Create Project] -> [Create Entity] -> [Add Spatial Element] -> [Write File]
                                          ^                      ^
                                          |                      |
                              [Pick IFC Class]          [Add Pset]
                                          |
                              [BMesh to IFC Repr]    OR    [Sverchok to IFC Repr]
                                     ^                              ^
                                     |                              |
                              [Blender Objects]          [Sverchok Verts/Edges/Faces]
```

### 1.5 Two Geometry Modes

IfcSverchok provides two distinct paths for converting geometry into IFC representations, plus a third path using the Shape Builder nodes.

#### Mode 1: From Blender Objects (SvIfcBMeshToIfcRepr)

**Inputs**: context_type (Model/Plan), context_identifier (Body/Annotation/Box/Axis), target_view, blender_objects
**Outputs**: Representations (repr IDs), Locations (world matrices)
**Process**: Separates joined geometry using `bpy.ops.mesh.separate(type="LOOSE")`, then calls `ifcopenshell.api.geometry.add_representation()` for each mesh.
**Use case**: When users model geometry in Blender's viewport and want to classify it as IFC elements.

#### Mode 2: From Sverchok Geometry (SvIfcSverchokToIfcRepr)

**Inputs**: context_type, context_identifier, target_view, Vertices, Edges, Faces
**Outputs**: Representation(s) (repr IDs)
**Process**: Normalizes input data, zips verts/edges/faces into groups, calls `ifcopenshell.api.geometry.add_mesh_representation()` for each.
**Use case**: When users generate parametric geometry entirely within Sverchok.

#### Mode 3: From Shape Builder Nodes

**Nodes**: SvIfcSbRectangle, SvSbPolyline, SvIfcSbExtrude, SvSbMesh, SvIfcSbRepresentation
**Process**: Uses IfcOpenShell's ShapeBuilder utility to construct IFC geometry at the representation level.
**Use case**: Creates proper IfcExtrudedAreaSolid geometry (cleaner BIM geometry).

#### Comparison

| Aspect | BMesh to IFC | Sverchok to IFC | Shape Builder |
|--------|-------------|-----------------|---------------|
| Input source | Blender viewport objects | Sverchok geometry sockets | IFC primitives |
| Transformation | World matrix preserved | Must supply location separately | Position input on Extrude |
| API call | `geometry.add_representation()` | `geometry.add_mesh_representation()` | `ShapeBuilder` methods |
| IFC geometry quality | Tessellated (IfcFacetedBrep) | Tessellated (IfcPolygonalFaceSet) | Parametric (IfcExtrudedAreaSolid) |

### 1.6 Integration with Bonsai

#### Dependency Relationship

IfcSverchok requires Bonsai (formerly BlenderBIM) as a dependency:

```python
ensure_addons_are_enabled("bonsai", "sverchok")
```

Bonsai provides: the `ifcopenshell` Python library, IFC element tools for `SvIfcSelectBlenderObjects`, and the IfcImporter class for `SvIfcCreateShape`.

#### Output Workflow: IfcSverchok to Bonsai

1. **Generate IFC file** using IfcSverchok nodes.
2. **Open in Bonsai** via `File > Open IFC Project` or `bpy.ops.bim.load_project()`.
3. **Inspect and edit** using Bonsai's full IFC editing capabilities.
4. **Reload workflow**: Use `bpy.ops.bim.reload_ifc_file()` for live updates.

#### Bi-directional Workflow

- **Sverchok to Bonsai**: Generate IFC file, load in Bonsai for inspection.
- **Bonsai to Sverchok**: Use `SvIfcReadFile` to read existing IFC, query with `SvIfcByType`, `SvIfcByQuery`, `SvIfcGetProperty`.
- **Select Blender Objects**: `SvIfcSelectBlenderObjects` bridges IFC entities to Bonsai objects by matching GlobalId.

#### Ionut BIM Studio Demonstrated Workflows

Community member Ionut demonstrated advanced integration patterns:
- Parametric IFC wall with material layers (variable core thickness, fixed finish layers).
- Excel-to-IFC pipeline using LibreCalc spreadsheets.
- Parametric window creation combining Sverchok (2D) and Geometry Nodes (3D).
- Custom property integration between Blender objects and Bonsai elements.
- Floorplan generation with Topologic integration for spatial analysis.

### 1.7 Installation and Dependencies

#### Prerequisites

1. **Bonsai** -- Download from https://bonsaibim.org/download.html
2. **Sverchok** -- Download from https://github.com/nortikin/sverchok

#### Packaged Installation

1. Download IfcSverchok ZIP from https://github.com/IfcOpenShell/IfcOpenShell/releases
2. In Blender: `Edit > Preferences > Addons > Install`
3. Select ZIP, enable checkbox, save settings.

#### Source Installation (Developers)

```bash
git clone https://github.com/IfcOpenShell/IfcOpenShell.git
cd IfcOpenShell
ln -s src/ifcsverchok /path/to/blender/X.XX/scripts/addons/ifcsverchok
# Windows: mklink /D target src\ifcsverchok
```

#### Known Installation Issues

1. **Sverchok naming mismatch** (Issue #5913): Folder must be named "sverchok" exactly. Fix merged January 2025.
2. **Outdated build in docs** (Issue #5657): Old builds checked for "blenderbim" instead of "bonsai".
3. **Blender 4.x extension system**: Non-standard folder names cause detection failures.

### 1.8 GSoC 2022 Expansion

#### Project Information

- **Program**: Google Summer of Code 2022, BRL-CAD/IfcOpenShell umbrella.
- **Project**: "Create visual programming nodes for generating BIM data with IfcSverchok" (opencax/GSoC #43).
- **Contributor**: Martina Jakubowska (mdjska).
- **Mentors**: Dion Moult (Moult) and Thomas Krijnen (aothms).
- **Track**: Long (350 hours), Medium difficulty.
- **Documentation**: https://mdjska.github.io/GSoC/

#### Design Decisions

Three approaches explored:
1. **Schema-based**: Direct IFC schema object nodes. Rejected as too verbose.
2. **Relationship-based**: Explicit parent/child hierarchy.
3. **Geometry-based**: Leveraging existing geometry nodes with implicit hierarchies.

The "IfcSpaces-first" methodology was adopted: create spaces, associate elements, assign metadata, build hierarchy upward, export.

#### What Was Added

Approximately 15 core nodes delivered by November 2022:
- Entity creation/reading: SvIfcCreateEntity, SvIfcReadEntity, SvIfcPickIfcClass
- Geometry conversion: SvIfcBMeshToIfcRepr, SvIfcSverchokToIfcRepr, SvIfcCreateShape
- Spatial organization: SvIfcAddSpatialElement
- Data operations: SvIfcAddPset, SvIfcGetProperty, SvIfcGetAttribute
- Entity lookup: SvIfcById, SvIfcByGuid, SvIfcByType
- File operations: SvIfcWriteFile
- Infrastructure: SvIfcStore, SvIfcCore, panel operators

#### Post-GSoC Development

- Merged into official IfcOpenShell repository (January 2023).
- Added: SvIfcApi (generic API node), Shape Builder nodes, SvIfcQuickProjectSetup, SvIfcSelectBlenderObjects, SvIfcByQuery.

#### Early History (Pre-GSoC)

- **November 2020**: Initial OSArch forum discussions.
- **September 2020**: Dion Moult opened Issue #1010 "Visual programming nodes for IFC!"
- **2020-2021**: Foundational node files created. IFC Create Shape existed for debugging.
- **July 2021**: Bug-fixing sessions with community contributors.
- **Decision**: IFC nodes live in IfcOpenShell repo (not upstream Sverchok) because "IFC editing is highly niche and requires many dependencies."

#### Proposed but Unimplemented Features

- Individual nodes per IfcBuildingElement subclass
- Material/construction library management
- Space adjacency detection
- Occupancy and schedule nodes
- MVD support, COBie generation
- Multiple export formats (JSON, CSV, XML)

### 1.9 Status: Alpha

#### What Works

- File creation and writing (IFC4, IFC2X3)
- Entity creation with geometry and placement
- Property set management
- Spatial structure (IfcSpace, IfcBuildingStorey, containment)
- Both geometry paths (BMesh and Sverchok)
- Shape Builder for parametric profiles
- Entity queries (ID, GUID, type, query string)
- File reading for existing IFC files
- Generic API node for any ifcopenshell function
- Entity editing (nodes track state for updates)

#### Known Issues

1. **Undo crashes** (critical): Ctrl+Z can crash Blender. Known since GSoC 2022.
2. **Classification loss**: Reloaded objects may appear as generic IfcElement instead of specific types.
3. **Addon detection failures**: Non-standard folder names cause "not installed" errors (Issues #5913, #5657). Fixed January 2025.
4. **GUID handling**: By Guid node expects base64-encoded GUID format. Hex GUIDs from Tag attribute fail (Issue #1479).
5. **No material nodes**: Must use generic API node for materials.
6. **No type/style nodes**: No dedicated IfcTypeProduct, IfcMaterialLayerSet nodes.
7. **Incomplete spatial hierarchy**: `ensure_hirarchy()` auto-creates missing structure as workaround.
8. **Limited documentation**: Only installation page and GSoC blog posts exist.
9. **Blender 4.x compatibility**: Extension system changes require fixes.
10. **Version**: `bl_info` version is `(0, 0, 0)`.

### 1.10 Working Examples

#### Example 1: IFC Wall from Sverchok Geometry

```
[Box MK2] -> [SvIfcSverchokToIfcRepr] -> [SvIfcCreateEntity] -> [SvIfcAddSpatialElement]
  (5x0.3x3)    (Model/Body/MODEL_VIEW)       ^  (IfcWall)                |
                                              |                           v
[SvIfcPickIfcClass] -------------------------+          [SvIfcWriteFile]
[SvIfcCreateFile] -> [SvIfcCreateProject] ------------------------------>
```

Steps:
1. Box MK2: X=5.0, Y=0.3, Z=3.0
2. Sverchok to Repr: context_type=Model, context_identifier=Body, target_view=MODEL_VIEW
3. Pick Class: ifc_product=IfcElement, ifc_class=IfcWall
4. Create Entity: connect IfcClass, Representations, Names="My Wall"
5. Create File (IFC4) -> Create Project -> Write File

#### Example 2: IFC Wall with Shape Builder

```
[SvIfcSbRectangle] -> [SvIfcSbExtrude] -> [SvIfcSbRepresentation] -> [SvIfcCreateEntity]
     (0.3, 3.0)          (Z-axis, 5.0)                                      (IfcWall)
```

Produces proper IfcExtrudedAreaSolid geometry.

#### Example 3: Reading and Querying

```
[SvIfcReadFile] -> [SvIfcByType] -> [SvIfcGetProperty]
   (path)          (IfcWall)        (Pset_WallCommon, IsExternal)
                       |
              [SvIfcReadEntity]  -- id, is_a, Name, Description, etc.
```

#### Example 4: Generic API Access

```
[SvIfcApi]
  usecase: "material.add_material"
  (dynamic inputs: Name, Category, Description)
```

#### Python Equivalent of Example 1

```python
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.template

file = ifcopenshell.template.create(filename="my_wall.ifc")
project = file.by_type("IfcProject")[0]
body = ifcopenshell.util.representation.get_context(file, "Model", "Body", "MODEL_VIEW")

wall = ifcopenshell.api.run("root.create_entity", file, ifc_class="IfcWall", name="My Wall")

vertices = [
    (0.0, 0.0, 0.0), (5.0, 0.0, 0.0), (5.0, 0.3, 0.0), (0.0, 0.3, 0.0),
    (0.0, 0.0, 3.0), (5.0, 0.0, 3.0), (5.0, 0.3, 3.0), (0.0, 0.3, 3.0),
]
faces = [
    (0, 1, 2, 3), (4, 5, 6, 7), (0, 1, 5, 4),
    (1, 2, 6, 5), (2, 3, 7, 6), (3, 0, 4, 7),
]
representation = ifcopenshell.api.run(
    "geometry.add_mesh_representation", file, context=body, vertices=vertices, faces=faces
)
ifcopenshell.api.run("geometry.assign_representation", file, product=wall, representation=representation)

site = ifcopenshell.api.run("root.create_entity", file, ifc_class="IfcSite")
building = ifcopenshell.api.run("root.create_entity", file, ifc_class="IfcBuilding")
storey = ifcopenshell.api.run("root.create_entity", file, ifc_class="IfcBuildingStorey")
ifcopenshell.api.run("aggregate.assign_object", file, products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", file, products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", file, products=[storey], relating_object=building)
ifcopenshell.api.run("spatial.assign_container", file, products=[wall], relating_structure=storey)

pset = ifcopenshell.api.run("pset.add_pset", file, product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", file, pset=pset, properties={"IsExternal": True})

file.write("/tmp/my_wall.ifc")
```

---

## 2. TopologicSverchok

### 2.1 Overview

**Source**: https://github.com/wassimj/TopologicSverchok
**License**: AGPL-3.0
**Author**: Wassim Jabi (Cardiff University)
**Stats**: 96 stars, 17 forks, 215 commits, 20 releases

Topologic is a software modeling library for hierarchical and topological representations of architectural spaces through non-manifold topology (NMT). TopologicSverchok brings this into Blender's Sverchok environment.

### 2.2 Topological Hierarchy

| Topology Class | Dimension | Description |
|---------------|-----------|-------------|
| **Vertex** | 0D | A point in space |
| **Edge** | 1D | A line segment between two vertices |
| **Wire** | 1D | A connected sequence of edges (open or closed) |
| **Face** | 2D | A bounded surface defined by a closed wire |
| **Shell** | 2D | A connected set of faces sharing edges |
| **Cell** | 3D | A closed volume bounded by a shell |
| **CellComplex** | 3D | A connected set of cells sharing faces |
| **Cluster** | Mixed | A collection of any topological entities |

### 2.3 Key Node Categories (~60 nodes)

- **Vertex**: ByCoordinates, AdjacentEdges, AdjacentFaces, Coordinates, Distance, NearestVertex
- **Edge**: ByStartVertexEndVertex, AdjacentEdges, AdjacentFaces, Length, Bisect, StartVertex, EndVertex
- **Wire**: ByEdges, Edges, Vertices, IsClosed, Length
- **Face**: ByWire, Area, Edges, Normal, Vertices, AdjacentCells, InternalBoundaries
- **Shell**: ByFaces, Faces, Edges, Vertices, IsClosed
- **Cell**: ByFaces, Volume, AdjacentCells, Faces, Vertices, InternalFaces
- **CellComplex**: ByFaces, Cells, ExternalBoundary, InternalFaces, NonManifoldFaces
- **Cluster**: ByTopologies, Topologies
- **Topology**: Boolean (Union, Intersection, Difference, Merge), Transform, Geometry
- **Graph**: Graph creation, adjacency analysis, shortest path
- **Dictionary**: Metadata attachment/retrieval
- **Conversion**: Topology.ByGeometry, Topology.Geometry

### 2.4 AEC Relevance

- **Building topology**: CellComplex = building, Cells = rooms, shared Faces = walls.
- **Spatial analysis**: Adjacency queries, connectivity, pathfinding.
- **Energy modeling**: Integration with OpenStudio, Ladybug, Honeybee.
- **IFC integration**: Optional ifcopenshell dependency for IFC spatial mapping.

### 2.5 Dependencies

Required: Blender >= 3.2, Sverchok >= 1.1.0, NumPy >= 1.22.4
Optional: IfcOpenShell, OpenStudio, Ladybug, Honeybee, Pandas, SciPy, DGL, Specklepy

---

## 3. Sverchok-Extra

### 3.1 Overview

**Source**: https://github.com/portnov/sverchok-extra
**License**: GPL-3.0
**Author**: portnov
**Stats**: 62 stars, 10 forks, 481 commits

Sandbox/nursery for experimental Sverchok nodes requiring additional dependencies.

### 3.2 Policy

- New dependency = Sverchok-Extra first.
- Niche features that most users do not need = separate addon.
- Mature nodes graduate to Sverchok core.

### 3.3 Node Categories

- **Surface Extra**: Smooth Bivariate Spline (SciPy), Implicit Surface Solver/Wrap, Surface Curvature Lines (SciPy)
- **Field Extra**: Vector Field Lines on a Surface (SciPy)
- **Data**: Spreadsheet, Data Item
- **Matrix Extra**: Project Matrix on Plane
- **Solid Extra**: Solid Waffle
- **Spatial Extra**: Delaunay 3D on Surface, Delaunay Mesh
- **SDF Primitives** (sdf library): SDF Box, Cylinder, and others
- **SDF Operations** (sdf library): Translate, Scale, Rotate, Orient, Boolean, Blend, Dilate/Erode, Shell, Twist, Bend, Slice, Extrude, Revolve, Generate

### 3.4 NURBS History

Originally developed NURBS nodes (using Geomdl): NURBS Curve, Interpolation Curve, Approximation Curve, Blend Curves, JSON to NURBS. Many have since migrated to Sverchok core.

---

## 4. Other Extensions

### 4.1 Sverchok-Open3d

**Source**: https://github.com/vicdoval/sverchok-open3d. GPL-3.0.

Point cloud and triangle mesh processing via Open3D:
- **Import/Export**: Open 3D Import, Export
- **Point Cloud**: Point Cloud In, Calc Normals, Downsampling, Masking, Transform
- **Triangle Mesh**: Mesh In/Out, from Point Cloud (Alpha Shape, Ball Pivoting, Poisson), Simplification, Cleaning, Smoothing, Sharpening, Subdivision, Deformation, Intersection Detection

AEC relevance: Scan-to-BIM workflows, as-built documentation, heritage digitization. Also serves as a template for extension development.

### 4.2 Ladybug Tools

**Source**: https://github.com/ladybug-tools/ladybug-blender. Alpha state.

~60 nodes for environmental analysis: climate analysis (EPW), solar radiation, daylighting, energy simulation (Honeybee/EnergyPlus), comfort analysis. LB Out node bridges to Blender/Sverchok geometry.

### 4.3 Other

- **Mega-Polis**: Urban design toolkit
- **Sverchok-Bmesh**: BMesh API nodes as separate addon

### 4.4 Extensions List

| Extension | Purpose | Status |
|-----------|---------|--------|
| Sverchok-Extra | Experimental geometry nodes | Active |
| Sverchok-Open3d | Point Cloud / Triangle Mesh | Active |
| Ladybug Tools | Environmental analysis | Alpha |
| IfcSverchok | BIM/IFC nodes | Alpha |
| TopologicSverchok | Non-manifold topology | Alpha |
| Mega-Polis | Urban design | Active |
| Sverchok-Bmesh | BMesh operations | Active |

### 4.5 Extension Loading Mechanism

Sverchok has no formal extension API. Extensions are independent Blender addons that:
1. Import `SverchCustomTreeNode` from `sverchok.node_tree`
2. Register node classes via `bpy.utils.register_class()`
3. Create menu entries via Sverchok's `add_node_menu` system
4. Verify Sverchok is installed before initializing

---

## 5. Extension Development

### 5.1 Minimal Node

```python
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

class SvMyNode(bpy.types.Node, SverchCustomTreeNode):
    """Description. Keywords: word1, word2"""
    bl_idname = "SvMyNode"  # Must match class name, start with Sv
    bl_label = "My Node"
    bl_icon = "OUTLINER_OB_EMPTY"

    my_param: bpy.props.FloatProperty(name="Param", default=1.0, update=updateNode)

    def sv_init(self, context):
        self.inputs.new("SvStringsSocket", "Input")
        self.outputs.new("SvStringsSocket", "Output")

    def draw_buttons(self, context, layout):
        layout.prop(self, "my_param")

    def process(self):
        if not self.outputs["Output"].is_linked:
            return
        data = self.inputs["Input"].sv_get()
        result = [[x * self.my_param for x in sub] for sub in data]
        self.outputs["Output"].sv_set(result)

def register():
    bpy.utils.register_class(SvMyNode)
def unregister():
    bpy.utils.unregister_class(SvMyNode)
```

### 5.2 Naming Conventions

- Class name: `Sv` prefix, alphanumeric only
- `bl_idname`: Must equal class name
- `bl_label`: Display name
- Docstring before `///` is searchable

### 5.3 Socket Types

`SvStringsSocket` (general), `SvVerticesSocket` (3D coords), `SvMatrixSocket` (4x4 matrices), `SvObjectSocket` (Blender objects), `SvColorSocket` (RGBA), `SvSurfaceSocket`, `SvCurveSocket`, `SvSolidSocket`

### 5.4 Registration Patterns

**Direct**: Each module has `register()`/`unregister()`, called from `__init__.py`.

**Dynamic (IfcSverchok pattern)**: `nodes_index()` returns category-organized list, `make_node_list()` imports dynamically.

### 5.5 Domain-Specific Base Class

IfcSverchok pattern: third base class (SvIfcCore) for shared domain logic:

```python
class MyDomainCore:
    def process(self):
        # iterate inputs, call process_domain()
    def process_domain(self, *args):
        raise NotImplementedError

class SvMyDomainNode(bpy.types.Node, SverchCustomTreeNode, MyDomainCore):
    ...
```

### 5.6 Packaging

```
my_extension/
    __init__.py       # bl_info (category="Node", location="Node Editor"), register(), unregister()
    nodes/
        __init__.py
        node_1.py
        node_2.py
```

---

## Appendix A: IfcSverchok Node Quick Reference

### File Operations
| Node | Action |
|------|--------|
| SvIfcCreateFile | Create new IFC file |
| SvIfcReadFile | Open existing IFC file |
| SvIfcWriteFile | Save IFC file to disk |
| SvIfcCreateProject | Set up project with units/contexts |
| SvIfcQuickProjectSetup | One-step project creation |

### Entity Management
| Node | Action |
|------|--------|
| SvIfcCreateEntity | Create IFC elements with geometry |
| SvIfcReadEntity | Decompose entity into attributes |
| SvIfcPickIfcClass | Select IFC class from dropdown |
| SvIfcAdd | Add entity to file |
| SvIfcRemove | Remove entity from file |
| SvIfcGenerateGuid | Generate new GUID |

### Queries
| Node | Action |
|------|--------|
| SvIfcByGuid | Find by GlobalId |
| SvIfcById | Find by STEP ID |
| SvIfcByType | Find all of a type |
| SvIfcByQuery | Filter with query syntax |
| SvIfcGetAttribute | Get attribute value |
| SvIfcGetProperty | Get property from pset |

### Geometry
| Node | Action |
|------|--------|
| SvIfcBMeshToIfcRepr | Blender mesh to IFC repr |
| SvIfcSverchokToIfcRepr | Sverchok geometry to IFC repr |
| SvIfcCreateShape | IFC entity to Blender object |

### Spatial/Data
| Node | Action |
|------|--------|
| SvIfcAddSpatialElement | Create/assign spatial containers |
| SvIfcAddPset | Add property sets |
| SvIfcSelectBlenderObjects | Select Bonsai objects by entity |
| SvIfcApi | Generic ifcopenshell API access |

### Shape Builder
| Node | Action |
|------|--------|
| SvIfcSbRectangle | Create rectangle profile |
| SvSbPolyline | Create polyline from vertices |
| SvIfcSbExtrude | Extrude profile along axis |
| SvSbMesh | Create mesh from verts/faces |
| SvIfcSbRepresentation | Wrap into ShapeRepresentation |
| SvSbShapeOutput | Convert repr to geometry |

---

## Appendix B: Source References

- IfcSverchok source: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
- Installation docs: https://docs.ifcopenshell.org/ifcsverchok/installation.html
- GSoC 2022 proposal: https://mdjska.github.io/GSoC/notes/GSoC_proposal_mdj/
- GSoC 2022 update: https://community.osarch.org/discussion/1244/sverchok-update-on-development-of-ifcsverchok-nodes-gsoc-2022
- GSoC 2022 start: https://community.osarch.org/discussion/1024/sverchok-development-of-ifcsverchok-nodes-gsoc-2022
- GSoC issue: https://github.com/opencax/GSoC/issues/43
- IFC wall discussion: https://community.osarch.org/discussion/606/sverchok-ifc-nodes-create-ifcwall
- Sverchok IFC forum: https://community.osarch.org/discussion/284/sverchok-ifc
- Feature request #1010: https://github.com/IfcOpenShell/IfcOpenShell/issues/1010
- Ionut BIM Studio: https://community.osarch.org/discussion/1986/ionut-bim-studio-experiments-sverchok-blender3d-blenderbim-and-other-free-and-os-tech
- TopologicSverchok: https://github.com/wassimj/TopologicSverchok
- Topologic software: https://topologic.app/software/
- Sverchok-Extra: https://github.com/portnov/sverchok-extra
- Sverchok-Open3d: https://github.com/vicdoval/sverchok-open3d
- Ladybug Tools: https://github.com/ladybug-tools/ladybug-blender
- Extensions page: https://nortikin.github.io/sverchok/docs/introduction/sverchok_extensions.html
- Node tutorial: https://github.com/nortikin/sverchok/wiki/Sverchok-Node-Tutorial
- HighLevelNode: https://github.com/nortikin/sverchok/wiki/HighLevelNode
- Issue #5913: https://github.com/IfcOpenShell/IfcOpenShell/issues/5913
- Issue #5657: https://github.com/IfcOpenShell/IfcOpenShell/issues/5657
- Issue #1479: https://github.com/IfcOpenShell/IfcOpenShell/issues/1479
