---
name: sverchok-syntax-api
description: 'Guides programmatic control of Sverchok node trees from Python including creating nodes, connecting sockets, setting parameters, triggering updates, and batch processing with parameter sweeps. Use this when automating Sverchok, generating trees from scripts, running batch parametric studies, or building Sverchok workflows programmatically.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-syntax-api

## Quick Reference

### What This Skill Covers

Programmatic (external) control of Sverchok node trees via Python scripts — creating, modifying, querying, and batch-processing node trees from outside the node editor. This skill covers the **external API surface** for automating Sverchok.

- **Tree access**: `bpy.data.node_groups` filtered by `bl_idname='SverchCustomTreeType'`
- **Node creation**: `tree.nodes.new(bl_idname)` with location and property assignment
- **Link creation**: `tree.links.new(output_socket, input_socket)`
- **Update control**: `tree.force_update()`, `tree.update_nodes()`, `tree.sv_process`
- **Data extraction**: `node.outputs[name].sv_get()` for reading computed results
- **Batch processing**: Parameter sweeps with `setattr()` + `force_update()` loops
- **Performance**: `tree.init_tree()` context manager for suppressing intermediate updates
- **Data utilities**: `match_long_repeat`, `fullList`, `repeat_last` from `sverchok.data_structure`

> Cross-reference: This skill builds on `bpy.data` knowledge covered by the **blender-core-api** skill. See that skill for general Blender data-block access patterns.

### Critical Warnings

**NEVER** add nodes or links without `tree.init_tree()` context manager — every `tree.nodes.new()` and `tree.links.new()` call triggers a full tree re-evaluation without it, causing O(n²) performance.

**NEVER** call `node.process()` directly — ALWAYS use `tree.force_update()` or `tree.update_nodes()` to trigger evaluation through the update system.

**NEVER** mutate data returned by `sv_get(deepcopy=False)` — this corrupts the upstream node's cached output and poisons all downstream nodes.

**NEVER** forget to call `tree.force_update()` after programmatic property changes — `setattr()` on node properties does NOT automatically trigger the update system.

**ALWAYS** set `tree.sv_process = True` before expecting tree updates — trees with processing disabled silently ignore all events.

**ALWAYS** use `'SverchCustomTreeType'` as the tree type — using `'ShaderNodeTree'` or other Blender tree types will fail to create Sverchok nodes.

**ALWAYS** wrap socket data in nested lists — `sv_set([[(0,0,0), (1,0,0)]])` not `sv_set([(0,0,0), (1,0,0)])`.

### Decision Tree

```
Need to automate Sverchok from Python?
├── Create a new node tree
│   ├── tree = bpy.data.node_groups.new(name, 'SverchCustomTreeType')
│   └── ALWAYS wrap construction in: with tree.init_tree():
├── Access an existing tree
│   ├── tree = bpy.data.node_groups.get('TreeName')
│   └── Verify: tree.bl_idname == 'SverchCustomTreeType'
├── Add nodes to a tree
│   ├── node = tree.nodes.new('SvBoxNodeMk2')
│   ├── Set location: node.location = (x, y)
│   └── Set properties: node.property_name = value
├── Connect nodes
│   ├── By index: tree.links.new(node_a.outputs[0], node_b.inputs[0])
│   └── By name: tree.links.new(node_a.outputs['Vertices'], node_b.inputs['vertices'])
├── Trigger evaluation
│   ├── Full tree: tree.force_update()
│   ├── Specific nodes: tree.update_nodes([node_a, node_b])
│   └── First ensure: tree.sv_process = True
├── Read output data
│   ├── data = node.outputs['Name'].sv_get()
│   ├── Read-only: sv_get(deepcopy=False)
│   └── Data is ALWAYS nested: [[obj1_data], [obj2_data], ...]
└── Batch parameter sweep
    ├── Loop: setattr(node, prop, value) → tree.force_update() → sv_get()
    └── Disable viewers for speed: tree.sv_show = False
```

---

## Essential Patterns

### Pattern 1: Create a New Node Tree

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

# Create a new Sverchok tree (uses bpy.data — see blender-core-api skill)
tree = bpy.data.node_groups.new("MyParametricTree", 'SverchCustomTreeType')

# ALWAYS use init_tree() to suppress updates during construction
with tree.init_tree():
    # Add nodes by bl_idname
    num = tree.nodes.new('SvNumberNode')
    num.location = (0, 0)

    box = tree.nodes.new('SvBoxNodeMk2')
    box.location = (200, 0)

    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (400, 0)

    # Create links: output socket → input socket
    tree.links.new(num.outputs[0], box.inputs['Size'])
    tree.links.new(box.outputs['Vertices'], viewer.inputs['vertices'])
    tree.links.new(box.outputs['Edges'], viewer.inputs['edges'])
    tree.links.new(box.outputs['Faces'], viewer.inputs['faces'])

# Force initial evaluation after construction
tree.force_update()
```

### Pattern 2: Access and Query Existing Trees

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

# List all Sverchok trees (bpy.data.node_groups — see blender-core-api skill)
sv_trees = [
    ng for ng in bpy.data.node_groups
    if ng.bl_idname == 'SverchCustomTreeType'
]

# Access a specific tree with validation
tree = bpy.data.node_groups.get("MyTree")
if tree is None or tree.bl_idname != 'SverchCustomTreeType':
    raise RuntimeError("Sverchok tree 'MyTree' not found")

# Query tree state
print(f"Processing: {tree.sv_process}")
print(f"Animation: {tree.sv_animate}")
print(f"Draft: {tree.sv_draft}")
print(f"Nodes: {len(tree.nodes)}, Links: {len(tree.links)}")
```

### Pattern 3: Read Node Output Data

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyTree"]
tree.sv_process = True
tree.force_update()

node = tree.nodes["Box"]

# sv_get() returns nested lists: [[object1_data], [object2_data], ...]
try:
    vertices = node.outputs['Vertices'].sv_get(deepcopy=False)  # Read-only
    for obj_idx, verts in enumerate(vertices):
        print(f"Object {obj_idx}: {len(verts)} vertices")
except Exception:
    print("No data — node may not have been processed")

# ALWAYS use deepcopy=True (default) if you will modify the data
vertices = node.outputs['Vertices'].sv_get()  # deepcopy=True is default
```

### Pattern 4: Modify Node Properties

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyTree"]
tree.sv_process = True

# Set node properties via setattr or direct assignment
node = tree.nodes["Formula"]
node.formula1 = "sin(x) * R"

# Set properties on SNLite script nodes
snlite = tree.nodes["Scripted Node Lite"]
snlite.script_name = "my_script.py"

# ALWAYS force update after programmatic property changes
tree.force_update()
```

### Pattern 5: Batch Parameter Sweep

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def batch_sweep(tree_name, param_sets):
    """Run a parameter sweep across a Sverchok tree.

    Args:
        tree_name: Name of the Sverchok node tree
        param_sets: list of [(node_name, prop_name, value), ...]

    Returns:
        list of extracted output data per parameter set
    """
    tree = bpy.data.node_groups[tree_name]
    tree.sv_process = True
    tree.sv_show = False  # Disable viewport display for speed

    results = []
    for params in param_sets:
        # Apply all parameter values
        for node_name, prop_name, value in params:
            setattr(tree.nodes[node_name], prop_name, value)

        # Force full recalculation
        tree.force_update()

        # Extract results
        output_node = tree.nodes["Output"]
        data = output_node.outputs[0].sv_get(deepcopy=False)
        results.append(data)

    tree.sv_show = True  # Re-enable viewport display
    return results

# Usage:
# results = batch_sweep("MyTree", [
#     [("NumberNode", "int_", 5), ("ScaleNode", "factor", 2.0)],
#     [("NumberNode", "int_", 10), ("ScaleNode", "factor", 3.0)],
# ])
```

### Pattern 6: Trigger Updates Correctly

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyTree"]

# ALWAYS ensure processing is enabled first
tree.sv_process = True

# Full tree recalculation (ForceEvent)
tree.force_update()

# Update specific nodes and their downstream dependents only
tree.update_nodes([tree.nodes["NodeA"], tree.nodes["NodeB"]])

# Enable animation updates (tree reacts to frame changes)
tree.sv_animate = True

# Enable scene change updates
tree.sv_scene_update = True
```

### Pattern 7: Data Matching Utilities

```python
# Sverchok v1.4.0+
from sverchok.data_structure import match_long_repeat, fullList, repeat_last

# match_long_repeat: extend shorter lists by repeating last element
verts = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]
scales = [[2.0]]  # 1 object, but 2 objects of verts

matched = match_long_repeat([verts, scales])
# scales becomes [[2.0], [2.0]] — repeated to match 2 objects

# fullList: in-place extend a list to minimum length
my_list = [1.0, 2.0]
fullList(my_list, 5)
# my_list is now [1.0, 2.0, 2.0, 2.0, 2.0]

# repeat_last: infinite iterator that repeats the last element
gen = repeat_last([10, 20, 30])
values = [next(gen) for _ in range(6)]
# values: [10, 20, 30, 30, 30, 30]
```

---

## Common Operations

### Tree Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `tree.sv_process` | `BoolProperty` | `True` | Enable/disable tree processing |
| `tree.sv_animate` | `BoolProperty` | `False` | Process on frame change |
| `tree.sv_show` | `BoolProperty` | `True` | Show viewer outputs in viewport |
| `tree.sv_draft` | `BoolProperty` | `False` | Draft mode (reduced quality for speed) |
| `tree.sv_scene_update` | `BoolProperty` | `False` | React to scene changes |

### Tree Methods

| Method | Description |
|--------|-------------|
| `tree.force_update()` | Force complete tree recalculation (ForceEvent) |
| `tree.update_nodes(nodes)` | Update specific nodes and downstream dependents |
| `tree.init_tree()` | Context manager — suppresses updates during construction |

### Socket Data Methods

| Method | Description |
|--------|-------------|
| `socket.sv_get(default=None, deepcopy=True)` | Read data from socket cache |
| `socket.sv_set(data)` | Write data to socket cache |
| `socket.sv_forget()` | Clear cached data |

### Key Data Structure Functions

| Function | Import | Description |
|----------|--------|-------------|
| `updateNode` | `sverchok.data_structure` | Property update callback for `bpy.props` |
| `match_long_repeat(lists)` | `sverchok.data_structure` | Match list lengths by repeating last element |
| `fullList(lst, count)` | `sverchok.data_structure` | Extend list in-place to target length |
| `repeat_last(lst)` | `sverchok.data_structure` | Infinite iterator repeating last element |
| `flat_iter(data)` | `sverchok.data_structure` | Deep flatten nested structures |
| `get_data_nesting_level(data)` | `sverchok.data_structure` | Determine nesting depth |

### List Matching Modes

| Mode | Function | Behavior |
|------|----------|----------|
| `"SHORT"` | `match_short` | Truncate to shortest list |
| `"CYCLE"` | `match_long_cycle` | Cycle shorter lists |
| `"REPEAT"` | `match_long_repeat` | Repeat last element of shorter lists |
| `"XREF"` | `match_cross` | Cross-reference (all combinations) |

Access via: `from sverchok.data_structure import list_match_func`

---

## Core Architecture Files

| File | Key Exports |
|------|-------------|
| `node_tree.py` | `SverchCustomTree`, `SverchCustomTreeNode` |
| `data_structure.py` | `updateNode`, `match_long_repeat`, `fullList`, `repeat_last` |
| `core/socket_data.py` | Socket data cache: `sv_get_socket`, `sv_set_socket` |
| `core/events.py` | `TreeEvent`, `ForceEvent`, `PropertyEvent`, `AnimationEvent` |
| `core/update_system.py` | `SearchTree`, `UpdateTree`, `control_center` |
| `utils/sv_itertools.py` | Extended iteration utilities |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for tree, node, socket, and data structure methods
- [references/examples.md](references/examples.md) — Working code examples for tree automation, batch processing, data extraction
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do when scripting Sverchok, with WHY explanations

### Official Sources

- https://github.com/nortikin/sverchok
- https://github.com/nortikin/sverchok/blob/master/node_tree.py
- https://github.com/nortikin/sverchok/blob/master/data_structure.py
- https://sverchok.readthedocs.io/
