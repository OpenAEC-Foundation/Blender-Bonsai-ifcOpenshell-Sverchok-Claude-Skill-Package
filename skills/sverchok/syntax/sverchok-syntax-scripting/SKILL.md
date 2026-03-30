---
name: sverchok-syntax-scripting
description: >
  Use when writing SNLite scripts, Formula Mk5 expressions, Profile Mk3 definitions, or
  embedding custom Python logic in Sverchok node trees. Prevents the common mistake of
  wrong socket declaration syntax in SNLite (must use specific type identifiers like 'v'
  for vertices, 's' for strings). Covers all scripting nodes, built-in aliases, special
  functions, and the template system.
  Keywords: SNLite, Formula, Profile, scripting node, SN Functor B, socket declaration, type identifier, custom Python, Sverchok scripting, write Python in Sverchok, script node, inline code.
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-syntax-scripting

## Quick Reference

### Script Node Overview

Sverchok provides **4 primary scripting nodes** for embedding custom Python logic in node trees:

| Node | ID | Menu | Purpose |
|------|----|------|---------|
| **Script Node Lite (SNLite)** | `SvScriptNodeLite` | `snl` | Inline scripting with text-based socket declarations |
| **SN Functor B** | `SvSNFunctorB` | `functorB` | Structured scripting with init/process/draw functions |
| **Formula Mk5** | `SvFormulaNodeMk5` | — | Safe math expression evaluation (up to 4 formulas) |
| **Profile Mk3** | `SvProfileNodeMK3` | — | SVG-like DSL for 2D parametric profiles |

### Critical Warnings

**NEVER** use `import os`, `import subprocess`, or file-system operations inside Formula Mk5 — the `safe_eval` system blocks all builtins and ONLY allows whitelisted functions.

**NEVER** store mutable state in SNLite script-body variables across evaluations — use `get_user_dict()` for persistent per-node storage.

**NEVER** define sockets manually in SNLite (via `self.inputs.new(...)`) — ALWAYS use the text-based declaration syntax in the docstring header.

**ALWAYS** wrap SNLite output data in the correct nesting level — `verts = [[(x,y,z), ...]]` not `verts = [(x,y,z), ...]`.

**ALWAYS** use `functor_init()` (not `__init__`) to create sockets in SN Functor B scripts.

**ALWAYS** end Profile Mk3 `H` and `V` commands with a semicolon `;`.

### Decision Tree

```
Need custom Python logic in a node tree?
├── Quick prototype / simple transform → SNLite
├── Need custom UI + multiple properties → SN Functor B
├── Math expression only (no logic) → Formula Mk5
└── 2D parametric shape / architectural profile → Profile Mk3

Writing an SNLite script?
├── Need persistent storage → get_user_dict()
├── Need one-time init → setup() function
├── Need custom UI → ui(self, context, layout) function
├── Need BMesh operations → bmesh_from_pydata / pydata_from_bmesh aliases
└── Need NumPy performance → np alias (pre-imported)

Choosing SNLite vs Functor B?
├── Simple I/O, few properties → SNLite
├── Need > 2 enums or typed properties → Functor B (5x int/float/bool)
├── Need full self access in process() → Functor B
└── Learning / quick iteration → SNLite
```

---

## SNLite: Socket Declaration Syntax

### Declaration Format

Socket declarations go in the script's opening docstring:

```python
"""
in  socket_name  type  [default=value]  [nested=level]  [required=True]
out socket_name  type
"""
```

### Socket Type Identifiers

| ID | Socket Class | Data Type |
|----|-------------|-----------|
| `s` | `SvStringsSocket` | Scalars, integers, generic numeric |
| `v` | `SvVerticesSocket` | 3D coordinates `(x, y, z)` |
| `m` | `SvMatrixSocket` | 4x4 transformation matrices |
| `o` | `SvObjectSocket` | Blender object references |
| `C` | `SvCurveSocket` | Curve objects |
| `S` | `SvSurfaceSocket` | Surface objects |
| `So` | `SvSolidSocket` | Solid objects (FreeCAD) |
| `SF` | `SvScalarFieldSocket` | Scalar field functions |
| `VF` | `SvVectorFieldSocket` | Vector field functions |
| `D` | `SvDictionarySocket` | Python dictionaries |
| `FP` | `SvFilePathSocket` | File path strings |

### Declaration Options

| Option | Syntax | Effect |
|--------|--------|--------|
| Default value | `default=1.0` or `;=1.0` | Value when socket unconnected |
| Nesting level | `nested=2` or `n=2` | 0=raw, 1=list, 2=list of lists |
| Required | `required=True` | Node skips processing if unconnected |
| Display name | 5th positional element | Custom label in UI |

### Built-in Aliases

These names are pre-imported and available in every SNLite script:

```python
vectorize           # sverchok.utils.snlite_utils — auto-vectorize functions
bpy                 # Blender Python API
np                  # NumPy
bmesh_from_pydata   # Build BMesh from verts/edges/faces lists
pydata_from_bmesh   # Extract verts/edges/faces from BMesh
get_user_dict       # Persistent per-node dict storage
reset_user_dict     # Clear per-node storage (hard=True clears all)
cprint              # Console output (alias: console_print)
ddir                # Filtered dir() utility
sv_njit             # Numba JIT compilation wrapper
sv_njit_clear       # Clear Numba JIT cache
```

### Special Functions

**`setup()`** — Runs once on script load. Return values persist via `return locals()`:

```python
def setup():
    import random
    seed_data = [random.random() for _ in range(100)]
    return locals()
# seed_data is now available in the main script body
```

**`ui(self, context, layout)`** — Custom node panel UI:

```python
def ui(self, context, layout):
    layout.label(text="My Custom Node")
    layout.prop(self, "custom_enum")
```

**`sv_internal_links(self)`** — Controls pass-through when node is muted:

```python
def sv_internal_links(self):
    return [(self.inputs[0], self.outputs[0])]
```

### Additional Features

**Custom enums** (maximum 2 per script):

```python
"""
enum mode = Add Subtract Multiply
in  value s  default=1.0
out result s
"""
if mode == "Add":
    result = [[v + 1 for v in value]]
```

**File handler** — adds a text datablock selector to the UI:

```python
"""
display_file_handler
in  scale s  default=1.0
out verts v
"""
```

**Includes** — import other Blender text datablocks:

```python
"""
includes: helper_functions.py, config.py
in  data s
out result s
"""
```

**Persistent storage** via `get_user_dict()`:

```python
"""
in  trigger s  default=0
out count   s
"""
storage = get_user_dict()
if 'counter' not in storage:
    storage['counter'] = 0
storage['counter'] += 1
count = [[storage['counter']]]
```

### Template System

SNLite templates are located in `node_scripts/SNLite_templates/` with categories:

| Category | Content |
|----------|---------|
| `demo` | Spirals, voronoi, genetic algorithms |
| `bpy_stuff` | Blender API interaction |
| `bmesh` | BMesh geometry manipulation |
| `utils` | DXF export, IFC export, SVG import |
| `templates` | Reusable script patterns |

---

## SN Functor B: Structured Scripting

### Three-Function Architecture

```python
import bpy
from sverchok.data_structure import updateNode

def functor_init(self, context):
    """Called once when script is loaded. Define sockets here."""
    self.inputs.new('SvStringsSocket', 'radius')
    self.inputs.new('SvStringsSocket', 'segments')
    self.outputs.new('SvVerticesSocket', 'verts')
    self.outputs.new('SvStringsSocket', 'edges')

def process(self):
    """Called on each evaluation. Read inputs, write outputs."""
    if not self.inputs['radius'].is_linked:
        return
    radius_data = self.inputs['radius'].sv_get()
    # ... processing logic ...
    self.outputs['verts'].sv_set(all_verts)

def draw_buttons(self, context, layout):
    """Optional custom UI in the node body."""
    layout.prop(self, 'float_00', text='Scale')
    layout.prop(self, 'int_00', text='Count')
```

### Pre-defined Properties

Functor B provides **15 pre-defined properties** (5 of each type), all with `update=updateNode`:

| Property | Type | Range |
|----------|------|-------|
| `int_00` — `int_04` | `IntProperty` | 0–4 index |
| `float_00` — `float_04` | `FloatProperty` | 0–4 index |
| `bool_00` — `bool_04` | `BoolProperty` | 0–4 index |

Access in `draw_buttons`: `layout.prop(self, 'float_00', text='My Label')`
Access in `process`: `scale = self.float_00`

### SNLite vs Functor B

| Feature | SNLite | Functor B |
|---------|--------|-----------|
| Socket definition | Docstring declarations | `functor_init()` |
| Process logic | Script body | `process(self)` |
| UI customization | `ui()` | `draw_buttons()` |
| Properties | 2 custom enums | 5x int + 5x float + 5x bool |
| Self access | Limited (via aliases) | Full `self` reference |
| Initialization | `setup()` | `functor_init()` |
| Best for | Quick prototyping | Complex custom nodes |

---

## Formula Mk5: Expression Evaluation

### Basic Usage

The Formula node evaluates Python math expressions with safety restrictions. Supports **4 simultaneous formulas**. Variables in expressions automatically become input sockets.

```
Formula 1: x + 1
Formula 2: R * sin(phi)
Formula 3: 0.75 * X + 0.25 * Y
Formula 4: [v**2 for v in range(n)]
```

### Safe Evaluation Environment

The `safe_eval()` system blocks ALL Python builtins and ONLY allows:

**Math functions**: `acos`, `acosh`, `asin`, `asinh`, `atan`, `atan2`, `atanh`, `ceil`, `copysign`, `cos`, `cosh`, `degrees`, `erf`, `erfc`, `exp`, `expm1`, `fabs`, `factorial`, `floor`, `fmod`, `frexp`, `fsum`, `gamma`, `hypot`, `isfinite`, `isinf`, `isnan`, `ldexp`, `lgamma`, `log`, `log10`, `log1p`, `log2`, `modf`, `pow`, `radians`, `sin`, `sinh`, `sqrt`, `tan`, `tanh`, `trunc`

**Constants**: `pi`, `e`

**Utility functions**: `abs`, `sign`, `max`, `min`, `len`, `sum`, `zip`, `any`, `all`, `dir`

**Type constructors**: `int`, `float`, `str`, `list`, `tuple`, `dict`, `set`

**Objects**: `Vector`, `Matrix`, `np` (NumPy), `bpy`

### Output Configuration

| Property | Values | Purpose |
|----------|--------|---------|
| `output_dimensions` | 1–4 | Number of active formula slots |
| `output_type` | Number/Generic, Vertices, Matrices | Output socket type |
| `wrapping` | -1, 0, +1 | Output nesting level adjustment |

### NumPy Mode

When enabled, math functions use NumPy array variants (`sin` → `np.sin`), enabling vectorized computation on arrays instead of scalar-by-scalar evaluation.

---

## Profile Mk3: SVG-like 2D Profiles

### DSL Command Reference

| Command | Syntax | Description |
|---------|--------|-------------|
| `M` / `m` | `M x y` | Move to (absolute / relative) |
| `L` / `l` | `L x y [n=segments] [z]` | Line to |
| `H` / `h` | `H x [n=segments] ;` | Horizontal line (semicolon required) |
| `V` / `v` | `V y [n=segments] ;` | Vertical line (semicolon required) |
| `C` / `c` | `C x1 y1 x2 y2 x y [n=verts] [z]` | Cubic Bezier |
| `S` / `s` | `S x2 y2 x y [n=verts] [z]` | Smooth cubic Bezier |
| `Q` / `q` | `Q x1 y1 x y [n=segments] [z]` | Quadratic Bezier |
| `T` / `t` | `T x y [n=segments] [z]` | Smooth quadratic Bezier |
| `A` / `a` | `A rx ry rot flag1 flag2 x y [n=verts] [z]` | Arc |
| `@I` / `@i` | `@I [@smooth] degree p1 p2 ... [n=segments] [z] ;` | NURBS interpolation |
| `X` | `X` | Close path cyclically |
| `#` | `# comment text` | Single-line comment |

### Variable System

```
default width = 0.3          # Becomes input socket with default value
default height = 2.7         # Becomes input socket with default value
let half_width = {width / 2} # Computed variable, does NOT become socket
let ft = {flange_thickness}  # Reusable shorthand

M -{half_width} 0            # Use variables in commands
L {width + margin} {height}  # Expressions in curly braces
```

- `default` variables become input sockets (auto-created)
- `let` variables are computed internally — NEVER become sockets
- Expressions use `{curly braces}` for inline math
- `close_threshold` property (default 0.0005) detects overlapping start/end vertices

### Architectural Profile Example: I-Beam

```
default width = 0.2
default height = 0.4
default flange_thickness = 0.02
default web_thickness = 0.01

let hw = {width / 2}
let hh = {height / 2}
let ft = {flange_thickness}
let wt = {web_thickness / 2}

M -{hw} -{hh}
L {hw} -{hh}
L {hw} {-hh + ft}
L {wt} {-hh + ft}
L {wt} {hh - ft}
L {hw} {hh - ft}
L {hw} {hh}
L -{hw} {hh}
L -{hw} {hh - ft}
L -{wt} {hh - ft}
L -{wt} {-hh + ft}
L -{hw} {-hh + ft}
X
```

---

## Other Script Nodes

| Node | Source | Purpose |
|------|--------|---------|
| **Generative Art** | `generative_art.py` | L-System generative art via XML rules |
| **Formula Interpolate** | `formula_interpolate.py` | Interpolated formulas over ranges |
| **Mesh Expression** | `mesh_eval.py` | Mesh generation from JSON structures |
| **Multi Exec** | `multi_exec.py` | Multiple sequential Python expressions |
| **NumExpr** | `numexpr_node.py` | Accelerated expressions via `numexpr` library |

---

## Reference Links

- [references/methods.md](references/methods.md) — API signatures for SNLite, Functor B, Formula Mk5, Profile Mk3
- [references/examples.md](references/examples.md) — Working code examples for each scripting node
- [references/anti-patterns.md](references/anti-patterns.md) — Common mistakes and WHY they fail

### Official Sources

- https://github.com/nortikin/sverchok/blob/master/nodes/script/script1_lite.py
- https://github.com/nortikin/sverchok/blob/master/nodes/script/sn_functor_b.py
- https://github.com/nortikin/sverchok/blob/master/nodes/script/formula_mk5.py
- https://github.com/nortikin/sverchok/blob/master/nodes/script/profile_mk3.py
- https://sverchok.readthedocs.io/
