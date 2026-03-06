# blender-syntax-panels: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/bpy.types.Panel.html
- https://docs.blender.org/api/current/bpy.types.UILayout.html
- https://docs.blender.org/api/current/bpy.types.Menu.html
- https://docs.blender.org/api/current/bpy.types.UIList.html

---

## Example 1: Complete Sidebar Panel with Properties

```python
# Blender 3.x/4.x/5.x — full addon panel with custom properties
import bpy
from bpy.props import FloatProperty, IntProperty, BoolProperty, EnumProperty, StringProperty

class MY_PanelSettings(bpy.types.PropertyGroup):
    count: IntProperty(name="Count", default=1, min=1, max=100)
    scale: FloatProperty(name="Scale", default=1.0, min=0.01, max=100.0)
    enabled: BoolProperty(name="Enabled", default=True)
    mode: EnumProperty(
        name="Mode",
        items=[
            ('ADD', "Add", "Add new objects"),
            ('REPLACE', "Replace", "Replace existing objects"),
            ('MERGE', "Merge", "Merge with existing"),
        ],
        default='ADD',
    )

class MY_PT_main(bpy.types.Panel):
    bl_label = "My Addon"
    bl_idname = "MY_PT_main"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Addon"

    def draw(self, context):
        layout = self.layout
        props = context.scene.my_panel_settings

        layout.use_property_split = True

        layout.prop(props, "enabled")

        col = layout.column()
        col.active = props.enabled
        col.prop(props, "mode")
        col.prop(props, "count")
        col.prop(props, "scale")

        layout.separator()
        layout.operator("mesh.primitive_cube_add", text="Add Cube", icon='MESH_CUBE')

classes = (MY_PanelSettings, MY_PT_main)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_panel_settings = bpy.props.PointerProperty(type=MY_PanelSettings)

def unregister():
    del bpy.types.Scene.my_panel_settings
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 2: Parent and Child Panels

```python
# Blender 3.x/4.x/5.x — panel hierarchy with sub-panels
import bpy

class MY_PT_parent(bpy.types.Panel):
    bl_label = "Object Tools"
    bl_idname = "MY_PT_parent"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tools"

    def draw(self, context):
        self.layout.label(text="Select a sub-panel below")

class MY_PT_transform(bpy.types.Panel):
    bl_label = "Transform"
    bl_idname = "MY_PT_transform"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tools"
    bl_parent_id = "MY_PT_parent"

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def draw(self, context):
        layout = self.layout
        layout.use_property_split = True
        obj = context.active_object
        col = layout.column()
        col.prop(obj, "location")
        col.prop(obj, "rotation_euler")
        col.prop(obj, "scale")

class MY_PT_display(bpy.types.Panel):
    bl_label = "Display"
    bl_idname = "MY_PT_display"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tools"
    bl_parent_id = "MY_PT_parent"
    bl_options = {'DEFAULT_CLOSED'}

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def draw(self, context):
        layout = self.layout
        obj = context.active_object
        layout.prop(obj, "display_type")
        layout.prop(obj, "show_wire")
        layout.prop(obj, "show_in_front")

classes = (MY_PT_parent, MY_PT_transform, MY_PT_display)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 3: Collapsible Sections with layout.panel() (Blender 4.1+)

```python
# Blender 4.1+ ONLY — inline collapsible sections without extra class registration
import bpy

class MY_PT_modern_panel(bpy.types.Panel):
    bl_label = "Modern Settings"
    bl_idname = "MY_PT_modern_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Modern"

    def draw(self, context):
        layout = self.layout
        obj = context.active_object

        if obj is None:
            layout.label(text="No object selected", icon='INFO')
            return

        layout.prop(obj, "name")

        # First collapsible section
        header, body = layout.panel("MY_PT_modern_transform", default_closed=False)
        header.label(text="Transform")
        if body is not None:
            body.use_property_split = True
            body.prop(obj, "location")
            body.prop(obj, "rotation_euler")
            body.prop(obj, "scale")

        # Second collapsible section (starts closed)
        header, body = layout.panel("MY_PT_modern_display", default_closed=True)
        header.label(text="Display Options")
        if body is not None:
            body.prop(obj, "display_type")
            body.prop(obj, "show_wire")

def register():
    bpy.utils.register_class(MY_PT_modern_panel)

def unregister():
    bpy.utils.unregister_class(MY_PT_modern_panel)
```

---

## Example 4: Properties Editor Panel with bl_context

```python
# Blender 3.x/4.x/5.x — panel in the Properties editor (Object tab)
import bpy

class MY_PT_object_info(bpy.types.Panel):
    bl_label = "Custom Object Info"
    bl_idname = "MY_PT_object_info"
    bl_space_type = 'PROPERTIES'
    bl_region_type = 'WINDOW'
    bl_context = "object"

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def draw(self, context):
        layout = self.layout
        obj = context.active_object

        layout.use_property_split = True
        layout.prop(obj, "name")
        layout.prop(obj, "type", emboss=False)

        box = layout.box()
        box.label(text="Dimensions", icon='OBJECT_DATA')
        col = box.column(align=True)
        col.prop(obj, "dimensions", index=0, text="X")
        col.prop(obj, "dimensions", index=1, text="Y")
        col.prop(obj, "dimensions", index=2, text="Z")

def register():
    bpy.utils.register_class(MY_PT_object_info)

def unregister():
    bpy.utils.unregister_class(MY_PT_object_info)
```

---

## Example 5: UILayout Techniques

```python
# Blender 3.x/4.x/5.x — various UILayout techniques in one panel
import bpy

class MY_PT_layout_demo(bpy.types.Panel):
    bl_label = "Layout Demo"
    bl_idname = "MY_PT_layout_demo"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Demo"

    def draw(self, context):
        layout = self.layout
        obj = context.active_object

        # === Row with aligned buttons ===
        row = layout.row(align=True)
        row.operator("transform.translate", text="Move")
        row.operator("transform.rotate", text="Rotate")
        row.operator("transform.resize", text="Scale")

        layout.separator()

        # === Split layout (label : value) ===
        split = layout.split(factor=0.4)
        col_label = split.column()
        col_value = split.column()
        col_label.label(text="Object:")
        if obj:
            col_value.label(text=obj.name)
        else:
            col_value.label(text="None")

        layout.separator()

        # === Box with content ===
        box = layout.box()
        box.label(text="Visibility", icon='HIDE_OFF')
        if obj:
            row = box.row()
            row.prop(obj, "hide_viewport", text="Viewport")
            row.prop(obj, "hide_render", text="Render")

        layout.separator()

        # === Grid flow ===
        layout.label(text="Grid Layout:")
        grid = layout.grid_flow(row_major=True, columns=3, even_columns=True, align=True)
        for i in range(6):
            grid.operator("mesh.primitive_cube_add", text=f"Cube {i+1}")

        layout.separator()

        # === Conditional enabled/disabled ===
        row = layout.row()
        row.enabled = obj is not None
        row.operator("object.delete", text="Delete Selected")

        # === Alert row ===
        if obj and obj.type != 'MESH':
            row = layout.row()
            row.alert = True
            row.label(text="Not a mesh object!", icon='ERROR')

        # === Scale ===
        row = layout.row()
        row.scale_y = 2.0
        row.operator("render.render", text="RENDER", icon='RENDER_STILL')
```

---

## Example 6: Custom Menu with Submenu

```python
# Blender 3.x/4.x/5.x — custom menu with sub-menus and dynamic items
import bpy

class MY_MT_object_tools(bpy.types.Menu):
    bl_label = "Object Tools"
    bl_idname = "MY_MT_object_tools"

    def draw(self, context):
        layout = self.layout
        layout.operator("object.shade_smooth", icon='SMOOTH')
        layout.operator("object.shade_flat", icon='MESH_PLANE')
        layout.separator()
        layout.menu("MY_MT_transform_tools")  # Submenu
        layout.separator()
        layout.operator("object.delete", text="Delete", icon='TRASH')

class MY_MT_transform_tools(bpy.types.Menu):
    bl_label = "Transform Tools"
    bl_idname = "MY_MT_transform_tools"

    def draw(self, context):
        layout = self.layout
        layout.operator("object.location_clear", text="Clear Location")
        layout.operator("object.rotation_clear", text="Clear Rotation")
        layout.operator("object.scale_clear", text="Clear Scale")

class MY_PT_menu_panel(bpy.types.Panel):
    bl_label = "Menu Example"
    bl_idname = "MY_PT_menu_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Demo"

    def draw(self, context):
        layout = self.layout
        layout.menu("MY_MT_object_tools", icon='DOWNARROW_HLT')

# Append to existing Blender menu
def draw_in_object_menu(self, context):
    self.layout.separator()
    self.layout.menu("MY_MT_object_tools", text="My Tools")

classes = (MY_MT_transform_tools, MY_MT_object_tools, MY_PT_menu_panel)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.VIEW3D_MT_object.append(draw_in_object_menu)

def unregister():
    bpy.types.VIEW3D_MT_object.remove(draw_in_object_menu)
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 7: Complete UIList with Add/Remove

```python
# Blender 3.x/4.x/5.x — UIList with add/remove operators
import bpy
from bpy.props import CollectionProperty, IntProperty, StringProperty, BoolProperty

# --- Data Model ---
class MY_ListItem(bpy.types.PropertyGroup):
    name: StringProperty(name="Name", default="Untitled")
    enabled: BoolProperty(name="Enabled", default=True)
    value: bpy.props.FloatProperty(name="Value", default=0.0)

# --- UIList ---
class MY_UL_items(bpy.types.UIList):
    bl_idname = "MY_UL_items"

    def draw_item(self, context, layout, data, item, icon, active_data,
                  active_propname, index, flt_flag=0):
        if self.layout_type in {'DEFAULT', 'COMPACT'}:
            row = layout.row(align=True)
            row.prop(item, "enabled", text="", icon='CHECKBOX_HLT' if item.enabled else 'CHECKBOX_DEHLT')
            row.prop(item, "name", text="", emboss=False)
            row.prop(item, "value", text="")
        elif self.layout_type == 'GRID':
            layout.alignment = 'CENTER'
            layout.label(text=item.name)

# --- Operators ---
class MY_OT_list_add(bpy.types.Operator):
    bl_idname = "my_list.add_item"
    bl_label = "Add Item"

    def execute(self, context):
        settings = context.scene.my_list_settings
        item = settings.items.add()
        item.name = f"Item {len(settings.items)}"
        settings.active_index = len(settings.items) - 1
        return {'FINISHED'}

class MY_OT_list_remove(bpy.types.Operator):
    bl_idname = "my_list.remove_item"
    bl_label = "Remove Item"

    @classmethod
    def poll(cls, context):
        settings = context.scene.my_list_settings
        return len(settings.items) > 0

    def execute(self, context):
        settings = context.scene.my_list_settings
        settings.items.remove(settings.active_index)
        settings.active_index = min(settings.active_index, len(settings.items) - 1)
        return {'FINISHED'}

class MY_OT_list_move(bpy.types.Operator):
    bl_idname = "my_list.move_item"
    bl_label = "Move Item"
    direction: bpy.props.EnumProperty(
        items=[('UP', "Up", ""), ('DOWN', "Down", "")],
    )

    @classmethod
    def poll(cls, context):
        settings = context.scene.my_list_settings
        return len(settings.items) > 1

    def execute(self, context):
        settings = context.scene.my_list_settings
        idx = settings.active_index
        if self.direction == 'UP' and idx > 0:
            settings.items.move(idx, idx - 1)
            settings.active_index -= 1
        elif self.direction == 'DOWN' and idx < len(settings.items) - 1:
            settings.items.move(idx, idx + 1)
            settings.active_index += 1
        return {'FINISHED'}

# --- Settings ---
class MY_ListSettings(bpy.types.PropertyGroup):
    items: CollectionProperty(type=MY_ListItem)
    active_index: IntProperty(name="Active Index", default=0)

# --- Panel ---
class MY_PT_list_panel(bpy.types.Panel):
    bl_label = "Item List"
    bl_idname = "MY_PT_list_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "List Demo"

    def draw(self, context):
        layout = self.layout
        settings = context.scene.my_list_settings

        row = layout.row()
        row.template_list("MY_UL_items", "", settings, "items", settings, "active_index")

        col = row.column(align=True)
        col.operator("my_list.add_item", icon='ADD', text="")
        col.operator("my_list.remove_item", icon='REMOVE', text="")
        col.separator()
        col.operator("my_list.move_item", icon='TRIA_UP', text="").direction = 'UP'
        col.operator("my_list.move_item", icon='TRIA_DOWN', text="").direction = 'DOWN'

        # Show active item details
        if settings.items and 0 <= settings.active_index < len(settings.items):
            item = settings.items[settings.active_index]
            box = layout.box()
            box.label(text="Active Item Details:")
            box.prop(item, "name")
            box.prop(item, "value")

classes = (
    MY_ListItem,
    MY_OT_list_add,
    MY_OT_list_remove,
    MY_OT_list_move,
    MY_UL_items,
    MY_ListSettings,
    MY_PT_list_panel,
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_list_settings = bpy.props.PointerProperty(type=MY_ListSettings)

def unregister():
    del bpy.types.Scene.my_list_settings
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 8: Pie Menu

```python
# Blender 3.x/4.x/5.x — pie menu with 8 items
import bpy

class MY_MT_transform_pie(bpy.types.Menu):
    bl_label = "Transform Pie"
    bl_idname = "MY_MT_transform_pie"

    def draw(self, context):
        pie = self.layout.menu_pie()
        # Order: West, East, South, North, NW, NE, SW, SE
        pie.operator("transform.translate", text="Move", icon='ORIENTATION_GLOBAL')     # W
        pie.operator("transform.rotate", text="Rotate", icon='ORIENTATION_GIMBAL')      # E
        pie.operator("transform.resize", text="Scale", icon='FULLSCREEN_ENTER')         # S
        pie.operator("object.origin_set", text="Origin", icon='OBJECT_ORIGIN')          # N
        pie.operator("object.location_clear", text="Clear Loc")                          # NW
        pie.operator("object.rotation_clear", text="Clear Rot")                          # NE
        pie.operator("object.scale_clear", text="Clear Scale")                           # SW
        pie.operator("transform.mirror", text="Mirror")                                  # SE

# Invoke with: bpy.ops.wm.call_menu_pie(name="MY_MT_transform_pie")
# Bind to keymap for best UX

def register():
    bpy.utils.register_class(MY_MT_transform_pie)

def unregister():
    bpy.utils.unregister_class(MY_MT_transform_pie)
```

---

## Example 9: Panel with draw_header and Header Preset

```python
# Blender 3.x/4.x/5.x — panel with header checkbox and mode indicator
import bpy

class MY_PT_header_demo(bpy.types.Panel):
    bl_label = "Render Settings"
    bl_idname = "MY_PT_header_demo"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Render"
    bl_options = {'DEFAULT_CLOSED'}

    def draw_header(self, context):
        scene = context.scene
        # Checkbox in header enables/disables the panel
        self.layout.prop(scene.render, "use_motion_blur", text="")

    def draw(self, context):
        layout = self.layout
        scene = context.scene

        # Grey out panel body when motion blur is disabled
        layout.active = scene.render.use_motion_blur

        layout.use_property_split = True
        layout.prop(scene.render, "motion_blur_shutter")

def register():
    bpy.utils.register_class(MY_PT_header_demo)

def unregister():
    bpy.utils.unregister_class(MY_PT_header_demo)
```

---

## Example 10: Popover Panel

```python
# Blender 3.x/4.x/5.x — popover panel (floating panel from button click)
import bpy

class MY_PT_popover_settings(bpy.types.Panel):
    bl_label = "Quick Settings"
    bl_idname = "MY_PT_popover_settings"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'WINDOW'
    bl_options = {'INSTANCED'}  # REQUIRED for popover panels

    def draw(self, context):
        layout = self.layout
        layout.use_property_split = True
        scene = context.scene
        layout.prop(scene.render, "engine")
        layout.prop(scene.render, "film_transparent")
        layout.prop(scene, "frame_current")

class MY_PT_main_with_popover(bpy.types.Panel):
    bl_label = "Main Panel"
    bl_idname = "MY_PT_main_with_popover"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Popover"

    def draw(self, context):
        layout = self.layout
        layout.label(text="Click to open settings:")
        layout.popover("MY_PT_popover_settings", text="Settings", icon='PREFERENCES')

classes = (MY_PT_popover_settings, MY_PT_main_with_popover)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 11: Dynamic Panel Content Based on Object Type

```python
# Blender 3.x/4.x/5.x — panel adapts to the active object type
import bpy

class MY_PT_adaptive(bpy.types.Panel):
    bl_label = "Object Inspector"
    bl_idname = "MY_PT_adaptive"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Inspector"

    def draw(self, context):
        layout = self.layout
        obj = context.active_object

        if obj is None:
            layout.label(text="No object selected", icon='INFO')
            return

        layout.prop(obj, "name", icon='OBJECT_DATA')
        layout.separator()

        if obj.type == 'MESH':
            mesh = obj.data
            box = layout.box()
            box.label(text="Mesh Stats", icon='MESH_DATA')
            col = box.column(align=True)
            col.label(text=f"Vertices: {len(mesh.vertices)}")
            col.label(text=f"Edges: {len(mesh.edges)}")
            col.label(text=f"Faces: {len(mesh.polygons)}")

            if mesh.materials:
                box = layout.box()
                box.label(text="Materials", icon='MATERIAL')
                for i, mat in enumerate(mesh.materials):
                    if mat:
                        box.label(text=f"  {i}: {mat.name}")

        elif obj.type == 'LIGHT':
            light = obj.data
            layout.prop(light, "type")
            layout.prop(light, "energy")
            layout.prop(light, "color")

        elif obj.type == 'CAMERA':
            cam = obj.data
            layout.prop(cam, "type")
            layout.prop(cam, "lens")
            layout.prop(cam, "clip_start")
            layout.prop(cam, "clip_end")

        else:
            layout.label(text=f"Type: {obj.type}")

def register():
    bpy.utils.register_class(MY_PT_adaptive)

def unregister():
    bpy.utils.unregister_class(MY_PT_adaptive)
```
