# sverchok-syntax-sockets — Anti-Patterns

Sverchok v1.4.0, Blender 4.0+/5.x
Things you MUST NOT do with Sverchok sockets, and why.

---

## Anti-Pattern 1: Mutating sv_get() Data Without deepcopy

```python
# WRONG — mutates the upstream node's cached output
def process(self):
    verts = self.inputs['Vertices'].sv_get(deepcopy=False)
    verts[0][0] = (999.0, 0.0, 0.0)  # CORRUPTS the cache
    self.outputs['Vertices'].sv_set(verts)
```

```python
# CORRECT — deepcopy=True (default) returns an independent copy
def process(self):
    verts = self.inputs['Vertices'].sv_get(deepcopy=True)  # or just sv_get()
    verts[0][0] = (999.0, 0.0, 0.0)  # Safe: modifies only local copy
    self.outputs['Vertices'].sv_set(verts)
```

WHY: `sv_get(deepcopy=False)` returns a direct reference to the data in `socket_data_cache`. Mutating it corrupts every downstream node that reads from the same upstream socket in the same update cycle.

Only use `deepcopy=False` when you are CERTAIN you will not modify the data (e.g., only reading lengths or iterating read-only).

---

## Anti-Pattern 2: No Default on sv_get()

```python
# WRONG — raises SvNoDataError when socket is unconnected
def process(self):
    verts = self.inputs['Vertices'].sv_get()  # Crashes if disconnected
    ...
```

```python
# CORRECT — provide a sensible default
def process(self):
    verts = self.inputs['Vertices'].sv_get(default=[[]])
    if not verts or not verts[0]:
        return  # Nothing to process
    ...
```

WHY: Unconnected input sockets have no cached data. Without a default, `sv_get()` raises `SvNoDataError`, which marks the node as failed (red) and halts the downstream chain. ALWAYS provide `default=[[]]` for optional inputs.

---

## Anti-Pattern 3: replace_socket() Without sv_forget()

```python
# WRONG — stale data remains in cache under old socket_id
def sv_update(self):
    sock = self.inputs['Data']
    if sock.bl_idname != 'SvCurveSocket':
        sock.replace_socket('SvCurveSocket')  # Old cache entry orphaned
```

```python
# CORRECT — clear cache first
def sv_update(self):
    sock = self.inputs.get('Data')
    if sock is not None and sock.bl_idname != 'SvCurveSocket':
        sock.sv_forget()
        new_sock = sock.replace_socket('SvCurveSocket', 'Data')
        # Use new_sock; the old sock reference is now invalid
```

WHY: Each socket has a `socket_id` based on a hash. After `replace_socket()`, the old socket is destroyed but the new socket may get a different `socket_id`. Without `sv_forget()`, the old data entry in `socket_data_cache` is never cleaned up, causing a memory leak across repeated socket replacements.

---

## Anti-Pattern 4: Using sock.other on Output Sockets With Multiple Connections

```python
# WRONG — returns only ONE arbitrary connected socket
def process(self):
    out_sock = self.outputs['Result']
    if out_sock.is_linked:
        downstream = out_sock.other  # Only one of possibly many!
        downstream_type = downstream.bl_idname  # May be wrong socket
```

```python
# CORRECT — iterate over all links for output sockets
def process(self):
    out_sock = self.outputs['Result']
    all_downstream = [link.to_socket for link in out_sock.links]
    for ds in all_downstream:
        print(f"{ds.node.name}.{ds.name}: {ds.bl_idname}")
```

WHY: `other` calls `get_other_socket()`, which for output sockets returns `self.links[0].to_socket` — always the first link only. An output can drive many nodes; iterating `sock.links` is the only reliable way to reach all of them.

---

## Anti-Pattern 5: Storing socket_id Across Update Cycles

```python
# WRONG — socket_id may become stale after undo/redo
cached_ids = {}

def process(self):
    sock = self.inputs['Data']
    sock_id = sock.socket_id
    cached_ids[sock_id] = sock.sv_get()  # External cache by socket_id

# After undo: sock.s_id is cleared (SKIP_SAVE), so socket_id is re-hashed
# The old entry in cached_ids is now orphaned
```

```python
# CORRECT — let Sverchok manage the socket data cache entirely
def process(self):
    data = self.inputs['Data'].sv_get(default=[[]])
    # Process and write to output — Sverchok handles caching internally
    self.outputs['Result'].sv_set(data)
```

WHY: `s_id` has `options={'SKIP_SAVE'}`, meaning it is not saved in `.blend` files and is cleared on undo/redo. Any external dict keyed by `socket_id` will have stale entries after undo operations. ALWAYS use the built-in `sv_get`/`sv_set` pipeline.

---

## Anti-Pattern 6: Wrong Nesting Level for Socket Type

```python
# WRONG — SvVerticesSocket expects nesting_level=3
def process(self):
    result = [(0.0, 0.0, 0.0), (1.0, 0.0, 0.0)]  # Flat list
    self.outputs['Vertices'].sv_set(result)  # Level 1 — WRONG
```

```python
# CORRECT — wrap to match nesting_level=3: [objects[verts[(x,y,z)]]]
def process(self):
    result = [[(0.0, 0.0, 0.0), (1.0, 0.0, 0.0)]]  # One object, two vertices
    self.outputs['Vertices'].sv_set(result)  # Level 2 outer + tuples = level 3
```

Nesting requirements per socket type:

| Socket | Required structure |
|---|---|
| `SvStringsSocket` | `[[val, val], [val, val], ...]` |
| `SvVerticesSocket` | `[[(x,y,z), (x,y,z)], ...]` |
| `SvMatrixSocket` | `[Matrix, Matrix, ...]` |
| `SvCurveSocket` | `[[SvCurve, SvCurve], ...]` |
| `SvScalarFieldSocket` | `[[SvScalarField, ...], ...]` |

WHY: Downstream nodes call `match_long_repeat` on inputs which assumes the outer list dimension represents "objects". If your data is not properly nested, `match_long_repeat` produces incorrect pairing and the downstream geometry is silently broken.

---

## Anti-Pattern 7: Assuming SvStringsSocket Enforces a Type

```python
# WRONG — SvStringsSocket accepts anything; no type safety
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    def sv_init(self, context):
        # This should carry SvCurve objects, but uses SvStringsSocket
        self.inputs.new('SvStringsSocket', 'Curve')

    def process(self):
        curves = self.inputs['Curve'].sv_get()
        for obj in curves[0]:
            obj.evaluate(0.5)  # AttributeError: list has no evaluate()
```

```python
# CORRECT — use the typed socket; FieldImplicitConversionPolicy handles conversions
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    def sv_init(self, context):
        self.inputs.new('SvCurveSocket', 'Curve')

    def process(self):
        curves = self.inputs['Curve'].sv_get(default=[[]])
        for obj_curves in curves:
            for curve in obj_curves:
                point = curve.evaluate(0.5)  # Correct: SvCurve object
```

WHY: `SvStringsSocket` is a lenient catch-all. It silently accepts any Python object. Using it for typed data (curves, surfaces, fields, solids) removes all implicit conversion support and produces `AttributeError` at evaluation time instead of a clear connection-time type error.

---

## Anti-Pattern 8: Calling replace_socket() in process()

```python
# WRONG — replace_socket() modifies node topology, not data
def process(self):
    data = self.inputs['Data'].sv_get()
    if isinstance(data[0][0], float):
        # WRONG: replacing sockets in process() triggers topology events
        self.inputs['Data'].replace_socket('SvStringsSocket')
```

```python
# CORRECT — topology changes belong in sv_update() or sv_init()
def sv_update(self):
    # Topology change in response to connected socket type
    data_sock = self.inputs.get('Data')
    if data_sock and data_sock.is_linked:
        from_type = data_sock.other.bl_idname
        if from_type == 'SvStringsSocket' and data_sock.bl_idname != 'SvStringsSocket':
            data_sock.sv_forget()
            data_sock.replace_socket('SvStringsSocket', 'Data')

def process(self):
    # Only read/write data here
    data = self.inputs['Data'].sv_get(default=[[]])
    self.outputs['Result'].sv_set(data)
```

WHY: `process()` is called during the data evaluation phase. Calling `replace_socket()` inside `process()` modifies the node tree topology, which fires `TreeEvent` and triggers a full tree re-evaluation while the current evaluation is still running — causing recursion, incorrect data, or crashes. Socket type decisions ALWAYS belong in `sv_update()` or `sv_init()`.

---

## Anti-Pattern 9: Connecting SvSolidSocket Without FreeCAD Guard

```python
# WRONG — SolidImplicitConversionPolicy raises if FreeCAD not installed
import bpy

tree = bpy.data.node_groups["MyTree"]
with tree.init_tree():
    bool_node = tree.nodes.new('SvSolidBooleanNode')
    mesh_node = tree.nodes.new('SvBMeshViewerNode')
    # Connecting non-solid data to SvSolidSocket without checking FreeCAD
    tree.links.new(mesh_node.outputs[0], bool_node.inputs['Solid A'])
    # Crashes at runtime: to_solid_recursive() fails, raises ImplicitConversionProhibited
```

```python
# CORRECT — guard FreeCAD availability before using solid nodes
try:
    import Part
    FREECAD_AVAILABLE = True
except ImportError:
    FREECAD_AVAILABLE = False

if FREECAD_AVAILABLE:
    with tree.init_tree():
        bool_node = tree.nodes.new('SvSolidBooleanNode')
        # ... connect solid nodes
```

WHY: `SvSolidSocket` uses `SolidImplicitConversionPolicy` which calls `to_solid_recursive()` from FreeCAD's `Part` module. If FreeCAD is not installed alongside Blender, this import fails and raises `ImplicitConversionProhibited` at connection time.

---

## Anti-Pattern 10: Setting use_* Flags Instead of allow_* Flags in sv_init()

```python
# WRONG — enables user-controlled flag without advertising it
def sv_init(self, context):
    sock = self.inputs.new('SvVerticesSocket', 'Vertices')
    sock.use_flatten = True   # Sets flag active, but UI control won't show
    # User cannot toggle this; it is permanently on with no UI indication
```

```python
# CORRECT — set allow_* to expose the flag in the UI
def sv_init(self, context):
    sock = self.inputs.new('SvVerticesSocket', 'Vertices')
    sock.allow_flatten = True  # Makes the Flatten toggle visible in socket UI
    # sock.use_flatten defaults to False; user can toggle it
```

WHY: `allow_*` properties control UI visibility of processing flags. `use_*` properties control whether the flag is active. Setting `use_*` in `sv_init()` silently activates the flag without the user knowing it is on — creating invisible data transformations that are impossible to debug. ALWAYS set `allow_*` first to make the flag discoverable.
