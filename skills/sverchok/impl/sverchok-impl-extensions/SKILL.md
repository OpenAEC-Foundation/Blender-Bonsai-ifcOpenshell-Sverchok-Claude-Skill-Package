---
name: sverchok-impl-extensions
description: 'Covers the Sverchok extension ecosystem and extension development pattern. Includes Sverchok-Extra for advanced geometry (surfaces, fields, solids, SDF), Open3d integration, and a guide to developing and registering custom extensions. Activates when working with SDF nodes, point cloud processing, advanced surface/field/solid operations, or building a custom Sverchok extension add-on.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.4.0+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-impl-extensions

## Quick Reference

### What Are Sverchok Extensions

Sverchok extensions are Blender add-ons that register additional nodes into Sverchok's node editor. They follow a standard pattern: a `bl_info` dictionary, a `nodes_index()` function that declares node categories and classes, and `register()`/`unregister()` functions that hook into Sverchok's menu and node discovery system.

- **Sverchok-Extra**: Advanced geometry (surfaces, fields, solids, SDF) by Ilya Portnov
- **Sverchok-Open3d**: Point cloud and mesh processing via Open3D by Victor Doval
- **Mega-Polis**: Urban design and GIS data integration
- **Ladybug Tools**: Environmental analysis (weather, radiation, comfort)
- **Sverchok-Bmesh**: BMesh operations within Sverchok
- **TopologicSverchok**: Non-manifold topology (covered in skill sverchok-impl-topologic)

### Critical Warnings

**NEVER** assume Sverchok-Extra nodes are available without checking — they require optional dependencies (SciPy, python-sdf). Use dependency guards in code that references these nodes.

**NEVER** use SDF nodes without the `python-sdf` library installed — the SDF Primitives and SDF Operations categories depend entirely on it.

**NEVER** mix TopologicSverchok content with this skill — TopologicSverchok is documented separately in sverchok-impl-topologic.

**ALWAYS** install Sverchok core first before any extension — all extensions depend on Sverchok's `SverchCustomTreeNode`, `updateNode`, and `add_node_menu` infrastructure.

**ALWAYS** use the standard extension pattern (bl_info + nodes_index + register/unregister) when creating extensions — Sverchok discovers extensions through this contract.

### Decision Tree

```
Need advanced surface/field operations?
  -> Sverchok-Extra (requires SciPy)
  -> Surface Extra: Smooth Bivariate Spline, Implicit Surface, Curvature Lines
  -> Field Extra: Vector Field Lines on Surface

Need SDF (Signed Distance Field) modeling?
  -> Sverchok-Extra (requires python-sdf library)
  -> SDF Primitives: Box, Sphere, Cylinder, Torus, Capsule, Platonic Solid
  -> SDF Operations: Boolean, Blend, Extrude, Revolve, Twist, Bend, Shell

Need point cloud or mesh reconstruction?
  -> Sverchok-Open3d (requires Open3D library)
  -> Point Cloud: import, downsample, normals, mask, join
  -> Triangle Mesh: Alpha Shape, Ball Pivoting, Poisson reconstruction

Need to create a custom Sverchok extension?
  -> Follow the extension development pattern (see Pattern 1 below)
  -> 7 mandatory elements per node class
  -> Standard directory layout: __init__.py + nodes/ directory

Need urban design / GIS?
  -> Mega-Polis (requires GDAL + many optional deps)

Need environmental analysis?
  -> Ladybug Tools (alpha, requires ladybug-tools libraries)
```

---

## Sverchok-Extra — Advanced Geometry Nodes

**Source**: https://github.com/portnov/sverchok-extra
**Author**: Ilya Portnov
**Status**: Experimental ("a sandbox/nursery for new Sverchok nodes")
**Version**: 0.1.0.0
**License**: GPL
**Blender**: 2.81.0+ (compatible with 4.0+/5.x)

### Node Categories

#### Surface Extra (dependency: SciPy)

| Node | bl_idname Pattern | Purpose |
|------|-------------------|---------|
| Smooth Bivariate Spline | `surface.smooth_spline` | Smooth interpolation of scattered 3D data |
| Blend Surface | `surface.blend_surface_ex` | Blend between two surfaces |
| Implicit Surface Solver | `surface.implicit_surface_solver` | Solve implicit surface equations |
| Curvature Lines | `surface.curvature_lines` | Extract principal curvature lines |
| Triangular Mesh | `surface.triangular_mesh` | Generate triangular surface mesh |

#### Field Extra (dependency: SciPy)

| Node | Purpose |
|------|---------|
| Vector Field Lines on Surface | Trace field lines across a surface |
| Exponential Map | Compute exponential map on surfaces |
| SDF Bounds Estimation | Estimate bounding box for SDF fields |

#### Solid Extra

| Node | Purpose |
|------|---------|
| Solid Waffle | Generate waffle/egg-crate structures from solids |

#### Spatial Extra

| Node | Purpose |
|------|---------|
| Delaunay 3D on Surface | Constrained Delaunay triangulation on surfaces |
| Delaunay Mesh | Delaunay triangulation for mesh generation |

#### Matrix Extra

| Node | Purpose |
|------|---------|
| Project Matrix on Plane | Project a transformation matrix onto a plane |

#### SDF Primitives (dependency: python-sdf)

| Node File | Primitive |
|-----------|-----------|
| `sdf_sphere.py` | Sphere |
| `sdf_box.py` | Box |
| `sdf_rounded_box.py` | Rounded Box |
| `sdf_cylinder.py` | Cylinder |
| `sdf_rounded_cylinder.py` | Rounded Cylinder |
| `sdf_torus.py` | Torus |
| `sdf_capsule.py` | Capsule |
| `sdf_plane.py` | Infinite Plane |
| `sdf_slab.py` | Slab (bounded plane) |
| `sdf_platonic_solid.py` | Platonic Solids (tetra, cube, octa, dodeca, icosa) |
| `sdf2d_circle.py` | 2D Circle |
| `sdf2d_hexagon.py` | 2D Hexagon |
| `sdf2d_polygon.py` | 2D Polygon |

#### SDF Operations (dependency: python-sdf)

| Node File | Operation | Description |
|-----------|-----------|-------------|
| `sdf_translate.py` | Translate | Move SDF in space |
| `sdf_scale.py` | Scale | Uniform/non-uniform scaling |
| `sdf_rotate.py` | Rotate | Rotation by axis/angle |
| `sdf_orient.py` | Orient | Orient SDF to match direction |
| `sdf_transform.py` | General Transform | Full 4x4 matrix transform |
| `sdf_boolean.py` | Boolean | Union, intersection, difference |
| `sdf_blend.py` | Blend | Smooth blend between two SDFs |
| `sdf_transition_linear.py` | Linear Transition | Linear morph between SDFs |
| `sdf_transition_radial.py` | Radial Transition | Radial morph between SDFs |
| `sdf_dilate_erode.py` | Dilate/Erode | Offset surface inward or outward |
| `sdf_shell.py` | Shell | Hollow out an SDF to create a shell |
| `sdf_twist.py` | Twist | Twist SDF around an axis |
| `sdf_linear_bend.py` | Linear Bend | Bend SDF along an axis |
| `sdf_slice.py` | Slice | Cut SDF with a plane |
| `sdf_extrude.py` | Extrude | Extrude 2D SDF to 3D |
| `sdf_extrude_to.py` | Extrude To | Extrude between two profiles |
| `sdf_revolve.py` | Revolve | Revolve 2D SDF around axis |
| `sdf_generate.py` | Generate | Convert SDF to mesh (marching cubes) |
| `estimate_bounds.py` | Estimate Bounds | Compute bounding box for SDF evaluation |

#### Additional Categories

- **2D Geometry** (Shapely): ~20 nodes for polygon, polyline, boolean, buffer, hull, simplify
- **Curve Extra**: ~12 nodes for surface intersection, Fourier, geodesic, NURBS solving
- **Data/Exchange**: Spreadsheet, Data Item, Excel I/O, SVG Reader
- **Array**: 41+ nodes | **API**: API In/Out nodes

### Dependencies

| Library | Required By | Install |
|---------|------------|---------|
| SciPy | Surface Extra, Field Extra | Sverchok installer or pip |
| python-sdf | SDF Primitives, SDF Operations | `pip install sdf` |
| Shapely | 2D Geometry | pip |

---

## Sverchok-Open3d — Point Cloud and Mesh Processing

**Source**: https://github.com/vicdoval/sverchok-open3d
**Author**: Victor Doval
**Version**: 0.1.0.0
**License**: GPL-3.0
**Dependency**: Open3D library (installable via addon preferences)

### Point Cloud Nodes

| Operation | Description |
|-----------|-------------|
| Import/Export | Load and save point cloud files |
| Downsample | Reduce point density (voxel, uniform, random) |
| Mask/Filter | Filter points by criteria |
| Normal Calculation | Estimate normals (standard and tangent plane methods) |
| Join | Combine multiple point clouds |
| Transform | Matrix-based transformations |

### Triangle Mesh Nodes

| Operation | Description |
|-----------|-------------|
| Alpha Shape | Reconstruct mesh from point cloud via alpha shapes |
| Ball Pivoting | Ball pivoting algorithm for mesh reconstruction |
| Poisson Reconstruction | Poisson surface reconstruction from oriented points |
| Quadric Decimation | Simplify mesh via quadric error metrics |
| Vertex Clustering | Simplify by merging nearby vertices |
| Merge by Distance | Remove near-duplicate vertices |
| Smoothing (Simple) | Iterative Laplacian smoothing |
| Smoothing (Laplacian) | Cotangent-weighted Laplacian smoothing |
| Smoothing (Taubin) | Taubin smoothing (avoids shrinkage) |
| Subdivision (Loop) | Loop subdivision for triangle meshes |
| Subdivision (Midpoint) | Midpoint subdivision |
| Rigid Deformation | As-Rigid-As-Possible surface deformation |
| Intersection Detection | Detect mesh-mesh intersections |
| Self-Intersection | Find self-intersecting triangles |
| Join | Combine multiple meshes |
| Separate Loose | Split mesh into connected components |
| Clean | Remove duplicates, fix normals, repair non-manifold edges |

### Custom Socket Types

Sverchok-Open3d defines custom socket types for Open3D data structures (PointCloud, TriangleMesh) to maintain type safety in the node graph.

**Stability warning**: Some nodes may crash Blender. Save frequently when using Open3D nodes.

---

## Other Extensions

### Mega-Polis — Urban Design and GIS

**Source**: https://github.com/victorcalixto/mega-polis
**Status**: Experimental
**Dependencies**: GDAL, OpenMPI, Boost, RichDEM, plus many optional (YOLOv5, Detectron2, Bokeh, Plotly)

| Category | Capabilities |
|----------|-------------|
| **Gathering** | Shapefile, GeoPackage, GeoJSON, CSV, DEM/LAS, OSM download, Mapillary |
| **Analysis** | Terrain processing, network analysis, isovist, object detection, ML models |
| **Generation** | Mesh from coordinates, lat/lon conversion, GeoJSON export |
| **Visualisation** | Bokeh/Plotly dashboards, Seaborn plots, WebVR/A-Frame |

### Ladybug Tools — Environmental Analysis

**Source**: https://github.com/ladybug-tools/ladybug-blender
**Status**: Alpha (incomplete)
**Dependencies**: ladybug-core, ladybug-geometry, ladybug-comfort

Provides Sverchok nodes for weather data analysis, radiation calculations, and comfort analysis. Currently supports only Ladybug nodes (Honeybee, Dragonfly, Butterfly not yet available).

### Sverchok-Bmesh — BMesh Operations

**Status**: Stable
**Dependencies**: None (uses Blender's built-in BMesh API)

Exposes Blender's BMesh operators as Sverchok nodes for advanced mesh editing operations within the node tree.

---

## Essential Patterns

### Pattern 1: Extension Development — Minimal Skeleton

```python
# __init__.py — Minimum viable Sverchok extension
# Blender 4.0+/5.x with Sverchok v1.4.0+

bl_info = {
    "name": "My Sverchok Extension",
    "author": "Your Name",
    "version": (0, 1, 0),
    "blender": (4, 0, 0),
    "category": "Node",
}

from sverchok.ui.nodeview_space_menu import add_node_menu

def nodes_index():
    """Return list of (category_name, node_list) tuples.
    Each node entry is (module_path, class_name)."""
    return [
        ("My Category", [
            ("category_name.node_one", "SvMyNodeOne"),
            ("category_name.node_two", "SvMyNodeTwo"),
        ]),
    ]

def make_node_list():
    """Convert nodes_index into importable modules."""
    # Implementation: iterate nodes_index(), import each module
    # from my_extension.nodes.category_name.node_one import ...
    pass

def register():
    for module in make_node_list():
        module.register()
    add_node_menu.register()

def unregister():
    for module in reversed(make_node_list()):
        module.unregister()
```

### Pattern 2: Extension Directory Structure

```
my_sverchok_extension/
  __init__.py              # bl_info, nodes_index(), register(), unregister()
  nodes_index.py           # (optional) separate file for nodes_index()
  dependencies.py          # (optional) dependency checking
  settings.py              # (optional) addon preferences
  icons.py                 # (optional) custom icons
  icons/                   # (optional) icon image files
  sockets.py               # (optional) custom socket types
  nodes/
    category_a/
      __init__.py
      node_one.py           # Single node per file
      node_two.py
    category_b/
      __init__.py
      node_three.py
  tests/                   # (optional) test files
  json_examples/           # (optional) example .json node trees
```

### Pattern 3: Node Registration — 7 Mandatory Elements

Every Sverchok extension node MUST implement these 7 elements:

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

# Element 1: Inherit from SverchCustomTreeNode (Sverchok mixin)
# Element 2: Inherit from bpy.types.Node (Blender node base)
class SvMyExtensionNode(SverchCustomTreeNode, bpy.types.Node):
    # Element 3: Unique bl_idname (convention: 'Sv' prefix)
    bl_idname = 'SvMyExtensionNode'
    # Element 4: Display name
    bl_label = 'My Extension Node'

    my_prop: bpy.props.FloatProperty(
        name="Value", default=1.0,
        # Element 7: updateNode callback on ALL properties
        update=updateNode
    )

    # Element 5: sv_init — socket creation
    def sv_init(self, context):
        self.inputs.new('SvStringsSocket', 'Value').prop_name = 'my_prop'
        self.outputs.new('SvStringsSocket', 'Result')

    # Element 6: process — main computation
    def process(self):
        if not self.outputs['Result'].is_linked:
            return
        val = self.inputs['Value'].sv_get(default=[[self.my_prop]])
        result = [[v * 2 for v in obj] for obj in val]
        self.outputs['Result'].sv_set(result)

def register():
    bpy.utils.register_class(SvMyExtensionNode)

def unregister():
    bpy.utils.unregister_class(SvMyExtensionNode)
```

### Pattern 4: Dependency-Guarded Node

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Pattern used by Sverchok-Extra for optional dependency nodes
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode

try:
    import scipy
    scipy_available = True
except ImportError:
    scipy_available = False

if scipy_available:
    class SvSmoothSplineNode(SverchCustomTreeNode, bpy.types.Node):
        bl_idname = 'SvExSmoothSplineNode'
        bl_label = 'Smooth Bivariate Spline'

        def sv_init(self, context):
            self.inputs.new('SvVerticesSocket', 'Vertices')
            self.outputs.new('SvSurfaceSocket', 'Surface')

        def process(self):
            from scipy.interpolate import SmoothBivariateSpline
            # ... node implementation using SciPy
            pass

    def register():
        bpy.utils.register_class(SvSmoothSplineNode)

    def unregister():
        bpy.utils.unregister_class(SvSmoothSplineNode)
else:
    def register():
        pass  # Node not available without SciPy

    def unregister():
        pass
```

### Pattern 5: nodes_index Format (Real-World Example)

```python
# From Sverchok-Extra nodes_index.py (simplified)
# Each category is a tuple: (category_name, node_entries)
# Node entries can include icon definitions and separators (None)

def nodes_index():
    return [
        {"Extra Surfaces": [
            ({'icon_name': 'SV_SURFACE'}, ),  # Category icon
            ("surface.smooth_spline", "SvExBiSmoothSplineNode"),
            ("surface.blend_surface_ex", "SvExBlendSurfaceNode"),
            None,  # Menu separator
            ("surface.implicit_surface_solver", "SvExImplSurfSolverNode"),
            ("surface.curvature_lines", "SvExCurvatureLinesNode"),
        ]},
        {"SDF Primitives": [
            ({'icon_name': 'SV_SDF'}, ),
            ("sdf_primitives.sdf_sphere", "SvSdfSphereNode"),
            ("sdf_primitives.sdf_box", "SvSdfBoxNode"),
            ("sdf_primitives.sdf_cylinder", "SvSdfCylinderNode"),
            ("sdf_primitives.sdf_torus", "SvSdfTorusNode"),
            # ... more primitives
        ]},
        {"SDF Operations": [
            ("sdf.sdf_boolean", "SvSdfBooleanNode"),
            ("sdf.sdf_blend", "SvSdfBlendNode"),
            ("sdf.sdf_extrude", "SvSdfExtrudeNode"),
            ("sdf.sdf_revolve", "SvSdfRevolveNode"),
            ("sdf.sdf_generate", "SvSdfGenerateNode"),
            # ... more operations
        ]},
    ]
```

---

## Common Operations

### Installing Extensions

All extensions follow the same base install: download zip from GitHub, Blender > Preferences > Add-ons > Install from File, enable. Then install dependencies: Sverchok-Extra needs SciPy (via Sverchok prefs) and optionally python-sdf (pip). Sverchok-Open3d has a one-click Open3D installer in addon preferences. Mega-Polis needs manual GDAL installation.

### Extension Architecture Summary

| Component | Purpose | Required |
|-----------|---------|----------|
| `bl_info` | Addon metadata (name, version, blender version) | Yes |
| `nodes_index()` | Declares node categories and class mappings | Yes |
| `register()` | Registers nodes, menus, sockets, icons | Yes |
| `unregister()` | Reverses registration in reverse order | Yes |
| `dependencies.py` | Optional dependency detection and management | No |
| `settings.py` | Addon preferences panel | No |
| `sockets.py` | Custom socket type definitions | No |
| `icons/` | Custom node icons (SVG or PNG) | No |

### Sverchok-Extra Node Category Summary

| Category | Node Count | Dependency | Status |
|----------|-----------|------------|--------|
| Surface Extra | 5 | SciPy | Experimental |
| Curve Extra | ~12 | SciPy | Experimental |
| Field Extra | 3 | SciPy | Experimental |
| Solid Extra | 1 | None | Experimental |
| Spatial Extra | 2 | None | Experimental |
| Matrix Extra | 1 | None | Experimental |
| SDF Primitives | 13 | python-sdf | Experimental |
| SDF Operations | 19 | python-sdf | Experimental |
| 2D Geometry | ~20 | Shapely | Experimental |
| Data | 4 | openpyxl (Excel) | Experimental |
| Exchange | 1 | None | Experimental |
| Array | 41+ | None | Experimental |
| API | 2 | None | Experimental |

---

## Reference Links

- [references/methods.md](references/methods.md) -- Extension registration API, nodes_index format, dependency management patterns
- [references/examples.md](references/examples.md) -- Working code examples for extension development, SDF workflows, Open3D integration
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes when developing or using Sverchok extensions

### Official Sources
- https://github.com/portnov/sverchok-extra
- https://github.com/vicdoval/sverchok-open3d
- https://github.com/victorcalixto/mega-polis
- https://github.com/ladybug-tools/ladybug-blender
- https://nortikin.github.io/sverchok/docs/introduction/sverchok_extensions.html
