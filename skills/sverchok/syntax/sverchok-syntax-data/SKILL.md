---
name: sverchok-syntax-data
description: 'Explains Sverchok''s critical data nesting system and list matching — the #1 source of errors. Covers nesting levels for vertices (level 3), edges/faces (level 2), matrices (level 1), plus the 5 list matching modes. Use this when data looks wrong, vertices are flattened, list lengths don''t match, or you''re debugging data structure errors.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-syntax-data

## Quick Reference

### Why This Matters

Incorrect data nesting is the **#1 source of errors** in Sverchok. Every socket expects data at a specific nesting level. If you pass level-2 data where level-3 is expected, the node silently produces wrong geometry or crashes. There is no automatic correction.

### The Nesting Level Convention

```
Level 0: scalar              5.0
Level 1: list of scalars     [1, 2, 3]
Level 2: list of lists       [[1, 2, 3], [4, 5, 6]]
Level 3: list of list of lists  [[(x,y,z), (x,y,z)], [(x,y,z)]]
```

### Standard Socket Nesting Levels

| Socket Type | Required Level | Example | Mental Model |
|-------------|---------------|---------|--------------|
| `SvStringsSocket` | **2** | `[[1, 2, 3], [4, 5]]` | Objects of values |
| `SvVerticesSocket` | **3** | `[[(0,0,0), (1,0,0)], [(2,0,0)]]` | Objects of vertex lists |
| `SvMatrixSocket` | **1** | `[Matrix(), Matrix()]` | List of matrices |
| Edge data | **2** | `[[(0,1), (1,2)], [(0,1)]]` | Objects of edge index pairs |
| Face data | **2** | `[[(0,1,2)], [(0,1,2,3)]]` | Objects of face index tuples |

### Critical Warnings

**NEVER** pass a flat vertex list like `[(0,0,0), (1,0,0)]` to a `SvVerticesSocket` — this is level 2 but level 3 is required. ALWAYS wrap in an object list: `[[(0,0,0), (1,0,0)]]`.

**NEVER** pass a flat number list like `[1, 2, 3]` to a `SvStringsSocket` — this is level 1 but level 2 is required. ALWAYS wrap: `[[1, 2, 3]]`.

**NEVER** double-wrap matrices like `[[Matrix()]]` — `SvMatrixSocket` expects level 1: `[Matrix()]`.

**NEVER** wrap edges/faces at level 3 like `[[[(0,1)]]]` — edge and face data uses `SvStringsSocket` at level 2: `[[(0,1)]]`.

**NEVER** assume `sv_get()` returns flat data — it ALWAYS returns data at the socket's nesting level.

**NEVER** assume `match_long_repeat` deep-copies data — it produces shallow copies only. Mutating returned data may corrupt the original.

**NEVER** pass positional arguments to a `vectorize()`-wrapped function — it ALWAYS raises `TypeError`. Use keyword-only arguments.

**ALWAYS** wrap single-object output in the object-level list: `[[data]]` not `[data]`.

**ALWAYS** verify nesting level with `get_data_nesting_level()` when debugging unexpected results. Use `describe_data_shape()` for human-readable output.

**ALWAYS** use `ensure_min_nesting()` (not `ensure_nesting_level()`) when socket input may already be deeper than target.

### Decision Tree

```
Data looks wrong or node errors?
├── "list index out of range" → Data nesting level is wrong
│   ├── Vertices flat? → Wrap: [verts] -> [[verts]]
│   ├── Numbers flat? → Wrap: [nums] -> [[nums]]
│   └── Use get_data_nesting_level() to check
│       └── Or describe_data_shape() for readable output
├── Wrong number of objects → List matching issue
│   ├── Too many objects repeated? → Check match mode (REPEAT vs SHORT)
│   ├── Combinatorial explosion? → Switch from XREF to REPEAT
│   └── Missing objects? → SHORT mode truncated, use REPEAT
├── Single value applied to all → pre_processing = 'ONE_ITEM'
│   └── [[1,2]] and [[1],[2]] both become [1, 2] (one per object)
├── Node produces nothing → Check is_mandatory sockets are connected
└── Need automatic recursion → Use SvRecursiveNode mixin

Which matching mode to use?
├── Default / most cases → REPEAT (repeats last element)
├── Cyclic pattern needed → CYCLE (wraps around)
├── Strict pairing only → SHORT (truncates to shortest)
├── All combinations → XREF (cross product, fast cycle of long)
└── All combinations alt → XREF2 (cross product, fast cycle of short)

Which vectorization approach?
├── Custom node class → SvRecursiveNode mixin (recommended)
│   └── NOTE: Only SHORT/CYCLE/REPEAT (no XREF/XREF2)
├── Standalone function → vectorize() decorator
│   └── NOTE: Keyword-only arguments required
├── Manual control → match_long_repeat() + loop
│   └── Or zip_long_repeat() for convenient iteration
└── Simple two-list match → match_sockets() generator
```

---

## Essential Patterns

### Pattern 1: The "Objects" Mental Model

The outermost list dimension ALWAYS represents **objects** (separate geometric entities):

```python
# Sverchok v1.4.0+
# ONE object with 3 vertices
vertices = [[(0, 0, 0), (1, 0, 0), (1, 1, 0)]]

# TWO objects with different vertex counts
vertices = [
    [(0, 0, 0), (1, 0, 0), (1, 1, 0)],  # Object 0: triangle
    [(2, 0, 0), (3, 0, 0)],              # Object 1: line segment
]

# ONE object with 3 numeric values
numbers = [[1.0, 2.0, 3.0]]

# THREE objects with 1 value each
numbers = [[1.0], [2.0], [3.0]]
```

### Pattern 2: Nesting Level Detection and Debugging

```python
# Sverchok v1.4.0+
from sverchok.data_structure import get_data_nesting_level, describe_data_shape

get_data_nesting_level(5.0)                        # 0 (scalar)
get_data_nesting_level([1, 2, 3])                  # 1 (flat list)
get_data_nesting_level([[1, 2], [3, 4]])            # 2 (SvStringsSocket level)
get_data_nesting_level([[(0,0,0), (1,0,0)]])        # 3 (SvVerticesSocket level)

# SIMPLE_DATA_TYPES recognized as level-0 atoms:
# float, int, float64, int32, int64, str, Matrix

# Human-readable shape description for debugging:
describe_data_shape([[(0,0,0), (1,0,0)]])
# "Level 3: list [1] of list [2] of tuple [3] of float"

describe_data_shape([[1, 2, 3], [4, 5]])
# "Level 2: list [2] of list [3] of int"
```

### Pattern 3: List Matching with match_long_repeat

```python
# Sverchok v1.4.0+
from sverchok.data_structure import match_long_repeat

# Two vertex objects vs one scale value
verts = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0), (4,0,0)]]
scales = [[2.0]]  # 1 object

matched = match_long_repeat([verts, scales])
# verts:  unchanged (2 objects)
# scales: [[2.0], [2.0]] (repeated to 2 objects)

# Convenient zip iteration with matching:
from sverchok.data_structure import zip_long_repeat
for v, s in zip_long_repeat(verts, scales):
    process(v, s)  # Iterates 2 times, scales auto-repeated
```

### Pattern 4: All Five Matching Modes

```python
# Sverchok v1.4.0+
from sverchok.data_structure import list_match_func

a = [1, 2, 3]
b = [10, 20]

list_match_func["REPEAT"]([a, b])  # [[1,2,3], [10,20,20]]
list_match_func["CYCLE"]([a, b])   # [[1,2,3], [10,20,10]]
list_match_func["SHORT"]([a, b])   # [[1,2], [10,20]]
list_match_func["XREF"]([a, b])    # [[1,1,2,2,3,3], [10,20,10,20,10,20]]
list_match_func["XREF2"]([a, b])   # [[1,2,3,1,2,3], [10,10,10,20,20,20]]
```

### Pattern 5: SvRecursiveNode Mixin

```python
# Sverchok v1.4.0+ — the recommended vectorization approach for custom nodes
from sverchok.utils.nodes_mixins.recursive_nodes import SvRecursiveNode

class SvMyNode(SverchCustomTreeNode, bpy.types.Node, SvRecursiveNode):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'

    def sv_init(self, context):
        s_verts = self.inputs.new('SvVerticesSocket', "Vertices")
        s_verts.is_mandatory = True
        s_verts.nesting_level = 3       # vertex data
        s_verts.default_mode = 'NONE'

        s_scale = self.inputs.new('SvStringsSocket', "Scale")
        s_scale.nesting_level = 2       # numeric data
        s_scale.default_mode = 'EMPTY_LIST'
        s_scale.pre_processing = 'ONE_ITEM'  # one value per object

        self.outputs.new('SvVerticesSocket', "Vertices")

    def process_data(self, params):
        verts, scale = params
        # verts and scale are ALREADY matched and at correct nesting
        result = [(v[0]*scale, v[1]*scale, v[2]*scale) for v in verts]
        return [result]  # single output: return list

    def draw_buttons_ext(self, context, layout):
        layout.prop(self, 'list_match')  # inherited from SvRecursiveNode
```

### Pattern 6: The vectorize Decorator

```python
# Sverchok v1.4.0+
from typing import List, Tuple
from sverchok.utils.vectorize import vectorize

def scale_verts(*, vertices: List[Tuple[float, float, float]],
                   factor: float) -> list:
    return [(v[0]*factor, v[1]*factor, v[2]*factor) for v in vertices]

class SvScaleNode:
    def process(self):
        verts = self.inputs['Vertices'].sv_get()
        factors = self.inputs['Factor'].sv_get()

        fn = vectorize(scale_verts, match_mode=self.list_match)
        result = fn(vertices=verts, factor=factors)  # MUST use keyword args
        self.outputs['Vertices'].sv_set(result)
```

**Annotation nesting levels** (determines how deeply `vectorize` unwraps):
```
float, int, bool, str, Matrix  => level 0
list, tuple (bare)              => level 1
List[float]                     => level 1
List[Tuple[float, float, float]] => level 2
List[List[float]]               => level 2
```

**Return annotation:** `Tuple[list, list]` → multiple outputs. Anything else → single output.

### Pattern 7: match_sockets Generator

```python
# Sverchok v1.4.0+
from sverchok.utils.vectorize import match_sockets

verts = [[(0,0,0), (1,0,0)], [(2,0,0), (3,0,0)]]
colors = [[(1,0,0)]]  # 1 object, 1 color

for v, c in match_sockets(verts, colors):
    # Iteration 1: v=[(0,0,0),(1,0,0)], c=[(1,0,0),(1,0,0)]
    # Iteration 2: v=[(2,0,0),(3,0,0)], c=[(1,0,0),(1,0,0)]
    process(v, c)
```

### Pattern 8: Recursive Processing Utilities

```python
# Sverchok v1.4.0+
from sverchok.utils.sv_itertools import recurse_fx, recurse_fxy

# Apply function to every leaf element
result = recurse_fx([[1, 2], [3, 4]], lambda x: x * 2)
# Result: [[2, 4], [6, 8]]

# Binary operation on two nested structures (REPEAT-last matching)
result = recurse_fxy([1, 2, 3], [10, 20], lambda x, y: x + y)
# Result: [11, 22, 33] (shorter list's last element repeated)
```

---

## Common Operations

### Socket Configuration Properties (on input sockets)

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `s.nesting_level` | `int` | 2 (3 for SvVerticesSocket) | Expected nesting depth |
| `s.is_mandatory` | `bool` | `False` | Node skips if unconnected |
| `s.default_mode` | `str` | `'EMPTY_LIST'` | Default value when unconnected |
| `s.pre_processing` | `str` | `'NONE'` | Input preprocessing mode |

### Default Mode Options

| Mode | Value | Use Case |
|------|-------|----------|
| `'NONE'` | `...` (Ellipsis) | Socket must be connected or is truly optional |
| `'EMPTY_LIST'` | `[[]]` | Safe empty default for most sockets |
| `'MATRIX'` | `[Matrix()]` | Identity matrix default |
| `'MASK'` | `[[True]]` | Boolean mask default (all selected) |

### Pre-processing Options

| Mode | Behavior |
|------|----------|
| `'NONE'` | No preprocessing (default) |
| `'ONE_ITEM'` | Collapse to one value per object: `[[1,2]]` → `[1, 2]`, `[[1],[2]]` → `[1, 2]` |

### Input Socket Preprocessing Pipeline (Socket Processing Modes)

When `preprocess_input(data)` is called on a socket, transformations apply in this order:

| Step | Flag | Effect |
|------|------|--------|
| 1 | `use_flatten` | Reduce nesting by concatenating sublists |
| 2 | `use_simplify` | Mutually exclusive with flatten; simplifies structure |
| 3 | `use_graft` | Add one nesting level to each element |
| 4 | `use_unwrap` | Remove one layer of wrapping |
| 5 | `use_wrap` | Add one layer of wrapping (mutually exclusive with unwrap) |

Output sockets apply the same pipeline via `postprocess_output(data)`, plus `use_flatten_topology`.

**Note:** Socket processing flag *definitions* (what they are on socket objects) are documented in `sverchok-syntax-sockets`. This skill covers their *behavior* in the data pipeline.

### Socket Mode Display Labels

| Label | Mode |
|-------|------|
| `F` | Flatten |
| `FT` | Flatten Topology |
| `S` | Simplify |
| `G` | Graft |
| `G2` | Graft Topology (SvStringsSocket only) |
| `U` | Unwrap |
| `W` | Wrap |
| `R` | Reparametrize (SvCurveSocket only) |

### Nesting Level Adjustment Functions

| Function | Purpose |
|----------|---------|
| `ensure_nesting_level(data, target)` | Wrap data to reach target; **raises exception** if already deeper |
| `ensure_min_nesting(data, target)` | Wrap data to reach minimum; returns as-is if already deeper |
| `flatten_data(data, target)` | Reduce nesting to target by concatenating sublists |
| `graft_data(data, item_level, wrap_level)` | Add wrapping at specified nesting depth |

### Key API Surface

| Function / Class | Module | Purpose |
|-----------------|--------|---------|
| `match_long_repeat(lsts)` | `data_structure` | Match lists by repeating last element (DEFAULT mode) |
| `zip_long_repeat(*lists)` | `data_structure` | Convenience: `zip(*match_long_repeat(lists))` |
| `fullList(l, count)` | `data_structure` | Extend list in-place by repeating last element |
| `fullList_deep_copy(l, count)` | `data_structure` | Same but deep-copies repeated element (safe for mutable items) |
| `repeat_last(lst)` | `data_structure` | Infinite iterator repeating last element |
| `repeat_last_for_length(lst, count)` | `data_structure` | Return new list of exact length, repeating last |
| `get_data_nesting_level(data)` | `data_structure` | Detect nesting depth of data |
| `describe_data_shape(data)` | `data_structure` | Human-readable data shape string for debugging |
| `list_match_func[mode]` | `data_structure` | Dict of all 5 matching functions |
| `numpy_list_match_func[mode]` | `data_structure` | Dict of 3 NumPy matching functions (no XREF) |
| `vectorize(func, match_mode)` | `utils.vectorize` | Decorator for automatic vectorization |
| `match_sockets(*data)` | `utils.vectorize` | Generator matching object-level lists |
| `DataWalker` | `utils.vectorize` | Tree walker for nested data traversal |
| `walk_data(walkers, out_list)` | `utils.vectorize` | Generator driving DataWalker traversal |
| `SvRecursiveNode` | `utils.nodes_mixins.recursive_nodes` | Mixin for automatic node vectorization |
| `process_matched(...)` | `utils.sv_itertools` | Recursive matched processing engine |
| `recurse_fx(l, f)` | `utils.sv_itertools` | Unary recursive leaf application |
| `recurse_fxy(l1, l2, f)` | `utils.sv_itertools` | Binary recursive leaf application |
| `recurse_f_level_control(...)` | `utils.sv_itertools` | Level-controlled recursive processing |
| `recurse_f_multipar(params, f, matching_f)` | `utils.sv_itertools` | N-ary recursive application with matching |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for all data matching, nesting, and vectorization functions
- [references/examples.md](references/examples.md) — Working code examples for data nesting, matching, SvRecursiveNode, and vectorize
- [references/anti-patterns.md](references/anti-patterns.md) — The most common data nesting and matching mistakes, with corrections

### Source Files

- https://github.com/nortikin/sverchok/blob/master/data_structure.py
- https://github.com/nortikin/sverchok/blob/master/utils/vectorize.py
- https://github.com/nortikin/sverchok/blob/master/utils/sv_itertools.py
- https://github.com/nortikin/sverchok/blob/master/utils/nodes_mixins/recursive_nodes.py
