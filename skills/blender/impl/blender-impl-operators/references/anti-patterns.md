# blender-impl-operators — Anti-Patterns

Implementation mistakes when building complex Blender operators. Each entry includes severity, broken code, fixed code, and explanation.

---

## AP-01: Leaking Timers on Finish or Cancel

**Severity:** HIGH — timer fires indefinitely, references deleted operator, memory leak

```python
# WRONG — timer not removed on FINISHED path
def modal(self, context, event):
    if event.type == 'TIMER':
        if self._done():
            return {'FINISHED'}  # Timer still firing!
    if event.type == 'ESC':
        return {'CANCELLED'}  # Timer still firing!
    return {'PASS_THROUGH'}

# CORRECT — shared cleanup for ALL exit paths
def modal(self, context, event):
    if event.type == 'TIMER':
        if self._done():
            self._cleanup(context)
            return {'FINISHED'}
    if event.type in {'RIGHTMOUSE', 'ESC'}:
        self._cleanup(context)
        return {'CANCELLED'}
    return {'PASS_THROUGH'}

def _cleanup(self, context):
    if self._timer is not None:
        context.window_manager.event_timer_remove(self._timer)
        self._timer = None
    context.window_manager.progress_end()
    context.area.header_text_set(None)

def cancel(self, context):
    self._cleanup(context)
```

**Why:** Timers are global WindowManager resources. If not removed, they continue generating events indefinitely. The deleted operator reference causes errors. ALWAYS use a shared `_cleanup()` method.

---

## AP-02: Not Restoring Header Text After Modal

**Severity:** MEDIUM — header permanently shows custom text

```python
# WRONG — header_text_set never restored
def modal(self, context, event):
    if event.type == 'TIMER':
        context.area.header_text_set(f"Working: {self._progress}")
    if event.type == 'ESC':
        return {'CANCELLED'}  # Header stuck with custom text!
    return {'PASS_THROUGH'}

# CORRECT — restore on ALL exit paths
def modal(self, context, event):
    if event.type in {'RIGHTMOUSE', 'ESC'}:
        context.area.header_text_set(None)  # Restore original
        return {'CANCELLED'}
    if event.type == 'TIMER':
        if self._done():
            context.area.header_text_set(None)  # Restore original
            return {'FINISHED'}
        context.area.header_text_set(f"Working: {self._progress}")
    return {'PASS_THROUGH'}
```

**Why:** `header_text_set()` overrides the area's header. If not restored to `None`, the custom text persists after the operator ends, confusing the user.

---

## AP-03: Missing ExportHelper filename_ext

**Severity:** BREAKING — AttributeError when user changes filename

```python
# WRONG — filename_ext not defined
class MY_OT_export(bpy.types.Operator, ExportHelper):
    bl_idname = "my.export"
    bl_label = "Export"
    # Missing: filename_ext = ".ext"

    def execute(self, context):
        return {'FINISHED'}
    # AttributeError in check() when filename changes

# CORRECT — always define filename_ext
class MY_OT_export(bpy.types.Operator, ExportHelper):
    bl_idname = "my.export"
    bl_label = "Export"
    filename_ext = ".csv"  # REQUIRED

    filter_glob: bpy.props.StringProperty(default="*.csv", options={'HIDDEN'})

    def execute(self, context):
        return {'FINISHED'}
```

**Why:** `ExportHelper.check()` reads `self.filename_ext` to correct the file extension. Omitting this attribute causes `AttributeError` when the user edits the filename in the file browser.

---

## AP-04: Modifying bpy.data from Background Thread

**Severity:** CRITICAL — crashes Blender (segfault)

```python
# WRONG — direct bpy access from thread
import threading

def background_work():
    # CRASHES: bpy is NOT thread-safe
    bpy.data.objects["Cube"].location.z += 1.0

thread = threading.Thread(target=background_work)
thread.start()

# CORRECT — use queue + bpy.app.timers
import queue
import threading

_queue = queue.Queue()

def _process_queue():
    while not _queue.empty():
        fn = _queue.get()
        fn()
    return 1.0

bpy.app.timers.register(_process_queue)

def background_work():
    result = heavy_computation()
    def apply():
        bpy.data.objects["Cube"].location.z = result
    _queue.put(apply)

thread = threading.Thread(target=background_work, daemon=True)
thread.start()
```

**Why:** Blender's `bpy` module is NOT thread-safe. All `bpy.data` modifications MUST happen on the main thread. Use `bpy.app.timers` with a `queue.Queue` to marshal work from background threads.

---

## AP-05: Using threading.Timer Instead of bpy.app.timers

**Severity:** HIGH — crashes or undefined behavior

```python
# WRONG — threading.Timer calls function on a background thread
import threading

def deferred_work():
    bpy.ops.mesh.primitive_cube_add()  # CRASH: not on main thread

threading.Timer(5.0, deferred_work).start()

# CORRECT — bpy.app.timers runs on Blender's main thread
def deferred_work():
    bpy.ops.mesh.primitive_cube_add()
    return None  # Run once

bpy.app.timers.register(deferred_work, first_interval=5.0)
```

**Why:** `threading.Timer` creates a new thread. `bpy.app.timers.register()` schedules execution on Blender's main thread, which is the ONLY thread where bpy calls are safe.

---

## AP-06: Expecting UI Updates During Synchronous execute()

**Severity:** MEDIUM — UI appears frozen, no progress visible to user

```python
# WRONG — UI frozen during execute(), progress invisible
def execute(self, context):
    wm = context.window_manager
    wm.progress_begin(0, 10000)
    for i in range(10000):
        self._heavy_work(i)
        wm.progress_update(i)  # Updates cursor but UI never redraws
    wm.progress_end()
    return {'FINISHED'}
    # User sees: frozen UI for N seconds, then sudden completion

# CORRECT — use modal operator with timer for visual progress
# (See Example 1 in references/examples.md)
```

**Why:** During a synchronous `execute()`, Blender's event loop is blocked. The viewport never redraws. `progress_update()` changes the cursor animation but the viewport is frozen. For visible progress, use a modal operator with timer events.

---

## AP-07: Iterating and Modifying Collections Simultaneously

**Severity:** HIGH — undefined behavior, skipped items, crashes

```python
# WRONG — modifying collection during iteration
def execute(self, context):
    for obj in context.selected_objects:
        bpy.data.objects.remove(obj)  # Modifies the list being iterated!

# WRONG — applying modifiers without copying list
def execute(self, context):
    obj = context.active_object
    for mod in obj.modifiers:
        bpy.ops.object.modifier_apply(modifier=mod.name)
        # modifiers list changes on each apply!

# CORRECT — copy to list first
def execute(self, context):
    objects = list(context.selected_objects)  # Snapshot
    for obj in objects:
        bpy.data.objects.remove(obj, do_unlink=True)

# CORRECT — copy modifier list
def execute(self, context):
    obj = context.active_object
    for mod in obj.modifiers[:]:  # Slice copy
        with bpy.context.temp_override(active_object=obj, object=obj):
            bpy.ops.object.modifier_apply(modifier=mod.name)
```

**Why:** `context.selected_objects`, `obj.modifiers`, and other Blender collections are live views of internal data. Modifying them during iteration causes items to be skipped, double-processed, or causes crashes.

---

## AP-08: Using UNDO Without REGISTER

**Severity:** MEDIUM — undo silently has no effect

```python
# WRONG — UNDO alone does nothing
class MY_OT_broken_undo(bpy.types.Operator):
    bl_idname = "my.broken_undo"
    bl_label = "Broken Undo"
    bl_options = {'UNDO'}  # No REGISTER = UNDO has no effect

    def execute(self, context):
        bpy.ops.mesh.primitive_cube_add()
        return {'FINISHED'}
    # User cannot undo this operation!

# CORRECT — REGISTER required for UNDO to work
class MY_OT_working_undo(bpy.types.Operator):
    bl_idname = "my.working_undo"
    bl_label = "Working Undo"
    bl_options = {'REGISTER', 'UNDO'}  # Both required

    def execute(self, context):
        bpy.ops.mesh.primitive_cube_add()
        return {'FINISHED'}
```

**Why:** `'UNDO'` depends on `'REGISTER'` to track the operation. Without `'REGISTER'`, Blender does not record the operation and `'UNDO'` is silently ignored.

---

## AP-09: Modifying Data Then Returning CANCELLED

**Severity:** MEDIUM — changes persist without undo possibility

```python
# WRONG — modifies data, then cancels (no undo step created)
def execute(self, context):
    obj = context.active_object
    obj.location.z += 10.0  # Data modified
    if not self._validate(obj):
        self.report({'ERROR'}, "Validation failed")
        return {'CANCELLED'}  # Changes persist without undo!

# CORRECT — restore state before cancelling
def execute(self, context):
    obj = context.active_object
    original_z = obj.location.z
    obj.location.z += 10.0
    if not self._validate(obj):
        obj.location.z = original_z  # Restore original state
        self.report({'ERROR'}, "Validation failed")
        return {'CANCELLED'}
    return {'FINISHED'}
```

**Why:** `{'CANCELLED'}` tells Blender "nothing happened" — no undo step is created. If data was already modified, those modifications persist but cannot be undone with Ctrl+Z. ALWAYS restore original state before returning `{'CANCELLED'}`.

---

## AP-10: Using progress_begin/end Without try/finally

**Severity:** LOW — cursor stays in busy state on error

```python
# WRONG — progress_end not called on exception
def execute(self, context):
    wm = context.window_manager
    wm.progress_begin(0, 100)
    for i in range(100):
        result = self._may_raise(i)  # Exception here = progress_end never called
        wm.progress_update(i)
    wm.progress_end()
    return {'FINISHED'}

# CORRECT — use try/finally
def execute(self, context):
    wm = context.window_manager
    wm.progress_begin(0, 100)
    try:
        for i in range(100):
            result = self._may_raise(i)
            wm.progress_update(i)
    except Exception as e:
        self.report({'ERROR'}, str(e))
        return {'CANCELLED'}
    finally:
        wm.progress_end()  # ALWAYS called
    return {'FINISHED'}
```

**Why:** If an exception occurs between `progress_begin()` and `progress_end()`, the cursor remains in busy state. Use `try/finally` to guarantee cleanup.

---

## AP-11: Forgetting check() Return Value in Dialog Operators

**Severity:** LOW — dialog does not update dynamically

```python
# WRONG — check() does not return True
class MY_OT_dialog(bpy.types.Operator):
    bl_idname = "my.dialog"
    bl_label = "Dialog"

    mode: bpy.props.EnumProperty(
        name="Mode", items=[('A', "A", ""), ('B', "B", "")],
    )

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self)

    def check(self, context):
        # Missing return True!
        pass  # Dialog never redraws when properties change

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "mode")
        if self.mode == 'B':
            layout.label(text="Extra options for B")  # Never updates!

# CORRECT
    def check(self, context):
        return True  # Triggers dialog redraw on property change
```

**Why:** `invoke_props_dialog` calls `check()` when properties change. If `check()` does not return `True`, the dialog is not redrawn, so conditional UI elements (from `draw()`) do not update.

---

## AP-12: Using area.header_text_set in Background Mode

**Severity:** MEDIUM — AttributeError (context.area is None)

```python
# WRONG — context.area is None in background mode
def invoke(self, context, event):
    context.area.header_text_set("Working...")  # AttributeError in --background

# CORRECT — check for area availability
def invoke(self, context, event):
    if context.area:
        context.area.header_text_set("Working...")
```

**Why:** In `blender --background` mode, there is no UI. `context.area` is `None`. ALWAYS guard `area.header_text_set()` calls with a None check when the operator may run in background mode.

---

## AP-13: Not Using Keyword Argument for event_timer_add window

**Severity:** BREAKING in some versions — wrong argument passed

```python
# WRONG — positional argument for window (ambiguous)
self._timer = wm.event_timer_add(0.1, context.window)

# CORRECT — explicit keyword
self._timer = wm.event_timer_add(0.1, window=context.window)
```

**Why:** In Blender 2.80+, `window` is a keyword-only argument. Passing it positionally may cause `TypeError` or pass it as wrong parameter. ALWAYS use the keyword form.

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy.types.WindowManager.html
- https://docs.blender.org/api/current/info_gotcha.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
