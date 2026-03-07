# sverchok-syntax-api: API Method Reference

Sources: https://github.com/nortikin/sverchok (v1.4.0+),
vooronderzoek-sverchok.md §9

---

## Tree Access — bpy.data.node_groups

Sverchok trees are Blender node groups accessible via `bpy.data.node_groups` (see **blender-core-api** skill for general `bpy.data` patterns).

```python
# Create a new Sverchok tree
tree = bpy.data.node_groups.new(name: str, 'SverchCustomTreeType') -> SverchCustomTree

# Access an existing tree
tree = bpy.data.node_groups.get(name: str) -> SverchCustomTree | None

# Filter all Sverchok trees
sv_trees = [ng for ng in bpy.data.node_groups if ng.bl_idname == 'SverchCustomTreeType']

# Remove a tree
bpy.data.node_groups.remove(tree)
```

---

## SverchCustomTree — Tree Methods

**Source**: `node_tree.py`
**bl_idname**: `'SverchCustomTreeType'`

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `sv_process` | `BoolProperty` | `True` | Enable/disable tree processing |
| `sv_animate` | `BoolProperty` | `False` | Process on frame change |
| `sv_show` | `BoolProperty` | `True` | Show viewer outputs in viewport |
| `sv_draft` | `BoolProperty` | `False` | Draft mode (reduced quality for speed) |
| `sv_scene_update` | `BoolProperty` | `False` | React to scene changes |
| `tree_id` | `StringProperty` | (auto) | Unique identifier, persists across file operations |

### force_update()

```python
def force_update(self):
    """Force complete tree recalculation.
    Dispatches a ForceEvent to control_center.
    Resets all node states and re-evaluates entire tree.

    ALWAYS call after programmatic property changes.
    ALWAYS call after tree construction (outside init_tree block).
    """
```

### update_nodes(nodes)

```python
def update_nodes(self, nodes: list):
    """Update specific nodes and their downstream dependents.
    Parameters:
        nodes: list[Node] — nodes to mark as outdated
    Dispatches a PropertyEvent for each node.
    More efficient than force_update() when only some nodes changed.
    """
```

### init_tree()

```python
@contextmanager
def init_tree(self):
    """Context manager to suppress updates during tree construction.

    ALWAYS use when programmatically adding nodes/links.
    Without this, every node/link addition triggers a full TreeEvent.
    For N additions, this avoids O(N²) event processing.

    Usage:
        with tree.init_tree():
            node_a = tree.nodes.new('SvBoxNodeMk2')
            node_b = tree.nodes.new('SvViewerDrawMk4')
            tree.links.new(node_a.outputs[0], node_b.inputs[0])
        # Single update on exit
    """
```

### scene_update()

```python
def scene_update(self):
    """Called by scene change handlers.
    Updates nodes with is_interactive enabled.
    Only fires when tree.sv_scene_update is True."""
```

### process_ani(frame_changed, animation_playing)

```python
def process_ani(self, frame_changed: bool, animation_playing: bool):
    """Processes tree for animation events.
    Only fires when tree.sv_animate is True.
    Parameters:
        frame_changed: bool — whether the current frame changed
        animation_playing: bool — whether animation is actively playing
    """
```

### turn_off_ng(context)

```python
def turn_off_ng(self, context):
    """Toggle viewport visibility for all viewer nodes.
    Calls show_viewport() on each viewer node."""
```

---

## Node Collection — tree.nodes

```python
# Add a node by bl_idname
node = tree.nodes.new(type: str) -> Node
# Example: tree.nodes.new('SvBoxNodeMk2')

# Access by name
node = tree.nodes.get(name: str) -> Node | None
node = tree.nodes[name: str] -> Node  # raises KeyError if not found

# Remove a node
tree.nodes.remove(node)

# Iterate all nodes
for node in tree.nodes:
    print(node.bl_idname, node.name)
```

### Node Common Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `node.bl_idname` | `str` | Node type identifier (e.g., `'SvBoxNodeMk2'`) |
| `node.name` | `str` | User-visible name (auto-generated, editable) |
| `node.location` | `(float, float)` | Position in node editor |
| `node.inputs` | `NodeInputs` | Input socket collection |
| `node.outputs` | `NodeOutputs` | Output socket collection |

### Node Execution Statistics

Set by the update system after each `process()` call:

| Attribute | Type | Description |
|-----------|------|-------------|
| `node.US_is_updated` | `bool` | Whether node executed successfully |
| `node.US_error` | `str` | Error message if execution failed |
| `node.US_error_stack` | `str` | Full traceback string |
| `node.US_time` | `float` | Execution time in seconds |
| `node.US_warning` | `str` | Warning messages from logging |

---

## Link Collection — tree.links

```python
# Create a link from output to input socket
link = tree.links.new(output_socket, input_socket) -> NodeLink

# By index:
tree.links.new(node_a.outputs[0], node_b.inputs[0])

# By name:
tree.links.new(node_a.outputs['Vertices'], node_b.inputs['vertices'])

# Remove a link
tree.links.remove(link)

# Iterate all links
for link in tree.links:
    print(f"{link.from_node.name}.{link.from_socket.name} -> "
          f"{link.to_node.name}.{link.to_socket.name}")
```

---

## Socket Data Methods

### sv_get(default=None, deepcopy=True)

```python
socket.sv_get(default=None, deepcopy=True) -> list:
    """Read data from the socket cache.
    Parameters:
        default: value returned if socket has no data and no links.
                 If None and no data exists, raises SvNoDataError.
        deepcopy: bool — if True (default), returns a deep copy.
                  Set to False ONLY when you will NOT mutate the data.
    Returns:
        list — nested list data: [[obj1_data], [obj2_data], ...]
    Raises:
        SvNoDataError — if no data and no default provided
    """
```

### sv_set(data)

```python
socket.sv_set(data: list):
    """Write data to the socket cache.
    Parameters:
        data: list — MUST be a nested list: [[obj1_data], [obj2_data], ...]
    ALWAYS wrap data in outer list representing objects.
    """
```

### sv_forget()

```python
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

## Data Structure Utilities — sverchok.data_structure

### updateNode(self, context)

```python
def updateNode(self, context):
    """Standard update callback for bpy.props.
    ALWAYS use as: update=updateNode on property definitions.
    Dispatches a PropertyEvent for the node to the update system.

    Usage:
        my_prop: bpy.props.FloatProperty(update=updateNode)
    """
```

### match_long_repeat(list_of_lists)

```python
def match_long_repeat(list_of_lists: list[list]) -> list[list]:
    """Match list lengths by repeating the last element of shorter lists.
    Parameters:
        list_of_lists: list[list] — lists to match
    Returns:
        list[list] — all lists matched to length of the longest

    Example:
        match_long_repeat([[1, 2, 3], [10]]) -> [[1, 2, 3], [10, 10, 10]]
    """
```

### fullList(lst, count)

```python
def fullList(lst: list, count: int):
    """Extend list in-place to target length by repeating last element.
    Parameters:
        lst: list — list to extend (MODIFIED IN-PLACE)
        count: int — target minimum length

    Example:
        lst = [1.0, 2.0]
        fullList(lst, 5)
        # lst is now [1.0, 2.0, 2.0, 2.0, 2.0]
    """
```

### repeat_last(lst)

```python
def repeat_last(lst: list) -> Iterator:
    """Infinite iterator that yields list items, then repeats the last element.
    Parameters:
        lst: list — source data
    Returns:
        Iterator — yields each item, then repeats last element forever

    Example:
        gen = repeat_last([10, 20, 30])
        [next(gen) for _ in range(6)]  # [10, 20, 30, 30, 30, 30]
    """
```

### get_data_nesting_level(data, data_types, search_first_data)

```python
def get_data_nesting_level(data, data_types=SIMPLE_DATA_TYPES, search_first_data=False) -> int:
    """Determine nesting depth of a data structure.
    Parameters:
        data: the data to analyze
        data_types: tuple of types considered as leaf values
        search_first_data: if True, search only the first branch
    Returns:
        int — nesting depth
    """
```

### flat_iter(data)

```python
def flat_iter(data) -> Iterator:
    """Deep flatten nested structures into a single sequence.
    Example:
        list(flat_iter([1, [2, 3, [4]], 5]))  # [1, 2, 3, 4, 5]
    """
```

### fixed_iter(data, iter_number, fill_value)

```python
def fixed_iter(data, iter_number: int, fill_value=0) -> Iterator:
    """Iterator yielding exactly iter_number items.
    Cycles last element if data is exhausted.
    Parameters:
        data: source data
        iter_number: exact number of items to yield
        fill_value: fallback if data is empty
    """
```

### get_edge_loop(n)

```python
def get_edge_loop(n: int) -> list[tuple[int, int]]:
    """Generate cyclic edge connectivity for n vertices.
    Returns:
        list of (i, j) tuples forming a closed loop
    Example:
        get_edge_loop(4)  # [(0,1), (1,2), (2,3), (3,0)]
    """
```

### list_match_func

```python
list_match_func: dict[str, Callable] = {
    "SHORT":  match_short,        # Truncate to shortest
    "CYCLE":  match_long_cycle,   # Cycle shorter lists
    "REPEAT": match_long_repeat,  # Repeat last element
    "XREF":   match_cross,        # Cross-reference (all combinations)
    "XREF2":  match_cross2,       # Cross-reference variant
}
```

### Other Utilities

```python
def changable_sockets(node, inputsocketname: str, outputsocketname: str):
    """Dynamically match output socket type to connected input socket type."""

def replace_socket(socket, new_type: str, new_name: str = None, new_pos: int = None):
    """Replace a socket with a different type, preserving existing connections."""

def multi_socket(node, min: int = 1, start: int = 0, breck: bool = False, out_count: int = None):
    """Manage dynamic multi-socket creation on a node."""

def get_other_socket(socket) -> NodeSocket | None:
    """Get the socket connected to this one, following through reroute nodes."""

def enum_item_4(s: list) -> list:
    """Format a list as enum items usable in bpy.props.EnumProperty."""
```
