# sverchok-syntax-data: Anti-Patterns

> The most common data nesting and list matching mistakes in Sverchok.
> Every anti-pattern includes the WRONG code, WHY it fails, and the CORRECT fix.
> All examples verified against Sverchok v1.4.0+ source code. Blender 4.0+/5.x.

---

## Anti-Pattern 1: Flat Vertex List (Missing Object Wrapper)

### WRONG

```python
# Passing vertices without the object-level list
vertices = [(0, 0, 0), (1, 0, 0), (1, 1, 0)]
self.outputs['Vertices'].sv_set(vertices)
```

### WHY IT FAILS

SvVerticesSocket expects **level 3** data: a list of objects, where each object is a list of (x,y,z) tuples. The code above is level 2 — Sverchok interprets each tuple as a separate "object" containing a list of 3 numbers, not as 3 vertices of one object.

### CORRECT

```python
# ALWAYS wrap in the object-level list
vertices = [[(0, 0, 0), (1, 0, 0), (1, 1, 0)]]
self.outputs['Vertices'].sv_set(vertices)
```

---

## Anti-Pattern 2: Flat Number List for SvStringsSocket

### WRONG

```python
# Passing numbers as a flat list
numbers = [1.0, 2.0, 3.0]
self.outputs['Values'].sv_set(numbers)
```

### WHY IT FAILS

SvStringsSocket expects **level 2** data: `[[1.0, 2.0, 3.0]]`. A flat list at level 1 causes downstream nodes to misinterpret 3 values as 3 separate objects with 1 value each, producing wrong matching behavior.

### CORRECT

```python
numbers = [[1.0, 2.0, 3.0]]  # One object, three values
self.outputs['Values'].sv_set(numbers)
```

---

## Anti-Pattern 3: Double-Wrapping Matrix Data

### WRONG

```python
from mathutils import Matrix
# Wrapping matrices in an extra list
matrices = [[Matrix(), Matrix.Translation((1,0,0))]]
self.outputs['Matrix'].sv_set(matrices)
```

### WHY IT FAILS

SvMatrixSocket expects **level 1** data: `[Matrix(), Matrix()]`. Adding an extra `[]` creates level 2, which causes matrix operations to fail or produce identity matrices instead of the intended transforms.

### CORRECT

```python
from mathutils import Matrix
matrices = [Matrix(), Matrix.Translation((1,0,0))]
self.outputs['Matrix'].sv_set(matrices)
```

---

## Anti-Pattern 4: Using ensure_nesting_level When Data Might Be Deeper

### WRONG

```python
from sverchok.data_structure import ensure_nesting_level

# This RAISES TypeError if data is already at level 3+
data = self.inputs['Data'].sv_get()
data = ensure_nesting_level(data, 2)  # Crashes if data is [[[1,2]]]
```

### WHY IT FAILS

`ensure_nesting_level()` raises `TypeError` if the data is already deeper than `target_level`. When reading from a socket, the data might be at any nesting level depending on what is connected upstream.

### CORRECT

```python
from sverchok.data_structure import ensure_min_nesting

# ensure_min_nesting returns data as-is if already deeper
data = self.inputs['Data'].sv_get()
data = ensure_min_nesting(data, 2)  # Safe: [[[1,2]]] -> [[[1,2]]] (no change)
```

---

## Anti-Pattern 5: Forgetting is_mandatory on Required Sockets

### WRONG

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    def sv_init(self, context):
        v = self.inputs.new('SvVerticesSocket', "Vertices")
        v.nesting_level = 3
        # Missing: v.is_mandatory = True

        self.outputs.new('SvVerticesSocket', "Result")
```

### WHY IT FAILS

Without `is_mandatory = True`, SvRecursiveNode's `process()` will attempt to execute even when the Vertices input is not connected. It will use the `default_mode` value (which defaults to `'EMPTY_LIST'` = `[[]]`), causing the node to process empty data instead of gracefully skipping.

### CORRECT

```python
v = self.inputs.new('SvVerticesSocket', "Vertices")
v.is_mandatory = True    # Node will skip process() if unconnected
v.nesting_level = 3
v.default_mode = 'NONE'  # No default — must be connected
```

---

## Anti-Pattern 6: Mutating sv_get() Data Without deepcopy

### WRONG

```python
def process(self):
    verts = self.inputs['Vertices'].sv_get(deepcopy=False)
    # Modifying in place corrupts the upstream node's cache!
    for i, obj_verts in enumerate(verts):
        for j, v in enumerate(obj_verts):
            verts[i][j] = (v[0] * 2, v[1] * 2, v[2] * 2)
    self.outputs['Vertices'].sv_set(verts)
```

### WHY IT FAILS

With `deepcopy=False`, `sv_get()` returns a reference to the upstream node's cached output data. Modifying it in-place corrupts the cache, causing all other nodes reading from the same output to receive corrupted data. This is a silent data corruption bug.

### CORRECT

```python
def process(self):
    # Option A: Use deepcopy=True (default)
    verts = self.inputs['Vertices'].sv_get(deepcopy=True)
    # Safe to modify

    # Option B: Use deepcopy=False but create new data
    verts = self.inputs['Vertices'].sv_get(deepcopy=False)
    new_verts = [[(v[0]*2, v[1]*2, v[2]*2) for v in obj] for obj in verts]
    self.outputs['Vertices'].sv_set(new_verts)  # New object, no mutation
```

---

## Anti-Pattern 7: Ignoring match_long_repeat for Multi-Input Nodes

### WRONG

```python
def process(self):
    verts = self.inputs['Vertices'].sv_get()    # 3 objects
    scales = self.inputs['Scale'].sv_get()       # 1 object

    # Direct zip loses objects 2 and 3!
    result = []
    for v, s in zip(verts, scales):
        result.append([(x*s[0], y*s[0], z*s[0]) for x, y, z in v])
    self.outputs['Vertices'].sv_set(result)
```

### WHY IT FAILS

`zip()` truncates to the shortest input. With 3 vertex objects and 1 scale object, only the first object is processed. The user expects the single scale to apply to all 3 objects.

### CORRECT

```python
from sverchok.data_structure import match_long_repeat

def process(self):
    verts = self.inputs['Vertices'].sv_get()
    scales = self.inputs['Scale'].sv_get()

    # Match lists before processing
    verts, scales = match_long_repeat([verts, scales])

    result = []
    for v, s in zip(verts, scales):
        result.append([(x*s[0], y*s[0], z*s[0]) for x, y, z in v])
    self.outputs['Vertices'].sv_set(result)
```

**Better alternative:** Use `SvRecursiveNode` which handles matching automatically. Or use `zip_long_repeat(verts, scales)` for convenient iteration with matching.

---

## Anti-Pattern 8: Using Positional Arguments with vectorize()

### WRONG

```python
from sverchok.utils.vectorize import vectorize

def my_func(*, vertices: list, factor: float) -> list:
    return [v * factor for v in vertices]

fn = vectorize(my_func)
result = fn(verts_data, factor_data)  # POSITIONAL ARGS — CRASHES!
```

### WHY IT FAILS

The `vectorize` decorator explicitly raises `TypeError` if positional arguments are passed. This is by design — the decorator uses keyword names to match arguments to their type annotations for nesting level detection.

### CORRECT

```python
fn = vectorize(my_func)
result = fn(vertices=verts_data, factor=factor_data)  # KEYWORD-ONLY
```

---

## Anti-Pattern 9: Using XREF/XREF2 Without Understanding Combinatorial Explosion

### WRONG

```python
from sverchok.data_structure import list_match_func

# 100 vertices cross-referenced with 100 scales
# produces 100 * 100 = 10,000 combinations!
verts = list(range(100))
scales = list(range(100))
result = list_match_func["XREF"]([verts, scales])
# Each output list now has 10,000 elements
```

### WHY IT FAILS

XREF and XREF2 compute the Cartesian product via `itertools.product()`. With N and M elements, the output has N*M elements. This causes memory and performance issues with large inputs. Most use cases actually need REPEAT, not XREF.

### CORRECT

```python
# Use REPEAT unless you specifically need all combinations
result = list_match_func["REPEAT"]([verts, scales])
# Output: 100 elements each (last of shorter list repeated)

# If you DO need cross-reference, be aware of the size:
# XREF with [10, 10] inputs = 100 outputs (manageable)
# XREF with [100, 100] inputs = 10,000 outputs (problematic)
# XREF with [1000, 1000] inputs = 1,000,000 outputs (crash risk)
```

---

## Anti-Pattern 10: Wrong process_data Return for Multiple Outputs

### WRONG

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    def sv_init(self, context):
        # ... inputs ...
        self.outputs.new('SvVerticesSocket', "Vertices")
        self.outputs.new('SvStringsSocket', "Count")

    def process_data(self, params):
        verts, = params
        count = len(verts)
        # WRONG: returning as flat list, not per-output
        return [verts, count]
```

### WHY IT FAILS

With multiple outputs, `process_data` must return a tuple/list where each element corresponds to one output socket. The code above returns a 2-element list, but `verts` itself is a list of vertices, so the unpacking into outputs will be wrong.

### CORRECT

```python
def process_data(self, params):
    verts, = params
    count = len(verts)
    # Return as separate results: (output_0_data, output_1_data)
    return verts, [count]
```

---

## Anti-Pattern 11: Not Freeing BMesh Objects

### WRONG

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    build_bmesh = True
    bmesh_inputs = [0, 1, 2]

    def process_data(self, params):
        bmesh_list, other = params
        results = []
        for bm in bmesh_list:
            results.append(len(bm.verts))
            # Missing: bm.free() — memory leak!
        return results
```

### WHY IT FAILS

BMesh objects consume significant memory and are not garbage collected by Python. Without `bm.free()`, each node update leaks memory, eventually crashing Blender.

### CORRECT

```python
def process_data(self, params):
    bmesh_list, other = params
    results = []
    for bm in bmesh_list:
        results.append(len(bm.verts))
        bm.free()  # ALWAYS free BMesh objects
    return results
```

---

## Anti-Pattern 12: Confusing levels_of_list_or_np with get_data_nesting_level

### WRONG

```python
from sverchok.data_structure import levels_of_list_or_np

# Using levels_of_list_or_np for user-facing nesting detection
data = [[(0,0,0), (1,0,0)]]
level = levels_of_list_or_np(data)
# Returns 2 (counts containment levels only)
# But get_data_nesting_level returns 3 (counts tuples as a level)
```

### WHY IT FAILS

`levels_of_list_or_np` counts only list/tuple containment depth and ONLY inspects the first element at each level. It returns immediately after the first recursion. It does NOT treat tuples-of-numbers as "atomic data" like `get_data_nesting_level` does. The two functions have different semantics.

- `levels_of_list_or_np` is used internally by `process_matched()` and `DataWalker` for recursion control.
- `get_data_nesting_level` is the correct function for detecting data format.

### CORRECT

```python
from sverchok.data_structure import get_data_nesting_level

data = [[(0,0,0), (1,0,0)]]
level = get_data_nesting_level(data)  # 3 — correct for vertex data
```

---

## Anti-Pattern 13: Assuming list_match on SvRecursiveNode Supports XREF

### WRONG

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    def draw_buttons(self, context, layout):
        layout.prop(self, 'list_match')

    # User expects to select XREF mode in the UI...
    # But SvRecursiveNode.list_match uses numpy_list_match_modes
    # which only has SHORT, CYCLE, REPEAT — no XREF or XREF2!
```

### WHY IT FAILS

The `list_match` EnumProperty on SvRecursiveNode uses `numpy_list_match_modes = list_match_modes[:3]`, which only includes SHORT, CYCLE, and REPEAT. XREF and XREF2 are NOT available through this mixin.

### CORRECT

If you need XREF support, either:
1. Use `list_match_func` directly with manual processing instead of SvRecursiveNode.
2. Define your own `list_match` EnumProperty using the full `list_match_modes`.

```python
# Manual approach with full mode support:
from sverchok.data_structure import list_match_func, list_match_modes

class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    list_match: EnumProperty(
        name="List Match",
        items=list_match_modes,  # All 5 modes
        default="REPEAT",
        update=updateNode
    )

    def process(self):
        matching_f = list_match_func[self.list_match]
        # Manual matching with full mode support
```

---

## Anti-Pattern 14: Nesting Edge/Face Data at Level 3

### WRONG

```python
# Wrapping edges like vertices (level 3)
edges = [[[(0, 1), (1, 2), (2, 0)]]]  # Level 3 — WRONG for edges
faces = [[[(0, 1, 2)]]]               # Level 3 — WRONG for faces
```

### WHY IT FAILS

Edge and face data use `SvStringsSocket` which expects **level 2**, not level 3. Only vertex data (`SvVerticesSocket`) uses level 3. Extra wrapping causes nodes to interpret the structure incorrectly.

### CORRECT

```python
# Edges and faces at level 2
edges = [[(0, 1), (1, 2), (2, 0)]]    # Level 2 — correct
faces = [[(0, 1, 2)]]                  # Level 2 — correct

# Vertices at level 3
vertices = [[(0,0,0), (1,0,0), (0,1,0)]]  # Level 3 — correct
```

---

## Anti-Pattern 15: Using fullList on Mutable Objects Without Deep Copy

### WRONG

```python
from sverchok.data_structure import fullList

# Extending a list of mutable lists
data = [[1, 2], [3, 4]]
fullList(data, 4)
# data = [[1, 2], [3, 4], [3, 4], [3, 4]]
# BUT: data[2], data[3] are the SAME object as data[1]!
data[2].append(99)
# data is now [[1, 2], [3, 4, 99], [3, 4, 99], [3, 4, 99]]
```

### WHY IT FAILS

`fullList` copies references to the last element, not the element itself. For immutable objects (numbers, tuples) this is fine. For mutable objects (lists, dicts), all extended elements share the same reference — mutating one mutates all.

### CORRECT

```python
from sverchok.data_structure import fullList_deep_copy

data = [[1, 2], [3, 4]]
fullList_deep_copy(data, 4)
# Each repeated element is an independent deep copy
data[2].append(99)
# data = [[1, 2], [3, 4], [3, 4, 99], [3, 4]]  (only data[2] affected)
```

---

## Summary: Quick Reference Table

| # | Anti-Pattern | Symptom | Fix |
|---|-------------|---------|-----|
| 1 | Flat vertex list | Wrong geometry or crash | Wrap: `[verts]` → `[[verts]]` |
| 2 | Flat number list | Wrong matching | Wrap: `[nums]` → `[[nums]]` |
| 3 | Double-wrapped matrices | Identity matrices | Remove wrapper: `[[M]]` → `[M]` |
| 4 | ensure_nesting_level on unknown data | TypeError crash | Use `ensure_min_nesting` instead |
| 5 | Missing is_mandatory | Node runs on empty data | Set `s.is_mandatory = True` |
| 6 | Mutating sv_get(deepcopy=False) | Silent data corruption | Use deepcopy=True or create new data |
| 7 | zip() instead of match_long_repeat | Lost objects | Use match_long_repeat or SvRecursiveNode |
| 8 | Positional args to vectorize | TypeError | Use keyword-only arguments |
| 9 | XREF on large inputs | Memory explosion | Use REPEAT unless cross-product needed |
| 10 | Wrong multi-output return | Garbled outputs | Return tuple of (out1, out2, ...) |
| 11 | Not freeing BMesh | Memory leak | ALWAYS call bm.free() |
| 12 | levels_of_list_or_np for detection | Wrong level count | Use get_data_nesting_level |
| 13 | Expecting XREF on SvRecursiveNode | Mode not available | Use manual matching with full modes |
| 14 | Level-3 edges/faces | Misinterpreted topology | Edges/faces are level 2, not 3 |
| 15 | fullList on mutable objects | Shared references | Use fullList_deep_copy |
