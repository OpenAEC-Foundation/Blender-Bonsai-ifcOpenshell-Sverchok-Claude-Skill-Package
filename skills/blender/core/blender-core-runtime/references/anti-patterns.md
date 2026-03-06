# blender-core-runtime: Anti-Patterns

These are confirmed error patterns for Blender Python runtime behavior. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://docs.blender.org/api/current/info_gotchas.html
- https://docs.blender.org/api/current/mathutils.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy.app.handlers.html
- supplementary-blender-gaps.md §7 (mathutils) and §8 (runtime quirks)

---

## AP-001: Calling bpy from a Background Thread

**WHY this is wrong**: Blender's C/C++ core is NOT thread-safe. Calling any `bpy` API from a non-main thread causes immediate segfault or memory corruption. The Python GIL does NOT protect against this — the issue is in Blender's C code, not Python's.

```python
# WRONG — CRASHES Blender immediately or corrupts memory
import threading
import bpy

def bad_worker():
    bpy.data.objects["Cube"].location.x = 5.0  # CRASH

thread = threading.Thread(target=bad_worker)
thread.start()
```

```python
# CORRECT — Queue work for main thread via timer
import threading
import queue
import bpy

_queue = queue.Queue()

def _drain_queue():
    while not _queue.empty():
        _queue.get()()
    return 0.1

bpy.app.timers.register(_drain_queue)

def safe_worker():
    import time
    time.sleep(2)  # Heavy computation (thread-safe)
    _queue.put(lambda: setattr(bpy.data.objects["Cube"].location, 'x', 5.0))

threading.Thread(target=safe_worker, daemon=True).start()
```

NEVER call `bpy.*` from any thread except the main thread. ALWAYS use `queue.Queue` + `bpy.app.timers` to schedule main-thread execution.

---

## AP-002: Storing bpy.data References Across Undo

**WHY this is wrong**: Blender's undo system completely rebuilds the data model. ALL Python references to `bpy.data` objects become dangling pointers. Accessing them causes `ReferenceError` or crashes.

```python
# WRONG — reference becomes invalid after undo
obj = bpy.data.objects["Wall"]
bpy.ops.ed.undo()
print(obj.name)  # CRASH or ReferenceError
```

```python
# ALSO WRONG — storing reference in class/module variable
class MyAddon:
    cached_obj = None

    def setup(self):
        self.cached_obj = bpy.context.active_object  # Stored reference

    def later(self):
        print(self.cached_obj.name)  # CRASH if undo happened
```

```python
# CORRECT — store name, re-fetch on access
obj_name = bpy.data.objects["Wall"].name  # Store STRING
bpy.ops.ed.undo()
obj = bpy.data.objects.get(obj_name)  # Re-fetch
if obj is not None:
    print(obj.name)
```

ALWAYS store object names (strings), NEVER store `bpy.data` references beyond the current function scope. Re-fetch by name when needed.

**When invalidation occurs:**
- `bpy.ops.ed.undo()` / `bpy.ops.ed.redo()`
- `bpy.ops.wm.open_mainfile()`
- `bpy.ops.wm.revert_mainfile()`
- Any operator with undo support that the user undoes interactively

---

## AP-003: Matrix Multiplication with * Operator

**WHY this is wrong**: In Python/mathutils, `*` on matrices performs element-wise multiplication, NOT matrix product. The `@` operator (PEP 465) is the correct matrix multiplication operator.

```python
# WRONG — element-wise multiplication, NOT matrix product
from mathutils import Matrix
m1 = Matrix.Rotation(0.5, 4, 'X')
m2 = Matrix.Translation((1, 0, 0))
result = m1 * m2  # WRONG: element-wise, not matrix multiply
```

```python
# CORRECT — @ operator for matrix multiplication
result = m1 @ m2  # Matrix product
transformed_point = result @ Vector((0, 0, 0))  # Apply to point
```

ALWAYS use `@` for matrix multiplication. NEVER use `*` between matrices or matrix-vector operations.

---

## AP-004: Forgetting kd.balance() After KDTree Inserts

**WHY this is wrong**: `KDTree.balance()` builds the internal tree structure. Without it, queries return incorrect results (wrong nearest neighbor, wrong distances, or empty results).

```python
# WRONG — queries return incorrect or no results
from mathutils.kdtree import KDTree

kd = KDTree(100)
for i, v in enumerate(mesh.vertices):
    kd.insert(v.co, i)
# MISSING: kd.balance()
result = kd.find((0, 0, 0))  # UNRELIABLE results
```

```python
# CORRECT — ALWAYS balance after all inserts
kd = KDTree(100)
for i, v in enumerate(mesh.vertices):
    kd.insert(v.co, i)
kd.balance()  # MANDATORY before any query
result = kd.find((0, 0, 0))  # Correct result
```

ALWAYS call `kd.balance()` after all inserts and before any query. Balance ONCE — inserting after balance requires rebuilding the tree.

---

## AP-005: BVHTree.FromObject Without Depsgraph

**WHY this is wrong**: Without the evaluated depsgraph, `BVHTree.FromObject()` uses the un-evaluated mesh (no modifiers applied, no deformations). The BVH tree will not match what the user sees in the viewport.

```python
# WRONG — missing depsgraph, modifiers are ignored
from mathutils.bvhtree import BVHTree

bvh = BVHTree.FromObject(obj, None)  # No depsgraph — original mesh only
```

```python
# CORRECT — use evaluated depsgraph
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
bvh = BVHTree.FromObject(obj_eval, depsgraph)
```

ALWAYS pass the evaluated depsgraph and use `obj.evaluated_get(depsgraph)` to get the post-modifier object.

---

## AP-006: Handler Without @persistent Decorator

**WHY this is wrong**: Handlers without `@persistent` are automatically removed when a new file is loaded. This means addon handlers silently stop working after the user opens a different file.

```python
# WRONG — handler removed on file load
def on_load(filepath):
    print("File loaded")

bpy.app.handlers.load_post.append(on_load)
# After user opens new file: handler is GONE
```

```python
# CORRECT — use @persistent decorator
from bpy.app.handlers import persistent

@persistent
def on_load(filepath):
    print("File loaded")

bpy.app.handlers.load_post.append(on_load)
# Handler survives file loads
```

ALWAYS use `@persistent` on addon handlers. The ONLY exception is handlers that are intentionally temporary (one file only).

---

## AP-007: Forgetting to Remove Handlers on Addon Unregister

**WHY this is wrong**: If an addon registers handlers but does not remove them in `unregister()`, handlers accumulate every time the addon is reloaded during development. This causes duplicate handler calls and memory leaks.

```python
# WRONG — handlers accumulate on addon reload
def register():
    bpy.app.handlers.load_post.append(my_handler)

def unregister():
    pass  # MISSING: handler removal
# After 5 reloads: my_handler runs 5 times per file load
```

```python
# CORRECT — always clean up in unregister
def register():
    bpy.app.handlers.load_post.append(my_handler)

def unregister():
    if my_handler in bpy.app.handlers.load_post:
        bpy.app.handlers.load_post.remove(my_handler)
```

ALWAYS remove handlers in `unregister()`. Check `if handler in list` before removing to avoid `ValueError`.

---

## AP-008: Heavy Computation in Timer Callbacks

**WHY this is wrong**: Timer callbacks execute on the main thread during Blender's event loop. Long-running callbacks freeze the entire UI. Users will think Blender has crashed.

```python
# WRONG — blocks UI for duration of computation
def bad_timer():
    import time
    time.sleep(5)  # Blocks Blender UI for 5 seconds
    bpy.data.objects["Cube"].location.z = 10
    return None

bpy.app.timers.register(bad_timer)
```

```python
# CORRECT — offload heavy work to thread, use timer only for bpy updates
import threading
import queue

_results = queue.Queue()

def heavy_computation():
    import time
    time.sleep(5)  # Runs in background thread (UI stays responsive)
    _results.put(10.0)

def check_results():
    if not _results.empty():
        z_value = _results.get()
        bpy.data.objects["Cube"].location.z = z_value
        return None  # Done
    return 0.1  # Keep checking

threading.Thread(target=heavy_computation, daemon=True).start()
bpy.app.timers.register(check_results)
```

NEVER do heavy computation inside timer callbacks. ALWAYS offload to a background thread and use the timer only to transfer results to the main thread.

---

## AP-009: frame_change_post Handler Modifying Viewport Data

**WHY this is wrong**: During rendering, `frame_change_pre` and `frame_change_post` run on ONE thread while the viewport updates on a DIFFERENT thread. Modifying data accessed by the viewport causes race conditions and crashes.

```python
# WRONG — modifying viewport data from frame_change handler during render
@persistent
def on_frame(scene, depsgraph):
    for obj in scene.objects:
        obj.color = (1, 0, 0, 1)  # CRASH during render (thread conflict)

bpy.app.handlers.frame_change_post.append(on_frame)
```

```python
# CORRECT — guard against concurrent render
@persistent
def on_frame(scene, depsgraph):
    if bpy.app.is_job_running('RENDER'):
        return  # Skip viewport updates during render
    for obj in scene.objects:
        obj.color = (1, 0, 0, 1)

bpy.app.handlers.frame_change_post.append(on_frame)
```

ALWAYS check `bpy.app.is_job_running('RENDER')` in frame_change handlers that modify viewport-visible data. Alternatively, use `depsgraph_update_post` which is thread-safe for this purpose.

---

## AP-010: Using bgl Module in Blender 5.0+

**WHY this is wrong**: The `bgl` module (OpenGL wrapper) was deprecated in Blender 3.5 and FULLY REMOVED in Blender 5.0. Any code using `bgl` raises `ImportError` in 5.0+.

```python
# WRONG — REMOVED in Blender 5.0
import bgl
bgl.glEnable(bgl.GL_BLEND)
bgl.glLineWidth(2)
```

```python
# CORRECT — gpu module (works 3.x/4.x/5.x)
import gpu
gpu.state.blend_set('ALPHA')
# Line width: use POLYLINE shaders instead of global state
```

NEVER use `bgl` in any new code. ALWAYS use the `gpu` module and `gpu_extras.batch`.

---

## AP-011: Using v.length in Tight Loops for Comparisons

**WHY this is wrong**: `Vector.length` computes a square root, which is expensive. When comparing distances, the square root is unnecessary — comparing squared lengths gives the same ordering.

```python
# WRONG — unnecessary sqrt in every iteration
closest = None
closest_dist = float('inf')
for v in vertices:
    dist = (target - v.co).length  # sqrt every iteration
    if dist < closest_dist:
        closest_dist = dist
        closest = v
```

```python
# CORRECT — use length_squared (no sqrt)
closest = None
closest_dist_sq = float('inf')
for v in vertices:
    dist_sq = (target - v.co).length_squared  # No sqrt — faster
    if dist_sq < closest_dist_sq:
        closest_dist_sq = dist_sq
        closest = v
# If actual distance needed later: dist = closest_dist_sq ** 0.5
```

ALWAYS use `.length_squared` when comparing distances. Only compute `.length` when the actual distance value is needed (e.g., for display).

---

## AP-012: Confusing normalized() vs normalize()

**WHY this is wrong**: `v.normalized()` returns a NEW vector and does NOT modify the original. `v.normalize()` modifies the vector IN PLACE and returns None. Confusing them leads to unchanged vectors or lost results.

```python
# WRONG — result is discarded, v is unchanged
v = Vector((3, 4, 0))
v.normalized()  # Returns new vector but result is THROWN AWAY
print(v.length)  # Still 5.0, not 1.0
```

```python
# WRONG — tries to use None return value
v = Vector((3, 4, 0))
result = v.normalize()  # Returns None, modifies v in place
print(result)  # None
```

```python
# CORRECT — two valid patterns
v = Vector((3, 4, 0))

# Pattern A: get new normalized vector (original unchanged)
v_norm = v.normalized()
print(v_norm.length)  # 1.0
print(v.length)       # Still 5.0

# Pattern B: normalize in place
v.normalize()
print(v.length)       # Now 1.0
```

ALWAYS store the return value of `normalized()` OR use `normalize()` for in-place mutation. NEVER call `normalized()` without capturing the result.

---

## AP-013: Using del obj["prop"] in Blender 5.0+ for RNA Properties

**WHY this is wrong**: Blender 5.0 removed the ability to use `del` on RNA-defined properties via IDProperty dict access. Custom properties (user-defined) still support `del`, but built-in RNA properties do not.

```python
# WRONG — Blender 5.0+ (works in 3.x/4.x)
del obj["location"]  # TypeError in 5.0+ for RNA properties
```

```python
# CORRECT — version-aware pattern
if bpy.app.version >= (5, 0, 0):
    obj.property_unset("location")  # Reset RNA property to default
else:
    del obj["location"]

# Custom properties — del STILL works in all versions
obj["my_custom"] = 42
del obj["my_custom"]  # Works in 3.x/4.x/5.x
```

For Blender 5.0+, ALWAYS use `obj.property_unset("prop_name")` to reset RNA properties to their defaults. `del` ONLY works for custom IDProperties.

---

## AP-014: Dict-Style Access to RNA Properties in Blender 5.0+

**WHY this is wrong**: Blender 5.0 removed IDProperty dict-style access to RNA-defined properties. `scene["cycles"]` no longer returns the Cycles render settings — it raises `KeyError`.

```python
# WRONG — Blender 5.0+
settings = scene["cycles"]  # KeyError in 5.0+
```

```python
# CORRECT — attribute access (ALL versions)
settings = scene.cycles  # Works in 3.x/4.x/5.x
```

ALWAYS use attribute access for RNA properties. NEVER use dict-style access (`[]`) for built-in Blender properties — reserve dict access for custom IDProperties only.

---

## AP-015: msgbus Subscription Without PERSISTENT on Addon Handlers

**WHY this is wrong**: msgbus subscriptions are cleared on file load unless the `PERSISTENT` option is set. If an addon relies on msgbus subscriptions, they silently stop working after the user opens a file.

```python
# WRONG — subscription lost on file load
bpy.msgbus.subscribe_rna(
    key=bpy.context.object.location,
    owner=owner,
    args=(),
    notify=callback,
    options=set(),  # No PERSISTENT — cleared on file load
)
```

```python
# CORRECT — persistent subscription
bpy.msgbus.subscribe_rna(
    key=(bpy.types.Object, "location"),  # Type-level (survives object changes)
    owner=owner,
    args=(),
    notify=callback,
    options={'PERSISTENT'},  # Survives file load
)
```

For addon msgbus subscriptions, ALWAYS use `options={'PERSISTENT'}`. Also prefer type-level keys `(bpy.types.X, "prop")` over instance-level keys — instance keys break when the object is deleted or replaced.

---

## AP-016: Using threading.Timer Instead of bpy.app.timers

**WHY this is wrong**: `threading.Timer` executes its callback on a background thread. Any `bpy` calls from that thread crash Blender (see AP-001). `bpy.app.timers` executes on the main thread.

```python
# WRONG — callback runs on background thread
import threading
threading.Timer(2.0, lambda: bpy.ops.object.select_all(action='SELECT')).start()
# CRASH when timer fires
```

```python
# CORRECT — use Blender's timer system
def select_all():
    bpy.ops.object.select_all(action='SELECT')
    return None  # One-shot

bpy.app.timers.register(select_all, first_interval=2.0)
```

NEVER use `threading.Timer` for delayed `bpy` calls. ALWAYS use `bpy.app.timers.register()`.
