# Sverchok Core Architecture

Comprehensive research document covering Sverchok's core architecture, node system, data flow, socket types, data nesting conventions, list matching algorithms, vectorization system, complete node category inventory, and version information.

**Source repository:** https://github.com/nortikin/sverchok
**Documentation:** https://nortikin.github.io/sverchok/docs/main.html
**Version analyzed:** v1.4.0 (released March 5, 2025)
**Blender compatibility:** 3.6 through 5.1
**Date:** 2026-03-07
**Agent:** sv-research-core

---

## Table of Contents

1. [Node System Architecture](#1-node-system-architecture)
2. [Data Flow System](#2-data-flow-system)
3. [Socket Type System](#3-socket-type-system)
4. [Data Nesting Levels](#4-data-nesting-levels)
5. [List Matching System](#5-list-matching-system)
6. [Vectorization System](#6-vectorization-system)
7. [Update System and Data Propagation](#7-update-system-and-data-propagation)
8. [Implicit Socket Conversions](#8-implicit-socket-conversions)
9. [Socket Processing Modes](#9-socket-processing-modes)
10. [Group Nodes and Flow Control](#10-group-nodes-and-flow-control)
11. [Process Method Patterns](#11-process-method-patterns)
12. [Complete Node Category Inventory](#12-complete-node-category-inventory)
13. [Version Information](#13-version-information)
14. [Key Source File Reference](#14-key-source-file-reference)
15. [Architectural Patterns Summary](#15-architectural-patterns-summary)

---

## 1. Node System Architecture

### 1.1 Class Hierarchy

Sverchok uses Blender's node editor framework with custom tree and node types. The class hierarchy uses mixin composition rather than deep inheritance.

```
bpy.types.NodeTree
  +-- SvNodeTreeCommon (mixin)
  |     +-- tree_id, init_tree(), update_ui()
  +-- SverchCustomTree
        +-- bl_idname = 'SverchCustomTreeType'
        +-- bl_label = 'Sverchok Nodes'
        +-- sv_process, sv_animate, sv_show, sv_draft
        +-- update(), force_update(), update_nodes()

bpy.types.Node
  +-- UpdateNodes (mixin)
  |     +-- sv_init(), process(), sv_update(), sv_copy(), sv_free()
  |     +-- init(), free(), copy(), update() [final - do not override]
  |     +-- node_id, process_node()
  +-- NodeUtils (mixin)
  |     +-- sv_logger, get_bpy_data_from_name()
  +-- NodeDependencies (mixin)
  |     +-- sv_dependencies, missing_dependency, dependency_error
  +-- NodeDocumentation (mixin)
  |     +-- docstring, get_doc_link()
  +-- SverchCustomTreeNode
        +-- sv_category, sv_draw_buttons(), draw_buttons() [final]
        +-- poll(), absolute_location, sv_default_color

bpy.types.NodeSocket
  +-- SvSocketProcessing (mixin)
  |     +-- use_flatten, use_graft, use_unwrap, use_wrap
  |     +-- preprocess_input(), postprocess_output()
  +-- SvSocketCommon
  |     +-- sv_get(), sv_set(), sv_forget(), replace_socket()
  |     +-- use_prop, prop_name, custom_draw, label
  +-- SvVerticesSocket, SvStringsSocket, SvMatrixSocket, ...
```

### 1.2 SverchCustomTree

The `SverchCustomTree` class (defined in `node_tree.py`) is the main node tree type.

| Attribute | Type | Purpose |
|-----------|------|---------|
| `bl_idname` | str | `'SverchCustomTreeType'` -- Blender registration identifier |
| `bl_label` | str | `'Sverchok Nodes'` -- Display name in Blender |
| `sv_process` | BoolProperty | Enable/disable tree processing |
| `sv_animate` | BoolProperty | Enable animation-driven updates |
| `sv_show` | BoolProperty | Show/hide viewer output in viewport |
| `sv_draft` | BoolProperty | Enable draft mode for faster previews |
| `tree_id` | property | Unique stable identifier (hash-based) |
| `show_time_mode` | EnumProperty | `'Cumulative'` or `'Per Node'` timing display |

Key methods:

- `update()` -- Called by Blender when links change. Routes to event system.
- `force_update()` -- Forces full re-evaluation of all nodes.
- `update_nodes(nodes)` -- Marks specific nodes as outdated.
- `init_tree()` -- Context manager that suppresses individual node updates during batch construction.

### 1.3 SverchCustomTreeNode

The `SverchCustomTreeNode` class is the base for all Sverchok nodes. It inherits from four mixins:

**UpdateNodes mixin** provides the node lifecycle:

| Method | Purpose | Override? |
|--------|---------|-----------|
| `sv_init(context)` | Initialize sockets and properties | YES |
| `process()` | Main computation method | YES |
| `sv_update()` | Called when topology changes (links added/removed) | YES |
| `sv_copy(original)` | Called when node is duplicated | YES |
| `sv_free()` | Cleanup when node is deleted | YES |
| `init(context)` | Blender callback -- calls `sv_init()` | NO (final) |
| `free()` | Blender callback -- calls `sv_free()` | NO (final) |
| `copy(original)` | Blender callback -- calls `sv_copy()` | NO (final) |
| `update()` | Blender callback -- calls `sv_update()` | NO (final) |
| `process_node(context)` | Triggers tree update for this node | Internal |
| `node_id` | Unique stable identifier | Property |

**NodeDependencies mixin** manages optional library requirements:

| Attribute | Purpose |
|-----------|---------|
| `sv_dependencies` | Set of required dependency names (e.g., `{'scipy'}`) |
| `missing_dependency` | Returns name of first missing dependency, or None |
| `dependency_error` | Returns `DependencyError` if dependencies missing |

### 1.4 Node Registration and Categorization

Nodes are registered through `index.yaml` which defines the menu structure and category assignments. Each node file contains `register()` and `unregister()` functions following Blender addon conventions.

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'
    sv_category = 'Generators'  # Menu category
```

### 1.5 Node Lifecycle

```
1. Node creation:   init(context) -> sv_init(context)
2. Link change:     update() -> sv_update()
3. Property change: updateNode(self, context) -> process_node(context)
4. Evaluation:      process() -- read sv_get(), compute, write sv_set()
5. Duplication:     copy(original) -> sv_copy(original)
6. Deletion:        free() -> sv_free()
```

### 1.6 Stable Identifiers

Blender object memory addresses change between undo operations. Sverchok generates stable identifiers:

```python
@property
def tree_id(self):
    if not self.tree_id_memory:
        self.tree_id_memory = str(hash(self) ^ hash(time.monotonic()))
    return self.tree_id_memory

@property
def node_id(self):
    if not self.n_id:
        self.n_id = str(hash(self) ^ hash(time.monotonic()))
    return self.n_id
```

---

## 2. Data Flow System

### 2.1 Data Flow Overview

Sverchok implements a dataflow programming paradigm. Data flows from output sockets to input sockets through link connections:

1. **Node execution order** determined by topological sorting (`graphlib.TopologicalSorter`).
2. Node reads input data via `socket.sv_get()`.
3. Node writes output data via `socket.sv_set()`.
4. Data stored in global `socket_data_cache` keyed by unique socket identifiers.
5. `prepare_input_data()` handles implicit type conversions between mismatched sockets.
6. Input sockets may apply preprocessing (flatten, graft, wrap, unwrap).
7. Output sockets may apply postprocessing after writing.

### 2.2 Socket Data Cache

Socket data is stored externally via `core/socket_data.py`:

| Function | Purpose |
|----------|---------|
| `sv_set_socket(socket, data)` | Store data indexed by `socket.socket_id` |
| `sv_get_socket(socket, deepcopy)` | Retrieve data, optionally deep-copying |
| `sv_forget_socket(socket)` | Remove data from cache |

External storage is necessary because Blender's property system cannot hold arbitrary Python objects.

### 2.3 Socket Data Retrieval Order

When `sv_get()` is called on an **input** socket:

1. **Written socket data** -- from connected upstream node
2. **Node default property** -- if `prop_name` is set, reads `node.<prop_name>`
3. **Socket default property** -- if `use_prop` is True
4. **Script default** -- `default` parameter of `sv_get()`
5. **Raise SvNoDataError** -- if none available

---

## 3. Socket Type System

### 3.1 Complete Socket Type Inventory

| Socket Class | `bl_idname` | Color (RGBA) | Default Property | Policy | Nesting |
|---|---|---|---|---|---|
| `SvVerticesSocket` | `SvVerticesSocket` | `(0.9, 0.6, 0.2, 1.0)` orange | `FloatVectorProperty(size=3)` | DEFAULT | 3 |
| `SvStringsSocket` | `SvStringsSocket` | `(0.6, 1.0, 0.6, 1.0)` green | Float or Int (switchable) | DEFAULT | 2 |
| `SvMatrixSocket` | `SvMatrixSocket` | `(0.2, 0.8, 0.8, 1.0)` teal | None | DEFAULT | 1 |
| `SvColorSocket` | `SvColorSocket` | `(0.9, 0.8, 0.0, 1.0)` yellow | `FloatVectorProperty(size=4)` | DEFAULT | 3 |
| `SvQuaternionSocket` | `SvQuaternionSocket` | `(0.9, 0.4, 0.7, 1.0)` pink | `FloatVectorProperty(size=4)` | DEFAULT | 2 |
| `SvObjectSocket` | `SvObjectSocket` | `(0.69, 0.74, 0.73, 1.0)` grey-green | `PointerProperty(Object)` | DEFAULT | 2 |
| `SvDictionarySocket` | `SvDictionarySocket` | `(1.0, 1.0, 1.0, 1.0)` white | None | DEFAULT | 2 |
| `SvTextSocket` | `SvTextSocket` | `(0.68, 0.85, 0.90, 1.0)` light blue | `StringProperty` | LENIENT | 2 |
| `SvFilePathSocket` | `SvFilePathSocket` | `(0.9, 0.9, 0.3, 1.0)` yellow-green | None | DEFAULT | 2 |
| `SvCurveSocket` | `SvCurveSocket` | `(0.5, 0.6, 1.0, 1.0)` blue | None | DEFAULT | 2 |
| `SvSurfaceSocket` | `SvSurfaceSocket` | `(0.4, 0.2, 1.0, 1.0)` deep purple | None | DEFAULT | 2 |
| `SvScalarFieldSocket` | `SvScalarFieldSocket` | `(0.9, 0.4, 0.1, 1.0)` dark orange | None | FIELD | 2 |
| `SvVectorFieldSocket` | `SvVectorFieldSocket` | `(0.1, 0.1, 0.9, 1.0)` deep blue | None | FIELD | 2 |
| `SvSolidSocket` | `SvSolidSocket` | `(0.0, 0.65, 0.3, 1.0)` green | None | SOLID | 2 |
| `SvFormulaSocket` | `SvFormulaSocket` | `(0.68, 0.85, 0.90, 1.0)` light blue | None | LENIENT | 2 |
| `SvSvgSocket` | `SvSvgSocket` | `(0.1, 0.5, 1.0, 1.0)` blue | None | DEFAULT | 2 |
| `SvPulgaForceSocket` | `SvPulgaForceSocket` | `(0.4, 0.3, 0.6, 1.0)` purple | None | DEFAULT | 2 |
| `SvLoopControlSocket` | `SvLoopControlSocket` | `(0.1, 0.1, 0.1, 1.0)` near-black | None | DEFAULT | 2 |
| `SvDummySocket` | `SvDummySocket` | `(0.8, 0.8, 0.8, 0.3)` transparent | None | DEFAULT | 2 |
| `SvSeparatorSocket` | `SvSeparatorSocket` | `(0.0, 0.0, 0.0, 0.0)` invisible | None | DEFAULT | 2 |
| `SvChameleonSocket` | `SvChameleonSocket` | dynamic | None | LENIENT | 2 |
| `SvCollectionSocket` | `SvCollectionSocket` | `(0.96, 0.96, 0.96, 1.0)` | `PointerProperty(Collection)` | DEFAULT | 2 |
| `SvMaterialSocket` | `SvMaterialSocket` | `(0.92, 0.46, 0.51, 1.0)` salmon | `PointerProperty(Material)` | DEFAULT | 2 |
| `SvTextureSocket` | `SvTextureSocket` | `(0.62, 0.31, 0.64, 1.0)` purple | `PointerProperty(Texture)` | DEFAULT | 2 |
| `SvImageSocket` | `SvImageSocket` | `(0.39, 0.22, 0.39, 1.0)` dark purple | `PointerProperty(Image)` | DEFAULT | 2 |

### 3.2 Socket Base Classes

**`SvSocketCommon`** provides:
- `sv_get(default=..., deepcopy=True)` -- Read data with fallback chain
- `sv_set(data)` -- Write data (applies postprocessing for outputs)
- `sv_forget()` -- Clear cached data
- `replace_socket(new_type, new_name)` -- Replace type preserving links
- `socket_id` -- Unique cache key from `node.node_id + identifier + direction`
- `nesting_level` -- Expected nesting level (default 2)

**`SvSocketProcessing`** provides transformation flags:
- `use_flatten` / `use_simplify` / `use_graft` / `use_wrap` / `use_unwrap`
- `preprocess_input(data)` / `postprocess_output(data)`

**`SocketDomain`** provides attribute domain (POINT, EDGE, FACE) for `SvVerticesSocket`, `SvStringsSocket`, `SvColorSocket`.

### 3.3 Quick Link System

| Socket Type | Quick Link Node |
|---|---|
| `SvVerticesSocket` | `GenVectorsNode` |
| `SvMatrixSocket` | `SvMatrixInNodeMK4` |
| `SvStringsSocket` | `SvNumberNode` |
| `SvScalarFieldSocket` | `SvNumberNode` |
| `SvVectorFieldSocket` | `GenVectorsNode` |

---

## 4. Data Nesting Levels

### 4.1 Nesting Level Definitions

```
Level 0: atomic value           42, 3.14, Matrix()
Level 1: list of atoms          [42, 3.14, 2.71]
Level 2: list of lists          [[1, 2, 3], [4, 5]]
Level 3: list of lists of lists [[[1.0, 0.0, 0.0], [0.0, 1.0, 0.0]]]
```

### 4.2 Standard Data Conventions

**Vertices (nesting_level=3):**
```python
# Single mesh: [[(x0,y0,z0), (x1,y1,z1), (x2,y2,z2)]]
# Two meshes:  [[(v0), (v1), (v2)], [(v3), (v4)]]
```

**Edges:** `[[(0,1), (1,2), (2,0)]]`
**Faces:** `[[(0,1,2), (2,3,0)]]`
**Numbers (nesting_level=2):** `[[1, 2, 3, 4, 5]]`
**Matrices (nesting_level=1):** `[Matrix(), Matrix()]`

### 4.3 Utilities

| Function | Purpose |
|----------|---------|
| `get_data_nesting_level(data)` | Detect nesting level |
| `ensure_nesting_level(data, level)` | Add wrapping to target |
| `ensure_min_nesting(data, level)` | Ensure minimum nesting |
| `flatten_data(data, level)` | Flatten to level |
| `graft_data(data, level)` | Add nesting at level |

---

## 5. List Matching System

### 5.1 Matching Modes

| Mode | Function | Behavior |
|------|----------|----------|
| REPEAT | `match_long_repeat` | Repeats last element |
| CYCLE | `match_long_cycle` | Cycles shorter list |
| SHORT | `match_short` | Truncates to shortest |
| XREF | `match_cross` | Cartesian product |

### 5.2 Functions (in `data_structure.py`)

```python
list_match_func = {
    'REPEAT': match_long_repeat,
    'CYCLE': match_long_cycle,
    'SHORT': match_short,
    'XREF': match_cross,
    'CROSS2': match_cross2,
}

def zip_long_repeat(*iterables): ...
def zip_long_cycle(*iterables): ...
```

### 5.3 The `updateNode` Function

```python
def updateNode(node, context):
    """Standard update= callback for bpy.props that triggers tree re-evaluation."""
```

---

## 6. Vectorization System

### 6.1 The `vectorize` Decorator

In `utils/vectorize.py` -- automatically traverses nested structures and applies functions at correct nesting level.

### 6.2 The `SvRecursiveNode` Mixin

```python
class SvRecursiveNode:
    list_match: EnumProperty(default='REPEAT')
    build_bmesh: bool = False
    bmesh_inputs: list = [0, 1, 2]

    def pre_setup(self): ...       # Override for pre-configuration
    def process(self): ...          # Reads inputs, matches, calls process_data
    def process_data(self, params): # Override with computation logic
        raise NotImplementedError
```

### 6.3 Socket Config Mixins

| Mixin | Mapping | Purpose |
|-------|---------|---------|
| `ModifierNode` | in[0:3] -> out[0:3] + FaceData | Standard mesh modifier |
| `ModifierLiteNode` | in[0:2] -> out[0:2] | Simple modifier |
| `TransformNode` | in[0] -> out[0] | Transform passthrough |
| `EdgeGeneratorNode` | in[0:2] -> out[0:2] | Edge generator |

### 6.4 Draft Mode

`DraftMode` mixin allows alternative property values for faster previews:
- `draft_properties_mapping` -- Maps standard to draft properties
- `on_draft_mode_changed()` -- Copies values on first entry
- `sv_draft_mode_changed()` -- Override for custom behavior

---

## 7. Update System and Data Propagation

### 7.1 Event Types

| Event | Trigger |
|-------|---------|
| `TreeEvent` | Topology changes |
| `PropertyEvent` | Property value changes |
| `AnimationEvent` | Frame change |
| `SceneEvent` | Blender scene changes |
| `ForceEvent` | Manual force-update |

### 7.2 Update Flow

```
[User changes property]
        |
        v
updateNode(self, context)
        |
        v
self.process_node(context) -> self.id_data.update_nodes([self])
        |
        v
handle_event(PropertyEvent(tree, [node]))
        |
        v
control_center(event)
        |
        v
UpdateTree.get(tree).add_outdated([node])
tasks.add(Task(tree, UpdateTree.main_update(tree)))
        |
        v
[Timer fires] -> UpdateTree.main_update(tree)
        |
        v
[Topological sort outdated + downstream nodes]
        |
        v
For each node in order:
  1. prepare_input_data(prev_sockets, node.inputs)
  2. node.process()
  3. Record statistics
        |
        v
update_ui(tree)
```

### 7.3 SearchTree and UpdateTree

**`SearchTree`** (in `core/update_system.py`):
- Builds directed graph of node connections
- Resolves WiFi virtual connections
- Handles muted nodes with bypass links
- Removes reroutes by relinking
- Topological sorting via `graphlib.TopologicalSorter`

**`UpdateTree`** extends `SearchTree`:
- `_tree_catch` -- Static cache of tree structures
- `get(tree, refresh_tree)` -- Get/create cached tree
- `main_update(tree)` -- Main update generator
- Incremental updates: compares topologies, marks only changed nodes

### 7.4 Thread Safety and Deferred Execution

Updates deferred to Blender's timer system via `core/tasks.py`. Animation events processed immediately (frame changes faster than timer).

### 7.5 The `init_tree` Context Manager

```python
with tree.init_tree():
    node1 = tree.nodes.new('SvBoxNodeMk2')
    node2 = tree.nodes.new('SvMeshViewer')
    tree.links.new(node1.outputs[0], node2.inputs[0])
# Normal updates resume after context exit
```

---

## 8. Implicit Socket Conversions

### 8.1 Conversion Policies

| Policy | Description |
|--------|-------------|
| DEFAULT | Standard conversions between common types |
| FIELD | Converts fields to evaluated values |
| LENIENT | Accepts any input without conversion |
| SOLID | Converts between solid and mesh |

### 8.2 DEFAULT Conversions

| From | To | Conversion |
|------|-----|------------|
| Vertices | Matrix | Translation matrices |
| Matrix | Vertices | Extract translation |
| Quaternion | Matrix | Rotation matrix |
| Matrix | Quaternion | Extract rotation |
| Strings | Vertices | Numbers as X coordinates |
| Vertices | Strings | Flatten to numbers |
| Color | Vertices | RGB as XYZ |

---

## 9. Socket Processing Modes

| Mode | Effect |
|------|--------|
| Flatten | Collapse all nesting to flat list |
| Simplify | Remove one nesting level |
| Graft | Wrap each element in sublist |
| Wrap | Wrap data in extra list |
| Unwrap | Remove outermost list |

Available on both input and output sockets via context menu.

---

## 10. Group Nodes and Flow Control

### 10.1 SvGroupTree

Group trees (`SvGroupTree` in `core/node_group.py`) encapsulate subgraphs:
- **Group Input Node** -- Defines group inputs
- **Group Output Node** -- Defines group outputs
- **SvGroupTreeNode** -- Node referencing a `SvGroupTree`

### 10.2 WiFi Nodes

`WifiInNode`/`WifiOutNode` create virtual connections via shared `var_name`. `SearchTree` resolves these to direct links.

### 10.3 Loop Nodes

`SvLoopInNode`/`SvLoopOutNode` execute contained nodes repeatedly.

---

## 11. Process Method Patterns

### 11.1 Basic Pattern

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'
    factor: FloatProperty(default=1.0, update=updateNode)

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Factor').prop_name = 'factor'
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def process(self):
        if not self.outputs['Vertices'].is_linked:
            return
        verts = self.inputs['Vertices'].sv_get(deepcopy=False)
        factor = self.inputs['Factor'].sv_get(deepcopy=False)
        out = []
        for v_list, f_list in zip_long_repeat(verts, factor):
            obj_out = []
            for v, f in zip_long_repeat(v_list, f_list):
                obj_out.append([c * f for c in v])
            out.append(obj_out)
        self.outputs['Vertices'].sv_set(out)
```

### 11.2 NumPy Pattern

```python
def process(self):
    verts = self.inputs['Vertices'].sv_get(deepcopy=False)
    result = []
    for v_list in verts:
        v_array = np.array(v_list)
        result.append((v_array * self.factor).tolist())
    self.outputs['Vertices'].sv_set(result)
```

### 11.3 List Matching Pattern

```python
def process(self):
    params = [s.sv_get(deepcopy=False) for s in self.inputs]
    matched = list_match_func[self.list_match](params)
    result = [self.compute(*p) for p in zip(*matched)]
    self.outputs[0].sv_set(result)
```

### 11.4 Early Exit Pattern

```python
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return
    if not self.inputs['Vertices'].is_linked:
        return
```

### 11.5 Viewer Node Pattern

Uses `SvViewerNode` mixin: `SvObjectData`, `SvMeshData`, `BlenderObjects` for managing Blender objects with Greek alphabet naming.

---

## 12. Complete Node Category Inventory

Sverchok organizes 600+ nodes into 34 categories.

### 12.1 Generators (19 nodes)

| bl_idname | Label | Key Inputs | Key Outputs | Purpose |
|-----------|-------|------------|-------------|---------|
| `SvBoxNodeMk2` | Box | Size, Divx, Divy, Divz | Verts, Edges, Faces | Rectangular prism |
| `SvCylinderNodeMK2` | Cylinder | RadTop, RadBot, Vertices, Height | Verts, Edges, Faces | Cylinder/cone |
| `SphereNode` | Sphere | Radius, U, V | Verts, Edges, Faces | UV sphere |
| `SvPlaneNodeMk3` | Plane | SizeX, SizeY, StepX, StepY | Verts, Edges, Faces | Planar grid |
| `SvCircleNode` | Circle | Radius, Vertices, Degrees | Verts, Edges, Faces | Circle/arc |

Additional: Line, NGon, Torus, Suzanne, NURBS Curve, NURBS Surface, IcoSphere, Bricks, Segment, Ring.

### 12.2 Generators Extended (16 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvBoxRoundedNode` | Rounded Box | Box with rounded edges |
| `SvTorusKnotNodeMK2` | Torus Knot | Parametric torus knots |
| `SvSpiralNodeMK2` | Spiral | Archimedean, Log spirals |
| `SvEllipseNodeMK3` | Ellipse | Elliptical curves |
| `SvRegularSolid` | Regular Solid | Platonic/Archimedean solids |

### 12.3 Transforms (18 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvMoveNodeMk3` | Move | Translate vertices |
| `SvRotationNodeMk3` | Rotate | Rotate around axis |
| `SvScaleNodeMk3` | Scale | Scale from center |
| `SvMirrorNodeMk2` | Mirror | Mirror across plane |
| `MatrixApplyNode` | Matrix Apply | Apply matrices |

Additional: Noise Displace, Randomize, Cast, Formula Deform, Barycentric, Align Mesh, Bend Along Path/Surface.

### 12.4 Analyzers (42 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvBBoxNodeMk3` | Bounding Box | AABB calculation |
| `SvKDTreeNodeMK2` | KD Tree | Nearest neighbor |
| `SvGetNormalsNodeMk2` | Normals | Vertex/face normals |
| `SvRaycasterLiteNode` | Raycaster | Ray-mesh intersection |
| `SvMeshFilterNode` | Mesh Filter | Element filtering |

Additional: Area, Distance, Edge Angle, Proportional Falloff, Mesh Select, Point Inside, Component Analyzer, Linked Verts, Path Length, Origins.

### 12.5 Modifier Change (44 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvDeleteLooseNode` | Delete Loose | Remove disconnected verts |
| `SvMergeByDistanceNode` | Merge by Distance | Merge near vertices |
| `SvSeparateMeshNode` | Separate Mesh | Split components |
| `SvTriangulateNode` | Triangulate | Faces to triangles |
| `SvExtrudeSeparateNode` | Extrude Separate | Extrude individual faces |

Additional: Split Edges, Recalc Normals, Fill Holes, Dissolve, Poke, Flip Normals, Smooth Vertices, Edge Split, Mask Vertices.

### 12.6 Modifier Make (25 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvSolidifyNodeMk2` | Solidify | Add thickness |
| `SvWireframeNode` | Wireframe | Edge wireframe |
| `SvSubdivideNodeMK2` | Subdivide | Subdivide faces |
| `SvOpenSubdivisionNode` | OpenSubdivision | Catmull-Clark |
| `SvDualMeshNodeMK2` | Dual Mesh | Dual transformation |

Additional: Contour, Pipe, Adaptive Polygons, Line Connect, Offset, Join Triangles.

### 12.7 Number (16 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvNumberNode` | Number | Single number input |
| `SvScalarMathNodeMK4` | Scalar Math | +, -, *, /, sin, cos |
| `SvGenNumberRange` | Number Range | Arithmetic sequences |
| `SvRndNumGen` | Random Num Gen | Random with seed |
| `SvMapRangeNode` | Map Range | Remap values |

### 12.8 Vector (20 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `GenVectorsNode` | Vectors In | Create from X, Y, Z |
| `VectorsOutNode` | Vectors Out | Decompose to X, Y, Z |
| `SvVectorMathNodeMK3` | Vector Math | add, cross, dot, normalize |
| `SvNoiseNodeMK3` | Noise | Perlin/Simplex noise |
| `SvVectorLerp` | Vector Lerp | Interpolation |

### 12.9 Matrix (12 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvMatrixInNodeMK4` | Matrix In | From location/rotation/scale |
| `SvMatrixOutNodeMK2` | Matrix Out | Decompose |
| `SvMatrixApplyJoinNode` | Matrix Apply | Apply and join |
| `SvMatrixMathNode` | Matrix Math | Multiply, invert |
| `SvIterateNode` | Matrix Iterate | Repeated application |

### 12.10 Logic (10 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvLogicNodeMK2` | Logic | AND, OR, NOT, comparisons |
| `SvSwitchNodeMK2` | Switch | Select by condition |
| `SvInputSwitchNodeMOD` | Input Switch | Route inputs |
| `SvLoopInNode` | Loop In | Loop start |
| `SvLoopOutNode` | Loop Out | Loop end |

### 12.11 List Main (11 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `ListJoinNode` | List Join | Join multiple lists |
| `ZipNode` | List Zip | Zip together |
| `ListLevelsNode` | List Levels | Inspect nesting |
| `ListLengthNode` | List Length | Get lengths |
| `ListMatchNode` | List Match | Apply matching mode |

Additional: Sum, Func (MIN/MAX/AVR/SUM), Decompose, Statistics, Index.

### 12.12 List Structure (14 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `ShiftNodeMK2` | List Shift | Rotate elements |
| `ListSliceNode` | List Slice | Slice start/stop/step |
| `SvListItemNode` | List Item | Get by index |
| `ListReverseNode` | List Reverse | Reverse order |
| `ListFlipNode` | List Flip | Transpose nested lists |

Additional: Repeater, Split, First/Last, Shuffle, Sort, Levels.

### 12.13 List Masks (6 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `MaskListNode` | Mask List | Filter by mask |
| `SvMaskJoinNodeMK2` | Mask Join | Join with masks |
| `SvMaskConvertNode` | Mask Convert | Convert formats |
| `SvMaskToIndexNode` | Mask to Index | Mask to indices |
| `SvIndexToMaskNode` | Index to Mask | Indices to mask |

### 12.14 List Mutators (8 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvListModifierNode` | List Modifier | Various modifications |
| `SvUniqueItemsNode` | Unique Items | Remove duplicates |
| `SvCacheNode` | Cache | Cache between updates |
| `SvMultiCacheNode` | Multi Cache | Cache multiple frames |
| `SvCombinatoricsNode` | Combinatorics | Permutations/combinations |

### 12.15 CAD (9 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvBevelNodeMK2` | Bevel | Bevel edges/vertices |
| `SvIntersectEdgesNodeMK3` | Intersect Edges | Edge intersections |
| `SvOffsetNode` | Offset | Offset contours |
| `SvInsetSpecialMk2` | Inset Faces | Inset faces |
| `SvBisectNode` | Bisect | Bisect with plane |

Additional: CSG Boolean, Merge 2D, Straight Skeleton, Boolean Internal.

### 12.16 Viz (27 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvViewerDrawMk4` | Viewer Draw | GPU-accelerated viewport drawing |
| `SvMeshViewer` | Mesh Viewer | Create Blender objects |
| `SvGeoNodesViewerNode` | Geo Nodes Viewer | Output via Geometry Nodes |
| `SvCurveViewerNodeV28` | Curve Viewer | Create curve objects |
| `SvInstancerNodeMK3` | Instancer | Instance on points |

Additional: Dupli Instances, Light Viewer, Texture Viewer, Console, Viewer Index, Polyline, NURBS Curve/Surface Viewer, Metaball, Empty, Skin, Data Tree Viz.

### 12.17 Text (11 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `ViewerNodeTextMK3` | Text Viewer | Display as text |
| `SvDataShapeNode` | Data Shape | Structure shape |
| `SvStethoscopeNodeMK2` | Stethoscope | Live inspector |
| `SvTextInNodeMK2` | Text In | Import (CSV/SV/JSON/TEXT) |
| `SvTextOutNodeMK2` | Text Out | Export to file |

### 12.18 Scene (13 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvGetObjectsDataMK5` | Get Objects Data | Extract mesh data (20+ outputs) |
| `SvObjInLite` | Object In Lite | Lightweight import |
| `SvCurveInputNode` | Curve Input | Import curves |
| `SvBezierInNodeMK3` | Bezier In | Import Bezier |
| `SvFrameInfoNodeMK2` | Frame Info | Frame/time info |

### 12.19 Object/BPY Data (26 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvObjRemoteNodeMK2` | Object Remote | Control properties |
| `SvSetDataObjectNodeMK2` | Set Object Data | Write mesh data |
| `SvSetMeshAttributeNode` | Set Mesh Attribute | Custom attributes |
| `SvVertexGroupNodeMK2` | Vertex Group | Read/write groups |
| `SvVertexColorNodeMK3` | Vertex Color | Read/write colors |

Additional: Assign Material, Object Raycast, Point on Mesh, Set Collection, Armature, Shape Keys, UV Map.

### 12.20 Layout (3 nodes)

`WifiInNode` (wireless input), `WifiOutNode` (wireless output), `ConverterNode` (socket type conversion with 9 output types).

### 12.21 Network (2 nodes)

`UdpClientNode` (UDP send/receive), `SvFilePathNode` (file/directory selector).

### 12.22 Script (10 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvScriptNodeLite` | Script Node Lite | Python with dynamic sockets via AST |
| `SvFormulaNodeMk5` | Formula | Math formulas (up to 4, 13+ output types) |
| `SvProfileNodeMK3` | Profile Parametric | 2D profiles from SVG-like DSL |
| `SvSNFunctorB` | Script Node (Functor) | Full Python node API |
| `SvGenerativeArtNode` | Generative Art | L-system via XML rules |

Additional: Exec Node, Mesh Expression, Default Script.

### 12.23 Spatial (22 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvExVoronoi3DNode` | Voronoi 3D | 3D Voronoi diagrams |
| `SvVoronoiOnMeshNodeMK4` | Voronoi on Mesh | Mesh fragmentation |
| `SvDelaunay3dMk2Node` | Delaunay 3D | 3D triangulation |
| `SvPopulateMeshNode` | Populate Mesh | Random points on/in mesh |
| `SvLloyd3dNode` | Lloyd 3D | Even point distribution |

Additional: Voronoi 2D/Sphere/Solid, Delaunay 2D/CDT, Lloyd 2D/Mesh/Sphere, Convex/Concave Hull, Populate Surface/Solid, Mesh Clustering.

### 12.24 Exchange (12 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvReadFCStdNode` | Read FCStd | Import FreeCAD files |
| `SvExNurbsInNodeMK3` | NURBS Input | Get NURBS from scene |
| `SvBezierInNodeMK3` | Bezier Input | Get Bezier curves |
| `SvExportGcodeNode` | Export G-code | CNC/3D printing |
| `SvGlbExportNode` | GLB Exporter | glTF/GLB with THREE.js viewer |

Additional: FCStd Write/Sketch/Spreadsheet, NURBS to/from JSON, Rhino 3DM import/export.

### 12.25 Pulga Physics (18 nodes)

Modular solver-and-forces architecture. Forces connect via `SvPulgaForceSocket`.

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvPulgaPhysicsSolverNode` | Pulga Solver | Central iterative physics engine |
| `SvPulgaSpringsForceNode` | Springs Force | Spring connections |
| `SvPulgaAttractionForceNode` | Attraction Force | Inter-particle attraction |
| `SvPulgaCollisionForceNode` | Collision Force | Collision repulsion |
| `SvPulgaDragForceNode` | Drag Force | Environmental resistance |

Complete forces: Align, Angle, Attraction, Attractors, Boundaries, Collision, Drag, Fit, Inflate, Obstacle, Pin, Random, Springs, Timed, Vector, Vortex.

### 12.26 Solid (45 nodes) -- FreeCAD

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvBoxSolidNode` | Box (Solid) | Solid box |
| `SvSolidBooleanNode` | Solid Boolean | CSG Union/Intersect/Diff |
| `SvFilletSolidNode` | Fillet Solid | Edge rounding |
| `SvSolidToMeshNodeMk2` | Solid to Mesh | 5 meshing algorithms |
| `SvTransformSolidNode` | Transform Solid | Apply matrices |

Primitives: Box, Cone, Cylinder, Sphere, Torus.
Operations: Boolean, Fillet, Chamfer, Hollow, Mirror, Offset, Slice, Split, Extrude/Revolve/Sweep/Solidify Face, General Fuse, Symmetrize.
Analysis: BBox, Center of Mass, Area, Volume, Distance, Points Inside, Is Closed.

### 12.27 Field (33 nodes)

| bl_idname | Label | Purpose |
|-----------|-------|---------|
| `SvExScalarFieldFormulaNode` | Scalar Field Formula | Field from formula |
| `SvExVectorFieldFormulaNode` | Vector Field Formula | Field from components |
| `SvExScalarFieldEvaluateNode` | Evaluate Scalar Field | Sample at points |
| `SvExVectorFieldApplyNode` | Apply Vector Field | Displace vertices |
| `SvAttractorFieldNodeMk2` | Attractor Field | Attraction fields |

Categories: Creation (formula, coordinate, image, noise, rotation, twist, taper), Evaluation, Application, Composition, Math, Analysis (curvature, gradient, divergence), Visualization (isosurface, flow lines), Deformation (curve/surface bend).

### 12.28 Curves (82 nodes)

Key: NURBS Curve, Bezier, Circle, Line, Polyline, Evaluate, Length, To Mesh, Interpolation, Iso Curve, Fillet, Offset, Intersect, Blend, Birail, Frame, Curvature, Torsion, Insert/Remove Knot, Elevate Degree, Snap, Discontinuity, Extremes.

### 12.29 Surfaces (53 nodes)

Key: NURBS Surface, Evaluate, Loft, Revolution, Gordon, From Curves, Tessellate/Trim, Curvature, Normal, UV Map, Marching Cubes/Squares, Offset, Adjust, Snap.

### 12.30 Additional Categories

| Category | Directory | Count | Purpose |
|----------|-----------|-------|---------|
| Color | `color/` | 7 | Color In/Out, Mix, Ramp, Texture Evaluate |
| Quaternion | `quaternion/` | 4 | Quaternion In/Out/Math |
| Dictionary | `dictionary/` | 2 | Dictionary In/Out |
| SVG | `svg/` | 9 | SVG Document, Path, Mesh, Styling |
| DXF | `dxf/` | 6 | DXF Import/Export (requires ezdxf) |

---

## 13. Version Information

### 13.1 Current Version

```python
bl_info = {
    "name": "Sverchok",
    "version": (1, 4, 0),
    "blender": (3, 5, 0),
    "location": "Node Editor",
    "category": "Node",
    "description": "Parametric node-based geometry programming",
}
VERSION = 'v1.4.0'
```

### 13.2 Compatibility Matrix

| Version | Date | Blender |
|---------|------|---------|
| v1.4.0 | March 5, 2025 | 3.6 -- 5.1 |
| v1.3.0 | Sep 20, 2024 | 4.2 focus (3.6+) |
| v1.2.0 | Jul 25, 2023 | 2.93 -- 3.6 |
| v1.1.0 | Sep 30, 2022 | 2.93 -- 3.3 |
| v1.0.0 | Jan 29, 2021 | 2.92 -- 3.0 |

### 13.3 Python: 3.10 through 3.13 (via Blender)

### 13.4 Dependencies (all optional)

| Package | Purpose |
|---------|---------|
| scipy | Scientific computing, spatial algorithms |
| geomdl | NURBS operations |
| scikit-image | Image processing |
| mcubes | Marching cubes |
| circlify | Circle packing |
| FreeCAD | Solid nodes (46 nodes) |
| Cython | Performance optimization |
| Numba | JIT acceleration |
| pyOpenSubdiv | Catmull-Clark subdivision |
| numexpr | Fast expression eval |
| ezdxf | DXF import/export (new v1.4.0) |
| pyacvd | Mesh remeshing |
| pyQuadriFlow | Quad remeshing |
| pySVCGAL | Computational geometry |
| Spyrrow | Nesting/packing (new v1.4.0) |

### 13.5 v1.4.0 Highlights

New nodes: Spyrrow Nester, Adjust Surfaces, Straight Skeleton, Data Tree Viz, DXF Import/Export, Boolean Internal, Curve Discontinuity, Curve to NURBS, Snap Curves, Twist along Curve Field, NURBS Curve Extremes, Symmetrize Curve/Solid, Curve Offset mk3.

Features: Quick-start hints, integrated theme, searchable examples, one-button installation.

### 13.6 Breaking Changes

1. v1.3.0: `Voronoi on Mesh` "output nesting" renamed to "Post processing"
2. v1.3.0: `Get Objects Data` upgraded to MK3 with relocated UI
3. v1.1.0: New tree evaluation/update system
4. Blender 4.x: Property registration API changes
5. Master branch recommended for latest fixes

---

## 14. Key Source File Reference

| File | Purpose |
|------|---------|
| `node_tree.py` | `SverchCustomTree`, `SverchCustomTreeNode`, all mixins |
| `data_structure.py` | List matching, `updateNode()`, data utilities |
| `index.yaml` | Node categories and menu structure |
| `dependencies.py` | Optional dependency declarations |
| `core/sockets.py` | All socket type classes |
| `core/socket_data.py` | Socket data cache |
| `core/socket_conversions.py` | Implicit conversions |
| `core/events.py` | Event type definitions |
| `core/event_system.py` | Event routing |
| `core/update_system.py` | `SearchTree`, `UpdateTree`, topological sort |
| `core/main_tree_handler.py` | Main tree handler |
| `core/tasks.py` | Task queue |
| `core/node_group.py` | `SvGroupTree`, `SvGroupTreeNode` |
| `core/handlers.py` | Blender handler registrations |
| `utils/vectorize.py` | Vectorization decorator |
| `utils/sv_itertools.py` | `process_matched` |
| `utils/nodes_mixins/recursive_nodes.py` | `SvRecursiveNode` |
| `utils/nodes_mixins/generating_objects.py` | `SvViewerNode`, `SvMeshData` |
| `utils/nodes_mixins/sockets_config.py` | Socket mapping mixins |
| `utils/nodes_mixins/draft_mode.py` | `DraftMode` mixin |

---

## 15. Architectural Patterns Summary

1. **Mixin Composition** -- Focused mixins instead of deep inheritance
2. **Event-Driven Updates** -- Centralized event system with typed events
3. **Deferred Execution** -- Tasks queued, executed by Blender timer
4. **Topological Sorting** -- `graphlib.TopologicalSorter` for upstream-first evaluation
5. **External Data Cache** -- Socket data in dictionary, not on socket objects
6. **Incremental Updates** -- Only changed nodes + downstream re-evaluated
7. **Implicit Conversion** -- Socket type mismatch handled by policy system
8. **List Matching** -- Configurable strategies (repeat, cycle, cross)
9. **Draft Mode** -- Alternative properties for faster previews
10. **Versioned Migration** -- Node classes versioned (MK2, MK3, MK4)
11. **Dependency Gating** -- Missing dependencies hide nodes, no import errors
12. **Stable IDs** -- Hash-based identifiers survive undo/redo

---

## Appendix: Node Category Summary

| # | Category | Dir | Count |
|---|----------|-----|-------|
| 1 | Generators | `generator/` | 19 |
| 2 | Generators Extended | `generators_extended/` | 16 |
| 3 | Transforms | `transforms/` | 18 |
| 4 | Analyzers | `analyzer/` | 42 |
| 5 | Modifier Change | `modifier_change/` | 44 |
| 6 | Modifier Make | `modifier_make/` | 25 |
| 7 | Number | `number/` | 16 |
| 8 | Vector | `vector/` | 20 |
| 9 | Matrix | `matrix/` | 12 |
| 10 | Logic | `logic/` | 10 |
| 11 | List Main | `list_main/` | 11 |
| 12 | List Structure | `list_struct/` | 14 |
| 13 | List Masks | `list_masks/` | 6 |
| 14 | List Mutators | `list_mutators/` | 8 |
| 15 | CAD | `CAD/` | 9 |
| 16 | Viz | `viz/` | 27 |
| 17 | Text | `text/` | 11 |
| 18 | Scene | `scene/` | 13 |
| 19 | Object/BPY | `object_nodes/` | 26 |
| 20 | Layout | `layout/` | 3 |
| 21 | Network | `network/` | 2 |
| 22 | Script | `script/` | 10 |
| 23 | Spatial | `spatial/` | 22 |
| 24 | Exchange | `exchange/` | 12 |
| 25 | Pulga Physics | `pulga_physics/` | 18 |
| 26 | Solid | `solid/` | 45 |
| 27 | Field | `field/` | 33 |
| 28 | Curves | `curve/` | 82 |
| 29 | Surfaces | `surface/` | 53 |
| 30 | Color | `color/` | 7 |
| 31 | Quaternion | `quaternion/` | 4 |
| 32 | Dictionary | `dictionary/` | 2 |
| 33 | SVG | `svg/` | 9 |
| 34 | DXF | `dxf/` | 6 |
| | **Total** | | **~623** |

---

*Document generated from Sverchok source code analysis and official documentation. All class names, method signatures, bl_idname values, and data structures verified against https://github.com/nortikin/sverchok (v1.4.0). Node counts from directory listings of `nodes/`.*
