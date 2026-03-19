---
name: blender-core-runtime
description: >
  Use when dealing with Blender Python runtime behavior -- mathutils, threading, handlers,
  timers, or crashes after undo/redo. Prevents the #1 runtime crash: using threading for
  bpy operations instead of bpy.app.timers, or caching bpy.data references across undo
  boundaries. Covers Vector/Matrix/Quaternion, KDTree, BVHTree, @persistent handlers,
  bpy.msgbus subscriptions, and background mode limitations.
  Keywords: mathutils, threading, undo, ReferenceError, bpy.app.timers, @persistent,
  bpy.msgbus, KDTree, BVHTree, background mode, Blender crash.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-core-runtime

## Quick Reference

### Critical Warnings

**NEVER** call ANY `bpy` API from a background thread — Blender's C/C++ core is NOT thread-safe. Crashes are immediate and unrecoverable.

**NEVER** store `bpy.data` references (objects, meshes, materials) across undo/redo boundaries — undo rebuilds the entire data model, invalidating ALL Python pointers.

**NEVER** omit `@persistent` on handlers in addons — without it, handlers are silently removed on file load.

**NEVER** do heavy computation inside `bpy.app.timers` callbacks — they run on the main thread and freeze the UI.

**NEVER** use `matrix1 * matrix2` for matrix multiplication — the `*` operator is element-wise. ALWAYS use `@` operator.

**NEVER** use `bgl` module in Blender 5.0+ — fully removed. Use `gpu` module.

**ALWAYS** call `kd.balance()` after inserting all points into a KDTree — queries return incorrect results without it.

**ALWAYS** pass `depsgraph` to `BVHTree.FromObject()` — missing depsgraph skips modifiers and evaluated mesh.

**ALWAYS** remove handlers in your addon's `unregister()` function — handlers accumulate on addon reload.

### Decision Tree: Thread-Safe Execution

```
Need to run code from a worker thread?
├── Is it a bpy.data read/write? ──────── YES → Queue it via bpy.app.timers
├── Is it pure Python / mathutils? ────── YES → Safe to run in thread
├── Does it call bpy.ops.*? ───────────── YES → Queue it via bpy.app.timers
└── Does it use bpy.context? ──────────── YES → Queue it via bpy.app.timers
```

### Decision Tree: Choosing a Spatial Query Structure

```
What do you need?
├── Nearest points (vertex/position queries) ──── Use KDTree
│   ├── find()       → single nearest
│   ├── find_n()     → N nearest
│   └── find_range() → all within radius
├── Ray casting / surface intersection ─────────── Use BVHTree
│   ├── ray_cast()     → ray-surface intersection
│   ├── find_nearest() → closest point on surface
│   └── overlap()      → two-mesh intersection
└── Simple distance between two points ─────────── Use Vector math
    └── (v1 - v2).length_squared for comparisons
```

### Decision Tree: Event Subscription

```
What kind of event?
├── Property changed on specific object ──── bpy.msgbus.subscribe_rna()
├── File load/save ───────────────────────── bpy.app.handlers.load_post / save_pre
├── Undo/Redo ────────────────────────────── bpy.app.handlers.undo_post / redo_post
├── Frame change (animation) ─────────────── bpy.app.handlers.frame_change_post
├── Depsgraph update ─────────────────────── bpy.app.handlers.depsgraph_update_post
├── Deferred one-shot execution ──────────── bpy.app.timers (return None)
└── Periodic polling ─────────────────────── bpy.app.timers (return float)
```

---

## Essential Patterns

### Pattern 1: mathutils Core Types

All types available in ALL versions (3.x/4.x/5.x). Import from `mathutils`.

```python
from mathutils import Vector, Matrix, Quaternion, Euler
```

| Type | Purpose | Key Operations |
|------|---------|----------------|
| `Vector` | 2D/3D/4D point/direction | `+`, `-`, `*` (scalar), `@` (matrix), `.dot()`, `.cross()`, `.normalized()`, `.lerp()` |
| `Matrix` | 4x4 transformation | `@` (multiply), `.decompose()`, `.inverted()`, `.Translation()`, `.Rotation()` |
| `Quaternion` | Rotation (no gimbal lock) | `@` (combine), `.slerp()`, `.to_euler()`, `.to_matrix()` |
| `Euler` | Rotation with order (XYZ, etc.) | `.to_quaternion()`, `.to_matrix()`, `.make_compatible()` |

**Matrix multiplication order** (right-to-left application):
```python
# Blender 3.x/4.x/5.x — Scale → Rotate → Translate
transform = translation_matrix @ rotation_matrix @ scale_matrix
point_transformed = transform @ point_vector
```

### Pattern 2: Thread-Safe bpy Access

```python
# Blender 3.x/4.x/5.x — CORRECT thread-safe pattern
import threading, queue, bpy

execution_queue = queue.Queue()

def run_in_main_thread(fn):
    execution_queue.put(fn)

def _process_queue():
    while not execution_queue.empty():
        execution_queue.get()()
    return 0.1  # Reschedule every 100ms

bpy.app.timers.register(_process_queue)

# Worker thread schedules bpy calls safely
def worker():
    import time
    time.sleep(2)  # Simulate work
    run_in_main_thread(lambda: setattr(
        bpy.data.objects["Cube"].location, 'x', 5.0))

threading.Thread(target=worker, daemon=True).start()
```

### Pattern 3: Safe Reference Handling (Undo-Proof)

```python
# Blender 3.x/4.x/5.x — ALWAYS store names, NEVER store bpy references
obj_name = bpy.context.active_object.name  # Store name

# ... undo may happen here ...

obj = bpy.data.objects.get(obj_name)  # Re-fetch by name
if obj is None:
    raise RuntimeError(f"Object '{obj_name}' was deleted or renamed")
```

### Pattern 4: Application Handlers with @persistent

```python
# Blender 3.x/4.x/5.x — Handler registration
from bpy.app.handlers import persistent
import bpy

@persistent
def on_file_loaded(dummy):
    print(f"File loaded: {bpy.data.filepath}")

# Register
bpy.app.handlers.load_post.append(on_file_loaded)

# ALWAYS unregister in addon unregister()
def unregister():
    if on_file_loaded in bpy.app.handlers.load_post:
        bpy.app.handlers.load_post.remove(on_file_loaded)
```

### Pattern 5: Timers (One-Shot and Repeating)

```python
# Blender 3.x/4.x/5.x
import bpy

# One-shot: return None to stop
def delayed_action():
    print("Executed once after 2 seconds")
    return None  # Unregister

bpy.app.timers.register(delayed_action, first_interval=2.0)

# Repeating: return float to reschedule
def periodic_check():
    print(f"Frame: {bpy.context.scene.frame_current}")
    return 0.5  # Run again in 0.5s

bpy.app.timers.register(periodic_check)

# Persistent timer (survives file load)
bpy.app.timers.register(periodic_check, persistent=True)

# Unregister
if bpy.app.timers.is_registered(periodic_check):
    bpy.app.timers.unregister(periodic_check)
```

### Pattern 6: Message Bus Subscriptions

```python
# Blender 3.x/4.x/5.x — Property change notifications
import bpy

owner = object()  # Subscription owner (identity-compared)

def on_location_changed(*args):
    print(f"Location changed: {args}")

# Instance-level subscription
bpy.msgbus.subscribe_rna(
    key=bpy.context.object.location,
    owner=owner,
    args=(),
    notify=on_location_changed,
    options=set(),  # Or {'PERSISTENT'} to survive file loads
)

# Type-level subscription (ANY object)
bpy.msgbus.subscribe_rna(
    key=(bpy.types.Object, "location"),
    owner=owner,
    args=(),
    notify=on_location_changed,
)

# Cleanup
bpy.msgbus.clear_by_owner(owner)
```

**Key differences from property update callbacks:**
- msgbus callbacks are POSTPONED until all operators finish
- msgbus fires ONCE per update cycle, even if property changed multiple times
- Subscriptions cleared on file load unless `PERSISTENT` option is set

---

## Spatial Query Structures

### KDTree — Nearest-Neighbor Queries

```python
# Blender 3.x/4.x/5.x
from mathutils.kdtree import KDTree

mesh = bpy.context.active_object.data
kd = KDTree(len(mesh.vertices))
for i, v in enumerate(mesh.vertices):
    kd.insert(v.co, i)
kd.balance()  # MANDATORY after all inserts

co, index, dist = kd.find((5.0, 3.0, 0.0))        # Nearest
results = kd.find_n((5.0, 3.0, 0.0), 10)           # 10 nearest
results = kd.find_range((5.0, 3.0, 0.0), 2.0)      # Within radius
```

### BVHTree — Ray Casting and Surface Queries

```python
# Blender 3.x/4.x/5.x
from mathutils.bvhtree import BVHTree
from mathutils import Vector

depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
bvh = BVHTree.FromObject(obj_eval, depsgraph)

# Ray cast
loc, normal, idx, dist = bvh.ray_cast(
    Vector((0, 0, 10)), Vector((0, 0, -1)))

# Nearest surface point
loc, normal, idx, dist = bvh.find_nearest(Vector((5, 3, 0.5)))

# Overlap detection between two meshes
overlap_pairs = bvh1.overlap(bvh2)  # List of (idx1, idx2)
```

---

## Background Mode

```bash
# Run script without UI
blender --background scene.blend --python script.py
blender -b scene.blend -P script.py -- --custom-arg value
```

```python
# Blender 3.x/4.x/5.x — detect background mode
if bpy.app.background:
    # CANNOT: viewport ops, UI drawing, modal operators, GPU ops
    # CAN: render, data manipulation, file I/O, bpy.data access
    pass
```

---

## Version-Specific Changes

| Feature | Blender 3.x/4.x | Blender 5.0+ |
|---------|-----------------|--------------|
| `bgl` module | Available (deprecated 3.5+) | REMOVED — use `gpu` module |
| `del obj["prop"]` on RNA props | Works | REMOVED — use `obj.property_unset("prop")` |
| `scene["cycles"]` dict access | Works | REMOVED — use `scene.cycles` attribute |
| `img.bindcode` | Available | REMOVED — use `gpu.texture.from_image(img)` |
| `img.gl_load()` | Available | REMOVED |

---

## bl_math Utility Functions

```python
# Blender 3.x/4.x/5.x — float-only utilities (also work in driver expressions)
import bl_math

bl_math.lerp(0.0, 10.0, 0.25)           # → 2.5
bl_math.clamp(15.0, 0.0, 10.0)          # → 10.0
bl_math.clamp(-5.0)                      # → 0.0 (clamp to [0, 1])
bl_math.smoothstep(0.0, 1.0, 0.5)       # → smooth S-curve value
```

---

## Available Handlers Reference

| Handler | Trigger | Signature |
|---------|---------|-----------|
| `load_pre` / `load_post` | File load | `(filepath)` |
| `save_pre` / `save_post` | File save | `(filepath)` |
| `undo_pre` / `undo_post` | Undo | `(scene)` |
| `redo_pre` / `redo_post` | Redo | `(scene)` |
| `depsgraph_update_pre/post` | Depsgraph eval | `(scene, depsgraph)` |
| `frame_change_pre/post` | Frame change | `(scene)` / `(scene, depsgraph)` |
| `render_pre` / `render_post` | Render start/end | `(scene)` |
| `render_init` | Engine init | `(engine)` |
| `render_complete` / `render_cancel` | Render finish | `(scene)` |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for mathutils, timers, handlers, msgbus
- [references/examples.md](references/examples.md) — Working code examples for all runtime patterns
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with threading, handlers, mathutils

### Official Sources

- https://docs.blender.org/api/current/mathutils.html
- https://docs.blender.org/api/current/mathutils.kdtree.html
- https://docs.blender.org/api/current/mathutils.bvhtree.html
- https://docs.blender.org/api/current/bl_math.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy.msgbus.html
- https://docs.blender.org/api/current/bpy.app.handlers.html
- https://docs.blender.org/api/current/info_gotchas.html
