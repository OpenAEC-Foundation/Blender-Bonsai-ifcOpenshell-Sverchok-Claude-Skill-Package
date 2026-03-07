---
name: sverchok-impl-custom-nodes
description: 'Complete guide to developing custom Sverchok nodes including the full node lifecycle, socket creation, property management, BMesh integration, and registration patterns. Use this when writing a custom Sverchok node, creating a new node type, or packaging nodes as an add-on.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-impl-custom-nodes

## Quick Reference

### Custom Node Inheritance

Every Sverchok custom node MUST inherit from both `SverchCustomTreeNode` and `bpy.types.Node`:

```python
from sverchok.node_tree import SverchCustomTreeNode
import bpy

class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'
```

`SverchCustomTreeNode` provides the mixins `UpdateNodes`, `NodeUtils`, `NodeDependencies`, and `NodeDocumentation`.

### Critical Warnings

**NEVER** call `process()` directly — ALWAYS let the update system invoke it via `updateNode` or `tree.force_update()`.

**NEVER** use `bpy.props` `update=` callbacks other than `updateNode` for triggering node re-evaluation — custom callbacks bypass the Sverchok update system.

**NEVER** mutate data returned by `sv_get()` unless you used `deepcopy=True` (the default) — mutating shared cache data corrupts upstream outputs.

**NEVER** create sockets outside of `sv_init()` or `sv_update()` — socket creation during `process()` causes infinite update loops.

**ALWAYS** use `updateNode` from `sverchok.data_structure` as the `update=` callback on ALL `bpy.props` properties that affect node output.

**ALWAYS** check `self.outputs[name].is_linked` before computing — skip processing when no downstream node consumes the output.

**ALWAYS** call `bm.free()` after using a BMesh — leaked BMesh objects cause memory leaks that persist until Blender restarts.

**ALWAYS** wrap socket data in the correct nesting level: `[[data]]` for single-object output, `[[data1], [data2]]` for multi-object.

### Decision Tree

```
Writing a custom Sverchok node?
├── Simple data transformation → Minimal node template (Pattern 1)
├── Needs UI controls → Add sv_draw_buttons (Pattern 3)
├── Geometry processing → BMesh integration pattern (Pattern 5)
├── Performance-critical → NumPy vectorized pattern (Pattern 6)
└── External add-on → Registration pattern (Pattern 7)

Choosing socket creation method?
├── Fixed sockets → Direct creation in sv_init()
├── Socket with default property → sv_new_input() with prop_name
├── Mandatory input (error if empty) → sv_new_input() with is_mandatory=True
└── Dynamic socket count → multi_socket() in sv_update()

Node not updating?
├── Property changed but no effect → Check update=updateNode on bpy.props
├── Animation not triggering → Set is_animation_dependent = True
├── Scene changes ignored → Set is_scene_dependent = True
└── Missing library → Set sv_dependencies = {'library_name'}
```

---

## Essential Patterns

### Pattern 1: Minimal Custom Node Template

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
from bpy.props import FloatProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvScaleVerticesNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: scale transform multiply
    Tooltip: Scales vertices by a factor

    Multiplies all input vertices by a scale factor.
    """
    bl_idname = 'SvScaleVerticesNode'
    bl_label = 'Scale Vertices'
    bl_icon = 'NONE'

    scale_factor: FloatProperty(
        name='Scale', default=1.0, update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Scale').prop_name = 'scale_factor'
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def process(self):
        if not self.outputs['Vertices'].is_linked:
            return

        verts = self.inputs['Vertices'].sv_get(default=[[]])
        scale = self.inputs['Scale'].sv_get()
        verts, scale = match_long_repeat([verts, scale])

        result = []
        for vert_list, scale_list in zip(verts, scale):
            scaled = [(v[0]*s, v[1]*s, v[2]*s)
                      for v, s in zip(vert_list, scale_list)]
            result.append(scaled)

        self.outputs['Vertices'].sv_set(result)

def register():
    bpy.utils.register_class(SvScaleVerticesNode)

def unregister():
    bpy.utils.unregister_class(SvScaleVerticesNode)
```

### Pattern 2: Node Docstring Format

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: keyword1 keyword2 keyword3
    Tooltip: Short description shown on hover

    Longer description of the node functionality.
    """
```

- **Triggers**: Space-separated keywords for the node search menu (Shift+S). ALWAYS include the most common terms users would search for.
- **Tooltip**: One-line description shown on node hover. ALWAYS keep under 80 characters.

### Pattern 3: Node Lifecycle Methods

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'

    my_prop: FloatProperty(name='Value', default=1.0, update=updateNode)

    def sv_init(self, context):
        """Called ONCE when node is created. Create sockets here."""
        self.inputs.new('SvStringsSocket', 'Input')
        self.outputs.new('SvStringsSocket', 'Output')

    def process(self):
        """Called on each evaluation. Read inputs, compute, write outputs."""
        if not self.outputs['Output'].is_linked:
            return
        data = self.inputs['Input'].sv_get(default=[[0.0]])
        self.outputs['Output'].sv_set(data)

    def sv_update(self):
        """Called on tree topology changes (links added/removed).
        Use for dynamic socket type changes. NEVER read/write socket data here."""
        pass

    def sv_copy(self, original):
        """Called when node is duplicated. Reset instance-specific state."""
        pass

    def sv_free(self):
        """Called when node is deleted. Release external resources."""
        pass

    def sv_draw_buttons(self, context, layout):
        """Draw UI elements in the node body."""
        layout.prop(self, 'my_prop')

    def sv_draw_buttons_ext(self, context, layout):
        """Draw extended UI in the sidebar properties panel (N-panel)."""
        self.sv_draw_buttons(context, layout)
```

### Pattern 4: Socket Creation Methods

#### Direct creation in sv_init

```python
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Count')
    self.inputs.new('SvVerticesSocket', 'Vertices')
    self.outputs.new('SvVerticesSocket', 'Result')
```

#### Using sv_new_input helper

```python
def sv_init(self, context):
    # Links socket to a bpy.props property for default display
    self.sv_new_input('SvStringsSocket', 'Count',
                      prop_name='count_prop', hide_safe=True)
    # Mandatory: raises SvNoDataError if no data available
    self.sv_new_input('SvVerticesSocket', 'Vertices',
                      is_mandatory=True)
```

#### Available socket types

| Socket Type | Data | Color |
|-------------|------|-------|
| `SvStringsSocket` | Numbers, strings, generic lists | Green |
| `SvVerticesSocket` | Vertex coordinates `(x,y,z)` | Yellow |
| `SvMatrixSocket` | 4x4 matrices | Lilac |
| `SvColorSocket` | RGBA colors | Dark yellow |
| `SvQuaternionSocket` | Quaternions | Purple |
| `SvObjectSocket` | Blender object references | Orange |
| `SvSurfaceSocket` | Surface objects | Blue |
| `SvCurveSocket` | Curve objects | Cyan |
| `SvSolidSocket` | Solid objects (FreeCAD) | Gray |
| `SvFilePathSocket` | File paths | Light gray |
| `SvDictionarySocket` | Python dicts | Dark green |

### Pattern 5: Standard Process Method

The standard `process()` method follows a 5-step pattern:

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
def process(self):
    # 1. Output check — skip if nothing connected
    if not any(s.is_linked for s in self.outputs):
        return

    # 2. Read inputs with safe defaults
    verts = self.inputs['Vertices'].sv_get(default=[[]])
    scale = self.inputs['Scale'].sv_get(default=[[1.0]])

    # 3. Match input list lengths
    verts, scale = match_long_repeat([verts, scale])

    # 4. Process each object
    result_verts = []
    for vert_list, scale_list in zip(verts, scale):
        vert_list, scale_list = match_long_repeat([vert_list, scale_list])
        new_verts = [(v[0]*s, v[1]*s, v[2]*s)
                     for v, s in zip(vert_list, scale_list)]
        result_verts.append(new_verts)

    # 5. Set outputs
    self.outputs['Vertices'].sv_set(result_verts)
```

### Pattern 6: BMesh Integration

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata, pydata_from_bmesh
import bmesh

def process(self):
    if not self.outputs['Vertices'].is_linked:
        return

    verts = self.inputs['Vertices'].sv_get()
    edges = self.inputs['Edges'].sv_get(default=[[]])
    faces = self.inputs['Faces'].sv_get(default=[[]])

    out_v, out_e, out_f = [], [], []
    for v, e, f in zip(verts, edges, faces):
        bm = bmesh_from_pydata(v, e, f, normal_update=True)
        # Perform BMesh operations
        bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=1)
        new_v, new_e, new_f = pydata_from_bmesh(bm)
        bm.free()  # ALWAYS free BMesh after use
        out_v.append(new_v)
        out_e.append(new_e)
        out_f.append(new_f)

    self.outputs['Vertices'].sv_set(out_v)
    self.outputs['Edges'].sv_set(out_e)
    self.outputs['Faces'].sv_set(out_f)
```

### Pattern 7: NumPy Vectorized Operations

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import numpy as np

def process(self):
    if not self.outputs['Vertices'].is_linked:
        return

    verts = self.inputs['Vertices'].sv_get()
    scale = self.inputs['Scale'].sv_get(default=[[1.0]])
    verts, scale = match_long_repeat([verts, scale])

    result = []
    for v_list, s_list in zip(verts, scale):
        v_arr = np.array(v_list)
        s_arr = np.array(s_list).reshape(-1, 1)
        result.append((v_arr * s_arr).tolist())

    self.outputs['Vertices'].sv_set(result)
```

### Pattern 8: Error Handling

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
from sverchok.core.sv_custom_exceptions import SvNoDataError

def process(self):
    try:
        data = self.inputs['Data'].sv_get()
        if not data or not data[0]:
            raise SvNoDataError(self)
        result = self.compute(data)
        self.outputs['Result'].sv_set(result)
    except SvNoDataError:
        raise  # Shows "no data" indicator (not error color)
    except Exception as e:
        self.exception(f"Processing failed: {e}")
        raise
```

### Pattern 9: Node Properties and Dependency Flags

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
from bpy.props import FloatProperty, EnumProperty

class SvMyAnimatedNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyAnimatedNode'
    bl_label = 'Animated Node'

    # Dependency flags
    is_animation_dependent = True   # Re-evaluate on frame change
    is_scene_dependent = True       # Re-evaluate on scene changes

    # External library dependencies
    sv_dependencies = {'scipy'}

    mode: EnumProperty(
        name='Mode',
        items=[('ADD', 'Add', ''), ('MUL', 'Multiply', '')],
        default='ADD',
        update=updateNode
    )

    factor: FloatProperty(
        name='Factor', default=1.0,
        min=0.0, max=10.0,
        update=updateNode
    )

    def sv_draw_buttons(self, context, layout):
        layout.prop(self, 'mode', text='')
        # factor shown via socket prop_name, not drawn here

    def process(self):
        if self.dependency_error:
            raise self.dependency_error
        import scipy
        # ... node logic using scipy
```

### Pattern 10: Registration

#### Internal node (within Sverchok repo)

```python
classes = [SvMyNodeA, SvMyNodeB]
register, unregister = bpy.utils.register_classes_factory(classes)
```

#### External add-on

```python
bl_info = {
    "name": "My Sverchok Nodes",
    "author": "Author",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "category": "Node",
}

def register():
    bpy.utils.register_class(SvMyNodeA)
    bpy.utils.register_class(SvMyNodeB)

def unregister():
    bpy.utils.unregister_class(SvMyNodeB)
    bpy.utils.unregister_class(SvMyNodeA)
```

ALWAYS unregister classes in reverse order of registration.

### Pattern 11: Logging

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    def process(self):
        self.debug("Debug message")      # sv_logger.debug
        self.info("Info message")        # sv_logger.info
        self.warning("Warning message")  # Shown in node UI via US_warning
        self.error("Error message")      # sv_logger.error
        self.exception("With traceback") # sv_logger.exception
```

ALWAYS use `self.debug/info/warning/error/exception` instead of `print()` — these integrate with Sverchok's logging system and node UI error display.

---

## Common Operations

### Node Lifecycle Call Order

| Event | Methods Called |
|-------|---------------|
| Node created | `init()` → `sv_init()` |
| Property changed | `updateNode` → `process()` |
| Link added/removed | `update()` → `sv_update()` → `process()` |
| Node duplicated | `copy()` → `sv_copy()` |
| Node deleted | `free()` → `sv_free()` |
| Tree force update | `process()` on all outdated nodes |
| Frame change | `process()` on nodes with `is_animation_dependent=True` |
| Scene change | `process()` on nodes with `is_scene_dependent=True` |

### Key API Functions

| Function | Module | Purpose |
|----------|--------|---------|
| `updateNode` | `sverchok.data_structure` | Property update callback triggering re-evaluation |
| `match_long_repeat(lists)` | `sverchok.data_structure` | Match list lengths by repeating last element |
| `fullList(lst, count)` | `sverchok.data_structure` | Extend list to length by repeating last element |
| `repeat_last_for_length(lst, count)` | `sverchok.data_structure` | Return new list of exact length |
| `bmesh_from_pydata(v, e, f)` | `sverchok.utils.sv_bmesh_utils` | Create BMesh from vertex/edge/face lists |
| `pydata_from_bmesh(bm)` | `sverchok.utils.sv_bmesh_utils` | Extract vertex/edge/face lists from BMesh |

### sv_get() and sv_set() Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `sv_get(deepcopy=True)` | `True` | Safe copy; set `False` ONLY for read-only access |
| `sv_get(default=None)` | `None` | Return default if socket has no data (avoids SvNoDataError) |
| `sv_set(data)` | — | Write nested list data to output socket cache |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for SverchCustomTreeNode, lifecycle methods, data utilities, BMesh helpers
- [references/examples.md](references/examples.md) — Full working custom node examples with BMesh, NumPy, and multi-output patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Common mistakes in custom node development with explanations

### Cross-References

- **blender-syntax-addons** — For `bl_info`, `register_class`, and Blender add-on packaging patterns
- **sverchok-core-concepts** — For node tree architecture, data flow, socket data cache, and update system

### Official Sources

- https://github.com/nortikin/sverchok
- https://sverchok.readthedocs.io/
- https://github.com/nortikin/sverchok/blob/master/node_tree.py
- https://github.com/nortikin/sverchok/blob/master/data_structure.py
- https://github.com/nortikin/sverchok/blob/master/utils/sv_bmesh_utils.py
