# blender-syntax-operators — Anti-Patterns

Common mistakes when creating Blender operators, with explanations and fixes.

---

## AP-01: Using Dict Context Override in Blender 4.0+

**Severity:** BREAKING — code will crash

```python
# WRONG — REMOVED in Blender 4.0
override = bpy.context.copy()
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="Subsurf")
# TypeError: Converting py args to operator properties: ...

# CORRECT — Blender 3.2+ / 4.0+ / 5.x
with bpy.context.temp_override(active_object=obj, object=obj):
    bpy.ops.object.modifier_apply(modifier="Subsurf")
```

**Why:** Blender 4.0 removed the dict-based context override mechanism entirely. The `temp_override()` context manager was introduced in 3.2 as the replacement and became REQUIRED in 4.0.

---

## AP-02: Forgetting @classmethod on poll()

**Severity:** BREAKING — registration fails

```python
# WRONG — missing @classmethod
class MY_OT_broken(bpy.types.Operator):
    bl_idname = "my.broken"
    bl_label = "Broken"

    def poll(self, context):  # Takes 'self' instead of 'cls'
        return True
    # TypeError during registration or unexpected behavior

# CORRECT
class MY_OT_fixed(bpy.types.Operator):
    bl_idname = "my.fixed"
    bl_label = "Fixed"

    @classmethod
    def poll(cls, context):
        return True
```

**Why:** Blender calls `poll()` on the class, not on an instance. Without `@classmethod`, Python passes an unexpected first argument.

---

## AP-03: Returning None or String from execute/invoke/modal

**Severity:** BREAKING — crashes Blender or produces RuntimeError

```python
# WRONG — returns None
def execute(self, context):
    context.active_object.location.z += 1.0
    # Forgot return statement

# WRONG — returns a string instead of a set
def execute(self, context):
    return 'FINISHED'  # String, not a set

# CORRECT
def execute(self, context):
    context.active_object.location.z += 1.0
    return {'FINISHED'}
```

**Why:** Blender expects a `set` of strings as the return value. Returning `None` or a plain string causes a `TypeError` or undefined behavior.

---

## AP-04: Uppercase Characters in bl_idname

**Severity:** BREAKING — registration fails in Blender 4.0+

```python
# WRONG — uppercase letters in bl_idname
class MY_OT_bad(bpy.types.Operator):
    bl_idname = "My.BadOperator"  # FAILS registration

# WRONG — mixed case
class MY_OT_bad2(bpy.types.Operator):
    bl_idname = "myTools.doSomething"  # FAILS in 4.0+

# CORRECT — all lowercase with underscores
class MY_OT_good(bpy.types.Operator):
    bl_idname = "my.good_operator"
```

**Why:** `bl_idname` MUST be `"category.operator_name"` — both parts lowercase, separated by a single dot. Blender 4.0+ strictly enforces this.

---

## AP-05: Not Handling Cancellation in Modal Operators

**Severity:** HIGH — user gets stuck in modal with no escape

```python
# WRONG — no ESC/RIGHTMOUSE handling
def modal(self, context, event):
    if event.type == 'LEFTMOUSE':
        return {'FINISHED'}
    return {'RUNNING_MODAL'}
    # User CANNOT cancel this operator — stuck forever

# CORRECT — always handle cancellation
def modal(self, context, event):
    if event.type in {'RIGHTMOUSE', 'ESC'}:
        self.cancel(context)
        return {'CANCELLED'}

    if event.type == 'LEFTMOUSE' and event.value == 'PRESS':
        return {'FINISHED'}

    return {'PASS_THROUGH'}
```

**Why:** Modal operators intercept ALL events. Without an escape mechanism, the user is locked into the modal state with no way to cancel.

---

## AP-06: Not Cleaning Up Timer in Modal cancel()

**Severity:** HIGH — memory leak, timer keeps firing after operator ends

```python
# WRONG — timer created but never removed
class MY_OT_leaked_timer(bpy.types.Operator):
    bl_idname = "my.leaked_timer"
    bl_label = "Leaked Timer"
    _timer = None

    def invoke(self, context, event):
        self._timer = context.window_manager.event_timer_add(0.1, window=context.window)
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'ESC':
            return {'CANCELLED'}  # Timer still running!
        return {'PASS_THROUGH'}

# CORRECT — cleanup in both cancel() and finish paths
class MY_OT_clean_timer(bpy.types.Operator):
    bl_idname = "my.clean_timer"
    bl_label = "Clean Timer"
    _timer = None

    def invoke(self, context, event):
        self._timer = context.window_manager.event_timer_add(0.1, window=context.window)
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'ESC':
            self.cancel(context)
            return {'CANCELLED'}
        if event.type == 'LEFTMOUSE':
            self.cancel(context)  # Also clean up on finish
            return {'FINISHED'}
        return {'PASS_THROUGH'}

    def cancel(self, context):
        if self._timer is not None:
            context.window_manager.event_timer_remove(self._timer)
            self._timer = None
```

**Why:** Timers are global resources managed by the WindowManager. If not explicitly removed, they continue generating events indefinitely, consuming resources and potentially causing errors.

---

## AP-07: Calling bpy.ops Inside Panel.draw()

**Severity:** HIGH — causes RuntimeError or undefined behavior

```python
# WRONG — calling operator in draw callback
class MY_PT_bad_panel(bpy.types.Panel):
    bl_label = "Bad Panel"
    bl_idname = "MY_PT_bad_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'

    def draw(self, context):
        if context.active_object:
            bpy.ops.object.shade_smooth()  # FORBIDDEN — draw is read-only
        self.layout.label(text="Panel")

# CORRECT — use layout.operator() to create a button
class MY_PT_good_panel(bpy.types.Panel):
    bl_label = "Good Panel"
    bl_idname = "MY_PT_good_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'

    def draw(self, context):
        self.layout.operator("object.shade_smooth", text="Smooth Shading")
```

**Why:** `Panel.draw()` is called on every UI redraw and runs in a restricted context. Calling operators or modifying data causes errors, infinite loops, or crashes.

---

## AP-08: Calling Operator Without Checking poll()

**Severity:** MEDIUM — RuntimeError at runtime

```python
# WRONG — blindly calling operator
bpy.ops.object.mode_set(mode='EDIT')
# RuntimeError if no active object or already in edit mode

# CORRECT — check poll first
if bpy.ops.object.mode_set.poll():
    bpy.ops.object.mode_set(mode='EDIT')
else:
    print("Cannot switch to edit mode in current context")
```

**Why:** Every operator has a `poll()` method that checks preconditions. Calling an operator when its poll fails raises `RuntimeError: Operator bpy.ops.X.poll() failed, context is incorrect`.

---

## AP-09: Using modal_handler_add Without Returning RUNNING_MODAL

**Severity:** BREAKING — modal never activates or crashes

```python
# WRONG — returns FINISHED after adding modal handler
def invoke(self, context, event):
    context.window_manager.modal_handler_add(self)
    return {'FINISHED'}  # Operator finishes immediately, modal() never called

# WRONG — returns RUNNING_MODAL without adding modal handler
def invoke(self, context, event):
    return {'RUNNING_MODAL'}  # No handler registered, events go nowhere

# CORRECT — add handler THEN return RUNNING_MODAL
def invoke(self, context, event):
    context.window_manager.modal_handler_add(self)
    return {'RUNNING_MODAL'}
```

**Why:** `modal_handler_add()` registers the operator for events, and `{'RUNNING_MODAL'}` tells Blender to keep the operator alive. Both are required — without the handler, events are lost; without RUNNING_MODAL, the operator terminates.

---

## AP-10: Using Assignment Instead of Annotation for Properties

**Severity:** BREAKING — properties silently ignored

```python
# WRONG — assignment syntax (=)
class MY_OT_broken_props(bpy.types.Operator):
    bl_idname = "my.broken_props"
    bl_label = "Broken"

    size = bpy.props.FloatProperty(name="Size", default=1.0)
    # Property is a class variable, NOT registered with Blender's RNA system
    # self.size returns the property descriptor, not a float value

# CORRECT — annotation syntax (:)
class MY_OT_working_props(bpy.types.Operator):
    bl_idname = "my.working_props"
    bl_label = "Working"

    size: bpy.props.FloatProperty(name="Size", default=1.0)
    # Property is properly registered, self.size returns the float value
```

**Why:** Blender uses Python annotations (`:`) to detect properties for RNA registration. Assignment (`=`) creates a regular class attribute that Blender ignores — the property won't appear in the UI and `self.size` will return the property descriptor object instead of the value.

---

## AP-11: Missing bl_options UNDO for Data-Modifying Operators

**Severity:** MEDIUM — user cannot undo the operation

```python
# WRONG — modifies data without UNDO flag
class MY_OT_no_undo(bpy.types.Operator):
    bl_idname = "my.no_undo"
    bl_label = "No Undo"
    bl_options = {'REGISTER'}  # Missing 'UNDO'

    def execute(self, context):
        bpy.ops.mesh.primitive_cube_add()  # Creates object with no undo step
        return {'FINISHED'}

# CORRECT
class MY_OT_with_undo(bpy.types.Operator):
    bl_idname = "my.with_undo"
    bl_label = "With Undo"
    bl_options = {'REGISTER', 'UNDO'}  # User can Ctrl+Z

    def execute(self, context):
        bpy.ops.mesh.primitive_cube_add()
        return {'FINISHED'}
```

**Why:** Without `'UNDO'` in `bl_options`, Blender does not create an undo step when the operator finishes. Users expect Ctrl+Z to work on any operation that changes the scene.

---

## AP-12: Storing State as Class Attributes Shared Across Instances

**Severity:** MEDIUM — subtle bugs with shared mutable state

```python
# WRONG — mutable class-level state shared across all invocations
class MY_OT_shared_state(bpy.types.Operator):
    bl_idname = "my.shared_state"
    bl_label = "Shared State"

    collected_objects = []  # Class attribute — shared across ALL instances

    def execute(self, context):
        self.collected_objects.append(context.active_object)
        # This list persists and grows across multiple operator calls
        return {'FINISHED'}

# CORRECT — initialize per-invocation state in invoke() or execute()
class MY_OT_instance_state(bpy.types.Operator):
    bl_idname = "my.instance_state"
    bl_label = "Instance State"

    def execute(self, context):
        collected = []  # Local variable — fresh each call
        collected.append(context.active_object)
        return {'FINISHED'}

# For modal operators, use instance attributes set in invoke():
class MY_OT_modal_state(bpy.types.Operator):
    bl_idname = "my.modal_state"
    bl_label = "Modal State"

    _items: list = None  # Type hint only, initialized in invoke

    def invoke(self, context, event):
        self._items = []  # Fresh list per invocation
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}
```

**Why:** Python class attributes are shared across all instances. For operators, Blender may reuse instances or create new ones unpredictably. Mutable class-level state (lists, dicts) accumulates data across calls, leading to subtle bugs.

---

## AP-13: Forgetting to Unregister Menu Functions

**Severity:** LOW — duplicate menu entries accumulate

```python
# WRONG — registers menu but never unregisters
def register():
    bpy.utils.register_class(MY_OT_tool)
    bpy.types.VIEW3D_MT_object.append(menu_func)

def unregister():
    bpy.utils.unregister_class(MY_OT_tool)
    # Forgot to remove menu_func — entry duplicates on addon reload

# CORRECT
def unregister():
    bpy.types.VIEW3D_MT_object.remove(menu_func)
    bpy.utils.unregister_class(MY_OT_tool)
```

**Why:** Menu append functions are stored in a list. If not removed on unregister, reloading the addon appends the function again, creating duplicate menu entries.

---

## AP-14: Running Viewport Operators Without Area Context

**Severity:** MEDIUM — RuntimeError in scripts and background mode

```python
# WRONG — calling viewport operator without VIEW_3D context
bpy.ops.view3d.snap_cursor_to_center()
# RuntimeError: Operator bpy.ops.view3d.snap_cursor_to_center.poll() failed

# CORRECT — provide viewport context
for window in bpy.context.window_manager.windows:
    for area in window.screen.areas:
        if area.type == 'VIEW_3D':
            with bpy.context.temp_override(window=window, area=area):
                bpy.ops.view3d.snap_cursor_to_center()
            break
```

**Why:** Operators in the `view3d` namespace require a VIEW_3D area in the context. Scripts, timers, and background mode do not have this by default.

---

## AP-15: Using invoke_props_dialog Without draw()

**Severity:** LOW — dialog shows auto-generated layout (may be acceptable)

```python
# SUBOPTIMAL — no draw(), dialog auto-generates from properties
class MY_OT_auto_layout(bpy.types.Operator):
    bl_idname = "my.auto_layout"
    bl_label = "Auto Layout"

    name: bpy.props.StringProperty(name="Name")
    count: bpy.props.IntProperty(name="Count")
    mode: bpy.props.EnumProperty(
        name="Mode",
        items=[('A', "A", ""), ('B', "B", "")],
    )

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self)

    def execute(self, context):
        return {'FINISHED'}
    # Auto-generated layout shows ALL properties in declaration order
    # This is acceptable for simple dialogs

# BETTER — custom draw() for complex dialogs
class MY_OT_custom_layout(bpy.types.Operator):
    bl_idname = "my.custom_layout"
    bl_label = "Custom Layout"

    name: bpy.props.StringProperty(name="Name")
    count: bpy.props.IntProperty(name="Count")
    mode: bpy.props.EnumProperty(
        name="Mode",
        items=[('A', "A", ""), ('B', "B", "")],
    )

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self)

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "name")
        row = layout.row()
        row.prop(self, "mode", expand=True)  # Radio buttons
        layout.prop(self, "count")

    def execute(self, context):
        return {'FINISHED'}
```

**Why:** Without `draw()`, Blender auto-generates the dialog layout from all properties in declaration order. For simple operators this is fine, but for complex dialogs with conditional fields or custom layouts, implementing `draw()` provides better UX.

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/info_gotcha.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://devtalk.blender.org/t/deprecationwarning-passing-in-context-overrides-is-deprecated/27870
