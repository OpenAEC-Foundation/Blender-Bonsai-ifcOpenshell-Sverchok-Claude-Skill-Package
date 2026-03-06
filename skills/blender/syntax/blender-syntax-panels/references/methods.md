# blender-syntax-panels: API Method Reference

Sources:
- https://docs.blender.org/api/current/bpy.types.Panel.html
- https://docs.blender.org/api/current/bpy.types.UILayout.html
- https://docs.blender.org/api/current/bpy.types.Menu.html
- https://docs.blender.org/api/current/bpy.types.UIList.html

---

## bpy.types.Panel — Panel Base Class

All custom panels MUST subclass `bpy.types.Panel`. Panels are never instantiated directly — Blender creates instances during UI drawing.

### Class Variables (set on the class, not on instances)

| Variable | Type | Required | Description |
|----------|------|----------|-------------|
| `bl_idname` | `str` | Yes | Unique identifier. MUST use `{PREFIX}_PT_{name}` convention |
| `bl_label` | `str` | Yes | Text displayed in panel header |
| `bl_space_type` | `str` (enum) | Yes | Editor type where panel appears |
| `bl_region_type` | `str` (enum) | Yes | Region within editor |
| `bl_category` | `str` | Yes* | Sidebar tab name. *Required when `bl_region_type = 'UI'` |
| `bl_context` | `str` | No | Mode/context filter. Used mainly with `bl_space_type = 'PROPERTIES'` |
| `bl_options` | `set[str]` | No | Panel behavior flags: `{'DEFAULT_CLOSED'}`, `{'HIDE_HEADER'}`, `{'INSTANCED'}`, `{'HEADER_LAYOUT_EXPAND'}` |
| `bl_parent_id` | `str` | No | `bl_idname` of parent panel for sub-panel hierarchy |
| `bl_order` | `int` | No | Sort order within same parent/category (lower = higher). Blender 3.0+ |
| `bl_description` | `str` | No | Tooltip text |
| `bl_translation_context` | `str` | No | Translation context for `bl_label` |
| `bl_owner_id` | `str` | No | Addon module name that owns this panel |
| `bl_ui_units_x` | `int` | No | Width hint in UI units (for popover panels) |

### Methods

```python
# draw(self, context) — REQUIRED
# Called every time the panel needs redrawing. Access self.layout to build UI.
# NEVER modify scene data here. Read-only access only.
def draw(self, context):
    # context: bpy.types.Context
    layout = self.layout  # bpy.types.UILayout
    layout.label(text="Content")

# draw_header(self, context) — OPTIONAL
# Draw additional elements in the panel header (e.g., enable checkbox).
def draw_header(self, context):
    # self.layout is the header layout, NOT the panel body
    self.layout.prop(context.scene.my_props, "enabled", text="")

# draw_header_preset(self, context) — OPTIONAL
# Draw preset selector in the header (right side). Blender 2.81+.
def draw_header_preset(self, _context):
    # Typically used for preset menus
    pass

# poll(cls, context) — OPTIONAL, classmethod
# Return True to show panel, False to hide it. Called before draw().
@classmethod
def poll(cls, context):
    # context: bpy.types.Context
    return context.active_object is not None  # -> bool
```

### Instance Properties (available in draw methods)

| Property | Type | Description |
|----------|------|-------------|
| `self.layout` | `UILayout` | Layout object for adding UI elements |
| `self.bl_rna` | `StructRNA` | RNA type info |
| `self.is_popover` | `bool` | True if panel is displayed as a popover |

---

## bpy.types.UILayout — Layout Building API

UILayout is the central API for building panel UI. All container methods return a new UILayout for nesting.

### Container Methods

```python
# row(align=False) -> UILayout
# Creates a horizontal row. align=True removes padding between elements.
row = layout.row(align=True)

# column(align=False) -> UILayout
# Creates a vertical column.
col = layout.column(align=True)

# box() -> UILayout
# Creates a bordered container.
box = layout.box()

# split(factor=0.5, align=False) -> UILayout
# Splits layout into columns. factor = proportion of first column (0.0–1.0).
split = layout.split(factor=0.3)
col_left = split.column()
col_right = split.column()

# grid_flow(row_major=False, columns=0, even_columns=False, even_rows=False, align=False) -> UILayout
# Creates a grid layout. columns=0 means auto-detect based on available width.
grid = layout.grid_flow(row_major=True, columns=3, even_columns=True)

# panel(idname, default_closed=False) -> tuple[UILayout, UILayout | None]
# Blender 4.1+ ONLY. Creates a collapsible inline section.
# Returns (header_layout, body_layout). body_layout is None when collapsed.
# idname MUST be unique and is used to persist open/closed state.
header, body = layout.panel("MY_PT_section_id", default_closed=True)
header.label(text="Section Title")
if body is not None:
    body.prop(obj, "name")
```

### Content Methods

```python
# label(text="", icon='NONE', icon_value=0) -> None
# Displays read-only text. icon is a Blender icon enum string.
layout.label(text="Info", icon='INFO')

# prop(data, property, text="", text_ctxt="", translate=True, icon='NONE',
#      expand=False, slider=False, toggle=-1, icon_only=False, event=False,
#      full_event=False, emboss=True, index=-1, icon_value=0, invert_checkbox=False)
#      -> None
# Displays and edits an RNA property.
layout.prop(obj, "name")                      # Auto-labeled
layout.prop(obj, "name", text="Object Name")  # Custom label
layout.prop(obj, "location", index=0)         # Single array element (X)
layout.prop(obj, "show_wire", toggle=True)    # Toggle button style
layout.prop(obj, "type", expand=True)         # Enum as row of buttons

# operator(operator, text="", text_ctxt="", translate=True, icon='NONE',
#          emboss=True, depress=False, icon_value=0) -> OperatorProperties
# Creates a button that runs an operator. Returns operator properties for pre-setting.
op = layout.operator("mesh.primitive_cube_add", text="Add Cube", icon='MESH_CUBE')
op.size = 2.0  # Set operator property BEFORE user clicks

# menu(menu, text="", text_ctxt="", translate=True, icon='NONE', icon_value=0) -> None
# Creates a dropdown button that opens a Menu class.
layout.menu("MY_MT_example", text="Open Menu")

# popover(panel, text="", text_ctxt="", translate=True, icon='NONE', icon_value=0) -> None
# Creates a button that opens a popover panel.
layout.popover("MY_PT_popover_panel")

# separator(factor=1.0) -> None
# Adds vertical spacing. factor multiplies default spacing.
layout.separator()
layout.separator(factor=2.0)  # Double spacing

# separator_spacer() -> None
# Adds flexible spacer (pushes items apart in headers).
layout.separator_spacer()

# template_list(listtype_name, list_id, dataptr, propname, active_dataptr,
#               active_propname, item_dyntip_propname="", rows=5, maxrows=5,
#               type='DEFAULT', columns=9, sort_reverse=False, sort_lock=False) -> None
# Displays a UIList widget.
# listtype_name: bl_idname of the UIList subclass
# list_id: unique string for this list instance (use "" for single instance)
# dataptr: object containing the collection property
# propname: name of the collection property on dataptr
# active_dataptr: object containing the active index property
# active_propname: name of the integer property for active index
layout.template_list("MY_UL_items", "", scene, "my_items", scene, "my_active_index")

# prop_search(data, property, search_data, search_property, text="", text_ctxt="",
#             translate=True, icon='NONE', results_are_suggestions=False) -> None
# Property field with search dropdown. Searches names in a collection.
layout.prop_search(obj, "parent_bone", armature, "bones")

# props_enum(data, property) -> None
# Displays all enum items as individual buttons.
layout.props_enum(obj, "type")

# prop_menu_enum(data, property, text="", text_ctxt="", translate=True, icon='NONE') -> None
# Displays property as a dropdown menu.
layout.prop_menu_enum(obj, "type")

# prop_tabs_enum(data, property, data_highlight=None, property_highlight="",
#                icon_only=False) -> None
# Displays enum as tab buttons.
layout.prop_tabs_enum(scene, "my_mode")
```

### Layout Properties

```python
# active: bool — When False, layout and children are greyed out (still clickable)
layout.active = False

# active_default: bool — Default active state for items without explicit active
layout.active_default = True

# alert: bool — When True, layout shows with red/warning highlight
row = layout.row()
row.alert = True
row.label(text="Error!", icon='ERROR')

# alignment: str — 'EXPAND' (default), 'LEFT', 'CENTER', 'RIGHT'
row = layout.row()
row.alignment = 'RIGHT'

# direction: str — 'HORIZONTAL' or 'VERTICAL'
# Read-only. Set by row()/column() creation.

# emboss: str — Button style: 'NORMAL', 'NONE', 'PULLDOWN_MENU', 'RADIAL_MENU', 'NONE_OR_STATUS'
layout.emboss = 'NONE'

# enabled: bool — When False, layout and children are greyed out AND non-interactive
row = layout.row()
row.enabled = False
row.operator("my.disabled_action")  # Visible but cannot be clicked

# operator_context: str — Override for how operators are invoked:
# 'INVOKE_DEFAULT', 'INVOKE_REGION_WIN', 'INVOKE_REGION_CHANNELS',
# 'INVOKE_REGION_PREVIEW', 'INVOKE_AREA', 'INVOKE_SCREEN',
# 'EXEC_DEFAULT', 'EXEC_REGION_WIN', 'EXEC_REGION_CHANNELS',
# 'EXEC_REGION_PREVIEW', 'EXEC_AREA', 'EXEC_SCREEN'
layout.operator_context = 'EXEC_DEFAULT'

# scale_x: float — Horizontal scale factor (1.0 = default width)
layout.scale_x = 2.0  # Double width

# scale_y: float — Vertical scale factor (1.0 = default height)
layout.scale_y = 1.5  # 1.5x height

# ui_units_x: float — Fixed width in UI units (overrides scale_x)
layout.ui_units_x = 10

# ui_units_y: float — Fixed height in UI units (overrides scale_y)
layout.ui_units_y = 3

# use_property_split: bool — Split property labels from values (label left, widget right)
layout.use_property_split = True

# use_property_decorate: bool — Show keyframe/driver dots next to properties
layout.use_property_decorate = True
```

---

## bpy.types.Menu — Menu Base Class

### Class Variables

| Variable | Type | Required | Description |
|----------|------|----------|-------------|
| `bl_idname` | `str` | Yes | Unique identifier. MUST use `{PREFIX}_MT_{name}` convention |
| `bl_label` | `str` | Yes | Menu title text |
| `bl_description` | `str` | No | Tooltip description |
| `bl_options` | `set[str]` | No | Menu options: `{'SEARCH_ON_KEY_PRESS'}` |
| `bl_translation_context` | `str` | No | Translation context |
| `bl_owner_id` | `str` | No | Addon module name |

### Methods

```python
# draw(self, context) -> None
# Build menu items. Same UILayout API as panels.
def draw(self, context):
    layout = self.layout
    layout.operator("my.action_a")
    layout.separator()
    layout.operator("my.action_b")
    layout.menu("MY_MT_submenu")  # Nested submenu

# poll(cls, context) -> bool
# Control when menu is available.
@classmethod
def poll(cls, context):
    return context.active_object is not None

# draw_preset(self, context) -> None
# Override for preset-style menus.
```

### Appending to Existing Menus

```python
# Blender 3.x/4.x/5.x — add items to built-in menus
def my_draw_func(self, context):
    self.layout.separator()
    self.layout.operator("my.custom_op")

# Append to end of menu
bpy.types.VIEW3D_MT_object.append(my_draw_func)

# Prepend to beginning of menu
bpy.types.VIEW3D_MT_object.prepend(my_draw_func)

# Remove in unregister()
bpy.types.VIEW3D_MT_object.remove(my_draw_func)
```

### Invoking Menus

```python
# From Python (inside an operator or button):
bpy.ops.wm.call_menu(name="MY_MT_example")

# Pie menu:
bpy.ops.wm.call_menu_pie(name="MY_MT_pie")

# From a panel layout:
layout.menu("MY_MT_example")
layout.menu("MY_MT_example", text="Custom Label", icon='DOWNARROW_HLT')
```

### Common Built-in Menus (for appending)

| Menu | Location |
|------|----------|
| `VIEW3D_MT_object` | 3D Viewport > Object menu |
| `VIEW3D_MT_mesh_add` | Add > Mesh |
| `VIEW3D_MT_add` | 3D Viewport > Add menu |
| `TOPBAR_MT_file` | File menu |
| `TOPBAR_MT_edit` | Edit menu |
| `VIEW3D_MT_object_context_menu` | Right-click context menu |
| `NODE_MT_add` | Node Editor > Add menu |
| `OUTLINER_MT_context_menu` | Outliner right-click |

---

## bpy.types.UIList — Custom List Widget

### Class Variables

| Variable | Type | Required | Description |
|----------|------|----------|-------------|
| `bl_idname` | `str` | Yes | Unique identifier. MUST use `{PREFIX}_UL_{name}` convention |
| `bl_label` | `str` | No | Display label |

### Instance Properties

| Property | Type | Description |
|----------|------|-------------|
| `layout_type` | `str` | Current display mode: `'DEFAULT'`, `'COMPACT'`, `'GRID'` |
| `filter_name` | `str` | Current name filter string (built-in filter UI) |
| `use_filter_sort_alpha` | `bool` | Alphabetical sort enabled |
| `use_filter_sort_reverse` | `bool` | Reverse sort order |
| `use_filter_sort_lock` | `bool` | Lock sort order |
| `use_filter_invert` | `bool` | Invert filter results |
| `bitflag_filter_item` | `int` | Bit flag value indicating item passes filter (default: `1 << 30`) |

### Methods

```python
# draw_item(self, context, layout, data, item, icon, active_data,
#           active_propname, index, flt_flag) -> None
# Draw a single list item. Called for each visible item.
# layout: UILayout — layout for this item
# data: the object containing the collection
# item: the current collection item
# icon: default icon for this item
# active_data: object containing active index
# active_propname: name of active index property
# index: item index in the collection
# flt_flag: result of filter_items for this item
def draw_item(self, context, layout, data, item, icon, active_data,
              active_propname, index, flt_flag=0):
    if self.layout_type in {'DEFAULT', 'COMPACT'}:
        layout.prop(item, "name", text="", emboss=False, icon_value=icon)
    elif self.layout_type == 'GRID':
        layout.alignment = 'CENTER'
        layout.label(text="", icon_value=icon)

# draw_filter(self, context, layout) -> None
# Draw the filter UI above the list. Override to add custom filter controls.
def draw_filter(self, context, layout):
    row = layout.row(align=True)
    row.prop(self, "filter_name", text="", icon='VIEWZOOM')
    row.prop(self, "use_filter_sort_alpha", text="", icon='SORTALPHA')
    row.prop(self, "use_filter_invert", text="", icon='ARROW_LEFTRIGHT')

# filter_items(self, context, data, propname) -> tuple[list[int], list[int]]
# Return (filter_flags, neworder).
# filter_flags: list of int, one per item. 0 = hidden, self.bitflag_filter_item = shown.
# neworder: list of int, mapping old index -> new position (or empty for no reorder).
def filter_items(self, context, data, propname):
    items = getattr(data, propname)
    flags = [self.bitflag_filter_item] * len(items)
    order = list(range(len(items)))

    # Name filter
    if self.filter_name:
        for i, item in enumerate(items):
            if self.filter_name.lower() not in item.name.lower():
                flags[i] = 0

    # Alphabetical sort
    if self.use_filter_sort_alpha:
        order = sorted(range(len(items)), key=lambda i: items[i].name.lower())

    if self.use_filter_sort_reverse:
        order.reverse()

    return flags, order
```

### CollectionProperty + UIList Integration Pattern

```python
# Blender 3.x/4.x/5.x — complete UIList data model
import bpy
from bpy.props import CollectionProperty, IntProperty, StringProperty, BoolProperty

class MyItem(bpy.types.PropertyGroup):
    name: StringProperty(name="Name", default="Item")
    enabled: BoolProperty(name="Enabled", default=True)

class MySettings(bpy.types.PropertyGroup):
    items: CollectionProperty(type=MyItem)
    active_index: IntProperty(name="Active Index", default=0)

# Register:
bpy.utils.register_class(MyItem)
bpy.utils.register_class(MySettings)
bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MySettings)

# In panel draw():
settings = context.scene.my_settings
layout.template_list("MY_UL_items", "", settings, "items", settings, "active_index")

# Add/remove operators should manipulate settings.items:
# settings.items.add()
# settings.items.remove(settings.active_index)
```

---

## Pie Menu Layout Order

```
        North (top)
          [3]
           |
West [0]---+---[1] East
           |
          [2]
        South (bottom)

Items 4-7: NW, NE, SW, SE (diagonal positions)
```

Items are added in this specific order via `menu_pie()`. The first `operator()` or `menu()` call goes West, second East, third South, fourth North.

---

## Registration Order

ALWAYS register classes in dependency order:
1. `PropertyGroup` subclasses (data models)
2. `Operator` subclasses
3. `UIList` subclasses
4. `Menu` subclasses
5. `Panel` subclasses (panels depend on everything else)

ALWAYS unregister in reverse order.

```python
classes = (
    MyPropertyGroup,
    MY_OT_add_item,
    MY_OT_remove_item,
    MY_UL_items,
    MY_MT_context_menu,
    MY_PT_main_panel,
    MY_PT_child_panel,
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MyPropertyGroup)

def unregister():
    del bpy.types.Scene.my_settings
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```
