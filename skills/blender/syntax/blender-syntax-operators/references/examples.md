# blender-syntax-operators — Working Examples

All examples verified against Blender 4.x API. Version-specific notes included where applicable.

---

## Example 1: Complete Addon with Operator

Minimal addon structure with a single operator, menu entry, and proper registration.

```python
# Blender 3.x/4.x/5.x
bl_info = {
    "name": "My Custom Tools",
    "author": "Developer",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "location": "View3D > Object > My Custom Tools",
    "description": "Example addon with a custom operator",
    "category": "Object",
}
# NOTE: In Blender 4.2+ extensions, use blender_manifest.toml instead of bl_info

import bpy


class MYCTOOLS_OT_duplicate_linked(bpy.types.Operator):
    """Duplicate selected objects as linked copies"""
    bl_idname = "myctools.duplicate_linked"
    bl_label = "Duplicate Linked"
    bl_options = {'REGISTER', 'UNDO'}

    offset: bpy.props.FloatVectorProperty(
        name="Offset",
        default=(2.0, 0.0, 0.0),
        subtype='TRANSLATION',
    )

    @classmethod
    def poll(cls, context):
        return (
            context.mode == 'OBJECT'
            and len(context.selected_objects) > 0
        )

    def execute(self, context):
        original_selection = context.selected_objects[:]
        for obj in original_selection:
            new_obj = obj.copy()  # Linked copy (shares mesh data)
            new_obj.location = obj.location.copy()
            new_obj.location.x += self.offset[0]
            new_obj.location.y += self.offset[1]
            new_obj.location.z += self.offset[2]
            context.collection.objects.link(new_obj)
        self.report({'INFO'}, f"Duplicated {len(original_selection)} objects")
        return {'FINISHED'}


def menu_func(self, context):
    self.layout.operator(MYCTOOLS_OT_duplicate_linked.bl_idname,
                         icon='DUPLICATE')


def register():
    bpy.utils.register_class(MYCTOOLS_OT_duplicate_linked)
    bpy.types.VIEW3D_MT_object.append(menu_func)


def unregister():
    bpy.types.VIEW3D_MT_object.remove(menu_func)
    bpy.utils.unregister_class(MYCTOOLS_OT_duplicate_linked)


if __name__ == "__main__":
    register()
```

---

## Example 2: Modal Operator — Interactive Move Along Normal

Modal operator that lets the user interactively move an object along its local Z axis by moving the mouse.

```python
# Blender 3.x/4.x/5.x
import bpy
from mathutils import Vector


class MYTOOLS_OT_move_along_normal(bpy.types.Operator):
    """Move active object along its local Z axis with mouse"""
    bl_idname = "mytools.move_along_normal"
    bl_label = "Move Along Normal"
    bl_options = {'REGISTER', 'UNDO', 'GRAB_CURSOR_Y'}

    offset: bpy.props.FloatProperty(
        name="Offset",
        default=0.0,
    )

    _initial_mouse_y: int = 0
    _initial_location: Vector = None

    @classmethod
    def poll(cls, context):
        return context.active_object is not None and context.mode == 'OBJECT'

    def invoke(self, context, event):
        self._initial_mouse_y = event.mouse_y
        self._initial_location = context.active_object.location.copy()
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'MOUSEMOVE':
            delta = (event.mouse_y - self._initial_mouse_y) * 0.01
            self.offset = delta
            obj = context.active_object
            local_z = obj.matrix_world.to_3x3() @ Vector((0, 0, 1))
            obj.location = self._initial_location + local_z * delta
            context.area.header_text_set(f"Offset: {delta:.3f}")

        elif event.type == 'LEFTMOUSE' and event.value == 'PRESS':
            context.area.header_text_set(None)  # Restore header
            return {'FINISHED'}

        elif event.type in {'RIGHTMOUSE', 'ESC'}:
            context.active_object.location = self._initial_location
            context.area.header_text_set(None)
            return {'CANCELLED'}

        return {'RUNNING_MODAL'}
```

---

## Example 3: Operator with File Browser

Operator that opens a file browser for the user to select a file.

```python
# Blender 3.x/4.x/5.x
import bpy
import os


class MYTOOLS_OT_import_csv(bpy.types.Operator):
    """Import data from a CSV file"""
    bl_idname = "mytools.import_csv"
    bl_label = "Import CSV"
    bl_options = {'REGISTER', 'UNDO'}

    filepath: bpy.props.StringProperty(
        name="File Path",
        subtype='FILE_PATH',
    )
    filter_glob: bpy.props.StringProperty(
        default="*.csv",
        options={'HIDDEN'},
    )

    def invoke(self, context, event):
        context.window_manager.fileselect_add(self)
        return {'RUNNING_MODAL'}

    def execute(self, context):
        if not os.path.exists(self.filepath):
            self.report({'ERROR'}, f"File not found: {self.filepath}")
            return {'CANCELLED'}

        with open(self.filepath, 'r') as f:
            lines = f.readlines()

        self.report({'INFO'}, f"Read {len(lines)} lines from {self.filepath}")
        return {'FINISHED'}


# Register in Import menu
def menu_func_import(self, context):
    self.layout.operator(MYTOOLS_OT_import_csv.bl_idname, text="CSV (.csv)")

def register():
    bpy.utils.register_class(MYTOOLS_OT_import_csv)
    bpy.types.TOPBAR_MT_file_import.append(menu_func_import)

def unregister():
    bpy.types.TOPBAR_MT_file_import.remove(menu_func_import)
    bpy.utils.unregister_class(MYTOOLS_OT_import_csv)
```

---

## Example 4: Modal Timer Operator

Operator that executes periodic work using a timer.

```python
# Blender 3.x/4.x/5.x
import bpy


class MYTOOLS_OT_auto_save(bpy.types.Operator):
    """Auto-save the file at regular intervals"""
    bl_idname = "mytools.auto_save"
    bl_label = "Auto Save Timer"
    bl_options = {'REGISTER'}

    interval: bpy.props.FloatProperty(
        name="Interval (seconds)",
        default=60.0,
        min=5.0,
    )

    _timer = None
    _count = 0

    def modal(self, context, event):
        if event.type in {'ESC'}:
            self.cancel(context)
            self.report({'INFO'}, "Auto-save stopped")
            return {'CANCELLED'}

        if event.type == 'TIMER':
            self._count += 1
            if bpy.data.is_saved:
                bpy.ops.wm.save_mainfile()
                self.report({'INFO'}, f"Auto-saved ({self._count})")

        return {'PASS_THROUGH'}

    def invoke(self, context, event):
        self._timer = context.window_manager.event_timer_add(
            self.interval, window=context.window
        )
        context.window_manager.modal_handler_add(self)
        self.report({'INFO'}, f"Auto-save started (every {self.interval}s)")
        return {'RUNNING_MODAL'}

    def cancel(self, context):
        if self._timer is not None:
            context.window_manager.event_timer_remove(self._timer)
            self._timer = None
```

---

## Example 5: Operator with Enum Property and Draw Override

```python
# Blender 3.x/4.x/5.x
import bpy


class MYTOOLS_OT_create_primitive(bpy.types.Operator):
    """Create a primitive mesh object with options"""
    bl_idname = "mytools.create_primitive"
    bl_label = "Create Primitive"
    bl_options = {'REGISTER', 'UNDO'}

    primitive_type: bpy.props.EnumProperty(
        name="Type",
        items=[
            ('CUBE', "Cube", "Create a cube", 'MESH_CUBE', 0),
            ('SPHERE', "Sphere", "Create a UV sphere", 'MESH_UVSPHERE', 1),
            ('CYLINDER', "Cylinder", "Create a cylinder", 'MESH_CYLINDER', 2),
            ('CONE', "Cone", "Create a cone", 'MESH_CONE', 3),
        ],
        default='CUBE',
    )
    size: bpy.props.FloatProperty(name="Size", default=1.0, min=0.01)
    location_offset: bpy.props.FloatVectorProperty(
        name="Location",
        default=(0.0, 0.0, 0.0),
        subtype='TRANSLATION',
    )

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self, width=250)

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "primitive_type")
        layout.prop(self, "size")
        layout.prop(self, "location_offset")

    def execute(self, context):
        ops_map = {
            'CUBE': lambda: bpy.ops.mesh.primitive_cube_add(
                size=self.size, location=self.location_offset),
            'SPHERE': lambda: bpy.ops.mesh.primitive_uv_sphere_add(
                radius=self.size / 2, location=self.location_offset),
            'CYLINDER': lambda: bpy.ops.mesh.primitive_cylinder_add(
                radius=self.size / 2, location=self.location_offset),
            'CONE': lambda: bpy.ops.mesh.primitive_cone_add(
                radius1=self.size / 2, location=self.location_offset),
        }
        ops_map[self.primitive_type]()
        self.report({'INFO'}, f"Created {self.primitive_type.lower()}")
        return {'FINISHED'}
```

---

## Example 6: Context Override for Batch Operations (Blender 4.0+)

```python
# Blender 4.0+ ONLY — context.temp_override is REQUIRED
import bpy


def apply_all_modifiers(obj):
    """Apply all modifiers on an object using context override."""
    # MUST override active_object because modifier_apply checks context
    with bpy.context.temp_override(active_object=obj, object=obj):
        for mod in obj.modifiers[:]:  # Copy list — modifiers are removed during iteration
            try:
                bpy.ops.object.modifier_apply(modifier=mod.name)
            except RuntimeError as e:
                print(f"Cannot apply {mod.name}: {e}")


def batch_shade_smooth(objects):
    """Apply smooth shading to a list of objects."""
    for obj in objects:
        with bpy.context.temp_override(
            active_object=obj,
            selected_objects=[obj],
            object=obj,
        ):
            bpy.ops.object.shade_smooth()


# Usage:
# apply_all_modifiers(bpy.context.active_object)
# batch_shade_smooth(bpy.context.selected_objects)
```

---

## Example 7: Operator with Multiple Return Paths

Demonstrates proper use of different return values.

```python
# Blender 3.x/4.x/5.x
import bpy


class MYTOOLS_OT_safe_delete(bpy.types.Operator):
    """Delete active object with safety checks"""
    bl_idname = "mytools.safe_delete"
    bl_label = "Safe Delete"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def invoke(self, context, event):
        obj = context.active_object

        # Check for protected objects
        if obj.get("protected"):
            self.report({'WARNING'}, f"'{obj.name}' is protected")
            return {'CANCELLED'}

        # Check for objects with children
        if obj.children:
            return context.window_manager.invoke_confirm(self, event)

        # Simple case — execute directly
        return self.execute(context)

    def execute(self, context):
        obj = context.active_object
        name = obj.name
        bpy.data.objects.remove(obj, do_unlink=True)
        self.report({'INFO'}, f"Deleted '{name}'")
        return {'FINISHED'}
```

---

## Example 8: Viewport Operator with Area Override

Running a viewport-dependent operator from a script or timer.

```python
# Blender 4.0+
import bpy


def find_view3d_context():
    """Find a VIEW_3D area and return override kwargs."""
    for window in bpy.context.window_manager.windows:
        for area in window.screen.areas:
            if area.type == 'VIEW_3D':
                for region in area.regions:
                    if region.type == 'WINDOW':
                        return {
                            'window': window,
                            'area': area,
                            'region': region,
                        }
    return None


def snap_cursor_to_selected():
    """Snap 3D cursor to selected objects, handling missing viewport."""
    ctx = find_view3d_context()
    if ctx is None:
        print("No 3D Viewport found")
        return False

    with bpy.context.temp_override(**ctx):
        bpy.ops.view3d.snap_cursor_to_selected()
    return True
```

---

## Example 9: Macro Operator (Combining Operators)

```python
# Blender 3.x/4.x/5.x
import bpy


class MYTOOLS_OT_duplicate_and_move(bpy.types.Macro):
    """Duplicate object and move it"""
    bl_idname = "mytools.duplicate_and_move"
    bl_label = "Duplicate and Move"
    bl_options = {'REGISTER', 'UNDO'}


def register():
    bpy.utils.register_class(MYTOOLS_OT_duplicate_and_move)

    # Define the macro steps
    MYTOOLS_OT_duplicate_and_move.define("OBJECT_OT_duplicate")
    MYTOOLS_OT_duplicate_and_move.define("TRANSFORM_OT_translate")


def unregister():
    bpy.utils.unregister_class(MYTOOLS_OT_duplicate_and_move)
```

---

## Example 10: Registering Multiple Operators with Factory

```python
# Blender 3.x/4.x/5.x
import bpy


class MYTOOLS_OT_action_a(bpy.types.Operator):
    bl_idname = "mytools.action_a"
    bl_label = "Action A"
    bl_options = {'REGISTER', 'UNDO'}

    def execute(self, context):
        self.report({'INFO'}, "Action A executed")
        return {'FINISHED'}


class MYTOOLS_OT_action_b(bpy.types.Operator):
    bl_idname = "mytools.action_b"
    bl_label = "Action B"
    bl_options = {'REGISTER', 'UNDO'}

    def execute(self, context):
        self.report({'INFO'}, "Action B executed")
        return {'FINISHED'}


class MYTOOLS_PT_tools_panel(bpy.types.Panel):
    bl_label = "My Tools"
    bl_idname = "MYTOOLS_PT_tools_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Tools"

    def draw(self, context):
        layout = self.layout
        layout.operator("mytools.action_a")
        layout.operator("mytools.action_b")


# Factory registration — handles order automatically
classes = (
    MYTOOLS_OT_action_a,
    MYTOOLS_OT_action_b,
    MYTOOLS_PT_tools_panel,
)
register, unregister = bpy.utils.register_classes_factory(classes)
```

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/info_tutorial_addon.html
- https://docs.blender.org/api/current/bpy.types.Macro.html
- https://docs.blender.org/api/current/bpy.ops.html
