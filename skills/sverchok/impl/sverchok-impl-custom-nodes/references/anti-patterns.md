# sverchok-impl-custom-nodes — Anti-Patterns

> Sverchok v1.4.0+ / Blender 4.0+/5.x

## Anti-Pattern 1: Calling process() Directly

### Wrong

```python
# Somewhere in your code
my_node.process()  # NEVER do this
```

### Why It Fails

Calling `process()` directly bypasses the Sverchok update system. The node's execution statistics (`US_time`, `US_is_updated`, `US_error`) are not recorded, the update propagation chain is skipped, and downstream nodes are not notified. The tree state becomes inconsistent.

### Correct

```python
from sverchok.data_structure import updateNode

# Trigger via property change (inside node class)
self.my_prop = new_value  # If update=updateNode is set

# Or force tree update (from external script)
tree.force_update()

# Or update specific nodes
tree.update_nodes([node_a, node_b])
```

---

## Anti-Pattern 2: Custom Property Update Callback

### Wrong

```python
def my_custom_update(self, context):
    self.process()  # Bypasses update system
    # or
    print("value changed")  # Does nothing useful

my_prop: FloatProperty(update=my_custom_update)
```

### Why It Fails

Custom update callbacks bypass `updateNode`, which is the ONLY mechanism that triggers a `PropertyEvent` in the Sverchok update system. Without it, downstream nodes never re-evaluate, and the tree state becomes stale.

### Correct

```python
from sverchok.data_structure import updateNode

my_prop: FloatProperty(update=updateNode)
```

If you need side effects on property change, use `updateNode` and handle them in `process()`.

---

## Anti-Pattern 3: Mutating sv_get() Data Without deepcopy

### Wrong

```python
def process(self):
    verts = self.inputs['Vertices'].sv_get(deepcopy=False)
    for v in verts[0]:
        v[0] *= 2  # MUTATES upstream node's cached output
        v[1] *= 2
        v[2] *= 2
    self.outputs['Vertices'].sv_set(verts)
```

### Why It Fails

With `deepcopy=False`, `sv_get()` returns a reference to the upstream node's cached data. Mutating it corrupts the cache — every downstream node reading the same output sees corrupted data. This bug is silent and extremely difficult to debug.

### Correct

```python
def process(self):
    # Option A: Use deepcopy=True (default)
    verts = self.inputs['Vertices'].sv_get()  # deepcopy=True by default
    for v in verts[0]:
        v[0] *= 2  # Safe — working on a copy

    # Option B: Create new data instead of mutating
    verts = self.inputs['Vertices'].sv_get(deepcopy=False)
    result = [[(v[0]*2, v[1]*2, v[2]*2) for v in obj] for obj in verts]
    self.outputs['Vertices'].sv_set(result)
```

---

## Anti-Pattern 4: Creating Sockets in process()

### Wrong

```python
def process(self):
    if 'Extra' not in self.outputs:
        self.outputs.new('SvStringsSocket', 'Extra')  # Creates socket during evaluation
    self.outputs['Extra'].sv_set([[1, 2, 3]])
```

### Why It Fails

Adding sockets during `process()` triggers a `TreeEvent` (topology change), which causes the tree to re-evaluate, which calls `process()` again — creating an infinite update loop. Blender may freeze or crash.

### Correct

```python
def sv_init(self, context):
    # Create ALL known sockets here
    self.outputs.new('SvStringsSocket', 'Extra')

def sv_update(self):
    # Or manage dynamic sockets here (called on topology changes, not during evaluation)
    if some_condition and 'Dynamic' not in self.outputs:
        self.outputs.new('SvStringsSocket', 'Dynamic')

def process(self):
    if self.outputs['Extra'].is_linked:
        self.outputs['Extra'].sv_set([[1, 2, 3]])
```

---

## Anti-Pattern 5: Wrong Data Nesting Level

### Wrong

```python
def process(self):
    result = [(0,0,0), (1,0,0), (1,1,0)]  # Missing outer list levels
    self.outputs['Vertices'].sv_set(result)
```

### Why It Fails

Sverchok expects ALL socket data in nested list format: `[[data_per_object]]`. The outer list represents objects, the inner list represents data items per object. Without proper nesting, downstream nodes misinterpret the data structure — vertices become object separators, scalars become vertex components, etc.

### Correct

```python
def process(self):
    # Single object with 3 vertices
    result = [[(0,0,0), (1,0,0), (1,1,0)]]
    self.outputs['Vertices'].sv_set(result)

    # Two objects with 2 vertices each
    result = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0)]]
    self.outputs['Vertices'].sv_set(result)

    # Single scalar value
    self.outputs['Value'].sv_set([[42.0]])
```

---

## Anti-Pattern 6: Forgetting to Free BMesh

### Wrong

```python
def process(self):
    for v, e, f in zip(verts, edges, faces):
        bm = bmesh_from_pydata(v, e, f)
        result = pydata_from_bmesh(bm)
        # bm.free() missing — memory leak!
        out.append(result)
```

### Why It Fails

BMesh objects allocate C-level memory that is NOT garbage collected by Python. Each leaked BMesh persists until Blender exits. In a node that processes on every update, this can consume gigabytes of memory within minutes.

### Correct

```python
def process(self):
    for v, e, f in zip(verts, edges, faces):
        bm = bmesh_from_pydata(v, e, f)
        try:
            result = pydata_from_bmesh(bm)
            out.append(result)
        finally:
            bm.free()  # ALWAYS free, even if an exception occurs
```

---

## Anti-Pattern 7: Processing When Outputs Are Unconnected

### Wrong

```python
def process(self):
    # Computes everything even if nothing uses the output
    verts = self.inputs['Vertices'].sv_get()
    result = expensive_computation(verts)
    self.outputs['Result'].sv_set(result)
```

### Why It Fails

If no output is connected, the computed data is written to the cache but never read. For expensive operations (BMesh, NumPy, external libraries), this wastes significant CPU time on every tree update.

### Correct

```python
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return

    # Or check individual outputs for selective computation
    if not self.outputs['Result'].is_linked:
        return

    verts = self.inputs['Vertices'].sv_get()
    result = expensive_computation(verts)
    self.outputs['Result'].sv_set(result)
```

---

## Anti-Pattern 8: Using print() Instead of Node Logging

### Wrong

```python
def process(self):
    print(f"Processing {len(verts)} vertices")  # Lost in console
    print(f"ERROR: invalid input")               # Not visible in node UI
```

### Why It Fails

`print()` output goes to the Blender system console, which users rarely monitor. Warnings and errors are invisible in the node editor. The Sverchok logging system provides node-level error display (`US_error`, `US_warning`) directly in the node UI.

### Correct

```python
def process(self):
    self.debug(f"Processing {len(verts)} vertices")  # Console only (debug level)
    self.warning("Degenerate face detected")          # Shown in node UI
    self.error("Invalid input")                       # Logged as error
    self.exception("Failed with traceback")           # Includes stack trace
```

---

## Anti-Pattern 9: Silently Catching SvNoDataError

### Wrong

```python
def process(self):
    try:
        data = self.inputs['Data'].sv_get()
    except SvNoDataError:
        return  # Silently hides "no data" state
```

### Why It Fails

`SvNoDataError` is a signal to the update system that the node has no data to process. When caught silently, the node appears successful (green) when it actually has no data. The user sees no indication that the node is waiting for input.

### Correct

```python
def process(self):
    try:
        data = self.inputs['Data'].sv_get()
    except SvNoDataError:
        raise  # Re-raise to show "no data" indicator in node UI

    # Or use default to avoid the exception entirely
    data = self.inputs['Data'].sv_get(default=[[]])
```

---

## Anti-Pattern 10: Inheriting Only from bpy.types.Node

### Wrong

```python
class SvMyNode(bpy.types.Node):  # Missing SverchCustomTreeNode!
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'

    def init(self, context):  # Wrong method name (not sv_init)
        pass
```

### Why It Fails

Without `SverchCustomTreeNode`, the node lacks the `UpdateNodes`, `NodeUtils`, `NodeDependencies`, and `NodeDocumentation` mixins. It will not have `sv_init`, `process`, `sv_get`/`sv_set` integration, logging, or the update system. The node will not appear in the Sverchok node search menu and will not function in a Sverchok tree.

### Correct

```python
from sverchok.node_tree import SverchCustomTreeNode

class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'

    def sv_init(self, context):  # Correct lifecycle method
        pass
```

---

## Anti-Pattern 11: Unregistering in Wrong Order

### Wrong

```python
def register():
    bpy.utils.register_class(SvNodeA)
    bpy.utils.register_class(SvNodeB)

def unregister():
    bpy.utils.unregister_class(SvNodeA)  # Same order as register
    bpy.utils.unregister_class(SvNodeB)
```

### Why It Fails

Blender classes may have dependencies (e.g., one node references another as a pointer property). Unregistering in the same order as registration can fail if later classes depend on earlier ones. The convention is LIFO (last in, first out).

### Correct

```python
def unregister():
    bpy.utils.unregister_class(SvNodeB)  # Reverse order
    bpy.utils.unregister_class(SvNodeA)
```
