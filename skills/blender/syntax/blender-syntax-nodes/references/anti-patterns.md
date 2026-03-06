# blender-syntax-nodes: Anti-Patterns

These are confirmed error patterns for Blender node system scripting. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- docs/research/supplementary-blender-gaps.md §1

---

## AP-001: Using Legacy Socket API in Blender 4.0+

**WHY this is wrong**: The `node_group.inputs` and `node_group.outputs` collections were removed in Blender 4.0. The `NodeTreeInterface` API replaced them entirely. Using the old API raises `AttributeError` in 4.0+.

```python
# WRONG — BROKEN in Blender 4.0+ (AttributeError)
node_group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
node_group.inputs.new("NodeSocketFloat", "My Input")
node_group.outputs.new("NodeSocketGeometry", "Geometry")
```

```python
# CORRECT — Blender 4.0+
node_group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
node_group.interface.new_socket(
    name="My Input",
    in_out='INPUT',
    socket_type='NodeSocketFloat'
)
node_group.interface.new_socket(
    name="Geometry",
    in_out='OUTPUT',
    socket_type='NodeSocketGeometry'
)
```

ALWAYS use `node_group.interface.new_socket()` in Blender 4.0+. NEVER use `node_group.inputs.new()` or `node_group.outputs.new()` in new code.

---

## AP-002: Accessing Sockets by Index

**WHY this is wrong**: Socket indices change when nodes are updated, sockets are added/removed, or node types change between Blender versions. Index-based access silently connects to the wrong socket or raises `IndexError`.

```python
# WRONG — fragile, breaks when socket order changes
tree.links.new(node_a.outputs[0], node_b.inputs[0])
value = node.inputs[2].default_value
```

```python
# CORRECT — access by name (stable across versions)
tree.links.new(node_a.outputs["Geometry"], node_b.inputs["Geometry"])
value = node.inputs["Scale"].default_value

# CORRECT — with safety check
output = node_a.outputs.get("Geometry")
input_sock = node_b.inputs.get("Geometry")
if output and input_sock:
    tree.links.new(output, input_sock)
```

ALWAYS access sockets by name using `node.inputs["Name"]` or `node.outputs["Name"]`. NEVER use numeric indices for socket access.

---

## AP-003: Using scene.node_tree for Compositor in Blender 5.0+

**WHY this is wrong**: In Blender 5.0, the compositor was refactored. `scene.node_tree` is removed. The compositor now uses a node group assigned via `scene.compositing_node_group`. Additionally, `scene.use_nodes` is deprecated (compositor is always enabled).

```python
# WRONG — BROKEN in Blender 5.0+
scene = bpy.context.scene
scene.use_nodes = True          # Deprecated in 5.0 (no effect)
comp_tree = scene.node_tree     # AttributeError in 5.0
```

```python
# CORRECT — Blender 5.0+
scene = bpy.context.scene
comp_tree = bpy.data.node_groups.new("MyCompositor", 'CompositorNodeTree')
scene.compositing_node_group = comp_tree

# CORRECT — Version-aware pattern
if bpy.app.version >= (5, 0, 0):
    comp_tree = bpy.data.node_groups.new("Compositor", 'CompositorNodeTree')
    scene.compositing_node_group = comp_tree
else:
    scene.use_nodes = True
    comp_tree = scene.node_tree
```

ALWAYS check `bpy.app.version` before accessing compositor trees. NEVER assume `scene.node_tree` exists in code targeting Blender 5.0+.

---

## AP-004: Setting Modifier Inputs by Socket Name

**WHY this is wrong**: Geometry Nodes modifier inputs are accessed via auto-generated identifiers (`Socket_0`, `Socket_1`, etc.), NOT by socket display names. Using the name as a key raises `KeyError` or silently creates a custom property on the modifier object instead of setting the node input.

```python
# WRONG — creates a custom property, does NOT set the node input
modifier = obj.modifiers["GeoNodes"]
modifier["Width"] = 3.5       # KeyError or creates unrelated custom property
modifier["Height"] = 2.0      # Same problem
```

```python
# CORRECT — look up identifier from interface
modifier = obj.modifiers["GeoNodes"]
for item in modifier.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.name == "Width":
            modifier[item.identifier] = 3.5  # e.g., modifier["Socket_2"] = 3.5
        elif item.name == "Height":
            modifier[item.identifier] = 2.0
```

ALWAYS iterate `modifier.node_group.interface.items_tree` to find the correct `identifier` for modifier inputs. NEVER use socket display names as modifier keys.

---

## AP-005: Modifying Node Trees During Active Render

**WHY this is wrong**: Blender's render engine reads node trees on a separate thread. Modifying a node tree while rendering causes race conditions: corrupted node data, crashes, or incorrect render output. This applies to all node tree types (shader, compositor, geometry nodes).

```python
# WRONG — modifying nodes during render
import bpy
from bpy.app.handlers import persistent

@persistent
def update_during_render(scene, depsgraph):
    mat = bpy.data.materials["MyMaterial"]
    tree = mat.node_tree
    tree.nodes["Principled BSDF"].inputs["Roughness"].default_value = 0.5  # RACE CONDITION

bpy.app.handlers.render_pre.append(update_during_render)
```

```python
# CORRECT — modify BEFORE starting render
import bpy

# Set up materials before render
mat = bpy.data.materials.get("MyMaterial")
if mat and mat.node_tree:
    tree = mat.node_tree
    principled = tree.nodes.get("Principled BSDF")
    if principled:
        principled.inputs["Roughness"].default_value = 0.5

# Then render
bpy.ops.render.render(write_still=True)
```

ALWAYS modify node trees before starting a render. NEVER modify node data inside `render_pre`, `render_post`, or during active rendering.

---

## AP-006: Not Clearing Default Nodes After use_nodes = True

**WHY this is wrong**: When setting `material.use_nodes = True`, Blender creates default nodes (Principled BSDF + Material Output). If you add your own nodes without clearing the defaults, the tree has duplicate output nodes, causing confusion about which output is active.

```python
# WRONG — default nodes remain, tree has duplicates
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree

# Default tree already has: Principled BSDF + Material Output
# Adding another output creates ambiguity
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')  # Duplicate!
output = tree.nodes.new('ShaderNodeOutputMaterial')       # Duplicate!
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
# Now 2 Principled nodes and 2 Output nodes exist — confusing
```

```python
# CORRECT — clear defaults before building
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()  # Remove all default nodes

principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (0, 0)
output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (300, 0)
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
```

ALWAYS call `tree.nodes.clear()` before building a complete node tree from scratch. If you want to MODIFY the existing tree, skip the clear and use `tree.nodes.get()` to find existing nodes.

---

## AP-007: Wrong Argument Order in links.new()

**WHY this is wrong**: `tree.links.new()` takes the SOURCE socket (output) as the first argument and the DESTINATION socket (input) as the second argument. The API parameter names (`input`, `output`) are confusingly reversed from their actual meaning. Swapping them creates invalid links or `TypeError`.

```python
# WRONG — reversed argument order
tree.links.new(
    node_b.inputs["Geometry"],    # This is an INPUT socket
    node_a.outputs["Geometry"]    # This is an OUTPUT socket
)
# TypeError or creates an invalid link
```

```python
# CORRECT — SOURCE first, DESTINATION second
tree.links.new(
    node_a.outputs["Geometry"],   # FROM: output socket (source)
    node_b.inputs["Geometry"]     # TO: input socket (destination)
)
# Think: "data flows FROM first argument TO second argument"
```

ALWAYS pass the output socket (source) as the first argument to `links.new()`. NEVER reverse the order. Think "FROM, TO".

---

## AP-008: Using ShaderNodeMixRGB in Blender 4.0+

**WHY this is wrong**: `ShaderNodeMixRGB` was replaced by `ShaderNodeMix` in Blender 4.0. While the old node may still work in some contexts, it is deprecated and `ShaderNodeMix` provides a more flexible interface with support for float, vector, and color mixing.

```python
# WRONG — deprecated node type in 4.0+
mix = tree.nodes.new('ShaderNodeMixRGB')
mix.blend_type = 'MIX'
```

```python
# CORRECT — Blender 4.0+
mix = tree.nodes.new('ShaderNodeMix')
mix.data_type = 'RGBA'     # 'FLOAT', 'VECTOR', or 'RGBA'
mix.blend_type = 'MIX'
```

ALWAYS use `ShaderNodeMix` in Blender 4.0+ code. `ShaderNodeMixRGB` is acceptable only when targeting Blender 3.x.

---

## AP-009: Forgetting to Set use_nodes = True for Materials

**WHY this is wrong**: Newly created materials have `use_nodes = False` by default. Without enabling it, the material has no node tree, and accessing `material.node_tree` returns `None`. Attempting to add nodes to `None` raises `AttributeError`.

```python
# WRONG — node_tree is None
mat = bpy.data.materials.new("MyMat")
tree = mat.node_tree  # None!
tree.nodes.new('ShaderNodeBsdfPrincipled')  # AttributeError: NoneType has no attribute 'nodes'
```

```python
# CORRECT — enable nodes first
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True   # Creates the default node tree
tree = mat.node_tree   # Now a valid NodeTree
tree.nodes.clear()     # Clear defaults
tree.nodes.new('ShaderNodeBsdfPrincipled')
```

ALWAYS set `material.use_nodes = True` before accessing `material.node_tree`. This also applies to `world.use_nodes = True` for world shader nodes.

---

## AP-010: Creating Node Groups Without Group Input/Output Nodes

**WHY this is wrong**: A node group without `NodeGroupInput` and `NodeGroupOutput` nodes has no way to receive or pass data. The group appears to do nothing when used. Additionally, interface sockets (defined via `interface.new_socket()`) only appear on these special nodes.

```python
# WRONG — group has no way to receive or return data
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
group.nodes.clear()

group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

# Added a processing node but no Group Input/Output
transform = group.nodes.new('GeometryNodeTransform')
# No way to connect external inputs/outputs
```

```python
# CORRECT — always add Group Input and Group Output nodes
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
group.nodes.clear()

group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

group_in = group.nodes.new('NodeGroupInput')
group_in.location = (-300, 0)
group_out = group.nodes.new('NodeGroupOutput')
group_out.location = (300, 0)

transform = group.nodes.new('GeometryNodeTransform')
transform.location = (0, 0)

group.links.new(group_in.outputs["Geometry"], transform.inputs["Geometry"])
group.links.new(transform.outputs["Geometry"], group_out.inputs["Geometry"])
```

ALWAYS add `NodeGroupInput` and `NodeGroupOutput` nodes when creating node groups. NEVER define interface sockets without corresponding I/O nodes.

---

## AP-011: Linking Incompatible Socket Types

**WHY this is wrong**: Not all socket types can be linked. For example, linking a `NodeSocketFloat` output to a `NodeSocketGeometry` input creates an invalid link. Blender may silently create the link but mark it as invalid (`link.is_valid == False`), leading to unexpected evaluation failures.

```python
# WRONG — incompatible socket types
noise = tree.nodes.new('ShaderNodeTexNoise')
set_pos = tree.nodes.new('GeometryNodeSetPosition')
# Noise "Fac" (Float) -> SetPosition "Geometry" (Geometry) — INVALID
tree.links.new(noise.outputs["Fac"], set_pos.inputs["Geometry"])
```

```python
# CORRECT — link compatible socket types
noise = tree.nodes.new('ShaderNodeTexNoise')
set_pos = tree.nodes.new('GeometryNodeSetPosition')
# Noise "Fac" (Float) -> SetPosition "Offset" (Vector) — valid (auto-conversion)
tree.links.new(noise.outputs["Fac"], set_pos.inputs["Offset"])
```

ALWAYS verify socket type compatibility before linking. Blender supports automatic conversion between some types (Float->Vector, Float->Color) but NOT between fundamentally different types (Float->Geometry, String->Shader).

---

## AP-012: Using bpy.ops.node Instead of Direct API

**WHY this is wrong**: `bpy.ops.node.*` operators require the Node Editor to be active and in the correct context. They fail silently or raise `RuntimeError` when called from scripts without a visible Node Editor. The direct `tree.nodes.new()` / `tree.links.new()` API works without any editor context.

```python
# WRONG — requires Node Editor context
bpy.ops.node.add_node(type='ShaderNodeBsdfPrincipled')  # RuntimeError: context is incorrect
bpy.ops.node.links_cut()  # Same problem
```

```python
# CORRECT — direct API, no context required
tree = mat.node_tree
node = tree.nodes.new('ShaderNodeBsdfPrincipled')
tree.links.new(node.outputs["BSDF"], output.inputs["Surface"])
```

ALWAYS use `tree.nodes.new()` and `tree.links.new()` for scripted node tree construction. ONLY use `bpy.ops.node.*` when specifically operating within the Node Editor UI context.
