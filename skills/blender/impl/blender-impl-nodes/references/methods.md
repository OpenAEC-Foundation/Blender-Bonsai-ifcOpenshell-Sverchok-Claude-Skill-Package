# blender-impl-nodes: Implementation Helper Methods

Implementation-focused helper functions and utility patterns for building node systems in Blender. These methods complement the raw API documented in `blender-syntax-nodes/references/methods.md`.

Sources:
- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/

---

## Node Group Factory

### create_node_group()

Creates a complete node group with interface sockets and I/O nodes in one call.

```python
# Blender 4.0+
import bpy

def create_node_group(name, tree_type, inputs=None, outputs=None):
    """Create a node group with interface sockets and I/O nodes.

    Args:
        name: Node group name
        tree_type: 'GeometryNodeTree', 'ShaderNodeTree', or 'CompositorNodeTree'
        inputs: list of (name, socket_type) or (name, socket_type, default) tuples
        outputs: list of (name, socket_type) tuples

    Returns:
        tuple: (group, group_input_node, group_output_node)
    """
    group = bpy.data.node_groups.new(name, tree_type)
    group.nodes.clear()

    # Define interface sockets
    if inputs:
        for inp in inputs:
            group.interface.new_socket(
                name=inp[0], in_out='INPUT', socket_type=inp[1]
            )
    if outputs:
        for out in outputs:
            group.interface.new_socket(
                name=out[0], in_out='OUTPUT', socket_type=out[1]
            )

    # Set defaults
    if inputs:
        for item in group.interface.items_tree:
            if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
                for inp in inputs:
                    if item.name == inp[0] and len(inp) > 2:
                        item.default_value = inp[2]

    # Add I/O nodes
    group_in = group.nodes.new('NodeGroupInput')
    group_in.location = (-400, 0)
    group_out = group.nodes.new('NodeGroupOutput')
    group_out.location = (400, 0)

    return group, group_in, group_out
```

---

## Modifier Input Management

### get_modifier_input_map()

```python
# Blender 4.0+
def get_modifier_input_map(modifier):
    """Build a name→identifier mapping for a Geometry Nodes modifier.

    Returns:
        dict: {"socket_name": "Socket_N", ...}
    """
    if modifier.type != 'NODES' or modifier.node_group is None:
        return {}
    input_map = {}
    for item in modifier.node_group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            input_map[item.name] = item.identifier
    return input_map
```

### set_modifier_input()

```python
# Blender 4.0+
def set_modifier_input(modifier, name, value):
    """Set a Geometry Nodes modifier input by socket name.

    Raises:
        KeyError: If no socket with the given name exists
    """
    input_map = get_modifier_input_map(modifier)
    identifier = input_map.get(name)
    if identifier is None:
        raise KeyError(f"No input socket named '{name}' in modifier '{modifier.name}'")
    modifier[identifier] = value
```

### set_modifier_inputs_bulk()

```python
# Blender 4.0+
def set_modifier_inputs_bulk(modifier, values):
    """Set multiple Geometry Nodes modifier inputs at once.

    Args:
        modifier: The Geometry Nodes modifier
        values: {"socket_name": value, ...}
    """
    input_map = get_modifier_input_map(modifier)
    for name, value in values.items():
        identifier = input_map.get(name)
        if identifier is not None:
            modifier[identifier] = value
```

---

## Version-Aware Utilities

### create_socket_compat()

```python
# Blender 3.x / 4.0+ compatible
def create_socket_compat(group, name, in_out, socket_type, panel=None):
    """Create a group socket using the correct API for the Blender version.

    Args:
        group: The node group
        name: Socket display name
        in_out: 'INPUT' or 'OUTPUT'
        socket_type: e.g., 'NodeSocketFloat', 'NodeSocketGeometry'
        panel: Optional panel (4.1+ only, ignored on older versions)

    Returns:
        The created socket item
    """
    if bpy.app.version >= (4, 0, 0):
        kwargs = {"name": name, "in_out": in_out, "socket_type": socket_type}
        if panel is not None and bpy.app.version >= (4, 1, 0):
            kwargs["parent"] = panel
        return group.interface.new_socket(**kwargs)
    else:
        if in_out == 'INPUT':
            return group.inputs.new(socket_type, name)
        else:
            return group.outputs.new(socket_type, name)
```

### clear_sockets_compat()

```python
# Blender 3.x / 4.0+ compatible
def clear_sockets_compat(group):
    """Clear all group sockets using the correct API."""
    if bpy.app.version >= (4, 0, 0):
        group.interface.clear()
    else:
        group.inputs.clear()
        group.outputs.clear()
```

### get_compositor_tree()

```python
# Blender 3.x / 4.x / 5.0+ compatible
def get_compositor_tree(scene, name="Compositor"):
    """Get or create the compositor node tree for the given scene.

    Returns:
        NodeTree: The compositor node tree
    """
    if bpy.app.version >= (5, 0, 0):
        tree = bpy.data.node_groups.new(name, 'CompositorNodeTree')
        scene.compositing_node_group = tree
    else:
        scene.use_nodes = True
        tree = scene.node_tree
    return tree
```

---

## Node Layout and Organization

### layout_nodes_horizontal()

```python
def layout_nodes_horizontal(nodes, x_start=-400, x_spacing=250, y=0):
    """Position a list of nodes in a horizontal row.

    Args:
        nodes: List of Node objects
        x_start: X position of the first node
        x_spacing: Horizontal distance between nodes
        y: Y position for all nodes
    """
    for i, node in enumerate(nodes):
        node.location = (x_start + i * x_spacing, y)
```

### layout_nodes_grid()

```python
def layout_nodes_grid(nodes, columns=3, x_start=-400, x_spacing=250,
                      y_start=0, y_spacing=-200):
    """Position nodes in a grid layout.

    Args:
        nodes: List of Node objects
        columns: Number of columns in the grid
        x_start: X position of the first column
        x_spacing: Horizontal distance between columns
        y_start: Y position of the first row
        y_spacing: Vertical distance between rows (negative = downward)
    """
    for i, node in enumerate(nodes):
        col = i % columns
        row = i // columns
        node.location = (x_start + col * x_spacing, y_start + row * y_spacing)
```

### create_frame()

```python
def create_frame(tree, label, nodes, color=None):
    """Create a frame node and parent other nodes to it.

    Args:
        tree: The node tree
        label: Frame display label
        nodes: List of nodes to parent to the frame
        color: Optional (R, G, B) tuple for frame color

    Returns:
        The created NodeFrame
    """
    frame = tree.nodes.new('NodeFrame')
    frame.label = label
    if color is not None:
        frame.use_custom_color = True
        frame.color = color
    for node in nodes:
        node.parent = frame
    return frame
```

---

## Node Chain Builder

### build_node_chain()

```python
def build_node_chain(tree, specs, x_start=-600, x_spacing=250, y=0):
    """Create a chain of nodes and link them sequentially.

    Args:
        tree: NodeTree to add nodes to
        specs: List of dicts with keys:
            - type: bl_idname string
            - links: list of (output_name, input_name) to link to NEXT node
            - settings: dict of {input_name: default_value} (optional)
        x_start: Starting X position
        x_spacing: Horizontal spacing
        y: Y position

    Returns:
        List of created Node objects
    """
    nodes = []
    for i, spec in enumerate(specs):
        node = tree.nodes.new(spec["type"])
        node.location = (x_start + i * x_spacing, y)

        # Apply settings
        for input_name, value in spec.get("settings", {}).items():
            sock = node.inputs.get(input_name)
            if sock and not sock.is_linked:
                sock.default_value = value

        nodes.append(node)

    # Link consecutive nodes
    for i in range(len(specs) - 1):
        link_pairs = specs[i].get("links", [])
        for out_name, in_name in link_pairs:
            out_sock = nodes[i].outputs.get(out_name)
            in_sock = nodes[i + 1].inputs.get(in_name)
            if out_sock and in_sock:
                tree.links.new(out_sock, in_sock)

    return nodes
```

---

## Material Utilities

### create_principled_material()

```python
# Blender 3.x/4.x/5.x
def create_principled_material(name, base_color=(0.8, 0.8, 0.8, 1.0),
                                roughness=0.5, metallic=0.0):
    """Create a simple Principled BSDF material.

    Args:
        name: Material name
        base_color: RGBA tuple
        roughness: Roughness value (0.0-1.0)
        metallic: Metallic value (0.0-1.0)

    Returns:
        The created Material
    """
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    tree = mat.node_tree
    tree.nodes.clear()

    principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
    principled.location = (0, 0)
    principled.inputs["Base Color"].default_value = base_color
    principled.inputs["Roughness"].default_value = roughness
    principled.inputs["Metallic"].default_value = metallic

    output = tree.nodes.new('ShaderNodeOutputMaterial')
    output.location = (300, 0)
    tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])

    return mat
```

### assign_material()

```python
def assign_material(obj, mat, slot_index=None):
    """Assign a material to an object.

    Args:
        obj: The target object
        mat: The material to assign
        slot_index: Slot index to replace (None = append new slot)
    """
    if slot_index is not None and slot_index < len(obj.data.materials):
        obj.data.materials[slot_index] = mat
    else:
        obj.data.materials.append(mat)
```

---

## Node Group Management

### get_or_create_node_group()

```python
def get_or_create_node_group(name, tree_type):
    """Get an existing node group by name, or create a new one.

    Args:
        name: Node group name
        tree_type: 'GeometryNodeTree', 'ShaderNodeTree', etc.

    Returns:
        tuple: (group, is_new) where is_new indicates if freshly created
    """
    group = bpy.data.node_groups.get(name)
    if group is not None:
        return group, False
    group = bpy.data.node_groups.new(name, tree_type)
    return group, True
```

### embed_node_group()

```python
# Blender 4.0+
def embed_node_group(tree, group_name, location=(0, 0), input_values=None):
    """Add an existing node group as a node inside a tree.

    Args:
        tree: Target node tree
        group_name: Name of the node group in bpy.data.node_groups
        location: (x, y) position
        input_values: {"input_name": value, ...} (optional)

    Returns:
        The created group node, or None if group not found

    Note:
        The group node bl_idname depends on tree type:
        - GeometryNodeTree → 'GeometryNodeGroup'
        - ShaderNodeTree → 'ShaderNodeGroup'
        - CompositorNodeTree → 'CompositorNodeGroup'
    """
    source = bpy.data.node_groups.get(group_name)
    if source is None:
        return None

    # Determine correct group node type
    type_map = {
        'GEOMETRY': 'GeometryNodeGroup',
        'SHADER': 'ShaderNodeGroup',
        'COMPOSITING': 'CompositorNodeGroup',
    }
    group_node_type = type_map.get(tree.type)
    if group_node_type is None:
        return None

    group_node = tree.nodes.new(group_node_type)
    group_node.node_tree = source
    group_node.location = location

    if input_values:
        for name, value in input_values.items():
            sock = group_node.inputs.get(name)
            if sock and not sock.is_linked:
                sock.default_value = value

    return group_node
```

---

## AEC Component Factory

### create_aec_component()

```python
# Blender 4.0+
def create_aec_component(name, socket_config):
    """Create an AEC parametric component node group.

    Args:
        name: Component name (e.g., "AEC_Wall")
        socket_config: list of tuples:
            (name, in_out, socket_type) — basic socket
            (name, in_out, socket_type, default) — with default
            (name, in_out, socket_type, default, min, max) — with limits

    Returns:
        tuple: (group, group_input_node, group_output_node)
    """
    group = bpy.data.node_groups.new(name, 'GeometryNodeTree')
    group.nodes.clear()

    # Create interface sockets
    for cfg in socket_config:
        sock_name, in_out, sock_type = cfg[0], cfg[1], cfg[2]
        group.interface.new_socket(
            name=sock_name, in_out=in_out, socket_type=sock_type
        )

    # Set defaults and limits
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            for cfg in socket_config:
                if cfg[0] == item.name and len(cfg) > 3:
                    item.default_value = cfg[3]
                    if len(cfg) > 4 and hasattr(item, 'min_value'):
                        item.min_value = cfg[4]
                    if len(cfg) > 5 and hasattr(item, 'max_value'):
                        item.max_value = cfg[5]

    # Add I/O nodes
    group_in = group.nodes.new('NodeGroupInput')
    group_in.location = (-400, 0)
    group_out = group.nodes.new('NodeGroupOutput')
    group_out.location = (400, 0)

    return group, group_in, group_out
```

---

## Inspection Utilities

### print_node_tree_info()

```python
def print_node_tree_info(tree):
    """Print a summary of a node tree for debugging."""
    print(f"Tree: {tree.name} (type: {tree.type})")
    print(f"  Nodes ({len(tree.nodes)}):")
    for node in tree.nodes:
        print(f"    {node.name} ({node.bl_idname}) at {tuple(node.location)}")
    print(f"  Links ({len(tree.links)}):")
    for link in tree.links:
        print(f"    {link.from_node.name}:{link.from_socket.name}"
              f" → {link.to_node.name}:{link.to_socket.name}")
```

### print_interface_info()

```python
# Blender 4.0+
def print_interface_info(group):
    """Print the interface definition of a node group."""
    print(f"Interface for {group.name}:")
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET':
            default = getattr(item, 'default_value', 'N/A')
            print(f"  {item.in_out}: {item.name} ({item.socket_type})"
                  f" id={item.identifier} default={default}")
        elif item.item_type == 'PANEL':
            print(f"  PANEL: {item.name}")
```
