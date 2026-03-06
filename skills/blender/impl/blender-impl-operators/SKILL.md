---
name: blender-impl-operators
description: "Provides implementation patterns for Blender operators including modal operators with timer callbacks, file browser integration, undo/redo support, batch processing operators, progress reporting, and multi-step workflows. Activates when implementing complex Blender operators, modal tools, file dialogs, or batch automation scripts."
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-impl-operators

## Quick Reference

### Critical Warnings

**ALWAYS** clean up timers in BOTH `cancel()` AND the `{'FINISHED'}` path. Leaked timers fire indefinitely and reference deleted operator instances.

**ALWAYS** call `wm.modal_handler_add(self)` BEFORE returning `{'RUNNING_MODAL'}` from `invoke()`. Returning `{'RUNNING_MODAL'}` without a handler registration means events go nowhere.

**ALWAYS** handle ESC and RIGHTMOUSE in `modal()` to allow user cancellation. Without an escape path, the user is locked in the modal state.

**ALWAYS** set `bl_options = {'REGISTER', 'UNDO'}` for operators that modify scene data. `'UNDO'` alone has NO effect — `'REGISTER'` is required for undo to work.

**NEVER** modify `bpy.data` from a background thread. Use `bpy.app.timers.register()` with a `queue.Queue` to marshal work to the main thread.

**NEVER** expect viewport redraws during a synchronous `execute()`. The UI is frozen until `execute()` returns. Use a modal operator with timer for progressive visual updates.

**NEVER** use `threading.Timer` for deferred Blender operations. Use `bpy.app.timers.register()` instead.

### Implementation Decision Tree

```
Need to implement a complex Blender operator?
│
├─ Long-running operation that must show progress?
│  └─ Use modal operator with timer (Section 1)
│     └─ Add wm.progress_begin/update/end (Section 5)
│     └─ Process items incrementally per TIMER event
│
├─ Need user to select a file?
│  └─ Use ImportHelper/ExportHelper mixins (Section 2)
│     └─ Or manual fileselect_add for custom file dialogs
│
├─ Need to process many objects in batch?
│  ├─ Fast operation (< 1 second)?
│  │  └─ Use synchronous execute() with context.temp_override (Section 4)
│  └─ Slow operation (> 1 second)?
│     └─ Use modal timer + progress reporting (Section 1 + 5)
│
├─ Need multi-step user interaction?
│  ├─ Sequential clicks/points in viewport?
│  │  └─ Use state machine in modal() (Section 6)
│  └─ Parameter dialog before execution?
│     └─ Use invoke_props_dialog (Section 6)
│
├─ Need deferred execution outside an operator?
│  └─ Use bpy.app.timers.register() (Section 7)
│     └─ Return None to run once, return float to repeat
│
└─ Need undo support?
   ├─ Single operation?
   │  └─ bl_options = {'REGISTER', 'UNDO'} (Section 3)
   └─ Repeated rapid calls (e.g., timer-driven)?
      └─ bl_options = {'REGISTER', 'UNDO_GROUPED'} (Section 3)
```

### Version Compatibility Matrix

| Feature | Blender 3.x | Blender 4.0+ | Blender 5.x |
|---------|-------------|--------------|-------------|
| `wm.event_timer_add(t, window=w)` | Keyword arg | Keyword arg | Keyword arg |
| `bpy.app.timers.register()` | Available | Available | Available |
| `context.temp_override()` | Available from 3.2 | REQUIRED | REQUIRED |
| Dict context override | Deprecated (3.2+) | REMOVED | REMOVED |
| `UILayout.progress()` | Not available | Available (4.0+) | Available |
| `ImportHelper/ExportHelper` | Stable | Stable | Stable |
| `UNDO_GROUPED` bl_option | Available | Available | Available |

---

## Section 1: Modal Operators with Timer Callbacks

### When to Use

Use modal operators with timers when an operation takes more than ~0.5 seconds and must:
- Show progress to the user
- Allow cancellation mid-operation
- Keep Blender's UI responsive during processing

### Implementation Pattern

```python
import bpy

class MYCAT_OT_long_process(bpy.types.Operator):
    """Process objects over multiple frames with progress"""
    bl_idname = "mycat.long_process"
    bl_label = "Long Process"
    bl_options = {'REGISTER', 'UNDO'}

    _timer = None
    _items: list = None
    _index: int = 0

    @classmethod
    def poll(cls, context):
        return len(context.selected_objects) > 0

    def invoke(self, context, event):
        self._items = list(context.selected_objects)
        self._index = 0
        wm = context.window_manager
        wm.progress_begin(0, len(self._items))
        self._timer = wm.event_timer_add(0.01, window=context.window)
        wm.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type in {'RIGHTMOUSE', 'ESC'}:
            self._cleanup(context)
            self.report({'INFO'}, "Processing cancelled")
            return {'CANCELLED'}

        if event.type == 'TIMER':
            if self._index >= len(self._items):
                self._cleanup(context)
                self.report({'INFO'}, f"Processed {len(self._items)} items")
                return {'FINISHED'}

            obj = self._items[self._index]
            self._process_item(context, obj)
            self._index += 1
            context.window_manager.progress_update(self._index)

        return {'PASS_THROUGH'}

    def _process_item(self, context, obj):
        # Per-item work here
        pass

    def _cleanup(self, context):
        wm = context.window_manager
        wm.progress_end()
        if self._timer is not None:
            wm.event_timer_remove(self._timer)
            self._timer = None

    def cancel(self, context):
        self._cleanup(context)
```

### Rules

- Timer interval is a minimum — actual intervals may be longer due to Blender processing
- `event_timer_add()` requires `window` as keyword: `event_timer_add(0.1, window=context.window)`
- Return `{'PASS_THROUGH'}` from TIMER events to let keyboard shortcuts work
- Return `{'RUNNING_MODAL'}` from TIMER events to block all other input
- ALWAYS create a shared `_cleanup()` method called from both FINISHED and CANCELLED paths

---

## Section 2: File Browser Integration

### Using ImportHelper / ExportHelper

```python
import bpy
import os
from bpy_extras.io_utils import ImportHelper, ExportHelper

class MYCAT_OT_import_data(bpy.types.Operator, ImportHelper):
    """Import data from file"""
    bl_idname = "mycat.import_data"
    bl_label = "Import Data"
    bl_options = {'REGISTER', 'UNDO'}

    # ImportHelper provides: filepath, invoke()
    filter_glob: bpy.props.StringProperty(
        default="*.csv;*.json",
        options={'HIDDEN'},
    )
    # Custom options shown in file browser sidebar
    skip_header: bpy.props.BoolProperty(name="Skip Header", default=True)

    def execute(self, context):
        if not os.path.isfile(self.filepath):
            self.report({'ERROR'}, f"File not found: {self.filepath}")
            return {'CANCELLED'}
        # Process self.filepath here
        self.report({'INFO'}, f"Imported: {self.filepath}")
        return {'FINISHED'}
```

### ExportHelper Requirements

```python
class MYCAT_OT_export_data(bpy.types.Operator, ExportHelper):
    bl_idname = "mycat.export_data"
    bl_label = "Export Data"

    filename_ext = ".csv"  # REQUIRED — ExportHelper.check() uses this

    filter_glob: bpy.props.StringProperty(default="*.csv", options={'HIDDEN'})

    def execute(self, context):
        with open(self.filepath, 'w') as f:
            f.write("data")
        return {'FINISHED'}
```

### Manual File Browser (without mixins)

Use `filepath: StringProperty(subtype='FILE_PATH')` or `directory: StringProperty(subtype='DIR_PATH')`, call `context.window_manager.fileselect_add(self)` in `invoke()`, return `{'RUNNING_MODAL'}`.

### Menu Registration

Register import operators to `bpy.types.TOPBAR_MT_file_import.append(menu_func)`. Register export operators to `bpy.types.TOPBAR_MT_file_export.append(menu_func)`. ALWAYS remove in `unregister()` to prevent duplicate entries on addon reload.

---

## Section 3: Undo/Redo Support

### bl_options for Undo

| Option | Effect |
|--------|--------|
| `{'REGISTER', 'UNDO'}` | One undo step on `{'FINISHED'}`. ALWAYS use for data-modifying operators. |
| `{'REGISTER', 'UNDO_GROUPED'}` | Consecutive calls of the same operator produce one undo step. Use for repeated rapid calls. |
| `{'REGISTER'}` alone | NO undo step. Use only for read-only / reporting operators. |

### Rules

1. `'UNDO'` requires `'REGISTER'` — without `'REGISTER'`, `'UNDO'` has NO effect
2. Returning `{'CANCELLED'}` NEVER creates an undo step
3. Modal operators with `{'UNDO'}` push ONE undo step when `{'FINISHED'}` is returned — all intermediate modifications are bundled
4. If `execute()` modifies data then returns `{'CANCELLED'}`, those changes persist WITHOUT being undoable — ALWAYS restore state before cancelling
5. `UNDO_GROUPED` consolidates consecutive calls of the SAME operator into one undo step — use for timer-driven modifications

### Cancel-Safe Pattern

```python
def execute(self, context):
    obj = context.active_object
    original_location = obj.location.copy()
    try:
        self._do_work(obj)
    except Exception as e:
        obj.location = original_location  # Restore on failure
        self.report({'ERROR'}, str(e))
        return {'CANCELLED'}
    return {'FINISHED'}
```

---

## Section 4: Batch Processing Operators

### Synchronous Batch (fast operations)

```python
class MYCAT_OT_batch_rename(bpy.types.Operator):
    bl_idname = "mycat.batch_rename"
    bl_label = "Batch Rename"
    bl_options = {'REGISTER', 'UNDO'}

    prefix: bpy.props.StringProperty(name="Prefix", default="obj_")

    def execute(self, context):
        objects = list(context.selected_objects)  # Copy — NEVER iterate and modify
        for i, obj in enumerate(objects):
            obj.name = f"{self.prefix}{i:03d}"
        self.report({'INFO'}, f"Renamed {len(objects)} objects")
        return {'FINISHED'}
```

### Batch with Context Override (Blender 4.0+)

```python
def batch_apply_modifiers(objects):
    """Apply all modifiers on multiple objects."""
    for obj in objects:
        with bpy.context.temp_override(active_object=obj, object=obj):
            for mod in obj.modifiers[:]:  # Copy list — modifiers removed during iteration
                try:
                    bpy.ops.object.modifier_apply(modifier=mod.name)
                except RuntimeError as e:
                    print(f"Cannot apply {mod.name} on {obj.name}: {e}")
```

### Rules

- ALWAYS copy collections before iterating if the loop modifies them: `list(context.selected_objects)`
- ALWAYS copy modifier lists before applying: `obj.modifiers[:]`
- Use `context.temp_override()` when calling operators that check `context.active_object`
- For batch operations on 100+ objects, use the modal timer pattern (Section 1) for responsiveness

---

## Section 5: Progress Reporting

### WindowManager Progress API

```python
wm = context.window_manager
wm.progress_begin(0, total_count)  # Initialize range
wm.progress_update(current_index)  # Update cursor indicator
wm.progress_end()                  # Finish — ALWAYS call, even on error
```

### Header Text as Progress (alternative)

```python
# In modal() during processing:
context.area.header_text_set(f"Processing: {i}/{total} ({i/total*100:.0f}%)")

# On completion or cancel — ALWAYS restore:
context.area.header_text_set(None)
```

### UILayout.progress() (Blender 4.0+)

```python
# In Panel.draw() or Operator.draw() only:
layout.progress(factor=0.66, type='BAR', text="66%")
```

### Known Issue

`wm.progress_end()` does not immediately reset the cursor — it stays as a busy indicator until the user moves the cursor. This is a confirmed Blender bug, not a coding error.

---

## Section 6: Multi-Step Workflows

### Parameter Dialog Before Execution

```python
class MYCAT_OT_configured_action(bpy.types.Operator):
    bl_idname = "mycat.configured_action"
    bl_label = "Configured Action"
    bl_options = {'REGISTER', 'UNDO'}

    count: bpy.props.IntProperty(name="Count", default=5, min=1, max=100)
    mode: bpy.props.EnumProperty(
        name="Mode",
        items=[('ADD', "Add", ""), ('REPLACE', "Replace", "")],
    )

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self, width=300)

    def check(self, context):
        return True  # Redraw dialog when properties change

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "mode", expand=True)
        layout.prop(self, "count")

    def execute(self, context):
        # Runs when user clicks OK
        return {'FINISHED'}
```

### State Machine Modal Operator

```python
class MYCAT_OT_two_point_tool(bpy.types.Operator):
    bl_idname = "mycat.two_point_tool"
    bl_label = "Two Point Tool"
    bl_options = {'REGISTER', 'UNDO'}

    _state: str = 'PICK_FIRST'
    _first_point: tuple = None

    def invoke(self, context, event):
        self._state = 'PICK_FIRST'
        self._first_point = None
        context.window_manager.modal_handler_add(self)
        context.area.header_text_set("Click first point (ESC to cancel)")
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type in {'RIGHTMOUSE', 'ESC'}:
            context.area.header_text_set(None)
            return {'CANCELLED'}

        if event.type == 'LEFTMOUSE' and event.value == 'PRESS':
            if self._state == 'PICK_FIRST':
                self._first_point = (event.mouse_region_x, event.mouse_region_y)
                self._state = 'PICK_SECOND'
                context.area.header_text_set("Click second point (ESC to cancel)")
                return {'RUNNING_MODAL'}

            elif self._state == 'PICK_SECOND':
                second = (event.mouse_region_x, event.mouse_region_y)
                self._execute_action(context, self._first_point, second)
                context.area.header_text_set(None)
                return {'FINISHED'}

        return {'PASS_THROUGH'}

    def _execute_action(self, context, p1, p2):
        pass  # Implementation here
```

### Invocation Methods Reference

| Method | Dialog | Calls execute() | Use Case |
|--------|--------|-----------------|----------|
| `invoke_props_dialog(self, width)` | Property form + OK | On OK click | Parameter gathering |
| `invoke_props_popup(self, event)` | Property popup | Immediately + on change | Interactive adjustment |
| `invoke_confirm(self, event)` | OK/Cancel only | On OK click | Destructive actions |
| `invoke_popup(self, width)` | Custom draw only | NEVER | Info display |
| `invoke_search_popup(self)` | Searchable enum | On selection | Enum with many items |

---

## Section 7: Deferred Execution with bpy.app.timers

### When to Use (instead of modal operators)

- Fire-and-forget deferred work (no UI interaction needed)
- Thread-safe bridge from background threads to main thread
- Persistent recurring tasks that survive file loads

### API

```python
bpy.app.timers.register(function, first_interval=0, persistent=False)
bpy.app.timers.unregister(function)
bpy.app.timers.is_registered(function) -> bool
```

### Return Value Semantics

| Return | Effect |
|--------|--------|
| `None` | Timer unregistered — function never called again |
| `float` | Function called again after that many seconds |

### Thread-Safe Queue Pattern

```python
import bpy
import queue
import threading

_queue = queue.Queue()

def _process_queue():
    while not _queue.empty():
        fn = _queue.get()
        fn()
    return 1.0  # Check every second

def run_on_main_thread(fn):
    """Schedule a function to run on Blender's main thread."""
    _queue.put(fn)

# Register in addon register():
bpy.app.timers.register(_process_queue, persistent=True)
```

### bpy.app.timers vs wm.event_timer_add

| | `bpy.app.timers` | `wm.event_timer_add` |
|--|-------------------|----------------------|
| Requires modal operator | No | Yes |
| Tied to specific window | No | Yes |
| Survives file load | Yes (persistent=True) | No |
| User-cancellable | No (must unregister) | Yes (ESC in modal) |
| Use case | Background tasks, thread bridge | Interactive modal tools |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for modal, timer, progress, and file browser APIs
- [references/examples.md](references/examples.md) — Full working examples for all implementation patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Implementation mistakes with explanations and fixes

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy_extras.io_utils.html
- https://docs.blender.org/api/current/bpy.types.WindowManager.html
- https://docs.blender.org/api/current/bpy.types.Timer.html

### Related Skills

- [blender-syntax-operators](../../syntax/blender-syntax-operators/SKILL.md) — Operator class structure, return values, bl_options, naming conventions
- [blender-core-api](../../core/blender-core-api/SKILL.md) — bpy module structure, context system, data access patterns
