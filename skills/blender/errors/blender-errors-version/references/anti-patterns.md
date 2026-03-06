# Version Compatibility Anti-Patterns

Common mistakes that cause version compatibility errors. Each anti-pattern explains WHAT goes wrong, WHY it fails, and HOW to fix it.

---

## AP-1: Catching AttributeError as Version Detection

### The Mistake

```python
# WRONG — Using exception handling for version detection
try:
    mesh.use_auto_smooth = True
except AttributeError:
    pass  # "Must be 4.1+"

try:
    scene.node_tree.nodes.clear()
except AttributeError:
    scene.compositing_node_group.nodes.clear()
```

### WHY This Fails

1. `except AttributeError: pass` silently swallows ALL attribute errors, not just version-related ones. A typo like `mesh.use_auto_smoth` (missing 'o') is also silently ignored.
2. The `try/except` pattern hides the actual version requirement from code reviewers.
3. When both branches have bugs, the exception-based approach makes debugging significantly harder because the original error is lost.
4. Performance: exception handling in Python is slower than a conditional check for the expected common case.

### Correct Approach

```python
# CORRECT — Explicit version check with clear intent
import bpy

if bpy.app.version >= (4, 1, 0):
    # Blender 4.1+ — auto smooth is always active
    pass
else:
    mesh.use_auto_smooth = True

# CORRECT — Feature detection with hasattr (acceptable alternative)
if hasattr(mesh, 'use_auto_smooth'):
    mesh.use_auto_smooth = True
else:
    pass  # 4.1+ — always active
```

---

## AP-2: Partial Migration of BGL Code

### The Mistake

```python
# WRONG — Replacing only some bgl calls, leaving others
import gpu

def draw_callback():
    gpu.state.blend_set('ALPHA')        # Migrated
    gpu.state.line_width_set(2.0)       # Migrated

    import bgl                           # NOT migrated — crashes in 5.0
    bgl.glEnable(bgl.GL_DEPTH_TEST)     # NOT migrated
    bgl.glEnable(bgl.GL_LINE_SMOOTH)    # NOT migrated

    shader = gpu.shader.from_builtin('3D_UNIFORM_COLOR')  # NOT migrated — fails in 4.0+

    shader.bind()
    batch.draw(shader)

    gpu.state.blend_set('NONE')         # Migrated
    # MISSING: depth_test restoration
    # MISSING: line_width restoration
```

### WHY This Fails

1. `import bgl` itself raises `ModuleNotFoundError` in Blender 5.0. Even one leftover `bgl` reference crashes the entire addon.
2. `'3D_UNIFORM_COLOR'` was renamed to `'POLYLINE_UNIFORM_COLOR'` in Blender 4.0. Mixing old and new names creates a shader that exists in neither version.
3. Incomplete state restoration causes viewport corruption that persists across frames and affects all other addons' draw callbacks.

### Correct Approach

```python
# CORRECT — Complete migration with full state management
import gpu
from gpu_extras.batch import batch_for_shader

def draw_callback():
    # Set ALL state at the beginning
    gpu.state.blend_set('ALPHA')
    gpu.state.depth_test_set('LESS_EQUAL')
    gpu.state.depth_mask_set(True)
    gpu.state.line_width_set(2.0)

    # Use 4.0+ shader names
    shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
    batch = batch_for_shader(shader, 'LINES', {"pos": coords})
    shader.bind()
    shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])
    shader.uniform_float("lineWidth", 2.0)
    shader.uniform_float("color", (1.0, 0.5, 0.0, 0.8))
    batch.draw(shader)

    # Restore ALL state at the end
    gpu.state.blend_set('NONE')
    gpu.state.depth_test_set('NONE')
    gpu.state.depth_mask_set(False)
    gpu.state.line_width_set(1.0)
```

### Migration Checklist

- [ ] Remove ALL `import bgl` statements
- [ ] Replace ALL `bgl.gl*` calls with `gpu.state.*`
- [ ] Replace ALL `3D_*` shader names with 4.0+ names
- [ ] Replace `image.gl_load()` / `image.bindcode` with `gpu.texture.from_image()`
- [ ] Add state restoration for EVERY state change
- [ ] Test on Blender 5.0+ to verify zero `bgl` references remain

---

## AP-3: Hardcoding a Single EEVEE Identifier

### The Mistake

```python
# WRONG — Hardcoded EEVEE identifier that breaks in other versions
scene.render.engine = 'BLENDER_EEVEE'       # Fails in 4.2-4.x
scene.render.engine = 'BLENDER_EEVEE_NEXT'  # Fails in 3.x and 5.0+

# WRONG — Insufficient version check
if bpy.app.version >= (4, 0, 0):
    scene.render.engine = 'BLENDER_EEVEE_NEXT'  # Wrong for 5.0+
```

### WHY This Fails

The EEVEE engine identifier changed THREE times across versions:
- Blender 3.x: `'BLENDER_EEVEE'`
- Blender 4.2–4.x: `'BLENDER_EEVEE_NEXT'` (renamed during EEVEE rewrite)
- Blender 5.0+: `'BLENDER_EEVEE'` (reverted to original name)

A single `>=` check at 4.0 misses the 4.2 rename. A single check at 4.2 misses the 5.0 revert. Both produce `RuntimeError` or `ValueError` when the identifier is not in the enum.

### Correct Approach

```python
# CORRECT — Handle all three version boundaries
import bpy

def get_eevee_identifier():
    if bpy.app.version >= (5, 0, 0):
        return 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        return 'BLENDER_EEVEE_NEXT'
    return 'BLENDER_EEVEE'

scene.render.engine = get_eevee_identifier()
```

---

## AP-4: Using Index-Based Node Socket Access

### The Mistake

```python
# WRONG — Socket indices change between versions
value = node.outputs[0].default_value         # Which output?
node.inputs[2].default_value = 0.5            # Which input?
link = tree.links.new(node_a.outputs[0], node_b.inputs[1])
```

### WHY This Fails

1. Blender 4.0 renamed and reordered Principled BSDF sockets. Index `[2]` pointed to `Subsurface` in 3.x but `Metallic` in 4.0.
2. Blender 4.1 introduced dynamic socket types that change indices at runtime based on socket availability flags.
3. Even within a single version, different node types have different socket orders.
4. Future versions may add, remove, or reorder sockets without warning.

### Correct Approach

```python
# CORRECT — Name-based access is stable
value = node.outputs["BSDF"].default_value
node.inputs["Roughness"].default_value = 0.5
link = tree.links.new(node_a.outputs["BSDF"], node_b.inputs["Surface"])

# For Principled BSDF — version-safe socket names
import bpy

SOCKET_RENAMES = {
    "Subsurface": "Subsurface Weight",
    "Specular": "Specular IOR Level",
    "Transmission": "Transmission Weight",
    "Clearcoat": "Coat Weight",
    "Clearcoat Roughness": "Coat Roughness",
    "Emission": "Emission Color",
}

def get_socket(node, name):
    """Get node input socket by version-safe name."""
    if name in node.inputs:
        return node.inputs[name]
    mapped = SOCKET_RENAMES.get(name, name)
    if mapped in node.inputs:
        return node.inputs[mapped]
    return None
```

---

## AP-5: Suppressing DeprecationWarning in Blender Scripts

### The Mistake

```python
# WRONG — Suppressing deprecation warnings hides migration signals
import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)

import bgl  # Deprecated since 3.5, REMOVED in 5.0
bgl.glEnable(bgl.GL_BLEND)
mesh.use_auto_smooth = True  # Deprecated in 4.0, REMOVED in 4.1
```

### WHY This Fails

1. Blender's deprecation warnings include the target removal version. Suppressing them means the developer never sees "bgl is deprecated and will be removed in 5.0".
2. When the user upgrades Blender, the addon crashes with no prior warning.
3. Deprecation warnings are the ONLY signal that an API will be removed. Without them, the developer has to manually check every release note.
4. Blender's deprecation-to-removal cycle is typically 1-4 releases. `use_auto_smooth` had only 1 release of warning before removal.

### Correct Approach

```python
# CORRECT — Act on deprecation warnings immediately
# When you see a DeprecationWarning, migrate the API NOW.

# Step 1: Check the deprecation message for the replacement API
# Step 2: Implement the replacement
# Step 3: Add a version check if supporting multiple versions

import bpy

# Instead of suppressing the bgl warning, migrate:
import gpu
gpu.state.blend_set('ALPHA')

# Instead of ignoring auto_smooth warning, use version check:
if bpy.app.version < (4, 1, 0):
    mesh.use_auto_smooth = True
# else: auto smooth is always active in 4.1+
```

---

## AP-6: Assuming bpy.ops Context Works Across Versions

### The Mistake

```python
# WRONG — Assumes context override syntax is universal
def my_operator_function(context, obj):
    # This pattern appears in hundreds of tutorials and StackOverflow answers
    override = context.copy()
    override['active_object'] = obj
    override['object'] = obj
    override['selected_objects'] = [obj]
    bpy.ops.object.modifier_apply(override, modifier="My Modifier")
```

### WHY This Fails

This is the MOST common version error in Blender Python scripts. The dict-based context override was:
- Never officially documented (it was an implementation detail)
- Used in thousands of community tutorials
- The #1 source of addon breakage when users upgraded to 4.0
- Replaced by `context.temp_override()` which was introduced as a proper API

The error message (`TypeError`) is not immediately obvious as a version issue, leading developers to debug the wrong thing.

### Correct Approach

```python
# CORRECT — temp_override for 4.0+, legacy for 3.x
import bpy

def apply_modifier_safe(context, obj, modifier_name):
    if bpy.app.version >= (4, 0, 0):
        with context.temp_override(
            active_object=obj, object=obj, selected_objects=[obj]
        ):
            bpy.ops.object.modifier_apply(modifier=modifier_name)
    else:
        override = context.copy()
        override['active_object'] = obj
        override['object'] = obj
        override['selected_objects'] = [obj]
        bpy.ops.object.modifier_apply(override, modifier=modifier_name)
```

---

## AP-7: Using Legacy Action API in Blender 5.0+

### The Mistake

```python
# WRONG — Using removed legacy action.fcurves in 5.0+
action = bpy.data.actions.new("Walk")
fc = action.fcurves.new(data_path="location", index=0)  # AttributeError in 5.0
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (60.0, 5.0)

obj.animation_data_create()
obj.animation_data.action = action
```

### WHY This Fails

Blender 4.4 introduced Slotted Actions as the replacement for the legacy flat action API. In Blender 5.0, the legacy `action.fcurves` and `action.groups` properties are completely removed. The new system uses slots, layers, strips, and channelbags to organize animation data, allowing a single Action to contain animation for multiple datablocks.

### Correct Approach

```python
# CORRECT for Blender 5.0+ — Slotted Actions
import bpy

action = bpy.data.actions.new("Walk")
slot = action.slots.new()
slot.target_id_type = 'OBJECT'
layer = action.layers.new(name="Base Layer")
strip = layer.strips.new(type='KEYFRAME')
channelbag = strip.channelbags.new(slot=slot)
fc = channelbag.fcurves.new(data_path="location", index=0)
fc.keyframe_points.add(count=2)
fc.keyframe_points[0].co = (1.0, 0.0)
fc.keyframe_points[1].co = (60.0, 5.0)
fc.update()

obj = bpy.context.active_object
if obj.animation_data is None:
    obj.animation_data_create()
obj.animation_data.action = action
obj.animation_data.action_slot = slot

# VERSION-SAFE for 4.4+/5.0+
if bpy.app.version >= (5, 0, 0):
    # Slotted Actions (REQUIRED)
    slot = action.slots.new()
    layer = action.layers.new(name="Layer")
    strip = layer.strips.new(type='KEYFRAME')
    channelbag = strip.channelbags.new(slot=slot)
    fc = channelbag.fcurves.new(data_path="location", index=0)
elif bpy.app.version >= (4, 4, 0):
    # Slotted Actions available, legacy still works
    fc = action.fcurves.new(data_path="location", index=0)  # Legacy, deprecated
else:
    # Legacy only
    fc = action.fcurves.new(data_path="location", index=0)
```

---

## AP-8: Not Testing Against Minimum Supported Version

### The Mistake

```python
# WRONG — Developing and testing only on latest Blender
# Developer uses Blender 5.1, sets blender_version_min = "4.0.0"
# But never tests on 4.0, 4.1, 4.2, etc.

# blender_manifest.toml
# blender_version_min = "4.0.0"

# Code uses 4.3+ API without version check:
attrs = obj.data.attributes
size = attrs.domain_size('POINT')  # domain_size() added in 4.3 — crashes in 4.0-4.2
```

### WHY This Fails

1. The `blender_version_min` in the manifest is a PROMISE to the user that the addon works from that version.
2. APIs are added in specific versions. `domain_size()` was added in 4.3. Using it while claiming 4.0 support breaks on 4.0, 4.1, and 4.2.
3. Blender LTS versions (4.2, 4.5) have long support windows. Users on LTS versions will encounter these errors long after the developer has moved on.

### Correct Approach

ALWAYS test on the OLDEST declared supported version. If `blender_version_min = "4.0.0"`, test on Blender 4.0.0.

```python
# CORRECT — Version check for every API used above minimum version
import bpy

if bpy.app.version >= (4, 3, 0):
    size = obj.data.attributes.domain_size('POINT')
else:
    size = len(obj.data.vertices)  # Fallback for 4.0-4.2

# OR — Set the minimum version accurately
# blender_manifest.toml
# blender_version_min = "4.3.0"  # Honest about actual minimum
```

---

## Summary Table

| # | Anti-Pattern | Error Type | Severity | Version Impact |
|---|-------------|-----------|----------|---------------|
| AP-1 | Catching AttributeError as version detection | AttributeError | HIGH | All versions |
| AP-2 | Partial BGL migration | ImportError / ValueError | CRITICAL | 5.0+ |
| AP-3 | Hardcoded EEVEE identifier | RuntimeError / ValueError | HIGH | 4.2+ / 5.0+ |
| AP-4 | Index-based node socket access | KeyError / wrong values | HIGH | 4.0+ |
| AP-5 | Suppressing DeprecationWarning | Various (delayed crash) | HIGH | All versions |
| AP-6 | Assuming bpy.ops context syntax | TypeError | CRITICAL | 4.0+ |
| AP-7 | Legacy action API in 5.0+ | AttributeError | HIGH | 5.0+ |
| AP-8 | Not testing minimum version | Various | HIGH | All versions |

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
- https://github.com/IfcOpenShell/IfcOpenShell/issues/2897
