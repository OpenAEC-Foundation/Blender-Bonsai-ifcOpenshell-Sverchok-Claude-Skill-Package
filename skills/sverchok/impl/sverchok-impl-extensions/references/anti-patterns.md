# sverchok-impl-extensions: Anti-Patterns

These are confirmed error patterns when developing or using Sverchok extensions. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://github.com/portnov/sverchok-extra
- https://github.com/vicdoval/sverchok-open3d
- vooronderzoek-sverchok.md section 11

---

## AP-001: Missing bl_info or Wrong Category

**WHY this is wrong**: Sverchok discovers extensions through Blender's add-on system. The `bl_info` dictionary is required for Blender to recognize the add-on. If `"category"` is not `"Node"`, Sverchok will not include the extension in its node menu discovery. The extension may load as a Blender add-on but its nodes will not appear in the Shift+A menu.

```python
# WRONG — missing bl_info entirely
from sverchok.ui.nodeview_space_menu import add_node_menu

def nodes_index():
    return [("My Nodes", [("my.node", "SvMyNode")])]

def register():
    # Blender cannot discover this add-on without bl_info
    pass
```

```python
# WRONG — wrong category
bl_info = {
    "name": "My Extension",
    "author": "Author",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "category": "3D View",  # WRONG — Sverchok expects "Node"
}
```

```python
# CORRECT — proper bl_info with "Node" category
bl_info = {
    "name": "My Sverchok Extension",
    "author": "Author",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "category": "Node",  # Required for Sverchok integration
}
```

ALWAYS include a complete `bl_info` dictionary. ALWAYS set `"category": "Node"`.

---

## AP-002: Missing nodes_index Function

**WHY this is wrong**: Sverchok extensions MUST provide a `nodes_index()` function that returns the list of node categories and their class mappings. Without it, Sverchok cannot discover the extension's nodes. The `add_node_menu` system calls `nodes_index()` during registration to populate the Shift+A menu. If the function is missing or returns an empty list, no nodes appear.

```python
# WRONG — no nodes_index, nodes are registered but not discoverable
bl_info = {"name": "My Extension", "category": "Node", ...}

def register():
    import bpy
    bpy.utils.register_class(SvMyNode)
    # Node is registered with Blender but NOT in Sverchok's menu
```

```python
# CORRECT — nodes_index declares the menu structure
def nodes_index():
    return [
        ("My Category", [
            ("my_folder.my_node", "SvMyNode"),
        ]),
    ]

def register():
    for module in make_node_list():
        module.register()
    add_node_menu.register()  # Reads nodes_index() to build menu
```

ALWAYS define `nodes_index()` in your extension's `__init__.py`. ALWAYS call `add_node_menu.register()` after registering node modules.

---

## AP-003: Using SDF Nodes Without python-sdf Library

**WHY this is wrong**: The SDF Primitives and SDF Operations node categories in Sverchok-Extra depend entirely on the `python-sdf` library. If this library is not installed, these node categories will not appear in the menu. Attempting to create these nodes programmatically (e.g., `tree.nodes.new('SvSdfBoxNode')`) will raise a RuntimeError because the node class was never registered.

```python
# WRONG — assuming SDF nodes exist without checking
tree = bpy.data.node_groups.new("SDF_Test", 'SverchCustomTreeType')
with tree.init_tree():
    box = tree.nodes.new('SvSdfBoxNode')  # RuntimeError if python-sdf not installed
```

```python
# CORRECT — check dependency before using SDF nodes
try:
    import sdf
    sdf_available = True
except ImportError:
    sdf_available = False

if sdf_available:
    tree = bpy.data.node_groups.new("SDF_Test", 'SverchCustomTreeType')
    with tree.init_tree():
        box = tree.nodes.new('SvSdfBoxNode')
        generate = tree.nodes.new('SvSdfGenerateNode')
        tree.links.new(box.outputs[0], generate.inputs[0])
    tree.force_update()
else:
    print("SDF nodes require: pip install sdf")
```

ALWAYS check for the python-sdf library before using SDF nodes. ALWAYS provide clear error messages about missing dependencies.

---

## AP-004: Forgetting to Install Sverchok Core First

**WHY this is wrong**: ALL Sverchok extensions import from `sverchok.node_tree`, `sverchok.data_structure`, and `sverchok.ui.nodeview_space_menu`. If Sverchok core is not installed and enabled, these imports fail with ModuleNotFoundError during registration. The extension will fail to load entirely.

```python
# WRONG — no check for Sverchok availability
from sverchok.node_tree import SverchCustomTreeNode  # ModuleNotFoundError
from sverchok.data_structure import updateNode        # if Sverchok not installed
```

```python
# CORRECT — graceful handling of missing Sverchok
try:
    from sverchok.node_tree import SverchCustomTreeNode
    from sverchok.data_structure import updateNode
    sverchok_available = True
except ImportError:
    sverchok_available = False
    print("ERROR: Sverchok core must be installed and enabled first")

# In register():
def register():
    if not sverchok_available:
        return  # Silently skip if Sverchok is not available
    # ... normal registration
```

ALWAYS install and enable Sverchok core before installing any extension. Extensions should handle missing Sverchok gracefully.

---

## AP-005: Wrong Unregister Order

**WHY this is wrong**: Registration and unregistration MUST be symmetric. If nodes are registered in order A, B, C, they must be unregistered in reverse order C, B, A. Failing to reverse the order can cause Blender errors when classes have dependencies on each other (e.g., a node using a custom socket type that gets unregistered first).

```python
# WRONG — unregister in same order as register
def register():
    sockets.register()
    icons.register()
    register_nodes()

def unregister():
    sockets.unregister()  # WRONG — nodes still reference these sockets
    icons.unregister()
    unregister_nodes()    # Nodes try to use already-unregistered sockets
```

```python
# CORRECT — reverse order
def register():
    sockets.register()
    icons.register()
    register_nodes()

def unregister():
    unregister_nodes()    # Nodes removed first
    icons.unregister()    # Then icons
    sockets.unregister()  # Then sockets (no longer referenced)
```

ALWAYS unregister in the reverse order of registration. ALWAYS use `reversed()` when iterating node modules for unregistration.

---

## AP-006: Hardcoding bl_idname Without Sv Prefix

**WHY this is wrong**: Sverchok uses the `Sv` prefix convention for all node `bl_idname` values to avoid collisions with other Blender add-ons. Without this prefix, a node could collide with a Geometry Nodes, Shader Nodes, or another add-on's node identifier, causing unpredictable behavior or registration failures.

```python
# WRONG — generic bl_idname without Sv prefix
class MyScaleNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'ScaleNode'  # Could collide with other add-ons
    bl_label = 'Scale'
```

```python
# CORRECT — Sv prefix for namespace safety
class SvMyScaleNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyScaleNode'  # Unique within Sverchok ecosystem
    bl_label = 'My Scale'
```

ALWAYS prefix `bl_idname` with `Sv` (or `SvEx` for Sverchok-Extra, `SvO3` for Open3d). NEVER use generic names that could collide.

---

## AP-007: Not Handling Module Reload During Development

**WHY this is wrong**: During development, disabling and re-enabling an add-on in Blender reimports the `__init__.py` module. If submodules are not explicitly reloaded, Python uses cached versions from the first import. Changes to node files will not take effect without restarting Blender, making the development cycle extremely slow.

```python
# WRONG — no reload handling
from . import nodes_index
from . import settings

def register():
    # After disable/enable, nodes_index and settings are stale cached versions
    pass
```

```python
# CORRECT — detect reimport and reload submodules
if "bpy" in locals():
    # Already imported once — reload all submodules
    import importlib
    importlib.reload(nodes_index)
    importlib.reload(settings)
    # Reload all node modules too
    for mod in _imported_modules:
        importlib.reload(mod)
else:
    from . import nodes_index
    from . import settings

import bpy
```

ALWAYS implement reload detection using `if "bpy" in locals()` at the top of `__init__.py`. ALWAYS reload all submodules during reimport.

---

## AP-008: Using Open3D Nodes Without Saving First

**WHY this is wrong**: The Sverchok-Open3d README explicitly warns: "Sverchok-Open3d contains nodes that may crash Blender." Some Open3D operations (particularly mesh reconstruction and boolean operations) can trigger segmentation faults in the underlying C++ library. Without saving, all work is lost.

```python
# WRONG — running Open3D operations without saving
tree = bpy.data.node_groups.new("PointCloud", 'SverchCustomTreeType')
with tree.init_tree():
    importer = tree.nodes.new('SvO3ImportNode')
    poisson = tree.nodes.new('SvO3PoissonNode')
    tree.links.new(importer.outputs[0], poisson.inputs[0])
tree.force_update()  # May crash Blender
```

```python
# CORRECT — save before running Open3D operations
bpy.ops.wm.save_as_mainfile(filepath=bpy.data.filepath or "/tmp/backup.blend")

tree = bpy.data.node_groups.new("PointCloud", 'SverchCustomTreeType')
with tree.init_tree():
    importer = tree.nodes.new('SvO3ImportNode')
    poisson = tree.nodes.new('SvO3PoissonNode')
    tree.links.new(importer.outputs[0], poisson.inputs[0])
tree.force_update()
```

ALWAYS save your Blender file before running Open3D reconstruction nodes. ALWAYS assume Open3D operations may crash Blender.

---

## AP-009: Extension Node Missing One of 7 Mandatory Elements

**WHY this is wrong**: Sverchok nodes MUST implement all 7 mandatory elements. Missing any one causes specific failures:

| Missing Element | Failure Mode |
|----------------|--------------|
| SverchCustomTreeNode inheritance | Node has no Sverchok integration (no sv_init, no process) |
| bpy.types.Node inheritance | Node cannot be registered with Blender |
| bl_idname | Registration fails: no identifier |
| bl_label | Node appears with empty name in menu |
| sv_init() | No sockets created, node is non-functional |
| process() | Node never computes results |
| updateNode callback | Property changes do not trigger recomputation |

```python
# WRONG — missing updateNode on property
class SvBrokenExtNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvBrokenExtNode'
    bl_label = 'Broken'

    size: bpy.props.FloatProperty(name="Size", default=1.0)
    # Missing: update=updateNode
    # Result: changing "Size" in UI does nothing

    def sv_init(self, context):
        self.outputs.new('SvStringsSocket', 'Result')

    def process(self):
        self.outputs['Result'].sv_set([[self.size]])
```

```python
# CORRECT — all 7 elements present
class SvWorkingExtNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvWorkingExtNode'
    bl_label = 'Working'

    size: bpy.props.FloatProperty(
        name="Size", default=1.0,
        update=updateNode  # Element 7: triggers recomputation
    )

    def sv_init(self, context):  # Element 5
        self.outputs.new('SvStringsSocket', 'Result')

    def process(self):  # Element 6
        self.outputs['Result'].sv_set([[self.size]])
```

ALWAYS verify all 7 mandatory elements are present in every extension node. ALWAYS use `update=updateNode` on every `bpy.props` property.

---

## AP-010: Assuming Extension Nodes Are Available in Core Sverchok

**WHY this is wrong**: Extension nodes are only available when the corresponding extension add-on is installed and enabled. Scripting code that references extension node bl_idnames (like `SvSdfBoxNode`, `SvO3ImportNode`, `SvExSmoothSplineNode`) without checking will raise RuntimeError when the extension is not present. This is a common mistake when sharing scripts between users with different extension setups.

```python
# WRONG — assumes Sverchok-Extra is installed
tree = bpy.data.node_groups.new("Test", 'SverchCustomTreeType')
with tree.init_tree():
    node = tree.nodes.new('SvExSmoothSplineNode')
    # RuntimeError: "SvExSmoothSplineNode" not found if extension missing
```

```python
# CORRECT — check extension availability
import addon_utils

def is_extension_enabled(module_name):
    loaded, enabled = addon_utils.check(module_name)
    return loaded and enabled

if is_extension_enabled('sverchok_extra'):
    tree = bpy.data.node_groups.new("Test", 'SverchCustomTreeType')
    with tree.init_tree():
        node = tree.nodes.new('SvExSmoothSplineNode')
    tree.force_update()
else:
    print("Sverchok-Extra is required but not installed")
```

ALWAYS check if an extension is enabled before using its nodes programmatically. NEVER assume extension nodes are part of core Sverchok.
