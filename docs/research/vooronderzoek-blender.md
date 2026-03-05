# Vooronderzoek: Blender Python API (bpy)

**Date**: 2026-03-05
**Status**: COMPLETE
**Author**: Research Agent (Phase 2)
**Scope**: Blender Python API comprehensive analysis for Claude Skill Package
**Versions Covered**: Blender 3.x, 4.0, 4.1, 4.2 LTS, 4.3, 5.0

---

## Table of Contents

1. [Blender Python API Overview](#1-blender-python-api-bpy-complete-overview)
2. [Version Matrix: 3.x vs 4.x vs 5.0](#2-version-matrix-blender-3x-vs-4x-vs-50-breaking-changes)
3. [Addon Architecture](#3-addon-architecture)
4. [Operators](#4-operators)
5. [Properties](#5-properties)
6. [UI Panels](#6-ui-panels)
7. [Mesh & BMesh](#7-mesh--bmesh)
8. [Modifiers](#8-modifiers)
9. [Context System](#9-context-system)
10. [Dependency Graph](#10-dependency-graph-depsgraph)
11. [Common Error Patterns](#11-common-error-patterns)
12. [AI Common Mistakes](#12-ai-common-mistakes)
13. [Real-World Usage: OpenAEC Projects](#13-real-world-usage-openaec-projects)

---

## 1. Blender Python API (bpy) Complete Overview

The Blender Python API is exposed through the `bpy` module, which provides programmatic access to nearly all Blender functionality. The API is tightly coupled with Blender's internal C/C++ data model via RNA (Blender's introspection system).

### Core Module Structure

| Module | Purpose | Access Pattern |
|--------|---------|---------------|
| `bpy.data` | All blend-file data (objects, meshes, materials, scenes) | `bpy.data.objects["Cube"]` |
| `bpy.context` | Current state (active object, selected objects, mode) | `bpy.context.active_object` |
| `bpy.ops` | Operators (actions that modify data, with undo support) | `bpy.ops.mesh.primitive_cube_add()` |
| `bpy.types` | Type definitions for all Blender structs | `class MyOp(bpy.types.Operator):` |
| `bpy.props` | Property definitions for custom data | `bpy.props.FloatProperty()` |
| `bpy.utils` | Utility functions (registration, paths, previews) | `bpy.utils.register_class()` |
| `bpy.app` | Application info (version, paths, handlers) | `bpy.app.version` |
| `bpy.path` | Path utilities (Blender-aware) | `bpy.path.abspath("//file.png")` |
| `bpy.msgbus` | Message bus for property change notifications | `bpy.msgbus.subscribe_rna()` |

### Data Access Hierarchy

```
bpy.data (BlendData)
├── .objects          # All objects in the file
├── .meshes           # Mesh data blocks
├── .materials        # Materials
├── .scenes           # Scenes
├── .collections      # Collections
├── .node_groups      # Node groups (including Geometry Nodes)
├── .images           # Image data
├── .textures         # Textures
├── .lights           # Lights (renamed from lamps in 2.80)
├── .cameras          # Cameras
├── .curves           # Curve data
├── .armatures        # Armature data
├── .worlds           # World settings
├── .actions          # Animation actions
└── ... (30+ collection types)
```

### Key Architectural Principles

1. **RNA System**: Every Blender data type is introspectable. Properties are defined in C and exposed to Python automatically.
2. **ID Data Blocks**: Top-level data (Objects, Meshes, Materials) are `bpy.types.ID` subclasses with unique names, user counts, and fake-user flags.
3. **Operators vs Direct Data Access**: Operators provide undo/redo and user feedback. Direct data access (`bpy.data`) is faster but has no undo.
4. **Reference Counting**: Data blocks have `users` count. Zero-user data is cleaned on save/reload unless `use_fake_user = True`.

### Sources
- https://docs.blender.org/api/current/info_quickstart.html
- https://docs.blender.org/api/current/info_overview.html

---

## 2. Version Matrix: Blender 3.x vs 4.x vs 5.0 Breaking Changes

### Critical Breaking Changes Matrix

| Feature | Blender 3.x | Blender 4.0 | Blender 4.2 LTS | Blender 5.0 |
|---------|-------------|-------------|-----------------|-------------|
| **Addon metadata** | `bl_info` dict in `__init__.py` | `bl_info` dict (unchanged) | `blender_manifest.toml` (new) + `bl_info` (legacy) | `blender_manifest.toml` (legacy still works) |
| **Context overrides** | Dict arg to `bpy.ops` | **REMOVED** — use `context.temp_override()` | `context.temp_override()` | `context.temp_override()` |
| **Mesh bevel weights** | `MeshEdge.bevel_weight` property | **REMOVED** — use `bevel_weight_edge` attribute | Attribute-based | Attribute-based |
| **Mesh creases** | `MeshEdge.crease` property | **REMOVED** — use `crease_edge` attribute | Attribute-based | Attribute-based |
| **Face Maps** | `obj.face_maps` | **REMOVED** — integer face attributes | Removed | Removed |
| **Bone layers** | `bone.layers[i]` | **REMOVED** — use `bone.collections` | Collections | Collections |
| **Bone groups** | `pose.bone_groups` | **REMOVED** — use bone collections + colors | Collections | Collections |
| **Node socket access** | `node.inputs["Name"]` or `node.inputs[index]` | Now uses identifiers + availability | Identifiers | Identifiers |
| **Node group interface** | `NodeTree.inputs/outputs` | **MOVED** to `NodeTree.interface.new_socket()` | `interface` API | `interface` API |
| **Principled BSDF sockets** | `Subsurface`, `Specular`, `Transmission` | Renamed: `Subsurface Weight`, `Specular IOR Level`, `Transmission Weight` | Renamed sockets | Renamed sockets |
| **Calc normals** | `mesh.calc_normals()` | **REMOVED** (auto-calculated) | Removed | Removed |
| **Python OBJ/PLY IO** | Python importers/exporters | **REMOVED** — use `bpy.ops.wm.obj_import/export` | C++ IO only | C++ IO only |
| **BGL module** | Available | Available (deprecated) | Deprecated | **REMOVED** — use `gpu` module |
| **EEVEE identifier** | `BLENDER_EEVEE` | `BLENDER_EEVEE` | `BLENDER_EEVEE_NEXT` | **Changed back** to `BLENDER_EEVEE` |
| **IDProperties** | Dict-like access for RNA | Dict-like access | Static typing enforced | `scene['cycles']` **REMOVED** — use attribute access |
| **Asset context** | `context.asset_file_handle` | `context.asset` (AssetRepresentation) | `context.asset` | `context.active_file` **REMOVED** |
| **Compositing nodes** | `scene.node_tree` | `scene.node_tree` | `scene.node_tree` | `scene.compositing_node_group` |

### Blender 4.0 — Major Breaking Changes Detail

**Context Override Removal** (most impactful for scripts):
```python
# BROKEN in Blender 4.0+
override = {"object": obj, "active_object": obj}
bpy.ops.object.modifier_apply(override, modifier="Boolean")

# CORRECT for Blender 4.0+
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Boolean")
```

**Mesh Data Migration**:
```python
# BROKEN in Blender 4.0+
weight = mesh.edges[0].bevel_weight  # AttributeError
crease = mesh.edges[0].crease        # AttributeError

# CORRECT for Blender 4.0+
bevel_attr = mesh.attributes.get("bevel_weight_edge")
if bevel_attr:
    weight = bevel_attr.data[0].value

crease_attr = mesh.attributes.get("crease_edge")
if crease_attr:
    crease = crease_attr.data[0].value
```

**Node Group Interface Migration**:
```python
# BROKEN in Blender 4.0+
node_group.inputs.new("NodeSocketFloat", "My Input")
node_group.outputs.new("NodeSocketGeometry", "Geometry")

# CORRECT for Blender 4.0+
node_group.interface.new_socket(name="My Input", in_out='INPUT', socket_type='NodeSocketFloat')
node_group.interface.new_socket(name="Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')
```

### Blender 4.2 LTS — Extension System

The extension system introduced `blender_manifest.toml` as replacement for `bl_info`:

```toml
# blender_manifest.toml
schema_version = "1.0.0"
id = "my_addon"
version = "1.0.0"
name = "My Addon"
tagline = "Short description up to 64 chars"
maintainer = "Developer Name <email@example.com>"
type = "add-on"
blender_version_min = "4.2.0"

# Optional
website = "https://example.com"
tags = ["Modeling", "Mesh"]
license = ["SPDX:GPL-3.0-or-later"]
```

**Key differences from bl_info:**
- File-based metadata (not embedded in Python)
- `tagline` replaces `description` (64 char max, no trailing punctuation)
- `blender_version_min` replaces `blender` tuple
- `id` must match package directory name
- `type` is `"add-on"` or `"theme"` (string, not category)
- Permissions system for internet access: `[permissions] network = "Network access needed for..."`

### Blender 4.1 — Python API Changes

**Python version**: Upgraded to **Python 3.11** (VFX Platform 2024).

#### Breaking Changes

1. **Mesh Auto Smooth Removal** (HIGH impact)
   - `Mesh.use_auto_smooth` — REMOVED in 4.1
   - `Mesh.auto_smooth_angle` — REMOVED in 4.1
   - `Mesh.create_normals_split()` — REMOVED in 4.1
   - `Mesh.calc_normals_split()` — REMOVED in 4.1
   - `Mesh.free_normals_split()` — REMOVED in 4.1
   - `MeshLoop.normal` — now READ-ONLY in 4.1
   - **Replacement**: Use `Mesh.corner_normals` collection (auto-updated). For angle-based smoothing, use the "Smooth by Angle" modifier node group asset.
   - Custom normals MUST use `normals_split_custom_set()` or `normals_split_custom_set_from_vertices()`.

   ```python
   # BROKEN in Blender 4.1+ — use_auto_smooth removed
   mesh.use_auto_smooth = True
   mesh.auto_smooth_angle = 0.523599  # 30 degrees

   # Blender 4.1+ — use corner_normals instead
   # Auto smooth is now ALWAYS active; use "Smooth by Angle" modifier for angle control
   corner_normals = mesh.corner_normals  # read-only, auto-updated
   # For custom normals:
   mesh.normals_split_custom_set(normals_list)
   ```

2. **Light Probe Type Renames** (MEDIUM impact)
   - `CUBEMAP` → `SPHERE`
   - `PLANAR` → `PLANE`
   - `GRID` → `VOLUME`
   - `show_data` deprecated → use `use_data_display`

   ```python
   # BROKEN in Blender 4.1+ — old enum values
   if probe.type == 'CUBEMAP':
       pass

   # Blender 4.1+ — use new enum values
   if probe.type == 'SPHERE':  # Blender 4.1+
       pass
   ```

3. **Material Displacement Method** (MEDIUM impact)
   - `displacement_method` moved from `CyclesMaterialSettings` to base `Material` class.

   ```python
   # BROKEN in Blender 4.1+ — Cycles-specific path removed
   mat.cycles.displacement_method = 'BOTH'

   # Blender 4.1+ — property on Material directly
   mat.displacement_method = 'BOTH'  # Blender 4.1+
   ```

4. **foreach_set() Validation** (LOW impact)
   - `foreach_set()` now raises `TypeError` for invalid data types that previously failed silently.
   - Scripts that relied on silent failure MUST add proper type checking.

5. **Sequencer Transform Filter Rename** (LOW impact)
   - `SUBSAMPLING_3x3` renamed to `BOX`.

6. **Node Socket Access** (MEDIUM impact)
   - Dynamic socket types replaced static ones. Use socket **identifiers** instead of **indices** for reliable access.

#### New Features (4.1)

- **Layout Panels**: `layout.panel()` and `layout.panel_prop()` for collapsible UI sections without registration.
- **Enum ID Properties**: Integer properties support enum items via `id_properties_ui()`.
- **Asset Library API**: `Preferences.filepaths.asset_libraries` for programmatic asset library management.
- **Session UIDs**: `ID.session_uid` for unique identification of datablocks within a session.
- **Shape Key Points**: `ShapeKey.points` for direct point access.
- **Translation Handler**: `handlers.translation_update_post` fires when UI language changes.

### Blender 4.3 — Python API Changes

**Python version**: **Python 3.11** (same as 4.1).

#### Breaking Changes

1. **AttributeGroup Split** (MEDIUM impact)
   - `bpy.types.AttributeGroup` — REMOVED in 4.3
   - Replaced by type-specific classes:
     - `AttributeGroupMesh`
     - `AttributeGroupPointCloud`
     - `AttributeGroupCurves`
     - `AttributeGroupGreasePencil`
   - Mesh-only properties (`active_color`, `active_color_index`, `default_color_name`, `render_color_index`) are NOW accessible ONLY on `AttributeGroupMesh`.

   ```python
   # BROKEN in Blender 4.3+ — generic AttributeGroup
   attrs = obj.data.attributes  # was bpy.types.AttributeGroup
   color_name = attrs.active_color_name  # AttributeError in 4.3 if not mesh

   # Blender 4.3+ — type-specific AttributeGroup
   attrs = obj.data.attributes  # now AttributeGroupMesh for mesh objects
   color_name = attrs.active_color_name  # ONLY works on AttributeGroupMesh
   # Use domain_size() to check support:
   size = attrs.domain_size('POINT')  # returns 0 if unsupported  # Blender 4.3+
   ```

2. **Grease Pencil Python API Rewrite** (CRITICAL impact for GP add-ons)
   - The ENTIRE Grease Pencil Python API has been rewritten for the new data structure.
   - NOT backward compatible: files saved in 4.3+ do NOT load correctly in 4.2 or lower.
   - `pixel_factor` (Thickness Scale) — REMOVED in 4.3
   - Screen Space thickness mode — REMOVED (strokes are ALWAYS in World Space)
   - Selection order for stroke interpolation — REMOVED
   - Draw Mode guides — NOT ported to 4.3
   - See official migration guide: https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/

3. **Embedded ID Pointer Assignment** (LOW impact)
   - Assigning embedded IDs (e.g., `scene.collection`, root node trees) to `PointerProperty` now raises `RuntimeError`.

4. **Reroute Node Socket Changes** (LOW impact)
   - Reroute node data type changes via `socket_idname` property instead of direct socket modification.

5. **EEVEE Legacy Property Removal** (MEDIUM impact)
   - REMOVED properties: contact shadows, SSR, volumetric lighting, GTAO, bokeh, bloom, and shadow settings.
   - Scripts referencing these EEVEE properties MUST be updated.

#### New Features (4.3)

- **`bpy.app.python_args`**: Call Python in Blender's environment.
- **Blend import handlers**: `blend_import_pre` and `blend_import_post` handlers with `BlendImportContext` parameter.
- **`ID.rename()`**: Complex renaming behavior (direct `ID.name` assignment unchanged).
- **`domain_size()`**: Returns attribute domain size (0 if unsupported).
- **`foreach_set()` triggers updates**: Calling `foreach_set()` on attribute data now triggers property updates.
- **Curves API**: `curves.remove_curves(indices=[])` and `curves.resize_curves(sizes, indices=[])`.

### Blender 5.0 — Additional Breaking Changes

- **BGL completely removed**: ALL `bgl.*` calls must migrate to `gpu` module
- **Properties runtime storage**: `bpy.context.scene['cycles']` no longer works; use attribute access
- **Compositing**: `scene.node_tree` → `scene.compositing_node_group`
- **VSE**: `Sequence.end_frame` replaced with `length` property
- **Brush system**: Renamed `sculpt_tool` → `sculpt_brush_type`

### BGL-to-gpu Migration Guide

**Context**: The `bgl` module (OpenGL wrapper) is deprecated since Blender 3.5 and is **completely removed in Blender 5.0**. ALL drawing code MUST migrate to the `gpu` module. The `gpu` module provides a graphics-API-independent abstraction that works across OpenGL, Vulkan, and Metal.

> **Note**: The IfcOpenShell/Bonsai project completed this migration (see [IfcOpenShell issue #2897](https://github.com/IfcOpenShell/IfcOpenShell/issues/2897)), demonstrating that this migration is essential for BIM/AEC add-ons.

#### Before (bgl — Blender ≤ 4.x, BROKEN in 5.0)

```python
# bgl viewport overlay example — DEPRECATED since Blender 3.5
# BROKEN on Apple M1/M2 (Metal backend), WILL BREAK on Vulkan backend
import bpy
import bgl
from gpu_extras.batch import batch_for_shader
import gpu

coords = [
    (0.0, 0.0, 0.0), (5.0, 0.0, 0.0),
    (5.0, 0.0, 0.0), (5.0, 3.0, 0.0),
    (5.0, 3.0, 0.0), (0.0, 3.0, 0.0),
    (0.0, 3.0, 0.0), (0.0, 0.0, 0.0),
]

shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')  # REMOVED in Blender 4.0+
batch = batch_for_shader(shader, 'LINES', {"pos": coords})


def draw_callback():
    bgl.glEnable(bgl.GL_BLEND)                    # DEPRECATED — use gpu.state.blend_set()
    bgl.glEnable(bgl.GL_LINE_SMOOTH)               # DEPRECATED — no direct replacement
    bgl.glLineWidth(2)                              # DEPRECATED — use gpu.state.line_width_set()
    bgl.glEnable(bgl.GL_DEPTH_TEST)                 # DEPRECATED — use gpu.state.depth_test_set()
    bgl.glDepthFunc(bgl.GL_LEQUAL)                  # DEPRECATED — use gpu.state.depth_test_set()

    shader.bind()
    shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))
    batch.draw(shader)

    bgl.glDisable(bgl.GL_BLEND)
    bgl.glDisable(bgl.GL_LINE_SMOOTH)
    bgl.glLineWidth(1)
    bgl.glDisable(bgl.GL_DEPTH_TEST)


_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_callback, (), 'WINDOW', 'POST_VIEW'
)
```

#### After (gpu — Blender 4.0+, REQUIRED for 5.0)

```python
# gpu viewport overlay example — Blender 4.0+
# Works on ALL backends: OpenGL, Vulkan, Metal
import bpy
import gpu
from gpu_extras.batch import batch_for_shader  # Blender 3.0+

coords = [
    (0.0, 0.0, 0.0), (5.0, 0.0, 0.0),
    (5.0, 0.0, 0.0), (5.0, 3.0, 0.0),
    (5.0, 3.0, 0.0), (0.0, 3.0, 0.0),
    (0.0, 3.0, 0.0), (0.0, 0.0, 0.0),
]

# Blender 4.0+: use 'POLYLINE_UNIFORM_COLOR' (replaces '3D_UNIFORM_COLOR')
shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')  # Blender 4.0+
batch = batch_for_shader(shader, 'LINES', {"pos": coords})


def draw_callback():
    gpu.state.blend_set('ALPHA')                    # Replaces bgl.glEnable(bgl.GL_BLEND)
    gpu.state.depth_test_set('LESS_EQUAL')          # Replaces bgl.glEnable(bgl.GL_DEPTH_TEST)
    gpu.state.depth_mask_set(True)                  # Replaces bgl.glDepthMask(bgl.GL_TRUE)
    gpu.state.line_width_set(2.0)                   # Replaces bgl.glLineWidth(2)

    shader.bind()
    # POLYLINE_UNIFORM_COLOR requires viewportSize and lineWidth uniforms
    shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])  # Blender 4.0+
    shader.uniform_float("lineWidth", 2.0)          # Line width in pixels  # Blender 4.0+
    shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))
    batch.draw(shader)

    # Restore gpu state to defaults
    gpu.state.blend_set('NONE')
    gpu.state.depth_test_set('NONE')
    gpu.state.depth_mask_set(False)
    gpu.state.line_width_set(1.0)


_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_callback, (), 'WINDOW', 'POST_VIEW'
)
# To remove: bpy.types.SpaceView3D.draw_handler_remove(_handle, 'WINDOW')
```

#### BGL-to-gpu API Mapping Reference

| bgl (DEPRECATED) | gpu replacement (Blender 3.5+) | Notes |
|---|---|---|
| `bgl.glEnable(bgl.GL_BLEND)` | `gpu.state.blend_set('ALPHA')` | Use `'ALPHA'`, `'ADDITIVE'`, `'ADDITIVE_PREMUL'`, or `'MULTIPLY'` |
| `bgl.glDisable(bgl.GL_BLEND)` | `gpu.state.blend_set('NONE')` | |
| `bgl.glLineWidth(w)` | `gpu.state.line_width_set(w)` | Also set `lineWidth` uniform for POLYLINE shaders |
| `bgl.glEnable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('LESS_EQUAL')` | Options: `'NONE'`, `'LESS'`, `'LESS_EQUAL'`, `'EQUAL'`, `'GREATER'` |
| `bgl.glDisable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('NONE')` | |
| `bgl.glDepthMask(GL_TRUE)` | `gpu.state.depth_mask_set(True)` | |
| `bgl.glEnable(bgl.GL_LINE_SMOOTH)` | *(no direct replacement)* | Use `POLYLINE_*` shaders for antialiased lines |
| `bgl.glPointSize(s)` | `gpu.state.point_size_set(s)` | |
| `gpu.shader.from_builtin('3D_UNIFORM_COLOR')` | `gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')` | `3D_` prefix removed in Blender 4.0 |
| `gpu.shader.from_builtin('3D_FLAT_COLOR')` | `gpu.shader.from_builtin('POLYLINE_FLAT_COLOR')` | `3D_` prefix removed in Blender 4.0 |
| `image.gl_load()` / `bgl.glBindTexture()` | `gpu.texture.from_image(image)` | Completely different API |

#### BGL Migration Checklist

- Replace ALL `import bgl` statements — remove entirely
- Replace `bgl.glEnable(bgl.GL_BLEND)` with `gpu.state.blend_set('ALPHA')`
- Replace `bgl.glDisable(bgl.GL_BLEND)` with `gpu.state.blend_set('NONE')`
- Replace `bgl.glLineWidth(n)` with `gpu.state.line_width_set(n)`
- Replace `bgl.glEnable(bgl.GL_DEPTH_TEST)` with `gpu.state.depth_test_set('LESS_EQUAL')`
- Replace `bgl.glDisable(bgl.GL_DEPTH_TEST)` with `gpu.state.depth_test_set('NONE')`
- Replace `3D_UNIFORM_COLOR` with `POLYLINE_UNIFORM_COLOR` (Blender 4.0+)
- Replace `3D_FLAT_COLOR` with `POLYLINE_FLAT_COLOR` (Blender 4.0+)
- Replace `image.gl_load()` with `gpu.texture.from_image(image)`
- ALWAYS restore `gpu.state` to defaults at end of draw callbacks
- Test on Apple Silicon (Metal backend) — bgl is ALREADY non-functional there

### Sources
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.2/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- https://docs.blender.org/api/current/change_log.html
- https://developer.blender.org/docs/release_notes/
- https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/
- https://github.com/IfcOpenShell/IfcOpenShell/issues/2897

---

## 3. Addon Architecture

### Legacy Addon (Blender ≤ 4.1): bl_info System

Single-file addon:
```python
bl_info = {
    "name": "My Addon",
    "author": "Developer",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),
    "location": "View3D > Sidebar > My Tab",
    "description": "Does something useful",
    "warning": "",
    "doc_url": "https://example.com/docs",
    "category": "Object",
}

import bpy

class MY_OT_example(bpy.types.Operator):
    bl_idname = "my.example"
    bl_label = "Example Operator"

    def execute(self, context):
        return {'FINISHED'}

class MY_PT_panel(bpy.types.Panel):
    bl_label = "My Panel"
    bl_idname = "MY_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        self.layout.operator("my.example")

classes = (MY_OT_example, MY_PT_panel)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)

if __name__ == "__main__":
    register()
```

### Multi-File Addon Structure

```
my_addon/
├── __init__.py          # bl_info + register/unregister + imports
├── operators.py         # Operator classes
├── panels.py            # Panel classes
├── properties.py        # PropertyGroup classes
├── preferences.py       # AddonPreferences class
└── utils.py             # Helper functions
```

**`__init__.py`** for multi-file:
```python
bl_info = {
    "name": "My Addon",
    "blender": (3, 6, 0),
    "category": "Object",
    "version": (1, 0, 0),
}

# Reload support
if "bpy" in locals():
    import importlib
    importlib.reload(operators)
    importlib.reload(panels)
    importlib.reload(properties)
else:
    from . import operators
    from . import panels
    from . import properties

import bpy

classes = (
    properties.MyProperties,
    operators.MY_OT_example,
    panels.MY_PT_panel,
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=properties.MyProperties)

def unregister():
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

### Extension System (Blender 4.2+)

```
my_extension/
├── blender_manifest.toml  # Replaces bl_info
├── __init__.py            # register/unregister (NO bl_info needed)
├── operators.py
├── panels.py
└── wheels/                # Bundled Python packages (optional)
    └── some_lib.whl
```

### Addon Preferences

```python
class MyAddonPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__  # MUST match addon module name

    api_key: bpy.props.StringProperty(
        name="API Key",
        subtype='PASSWORD',
    )
    debug_mode: bpy.props.BoolProperty(
        name="Debug Mode",
        default=False,
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "api_key")
        layout.prop(self, "debug_mode")

# Access preferences at runtime:
prefs = bpy.context.preferences.addons[__package__].preferences
print(prefs.api_key)
```

### Class Naming Convention (REQUIRED)

Blender enforces a naming pattern: `{ADDON}_{TYPE}_{name}`

| Type Code | Base Class | Example |
|-----------|-----------|---------|
| `OT` | `bpy.types.Operator` | `MY_OT_do_something` |
| `PT` | `bpy.types.Panel` | `MY_PT_main_panel` |
| `MT` | `bpy.types.Menu` | `MY_MT_context_menu` |
| `UL` | `bpy.types.UIList` | `MY_UL_item_list` |
| `HT` | `bpy.types.Header` | `MY_HT_header` |
| `KI` | `bpy.types.KeyingSetInfo` | `MY_KI_keying_set` |

### Sources
- https://docs.blender.org/api/current/info_overview.html
- https://b3d.interplanety.org/en/creating-multifile-add-on-for-blender/
- https://docs.blender.org/manual/en/latest/advanced/scripting/addon_tutorial.html

---

## 4. Operators

### Operator Basics

```python
class MY_OT_simple(bpy.types.Operator):
    """Tooltip shown on hover"""          # Docstring = tooltip
    bl_idname = "my.simple_operator"      # Unique identifier (lowercase.lowercase)
    bl_label = "Simple Operator"          # Display name
    bl_description = "Detailed description"  # Optional, overrides docstring
    bl_options = {'REGISTER', 'UNDO'}     # Operator options

    # Properties (shown in operator panel, F9 after execution)
    size: bpy.props.FloatProperty(name="Size", default=1.0, min=0.0)

    @classmethod
    def poll(cls, context):
        """Return True if operator can run in current context"""
        return context.active_object is not None

    def execute(self, context):
        """Main operator logic. Called by invoke() or directly."""
        context.active_object.scale *= self.size
        return {'FINISHED'}
```

### bl_options Flags

| Flag | Effect |
|------|--------|
| `'REGISTER'` | Operator appears in info log and can be repeated (F3/spacebar) |
| `'UNDO'` | Creates undo step on `{'FINISHED'}` — **ALWAYS use for data-modifying ops** |
| `'UNDO_GROUPED'` | Groups multiple calls into one undo step |
| `'BLOCKING'` | Block all non-default events from being handled |
| `'MACRO'` | Operator is a macro (contains sub-operators) |
| `'GRAB_CURSOR'` | Enables wrapping when continuous grab is enabled |
| `'GRAB_CURSOR_X/Y'` | Wraps cursor on X or Y axis only |
| `'PRESET'` | Display a preset button with operator settings |
| `'INTERNAL'` | Do not show in search menu |
| `'MODAL_PRIORITY'` | Receive priority event handling in modal (4.2+) |

### Operator Methods Flow

```
User Action
    │
    ▼
  poll() ──── False ──▶ (operator disabled in UI, RuntimeError in script)
    │
  True
    │
    ▼
  invoke(context, event)     ← Called when user triggers operator
    │                           (button click, menu, shortcut)
    │
    ├──▶ return {'FINISHED'}   ← Done, no modal needed
    │
    ├──▶ call execute() ──▶ return execute result
    │
    ├──▶ wm.invoke_props_dialog(self) ──▶ shows dialog ──▶ execute()
    │
    └──▶ wm.modal_handler_add(self) + return {'RUNNING_MODAL'}
                │
                ▼
            modal(context, event)  ← Called repeatedly for events
                │
                ├──▶ return {'RUNNING_MODAL'}  ← Continue
                ├──▶ return {'FINISHED'}        ← Done
                ├──▶ return {'CANCELLED'}       ← Abort
                └──▶ return {'PASS_THROUGH'}    ← Let other handlers process
```

### Modal Operator Example

```python
class MY_OT_modal_timer(bpy.types.Operator):
    bl_idname = "my.modal_timer"
    bl_label = "Modal Timer"

    _timer = None
    _count = 0

    def modal(self, context, event):
        if event.type in {'RIGHTMOUSE', 'ESC'}:
            self.cancel(context)
            return {'CANCELLED'}

        if event.type == 'TIMER':
            self._count += 1
            if self._count >= 100:
                self.cancel(context)
                return {'FINISHED'}

        return {'PASS_THROUGH'}

    def execute(self, context):
        wm = context.window_manager
        self._timer = wm.event_timer_add(0.1, window=context.window)
        wm.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def cancel(self, context):
        wm = context.window_manager
        wm.event_timer_remove(self._timer)

    invoke = execute  # invoke just calls execute
```

### Invoke with Dialog

```python
class MY_OT_confirm(bpy.types.Operator):
    bl_idname = "my.confirm_action"
    bl_label = "Confirm Action"
    bl_options = {'REGISTER', 'UNDO'}

    message: bpy.props.StringProperty(default="Are you sure?")

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self)

    def draw(self, context):
        self.layout.label(text=self.message)

    def execute(self, context):
        self.report({'INFO'}, "Action confirmed")
        return {'FINISHED'}
```

### Operator Return Values

| Return Value | Meaning |
|-------------|---------|
| `{'FINISHED'}` | Operator completed successfully. Undo step created if `'UNDO'` in bl_options. |
| `{'CANCELLED'}` | Operator was cancelled. No undo step. |
| `{'RUNNING_MODAL'}` | Operator is running modally (receiving events). |
| `{'PASS_THROUGH'}` | Modal: let other operators handle this event too. |
| `{'INTERFACE'}` | Handled but not executed (popup displayed). |

### Sources
- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.ops.html

---

## 5. Properties

### Property Types

| Property Type | Python Type | Common Use |
|---------------|-------------|-----------|
| `BoolProperty` | `bool` | Toggle flags |
| `IntProperty` | `int` | Counts, indices |
| `FloatProperty` | `float` | Sizes, factors |
| `StringProperty` | `str` | Names, paths |
| `EnumProperty` | `str` (identifier) | Dropdowns, mode selectors |
| `BoolVectorProperty` | `tuple[bool]` | Axis toggles (X, Y, Z) |
| `IntVectorProperty` | `tuple[int]` | Dimensions, coordinates |
| `FloatVectorProperty` | `tuple[float]` | Colors (RGBA), vectors (XYZ) |
| `PointerProperty` | Reference to type | Link to PropertyGroup or ID type |
| `CollectionProperty` | List of items | Dynamic lists of PropertyGroups |

### Common Property Parameters

```python
my_float: bpy.props.FloatProperty(
    name="Display Name",           # Label in UI
    description="Tooltip text",    # Hover description
    default=1.0,                   # Default value
    min=0.0,                       # Hard minimum
    max=100.0,                     # Hard maximum
    soft_min=0.1,                  # Soft minimum (slider range)
    soft_max=10.0,                 # Soft maximum (slider range)
    step=10,                       # UI increment (value * 0.01 for float)
    precision=3,                   # Decimal places shown
    subtype='FACTOR',              # UI display hint
    unit='LENGTH',                 # Unit system
    options={'ANIMATABLE'},        # Property flags
    update=my_update_callback,     # Called when value changes
    get=my_getter,                 # Custom getter (overrides storage)
    set=my_setter,                 # Custom setter (overrides storage)
)
```

### Subtypes

| Subtype | Applies To | Display |
|---------|-----------|---------|
| `'NONE'` | All | Default |
| `'FILEPATH'` | String | File browser button |
| `'DIRPATH'` | String | Directory browser button |
| `'FILENAME'` | String | File name field |
| `'PASSWORD'` | String | Masked input |
| `'PIXEL'` | Int/Float | Pixel unit |
| `'FACTOR'` | Float | 0-1 slider |
| `'PERCENTAGE'` | Float | Percentage display |
| `'ANGLE'` | Float | Radians with degree display |
| `'COLOR'` | FloatVector | Color picker (RGB) |
| `'COLOR_GAMMA'` | FloatVector | Color picker (gamma space) |
| `'TRANSLATION'` | FloatVector | Position in scene units |
| `'DIRECTION'` | FloatVector | Normalized direction |
| `'EULER'` | FloatVector | Euler rotation |
| `'QUATERNION'` | FloatVector | Quaternion rotation |

### EnumProperty — Static Items

```python
my_enum: bpy.props.EnumProperty(
    name="Mode",
    items=[
        ('OPT_A', "Option A", "Description of option A", 'ICON_NAME', 0),
        ('OPT_B', "Option B", "Description of option B", 'ICON_NAME', 1),
        ('OPT_C', "Option C", "Description of option C", 'ICON_NAME', 2),
    ],
    default='OPT_A',
)
# items tuple: (identifier, name, description, icon, number)
# identifier: internal value (string returned by property)
# name: display text
# description: tooltip
# icon: optional icon identifier
# number: unique integer (MUST be unique, MUST be explicitly set for stable serialization)
```

### EnumProperty — Dynamic Items (Callback)

```python
def get_items(self, context):
    """Dynamic enum items callback.
    WARNING: Returned list MUST be stored to prevent garbage collection crashes."""
    items = []
    for i, obj in enumerate(bpy.data.objects):
        items.append((obj.name, obj.name, f"Select {obj.name}", 'OBJECT_DATA', i))
    return items

# CRITICAL: Store reference to prevent GC
_enum_items = []

def get_items_safe(self, context):
    global _enum_items
    _enum_items = [
        (obj.name, obj.name, "", i)
        for i, obj in enumerate(bpy.data.objects)
    ]
    return _enum_items

my_enum: bpy.props.EnumProperty(
    name="Object",
    items=get_items_safe,
)
```

### PropertyGroup

```python
class MySubProperties(bpy.types.PropertyGroup):
    value: bpy.props.FloatProperty(name="Value", default=0.0)
    name: bpy.props.StringProperty(name="Name", default="")

class MyProperties(bpy.types.PropertyGroup):
    enabled: bpy.props.BoolProperty(name="Enabled", default=True)
    count: bpy.props.IntProperty(name="Count", default=5, min=1)

    # Nested PropertyGroup
    sub: bpy.props.PointerProperty(type=MySubProperties)

    # Collection of PropertyGroups
    items: bpy.props.CollectionProperty(type=MySubProperties)
    active_item_index: bpy.props.IntProperty()

# Registration order matters: register sub-types FIRST
def register():
    bpy.utils.register_class(MySubProperties)   # FIRST
    bpy.utils.register_class(MyProperties)       # SECOND
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyProperties)

def unregister():
    del bpy.types.Scene.my_props
    bpy.utils.unregister_class(MyProperties)     # Reverse order
    bpy.utils.unregister_class(MySubProperties)
```

### Update Callbacks

```python
def on_count_changed(self, context):
    """Called whenever 'count' property changes.
    WARNING: Do NOT modify the property that triggered this callback
    (infinite recursion). Do NOT call operators that trigger undo here."""
    print(f"Count changed to {self.count}")
    # Safe: modify OTHER properties
    if self.count > 10:
        self.warning_text = "High count may be slow"

class MyProperties(bpy.types.PropertyGroup):
    count: bpy.props.IntProperty(
        name="Count",
        default=5,
        update=on_count_changed,
    )
    warning_text: bpy.props.StringProperty()
```

### Sources
- https://docs.blender.org/api/current/bpy.props.html
- https://docs.blender.org/api/current/bpy.types.PropertyGroup.html

---

## 6. UI Panels

### Panel Definition

```python
class MY_PT_main_panel(bpy.types.Panel):
    bl_label = "My Panel"               # Panel header text
    bl_idname = "MY_PT_main_panel"      # Unique identifier
    bl_space_type = 'VIEW_3D'           # Editor type
    bl_region_type = 'UI'               # Region within editor
    bl_category = "My Tab"              # Sidebar tab name
    bl_context = "objectmode"           # Only show in this mode (optional)
    bl_options = {'DEFAULT_CLOSED'}     # Start collapsed (optional)

    @classmethod
    def poll(cls, context):
        """Control when panel is visible"""
        return context.active_object is not None

    def draw_header(self, context):
        """Draw in panel header (e.g., enable checkbox)"""
        self.layout.prop(context.scene.my_props, "enabled", text="")

    def draw(self, context):
        layout = self.layout
        scene = context.scene
        obj = context.active_object

        # Properties
        layout.prop(scene.my_props, "count")
        layout.prop(obj, "name")

        # Operators
        layout.operator("my.simple_operator")

        # Labels
        layout.label(text="Info text", icon='INFO')
```

### Common bl_space_type Values

| Space Type | Editor |
|-----------|--------|
| `'VIEW_3D'` | 3D Viewport |
| `'PROPERTIES'` | Properties editor |
| `'OUTLINER'` | Outliner |
| `'NODE_EDITOR'` | Node editor (shader, compositor, geometry) |
| `'TEXT_EDITOR'` | Text editor |
| `'IMAGE_EDITOR'` | UV/Image editor |
| `'SEQUENCE_EDITOR'` | Video Sequencer |
| `'CLIP_EDITOR'` | Movie Clip editor |
| `'PREFERENCES'` | Preferences window |

### Common bl_region_type Values

| Region Type | Location |
|------------|---------|
| `'UI'` | Sidebar (N-panel) |
| `'TOOLS'` | Tool shelf (T-panel) |
| `'HEADER'` | Header bar |
| `'WINDOW'` | Main area (Properties editor panels) |
| `'TOOL_PROPS'` | Active tool properties |

### UILayout API

```python
def draw(self, context):
    layout = self.layout

    # Flow control
    layout.use_property_split = True     # Split labels from values
    layout.use_property_decorate = True  # Show keyframe buttons

    # Rows (horizontal)
    row = layout.row(align=True)
    row.prop(obj, "location", index=0, text="X")
    row.prop(obj, "location", index=1, text="Y")
    row.prop(obj, "location", index=2, text="Z")

    # Columns (vertical)
    col = layout.column(align=True)
    col.prop(props, "setting_a")
    col.prop(props, "setting_b")

    # Box (bordered container)
    box = layout.box()
    box.label(text="Section Title")
    box.prop(props, "option")

    # Split (percentage-based columns)
    split = layout.split(factor=0.3)
    col1 = split.column()
    col2 = split.column()
    col1.label(text="Label:")
    col2.prop(props, "value", text="")

    # Separator
    layout.separator()

    # Operator with properties
    op = layout.operator("my.operator", text="Custom Text", icon='MESH_CUBE')
    op.size = 2.0  # Set operator property

    # Enabled/disabled state
    row = layout.row()
    row.enabled = props.enabled
    row.operator("my.action")

    # Alert state (red highlight)
    row = layout.row()
    row.alert = True
    row.label(text="Warning!", icon='ERROR')

    # Template for list widget
    layout.template_list("MY_UL_items", "", props, "items", props, "active_item_index")
```

### Sub-Panels (Panel Hierarchy)

```python
class MY_PT_parent(bpy.types.Panel):
    bl_label = "Parent Panel"
    bl_idname = "MY_PT_parent"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        self.layout.label(text="Parent content")

class MY_PT_child(bpy.types.Panel):
    bl_label = "Child Panel"
    bl_idname = "MY_PT_child"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"
    bl_parent_id = "MY_PT_parent"  # Makes this a sub-panel
    bl_options = {'DEFAULT_CLOSED'}

    def draw(self, context):
        self.layout.label(text="Child content")
```

### Sources
- https://docs.blender.org/api/current/bpy.types.Panel.html
- https://docs.blender.org/api/current/bpy.types.UILayout.html

---

## 7. Mesh & BMesh

### Direct Mesh Data Access (Object Mode)

```python
import bpy

# Access existing mesh
obj = bpy.context.active_object
mesh = obj.data  # bpy.types.Mesh

# Read vertices
for v in mesh.vertices:
    print(v.co)     # mathutils.Vector
    print(v.normal)  # Read-only normal
    print(v.index)   # Vertex index

# Read polygons (faces)
for p in mesh.polygons:
    print(p.vertices[:])     # Vertex indices
    print(p.normal)          # Face normal
    print(p.center)          # Face center
    print(p.loop_start)      # Start index in loop array
    print(p.loop_total)      # Number of loops (= vertex count)

# Read loops (vertex-face corners, for UV/vertex colors)
for l in mesh.loops:
    print(l.vertex_index)    # Which vertex
    print(l.normal)          # Loop normal (smooth shading)

# UV access
if mesh.uv_layers:
    uv_layer = mesh.uv_layers.active.data
    for loop_idx, loop in enumerate(mesh.loops):
        uv = uv_layer[loop_idx].uv
        print(f"Vertex {loop.vertex_index}: UV {uv}")
```

### Creating Mesh from Data

```python
import bpy

# Method 1: from_pydata (simplest)
mesh = bpy.data.meshes.new("MyMesh")
verts = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
edges = []  # Empty = auto-generate from faces
faces = [(0, 1, 2, 3)]
mesh.from_pydata(verts, edges, faces)
mesh.update()  # ALWAYS call after from_pydata

obj = bpy.data.objects.new("MyObject", mesh)
bpy.context.collection.objects.link(obj)

# Method 2: Pre-allocate arrays (faster for large meshes)
import numpy as np

vert_count = 1000
face_count = 998

mesh = bpy.data.meshes.new("FastMesh")
mesh.vertices.add(vert_count)
mesh.loops.add(face_count * 3)       # Triangles: 3 loops per face
mesh.polygons.add(face_count)

# Use foreach_set for bulk data (10-100x faster than per-element)
coords = np.random.rand(vert_count * 3).astype(np.float32)
mesh.vertices.foreach_set("co", coords)

# Loop indices
loop_verts = np.array([...], dtype=np.int32)  # Vertex indices for each loop
mesh.loops.foreach_set("vertex_index", loop_verts)

# Polygon loop start/total
loop_starts = np.arange(0, face_count * 3, 3, dtype=np.int32)
loop_totals = np.full(face_count, 3, dtype=np.int32)
mesh.polygons.foreach_set("loop_start", loop_starts)
mesh.polygons.foreach_set("loop_total", loop_totals)

mesh.update()
mesh.validate()
```

### BMesh (Edit-Mode Mesh API)

```python
import bmesh
import bpy

# Method 1: From existing mesh (Object Mode)
mesh = bpy.context.active_object.data
bm = bmesh.new()
bm.from_mesh(mesh)

# Ensure lookup tables (REQUIRED before index access)
bm.verts.ensure_lookup_table()
bm.edges.ensure_lookup_table()
bm.faces.ensure_lookup_table()

# Access elements
vert = bm.verts[0]
print(vert.co)

# Create geometry
v1 = bm.verts.new((0, 0, 0))
v2 = bm.verts.new((1, 0, 0))
v3 = bm.verts.new((1, 1, 0))
face = bm.faces.new((v1, v2, v3))

# Write back to mesh
bm.to_mesh(mesh)
bm.free()  # ALWAYS free BMesh when done

# Method 2: From edit mode (Edit Mode)
obj = bpy.context.edit_object
bm = bmesh.from_edit_mesh(obj.data)

# Modify...
bm.verts.new((0, 0, 0))

# Update edit mesh (no free needed)
bmesh.update_edit_mesh(obj.data)
```

### Performance Guidelines

| Operation | Method | Speed |
|-----------|--------|-------|
| Read vertices (< 1000) | `for v in mesh.vertices` | Fast enough |
| Read vertices (> 1000) | `mesh.vertices.foreach_get("co", array)` | 10-100x faster |
| Write vertices | `mesh.vertices.foreach_set("co", array)` | 10-100x faster |
| Create mesh | `mesh.from_pydata()` | Good for simple meshes |
| Create large mesh | Pre-allocate + `foreach_set` | Fastest |
| Edit operations | BMesh | Best for topology changes |
| Incremental edits | BMesh from edit mode | Best for interactive |

**CRITICAL**: NEVER access `mesh.vertices`, `mesh.polygons` etc. in Edit Mode. The data is not synchronized. Either exit edit mode first or use `bmesh.from_edit_mesh()`.

### Sources
- https://docs.blender.org/api/current/bmesh.html
- https://docs.blender.org/api/current/bmesh.types.html
- https://docs.blender.org/api/current/bpy.types.Mesh.html

---

## 8. Modifiers

### Modifier Stack Access

```python
obj = bpy.context.active_object

# Add modifier
mod = obj.modifiers.new(name="MySubsurf", type='SUBSURF')
mod.levels = 2
mod.render_levels = 3

# Access existing modifier
mod = obj.modifiers["MySubsurf"]
print(mod.type)  # 'SUBSURF'

# Iterate modifiers
for mod in obj.modifiers:
    print(f"{mod.name}: {mod.type}")

# Reorder
bpy.ops.object.modifier_move_up(modifier="MySubsurf")
# Or in 4.0+:
with bpy.context.temp_override(object=obj):
    bpy.ops.object.modifier_move_to_index(modifier="MySubsurf", index=0)

# Apply modifier (Object mode only)
with bpy.context.temp_override(object=obj):
    bpy.ops.object.modifier_apply(modifier="MySubsurf")

# Remove modifier
obj.modifiers.remove(mod)
```

### Common Modifier Types

| Type String | Modifier | Category |
|------------|----------|----------|
| `'SUBSURF'` | Subdivision Surface | Generate |
| `'MIRROR'` | Mirror | Generate |
| `'ARRAY'` | Array | Generate |
| `'BOOLEAN'` | Boolean | Generate |
| `'SOLIDIFY'` | Solidify | Generate |
| `'BEVEL'` | Bevel | Generate |
| `'DECIMATE'` | Decimate | Generate |
| `'SHRINKWRAP'` | Shrinkwrap | Deform |
| `'ARMATURE'` | Armature | Deform |
| `'NODES'` | Geometry Nodes | Generate |
| `'MESH_DEFORM'` | Mesh Deform | Deform |
| `'LATTICE'` | Lattice | Deform |
| `'CURVE'` | Curve | Deform |

### Geometry Nodes Modifier

```python
# Add Geometry Nodes modifier
mod = obj.modifiers.new(name="GeoNodes", type='NODES')

# Assign a node group
mod.node_group = bpy.data.node_groups["My Node Group"]

# Access input parameters
# Inputs are stored as modifier properties with names like "Socket_0", "Socket_1"
# The naming depends on the node group's interface
for item in mod.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        socket_id = item.identifier    # e.g., "Socket_0"
        # Access value:
        value = mod[socket_id]
        print(f"{item.name}: {value}")

# Set input value
mod["Socket_0"] = 5.0

# Force update after changing values
obj.data.update()
# Or:
bpy.context.view_layer.update()
```

### Getting Evaluated (Modified) Mesh

```python
# Get mesh AFTER all modifiers applied (read-only)
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()

# Read evaluated data
print(f"Original verts: {len(obj.data.vertices)}")
print(f"Evaluated verts: {len(mesh_eval.vertices)}")

# ALWAYS clean up
obj_eval.to_mesh_clear()
```

### Sources
- https://docs.blender.org/api/current/bpy.types.NodesModifier.html
- https://docs.blender.org/api/current/bpy.types.ObjectModifiers.html

---

## 9. Context System

### Context Basics

`bpy.context` provides read-only access to the current Blender state. It changes based on where code is executed (which editor, which mode, what's selected).

```python
# Common context members
context = bpy.context
context.scene            # Active scene
context.view_layer       # Active view layer
context.collection       # Active collection
context.object           # Active object (alias for active_object)
context.active_object    # Active object
context.selected_objects # List of selected objects
context.mode             # 'OBJECT', 'EDIT_MESH', 'SCULPT', etc.
context.area             # Current editor area (None in background)
context.region           # Current region within area
context.space_data       # Editor-specific data
context.window           # Active window
context.screen           # Active screen layout
context.preferences      # User preferences
```

### Context Override with temp_override (Blender 3.2+, REQUIRED in 4.0+)

```python
# Override active object for an operator
with bpy.context.temp_override(active_object=obj, selected_objects=[obj]):
    bpy.ops.object.shade_smooth()

# Override area type (e.g., to run 3D viewport operator from script)
for window in bpy.context.window_manager.windows:
    for area in window.screen.areas:
        if area.type == 'VIEW_3D':
            with bpy.context.temp_override(window=window, area=area):
                bpy.ops.view3d.camera_to_view()
            break

# RULES for temp_override:
# 1. Region must belong to the specified (or current) area
# 2. Area must belong to the specified (or current) window
# 3. Cannot switch to/from fullscreen areas
# 4. Use logging_set(True) to debug which context members an operator needs
```

### Restricted Context

Some Blender callbacks have restricted context — certain operations are forbidden:

| Callback | Restriction | What Fails |
|---------|-------------|------------|
| `draw()` in Panel/Menu | No data modification | `bpy.ops.*`, property changes |
| `depsgraph_update_post` handler | No scene modifications | Adding/removing objects |
| `render_pre/post` handler | Limited modifications | Most `bpy.ops.*` |
| `load_post` handler | Data may not be ready | Accessing specific objects |
| Timer callbacks | No UI access | `bpy.ops.*` that need area |

```python
# WRONG: Modifying data in draw callback
class MY_PT_bad(bpy.types.Panel):
    def draw(self, context):
        # This WILL cause errors or undefined behavior:
        context.scene.my_prop = 5  # FORBIDDEN in draw()
        bpy.ops.object.select_all()  # FORBIDDEN in draw()

        # This is fine:
        self.layout.prop(context.scene, "my_prop")  # Reading for UI is OK
```

### Application Handlers

```python
from bpy.app.handlers import persistent

@persistent  # Survives file load
def on_load_post(dummy):
    """Called after a .blend file is loaded"""
    print("File loaded!")

@persistent
def on_depsgraph_update(scene, depsgraph):
    """Called after dependency graph update"""
    for update in depsgraph.updates:
        print(f"Updated: {update.id.name}")

# Register handlers
bpy.app.handlers.load_post.append(on_load_post)
bpy.app.handlers.depsgraph_update_post.append(on_depsgraph_update)

# Unregister (in addon unregister)
bpy.app.handlers.load_post.remove(on_load_post)
bpy.app.handlers.depsgraph_update_post.remove(on_depsgraph_update)
```

### Sources
- https://docs.blender.org/api/current/bpy.types.Context.html
- https://docs.blender.org/api/current/bpy.ops.html
- https://devtalk.blender.org/t/deprecationwarning-passing-in-context-overrides-is-deprecated-in-favor-of-context-temp-override/27870

---

## 10. Dependency Graph (Depsgraph)

The dependency graph manages evaluation order and caching of all scene data.

### Core Concepts

- **Original data**: What the user edits (`bpy.data.objects["Cube"]`)
- **Evaluated data**: Result after modifiers, constraints, drivers, particles
- **Object instances**: Includes particle instances, collection instances, geometry nodes instances

### Basic Usage

```python
import bpy

depsgraph = bpy.context.evaluated_depsgraph_get()

# Get evaluated version of an object
obj = bpy.context.active_object
obj_eval = obj.evaluated_get(depsgraph)

# Get evaluated mesh (with all modifiers applied)
mesh_eval = obj_eval.to_mesh()
print(f"Evaluated vertex count: {len(mesh_eval.vertices)}")
obj_eval.to_mesh_clear()  # Clean up temporary mesh

# Original from evaluated
original = obj_eval.original  # Back to original data
```

### Iterating All Object Instances

```python
depsgraph = bpy.context.evaluated_depsgraph_get()

for instance in depsgraph.object_instances:
    obj = instance.object          # The evaluated object
    is_instance = instance.is_instance  # True for particle/collection instances
    matrix = instance.matrix_world      # World-space transform

    if is_instance:
        parent = instance.parent        # Parent object that creates instances

    print(f"{'Instance' if is_instance else 'Object'}: {obj.name} at {matrix.translation}")
```

### Depsgraph Updates (Handler)

```python
from bpy.app.handlers import persistent

@persistent
def on_depsgraph_update(scene, depsgraph):
    for update in depsgraph.updates:
        # update.id: the ID datablock that was updated
        # update.is_updated_geometry: mesh/curve data changed
        # update.is_updated_transform: location/rotation/scale changed
        # update.is_updated_shading: material/shader changed
        if update.is_updated_geometry:
            print(f"Geometry updated: {update.id.name}")

bpy.app.handlers.depsgraph_update_post.append(on_depsgraph_update)
```

### Key Rules

1. **ALWAYS use `evaluated_get(depsgraph)`** to get post-modifier data. Do NOT read `obj.data.vertices` expecting modified results.
2. **ALWAYS call `to_mesh_clear()`** after `to_mesh()` to prevent memory leaks.
3. **Depsgraph is read-only.** You cannot modify evaluated data. Modify originals, then let depsgraph re-evaluate.
4. **`evaluated_depsgraph_get()` may trigger evaluation.** This can be expensive if scene has pending changes.
5. In Blender 4.0+, `bmesh.from_object()` and `BVHTree.FromObject()` REQUIRE the depsgraph parameter.

### Sources
- https://docs.blender.org/api/current/bpy.types.Depsgraph.html
- https://docs.blender.org/api/current/bpy.types.DepsgraphObjectInstance.html
- https://developer.blender.org/docs/features/core/depsgraph/

---

## 11. Common Error Patterns

### Error 1: Context Is Incorrect (RuntimeError)

```python
# WRONG: Running viewport operator without correct area context
bpy.ops.view3d.snap_cursor_to_center()
# RuntimeError: Operator bpy.ops.view3d.snap_cursor_to_center.poll() failed, context is incorrect

# FIX: Provide correct context
for area in bpy.context.screen.areas:
    if area.type == 'VIEW_3D':
        with bpy.context.temp_override(area=area):
            bpy.ops.view3d.snap_cursor_to_center()
        break
```

**Root cause**: Operators check `poll()` which requires specific area types, active objects, or modes. Scripts executing outside the expected editor context will fail.

### Error 2: Accessing Mesh Data in Wrong Mode

```python
# WRONG: Accessing mesh vertices while in Edit Mode
bpy.ops.object.mode_set(mode='EDIT')
verts = bpy.context.active_object.data.vertices
for v in verts:
    print(v.co)  # Data is stale / may crash

# FIX: Exit edit mode first, or use BMesh
bpy.ops.object.mode_set(mode='OBJECT')
for v in bpy.context.active_object.data.vertices:
    print(v.co)  # Now safe

# OR use BMesh in edit mode:
import bmesh
bm = bmesh.from_edit_mesh(bpy.context.edit_object.data)
for v in bm.verts:
    print(v.co)  # Safe
```

### Error 3: Stale References After Data Modification

```python
# WRONG: Keeping reference after re-allocation
items = bpy.context.scene.my_collection
first_item = items.add()
first_item.name = "First"

for i in range(100):
    items.add()  # Re-allocates the entire array

first_item.name = "Updated"  # CRASH — first_item pointer is now invalid

# FIX: Re-fetch after modifications
items = bpy.context.scene.my_collection
items.add()
for i in range(100):
    items.add()
first_item = items[0]  # Re-fetch reference
first_item.name = "Updated"  # Safe
```

### Error 4: Name Collision Assumption

```python
# WRONG: Assuming data gets the exact name you request
bpy.data.meshes.new(name="MyMesh")
mesh = bpy.data.meshes["MyMesh"]  # May be "MyMesh.001" if "MyMesh" already existed!

# FIX: Store the reference directly
mesh = bpy.data.meshes.new(name="MyMesh")
# Use 'mesh' variable, never look up by name
```

### Error 5: Undo Invalidates All References

```python
# WRONG: Holding references across undo operations (in modal operators)
class MY_OT_bad_modal(bpy.types.Operator):
    _target_obj = None

    def invoke(self, context, event):
        self._target_obj = context.active_object  # Stored reference
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        # After user presses Ctrl+Z:
        self._target_obj.location.x += 1  # CRASH — reference invalidated

# FIX: Store name/identifier, re-fetch each frame
class MY_OT_safe_modal(bpy.types.Operator):
    _target_name = ""

    def invoke(self, context, event):
        self._target_name = context.active_object.name
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        obj = bpy.data.objects.get(self._target_name)
        if obj is None:
            return {'CANCELLED'}
        obj.location.x += 1  # Safe — fresh reference
        return {'RUNNING_MODAL'}
```

### Error 6: Threading Violations

```python
# WRONG: Modifying Blender data from a thread
import threading

def background_work():
    bpy.data.objects["Cube"].location.x = 5.0  # UNDEFINED BEHAVIOR / CRASH

thread = threading.Thread(target=background_work)
thread.start()

# FIX: Do computation in thread, apply results in main thread
import queue

result_queue = queue.Queue()

def background_compute():
    result = heavy_computation()
    result_queue.put(result)

thread = threading.Thread(target=background_compute)
thread.start()

# Apply in timer or modal (main thread):
def apply_result():
    if not result_queue.empty():
        result = result_queue.get()
        bpy.data.objects["Cube"].location.x = result
        return None  # Stop timer
    return 0.1  # Check again in 0.1s

bpy.app.timers.register(apply_result)
```

### Error 7: Modifying Data in draw() Callback

```python
# WRONG: Changing properties inside Panel.draw()
class MY_PT_bad(bpy.types.Panel):
    bl_label = "Bad Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'

    def draw(self, context):
        # This causes infinite redraw loops or errors:
        context.scene.frame_current += 1  # FORBIDDEN
        bpy.ops.object.select_all(action='SELECT')  # FORBIDDEN

        self.layout.label(text="This panel causes problems")

# FIX: Only read data and build UI in draw(). Use operators for modifications.
```

### Sources
- https://docs.blender.org/api/blender2.8/info_gotcha.html
- https://docs.blender.org/api/current/info_gotcha.html
- https://devtalk.blender.org/t/context-incorrect-or-active-object-missing/25843

---

## 12. AI Common Mistakes

This section documents patterns that Claude and other LLMs frequently get wrong when generating Blender Python code.

### Mistake 1: Using Removed Context Override Syntax

**What Claude does wrong**: Generates Blender 3.x context override dict syntax for Blender 4.0+ code.

```python
# WRONG (Claude generates this frequently)
override = bpy.context.copy()
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="MyMod")

# CORRECT for Blender 4.0+
with bpy.context.temp_override(active_object=obj, object=obj):
    bpy.ops.object.modifier_apply(modifier="MyMod")
```

**Why**: Claude's training data contains vastly more 3.x examples than 4.x examples.

### Mistake 2: Not Calling ensure_lookup_table() on BMesh

**What Claude does wrong**: Accesses BMesh elements by index without calling `ensure_lookup_table()`.

```python
# WRONG
bm = bmesh.new()
bm.from_mesh(mesh)
v = bm.verts[0]  # IndexError or undefined behavior

# CORRECT
bm = bmesh.new()
bm.from_mesh(mesh)
bm.verts.ensure_lookup_table()  # REQUIRED before index access
v = bm.verts[0]
```

### Mistake 3: Not Freeing BMesh

**What Claude does wrong**: Creates BMesh without calling `bm.free()`, causing memory leaks.

```python
# WRONG (memory leak)
bm = bmesh.new()
bm.from_mesh(mesh)
# ... do work ...
bm.to_mesh(mesh)
# Missing bm.free()!

# CORRECT
bm = bmesh.new()
bm.from_mesh(mesh)
# ... do work ...
bm.to_mesh(mesh)
bm.free()  # ALWAYS free non-edit-mode BMesh
```

### Mistake 4: Using Removed/Renamed API

**What Claude does wrong**: Uses API from older Blender versions.

```python
# WRONG (removed in 4.0)
mesh.calc_normals()  # REMOVED — normals auto-calculated
bone.layers[0] = True  # REMOVED — use bone.collections
obj.face_maps  # REMOVED — use integer attributes

# WRONG (renamed in 2.80+)
bpy.data.lamps  # It's bpy.data.lights
bpy.context.scene.render.layers  # It's bpy.context.scene.view_layers
obj.select = True  # It's obj.select_set(True)
context.scene.objects.active = obj  # It's context.view_layer.objects.active = obj
```

### Mistake 5: Forgetting mesh.update() After from_pydata()

```python
# WRONG
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata(verts, edges, faces)
# Missing mesh.update()!
obj = bpy.data.objects.new("Object", mesh)
# Mesh may display incorrectly, normals wrong

# CORRECT
mesh = bpy.data.meshes.new("Mesh")
mesh.from_pydata(verts, edges, faces)
mesh.update()  # ALWAYS call after from_pydata
obj = bpy.data.objects.new("Object", mesh)
```

### Mistake 6: Not Linking Object to Collection

```python
# WRONG (object created but invisible)
mesh = bpy.data.meshes.new("Mesh")
obj = bpy.data.objects.new("Object", mesh)
# Object exists in bpy.data but is not in any collection = invisible

# CORRECT
mesh = bpy.data.meshes.new("Mesh")
obj = bpy.data.objects.new("Object", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED to make visible
```

### Mistake 7: Using bpy.ops When Direct Data Access Is Better

```python
# WRONG (slow, fragile, context-dependent)
bpy.ops.object.select_all(action='DESELECT')
for obj_name in object_names:
    bpy.data.objects[obj_name].select_set(True)
    bpy.context.view_layer.objects.active = bpy.data.objects[obj_name]
    bpy.ops.object.location_clear()

# CORRECT (fast, reliable, no context issues)
for obj_name in object_names:
    obj = bpy.data.objects[obj_name]
    obj.location = (0, 0, 0)
```

**Rule of thumb**: Use `bpy.ops` ONLY when you need undo support, user-facing actions, or functionality not available via direct data access.

### Mistake 8: Dynamic EnumProperty Items Garbage Collection

```python
# WRONG (items list gets garbage collected → crash or empty dropdown)
def get_items(self, context):
    return [(obj.name, obj.name, "") for obj in bpy.data.objects]

my_enum: bpy.props.EnumProperty(items=get_items)

# CORRECT (keep reference alive)
_cached_items = []
def get_items(self, context):
    global _cached_items
    _cached_items = [(obj.name, obj.name, "") for obj in bpy.data.objects]
    return _cached_items

my_enum: bpy.props.EnumProperty(items=get_items)
```

### Mistake 9: Incorrect Register/Unregister Order

```python
# WRONG (PropertyGroup used before registered)
class MySettings(bpy.types.PropertyGroup):
    value: bpy.props.FloatProperty()

class MY_OT_op(bpy.types.Operator):
    bl_idname = "my.op"
    bl_label = "Op"

    def execute(self, context):
        print(context.scene.my_settings.value)
        return {'FINISHED'}

def register():
    bpy.utils.register_class(MY_OT_op)
    bpy.utils.register_class(MySettings)  # TOO LATE — MySettings needed by PointerProperty
    bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MySettings)

# CORRECT order
def register():
    bpy.utils.register_class(MySettings)  # FIRST — register types before they're referenced
    bpy.utils.register_class(MY_OT_op)
    bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MySettings)

def unregister():
    del bpy.types.Scene.my_settings
    bpy.utils.unregister_class(MY_OT_op)
    bpy.utils.unregister_class(MySettings)  # LAST — unregister in reverse order
```

### Mistake 10: Generating Blender 2.7x Code Patterns

```python
# WRONG (pre-2.80 patterns that Claude still generates)
bpy.context.scene.objects.link(obj)       # 2.7x → use collection.objects.link()
bpy.context.scene.objects.active = obj    # 2.7x → view_layer.objects.active = obj
obj.select = True                          # 2.7x → obj.select_set(True)
bpy.context.scene.render.layers           # 2.7x → context.scene.view_layers
mesh.uv_textures                          # 2.7x → mesh.uv_layers
bpy.utils.register_module(__name__)       # 2.7x → removed, register classes manually
bpy.data.lamps                            # 2.7x → bpy.data.lights
```

### Summary: Version-Specific Code Generation Rules

| Target Version | ALWAYS | NEVER |
|---------------|--------|-------|
| Blender 3.6+ | Use `context.temp_override()` | Use dict context overrides |
| Blender 4.0+ | Use attribute-based mesh data | Use `MeshEdge.bevel_weight`/`.crease` |
| Blender 4.0+ | Use `NodeTree.interface` | Use `NodeTree.inputs/outputs.new()` |
| Blender 4.0+ | Use bone collections | Use `bone.layers` or `bone_groups` |
| Blender 4.2+ | Use `blender_manifest.toml` for new addons | Assume `bl_info` is the only option |
| Blender 5.0+ | Use `gpu` module for drawing | Use `bgl` module |
| ANY version | Call `mesh.update()` after `from_pydata()` | Skip mesh update |
| ANY version | Call `bm.free()` after BMesh use | Leave BMesh in memory |
| ANY version | Use `obj.select_set(True)` | Use `obj.select = True` |
| ANY version | Link objects to collections | Create objects without linking |

---

## Research Quality Assessment

| Criterion | Status |
|-----------|--------|
| Official docs as primary source | Yes — docs.blender.org, developer.blender.org |
| Code examples tested/validated | Based on official examples and community-verified patterns |
| Version-explicit information | Yes — 3.x, 4.0, 4.2, 5.0 matrix included |
| Anti-patterns documented | Yes — 10 AI-specific mistakes + 7 common error patterns |
| Deterministic language | Yes — ALWAYS/NEVER/REQUIRED used throughout |
| Minimum 2000 words | Yes — exceeds requirement |

---

## 13. Real-World Usage: OpenAEC Projects

The OpenAEC Foundation maintains several repositories relevant to the Blender-Bonsai-IfcOpenShell-Sverchok skill package. Two repositories are directly referenced: GIS-to-Blender and aec-scripts. A third related repository, building-py, provides additional context for Blender integration within the OpenAEC ecosystem.

### GIS-to-Blender (3D Environment Automation)

**Repository**: https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
**Purpose**: Prompt LLMs (primarily Claude Code) to build 3D environments in Blender from GIS data.
**Blender Version**: Not yet specified (repository is in early stage).
**Status**: Pre-development. The repository contains only a README and LICENSE (LGPL 3.0) as of February 2026. No Python code exists yet.

#### Key Findings

The repository's README states: *"With this repo you can prompt LLM's to build a 3D environment in Blender. Mainly built for Claude Code."*

This repository is the **direct target use case** for the skill package being developed. It represents the intended consumer of Claude skills that automate Blender via bpy. The fact that it contains no code yet confirms that the skill package MUST provide the foundational bpy patterns that this project will rely on.

#### Architecture Notes

- The repository is designed as an LLM-promptable project, NOT a traditional addon.
- LGPL 3.0 license is consistent with the Blender ecosystem licensing requirements.
- Created January 2026, last pushed February 2026 — active but early stage.

---

### AEC Scripts

**Repository**: https://github.com/OpenAEC-Foundation/aec-scripts
**Purpose**: Python scripts and Revit addins for AEC (Architecture, Engineering, Construction) workflows.

#### Key Finding: No bpy Usage

The aec-scripts repository contains **zero Blender/bpy code**. All Python scripts are pyRevit pushbutton scripts targeting Autodesk Revit via IronPython. The repository uses:

- `clr` (Common Language Runtime) for .NET interop
- `Autodesk.Revit.DB` for Revit API access
- `pyrevit` framework for script hosting
- `System.Windows.Forms` for UI

#### Relevant GIS/Mesh Patterns (Transferable to Blender)

Despite being Revit-targeted, the GIS2BIM pushbutton contains **CityJSON parsing and mesh creation patterns** that are directly transferable to Blender/bpy.

##### Pattern 1: CityJSON Vertex Parsing and Coordinate Transformation

```python
# From aec-scripts GIS2BIM.pushbutton/cityjson_parser.py (Revit version)
def transform_vertices(vertices, scale, translate, rd_x, rd_y):
    """Transform CityJSON integer vertices to relative meters."""
    converted = []
    for vx, vy, vz in vertices:
        x = float(vx) * scale[0] + translate[0]
        y = float(vy) * scale[1] + translate[1]
        z = float(vz) * scale[2] + translate[2]
        converted.append((x - rd_x, y - rd_y, z))
    return converted
```

The Blender equivalent ALWAYS uses `bpy.data.meshes.new()` and `mesh.from_pydata()`:

```python
# Blender equivalent pattern (Blender 4.x+)
import bpy

def cityjson_to_blender_mesh(name, vertices, faces):
    """Create a Blender mesh from CityJSON vertices and faces."""
    mesh = bpy.data.meshes.new(name)
    mesh.from_pydata(vertices, [], faces)  # Blender 2.80+
    mesh.update()
    mesh.validate()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj
```

##### Pattern 2: Fan Triangulation (NOT Needed in Blender)

In Blender, fan triangulation is NEVER necessary when using `from_pydata()` because Blender natively handles n-gons:

```python
# Blender 4.x+ — n-gons are natively supported
mesh = bpy.data.meshes.new("building")
mesh.from_pydata(vertices, [], polygon_faces)  # polygon_faces can have 3+ vertices
mesh.update()
```

##### Pattern 3: CityJSON LOD Selection Strategy

The GIS2BIM script implements a LOD preference hierarchy for 3D BAG data: LOD 2.2 > 1.3 > 1.2. This strategy ALWAYS applies regardless of target application:

```python
# LOD preference for 3D BAG data (application-independent)
LOD_PREFERENCE = ['2.2', '1.3', '1.2']

for geom in obj.get('geometry', []):
    lod_str = str(geom.get('lod', ''))
    if lod_str in LOD_PREFERENCE:
        if best_lod is None or LOD_PREFERENCE.index(lod_str) < LOD_PREFERENCE.index(best_lod):
            best_geom = geom
            best_lod = lod_str
```

##### Pattern 4: Batch Building Import with Error Isolation

```python
# Blender 4.x+ batch import pattern
import bpy

def import_buildings_batch(buildings, collection_name="3D BAG"):
    """Import multiple CityJSON buildings with per-building error isolation."""
    collection = bpy.data.collections.new(collection_name)
    bpy.context.scene.collection.children.link(collection)

    count = 0
    skipped = 0
    for building in buildings:
        try:
            verts = building['vertices']
            faces = building['polygon_faces']
            obj_id = building.get('id', 'unknown')

            mesh = bpy.data.meshes.new(f"3DBAG_{obj_id}")
            mesh.from_pydata(verts, [], faces)
            mesh.update()

            obj = bpy.data.objects.new(f"3DBAG_{obj_id}", mesh)
            collection.objects.link(obj)
            count += 1
        except Exception as e:
            skipped += 1
            print(f"Skipped {building.get('id', '?')}: {e}")

    return count, skipped
```

---

### Related: building-py Library

**Repository**: https://github.com/OpenAEC-Foundation/building-py
**Purpose**: Python library for creating buildings with export to multiple programs including Blender, Revit, FreeCAD, and Speckle.

- The README explicitly lists **Blender** as a target export platform.
- As of March 2026, the Blender exchange module does **NOT yet exist** (only FreeCAD, Revit, Speckle, IFC, DXF modules are implemented).
- The skill package being developed will directly enable this planned integration.

---

### Cross-Cutting Patterns

#### 1. GIS-to-3D Pipeline Architecture

Both repositories demonstrate a consistent GIS-to-3D pipeline applicable to Blender:

1. **Data acquisition**: HTTP requests to Dutch GIS APIs (3D BAG, PDOK, WFS/WMS services)
2. **Coordinate transformation**: RD (Rijksdriehoekscoordinaten) to relative project coordinates
3. **Geometry parsing**: CityJSON/GeoJSON to vertices + faces
4. **Mesh creation**: Vertices + faces to application-specific mesh objects
5. **Error-tolerant batch processing**: Per-object try/except with skip-and-continue

The Blender equivalent of step 4 ALWAYS uses `bpy.data.meshes.new()` + `mesh.from_pydata()`.

#### 2. No Existing bpy Code in Either Repository

Neither GIS-to-Blender nor aec-scripts contains any `import bpy` statements. This means:
- The skill package MUST define the canonical bpy patterns for the OpenAEC ecosystem from scratch.
- The GIS-to-Blender repo is explicitly waiting for LLM-generated Blender code (Claude Code).

#### 3. Licensing Consistency

Both repositories use **LGPL 3.0**, which is compatible with Blender's GPL licensing. Any bpy code generated by the skill package MUST also be GPL/LGPL compatible.

#### 4. Dutch AEC Domain Context

The OpenAEC projects focus on Dutch AEC workflows with specific data sources:
- **3D BAG** (3D Basisregistratie Adressen en Gebouwen): Dutch building registry with LOD 2.2 CityJSON
- **PDOK** (Publieke Dienstverlening Op de Kaart): Dutch geodata platform
- **BGT** (Basisregistratie Grootschalige Topografie): Large-scale topography
- **Kadaster**: Dutch land registry

The skill package MUST understand these data sources because Claude Code will be asked to create Blender scripts that import this Dutch GIS data.

### Sources
- https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
- https://github.com/OpenAEC-Foundation/aec-scripts
- https://github.com/OpenAEC-Foundation/building-py

---

## Sources

### Official Documentation
- Blender Python API Reference: https://docs.blender.org/api/current/
- Blender Python API Quickstart: https://docs.blender.org/api/current/info_quickstart.html
- Blender Python API Overview: https://docs.blender.org/api/current/info_overview.html
- Blender Python API Gotchas: https://docs.blender.org/api/current/info_gotcha.html
- Blender Python API Best Practices: https://docs.blender.org/api/current/info_best_practice.html
- BMesh Module: https://docs.blender.org/api/current/bmesh.html

### Release Notes (Breaking Changes)
- Release Notes Index: https://developer.blender.org/docs/release_notes/
- Blender 4.0 Python API: https://developer.blender.org/docs/release_notes/4.0/python_api/
- Blender 4.1 Python API: https://developer.blender.org/docs/release_notes/4.1/python_api/
- Blender 4.2 Python API: https://developer.blender.org/docs/release_notes/4.2/python_api/
- Blender 4.3 Python API: https://developer.blender.org/docs/release_notes/4.3/python_api/
- Blender 5.0 Python API: https://developer.blender.org/docs/release_notes/5.0/python_api/
- Python API Changelog: https://docs.blender.org/api/current/change_log.html
- Blender 4.0 Breaking Changes List: https://projects.blender.org/blender/blender/issues/105523
- Grease Pencil 4.3 Migration Guide: https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/

### Extension System
- Addon Tutorial (Manual): https://docs.blender.org/manual/en/latest/advanced/scripting/addon_tutorial.html
- Extension Getting Started: https://docs.blender.org/manual/en/latest/advanced/extensions/getting_started.html

### Community (Verified)
- Context Override Migration: https://b3d.interplanety.org/en/context-overriding-in-blender-3-2-and-later/
- Multi-File Addon Structure: https://b3d.interplanety.org/en/creating-multifile-add-on-for-blender/
- Context Override Deprecation Discussion: https://devtalk.blender.org/t/deprecationwarning-passing-in-context-overrides-is-deprecated-in-favor-of-context-temp-override/27870
