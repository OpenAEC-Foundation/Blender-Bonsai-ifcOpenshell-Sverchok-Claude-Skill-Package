# sverchok-impl-extensions: API Method Reference

Sources: https://github.com/portnov/sverchok-extra, https://github.com/vicdoval/sverchok-open3d,
vooronderzoek-sverchok.md section 11

---

## Extension Registration API

### bl_info Dictionary

```python
# Required fields for any Sverchok extension __init__.py
bl_info = {
    "name": "My Sverchok Extension",   # Display name in Blender preferences
    "author": "Author Name",            # Author identifier
    "version": (0, 1, 0),               # Semantic version tuple (major, minor, patch)
    "blender": (4, 0, 0),               # Minimum Blender version required
    "category": "Node",                 # MUST be "Node" for Sverchok extensions
}

# Optional fields:
bl_info["description"] = "Short description of the extension"
bl_info["doc_url"] = "https://link-to-documentation"
bl_info["tracker_url"] = "https://link-to-issue-tracker"
```

### nodes_index() Function

```python
def nodes_index():
    """Declare all node categories and their node class mappings.

    Returns:
        list — A list of category definitions. Each definition is either:
        - A tuple: (category_name, node_list)
        - A dict: {category_name: node_list}

    node_list entries:
        - (module_path, class_name) — registers a node
        - ({'icon_name': 'ICON_ID'}, ) — sets category icon
        - None — inserts a menu separator

    module_path is relative to the extension's nodes/ directory.
    Example: "surface.smooth_spline" resolves to nodes/surface/smooth_spline.py
    """
    return [
        ("Category Name", [
            ("subfolder.module_name", "SvNodeClassName"),
        ]),
    ]
```

### register() Function

```python
def register():
    """Called by Blender when the extension is enabled.

    Standard sequence:
    1. Register the node menu via add_node_menu.register()
    2. Register settings (addon preferences)
    3. Register custom icons
    4. Register custom socket types
    5. Register all node modules (iterate make_node_list)
    6. Display welcome/info message (optional)
    """
    from sverchok.ui.nodeview_space_menu import add_node_menu
    add_node_menu.register()
    # ... register settings, icons, sockets
    for module in make_node_list():
        module.register()
```

### unregister() Function

```python
def unregister():
    """Called by Blender when the extension is disabled.

    MUST reverse the registration order:
    1. Remove node categories from Blender's registry
    2. Unregister menu classes
    3. Unregister all node modules (in REVERSE order)
    4. Unregister custom sockets
    5. Unregister custom icons
    6. Unregister settings
    """
    for module in reversed(make_node_list()):
        module.unregister()
```

---

## Node Registration — 7 Mandatory Elements

Every node in a Sverchok extension MUST implement these elements:

```python
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    """Docstring becomes the node tooltip."""

    # --- 7 MANDATORY ELEMENTS ---

    # 1. Inherit from SverchCustomTreeNode (provides Sverchok integration)
    #    Already done in class definition above

    # 2. Inherit from bpy.types.Node (provides Blender node system)
    #    Already done in class definition above

    # 3. bl_idname — unique string identifier
    #    Convention: 'Sv' prefix + descriptive name
    bl_idname = 'SvMyNode'

    # 4. bl_label — display name shown in UI
    bl_label = 'My Node'

    # 5. sv_init() — called once when node is created
    #    Define input/output sockets here
    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Input')
        self.outputs.new('SvStringsSocket', 'Output')

    # 6. process() — main computation method
    #    Called by update system, NEVER directly
    def process(self):
        if not self.outputs['Output'].is_linked:
            return
        data = self.inputs['Input'].sv_get(default=[[]])
        self.outputs['Output'].sv_set(data)

    # 7. updateNode callback — on ALL bpy.props properties
    my_prop: bpy.props.FloatProperty(
        name="Value",
        default=1.0,
        update=updateNode  # Triggers PropertyEvent -> process()
    )

# Per-module register/unregister
def register():
    bpy.utils.register_class(SvMyNode)

def unregister():
    bpy.utils.unregister_class(SvMyNode)
```

---

## Sverchok-Extra — bl_info and Registration

### bl_info (from __init__.py)

```python
bl_info = {
    "name": "Sverchok-Extra",
    "author": "Ilya Portnov",
    "version": (0, 1, 0, 0),
    "blender": (2, 81, 0),  # Minimum, works with 4.0+/5.x
    "category": "Node",
}
```

### Registration Flow

```python
# Sverchok-Extra register() sequence:
def register():
    # 1. Activate node menu system
    add_node_menu.register()
    # 2. Register settings and icon resources
    settings.register()
    icons.register()
    # 3. Dynamically load all node modules via nodes_index()
    register_nodes()  # Uses convert_config() to recursively import
    # 4. Display welcome information
    show_welcome()

def unregister():
    # Reverse order
    # 1. Remove node categories from Blender registry
    # 2. Unregister menu classes
    # 3. Unregister all node modules
    unregister_nodes()
    # 4. Clean up sockets and icons
    icons.unregister()
    settings.unregister()
```

### convert_config() — Dynamic Node Loading

```python
def convert_config(config):
    """Recursively traverse nodes_index() structure.
    Dynamically imports node modules from sverchok_extra.nodes.
    Resolves module paths like 'surface.smooth_spline'
    to sverchok_extra.nodes.surface.smooth_spline.
    Returns list of imported modules with register/unregister functions."""
```

---

## Sverchok-Open3d — bl_info and Registration

### bl_info (from __init__.py)

```python
bl_info = {
    "name": "Sverchok-Open3d",
    "author": "Victor Doval",
    "version": (0, 1, 0, 0),
    "blender": (2, 81, 0),  # Minimum, works with 4.0+/5.x
    "category": "Node",
}
```

### Registration Flow

```python
def register():
    add_node_menu.register()
    settings.register()
    icons.register()
    sockets.register()  # Custom Open3D socket types
    register_nodes()
    show_welcome()

def unregister():
    # Note: does NOT unregister the node menu itself (known behavior)
    unregister_nodes()
    sockets.unregister()
    icons.unregister()
    settings.unregister()
```

### Custom Socket Types

```python
# Sverchok-Open3d defines custom sockets in sockets.py:
# - SvO3PointCloudSocket — carries Open3D PointCloud objects
# - SvO3TriangleMeshSocket — carries Open3D TriangleMesh objects
# These maintain type safety between Open3D nodes
```

### nodes_index Format (Open3d)

```python
def nodes_index():
    return [
        {"Open 3D": [
            ({'icon_name': 'SV_O3_OPEN3D_LOGO'}, ),
            ("utils.o3_import", "SvO3ImportNode"),
            ("utils.o3_export", "SvO3ExportNode"),
            ("utils.o3_transform", "SvO3TransformNode"),
            None,  # Separator
            {"Point Cloud": [
                ("point_cloud.o3_downsample", "SvO3DownsampleNode"),
                ("point_cloud.o3_mask", "SvO3MaskNode"),
                ("point_cloud.o3_normals", "SvO3NormalsNode"),
                ("point_cloud.o3_join", "SvO3JoinNode"),
                # ... more point cloud nodes
            ]},
            {"Triangle Mesh": [
                ("triangle_mesh.o3_alpha_shape", "SvO3AlphaShapeNode"),
                ("triangle_mesh.o3_ball_pivoting", "SvO3BallPivotingNode"),
                ("triangle_mesh.o3_poisson", "SvO3PoissonNode"),
                ("triangle_mesh.o3_simplify", "SvO3SimplifyNode"),
                ("triangle_mesh.o3_smooth", "SvO3SmoothNode"),
                # ... more mesh nodes
            ]},
        ]},
    ]
```

---

## Dependency Management Pattern

### dependencies.py (Sverchok-Extra Pattern)

```python
# Pattern for optional dependency detection
# Used by Sverchok-Extra to gracefully handle missing libraries

import importlib

def check_dependency(module_name):
    """Check if a Python module is available.
    Returns True if importable, False otherwise."""
    try:
        importlib.import_module(module_name)
        return True
    except ImportError:
        return False

# Dependency flags
scipy_available = check_dependency('scipy')
sdf_available = check_dependency('sdf')
shapely_available = check_dependency('shapely')

# Nodes use these flags to conditionally register:
# if scipy_available:
#     class SvSmoothSplineNode(...): ...
```

### Addon Preferences for Dependency Installation

```python
# Pattern used by Sverchok-Open3d for one-click pip install
class SvOpen3dPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__

    def draw(self, context):
        layout = self.layout
        try:
            import open3d
            layout.label(text=f"Open3D {open3d.__version__} installed")
        except ImportError:
            layout.operator("sverchok_open3d.install_open3d",
                          text="Install Open3D")
```

---

## Sverchok Core Imports Used by Extensions

```python
# Standard imports that ALL Sverchok extensions use:

# Node base class
from sverchok.node_tree import SverchCustomTreeNode

# Update callback for properties
from sverchok.data_structure import updateNode

# List matching utility
from sverchok.data_structure import match_long_repeat

# Menu registration
from sverchok.ui.nodeview_space_menu import add_node_menu

# Socket types (for sv_init)
# Used as strings: 'SvStringsSocket', 'SvVerticesSocket',
#   'SvMatrixSocket', 'SvColorSocket', 'SvObjectSocket',
#   'SvSurfaceSocket', 'SvCurveSocket', 'SvSolidSocket'
```

---

## Module Reload Handling

```python
# Both Sverchok-Extra and Sverchok-Open3d implement reload detection:
# When Blender reimports the addon (during development), all previously
# imported modules plus utility submodules are reloaded to maintain
# consistency.

if "bpy" in locals():
    # Already imported — reload all submodules
    import importlib
    for mod in _imported_modules:
        importlib.reload(mod)
else:
    # First import — normal initialization
    from . import nodes_index
    from . import settings
    # ...
```
