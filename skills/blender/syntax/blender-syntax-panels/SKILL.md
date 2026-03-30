---
name: blender-syntax-panels
description: >
  Use when creating custom Blender UI panels, menus, or UIList elements. Prevents the
  common mistake of using wrong bl_space_type/bl_region_type combinations (panel won't show).
  Covers bpy.types.Panel, draw() method, UILayout API (row/column/box/split), bl_category,
  sub-panels, draw_header, menus, pie menus, and UIList.
  Keywords: Panel, UILayout, bl_space_type, bl_region_type, bl_category, draw, row, column,
  box, split, sub-panel, UIList, menu, Blender UI.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-panels

## Quick Reference

### Critical Warnings

**NEVER** modify scene data inside `Panel.draw()` — draw callbacks are read-only. Modifying properties triggers infinite redraw loops.

**NEVER** call `bpy.ops.*` inside `draw()` — operators MUST be exposed as UI buttons, not called directly during drawing.

**NEVER** omit `bl_space_type` or `bl_region_type` — registration fails without both.

**NEVER** use `bl_idname` values without the correct type tag (`_PT_` for panels, `_MT_` for menus, `_UL_` for UILists).

**ALWAYS** use `layout.panel()` for collapsible sections in Blender 4.1+ instead of registering separate sub-panel classes.

**ALWAYS** match `bl_space_type`, `bl_region_type`, and `bl_category` between parent and child panels.

### Panel Naming Convention

```
{ADDON_PREFIX}_{TYPE_TAG}_{name}

Type tags:
  _PT_  = Panel
  _MT_  = Menu
  _UL_  = UIList
  _HT_  = Header

Examples:
  MY_PT_main_panel
  MY_MT_object_menu
  MY_UL_item_list
```

### Decision Tree: Which Layout Container?

```
Need horizontal arrangement?  → layout.row(align=False)
Need vertical arrangement?    → layout.column(align=False)
Need bordered section?        → layout.box()
Need percentage split?        → layout.split(factor=0.5)
Need grid arrangement?        → layout.grid_flow(row_major=True, columns=0)
Need collapsible section?
  Blender 4.1+?              → layout.panel("ID", text="Header")
  Blender < 4.1?             → Register a sub-panel with bl_parent_id
Need separator line?          → layout.separator()
Need header spacing?          → layout.separator_spacer()
```

### Decision Tree: Panel Location

```
Sidebar (N-panel)?
  → bl_space_type = 'VIEW_3D', bl_region_type = 'UI'

Properties Editor?
  → bl_space_type = 'PROPERTIES', bl_region_type = 'WINDOW'
  → bl_context = 'object' | 'scene' | 'render' | 'data' | ...

Tool Shelf?
  → bl_space_type = 'VIEW_3D', bl_region_type = 'TOOLS'

Node Editor sidebar?
  → bl_space_type = 'NODE_EDITOR', bl_region_type = 'UI'

Image Editor sidebar?
  → bl_space_type = 'IMAGE_EDITOR', bl_region_type = 'UI'
```

---

## Essential Patterns

### Pattern 1: Minimal Panel

```python
# Blender 3.x/4.x/5.x: minimum viable panel
import bpy

class MY_PT_example(bpy.types.Panel):
    bl_label = "My Panel"           # Header text (REQUIRED)
    bl_idname = "MY_PT_example"     # Unique ID (REQUIRED)
    bl_space_type = 'VIEW_3D'      # Editor type (REQUIRED)
    bl_region_type = 'UI'          # Region type (REQUIRED)
    bl_category = "My Tab"         # Sidebar tab name (REQUIRED for UI region)

    def draw(self, context):
        layout = self.layout
        layout.label(text="Hello World")

def register():
    bpy.utils.register_class(MY_PT_example)

def unregister():
    bpy.utils.unregister_class(MY_PT_example)
```

### Pattern 2: Panel with poll() and draw_header()

```python
# Blender 3.x/4.x/5.x: conditional visibility + header checkbox
class MY_PT_conditional(bpy.types.Panel):
    bl_label = "Object Info"
    bl_idname = "MY_PT_conditional"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"
    bl_options = {'DEFAULT_CLOSED'}  # Start collapsed

    @classmethod
    def poll(cls, context):
        """Panel is visible ONLY when a mesh object is active."""
        return context.active_object is not None and context.active_object.type == 'MESH'

    def draw_header(self, context):
        """Draw a checkbox in the panel header."""
        self.layout.prop(context.scene.my_settings, "enabled", text="")

    def draw(self, context):
        layout = self.layout
        layout.active = context.scene.my_settings.enabled  # Grey out if disabled
        obj = context.active_object
        layout.prop(obj, "name")
        layout.prop(obj, "location")
```

### Pattern 3: Sub-Panels (Parent-Child Hierarchy)

```python
# Blender 3.x/4.x/5.x: sub-panel via bl_parent_id
class MY_PT_parent(bpy.types.Panel):
    bl_label = "Main Settings"
    bl_idname = "MY_PT_parent"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        self.layout.label(text="Parent content")

class MY_PT_child_a(bpy.types.Panel):
    bl_label = "Transform"
    bl_idname = "MY_PT_child_a"
    bl_space_type = 'VIEW_3D'       # MUST match parent
    bl_region_type = 'UI'           # MUST match parent
    bl_category = "My Tab"          # MUST match parent
    bl_parent_id = "MY_PT_parent"   # Links to parent
    bl_options = {'DEFAULT_CLOSED'}

    def draw(self, context):
        self.layout.prop(context.active_object, "location")
```

### Pattern 4: Collapsible Sections with layout.panel() (Blender 4.1+)

```python
# Blender 4.1+ ONLY: collapsible section WITHOUT separate class registration
class MY_PT_modern(bpy.types.Panel):
    bl_label = "Modern Panel"
    bl_idname = "MY_PT_modern"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tab"

    def draw(self, context):
        layout = self.layout
        layout.label(text="Always visible")

        # layout.panel() returns (header_layout, panel_layout) or (header_layout, None)
        header, body = layout.panel("MY_PT_modern_section", default_closed=True)
        header.label(text="Collapsible Section")
        if body is not None:
            body.prop(context.active_object, "name")
            body.prop(context.active_object, "location")
```

### Pattern 5: UILayout API: Common Elements

```python
# Blender 3.x/4.x/5.x: UILayout methods
def draw(self, context):
    layout = self.layout
    obj = context.active_object
    props = context.scene.my_settings

    # Property split (label left, widget right)
    layout.use_property_split = True
    layout.use_property_decorate = True  # Show keyframe dots

    # Row (horizontal)
    row = layout.row(align=True)
    row.prop(obj, "location", index=0, text="X")
    row.prop(obj, "location", index=1, text="Y")
    row.prop(obj, "location", index=2, text="Z")

    # Column (vertical)
    col = layout.column(align=True)
    col.prop(props, "count")
    col.prop(props, "scale")

    # Box (bordered container)
    box = layout.box()
    box.label(text="Section", icon='PREFERENCES')
    box.prop(props, "option_a")

    # Split (percentage-based columns)
    split = layout.split(factor=0.3)
    split.column().label(text="Label:")
    split.column().prop(props, "value", text="")

    # Grid flow
    grid = layout.grid_flow(row_major=True, columns=3, even_columns=True)
    for i in range(9):
        grid.label(text=f"Item {i}")

    # Separator
    layout.separator()

    # Operator button
    op = layout.operator("my.operator", text="Run", icon='PLAY')
    op.my_prop = 42  # Set operator property BEFORE execution

    # Conditional enable/disable
    row = layout.row()
    row.enabled = props.enabled
    row.operator("my.action")

    # Alert styling (red highlight)
    row = layout.row()
    row.alert = True
    row.label(text="Warning!", icon='ERROR')
```

### Pattern 6: Menu Definition

```python
# Blender 3.x/4.x/5.x: custom menu
class MY_MT_example(bpy.types.Menu):
    bl_label = "My Menu"
    bl_idname = "MY_MT_example"

    def draw(self, context):
        layout = self.layout
        layout.operator("my.action_a", text="Action A", icon='FILE')
        layout.operator("my.action_b", text="Action B")
        layout.separator()
        layout.menu("MY_MT_submenu")  # Nested submenu
        layout.operator("wm.call_menu", text="Open Other").name = "MY_MT_other"

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

# Invoke from panel:
# layout.menu("MY_MT_example")

# Append to existing Blender menu:
def draw_my_menu_item(self, context):
    self.layout.menu("MY_MT_example")

bpy.types.VIEW3D_MT_object.append(draw_my_menu_item)

# Remove in unregister():
bpy.types.VIEW3D_MT_object.remove(draw_my_menu_item)
```

### Pattern 7: UIList

```python
# Blender 3.x/4.x/5.x: custom list widget
class MY_UL_items(bpy.types.UIList):
    bl_idname = "MY_UL_items"

    def draw_item(self, context, layout, data, item, icon, active_data,
                  active_propname, index):
        if self.layout_type in {'DEFAULT', 'COMPACT'}:
            row = layout.row(align=True)
            row.prop(item, "name", text="", emboss=False)
            row.prop(item, "enabled", text="")
        elif self.layout_type == 'GRID':
            layout.alignment = 'CENTER'
            layout.label(text=item.name, icon_value=icon)

    def filter_items(self, context, data, propname):
        items = getattr(data, propname)
        flt_flags = [self.bitflag_filter_item] * len(items)
        flt_neworder = list(range(len(items)))

        # Filter by name
        if self.filter_name:
            for i, item in enumerate(items):
                if self.filter_name.lower() not in item.name.lower():
                    flt_flags[i] = 0

        return flt_flags, flt_neworder

# Use in panel draw():
# layout.template_list("MY_UL_items", "", data, "items", data, "active_index")
```

### Pattern 8: Properties Editor Panel

```python
# Blender 3.x/4.x/5.x: panel in Properties editor
class MY_PT_object_props(bpy.types.Panel):
    bl_label = "Custom Properties"
    bl_idname = "MY_PT_object_props"
    bl_space_type = 'PROPERTIES'     # Properties editor
    bl_region_type = 'WINDOW'        # Main area (NOT 'UI')
    bl_context = "object"            # Object properties tab

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def draw(self, context):
        self.layout.prop(context.active_object, "name")
```

---

## bl_space_type Values

| Value | Editor |
|-------|--------|
| `'VIEW_3D'` | 3D Viewport |
| `'PROPERTIES'` | Properties editor |
| `'OUTLINER'` | Outliner |
| `'NODE_EDITOR'` | Node editor (shader, compositor, geometry) |
| `'TEXT_EDITOR'` | Text editor |
| `'IMAGE_EDITOR'` | UV/Image editor |
| `'SEQUENCE_EDITOR'` | Video Sequencer |
| `'CLIP_EDITOR'` | Movie Clip editor |
| `'PREFERENCES'` | Preferences window |
| `'GRAPH_EDITOR'` | Graph editor (FCurves) |
| `'DOPESHEET_EDITOR'` | Dope Sheet |
| `'NLA_EDITOR'` | NLA editor |
| `'FILE_BROWSER'` | File browser |
| `'SPREADSHEET'` | Spreadsheet (Blender 3.0+) |

## bl_region_type Values

| Value | Location |
|-------|----------|
| `'UI'` | Sidebar (N-panel) |
| `'TOOLS'` | Tool shelf (T-panel) |
| `'HEADER'` | Header bar |
| `'WINDOW'` | Main area (Properties editor panels) |
| `'TOOL_PROPS'` | Active tool properties |
| `'TOOL_HEADER'` | Tool header (Blender 2.83+) |
| `'EXECUTE'` | Operator redo region |
| `'FOOTER'` | Footer bar |
| `'NAVIGATION_BAR'` | Navigation bar (Preferences) |

## bl_context Values (Properties editor only)

| Value | Tab |
|-------|-----|
| `"render"` | Render properties |
| `"output"` | Output properties |
| `"view_layer"` | View Layer properties |
| `"scene"` | Scene properties |
| `"world"` | World properties |
| `"object"` | Object properties |
| `"modifier"` | Modifier properties |
| `"particle"` | Particle properties |
| `"physics"` | Physics properties |
| `"constraint"` | Constraint properties |
| `"data"` | Object data properties (mesh, curve, etc.) |
| `"material"` | Material properties |
| `"texture"` | Texture properties |
| `"bone"` | Bone properties |
| `"bone_constraint"` | Bone constraint properties |

## bl_options Values (Panel)

| Value | Effect |
|-------|--------|
| `'DEFAULT_CLOSED'` | Panel starts collapsed |
| `'HIDE_HEADER'` | No header bar (cannot collapse) |
| `'INSTANCED'` | Panel can be used as template in multiple contexts |
| `'HEADER_LAYOUT_EXPAND'` | Header expands to fill available space |

---

## Common Operations

### Adding Popover Panels

```python
# Blender 3.x/4.x/5.x: popover panel (floating panel from header button)
class MY_PT_popover(bpy.types.Panel):
    bl_label = "Popover Settings"
    bl_idname = "MY_PT_popover"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'WINDOW'  # WINDOW for popover panels
    bl_options = {'INSTANCED'}  # Required for popover usage

    def draw(self, context):
        self.layout.prop(context.scene, "frame_current")

# In a header or panel draw():
# layout.popover("MY_PT_popover", text="Settings")
```

### Pie Menus

```python
# Blender 3.x/4.x/5.x: pie menu
class MY_MT_pie(bpy.types.Menu):
    bl_label = "My Pie Menu"
    bl_idname = "MY_MT_pie"

    def draw(self, context):
        pie = self.layout.menu_pie()
        pie.operator("transform.translate")   # West (left)
        pie.operator("transform.rotate")      # East (right)
        pie.operator("transform.resize")      # South (bottom)
        pie.operator("object.shade_smooth")   # North (top)
        # Additional items: NW, NE, SW, SE

# Invoke: bpy.ops.wm.call_menu_pie(name="MY_MT_pie")
```

### Dynamic Panel Content

```python
# Blender 3.x/4.x/5.x: panel content based on selection
def draw(self, context):
    layout = self.layout
    obj = context.active_object

    if obj is None:
        layout.label(text="No object selected", icon='INFO')
        return

    layout.prop(obj, "name")

    if obj.type == 'MESH':
        layout.label(text=f"Vertices: {len(obj.data.vertices)}")
        layout.label(text=f"Faces: {len(obj.data.polygons)}")
    elif obj.type == 'LIGHT':
        layout.prop(obj.data, "energy")
        layout.prop(obj.data, "color")
    elif obj.type == 'CAMERA':
        layout.prop(obj.data, "lens")
```

---

## Version-Specific Notes

| Feature | Version | Notes |
|---------|---------|-------|
| `layout.panel()` | 4.1+ | Inline collapsible sections without class registration |
| `bl_order` | 3.0+ | Integer controlling panel sort order |
| `SPREADSHEET` space type | 3.0+ | New editor type |
| `TOOL_HEADER` region | 2.83+ | Separate tool header region |
| Sub-panel support | 2.80+ | `bl_parent_id` for panel hierarchy |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for Panel, UILayout, Menu, UIList
- [references/examples.md](references/examples.md) — Working code examples for common panel patterns
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do when creating panels and UI

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Panel.html
- https://docs.blender.org/api/current/bpy.types.UILayout.html
- https://docs.blender.org/api/current/bpy.types.Menu.html
- https://docs.blender.org/api/current/bpy.types.UIList.html
- https://docs.blender.org/api/current/info_quickstart.html
