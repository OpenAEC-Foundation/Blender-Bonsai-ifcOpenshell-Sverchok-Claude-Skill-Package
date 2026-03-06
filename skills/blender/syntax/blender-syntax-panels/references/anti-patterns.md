# blender-syntax-panels: Anti-Patterns

These are confirmed error patterns for Blender UI panel development. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/bpy.types.Panel.html
- https://docs.blender.org/api/current/bpy.types.UILayout.html
- vooronderzoek-blender.md §6 (UI Panels)

---

## AP-001: Modifying Data Inside Panel.draw()

**WHY this is wrong**: `draw()` is called every UI redraw (multiple times per second). Modifying properties triggers another redraw, creating an infinite loop. Blender's draw contract forbids side effects during UI rendering.

```python
# WRONG — modifying scene data in draw callback
class MY_PT_bad(bpy.types.Panel):
    bl_label = "Bad Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Bad"

    def draw(self, context):
        context.scene.frame_current += 1              # INFINITE LOOP
        context.active_object.location.x = 0          # FORBIDDEN
        bpy.ops.object.select_all(action='SELECT')    # FORBIDDEN
        self.layout.label(text="This panel is broken")
```

```python
# CORRECT — draw() only reads data and builds UI elements
class MY_PT_good(bpy.types.Panel):
    bl_label = "Good Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Good"

    def draw(self, context):
        layout = self.layout
        layout.label(text=f"Frame: {context.scene.frame_current}")  # Read-only
        layout.operator("my.advance_frame")  # User triggers action, not draw()
```

ALWAYS keep `draw()` free of data modifications and operator calls. ALWAYS use `layout.operator()` to let the user trigger actions.

---

## AP-002: Missing bl_space_type or bl_region_type

**WHY this is wrong**: Both `bl_space_type` and `bl_region_type` are REQUIRED for panel registration. Omitting either causes `RuntimeError` during `bpy.utils.register_class()`. Blender cannot determine where to display the panel.

```python
# WRONG — missing bl_region_type
class MY_PT_incomplete(bpy.types.Panel):
    bl_label = "Broken Panel"
    bl_idname = "MY_PT_incomplete"
    bl_space_type = 'VIEW_3D'
    # Missing bl_region_type — registration fails!
    bl_category = "My Tab"

    def draw(self, context):
        self.layout.label(text="Never displayed")
```

```python
# CORRECT — all required class variables present
class MY_PT_complete(bpy.types.Panel):
    bl_label = "Working Panel"
    bl_idname = "MY_PT_complete"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        self.layout.label(text="Displayed correctly")
```

ALWAYS define `bl_space_type`, `bl_region_type`, and `bl_category` (when region is `'UI'`).

---

## AP-003: Wrong Naming Convention for bl_idname

**WHY this is wrong**: Blender enforces naming conventions for registered types. Panel `bl_idname` MUST contain `_PT_`, Menu MUST contain `_MT_`, UIList MUST contain `_UL_`. Using wrong tags or no tags causes registration errors or unexpected behavior.

```python
# WRONG — missing type tag
class MyPanel(bpy.types.Panel):
    bl_idname = "my_panel"         # Missing _PT_ — registration error
    bl_label = "My Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

# WRONG — wrong type tag for a panel
class MyPanel(bpy.types.Panel):
    bl_idname = "MY_MT_panel"      # _MT_ is for Menu, not Panel
    bl_label = "My Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"
```

```python
# CORRECT — proper naming convention
class MY_PT_panel(bpy.types.Panel):
    bl_idname = "MY_PT_panel"      # Prefix_PT_name
    bl_label = "My Panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

class MY_MT_menu(bpy.types.Menu):
    bl_idname = "MY_MT_menu"       # Prefix_MT_name
    bl_label = "My Menu"

class MY_UL_list(bpy.types.UIList):
    bl_idname = "MY_UL_list"       # Prefix_UL_name
```

ALWAYS use `_PT_` for panels, `_MT_` for menus, `_UL_` for UILists, `_HT_` for headers, `_OT_` for operators.

---

## AP-004: Sub-Panel with Mismatched Space/Region/Category

**WHY this is wrong**: A sub-panel (child panel with `bl_parent_id`) MUST have the same `bl_space_type`, `bl_region_type`, and `bl_category` as its parent. Mismatched values cause the sub-panel to not appear or to appear in the wrong location.

```python
# WRONG — child has different bl_category than parent
class MY_PT_parent(bpy.types.Panel):
    bl_label = "Parent"
    bl_idname = "MY_PT_parent"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"
    def draw(self, context): pass

class MY_PT_child(bpy.types.Panel):
    bl_label = "Child"
    bl_idname = "MY_PT_child"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Other Tab"       # MISMATCH — child won't appear as sub-panel
    bl_parent_id = "MY_PT_parent"
    def draw(self, context): pass
```

```python
# CORRECT — all values match parent
class MY_PT_child_correct(bpy.types.Panel):
    bl_label = "Child"
    bl_idname = "MY_PT_child_correct"
    bl_space_type = 'VIEW_3D'       # Same as parent
    bl_region_type = 'UI'           # Same as parent
    bl_category = "My Tab"          # Same as parent
    bl_parent_id = "MY_PT_parent"
    def draw(self, context): pass
```

ALWAYS match `bl_space_type`, `bl_region_type`, and `bl_category` between parent and child panels.

---

## AP-005: Using layout.panel() in Blender < 4.1

**WHY this is wrong**: `UILayout.panel()` was added in Blender 4.1. Calling it in earlier versions raises `AttributeError`. Code targeting Blender 3.x or 4.0 MUST use the sub-panel class pattern instead.

```python
# WRONG — crashes in Blender < 4.1
class MY_PT_bad_panel(bpy.types.Panel):
    bl_label = "Panel"
    bl_idname = "MY_PT_bad_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tab"

    def draw(self, context):
        header, body = self.layout.panel("section_id")  # AttributeError in < 4.1
```

```python
# CORRECT — version-guarded
class MY_PT_versioned(bpy.types.Panel):
    bl_label = "Panel"
    bl_idname = "MY_PT_versioned"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tab"

    def draw(self, context):
        layout = self.layout
        if bpy.app.version >= (4, 1, 0):
            header, body = layout.panel("section_id")
            header.label(text="Section")
            if body is not None:
                body.label(text="Content")
        else:
            # Fallback: use box or separate sub-panel class
            box = layout.box()
            box.label(text="Section")
            box.label(text="Content")
```

ALWAYS guard `layout.panel()` calls with a version check when supporting Blender < 4.1. ALWAYS use `bpy.app.version >= (4, 1, 0)`.

---

## AP-006: Forgetting to Check None in draw() When Using context.active_object

**WHY this is wrong**: `context.active_object` can be `None` when no object is selected or in certain contexts. Accessing properties on `None` raises `AttributeError` and causes Blender to disable the panel with an error message.

```python
# WRONG — no None check, crashes when nothing is selected
class MY_PT_crash(bpy.types.Panel):
    bl_label = "Crash Panel"
    bl_idname = "MY_PT_crash"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tab"

    def draw(self, context):
        layout = self.layout
        obj = context.active_object
        layout.prop(obj, "name")  # AttributeError when obj is None
```

```python
# CORRECT option 1 — guard inside draw()
class MY_PT_safe(bpy.types.Panel):
    bl_label = "Safe Panel"
    bl_idname = "MY_PT_safe"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tab"

    def draw(self, context):
        layout = self.layout
        obj = context.active_object
        if obj is None:
            layout.label(text="No object selected", icon='INFO')
            return
        layout.prop(obj, "name")

# CORRECT option 2 — use poll() to hide panel entirely
class MY_PT_polled(bpy.types.Panel):
    bl_label = "Polled Panel"
    bl_idname = "MY_PT_polled"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tab"

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def draw(self, context):
        layout = self.layout
        layout.prop(context.active_object, "name")  # Safe — poll guarantees non-None
```

ALWAYS check `context.active_object is not None` before accessing its properties, either via `poll()` or an early return in `draw()`.

---

## AP-007: Wrong Registration Order

**WHY this is wrong**: Panels reference Operators (via `layout.operator()`), UILists (via `template_list()`), and Menus (via `layout.menu()`). If these are not registered before the Panel, UI elements referencing them show errors. PropertyGroups referenced by PointerProperty MUST be registered before the class that uses the PointerProperty.

```python
# WRONG — Panel registered before its dependencies
def register():
    bpy.utils.register_class(MY_PT_main_panel)        # Panel first — ERROR
    bpy.utils.register_class(MY_UL_items)              # UIList after Panel
    bpy.utils.register_class(MY_OT_add_item)           # Operator after Panel
    bpy.utils.register_class(MY_ListItem)              # PropertyGroup last — CRASH
    bpy.types.Scene.items = bpy.props.PointerProperty(type=MY_ListItem)
```

```python
# CORRECT — dependency order (PropertyGroups → Operators → UILists → Menus → Panels)
def register():
    bpy.utils.register_class(MY_ListItem)              # 1. PropertyGroups
    bpy.utils.register_class(MY_OT_add_item)           # 2. Operators
    bpy.utils.register_class(MY_UL_items)              # 3. UILists
    bpy.utils.register_class(MY_MT_context_menu)       # 4. Menus
    bpy.utils.register_class(MY_PT_main_panel)         # 5. Panels (last)
    bpy.types.Scene.items = bpy.props.PointerProperty(type=MY_ListItem)

def unregister():
    del bpy.types.Scene.items
    bpy.utils.unregister_class(MY_PT_main_panel)       # Reverse order
    bpy.utils.unregister_class(MY_MT_context_menu)
    bpy.utils.unregister_class(MY_UL_items)
    bpy.utils.unregister_class(MY_OT_add_item)
    bpy.utils.unregister_class(MY_ListItem)
```

ALWAYS register in dependency order. ALWAYS unregister in reverse order. ALWAYS register PropertyGroups before any class that references them.

---

## AP-008: Using 'WINDOW' Region for Sidebar Panels

**WHY this is wrong**: `bl_region_type = 'WINDOW'` places panels in the main editor area (used by Properties editor panels and popover panels). For sidebar panels in 3D Viewport, Node Editor, etc., you MUST use `'UI'`. Using `'WINDOW'` in `VIEW_3D` causes the panel to not appear.

```python
# WRONG — panel intended for sidebar but using WINDOW region
class MY_PT_wrong_region(bpy.types.Panel):
    bl_label = "My Panel"
    bl_idname = "MY_PT_wrong_region"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'WINDOW'       # WRONG for sidebar — panel won't appear
    bl_category = "My Tab"
    def draw(self, context): pass
```

```python
# CORRECT — 'UI' for sidebar panels
class MY_PT_correct_region(bpy.types.Panel):
    bl_label = "My Panel"
    bl_idname = "MY_PT_correct_region"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'           # Correct for N-panel sidebar
    bl_category = "My Tab"
    def draw(self, context): pass

# CORRECT — 'WINDOW' for Properties editor
class MY_PT_props_panel(bpy.types.Panel):
    bl_label = "My Properties"
    bl_idname = "MY_PT_props_panel"
    bl_space_type = 'PROPERTIES'
    bl_region_type = 'WINDOW'       # Correct for Properties editor
    bl_context = "object"
    def draw(self, context): pass
```

ALWAYS use `'UI'` for sidebar panels. ONLY use `'WINDOW'` with `bl_space_type = 'PROPERTIES'` or for popover panels with `bl_options = {'INSTANCED'}`.

---

## AP-009: Not Removing Appended Menu Functions in unregister()

**WHY this is wrong**: When an addon appends a draw function to a built-in menu using `bpy.types.MENU_NAME.append()`, this function persists even after the addon is disabled. On next reload, it appends again, causing duplicate menu entries. Eventually, the UI accumulates many duplicate items.

```python
# WRONG — append without corresponding remove
def draw_my_item(self, context):
    self.layout.operator("my.action")

def register():
    bpy.utils.register_class(MY_PT_panel)
    bpy.types.VIEW3D_MT_object.append(draw_my_item)

def unregister():
    bpy.utils.unregister_class(MY_PT_panel)
    # Missing: bpy.types.VIEW3D_MT_object.remove(draw_my_item)
```

```python
# CORRECT — always remove in unregister()
def draw_my_item(self, context):
    self.layout.operator("my.action")

def register():
    bpy.utils.register_class(MY_PT_panel)
    bpy.types.VIEW3D_MT_object.append(draw_my_item)

def unregister():
    bpy.types.VIEW3D_MT_object.remove(draw_my_item)  # MUST match append
    bpy.utils.unregister_class(MY_PT_panel)
```

ALWAYS call `.remove()` in `unregister()` for every `.append()` or `.prepend()` in `register()`. ALWAYS keep the draw function as a named function (not a lambda) so it can be removed.

---

## AP-010: Setting Operator Properties After layout.operator() Return

**WHY this is wrong**: `layout.operator()` returns an `OperatorProperties` object for setting properties. This object is only valid immediately after the call. Storing it and setting properties later (e.g., in a loop after creating all buttons) can lead to properties being set on the wrong operator or being lost.

```python
# WRONG — storing operator properties and setting them later
buttons = []
for name in ["A", "B", "C"]:
    op = layout.operator("my.action", text=name)
    buttons.append(op)
# Later...
for i, op in enumerate(buttons):
    op.index = i  # May not work correctly — set IMMEDIATELY after operator()
```

```python
# CORRECT — set properties immediately after each operator() call
for i, name in enumerate(["A", "B", "C"]):
    op = layout.operator("my.action", text=name)
    op.index = i  # Set immediately — guaranteed to work
```

ALWAYS set operator properties immediately after `layout.operator()`. NEVER store the returned `OperatorProperties` for later modification.

---

## AP-011: Using layout.template_list() with Wrong Property References

**WHY this is wrong**: `template_list()` requires exact string names of properties on the data object. Passing the wrong property name, a non-existent property, or the collection object itself instead of its name string causes runtime errors or blank lists.

```python
# WRONG — passing the collection object instead of its name
class MY_PT_bad_list(bpy.types.Panel):
    def draw(self, context):
        settings = context.scene.my_settings
        # WRONG: passing settings.items (the collection) instead of "items" (the string name)
        self.layout.template_list("MY_UL_items", "", settings.items, "", settings, "active_index")
```

```python
# CORRECT — pass the container object and property name as string
class MY_PT_good_list(bpy.types.Panel):
    def draw(self, context):
        settings = context.scene.my_settings
        # Correct: settings is the container, "items" is the property name string
        self.layout.template_list("MY_UL_items", "", settings, "items", settings, "active_index")
```

ALWAYS pass the data object and property name as separate arguments to `template_list()`. The property name MUST be a string, not the property value.

---

## AP-012: Popover Panel Without INSTANCED Option

**WHY this is wrong**: Panels used as popovers (via `layout.popover()`) MUST have `bl_options = {'INSTANCED'}`. Without this flag, the popover may fail to display or display incorrectly because Blender expects popover panels to be instancable.

```python
# WRONG — popover panel without INSTANCED flag
class MY_PT_bad_popover(bpy.types.Panel):
    bl_label = "Settings"
    bl_idname = "MY_PT_bad_popover"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'WINDOW'
    # Missing: bl_options = {'INSTANCED'}
    def draw(self, context): pass
```

```python
# CORRECT — INSTANCED flag set
class MY_PT_good_popover(bpy.types.Panel):
    bl_label = "Settings"
    bl_idname = "MY_PT_good_popover"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'WINDOW'
    bl_options = {'INSTANCED'}      # REQUIRED for popover usage
    def draw(self, context): pass
```

ALWAYS add `bl_options = {'INSTANCED'}` to panels intended for use with `layout.popover()`.
