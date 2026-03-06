# blender-impl-nodes: Implementation Examples

Complete, working implementation examples focused on AEC node workflows. All examples verified against official Blender Python API documentation.

Sources:
- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- docs/research/supplementary-blender-gaps.md §1, §3.4

---

## Example 1: AEC Wall Generator with Geometry Nodes (Blender 4.1+)

Complete parametric wall component with organized interface panels.

```python
# Blender 4.1+ — Parametric wall generator
import bpy

def create_wall_generator():
    """Create a parametric wall Geometry Nodes group with panels."""
    group = bpy.data.node_groups.new("AEC_Wall_Generator", 'GeometryNodeTree')
    group.nodes.clear()

    # Create interface panels (4.1+)
    dims_panel = group.interface.new_panel("Dimensions")
    mat_panel = group.interface.new_panel("Material")

    # Input sockets
    group.interface.new_socket(
        name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry'
    )
    group.interface.new_socket(
        name="Length", in_out='INPUT',
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
    # Output socket
    group.interface.new_socket(
        name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry'
    )

    # Set defaults and limits
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == "Length":
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

    # Add nodes
    group_in = group.nodes.new('NodeGroupInput')
    group_in.location = (-600, 0)

    # Combine XYZ for cube size: (Length, Height, Thickness)
    combine = group.nodes.new('ShaderNodeCombineXYZ')
    combine.location = (-200, 100)

    cube = group.nodes.new('GeometryNodeMeshCube')
    cube.location = (0, 0)

    set_mat = group.nodes.new('GeometryNodeSetMaterial')
    set_mat.location = (200, 0)

    # Join with input geometry (passthrough)
    join = group.nodes.new('GeometryNodeJoinGeometry')
    join.location = (400, 0)

    group_out = group.nodes.new('NodeGroupOutput')
    group_out.location = (600, 0)

    # Link nodes
    group.links.new(group_in.outputs["Length"], combine.inputs["X"])
    group.links.new(group_in.outputs["Height"], combine.inputs["Y"])
    group.links.new(group_in.outputs["Thickness"], combine.inputs["Z"])
    group.links.new(combine.outputs["Vector"], cube.inputs["Size"])
    group.links.new(cube.outputs["Mesh"], set_mat.inputs["Geometry"])
    group.links.new(group_in.outputs["Material"], set_mat.inputs["Material"])
    group.links.new(group_in.outputs["Geometry"], join.inputs["Geometry"])
    group.links.new(set_mat.outputs["Geometry"], join.inputs["Geometry"])
    group.links.new(join.outputs["Geometry"], group_out.inputs["Geometry"])

    return group


def assign_wall_to_object(obj, length=5.0, height=3.0, thickness=0.2, material=None):
    """Assign the wall generator to an object and set parameters."""
    group = bpy.data.node_groups.get("AEC_Wall_Generator")
    if group is None:
        group = create_wall_generator()

    mod = obj.modifiers.new(name="AEC_Wall", type='NODES')
    mod.node_group = group

    # Set inputs using identifier lookup
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == "Length":
                mod[item.identifier] = length
            elif item.name == "Height":
                mod[item.identifier] = height
            elif item.name == "Thickness":
                mod[item.identifier] = thickness
            elif item.name == "Material" and material is not None:
                mod[item.identifier] = material


# Usage
obj = bpy.context.active_object
if obj:
    assign_wall_to_object(obj, length=8.0, height=2.8, thickness=0.25)
```

---

## Example 2: AEC Material Library (Blender 4.0+)

Batch-create standard AEC materials with consistent naming.

```python
# Blender 4.0+ — AEC material library
import bpy

# Material definitions: (name, base_color, roughness, metallic, extra_settings)
AEC_MATERIALS = [
    ("AEC_Concrete_Cast", (0.45, 0.43, 0.40, 1.0), 0.85, 0.0, {}),
    ("AEC_Concrete_Polished", (0.55, 0.53, 0.50, 1.0), 0.3, 0.0, {}),
    ("AEC_Steel_Brushed", (0.7, 0.7, 0.72, 1.0), 0.35, 0.95, {}),
    ("AEC_Steel_Corten", (0.45, 0.22, 0.1, 1.0), 0.9, 0.7, {}),
    ("AEC_Wood_Oak", (0.42, 0.27, 0.12, 1.0), 0.7, 0.0, {}),
    ("AEC_Brick_Red", (0.5, 0.15, 0.08, 1.0), 0.9, 0.0, {}),
    ("AEC_Plaster_White", (0.9, 0.88, 0.85, 1.0), 0.8, 0.0, {}),
    ("AEC_Glass_Clear", (0.8, 0.9, 1.0, 1.0), 0.0, 0.0,
     {"transmission": 1.0, "ior": 1.45, "alpha": 0.3}),
]


def create_aec_material(name, base_color, roughness, metallic, extra):
    """Create a single AEC material with Principled BSDF."""
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    tree = mat.node_tree
    tree.nodes.clear()

    # Principled BSDF
    principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
    principled.location = (0, 0)
    principled.inputs["Base Color"].default_value = base_color
    principled.inputs["Roughness"].default_value = roughness
    principled.inputs["Metallic"].default_value = metallic

    # Apply extra settings
    if "transmission" in extra:
        # Blender 4.0+ uses "Transmission Weight"
        principled.inputs["Transmission Weight"].default_value = extra["transmission"]
    if "ior" in extra:
        principled.inputs["IOR"].default_value = extra["ior"]
    if "alpha" in extra:
        principled.inputs["Alpha"].default_value = extra["alpha"]

    # Material Output
    output = tree.nodes.new('ShaderNodeOutputMaterial')
    output.location = (300, 0)
    tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])

    # Glass-specific: enable alpha blending for EEVEE
    if "transmission" in extra:
        if bpy.app.version >= (4, 2, 0):
            mat.surface_render_method = 'BLENDED'
        else:
            mat.blend_method = 'BLEND'

    return mat


def create_all_aec_materials():
    """Create all AEC materials. Skips materials that already exist."""
    created = []
    for name, color, rough, metal, extra in AEC_MATERIALS:
        if bpy.data.materials.get(name) is None:
            mat = create_aec_material(name, color, rough, metal, extra)
            created.append(mat.name)
    return created


# Usage
created = create_all_aec_materials()
print(f"Created {len(created)} AEC materials: {created}")
```

---

## Example 3: Procedural Concrete Material with Bump (Blender 4.0+)

```python
# Blender 4.0+ — Procedural concrete material with noise-based bump
import bpy

def create_procedural_concrete(name="AEC_Concrete_Procedural"):
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    tree = mat.node_tree
    tree.nodes.clear()

    # Texture Coordinates
    tex_coord = tree.nodes.new('ShaderNodeTexCoord')
    tex_coord.location = (-800, 0)

    # Mapping (scale texture)
    mapping = tree.nodes.new('ShaderNodeMapping')
    mapping.location = (-600, 0)
    mapping.inputs["Scale"].default_value = (2.0, 2.0, 2.0)

    # Noise for base color variation
    noise_color = tree.nodes.new('ShaderNodeTexNoise')
    noise_color.location = (-400, 150)
    noise_color.inputs["Scale"].default_value = 8.0
    noise_color.inputs["Detail"].default_value = 12.0
    noise_color.inputs["Roughness"].default_value = 0.7

    # Color Ramp for concrete tones
    ramp = tree.nodes.new('ShaderNodeValToRGB')
    ramp.location = (-200, 150)
    ramp.color_ramp.elements[0].color = (0.35, 0.33, 0.30, 1.0)
    ramp.color_ramp.elements[1].color = (0.55, 0.53, 0.50, 1.0)

    # Noise for bump
    noise_bump = tree.nodes.new('ShaderNodeTexNoise')
    noise_bump.location = (-400, -100)
    noise_bump.inputs["Scale"].default_value = 20.0
    noise_bump.inputs["Detail"].default_value = 8.0

    # Bump node
    bump = tree.nodes.new('ShaderNodeBump')
    bump.location = (-200, -100)
    bump.inputs["Strength"].default_value = 0.15
    bump.inputs["Distance"].default_value = 0.02

    # Principled BSDF
    principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
    principled.location = (100, 0)
    principled.inputs["Roughness"].default_value = 0.85
    principled.inputs["Specular IOR Level"].default_value = 0.3

    # Output
    output = tree.nodes.new('ShaderNodeOutputMaterial')
    output.location = (400, 0)

    # Link chain
    tree.links.new(tex_coord.outputs["Object"], mapping.inputs["Vector"])
    tree.links.new(mapping.outputs["Vector"], noise_color.inputs["Vector"])
    tree.links.new(mapping.outputs["Vector"], noise_bump.inputs["Vector"])
    tree.links.new(noise_color.outputs["Fac"], ramp.inputs["Fac"])
    tree.links.new(ramp.outputs["Color"], principled.inputs["Base Color"])
    tree.links.new(noise_bump.outputs["Fac"], bump.inputs["Height"])
    tree.links.new(bump.outputs["Normal"], principled.inputs["Normal"])
    tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])

    return mat
```

---

## Example 4: Column Array with Geometry Nodes (Blender 4.0+)

```python
# Blender 4.0+ — Parametric column array
import bpy

def create_column_array():
    group = bpy.data.node_groups.new("AEC_Column_Array", 'GeometryNodeTree')
    group.nodes.clear()

    # Interface sockets
    group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
    group.interface.new_socket(name="Column Radius", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Column Height", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Count X", in_out='INPUT', socket_type='NodeSocketInt')
    group.interface.new_socket(name="Count Y", in_out='INPUT', socket_type='NodeSocketInt')
    group.interface.new_socket(name="Spacing X", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Spacing Y", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Material", in_out='INPUT', socket_type='NodeSocketMaterial')
    group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

    # Set defaults
    defaults = {
        "Column Radius": (0.15, 0.05, 2.0),
        "Column Height": (3.0, 0.5, 50.0),
        "Count X": (5, 1, 100),
        "Count Y": (3, 1, 100),
        "Spacing X": (5.0, 0.5, 50.0),
        "Spacing Y": (5.0, 0.5, 50.0),
    }
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name in defaults:
                val, mn, mx = defaults[item.name]
                item.default_value = val
                if hasattr(item, 'min_value'):
                    item.min_value = mn
                    item.max_value = mx

    # Nodes
    group_in = group.nodes.new('NodeGroupInput')
    group_in.location = (-800, 0)

    # Create cylinder for column
    cylinder = group.nodes.new('GeometryNodeMeshCylinder')
    cylinder.location = (-400, 200)

    # Create grid of points for instancing
    grid = group.nodes.new('GeometryNodeMeshGrid')
    grid.location = (-400, -100)

    # Instance columns on grid points
    instance = group.nodes.new('GeometryNodeInstanceOnPoints')
    instance.location = (0, 0)

    # Set material
    set_mat = group.nodes.new('GeometryNodeSetMaterial')
    set_mat.location = (200, 0)

    # Join with input geometry
    join = group.nodes.new('GeometryNodeJoinGeometry')
    join.location = (400, 0)

    group_out = group.nodes.new('NodeGroupOutput')
    group_out.location = (600, 0)

    # Links
    group.links.new(group_in.outputs["Column Radius"], cylinder.inputs["Radius"])
    group.links.new(group_in.outputs["Column Height"], cylinder.inputs["Depth"])
    group.links.new(group_in.outputs["Count X"], grid.inputs["Vertices X"])
    group.links.new(group_in.outputs["Count Y"], grid.inputs["Vertices Y"])
    group.links.new(group_in.outputs["Spacing X"], grid.inputs["Size X"])
    group.links.new(group_in.outputs["Spacing Y"], grid.inputs["Size Y"])
    group.links.new(grid.outputs["Mesh"], instance.inputs["Points"])
    group.links.new(cylinder.outputs["Mesh"], instance.inputs["Instance"])
    group.links.new(instance.outputs["Instances"], set_mat.inputs["Geometry"])
    group.links.new(group_in.outputs["Material"], set_mat.inputs["Material"])
    group.links.new(group_in.outputs["Geometry"], join.inputs["Geometry"])
    group.links.new(set_mat.outputs["Geometry"], join.inputs["Geometry"])
    group.links.new(join.outputs["Geometry"], group_out.inputs["Geometry"])

    return group
```

---

## Example 5: Compositor Post-Processing for AEC Visualization (Version-Aware)

```python
# Blender 3.x/4.x/5.x — AEC render post-processing
import bpy

def setup_aec_compositor(scene=None):
    """Set up a compositor pipeline for AEC visualization renders."""
    if scene is None:
        scene = bpy.context.scene

    # Version-aware compositor access
    if bpy.app.version >= (5, 0, 0):
        comp = bpy.data.node_groups.new("AEC_PostProcess", 'CompositorNodeTree')
        scene.compositing_node_group = comp
    else:
        scene.use_nodes = True
        comp = scene.node_tree

    comp.nodes.clear()

    # Render Layers
    rl = comp.nodes.new('CompositorNodeRLayers')
    rl.location = (-600, 0)

    # Glare for window reflections
    glare = comp.nodes.new('CompositorNodeGlare')
    glare.location = (-300, 200)
    glare.glare_type = 'FOG_GLOW'
    glare.quality = 'HIGH'
    glare.mix = 0.3
    glare.threshold = 2.0

    # Color Balance for architectural mood
    color_bal = comp.nodes.new('CompositorNodeColorBalance')
    color_bal.location = (0, 0)
    color_bal.correction_method = 'LIFT_GAMMA_GAIN'

    # Lens Distortion (subtle)
    lens = comp.nodes.new('CompositorNodeLensdist')
    lens.location = (300, 0)
    lens.inputs["Distort"].default_value = -0.01
    lens.inputs["Dispersion"].default_value = 0.005

    # Composite output
    composite = comp.nodes.new('CompositorNodeComposite')
    composite.location = (600, 100)

    # Viewer for preview
    viewer = comp.nodes.new('CompositorNodeViewer')
    viewer.location = (600, -100)

    # Link chain
    comp.links.new(rl.outputs["Image"], glare.inputs["Image"])
    comp.links.new(glare.outputs["Image"], color_bal.inputs["Image"])
    comp.links.new(color_bal.outputs["Image"], lens.inputs["Image"])
    comp.links.new(lens.outputs["Image"], composite.inputs["Image"])
    comp.links.new(lens.outputs["Image"], viewer.inputs["Image"])

    return comp
```

---

## Example 6: Nested Node Groups — Facade Pattern (Blender 4.0+)

```python
# Blender 4.0+ — Nested groups: window unit → facade pattern
import bpy

def create_window_unit():
    """Create a single window unit as a reusable node group."""
    group = bpy.data.node_groups.new("AEC_Window_Unit", 'GeometryNodeTree')
    group.nodes.clear()

    group.interface.new_socket(name="Width", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Height", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Depth", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Frame Material", in_out='INPUT', socket_type='NodeSocketMaterial')
    group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == "Width":
                item.default_value = 1.2
            elif item.name == "Height":
                item.default_value = 1.5
            elif item.name == "Depth":
                item.default_value = 0.1

    group_in = group.nodes.new('NodeGroupInput')
    group_in.location = (-400, 0)
    group_out = group.nodes.new('NodeGroupOutput')
    group_out.location = (400, 0)

    # Simple cube for window shape
    combine = group.nodes.new('ShaderNodeCombineXYZ')
    combine.location = (-200, 100)

    cube = group.nodes.new('GeometryNodeMeshCube')
    cube.location = (0, 0)

    set_mat = group.nodes.new('GeometryNodeSetMaterial')
    set_mat.location = (200, 0)

    group.links.new(group_in.outputs["Width"], combine.inputs["X"])
    group.links.new(group_in.outputs["Height"], combine.inputs["Y"])
    group.links.new(group_in.outputs["Depth"], combine.inputs["Z"])
    group.links.new(combine.outputs["Vector"], cube.inputs["Size"])
    group.links.new(cube.outputs["Mesh"], set_mat.inputs["Geometry"])
    group.links.new(group_in.outputs["Frame Material"], set_mat.inputs["Material"])
    group.links.new(set_mat.outputs["Geometry"], group_out.inputs["Geometry"])

    return group


def create_facade_pattern():
    """Create a facade that instances window units in a grid."""
    # Ensure window unit exists
    if bpy.data.node_groups.get("AEC_Window_Unit") is None:
        create_window_unit()

    group = bpy.data.node_groups.new("AEC_Facade_Pattern", 'GeometryNodeTree')
    group.nodes.clear()

    group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
    group.interface.new_socket(name="Columns", in_out='INPUT', socket_type='NodeSocketInt')
    group.interface.new_socket(name="Rows", in_out='INPUT', socket_type='NodeSocketInt')
    group.interface.new_socket(name="Spacing X", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Spacing Z", in_out='INPUT', socket_type='NodeSocketFloat')
    group.interface.new_socket(name="Window Material", in_out='INPUT', socket_type='NodeSocketMaterial')
    group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == "Columns":
                item.default_value = 5
            elif item.name == "Rows":
                item.default_value = 4
            elif item.name == "Spacing X":
                item.default_value = 2.0
            elif item.name == "Spacing Z":
                item.default_value = 3.0

    group_in = group.nodes.new('NodeGroupInput')
    group_in.location = (-600, 0)
    group_out = group.nodes.new('NodeGroupOutput')
    group_out.location = (600, 0)

    # Grid for instancing
    grid = group.nodes.new('GeometryNodeMeshGrid')
    grid.location = (-200, -100)

    # Embed the window unit group
    window = group.nodes.new('GeometryNodeGroup')
    window.node_tree = bpy.data.node_groups["AEC_Window_Unit"]
    window.location = (-200, 200)

    # Instance window units on grid points
    instance = group.nodes.new('GeometryNodeInstanceOnPoints')
    instance.location = (100, 0)

    # Join with input geometry
    join = group.nodes.new('GeometryNodeJoinGeometry')
    join.location = (350, 0)

    # Links
    group.links.new(group_in.outputs["Columns"], grid.inputs["Vertices X"])
    group.links.new(group_in.outputs["Rows"], grid.inputs["Vertices Y"])
    group.links.new(group_in.outputs["Spacing X"], grid.inputs["Size X"])
    group.links.new(group_in.outputs["Spacing Z"], grid.inputs["Size Y"])
    group.links.new(group_in.outputs["Window Material"], window.inputs["Frame Material"])
    group.links.new(grid.outputs["Mesh"], instance.inputs["Points"])
    group.links.new(window.outputs["Geometry"], instance.inputs["Instance"])
    group.links.new(group_in.outputs["Geometry"], join.inputs["Geometry"])
    group.links.new(instance.outputs["Instances"], join.inputs["Geometry"])
    group.links.new(join.outputs["Geometry"], group_out.inputs["Geometry"])

    return group
```

---

## Example 7: Linking Node Groups from External .blend Files (Blender 3.x/4.x/5.x)

```python
# Blender 3.x/4.x/5.x — Append or link node groups from another .blend file
import bpy
import os

def append_node_group(filepath, group_name):
    """Append a node group from an external .blend file (full copy).

    Args:
        filepath: Path to the .blend file
        group_name: Name of the node group to append

    Returns:
        The appended NodeTree, or None if not found
    """
    blend_path = os.path.abspath(filepath)
    inner_path = "NodeTree"

    bpy.ops.wm.append(
        filepath=os.path.join(blend_path, inner_path, group_name),
        directory=os.path.join(blend_path, inner_path),
        filename=group_name,
        link=False  # Append = full copy
    )
    return bpy.data.node_groups.get(group_name)


def link_node_group(filepath, group_name):
    """Link a node group from an external .blend file (reference, read-only).

    Args:
        filepath: Path to the .blend file
        group_name: Name of the node group to link

    Returns:
        The linked NodeTree, or None if not found
    """
    blend_path = os.path.abspath(filepath)
    inner_path = "NodeTree"

    bpy.ops.wm.link(
        filepath=os.path.join(blend_path, inner_path, group_name),
        directory=os.path.join(blend_path, inner_path),
        filename=group_name,
    )
    return bpy.data.node_groups.get(group_name)


# Usage
# group = append_node_group("/path/to/library.blend", "AEC_Wall_Generator")
# if group:
#     mod = obj.modifiers.new("Wall", type='NODES')
#     mod.node_group = group
```

---

## Example 8: HDRI World Environment for AEC (Blender 3.x/4.x/5.x)

```python
# Blender 3.x/4.x/5.x — HDRI environment for architectural rendering
import bpy

def setup_hdri_world(hdri_path, strength=1.0, rotation_z=0.0):
    """Set up an HDRI world environment for AEC renders.

    Args:
        hdri_path: Path to the .hdr or .exr file
        strength: Background light strength
        rotation_z: Z rotation in radians for sun position
    """
    world = bpy.data.worlds.new("AEC_HDRI_World")
    world.use_nodes = True
    tree = world.node_tree
    tree.nodes.clear()

    # Build node chain
    tex_coord = tree.nodes.new('ShaderNodeTexCoord')
    tex_coord.location = (-600, 0)

    mapping = tree.nodes.new('ShaderNodeMapping')
    mapping.location = (-400, 0)
    mapping.inputs["Rotation"].default_value = (0, 0, rotation_z)

    env_tex = tree.nodes.new('ShaderNodeTexEnvironment')
    env_tex.location = (-200, 0)
    env_tex.image = bpy.data.images.load(hdri_path)

    background = tree.nodes.new('ShaderNodeBackground')
    background.location = (100, 0)
    background.inputs["Strength"].default_value = strength

    world_out = tree.nodes.new('ShaderNodeOutputWorld')
    world_out.location = (300, 0)

    # Link
    tree.links.new(tex_coord.outputs["Generated"], mapping.inputs["Vector"])
    tree.links.new(mapping.outputs["Vector"], env_tex.inputs["Vector"])
    tree.links.new(env_tex.outputs["Color"], background.inputs["Color"])
    tree.links.new(background.outputs["Background"], world_out.inputs["Surface"])

    # Assign to scene
    bpy.context.scene.world = world
    return world
```

---

## Example 9: Batch Assign Geometry Nodes to Multiple Objects (Blender 4.0+)

```python
# Blender 4.0+ — Assign same Geometry Nodes group to multiple objects
import bpy

def batch_assign_geonodes(objects, group_name, input_overrides=None):
    """Assign a Geometry Nodes group to multiple objects.

    Args:
        objects: List of Object references or names
        group_name: Name of the node group in bpy.data.node_groups
        input_overrides: Optional dict of {"socket_name": value} to set on each modifier
    """
    group = bpy.data.node_groups.get(group_name)
    if group is None:
        raise ValueError(f"Node group '{group_name}' not found")

    # Build input map once
    input_map = {}
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            input_map[item.name] = item.identifier

    for obj in objects:
        if isinstance(obj, str):
            obj = bpy.data.objects.get(obj)
        if obj is None:
            continue

        mod = obj.modifiers.new(name=group_name, type='NODES')
        mod.node_group = group

        if input_overrides:
            for name, value in input_overrides.items():
                identifier = input_map.get(name)
                if identifier:
                    mod[identifier] = value


# Usage
selected = [obj for obj in bpy.context.selected_objects if obj.type == 'MESH']
batch_assign_geonodes(selected, "AEC_Wall_Generator", {"Length": 10.0, "Height": 3.5})
```
