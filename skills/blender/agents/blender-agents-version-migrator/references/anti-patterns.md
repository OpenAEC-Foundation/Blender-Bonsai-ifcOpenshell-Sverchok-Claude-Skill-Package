# Migration Anti-Patterns

Common migration mistakes that cause errors, data loss, or subtle bugs. Each anti-pattern includes the mistake, why it fails, and the correct approach.

---

## AP-01: Using try/except AttributeError as Version Detection

### The Mistake

```python
# WRONG — Using exception handling for version detection
import bpy

try:
    mesh.use_auto_smooth = True
    mesh.auto_smooth_angle = 0.523599
except AttributeError:
    pass  # Must be 4.1+
```

### Why It Fails

- Silently swallows legitimate `AttributeError` exceptions caused by typos or actual bugs
- Masks the migration signal — the developer never knows which specific API was removed
- Creates untestable code — the except branch has no explicit version requirements
- Performance cost: exception handling is slower than a simple version comparison

### Correct Approach

```python
# CORRECT — Explicit version check
import bpy

if bpy.app.version < (4, 1, 0):
    mesh.use_auto_smooth = True
    mesh.auto_smooth_angle = 0.523599
else:
    # Auto-smooth is always active in 4.1+
    # Use "Smooth by Angle" modifier for angle-based control
    pass
```

---

## AP-02: Importing bgl at Module Level in Multi-Version Addons

### The Mistake

```python
# WRONG — Module-level bgl import crashes on Blender 5.0+
import bpy
import bgl  # ImportError on Blender 5.0+
import gpu

def draw():
    if bpy.app.version >= (5, 0, 0):
        gpu.state.blend_set('ALPHA')
    else:
        bgl.glEnable(bgl.GL_BLEND)
```

### Why It Fails

- Python executes `import bgl` at module load time, BEFORE any version check runs
- On Blender 5.0+, this raises `ImportError` and prevents the entire module from loading
- The version-conditional code inside `draw()` never executes because the import already failed

### Correct Approach

```python
# CORRECT — Conditional import with fallback flag
import bpy
import gpu

try:
    import bgl
    HAS_BGL = True
except ImportError:
    HAS_BGL = False  # Blender 5.0+

def draw():
    gpu.state.blend_set('ALPHA')  # ALWAYS use gpu module for new code
    # Only fall back to bgl if explicitly required for 3.x compatibility
```

---

## AP-03: Partial Grease Pencil Migration

### The Mistake

```python
# WRONG — Mixing old and new Grease Pencil API
import bpy

# Using new 4.3+ access pattern...
gp_obj = bpy.context.active_object
gp_data = gp_obj.data  # GreasePencilv3 in 4.3+

# ...but then using old API patterns
for layer in gp_data.layers:
    for frame in layer.frames:
        for stroke in frame.strokes:  # OLD API — structure changed
            stroke.line_width = 10     # OLD API — property renamed/moved
```

### Why It Fails

- The Grease Pencil API was COMPLETELY rewritten in 4.3 — not just renamed
- Internal data structures changed: layers, frames, strokes all have different access patterns
- Mixing old API calls with new data objects produces `AttributeError` or corrupt data
- Files saved with mixed API calls do NOT load correctly in older Blender versions

### Correct Approach

```python
# CORRECT — Complete migration to new Grease Pencil API (4.3+)
import bpy

gp_obj = bpy.context.active_object
gp_data = gp_obj.data  # GreasePencilv3

# Use the complete new API — consult the Grease Pencil 4.3 Migration Guide
# https://developer.blender.org/docs/release_notes/4.3/grease_pencil_migration/
# NEVER mix old and new API calls
```

---

## AP-04: Forgetting to Restore gpu.state After Draw Callbacks

### The Mistake

```python
# WRONG — State leak in draw callback
import gpu
from gpu_extras.batch import batch_for_shader

def draw_overlay():
    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(3.0)
    gpu.state.depth_test_set('LESS_EQUAL')

    shader.bind()
    shader.uniform_float("color", (1, 0, 0, 0.5))
    batch.draw(shader)
    # MISSING: state restoration — affects ALL subsequent drawing
```

### Why It Fails

- GPU state is global — setting blend mode to `'ALPHA'` affects ALL subsequent draw calls
- Other addons, overlays, and Blender's own UI rendering inherit the leaked state
- Causes visual glitches: transparent UI elements, incorrect depth testing, wrong line widths
- Intermittent and hard to debug — depends on draw call order

### Correct Approach

```python
# CORRECT — ALWAYS restore gpu state at the end of draw callbacks
import gpu
from gpu_extras.batch import batch_for_shader

def draw_overlay():
    gpu.state.blend_set('ALPHA')
    gpu.state.line_width_set(3.0)
    gpu.state.depth_test_set('LESS_EQUAL')

    shader.bind()
    shader.uniform_float("color", (1, 0, 0, 0.5))
    batch.draw(shader)

    # ALWAYS restore defaults
    gpu.state.blend_set('NONE')
    gpu.state.line_width_set(1.0)
    gpu.state.depth_test_set('NONE')
    gpu.state.depth_mask_set(False)
```

---

## AP-05: Suppressing DeprecationWarning During Migration

### The Mistake

```python
# WRONG — Hiding deprecation warnings
import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)

import bpy
# Now ALL deprecation warnings are hidden, including migration-critical ones
```

### Why It Fails

- Blender emits `DeprecationWarning` for APIs scheduled for removal in the NEXT version
- Suppressing warnings hides the exact list of APIs that need migration
- The addon appears to work correctly until the deprecated API is removed — then it breaks with no prior warning
- This turns a gradual, manageable migration into an emergency when the API is finally removed

### Correct Approach

```python
# CORRECT — Let deprecation warnings surface
import bpy

# Run the addon and collect ALL DeprecationWarning messages
# Each warning identifies a specific API that needs migration
# Address each warning by consulting the migration tables in SKILL.md

# For CI/CD: convert warnings to errors to catch deprecations early
# python -W error::DeprecationWarning -m pytest
```

---

## AP-06: Skipping Intermediate Versions During Migration

### The Mistake

Attempting to migrate directly from Blender 3.x to 5.0 without applying intermediate version changes:

```python
# WRONG — Jumping from 3.x to 5.0 and only fixing the obvious errors
# Developer fixes: import bgl → import gpu
# Developer fixes: context override dicts → temp_override
# Developer MISSES: auto_smooth removal (4.1), EEVEE renames (4.2),
#   AttributeGroup split (4.3), super().__init__ (4.4)
```

### Why It Fails

- Each Blender version removes APIs that were deprecated in the previous version
- Skipping versions means missing deprecations that had short warning periods (1 release)
- Some changes compound: EEVEE identifier changed in 4.2, changed AGAIN in 5.0
- Slotted Actions were introduced in 4.4 and became mandatory in 5.0 — missing the 4.4 step means missing the transition period

### Correct Approach

Apply migration changes in sequential version order:
1. 3.x → 4.0
2. 4.0 → 4.1
3. 4.1 → 4.2
4. 4.2 → 4.3
5. 4.3 → 4.4
6. 4.4 → 5.0
7. 5.0 → 5.1

At each step, search for ALL patterns listed in the corresponding Find-and-Replace Table. NEVER skip a version.

---

## AP-07: Using String Parsing for Version Detection

### The Mistake

```python
# WRONG — Parsing version string
import bpy

version_str = bpy.app.version_string  # e.g., "4.2.0 Release"
major = int(version_str.split(".")[0])
if major >= 4:
    # This misses minor version differences (4.0 vs 4.1 vs 4.2)
    pass
```

### Why It Fails

- `bpy.app.version_string` includes text like "Release", "Beta", "Alpha" — string parsing is fragile
- Comparing only the major version misses critical minor version differences
- Blender 4.1 removed `use_auto_smooth`, 4.2 changed EEVEE name, 4.3 rewrote Grease Pencil — these are all "4.x" but with different APIs
- `bpy.app.version` already provides a clean integer tuple for comparison

### Correct Approach

```python
# CORRECT — Use bpy.app.version tuple directly
import bpy

# bpy.app.version returns (major, minor, patch) as integers
if bpy.app.version >= (4, 2, 0):
    # Blender 4.2+ code
    pass
elif bpy.app.version >= (4, 0, 0):
    # Blender 4.0–4.1 code
    pass
else:
    # Blender 3.x code
    pass
```

---

## AP-08: Missing POLYLINE Shader Uniforms After Migration

### The Mistake

```python
# WRONG — Using POLYLINE shader without required uniforms
import gpu
from gpu_extras.batch import batch_for_shader

shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
batch = batch_for_shader(shader, 'LINES', {"pos": coords})

def draw():
    shader.bind()
    shader.uniform_float("color", (1, 0, 0, 1))
    batch.draw(shader)  # Renders incorrectly or crashes
```

### Why It Fails

- `POLYLINE_UNIFORM_COLOR` (Blender 4.0+) requires two additional uniforms that `3D_UNIFORM_COLOR` did not need
- Without `viewportSize`, the shader cannot calculate line projection correctly
- Without `lineWidth`, the shader uses an undefined width value
- The result is invisible lines, incorrectly sized lines, or a GPU error

### Correct Approach

```python
# CORRECT — Set ALL required POLYLINE shader uniforms
import gpu
from gpu_extras.batch import batch_for_shader

shader = gpu.shader.from_builtin('POLYLINE_UNIFORM_COLOR')
batch = batch_for_shader(shader, 'LINES', {"pos": coords})

def draw():
    shader.bind()
    shader.uniform_float("viewportSize", gpu.state.viewport_get()[2:])  # REQUIRED
    shader.uniform_float("lineWidth", 2.0)  # REQUIRED
    shader.uniform_float("color", (1, 0, 0, 1))
    batch.draw(shader)
```
