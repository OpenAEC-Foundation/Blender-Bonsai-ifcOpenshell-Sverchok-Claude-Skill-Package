# sverchok-syntax-scripting — API Methods Reference

> Sverchok v1.4.0 · Blender 4.0+/5.x

## SvScriptNodeLite (SNLite)

### Socket Declaration Parsing

```python
# Internal: parse_sockets(self) → dict
# Returns: {'inputs': [...], 'outputs': [...]}
# Each socket: [socket_type, socket_name, default_value, nested_level]
# Optional 5th element: display_name

# Socket type mapping (parse_socket_line):
# 's' → SvStringsSocket    'v' → SvVerticesSocket
# 'm' → SvMatrixSocket     'o' → SvObjectSocket
# 'C' → SvCurveSocket      'S' → SvSurfaceSocket
# 'So' → SvSolidSocket     'SF' → SvScalarFieldSocket
# 'VF' → SvVectorFieldSocket  'D' → SvDictionarySocket
# 'FP' → SvFilePathSocket
```

### Aliases Dict

```python
# Available as self.snlite_aliases property
# Injected into script namespace before execution

snlite_aliases = {
    'vectorize': vectorize,              # from sverchok.utils.snlite_utils
    'bpy': bpy,                          # blender python
    'np': numpy,                         # numpy
    'ddir': ddir,                        # filtered dir()
    'get_user_dict': self.get_user_dict, # persistent dict keyed by hash(self)
    'reset_user_dict': self.reset_user_dict,  # clear storage
    'cprint': console_print,             # console output
    'console_print': console_print,
    'sv_njit': sv_njit,                  # numba JIT wrapper
    'sv_njit_clear': sv_njit_clear,
    'bmesh_from_pydata': bmesh_from_pydata,
    'pydata_from_bmesh': pydata_from_bmesh,
}
```

### get_user_dict / reset_user_dict

```python
def get_user_dict(self) -> dict:
    """Return persistent per-node storage dict.
    Keyed by hash(self). Survives across process() calls.
    Cleared when node is deleted or reset_user_dict() called."""
    h = hash(self)
    if h not in self.user_dict:
        self.user_dict[h] = {}
    return self.user_dict[h]

def reset_user_dict(self, hard=False):
    """Clear persistent storage.
    hard=False: clear this node's dict only
    hard=True: clear ALL nodes' dicts"""
```

### Special Functions (Hoisted)

```python
def setup():
    """One-time initialization. Called once on script load.
    MUST return locals() — returned values persist in script namespace.
    Example: seed_data = [...]; return locals()"""

def ui(self, context, layout):
    """Custom UI drawing in node's side panel.
    self = the SNLite node instance
    layout = bpy.types.UILayout"""

def sv_internal_links(self):
    """Define pass-through links when node is muted.
    Returns: list of (input_socket, output_socket) tuples
    Default: zip(self.inputs, self.outputs)"""
```

### Custom Enum

```python
# Declaration (in docstring, max 2 enums):
# enum enum_name = Value1 Value2 Value3

# Internal: keep_enum_reference(enum_name)
# Returns: [(value, label, description, index), ...]
# via return_enumeration(enum_name)
```

### Template System

```python
# Template location:
# {sverchok_path}/node_scripts/SNLite_templates/

# Categories (subdirectories):
CATEGORIES = ['demo', 'bpy_stuff', 'bmesh', 'utils', 'templates']

# Templates are .py files loaded via bpy.data.texts
```

---

## SvSNFunctorB (Functor B)

### Class Definition

```python
class SvSNFunctorB:
    bl_idname = 'SvSNFunctorB'
    bl_label = 'SN Functor B'
```

### Hoisted Functions

```python
def functor_init(self, context):
    """Called once when script is loaded.
    self = the SvSNFunctorB node instance
    context = bpy.context
    Create sockets via self.inputs.new() / self.outputs.new()"""

def process(self):
    """Called on each evaluation (same as node.process).
    self = the SvSNFunctorB node instance
    Read: self.inputs['name'].sv_get()
    Write: self.outputs['name'].sv_set(data)"""

def draw_buttons(self, context, layout):
    """Optional. Draw custom UI in the node body.
    self = the SvSNFunctorB node instance
    layout = bpy.types.UILayout"""
```

### Pre-defined Properties

```python
# Generated via make_annotations() for indices 0–4
# All have update=updateNode callback

int_00: IntProperty      # through int_04
float_00: FloatProperty  # through float_04
bool_00: BoolProperty    # through bool_04

# Access in process():   self.float_00, self.int_02, self.bool_01
# Access in draw_buttons(): layout.prop(self, 'float_00', text='Label')
```

### Script Loading

```python
# Script loaded from bpy.types.Text datablock
# Compiled via exec() into temporary module
# Functions extracted via get_functions() → stored in node_dict[hash(node)]
# Executed via handle_execution_nid(self, func_name, ...)
```

---

## SvFormulaNodeMk5 (Formula Mk5)

### Class Definition

```python
class SvFormulaNodeMk5:
    bl_idname = 'SvFormulaNodeMk5'
    bl_label = 'Formula'
```

### Formula Properties

```python
formula1: StringProperty  # First formula slot
formula2: StringProperty  # Second formula slot
formula3: StringProperty  # Third formula slot
formula4: StringProperty  # Fourth formula slot

output_dimensions: IntProperty  # 1–4, controls active formula count
output_type: EnumProperty       # 'Number_/_Generic', 'Vertices', 'Matrices'
wrapping: EnumProperty          # '-1', '0', '+1' — output nesting adjustment
list_match: EnumProperty        # 'REPEAT' (default), numpy_list_match_modes
```

### safe_eval()

```python
# From sverchok.utils.modules.eval_formula
def safe_eval(string: str, variables: dict):
    """Evaluate expression in restricted environment.
    - Blocks ALL Python builtins (__builtins__ = {})
    - Only allows safe_names dict + user variables
    - Parses via ast.parse(string, mode='eval')
    - Compiles and evals the AST"""

# From sverchok.utils.script_importhelper
safe_names = {
    # math module functions (42):
    'acos', 'acosh', 'asin', 'asinh', 'atan', 'atan2', 'atanh',
    'ceil', 'copysign', 'cos', 'cosh', 'degrees',
    'erf', 'erfc', 'exp', 'expm1',
    'fabs', 'factorial', 'floor', 'fmod', 'frexp', 'fsum',
    'gamma', 'hypot', 'isfinite', 'isinf', 'isnan',
    'ldexp', 'lgamma', 'log', 'log10', 'log1p', 'log2',
    'modf', 'pow', 'radians', 'sin', 'sinh', 'sqrt',
    'tan', 'tanh', 'trunc',
    # utility (11):
    'abs', 'sign', 'max', 'min', 'len', 'sum', 'zip',
    'any', 'all', 'dir', 'tuple',
    # type constructors (7):
    'int', 'float', 'str', 'list', 'dict', 'set',
    # constants (2):
    'pi', 'e',
    # objects (4):
    'Vector', 'Matrix', 'np', 'bpy',
}

# NumPy mode variant: safe_names_np
# Maps math names to numpy equivalents: 'acos' → np.arccos, etc.
```

### Variable Extraction

```python
def get_variables(formula: str) -> list[str]:
    """Parse formula string and return list of variable names.
    Variables automatically become input sockets via adjust_sockets()."""

def adjust_sockets(self):
    """Compare current input sockets against formula variables.
    Creates/removes sockets to match. Uses hot_reload_sockets()
    to preserve existing connections during socket regeneration."""
```

---

## SvProfileNodeMK3 (Profile Mk3)

### Class Definition

```python
class SvProfileNodeMK3:
    bl_idname = 'SvProfileNodeMK3'
    bl_label = 'Profile Parametric'
```

### DSL Parsing

```python
def parse_profile(text: str) -> list:
    """Parse profile DSL text into statement objects.
    Each statement has:
    - get_variables() → list of variable names
    - get_hidden_inputs() → variables with defaults
    - get_optional_inputs() → let-defined variables"""

class Interpreter:
    """Processes parsed statements with variable substitution.
    Supports Bezier and NURBS curve output modes."""
```

### Variable Management

```python
def adjust_sockets(self):
    """Extract variables from profile text.
    - 'default' variables → create SvStringsSocket inputs
    - 'let' variables → internal only, no socket created
    - Other variables in commands → create SvStringsSocket inputs"""

def get_optional_inputs(self) -> set:
    """Return variable names that have defaults or are let-defined.
    These do NOT require connected inputs."""
```

### Properties

```python
close_threshold: FloatProperty  # Default 0.0005
    # Distance below which first/last vertices merge on path close (X command)

filename: StringProperty
    # Name of bpy.data.texts datablock containing the profile DSL

addnodes: BoolProperty
    # Whether to show "add nodes" operator in UI

axis: EnumProperty
    # 'X', 'Y', 'Z' — profile plane orientation
```
