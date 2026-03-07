# sverchok-errors-common: Anti-Patterns

These are confirmed error patterns for Sverchok. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY. Organized by severity: data corruption first, then silent failures, then performance issues.

Sources:
- https://github.com/nortikin/sverchok (v1.4.0+)
- https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
- vooronderzoek-sverchok.md §12, §13

---

## AP-001: Flat Data Output (Missing Object Wrapper)

**WHY this is wrong**: ALL Sverchok socket data uses nested lists where the outer list represents objects. Flat data causes downstream nodes to misinterpret each element as a separate object. For vertices, each (x,y,z) tuple is treated as a single-vertex object instead of one vertex in a multi-vertex object.

```python
# WRONG — flat data, each vertex becomes a separate "object"
verts = [(0,0,0), (1,0,0), (1,1,0)]
self.outputs['Vertices'].sv_set(verts)
# Result: 3 objects with 1 vertex each (3 isolated points)
```

```python
# CORRECT — object-wrapped, one object with 3 vertices
verts = [[(0,0,0), (1,0,0), (1,1,0)]]
self.outputs['Vertices'].sv_set(verts)
# Result: 1 object with 3 vertices (a triangle)
```

ALWAYS wrap socket data in the outer object list. NEVER pass flat data to `sv_set()`. This applies to vertices, edges, faces, numbers, and strings.

---

## AP-002: Mutating sv_get(deepcopy=False) Data

**WHY this is wrong**: When `deepcopy=False` is used, the returned data IS the same Python object stored in the socket data cache. Mutating it changes the upstream node's cached output. Every other downstream node reading from the same upstream output receives the corrupted data, and the corruption persists until the upstream node re-executes.

```python
# WRONG — mutates the shared cache, corrupts upstream + all downstream consumers
data = self.inputs['Data'].sv_get(deepcopy=False)
data[0].append(999)  # This modifies the global socket_data_cache entry
```

```python
# CORRECT — use deepcopy=True (default) when mutating
data = self.inputs['Data'].sv_get(deepcopy=True)
data[0].append(999)  # Safe — independent copy

# CORRECT — or build new data without mutating (best performance)
data = self.inputs['Data'].sv_get(deepcopy=False)
result = [sublist + [999] for sublist in data]  # New list
self.outputs['Result'].sv_set(result)
```

ALWAYS use `deepcopy=True` (default) if you modify data in-place. Use `deepcopy=False` ONLY when building new output data without mutating the input.

---

## AP-003: Missing updateNode on Properties

**WHY this is wrong**: The `updateNode` function from `sverchok.data_structure` dispatches a `PropertyEvent` that marks the node as outdated and triggers the update system. Without it, the node's `process()` method is never called when the property changes. The user sees the property change in the UI but the output remains stale — a silent failure.

```python
# WRONG — property changes are silently ignored
count: IntProperty(name='Count', default=5)

# WRONG — custom callback does NOT trigger Sverchok's update system
count: IntProperty(name='Count', default=5, update=lambda s, c: None)
```

```python
# CORRECT — updateNode dispatches PropertyEvent -> process() is called
from sverchok.data_structure import updateNode
count: IntProperty(name='Count', default=5, update=updateNode)
```

ALWAYS use `update=updateNode` on every `bpy.props` property in a Sverchok node. NEVER use custom callbacks that bypass the update system.

---

## AP-004: Using __init__ Instead of sv_init

**WHY this is wrong**: Blender node classes use a registration system that does not support standard Python `__init__`. The `__init__` method is either never called or called at the wrong time. Sverchok provides `sv_init(self, context)` which is called exactly once when the node is first created, at the correct point in the node lifecycle.

```python
# WRONG — __init__ is not part of the Sverchok node lifecycle
class SvBrokenNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvBrokenNode'
    bl_label = 'Broken'

    def __init__(self):
        self.inputs.new('SvStringsSocket', 'Data')  # Never called or fails
```

```python
# CORRECT — sv_init is the node creation lifecycle method
class SvWorkingNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvWorkingNode'
    bl_label = 'Working'

    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Data')  # Called once on creation
```

ALWAYS use `sv_init(self, context)` for node initialization. NEVER use `__init__`.

---

## AP-005: Creating Sockets in process()

**WHY this is wrong**: `process()` is called on every update cycle — every property change, every upstream data change, every frame in animation mode. Creating sockets in `process()` adds duplicate sockets on each cycle. After 10 updates, the node has 10 copies of the same socket.

```python
# WRONG — accumulates duplicate sockets on every update
def process(self):
    self.inputs.new('SvStringsSocket', 'Extra')  # Duplicated each cycle
    data = self.inputs['Extra'].sv_get()
    ...
```

```python
# CORRECT — create sockets in sv_init (once on creation)
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Extra')

# Or in sv_update (on topology changes) with existence check
def sv_update(self):
    if 'Extra' not in self.inputs:
        self.inputs.new('SvStringsSocket', 'Extra')
```

ALWAYS create sockets in `sv_init()` or `sv_update()`. NEVER create sockets in `process()`.

---

## AP-006: Using zip() Instead of match_long_repeat

**WHY this is wrong**: Python's `zip()` stops at the shortest iterable, silently discarding data from longer lists. Sverchok's design expects lists to be matched by repeating the last element of shorter lists. Using `zip()` causes data loss that produces no error message — the output simply has fewer elements than expected.

```python
# WRONG — silently drops data from the longer list
verts = [[(0,0,0), (1,0,0), (2,0,0)]]  # 3 vertices
scale = [[2.0]]                          # 1 value
for v_list, s_list in zip(verts, scale):
    for v, s in zip(v_list, s_list):     # Only 1 iteration — 2 vertices lost
        ...
```

```python
# CORRECT — match_long_repeat extends shorter lists
from sverchok.data_structure import match_long_repeat
verts, scale = match_long_repeat([verts, scale])
for v_list, s_list in zip(verts, scale):
    v_list, s_list = match_long_repeat([v_list, s_list])
    for v, s in zip(v_list, s_list):     # 3 iterations — all vertices processed
        ...
```

ALWAYS use `match_long_repeat()` before `zip()` on Sverchok data. NEVER assume inputs have matching lengths.

---

## AP-007: Wrong SNLite Socket Type Aliases

**WHY this is wrong**: SNLite uses single-letter abbreviations for socket types. Using full type names like "vertices", "string", "matrix" causes a parse error or creates the wrong socket type. The node fails silently or produces type mismatches.

```python
# WRONG — these type names do not exist in SNLite
"""
in  verts  vertices   # Parse error — not a valid alias
out result string     # Parse error — not a valid alias
in  mat    matrix     # Parse error — not a valid alias
"""
```

```python
# CORRECT — use single-letter aliases
"""
in  verts  v    # v = SvVerticesSocket
out result s    # s = SvStringsSocket
in  mat    m    # m = SvMatrixSocket
"""
# Complete alias list: v=Vertices, s=Strings, m=Matrix, c=Color
```

ALWAYS use single-letter socket aliases in SNLite scripts. NEVER use full type names.

---

## AP-008: Wrong Matrix Nesting Level

**WHY this is wrong**: Matrices follow a different nesting convention than vertices. Vertices are `[[(x,y,z),...]]` (depth 3), but matrices are `[M1, M2, ...]` (depth 1). Double-nesting matrices like `[[M1, M2]]` causes downstream Matrix Apply nodes to misinterpret the data structure.

```python
# WRONG — over-nested, treats the inner list as the matrix data
from mathutils import Matrix
matrices = [[Matrix.Identity(4), Matrix.Translation((1,0,0))]]
self.outputs['Matrix'].sv_set(matrices)  # Wrong nesting
```

```python
# CORRECT — matrices at level 1
from mathutils import Matrix
matrices = [Matrix.Identity(4), Matrix.Translation((1,0,0))]
self.outputs['Matrix'].sv_set(matrices)
```

ALWAYS output matrices as a flat list `[M1, M2, ...]`. NEVER double-nest matrices.

---

## AP-009: Storing IfcSverchok STEP IDs Across Runs

**WHY this is wrong**: `SvIfcStore.purge()` is called before every full IfcSverchok tree update. This clears all entity data and recreates entities from scratch. STEP IDs (the numeric identifiers like `#42`) are reassigned during recreation and are not guaranteed to remain the same. Code that caches STEP IDs between runs will reference wrong or nonexistent entities.

```python
# WRONG — STEP ID is invalid after tree re-run
wall_id = 42
wall = ifc_file.by_id(wall_id)  # May return wrong entity or crash
```

```python
# CORRECT — use GlobalId (GUID) which persists across re-runs
wall = ifc_file.by_guid("2O2Fr$t4X7Z8jBCV$0Oo9a")

# CORRECT — query by type
walls = ifc_file.by_type("IfcWall")
```

NEVER cache IFC entity STEP IDs in IfcSverchok workflows. ALWAYS use GlobalId (GUID) or type queries for stable entity references.

---

## AP-010: Single-Nested Data to IfcSverchok Nodes

**WHY this is wrong**: IfcSverchok nodes consume data through standard Sverchok sockets, which expect the object-level outer list. Sending `["Wall_001"]` instead of `[["Wall_001"]]` causes the IFC node to misinterpret each character as a separate data item, or to fail with a type error.

```python
# WRONG — missing object-level nesting
self.outputs['Names'].sv_set(["Wall_001"])       # Each char = separate item
self.outputs['Classes'].sv_set(["IfcWall"])       # Same problem
```

```python
# CORRECT — standard Sverchok double-nesting
self.outputs['Names'].sv_set([["Wall_001"]])
self.outputs['Classes'].sv_set([["IfcWall"]])
```

ALWAYS use standard Sverchok nesting (`[[data]]`) when sending data to IfcSverchok nodes. NEVER use single-nested lists.

---

## AP-011: Not Checking Output Connections

**WHY this is wrong**: Computing results that no downstream node reads is pure waste. For nodes with expensive operations (mesh generation, IFC queries, file I/O), this causes noticeable performance degradation. Sverchok's built-in nodes consistently check `is_linked` before computing.

```python
# WRONG — expensive computation runs even when output is unused
def process(self):
    result = very_expensive_computation()
    self.outputs['Result'].sv_set(result)
```

```python
# CORRECT — skip computation when output is not connected
def process(self):
    if not self.outputs['Result'].is_linked:
        return
    result = very_expensive_computation()
    self.outputs['Result'].sv_set(result)
```

ALWAYS check `output.is_linked` before computing expensive results. ALWAYS add an early return when no outputs are connected.
