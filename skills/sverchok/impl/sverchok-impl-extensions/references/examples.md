# sverchok-impl-extensions: Working Code Examples

All examples based on Sverchok v1.4.0+ and extension source code analysis.
Sources: https://github.com/portnov/sverchok-extra, https://github.com/vicdoval/sverchok-open3d,
vooronderzoek-sverchok.md section 11

---

## Example 1: Create a Minimal Sverchok Extension

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# File: my_sverchok_ext/__init__.py

bl_info = {
    "name": "My Sverchok Extension",
    "author": "Your Name",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "category": "Node",
    "description": "Example Sverchok extension with one node",
}

import importlib
import sys

# Module reload handling for development
if "bpy" in locals():
    importlib.reload(sys.modules.get(f"{__package__}.nodes.my_category.my_node"))

from sverchok.ui.nodeview_space_menu import add_node_menu

_node_modules = []

def nodes_index():
    return [
        ("My Extension", [
            ("my_category.my_node", "SvMyExampleNode"),
        ]),
    ]

def make_node_list():
    global _node_modules
    _node_modules = []
    for category_name, nodes in nodes_index():
        for module_path, class_name in nodes:
            mod = importlib.import_module(f".nodes.{module_path}", __package__)
            _node_modules.append(mod)
    return _node_modules

def register():
    for module in make_node_list():
        module.register()
    add_node_menu.register()

def unregister():
    for module in reversed(_node_modules):
        module.unregister()
```

```python
# File: my_sverchok_ext/nodes/__init__.py
# (empty file, required for Python package)
```

```python
# File: my_sverchok_ext/nodes/my_category/__init__.py
# (empty file, required for Python package)
```

```python
# File: my_sverchok_ext/nodes/my_category/my_node.py
# Blender 4.0+/5.x with Sverchok v1.4.0+

import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvMyExampleNode(SverchCustomTreeNode, bpy.types.Node):
    """Multiply input values by a factor."""
    bl_idname = 'SvMyExampleNode'
    bl_label = 'My Example'

    factor: bpy.props.FloatProperty(
        name="Factor", default=2.0,
        update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Values')
        self.inputs.new('SvStringsSocket', 'Factor').prop_name = 'factor'
        self.outputs.new('SvStringsSocket', 'Result')

    def process(self):
        if not self.outputs['Result'].is_linked:
            return

        values = self.inputs['Values'].sv_get(default=[[1.0]])
        factors = self.inputs['Factor'].sv_get(default=[[self.factor]])

        matched = match_long_repeat([values, factors])
        result = []
        for vals, facs in zip(*matched):
            inner_matched = match_long_repeat([vals, facs])
            result.append([v * f for v, f in zip(*inner_matched)])

        self.outputs['Result'].sv_set(result)

def register():
    bpy.utils.register_class(SvMyExampleNode)

def unregister():
    bpy.utils.unregister_class(SvMyExampleNode)
```

---

## Example 2: SDF Modeling Workflow (Sverchok-Extra)

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and Sverchok-Extra
# Programmatic SDF workflow: create primitive, apply operations, generate mesh

import bpy

tree = bpy.data.node_groups.new("SDF_Workflow", 'SverchCustomTreeType')

with tree.init_tree():
    # SDF Primitive: Box
    box = tree.nodes.new('SvSdfBoxNode')
    box.location = (0, 0)

    # SDF Primitive: Sphere
    sphere = tree.nodes.new('SvSdfSphereNode')
    sphere.location = (0, -200)

    # SDF Boolean: Intersection
    boolean = tree.nodes.new('SvSdfBooleanNode')
    boolean.location = (250, -100)
    # Connect box and sphere to boolean inputs
    tree.links.new(box.outputs[0], boolean.inputs[0])
    tree.links.new(sphere.outputs[0], boolean.inputs[1])

    # SDF Twist: Apply twist deformation
    twist = tree.nodes.new('SvSdfTwistNode')
    twist.location = (500, -100)
    tree.links.new(boolean.outputs[0], twist.inputs[0])

    # SDF Generate: Convert to mesh via marching cubes
    generate = tree.nodes.new('SvSdfGenerateNode')
    generate.location = (750, -100)
    tree.links.new(twist.outputs[0], generate.inputs[0])

    # Viewer Draw: Display result
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (1000, -100)
    tree.links.new(generate.outputs['Vertices'], viewer.inputs['vertices'])
    tree.links.new(generate.outputs['Faces'], viewer.inputs['faces'])

tree.force_update()
```

---

## Example 3: Dependency-Guarded Extension Node

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Pattern: Node that requires an optional library (e.g., SciPy)

import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

# Guard the import
try:
    from scipy.spatial import Delaunay
    scipy_available = True
except ImportError:
    scipy_available = False

if scipy_available:
    class SvExDelaunayMeshNode(SverchCustomTreeNode, bpy.types.Node):
        """Compute Delaunay triangulation of 2D/3D point sets.
        Requires SciPy."""
        bl_idname = 'SvExDelaunayMeshNode'
        bl_label = 'Delaunay Mesh (Extra)'

        def sv_init(self, context):
            self.inputs.new('SvVerticesSocket', 'Vertices')
            self.outputs.new('SvVerticesSocket', 'Vertices')
            self.outputs.new('SvStringsSocket', 'Faces')

        def process(self):
            if not any(s.is_linked for s in self.outputs):
                return

            verts_in = self.inputs['Vertices'].sv_get(default=[[]])
            verts_out = []
            faces_out = []

            for verts in verts_in:
                if len(verts) < 4:
                    verts_out.append(verts)
                    faces_out.append([])
                    continue
                points = [(v[0], v[1], v[2]) for v in verts]
                tri = Delaunay(points)
                verts_out.append(points)
                faces_out.append(tri.simplices.tolist())

            self.outputs['Vertices'].sv_set(verts_out)
            self.outputs['Faces'].sv_set(faces_out)

    def register():
        bpy.utils.register_class(SvExDelaunayMeshNode)

    def unregister():
        bpy.utils.unregister_class(SvExDelaunayMeshNode)

else:
    # Fallback: no-op registration when dependency is missing
    def register():
        pass

    def unregister():
        pass
```

---

## Example 4: Extension with Custom Socket Type (Open3d Pattern)

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Pattern from Sverchok-Open3d: custom socket for library-specific data types

import bpy
from sverchok.core.sockets import SvSocketCommon

# Custom socket for carrying Open3D PointCloud objects
class SvO3PointCloudSocket(bpy.types.NodeSocket, SvSocketCommon):
    bl_idname = 'SvO3PointCloudSocket'
    bl_label = 'Open3D Point Cloud'

    # Custom color for the socket (RGBA)
    color = (0.2, 0.6, 1.0, 1.0)

    def draw(self, context, layout, node, text):
        layout.label(text=text)

    def draw_color(self, context, node):
        return self.color

def register():
    bpy.utils.register_class(SvO3PointCloudSocket)

def unregister():
    bpy.utils.unregister_class(SvO3PointCloudSocket)
```

---

## Example 5: nodes_index with Icons and Nested Submenus

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Real-world nodes_index structure with icons, separators, and nesting

def nodes_index():
    return [
        # Top-level category with icon
        {"My Geometry": [
            ({'icon_name': 'SV_MY_ICON'}, ),  # Category icon (optional)
            ("generators.basic_shape", "SvMyBasicShapeNode"),
            ("generators.advanced_shape", "SvMyAdvancedShapeNode"),
            None,  # Menu separator (renders as horizontal line)
            ("analyzers.measure", "SvMyMeasureNode"),
        ]},

        # Nested submenu within a category
        {"My Tools": [
            ({'icon_name': 'SV_MY_TOOLS'}, ),
            ("tools.convert", "SvMyConvertNode"),

            # Nested subcategory (dict within the list)
            {"Transform Tools": [
                ("tools.transform.move", "SvMyMoveNode"),
                ("tools.transform.rotate", "SvMyRotateNode"),
                ("tools.transform.scale", "SvMyScaleNode"),
            ]},

            {"Analysis Tools": [
                ("tools.analysis.area", "SvMyAreaNode"),
                ("tools.analysis.volume", "SvMyVolumeNode"),
            ]},
        ]},
    ]
```

---

## Example 6: Check if Sverchok Extensions Are Available

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Programmatically check which Sverchok extensions are installed

import bpy
import addon_utils

def check_sverchok_extensions():
    """Check availability of Sverchok and its extensions."""
    extensions = {
        'sverchok': 'Sverchok (core)',
        'sverchok_extra': 'Sverchok-Extra',
        'sverchok_open3d': 'Sverchok-Open3d',
        'mega_polis': 'Mega-Polis',
        'ladybug_blender': 'Ladybug Tools',
    }

    results = {}
    for module_name, display_name in extensions.items():
        loaded, enabled = addon_utils.check(module_name)
        if loaded and enabled:
            results[module_name] = 'enabled'
        elif loaded:
            results[module_name] = 'loaded but not enabled'
        else:
            results[module_name] = 'not installed'
        print(f"  {display_name}: {results[module_name]}")

    return results

# Usage:
# check_sverchok_extensions()
```

---

## Example 7: Extension with Addon Preferences (Dependency Installer)

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Pattern: addon preferences with one-click pip install button

import bpy
import subprocess
import sys

class SvMyExtInstallDep(bpy.types.Operator):
    bl_idname = "my_ext.install_dependency"
    bl_label = "Install Required Library"

    library: bpy.props.StringProperty()

    def execute(self, context):
        python = sys.executable
        try:
            subprocess.check_call([python, "-m", "pip", "install", self.library])
            self.report({'INFO'}, f"{self.library} installed successfully")
        except subprocess.CalledProcessError as e:
            self.report({'ERROR'}, f"Failed to install {self.library}: {e}")
        return {'FINISHED'}

class SvMyExtPreferences(bpy.types.AddonPreferences):
    bl_idname = "my_sverchok_ext"

    def draw(self, context):
        layout = self.layout

        # Check each dependency
        deps = [
            ("scipy", "SciPy"),
            ("open3d", "Open3D"),
            ("sdf", "python-sdf"),
        ]

        for module_name, display_name in deps:
            row = layout.row()
            try:
                mod = __import__(module_name)
                version = getattr(mod, '__version__', 'unknown')
                row.label(text=f"{display_name} {version}", icon='CHECKMARK')
            except ImportError:
                row.label(text=f"{display_name} not found", icon='ERROR')
                op = row.operator("my_ext.install_dependency",
                                text=f"Install {display_name}")
                op.library = module_name

def register():
    bpy.utils.register_class(SvMyExtInstallDep)
    bpy.utils.register_class(SvMyExtPreferences)

def unregister():
    bpy.utils.unregister_class(SvMyExtPreferences)
    bpy.utils.unregister_class(SvMyExtInstallDep)
```

---

## Example 8: Using SDF Nodes — AEC Facade Panel

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and Sverchok-Extra
# AEC use case: parametric facade panel using SDF operations

import bpy

tree = bpy.data.node_groups.new("FacadePanel_SDF", 'SverchCustomTreeType')

with tree.init_tree():
    # Base panel: SDF Box
    panel = tree.nodes.new('SvSdfBoxNode')
    panel.location = (0, 0)
    # Set panel dimensions via node properties

    # Window opening: SDF Rounded Box (subtracted from panel)
    window = tree.nodes.new('SvSdfRoundedBoxNode')
    window.location = (0, -200)

    # Scale window to proper proportions
    scale = tree.nodes.new('SvSdfScaleNode')
    scale.location = (250, -200)
    tree.links.new(window.outputs[0], scale.inputs[0])

    # Translate window to position on panel
    translate = tree.nodes.new('SvSdfTranslateNode')
    translate.location = (500, -200)
    tree.links.new(scale.outputs[0], translate.inputs[0])

    # Boolean difference: panel - window
    boolean = tree.nodes.new('SvSdfBooleanNode')
    boolean.location = (500, 0)
    tree.links.new(panel.outputs[0], boolean.inputs[0])
    tree.links.new(translate.outputs[0], boolean.inputs[1])
    # Set operation to "difference"

    # Shell: hollow out the panel
    shell = tree.nodes.new('SvSdfShellNode')
    shell.location = (750, 0)
    tree.links.new(boolean.outputs[0], shell.inputs[0])

    # Generate mesh from SDF
    generate = tree.nodes.new('SvSdfGenerateNode')
    generate.location = (1000, 0)
    tree.links.new(shell.outputs[0], generate.inputs[0])

    # Output to viewer
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (1250, 0)
    tree.links.new(generate.outputs['Vertices'], viewer.inputs['vertices'])
    tree.links.new(generate.outputs['Faces'], viewer.inputs['faces'])

tree.force_update()
```

---

## Example 9: Point Cloud Workflow with Open3d Nodes

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and Sverchok-Open3d
# Workflow: import point cloud -> downsample -> reconstruct mesh

import bpy

tree = bpy.data.node_groups.new("PointCloud_Workflow", 'SverchCustomTreeType')

with tree.init_tree():
    # Import point cloud from file
    importer = tree.nodes.new('SvO3ImportNode')
    importer.location = (0, 0)
    # Set file path in node properties

    # Downsample to reduce density
    downsample = tree.nodes.new('SvO3DownsampleNode')
    downsample.location = (250, 0)
    tree.links.new(importer.outputs[0], downsample.inputs[0])

    # Estimate normals (required for Poisson reconstruction)
    normals = tree.nodes.new('SvO3NormalsNode')
    normals.location = (500, 0)
    tree.links.new(downsample.outputs[0], normals.inputs[0])

    # Poisson surface reconstruction
    poisson = tree.nodes.new('SvO3PoissonNode')
    poisson.location = (750, 0)
    tree.links.new(normals.outputs[0], poisson.inputs[0])

    # Simplify the resulting mesh
    simplify = tree.nodes.new('SvO3SimplifyNode')
    simplify.location = (1000, 0)
    tree.links.new(poisson.outputs[0], simplify.inputs[0])

    # View result
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (1250, 0)
    # Connect mesh outputs to viewer
    tree.links.new(simplify.outputs['Vertices'], viewer.inputs['vertices'])
    tree.links.new(simplify.outputs['Faces'], viewer.inputs['faces'])

tree.force_update()
```
