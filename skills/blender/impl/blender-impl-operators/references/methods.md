# blender-impl-operators — Method Reference

Complete API signatures for modal operators, timers, progress reporting, file browser, and deferred execution.

---

## WindowManager — Modal and Timer Methods

### modal_handler_add(operator) -> bool

```python
context.window_manager.modal_handler_add(self)
```

Registers the operator to receive modal events. MUST be called in `invoke()` before returning `{'RUNNING_MODAL'}`. Returns `True` on success.

**Rules:**
- MUST be called BEFORE returning `{'RUNNING_MODAL'}`
- Returning `{'RUNNING_MODAL'}` without calling this means events go nowhere
- Calling this and returning `{'FINISHED'}` means modal() is never called

### event_timer_add(time_step, *, window=None) -> Timer

```python
timer = context.window_manager.event_timer_add(0.1, window=context.window)
```

Creates a timer that generates `'TIMER'` events at the specified interval.

**Args:**
- `time_step`: float — interval in seconds between TIMER events (minimum, not exact)
- `window`: bpy.types.Window — keyword-only; window to attach timer to

**Returns:** `bpy.types.Timer` — store this for removal

**Rules:**
- `window` MUST be passed as keyword argument
- The interval is a minimum — actual intervals may be longer
- Store the returned Timer object as `self._timer` for cleanup

### event_timer_remove(timer) -> None

```python
context.window_manager.event_timer_remove(self._timer)
```

Removes a previously created timer. ALWAYS call in `cancel()`, on `{'FINISHED'}`, and on error paths.

---

## bpy.types.Timer — Properties

| Property | Type | Description |
|----------|------|-------------|
| `time_delta` | float | Seconds since last timer event fired |
| `time_duration` | float | Total seconds since timer was created |
| `time_step` | float | The interval that was set at creation |

---

## WindowManager — Progress Methods

### progress_begin(min, max) -> None

```python
context.window_manager.progress_begin(0, total_count)
```

Initiates progress tracking. Changes the cursor to a busy indicator.

**Args:**
- `min`: float — minimum progress value
- `max`: float — maximum progress value

### progress_update(value) -> None

```python
context.window_manager.progress_update(current_index)
```

Updates the current progress value.

**Args:**
- `value`: float — current progress (between min and max)

### progress_end() -> None

```python
context.window_manager.progress_end()
```

Terminates progress report. ALWAYS call, even on error/cancel paths. Use `try/finally`.

**Known issue:** Cursor does not immediately reset after `progress_end()` — stays as busy indicator until user moves cursor (confirmed Blender bug).

---

## WindowManager — Dialog Invocation Methods

### invoke_props_dialog(operator, width=300) -> set[str]

```python
context.window_manager.invoke_props_dialog(self, width=300)
```

Shows a dialog with operator properties. Calls `execute()` when user clicks OK.

**Args:**
- `operator`: Operator instance (use `self`)
- `width`: int — dialog width in pixels (default 300)

**Behavior:**
- Calls `draw()` to render dialog contents
- Calls `check(context)` when properties change (if defined)
- Calls `execute()` on OK
- Returns `{'CANCELLED'}` equivalent on Cancel / Escape

### invoke_props_popup(operator, event) -> set[str]

```python
context.window_manager.invoke_props_popup(self, event)
```

Shows a popup with properties. Calls `execute()` immediately AND on every property change. Non-blocking.

### invoke_confirm(operator, event) -> set[str]

```python
context.window_manager.invoke_confirm(self, event)
```

Shows OK/Cancel confirmation dialog. Calls `execute()` on OK.

### invoke_popup(operator, width=300) -> set[str]

```python
context.window_manager.invoke_popup(self, width=300)
```

Shows a popup that ONLY calls `draw()` — does NOT call `execute()` on close. Use for info display.

### invoke_search_popup(operator) -> None

```python
context.window_manager.invoke_search_popup(self)
```

Opens a searchable dropdown for an EnumProperty. Requires `bl_property` set on the operator class to the enum property name.

**Blender 4.0+ note:** `bl_property` must be explicitly set. Pre-4.0 defaulted to `"type"`.

---

## WindowManager — File Browser

### fileselect_add(operator) -> None

```python
context.window_manager.fileselect_add(self)
```

Opens the file browser, linked to the operator's `filepath` (or `directory`) property. Call in `invoke()`, return `{'RUNNING_MODAL'}`.

**Rules:**
- Operator MUST have a `filepath: StringProperty(subtype='FILE_PATH')` or `directory: StringProperty(subtype='DIR_PATH')` property
- Use `filter_glob: StringProperty(default="*.ext", options={'HIDDEN'})` to filter file types
- Semicolons separate multiple patterns: `"*.csv;*.json;*.txt"`

---

## bpy_extras.io_utils — Import/Export Helpers

### ImportHelper (mixin)

```python
from bpy_extras.io_utils import ImportHelper

class MY_OT_import(bpy.types.Operator, ImportHelper):
    # Provides: filepath (StringProperty), invoke() that calls fileselect_add
    filter_glob: bpy.props.StringProperty(default="*.ext", options={'HIDDEN'})
```

**Provided by mixin:**
- `filepath`: StringProperty — set by file browser on selection
- `invoke()`: calls `fileselect_add(self)` and returns `{'RUNNING_MODAL'}`

### ExportHelper (mixin)

```python
from bpy_extras.io_utils import ExportHelper

class MY_OT_export(bpy.types.Operator, ExportHelper):
    filename_ext = ".ext"  # REQUIRED — check() uses this to fix extension
    filter_glob: bpy.props.StringProperty(default="*.ext", options={'HIDDEN'})
```

**Provided by mixin:**
- `filepath`: StringProperty — set by file browser
- `filename_ext`: str — MUST be defined on subclass
- `invoke()`: calls `fileselect_add(self)` and returns `{'RUNNING_MODAL'}`
- `check(context)`: appends/corrects file extension when filename changes

**Rule:** Omitting `filename_ext` on an ExportHelper subclass causes `AttributeError` when `check()` is called.

---

## bpy.app.timers — Deferred Execution

### register(function, first_interval=0, persistent=False) -> None

```python
bpy.app.timers.register(my_callback, first_interval=5.0, persistent=True)
```

Registers a function to be called on Blender's main thread.

**Args:**
- `function`: callable() -> None | float
  - Return `None` to unregister (run once)
  - Return `float` to be called again after that many seconds
- `first_interval`: float — delay before first call (default: 0 = next main loop iteration)
- `persistent`: bool — survives file load when True (default: False)

### unregister(function) -> None

```python
bpy.app.timers.unregister(my_callback)
```

Removes a registered timer. Raises exception if function is not registered.

### is_registered(function) -> bool

```python
if bpy.app.timers.is_registered(my_callback):
    bpy.app.timers.unregister(my_callback)
```

Returns True if the function is currently registered as a timer.

---

## Operator check() Method

```python
def check(self, context) -> bool:
    """Called when operator properties change in a dialog.

    Returns:
        bool: True to trigger dialog redraw. False or None to skip redraw.

    Use case:
        - Validate property combinations
        - Update dependent properties
        - Show/hide conditional elements in draw()
    """
```

---

## UILayout.progress() (Blender 4.0+)

```python
layout.progress(factor=0.5, type='BAR', text="50%")
```

**Args:**
- `factor`: float — progress value from 0.0 to 1.0
- `type`: str — `'BAR'` (visual style)
- `text`: str — optional label displayed on the bar

**Restriction:** Only callable inside `draw()` contexts (Panel, Operator dialog). NOT available during `execute()`.

---

## context.area.header_text_set()

```python
context.area.header_text_set("Custom text")  # Set custom header
context.area.header_text_set(None)           # Restore original header
```

Use in modal operators to display status/progress in the editor header. ALWAYS restore with `None` on FINISHED or CANCELLED.

**Restriction:** `context.area` is None in background mode — check before using.

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.types.WindowManager.html
- https://docs.blender.org/api/current/bpy.types.Timer.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy_extras.io_utils.html
- https://docs.blender.org/api/current/bpy.types.UILayout.html
