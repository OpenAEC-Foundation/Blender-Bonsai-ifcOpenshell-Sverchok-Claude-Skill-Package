---
name: blender-syntax-nodes
description: >
  Use when building node trees via Python -- Geometry Nodes, Shader Nodes, or Compositor
  Nodes. Prevents the breaking change of using node.inputs/outputs by index instead of
  NodeTreeInterface (4.0+) for node group I/O. Covers node creation, linking, group
  management, and the NodeTreeInterface API migration.
  Keywords: node tree, Geometry Nodes, Shader Nodes, Compositor, NodeTreeInterface, node_group, links, nodes.new, node sockets, procedural geometry, create nodes from Python, connect nodes, add node.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-nodes

## Quick Reference

### Critical Warnings

**NEVER** use `node_group.inputs.new()` or `node_group.outputs.new()` in Blender 4.0+ — these are removed. ALWAYS use `node_group.interface.new_socket()`.

**NEVER** access node sockets by index (`node.inputs[0]`) — indices shift when sockets are added or removed. ALWAYS use `node.inputs["Name"]`.

**NEVER** use `scene.node_tree` for compositor access in Blender 5.0+ — it is removed. ALWAYS use `scene.compositing_node_group`.

**NEVER** set modifier inputs by socket name (`modifier["Width"]`) — modifier inputs use auto-generated identifiers (`Socket_N`). ALWAYS iterate `interface.items_tree` to find the correct identifier.

**NEVER** modify node trees during an active render — race conditions with the render thread cause crashes. ALWAYS modify nodes before render starts.

### Node Tree Architecture

All node systems in Blender share this architecture:

```
NodeTree
├── nodes: Collection[Node]        — all nodes in the tree
├── links: Collection[NodeLink]    — connections between sockets
└── interface: NodeTreeInterface   — group inputs/outputs (4.0+)

Node
├── inputs: Collection[NodeSocket]   — input sockets
├── outputs: Collection[NodeSocket]  — output sockets
├── location: (x, y)                 — position in editor
└── name / label / bl_idname         — identification

NodeLink
├── from_node / from_socket          — source
└── to_node / to_socket              — destination

NodeSocket
├── name / identifier                — naming
├── type                             — data type enum
├── is_linked                        — connection state
└── default_value                    — value when unconnected
```

### Node Tree Types

| `type` Parameter | Purpose | Access Pattern |
|-----------------|---------|----------------|
| `'GeometryNodeTree'` | Geometry Nodes | `bpy.data.node_groups.new(name, 'GeometryNodeTree')` |
| `'ShaderNodeTree'` | Material/World shaders | `material.node_tree` or `bpy.data.node_groups.new(name, 'ShaderNodeTree')` |
| `'CompositorNodeTree'` | Compositor | `scene.node_tree` (3.x/4.x) or `scene.compositing_node_group` (5.0+) |
| `'TextureNodeTree'` | Legacy texture nodes | `bpy.data.node_groups.new(name, 'TextureNodeTree')` |
| `'SverchCustomTreeType'` | Sverchok parametric nodes (third-party addon) | `bpy.data.node_groups` (filter by `bl_idname=='SverchCustomTreeType'`). Requires Sverchok addon enabled. See `sverchok-core-concepts` skill. |

### Decision Tree: Which Node API?

```
Q: Are you creating a node GROUP (reusable)?
├── YES → bpy.data.node_groups.new(name, tree_type)
│         Then: group.interface.new_socket() for inputs/outputs (4.0+)
└── NO → What node tree type?
    ├── Shader → material.use_nodes = True; tree = material.node_tree
    ├── Compositor (3.x/4.x) → scene.use_nodes = True; tree = scene.node_tree
    ├── Compositor (5.0+) → tree = bpy.data.node_groups.new(name, 'CompositorNodeTree')
    │                        scene.compositing_node_group = tree
    └── Geometry Nodes → Create group, assign to modifier
```

### Decision Tree: Socket Interface (Version-Critical)

```
Q: What Blender version?
├── 3.x → node_group.inputs.new(socket_type, name)
│         node_group.outputs.new(socket_type, name)
├── 4.0+ → node_group.interface.new_socket(name=, in_out=, socket_type=)
└── 4.1+ → Also supports: node_group.interface.new_panel(name)
           with parent= parameter on new_socket()
```

---

## Essential Patterns

### Pattern 1: Creating a Node Group (Blender 4.0+)

```python
import bpy

# Create the node group
group = bpy.data.node_groups.new("MyNodeGroup", 'GeometryNodeTree')
group.nodes.clear()

# Add Group Input/Output nodes
input_node = group.nodes.new('NodeGroupInput')
input_node.location = (-300, 0)
output_node = group.nodes.new('NodeGroupOutput')
output_node.location = (300, 0)

# Define interface sockets (4.0+)
group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Scale", in_out='INPUT', socket_type='NodeSocketFloat')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')
```

### Pattern 2: Creating and Linking Nodes

```python
# Add nodes
transform = group.nodes.new('GeometryNodeTransform')
transform.location = (0, 0)

# Link: source output -> destination input
group.links.new(input_node.outputs["Geometry"], transform.inputs["Geometry"])
group.links.new(input_node.outputs["Scale"], transform.inputs["Scale"])
group.links.new(transform.outputs["Geometry"], output_node.inputs["Geometry"])
```

### Pattern 3: Shader Node Material

```python
mat = bpy.data.materials.new("MyMaterial")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

# Add nodes
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (0, 0)
output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (300, 0)

# Link
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])

# Set values on unlinked sockets
principled.inputs["Base Color"].default_value = (0.8, 0.2, 0.1, 1.0)
principled.inputs["Roughness"].default_value = 0.4
```

### Pattern 4: Assigning Geometry Nodes to Object

```python
obj = bpy.context.active_object
modifier = obj.modifiers.new(name="GeoNodes", type='NODES')
modifier.node_group = bpy.data.node_groups["MyNodeGroup"]

# Set modifier input values: MUST find identifier first
for item in modifier.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.name == "Scale":
            modifier[item.identifier] = 2.5
            break
```

### Pattern 5: Compositor Nodes (Version-Aware)

```python
import bpy

scene = bpy.context.scene

if bpy.app.version >= (5, 0, 0):
    # Blender 5.0+: compositor uses node groups
    comp_tree = bpy.data.node_groups.new("MyCompositor", 'CompositorNodeTree')
    scene.compositing_node_group = comp_tree
else:
    # Blender 3.x/4.x: compositor on scene
    scene.use_nodes = True
    comp_tree = scene.node_tree

comp_tree.nodes.clear()

render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
render_layers.location = (-300, 0)
composite = comp_tree.nodes.new('CompositorNodeComposite')
composite.location = (300, 0)

comp_tree.links.new(render_layers.outputs["Image"], composite.inputs["Image"])
```

### Pattern 6: Interface Panels (Blender 4.1+)

```python
# Group sockets into collapsible panels
group = bpy.data.node_groups["MyNodeGroup"]

panel = group.interface.new_panel("Dimensions")
group.interface.new_socket(
    name="Width", in_out='INPUT',
    socket_type='NodeSocketFloat', parent=panel
)
group.interface.new_socket(
    name="Height", in_out='INPUT',
    socket_type='NodeSocketFloat', parent=panel
)
```

---

## Common Operations

### Adding Common Node Types

**Geometry Nodes:**

| Node Type String | Purpose |
|-----------------|---------|
| `'GeometryNodeMeshCube'` | Cube mesh primitive |
| `'GeometryNodeMeshCylinder'` | Cylinder mesh primitive |
| `'GeometryNodeMeshGrid'` | Grid mesh primitive |
| `'GeometryNodeMeshLine'` | Line mesh primitive |
| `'GeometryNodeTransform'` | Transform geometry |
| `'GeometryNodeSetPosition'` | Set vertex positions |
| `'GeometryNodeJoinGeometry'` | Merge geometries |
| `'GeometryNodeInstanceOnPoints'` | Instance objects on points |
| `'GeometryNodeCurvePrimitiveLine'` | Curve line |
| `'GeometryNodeExtrudeMesh'` | Extrude mesh faces/edges |
| `'GeometryNodeBooleanMath'` | Boolean operations |
| `'GeometryNodeInputPosition'` | Read position attribute |
| `'GeometryNodeStoreNamedAttribute'` | Write custom attribute |
| `'FunctionNodeInputVector'` | Vector input value |
| `'ShaderNodeMath'` | Math operations (shared) |
| `'ShaderNodeVectorMath'` | Vector math (shared) |

**Shader Nodes:**

| Node Type String | Purpose |
|-----------------|---------|
| `'ShaderNodeBsdfPrincipled'` | Principled BSDF shader |
| `'ShaderNodeOutputMaterial'` | Material output |
| `'ShaderNodeTexImage'` | Image texture |
| `'ShaderNodeTexNoise'` | Noise texture |
| `'ShaderNodeTexCoord'` | Texture coordinates |
| `'ShaderNodeMapping'` | Vector mapping |
| `'ShaderNodeMixRGB'` | Color mix (3.x) |
| `'ShaderNodeMix'` | Mix node (4.0+, replaces MixRGB) |
| `'ShaderNodeNormalMap'` | Normal map |
| `'ShaderNodeBump'` | Bump mapping |
| `'ShaderNodeValToRGB'` | Color ramp |
| `'ShaderNodeSeparateXYZ'` | Split vector components |
| `'ShaderNodeCombineXYZ'` | Combine vector components |

**Compositor Nodes:**

| Node Type String | Purpose |
|-----------------|---------|
| `'CompositorNodeRLayers'` | Render Layers input |
| `'CompositorNodeComposite'` | Final composite output |
| `'CompositorNodeViewer'` | Viewer node |
| `'CompositorNodeMixRGB'` | Mix colors |
| `'CompositorNodeBlur'` | Blur filter |
| `'CompositorNodeGlare'` | Glare effect |
| `'CompositorNodeColorBalance'` | Color balance |

### Common Socket Types

| Socket Type String | Data Type | Default Value Type |
|-------------------|-----------|-------------------|
| `'NodeSocketFloat'` | Single float | `float` |
| `'NodeSocketInt'` | Integer | `int` |
| `'NodeSocketBool'` | Boolean | `bool` |
| `'NodeSocketVector'` | 3D vector | `(float, float, float)` |
| `'NodeSocketColor'` | RGBA color | `(float, float, float, float)` |
| `'NodeSocketString'` | String | `str` |
| `'NodeSocketGeometry'` | Geometry data | N/A (link-only) |
| `'NodeSocketMaterial'` | Material ref | `Material` or `None` |
| `'NodeSocketObject'` | Object ref | `Object` or `None` |
| `'NodeSocketCollection'` | Collection ref | `Collection` or `None` |
| `'NodeSocketImage'` | Image ref | `Image` or `None` |
| `'NodeSocketShader'` | Shader closure | N/A (link-only) |

### Removing Nodes and Links

```python
# Remove a specific node
tree.nodes.remove(tree.nodes["NodeName"])

# Remove a specific link
for link in tree.links:
    if link.from_node.name == "Source" and link.to_node.name == "Dest":
        tree.links.remove(link)
        break

# Clear all nodes
tree.nodes.clear()

# Clear all links (keeps nodes)
tree.links.clear()
```

### Setting Socket Default Values

```python
# Set value on an UNLINKED input socket
node.inputs["Value"].default_value = 1.5
node.inputs["Vector"].default_value = (1.0, 2.0, 3.0)
node.inputs["Color"].default_value = (1.0, 0.0, 0.0, 1.0)

# Check if socket is linked before setting default
if not node.inputs["Value"].is_linked:
    node.inputs["Value"].default_value = 1.5
```

### Node Layout and Organization

```python
# Position nodes for readability
node_a.location = (-300, 0)
node_b.location = (0, 0)
node_c.location = (300, 0)

# Add a frame for visual grouping
frame = tree.nodes.new('NodeFrame')
frame.label = "Processing Stage"
frame.use_custom_color = True
frame.color = (0.2, 0.4, 0.6)

# Parent nodes to frame
node_a.parent = frame
node_b.parent = frame

# Mute a node (bypass it)
node_b.mute = True

# Hide a node (collapse)
node_b.hide = True
```

---

## Version Migration Matrix

| Feature | Blender 3.x | Blender 4.0+ | Blender 5.0+ |
|---------|-------------|--------------|---------------|
| Group socket creation | `group.inputs.new()` / `group.outputs.new()` | `group.interface.new_socket()` | `group.interface.new_socket()` |
| Interface panels | Not available | Not available | Available (added 4.1) |
| Compositor tree access | `scene.node_tree` | `scene.node_tree` | `scene.compositing_node_group` |
| `scene.use_nodes` (compositor) | Required | Required | Deprecated (always True) |
| Mix node | `ShaderNodeMixRGB` | `ShaderNodeMix` (new) | `ShaderNodeMix` |
| Shader name `3D_UNIFORM_COLOR` | Valid | Removed — use `UNIFORM_COLOR` | Use `UNIFORM_COLOR` |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for NodeTree, NodeTreeInterface, Node, NodeSocket, NodeLink
- [references/examples.md](references/examples.md) — Working code examples for all node tree types
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with node trees

### Official Sources

- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://docs.blender.org/api/current/bpy.types.Node.html
- https://docs.blender.org/api/current/bpy.types.NodeSocket.html
- https://docs.blender.org/api/current/bpy.types.NodeLink.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
