# blender-impl-nodes: Implementation Anti-Patterns

These are confirmed implementation workflow anti-patterns for building node systems in Blender. Each entry documents the WRONG workflow, the CORRECT workflow, and the WHY. These complement the syntax-level anti-patterns in `blender-syntax-nodes/references/anti-patterns.md`.

Sources:
- https://docs.blender.org/api/current/bpy.types.NodeTree.html
- https://docs.blender.org/api/current/bpy.types.NodeTreeInterface.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- docs/research/supplementary-blender-gaps.md §1

---

## IAP-001: Not Clearing Nodes Before Building From Scratch

**WHY this is wrong**: When building a complete node tree programmatically, failing to clear existing nodes results in duplicate nodes. For materials, `use_nodes = True` creates default Principled BSDF + Material Output. Adding your own copies creates ambiguity about which output is active and wastes memory.

```python
# WRONG — duplicates default nodes
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
# Default tree already has Principled BSDF + Material Output
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')  # Now there are TWO
output = tree.nodes.new('ShaderNodeOutputMaterial')       # Now there are TWO
```

```python
# CORRECT — clear defaults before building
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()  # Remove all default nodes
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
output = tree.nodes.new('ShaderNodeOutputMaterial')
```

ALWAYS call `tree.nodes.clear()` when building a complete node tree from scratch. If MODIFYING an existing tree, use `tree.nodes.get("NodeName")` to find and modify existing nodes instead.

---

## IAP-002: Skipping Node Positions (layout)

**WHY this is wrong**: Without explicit `location` values, all nodes are created at position `(0, 0)`. They stack on top of each other in the node editor, making the tree completely unreadable for users who open it. Even though the tree functions correctly, it is unusable for inspection or manual editing.

```python
# WRONG — all nodes at (0, 0), tree is unreadable
group_in = group.nodes.new('NodeGroupInput')
transform = group.nodes.new('GeometryNodeTransform')
group_out = group.nodes.new('NodeGroupOutput')
```

```python
# CORRECT — explicit positions for readability
group_in = group.nodes.new('NodeGroupInput')
group_in.location = (-400, 0)
transform = group.nodes.new('GeometryNodeTransform')
transform.location = (0, 0)
group_out = group.nodes.new('NodeGroupOutput')
group_out.location = (400, 0)
```

ALWAYS set `node.location = (x, y)` for every node. Use consistent spacing (200-300 units horizontal, 150-200 units vertical).

---

## IAP-003: Defining Interface Sockets Without Adding Group I/O Nodes

**WHY this is wrong**: Interface sockets (defined via `interface.new_socket()`) only appear on `NodeGroupInput` and `NodeGroupOutput` special nodes. Without these nodes in the tree, the interface sockets exist in the definition but have no physical representation — data cannot flow in or out of the group.

```python
# WRONG — interface defined but no I/O nodes to expose it
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
group.nodes.clear()
group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')
# No NodeGroupInput or NodeGroupOutput added — sockets are invisible
```

```python
# CORRECT — interface sockets AND I/O nodes
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
group.nodes.clear()
group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

group_in = group.nodes.new('NodeGroupInput')
group_in.location = (-300, 0)
group_out = group.nodes.new('NodeGroupOutput')
group_out.location = (300, 0)
```

ALWAYS add `NodeGroupInput` and `NodeGroupOutput` nodes when creating node groups. Define interface sockets BEFORE adding I/O nodes so the sockets appear on them immediately.

---

## IAP-004: Setting Modifier Inputs Before Assigning Node Group

**WHY this is wrong**: Modifier inputs are derived from the node group's interface. If you try to set inputs before assigning `modifier.node_group`, the modifier has no interface to reference. The input identifiers don't exist yet, and setting values has no effect or raises errors.

```python
# WRONG — node group not yet assigned
mod = obj.modifiers.new("GeoNodes", type='NODES')
mod["Socket_2"] = 5.0  # No node group → no sockets → no effect
mod.node_group = group  # Now assigned, but previous setting is lost
```

```python
# CORRECT — assign group first, then set inputs
mod = obj.modifiers.new("GeoNodes", type='NODES')
mod.node_group = group  # Assign first
# Now iterate interface to find identifiers and set values
for item in group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.name == "Width":
            mod[item.identifier] = 5.0
```

ALWAYS assign `modifier.node_group` BEFORE setting any modifier input values.

---

## IAP-005: Hardcoding Modifier Input Identifiers

**WHY this is wrong**: Modifier input identifiers (`Socket_0`, `Socket_1`, etc.) are auto-generated and depend on the order sockets are created. If sockets are reordered, added, or removed, the identifiers change. Hardcoded identifiers silently set the wrong input or create custom properties instead.

```python
# WRONG — hardcoded identifiers break when interface changes
mod["Socket_2"] = 5.0   # Was "Width" yesterday, might be "Height" today
mod["Socket_3"] = 3.0
```

```python
# CORRECT — dynamic lookup by name
for item in mod.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        if item.name == "Width":
            mod[item.identifier] = 5.0
        elif item.name == "Height":
            mod[item.identifier] = 3.0
```

NEVER hardcode `Socket_N` identifiers. ALWAYS iterate `interface.items_tree` and match by `item.name` to find the correct `item.identifier`.

---

## IAP-006: Using bpy.ops.node for Scripted Node Tree Construction

**WHY this is wrong**: `bpy.ops.node.*` operators require the Node Editor to be the active area with the correct node tree displayed. They fail with `RuntimeError: Operator bpy.ops.node.add_node.poll() failed` in scripts, background mode, or when the Node Editor isn't visible. The direct `tree.nodes.new()` API works in all contexts.

```python
# WRONG — requires active Node Editor UI context
bpy.ops.node.add_node(type='ShaderNodeBsdfPrincipled')
bpy.ops.node.links_cut(path=[])
```

```python
# CORRECT — direct API, no UI context needed
tree = mat.node_tree
node = tree.nodes.new('ShaderNodeBsdfPrincipled')
tree.links.new(node.outputs["BSDF"], output.inputs["Surface"])
```

ALWAYS use `tree.nodes.new()`, `tree.links.new()`, `tree.nodes.remove()` for scripted node operations. ONLY use `bpy.ops.node.*` when building a Node Editor addon that runs inside the editor context.

---

## IAP-007: Not Checking Version Before Using Interface Panels

**WHY this is wrong**: `node_group.interface.new_panel()` was added in Blender 4.1. Calling it in Blender 4.0 raises `AttributeError`. Code that uses panels without version checking breaks on 4.0.

```python
# WRONG — fails on Blender 4.0
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
panel = group.interface.new_panel("Settings")  # AttributeError in 4.0
```

```python
# CORRECT — version guard
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
if bpy.app.version >= (4, 1, 0):
    panel = group.interface.new_panel("Settings")
    group.interface.new_socket(
        name="Value", in_out='INPUT',
        socket_type='NodeSocketFloat', parent=panel
    )
else:
    group.interface.new_socket(
        name="Value", in_out='INPUT',
        socket_type='NodeSocketFloat'
    )
```

ALWAYS check `bpy.app.version >= (4, 1, 0)` before using `interface.new_panel()`. If targeting 4.0, add sockets without panels.

---

## IAP-008: Creating Duplicate Node Groups Without Checking

**WHY this is wrong**: Calling `bpy.data.node_groups.new()` always creates a NEW node group, even if one with the same name already exists. Blender appends `.001`, `.002` suffixes. This leads to orphaned duplicates that waste memory and confuse users.

```python
# WRONG — creates duplicates on every call
def setup_wall():
    group = bpy.data.node_groups.new("AEC_Wall", 'GeometryNodeTree')
    # ... build the tree
    return group
# Calling setup_wall() 3 times creates: AEC_Wall, AEC_Wall.001, AEC_Wall.002
```

```python
# CORRECT — check for existing group first
def setup_wall():
    group = bpy.data.node_groups.get("AEC_Wall")
    if group is not None:
        return group  # Reuse existing
    group = bpy.data.node_groups.new("AEC_Wall", 'GeometryNodeTree')
    # ... build the tree
    return group
```

ALWAYS check `bpy.data.node_groups.get(name)` before creating a new node group. Use `get_or_create` pattern to avoid duplicates.

---

## IAP-009: Wrong Workflow Order — Links Before Sockets Exist

**WHY this is wrong**: Attempting to create links before interface sockets are defined means the Group Input / Group Output nodes have no output / input sockets to link from/to. The socket names won't exist yet, causing `KeyError`.

```python
# WRONG — trying to link before interface is defined
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
group.nodes.clear()
group_in = group.nodes.new('NodeGroupInput')
group_out = group.nodes.new('NodeGroupOutput')
transform = group.nodes.new('GeometryNodeTransform')

# This fails: group_in has no "Geometry" output yet
group.links.new(group_in.outputs["Geometry"], transform.inputs["Geometry"])  # KeyError!

# Interface defined too late
group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
```

```python
# CORRECT — define interface BEFORE linking to I/O nodes
group = bpy.data.node_groups.new("MyGroup", 'GeometryNodeTree')
group.nodes.clear()

# 1. Define interface first
group.interface.new_socket(name="Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

# 2. Add I/O nodes (sockets appear because interface is defined)
group_in = group.nodes.new('NodeGroupInput')
group_out = group.nodes.new('NodeGroupOutput')
transform = group.nodes.new('GeometryNodeTransform')

# 3. Link — now "Geometry" exists on group_in/group_out
group.links.new(group_in.outputs["Geometry"], transform.inputs["Geometry"])
group.links.new(transform.outputs["Geometry"], group_out.inputs["Geometry"])
```

ALWAYS define interface sockets BEFORE creating links that reference Group Input / Group Output sockets. The recommended order is: 1) create group, 2) clear nodes, 3) define interface, 4) add I/O nodes, 5) add processing nodes, 6) link.

---

## IAP-010: Forgetting to Enable use_nodes for Materials/Worlds

**WHY this is wrong**: Newly created materials and worlds have `use_nodes = False` by default. Without enabling nodes, `material.node_tree` and `world.node_tree` return `None`. Any attempt to add nodes to `None` raises `AttributeError`.

```python
# WRONG — node_tree is None
mat = bpy.data.materials.new("MyMat")
tree = mat.node_tree  # None!
tree.nodes.new('ShaderNodeBsdfPrincipled')  # AttributeError
```

```python
# CORRECT
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True   # Creates the default node tree
tree = mat.node_tree   # Now a valid NodeTree
```

ALWAYS set `use_nodes = True` before accessing `node_tree` on materials and worlds. This is NOT needed for Geometry Nodes (they use `bpy.data.node_groups` directly).

---

## IAP-011: Building Compositor Without Version Check

**WHY this is wrong**: The compositor API changed fundamentally in Blender 5.0. In 3.x/4.x, the compositor lives on `scene.node_tree`. In 5.0+, it uses `scene.compositing_node_group` with a regular node group. Code that targets only one version breaks on the other.

```python
# WRONG — only works on 3.x/4.x, breaks on 5.0+
scene.use_nodes = True
comp_tree = scene.node_tree  # AttributeError in 5.0
```

```python
# WRONG — only works on 5.0+, breaks on 3.x/4.x
comp_tree = bpy.data.node_groups.new("Comp", 'CompositorNodeTree')
scene.compositing_node_group = comp_tree  # AttributeError in 3.x/4.x
```

```python
# CORRECT — version-aware
if bpy.app.version >= (5, 0, 0):
    comp_tree = bpy.data.node_groups.new("Comp", 'CompositorNodeTree')
    scene.compositing_node_group = comp_tree
else:
    scene.use_nodes = True
    comp_tree = scene.node_tree
```

ALWAYS check `bpy.app.version` before accessing the compositor tree. Use a helper function (see `get_compositor_tree()` in methods.md) to abstract this.

---

## IAP-012: Not Setting Socket Defaults After Interface Creation

**WHY this is wrong**: Interface sockets created with `interface.new_socket()` use Blender's built-in defaults (0.0 for floats, 0 for ints). For AEC components, these defaults are almost never useful. Users see "0.0" for wall height, requiring manual input before the node does anything meaningful.

```python
# WRONG — useless defaults
group.interface.new_socket(name="Height", in_out='INPUT', socket_type='NodeSocketFloat')
# Default is 0.0 — a wall with 0 height is invisible
```

```python
# CORRECT — meaningful defaults with limits
group.interface.new_socket(name="Height", in_out='INPUT', socket_type='NodeSocketFloat')
for item in group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT' and item.name == "Height":
        item.default_value = 3.0   # Reasonable wall height
        item.min_value = 0.1       # Prevent zero/negative
        item.max_value = 50.0      # Reasonable upper bound
```

ALWAYS set `default_value`, `min_value`, and `max_value` on numeric interface sockets for AEC components. Provide sensible architectural defaults.
