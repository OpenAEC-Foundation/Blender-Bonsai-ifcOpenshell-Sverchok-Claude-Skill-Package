# sverchok-syntax-api: Anti-Patterns

These are confirmed error patterns for programmatic Sverchok API usage. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://github.com/nortikin/sverchok (v1.4.0+)
- vooronderzoek-sverchok.md §9

---

## AP-001: Building Trees Without init_tree()

**WHY this is wrong**: Without the `init_tree()` context manager, every call to `tree.nodes.new()` or `tree.links.new()` triggers a `TreeEvent`, causing a full tree topology re-evaluation. For a tree with N node/link additions, this results in O(N²) event processing, making programmatic tree building extremely slow.

```python
# WRONG — O(N²) updates during construction
tree = bpy.data.node_groups.new("MyTree", 'SverchCustomTreeType')
node_a = tree.nodes.new('SvBoxNodeMk2')       # TreeEvent fired
node_b = tree.nodes.new('SvViewerDrawMk4')    # TreeEvent fired again
tree.links.new(node_a.outputs[0], node_b.inputs[0])  # TreeEvent fired again
```

```python
# CORRECT — suppress updates during construction
tree = bpy.data.node_groups.new("MyTree", 'SverchCustomTreeType')
with tree.init_tree():
    node_a = tree.nodes.new('SvBoxNodeMk2')
    node_b = tree.nodes.new('SvViewerDrawMk4')
    tree.links.new(node_a.outputs[0], node_b.inputs[0])
# Single update on context manager exit
tree.force_update()
```

ALWAYS use `with tree.init_tree():` when adding multiple nodes or links. NEVER add nodes/links in a loop without suppressing updates.

---

## AP-002: Calling node.process() Directly

**WHY this is wrong**: The update system manages execution order via topological sorting, tracks execution statistics, handles errors, and propagates data between sockets. Calling `process()` directly bypasses all of this — input data may not be prepared, downstream nodes will not update, and execution statistics will not be recorded.

```python
# WRONG — bypasses update system
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

## AP-003: Not Calling force_update() After Property Changes

**WHY this is wrong**: When you set node properties via `setattr()` or direct assignment from an external script, the `updateNode` callback may not fire (depending on how the property is defined). Even if it does fire, the update may be deferred. Calling `force_update()` ensures the tree is fully recalculated with the new values before reading results.

```python
# WRONG — assumes property change triggers immediate update
tree.nodes["NumberNode"].int_ = 10
data = tree.nodes["Output"].outputs[0].sv_get()  # May return stale data
```

```python
# CORRECT — explicitly trigger update after changes
tree.nodes["NumberNode"].int_ = 10
tree.force_update()  # Ensure tree recalculates
data = tree.nodes["Output"].outputs[0].sv_get()  # Fresh data
```

ALWAYS call `tree.force_update()` after programmatic property changes. NEVER assume properties auto-update when set from external scripts.

---

## AP-004: Not Checking sv_process Before Expecting Updates

**WHY this is wrong**: When `tree.sv_process` is `False`, the tree silently ignores ALL events. No nodes are executed. Scripts that set properties and expect results without checking this flag will silently produce stale or no data.

```python
# WRONG — assumes tree is processing
tree = bpy.data.node_groups["MyTree"]
tree.nodes["NumberNode"].int_ = 5
tree.force_update()  # Does nothing if sv_process is False!
result = tree.nodes["Output"].outputs[0].sv_get()  # Stale data
```

```python
# CORRECT — ensure processing is enabled
tree = bpy.data.node_groups["MyTree"]
tree.sv_process = True  # Ensure processing is enabled
tree.nodes["NumberNode"].int_ = 5
tree.force_update()
result = tree.nodes["Output"].outputs[0].sv_get()
```

ALWAYS set `tree.sv_process = True` before expecting tree updates.

---

## AP-005: Mutating Data from sv_get(deepcopy=False)

**WHY this is wrong**: When `sv_get(deepcopy=False)` is used, the returned data is the exact same object stored in the socket data cache. Mutating it corrupts the cached output of the upstream node. All other downstream nodes reading from the same output receive corrupted data.

```python
# WRONG — mutates shared cached data
data = node.outputs['Vertices'].sv_get(deepcopy=False)
for i, verts in enumerate(data):
    data[i] = [(v[0]*2, v[1]*2, v[2]*2) for v in verts]  # CORRUPTS CACHE
```

```python
# CORRECT option 1 — use deepcopy=True (default)
data = node.outputs['Vertices'].sv_get()  # deepcopy=True by default
# Safe to mutate — working on a copy

# CORRECT option 2 — create new structure, leave original untouched
data = node.outputs['Vertices'].sv_get(deepcopy=False)
result = [[(v[0]*2, v[1]*2, v[2]*2) for v in verts] for verts in data]
# Original cache untouched, result is a new list
```

ALWAYS use `deepcopy=True` (default) if you will modify data. Use `deepcopy=False` ONLY when creating new output structures without mutating input.

---

## AP-006: Using Wrong Tree Type Identifier

**WHY this is wrong**: Sverchok nodes can ONLY exist in trees with `bl_idname='SverchCustomTreeType'`. Creating a `'ShaderNodeTree'`, `'CompositorNodeTree'`, or generic `'NodeTree'` will fail when adding Sverchok nodes — they simply do not exist in those tree type registries.

```python
# WRONG — creates wrong tree type
tree = bpy.data.node_groups.new("MyTree", 'ShaderNodeTree')
node = tree.nodes.new('SvBoxNodeMk2')  # RuntimeError: node type not found
```

```python
# CORRECT — use Sverchok's tree type
tree = bpy.data.node_groups.new("MyTree", 'SverchCustomTreeType')
node = tree.nodes.new('SvBoxNodeMk2')  # Works correctly
```

ALWAYS use `'SverchCustomTreeType'` when creating Sverchok node trees.

---

## AP-007: Not Validating Tree Type Before Access

**WHY this is wrong**: `bpy.data.node_groups` contains ALL node group types (Shader, Compositor, Geometry Nodes, Sverchok). Accessing a non-Sverchok tree and calling Sverchok-specific methods (like `force_update()` or `init_tree()`) will raise `AttributeError` or produce undefined behavior.

```python
# WRONG — no type validation
tree = bpy.data.node_groups["MyTree"]  # Could be a Shader node group
tree.force_update()  # AttributeError if not a Sverchok tree
```

```python
# CORRECT — validate tree type
tree = bpy.data.node_groups.get("MyTree")
if tree is None or tree.bl_idname != 'SverchCustomTreeType':
    raise RuntimeError("'MyTree' is not a Sverchok tree")
tree.force_update()
```

ALWAYS check `tree.bl_idname == 'SverchCustomTreeType'` before calling Sverchok-specific methods.

---

## AP-008: Flat Data Without Object Nesting

**WHY this is wrong**: ALL Sverchok socket data uses nested lists. The outer list represents objects, inner lists represent data per object. Passing flat data causes downstream nodes to misinterpret the structure — each element is treated as a separate "object" instead of data within one object.

```python
# WRONG — flat data, no object-level nesting
vertices = [(0,0,0), (1,0,0), (0,1,0)]
node.outputs['Vertices'].sv_set(vertices)
# Each vertex becomes a separate "object" with 3 numbers
```

```python
# CORRECT — properly nested
vertices = [[(0,0,0), (1,0,0), (0,1,0)]]  # 1 object, 3 vertices
node.outputs['Vertices'].sv_set(vertices)

# Multiple objects:
vertices = [
    [(0,0,0), (1,0,0)],  # Object 0
    [(2,0,0), (3,0,0)],  # Object 1
]
node.outputs['Vertices'].sv_set(vertices)
```

ALWAYS wrap socket data in an outer list representing objects. NEVER pass flat data to `sv_set()`.

---

## AP-009: Batch Processing Without Disabling Viewers

**WHY this is wrong**: Viewer nodes (`SvViewerDrawMk4`, `SvViewerBMeshMk2`) create Blender objects or OpenGL draw calls on every tree update. During a parameter sweep with hundreds of iterations, this generates hundreds of unnecessary viewport objects, massively slowing down processing and potentially crashing Blender.

```python
# WRONG — viewers create objects on every iteration
tree = bpy.data.node_groups["MyTree"]
for params in param_sets:
    setattr(tree.nodes["Num"], "int_", params)
    tree.force_update()  # Viewer creates/updates viewport objects each time
    data = tree.nodes["Output"].outputs[0].sv_get()
```

```python
# CORRECT — disable viewport display during batch processing
tree = bpy.data.node_groups["MyTree"]
tree.sv_show = False  # Suppress viewer output

for params in param_sets:
    setattr(tree.nodes["Num"], "int_", params)
    tree.force_update()
    data = tree.nodes["Output"].outputs[0].sv_get()

tree.sv_show = True  # Re-enable after sweep
tree.force_update()  # Final update with display
```

ALWAYS set `tree.sv_show = False` during batch/sweep operations. ALWAYS re-enable and force_update after the sweep.

---

## AP-010: Forgetting sv_animate for Animation-Dependent Trees

**WHY this is wrong**: By default, `tree.sv_animate` is `False`. Trees that use Frame Info nodes or other animation-dependent nodes will NOT update during animation playback unless `sv_animate` is explicitly enabled. The tree appears to work in the editor but produces static results during animation.

```python
# WRONG — tree ignores frame changes
tree = bpy.data.node_groups["AnimTree"]
# User hits Play... tree does not react to frame changes
```

```python
# CORRECT — enable animation mode
tree = bpy.data.node_groups["AnimTree"]
tree.sv_animate = True
tree.sv_process = True
```

ALWAYS set `tree.sv_animate = True` when the tree should react to frame changes. NEVER assume animation mode is enabled by default.
