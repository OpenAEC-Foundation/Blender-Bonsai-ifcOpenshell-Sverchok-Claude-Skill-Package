---
name: blender-syntax-operators
description: "Defines Blender operator creation patterns including bpy.types.Operator class, execute/invoke/modal methods, poll() functions, bl_idname naming, bl_options flags, return values, operator properties, and context.temp_override. Activates when creating custom Blender operators, registering tools, or handling operator context issues."
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-operators

## Quick Reference

### Critical Warnings

**ALWAYS** return a `set` from `execute()`, `invoke()`, and `modal()` — e.g., `{'FINISHED'}`, `{'CANCELLED'}`, `{'RUNNING_MODAL'}`. Returning a plain string or `None` crashes Blender.

**ALWAYS** use `context.temp_override()` in Blender 4.0+. Dict-based context overrides (`bpy.ops.foo(override_dict, ...)`) were REMOVED in 4.0.

**ALWAYS** set `bl_options = {'REGISTER', 'UNDO'}` for operators that modify scene data. Omitting `'UNDO'` means users cannot Ctrl+Z the operation.

**ALWAYS** implement `poll()` as a `@classmethod`. Forgetting `@classmethod` causes a `TypeError` at registration.

**NEVER** call `bpy.ops.*` inside `Panel.draw()` — draw callbacks are read-only. Use operator buttons via `layout.operator()` instead.

**NEVER** use uppercase letters in the category part of `bl_idname`. The format is `"category.operator_name"` — both parts MUST be lowercase with underscores.

**NEVER** store mutable state as class-level attributes on operators expecting per-instance behavior. Use `self` instance attributes set in `invoke()` or `execute()`, or use operator properties.

### Operator Decision Tree

```
Need to create a Blender operation?
│
├─ Runs once, no user interaction needed?
│  └─ Implement execute() only
│     └─ Set invoke = execute (optional shorthand)
│
├─ Needs a dialog/popup before running?
│  └─ Implement invoke() → wm.invoke_props_dialog(self)
│     └─ Implement draw() for dialog layout
│     └─ Implement execute() for the actual work
│
├─ Needs continuous event handling (drag, timer, mouse)?
│  └─ Implement invoke() → wm.modal_handler_add(self) + return {'RUNNING_MODAL'}
│     └─ Implement modal() for event processing
│     └─ Implement cancel() for cleanup
│
└─ Needs confirmation popup?
   └─ Implement invoke() → wm.invoke_confirm(self, event)
      └─ Implement execute() for the confirmed action
```

### Version Compatibility Matrix

| Feature | Blender 3.x | Blender 4.0+ | Blender 4.2+ |
|---------|-------------|--------------|--------------|
| Context override dict | `bpy.ops.foo(override, ...)` | REMOVED | REMOVED |
| `context.temp_override()` | Available from 3.2 | REQUIRED | REQUIRED |
| `'MODAL_PRIORITY'` bl_option | Not available | Not available | Available |
| `bl_idname` format | `"CAT.name"` or `"cat.name"` | `"cat.name"` (lowercase enforced) | `"cat.name"` |
| Extension manifest | `bl_info` dict | `bl_info` or `blender_manifest.toml` | `blender_manifest.toml` preferred |

---

## Essential Patterns

### Pattern 1: Minimal Operator (execute only)

```python
import bpy

class MYCAT_OT_simple_action(bpy.types.Operator):
    """Tooltip for the operator"""
    bl_idname = "mycat.simple_action"
    bl_label = "Simple Action"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def execute(self, context):
        context.active_object.location.z += 1.0
        self.report({'INFO'}, "Moved object up")
        return {'FINISHED'}

def register():
    bpy.utils.register_class(MYCAT_OT_simple_action)

def unregister():
    bpy.utils.unregister_class(MYCAT_OT_simple_action)
```

### Pattern 2: Operator with Properties

```python
class MYCAT_OT_scale_object(bpy.types.Operator):
    """Scale the active object by a specified factor"""
    bl_idname = "mycat.scale_object"
    bl_label = "Scale Object"
    bl_options = {'REGISTER', 'UNDO'}

    # Operator properties — shown in F9 redo panel and invoke dialogs
    factor: bpy.props.FloatProperty(
        name="Scale Factor",
        default=2.0,
        min=0.01,
        max=100.0,
        description="Factor to scale the object by",
    )
    uniform: bpy.props.BoolProperty(
        name="Uniform",
        default=True,
        description="Scale uniformly on all axes",
    )
    axis: bpy.props.EnumProperty(
        name="Axis",
        items=[
            ('X', "X", "Scale on X axis only"),
            ('Y', "Y", "Scale on Y axis only"),
            ('Z', "Z", "Scale on Z axis only"),
        ],
        default='X',
    )

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def execute(self, context):
        obj = context.active_object
        if self.uniform:
            obj.scale *= self.factor
        else:
            setattr(obj.scale, self.axis.lower(),
                    getattr(obj.scale, self.axis.lower()) * self.factor)
        return {'FINISHED'}
```

### Pattern 3: Invoke with Props Dialog

```python
class MYCAT_OT_create_grid(bpy.types.Operator):
    """Create a grid of objects with user-specified parameters"""
    bl_idname = "mycat.create_grid"
    bl_label = "Create Grid"
    bl_options = {'REGISTER', 'UNDO'}

    count_x: bpy.props.IntProperty(name="Count X", default=3, min=1, max=50)
    count_y: bpy.props.IntProperty(name="Count Y", default=3, min=1, max=50)
    spacing: bpy.props.FloatProperty(name="Spacing", default=2.0, min=0.1)

    def invoke(self, context, event):
        # Shows a dialog with the properties; execute() runs on OK
        return context.window_manager.invoke_props_dialog(self)

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "count_x")
        layout.prop(self, "count_y")
        layout.prop(self, "spacing")

    def execute(self, context):
        for x in range(self.count_x):
            for y in range(self.count_y):
                bpy.ops.mesh.primitive_cube_add(
                    location=(x * self.spacing, y * self.spacing, 0)
                )
        self.report({'INFO'}, f"Created {self.count_x * self.count_y} objects")
        return {'FINISHED'}
```

### Pattern 4: Modal Operator

```python
class MYCAT_OT_modal_draw(bpy.types.Operator):
    """Interactive modal operator with timer"""
    bl_idname = "mycat.modal_draw"
    bl_label = "Modal Draw"
    bl_options = {'REGISTER', 'UNDO'}

    _timer = None

    def modal(self, context, event):
        if event.type in {'RIGHTMOUSE', 'ESC'}:
            self.cancel(context)
            return {'CANCELLED'}

        if event.type == 'TIMER':
            # Periodic update logic here
            pass

        if event.type == 'LEFTMOUSE' and event.value == 'PRESS':
            # Finish on left click
            self.cancel(context)
            return {'FINISHED'}

        return {'PASS_THROUGH'}

    def invoke(self, context, event):
        self._timer = context.window_manager.event_timer_add(
            0.1, window=context.window
        )
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def cancel(self, context):
        if self._timer is not None:
            context.window_manager.event_timer_remove(self._timer)
            self._timer = None
```

### Pattern 5: Context Override (4.0+ required)

```python
# Override active object for operator execution
with bpy.context.temp_override(active_object=obj, selected_objects=[obj]):
    bpy.ops.object.shade_smooth()

# Override area type for viewport operators
def run_in_viewport(operator_call):
    """Execute an operator that requires a VIEW_3D area context."""
    for window in bpy.context.window_manager.windows:
        for area in window.screen.areas:
            if area.type == 'VIEW_3D':
                with bpy.context.temp_override(window=window, area=area):
                    return operator_call()
    raise RuntimeError("No VIEW_3D area found")

# Usage:
run_in_viewport(lambda: bpy.ops.view3d.snap_cursor_to_center())
```

### Pattern 6: Confirmation Dialog

```python
class MYCAT_OT_delete_all(bpy.types.Operator):
    """Delete all objects with confirmation"""
    bl_idname = "mycat.delete_all"
    bl_label = "Delete All Objects"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return len(bpy.data.objects) > 0

    def invoke(self, context, event):
        return context.window_manager.invoke_confirm(self, event)

    def execute(self, context):
        for obj in bpy.data.objects:
            bpy.data.objects.remove(obj)
        self.report({'WARNING'}, "All objects deleted")
        return {'FINISHED'}
```

---

## Common Operations

### Calling Operators from Scripts

```python
# Direct call — uses current context
bpy.ops.mesh.primitive_cube_add(size=2.0, location=(0, 0, 1))

# With context override (Blender 4.0+)
with bpy.context.temp_override(active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Boolean")

# Check if operator can run before calling
if bpy.ops.object.mode_set.poll():
    bpy.ops.object.mode_set(mode='EDIT')
```

### Registration and Menus

```python
# Register operator and add to menu
def menu_func(self, context):
    self.layout.operator(MYCAT_OT_simple_action.bl_idname)

def register():
    bpy.utils.register_class(MYCAT_OT_simple_action)
    bpy.types.VIEW3D_MT_object.append(menu_func)

def unregister():
    bpy.types.VIEW3D_MT_object.remove(menu_func)
    bpy.utils.unregister_class(MYCAT_OT_simple_action)
```

### Operator Reporting

```python
# Report types: 'DEBUG', 'INFO', 'OPERATOR', 'WARNING', 'ERROR', 'ERROR_INVALID_INPUT'
self.report({'INFO'}, "Operation completed")
self.report({'WARNING'}, "Check the result")
self.report({'ERROR'}, "Something went wrong")  # Shows in status bar, does NOT raise
```

### bl_idname Naming Convention

```
CLASS_OT_operator_name
│      │   │
│      │   └─ snake_case descriptive name
│      └───── OT = Operator Type (ALWAYS "OT" for operators)
└──────────── UPPERCASE category prefix (matches addon/module)
```

The `bl_idname` string format: `"category.operator_name"` — both parts lowercase.
The class name format: `CATEGORY_OT_operator_name` — category uppercase, rest snake_case.

```python
# CORRECT:
class MESH_OT_add_custom(bpy.types.Operator):
    bl_idname = "mesh.add_custom"        # lowercase.lowercase

# WRONG — bl_idname has uppercase:
class MESH_OT_add_custom(bpy.types.Operator):
    bl_idname = "Mesh.AddCustom"         # WILL FAIL at registration
```

### bl_options Reference

| Flag | Use When |
|------|----------|
| `'REGISTER'` | ALWAYS — makes operator visible in info log and F3 search |
| `'UNDO'` | Operator modifies scene data (objects, meshes, materials) |
| `'UNDO_GROUPED'` | Multiple rapid calls should be one undo step (e.g., timer-based updates) |
| `'BLOCKING'` | Modal operator should block ALL other event handlers |
| `'GRAB_CURSOR'` | Modal with mouse movement should wrap cursor at screen edges |
| `'GRAB_CURSOR_X'` | Wrap cursor on X axis only |
| `'GRAB_CURSOR_Y'` | Wrap cursor on Y axis only |
| `'INTERNAL'` | Operator should NOT appear in F3 search menu |
| `'PRESET'` | Show preset selector in operator properties panel |
| `'MACRO'` | Operator is a macro containing sub-operators |
| `'MODAL_PRIORITY'` | (4.2+) Receive events before other modal operators |

### Return Values

| Value | When to Use |
|-------|------------|
| `{'FINISHED'}` | Operator completed successfully |
| `{'CANCELLED'}` | Operator was cancelled, no changes made |
| `{'RUNNING_MODAL'}` | Operator is entering modal mode (from `invoke()`) |
| `{'PASS_THROUGH'}` | Modal: allow other operators to also handle this event |
| `{'INTERFACE'}` | Operator handled event but did not execute (popup shown) |

---

## Operator Method Signatures

```python
class bpy.types.Operator:
    bl_idname: str          # "category.name" — REQUIRED
    bl_label: str           # Display name — REQUIRED
    bl_description: str     # Tooltip (overrides docstring)
    bl_options: set[str]    # {'REGISTER', 'UNDO', ...}
    bl_translation_context: str  # i18n context
    bl_undo_group: str      # Group name for undo grouping

    @classmethod
    def poll(cls, context) -> bool: ...

    def execute(self, context) -> set[str]: ...
    def invoke(self, context, event) -> set[str]: ...
    def modal(self, context, event) -> set[str]: ...
    def draw(self, context) -> None: ...
    def cancel(self, context) -> None: ...

    def report(self, type: set[str], message: str) -> None: ...

    # Access layout for draw()
    layout: bpy.types.UILayout  # Available in draw()
```

### Event Object (used in invoke/modal)

```python
event.type      # str: 'LEFTMOUSE', 'RIGHTMOUSE', 'ESC', 'TIMER', 'A', 'B', etc.
event.value     # str: 'PRESS', 'RELEASE', 'CLICK', 'DOUBLE_CLICK', 'NOTHING'
event.mouse_x   # int: absolute mouse X position
event.mouse_y   # int: absolute mouse Y position
event.mouse_region_x  # int: mouse X relative to region
event.mouse_region_y  # int: mouse Y relative to region
event.shift     # bool: Shift held
event.ctrl      # bool: Ctrl held
event.alt       # bool: Alt held
event.oskey     # bool: OS/Super key held
```

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for Operator, WindowManager, Event, and registration functions
- [references/examples.md](references/examples.md) — Working code examples for all operator patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Common mistakes when writing operators, with fixes

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.ops.html
- https://docs.blender.org/api/current/bpy.types.Event.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/

### Related Skills

- [blender-core-api](../../core/blender-core-api/SKILL.md) — bpy module structure, context system, data access patterns
