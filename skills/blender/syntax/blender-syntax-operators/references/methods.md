# blender-syntax-operators — Method Reference

Complete API signatures for Blender operator creation, registration, and invocation.

---

## bpy.types.Operator — Class Attributes

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `bl_idname` | `str` | REQUIRED | Unique identifier. Format: `"category.operator_name"` (lowercase only) |
| `bl_label` | `str` | REQUIRED | Display name shown in menus and buttons |
| `bl_description` | `str` | Optional | Tooltip text. Overrides class docstring if set |
| `bl_options` | `set[str]` | Optional | Set of option flags. Default: `{'REGISTER'}` |
| `bl_translation_context` | `str` | Optional | Translation context for i18n |
| `bl_undo_group` | `str` | Optional | Name for grouping undo steps |

---

## bpy.types.Operator — Instance Methods

### poll(cls, context) -> bool

```python
@classmethod
def poll(cls, context):
    """Determine if operator can execute in the current context.

    Args:
        cls: The operator class (NOT instance — this is a classmethod).
        context: bpy.types.Context — current Blender context.

    Returns:
        bool: True if operator can run. False disables the button in UI
              and raises RuntimeError if called from script.

    Rules:
        - ALWAYS decorate with @classmethod
        - NEVER modify any data in poll() — it is called frequently for UI updates
        - NEVER raise exceptions — return False instead
        - Keep poll() fast — it runs on every UI redraw
    """
```

### execute(self, context) -> set[str]

```python
def execute(self, context):
    """Main operator logic. Called after invoke() or directly.

    Args:
        self: Operator instance. Access properties via self.property_name.
        context: bpy.types.Context — current Blender context.

    Returns:
        set[str]: MUST be one of:
            {'FINISHED'}   — operation completed successfully
            {'CANCELLED'}  — operation was cancelled
            {'RUNNING_MODAL'} — entering modal mode (rare from execute)

    Rules:
        - ALWAYS return a set, NEVER return None or a plain string
        - Access operator properties via self (e.g., self.size)
        - Use self.report() to communicate results to the user
    """
```

### invoke(self, context, event) -> set[str]

```python
def invoke(self, context, event):
    """Called when user triggers the operator (button, menu, shortcut).

    Args:
        self: Operator instance.
        context: bpy.types.Context.
        event: bpy.types.Event — mouse position, modifier keys, etc.

    Returns:
        set[str]: MUST be one of:
            {'FINISHED'}       — done immediately
            {'CANCELLED'}      — abort
            {'RUNNING_MODAL'}  — entering modal mode
            {'INTERFACE'}      — UI shown but not executed yet

    Common patterns:
        return self.execute(context)                           # Direct execution
        return context.window_manager.invoke_props_dialog(self) # Show dialog
        return context.window_manager.invoke_confirm(self, event) # Confirm popup
        context.window_manager.modal_handler_add(self)         # Enter modal
        return {'RUNNING_MODAL'}

    Rules:
        - If invoke() is not defined, Blender calls execute() directly
        - Store event data needed later (e.g., self._init_mouse = event.mouse_x)
        - For modal: MUST call modal_handler_add() before returning RUNNING_MODAL
    """
```

### modal(self, context, event) -> set[str]

```python
def modal(self, context, event):
    """Called repeatedly for each event while operator is modal.

    Args:
        self: Operator instance.
        context: bpy.types.Context.
        event: bpy.types.Event — the current event to process.

    Returns:
        set[str]: MUST be one of:
            {'RUNNING_MODAL'} — continue receiving events
            {'FINISHED'}      — done, stop modal
            {'CANCELLED'}     — abort, stop modal
            {'PASS_THROUGH'}  — let other handlers also process this event

    Rules:
        - ALWAYS handle ESC/RIGHTMOUSE to allow user cancellation
        - ALWAYS clean up resources (timers, draw handlers) on finish/cancel
        - Return PASS_THROUGH for events you don't handle
        - Avoid heavy computation — modal() is called per-event
    """
```

### draw(self, context) -> None

```python
def draw(self, context):
    """Draw the operator's properties in a dialog or redo panel (F9).

    Args:
        self: Operator instance. self.layout is the UILayout.
        context: bpy.types.Context.

    Returns:
        None

    Rules:
        - NEVER modify data in draw() — it is a read-only callback
        - Use self.layout.prop(self, "property_name") to draw properties
        - If not defined, Blender auto-generates layout from properties
    """
```

### cancel(self, context) -> None

```python
def cancel(self, context):
    """Called when a modal operator is cancelled (ESC, RIGHTMOUSE, or system).

    Args:
        self: Operator instance.
        context: bpy.types.Context.

    Returns:
        None

    Rules:
        - ALWAYS remove timers: context.window_manager.event_timer_remove(self._timer)
        - ALWAYS remove draw handlers if added
        - Restore any temporary state changes
    """
```

### report(self, type, message) -> None

```python
def report(self, type, message):
    """Display a message to the user in the status bar / info editor.

    Args:
        type: set[str] — one of:
            {'DEBUG'}               — debug output (not shown to user)
            {'INFO'}                — information message (blue)
            {'OPERATOR'}            — operator log
            {'WARNING'}             — warning message (yellow)
            {'ERROR'}               — error message (red, does NOT raise exception)
            {'ERROR_INVALID_INPUT'} — invalid input error
        message: str — the message text.

    Rules:
        - type MUST be a set (e.g., {'INFO'}), not a string
        - {'ERROR'} does NOT raise a Python exception — the operator continues
        - Use return {'CANCELLED'} after reporting an error to stop execution
    """
```

---

## bpy.types.Event — Properties

| Property | Type | Description |
|----------|------|-------------|
| `type` | `str` | Event type: `'LEFTMOUSE'`, `'RIGHTMOUSE'`, `'MIDDLEMOUSE'`, `'MOUSEMOVE'`, `'WHEELUPMOUSE'`, `'WHEELDOWNMOUSE'`, `'ESC'`, `'RET'`, `'SPACE'`, `'TIMER'`, `'A'`..`'Z'`, `'ZERO'`..`'NINE'`, `'F1'`..`'F19'` |
| `value` | `str` | `'PRESS'`, `'RELEASE'`, `'CLICK'`, `'DOUBLE_CLICK'`, `'CLICK_DRAG'`, `'NOTHING'` |
| `mouse_x` | `int` | Absolute mouse X position (window coordinates) |
| `mouse_y` | `int` | Absolute mouse Y position (window coordinates) |
| `mouse_region_x` | `int` | Mouse X position relative to current region |
| `mouse_region_y` | `int` | Mouse Y position relative to current region |
| `mouse_prev_x` | `int` | Previous absolute mouse X |
| `mouse_prev_y` | `int` | Previous absolute mouse Y |
| `pressure` | `float` | Tablet pressure (0.0 to 1.0) |
| `tilt` | `tuple[float, float]` | Tablet tilt (X, Y) |
| `shift` | `bool` | Shift key held |
| `ctrl` | `bool` | Ctrl key held |
| `alt` | `bool` | Alt key held |
| `oskey` | `bool` | OS/Super/Windows key held |
| `is_tablet` | `bool` | Event from tablet device |
| `is_mouse_absolute` | `bool` | Mouse position is in absolute screen coordinates |

---

## WindowManager — Operator Invocation Methods

### invoke_props_dialog(operator, width=300) -> set[str]

```python
context.window_manager.invoke_props_dialog(self, width=300)
```

Shows a dialog with the operator's properties. When user clicks OK, `execute()` is called. Returns `{'RUNNING_MODAL'}` internally.

**Args:**
- `operator`: The operator instance (use `self`)
- `width`: Dialog width in pixels (default 300)

### invoke_props_popup(operator, event) -> set[str]

```python
context.window_manager.invoke_props_popup(self, event)
```

Shows a popup with properties. Calls `execute()` immediately AND when properties change. Useful for interactive adjustment.

### invoke_confirm(operator, event) -> set[str]

```python
context.window_manager.invoke_confirm(self, event)
```

Shows "OK?" confirmation dialog. Calls `execute()` on confirm.

### invoke_popup(operator, width=300) -> set[str]

```python
context.window_manager.invoke_popup(self, width=300)
```

Shows a popup that only calls `draw()` — does NOT call `execute()` on close. Useful for info popups.

### modal_handler_add(operator) -> bool

```python
context.window_manager.modal_handler_add(self)
```

Registers the operator to receive modal events. MUST be called in `invoke()` before returning `{'RUNNING_MODAL'}`. Returns `True` on success.

### event_timer_add(time_step, window) -> Timer

```python
timer = context.window_manager.event_timer_add(0.1, window=context.window)
```

Creates a timer that generates `'TIMER'` events at the specified interval (seconds). Store the returned timer for removal.

### event_timer_remove(timer) -> None

```python
context.window_manager.event_timer_remove(self._timer)
```

Removes a previously created timer. ALWAYS call in `cancel()` and on `{'FINISHED'}`.

---

## Registration Functions

### bpy.utils.register_class(cls)

```python
bpy.utils.register_class(MYCAT_OT_simple_action)
```

Registers a class with Blender's RNA system. The class MUST be a subclass of a `bpy.types` type (Operator, Panel, Menu, PropertyGroup, etc.).

**Rules:**
- Registration order matters: register dependencies (PropertyGroup) BEFORE dependents (Operator that uses them)
- Raises `ValueError` if class is already registered
- Raises `TypeError` if `bl_idname` format is invalid

### bpy.utils.unregister_class(cls)

```python
bpy.utils.unregister_class(MYCAT_OT_simple_action)
```

Unregisters a class. ALWAYS unregister in reverse order of registration.

### bpy.utils.register_classes_factory(classes)

```python
classes = (MYCAT_OT_action_one, MYCAT_OT_action_two, MYCAT_PT_panel)
register, unregister = bpy.utils.register_classes_factory(classes)
```

Returns a `(register, unregister)` tuple that handles all classes. Registration order follows tuple order; unregistration is reversed.

---

## bpy.ops Invocation

### Calling Operators

```python
# Standard call
bpy.ops.mesh.primitive_cube_add(size=2.0)

# With poll check
if bpy.ops.object.mode_set.poll():
    bpy.ops.object.mode_set(mode='EDIT')

# Returns: {'FINISHED'}, {'CANCELLED'}, or {'RUNNING_MODAL'}
result = bpy.ops.object.select_all(action='SELECT')
```

### context.temp_override() (Blender 3.2+, REQUIRED in 4.0+)

```python
# Signature
bpy.context.temp_override(
    window=None,      # bpy.types.Window
    area=None,        # bpy.types.Area
    region=None,      # bpy.types.Region
    **kw              # Any context attribute (active_object, selected_objects, etc.)
)
```

**Rules:**
- Region MUST belong to the specified (or current) area
- Area MUST belong to the specified (or current) window
- Cannot switch to/from fullscreen areas
- NEVER use dict-based overrides in Blender 4.0+ — they are REMOVED

---

## Operator Properties (bpy.props on operators)

Operator properties are defined as class annotations and appear in the F9 redo panel and invoke dialogs.

```python
class MY_OT_example(bpy.types.Operator):
    bl_idname = "my.example"
    bl_label = "Example"

    # Properties available via self.name, self.count, etc.
    name: bpy.props.StringProperty(name="Name", default="")
    count: bpy.props.IntProperty(name="Count", default=1, min=0)
    size: bpy.props.FloatProperty(name="Size", default=1.0)
    enabled: bpy.props.BoolProperty(name="Enabled", default=True)
    mode: bpy.props.EnumProperty(
        name="Mode",
        items=[
            ('ADD', "Add", "Add mode"),
            ('REMOVE', "Remove", "Remove mode"),
        ],
    )
```

**Rules:**
- Properties MUST use annotation syntax (`:`) not assignment (`=`)
- Properties are passed as keyword arguments when calling operator from script: `bpy.ops.my.example(count=5)`
- Properties persist in the F9 redo panel until the next operator runs
- `'SKIP_SAVE'` in property `options` prevents the value from being saved in presets/redo

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.types.Event.html
- https://docs.blender.org/api/current/bpy.ops.html
- https://docs.blender.org/api/current/bpy.types.WindowManager.html
- https://docs.blender.org/api/current/bpy.utils.html
- https://docs.blender.org/api/current/bpy.props.html
