# blender-syntax-nodes: API Method Reference

Sources:
- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://docs.blender.org/api/current/bpy.types.Node.html
- https://docs.blender.org/api/current/bpy.types.NodeSocket.html
- https://docs.blender.org/api/current/bpy.types.NodeLink.html
- https://docs.blender.org/api/current/bpy.types.NodeGroup.html

---

## bpy.types.NodeTree (inherits from ID)

NodeTree is the container for all node-based editors. Stores nodes, links, and (in 4.0+) the interface definition.

### Properties

| Property | Type | Description | Version |
|----------|------|-------------|---------|
| `nodes` | `Nodes` (bpy_prop_collection of `Node`) | All nodes in the tree | 3.x/4.x/5.x |
| `links` | `NodeLinks` (bpy_prop_collection of `NodeLink`) | All links connecting sockets | 3.x/4.x/5.x |
| `interface` | `NodeTreeInterface` | Group socket interface definition | 4.0+ only |
| `inputs` | `NodeTreeInputs` | Legacy group inputs | 3.x only (removed 4.0) |
| `outputs` | `NodeTreeOutputs` | Legacy group outputs | 3.x only (removed 4.0) |
| `type` | `str` (enum) | Tree type: `'SHADER'`, `'COMPOSITING'`, `'GEOMETRY'`, `'TEXTURE'` | 3.x/4.x/5.x |
| `bl_idname` | `str` | Internal type identifier | 3.x/4.x/5.x |
| `bl_label` | `str` | Display name | 3.x/4.x/5.x |
| `animation_data` | `AnimData` or `None` | Animation data | 3.x/4.x/5.x |

### Nodes Collection Methods

```python
# Blender 3.x/4.x/5.x
tree.nodes.new(type: str) -> Node
# Creates a new node. 'type' is the node's bl_idname string.
# Example: tree.nodes.new('ShaderNodeBsdfPrincipled')

tree.nodes.remove(node: Node) -> None
# Removes a node from the tree.

tree.nodes.clear() -> None
# Removes all nodes from the tree.

tree.nodes.active  # -> Node or None
# The currently active node (for the node editor UI).

# Access by name
tree.nodes["NodeName"]       # KeyError if missing
tree.nodes.get("NodeName")   # None if missing
```

### Links Collection Methods

```python
# Blender 3.x/4.x/5.x
tree.links.new(
    input: NodeSocket,   # Output socket (source) — confusingly named 'input'
    output: NodeSocket   # Input socket (destination) — confusingly named 'output'
) -> NodeLink
# Creates a link between two sockets.
# IMPORTANT: First argument is the SOURCE (output socket), second is DESTINATION (input socket).
# The parameter names in the API are misleading. Think: "data flows FROM first TO second".

tree.links.remove(link: NodeLink) -> None
# Removes a specific link.

tree.links.clear() -> None
# Removes all links.
```

### Creating Node Trees

```python
# Blender 3.x/4.x/5.x — create a reusable node group
bpy.data.node_groups.new(name: str, type: str) -> NodeTree
# type: 'GeometryNodeTree', 'ShaderNodeTree', 'CompositorNodeTree', 'TextureNodeTree'

# Access existing node groups
bpy.data.node_groups["GroupName"]         # KeyError if missing
bpy.data.node_groups.get("GroupName")     # None if missing

# Remove a node group
bpy.data.node_groups.remove(node_group: NodeTree) -> None
```

---

## bpy.types.NodeTreeInterface (Blender 4.0+)

Replaces the legacy `node_group.inputs` / `node_group.outputs` collections. Manages group socket definitions and interface panels.

### Properties

| Property | Type | Description | Version |
|----------|------|-------------|---------|
| `items_tree` | `bpy_prop_collection` of `NodeTreeInterfaceItem` | All items (sockets + panels) in display order | 4.0+ |

### Methods

```python
# Blender 4.0+ — create a new interface socket
interface.new_socket(
    name: str,                          # Display name
    in_out: str = 'INPUT',              # 'INPUT' or 'OUTPUT'
    socket_type: str = 'NodeSocketFloat',  # Socket type identifier
    parent: NodeTreeInterfacePanel = None   # Optional panel (4.1+)
) -> NodeTreeInterfaceSocket
# Returns the created socket item.
# The socket appears on the Group Input or Group Output node.

# Blender 4.1+ — create an interface panel
interface.new_panel(
    name: str,                          # Panel display name
    default_closed: bool = False        # Start collapsed
) -> NodeTreeInterfacePanel
# Returns the created panel.
# Use as 'parent' parameter in new_socket() to group sockets.

# Blender 4.0+ — remove an interface item
interface.remove(item: NodeTreeInterfaceItem) -> None
# Removes a socket or panel from the interface.

# Blender 4.0+ — clear all interface items
interface.clear() -> None
# Removes all sockets and panels.

# Blender 4.0+ — move an item in the interface
interface.move(item: NodeTreeInterfaceItem, to_position: int) -> None
# Moves item to the specified position index.

# Blender 4.0+ — move an item to a different parent panel
interface.move_to_parent(
    item: NodeTreeInterfaceItem,
    parent: NodeTreeInterfacePanel,
    to_position: int = -1
) -> None
# Moves item under a different panel.
```

### NodeTreeInterfaceItem Properties

| Property | Type | Description |
|----------|------|-------------|
| `item_type` | `str` (enum) | `'SOCKET'` or `'PANEL'` |
| `name` | `str` | Display name |
| `identifier` | `str` | Auto-generated identifier (e.g., `'Socket_0'`, `'Socket_1'`) |
| `in_out` | `str` | `'INPUT'` or `'OUTPUT'` (sockets only) |
| `socket_type` | `str` | Socket type string (sockets only) |
| `description` | `str` | Tooltip text |
| `default_value` | varies | Default value for the socket |
| `min_value` | `float` | Minimum value (numeric sockets) |
| `max_value` | `float` | Maximum value (numeric sockets) |
| `parent` | `NodeTreeInterfacePanel` or `None` | Parent panel |

---

## bpy.types.Node (inherits from bpy_struct)

Base class for all nodes. Specific node types (e.g., `ShaderNodeBsdfPrincipled`) inherit from this.

### Properties

| Property | Type | Description | Version |
|----------|------|-------------|---------|
| `inputs` | `NodeInputs` (bpy_prop_collection of `NodeSocket`) | Input sockets | 3.x/4.x/5.x |
| `outputs` | `NodeOutputs` (bpy_prop_collection of `NodeSocket`) | Output sockets | 3.x/4.x/5.x |
| `name` | `str` | Node name (unique within tree) | 3.x/4.x/5.x |
| `label` | `str` | Custom display label (overrides default) | 3.x/4.x/5.x |
| `bl_idname` | `str` | Node type identifier | 3.x/4.x/5.x |
| `bl_label` | `str` | Default display label | 3.x/4.x/5.x |
| `location` | `(float, float)` | Position in node editor (x, y) | 3.x/4.x/5.x |
| `width` | `float` | Node display width | 3.x/4.x/5.x |
| `width_hidden` | `float` | Width when collapsed | 3.x/4.x/5.x |
| `dimensions` | `(float, float)` | Computed node dimensions (read-only) | 3.x/4.x/5.x |
| `color` | `(float, float, float)` | Custom node color (RGB) | 3.x/4.x/5.x |
| `use_custom_color` | `bool` | Enable custom color | 3.x/4.x/5.x |
| `mute` | `bool` | Bypass/disable node | 3.x/4.x/5.x |
| `hide` | `bool` | Collapse node in editor | 3.x/4.x/5.x |
| `select` | `bool` | Selection state | 3.x/4.x/5.x |
| `parent` | `Node` or `None` | Parent frame node | 3.x/4.x/5.x |
| `type` | `str` | Node type enum (read-only) | 3.x/4.x/5.x |

### Socket Access

```python
# Blender 3.x/4.x/5.x — access by name (ALWAYS preferred)
socket = node.inputs["Base Color"]     # KeyError if not found
socket = node.inputs.get("Base Color") # None if not found

socket = node.outputs["BSDF"]
socket = node.outputs.get("BSDF")

# Access by index (AVOID — fragile)
socket = node.inputs[0]  # Works but breaks if socket order changes
```

### Special Node Types

| Node Type | bl_idname | Purpose |
|-----------|-----------|---------|
| Group Input | `'NodeGroupInput'` | Exposes group inputs inside the tree |
| Group Output | `'NodeGroupOutput'` | Collects outputs from the tree |
| Frame | `'NodeFrame'` | Visual grouping container |
| Reroute | `'NodeReroute'` | Clean up link routing |

---

## bpy.types.NodeSocket (inherits from bpy_struct)

Represents an input or output connection point on a node.

### Properties

| Property | Type | Description | Version |
|----------|------|-------------|---------|
| `name` | `str` | Socket display name | 3.x/4.x/5.x |
| `identifier` | `str` | Unique identifier within the node | 3.x/4.x/5.x |
| `bl_idname` | `str` | Socket type identifier | 3.x/4.x/5.x |
| `type` | `str` (enum) | Data type: `'VALUE'`, `'VECTOR'`, `'RGBA'`, `'SHADER'`, `'STRING'`, `'INT'`, `'GEOMETRY'`, `'BOOLEAN'`, `'OBJECT'`, `'COLLECTION'`, `'IMAGE'`, `'MATERIAL'` | 3.x/4.x/5.x |
| `is_linked` | `bool` | True if connected to another socket | 3.x/4.x/5.x |
| `is_output` | `bool` | True if this is an output socket | 3.x/4.x/5.x |
| `node` | `Node` | Parent node | 3.x/4.x/5.x |
| `links` | `bpy_prop_collection` of `NodeLink` | Connected links | 3.x/4.x/5.x |
| `enabled` | `bool` | Socket active state | 3.x/4.x/5.x |
| `hide` | `bool` | Hidden in node editor | 3.x/4.x/5.x |
| `display_shape` | `str` (enum) | Visual shape: `'CIRCLE'`, `'SQUARE'`, `'DIAMOND'`, `'CIRCLE_DOT'`, `'SQUARE_DOT'`, `'DIAMOND_DOT'` | 3.x/4.x/5.x |
| `description` | `str` | Tooltip text | 3.x/4.x/5.x |
| `default_value` | varies | Value when socket is unlinked | 3.x/4.x/5.x |

### Default Value Types by Socket

| Socket Type | `default_value` Type | Example |
|------------|---------------------|---------|
| `NodeSocketFloat` | `float` | `0.5` |
| `NodeSocketInt` | `int` | `10` |
| `NodeSocketBool` | `bool` | `True` |
| `NodeSocketVector` | `(float, float, float)` | `(1.0, 0.0, 0.0)` |
| `NodeSocketColor` | `(float, float, float, float)` | `(1.0, 0.0, 0.0, 1.0)` |
| `NodeSocketString` | `str` | `"hello"` |
| `NodeSocketGeometry` | N/A | Link-only, no default |
| `NodeSocketShader` | N/A | Link-only, no default |
| `NodeSocketMaterial` | `Material` or `None` | `bpy.data.materials["Mat"]` |
| `NodeSocketObject` | `Object` or `None` | `bpy.data.objects["Cube"]` |
| `NodeSocketCollection` | `Collection` or `None` | `bpy.data.collections["Col"]` |
| `NodeSocketImage` | `Image` or `None` | `bpy.data.images["Img"]` |

---

## bpy.types.NodeLink (inherits from bpy_struct)

Represents a connection between two node sockets.

### Properties

| Property | Type | Description | Version |
|----------|------|-------------|---------|
| `from_node` | `Node` | Source node | 3.x/4.x/5.x |
| `from_socket` | `NodeSocket` | Source output socket | 3.x/4.x/5.x |
| `to_node` | `Node` | Destination node | 3.x/4.x/5.x |
| `to_socket` | `NodeSocket` | Destination input socket | 3.x/4.x/5.x |
| `is_valid` | `bool` | Link is valid (types compatible) | 3.x/4.x/5.x |
| `is_muted` | `bool` | Link is visually muted | 3.x/4.x/5.x |
| `is_hidden` | `bool` | Link is visually hidden | 3.x/4.x/5.x |

---

## Modifier API for Geometry Nodes

### Adding a Geometry Nodes Modifier

```python
# Blender 3.x/4.x/5.x
modifier = obj.modifiers.new(name: str, type: str) -> Modifier
# type='NODES' for Geometry Nodes modifier

modifier.node_group = node_tree  # Assign the node group
# type: NodeTree or None
```

### Setting Modifier Input Values (4.0+)

```python
# Modifier inputs use auto-generated identifiers, NOT socket names
# ALWAYS look up the identifier via interface.items_tree

for item in modifier.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        # item.name = human-readable name (e.g., "Width")
        # item.identifier = auto-generated ID (e.g., "Socket_2")
        modifier[item.identifier] = value
```

### Setting Modifier Input Values (3.x)

```python
# Blender 3.x — inputs accessed differently
modifier["Input_N"] = value
# Where N corresponds to input order
```

---

## Legacy API (Blender 3.x — REMOVED in 4.0+)

These methods exist only in Blender 3.x. Using them in 4.0+ raises `AttributeError`.

```python
# REMOVED in 4.0 — do NOT use in new code
node_group.inputs.new(socket_type: str, name: str) -> NodeSocket
# Example: node_group.inputs.new("NodeSocketFloat", "My Input")

node_group.outputs.new(socket_type: str, name: str) -> NodeSocket
# Example: node_group.outputs.new("NodeSocketGeometry", "Geometry")

node_group.inputs.remove(socket: NodeSocket) -> None
node_group.outputs.remove(socket: NodeSocket) -> None

node_group.inputs.clear() -> None
node_group.outputs.clear() -> None
```

---

## bpy.ops.node — Node Operators

Operators for node editor actions. These require correct editor context (a Node Editor area).

```python
# Blender 3.x/4.x/5.x — common node operators
bpy.ops.node.add_node(type: str, use_transform: bool = False)
# Adds a node of the given type to the active node tree.
# Requires NODE_EDITOR context.

bpy.ops.node.links_cut(path: list)
# Cut links using a path defined by coordinates.

bpy.ops.node.select_all(action: str = 'TOGGLE')
# Select/deselect all nodes. action: 'TOGGLE', 'SELECT', 'DESELECT', 'INVERT'

bpy.ops.node.delete()
# Delete selected nodes.

bpy.ops.node.duplicate()
# Duplicate selected nodes.

bpy.ops.node.group_make()
# Create a node group from selected nodes.

bpy.ops.node.group_ungroup()
# Ungroup selected node group.

bpy.ops.node.group_insert()
# Insert selected nodes into the active group.
```

**IMPORTANT**: Prefer direct `tree.nodes.new()` / `tree.links.new()` over `bpy.ops.node.*` for scripting. Operators require editor context and are fragile in scripts.
