---
name: sverchok-errors-common
description: 'Documents the 17 most common Sverchok error patterns and AI mistakes — covering data nesting errors, missing updateNode callbacks, socket data mutation, list matching misunderstandings, and IfcSverchok-specific pitfalls. Activates when writing Sverchok node code, debugging silent data corruption, fixing custom node lifecycle issues, or working with IfcSverchok entity management.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-errors-common

## Quick Reference

### Purpose

This skill documents the 17 most frequent Sverchok errors: 7 common error patterns that cause silent data corruption, crashes, or wasted computation, and 10 AI-specific mistakes that LLMs consistently make when generating Sverchok code.

### Critical Warnings

**NEVER** output flat data to `sv_set()` — ALWAYS wrap in the object-level list: `sv_set([[data]])`, not `sv_set([data])`.

**NEVER** mutate data retrieved with `sv_get(deepcopy=False)` — this corrupts the upstream node's cached output and all downstream consumers.

**NEVER** use `__init__` for node initialization — ALWAYS use `sv_init(self, context)`.

**NEVER** create sockets inside `process()` — sockets are created in `sv_init()` or `sv_update()` only.

**NEVER** use `zip()` directly on mismatched Sverchok lists — ALWAYS use `match_long_repeat()` first.

**NEVER** store IFC entity STEP IDs between IfcSverchok tree runs — `SvIfcStore.purge()` invalidates all IDs on every re-run.

**ALWAYS** use `update=updateNode` on every `bpy.props` property in a Sverchok node.

**ALWAYS** check `output.is_linked` before computing expensive results in `process()`.

### Decision Tree

```
Node produces wrong geometry silently?
├── Vertices treated as separate objects → Missing object wrapper: use [[(v1),(v2)]] not [(v1),(v2)]
├── Data appears truncated → Using zip() instead of match_long_repeat()
├── Upstream data corrupted after run → Mutating sv_get(deepcopy=False) data in-place
└── Numbers wrong but no error → Wrong nesting level for data type

Node property changes have no effect?
├── No update= callback on bpy.props → Add update=updateNode
├── Using custom lambda callback → Replace with updateNode
└── Tree.sv_process is False → Enable processing on the tree

Node errors on initialization?
├── Using __init__ instead of sv_init → Use sv_init(self, context)
├── Creating sockets in process() → Move socket creation to sv_init()
└── Wrong socket type alias in SNLite → Use single-letter aliases (v, s, m)

IfcSverchok-specific issues?
├── Entity IDs change between runs → SvIfcStore.purge() resets everything; query by GUID
├── Crash on undo → Known issue; save frequently, use "Re-run all nodes" instead
├── Data not reaching IFC nodes → Use double-nested lists: [["value"]] not ["value"]
└── Entities lost after re-run → Purge clears all entities; rebuild is by design
```

---

## Part A: 7 Common Error Patterns

### Error 1: Nesting Level Errors (Silent Data Corruption)

Wrong nesting level causes downstream nodes to misinterpret data structure. Each vertex is treated as a separate object instead of part of one object.

```python
# WRONG: Flat vertex list — each vertex becomes a separate "object"
self.outputs['Vertices'].sv_set([(0,0,0), (1,0,0)])

# CORRECT: Object-wrapped vertex list — one object with 2 vertices
self.outputs['Vertices'].sv_set([[(0,0,0), (1,0,0)]])
```

**Rule**: ALWAYS wrap output data in the object-level list. Even a single object must be `[[...]]`. See sverchok-syntax-data for full nesting rules.

### Error 2: Missing `updateNode` Callback

Property changes do not trigger node re-evaluation. The node appears stuck on its initial value.

```python
# WRONG: No update callback — property changes are ignored
my_prop: FloatProperty(name='Scale', default=1.0)

# CORRECT: updateNode dispatches PropertyEvent to the update system
from sverchok.data_structure import updateNode
my_prop: FloatProperty(name='Scale', default=1.0, update=updateNode)
```

### Error 3: Not Checking Output Connections

Unnecessary computation runs when outputs are not connected, wasting CPU time.

```python
# WRONG: Always computes, even when no downstream node reads the output
def process(self):
    data = expensive_computation()
    self.outputs['Result'].sv_set(data)

# CORRECT: Early exit when no output is connected
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return
    data = expensive_computation()
    self.outputs['Result'].sv_set(data)
```

### Error 4: Socket Data Mutation (Upstream Corruption)

Modifying input data in-place corrupts the upstream node's cached output. All other downstream nodes reading from the same source receive corrupted data.

```python
# WRONG: Mutating sv_get() data modifies the shared cache
data = self.inputs['Data'].sv_get(deepcopy=False)
data[0].append(999)  # CORRUPTS upstream node's cached output

# CORRECT: Use deepcopy=True (default) when mutating
data = self.inputs['Data'].sv_get(deepcopy=True)
data[0].append(999)  # Safe — working on an independent copy

# CORRECT: Or create new data without mutating (deepcopy=False for performance)
data = self.inputs['Data'].sv_get(deepcopy=False)
result = [sublist + [999] for sublist in data]  # New list, original untouched
```

### Error 5: List Matching Misunderstandings

Using Python's `zip()` silently truncates data to the shortest input. Sverchok's convention is to repeat the last element of shorter lists.

```python
# WRONG: zip() truncates to shortest — data is silently lost
for v, s in zip(verts, scale):
    ...

# CORRECT: match_long_repeat extends shorter lists by repeating last element
from sverchok.data_structure import match_long_repeat
verts, scale = match_long_repeat([verts, scale])
for v, s in zip(verts, scale):
    ...
```

### Error 6: IfcSverchok Undo Crashes

Blender crashes when undoing operations in IfcSverchok node trees. This is a known limitation of the IfcSverchok integration.

**Workaround**: Save frequently. Avoid undo in IfcSverchok trees. Use the "Re-run all nodes" button to restore state instead. See sverchok-impl-ifcsverchok for details.

### Error 7: IfcSverchok Purge on Re-run

`SvIfcStore.purge()` is called before every full tree update, clearing all entity data. IFC entities are recreated from scratch on every update — STEP IDs change between runs.

**Consequence**: Code that stores IFC entity IDs will break on re-run. Query entities by GUID or type instead of by STEP ID. See sverchok-impl-ifcsverchok for entity management patterns.

---

## Part B: 10 AI Common Mistakes

### AI Mistake 1: Flat Vertex Data

AI generates flat number lists instead of the required nested structure.

```python
# AI MISTAKE: Returning flat list — downstream nodes fail or misinterpret
result = [1, 2, 3]
self.outputs['Numbers'].sv_set(result)  # WRONG

# CORRECT: Always wrap in object level
self.outputs['Numbers'].sv_set([[1, 2, 3]])
```

### AI Mistake 2: Missing Object Wrapper for Vertices

AI omits the outer list that represents the object level, causing each vertex to be treated as a separate object.

```python
# AI MISTAKE: Vertices at wrong nesting level
verts = [(0,0,0), (1,0,0), (1,1,0)]
self.outputs['Vertices'].sv_set(verts)  # WRONG — each vertex = separate object

# CORRECT: Vertices at level 3 (object list > vertex list > coordinate tuple)
self.outputs['Vertices'].sv_set([[(0,0,0), (1,0,0), (1,1,0)]])
```

### AI Mistake 3: `__init__` Instead of `sv_init`

AI uses Python's standard `__init__` instead of Sverchok's `sv_init` lifecycle method. Blender node classes do not support `__init__` for socket creation.

```python
# AI MISTAKE: __init__ is NOT how Sverchok nodes initialize
def __init__(self):
    self.inputs.new(...)  # WRONG — will fail or be ignored

# CORRECT: Use sv_init — called once when node is created
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Data')
    self.outputs.new('SvStringsSocket', 'Result')
```

### AI Mistake 4: Wrong SNLite Socket Type Aliases

AI invents socket type names instead of using Sverchok's single-letter aliases.

```python
# AI MISTAKE: Invalid type aliases in SNLite header
"""
in  verts  vertices  # WRONG — "vertices" is not a valid alias
out result string    # WRONG — "string" is not a valid alias
"""

# CORRECT: Use single-letter aliases
"""
in  verts  v    # v = SvVerticesSocket
out result s    # s = SvStringsSocket
"""
# Valid aliases: s=Strings, v=Vertices, m=Matrix, c=Color
```

### AI Mistake 5: Forgetting `updateNode` on Properties

AI defines bpy.props properties without the `update=updateNode` callback, causing property changes to have no visible effect.

```python
# AI MISTAKE: Property changes are silently ignored
count: IntProperty(name='Count', default=5)  # WRONG

# CORRECT: Always include updateNode
from sverchok.data_structure import updateNode
count: IntProperty(name='Count', default=5, update=updateNode)
```

### AI Mistake 6: `zip()` Instead of `match_long_repeat`

AI uses Python's `zip()` which silently truncates mismatched lists. Sverchok's design expects the last element to be repeated.

```python
# AI MISTAKE: zip() silently drops data from longer list
for v, s in zip(verts, scale):  # WRONG — truncates to shortest

# CORRECT: match_long_repeat extends shorter lists
from sverchok.data_structure import match_long_repeat
verts, scale = match_long_repeat([verts, scale])
for v, s in zip(verts, scale):
    ...
```

### AI Mistake 7: Creating Sockets in `process()`

AI creates sockets inside `process()`, which runs on every update. This adds duplicate sockets on each evaluation.

```python
# AI MISTAKE: Creates new socket on EVERY update cycle
def process(self):
    self.inputs.new('SvStringsSocket', 'Extra')  # WRONG — duplicates accumulate

# CORRECT: Create sockets in sv_init() (once) or sv_update() (on topology change)
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Extra')
```

### AI Mistake 8: Assuming Matrix Data Is Nested Like Vertices

AI wraps matrices in double-nested lists like vertices. Matrices use a different nesting convention — they are at the object level directly.

```python
# AI MISTAKE: Treating matrices like vertices (extra nesting level)
matrices = [[Matrix(), Matrix()]]  # WRONG — over-nested

# CORRECT: Matrices are at object level (level 1), not nested further
matrices = [Matrix(), Matrix()]
self.outputs['Matrix'].sv_set(matrices)
```

### AI Mistake 9: IfcSverchok Entity ID Persistence

AI stores IFC entity STEP IDs and assumes they persist across tree re-runs. `SvIfcStore.purge()` resets all data before each full update.

```python
# AI MISTAKE: Caching STEP IDs between runs
saved_id = 42  # WRONG — this ID is invalid after next tree update

# CORRECT: Query by GlobalId (GUID) or by type, never by STEP ID
# GUIDs are stable across re-runs; STEP IDs are not
wall = ifc_file.by_guid("2O2Fr$t4X7Z8jBCV$0Oo9a")
```

See sverchok-impl-ifcsverchok for stable entity reference patterns.

### AI Mistake 10: IfcSverchok Single vs Double Nesting

AI sends single-nested data to IfcSverchok nodes, which expect standard Sverchok double-nested lists.

```python
# AI MISTAKE: Single-nested data to IFC nodes
self.outputs['Names'].sv_set(["Wall_001"])  # WRONG — flat list

# CORRECT: IfcSverchok nodes expect standard Sverchok nesting
self.outputs['Names'].sv_set([["Wall_001"]])
```

---

## Error Detection Checklist

Use this checklist when debugging Sverchok node issues:

| Symptom | Likely Error | Fix |
|---------|-------------|-----|
| Each vertex rendered as separate object | Missing object wrapper (Error 1, AI 2) | Wrap in `[[...]]` |
| Property slider has no effect | Missing `updateNode` (Error 2, AI 5) | Add `update=updateNode` |
| Data silently truncated | Using `zip()` (Error 5, AI 6) | Use `match_long_repeat()` |
| Upstream data changes unexpectedly | Socket mutation (Error 4) | Use `deepcopy=True` |
| Sockets multiply on each run | Creating sockets in `process()` (AI 7) | Move to `sv_init()` |
| Node __init__ error | Using `__init__` (AI 3) | Use `sv_init(self, context)` |
| SNLite type error | Wrong socket alias (AI 4) | Use `v`, `s`, `m`, `c` |
| Matrix transform broken | Wrong nesting (AI 8) | Use `[M1, M2]` not `[[M1, M2]]` |
| IFC entities disappear on re-run | Purge resets (Error 7, AI 9) | Query by GUID, not STEP ID |
| IFC node receives no data | Single nesting (AI 10) | Use `[["value"]]` |
| Blender crash on undo in IFC tree | IfcSverchok undo bug (Error 6) | Save often, avoid undo |
| Slow node tree builds | Not using `init_tree()` | Wrap in `with tree.init_tree():` |

---

## Cross-References

- **sverchok-syntax-data** — Full data nesting rules, socket types, and nesting level specifications per data type
- **sverchok-impl-ifcsverchok** — IfcSverchok node patterns, SvIfcStore lifecycle, entity management
- **sverchok-impl-custom-nodes** — Custom node lifecycle (sv_init, process, sv_update), registration, socket management
- **sverchok-core-concepts** — Node tree architecture, update system, event types

## Reference Links

- [references/methods.md](references/methods.md) — Error detection methods, diagnostic functions, debugging utilities
- [references/examples.md](references/examples.md) — Complete working code examples demonstrating correct patterns for each error
- [references/anti-patterns.md](references/anti-patterns.md) — Detailed anti-patterns with WHY explanations and fix strategies

### Official Sources

- https://github.com/nortikin/sverchok
- https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok
