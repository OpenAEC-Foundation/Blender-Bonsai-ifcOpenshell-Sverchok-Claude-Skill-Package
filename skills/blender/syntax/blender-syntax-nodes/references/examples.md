# blender-syntax-nodes: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/

---

## Example 1: Complete Geometry Nodes Setup (Blender 4.0+)

```python
# Blender 4.0+ — Create a geometry nodes group and assign to an object
import bpy

# Create node group
group = bpy.data.node_groups.new("ScatterPoints", 'GeometryNodeTree')
group.nodes.clear()

# Add Group Input / Output
group_in = group.nodes.new('NodeGroupInput')
group_in.location = (-400, 0)
group_out = group.nodes.new('NodeGroupOutput')
group_out.location = (400, 0)

# Define interface sockets
group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Count", in_out='INPUT', socket_type='NodeSocketInt')
group.interface.new_socket(name="Seed", in_out='INPUT', socket_type='NodeSocketInt')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

# Set default values on interface sockets
for item in group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.name == "Count":
            item.default_value = 100
        elif item.name == "Seed":
            item.default_value = 0

# Add processing nodes
distribute = group.nodes.new('GeometryNodeDistributePointsOnFaces')
distribute.location = (0, 0)
distribute.distribute_method = 'POISSON'

# Link nodes
group.links.new(group_in.outputs["Geometry"], distribute.inputs["Mesh"])
group.links.new(group_in.outputs["Count"], distribute.inputs["Density Max"])
group.links.new(group_in.outputs["Seed"], distribute.inputs["Seed"])
group.links.new(distribute.outputs["Points"], group_out.inputs["Geometry"])

# Assign to active object
obj = bpy.context.active_object
if obj is not None:
    mod = obj.modifiers.new(name="ScatterPoints", type='NODES')
    mod.node_group = group

    # Set modifier input values using identifiers
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == "Count":
                mod[item.identifier] = 500
            elif item.name == "Seed":
                mod[item.identifier] = 42
```

---

## Example 2: Shader Node Material with Texture (Blender 3.x/4.x/5.x)

```python
# Blender 3.x/4.x/5.x — Create a PBR material with texture nodes
import bpy

mat = bpy.data.materials.new("PBR_Concrete")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

# Create nodes
tex_coord = tree.nodes.new('ShaderNodeTexCoord')
tex_coord.location = (-800, 0)

mapping = tree.nodes.new('ShaderNodeMapping')
mapping.location = (-600, 0)

noise = tree.nodes.new('ShaderNodeTexNoise')
noise.location = (-400, 100)
noise.inputs["Scale"].default_value = 5.0
noise.inputs["Detail"].default_value = 10.0

color_ramp = tree.nodes.new('ShaderNodeValToRGB')
color_ramp.location = (-200, 100)
# Set color ramp stops
color_ramp.color_ramp.elements[0].color = (0.15, 0.15, 0.15, 1.0)
color_ramp.color_ramp.elements[1].color = (0.35, 0.35, 0.35, 1.0)

bump = tree.nodes.new('ShaderNodeBump')
bump.location = (-200, -100)
bump.inputs["Strength"].default_value = 0.3

principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (100, 0)
principled.inputs["Roughness"].default_value = 0.85

output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (400, 0)

# Link nodes
tree.links.new(tex_coord.outputs["Object"], mapping.inputs["Vector"])
tree.links.new(mapping.outputs["Vector"], noise.inputs["Vector"])
tree.links.new(noise.outputs["Fac"], color_ramp.inputs["Fac"])
tree.links.new(color_ramp.outputs["Color"], principled.inputs["Base Color"])
tree.links.new(noise.outputs["Fac"], bump.inputs["Height"])
tree.links.new(bump.outputs["Normal"], principled.inputs["Normal"])
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])

# Assign to active object
obj = bpy.context.active_object
if obj is not None:
    if obj.data.materials:
        obj.data.materials[0] = mat
    else:
        obj.data.materials.append(mat)
```

---

## Example 3: Compositor Setup (Version-Aware)

```python
# Blender 3.x/4.x/5.x — Version-aware compositor setup
import bpy

scene = bpy.context.scene

# Get compositor tree based on version
if bpy.app.version >= (5, 0, 0):
    comp_tree = bpy.data.node_groups.new("PostProcessing", 'CompositorNodeTree')
    scene.compositing_node_group = comp_tree
else:
    scene.use_nodes = True
    comp_tree = scene.node_tree

comp_tree.nodes.clear()

# Render Layers input
render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
render_layers.location = (-400, 0)

# Glare effect
glare = comp_tree.nodes.new('CompositorNodeGlare')
glare.location = (-100, 100)
glare.glare_type = 'FOG_GLOW'
glare.quality = 'HIGH'
glare.mix = 0.5

# Color Balance
color_balance = comp_tree.nodes.new('CompositorNodeColorBalance')
color_balance.location = (200, 0)
color_balance.correction_method = 'LIFT_GAMMA_GAIN'

# Output nodes
composite = comp_tree.nodes.new('CompositorNodeComposite')
composite.location = (500, 100)

viewer = comp_tree.nodes.new('CompositorNodeViewer')
viewer.location = (500, -100)

# Link
comp_tree.links.new(render_layers.outputs["Image"], glare.inputs["Image"])
comp_tree.links.new(glare.outputs["Image"], color_balance.inputs["Image"])
comp_tree.links.new(color_balance.outputs["Image"], composite.inputs["Image"])
comp_tree.links.new(color_balance.outputs["Image"], viewer.inputs["Image"])
```

---

## Example 4: Node Group with Interface Panels (Blender 4.1+)

```python
# Blender 4.1+ — Create a node group with organized interface panels
import bpy

group = bpy.data.node_groups.new("WallGenerator", 'GeometryNodeTree')
group.nodes.clear()

# Create panels for organized inputs
dims_panel = group.interface.new_panel("Dimensions")
mat_panel = group.interface.new_panel("Material")

# Add sockets under panels
group.interface.new_socket(
    name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry'
)
group.interface.new_socket(
    name="Width", in_out='INPUT',
    socket_type='NodeSocketFloat', parent=dims_panel
)
group.interface.new_socket(
    name="Height", in_out='INPUT',
    socket_type='NodeSocketFloat', parent=dims_panel
)
group.interface.new_socket(
    name="Thickness", in_out='INPUT',
    socket_type='NodeSocketFloat', parent=dims_panel
)
group.interface.new_socket(
    name="Material", in_out='INPUT',
    socket_type='NodeSocketMaterial', parent=mat_panel
)
group.interface.new_socket(
    name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry'
)

# Set default values
for item in group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.name == "Width":
            item.default_value = 5.0
            item.min_value = 0.1
            item.max_value = 100.0
        elif item.name == "Height":
            item.default_value = 3.0
            item.min_value = 0.1
            item.max_value = 50.0
        elif item.name == "Thickness":
            item.default_value = 0.2
            item.min_value = 0.01
            item.max_value = 2.0

# Add Group Input/Output
group_in = group.nodes.new('NodeGroupInput')
group_in.location = (-400, 0)
group_out = group.nodes.new('NodeGroupOutput')
group_out.location = (400, 0)

# Add Cube for wall shape
cube = group.nodes.new('GeometryNodeMeshCube')
cube.location = (0, 0)

# Set material
set_mat = group.nodes.new('GeometryNodeSetMaterial')
set_mat.location = (200, 0)

# Link
group.links.new(group_in.outputs["Width"], cube.inputs["Size"])
group.links.new(cube.outputs["Mesh"], set_mat.inputs["Geometry"])
group.links.new(group_in.outputs["Material"], set_mat.inputs["Material"])
group.links.new(set_mat.outputs["Geometry"], group_out.inputs["Geometry"])
```

---

## Example 5: Iterating and Inspecting Node Trees

```python
# Blender 3.x/4.x/5.x — Inspect all node trees in the file
import bpy

# List all node groups
for ng in bpy.data.node_groups:
    print(f"Node Group: {ng.name} (type: {ng.type})")
    print(f"  Nodes: {len(ng.nodes)}")
    print(f"  Links: {len(ng.links)}")

    # List nodes
    for node in ng.nodes:
        print(f"    Node: {node.name} ({node.bl_idname})")
        for inp in node.inputs:
            linked = "LINKED" if inp.is_linked else f"default={getattr(inp, 'default_value', 'N/A')}"
            print(f"      IN:  {inp.name} [{inp.type}] {linked}")
        for out in node.outputs:
            linked = "LINKED" if out.is_linked else "unlinked"
            print(f"      OUT: {out.name} [{out.type}] {linked}")

# Inspect material node trees
for mat in bpy.data.materials:
    if mat.use_nodes and mat.node_tree:
        tree = mat.node_tree
        print(f"Material: {mat.name}")
        for node in tree.nodes:
            print(f"  {node.name}: {node.bl_idname}")

# Inspect interface sockets (4.0+)
if bpy.app.version >= (4, 0, 0):
    for ng in bpy.data.node_groups:
        print(f"\nInterface for {ng.name}:")
        for item in ng.interface.items_tree:
            if item.item_type == 'SOCKET':
                print(f"  {item.in_out}: {item.name} ({item.socket_type}) id={item.identifier}")
            elif item.item_type == 'PANEL':
                print(f"  PANEL: {item.name}")
```

---

## Example 6: Finding Modifier Input Identifiers

```python
# Blender 4.0+ — Map socket names to modifier identifiers
import bpy

def get_modifier_input_map(modifier):
    """Build a name->identifier mapping for a Geometry Nodes modifier."""
    if modifier.type != 'NODES' or modifier.node_group is None:
        return {}

    input_map = {}
    for item in modifier.node_group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            input_map[item.name] = item.identifier
    return input_map


def set_modifier_input(modifier, socket_name, value):
    """Set a modifier input by socket name (not identifier)."""
    input_map = get_modifier_input_map(modifier)
    identifier = input_map.get(socket_name)
    if identifier is None:
        raise KeyError(f"No input socket named '{socket_name}' found")
    modifier[identifier] = value


# Usage
obj = bpy.context.active_object
if obj is not None:
    for mod in obj.modifiers:
        if mod.type == 'NODES' and mod.node_group:
            print(f"Modifier: {mod.name}")
            input_map = get_modifier_input_map(mod)
            for name, identifier in input_map.items():
                print(f"  {name} -> {identifier} = {mod.get(identifier, 'unset')}")
```

---

## Example 7: Reusable Node Group (Nested Groups)

```python
# Blender 4.0+ — Create a node group and use it inside another group
import bpy

# Create inner group: a simple noise displacement
inner = bpy.data.node_groups.new("NoiseDisplace", 'GeometryNodeTree')
inner.nodes.clear()
inner.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
inner.interface.new_socket(name="Scale", in_out='INPUT', socket_type='NodeSocketFloat')
inner.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

inner_in = inner.nodes.new('NodeGroupInput')
inner_in.location = (-400, 0)
inner_out = inner.nodes.new('NodeGroupOutput')
inner_out.location = (400, 0)

set_pos = inner.nodes.new('GeometryNodeSetPosition')
set_pos.location = (0, 0)

noise = inner.nodes.new('ShaderNodeTexNoise')
noise.location = (-200, -100)

inner.links.new(inner_in.outputs["Geometry"], set_pos.inputs["Geometry"])
inner.links.new(noise.outputs["Fac"], set_pos.inputs["Offset"])
inner.links.new(inner_in.outputs["Scale"], noise.inputs["Scale"])
inner.links.new(set_pos.outputs["Geometry"], inner_out.inputs["Geometry"])

# Create outer group that USES the inner group
outer = bpy.data.node_groups.new("TerrainGenerator", 'GeometryNodeTree')
outer.nodes.clear()
outer.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
outer.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

outer_in = outer.nodes.new('NodeGroupInput')
outer_in.location = (-400, 0)
outer_out = outer.nodes.new('NodeGroupOutput')
outer_out.location = (400, 0)

# Add the inner group as a node
group_node = outer.nodes.new('GeometryNodeGroup')
group_node.node_tree = inner  # Assign the inner group
group_node.location = (0, 0)

# Set the group node's input
group_node.inputs["Scale"].default_value = 5.0

# Link
outer.links.new(outer_in.outputs["Geometry"], group_node.inputs["Geometry"])
outer.links.new(group_node.outputs["Geometry"], outer_out.inputs["Geometry"])
```

---

## Example 8: Version-Aware Node Group Socket Creation

```python
# Blender 3.x and 4.0+ — compatible socket creation
import bpy

def create_group_socket(node_group, name, in_out, socket_type):
    """Create a group socket using the correct API for the current Blender version."""
    if bpy.app.version >= (4, 0, 0):
        return node_group.interface.new_socket(
            name=name,
            in_out=in_out,  # 'INPUT' or 'OUTPUT'
            socket_type=socket_type
        )
    else:
        # Blender 3.x legacy API
        if in_out == 'INPUT':
            return node_group.inputs.new(socket_type, name)
        else:
            return node_group.outputs.new(socket_type, name)


def clear_group_sockets(node_group):
    """Clear all group sockets using the correct API."""
    if bpy.app.version >= (4, 0, 0):
        node_group.interface.clear()
    else:
        node_group.inputs.clear()
        node_group.outputs.clear()


# Usage
group = bpy.data.node_groups.new("CompatibleGroup", 'GeometryNodeTree')
group.nodes.clear()
clear_group_sockets(group)

create_group_socket(group, "Geometry", 'INPUT', 'NodeSocketGeometry')
create_group_socket(group, "Factor", 'INPUT', 'NodeSocketFloat')
create_group_socket(group, "Geometry", 'OUTPUT', 'NodeSocketGeometry')
```

---

## Example 9: Batch Node Creation with Automatic Layout

```python
# Blender 3.x/4.x/5.x — Create a chain of shader nodes with automatic positioning
import bpy


def create_node_chain(tree, node_specs, x_start=-600, x_spacing=250, y=0):
    """Create a chain of linked nodes.

    Args:
        tree: NodeTree to add nodes to
        node_specs: List of (bl_idname, socket_name_pairs) tuples
            socket_name_pairs: list of (output_name, input_name) for linking to next node
        x_start: Starting X position
        x_spacing: Horizontal spacing between nodes
        y: Y position

    Returns:
        List of created nodes
    """
    nodes = []
    for i, (node_type, _) in enumerate(node_specs):
        node = tree.nodes.new(node_type)
        node.location = (x_start + i * x_spacing, y)
        nodes.append(node)

    # Link consecutive nodes
    for i in range(len(node_specs) - 1):
        _, link_pairs = node_specs[i]
        if link_pairs:
            for out_name, in_name in link_pairs:
                tree.links.new(nodes[i].outputs[out_name], nodes[i + 1].inputs[in_name])

    return nodes


# Usage: Create a shader chain
mat = bpy.data.materials.new("ChainMaterial")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

nodes = create_node_chain(tree, [
    ('ShaderNodeTexCoord', [("Object", "Vector")]),
    ('ShaderNodeTexNoise', [("Fac", "Fac")]),
    ('ShaderNodeValToRGB', [("Color", "Base Color")]),
    ('ShaderNodeBsdfPrincipled', [("BSDF", "Surface")]),
    ('ShaderNodeOutputMaterial', None),
])
```

---

## Example 10: World Shader Nodes

```python
# Blender 3.x/4.x/5.x — Create an HDRI world shader
import bpy

world = bpy.data.worlds.new("HDRIWorld")
world.use_nodes = True
tree = world.node_tree
tree.nodes.clear()

# Create nodes
tex_coord = tree.nodes.new('ShaderNodeTexCoord')
tex_coord.location = (-600, 0)

mapping = tree.nodes.new('ShaderNodeMapping')
mapping.location = (-400, 0)

env_tex = tree.nodes.new('ShaderNodeTexEnvironment')
env_tex.location = (-200, 0)
# Set image: env_tex.image = bpy.data.images.load("/path/to/hdri.hdr")

background = tree.nodes.new('ShaderNodeBackground')
background.location = (100, 0)
background.inputs["Strength"].default_value = 1.0

world_output = tree.nodes.new('ShaderNodeOutputWorld')
world_output.location = (300, 0)

# Link
tree.links.new(tex_coord.outputs["Generated"], mapping.inputs["Vector"])
tree.links.new(mapping.outputs["Vector"], env_tex.inputs["Vector"])
tree.links.new(env_tex.outputs["Color"], background.inputs["Color"])
tree.links.new(background.outputs["Background"], world_output.inputs["Surface"])

# Assign to scene
bpy.context.scene.world = world
```
