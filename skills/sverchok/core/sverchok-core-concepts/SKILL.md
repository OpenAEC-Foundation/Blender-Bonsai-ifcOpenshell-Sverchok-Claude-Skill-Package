---
name: sverchok-core-concepts
description: >
  Use when learning Sverchok fundamentals or debugging node tree execution issues.
  Prevents the common mistake of expecting immediate execution (Sverchok uses deferred
  tree-level updates, not per-node). Covers node tree architecture, data flow, socket
  data cache, update triggers, and the 18+ node categories with 500+ nodes.
  Keywords: Sverchok, node tree, data flow, socket cache, update trigger, node categories, SverchCustomTreeType, parametric design, node execution, what is Sverchok, visual programming, node-based modeling.
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-core-concepts

## Quick Reference

### What Is Sverchok

Sverchok is a parametric/algorithmic design add-on for Blender implementing a **dataflow programming paradigm** within the node editor. Data flows from output sockets to input sockets through noodle connections.

- **500+ nodes** across **18+ categories** for geometry generation, transformation, analysis
- **Python scripting**: SNLite, SN Functor B, Formula nodes for custom logic
- **Extension ecosystem**: IfcSverchok, TopologicSverchok, Sverchok-Extra
- **External API**: Full programmatic access via `bpy.data.node_groups`

### Critical Warnings

**NEVER** call `process()` directly on a node — ALWAYS use the update system via `updateNode` or `tree.force_update()`.

**NEVER** modify socket data in-place from `sv_get()` when `deepcopy=True` was not used — this mutates the upstream node's cached output and corrupts the entire downstream chain.

**NEVER** skip `init_tree()` context manager when programmatically building node trees — without it, every link/node addition triggers a full tree update, causing O(n²) performance.

**NEVER** store socket data references across frame changes — the socket data cache is cleared on each update cycle.

**ALWAYS** use `updateNode` as the `update=` callback on `bpy.props` properties — this is the ONLY correct way to trigger re-evaluation when a property changes.

**ALWAYS** check `tree.sv_process` before assuming a tree will update — trees with processing disabled silently ignore all events.

### Decision Tree

```
Need to create geometry programmatically?
├── Simple primitives → Generator nodes (Box, Sphere, Cylinder, etc.)
├── From vertex/face data → Script nodes (SNLite) or List/Vector nodes
├── Parametric/repeating → Use Number + List nodes to drive generators
└── Complex algorithm → SNLite node with custom Python

Need to debug update issues?
├── Tree not updating at all → Check tree.sv_process is True
├── Node shows red/error → Read node.US_error for the error message
├── Partial updates → Check if upstream node has data (sv_get raises SvNoDataError?)
├── Animation not working → Check tree.sv_animate is True
└── Performance issues → Check node.US_time values, enable draft mode

Need to access Sverchok trees via script?
├── Get tree → bpy.data.node_groups['TreeName'] (type: SverchCustomTreeType)
├── Get node → tree.nodes['NodeName']
├── Read output → node.outputs[0].sv_get()
├── Force update → tree.force_update()
└── Build tree → Use tree.init_tree() context manager
```

---

## Essential Patterns

### Pattern 1: Access a Sverchok Node Tree

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

# Get existing Sverchok tree
tree = bpy.data.node_groups.get("MyTree")
if tree is None or tree.bl_idname != 'SverchCustomTreeType':
    raise RuntimeError("Sverchok tree 'MyTree' not found")

# Check tree state
print(f"Processing enabled: {tree.sv_process}")
print(f"Animation mode: {tree.sv_animate}")
print(f"Draft mode: {tree.sv_draft}")
```

### Pattern 2: Build a Node Tree Programmatically

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

# Create new Sverchok tree
tree = bpy.data.node_groups.new("ParametricGrid", 'SverchCustomTreeType')

# ALWAYS use init_tree() to suppress updates during construction
with tree.init_tree():
    # Add nodes
    num_node = tree.nodes.new('SvNumberNode')
    num_node.location = (0, 0)

    grid_node = tree.nodes.new('SvPlaneNode')
    grid_node.location = (200, 0)

    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (400, 0)

    # Create links
    tree.links.new(num_node.outputs[0], grid_node.inputs[0])
    tree.links.new(grid_node.outputs['Vertices'], viewer.inputs['vertices'])

# Force initial evaluation after construction
tree.force_update()
```

### Pattern 3: Read Data from Node Outputs

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyTree"]
node = tree.nodes["MyGeneratorNode"]

# Read output socket data
# sv_get() returns nested lists: [[data_object_1], [data_object_2], ...]
try:
    vertices = node.outputs['Vertices'].sv_get()
    # vertices structure: [[(x,y,z), (x,y,z), ...], [...], ...]
    # Outer list = objects, inner list = vertices per object
    for obj_idx, obj_verts in enumerate(vertices):
        print(f"Object {obj_idx}: {len(obj_verts)} vertices")
except Exception:
    print("No data available — node may not have been processed yet")
```

### Pattern 4: Trigger Updates Correctly

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
from sverchok.data_structure import updateNode
import bpy

# On a custom node: ALWAYS use updateNode as property callback
class MyCustomNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyCustomNode'
    bl_label = 'My Custom Node'

    my_param: bpy.props.FloatProperty(
        name="Parameter",
        default=1.0,
        update=updateNode  # CORRECT: triggers PropertyEvent -> tree update
    )

    def process(self):
        # Called by the update system — NEVER call directly
        value = self.my_param
        result = [[value * i for i in range(10)]]
        self.outputs['Result'].sv_set(result)
```

### Pattern 5: Data Nesting Convention

```python
# Sverchok v1.4.0+ data nesting convention
# ALL socket data follows this structure:
# Level 0 (outermost): list of objects
# Level 1: data per object (vertices, edges, faces, values)
# Level 2+: individual data items

# Vertices: [[(x,y,z), (x,y,z), ...]]         : 1 object, N vertices
# Edges:    [[(i,j), (i,j), ...]]               : 1 object, N edges
# Faces:    [[(i,j,k,...), ...]]                 : 1 object, N faces
# Numbers:  [[1.0, 2.0, 3.0]]                   : 1 object, N values

# Multiple objects:
# Vertices: [[(v1), (v2)], [(v3), (v4)]]        : 2 objects
# match_long_repeat handles mismatched list lengths between inputs
from sverchok.data_structure import match_long_repeat

verts_list = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]
scale_list = [[2.0]]  # Only 1 value for 2 objects

# match_long_repeat repeats the shorter list to match the longer
matched = match_long_repeat([verts_list, scale_list])
# Result: verts unchanged, scale becomes [[2.0], [2.0]]
```

### Pattern 6: Socket Data Cache Operations

```python
# Sverchok v1.4.0+: core/socket_data.py
# Socket data is stored in a global dict: socket_data_cache[SockId] = data
# SockId = hash of node.node_id + socket.identifier + direction

# Writing data to an output socket (inside process())
self.outputs['Vertices'].sv_set(vertex_data)

# Reading data from an input socket (inside process())
# deepcopy=True (default): safe, returns independent copy
vertices = self.inputs['Vertices'].sv_get(deepcopy=True)

# deepcopy=False: ONLY when you will NOT mutate the data (performance gain)
vertices = self.inputs['Vertices'].sv_get(deepcopy=False)

# Reading with default value (no error if unconnected)
vertices = self.inputs['Vertices'].sv_get(default=[[]])
```

### Pattern 7: Force Tree Update

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyTree"]

# Ensure processing is enabled
tree.sv_process = True

# Force complete recalculation (ForceEvent)
tree.force_update()

# Update specific nodes only
tree.update_nodes([tree.nodes["NodeA"], tree.nodes["NodeB"]])
```

---

## Common Operations

### Node Tree Properties

| Property | Type | Purpose |
|----------|------|---------|
| `tree.sv_process` | `BoolProperty` | Enable/disable tree processing |
| `tree.sv_animate` | `BoolProperty` | Process on frame change |
| `tree.sv_show` | `BoolProperty` | Show viewer outputs in viewport |
| `tree.sv_draft` | `BoolProperty` | Draft mode (reduced quality for speed) |
| `tree.sv_scene_update` | `BoolProperty` | React to scene changes |

### Update Event Types

| Event | Trigger | Scope |
|-------|---------|-------|
| `PropertyEvent` | `updateNode` callback on `bpy.props` | Marks specific node as outdated |
| `TreeEvent` | Node/link addition or removal | Re-evaluates entire tree topology |
| `ForceEvent` | `tree.force_update()` | Resets and updates entire tree |
| `AnimationEvent` | Frame change (requires `sv_animate=True`) | Animation-dependent nodes |
| `SceneEvent` | Scene modification (requires `sv_scene_update=True`) | Scene-dependent nodes |
| `FileEvent` | New file loaded | Full tree reload |
| `UndoEvent` | Undo operation | Restores tree state |

### Node Execution Statistics

Each node records after execution:

| Attribute | Type | Description |
|-----------|------|-------------|
| `node.US_is_updated` | `bool` | Whether node executed successfully |
| `node.US_error` | `str` | Error message if execution failed |
| `node.US_error_stack` | `str` | Full traceback string |
| `node.US_time` | `float` | Execution time in seconds |
| `node.US_warning` | `str` | Warning messages from logging |

### Node Categories (18+ categories, 500+ nodes)

| Category | Description | Key Nodes |
|----------|-------------|-----------|
| **Generators** | Create geometry primitives | Box, Sphere, Cylinder, Torus, Plane, Circle, Line |
| **Transforms** | Move, Rotate, Scale | Matrix Apply, Mirror, Shear, Bend, Twist |
| **Analyzers** | Measure and analyze | Distance, Area, Volume, BBox, KDTree, Normals |
| **Modifier Make** | Construct topology | Convex Hull, Voronoi, Delaunay, Join, Bridge |
| **Modifier Change** | Alter topology | Subdivide, Dissolve, Merge, Separate, Flip |
| **Modifier Deform** | Deform geometry | Noise, Smooth, Lattice, Proportional Edit |
| **List** | List processing | Join, Split, Shift, Sort, Mask, Filter, Zip, Repeat |
| **Number** | Numeric operations | Number, Range, Random, Map Range, Formula |
| **Vector** | Vector operations | Vector In/Out, Math, Noise, Interpolation |
| **Matrix** | Matrix operations | Matrix In/Out, Apply, Multiply, Invert, Track To |
| **Logic** | Flow control | Switch, Gate, Compare, Logic, Mask |
| **Viz** | Visualization | Viewer Draw, Viewer BMesh, Stethoscope, Spreadsheet |
| **Text** | String operations | Text In/Out, CSV, JSON |
| **Scene** | Blender scene access | Object In/Out, Frame Info, Collection Picker |
| **Layout** | Node organization | Frame, Reroute, WiFi In/Out, Group |
| **Script** | Python scripting | SNLite, SN Functor B, Formula Mk5, Profile Mk3 |
| **Curve/Surface/Field** | Advanced geometry | NURBS, Bezier, Spline, Marching Cubes |
| **Solid** | CAD operations (FreeCAD) | Boolean, Fillet, Chamfer, Shell, Offset |
| **Pulga Physics** | Physics simulation | Particle-based simulations |
| **Exchange** | Import/Export | SVG, DXF, JSON, NumPy, CSV |

### Core Architecture Files

| File | Description |
|------|-------------|
| `node_tree.py` | SverchCustomTree, SverchCustomTreeNode, UpdateNodes, NodeUtils |
| `data_structure.py` | match_long_repeat, fullList, updateNode, multi_socket |
| `core/sockets.py` | All socket type definitions (SvStringsSocket, etc.) |
| `core/events.py` | Event classes (TreeEvent, AnimationEvent, etc.) |
| `core/socket_data.py` | Socket data cache (sv_get_socket, sv_set_socket) |
| `core/update_system.py` | SearchTree, UpdateTree, control_center |
| `core/socket_conversions.py` | Automatic type conversion functions |
| `utils/vectorize.py` | vectorize decorator, DataWalker |

### Node Base Class Mixins

| Mixin | Purpose |
|-------|---------|
| `UpdateNodes` | Node lifecycle (sv_init, sv_update, sv_copy, sv_free), process management |
| `NodeUtils` | Logger shortcuts, tracked UI operators, data retrieval helpers |
| `NodeDependencies` | Optional library dependency checking |
| `NodeDocumentation` | Docstring parsing, help link generation |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for SverchCustomTree, SverchCustomTreeNode, socket data cache, SearchTree, update system
- [references/examples.md](references/examples.md) — Working code examples for tree construction, data flow, custom nodes
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with Sverchok, with WHY explanations

### Official Sources

- https://github.com/nortikin/sverchok
- https://sverchok.readthedocs.io/
- https://github.com/nortikin/sverchok/wiki
