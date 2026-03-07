# Sverchok Python Integration -- Comprehensive Research Document

**Research agent:** sv-research-python
**Date:** 2026-03-07
**Sources:** Sverchok documentation (nortikin.github.io/sverchok), GitHub source (github.com/nortikin/sverchok), verified against source code from master branch.

---

## Table of Contents

1. [Script Node Lite (SNLite)](#1-script-node-lite-snlite)
2. [SN Functor B](#2-sn-functor-b)
3. [Formula Node Mk4](#3-formula-node-mk4)
4. [Profile Parametric Node](#4-profile-parametric-node)
5. [Custom Node Development](#5-custom-node-development)
6. [External API Access](#6-external-api-access)
7. [Key Internal APIs](#7-key-internal-apis)
8. [Common Python Patterns](#8-common-python-patterns)
9. [Mesh Expression Node](#9-mesh-expression-node)
10. [Generative Art Node](#10-generative-art-node)

---

## 1. Script Node Lite (SNLite)

### 1.1 Overview

Script Node Lite (SvScriptNodeLite, bl_idname: SvScriptNodeLite) is Sverchok's primary mechanism for writing arbitrary Python logic inside node trees. It is located in the Script category of the node menu. The node allows users to write Python scripts in Blender's Text Editor and load them into the node, which then creates input/output sockets based on a header directive.

Source file: nodes/script/script1_lite.py

### 1.2 Header Directive Syntax

Every SNLite script must begin with a triple-quoted header directive that defines input and output sockets. The header must appear as the first non-blank content in the script.

```python
"""
in socket_name  socket_type  d=default_value  n=nestedness
in socket_name  socket_type
out socket_name socket_type
"""
```

Rules:
- Triple double-quotes or triple single-quotes are both accepted.
- Each line inside the directive starts with "in" (for inputs) or "out" (for outputs).
- The socket name follows, then the socket type code.
- Optional: d=value or default=value sets a default value.
- Optional: n=int or nested=int sets the nestedness/unwrapping level.
- No spaces are allowed inside default iterables (this breaks the parser).

### 1.3 Socket Type Codes

The socket type codes are defined in utils/snlite_importhelper.py in the sock_dict mapping:

| Code | Socket Class | Description |
|------|-------------|-------------|
| v | SvVerticesSocket | Vertices (list of 3-tuples) |
| s | SvStringsSocket | Strings/Numbers/Generic lists |
| m | SvMatrixSocket | 4x4 Matrices |
| o | SvObjectSocket | Blender Objects |
| c | SvColorSocket | RGBA Colors |
| C | SvCurveSocket | Curve objects |
| D | SvDictionarySocket | Dictionary data |
| T | SvTextSocket | Text data |
| S | SvSurfaceSocket | Surface objects |
| So | SvSolidSocket | Solid objects (FreeCAD) |
| SF | SvScalarFieldSocket | Scalar fields |
| VF | SvVectorFieldSocket | Vector fields |
| FP | SvFilePathSocket | File paths |

### 1.4 Nestedness Levels

The n= parameter controls how deeply the incoming data is unwrapped before being handed to the script:

- n=0 (default): The variable receives the full nested structure as returned by sv_get(), typically [[[values]]].
- n=1: The variable receives data[0], one level unwrapped.
- n=2: The variable receives data[0][0], two levels unwrapped -- typically a single scalar value.

### 1.5 Required Input Sockets

Sockets prefixed with >in instead of in are marked as required. The node will not process if required sockets are unconnected. Required sockets do not accept default or nestedness parameters.

```python
"""
>in verts v
in scale s d=1.0 n=2
out verts_out v
"""
```

### 1.6 Extended Input Syntax

Input sockets can use an extended format with +in for setting display names:

```python
"""
+in my_var s d=1.0 n=2 name="My Variable"
out result s
"""
```

### 1.7 Pre-imported Helpers

SNLite automatically makes the following available without explicit import:

- vectorize -- utility from sverchok.utils.snlite_utils for iterating through matched argument pairs
- bmesh_from_pydata -- converts (verts, edges, faces) into a BMesh object
- pydata_from_bmesh -- converts a BMesh back to (verts, edges, faces) tuples
- ddir(object, filter_str=None) -- filtered dir() that hides dunder attributes
- np -- NumPy
- bpy -- Blender Python API
- console_print -- print to Sverchok console
- get_user_dict() -- access per-instance state dictionary
- reset_user_dict(hard=False) -- clear the per-instance state dictionary

### 1.8 Auto-Injection Feature

Adding inject to the header directive automatically creates a parameters variable containing all input socket references as a list:

```python
"""
inject
in x s d=1.0 n=2
in y s d=2.0 n=2
out result s
"""
# parameters is automatically set to [x, y]
```

### 1.9 Enum Support

SNLite supports custom enumerations via the enum and enum2 directives in the header:

```python
"""
in verts v
enum = A B
enum2 = A B
out overts v
"""

def ui(self, context, layout):
    layout.prop(self, 'custom_enum', expand=True)
    layout.prop(self, 'custom_enum_2', expand=True)

for vex in verts:
    offset = 0 if self.custom_enum == "A" else 1
    offset2 = 0 if self.custom_enum_2 == "A" else 1
    overts.append([[v[0], v[1]+offset2, v[2]+offset] for v in vex])
```

### 1.10 Custom UI Drawing

Scripts can define a ui(self, context, layout) function to draw custom UI elements in the node:

```python
"""
in verts v
out result v
"""

def ui(self, context, layout):
    cb_str = 'node.scriptlite_custom_callback'
    layout.operator(cb_str, text='Click Me').cb_name = 'my_function'
```

### 1.11 Operator Callbacks

Scripts can define custom operator callbacks that are invoked from UI buttons:

```python
"""
in verts v
"""

def my_operator(self, context):
    print(self, context, self.inputs['verts'].sv_get())
    return {'FINISHED'}

self.make_operator('my_operator')

def ui(self, context, layout):
    cb_str = 'node.scriptlite_custom_callback'
    layout.operator(cb_str, text='show me').cb_name = 'my_operator'
```

### 1.12 Custom Properties (State Persistence)

SNLite nodes can store persistent state using Blender's ID properties:

```python
"""
out float_out s
out int_out s
out bool_out s
out vec_out v
out color_out c
out str_out T
out obj_out o
out coll_out o
"""

# Initialization -- only runs if property does not exist yet
if 'my_float' not in self.keys():
    self['my_float'] = 0.5
if 'my_integer' not in self.keys():
    self['my_integer'] = 5
if 'my_boolean' not in self.keys():
    self['my_boolean'] = 0
    bool_prop = self.id_properties_ui("my_boolean")
    bool_prop.update(min=0, max=1)
if 'my_vector' not in self.keys():
    self['my_vector'] = (0.0, 0.0, 1.0)
if 'my_color' not in self.keys():
    self['my_color'] = (1.0, 0.0, 0.0, 1.0)
    col_prop = self.id_properties_ui("my_color")
    col_prop.update(soft_min=0.0, soft_max=1.0, default=(0, 0, 0, 1.0), subtype="COLOR_GAMMA")
if 'my_string' not in self.keys():
    self['my_string'] = 'Hello World!'

# Outputs
float_out = [[self['my_float']]]
int_out = [[self['my_integer']]]
bool_out = [[self['my_boolean']]]
vec_out = [[tuple(self['my_vector'])]]
color_out = [[tuple(self['my_color'])]]
str_out = [[self['my_string']]]

# UI
def ui(self, context, layout):
    layout.prop(self, '["my_float"]', text="MyFloat")
    layout.prop(self, '["my_integer"]', text="MyInteger")
    layout.prop(self, '["my_boolean"]', text="MyBoolean")
    layout.prop(self, '["my_vector"]', text="MyVector")
    layout.prop(self, '["my_color"]', text="MyColor")
    layout.prop(self, '["my_string"]', text="MyString")
```

### 1.13 User Dictionary (Runtime Caching)

For caching computed results within a session:

```python
"""
in verts v
out vout v
"""

def some_long_calculation():
    return [(0.0, 0.0, 0.0), (1.0, 1.0, 1.0)]

cache = get_user_dict()

if not (data := cache.get("current_state")):
    data = some_long_calculation()
    cache["current_state"] = data

vout.append(data)
```

### 1.14 Include Directive

Scripts can embed dependencies from other text blocks:

```python
"""
include <my_helper_module>
in x s
out y s
"""
```

This ensures the dependency is included when exporting the node tree to JSON.

### 1.15 Complete Working Example: Lissajous Curves

```python
"""
in N s d=20 n=2
in t s d=1.0 n=2
in k1 s d=1.0 n=2
in k2 s d=2.0 n=2
in k3 s d=3.0 n=2
out vertices_out v
out edges_out s
"""

import math

verts = []
edges = []

N = abs(N)
for I in range(N + 1):
    v = t / N * I
    verts.append([math.sin(k1 * v),
                  math.cos(k2 * v),
                  math.sin(k3 * v)])
    edges.append([I, I + 1])
edges.pop()

vertices_out = [verts]
edges_out = [edges]
```

### 1.16 Complete Working Example: Petal Sine Curve

```python
"""
in   n_petals s        default=8       nested=2
in   vp_petal s        default=10      nested=2
in   profile_radius s  d=2.3           n=2
in   amp s             default=1.0     nested=2
in   origin v          default=(0,0,0) n=2
out  verts v
out  edges s
"""

from math import sin, cos, radians, pi
from mathutils import Vector, Euler

n_verts = n_petals * vp_petal
section_angle = 360.0 / n_verts
position = (2 * (pi / (n_verts / n_petals)))
x, y, z = origin[:3]

Verts = []
Edges = []

for i in range(n_verts):
    difference = amp * cos(i * position)
    arm = profile_radius + difference
    ampline = Vector((arm, 0.0, 0.0))
    rad_angle = radians(section_angle * i)
    myEuler = Euler((0.0, 0.0, rad_angle), 'XYZ')
    ampline.rotate(myEuler)
    Verts.append((ampline.x + x, ampline.y + y, 0.0 + z))

if Verts:
    Edges.extend([[i, i + 1] for i in range(n_verts - 1)])
    Edges.append([len(Edges), 0])

verts = [Verts]
edges = [Edges]
```

### 1.17 Complete Working Example: Fibonacci Sphere (NumPy)

```python
"""
in samples s d=400 n=2
in rseed s d=4 n=2
in mult s d=1.0 n=2
out verts_out v
"""
import numpy as np
import math, random

def fibonacci_sphere_np(samples, rseed):
    indices = np.arange(samples)
    rnd = 1.
    random.seed(rseed)
    rnd = random.random() * samples
    offset = 2. / samples
    increment = math.pi * (3. - math.sqrt(5.))
    y = ((indices * offset) - 1) + (offset / 2)
    r = np.sqrt(1 - pow(y, 2))
    phi = ((indices + rnd) % samples) * increment
    x = np.cos(phi) * r
    z = np.sin(phi) * r
    return (np.vstack([x, y, z]) * mult).T.tolist()

p = fibonacci_sphere_np(samples, rseed)
verts_out.extend([p])
```

### 1.18 Template System

SNLite provides a built-in template system located in node_scripts/SNLite_templates/ organized by category:

- demo/ -- petal_sine, fibonacci_sphere, voronoi_3d, archimedes_spiral, bezier_gear, etc.
- bpy_stuff/ -- Blender API integration (custom properties, empties, bgl drawing)
- bmesh/ -- BMesh operations (solidification, polyhedra wireframe, scanline fill)
- utils/ -- Utility scripts
- templates/ -- Reusable patterns (caching, custom_properties, enum_example, operator_example)

### 1.19 Keyboard Shortcuts

Ctrl+Enter: Reload the script from the Text Editor.

### 1.20 Node Controls (After Loading)

- Animate Node: Enable/disable updates during animation playback.
- Update Node: Manual trigger for re-execution.
- Reload: Re-parse and reload the script from the text block.
- Clear: Reset the node back to its unloaded state.

---

## 2. SN Functor B

### 2.1 Overview

SN Functor B (SvSNFunctorB, bl_idname: SvSNFunctorB) is a script node with more explicit (verbose) initialization syntax compared to SNLite. It uses named Python functions rather than a header-based socket declaration.

Source file: nodes/script/sn_functor_b.py

### 2.2 Script Structure

A Functor B script defines up to three top-level functions:

```python
def functor_init(self, context):
    """Called when the script is loaded. Creates sockets."""
    pass

def draw_buttons(self, context, layout):
    """Optional. Draws custom UI elements in the node."""
    pass

def process(self):
    """Called each time the node needs to update."""
    pass
```

### 2.3 Socket Creation in functor_init

```python
def functor_init(self, context):
    inew = self.inputs.new
    inew('SvStringsSocket', 'outer radius').prop_name = 'float_01'
    inew('SvStringsSocket', 'inner radius').prop_name = 'float_02'
    inew('SvStringsSocket', 'angle').prop_name = 'float_03'

    onew = self.outputs.new
    onew('SvVerticesSocket', 'verts')
    onew('SvStringsSocket', 'edges')
    onew('SvStringsSocket', 'faces')
```

### 2.4 Built-in Properties

Functor B provides pre-defined properties:
- int_01 through int_04 -- IntProperty instances
- float_01 through float_04 -- FloatProperty instances
- bool_01 through bool_04 -- BoolProperty instances

### 2.5 Complete Working Example

```python
from math import sin, cos, pi

def functor_init(self, context):
    inew = self.inputs.new
    inew('SvStringsSocket', 'segments').prop_name = 'int_01'
    inew('SvStringsSocket', 'radius').prop_name = 'float_01'
    inew('SvStringsSocket', 'height').prop_name = 'float_02'

    onew = self.outputs.new
    onew('SvVerticesSocket', 'verts')
    onew('SvStringsSocket', 'edges')
    onew('SvStringsSocket', 'faces')

def draw_buttons(self, context, layout):
    pass

def process(self):
    segments = int(self.inputs['segments'].sv_get()[0][0])
    radius = self.inputs['radius'].sv_get()[0][0]
    height = self.inputs['height'].sv_get()[0][0]

    verts = []
    for i in range(segments):
        angle = 2 * pi * i / segments
        x = radius * cos(angle)
        y = radius * sin(angle)
        verts.append((x, y, 0.0))
        verts.append((x, y, height))

    faces = []
    for i in range(segments):
        i0 = i * 2
        i1 = i * 2 + 1
        i2 = ((i + 1) % segments) * 2 + 1
        i3 = ((i + 1) % segments) * 2
        faces.append([i0, i1, i2, i3])

    edges = []
    self.outputs['verts'].sv_set([verts])
    self.outputs['edges'].sv_set([edges])
    self.outputs['faces'].sv_set([faces])
```

### 2.6 SNLite vs Functor B Comparison

| Aspect | SNLite | Functor B |
|--------|--------|-----------|
| Socket declaration | Header directive (concise) | Explicit Python code (verbose) |
| Default properties | Via d= in header | Via prop_name on predefined properties |
| UI customization | def ui(self, context, layout) | def draw_buttons(self, context, layout) |
| State management | User dict, ID properties | Direct access to node properties |
| Script structure | Flat script with header | Three named functions |
| Template system | Built-in template menu | Manual text block management |

---

## 3. Formula Node Mk4

### 3.1 Overview

The Formula Node Mk4 (source: formula_mk5.py) evaluates arbitrary Python expressions using connected inputs as variables. Up to 4 formulas simultaneously.

Source file: nodes/script/formula_mk5.py

### 3.2 Expression Syntax

Variables used in formulas automatically create input sockets.

Valid expressions:
```
1.0
x
x + 1
0.75*X + 0.25*Y
R * sin(phi)
sqrt(x**2 + y**2)
[x, y, z]
Vector((x, y, z)).normalized()
```

### 3.3 Permitted Functions

Math module: acos, acosh, asin, asinh, atan, atan2, atanh, ceil, copysign, cos, cosh, degrees, erf, erfc, exp, expm1, fabs, factorial, floor, fmod, frexp, fsum, gamma, hypot, isfinite, isinf, isnan, ldexp, lgamma, log, log10, log1p, log2, modf, pow, radians, sin, sinh, sqrt, tan, tanh, trunc

Constants: pi, e

Additional: abs, sign, max, min, len, sum, any, all, dir

Type constructors: tuple, list, str, dict, set, int, float

External: Vector (mathutils), Matrix (mathutils), np (NumPy), bpy

### 3.4 Input Configuration

Depth: Controls nesting level of exposed data (1=scalars, 2=scalar lists, 3+=nested lists)

Transform options: As Is, Vector, Array, Set, String

### 3.5 Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| Dimensions | Number of formulas (1-4) | 1 |
| Formula 1-4 | Expression text fields | Empty |
| Split | Wrap each result in a separate list | Off |
| Wrapping | Adjust bracket nesting (-1, 0, +1) | 0 |
| Use AST | Alternative evaluation mode | Off |
| List Match | Length-matching method | Repeat Last |

---

## 4. Profile Parametric Node

### 4.1 Overview

The Profile Parametric Node (source: profile_mk3.py) implements a subset of SVG path commands through a domain-specific language (DSL). It generates 2D curves from text-based profile definitions stored in Blender text blocks.

Source file: nodes/script/profile_mk3.py

### 4.2 Variable Declarations

```
default <name> = <value>    # Creates variable AND input socket
let <name> = <expression>   # Creates temporary variable, no socket
```

### 4.3 Supported Commands

| Command | Parameters | Description |
|---------|-----------|-------------|
| M, m | x y | Move to (absolute/relative) |
| L, l | (x y)+ [n=segments] [z] | Line to |
| H, h | (x)+ [n=segments] ; | Horizontal line to |
| V, v | (y)+ [n=segments] ; | Vertical line to |
| C, c | (cx1 cy1 cx2 cy2 x y)+ [n=verts] [z] | Cubic Bezier |
| S, s | (cx2 cy2 x y)+ [n=verts] [z] | Smooth cubic Bezier |
| Q, q | (cx cy x y)+ [n=segments] [z] | Quadratic Bezier |
| T, t | (x y)+ [n=segments] [z] | Smooth quadratic Bezier |
| A, a | rx ry rot flag1 flag2 x y [n=verts] [z] | Arc |
| @I, @i | [@smooth] degree (x y)+ [n=segments] [z] ; | NURBS interpolation |
| x | -- | Close path |
| X | -- | Close all edges cyclically |
| # | text | Comment |

Uppercase = absolute coordinates; lowercase = relative coordinates.

### 4.4 Value Representations

- Integer/float literals: 5, 7.5
- Variable names: width, height
- Negated variables: -width, -size
- Expressions in curly brackets: {width/2}, {sin(phi)}, {a + 1}

### 4.5 Architectural Profile Examples

Simple Rectangle:
```
l width,0 0,height {-width},0 z
```

I-Beam Profile:
```
default bottom_w = 0.6
default bottom_dx = 0.1
default bottom_height = 0.3
default base_height = 1
default base_w = 0.2
default top_w = 1.2
default top_dx = bottom_dx
default top_height = bottom_height
default base_dx = bottom_dx

H bottom_w ;
l bottom_dx, bottom_height
H base_w ;
l base_dx, base_height
H top_w ;
l top_dx, top_height
H 0 ;
```

Complex Rectangle with Rounded Corners:
```
default width = 4
default height = 3
default radius = 0.5
default b_width = 2
default b_height = 0.5

let w2 = {width/2 - radius}
let w = {width - 2*radius}
let h = {height - 2*radius}
let dw = {(w - b_width)/2}

H w2;
q radius,0 radius,radius
v h ;
q 0,radius -radius,radius
h -dw ;
v -b_height ;
h -b_width ;
v b_height ;
h -dw;
q -radius,0 -radius,-radius
v -h;
q 0,-radius radius,-radius
X
```

### 4.6 Node Outputs

- Vertices -- Curve vertices
- Edges -- Edge index pairs
- Knots -- Knot points from curve commands
- KnotNames -- Identifiers for knot points
- Curve -- Individual Curve objects per segment

---

## 5. Custom Node Development

### 5.1 Overview

Developers create new node types by subclassing SverchCustomTreeNode from sverchok.node_tree.

Source file: node_tree.py

### 5.2 Base Classes

```python
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode
```

SverchCustomTreeNode inherits from UpdateNodes, NodeUtils, NodeDependencies, NodeDocumentation.

Key methods:
- sv_init(context) -- Node initialization
- process() -- Node evaluation
- sv_draw_buttons(context, layout) -- Custom UI
- sv_draw_buttons_ext(context, layout) -- Extended panel UI
- sv_update() -- React to tree editor changes
- sv_copy(node) -- Custom copy behavior
- sv_free() -- Cleanup on removal

### 5.3 Node Tree Type

```python
class SverchCustomTree(NodeTree, SvNodeTreeCommon):
    bl_idname = 'SverchCustomTreeType'
    bl_label = 'Sverchok Nodes'
    bl_icon = 'RNA'
```

Key tree properties: sv_process, sv_animate, sv_show, sv_draft

Key tree methods: force_update(), update_nodes(nodes), init_tree()

### 5.4 Complete Custom Node Tutorial

```python
import bpy
from bpy.props import IntProperty, FloatProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode


class SvGridGeneratorNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: grid points
    Tooltip: Generate a grid of 3D points
    """

    bl_idname = 'SvGridGeneratorNode'
    bl_label = 'Grid Generator'
    bl_icon = 'MESH_GRID'

    grid_x: IntProperty(name='X Count', default=5, min=2, update=updateNode)
    grid_y: IntProperty(name='Y Count', default=5, min=2, update=updateNode)
    spacing: FloatProperty(name='Spacing', default=1.0, min=0.01, update=updateNode)

    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'X Count').prop_name = 'grid_x'
        self.inputs.new('SvStringsSocket', 'Y Count').prop_name = 'grid_y'
        self.inputs.new('SvStringsSocket', 'Spacing').prop_name = 'spacing'

        self.outputs.new('SvVerticesSocket', 'Vertices')
        self.outputs.new('SvStringsSocket', 'Edges')
        self.outputs.new('SvStringsSocket', 'Faces')

    def process(self):
        if not any(s.is_linked for s in self.outputs):
            return

        x_count = int(self.inputs['X Count'].sv_get()[0][0])
        y_count = int(self.inputs['Y Count'].sv_get()[0][0])
        spacing = self.inputs['Spacing'].sv_get()[0][0]

        verts = []
        for j in range(y_count):
            for i in range(x_count):
                verts.append((i * spacing, j * spacing, 0.0))

        edges = []
        for j in range(y_count):
            for i in range(x_count - 1):
                idx = j * x_count + i
                edges.append((idx, idx + 1))
        for j in range(y_count - 1):
            for i in range(x_count):
                idx = j * x_count + i
                edges.append((idx, idx + x_count))

        faces = []
        for j in range(y_count - 1):
            for i in range(x_count - 1):
                idx = j * x_count + i
                faces.append((idx, idx + 1, idx + x_count + 1, idx + x_count))

        self.outputs['Vertices'].sv_set([verts])
        self.outputs['Edges'].sv_set([edges])
        self.outputs['Faces'].sv_set([faces])


def register():
    bpy.utils.register_class(SvGridGeneratorNode)

def unregister():
    bpy.utils.unregister_class(SvGridGeneratorNode)
```

### 5.5 Docstring Convention

```python
class MyNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: keyword1 keyword2
    Tooltip: Short description shown on hover
    """
```

### 5.6 Node Registration

1. Each node file must have register() and unregister() functions.
2. Nodes are organized via the index.yaml file at the repository root.
3. Sverchok scans the nodes/ directory for Python modules.

### 5.7 The updateNode Callback

```python
def updateNode(self, context):
    """Triggers node.process_node(context) for partial update."""
    self.process_node(context)
```

### 5.8 Node Lifecycle

1. Creation: init() -> sv_init(context)
2. Property change: process_node(context) via updateNode
3. Tree topology change: update() -> sv_update()
4. Evaluation: process()
5. Copy: copy() -> sv_copy(node)
6. Deletion: free() -> sv_free()

---

## 6. External API Access

### 6.1 Accessing Node Trees

```python
import bpy

for ng in bpy.data.node_groups:
    if ng.bl_idname == 'SverchCustomTreeType':
        print(f"Sverchok tree: {ng.name}")
```

### 6.2 Creating Node Trees

```python
import bpy

tree = bpy.data.node_groups.new('MyTree', 'SverchCustomTreeType')

with tree.init_tree():
    plane = tree.nodes.new('SvPlaneNodeMk3')
    plane.location = (0, 0)

    move = tree.nodes.new('SvMoveNodeMk3')
    move.location = (200, 0)

    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (400, 0)

    tree.links.new(plane.outputs['Vertices'], move.inputs['Vertices'])
    tree.links.new(move.outputs['Vertices'], viewer.inputs['Vertices'])

tree.force_update()
```

### 6.3 Batch Processing

```python
import bpy

def batch_process_tree(tree_name, parameter_sets):
    tree = bpy.data.node_groups[tree_name]
    results = []

    for params in parameter_sets:
        for node_name, prop_name, value in params:
            tree.nodes[node_name][prop_name] = value
        tree.force_update()
        output_node = tree.nodes['Output Node']
        result = output_node.outputs['Result'].sv_get()
        results.append(result)

    return results
```

### 6.4 Tree Control

```python
tree.sv_process = True   # Enable/disable automatic processing
tree.sv_animate = True   # Enable/disable animation
tree.sv_show = True      # Toggle viewport visibility
tree.sv_draft = True     # Enable draft mode
tree.force_update()      # Force complete recalculation
```

### 6.5 The init_tree Context Manager

```python
with tree.init_tree():
    # Suppresses update() calls during construction
    node1 = tree.nodes.new('SvSomeNode')
    node2 = tree.nodes.new('SvOtherNode')
    tree.links.new(node1.outputs[0], node2.inputs[0])
```

---

## 7. Key Internal APIs

### 7.1 data_structure.py

Located at the repository root. Provides list matching, data nesting, and utility functions.

#### List Matching Functions

```python
from sverchok.data_structure import match_long_repeat

result = match_long_repeat([[1, 2, 3, 4, 5], [10, 11]])
# [[1, 2, 3, 4, 5], [10, 11, 11, 11, 11]]
```

```python
from sverchok.data_structure import zip_long_repeat

for a, b in zip_long_repeat([1, 2, 3, 4, 5], [10, 11]):
    print(a, b)
# 1 10, 2 11, 3 11, 4 11, 5 11
```

```python
from sverchok.data_structure import match_long_cycle

result = match_long_cycle([[1, 2, 3, 4, 5], [10, 11]])
# [[1, 2, 3, 4, 5], [10, 11, 10, 11, 10]]
```

```python
from sverchok.data_structure import match_short

result = match_short([[1, 2, 3, 4, 5], [10, 11]])
# [[1, 2], [10, 11]]
```

```python
from sverchok.data_structure import match_cross

result = match_cross([[1, 2], [5, 6, 7]])
# [[1, 1, 1, 2, 2, 2], [5, 6, 7, 5, 6, 7]]
```

#### List Match Mode Registry

```python
list_match_modes = [
    ("SHORT",  "Short",        "Match shortest List",    1),
    ("CYCLE",  "Cycle",        "Match by cycling",       2),
    ("REPEAT", "Repeat Last",  "Match by repeating",     3),
    ("XREF",   "X-Ref",        "Cross reference",        4),
    ("XREF2",  "X-Ref 2",      "Cross reference rev",    5),
]

list_match_func = {
    "SHORT":  match_short,
    "CYCLE":  match_long_cycle,
    "REPEAT": match_long_repeat,
    "XREF":   match_cross,
    "XREF2":  match_cross2,
}
```

#### List Extension Functions

```python
from sverchok.data_structure import fullList
data = [1, 2, 3]
fullList(data, 6)  # data is now [1, 2, 3, 3, 3, 3]

from sverchok.data_structure import repeat_last
gen = repeat_last([1, 2, 3])  # yields: 1, 2, 3, 3, 3, ... (infinite)

from sverchok.data_structure import fixed_iter
list(fixed_iter([1, 2, 3], 5))  # [1, 2, 3, 3, 3]

from sverchok.data_structure import cycle_for_length
cycle_for_length([1, 2, 3], 7)  # [1, 2, 3, 1, 2, 3, 1]

from sverchok.data_structure import repeat_last_for_length
repeat_last_for_length([1, 2], 5)  # [1, 2, 2, 2, 2]
```

#### Data Nesting Functions

```python
from sverchok.data_structure import get_data_nesting_level
get_data_nesting_level(1)            # 0
get_data_nesting_level([1])          # 1
get_data_nesting_level([[(1,2,3)]])  # 3

from sverchok.data_structure import ensure_nesting_level
ensure_nesting_level(17, 1)    # [17]
ensure_nesting_level([17], 2)  # [[17]]

from sverchok.data_structure import ensure_min_nesting
ensure_min_nesting([[[17]]], 1)  # [[[17]]] (unchanged)

from sverchok.data_structure import flatten_data
flatten_data([[[1, 2], [3, 4]], [[5, 6]]], target_level=1)  # [1, 2, 3, 4, 5, 6]

from sverchok.data_structure import flat_iter
list(flat_iter([1, [2, 3, [4]], 5]))  # [1, 2, 3, 4, 5]
```

#### NumPy List Matching

```python
from sverchok.data_structure import numpy_full_list
import numpy as np

arr = np.array([1.0, 2.0, 3.0])
result = numpy_full_list(arr, 6)  # array([1., 2., 3., 3., 3., 3.])

from sverchok.data_structure import numpy_match_long_repeat
arr1 = np.array([1.0, 2.0, 3.0, 4.0, 5.0])
arr2 = np.array([10.0, 11.0])
result = numpy_match_long_repeat([arr1, arr2])
# [array([1., 2., 3., 4., 5.]), array([10., 11., 11., 11., 11.])]
```

### 7.2 utils/vectorize.py

```python
from sverchok.utils.vectorize import vectorize
from typing import List, Tuple

def my_function(*, verts: List[Tuple[float, float, float]],
                scale: float) -> Tuple[list, list]:
    scaled = [(v[0]*scale, v[1]*scale, v[2]*scale) for v in verts]
    edges = [(i, i+1) for i in range(len(scaled)-1)]
    return scaled, edges

# In node process():
vectorized = vectorize(my_function, match_mode='REPEAT')
out_verts, out_edges = vectorized(verts=input_verts, scale=input_scale)
```

```python
from sverchok.utils.vectorize import match_sockets

data1 = [[1, 2, 3]]
data2 = [[4, 5], [6, 7]]
for d1, d2 in match_sockets(data1, data2):
    print(d1, d2)
```

### 7.3 utils/snlite_utils.py

```python
from sverchok.utils.snlite_utils import vectorize, ddir

vectorize([1.0, [2, 3, 4]])  # Match via match_long_repeat

ddir(bpy.types.Object)           # All non-dunder attributes
ddir(bpy.types.Object, 'mesh')   # Filtered
```

### 7.4 utils/sv_bmesh_utils.py

```python
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata, pydata_from_bmesh

bm = bmesh_from_pydata(verts, edges, faces)
# ... BMesh operations ...
verts_out, edges_out, faces_out = pydata_from_bmesh(bm)
bm.free()
```

### 7.5 core/ Directory Structure

| File | Purpose |
|------|---------|
| event_system.py | Central event dispatcher |
| events.py | Event type definitions |
| main_tree_handler.py | Main tree evaluation handler |
| update_system.py | Node update ordering |
| sockets.py | Socket type definitions |
| socket_data.py | Socket data storage (sv_get/sv_set) |
| socket_conversions.py | Automatic type conversions |
| node_group.py | Group node implementation |
| sv_custom_exceptions.py | SvNoDataError, DependencyError |
| handlers.py | Blender handler registrations |

---

## 8. Common Python Patterns

### 8.1 Standard Data Structure

```
Level 0: Single value          (5.0)
Level 1: List of values        ([1.0, 2.0, 3.0])
Level 2: List of lists         ([[1.0, 2.0], [3.0, 4.0]])
```

Standard flow: Vertices [[(x,y,z),...]], Edges [[(i,j),...]], Faces [[[i,j,k,...],...]]

### 8.2 Multi-Object Processing

```python
from sverchok.data_structure import zip_long_repeat

def process(self):
    verts_in = self.inputs['Vertices'].sv_get()
    scale_in = self.inputs['Scale'].sv_get()
    verts_out = []

    for verts, scale in zip_long_repeat(verts_in, scale_in):
        new_verts = []
        for v, s in zip_long_repeat(verts, scale):
            new_verts.append((v[0]*s, v[1]*s, v[2]*s))
        verts_out.append(new_verts)

    self.outputs['Vertices'].sv_set(verts_out)
```

### 8.3 NumPy Integration

```python
import numpy as np

def process(self):
    verts_in = self.inputs['Vertices'].sv_get()
    verts_out = []
    for verts in verts_in:
        np_verts = np.array(verts, dtype=np.float64)
        result = np_verts * 2.0  # vectorized operation
        verts_out.append(result.tolist())
    self.outputs['Vertices'].sv_set(verts_out)
```

### 8.4 BMesh Operations

```python
import bmesh
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata, pydata_from_bmesh

def process(self):
    verts_in = self.inputs['Vertices'].sv_get()
    faces_in = self.inputs['Faces'].sv_get(default=[[]])
    verts_out, edges_out, faces_out = [], [], []

    for verts, faces in zip_long_repeat(verts_in, faces_in):
        bm = bmesh_from_pydata(verts, [], faces)
        bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=1)
        v, e, f = pydata_from_bmesh(bm)
        bm.free()
        verts_out.append(v)
        edges_out.append(e)
        faces_out.append(f)

    self.outputs['Vertices'].sv_set(verts_out)
```

### 8.5 Error Handling

```python
from sverchok.core.sv_custom_exceptions import SvNoDataError

def process(self):
    try:
        data = self.inputs['Data'].sv_get()
        result = compute(data)
        self.outputs['Result'].sv_set(result)
    except SvNoDataError:
        pass  # Normal: socket has no data
```

### 8.6 Matrix Operations

```python
from mathutils import Matrix, Vector

def process(self):
    verts_in = self.inputs['Vertices'].sv_get()
    matrices = self.inputs['Matrix'].sv_get(default=[[Matrix()]])
    verts_out = []

    for verts, mats in zip_long_repeat(verts_in, matrices):
        new_verts = []
        for v, m in zip_long_repeat(verts, mats):
            transformed = m @ Vector(v)
            new_verts.append(tuple(transformed))
        verts_out.append(new_verts)

    self.outputs['Vertices'].sv_set(verts_out)
```

---

## 9. Mesh Expression Node

### 9.1 Overview

Generates meshes from JSON descriptions where vertex coordinates can contain mathematical expressions.

Source file: nodes/script/mesh_eval.py

### 9.2 JSON Structure

```json
{
    "vertices": [
        [0, 0, 0],
        ["Size", 0, 0],
        ["Size", "Size", 0],
        [0, "Size", 0]
    ],
    "edges": [[0, 1], [1, 2], [2, 3], [3, 0]],
    "faces": [[0, 1, 2, 3]],
    "defaults": {"Size": 1.0}
}
```

### 9.3 Adjustable Plane Example

```json
{
    "faces": [[0, 1, 3, 2]],
    "edges": [[0, 2], [0, 1], [1, 3], [2, 3]],
    "vertices": [
        ["-Size", "-Size", 0.0],
        ["Size", "-Size", 0.0],
        ["-Size", "Size", 0.0],
        ["Size", "Size", 0.0]
    ]
}
```

---

## 10. Generative Art Node

### 10.1 Overview

Generates recursive 3D structures following XML-based designs, similar to fractals or L-systems.

Source file: nodes/script/generative_art.py

### 10.2 XML Grammar

```xml
<rules max_depth="150">
    <rule name="entry">
        <call count="3" transforms="rz 120" rule="R1"/>
    </rule>
    <rule name="R1">
        <call transforms="tx 2.6 rz 12 sa 0.97" rule="R1"/>
        <instance transforms="sa 2.6" shape="box"/>
    </rule>
</rules>
```

Transform commands: tx, ty, tz (translation), rx, ry, rz (rotation), sx, sy, sz (scale), sa (uniform scale).

Variables: Replace numeric values with {variable_name} for parameterization.

---

## Appendix A: Socket Type Reference

| bl_idname | SNLite Code | Python Type |
|-----------|-------------|-------------|
| SvStringsSocket | s | int, float, str, list |
| SvVerticesSocket | v | (float, float, float) |
| SvMatrixSocket | m | mathutils.Matrix |
| SvColorSocket | c | (float, float, float, float) |
| SvObjectSocket | o | bpy.types.Object |
| SvTextSocket | T | str |
| SvDictionarySocket | D | dict |
| SvCurveSocket | C | SvCurve |
| SvSurfaceSocket | S | SvSurface |
| SvSolidSocket | So | Part.Shape |
| SvScalarFieldSocket | SF | SvScalarField |
| SvVectorFieldSocket | VF | SvVectorField |
| SvFilePathSocket | FP | str |
| SvQuaternionSocket | -- | mathutils.Quaternion |

## Appendix B: Key Import Paths

```python
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import (
    updateNode, match_long_repeat, zip_long_repeat,
    zip_long_repeat_recursive, match_long_cycle, match_short,
    match_cross, match_cross2, fullList, fullList_deep_copy,
    repeat_last, repeat_last_for_length, cycle_for_length,
    fixed_iter, flat_iter, ensure_nesting_level, ensure_min_nesting,
    flatten_data, graft_data, get_data_nesting_level,
    get_max_data_nesting_level, levelsOflist, dataCorrect,
    dataCorrect_np, node_id, list_match_modes, list_match_func,
    numpy_full_list, numpy_full_list_cycle,
    numpy_match_long_repeat, numpy_match_long_cycle,
    numpy_match_short, numpy_list_match_func,
    map_at_level, wrap_data, unwrap_data,
)
from sverchok.utils.vectorize import vectorize, match_sockets
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata, pydata_from_bmesh
from sverchok.utils.sv_logging import sv_logger
from sverchok.utils.snlite_utils import vectorize as snlite_vectorize, ddir
from sverchok.core.sv_custom_exceptions import SvNoDataError, DependencyError
from sverchok.core.event_system import handle_event
import sverchok.core.events as ev
```

## Appendix C: Script Node Category

| Node | Source File | Purpose |
|------|-----------|---------|
| Script Node Lite | script1_lite.py | Lightweight Python scripting |
| SN Functor B | sn_functor_b.py | Verbose Python scripting |
| Formula Node Mk4 | formula_mk5.py | Expression evaluation |
| Profile Parametric | profile_mk3.py | 2D profile DSL |
| Generative Art | generative_art.py | Recursive XML structures |
| Mesh Expression | mesh_eval.py | JSON mesh definition |
| Formula Interpolate | formula_interpolate.py | Interpolation formulas |
| Num Expression | numexpr_node.py | NumExpr evaluation |
| Exec Node Mod | multi_exec.py | Inline script execution |
| Topology Simple | topology_simple.py | Topology operations |

---

*Document generated from Sverchok source code (github.com/nortikin/sverchok, master branch) and official documentation (nortikin.github.io/sverchok/docs). All code examples verified against source code structure.*
