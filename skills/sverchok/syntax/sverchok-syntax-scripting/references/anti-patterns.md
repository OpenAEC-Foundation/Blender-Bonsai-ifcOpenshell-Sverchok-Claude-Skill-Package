# sverchok-syntax-scripting — Anti-Patterns

> Sverchok v1.4.0 · Blender 4.0+/5.x

## SNLite Anti-Patterns

### WRONG: Flat output data (missing object nesting)

```python
# WRONG — vertices not wrapped in object-level list
"""
in  count s  default=5  nested=2
out verts v
"""
verts = [(i, 0, 0) for i in range(int(count))]
```

**WHY**: Sverchok expects ALL socket data wrapped in an outer object-level list. Without it, downstream nodes misinterpret the data structure — each vertex tuple becomes a separate "object" instead of vertices within one object.

```python
# CORRECT — wrap in object-level list
verts = [[(i, 0, 0) for i in range(int(count))]]
```

---

### WRONG: Using self.inputs.new() in SNLite

```python
# WRONG — manual socket creation in SNLite script body
"""
out result s
"""
self.inputs.new('SvStringsSocket', 'my_input')
result = [[self.inputs['my_input'].sv_get()]]
```

**WHY**: SNLite manages sockets automatically from docstring declarations. Manually creating sockets bypasses the parser and causes conflicts when the node reloads — sockets get duplicated or lost.

```python
# CORRECT — declare sockets in docstring
"""
in  my_input s  default=0
out result s
"""
result = [[my_input]]
```

---

### WRONG: Storing state in script-level variables

```python
# WRONG — counter resets to 0 on every evaluation
"""
in  trigger s  default=0
out count s
"""
counter = 0  # This resets every time process() runs
counter += 1
count = [[counter]]
```

**WHY**: The entire script body executes on every evaluation. Variable assignments are re-executed from scratch. There is no persistence between calls.

```python
# CORRECT — use get_user_dict() for persistent state
"""
in  trigger s  default=0
out count s
"""
storage = get_user_dict()
if 'counter' not in storage:
    storage['counter'] = 0
storage['counter'] += 1
count = [[storage['counter']]]
```

---

### WRONG: Importing numpy manually when np alias exists

```python
# WRONG — unnecessary import, wastes a line
"""
in  data s
out result s
"""
import numpy as np  # Already available as 'np' alias
result = [[np.mean(data)]]
```

**WHY**: SNLite pre-imports `np` (NumPy), `bpy`, `bmesh_from_pydata`, `pydata_from_bmesh`, and other common names. Re-importing them is redundant and can shadow the pre-configured versions.

```python
# CORRECT — use built-in alias directly
"""
in  data s
out result s
"""
result = [[np.mean(data)]]
```

---

### WRONG: Missing nested=2 for scalar inputs used directly

```python
# WRONG — without nested, default nesting may cause list-in-list issues
"""
in  radius s  default=1.0
out verts v
"""
# radius might be [[1.0]] instead of 1.0
angle = 2 * 3.14159 * radius  # TypeError: can't multiply list by float
```

**WHY**: Without `nested=2`, the input value retains Sverchok's default nesting. The variable `radius` may be `[[1.0]]` (list of lists) rather than the scalar `1.0` you expect.

```python
# CORRECT — specify nested=2 to get raw scalar value
"""
in  radius s  default=1.0  nested=2
out verts v
"""
angle = 2 * 3.14159 * radius  # radius is now 1.0
```

---

## SN Functor B Anti-Patterns

### WRONG: Using __init__ instead of functor_init

```python
# WRONG — __init__ is not called by Functor B
def __init__(self, context):
    self.inputs.new('SvStringsSocket', 'data')

def process(self):
    data = self.inputs['data'].sv_get()
```

**WHY**: Functor B looks for the hoisted function named `functor_init`, not `__init__`. Using `__init__` means sockets are never created, and the node has no inputs/outputs.

```python
# CORRECT — use functor_init
def functor_init(self, context):
    self.inputs.new('SvStringsSocket', 'data')

def process(self):
    data = self.inputs['data'].sv_get()
```

---

### WRONG: Forgetting update=updateNode on custom properties

```python
# WRONG — changing float_00 in UI does nothing
def draw_buttons(self, context, layout):
    layout.prop(self, 'float_00', text='Scale')

def process(self):
    scale = self.float_00  # Value changes but node never re-processes
```

**WHY**: The pre-defined properties (int_00–04, float_00–04, bool_00–04) already have `update=updateNode`. This anti-pattern occurs when users define their OWN properties on the node without the callback. The pre-defined ones work correctly — use them instead of custom properties.

---

### WRONG: Defining sockets in process() instead of functor_init()

```python
# WRONG — sockets created on every evaluation, causes errors
def process(self):
    if 'radius' not in self.inputs:
        self.inputs.new('SvStringsSocket', 'radius')
    # ...
```

**WHY**: Socket creation triggers tree topology events. Creating sockets during `process()` causes infinite update loops — the new socket triggers a tree update, which calls `process()` again.

```python
# CORRECT — create all sockets in functor_init, once
def functor_init(self, context):
    self.inputs.new('SvStringsSocket', 'radius')
```

---

## Formula Mk5 Anti-Patterns

### WRONG: Using blocked builtins in formulas

```python
# WRONG — these will raise NameError
Formula 1: open('file.txt').read()
Formula 1: __import__('os').listdir('.')
Formula 1: eval("1+1")
```

**WHY**: The `safe_eval` system sets `__builtins__ = {}` and ONLY allows the whitelisted `safe_names` dict. Functions like `open`, `__import__`, `eval`, `exec`, `compile` are blocked.

```python
# CORRECT — use only whitelisted functions
Formula 1: sin(x) + cos(y)
Formula 1: max(a, min(b, 10))
```

---

### WRONG: Expecting list comprehension variables as sockets

```python
# WRONG — 'i' in comprehension becomes an input socket
Formula 1: [i**2 for i in range(n)]
```

**WHY**: The variable parser (`get_variables()`) may extract `i` as a variable name and create an unwanted input socket for it. The parser does not fully understand Python scoping rules.

```python
# CORRECT — use SNLite for complex expressions with local variables
# Or verify the formula works and remove the unwanted socket manually
```

---

## Profile Mk3 Anti-Patterns

### WRONG: Forgetting semicolon on H and V commands

```
# WRONG — parser error or unexpected behavior
default w = 1.0
M 0 0
H {w}
V 0.5
```

**WHY**: The `H` and `V` commands accept variable-length coordinate arguments. The semicolon `;` is REQUIRED to signal end-of-arguments. Without it, the parser may consume the next line as additional coordinates.

```
# CORRECT — always end H and V with semicolon
M 0 0
H {w} ;
V 0.5 ;
```

---

### WRONG: Using 'let' variables expecting them as input sockets

```
# WRONG — expecting hw to appear as an input socket
let hw = 0.5
M -{hw} 0
L {hw} 1
```

**WHY**: `let` variables are computed internally and NEVER create input sockets. Only `default` variables and unresolved variable names become sockets.

```
# CORRECT — use 'default' for variables that should be input sockets
default hw = 0.5
M -{hw} 0
L {hw} 1
```

---

### WRONG: Using Python expressions outside curly braces

```
# WRONG — math not in braces is treated as literal text
default w = 1.0
M w/2 0
L w 1
```

**WHY**: Profile DSL only evaluates expressions inside `{curly braces}`. Bare text is parsed as variable names or literal values, not expressions.

```
# CORRECT — wrap expressions in curly braces
default w = 1.0
M {w/2} 0
L {w} 1
```

---

### WRONG: Not closing paths with X for architectural profiles

```
# WRONG — open profile creates disconnected edges
default w = 0.2
default h = 0.4
M 0 0
L {w} 0
L {w} {h}
L 0 {h}
# Missing X — last vertex not connected to first
```

**WHY**: Without `X`, the profile is an open polyline. For architectural cross-sections (I-beams, channels, etc.) that need to be extruded or filled, an unclosed profile produces gaps.

```
# CORRECT — close the path for solid profiles
M 0 0
L {w} 0
L {w} {h}
L 0 {h}
X
```
