# Blender Version Additions for Vooronderzoek Enrichment

> Worker 2 output: Version-specific breaking changes (4.1, 4.3) and BGL-to-gpu migration guide.
> All findings are based on official Blender release notes and developer documentation.

---

## Version Matrix Rows to Insert

The following rows MUST be added to the existing version matrix in vooronderzoek-blender.md:

| Version | Python | Key Breaking Changes | Migration Impact |
|---------|--------|---------------------|------------------|
| 4.1 | 3.11 | `use_auto_smooth` / `auto_smooth_angle` removed; Light Probe types renamed (`CUBEMAP` -> `SPHERE`, `PLANAR` -> `PLANE`, `GRID` -> `VOLUME`); `displacement_method` moved from Cycles to base `Material`; `foreach_set()` now raises `TypeError` on invalid input; Layout Panels API added | HIGH for mesh/normals scripts |
| 4.3 | 3.11 | `AttributeGroup` split into type-specific classes (`AttributeGroupMesh`, `AttributeGroupPointCloud`, `AttributeGroupCurves`, `AttributeGroupGreasePencil`); Grease Pencil Python API completely rewritten; embedded ID pointer assignment raises `RuntimeError`; EEVEE legacy properties removed | HIGH for GP add-ons, MEDIUM for attribute scripts |

---

### Blender 4.1 -- Python API Changes

**Python version**: Upgraded to **Python 3.11** (VFX Platform 2024).

#### Breaking Changes

1. **Mesh Auto Smooth Removal** (HIGH impact)
   - `Mesh.use_auto_smooth` -- REMOVED in 4.1
   - `Mesh.auto_smooth_angle` -- REMOVED in 4.1
   - `Mesh.create_normals_split()` -- REMOVED in 4.1
   - `Mesh.calc_normals_split()` -- REMOVED in 4.1
   - `Mesh.free_normals_split()` -- REMOVED in 4.1
   - `MeshLoop.normal` -- now READ-ONLY in 4.1
   - **Replacement**: Use `Mesh.corner_normals` collection (auto-updated). For angle-based smoothing, use the "Smooth by Angle" modifier node group asset.
   - Custom normals MUST use `normals_split_custom_set()` or `normals_split_custom_set_from_vertices()`.

   ```python
   # BROKEN in Blender 4.1+ -- use_auto_smooth removed
   mesh.use_auto_smooth = True
   mesh.auto_smooth_angle = 0.523599  # 30 degrees

   # Blender 4.1+ -- use corner_normals instead
   # Auto smooth is now ALWAYS active; use "Smooth by Angle" modifier for angle control
   corner_normals = mesh.corner_normals  # read-only, auto-updated
   # For custom normals:
   mesh.normals_split_custom_set(normals_list)
   ```

2. **Light Probe Type Renames** (MEDIUM impact)
   - `CUBEMAP` -> `SPHERE`
   - `PLANAR` -> `PLANE`
   - `GRID` -> `VOLUME`
   - `show_data` deprecated -> use `use_data_display`

   ```python
   # BROKEN in Blender 4.1+ -- old enum values
   if probe.type == 'CUBEMAP':
       pass

   # Blender 4.1+ -- use new enum values
   if probe.type == 'SPHERE':  # Blender 4.1+
       pass
   ```

3. **Material Displacement Method** (MEDIUM impact)
   - `displacement_method` moved from `CyclesMaterialSettings` to base `Material` class.

   ```python
   # BROKEN in Blender 4.1+ -- Cycles-specific path removed
   mat.cycles.displacement_method = 'BOTH'

   # Blender 4.1+ -- property on Material directly
   mat.displacement_method = 'BOTH'  # Blender 4.1+
   ```

4. **foreach_set() Validation** (LOW impact)
   - `foreach_set()` now raises `TypeError` for invalid data types that previously failed silently.
   - Scripts that relied on silent failure MUST add proper type checking.

5. **Sequencer Transform Filter Rename** (LOW impact)
   - `SUBSAMPLING_3x3` renamed to `BOX`.

6. **Node Socket Access** (MEDIUM impact)
   - Dynamic socket types replaced static ones. Use socket **identifiers** instead of **indices** for reliable access.

#### New Features

- **Layout Panels**: `layout.panel()` and `layout.panel_prop()` for collapsible UI sections without registration.
- **Enum ID Properties**: Integer properties support enum items via `id_properties_ui()`.
- **Asset Library API**: `Preferences.filepaths.asset_libraries` for programmatic asset library management.
- **Session UIDs**: `ID.session_uid` for unique identification of datablocks within a session.
- **Shape Key Points**: `ShapeKey.points` for direct point access.
- **Translation Handler**: `handlers.translation_update_post` fires when UI language changes.
- **Shape Key Locking**: `object_report_if_active_shape_key_is_locked()` for operator safety checks.

---

### Blender 4.3 -- Python API Changes

**Python version**: **Python 3.11** (same as 4.1).

#### Breaking Changes

1. **AttributeGroup Split** (MEDIUM impact)
   - `bpy.types.AttributeGroup` -- REMOVED in 4.3
   - Replaced by type-specific classes:
     - `AttributeGroupMesh`
     - `AttributeGroupPointCloud`
     - `AttributeGroupCurves`
     - `AttributeGroupGreasePencil`
   - Mesh-only properties (`active_color`, `active_color_index`, `default_color_name`, `render_color_index`) are NOW accessible ONLY on `AttributeGroupMesh`.

   ```python
   # BROKEN in Blender 4.3+ -- generic AttributeGroup
   attrs = obj.data.attributes  # was bpy.types.AttributeGroup
   color_name = attrs.active_color_name  # AttributeError in 4.3 if not mesh

   # Blender 4.3+ -- type-specific AttributeGroup
   attrs = obj.data.attributes  # now AttributeGroupMesh for mesh objects
   color_name = attrs.active_color_name  # ONLY works on AttributeGroupMesh
   # Use domain_size() to check support:
   size = attrs.domain_size('POINT')  # returns 0 if unsupported  # Blender 4.3+
   ```

2. **Grease Pencil Python API Rewrite** (CRITICAL impact for GP add-ons)
   - The ENTIRE Grease Pencil Python API has been rewritten for the new data structure.
   - NOT backward compatible: files saved in 4.3+ do NOT load correctly in 4.2 or lower.
   - `pixel_factor` (Thickness Scale) -- REMOVED in 4.3
   - Screen Space thickness mode -- REMOVED (strokes are ALWAYS in World Space)
   - Selection order for stroke interpolation -- REMOVED
   - Draw Mode guides -- NOT ported to 4.3
   - See official migration guide: https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/

3. **Embedded ID Pointer Assignment** (LOW impact)
   - Assigning embedded IDs (e.g., `scene.collection`, root node trees) to `PointerProperty` now raises `RuntimeError`.

4. **Reroute Node Socket Changes** (LOW impact)
   - Reroute node data type changes via `socket_idname` property instead of direct socket modification.

5. **EEVEE Legacy Property Removal** (MEDIUM impact)
   - REMOVED properties: contact shadows, SSR, volumetric lighting, GTAO, bokeh, bloom, and shadow settings.
   - Scripts referencing these EEVEE properties MUST be updated.

#### New Features

- **`bpy.app.python_args`**: Call Python in Blender's environment.
- **Blend import handlers**: `blend_import_pre` and `blend_import_post` handlers with `BlendImportContext` parameter.
- **`ID.rename()`**: Complex renaming behavior (direct `ID.name` assignment unchanged).
- **`domain_size()`**: Returns attribute domain size (0 if unsupported).
- **`foreach_set()` triggers updates**: Calling `foreach_set()` on attribute data now triggers property updates.
- **Curves API**: `curves.remove_curves(indices=[])` and `curves.resize_curves(sizes, indices=[])`.
- **`uiLayout.template_search()`**: Accepts optional `text` parameter.

---

### BGL-to-gpu Migration Guide

**Context**: The `bgl` module (OpenGL wrapper) is deprecated since Blender 3.5 and is scheduled for **complete removal** when Blender fully transitions to Vulkan/Metal backends. ALL drawing code MUST migrate to the `gpu` module. The `gpu` module provides a graphics-API-independent abstraction that works across OpenGL, Vulkan, and Metal.

> **Note**: The IfcOpenShell/Bonsai project completed this migration (see [IfcOpenShell issue #2897](https://github.com/IfcOpenShell/IfcOpenShell/issues/2897)), demonstrating that this migration is essential for BIM/AEC add-ons.

#### Before (bgl -- Blender <= 4.x, BROKEN when OpenGL backend is removed)

```python
# bgl viewport overlay example -- DEPRECATED since Blender 3.5
# BROKEN on Apple M1/M2 (Metal backend), WILL BREAK on Vulkan backend
import bpy
import bgl
from gpu_extras.batch import batch_for_shader
import gpu

# Define geometry: a rectangle outline in 3D space
coords = [
    (0.0, 0.0, 0.0), (5.0, 0.0, 0.0),
    (5.0, 0.0, 0.0), (5.0, 3.0, 0.0),
    (5.0, 3.0, 0.0), (0.0, 3.0, 0.0),
    (0.0, 3.0, 0.0), (0.0, 0.0, 0.0),
]

shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')  # REMOVED in Blender 4.0+
batch = batch_for_shader(shader, 'LINES', {"pos": coords})


def draw_callback():
    # bgl OpenGL state management -- ALL of these calls are DEPRECATED
    bgl.glEnable(bgl.GL_BLEND)                    # DEPRECATED -- use gpu.state.blend_set()
    bgl.glEnable(bgl.GL_LINE_SMOOTH)               # DEPRECATED -- no direct replacement
    bgl.glLineWidth(2)                              # DEPRECATED -- use gpu.state.line_width_set()
    bgl.glEnable(bgl.GL_DEPTH_TEST)                 # DEPRECATED -- use gpu.state.depth_test_set()
    bgl.glDepthFunc(bgl.GL_LEQUAL)                  # DEPRECATED -- use gpu.state.depth_test_set()

    shader.bind()
    shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))  # Orange, 80% opacity
    batch.draw(shader)

    # Restore OpenGL state
    bgl.glDisable(bgl.GL_BLEND)
    bgl.glDisable(bgl.GL_LINE_SMOOTH)
    bgl.glLineWidth(1)
    bgl.glDisable(bgl.GL_DEPTH_TEST)


# Register the draw handler
_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_callback, (), 'WINDOW', 'POST_VIEW'
)
```

#### After (gpu -- Blender 4.0+, REQUIRED for Vulkan/Metal compatibility)

```python
# gpu viewport overlay example -- Blender 4.0+
# Works on ALL backends: OpenGL, Vulkan, Metal
import bpy
import gpu
from gpu_extras.batch import batch_for_shader  # Blender 3.0+

# Define geometry: a rectangle outline in 3D space
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
    # gpu.state -- graphics-API-independent state management (Blender 3.5+)
    gpu.state.blend_set('ALPHA')                    # Replaces bgl.glEnable(bgl.GL_BLEND)
    gpu.state.depth_test_set('LESS_EQUAL')          # Replaces bgl.glEnable(bgl.GL_DEPTH_TEST)
    gpu.state.depth_mask_set(True)                  # Replaces bgl.glDepthMask(bgl.GL_TRUE)
    gpu.state.line_width_set(2.0)                   # Replaces bgl.glLineWidth(2)

    shader.bind()
    # POLYLINE_UNIFORM_COLOR requires viewportSize and lineWidth uniforms
    shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])  # Blender 4.0+
    shader.uniform_float("lineWidth", 2.0)          # Line width in pixels  # Blender 4.0+
    shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))  # Orange, 80% opacity
    batch.draw(shader)

    # Restore gpu state to defaults
    gpu.state.blend_set('NONE')                     # Replaces bgl.glDisable(bgl.GL_BLEND)
    gpu.state.depth_test_set('NONE')                # Replaces bgl.glDisable(bgl.GL_DEPTH_TEST)
    gpu.state.depth_mask_set(False)
    gpu.state.line_width_set(1.0)


# Register the draw handler -- SAME API, no change needed here
_handle = bpy.types.SpaceView3D.draw_handler_add(
    draw_callback, (), 'WINDOW', 'POST_VIEW'
)

# To remove the handler (e.g., in addon unregister):
# bpy.types.SpaceView3D.draw_handler_remove(_handle, 'WINDOW')
```

#### API Mapping Reference

| bgl (DEPRECATED) | gpu replacement (Blender 3.5+) | Notes |
|---|---|---|
| `bgl.glEnable(bgl.GL_BLEND)` | `gpu.state.blend_set('ALPHA')` | Use `'ALPHA'`, `'ADDITIVE'`, `'ADDITIVE_PREMUL'`, or `'MULTIPLY'` |
| `bgl.glDisable(bgl.GL_BLEND)` | `gpu.state.blend_set('NONE')` | |
| `bgl.glLineWidth(w)` | `gpu.state.line_width_set(w)` | Also set `lineWidth` uniform for POLYLINE shaders |
| `bgl.glEnable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('LESS_EQUAL')` | Options: `'NONE'`, `'LESS'`, `'LESS_EQUAL'`, `'EQUAL'`, `'GREATER'`, `'GREATER_EQUAL'` |
| `bgl.glDisable(bgl.GL_DEPTH_TEST)` | `gpu.state.depth_test_set('NONE')` | |
| `bgl.glDepthMask(GL_TRUE)` | `gpu.state.depth_mask_set(True)` | |
| `bgl.glEnable(bgl.GL_LINE_SMOOTH)` | *(no direct replacement)* | Use `POLYLINE_*` shaders for antialiased lines |
| `bgl.glPointSize(s)` | `gpu.state.point_size_set(s)` | |
| `bgl.glActiveTexture(...)` / `bgl.glBindTexture(...)` | `gpu.texture.from_image(image)` + `shader.uniform_sampler("image", texture)` | Completely different API |
| `gpu.shader.from_builtin('3D_UNIFORM_COLOR')` | `gpu.shader.from_builtin('UNIFORM_COLOR')` or `'POLYLINE_UNIFORM_COLOR'` | `3D_` prefix removed in Blender 4.0 |
| `gpu.shader.from_builtin('3D_FLAT_COLOR')` | `gpu.shader.from_builtin('FLAT_COLOR')` or `'POLYLINE_FLAT_COLOR'` | `3D_` prefix removed in Blender 4.0 |
| `image.gl_load()` / `image.bindcode` | `gpu.texture.from_image(image)` | |

#### Migration Checklist

- [ ] Replace ALL `import bgl` statements -- remove entirely, no bgl imports allowed
- [ ] Replace `bgl.glEnable(bgl.GL_BLEND)` with `gpu.state.blend_set('ALPHA')`
- [ ] Replace `bgl.glDisable(bgl.GL_BLEND)` with `gpu.state.blend_set('NONE')`
- [ ] Replace `bgl.glLineWidth(n)` with `gpu.state.line_width_set(n)`
- [ ] Replace `bgl.glEnable(bgl.GL_DEPTH_TEST)` with `gpu.state.depth_test_set('LESS_EQUAL')`
- [ ] Replace `bgl.glDisable(bgl.GL_DEPTH_TEST)` with `gpu.state.depth_test_set('NONE')`
- [ ] Replace `bgl.glPointSize(n)` with `gpu.state.point_size_set(n)`
- [ ] Replace `bgl.glEnable(bgl.GL_LINE_SMOOTH)` with `POLYLINE_*` shader usage (no direct state equivalent)
- [ ] Replace `3D_UNIFORM_COLOR` with `UNIFORM_COLOR` or `POLYLINE_UNIFORM_COLOR` (Blender 4.0+)
- [ ] Replace `3D_FLAT_COLOR` with `FLAT_COLOR` or `POLYLINE_FLAT_COLOR` (Blender 4.0+)
- [ ] Replace `image.gl_load()` / `bgl.glBindTexture()` with `gpu.texture.from_image(image)` + `shader.uniform_sampler()`
- [ ] ALWAYS restore `gpu.state` to defaults at end of draw callbacks (blend, depth, line width)
- [ ] Test on Apple Silicon (Metal backend) -- bgl is ALREADY non-functional there
- [ ] Verify `POLYLINE_*` shaders set `viewportSize` and `lineWidth` uniforms

---

### Missing Source URLs to Add

These URLs MUST be added to the "Release Notes (Breaking Changes)" subsection of the Sources section in vooronderzoek-blender.md:

- Blender 4.1 Python API: https://developer.blender.org/docs/release_notes/4.1/python_api/
- Blender 4.3 Python API: https://developer.blender.org/docs/release_notes/4.3/python_api/
- Python API Changelog: https://docs.blender.org/api/current/change_log.html
- Release Notes Index: https://developer.blender.org/docs/release_notes/
- Grease Pencil 4.3 Migration Guide: https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/
- IfcOpenShell BGL Migration Issue: https://github.com/IfcOpenShell/IfcOpenShell/issues/2897

---

### Research Sources

All findings in this document are based on the following sources:

- [Blender 4.1 Python API Release Notes](https://developer.blender.org/docs/release_notes/4.1/python_api/)
- [Blender 4.3 Python API Release Notes](https://developer.blender.org/docs/release_notes/4.3/python_api/)
- [Blender GPU Module Documentation](https://docs.blender.org/api/current/gpu.html)
- [Blender BGL Module (Deprecated)](https://docs.blender.org/api/current/bgl.html)
- [Official GPU Drawing Examples (gpu.1.py)](https://github.com/blender/blender/blob/main/doc/python_api/examples/gpu.1.py)
- [IfcOpenShell BGL-to-GPU Migration Issue #2897](https://github.com/IfcOpenShell/IfcOpenShell/issues/2897)
- [Blender DevTalk: BGL Replacement Discussion](https://devtalk.blender.org/t/what-is-the-new-replacement-for-bgl-module-code-in-blender-3-5-0/28691)
- [Grease Pencil Migration Guide (4.3)](https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/)
- [Blender Release Notes Index](https://developer.blender.org/docs/release_notes/)
- [Python API Changelog](https://docs.blender.org/api/current/change_log.html)
