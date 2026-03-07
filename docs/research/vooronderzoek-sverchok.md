# Vooronderzoek: Sverchok — Visual Node Programming for Blender

**Date**: 2026-03-07
**Status**: COMPLETE
**Author**: Research Agents (Phase S1) — sv-research-core, sv-research-python, sv-research-extensions
**Scope**: Sverchok comprehensive analysis for Claude Skill Package
**Versions Covered**: Sverchok v1.4.0, Blender 4.0+/5.x

---

## Table of Contents

1. [Sverchok Overview & Architecture](#1-sverchok-overview--architecture)
2. [Node System & Data Flow](#2-node-system--data-flow)
3. [Socket Type System](#3-socket-type-system)
4. [Data Nesting Levels (CRITICAL)](#4-data-nesting-levels-critical)
5. [List Matching & Vectorization](#5-list-matching--vectorization)
6. [Node Categories (Complete Inventory)](#6-node-categories-complete-inventory)
7. [Python Scripting (SNLite, SN Functor B, Formula)](#7-python-scripting-snlite-sn-functor-b-formula)
8. [Custom Node Development](#8-custom-node-development)
9. [External API Access](#9-external-api-access)
10. [IfcSverchok (IFC/BIM Integration)](#10-ifcsverchok-ifcbim-integration)
11. [Extensions Ecosystem (TopologicSverchok, Sverchok-Extra, etc.)](#11-extensions-ecosystem-topologicsverchok-sverchok-extra-etc)
12. [Common Error Patterns](#12-common-error-patterns)
13. [AI Common Mistakes](#13-ai-common-mistakes)
14. [Real-World Usage Patterns](#14-real-world-usage-patterns)

---

## 1. Sverchok Overview & Architecture

### 1.1 What Is Sverchok

Sverchok is a parametric/algorithmic design add-on for Blender that provides a visual node-based programming environment. It implements a **dataflow programming paradigm** within Blender's node editor. Data flows from output sockets of upstream nodes to input sockets of downstream nodes through noodle (link) connections.

**Source repository**: https://github.com/nortikin/sverchok
**Version analyzed**: v1.4.0+ (master branch, compatible with Blender 4.0+)
**License**: GPL-3.0

Sverchok provides multiple levels of functionality:
- **Visual node programming**: 500+ nodes for geometry generation, transformation, analysis
- **Python scripting**: SNLite, SN Functor B, Formula nodes for custom logic
- **Extension ecosystem**: IfcSverchok, TopologicSverchok, Sverchok-Extra, etc.
- **External API**: Full programmatic access via `bpy.data.node_groups`

### 1.2 Core Architecture

The Sverchok architecture consists of these key components:

#### Node Tree: `SverchCustomTree`

The main node tree class in `node_tree.py`:

```python
class SverchCustomTree(NodeTree, SvNodeTreeCommon):
    bl_idname = 'SverchCustomTreeType'
    bl_label = 'Sverchok Nodes'

    # Key properties:
    sv_process: BoolProperty   # Enable/disable tree processing
    sv_animate: BoolProperty   # Process on frame change
    sv_show: BoolProperty      # Show viewer outputs in viewport
    sv_draft: BoolProperty     # Draft mode (reduced quality/speed)
    sv_scene_update: BoolProperty  # React to scene changes

    # Key methods:
    def force_update(self):
        """Force complete tree recalculation."""

    def update_nodes(self, nodes):
        """Update specific nodes."""

    @contextmanager
    def init_tree(self):
        """Context manager to suppress updates during tree construction."""
```

#### Node Base Class: `SverchCustomTreeNode`

Every Sverchok node inherits from `SverchCustomTreeNode`, a combination of multiple mixins:

```python
class SverchCustomTreeNode(UpdateNodes, NodeUtils, NodeDependencies, NodeDocumentation):
    """Base class for all Sverchok nodes."""
    sv_category = ''  # Category name for Shift+S menu
```

| Mixin | Purpose |
|-------|---------|
| `UpdateNodes` | Node lifecycle (sv_init, sv_update, sv_copy, sv_free), process management |
| `NodeUtils` | Logger shortcuts, tracked UI operators, data retrieval helpers |
| `NodeDependencies` | Optional library dependency checking |
| `NodeDocumentation` | Docstring parsing, help link generation |

#### Socket Data Cache

All socket data is stored in a single global dictionary in `core/socket_data.py`:

```python
socket_data_cache: dict[SockId, list] = dict()
```

Where `SockId` is a `NewType('SockId', str)` — a string hash of `node.node_id + socket.identifier + direction`.

### 1.3 Key Source Files

| File | Description |
|------|-------------|
| `node_tree.py` | SverchCustomTree, SverchCustomTreeNode, UpdateNodes, NodeUtils |
| `data_structure.py` | match_long_repeat, fullList, updateNode, multi_socket |
| `core/sockets.py` | All socket type definitions (SvStringsSocket, etc.) |
| `core/events.py` | Event classes (TreeEvent, AnimationEvent, etc.) |
| `core/socket_conversions.py` | Automatic type conversion functions |
| `core/socket_data.py` | Socket data cache (sv_get_socket, sv_set_socket, sv_forget_socket) |
| `core/update_system.py` | SearchTree, UpdateTree, control_center |
| `core/sv_custom_exceptions.py` | Exception hierarchy (SvNoDataError, etc.) |
| `utils/vectorize.py` | vectorize decorator, DataWalker, walk_data |
| `utils/sv_itertools.py` | Recursive data processing utilities |
| `utils/sv_bmesh_utils.py` | bmesh_from_pydata, pydata_from_bmesh |

---

## 2. Node System & Data Flow

### 2.1 Data Flow Pipeline

The core data flow pipeline operates as follows:

1. **Node execution order** is determined by topological sorting of the node graph (using `graphlib.TopologicalSorter`).
2. When a node's `process()` method executes, it reads input data via `socket.sv_get()`.
3. The node computes results and writes output data via `socket.sv_set()`.
4. Data flows downstream through links to the next node's input sockets.

```
+-------------------+     +------------------+     +-------------------+
| Input Node        |     | Processing Node  |     | Output Node       |
|                   |     |                  |     |                   |
| outputs[0].sv_set |---->| inputs[0].sv_get |     | inputs[0].sv_get  |
|                   |     |                  |     |                   |
|                   |     | process():       |     |                   |
|                   |     |   read inputs    |     |                   |
|                   |     |   compute        |     |                   |
|                   |     |   write outputs  |     |                   |
|                   |     |                  |     |                   |
|                   |     | outputs[0].sv_set|---->|                   |
+-------------------+     +------------------+     +-------------------+
```

### 2.2 Update System

The update system in `core/update_system.py` manages execution order and data propagation:

1. **Event arrival** — Blender triggers events (property change, link change, animation frame change, scene change).
2. **Control center** (`control_center()`) dispatches the event:
   - `PropertyEvent` — marks specific nodes as outdated
   - `TreeEvent` — marks entire tree as needing re-evaluation
   - `AnimationEvent` — triggers animation-dependent nodes
   - `SceneEvent` — triggers scene-dependent nodes
   - `ForceEvent` — resets and updates entire tree
3. **Task creation** — Creates a `Task` object that is processed by a timer.
4. **Tree walking** — `UpdateTree._walk()` yields nodes in topological order, starting from outdated nodes.
5. **Node execution** — For each node, `SearchTree.update_node()` is called.

#### Update Flow

```
Property Change (updateNode)
    -> PropertyEvent
        -> tree.update_nodes([node])
            -> topological sort
                -> node.process() for each dirty node
```

#### Tree Update Triggers

| Trigger | Condition |
|---------|-----------|
| Property change | `updateNode` callback on bpy.props |
| Link change | Tree topology modification |
| Frame change | `sv_animate = True` on tree |
| Scene change | `sv_scene_update = True` on tree |
| Force update | Manual `tree.force_update()` |
| Refresh button | User clicks refresh on interactive node |

#### Event Classes (`core/events.py`)

| Event | Description |
|-------|-------------|
| `TreeEvent` | Node/link addition/removal |
| `ForceEvent` | Full tree recalculation requested |
| `AnimationEvent` | Frame change or animation playback |
| `SceneEvent` | Scene modification |
| `PropertyEvent` | Node property changed |
| `GroupTreeEvent` | Change inside a group tree |
| `FileEvent` | New file loaded |
| `UndoEvent` | Undo operation executed |

### 2.3 Data Propagation Between Nodes

The `prepare_input_data` function propagates data between nodes:

```python
def prepare_input_data(prev_socks, input_socks):
    """Reads data from output sockets, converts if needed, writes to input sockets."""
    for ps, ns in zip(prev_socks, input_socks):
        if ps is None:
            continue
        try:
            data = ps.sv_get()  # read from upstream output socket
        except SvNoDataError:
            ns.sv_forget()  # clear input socket if no data
        else:
            # Type conversion if socket types differ
            if ps.bl_idname != ns.bl_idname:
                implicit_conversion = conversions[ns.default_conversion_name]
                try:
                    data = implicit_conversion.convert(ns, ps, data)
                except ImplicitConversionProhibited as e:
                    e.socket.links[0].is_valid = False
                    raise
            ns.sv_set(data)  # write to downstream input socket
```

### 2.4 SearchTree: Graph Navigation

`SearchTree` builds a bidirectional graph from the Blender node tree:

```python
class SearchTree:
    _from_nodes: dict[Node, set[Node]]   # predecessors
    _to_nodes: dict[Node, set[Node]]     # successors
    _from_sock: dict[NodeSocket, NodeSocket]  # input -> connected output
    _to_socks: dict[NodeSocket, set[NodeSocket]]  # output -> connected inputs
```

Key methods:
- `sort_nodes(nodes)` — Topological sort using `graphlib.TopologicalSorter`
- `nodes_from(from_nodes)` — BFS forward from given nodes
- `nodes_to(to_nodes)` — BFS backward to given nodes
- `previous_sockets(node)` — Get output sockets connected to node's inputs
- `update_node(node, suppress=True)` — Execute single node with error handling

The `SearchTree` constructor handles three special node types by "short-circuiting" them:
1. **Reroute nodes** (`NodeReroute`) — Transparent data pass-through; removed from graph and links reconnected.
2. **WiFi nodes** (`WifiInNode`/`WifiOutNode`) — Wireless link pairs; resolved to direct connections.
3. **Muted nodes** — Nodes with `is_active_output == False`; bypassed using `sv_internal_links` mapping.

### 2.5 Socket Data Cache Operations

**`sv_set_socket(socket, data)`**
```python
def sv_set_socket(socket, data):
    """Sets socket data in cache."""
    socket_data_cache[socket.socket_id] = data
```

**`sv_get_socket(socket, deepcopy=True)`**
```python
def sv_get_socket(socket, deepcopy=True):
    """Gets socket data from cache.
    If deepcopy is True, returns a deep copy.
    Set deepcopy=False when the node will not mutate input data (performance gain)."""
    data = socket_data_cache.get(socket.socket_id)
    if data is not None:
        return sv_deep_copy(data) if deepcopy else data
    else:
        raise SvNoDataError(socket)
```

**`sv_deep_copy(lst)`** — Custom deep copy optimized for Sverchok's data patterns. Faster than Python's `copy.deepcopy` for nested list structures:
```python
def sv_deep_copy(lst):
    if isinstance(lst, (list, tuple)):
        if lst and not isinstance(lst[0], (list, tuple)):
            return lst[:]  # shallow copy for flat lists
        return [sv_deep_copy(l) for l in lst]  # recurse for nested
    return lst  # atoms returned as-is
```

### 2.6 Execution Statistics

Each node execution records:
- `US_is_updated` — Boolean, whether node executed successfully
- `US_error` — Error message string if execution failed
- `US_error_stack` — Full traceback string
- `US_time` — Execution time in seconds (via `perf_counter`)
- `US_warning` — Warning messages from logging

---

## 3. Socket Type System

### 3.1 Available Socket Types

| Socket Class | Color (RGBA) | Data Type | SNLite Alias |
|-------------|------|-----------|-------------|
| `SvStringsSocket` | Green (0.6, 1.0, 0.6) | Numeric data (int, float, lists) | `s` |
| `SvVerticesSocket` | Orange (0.9, 0.6, 0.2) | 3D coordinates | `v` |
| `SvMatrixSocket` | Teal (0.2, 0.8, 0.8) | 4x4 transformation matrices | `m` |
| `SvQuaternionSocket` | Pink (0.9, 0.4, 0.7) | Rotation quaternions | — |
| `SvColorSocket` | Yellow (0.9, 0.8, 0.0) | RGBA color data | — |
| `SvCurveSocket` | Blue (0.5, 0.6, 1.0) | Parametric curves | `C` |
| `SvSurfaceSocket` | Deep purple (0.4, 0.2, 1.0) | Surface geometry | `S` |
| `SvSolidSocket` | Green (0.0, 0.65, 0.3) | Solids (FreeCAD/OpenCascade) | `So` |
| `SvScalarFieldSocket` | Dark orange (0.9, 0.4, 0.1) | Scalar field functions | `SF` |
| `SvVectorFieldSocket` | Deep blue (0.1, 0.1, 0.9) | Vector field functions | `VF` |
| `SvDictionarySocket` | White (1.0, 1.0, 1.0) | Key-value data | `D` |
| `SvObjectSocket` | Grey-green (0.69, 0.74, 0.73) | Blender object references | `o` |
| `SvFilePathSocket` | Yellow-green (0.9, 0.9, 0.3) | File paths | `FP` |
| `SvTextSocket` | Light blue (0.68, 0.85, 0.90) | Text data | — |
| `SvDummySocket` | Grey (0.8, 0.8, 0.8, 0.3) | Type placeholder | — |
| `SvChameleonSocket` | dynamic | Adopts linked socket type | — |

### 3.2 Socket Base Class: `SvSocketCommon`

```python
class SvSocketCommon(SvSocketProcessing):
    """Base socket class with data access methods."""

    def sv_get(self, default=None, deepcopy=True, implicit_conversions=None):
        """Retrieve socket data with fallback chain:
        1. Written data (from connected node)
        2. Node property (via prop_name)
        3. Socket property
        4. default parameter
        5. Raise error
        """

    def sv_set(self, data):
        """Store data in the socket."""

    def sv_forget(self):
        """Clear cached data."""
```

#### Socket Data Processing Flags

Sockets support automatic data transformation (see also [Section 5.5](#55-socket-processing-modes)):

```python
socket.use_flatten    # Flatten nested lists
socket.use_simplify   # Remove redundant nesting
socket.use_graft      # Add nesting level
socket.use_unwrap     # Remove one nesting level
socket.use_wrap       # Add one nesting level
```

### 3.3 Socket API Reference

```python
class SvSocketCommon(SvSocketProcessing):
    # Properties
    color: tuple                  # RGBA display color
    label: StringProperty         # Override display name
    use_prop: BoolProperty        # Use default property
    prop_name: StringProperty     # Node property to display
    objects_number: IntProperty   # Count of objects in data
    is_mandatory: BoolProperty    # Required for node execution
    nesting_level: IntProperty    # Expected data nesting (default 2)
    default_mode: EnumProperty    # 'NONE', 'EMPTY_LIST', 'MATRIX', 'MASK'
    pre_processing: EnumProperty  # 'NONE', 'ONE_ITEM'
    quick_link_to_node: str       # Node type for quick-link button
    default_conversion_name: str  # Conversion policy name

    # Methods
    def sv_get(default=..., deepcopy=True) -> list
    def sv_set(data: list) -> None
    def sv_forget() -> None
    def replace_socket(new_type, new_name=None) -> NodeSocket

    # Properties
    @property
    def socket_id(self) -> str    # Unique cache key
    @property
    def other(self) -> NodeSocket # Opposite linked socket
    @property
    def index(self) -> int        # Socket index in node
```

### 3.4 Implicit Socket Conversions

When sockets of different types are connected, Sverchok applies implicit conversion policies defined in `core/socket_conversions.py`.

#### DefaultImplicitConversionPolicy

| From Socket | To Socket | Conversion |
|---|---|---|
| `SvVerticesSocket` | `SvMatrixSocket` | Creates translation matrices from vertices |
| `SvVerticesSocket` | `SvColorSocket` | Appends alpha=1 to RGB |
| `SvMatrixSocket` | `SvVerticesSocket` | Extracts translation components |
| `SvMatrixSocket` | `SvQuaternionSocket` | Converts via `mat.to_quaternion()` |
| `SvQuaternionSocket` | `SvMatrixSocket` | Converts via `Quaternion.to_matrix().to_4x4()` |
| `SvStringsSocket` | `SvVerticesSocket` | Numbers become `(v, 0, 0)` vectors |
| `SvStringsSocket` | `SvColorSocket` | Numbers become `(v, v, v, 1)` colors |

Lenient socket types that accept arbitrary data without conversion:
```python
lenient_socket_types = {
    'SvStringsSocket', 'SvObjectSocket', 'SvColorSocket', 'SvVerticesSocket',
}
```

#### FieldImplicitConversionPolicy

| From Socket | To Socket | Conversion |
|---|---|---|
| `SvMatrixSocket` | `SvVectorFieldSocket` | Creates `SvMatrixVectorField` |
| `SvVerticesSocket` | `SvVectorFieldSocket` | Creates `SvConstantVectorField` |
| `SvStringsSocket` | `SvScalarFieldSocket` | Creates `SvConstantScalarField` |

#### Conversion Policy Registry

```python
class ConversionPolicies(Enum):
    DEFAULT = DefaultImplicitConversionPolicy
    FIELD   = FieldImplicitConversionPolicy
    LENIENT = LenientImplicitConversionPolicy
    SOLID   = SolidImplicitConversionPolicy
```

Each socket class specifies its policy:
```python
class SvVerticesSocket:
    default_conversion_name = ConversionPolicies.DEFAULT.conversion_name
class SvScalarFieldSocket:
    default_conversion_name = ConversionPolicies.FIELD.conversion_name
class SvTextSocket:
    default_conversion_name = ConversionPolicies.LENIENT.conversion_name
class SvSolidSocket:
    default_conversion_name = ConversionPolicies.SOLID.conversion_name
```

---

## 4. Data Nesting Levels (CRITICAL)

> **This section is CRITICAL for correct Sverchok usage. Incorrect nesting is the #1 source of errors.**

### 4.1 Nesting Level Convention

Sverchok uses nested Python lists as its primary data structure. EVERY socket carries data at a specific nesting level:

```
Level 0: scalar           (e.g., 5.0)
Level 1: list of scalars  (e.g., [1, 2, 3])
Level 2: list of lists    (e.g., [[1, 2, 3], [4, 5, 6]])
Level 3: list of list of lists (e.g., [[(x,y,z), ...], ...])
```

### 4.2 Standard Data Formats Per Socket Type

**SvStringsSocket** — Level 2:
```python
# CORRECT: Numeric data at level 2
numbers = [[1, 2, 3], [4, 5, 6]]  # Two objects, each with 3 values

# WRONG: Flat list
numbers = [1, 2, 3]  # Level 1 — WILL cause mismatches
```

**SvVerticesSocket** — Level 3:
```python
# CORRECT: Vertex data at level 3 — list of objects, each containing list of (x,y,z) tuples
vertices = [
    [(0, 0, 0), (1, 0, 0), (1, 1, 0)],  # Object 1: 3 verts
    [(2, 0, 0), (3, 0, 0), (3, 1, 0)],  # Object 2: 3 verts
]

# WRONG: Missing object wrapper
vertices = [(0, 0, 0), (1, 0, 0)]  # Level 2 — WILL fail
```

**Edge data** — Level 2:
```python
edges = [
    [(0, 1), (1, 2), (2, 0)],  # Object 1
    [(0, 1), (1, 2), (2, 0)],  # Object 2
]
```

**Face data** — Level 2:
```python
faces = [
    [(0, 1, 2)],        # Object 1: one triangle
    [(0, 1, 2, 3)],     # Object 2: one quad
]
```

**SvMatrixSocket** — Level 1:
```python
matrices = [Matrix(), Matrix()]  # List of matrices (one per object)
```

**Data format at each socket type:**
```
SvStringsSocket:  [[1, 2, 3], [4, 5, 6]]     (level 2)
SvVerticesSocket: [[(x,y,z), (x,y,z)], ...]  (level 3)
SvMatrixSocket:   [Matrix, Matrix, ...]       (level 1)
```

### 4.3 Nesting Level Detection

```python
from sverchok.data_structure import get_data_nesting_level

get_data_nesting_level(1)              # 0 (scalar)
get_data_nesting_level([1, 2, 3])      # 1 (flat list)
get_data_nesting_level([[1, 2], [3]])   # 2 (nested)
```

### 4.4 The "Objects" Mental Model

The outermost list dimension represents **objects**:
- `[[1, 2, 3]]` = ONE object with 3 values
- `[[1, 2, 3], [4, 5, 6]]` = TWO objects with 3 values each
- `[[(0,0,0), (1,0,0)]]` = ONE object with 2 vertices
- `[[(0,0,0)], [(1,0,0)]]` = TWO objects with 1 vertex each

**ALWAYS wrap output in the object-level list.** Even a single object must be `[[...]]`, not `[...]`.

---

## 5. List Matching & Vectorization

### 5.1 List Matching System

When inputs have different numbers of objects, Sverchok provides five matching modes:

| Mode | Behavior | Example: [1,2,3] + [10,20] |
|------|----------|----------------------------|
| **REPEAT** | Repeat last element | [1,2,3] + [10,20,20] |
| **CYCLE** | Cycle from beginning | [1,2,3] + [10,20,10] |
| **SHORT** | Truncate to shortest | [1,2] + [10,20] |
| **XREF** | Cross-reference (all combinations) | 3×2 = 6 pairs |
| **XREF2** | Cross-reference variant | Similar to XREF with different nesting |

The **default mode is REPEAT** — the last element is repeated to fill shorter lists.

### 5.2 Core Matching Functions

#### `match_long_repeat(lists)`

The most commonly used list matching function:

```python
from sverchok.data_structure import match_long_repeat

a = [[1, 2, 3]]
b = [[10, 20]]
c = [[100]]

matched = match_long_repeat([a, b, c])
# Result: [[[1, 2, 3]], [[10, 20, 20]], [[100, 100, 100]]]
```

#### `fullList(lst, count)`

Extends a list in-place to the given count by repeating the last element:

```python
from sverchok.data_structure import fullList

data = [1, 2, 3]
fullList(data, 6)
# data is now [1, 2, 3, 3, 3, 3]
```

#### `repeat_last(lst)`

Generator that yields all items and then repeats the last item indefinitely:

```python
from sverchok.data_structure import repeat_last

gen = repeat_last([1, 2, 3])
values = [next(gen) for _ in range(6)]
# values = [1, 2, 3, 3, 3, 3]
```

#### `list_match_func` Dictionary

```python
from sverchok.data_structure import list_match_func

# Available modes:
# 'REPEAT'  - repeat last
# 'CYCLE'   - cycle through
# 'XREF'    - cross reference
# 'TRIM'    - trim to shortest
# 'MATCH'   - match by index
```

### 5.3 NumPy Matching Variants

For numpy array inputs, dedicated matching functions exist:

```python
numpy_list_match_func = {
    "SHORT":  numpy_match_short,       # array[:min_len]
    "CYCLE":  numpy_match_long_cycle,  # np.tile + concatenate
    "REPEAT": numpy_match_long_repeat, # np.repeat last row
}
```

### 5.4 The `vectorize` Decorator

Defined in `utils/vectorize.py`, this decorator transforms a function that operates on single values into one that operates on nested lists of arbitrary depth:

```python
from sverchok.utils.vectorize import vectorize

# Type annotations control how deeply the decorator unwraps data
def my_function(*, vertices: List[Tuple[float, float, float]],
                   count: int,
                   mode: str) -> Tuple[list, list]:
    return result1, result2

# Usage in node:
class MyNode:
    def process(self):
        verts = self.inputs[0].sv_get()
        count = self.inputs[1].sv_get()
        vectorized_func = vectorize(my_function, match_mode=self.match_mode)
        out1, out2 = vectorized_func(vertices=verts, count=count, mode=self.mode)
```

**Nesting level detection from annotations:**
```python
# float, int, bool, Matrix, str => 0
# list, tuple (bare) => 1
# List[float] => 1
# List[List[float]] => 2
# List[Tuple[float, float, float]] => 2
```

### 5.5 The `SvRecursiveNode` Mixin

The `SvRecursiveNode` mixin (in `utils/nodes_mixins/recursive_nodes.py`) provides automatic vectorization for node classes:

```python
class SvAwesomeNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    def sv_init(self, context):
        p1 = self.inputs.new('SvVerticesSocket', "Param1")
        p1.is_mandatory = True     # node will not execute without this input
        p1.nesting_level = 3       # expects vertex-level nesting
        p1.default_mode = 'NONE'   # no default value

        p2 = self.inputs.new('SvStringsSocket', "Param2")
        p2.nesting_level = 2       # expects number-list nesting
        p2.default_mode = 'EMPTY_LIST'  # default: [[]]
        p2.pre_processing = 'ONE_ITEM'  # collapse to one value per object
```

**Default mode options:**
```python
DEFAULT_TYPES = {
    'NONE':       ...,          # Ellipsis (no default)
    'EMPTY_LIST': [[]],         # Empty nested list
    'MATRIX':     [Matrix()],   # Identity matrix
    'MASK':       [[True]],     # Single True mask
}
```

### 5.6 Socket Processing Modes

#### Input Preprocessing

When data is read from an input socket, `preprocess_input(data)` applies transformations in this order:

1. **Flatten** (if `use_flatten` is True)
2. **OR Simplify** (if `use_simplify` is True) — mutually exclusive with Flatten
3. **Graft** (if `use_graft` is True) — adds nesting to each element
4. **Unwrap** (if `use_unwrap` is True)
5. **Wrap** (if `use_wrap` is True) — mutually exclusive with Unwrap

#### Output Postprocessing

When data is written to an output socket, `postprocess_output(data)` applies the same transformations plus:
- **Flatten Topology** (if `use_flatten_topology` is True)

#### Mode Flags Display

Active modes are shown in the socket label:
- `F` — Flatten, `FT` — Flatten Topology, `S` — Simplify
- `G` — Graft, `G2` — Graft Topology (SvStringsSocket only)
- `U` — Unwrap, `W` — Wrap, `R` — Reparametrize (SvCurveSocket only)

### 5.7 The `match_sockets` Function

A simpler matching helper for common use cases:

```python
from sverchok.utils.vectorize import match_sockets

data1 = [[1,2,3]]
data2 = [[4,5], [6,7]]
data3 = [[8]]
for d1, d2, d3 in match_sockets(data1, data2, data3):
    print(f"{d1=}, {d2=}, {d3=}")
# iteration 1: d1=[1,2,3], d2=[4,5,5], d3=[8]
# iteration 2: d1=[1,2,3], d2=[6,7,7], d3=[8]
```

### 5.8 Recursive Processing Utilities

```python
from sverchok.utils.sv_itertools import recurse_fx, recurse_fxy

# Recursively apply function to leaf elements:
result = recurse_fx([[1, 2], [3, 4]], lambda x: x * 2)
# Result: [[2, 4], [6, 8]]

# Binary recursion:
result = recurse_fxy([1, 2, 3], [10, 20, 30], lambda x, y: x + y)
# Result: [11, 22, 33]
```

---

## 6. Node Categories (Complete Inventory)

Sverchok provides 500+ nodes organized into categories accessible via the Add menu (Shift+A in the node editor):

### 6.1 Generator Nodes
Create geometry primitives: Box, Sphere, Cylinder, Torus, Plane, Circle, Line, NGon, Suzanne, IcoSphere, etc.

### 6.2 Transforms
Move, Rotate, Scale, Mirror, Matrix Apply, Shear, Bend, Twist, and transformation utilities.

### 6.3 Analyzers
Measure distances, areas, volumes, curvatures. Bounding box calculations, normals, edge angles, KDTree searches.

### 6.4 Modifier Nodes
- **Make**: Convex Hull, Voronoi, Delaunay, Join, Bridge, Fill Holes
- **Change**: Subdivide, Dissolve, Merge by Distance, Separate, Flip
- **Deform**: Noise, Smooth, Lattice, Proportional Edit

### 6.5 List Processing
List Join, Split, Shift, Reverse, Slice, Sort, Shuffle, Mask, Filter, Zip, Repeat, Length, Sum, Statistics.

### 6.6 Number Nodes
Number, Integer, Float, Range, Random, Map Range, Clamp, Formula, Mix Numbers.

### 6.7 Vector Nodes
Vector In/Out, Math, Interpolation, Noise, Evaluate Field, Drop, Project, Reflect.

### 6.8 Matrix Nodes
Matrix In/Out, Apply, Multiply, Invert, Interpolation, Euler, Deform, Track To.

### 6.9 Logic Nodes
Switch, Gate, Compare, Logic, Mask operations.

### 6.10 Viz (Visualization)
Viewer Draw, Viewer BMesh, Stethoscope, Index Viewer, Polyline Viewer, Texture Viewer, Spreadsheet.

### 6.11 Text
Text In/Out, Simple Text, String operations, CSV handling, JSON operations.

### 6.12 Scene
Object In/Out, Frame Info, Collection Picker, Set Object Data, Instancer, Particle systems.

### 6.13 Layout
Frame, Reroute, WiFi In/Out, Group nodes, Node Notes.

### 6.14 Script Nodes
SNLite, SN Functor B, Formula Mk5, Profile Mk3, Generative Art, Mesh Expression, Multi Exec, NumExpr. *(Detailed in [Section 7](#7-python-scripting-snlite-sn-functor-b-formula))*

### 6.15 Curve/Surface/Field Nodes
NURBS operations, Bezier, Spline, Surface from curves, Iso curves, Field evaluation, Marching cubes.

### 6.16 Solid Nodes
FreeCAD/OpenCascade-based operations: Box, Cylinder, Sphere, Boolean, Fillet, Chamfer, Shell, Offset.

### 6.17 Pulga Physics
Physics simulation nodes for particle-based simulations.

### 6.18 Exchange
Import/Export: SVG, DXF, JSON, NumPy, CSV.

---

## 7. Python Scripting (SNLite, SN Functor B, Formula)

### 7.1 Script Node Lite (SNLite)

**Source**: `nodes/script/script1_lite.py`
**Node ID**: `SvScriptNodeLite`
**Menu Trigger**: `snl`

SNLite is the primary user-facing scripting node. It executes Python scripts stored in Blender's Text Editor, dynamically creating input/output sockets based on declarations in the script.

#### Socket Declaration Syntax

```python
"""
in  socket_name  socket_type  [default=value]  [nested=level]
out socket_name  socket_type
"""
```

**Socket type identifiers**:
- `s` = `SvStringsSocket` (scalars, integers, generic numeric data)
- `v` = `SvVerticesSocket` (3D coordinate data)
- `m` = `SvMatrixSocket` (4x4 transformation matrices)
- `o` = `SvObjectSocket` (Blender object references)
- `C` = `SvCurveSocket`, `S` = `SvSurfaceSocket`, `So` = `SvSolidSocket`
- `SF` = `SvScalarFieldSocket`, `VF` = `SvVectorFieldSocket`
- `D` = `SvDictionarySocket`, `FP` = `SvFilePathSocket`

**Options**:
- `default=<value>`: Default value when socket is unconnected
- `nested=<int>` or `n=<int>`: Nesting level (0=raw, 1=single list, 2=list of lists)
- `;=<value>`: Alternative default syntax

#### Complete SNLite Example: Flower Petal Generator

```python
"""
in   n_petals s        default=8       nested=2
in   vp_petal s        default=10      nested=2
in   profile_radius s  default=2.3     nested=2
in   amp s             default=1.0     nested=2
out  verts v
out  edges s
"""

import numpy as np
from mathutils import Vector, Euler
from math import pi, sin, cos

TAU = 2 * pi
N = int(n_petals * vp_petal)

pi_vals = np.tile(np.linspace(0, TAU, int(vp_petal), endpoint=False), int(n_petals))
amps = np.cos(pi_vals) * amp
theta = np.linspace(0, TAU, N, endpoint=False)

circle_coords = np.array([np.sin(theta), np.cos(theta), np.zeros(N)])
coords = circle_coords.T * (profile_radius + amps.reshape((-1, 1)))

verts = [coords.tolist()]
edges = [[(i, (i + 1) % N) for i in range(N)]]
```

#### SNLite Example: KDTree Neighbor Search

```python
"""
in  verts  v
in  radius s  default=0.3  nested=2
out edges  s
"""
import mathutils

size = len(verts)
kd = mathutils.kdtree.KDTree(size)
for i, xyz in enumerate(verts):
    kd.insert(xyz, i)
kd.balance()

edges = [[]]
edge_set = set()
r = radius
for idx, vtx in enumerate(verts):
    n_list = kd.find_range(vtx, r)
    for co, index, dist in n_list:
        if index == idx:
            continue
        edge_set.add(tuple(sorted([idx, index])))

edges[0] = list(edge_set)
```

#### Built-in Aliases Available in SNLite

```python
{
    'vectorize': vectorize,          # from sverchok.utils.snlite_utils
    'bpy': bpy,                      # Blender Python API
    'np': np,                        # NumPy
    'ddir': ddir,                    # filtered dir() utility
    'get_user_dict': self.get_user_dict,    # persistent per-node storage
    'reset_user_dict': self.reset_user_dict,
    'cprint': console_print,         # console output
    'console_print': console_print,
    'sv_njit': sv_njit,              # Numba JIT compilation
    'sv_njit_clear': sv_njit_clear,
    'bmesh_from_pydata': bmesh_from_pydata,  # BMesh construction
    'pydata_from_bmesh': pydata_from_bmesh   # BMesh extraction
}
```

#### SNLite Special Functions

**`setup()`** — One-time initialization, return values persist across `process()` calls:
```python
def setup():
    import random
    seed_data = [random.random() for _ in range(100)]
    return locals()
# seed_data is available in the main script body
```

**`ui(self, context, layout)`** — Custom UI drawing in the node panel:
```python
def ui(self, context, layout):
    layout.label(text="Custom Sphere Generator")
    layout.prop(self, "custom_enum")
```

**`sv_internal_links(self)`** — Controls mute behavior:
```python
def sv_internal_links(self):
    return [(self.inputs[0], self.outputs[0])]
```

#### SNLite Additional Features

**Required inputs**:
```python
"""
in  verts  v  .  required=True
in  scale  s  default=1.0
out result v
"""
```

**Custom enums** (up to 2):
```python
"""
enum custom_enum = A B C D
in  value s  default=1.0
out result s
"""
if custom_enum == "A":
    result = [[value * 2]]
```

**File handler**:
```python
"""
display_file_handler
in  scale s  default=1.0
out verts v
"""
```

**Include files**:
```python
"""
includes: helper_functions.py, config.py
in  data s
out result s
"""
```

#### SNLite Template System

Templates are stored in `node_scripts/SNLite_templates/` organized in categories:

| Category | Description |
|----------|-------------|
| `demo` | Demonstration scripts (spirals, voronoi, genetic algorithms) |
| `bpy_stuff` | Blender API interaction examples |
| `bmesh` | BMesh geometry manipulation |
| `utils` | Utility scripts (DXF export, IFC export, SVG import) |
| `templates` | Reusable template patterns |

#### Persistent Per-Node Storage

```python
"""
in  trigger s  default=0
out count   s
"""
storage = get_user_dict()
if 'counter' not in storage:
    storage['counter'] = 0
storage['counter'] += 1
count = [[storage['counter']]]
```

### 7.2 SN Functor B (Scripted Node Functor)

**Source**: `nodes/script/sn_functor_b.py`
**Node ID**: `SvSNFunctorB`
**Menu Trigger**: `functorB`

SN Functor B provides a more structured approach to scripting with three distinct functions: `functor_init`, `process`, and optionally `draw_buttons`.

```python
import bpy
from sverchok.data_structure import updateNode

def functor_init(self, context):
    """Called once when the script is loaded. Define sockets here."""
    self.inputs.new('SvStringsSocket', 'radius')
    self.inputs.new('SvStringsSocket', 'segments')
    self.outputs.new('SvVerticesSocket', 'verts')
    self.outputs.new('SvStringsSocket', 'edges')

def process(self):
    """Called on each evaluation. Read inputs and write outputs."""
    from math import sin, cos, pi

    if not self.inputs['radius'].is_linked:
        return

    radius_data = self.inputs['radius'].sv_get()
    segments_data = self.inputs['segments'].sv_get(default=[[24]])

    all_verts = []
    all_edges = []

    for radius_list, seg_list in zip(radius_data, segments_data):
        for radius, segments in zip(radius_list, seg_list):
            segments = int(segments)
            verts = []
            edges = []
            for i in range(segments):
                angle = 2 * pi * i / segments
                verts.append((radius * cos(angle), radius * sin(angle), 0))
                edges.append((i, (i + 1) % segments))
            all_verts.append(verts)
            all_edges.append(edges)

    self.outputs['verts'].sv_set(all_verts)
    self.outputs['edges'].sv_set(all_edges)

def draw_buttons(self, context, layout):
    """Optional custom UI drawing."""
    layout.label(text="Circle Generator")
```

#### Functor B Properties

```python
# Available properties (5 of each type):
# int_00 through int_04 - IntProperty
# float_00 through float_04 - FloatProperty
# bool_00 through bool_04 - BoolProperty

def draw_buttons(self, context, layout):
    layout.prop(self, 'float_00', text='Scale')
    layout.prop(self, 'int_00', text='Count')
```

#### SNLite vs Functor B Comparison

| Feature | SNLite | Functor B |
|---------|--------|-----------|
| Socket definition | Inline text declarations | `functor_init()` function |
| Process logic | Script body runs directly | `process()` function |
| UI customization | `ui()` function | `draw_buttons()` function |
| Properties | `custom_enum` (2 enums) | 5x int, 5x float, 5x bool |
| Self access | Limited (via aliases) | Full `self` reference |
| Initialization | `setup()` function | `functor_init()` function |
| Complexity | Lower | Higher |

**When to use which:**
- **SNLite**: Quick prototyping, simple transformations, learning Sverchok
- **Functor B**: Complex nodes requiring custom UI, multiple property types, or full control

### 7.3 Formula Node (Mk5)

**Source**: `nodes/script/formula_mk5.py`
**Node ID**: `SvFormulaNodeMk5`

The Formula Node evaluates Python mathematical expressions with safety restrictions. It supports up to 4 simultaneous formulas. Variables used in formulas automatically become input sockets.

```
x + 1
0.75 * X + 0.25 * Y
R * sin(phi)
Vector((x, y, z))
[x**2 for x in range(n)]
```

#### Available Functions and Constants

**Math functions** (from Python `math` module):
```
acos, acosh, asin, asinh, atan, atan2, atanh,
ceil, copysign, cos, cosh, degrees,
erf, erfc, exp, expm1,
fabs, factorial, floor, fmod, frexp, fsum,
gamma, hypot, isfinite, isinf, isnan,
ldexp, lgamma, log, log10, log1p, log2,
modf, pow, radians, sin, sinh, sqrt, tan, tanh, trunc
```

**Constants**: `pi`, `e`

**Additional functions**: `abs, sign, max, min, len, sum, zip, any, all, dir`

**Type constructors**: `int, float, str, list, tuple, dict, set`

**Blender/Sverchok objects**: `Vector`, `Matrix`, `np`, `bpy`

#### Safe Evaluation System

```python
def safe_eval(string, variables):
    """Evaluate expression, allowing only safe functions."""
    env = dict()
    env.update(safe_names)    # Allowed functions/constants
    env.update(variables)     # User-provided variable values
    env["__builtins__"] = {}  # Block all builtins
    root = ast.parse(string, mode='eval')
    return eval(compile(root, "<expression>", 'eval'), env)
```

#### NumPy-Accelerated Mode

The formula node can use NumPy-accelerated function variants:
```python
# Standard mode: sin(x) uses math.sin (scalars)
# NumPy mode: sin(x) uses np.sin (arrays)
```

#### Formula Node Example: Helix

```
Formula 1: radius * cos(t)
Formula 2: radius * sin(t)
Formula 3: pitch * t / (2 * pi)
```
With 3 dimensions enabled, this produces X, Y, Z coordinates. Connect a "Number Range" node to `t`.

### 7.4 Profile Node (Mk3)

**Source**: `nodes/script/profile_mk3.py`
**Node ID**: `SvProfileNodeMK3`

Defines 2D parametric profiles using an SVG-like DSL. Profiles are stored as text datablocks in Blender.

#### Profile Language Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `M`, `m` | `M x y` | Move to (absolute/relative) |
| `L`, `l` | `L x y [n=segments] [z]` | Line to |
| `H`, `h` | `H x [n=segments] ;` | Horizontal line to |
| `V`, `v` | `V y [n=segments] ;` | Vertical line to |
| `C`, `c` | `C x1 y1 x2 y2 x y [n=verts] [z]` | Cubic Bezier curve |
| `Q`, `q` | `Q x1 y1 x y [n=segments] [z]` | Quadratic Bezier |
| `A`, `a` | `A rx ry rot flag1 flag2 x y [n=verts] [z]` | Arc |
| `@I`, `@i` | `@I [@smooth] degree p1 p2 ... [n=segments] [z] ;` | NURBS interpolation |
| `x` | `x` | Close path (cyclic) |
| `X` | `X` | Close all edges cyclically |

#### Variable System

```
default width = 0.3
default height = 2.7
let half_width = {width / 2}
L {width + margin} {height * 0.5}
```

Variables used in the profile (except `let` declarations) automatically become input sockets.

#### Architectural Profile Example: I-Beam

```
default width = 0.2
default height = 0.4
default flange_thickness = 0.02
default web_thickness = 0.01

let hw = {width / 2}
let hh = {height / 2}
let ft = {flange_thickness}
let wt = {web_thickness / 2}

M -{hw} -{hh}
L {hw} -{hh}
L {hw} {-hh + ft}
L {wt} {-hh + ft}
L {wt} {hh - ft}
L {hw} {hh - ft}
L {hw} {hh}
L -{hw} {hh}
L -{hw} {hh - ft}
L -{wt} {hh - ft}
L -{wt} {-hh + ft}
L -{hw} {-hh + ft}
X
```

### 7.5 Other Script Nodes

| Node | Source | Description |
|------|--------|-------------|
| Generative Art | `nodes/script/generative_art.py` | L-System based generative art using XML rules |
| Formula Interpolate | `nodes/script/formula_interpolate.py` | Interpolated formulas over ranges |
| Mesh Expression | `nodes/script/mesh_eval.py` | Mesh generation from JSON structures |
| Multi Exec | `nodes/script/multi_exec.py` | Multiple sequential Python expressions |
| NumExpr | `nodes/script/numexpr_node.py` | Accelerated numerical expressions via `numexpr` |
| Topology Simple | `nodes/script/topology_simple.py` | Topological operation expressions |

---

## 8. Custom Node Development

### 8.1 Minimal Custom Node Template

```python
import bpy
from bpy.props import FloatProperty, IntProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvMyCustomNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: scale transform multiply
    Tooltip: Scales vertices by a factor

    Multiplies all input vertices by a scale factor.
    """
    bl_idname = 'SvMyCustomNode'
    bl_label = 'My Custom Node'
    bl_icon = 'NONE'

    scale_factor: FloatProperty(
        name='Scale',
        description='Scale factor',
        default=1.0,
        update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Scale').prop_name = 'scale_factor'
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def sv_draw_buttons(self, context, layout):
        pass  # Scale is shown via socket prop_name

    def process(self):
        if not self.outputs['Vertices'].is_linked:
            return

        verts = self.inputs['Vertices'].sv_get(default=[[]])
        scale = self.inputs['Scale'].sv_get()

        verts, scale = match_long_repeat([verts, scale])

        result = []
        for vert_list, scale_list in zip(verts, scale):
            scaled = []
            for v, s in zip(vert_list, scale_list):
                scaled.append((v[0] * s, v[1] * s, v[2] * s))
            result.append(scaled)

        self.outputs['Vertices'].sv_set(result)

def register():
    bpy.utils.register_class(SvMyCustomNode)

def unregister():
    bpy.utils.unregister_class(SvMyCustomNode)
```

### 8.2 Node Docstring Format

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: keyword1 keyword2 keyword3
    Tooltip: Short description shown on hover

    Longer description of the node functionality.
    """
```
- **Triggers**: Space-separated keywords for the node search (Shift+S)
- **Tooltip**: One-line description shown in the node tooltip

### 8.3 Socket Creation Methods

#### Direct creation in `sv_init`
```python
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Count')
    self.inputs.new('SvVerticesSocket', 'Vertices')
    self.outputs.new('SvVerticesSocket', 'Result')
```

#### Using `sv_new_input` helper
```python
def sv_init(self, context):
    self.sv_new_input('SvStringsSocket', 'Count',
                      prop_name='count_prop', hide_safe=True)
    self.sv_new_input('SvVerticesSocket', 'Vertices',
                      is_mandatory=True)
```

### 8.4 Node Lifecycle Methods

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):

    def sv_init(self, context):
        """Called once when node is created. Create sockets, set initial properties."""

    def process(self):
        """Called on each evaluation. Read inputs, compute, write outputs."""

    def sv_update(self):
        """Called on tree editor changes (add/remove links/nodes).
        Used for dynamic socket type changes."""

    def sv_copy(self, original):
        """Called when node is duplicated. Clean up instance-specific properties."""

    def sv_free(self):
        """Called when node is deleted. Clean up resources."""

    def sv_draw_buttons(self, context, layout):
        """Draw node UI elements in the node body."""

    def sv_draw_buttons_ext(self, context, layout):
        """Draw extended UI in the properties panel."""
```

### 8.5 Node Properties and Animation

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    is_animation_dependent = True   # Node updates on frame change
    is_scene_dependent = True       # Node updates on scene changes

    my_prop: FloatProperty(
        name='My Property',
        default=1.0,
        update=updateNode  # Triggers node re-evaluation
    )
```

The `updateNode` function from `sverchok.data_structure` is the standard callback that triggers node tree re-evaluation.

### 8.6 Node Dependencies

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    sv_dependencies = {'scipy', 'skimage'}

    def process(self):
        if self.dependency_error:
            raise self.dependency_error
        import scipy
```

### 8.7 Registration Pattern

```python
# For nodes within Sverchok repository:
classes = [SvMyNode]
register, unregister = bpy.utils.register_classes_factory(classes)

# For external addons extending Sverchok:
def register():
    bpy.utils.register_class(SvMyNode)

def unregister():
    bpy.utils.unregister_class(SvMyNode)
```

### 8.8 Standard Process Method Pattern

```python
def process(self):
    # 1. Early exit if no output is connected
    if not any(s.is_linked for s in self.outputs):
        return

    # 2. Read inputs with defaults
    verts = self.inputs['Vertices'].sv_get(default=[[]])
    scale = self.inputs['Scale'].sv_get(default=[[1.0]])

    # 3. Match input lengths
    verts, scale = match_long_repeat([verts, scale])

    # 4. Process each object
    result_verts = []
    for vert_list, scale_list in zip(verts, scale):
        vert_list, scale_list = match_long_repeat([vert_list, scale_list])
        new_verts = []
        for v, s in zip(vert_list, scale_list):
            new_verts.append((v[0] * s, v[1] * s, v[2] * s))
        result_verts.append(new_verts)

    # 5. Set outputs
    self.outputs['Vertices'].sv_set(result_verts)
```

### 8.9 BMesh Integration in Nodes

```python
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata, pydata_from_bmesh
import bmesh

def process(self):
    verts = self.inputs['Vertices'].sv_get()
    edges = self.inputs['Edges'].sv_get(default=[[]])
    faces = self.inputs['Faces'].sv_get(default=[[]])

    result_verts, result_edges, result_faces = [], [], []

    for v, e, f in zip(verts, edges, faces):
        bm = bmesh_from_pydata(v, e, f, normal_update=True)
        bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=1)
        new_v, new_e, new_f = pydata_from_bmesh(bm)
        bm.free()
        result_verts.append(new_v)
        result_edges.append(new_e)
        result_faces.append(new_f)

    self.outputs['Vertices'].sv_set(result_verts)
    self.outputs['Edges'].sv_set(result_edges)
    self.outputs['Faces'].sv_set(result_faces)
```

### 8.10 Performance: NumPy Vectorized Operations

```python
# Slow (Python loop):
result = []
for v in vertices:
    result.append((v[0] * 2, v[1] * 2, v[2] * 2))

# Fast (NumPy vectorized):
result = (np.array(vertices) * 2).tolist()
```

### 8.11 Error Handling in Nodes

```python
from sverchok.core.sv_custom_exceptions import SvNoDataError

def process(self):
    try:
        data = self.inputs['Data'].sv_get()
        if not data or not data[0]:
            raise SvNoDataError(self)
        result = self.compute(data)
        self.outputs['Result'].sv_set(result)
    except SvNoDataError:
        raise  # Shows "no data" color instead of error color
    except Exception as e:
        self.exception(f"Processing failed: {e}")
        raise
```

### 8.12 Logging

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    def process(self):
        self.debug("Debug message")
        self.info("Info message")
        self.warning("Warning message")
        self.error("Error message")
        self.exception("Exception with traceback")
```

---

## 9. External API Access

### 9.1 Accessing Sverchok Node Trees

Sverchok node trees are stored as `bpy.data.node_groups` with `bl_idname = 'SverchCustomTreeType'`:

```python
import bpy

# List all Sverchok node trees
for ng in bpy.data.node_groups:
    if ng.bl_idname == 'SverchCustomTreeType':
        print(f"Tree: {ng.name}")
        for node in ng.nodes:
            print(f"  Node: {node.name} ({node.bl_idname})")

# Access a specific tree
tree = bpy.data.node_groups.get('MyTree')
```

### 9.2 Creating Nodes Programmatically

```python
import bpy

tree = bpy.data.node_groups.new('ScriptedTree', 'SverchCustomTreeType')

number_node = tree.nodes.new('SvNumberNode')
number_node.location = (0, 0)

viewer_node = tree.nodes.new('SvViewerDrawMk4')
viewer_node.location = (400, 0)

tree.links.new(
    number_node.outputs['Value'],
    viewer_node.inputs['Vertices']
)
```

### 9.3 Modifying Node Properties

```python
tree = bpy.data.node_groups['MyTree']

node = tree.nodes['Formula']
node.formula1 = "sin(x) * R"

snlite = tree.nodes['Scripted Node Lite']
snlite.script_name = 'my_script.py'
```

### 9.4 Triggering Tree Updates

```python
tree = bpy.data.node_groups['MyTree']

tree.force_update()                       # Force full tree update
tree.update_nodes([tree.nodes['Formula']]) # Update specific nodes
tree.sv_process = True                    # Enable processing
tree.sv_process = False                   # Disable processing
```

### 9.5 Reading Node Output Data

```python
tree = bpy.data.node_groups['MyTree']
node = tree.nodes['Some Node']

output = node.outputs['Vertices']
if output.is_linked:
    data = output.sv_get()
    print(f"Vertices: {data}")
```

### 9.6 Batch Processing Node Trees

```python
def batch_process_with_params(tree_name, param_sets):
    """Process a tree with different parameter sets."""
    tree = bpy.data.node_groups[tree_name]
    results = []
    for params in param_sets:
        for node_name, prop_name, value in params:
            node = tree.nodes[node_name]
            setattr(node, prop_name, value)
        tree.force_update()
        output_node = tree.nodes['Output']
        data = output_node.outputs['Result'].sv_get()
        results.append(data)
    return results
```

### 9.7 Using `init_tree` Context Manager

When building trees programmatically, suppress intermediate updates for performance:

```python
tree = bpy.data.node_groups.new('BuildTree', 'SverchCustomTreeType')

with tree.init_tree():
    n1 = tree.nodes.new('SvNumberNode')
    n2 = tree.nodes.new('SvFormulaNodeMk5')
    n3 = tree.nodes.new('SvViewerDrawMk4')
    tree.links.new(n1.outputs[0], n2.inputs[0])
    tree.links.new(n2.outputs[0], n3.inputs[0])
# Tree updates once when exiting the context manager
```

### 9.8 Key Internal APIs

#### `sverchok.data_structure` (Complete Reference)

```python
from sverchok.data_structure import (
    updateNode,              # Standard property update callback
    match_long_repeat,       # REPEAT mode list matching
    fullList,                # In-place extend by repeating last
    repeat_last,             # Infinite iterator repeating last element
    get_data_nesting_level,  # Determine nesting depth
    changable_sockets,       # Dynamic socket type matching
    replace_socket,          # Replace socket preserving links
    multi_socket,            # Manage dynamic multi-socket nodes
    get_other_socket,        # Follow links through reroute nodes
    enum_item_4,             # Create enum items from simple list
    list_match_func,         # Dictionary of matching mode functions
    flat_iter,               # Deep flatten to atoms
    fixed_iter,              # Fixed-length iterator with cycling
    get_edge_loop,           # Generate cyclic edge connectivity
)
```

#### `sverchok.utils.sv_itertools`

```python
from sverchok.utils.sv_itertools import (
    sv_zip_longest,             # Class-based zip_longest
    recurse_fx,                 # Recursive unary function application
    recurse_fxy,                # Recursive binary function application
    recurse_f_level_control,    # Level-aware recursion with matching
    process_matched,            # Core recursive vectorization engine
)
```

---

## 10. IfcSverchok (IFC/BIM Integration)

### 10.1 Overview

**Source**: `https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok`
**Authors**: Martina Jakubowska, Dion Moult
**License**: GPL-3.0
**Status**: Alpha / Work-in-Progress (version 0.0.0)
**Minimum Requirements**: Blender 3.1+, Sverchok v1.2+, Bonsai addon

IfcSverchok enables creation of native IFC (Industry Foundation Classes) models using visual node-based programming in Blender. It bridges Sverchok's parametric geometry with the open BIM standard.

Key capabilities:
- Create IFC entities (walls, slabs, columns, spaces) from node connections
- Convert Blender meshes or Sverchok geometry to IFC representations
- Build IFC spatial hierarchies (project -> site -> building -> storey)
- Assign property sets and attributes to IFC elements
- Read, query, and modify existing IFC files
- Export valid IFC files with automatic hierarchy completion
- Generate IFC geometry using the ShapeBuilder API

### 10.2 Architecture

#### Dependency Chain

```
IfcSverchok
  ├── Bonsai (BlenderBIM) — must be enabled first
  ├── Sverchok v1.2+ — node tree framework
  └── ifcopenshell — IFC file manipulation
```

#### SvIfcStore — Transient IFC File

The `SvIfcStore` class manages a single transient IFC file shared across all nodes in a node tree:

```python
class SvIfcStore:
    file: Union[ifcopenshell.file, None] = None
    id_map: dict[str, Any] = {}    # node_id -> created entity data
    schema_identifiers = ["IFC4", "IFC2X3"]
    use_bonsai_file = False

    @staticmethod
    def purge() -> None:
        """Reset all state — called before each full node tree update."""

    @staticmethod
    def get_file() -> ifcopenshell.file:
        """Get current file, or create boilerplate if none exists."""
        if SvIfcStore.use_bonsai_file:
            return tool.Ifc.get()  # Use Bonsai's active file
        if SvIfcStore.file is None:
            SvIfcStore.create_boilerplate()
        return SvIfcStore.file
```

#### SvIfcCore — Node Base Class

```python
class SvIfcCore:
    sv_input_names: list[str]

    def process(self) -> None:
        """Process double-nested inputs and call process_ifc()."""
        sv_inputs_nested = []
        for name in self.sv_input_names:
            sv_inputs_nested.append(self.inputs[name].sv_get())
        for sv_input_nested in zip_long_repeat(*sv_inputs_nested):
            for sv_input in zip_long_repeat(*sv_input_nested):
                sv_input = list(sv_input)
                self.process_ifc(*sv_input)
```

### 10.3 Complete Node Catalog (31 nodes)

#### Category: IFC (24 nodes)

| # | Node Class | Label | Description |
|---|-----------|-------|-------------|
| 1 | `SvIfcCreateFile` | IFC Create File | Creates a new empty IFC file with specified schema |
| 2 | `SvIfcReadFile` | IFC Read File | Opens an existing IFC file from disk |
| 3 | `SvIfcWriteFile` | IFC Write File | Writes transient IFC file to disk with auto hierarchy completion |
| 4 | `SvIfcCreateEntity` | IFC Create Entity | Creates IFC entities with geometry and placement |
| 5 | `SvIfcCreateShape` | IFC Create Blender Shape | Converts IFC entities to Blender mesh objects |
| 6 | `SvIfcReadEntity` | IFC Read Entity | Reads entity and exposes all attributes as outputs |
| 7 | `SvIfcPickIfcClass` | IFC Class Picker | UI picker for IFC classes by product category |
| 8 | `SvIfcById` | IFC By Id | Retrieves IFC entities by STEP file ID |
| 9 | `SvIfcByGuid` | IFC By Guid | Retrieves IFC entities by GlobalId |
| 10 | `SvIfcByType` | IFC By Type | Queries all entities of a given type |
| 11 | `SvIfcByQuery` | IFC By Query | Queries using IfcOpenShell selector syntax |
| 12 | `SvIfcAdd` | IFC Add | Adds an IFC entity to an IFC file |
| 13 | `SvIfcAddPset` | IFC Add Pset | Creates or edits property sets on elements |
| 14 | `SvIfcAddSpatialElement` | IFC Add Spatial Element | Creates spatial elements and assigns contained elements |
| 15 | `SvIfcRemove` | IFC Remove | Removes an entity from an IFC file |
| 16 | `SvIfcGenerateGuid` | IFC Generate Guid | Generates a new IFC-compliant GUID |
| 17 | `SvIfcGetProperty` | IFC Get Property | Retrieves property value from a property set |
| 18 | `SvIfcGetAttribute` | IFC Get Attribute | Retrieves a direct attribute value |
| 19 | `SvIfcSelectBlenderObjects` | IFC Select Blender Objects | Selects Blender objects by matching GlobalId |
| 20 | `SvIfcApi` | IFC API | Generic node calling any ifcopenshell.api function |
| 21 | `SvIfcBMeshToIfcRepr` | IFC BMesh to IFC Repr | Converts Blender objects to IFC representations |
| 22 | `SvIfcSverchokToIfcRepr` | IFC Sverchok to IFC Repr | Converts Sverchok geometry to IFC representations |
| 23 | `SvIfcCreateProject` | IFC Create Project | Adds IfcProject, units, and representation context |
| 24 | `SvIfcQuickProjectSetup` | IFC Quick Project Setup | Creates complete IFC file with project metadata |

#### Category: IFC Shape Builder (7 nodes)

| # | Node Class | Label | Description |
|---|-----------|-------|-------------|
| 1 | `SvIfcSbRectangle` | IFC Rectangle | Creates IFC rectangle profile via ShapeBuilder |
| 2 | `SvIfcSbExtrude` | IFC Extrude | Extrudes IFC profile along axis |
| 3 | `SvIfcSbRepresentation` | IFC Representation | Wraps items into IfcShapeRepresentation |
| 4 | `SvIfcSbTest` | IFC SB Test | Test/debug node |
| 5 | `SvSbShapeOutput` | IFC Shape Output | Converts IFC shape to Sverchok geometry |
| 6 | `SvSbMesh` | IFC Mesh | Creates IFC mesh from vertices and polygons |
| 7 | `SvSbPolyline` | IFC Polyline | Creates IFC polyline from vertices |

### 10.4 Two Geometry Modes

#### Mode 1: From Blender Objects (`SvIfcBMeshToIfcRepr`)

Takes Blender mesh objects as input. Separates loose parts, generates IFC representations per object, preserves world matrices.

**Inputs**: context_type (Model/Plan), context_identifier (Body/Annotation/Box/Axis), target_view, blender_objects
**Outputs**: Representations (IFC repr IDs), Locations (world matrices)

#### Mode 2: From Sverchok Geometry (`SvIfcSverchokToIfcRepr`)

Takes raw Sverchok geometry data (vertices, edges, faces). Normalizes to 4-deep nesting, zips geometry objects, creates IFC representations.

**Inputs**: context_type, context_identifier, target_view, Vertices, Edges, Faces
**Outputs**: Representation(s) (IFC repr IDs)

### 10.5 IFC File Generation Workflow

```
Step 1: Generate Geometry
[Sverchok generators or Blender objects]

Step 2: Convert to IFC Representation
[SvIfcSverchokToIfcRepr] or [SvIfcBMeshToIfcRepr]

Step 3: Create IFC Entities
[SvIfcCreateEntity] with IfcClass="IfcWall", Names="Wall_001"

Step 4: Build Spatial Hierarchy
[SvIfcAddSpatialElement] with IfcClass="IfcBuildingStorey"

Step 5: Add Properties (Optional)
[SvIfcAddPset] with Name="Pset_WallCommon", Properties='{"IsExternal": true}'

Step 6: Export
[SvIfcWriteFile] with path="output.ifc"
```

#### Automatic Hierarchy Completion

When writing an IFC file, `ensure_hirarchy()` automatically:
1. Creates a default `IfcBuilding` if none exists
2. Assigns orphaned spatial elements to the building
3. Assigns uncontained `IfcElement` instances to the building
4. Creates a default `IfcSite` if needed
5. Links Building -> Site -> Project

### 10.6 Working Example: Simple IFC Wall

```
[Number (0.3)] --> [IFC Rectangle] --> [IFC Extrude] --> [IFC Representation] --> [IFC Create Entity] --> [IFC Write File]
[Number (3.0)] --> /                   Magnitude=3.0                            IfcClass="IfcWall"     path="wall.ifc"
                                                                                Names="Simple Wall"
```

### 10.7 Integration with Bonsai

**IfcSverchok -> Bonsai**: The "Use Bonsai File" button sets `SvIfcStore.use_bonsai_file = True`, making `get_file()` return Bonsai's active IFC file.

**Bonsai -> IfcSverchok**: IFC files exported by IfcSverchok can be opened in Bonsai — they are standards-compliant with proper spatial hierarchy.

### 10.8 Known Issues and Limitations

- **Blender crashes during undo**: Especially in the node tree
- **Single transient file**: All nodes share one `SvIfcStore.file`, purged on "Re-run all nodes"
- **No type/material library**: No nodes for managing IFC type objects or materials
- **Limited geometry types**: Only basic shapes through Shape Builder
- **No infrastructure support**: IfcBridge, IfcRoad not implemented
- **No validation**: No MVD validation, no clash detection
- **No quantity takeoff**: No IfcElementQuantity or cost estimation nodes

---

## 11. Extensions Ecosystem (TopologicSverchok, Sverchok-Extra, etc.)

### 11.1 TopologicSverchok — Non-Manifold Topology

**Source**: `https://github.com/wassimj/TopologicSverchok`
**Author**: Wassim Jabi (Cardiff University / UCL)
**License**: AGPL-3.0
**Version**: 0.8.3.0
**Requirements**: Blender >= 3.4.1, Sverchok >= 1.2.0

TopologicSverchok integrates the Topologic non-manifold topology library with Sverchok. Non-manifold topology (NMT) allows entities with mixed dimensionalities to coexist — lines, surfaces, and volumes simultaneously.

#### Topological Class Hierarchy

| Class | Dimension | Description | AEC Example |
|-------|-----------|-------------|-------------|
| **Vertex** | 0D | Point in 3D space | Column insertion point |
| **Edge** | 1D | Defined by two vertices | Structural beam axis |
| **Wire** | 1D (composite) | Connected edges | Room boundary outline |
| **Face** | 2D | Region defined by closed wires | Wall surface, floor slab |
| **Shell** | 2D (composite) | Connected faces | Building envelope |
| **Cell** | 3D | Region defined by closed shells | Room volume |
| **CellComplex** | 3D (composite) | Cells connected by shared faces — **non-manifold** | Multi-room building |
| **Cluster** | Mixed | Collection of any entities | Building group |

#### Key Node Categories (200+ nodes)

- **Vertex**: VertexByCoordinates, VertexDistance, VertexEnclosingCell, VertexAdjacentEdges
- **Edge**: EdgeByVertices, EdgeLength, EdgeDirection, EdgeAdjacentEdges
- **Wire**: WireByEdges, WireCircle, WireRectangle, WireIsClosed
- **Face**: FaceByEdges, FaceArea, FaceTrimByWire, FaceAdjacentCells
- **Shell**: ShellByFaces, ShellByLoft, ShellArea, ShellVolume
- **Cell**: CellByFaces, CellVolume, CellInternalVertex, CellAdjacentCells, CellPrism, CellCylinder
- **CellComplex**: CellComplexByFaces, CellComplexDecompose, CellComplexExternalBoundary
- **Topology**: TopologyByGeometry, TopologyBoolean, TopologyTransform, TopologyExportToJSON
- **Graph**: GraphByTopology (rooms as nodes, walls as edges), GraphShortestPath
- **Optional Integration**: IFC nodes (ifcopenshell), Energy nodes (honeybee), Speckle nodes, Neo4j

#### AEC Relevance

1. **Building envelope analysis**: CellComplex -> query external faces
2. **Space adjacency**: `CellAdjacentCells` for room adjacency
3. **Dual graph**: `GraphByTopology` — rooms as vertices, shared walls as edges
4. **Energy simulation**: Convert topology to OpenStudio zones
5. **IFC integration**: Read IFC -> convert to topological representations

### 11.2 Sverchok-Extra — Advanced Geometry

**Source**: `https://github.com/portnov/sverchok-extra`
**Author**: Ilya Portnov
**Status**: Experimental

Extends Sverchok with advanced geometry nodes that require external dependencies. Functions as "a sandbox/nursery for new Sverchok nodes."

#### Node Categories

- **Surface Extra** (SciPy): Smooth Bivariate Spline, Implicit Surface, Curvature Lines
- **Field Extra** (SciPy): Vector Field Lines on Surface
- **Solid Extra**: Solid Waffle structures
- **Spatial Extra**: Delaunay 3D on Surface, Delaunay Mesh
- **SDF Primitives** (SDF library): Box, Cylinder, Sphere, Capsule, Torus + variants
- **SDF Operations** (SDF library): Boolean, Dilate/Erode, Shell, Twist, Bend, Extrude, Revolve
- **Data**: Spreadsheet, Data Item

Note: NURBS nodes originally in Sverchok-Extra have been migrated to core Sverchok.

### 11.3 Other Extensions

| Extension | Focus | Status | Dependencies |
|-----------|-------|--------|-------------|
| **Sverchok-Open3d** | Point clouds, meshes | Active | Open3D |
| **Mega-Polis** | Urban design, GIS | Experimental | GDAL, many optional |
| **Ladybug Tools** | Environmental analysis | Alpha | ladybug-tools |
| **Sverchok-Bmesh** | BMesh operations | Stable | (none) |

### 11.4 Extension Development Pattern

All Sverchok extensions follow the same registration pattern:

```python
# __init__.py
bl_info = {
    "name": "My Sverchok Extension",
    "author": "Your Name",
    "version": (0, 1, 0),
    "blender": (3, 6, 0),
    "category": "Node",
}

from sverchok.ui.nodeview_space_menu import add_node_menu

def nodes_index():
    return [
        ("My Category", [
            ("category_name.node_one", "SvMyNodeOne"),
            ("category_name.node_two", "SvMyNodeTwo"),
        ]),
    ]

def register():
    for module in make_node_list():
        module.register()
    add_node_menu.register()

def unregister():
    for module in reversed(make_node_list()):
        module.unregister()
```

#### Extension Structure

```
my_sverchok_extension/
├── __init__.py              # Registration, bl_info, nodes_index()
├── nodes/
│   └── category_name/
│       ├── __init__.py
│       ├── node_one.py
│       └── node_two.py
└── (optional) icons/, sockets/, settings/
```

#### Node Registration Requirements

Sverchok nodes MUST:
1. Inherit from `SverchCustomTreeNode` (Sverchok mixin)
2. Inherit from `bpy.types.Node` (standard Blender node)
3. Define `bl_idname` (unique string, convention: `Sv` prefix)
4. Define `bl_label` (display name)
5. Implement `sv_init()` (socket creation)
6. Implement `process()` (main computation)
7. Use `updateNode` callback for properties

---

## 12. Common Error Patterns

### 12.1 Nesting Level Errors

**Problem**: Wrong nesting level causes silent data corruption or crashes.

```python
# WRONG: Flat vertex list
self.outputs['Vertices'].sv_set([(0,0,0), (1,0,0)])

# CORRECT: Object-wrapped vertex list
self.outputs['Vertices'].sv_set([[(0,0,0), (1,0,0)]])
```

**Rule**: ALWAYS wrap output data in the object-level list. Even a single object must be `[[...]]`.

### 12.2 Missing `updateNode` Callback

**Problem**: Property changes don't trigger node re-evaluation.

```python
# WRONG: No update callback
my_prop: FloatProperty(name='Scale', default=1.0)

# CORRECT: With updateNode callback
my_prop: FloatProperty(name='Scale', default=1.0, update=updateNode)
```

### 12.3 Not Checking Output Connections

**Problem**: Unnecessary computation when outputs are not connected.

```python
# WRONG: Always processes, even when nothing uses the output
def process(self):
    data = expensive_computation()
    self.outputs['Result'].sv_set(data)

# CORRECT: Early exit when no output is connected
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return
    data = expensive_computation()
    self.outputs['Result'].sv_set(data)
```

### 12.4 Socket Data Mutation

**Problem**: Modifying input data in-place affects upstream nodes.

```python
# WRONG: Mutating sv_get() data
data = self.inputs['Data'].sv_get()
data[0].append(999)  # This modifies the cached data!

# CORRECT: Use deepcopy=True (default) or work on copies
data = self.inputs['Data'].sv_get(deepcopy=True)
data[0].append(999)  # Safe — this is a copy
```

### 12.5 List Matching Misunderstandings

**Problem**: Expecting automatic matching without calling `match_long_repeat`.

```python
# WRONG: Direct zip (truncates to shortest)
for v, s in zip(verts, scale):
    ...

# CORRECT: Use match_long_repeat first
verts, scale = match_long_repeat([verts, scale])
for v, s in zip(verts, scale):
    ...
```

### 12.6 IfcSverchok: Undo Crashes

**Problem**: Blender crashes when undoing operations in IfcSverchok node trees.

**Workaround**: Save frequently. Avoid undo in IfcSverchok trees. Use the "Re-run all nodes" button instead.

### 12.7 IfcSverchok: Purge on Re-run

**Problem**: `SvIfcStore.purge()` is called before every full tree update, clearing all entity data.

**Consequence**: IFC entities are recreated from scratch on every update. IDs change between runs.

---

## 13. AI Common Mistakes

### 13.1 NEVER Output Flat Data

```python
# AI MISTAKE: Returning flat list instead of nested
result = [1, 2, 3]
self.outputs['Numbers'].sv_set(result)  # WRONG

# CORRECT: Always wrap in object level
self.outputs['Numbers'].sv_set([[1, 2, 3]])
```

### 13.2 NEVER Forget the Object Level for Vertices

```python
# AI MISTAKE: Missing object wrapper
verts = [(0,0,0), (1,0,0), (1,1,0)]
self.outputs['Vertices'].sv_set(verts)  # WRONG — level 2

# CORRECT: Vertices ALWAYS at level 3
self.outputs['Vertices'].sv_set([[(0,0,0), (1,0,0), (1,1,0)]])
```

### 13.3 NEVER Use `sv_init` Without `self`

```python
# AI MISTAKE: Using __init__ instead of sv_init
def __init__(self):  # WRONG — this is NOT how Sverchok nodes initialize
    self.inputs.new(...)

# CORRECT: Use sv_init
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Data')
```

### 13.4 NEVER Confuse Socket Type Aliases

```python
# AI MISTAKE: Wrong SNLite socket type
"""
in  verts  vertices  # WRONG — not a valid type
out result string    # WRONG — not a valid type
"""

# CORRECT: Use single-letter aliases
"""
in  verts  v    # SvVerticesSocket
out result s    # SvStringsSocket
"""
```

### 13.5 ALWAYS Use `updateNode` for Properties

```python
# AI MISTAKE: Forgetting updateNode
count: IntProperty(name='Count', default=5)  # WRONG — changes won't trigger updates

# CORRECT
count: IntProperty(name='Count', default=5, update=updateNode)
```

### 13.6 ALWAYS Match Lists Before Zipping

```python
# AI MISTAKE: Assuming inputs are always same length
for v, s in zip(verts, scale):  # WRONG — silently truncates

# CORRECT
verts, scale = match_long_repeat([verts, scale])
for v, s in zip(verts, scale):
    ...
```

### 13.7 NEVER Create Sockets in `process()`

```python
# AI MISTAKE: Creating sockets during processing
def process(self):
    self.inputs.new('SvStringsSocket', 'Extra')  # WRONG — creates new socket every update

# CORRECT: Create sockets only in sv_init() or sv_update()
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Extra')
```

### 13.8 NEVER Assume Matrix Data Is Nested

```python
# AI MISTAKE: Treating matrices like vertices (level 3)
matrices = [[Matrix(), Matrix()]]  # WRONG

# CORRECT: Matrices are at level 1
matrices = [Matrix(), Matrix()]
```

### 13.9 IfcSverchok: NEVER Assume Entity IDs Persist

```python
# AI MISTAKE: Storing IFC entity IDs between runs
saved_id = 42  # WRONG — IDs change on every re-run because SvIfcStore.purge() resets everything

# CORRECT: Query by GUID or type, not by STEP ID
```

### 13.10 IfcSverchok: ALWAYS Use Double-Nested Lists

```python
# AI MISTAKE: Single-nested data to IFC nodes
self.outputs['Names'].sv_set(["Wall_001"])  # WRONG

# CORRECT: IfcSverchok expects Sverchok standard nesting
self.outputs['Names'].sv_set([["Wall_001"]])
```

---

## 14. Real-World Usage Patterns

### 14.1 Parametric Building Elements

Create parametric building elements using Sverchok's generator nodes, then export to IFC:

```
[Profile Node (I-Beam)] --> [Extrude] --> [SvIfcSverchokToIfcRepr] --> [SvIfcCreateEntity]
                            Height=3m                                   IfcClass="IfcColumn"
```

### 14.2 Batch Element Generation

Generate arrays of elements using Sverchok's list processing:

```
[Grid of Points] --> [Matrix Apply] --> [SvIfcSverchokToIfcRepr] --> [SvIfcCreateEntity]
                     transforms                                      IfcClass="IfcColumn"
                     per column                                      Names=[list of names]
```

The IfcSverchok parametric facade example creates 252 columns, 9 floor slabs, and 353 facade slats from a single compact node tree.

### 14.3 Working with Matrices

```python
from mathutils import Matrix, Vector, Euler
import math

# Create transformation matrix
location = Vector((1, 2, 3))
rotation = Euler((0, 0, math.radians(45)), 'XYZ')
scale = Vector((2, 2, 2))

mat_loc = Matrix.Translation(location)
mat_rot = rotation.to_matrix().to_4x4()
mat_sca = Matrix.Diagonal(scale).to_4x4()

# Combined transform (order: scale, then rotate, then translate)
transform = mat_loc @ mat_rot @ mat_sca

# Apply to vertices
verts = self.inputs['Vertices'].sv_get()
result = []
for v_list in verts:
    transformed = [tuple(transform @ Vector(v)) for v in v_list]
    result.append(transformed)
self.outputs['Vertices'].sv_set(result)
```

### 14.4 Blender Object Integration

**Reading from Blender objects:**
```python
def process(self):
    objects = self.inputs['Objects'].sv_get()
    for obj_list in objects:
        for obj in obj_list:
            mesh = obj.data
            verts = [list(v.co) for v in mesh.vertices]
            edges = [list(e.vertices) for e in mesh.edges]
            faces = [list(f.vertices) for f in mesh.polygons]
            self.outputs['Vertices'].sv_set([verts])
```

**Writing back to Blender objects:**
```python
import bpy, bmesh

def process(self):
    verts = self.inputs['Vertices'].sv_get()
    faces = self.inputs['Faces'].sv_get()

    for idx, (v_list, f_list) in enumerate(zip(verts, faces)):
        name = f"SV_Object_{idx}"
        if name in bpy.data.meshes:
            mesh = bpy.data.meshes[name]
        else:
            mesh = bpy.data.meshes.new(name)

        bm = bmesh.new()
        bm_verts = [bm.verts.new(v) for v in v_list]
        bm.verts.ensure_lookup_table()
        for f in f_list:
            bm.faces.new([bm_verts[i] for i in f])
        bm.to_mesh(mesh)
        bm.free()

        if name not in bpy.data.objects:
            obj = bpy.data.objects.new(name, mesh)
            bpy.context.collection.objects.link(obj)
```

### 14.5 TopologicSverchok for Building Analysis

```
[IFC model] --> [TopologyByGeometry] --> [CellComplexByFaces] --> [CellAdjacentCells]
                                                                   (space adjacency)
                                                               --> [GraphByTopology]
                                                                   (dual graph for wayfinding)
                                                               --> [CellComplexExternalBoundary]
                                                                   (building envelope)
```

### 14.6 vectorize Utility in SNLite

```python
"""
in  radius  s  default=1.0
in  height  s  default=2.0
out volume  s
"""
import math

def calc_volume(r, h):
    return math.pi * r * r * h

volume = vectorize(calc_volume)(radius, height)
```

The `vectorize` function handles nested list matching automatically, applying the function element-wise across the data structure.

### 14.7 Performance Optimization: Pre-compute with setup()

```python
"""
in  count s  default=100  nested=2
out verts v
"""
def setup():
    import numpy as np
    angles = np.linspace(0, 2 * np.pi, 360)
    sin_table = np.sin(angles)
    cos_table = np.cos(angles)
    return locals()

n = int(count)
indices = np.linspace(0, 359, n, dtype=int)
x = cos_table[indices]
y = sin_table[indices]
z = np.zeros(n)
verts = [np.column_stack([x, y, z]).tolist()]
```

---

## 15. Advanced Data Types

### 15.1 Curves

Curves are instances of `SvCurve` (abstract base class from `utils/curve/__init__.py`). A curve maps parameter t to a 3D point.

Properties:
- **Domain**: Range `[t_min, t_max]` of valid parameter values
- **Parametrization**: Not necessarily by arc length
- Supports point evaluation, tangent/derivative evaluation, reparametrization

### 15.2 Surfaces

Surfaces are instances of `SvSurface` (from `utils/surface/__init__.py`). A surface maps (u, v) to 3D points.

Properties:
- **Domain**: Rectangle `[u_min, u_max] x [v_min, v_max]`
- Can be closed in u, v, or both directions

### 15.3 Scalar Fields

Instances of `SvScalarField` (from `utils/field/scalar.py`). Maps R^3 to R.

Operations: addition, multiplication, composition, gradient (produces vector field).

### 15.4 Vector Fields

Instances of `SvVectorField` (from `utils/field/vector.py`). Maps R^3 to R^3.

Two interpretations:
1. **Relative/Bound**: Vector starts at point P, ends at P + field(P) — used by "Apply Vector Field"
2. **Absolute/Free**: Vector starts at origin, ends at field(P) — used by "Evaluate Vector Field"

### 15.5 Solids

Depend on FreeCAD/OpenCascade (`Part.Shape` objects). BRep (Boundary Representation) objects.

### 15.6 Dictionaries

`SvDictionarySocket` carries Python `dict` objects. Special support:
- `unzip_dict_recursive(data)` — converts nested list of dicts into dict of nested lists

---

## Sources

| Source | URL | Content |
|--------|-----|---------|
| Sverchok repository | `https://github.com/nortikin/sverchok` | Full source code analysis |
| IfcSverchok source | `https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok` | Full source code analysis |
| TopologicSverchok | `https://github.com/wassimj/TopologicSverchok` | Repository analysis |
| Topologic docs | `https://topologic.app/software/` | Topology class definitions |
| Sverchok-Extra | `https://github.com/portnov/sverchok-extra` | Repository analysis |
| Sverchok extensions | `https://nortikin.github.io/sverchok/docs/introduction/sverchok_extensions.html` | Extension listing |
| Sverchok-Open3d | `https://github.com/vicdoval/sverchok-open3d` | Registration pattern |
| Mega-Polis | `https://github.com/victorcalixto/mega-polis` | Urban design toolkit |
| GSoC 2022 proposal | `https://mdjska.github.io/GSoC/notes/GSoC_proposal_mdj/` | IfcSverchok expansion goals |
| OSArch Sverchok-IFC | `https://community.osarch.org/discussion/284/sverchok-ifc` | Community discussion |

---

*Document generated by sv-merge-vooronderzoek agent. All class names, method signatures, and data structures verified against source repositories. Date: 2026-03-07.*
