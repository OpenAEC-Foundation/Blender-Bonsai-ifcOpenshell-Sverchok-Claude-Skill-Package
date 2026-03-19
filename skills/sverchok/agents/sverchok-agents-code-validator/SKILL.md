---
name: sverchok-agents-code-validator
description: >
  Use when reviewing, validating, or auditing Sverchok node code for correctness. Runs 19
  automated checks covering data nesting correctness, updateNode callbacks, list matching
  patterns, socket consistency, import validation, and IfcSverchok compliance. Prevents
  shipping node code with silent data corruption or missing update triggers.
  Keywords: Sverchok validation, code review, data nesting check, updateNode check,
  socket consistency, import validation, IfcSverchok compliance, code quality.
license: MIT
compatibility: "Designed for Claude Code. Requires Python 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
dependencies:
  - sverchok-errors-common
  - sverchok-impl-custom-nodes
  - sverchok-impl-ifcsverchok
---

# Sverchok Code Validator Agent

Systematic validation checklist for reviewing Sverchok node code. Run this checklist against any Sverchok Python code to identify errors, anti-patterns, and correctness issues. Targets Sverchok v1.4.0 on Blender 4.0+.

## Quick Reference — When to Activate

Activate this validator when:
- Reviewing, auditing, or validating Sverchok node scripts
- Checking custom node code before deployment
- Generating new Sverchok node code (run validator on output)
- Investigating data corruption, silent failures, or incorrect geometry
- Reviewing SNLite scripts, SN Functor B scripts, or custom node classes

## Validation Checklist

Run each check in order. Each check has a severity level:
- **BLOCKER**: Code will crash, produce data corruption, or cause silent failures
- **WARNING**: Code has a bug, performance issue, or missing best practice
- **INFO**: Code works but does not follow Sverchok conventions

---

### CHECK 1: Data Nesting Correctness

**Severity: BLOCKER**

Verify that all `sv_set()` calls use the correct nesting level for their socket type.

**Required nesting levels:**

| Socket Type | Required Level | Example |
|-------------|---------------|---------|
| `SvVerticesSocket` | 3 | `[[(x,y,z), (x,y,z)]]` |
| `SvStringsSocket` | 2 | `[[1, 2, 3]]` |
| Edges | 2 | `[[(0,1), (1,2)]]` |
| Faces | 2 | `[[(0,1,2)]]` |
| `SvMatrixSocket` | 1 | `[Matrix(), Matrix()]` |

**Detection**: Find all `self.outputs[...].sv_set(...)` calls. Trace the argument back to its construction. Verify the outermost list depth matches the socket type.

**Flag BLOCKER if:**
- Vertices are output at level 2 (missing object wrapper): `sv_set([(0,0,0), (1,0,0)])`
- Strings/edges/faces are output at level 1 (flat list): `sv_set([1, 2, 3])`
- Matrices are output at level 2 (double-wrapped): `sv_set([[Matrix()]])`

---

### CHECK 2: updateNode Callback on All Properties

**Severity: BLOCKER**

Every `bpy.props.*Property` declaration that affects node output MUST include `update=updateNode`.

**Detection**: Find all lines matching `*Property(` (FloatProperty, IntProperty, BoolProperty, EnumProperty, StringProperty, FloatVectorProperty, IntVectorProperty, BoolVectorProperty). Verify each contains `update=updateNode`.

**Flag BLOCKER if**: Any property lacks `update=updateNode`.

**Exception**: Properties that only control UI display (not computation) may omit `updateNode`. Flag INFO instead if the property name suggests UI-only use (e.g., `show_options`, `expand_ui`).

---

### CHECK 3: Output Connection Check

**Severity: WARNING**

The `process()` method SHOULD exit early if no outputs are connected.

**Detection**: Check the first lines of `process()` for:
```python
if not any(s.is_linked for s in self.outputs):
    return
```

**Flag WARNING if**: `process()` does not check output connections before performing computation. This causes unnecessary computation when no downstream node uses the output.

---

### CHECK 4: match_long_repeat Before zip

**Severity: BLOCKER**

When iterating over multiple inputs with `zip()`, inputs MUST first be matched using `match_long_repeat()`.

**Detection**: Find all `zip(...)` calls in `process()` where arguments are socket input data. Check that `match_long_repeat()` was called on those inputs before the `zip()`.

**Flag BLOCKER if**: `zip()` is used on socket inputs without prior `match_long_repeat()`. Plain `zip()` silently truncates data to the shortest input.

---

### CHECK 5: Socket Creation Only in sv_init/sv_update

**Severity: BLOCKER**

Socket creation (`self.inputs.new(...)` or `self.outputs.new(...)`) MUST only occur in `sv_init()` or `sv_update()`.

**Detection**: Find all `.inputs.new(` and `.outputs.new(` calls. Verify each occurs inside `sv_init` or `sv_update` methods.

**Flag BLOCKER if**: Socket creation occurs inside `process()`. This creates a new socket on every evaluation, corrupting the node.

---

### CHECK 6: deepcopy for Input Data

**Severity: WARNING**

Input data retrieved via `sv_get()` SHOULD NOT be mutated in-place without `deepcopy=True`.

**Detection**: Find `sv_get()` calls. If the returned data is subsequently modified (`.append()`, `[i] = ...`, `del`, `.pop()`, `.extend()`, `.insert()`, `.sort()`, `.reverse()`), verify that `deepcopy=True` was passed (or that a manual copy was made).

**Flag WARNING if**: Input data is mutated without deepcopy. This corrupts cached data and affects upstream nodes.

**Note**: `sv_get(deepcopy=True)` is the default in recent Sverchok versions. Flag only if `deepcopy=False` is explicitly set and data is mutated.

---

### CHECK 7: Correct SNLite Aliases

**Severity: BLOCKER**

In SNLite header declarations, socket type aliases MUST use the correct single-letter codes.

**Valid aliases:**

| Alias | Socket Type |
|-------|-------------|
| `s` | `SvStringsSocket` |
| `v` | `SvVerticesSocket` |
| `m` | `SvMatrixSocket` |
| `o` | `SvObjectSocket` |
| `C` | `SvCurveSocket` |
| `S` | `SvSurfaceSocket` |
| `So` | `SvSolidSocket` |
| `SF` | `SvScalarFieldSocket` |
| `VF` | `SvVectorFieldSocket` |
| `D` | `SvDictionarySocket` |
| `FP` | `SvFilePathSocket` |

**Flag BLOCKER if**: SNLite header uses invalid aliases such as `vertices`, `string`, `matrix`, `float`, `int`, `vector`.

---

### CHECK 8: Node Docstring Format

**Severity: INFO**

Custom node classes SHOULD include a docstring with `Triggers:` and `Tooltip:` lines.

**Expected format:**
```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: keyword1 keyword2
    Tooltip: Short description
    """
```

**Flag INFO if**: Docstring is missing, or `Triggers:` / `Tooltip:` lines are absent. These are used by Sverchok's node search (Shift+S).

---

### CHECK 9: Standard Process Method Pattern

**Severity: WARNING**

The `process()` method SHOULD follow the standard 5-step pattern:
1. Early exit if no output connected
2. Read inputs with `sv_get(default=...)`
3. Match input lengths with `match_long_repeat()`
4. Process each object in a loop
5. Set outputs with `sv_set()`

**Flag WARNING if**: `process()` deviates significantly (e.g., missing default values on `sv_get`, no list iteration, direct scalar output without wrapping).

---

### CHECK 10: IfcSverchok Double-Nesting

**Severity: BLOCKER**

Data sent to IfcSverchok nodes MUST use standard Sverchok nesting (level 2 for strings).

**Detection**: If the node tree contains IfcSverchok nodes (bl_idname starting with `SvIfc`), verify that all data passed to IFC node inputs follows Sverchok nesting conventions.

**Flag BLOCKER if**: Single-nested data is passed to IFC nodes:
```python
# WRONG
self.outputs['Names'].sv_set(["Wall_001"])
# CORRECT
self.outputs['Names'].sv_set([["Wall_001"]])
```

---

### CHECK 11: BMesh Cleanup (bm.free())

**Severity: BLOCKER**

Every `bmesh.new()` or `bmesh_from_pydata()` call MUST have a corresponding `bm.free()` call.

**Detection**: Track BMesh variable assignments. Verify each has a `bm.free()` call after the last usage. Exception: BMesh obtained via `bmesh.from_edit_mesh()` must NOT be freed manually.

**Flag BLOCKER if**: A standalone BMesh is created without `bm.free()`. This leaks memory.

---

### CHECK 12: NumPy Optimization Opportunities

**Severity: INFO**

Flag opportunities where Python loops over vertex/numeric data could be replaced with NumPy vectorized operations.

**Detection**: Find `for` loops that iterate over vertex lists performing element-wise arithmetic.

**Flag INFO if**: A loop performs operations like `(v[0] * s, v[1] * s, v[2] * s)` that could be `(np.array(verts) * s).tolist()`.

---

### CHECK 13: Import Validation

**Severity: BLOCKER**

Verify that all Sverchok imports are correct and available.

**Required imports per pattern:**

| Usage | Required Import |
|-------|----------------|
| `updateNode` | `from sverchok.data_structure import updateNode` |
| `match_long_repeat` | `from sverchok.data_structure import match_long_repeat` |
| `SverchCustomTreeNode` | `from sverchok.node_tree import SverchCustomTreeNode` |
| `bmesh_from_pydata` | `from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata` |
| `pydata_from_bmesh` | `from sverchok.utils.sv_bmesh_utils import pydata_from_bmesh` |
| `SvNoDataError` | `from sverchok.core.sv_custom_exceptions import SvNoDataError` |

**Flag BLOCKER if**: Code uses `updateNode`, `match_long_repeat`, or `SverchCustomTreeNode` without the correct import statement. Flag BLOCKER if code imports from non-existent Sverchok modules.

---

### CHECK 14: Socket Name Consistency sv_init↔process

**Severity: BLOCKER**

Socket names used in `sv_get()` / `sv_set()` calls in `process()` MUST match the names defined in `sv_init()`.

**Detection**: Extract socket names from `self.inputs.new('...', 'Name')` and `self.outputs.new('...', 'Name')` in `sv_init()`. Compare against `self.inputs['Name']` and `self.outputs['Name']` references in `process()`.

**Flag BLOCKER if**: A socket name in `process()` does not match any socket created in `sv_init()`. This causes a `KeyError` at runtime.

---

### CHECK 15: match_long_repeat Unpacking Pattern

**Severity: WARNING**

The result of `match_long_repeat()` MUST be unpacked correctly.

**Correct pattern:**
```python
verts, scale = match_long_repeat([verts, scale])
```

**Flag WARNING if**: Result is not unpacked (assigned to single variable) or unpacking count does not match input count.

---

### CHECK 16: sv_get Default Pattern

**Severity: WARNING**

`sv_get()` calls SHOULD provide a `default` parameter for optional inputs.

**Detection**: Find `sv_get()` calls without `default=` on non-mandatory inputs.

**Flag WARNING if**: An optional input uses `sv_get()` without a default value. This raises an exception when the socket is unconnected.

**Correct pattern:**
```python
verts = self.inputs['Vertices'].sv_get(default=[[]])
scale = self.inputs['Scale'].sv_get(default=[[1.0]])
```

---

### CHECK 17: Registration Completeness

**Severity: BLOCKER**

Custom node classes MUST define all required registration attributes.

**Required attributes:**

| Attribute | Purpose |
|-----------|---------|
| `bl_idname` | Unique identifier (must start with `Sv`) |
| `bl_label` | Display name in the node editor |

**Recommended attributes:**

| Attribute | Purpose |
|-----------|---------|
| `sv_category` | Category for Shift+S menu |
| `bl_icon` | Icon identifier |

**Flag BLOCKER if**: `bl_idname` or `bl_label` is missing. Flag INFO if `sv_category` or `bl_icon` is missing.

---

### CHECK 18: Property Type Validation

**Severity: BLOCKER**

Verify that `bpy.props` types are used correctly in node class annotations.

**Flag BLOCKER if:**
- `FloatProperty` used where `IntProperty` is needed (counts, indices)
- `CollectionProperty` or `PointerProperty` used without proper registration
- Property type annotation uses `=` instead of `:` (Blender 2.80+ requires annotation syntax)

**Correct:**
```python
count: IntProperty(name='Count', default=5, update=updateNode)
```

**Wrong:**
```python
count = IntProperty(name='Count', default=5, update=updateNode)  # Old syntax
```

---

### CHECK 19: Edge/Face Data Format Validation

**Severity: BLOCKER**

Edge data MUST be tuples/lists of exactly 2 integers. Face data MUST be tuples/lists of 3 or more integers.

**Detection**: Trace edge and face data construction. Verify tuple sizes.

**Flag BLOCKER if**:
- Edge tuples contain fewer or more than 2 elements
- Face tuples contain fewer than 3 elements
- Edge/face indices are floats instead of integers

---

## Severity Summary Decision Tree

```
Code will crash or corrupt data?
├── YES → BLOCKER
│   ├── Wrong nesting level on sv_set() (CHECK 1)
│   ├── Missing updateNode on property (CHECK 2)
│   ├── zip() without match_long_repeat (CHECK 4)
│   ├── Socket creation in process() (CHECK 5)
│   ├── Invalid SNLite alias (CHECK 7)
│   ├── Wrong IfcSverchok nesting (CHECK 10)
│   ├── Missing bm.free() (CHECK 11)
│   ├── Missing/wrong imports (CHECK 13)
│   ├── Socket name mismatch sv_init↔process (CHECK 14)
│   ├── Missing bl_idname or bl_label (CHECK 17)
│   ├── Wrong property annotation syntax (CHECK 18)
│   └── Invalid edge/face tuple size (CHECK 19)
├── NO → Code has bug or missing best practice?
│   ├── YES → WARNING
│   │   ├── No output connection check (CHECK 3)
│   │   ├── Input mutation without deepcopy (CHECK 6)
│   │   ├── Non-standard process() pattern (CHECK 9)
│   │   ├── Wrong match_long_repeat unpacking (CHECK 15)
│   │   └── sv_get() without default (CHECK 16)
│   └── NO → Convention issue?
│       ├── YES → INFO
│       │   ├── Missing Triggers/Tooltip docstring (CHECK 8)
│       │   ├── NumPy optimization opportunity (CHECK 12)
│       │   └── Missing sv_category or bl_icon (CHECK 17)
│       └── NO → PASS
```

---

## Auto-Fix Patterns

When a check fails, apply these fixes automatically where safe:

| Issue | Auto-Fix |
|-------|----------|
| Missing object wrapper on vertices | Wrap in `[data]` → `sv_set([vertices])` |
| Missing object wrapper on strings | Wrap in `[data]` → `sv_set([numbers])` |
| Missing `updateNode` | Add `update=updateNode` to property |
| Missing output check | Insert early-exit block at start of `process()` |
| `zip()` without matching | Insert `match_long_repeat()` before `zip()` |
| Missing `bm.free()` | Insert `bm.free()` after last BMesh usage |
| Invalid SNLite alias | Replace with correct single-letter alias |
| Missing `default=` on `sv_get()` | Add `default=[[]]` for geometry, `default=[[0]]` for numbers |
| Old property syntax (`=`) | Convert to annotation syntax (`:`) |
| Single-nested IFC data | Wrap in extra list level |
| Missing import for `updateNode` | Add `from sverchok.data_structure import updateNode` |
| Missing import for `match_long_repeat` | Add `from sverchok.data_structure import match_long_repeat` |

---

## Validation Report Format

After running all checks, produce a report:

```
## Validation Report
Target: Sverchok v1.4.0
Total Issues: N

### BLOCKERS (N)
- [CHECK X] Description — file:line

### WARNINGS (N)
- [CHECK X] Description — file:line

### INFO (N)
- [CHECK X] Description — file:line

### Auto-Fixes Applied (N)
- Description — file:line
```

---

## Reference Links

- **Validation Rules**: [references/methods.md](references/methods.md)
- **Before/After Examples**: [references/examples.md](references/examples.md)
- **Anti-Patterns Catalog**: [references/anti-patterns.md](references/anti-patterns.md)

### Dependency Skills
- `sverchok-errors-common` — Common Sverchok error patterns and diagnosis
- `sverchok-impl-custom-nodes` — Custom node development patterns
- `sverchok-impl-ifcsverchok` — IfcSverchok integration patterns

### Official Documentation
- Sverchok GitHub: https://github.com/nortikin/sverchok
- Sverchok Documentation: https://nortikin.github.io/sverchok/
- Sverchok Custom Nodes: https://github.com/nortikin/sverchok/wiki/Custom-Nodes
