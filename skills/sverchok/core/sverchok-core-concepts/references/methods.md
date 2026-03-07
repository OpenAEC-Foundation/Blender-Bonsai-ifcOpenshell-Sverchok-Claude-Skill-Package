# sverchok-core-concepts: API Method Reference

Sources: https://github.com/nortikin/sverchok (v1.4.0+),
vooronderzoek-sverchok.md §1, §2

---

## SverchCustomTree — Node Tree Class

**Source**: `node_tree.py`
**bl_idname**: `'SverchCustomTreeType'`

```python
class SverchCustomTree(NodeTree, SvNodeTreeCommon):
    bl_idname = 'SverchCustomTreeType'
    bl_label = 'Sverchok Nodes'
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `sv_process` | `BoolProperty` | `True` | Enable/disable tree processing |
| `sv_animate` | `BoolProperty` | `False` | Process on frame change |
| `sv_show` | `BoolProperty` | `True` | Show viewer outputs in viewport |
| `sv_draft` | `BoolProperty` | `False` | Draft mode (reduced quality/speed) |
| `sv_scene_update` | `BoolProperty` | `False` | React to scene changes |

### Methods

```python
def force_update(self):
    """Force complete tree recalculation.
    Dispatches a ForceEvent to control_center.
    Resets all node states and re-evaluates entire tree."""

def update_nodes(self, nodes):
    """Update specific nodes and their downstream dependents.
    Parameters:
        nodes: list[Node] — nodes to mark as outdated
    Dispatches PropertyEvent for each node."""

@contextmanager
def init_tree(self):
    """Context manager to suppress updates during tree construction.
    ALWAYS use when programmatically adding nodes/links.
    Without this, every node/link addition triggers a full tree update.
    Usage:
        with tree.init_tree():
            node_a = tree.nodes.new('SvBoxNodeMk2')
            node_b = tree.nodes.new('SvViewerDrawMk4')
            tree.links.new(node_a.outputs[0], node_b.inputs[0])
    """
```

---

## SverchCustomTreeNode — Node Base Class

**Source**: `node_tree.py`

```python
class SverchCustomTreeNode(UpdateNodes, NodeUtils, NodeDependencies, NodeDocumentation):
    """Base class for all Sverchok nodes."""
    sv_category = ''  # Category name for Shift+A menu
```

### Mixins

| Mixin | Source | Purpose |
|-------|--------|---------|
| `UpdateNodes` | `node_tree.py` | Node lifecycle management |
| `NodeUtils` | `node_tree.py` | Logger shortcuts, UI operator tracking |
| `NodeDependencies` | `node_tree.py` | Optional library dependency checking |
| `NodeDocumentation` | `node_tree.py` | Docstring parsing, help link generation |

### UpdateNodes Mixin — Lifecycle Methods

```python
def sv_init(self, context):
    """Called once when node is created.
    Override to define sockets and initialize properties.
    Parameters:
        context: bpy.types.Context — Blender context"""

def sv_update(self):
    """Called when node topology changes (links added/removed).
    Override for dynamic socket management.
    NEVER put computation logic here — use process() instead."""

def sv_copy(self, original):
    """Called when node is duplicated.
    Override to handle deep copy of custom data.
    Parameters:
        original: Node — the node being copied from"""

def sv_free(self):
    """Called when node is deleted.
    Override to clean up external resources."""

def process(self):
    """Main computation method. Called by the update system.
    NEVER call this directly — ALWAYS let the update system invoke it.
    Override to implement node logic:
        1. Read inputs via self.inputs[name].sv_get()
        2. Compute results
        3. Write outputs via self.outputs[name].sv_set(data)"""
```

### Node Execution Statistics Attributes

Set by the update system after each `process()` call:

| Attribute | Type | Description |
|-----------|------|-------------|
| `US_is_updated` | `bool` | Whether node executed successfully |
| `US_error` | `str` | Error message string if execution failed |
| `US_error_stack` | `str` | Full traceback string |
| `US_time` | `float` | Execution time in seconds (via `perf_counter`) |
| `US_warning` | `str` | Warning messages from logging |

---

## Socket Data Cache — core/socket_data.py

### Global Cache

```python
socket_data_cache: dict[SockId, list] = dict()
# SockId = NewType('SockId', str)
# SockId format: hash of node.node_id + socket.identifier + direction
```

### sv_set_socket(socket, data)

```python
def sv_set_socket(socket, data):
    """Sets socket data in cache.
    Parameters:
        socket: NodeSocket — the socket to store data for
        data: list — the data to store (MUST be a list)
    Stores data in socket_data_cache[socket.socket_id]."""
```

### sv_get_socket(socket, deepcopy=True)

```python
def sv_get_socket(socket, deepcopy=True):
    """Gets socket data from cache.
    Parameters:
        socket: NodeSocket — the socket to retrieve data from
        deepcopy: bool — if True, returns a deep copy (default True)
    Returns:
        list — the cached data (deep copied if deepcopy=True)
    Raises:
        SvNoDataError — if no data exists for this socket

    Set deepcopy=False ONLY when the node will NOT mutate input data.
    This provides a significant performance gain for read-only operations."""
```

### sv_deep_copy(lst)

```python
def sv_deep_copy(lst):
    """Custom deep copy optimized for Sverchok's nested list structures.
    Faster than Python's copy.deepcopy for typical Sverchok data.

    Strategy:
    - Flat lists (no nested lists): shallow copy via lst[:]
    - Nested lists: recursive deep copy
    - Atoms (non-list/tuple): returned as-is"""
```

### sv_forget_socket(socket)

```python
def sv_forget_socket(socket):
    """Removes socket data from cache.
    Called when upstream data is no longer available (SvNoDataError)."""
```

---

## Socket Classes — core/sockets.py

### Socket sv_get / sv_set Methods

```python
# On any Sverchok socket instance:

socket.sv_get(default=None, deepcopy=True):
    """Get data from this socket.
    Parameters:
        default: value returned if socket has no data and no links
        deepcopy: bool — deep copy the cached data (default True)
    Returns:
        list — socket data in nested list format
    Raises:
        SvNoDataError — if no data and no default provided"""

socket.sv_set(data):
    """Set data on this socket.
    Parameters:
        data: list — data in nested list format"""

socket.sv_forget():
    """Clear cached data for this socket."""
```

### Common Socket Types

| Socket Class | bl_idname | Data Type |
|-------------|-----------|-----------|
| `SvStringsSocket` | `'SvStringsSocket'` | Numbers, strings, generic lists |
| `SvVerticesSocket` | `'SvVerticesSocket'` | Vertex coordinates [(x,y,z), ...] |
| `SvMatrixSocket` | `'SvMatrixSocket'` | 4x4 transformation matrices |
| `SvColorSocket` | `'SvColorSocket'` | RGBA color values |
| `SvObjectSocket` | `'SvObjectSocket'` | Blender object references |
| `SvSurfaceSocket` | `'SvSurfaceSocket'` | NURBS/surface objects |
| `SvCurveSocket` | `'SvCurveSocket'` | Curve objects |
| `SvSolidSocket` | `'SvSolidSocket'` | Solid (FreeCAD) objects |
| `SvDictionarySocket` | `'SvDictionarySocket'` | Python dictionaries |

---

## SearchTree — core/update_system.py

### Graph Structure

```python
class SearchTree:
    _from_nodes: dict[Node, set[Node]]        # predecessors
    _to_nodes: dict[Node, set[Node]]          # successors
    _from_sock: dict[NodeSocket, NodeSocket]   # input -> connected output
    _to_socks: dict[NodeSocket, set[NodeSocket]]  # output -> connected inputs
```

### Methods

```python
def sort_nodes(self, nodes):
    """Topological sort using graphlib.TopologicalSorter.
    Parameters:
        nodes: iterable[Node] — nodes to sort
    Returns:
        list[Node] — nodes in execution order"""

def nodes_from(self, from_nodes):
    """BFS forward traversal from given nodes.
    Returns all downstream nodes that would be affected."""

def nodes_to(self, to_nodes):
    """BFS backward traversal to given nodes.
    Returns all upstream nodes that feed into the targets."""

def previous_sockets(self, node):
    """Get output sockets connected to this node's inputs.
    Returns:
        list[NodeSocket | None] — one entry per input socket,
        None for unconnected inputs"""

def update_node(self, node, suppress=True):
    """Execute a single node with error handling.
    Parameters:
        node: Node — the node to execute
        suppress: bool — if True, catch and record errors instead of raising
    Records US_time, US_error, US_is_updated on the node."""
```

### Special Node Handling

SearchTree constructor handles three special node types by short-circuiting:

| Node Type | Behavior |
|-----------|----------|
| `NodeReroute` | Transparent pass-through; removed from graph, links reconnected |
| `WifiInNode`/`WifiOutNode` | Wireless link pairs; resolved to direct connections |
| Muted nodes (`is_active_output == False`) | Bypassed using `sv_internal_links` mapping |

---

## Update System — core/update_system.py

### control_center()

```python
def control_center(event):
    """Main event dispatcher for the update system.
    Parameters:
        event: one of TreeEvent, PropertyEvent, AnimationEvent,
               SceneEvent, ForceEvent, GroupTreeEvent, FileEvent, UndoEvent

    Flow:
    1. Receives event from Blender callback or user action
    2. Creates a Task object
    3. Task is processed by a timer
    4. UpdateTree._walk() yields nodes in topological order
    5. SearchTree.update_node() executes each dirty node"""
```

### Event Classes — core/events.py

```python
class TreeEvent:
    """Node or link addition/removal.
    Triggers full topology re-evaluation."""

class ForceEvent:
    """Full tree recalculation requested.
    Resets all node states before updating."""

class AnimationEvent:
    """Frame change or animation playback.
    Only triggers if tree.sv_animate is True."""

class SceneEvent:
    """Scene modification detected.
    Only triggers if tree.sv_scene_update is True."""

class PropertyEvent:
    """Node property changed via updateNode callback.
    Marks specific node and its downstream as outdated."""

class GroupTreeEvent:
    """Change inside a group tree (node group).
    Propagates to parent tree."""

class FileEvent:
    """New file loaded.
    Triggers full tree reload."""

class UndoEvent:
    """Undo operation executed.
    Restores tree state from undo history."""
```

---

## Data Propagation — core/update_system.py

### prepare_input_data(prev_socks, input_socks)

```python
def prepare_input_data(prev_socks, input_socks):
    """Reads data from output sockets, converts if needed, writes to input sockets.
    Parameters:
        prev_socks: list[NodeSocket | None] — upstream output sockets
        input_socks: list[NodeSocket] — downstream input sockets

    For each socket pair:
    1. Reads data via ps.sv_get()
    2. If socket types differ, applies implicit conversion
    3. Writes data via ns.sv_set(data)
    4. If SvNoDataError, clears the input socket via ns.sv_forget()

    Raises:
        ImplicitConversionProhibited — if type conversion is not allowed
        (also marks the link as invalid via link.is_valid = False)"""
```

---

## Data Structure Utilities — data_structure.py

### updateNode(self, context)

```python
def updateNode(self, context):
    """Standard update callback for bpy.props.
    ALWAYS use as: update=updateNode on property definitions.
    Dispatches a PropertyEvent for the node."""
```

### match_long_repeat(list_of_lists)

```python
def match_long_repeat(list_of_lists):
    """Match list lengths by repeating the last element of shorter lists.
    Parameters:
        list_of_lists: list[list] — lists to match
    Returns:
        list[list] — all lists matched to the length of the longest

    Example:
        match_long_repeat([[1, 2, 3], [10]]) -> [[1, 2, 3], [10, 10, 10]]"""
```

### fullList(lst, count)

```python
def fullList(lst, count):
    """Extend list to count by repeating last element in-place.
    Parameters:
        lst: list — list to extend (modified in-place)
        count: int — target length"""
```
