# Sverchok Code Validator — Anti-Patterns Catalog

Common anti-patterns in Sverchok node code that the validator detects. Each entry describes the anti-pattern, why it is wrong, and the correct alternative.

---

## AP-1: Flat Vertex Output

**Severity: BLOCKER** | **CHECK: 1**

```python
# ANTI-PATTERN
verts = [(0,0,0), (1,0,0), (1,1,0)]
self.outputs['Vertices'].sv_set(verts)
```

**Why wrong:** Vertices must be at nesting level 3. Missing the object-level wrapper causes downstream nodes to misinterpret each coordinate tuple as a separate object.

**Correct:**
```python
self.outputs['Vertices'].sv_set([[(0,0,0), (1,0,0), (1,1,0)]])
```

---

## AP-2: Flat Number Output

**Severity: BLOCKER** | **CHECK: 1**

```python
# ANTI-PATTERN
result = [1, 2, 3]
self.outputs['Numbers'].sv_set(result)
```

**Why wrong:** String/number sockets expect level 2 nesting. Flat lists cause mismatches with downstream nodes that expect the object-level dimension.

**Correct:**
```python
self.outputs['Numbers'].sv_set([[1, 2, 3]])
```

---

## AP-3: Silent Property (Missing updateNode)

**Severity: BLOCKER** | **CHECK: 2**

```python
# ANTI-PATTERN
count: IntProperty(name='Count', default=5)
```

**Why wrong:** Without `update=updateNode`, changing this property in the UI does not trigger node re-evaluation. The node appears frozen.

**Correct:**
```python
count: IntProperty(name='Count', default=5, update=updateNode)
```

---

## AP-4: Unmatched zip

**Severity: BLOCKER** | **CHECK: 4**

```python
# ANTI-PATTERN
verts = self.inputs['Vertices'].sv_get()
colors = self.inputs['Colors'].sv_get()
for v, c in zip(verts, colors):
    ...
```

**Why wrong:** `zip()` silently truncates to the shorter input. If `verts` has 10 objects and `colors` has 3, 7 objects are silently dropped.

**Correct:**
```python
verts, colors = match_long_repeat([verts, colors])
for v, c in zip(verts, colors):
    ...
```

---

## AP-5: Dynamic Socket Creation in process()

**Severity: BLOCKER** | **CHECK: 5**

```python
# ANTI-PATTERN
def process(self):
    self.inputs.new('SvStringsSocket', 'DynamicInput')
    ...
```

**Why wrong:** `process()` is called on every evaluation. This creates a new socket each time, accumulating duplicate sockets and corrupting the node.

**Correct:**
```python
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'DynamicInput')
```

---

## AP-6: Mutating Cached Input Data

**Severity: WARNING** | **CHECK: 6**

```python
# ANTI-PATTERN
data = self.inputs['Data'].sv_get(deepcopy=False)
data[0].append(new_value)
```

**Why wrong:** With `deepcopy=False`, `sv_get()` returns a reference to the cached data. Modifying it corrupts the cache and affects all nodes reading from the same upstream output.

**Correct:**
```python
data = self.inputs['Data'].sv_get(deepcopy=True)
data[0].append(new_value)
```

---

## AP-7: Invalid SNLite Socket Type Names

**Severity: BLOCKER** | **CHECK: 7**

```python
# ANTI-PATTERN
"""
in  verts  vertices
in  count  integer
out result string
"""
```

**Why wrong:** SNLite uses single-letter aliases. Full type names like `vertices`, `integer`, `string` are not recognized and cause silent failures or crashes.

**Correct:**
```python
"""
in  verts  v
in  count  s
out result s
"""
```

---

## AP-8: Using __init__ Instead of sv_init

**Severity: BLOCKER** | **CHECK: 5, 17**

```python
# ANTI-PATTERN
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    def __init__(self):
        self.inputs.new('SvStringsSocket', 'Data')
```

**Why wrong:** Sverchok nodes use `sv_init(self, context)` for initialization, not `__init__`. Using `__init__` bypasses Sverchok's node lifecycle management.

**Correct:**
```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Data')
```

---

## AP-9: Double-Wrapped Matrices

**Severity: BLOCKER** | **CHECK: 1**

```python
# ANTI-PATTERN
matrices = [[Matrix(), Matrix()]]
self.outputs['Matrices'].sv_set(matrices)
```

**Why wrong:** Matrices are at nesting level 1 (flat list of Matrix objects). Double-wrapping causes downstream nodes to receive a list-of-lists instead of a list-of-matrices.

**Correct:**
```python
matrices = [Matrix(), Matrix()]
self.outputs['Matrices'].sv_set(matrices)
```

---

## AP-10: No Output Connection Check

**Severity: WARNING** | **CHECK: 3**

```python
# ANTI-PATTERN
def process(self):
    heavy_data = self.compute_expensive()
    self.outputs['Result'].sv_set(heavy_data)
```

**Why wrong:** The node performs expensive computation even when no downstream node uses the output. This wastes CPU time on every tree evaluation.

**Correct:**
```python
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return
    heavy_data = self.compute_expensive()
    self.outputs['Result'].sv_set(heavy_data)
```

---

## AP-11: Missing BMesh Cleanup

**Severity: BLOCKER** | **CHECK: 11**

```python
# ANTI-PATTERN
for v, e, f in zip(verts, edges, faces):
    bm = bmesh_from_pydata(v, e, f)
    bmesh.ops.dissolve_limit(bm, ...)
    new_v, new_e, new_f = pydata_from_bmesh(bm)
    result_verts.append(new_v)
    # Missing bm.free() — memory leak on every evaluation
```

**Why wrong:** Each iteration creates a BMesh that is never freed. Over multiple evaluations, this leaks significant memory.

**Correct:**
```python
for v, e, f in zip(verts, edges, faces):
    bm = bmesh_from_pydata(v, e, f)
    bmesh.ops.dissolve_limit(bm, ...)
    new_v, new_e, new_f = pydata_from_bmesh(bm)
    bm.free()
    result_verts.append(new_v)
```

---

## AP-12: Socket Name Typo

**Severity: BLOCKER** | **CHECK: 14**

```python
# ANTI-PATTERN
def sv_init(self, context):
    self.inputs.new('SvVerticesSocket', 'Vertices')

def process(self):
    verts = self.inputs['Verts'].sv_get()  # KeyError at runtime
```

**Why wrong:** Socket names are case-sensitive strings. A mismatch between `sv_init()` and `process()` causes a `KeyError` at runtime.

**Correct:**
```python
def process(self):
    verts = self.inputs['Vertices'].sv_get()
```

---

## AP-13: sv_get Without Default on Optional Input

**Severity: WARNING** | **CHECK: 16**

```python
# ANTI-PATTERN
scale = self.inputs['Scale'].sv_get()  # Crashes when unconnected
```

**Why wrong:** When the socket is not connected and has no default, `sv_get()` raises an exception. This makes the node unusable without all inputs connected.

**Correct:**
```python
scale = self.inputs['Scale'].sv_get(default=[[1.0]])
```

---

## AP-14: Old Property Assignment Syntax

**Severity: BLOCKER** | **CHECK: 18**

```python
# ANTI-PATTERN (pre-2.80 syntax)
count = IntProperty(name='Count', default=5, update=updateNode)
```

**Why wrong:** Blender 2.80+ requires annotation syntax (`:`) for property declarations. Assignment syntax (`=`) is silently ignored — the property is never registered.

**Correct:**
```python
count: IntProperty(name='Count', default=5, update=updateNode)
```

---

## AP-15: Single-Nested IfcSverchok Data

**Severity: BLOCKER** | **CHECK: 10**

```python
# ANTI-PATTERN
self.outputs['Names'].sv_set(["Wall_001", "Wall_002"])
```

**Why wrong:** IfcSverchok nodes expect standard Sverchok nesting (level 2 for strings). Single-nesting causes IFC entity creation to fail or produce incorrect results.

**Correct:**
```python
self.outputs['Names'].sv_set([["Wall_001", "Wall_002"]])
```

---

## AP-16: Assuming Entity IDs Persist in IfcSverchok

**Severity: BLOCKER** | **CHECK: 10**

```python
# ANTI-PATTERN
saved_entity_id = 42  # Stored from previous run
entity = ifc_file.by_id(saved_entity_id)  # May not exist
```

**Why wrong:** `SvIfcStore.purge()` is called before every full tree update, clearing all entity data. IFC STEP IDs change between runs.

**Correct:**
```python
# Query by GUID or type, not by STEP ID
entity = ifc_file.by_guid("2O2Fr$t4X7Z...")
# Or query by type
walls = ifc_file.by_type("IfcWall")
```

---

## AP-17: Float Indices in Edge/Face Data

**Severity: BLOCKER** | **CHECK: 19**

```python
# ANTI-PATTERN
edges = [[(0.0, 1.0), (1.0, 2.0)]]
```

**Why wrong:** Edge and face indices must be integers. Float indices cause type errors in BMesh operations and mesh creation.

**Correct:**
```python
edges = [[(0, 1), (1, 2)]]
```

---

## AP-18: Missing Registration Attributes

**Severity: BLOCKER** | **CHECK: 17**

```python
# ANTI-PATTERN
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_label = 'My Node'
    # Missing bl_idname — registration fails
```

**Why wrong:** `bl_idname` is required by Blender for node type registration. Without it, the node class cannot be registered and the addon fails to load.

**Correct:**
```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'
```

---

## AP-19: Importing from Wrong Sverchok Module

**Severity: BLOCKER** | **CHECK: 13**

```python
# ANTI-PATTERN
from sverchok.utils import updateNode  # Wrong module
from sverchok import match_long_repeat  # Wrong module
```

**Why wrong:** These utilities live in specific modules. Wrong import paths cause `ImportError` at addon load time.

**Correct:**
```python
from sverchok.data_structure import updateNode
from sverchok.data_structure import match_long_repeat
```
