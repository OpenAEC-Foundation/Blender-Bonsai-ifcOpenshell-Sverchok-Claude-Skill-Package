---
name: sverchok-syntax-sockets
description: 'Covers all 16 Sverchok socket types including choosing the right socket for geometry, numbers, matrices, curves, surfaces, and fields. Explains socket properties, implicit type conversions, and data processing flags. Use this when connecting nodes, understanding type conversions, debugging socket compatibility errors, or choosing the right socket type for your data.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-syntax-sockets

## Quick Reference

### All 16 Socket Types

| Socket Class | Color (RGBA) | Data Type | `nesting_level` | SNLite |
|---|---|---|---|---|
| `SvStringsSocket` | `(0.6, 1.0, 0.6, 1.0)` | int, float, generic lists | 2 | `s` |
| `SvVerticesSocket` | `(0.9, 0.6, 0.2, 1.0)` | 3D coordinates `(x,y,z)` | 3 | `v` |
| `SvMatrixSocket` | `(0.2, 0.8, 0.8, 1.0)` | 4x4 `mathutils.Matrix` | 1 | `m` |
| `SvQuaternionSocket` | `(0.9, 0.4, 0.7, 1.0)` | `mathutils.Quaternion` | 2 | — |
| `SvColorSocket` | `(0.9, 0.8, 0.0, 1.0)` | RGBA tuples `(r,g,b,a)` | 3 | — |
| `SvCurveSocket` | `(0.5, 0.6, 1.0, 1.0)` | `SvCurve` instances | 2 | `C` |
| `SvSurfaceSocket` | `(0.4, 0.2, 1.0, 1.0)` | `SvSurface` instances | 2 | `S` |
| `SvSolidSocket` | `(0.0, 0.65, 0.3, 1.0)` | `Part.Shape` (FreeCAD) | 2 | `So` |
| `SvScalarFieldSocket` | `(0.9, 0.4, 0.1, 1.0)` | `SvScalarField` instances | 2 | `SF` |
| `SvVectorFieldSocket` | `(0.1, 0.1, 0.9, 1.0)` | `SvVectorField` instances | 2 | `VF` |
| `SvDictionarySocket` | `(1.0, 1.0, 1.0, 1.0)` | Python `dict` objects | 2 | `D` |
| `SvObjectSocket` | `(0.69, 0.74, 0.73, 1.0)` | Blender `bpy.types.Object` | 2 | `o` |
| `SvFilePathSocket` | `(0.9, 0.9, 0.3, 1.0)` | File path strings | 2 | `FP` |
| `SvTextSocket` | `(0.68, 0.85, 0.90, 1.0)` | Text strings | 2 | — |
| `SvDummySocket` | `(0.8, 0.8, 0.8, 0.3)` | Type placeholder only | 2 | — |
| `SvChameleonSocket` | dynamic | Adopts linked socket type | dynamic | — |

### Critical Warnings

**NEVER** connect mismatched socket types without understanding the implicit conversion policy — silent data transformations can corrupt geometry.

**NEVER** use `SvStringsSocket` for structured geometry objects (curves, surfaces, fields) — the data will pass through but lose type safety and trigger silent conversion failures downstream.

**ALWAYS** use `sv_get(default=[[]])` with a default value on input sockets to avoid `SvNoDataError` when the socket is unconnected.

**ALWAYS** call `sv_forget()` before `replace_socket()` to clear stale cached data — the old socket_id becomes invalid after replacement.

**NEVER** read `socket.other` on an output socket with multiple connections — it returns one arbitrary linked socket, not all of them.

---

## Decision Trees

### Choosing the Right Socket Type

```
What data do you need to transport?

Numbers, booleans, or generic Python objects?
└── SvStringsSocket  (green)

3D point coordinates?
└── SvVerticesSocket  (orange)

4x4 transformation matrix?
└── SvMatrixSocket  (teal)

Rotation only (no translation/scale)?
└── SvQuaternionSocket  (pink)

RGBA color values?
└── SvColorSocket  (yellow)

Parametric curve object (maps t -> point)?
└── SvCurveSocket  (blue)

Parametric surface object (maps u,v -> point)?
└── SvSurfaceSocket  (deep purple)

CAD solid (requires FreeCAD/OpenCascade)?
└── SvSolidSocket  (green)

Scalar field (maps R³ -> R)?
└── SvScalarFieldSocket  (dark orange)

Vector field (maps R³ -> R³)?
└── SvVectorFieldSocket  (deep blue)

Python dict / key-value data?
└── SvDictionarySocket  (white)

Blender scene object reference?
└── SvObjectSocket  (grey-green)

File system path?
└── SvFilePathSocket  (yellow-green)

Raw text string?
└── SvTextSocket  (light blue)

Placeholder for dynamic socket (not yet typed)?
└── SvDummySocket  (grey, semi-transparent)

Socket that should mirror the connected socket type?
└── SvChameleonSocket  (dynamic color)
```

### Debugging Socket Compatibility Errors

```
Connection refused or TypeError on sv_get?

Are socket colors the same?
├── Yes → socket types match, error is in data structure
└── No  → implicit conversion will be attempted

Check the to_socket's default_conversion_name:
├── 'DefaultImplicitConversionPolicy'
│   └── See DefaultImplicitConversionPolicy conversion table
├── 'FieldImplicitConversionPolicy'
│   └── Matrices/vertices/strings auto-convert to fields
├── 'LenientImplicitConversionPolicy'
│   └── Any data passes through unchanged
├── 'SolidImplicitConversionPolicy'
│   └── Attempts to_solid_recursive(); raises on failure
└── None / no policy
    └── Connection is blocked — ImplicitConversionProhibited raised

Is from_socket a lenient type?
├── SvStringsSocket, SvObjectSocket, SvColorSocket, SvVerticesSocket → YES, data passes
└── Otherwise → conversion policy decides
```

---

## Essential Patterns

### Pattern 1: Read and Write Socket Data in process()

```python
# Sverchok v1.4.0+ — inside a custom node's process() method
def process(self):
    # Read input — ALWAYS provide default to handle unconnected socket
    verts_in = self.inputs['Vertices'].sv_get(default=[[]])
    # deepcopy=False safe ONLY if you will NOT mutate the list
    numbers_in = self.inputs['Count'].sv_get(default=[[1]], deepcopy=False)

    # ... compute result ...

    # Write output — data MUST match nesting_level of socket
    # SvVerticesSocket nesting_level=3: [[(x,y,z), ...], ...]
    self.outputs['Vertices'].sv_set([[(0.0, 0.0, 0.0)]])
    # SvStringsSocket nesting_level=2: [[val, val, ...], ...]
    self.outputs['Count'].sv_set([[1, 2, 3]])
    # SvMatrixSocket nesting_level=1: [Matrix, Matrix, ...]
    import mathutils
    self.outputs['Matrix'].sv_set([mathutils.Matrix.Identity(4)])
```

### Pattern 2: Replace a Socket Type at Runtime

```python
# Sverchok v1.4.0+ — inside sv_update() or sv_init()
def sv_update(self):
    # ALWAYS call sv_forget() before replace_socket()
    old_socket = self.inputs.get('Data')
    if old_socket is not None:
        old_socket.sv_forget()
        new_socket = old_socket.replace_socket('SvCurveSocket', 'Curve')
        # new_socket is the replacement; old_socket reference is now invalid
```

### Pattern 3: Access socket_id and other

```python
# Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyTree"]
node = tree.nodes["MyNode"]

sock = node.inputs['Vertices']

# socket_id: unique string key for the data cache
cache_key = sock.socket_id
# Returns: str (hash of node_id + identifier + direction)

# other: the socket on the far end of the link
if sock.is_linked:
    upstream = sock.other
    # For INPUT sockets: returns the single connected output socket
    # For OUTPUT sockets: returns ONE of the connected input sockets (arbitrary)
    print(upstream.node.name, upstream.name)
```

### Pattern 4: Set Socket Processing Flags

```python
# Sverchok v1.4.0+ — define allowed flags in sv_init()
def sv_init(self, context):
    sock = self.inputs.new('SvVerticesSocket', 'Vertices')
    # Enable which processing flags the user can toggle in the UI
    sock.allow_flatten = True
    sock.allow_graft = True
    sock.allow_wrap = True
    sock.allow_unwrap = True
    sock.allow_simplify = True
    # The actual use_* flags are set by the user in the socket UI
    # Processing flag *behavior* is handled by the sv_get() pipeline
    # See: sverchok-syntax-data for how flags transform data
```

### Pattern 5: Implicit Conversion — Field Sockets

```python
# Sverchok v1.4.0+ — SvScalarFieldSocket and SvVectorFieldSocket
# use FieldImplicitConversionPolicy automatically

# When a SvStringsSocket (numbers) is connected to SvScalarFieldSocket:
# → numbers are wrapped in SvConstantScalarField automatically
# When a SvVerticesSocket is connected to SvVectorFieldSocket:
# → vectors are wrapped in SvConstantVectorField automatically
# When a SvMatrixSocket is connected to SvVectorFieldSocket:
# → matrix creates SvMatrixVectorField automatically

# This means: nodes producing numbers/vectors can feed field-consuming nodes
# without explicit conversion nodes in the node tree.

# Reading field data in process():
def process(self):
    from sverchok.utils.field.scalar import SvScalarField
    fields = self.inputs['Field'].sv_get()
    # fields is [[SvScalarField, ...], ...]
    for field in fields[0]:
        point_value = field.evaluate(0.0, 0.0, 0.0)  # Returns float
```

---

## Socket Properties Reference

### SvSocketCommon Properties

| Property | Type | Default | Purpose |
|---|---|---|---|
| `color` | `tuple` | class-defined | RGBA display color in node editor |
| `label` | `StringProperty` | `''` | Override display name in UI |
| `use_prop` | `BoolProperty` | `False` | Show embedded property widget |
| `prop_name` | `StringProperty` | `''` | Node attribute to display as property |
| `objects_number` | `IntProperty` | `0` | Count of data objects (informational) |
| `is_mandatory` | `BoolProperty` | `False` | Block node execution if unconnected |
| `nesting_level` | `IntProperty` | `2` | Expected list nesting depth |
| `default_mode` | `EnumProperty` | `'EMPTY_LIST'` | Fallback when no data: NONE / EMPTY_LIST / MATRIX / MASK |
| `pre_processing` | `EnumProperty` | `'NONE'` | Pre-processing: NONE / ONE_ITEM |
| `quick_link_to_node` | `str` | class-defined | Node bl_idname shown in quick-link button |
| `default_conversion_name` | `str` | class-defined | Conversion policy class name |

### SvSocketProcessing Flags

| Flag Property | Enable Property | Default | Effect |
|---|---|---|---|
| `allow_flatten` | `use_flatten` | `False` | Flattens nested list to single level |
| `allow_flatten_topology` | `use_flatten_topology` | `False` | Flattens preserving topology structure |
| `allow_simplify` | `use_simplify` | `False` | Removes redundant outer nesting |
| `allow_graft` | `use_graft` | `False` | Adds a nesting level |
| `allow_unwrap` | `use_unwrap` | `False` | Removes one nesting level |
| `allow_wrap` | `use_wrap` | `False` | Adds one nesting level |

**Note:** `allow_*` properties control whether the flag is visible in the UI. `use_*` properties are the actual state set by the user. Processing flag behavior (how data is transformed) is documented in `sverchok-syntax-data`.

### quick_link_to_node Values

| Socket | quick_link_to_node |
|---|---|
| `SvMatrixSocket` | `'SvMatrixInNodeMK4'` |
| `SvVerticesSocket` | `'GenVectorsNode'` |
| `SvFilePathSocket` | `'SvFilePathNode'` |
| `SvDictionarySocket` | `'SvDictionaryIn'` |
| `SvScalarFieldSocket` | `'SvNumberNode'` |
| `SvVectorFieldSocket` | `'GenVectorsNode'` |
| `SvLoopControlSocket` | `'SvLoopInNode'` |

---

## Implicit Conversion Policies

### DefaultImplicitConversionPolicy — Explicit Conversions

| From | To | Conversion Applied |
|---|---|---|
| `SvVerticesSocket` | `SvMatrixSocket` | Creates translation matrices from vertices |
| `SvVerticesSocket` | `SvColorSocket` | Appends alpha=1.0: `(x,y,z)` → `(x,y,z,1)` |
| `SvMatrixSocket` | `SvVerticesSocket` | Extracts translation column |
| `SvMatrixSocket` | `SvQuaternionSocket` | `mat.to_quaternion()` |
| `SvQuaternionSocket` | `SvMatrixSocket` | `quat.to_matrix().to_4x4()` |
| `SvStringsSocket` | `SvVerticesSocket` | Number `v` → vector `(v, 0, 0)` |
| `SvStringsSocket` | `SvColorSocket` | Number `v` → color `(v, v, v, 1)` |

Lenient socket types accept data from any socket without conversion:
```python
lenient_socket_types = {
    'SvStringsSocket', 'SvObjectSocket', 'SvColorSocket', 'SvVerticesSocket',
}
```

### FieldImplicitConversionPolicy — Additional Field Conversions

Used by `SvScalarFieldSocket` and `SvVectorFieldSocket`.

| From | To | Conversion Applied |
|---|---|---|
| `SvStringsSocket` | `SvScalarFieldSocket` | Numbers → `SvConstantScalarField` |
| `SvVerticesSocket` | `SvVectorFieldSocket` | Vertices → `SvConstantVectorField` |
| `SvMatrixSocket` | `SvVectorFieldSocket` | Matrix → `SvMatrixVectorField` |

Falls through to `DefaultImplicitConversionPolicy` for all other type pairs.

### Policy Assignment Per Socket

```python
# Sverchok v1.4.0+ — source: core/sockets.py
class SvVerticesSocket:
    default_conversion_name = ConversionPolicies.DEFAULT.conversion_name
    # → 'DefaultImplicitConversionPolicy'

class SvScalarFieldSocket:
    default_conversion_name = ConversionPolicies.FIELD.conversion_name
    # → 'FieldImplicitConversionPolicy'

class SvVectorFieldSocket:
    default_conversion_name = ConversionPolicies.FIELD.conversion_name
    # → 'FieldImplicitConversionPolicy'

class SvTextSocket:
    default_conversion_name = ConversionPolicies.LENIENT.conversion_name
    # → 'LenientImplicitConversionPolicy'

class SvSolidSocket:
    default_conversion_name = ConversionPolicies.SOLID.conversion_name
    # → 'SolidImplicitConversionPolicy'
```

---

## Advanced Data Types

### SvCurve

Abstract base class from `sverchok/utils/curve/__init__.py`. Maps parameter `t ∈ [t_min, t_max]` to a 3D point. Not necessarily arc-length parametrized.

- `curve.get_u_bounds()` → `(t_min, t_max)`
- `curve.evaluate(t)` → `numpy.ndarray` shape `(3,)`
- `curve.evaluate_array(ts)` → `numpy.ndarray` shape `(N, 3)`

### SvSurface

Abstract base class from `sverchok/utils/surface/__init__.py`. Maps `(u, v)` to 3D points.

- `surface.get_u_min(), surface.get_u_max()`
- `surface.get_v_min(), surface.get_v_max()`
- `surface.evaluate(u, v)` → `numpy.ndarray` shape `(3,)`
- `surface.evaluate_array(us, vs)` → `numpy.ndarray` shape `(N, 3)`

### SvScalarField

From `sverchok/utils/field/scalar.py`. Maps `R³ → R`.

- `field.evaluate(x, y, z)` → `float`
- `field.evaluate_grid(xs, ys, zs)` → `numpy.ndarray`
- Supports: addition, multiplication, composition, gradient

### SvVectorField

From `sverchok/utils/field/vector.py`. Maps `R³ → R³`.

- `field.evaluate(x, y, z)` → `tuple(dx, dy, dz)`
- `field.evaluate_grid(xs, ys, zs)` → three `numpy.ndarray` (dxs, dys, dzs)
- Two interpretations: relative (P + field(P)) or absolute (field(P))

### Part.Shape (Solids)

FreeCAD/OpenCascade BRep objects. Requires FreeCAD installed alongside Blender. Transported by `SvSolidSocket`. `SolidImplicitConversionPolicy` attempts `to_solid_recursive()` on connected data.

### dict (Dictionaries)

Python `dict` objects in `SvDictionarySocket`. Use `unzip_dict_recursive(data)` from `sverchok.utils.dictionary` to convert nested list-of-dicts to dict-of-nested-lists.

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures: SvSocketCommon, SvSocketProcessing, ConversionPolicies, replace_socket, socket_id, other
- [references/examples.md](references/examples.md) — Working code examples: custom socket types, field conversions, dynamic socket replacement
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with sockets, with WHY explanations

### Official Sources

- https://github.com/nortikin/sverchok/blob/master/core/sockets.py
- https://github.com/nortikin/sverchok/blob/master/core/socket_conversions.py
- https://github.com/nortikin/sverchok/blob/master/core/socket_data.py
