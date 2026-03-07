# sverchok-impl-custom-nodes — API Reference

> Sverchok v1.4.0+ / Blender 4.0+/5.x

## SverchCustomTreeNode Base Class

`SverchCustomTreeNode` is the required mixin for all custom Sverchok nodes. It combines four mixins: `UpdateNodes`, `NodeUtils`, `NodeDependencies`, `NodeDocumentation`.

```python
from sverchok.node_tree import SverchCustomTreeNode
import bpy

class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'   # Unique identifier (MUST start with 'Sv')
    bl_label = 'My Node'     # Display name in node editor
    bl_icon = 'NONE'         # Optional Blender icon
```

### Class Attributes

| Attribute | Type | Default | Purpose |
|-----------|------|---------|---------|
| `bl_idname` | `str` | Required | Unique node type identifier |
| `bl_label` | `str` | Required | Display name |
| `bl_icon` | `str` | `'NONE'` | Blender icon identifier |
| `sv_category` | `str` | `''` | Category in Shift+S node search menu |
| `is_animation_dependent` | `bool` | `False` | Re-evaluate on frame change |
| `is_scene_dependent` | `bool` | `False` | Re-evaluate on scene change |
| `sv_dependencies` | `set[str]` | `set()` | Required Python modules |

---

## Lifecycle Methods (UpdateNodes mixin)

### sv_init(self, context)

Called once when the node is first created. ALWAYS create sockets here.

```python
def sv_init(self, context):
    self.inputs.new('SvVerticesSocket', 'Vertices')
    self.outputs.new('SvVerticesSocket', 'Result')
```

**Parameters:**
- `context` — Blender context (rarely used)

### process(self)

Called on each node evaluation. Implements the node's computation logic.

```python
def process(self):
    # Called by update system — NEVER call directly
    pass
```

### sv_update(self)

Called when tree topology changes (links/nodes added or removed). Used for dynamic socket management.

```python
def sv_update(self):
    # NEVER read/write socket data here — only manage socket types/visibility
    pass
```

### sv_copy(self, original)

Called when node is duplicated (Ctrl+D or Shift+D).

```python
def sv_copy(self, original):
    # Reset instance-specific state (e.g., cached data)
    pass
```

**Parameters:**
- `original` — The source node being copied from

### sv_free(self)

Called when node is deleted. Release external resources.

```python
def sv_free(self):
    # Clean up file handles, GPU resources, etc.
    pass
```

### sv_draw_buttons(self, context, layout)

Draw custom UI elements in the node body.

```python
def sv_draw_buttons(self, context, layout):
    layout.prop(self, 'my_property')
    layout.prop(self, 'mode', text='')
```

**Parameters:**
- `context` — Blender context
- `layout` — `bpy.types.UILayout` for drawing UI elements

### sv_draw_buttons_ext(self, context, layout)

Draw extended UI in the sidebar properties panel (N-panel). Called when user selects the node and views its properties.

```python
def sv_draw_buttons_ext(self, context, layout):
    self.sv_draw_buttons(context, layout)
    layout.prop(self, 'advanced_property')
```

### sv_new_input(self, socket_type, name, **attrib_dict)

Helper method to create and configure an input socket with additional attributes.

```python
self.sv_new_input('SvStringsSocket', 'Count',
                  prop_name='count_prop',    # Link to bpy.props for default display
                  hide_safe=True,            # Hide socket when not connected
                  is_mandatory=True)         # Raise SvNoDataError if no data
```

**Parameters:**
- `socket_type` (`str`) — Socket class name (e.g., `'SvStringsSocket'`)
- `name` (`str`) — Socket display name
- `**attrib_dict` — Additional socket attributes:
  - `prop_name` (`str`) — Name of a bpy.props property to display as default
  - `hide_safe` (`bool`) — Hide socket when not connected
  - `is_mandatory` (`bool`) — Error if socket has no data

---

## NodeUtils Mixin

### Logging Methods

| Method | Level | UI Display |
|--------|-------|------------|
| `self.debug(msg)` | DEBUG | No |
| `self.info(msg)` | INFO | No |
| `self.warning(msg)` | WARNING | Yes (US_warning) |
| `self.error(msg)` | ERROR | No |
| `self.exception(msg)` | ERROR + traceback | No |

### Utility Methods

```python
# Track UI operator origin
self.wrapper_tracked_ui_draw_op(layout, 'node.my_operator', text="Run")

# Get Blender data by name
obj = self.get_bpy_data_from_name('Cube', 'objects')

# Safe socket removal
self.safe_socket_remove('inputs', 'OptionalSocket',
                        failure_message="Socket not found")
```

---

## NodeDependencies Mixin

```python
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    sv_dependencies = {'scipy', 'skimage'}

    def process(self):
        if self.dependency_error:
            raise self.dependency_error
        import scipy
```

| Property | Type | Purpose |
|----------|------|---------|
| `sv_dependencies` | `set[str]` | Required module names |
| `dependency_error` | `Optional[DependencyError]` | Error if modules missing |
| `missing_dependency` | `bool` (classproperty) | True if any dependency absent |

---

## Data Structure Utilities

### updateNode(self, context)

```python
from sverchok.data_structure import updateNode
```

The ONLY correct callback for `bpy.props` `update=` parameter. Triggers a PropertyEvent that marks the node as outdated and initiates tree re-evaluation.

### match_long_repeat(lsts)

```python
from sverchok.data_structure import match_long_repeat

verts = [[(0,0,0), (1,0,0)], [(2,0,0)]]
scale = [[2.0]]
matched_verts, matched_scale = match_long_repeat([verts, scale])
# matched_scale becomes [[2.0], [2.0]] — last element repeated
```

**Parameters:**
- `lsts` (`list[list]`) — List of lists to match

**Returns:** Tuple of matched lists, shorter lists extended by repeating their last element.

### fullList(lst, count)

```python
from sverchok.data_structure import fullList

data = [1, 2, 3]
fullList(data, 5)
# data is now [1, 2, 3, 3, 3] — mutates in place
```

**Parameters:**
- `lst` (`list`) — List to extend (mutated in place)
- `count` (`int`) — Minimum target length

### repeat_last_for_length(lst, count, deepcopy=False)

```python
from sverchok.data_structure import repeat_last_for_length

result = repeat_last_for_length([1, 2, 3], 5)
# result = [1, 2, 3, 3, 3] — returns new list
```

**Parameters:**
- `lst` (`list`) — Source list
- `count` (`int`) — Target length
- `deepcopy` (`bool`) — Deep copy repeated elements (default: `False`)

**Returns:** New list of exactly `count` length.

### cycle_for_length(lst, count)

```python
from sverchok.data_structure import cycle_for_length

result = cycle_for_length([1, 2, 3], 7)
# result = [1, 2, 3, 1, 2, 3, 1]
```

### multi_socket(node, min=1, start=0, breck=False, out_count=None)

Manages dynamic socket counts based on connection state.

```python
from sverchok.data_structure import multi_socket

def sv_update(self):
    multi_socket(self, min=1, start=0)
```

---

## BMesh Utilities

### bmesh_from_pydata(verts, edges, faces, ...)

```python
from sverchok.utils.sv_bmesh_utils import bmesh_from_pydata

bm = bmesh_from_pydata(
    verts=[(0,0,0), (1,0,0), (1,1,0), (0,1,0)],
    edges=[],
    faces=[(0,1,2,3)],
    normal_update=True,    # Update normals after construction
    index_edges=False      # Index edges for manual iteration
)
```

**Parameters:**
- `verts` (`list[tuple]`) — Vertex coordinates (required)
- `edges` (`list[tuple]`) — Edge vertex index pairs (default: `[]`)
- `faces` (`list[tuple]`) — Face vertex index tuples (default: `[]`)
- `markup_face_data` (`bool`) — Mark face data layers (default: `False`)
- `markup_edge_data` (`bool`) — Mark edge data layers (default: `False`)
- `markup_vert_data` (`bool`) — Mark vertex data layers (default: `False`)
- `normal_update` (`bool`) — Update normals after construction (default: `False`)
- `index_edges` (`bool`) — Enable edge index lookup (default: `False`)

**Returns:** `bmesh.types.BMesh`

### pydata_from_bmesh(bm, ...)

```python
from sverchok.utils.sv_bmesh_utils import pydata_from_bmesh

new_verts, new_edges, new_faces = pydata_from_bmesh(bm)
```

**Parameters:**
- `bm` (`bmesh.types.BMesh`) — Source BMesh
- `face_data` (`optional`) — Face data to propagate (default: `None`)
- `ret_verts` (`bool`) — Include vertices (default: `True`)
- `ret_edges` (`bool`) — Include edges (default: `True`)
- `ret_faces` (`bool`) — Include faces (default: `True`)

**Returns:** `(verts, edges, faces)` or `(verts, edges, faces, face_data_out)` if `face_data` provided.

---

## Socket Data Methods

### sv_get(deepcopy=True, default=None)

Read data from an input socket.

```python
# Safe read (default)
data = self.inputs['Name'].sv_get(deepcopy=True)

# Performance read (ONLY when data will NOT be mutated)
data = self.inputs['Name'].sv_get(deepcopy=False)

# With fallback default (no error if unconnected)
data = self.inputs['Name'].sv_get(default=[[]])
```

### sv_set(data)

Write data to an output socket.

```python
# Single object output
self.outputs['Name'].sv_set([[(0,0,0), (1,0,0)]])

# Multi-object output
self.outputs['Name'].sv_set([[(0,0,0)], [(1,1,1)]])
```

---

## Error Handling

### SvNoDataError

```python
from sverchok.core.sv_custom_exceptions import SvNoDataError

# Raise to indicate missing data (shows "no data" indicator, not error)
raise SvNoDataError(self)
```

ALWAYS re-raise `SvNoDataError` — catching it silently hides the "no data" state from the user.
