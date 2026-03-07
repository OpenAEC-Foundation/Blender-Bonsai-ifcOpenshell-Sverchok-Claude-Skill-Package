# sverchok-core-concepts: Anti-Patterns

These are confirmed error patterns for Sverchok. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://github.com/nortikin/sverchok (v1.4.0+)
- vooronderzoek-sverchok.md §1, §2

---

## AP-001: Calling process() Directly

**WHY this is wrong**: The update system manages execution order via topological sorting, tracks execution statistics, handles error suppression, and propagates data between sockets. Calling `process()` directly bypasses all of this — input data may not be prepared, downstream nodes will not update, and execution statistics will not be recorded.

```python
# WRONG — bypasses update system entirely
node = tree.nodes["MyNode"]
node.process()  # Input data not prepared, downstream not updated
```

```python
# CORRECT — use the update system
tree.force_update()  # Updates entire tree in correct order

# Or update specific nodes via the tree
tree.update_nodes([tree.nodes["MyNode"]])  # Updates node and downstream
```

ALWAYS use `tree.force_update()` or `tree.update_nodes()` to trigger processing. NEVER call `node.process()` directly.

---

## AP-002: Building Trees Without init_tree()

**WHY this is wrong**: Without the `init_tree()` context manager, every call to `tree.nodes.new()` or `tree.links.new()` triggers a `TreeEvent`, causing a full tree topology re-evaluation. For a tree with N nodes, this results in O(N²) event processing during construction, making programmatic tree building extremely slow.

```python
# WRONG — O(N²) updates during construction
tree = bpy.data.node_groups.new("MyTree", 'SverchCustomTreeType')
node_a = tree.nodes.new('SvBoxNodeMk2')       # TreeEvent fired
node_b = tree.nodes.new('SvViewerDrawMk4')    # TreeEvent fired again
tree.links.new(node_a.outputs[0], node_b.inputs[0])  # TreeEvent fired again
# Each event triggers full topology re-evaluation
```

```python
# CORRECT — suppress updates during construction
tree = bpy.data.node_groups.new("MyTree", 'SverchCustomTreeType')
with tree.init_tree():
    node_a = tree.nodes.new('SvBoxNodeMk2')
    node_b = tree.nodes.new('SvViewerDrawMk4')
    tree.links.new(node_a.outputs[0], node_b.inputs[0])
# Single update happens after exiting the context manager
tree.force_update()
```

ALWAYS use `with tree.init_tree():` when adding multiple nodes or links programmatically. NEVER add nodes/links in a loop without suppressing updates.

---

## AP-003: Mutating Data Retrieved with deepcopy=False

**WHY this is wrong**: When `sv_get(deepcopy=False)` is used, the returned data is the exact same object stored in the socket data cache. Mutating it modifies the cached data, corrupting the output of the upstream node. All other downstream nodes reading from the same output will receive the corrupted data.

```python
# WRONG — mutates shared cached data
vertices = self.inputs['Vertices'].sv_get(deepcopy=False)
for obj_verts in vertices:
    for i, v in enumerate(obj_verts):
        obj_verts[i] = (v[0] * 2, v[1] * 2, v[2] * 2)  # CORRUPTS UPSTREAM CACHE
self.outputs['Vertices'].sv_set(vertices)
```

```python
# CORRECT option 1 — use deepcopy=True (default)
vertices = self.inputs['Vertices'].sv_get()  # deepcopy=True is default
for obj_verts in vertices:
    for i, v in enumerate(obj_verts):
        obj_verts[i] = (v[0] * 2, v[1] * 2, v[2] * 2)  # Safe — working on copy
self.outputs['Vertices'].sv_set(vertices)

# CORRECT option 2 — create new data structure (deepcopy=False for performance)
vertices = self.inputs['Vertices'].sv_get(deepcopy=False)
result = []
for obj_verts in vertices:
    result.append([(v[0] * 2, v[1] * 2, v[2] * 2) for v in obj_verts])
self.outputs['Vertices'].sv_set(result)  # New list, original untouched
```

ALWAYS use `deepcopy=True` (default) if you will modify the data in-place. Use `deepcopy=False` ONLY when creating new output data without mutating the input.

---

## AP-004: Wrong Property Update Callback

**WHY this is wrong**: The `updateNode` function from `sverchok.data_structure` dispatches a `PropertyEvent` to the update system, which marks the node as outdated and triggers re-evaluation of the node and its downstream dependents. Using a custom callback or no callback skips this entirely — the node will not re-process when the property changes.

```python
# WRONG — custom callback does not trigger update system
from bpy.props import FloatProperty

class SvBrokenNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvBrokenNode'
    bl_label = 'Broken Node'

    my_value: FloatProperty(
        name="Value",
        update=lambda self, ctx: print("changed")  # Does NOT trigger process()
    )
```

```python
# CORRECT — use updateNode from data_structure
from sverchok.data_structure import updateNode
from bpy.props import FloatProperty

class SvWorkingNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvWorkingNode'
    bl_label = 'Working Node'

    my_value: FloatProperty(
        name="Value",
        update=updateNode  # Dispatches PropertyEvent -> triggers process()
    )
```

ALWAYS use `update=updateNode` on property definitions in Sverchok nodes. NEVER use custom lambda or function callbacks that bypass the update system.

---

## AP-005: Ignoring Data Nesting Convention

**WHY this is wrong**: ALL Sverchok socket data follows a nested list convention: the outer list represents objects, the inner lists represent data per object. Providing flat data (a single list of vertices instead of a list-of-lists) causes downstream nodes to misinterpret the data structure, producing incorrect results or errors.

```python
# WRONG — flat data, missing object-level nesting
vertices = [(0, 0, 0), (1, 0, 0), (0, 1, 0)]
self.outputs['Vertices'].sv_set(vertices)  # Missing outer list
# Downstream nodes interpret each vertex as a separate "object"
```

```python
# CORRECT — properly nested: [[object_data]]
vertices = [[(0, 0, 0), (1, 0, 0), (0, 1, 0)]]  # 1 object, 3 vertices
self.outputs['Vertices'].sv_set(vertices)

# Multiple objects:
vertices = [
    [(0, 0, 0), (1, 0, 0), (0, 1, 0)],      # Object 0
    [(2, 0, 0), (3, 0, 0), (2, 1, 0)],      # Object 1
]
self.outputs['Vertices'].sv_set(vertices)
```

ALWAYS wrap socket data in an outer list representing objects. NEVER pass flat data to `sv_set()`.

---

## AP-006: Not Checking sv_process Before Expecting Updates

**WHY this is wrong**: When `tree.sv_process` is `False`, the tree silently ignores ALL events (PropertyEvent, TreeEvent, AnimationEvent, etc.). No nodes are executed. Scripts that set properties and expect results without checking this flag will silently produce stale or no data.

```python
# WRONG — assumes tree is processing
tree = bpy.data.node_groups["MyTree"]
tree.nodes["NumberNode"].my_value = 5.0
# Expects tree to update... but if sv_process is False, nothing happens
result = tree.nodes["OutputNode"].outputs[0].sv_get()
# Returns stale data or raises SvNoDataError
```

```python
# CORRECT — ensure processing is enabled
tree = bpy.data.node_groups["MyTree"]
tree.sv_process = True  # Ensure processing is enabled
tree.nodes["NumberNode"].my_value = 5.0
tree.force_update()  # Explicitly trigger update
result = tree.nodes["OutputNode"].outputs[0].sv_get()
```

ALWAYS check and set `tree.sv_process = True` before expecting tree updates. ALWAYS call `tree.force_update()` after programmatic changes.

---

## AP-007: Forgetting sv_animate for Animation

**WHY this is wrong**: By default, Sverchok trees do NOT react to frame changes. The `sv_animate` property must be explicitly enabled. Without it, nodes that depend on frame number (like Frame Info node) will not update during animation playback, producing static results.

```python
# WRONG — tree does not update during animation
tree = bpy.data.node_groups["AnimTree"]
# User hits Play... nothing changes because sv_animate is False
```

```python
# CORRECT — enable animation mode
tree = bpy.data.node_groups["AnimTree"]
tree.sv_animate = True  # React to frame changes
tree.sv_process = True  # Ensure processing is enabled
```

ALWAYS set `tree.sv_animate = True` when the tree should react to animation frame changes. NEVER assume animation mode is enabled by default.

---

## AP-008: Using Wrong bl_idname for Node Tree

**WHY this is wrong**: Sverchok node trees use `'SverchCustomTreeType'` as their `bl_idname`. Using `'NodeTree'`, `'ShaderNodeTree'`, `'CompositorNodeTree'`, or other Blender-native tree types will create a tree that cannot contain Sverchok nodes. The Shift+A menu will not show Sverchok nodes, and Sverchok's update system will not manage the tree.

```python
# WRONG — creates a generic Blender node tree
tree = bpy.data.node_groups.new("MyTree", 'ShaderNodeTree')
node = tree.nodes.new('SvBoxNodeMk2')  # RuntimeError: node type not found
```

```python
# CORRECT — use Sverchok's tree type
tree = bpy.data.node_groups.new("MyTree", 'SverchCustomTreeType')
node = tree.nodes.new('SvBoxNodeMk2')  # Works correctly
```

ALWAYS use `'SverchCustomTreeType'` when creating Sverchok node trees. NEVER use other tree type identifiers.

---

## AP-009: Computation in sv_update Instead of process

**WHY this is wrong**: `sv_update()` is called when the node's topology changes (links added/removed). It is intended for dynamic socket management (adding/removing sockets based on connections). Putting computation logic in `sv_update()` means computation runs at the wrong time — during topology changes, not during the normal update cycle. It also means `sv_update()` runs before input data is prepared by the update system.

```python
# WRONG — computation in sv_update
class SvBadNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvBadNode'
    bl_label = 'Bad Node'

    def sv_update(self):
        # This runs on link changes, NOT during normal processing
        data = self.inputs[0].sv_get()  # May fail — data not prepared yet
        result = [d * 2 for d in data]
        self.outputs[0].sv_set(result)
```

```python
# CORRECT — computation in process(), topology in sv_update()
class SvGoodNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvGoodNode'
    bl_label = 'Good Node'

    def sv_update(self):
        # ONLY topology management here
        # Example: add/remove sockets based on connections
        pass

    def process(self):
        # ALL computation here — called by update system in correct order
        data = self.inputs[0].sv_get(default=[[]])
        result = [[d * 2 for d in obj_data] for obj_data in data]
        self.outputs[0].sv_set(result)
```

ALWAYS put computation logic in `process()`. ALWAYS limit `sv_update()` to socket management and topology reactions.

---

## AP-010: Not Checking if Output Is Linked

**WHY this is wrong**: If no downstream node is connected to an output socket, computing that output is wasted work. For expensive computations, this significantly impacts performance. Sverchok's built-in nodes consistently check `is_linked` before computing.

```python
# WRONG — computes expensive result even when nobody reads it
class SvWastefulNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvWastefulNode'
    bl_label = 'Wasteful Node'

    def process(self):
        data = self.inputs[0].sv_get()
        # Expensive computation always runs
        result_a = expensive_operation_a(data)
        result_b = expensive_operation_b(data)
        self.outputs['A'].sv_set(result_a)
        self.outputs['B'].sv_set(result_b)
```

```python
# CORRECT — skip unneeded computations
class SvEfficientNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvEfficientNode'
    bl_label = 'Efficient Node'

    def process(self):
        if not any(out.is_linked for out in self.outputs):
            return  # Nothing connected — skip entirely

        data = self.inputs[0].sv_get()

        if self.outputs['A'].is_linked:
            result_a = expensive_operation_a(data)
            self.outputs['A'].sv_set(result_a)

        if self.outputs['B'].is_linked:
            result_b = expensive_operation_b(data)
            self.outputs['B'].sv_set(result_b)
```

ALWAYS check `output.is_linked` before computing expensive results. ALWAYS return early from `process()` if no outputs are connected.
