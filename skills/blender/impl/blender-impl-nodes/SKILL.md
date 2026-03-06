---
name: blender-impl-nodes
description: 'Provides implementation workflows for Blender node systems including creating Geometry Nodes modifiers from Python, building shader node trees programmatically, custom node groups, and linking node setups for AEC visualization. Activates when building node-based setups for parametric modeling, material generation, or procedural geometry.'
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-impl-nodes

## Quick Reference

### Critical Workflow Rules

**ALWAYS** follow this order when building node trees from Python:
1. Create/get the node tree container
2. Clear existing nodes if building from scratch
3. Define interface sockets (4.0+) or legacy inputs/outputs (3.x)
4. Add Group Input / Group Output nodes
5. Add processing nodes with explicit positions
6. Link nodes (source output FIRST, destination input SECOND)
7. Assign to object/modifier and set input values

**ALWAYS** use `tree.nodes.new()` / `tree.links.new()` for scripting. NEVER use `bpy.ops.node.*` operators — they require editor context.

**ALWAYS** set `location` on every node. Without explicit positions, all nodes stack at `(0, 0)` making the tree unreadable in the editor.

**ALWAYS** iterate `interface.items_tree` to find modifier input identifiers. NEVER use socket names as modifier keys.

### Master Decision Tree: Which Workflow?

```
Q: What are you building?
├── Geometry Nodes modifier on an object
│   └── → Workflow 1: Geometry Nodes Modifier
├── Material / shader for an object
│   └── → Workflow 2: Shader Material
├── World environment (HDRI, sky)
│   └── → Workflow 3: World Shader
├── Compositor post-processing
│   └── → Workflow 4: Compositor Setup
├── Reusable node group (shared across objects)
│   └── → Workflow 5: Custom Node Group
├── Nested / composed node groups
│   └── → Workflow 6: Nested Groups
├── Parametric AEC component (wall, slab, column)
│   └── → Workflow 7: AEC Parametric Component
└── Batch materials for AEC visualization
    └── → Workflow 8: AEC Material Library
```

### Version Decision Tree

```
Q: What Blender version?
├── 3.x
│   ├── Socket creation: node_group.inputs.new() / outputs.new()
│   ├── Compositor: scene.use_nodes = True; scene.node_tree
│   ├── Mix node: ShaderNodeMixRGB
│   └── No interface panels
├── 4.0-4.0.x
│   ├── Socket creation: node_group.interface.new_socket()
│   ├── Compositor: scene.use_nodes = True; scene.node_tree
│   ├── Mix node: ShaderNodeMix
│   └── No interface panels
├── 4.1+
│   ├── Socket creation: node_group.interface.new_socket()
│   ├── Interface panels: node_group.interface.new_panel()
│   ├── Compositor: scene.use_nodes = True; scene.node_tree
│   ├── Mix node: ShaderNodeMix
│   └── Alpha blending (4.2+): mat.surface_render_method = 'BLENDED'
└── 5.0+
    ├── Socket creation: node_group.interface.new_socket()
    ├── Interface panels: node_group.interface.new_panel()
    ├── Compositor: scene.compositing_node_group (scene.node_tree REMOVED)
    ├── scene.use_nodes: deprecated (always True)
    └── Mix node: ShaderNodeMix
```

---

## Workflow 1: Geometry Nodes Modifier

Use when: creating a Geometry Nodes setup and assigning it to an object.

### Step-by-Step

```
1. Create node group     → bpy.data.node_groups.new(name, 'GeometryNodeTree')
2. Clear nodes           → group.nodes.clear()
3. Define interface       → group.interface.new_socket() for each input/output
4. Set socket defaults    → iterate interface.items_tree, set default_value/min/max
5. Add I/O nodes         → group.nodes.new('NodeGroupInput') + 'NodeGroupOutput'
6. Add processing nodes  → group.nodes.new(node_type) with location
7. Link nodes            → group.links.new(output_socket, input_socket)
8. Get target object     → bpy.data.objects["Name"] or bpy.context.active_object
9. Add modifier          → obj.modifiers.new(name="Name", type='NODES')
10. Assign node group    → modifier.node_group = group
11. Set modifier inputs  → iterate interface.items_tree, modifier[identifier] = value
```

### Modifier Input Helper Pattern

```python
# Blender 4.0+ — ALWAYS use this pattern to set modifier inputs
def set_geonodes_inputs(modifier, values_dict):
    """Set Geometry Nodes modifier inputs by name.

    Args:
        modifier: The Geometry Nodes modifier
        values_dict: {"socket_name": value, ...}
    """
    for item in modifier.node_group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name in values_dict:
                modifier[item.identifier] = values_dict[item.name]
```

### Interface Socket Defaults Pattern

```python
# Blender 4.0+ — Set defaults, min, max on interface sockets
def configure_socket_defaults(group, config):
    """Configure socket defaults after creation.

    Args:
        group: The node group
        config: {"socket_name": {"default": val, "min": val, "max": val}, ...}
    """
    for item in group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name in config:
                cfg = config[item.name]
                if "default" in cfg:
                    item.default_value = cfg["default"]
                if "min" in cfg and hasattr(item, 'min_value'):
                    item.min_value = cfg["min"]
                if "max" in cfg and hasattr(item, 'max_value'):
                    item.max_value = cfg["max"]
```

---

## Workflow 2: Shader Material

Use when: creating a material with shader nodes for an object.

### Step-by-Step

```
1. Create material       → bpy.data.materials.new(name)
2. Enable nodes          → mat.use_nodes = True
3. Get node tree         → tree = mat.node_tree
4. Clear defaults        → tree.nodes.clear()
5. Add shader nodes      → tree.nodes.new(shader_type) with location
6. Add Material Output   → tree.nodes.new('ShaderNodeOutputMaterial')
7. Set socket values     → node.inputs["Name"].default_value = value
8. Link nodes            → tree.links.new(output_socket, input_socket)
9. Assign to object      → obj.data.materials.append(mat) or obj.data.materials[0] = mat
```

### Material Assignment Decision

```
Q: Does the object already have materials?
├── NO  → obj.data.materials.append(mat)
├── YES, replace first → obj.data.materials[0] = mat
├── YES, replace specific → obj.data.materials[slot_index] = mat
└── YES, add as extra slot → obj.data.materials.append(mat)
```

### Common Shader Chains

```
PBR Material:
  TexCoord → Mapping → ImageTex → Principled BSDF → Material Output
                                   ↑ NormalMap ←── ImageTex (normal)

Procedural Material:
  TexCoord → Mapping → NoiseTexture → ColorRamp → Principled BSDF → Output
                        ↓ (Fac)
                        Bump → Principled (Normal input)

Glass Material (AEC):
  Principled BSDF (Transmission Weight=1.0, IOR=1.45) → Material Output
  + mat.surface_render_method = 'BLENDED' (4.2+)
  + mat.blend_method = 'BLEND' (3.x/4.0-4.1)
```

---

## Workflow 3: World Shader

Use when: setting up environment lighting (HDRI, procedural sky).

### Step-by-Step

```
1. Create world          → bpy.data.worlds.new(name)
2. Enable nodes          → world.use_nodes = True
3. Get node tree         → tree = world.node_tree
4. Clear defaults        → tree.nodes.clear()
5. Add environment nodes → TexCoord, Mapping, TexEnvironment or TexSky
6. Add Background node   → ShaderNodeBackground (set Strength)
7. Add World Output      → ShaderNodeOutputWorld
8. Link chain            → TexCoord → Mapping → Env → Background → Output
9. Load HDRI (if needed) → env_tex.image = bpy.data.images.load(path)
10. Assign to scene      → bpy.context.scene.world = world
```

---

## Workflow 4: Compositor Setup

Use when: adding post-processing effects to rendered output.

### Step-by-Step (Version-Aware)

```
1. Check Blender version → bpy.app.version
2a. If 5.0+:
    - Create node group  → bpy.data.node_groups.new(name, 'CompositorNodeTree')
    - Assign to scene    → scene.compositing_node_group = comp_tree
2b. If 3.x/4.x:
    - Enable nodes       → scene.use_nodes = True
    - Get tree           → comp_tree = scene.node_tree
3. Clear nodes           → comp_tree.nodes.clear()
4. Add Render Layers     → CompositorNodeRLayers
5. Add processing nodes  → filters, color correction, etc.
6. Add Composite output  → CompositorNodeComposite
7. Add Viewer (optional) → CompositorNodeViewer
8. Link chain            → RenderLayers → processing → Composite
```

### Version-Aware Compositor Access

```python
import bpy

def get_compositor_tree(scene, name="Compositor"):
    """Get or create the compositor tree for the given scene."""
    if bpy.app.version >= (5, 0, 0):
        tree = bpy.data.node_groups.new(name, 'CompositorNodeTree')
        scene.compositing_node_group = tree
    else:
        scene.use_nodes = True
        tree = scene.node_tree
    return tree
```

---

## Workflow 5: Custom Node Group

Use when: creating a reusable node group (any type).

### Step-by-Step

```
1. Choose tree type      → 'GeometryNodeTree', 'ShaderNodeTree', or 'CompositorNodeTree'
2. Create group          → bpy.data.node_groups.new(name, tree_type)
3. Clear nodes           → group.nodes.clear()
4. Define interface:
   a. Create panels (4.1+) → group.interface.new_panel(name)
   b. Add input sockets   → group.interface.new_socket(name, 'INPUT', type, parent=panel)
   c. Add output sockets  → group.interface.new_socket(name, 'OUTPUT', type)
   d. Set defaults/limits → iterate items_tree, set default_value/min_value/max_value
5. Add I/O nodes         → NodeGroupInput + NodeGroupOutput
6. Add processing nodes  → with positions
7. Link everything       → I/O nodes to processing nodes
```

### Using a Group Inside Another Tree

```python
# Blender 4.0+ — Embed a node group inside a tree
group_node = tree.nodes.new('GeometryNodeGroup')  # or 'ShaderNodeGroup'
group_node.node_tree = bpy.data.node_groups["MyGroup"]
group_node.location = (x, y)

# Access the group's exposed sockets on the group node
group_node.inputs["InputName"].default_value = value
tree.links.new(some_node.outputs["Out"], group_node.inputs["InputName"])
```

### Group Node Type by Tree Type

| Tree Type | Group Node `bl_idname` |
|-----------|----------------------|
| Geometry Nodes | `'GeometryNodeGroup'` |
| Shader Nodes | `'ShaderNodeGroup'` |
| Compositor Nodes | `'CompositorNodeGroup'` |

---

## Workflow 6: Nested Groups

Use when: composing complex node setups from smaller reusable groups.

### Step-by-Step

```
1. Create inner group(s)  → each a standalone node group (Workflow 5)
2. Create outer group     → another node group
3. Add inner as node      → outer.nodes.new('GeometryNodeGroup')
4. Assign inner tree      → group_node.node_tree = inner_group
5. Link outer I/O         → connect outer's GroupInput/Output to inner group's sockets
6. Set inner defaults     → group_node.inputs["Name"].default_value = value
```

### Naming Convention for AEC Groups

```
AEC_{Component}_{Function}
Examples:
  AEC_Wall_Profile
  AEC_Slab_Generator
  AEC_Column_Array
  AEC_Facade_Pattern
  AEC_Material_Brick
```

---

## Workflow 7: AEC Parametric Component

Use when: building parametric AEC elements (walls, slabs, columns) with Geometry Nodes.

### Decision Tree: Component Approach

```
Q: What geometry primitive fits?
├── Rectangular profile → GeometryNodeMeshCube (set Size inputs)
├── Circular section   → GeometryNodeMeshCylinder
├── Custom profile     → GeometryNodeCurvePrimitiveLine + mesh conversion
├── Extruded shape     → Profile curve + GeometryNodeExtrudeMesh
└── Array/Pattern      → GeometryNodeInstanceOnPoints
```

### Standard AEC Interface Sockets

For parametric AEC components, ALWAYS include these interface sockets:

```
Inputs:
  - Geometry (NodeSocketGeometry) — base geometry passthrough
  - Width/Length/Height/Thickness (NodeSocketFloat) — dimensions
  - Material (NodeSocketMaterial) — material assignment
  - Count/Spacing (NodeSocketFloat/Int) — for arrays

Outputs:
  - Geometry (NodeSocketGeometry) — result geometry
```

See `create_aec_component()` in [references/methods.md](references/methods.md) for the reusable factory function.

---

## Workflow 8: AEC Material Library

Use when: creating a set of standard AEC materials programmatically.

### Step-by-Step

```
1. Define material specs  → list of (name, base_color, roughness, metallic, ...)
2. For each material:
   a. Create material    → bpy.data.materials.new(name)
   b. Enable nodes       → mat.use_nodes = True
   c. Clear defaults     → mat.node_tree.nodes.clear()
   d. Build shader chain → Principled BSDF → Output
   e. Set properties     → base color, roughness, metallic, etc.
3. Assign to objects     → by material slot
```

### AEC Material Naming Convention

```
AEC_{Category}_{Name}
Examples:
  AEC_Concrete_Cast
  AEC_Steel_Brushed
  AEC_Glass_Clear
  AEC_Wood_Oak
  AEC_Brick_Red
  AEC_Plaster_White
```

---

## Common Utilities

See [references/methods.md](references/methods.md) for complete implementations of:
- `layout_nodes_horizontal()` / `layout_nodes_grid()` — node positioning
- `get_or_create_node_group()` — safe retrieval
- `create_socket_compat()` — version-aware socket creation
- `build_node_chain()` — sequential node chain builder
- `create_principled_material()` — quick material factory
- `embed_node_group()` — embed groups inside other trees
- `create_aec_component()` — AEC parametric component factory

---

## Node Position Guidelines

ALWAYS use consistent positioning to keep node trees readable:

| Element | Recommended X | Purpose |
|---------|--------------|---------|
| Group Input | -400 to -600 | Far left |
| Input processing | -200 to -300 | Near left |
| Core processing | -100 to 100 | Center |
| Output processing | 200 to 300 | Near right |
| Group Output | 400 to 600 | Far right |

Vertical spacing: 200 units between parallel branches.

Use `NodeFrame` nodes to visually group related nodes:

```python
frame = tree.nodes.new('NodeFrame')
frame.label = "Section Name"
node_a.parent = frame
node_b.parent = frame
```

---

## Reference Links

- [references/methods.md](references/methods.md) — Implementation helper functions and utility patterns
- [references/examples.md](references/examples.md) — Complete AEC node implementation examples
- [references/anti-patterns.md](references/anti-patterns.md) — Implementation workflow anti-patterns
- Related syntax skill: `blender-syntax-nodes` — API signatures and code patterns

### Official Sources

- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://docs.blender.org/api/current/bpy.types.GeometryNode.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
