# Context-Related API Signatures

## bpy.context — Key Attributes

### Always Available

```python
bpy.context.scene         # bpy.types.Scene — current scene
bpy.context.view_layer    # bpy.types.ViewLayer — active view layer
bpy.context.window        # bpy.types.Window — current window
bpy.context.screen        # bpy.types.Screen — current screen layout
bpy.context.preferences   # bpy.types.Preferences — user preferences
bpy.context.blend_data    # bpy.types.BlendData — same as bpy.data
```

### Require Active Object

```python
bpy.context.active_object    # bpy.types.Object | None
bpy.context.object           # bpy.types.Object | None — same as active_object in most contexts
bpy.context.selected_objects # list[bpy.types.Object] — may be empty
```

### Require Specific Mode

```python
# Edit Mode only (returns None otherwise)
bpy.context.edit_object               # bpy.types.Object | None
bpy.context.selected_editable_objects # list[bpy.types.Object]

# Armature Edit Mode only
bpy.context.active_bone               # bpy.types.EditBone | None
bpy.context.selected_editable_bones   # list[bpy.types.EditBone]

# Armature Pose Mode only
bpy.context.active_pose_bone          # bpy.types.PoseBone | None
bpy.context.visible_pose_bones        # list[bpy.types.PoseBone]
bpy.context.selected_pose_bones       # list[bpy.types.PoseBone]

# Sculpt Mode only
bpy.context.sculpt_object             # bpy.types.Object | None

# Texture Paint Mode only
bpy.context.image_paint_object        # bpy.types.Object | None

# Weight Paint Mode only
bpy.context.vertex_paint_object       # bpy.types.Object | None
```

### Require Specific Area Type

```python
# VIEW_3D area only
bpy.context.space_data    # bpy.types.SpaceView3D (when area.type == 'VIEW_3D')
bpy.context.region_data   # bpy.types.RegionView3D

# Properties area only
bpy.context.space_data    # bpy.types.SpaceProperties (when area.type == 'PROPERTIES')

# Note: space_data type changes based on the current area type
```

---

## bpy.context.temp_override()

**Availability:** Blender 3.2+. REQUIRED in Blender 4.0+ (dict overrides removed).

### Signature

```python
bpy.context.temp_override(
    window: bpy.types.Window = None,
    area: bpy.types.Area = None,
    region: bpy.types.Region = None,
    **kwargs
) -> context manager
```

### Accepted Override Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `window` | `bpy.types.Window` | Override the current window |
| `area` | `bpy.types.Area` | Override the current area |
| `region` | `bpy.types.Region` | Override the current region |
| `screen` | `bpy.types.Screen` | Override the current screen |
| `scene` | `bpy.types.Scene` | Override the current scene |
| `active_object` | `bpy.types.Object` | Override the active object |
| `selected_objects` | `list[bpy.types.Object]` | Override selected objects |
| `edit_object` | `bpy.types.Object` | Override the edit object |

### Usage Pattern

```python
# Blender 3.2+
import bpy

# Find the 3D Viewport area
area = next((a for a in bpy.context.screen.areas if a.type == 'VIEW_3D'), None)
if area is None:
    raise RuntimeError("No 3D Viewport area found")

region = next(r for r in area.regions if r.type == 'WINDOW')

with bpy.context.temp_override(area=area, region=region):
    bpy.ops.view3d.snap_cursor_to_center()
```

---

## bpy.app.timers

### bpy.app.timers.register()

**Purpose:** Schedule a function to run on the main thread. Used to defer operations from restricted contexts.

```python
bpy.app.timers.register(
    function: callable,         # function to call — must accept no arguments
    first_interval: float = 0,  # seconds before first call (0 = next redraw)
    persistent: bool = False    # survive file load if True
) -> None
```

**Return value convention for the registered function:**
- Return `None` → function runs once
- Return `float` → function repeats after that many seconds
- Return value is the interval until next call

```python
# Blender 3.2+
import bpy

def deferred_operation():
    """Runs on main thread, safe to modify data."""
    obj = bpy.data.objects.get("Cube")
    if obj is not None:
        obj.location.x += 1.0
    return None  # run once

bpy.app.timers.register(deferred_operation)
```

### bpy.app.timers.unregister()

```python
bpy.app.timers.unregister(function: callable) -> None
```

### bpy.app.timers.is_registered()

```python
bpy.app.timers.is_registered(function: callable) -> bool
```

---

## bpy.app.handlers

### Handler Lists (context-restricted)

All handlers below run in restricted context. NEVER call `bpy.ops` or modify `bpy.data` directly from these handlers.

```python
bpy.app.handlers.depsgraph_update_pre   # list — before depsgraph evaluation
bpy.app.handlers.depsgraph_update_post  # list — after depsgraph evaluation
bpy.app.handlers.render_pre             # list — before render starts
bpy.app.handlers.render_post            # list — after render completes
bpy.app.handlers.render_write           # list — after render frame write
bpy.app.handlers.load_post              # list — after .blend file load
bpy.app.handlers.save_pre               # list — before .blend file save
bpy.app.handlers.save_post              # list — after .blend file save
bpy.app.handlers.frame_change_pre       # list — before frame change
bpy.app.handlers.frame_change_post      # list — after frame change
```

### Handler Function Signatures

```python
# depsgraph handlers
def handler(scene: bpy.types.Scene, depsgraph: bpy.types.Depsgraph) -> None: ...

# render handlers
def handler(scene: bpy.types.Scene) -> None: ...

# load/save handlers
def handler(filepath: str) -> None: ...  # load_post receives dummy arg
def handler() -> None: ...               # some handlers take no args

# frame_change handlers
def handler(scene: bpy.types.Scene, depsgraph: bpy.types.Depsgraph) -> None: ...
```

### @persistent Decorator

```python
from bpy.app.handlers import persistent

@persistent
def my_handler(scene, depsgraph):
    """Survives file load — without @persistent, handler is removed on file load."""
    pass

bpy.app.handlers.depsgraph_update_post.append(my_handler)
```

---

## bpy.types.Operator — Context-Related Methods

### poll() classmethod

```python
@classmethod
def poll(cls, context: bpy.types.Context) -> bool:
    """Return True if operator can execute in the given context.
    Called before execute/invoke. If False, operator button is grayed out
    and calling the operator raises RuntimeError.
    """
```

### invoke()

```python
def invoke(self, context: bpy.types.Context, event: bpy.types.Event) -> set[str]:
    """Called when operator is invoked by user action.
    Return: {'RUNNING_MODAL', 'FINISHED', 'CANCELLED', 'PASS_THROUGH'}
    """
```

### modal()

```python
def modal(self, context: bpy.types.Context, event: bpy.types.Event) -> set[str]:
    """Called on each event while modal. Context may differ from invoke().
    Return: {'RUNNING_MODAL', 'FINISHED', 'CANCELLED', 'PASS_THROUGH'}
    """
```

### execute()

```python
def execute(self, context: bpy.types.Context) -> set[str]:
    """Called to run the operator.
    Return: {'FINISHED', 'CANCELLED'}
    """
```

---

## Version Detection

```python
import bpy

bpy.app.version          # tuple[int, int, int] — e.g. (4, 0, 0)
bpy.app.version_string   # str — e.g. "4.0.0"
bpy.app.version_cycle    # str — "alpha", "beta", "rc", "release"

# Version comparison
if bpy.app.version >= (4, 0, 0):
    # 4.0+ code path
    pass
elif bpy.app.version >= (3, 2, 0):
    # 3.2+ code path
    pass
```
