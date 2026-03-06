# Version Migration Anti-Patterns

Common version migration mistakes with explanations of WHY they fail and WHAT to do instead.

---

## AP-1: Hardcoding Version Strings Instead of Tuple Comparison

### The Mistake

```python
# WRONG — String comparison is fragile and incorrect
if bpy.app.version_string == "4.2.0":
    pass
if bpy.app.version_string.startswith("4"):
    pass
if "4.2" in bpy.app.version_string:
    pass
```

### WHY This Fails

`bpy.app.version_string` includes build metadata in development builds (e.g., `"4.2.0 Alpha"` or `"4.2.1 Beta"`). String comparison breaks on patch versions, alpha/beta builds, and release candidates. String `startswith("4")` fails to distinguish Blender 4.0 from 4.9 when version-specific behavior differs.

### Correct Approach

```python
# CORRECT — ALWAYS use tuple comparison
if bpy.app.version >= (4, 2, 0):
    pass

# For version ranges
if (4, 0, 0) <= bpy.app.version < (5, 0, 0):
    pass
```

---

## AP-2: Ignoring Deprecation Warnings

### The Mistake

```python
# WRONG — Using deprecated API without migration plan
import bgl  # Deprecated since 3.5, REMOVED in 5.0
bgl.glEnable(bgl.GL_BLEND)

# WRONG — Suppressing deprecation warnings
import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)
mesh.use_auto_smooth = True  # Removed in 4.1
```

### WHY This Fails

Blender removes deprecated APIs on a defined schedule. `bgl` was deprecated in 3.5 and REMOVED in 5.0. `use_auto_smooth` was REMOVED in 4.1. Suppressing warnings hides the migration signal and guarantees a hard crash when the user upgrades Blender. Each deprecation notice includes the target removal version.

### Correct Approach

```python
# CORRECT — Migrate immediately when deprecation is announced
import gpu  # Replacement for bgl
gpu.state.blend_set('ALPHA')

# CORRECT — Check if API exists before using it
if hasattr(mesh, 'use_auto_smooth'):
    mesh.use_auto_smooth = True  # Blender < 4.1
else:
    # Blender 4.1+ — use corner_normals
    normals = mesh.corner_normals
```

---

## AP-3: Using Single Version Check for Multi-Version API

### The Mistake

```python
# WRONG — Assumes EEVEE identifier is the same for all 4.x versions
if bpy.app.version >= (4, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE'  # WRONG for 4.2–4.x
```

### WHY This Fails

The EEVEE identifier changed THREE times:
- Blender 3.x: `'BLENDER_EEVEE'`
- Blender 4.2–4.x: `'BLENDER_EEVEE_NEXT'`
- Blender 5.0+: `'BLENDER_EEVEE'` (changed back)

A single `>= (4, 0, 0)` check misses the 4.2 rename. The code silently sets an invalid engine identifier.

### Correct Approach

```python
# CORRECT — Handle each version boundary explicitly
if bpy.app.version >= (5, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE'
elif bpy.app.version >= (4, 2, 0):
    scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    scene.render.engine = 'BLENDER_EEVEE'
```

---

## AP-4: Assuming Backward Compatibility of Save Files

### The Mistake

```python
# WRONG — Assuming a file saved in 4.3+ works in 4.2
gp_data = bpy.data.grease_pencils["MyDrawing"]
# Save file → open in Blender 4.2 → corrupted Grease Pencil data
```

### WHY This Fails

Blender 4.3 completely rewrote the Grease Pencil data structure. Files saved with the new format are NOT readable by Blender 4.2 or earlier. This is explicitly documented in the 4.3 release notes. Similar forward-incompatibility exists for Slotted Actions (4.4+) and the new compositor node group system (5.0+).

### Correct Approach

ALWAYS document the minimum Blender version in `blender_manifest.toml` or `bl_info`. NEVER assume files saved in version N load in version N-1 when a data format change occurred.

```toml
# blender_manifest.toml — Set minimum version to match data format
blender_version_min = "4.3.0"  # Required for new Grease Pencil format
```

---

## AP-5: Using Legacy Context Override Dicts After 4.0

### The Mistake

```python
# WRONG — Legacy context override (crashes in 4.0+)
override = bpy.context.copy()
override["object"] = target_obj
bpy.ops.object.select_all(override, action='DESELECT')
```

### WHY This Fails

Blender 4.0 REMOVED the ability to pass a dict as the first argument to `bpy.ops` calls. This was the #1 breaking change in 4.0 and the #1 cause of addon breakage. The dict-based override was undocumented behavior that became widespread through community tutorials.

### Correct Approach

```python
# CORRECT — temp_override context manager (Blender 4.0+)
with bpy.context.temp_override(object=target_obj):
    bpy.ops.object.select_all(action='DESELECT')
```

---

## AP-6: Accessing Node Sockets by Index Instead of Name

### The Mistake

```python
# WRONG — Index-based access is fragile
output_value = node.outputs[0].default_value
node.inputs[2].default_value = 0.5
```

### WHY This Fails

Socket indices change when Blender adds, removes, or reorders sockets between versions. Blender 4.0 renamed Principled BSDF sockets and changed their order. Blender 4.1 introduced dynamic socket types that further change indices. Code using `[0]` or `[2]` silently accesses the wrong socket.

### Correct Approach

```python
# CORRECT — Name-based access is stable across versions
output_value = node.outputs["BSDF"].default_value
node.inputs["Roughness"].default_value = 0.5

# For Principled BSDF, use version-safe names
if bpy.app.version >= (4, 0, 0):
    subsurface = node.inputs["Subsurface Weight"]
else:
    subsurface = node.inputs["Subsurface"]
```

---

## AP-7: Not Restoring GPU State After Drawing

### The Mistake

```python
# WRONG — State leak corrupts Blender viewport
def draw_callback():
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    shader.bind()
    batch.draw(shader)
    # MISSING: state restoration
```

### WHY This Fails

The `gpu.state` module modifies global OpenGL/Vulkan/Metal state. If a draw callback sets blend mode to `'ALPHA'` and does not reset it, ALL subsequent Blender viewport drawing uses alpha blending, causing visual corruption. This is a persistent bug that survives between frames.

### Correct Approach

```python
# CORRECT — ALWAYS restore state at end of callback
def draw_callback():
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.line_width_set(2.0)

    shader.bind()
    batch.draw(shader)

    # Restore ALL modified state
    gpu.state.blend_set('NONE')
    gpu.state.depth_test_set('NONE')
    gpu.state.line_width_set(1.0)
```

---

## AP-8: Using `scene['cycles']` Dict Access in 5.0+

### The Mistake

```python
# WRONG — Dict-like access to RNA properties (REMOVED in 5.0)
scene = bpy.context.scene
cycles = scene['cycles']
samples = scene['cycles']['samples']
```

### WHY This Fails

Blender 5.0 removed runtime-defined IDProperty access to `bpy.props`-defined properties. The dict-like `scene['cycles']` syntax was an implementation detail that leaked through the API. In 5.0, this raises `KeyError` because the property is defined via RNA, not as a custom IDProperty.

### Correct Approach

```python
# CORRECT — Attribute access via RNA (works in ALL versions)
scene = bpy.context.scene
samples = scene.cycles.samples
scene.cycles.samples = 128
```

---

## AP-9: Mixing Legacy and Modern Addon Metadata

### The Mistake

```python
# WRONG — Both bl_info AND blender_manifest.toml with conflicting data
# __init__.py
bl_info = {
    "name": "My Tool",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
}

# blender_manifest.toml
# version = "2.0.0"  # DIFFERENT version!
# blender_version_min = "4.2.0"
```

### WHY This Fails

When both `bl_info` and `blender_manifest.toml` exist, Blender 4.2+ uses the manifest and ignores `bl_info`. If the versions differ, users see inconsistent information. If `blender_version_min` in the manifest is higher than `blender` in `bl_info`, the addon works differently in 4.1 vs 4.2.

### Correct Approach

For Blender 4.2+ extensions: use ONLY `blender_manifest.toml`. Remove `bl_info` entirely.
For addons that must support 4.0–4.1 AND 4.2+: keep both but ensure all metadata is identical.

---

## AP-10: Not Handling the 4.4 Subclass Constructor Change

### The Mistake

```python
# WRONG — Missing super().__init__() in Blender 4.4+
class MY_OT_tool(bpy.types.Operator):
    bl_idname = "my.tool"
    bl_label = "Tool"

    def __init__(self):
        self.state = {}  # Crashes in 4.4+
```

### WHY This Fails

Blender 4.4 changed the C-level class initialization for RNA types. All subclasses of `Operator`, `PropertyGroup`, `RenderEngine`, and other Blender types MUST call `super().__init__()` with forwarded arguments. Without this, the C-side initialization is incomplete, causing undefined behavior or crashes.

### Correct Approach

```python
# CORRECT — Forward args to parent constructor (Blender 4.4+)
class MY_OT_tool(bpy.types.Operator):
    bl_idname = "my.tool"
    bl_label = "Tool"

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.state = {}
```

---

## Summary Table

| # | Anti-Pattern | Version Impact | Severity |
|---|-------------|---------------|----------|
| AP-1 | Hardcoding version strings | All | HIGH |
| AP-2 | Ignoring deprecation warnings | All | HIGH |
| AP-3 | Single check for multi-version API | 4.2+ / 5.0+ | MEDIUM |
| AP-4 | Assuming save file backward compat | 4.3+ / 5.0+ | HIGH |
| AP-5 | Legacy context override dicts | 4.0+ | CRITICAL |
| AP-6 | Node socket access by index | 4.0+ | HIGH |
| AP-7 | Not restoring GPU state | All | MEDIUM |
| AP-8 | Dict access to RNA properties | 5.0+ | HIGH |
| AP-9 | Mixed addon metadata | 4.2+ | MEDIUM |
| AP-10 | Missing super().__init__() | 4.4+ | HIGH |

---

## Sources

- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.2/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
- https://developer.blender.org/docs/release_notes/4.4/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- https://developer.blender.org/docs/release_notes/5.1/python_api/
- https://developer.blender.org/docs/release_notes/compatibility/
