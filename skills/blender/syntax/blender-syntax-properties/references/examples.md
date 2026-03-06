# blender-syntax-properties: Working Code Examples

Sources: https://docs.blender.org/api/current/bpy.props.html,
https://docs.blender.org/api/current/bpy.types.PropertyGroup.html

All examples verified against Blender Python API documentation.

---

## Example 1: Complete Addon with PropertyGroup

```python
# Blender 3.x/4.x/5.x — full addon pattern with custom properties
import bpy

class MyAddonSettings(bpy.types.PropertyGroup):
    """Addon-wide settings stored on the Scene."""
    enabled: bpy.props.BoolProperty(
        name="Enabled",
        description="Enable the addon functionality",
        default=True,
    )
    iterations: bpy.props.IntProperty(
        name="Iterations",
        description="Number of processing iterations",
        default=3,
        min=1,
        max=100,
        soft_max=20,
    )
    scale_factor: bpy.props.FloatProperty(
        name="Scale Factor",
        description="Scale multiplier for output",
        default=1.0,
        min=0.001,
        max=1000.0,
        soft_min=0.1,
        soft_max=10.0,
        step=10,         # UI increment: 10/100 = 0.1
        precision=3,
        subtype='FACTOR',
    )
    output_path: bpy.props.StringProperty(
        name="Output Path",
        description="Directory for output files",
        default="//output/",
        subtype='DIR_PATH',
    )
    mode: bpy.props.EnumProperty(
        name="Mode",
        description="Processing mode",
        items=[
            ('FAST', "Fast", "Quick processing with lower quality", 'PLAY', 0),
            ('BALANCED', "Balanced", "Balanced quality and speed", 'PAUSE', 1),
            ('QUALITY', "Quality", "Best quality, slower processing", 'RENDER_STILL', 2),
        ],
        default='BALANCED',
    )


class MY_OT_process(bpy.types.Operator):
    bl_idname = "my_addon.process"
    bl_label = "Process"
    bl_options = {'REGISTER', 'UNDO'}

    def execute(self, context):
        settings = context.scene.my_addon_settings
        if not settings.enabled:
            self.report({'WARNING'}, "Addon is disabled")
            return {'CANCELLED'}

        self.report({'INFO'}, f"Processing {settings.iterations} iterations "
                              f"at scale {settings.scale_factor} "
                              f"in {settings.mode} mode")
        return {'FINISHED'}


class MY_PT_panel(bpy.types.Panel):
    bl_label = "My Addon"
    bl_idname = "MY_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "My Addon"

    def draw(self, context):
        layout = self.layout
        settings = context.scene.my_addon_settings

        layout.prop(settings, "enabled")

        col = layout.column()
        col.enabled = settings.enabled
        col.prop(settings, "mode")
        col.prop(settings, "iterations")
        col.prop(settings, "scale_factor")
        col.prop(settings, "output_path")
        col.operator("my_addon.process")


def register():
    bpy.utils.register_class(MyAddonSettings)
    bpy.utils.register_class(MY_OT_process)
    bpy.utils.register_class(MY_PT_panel)
    bpy.types.Scene.my_addon_settings = bpy.props.PointerProperty(
        type=MyAddonSettings
    )

def unregister():
    del bpy.types.Scene.my_addon_settings
    bpy.utils.unregister_class(MY_PT_panel)
    bpy.utils.unregister_class(MY_OT_process)
    bpy.utils.unregister_class(MyAddonSettings)

if __name__ == "__main__":
    register()
```

---

## Example 2: Nested PropertyGroups with CollectionProperty

```python
# Blender 3.x/4.x/5.x — dynamic list with UI list widget
import bpy

class MaterialSlot(bpy.types.PropertyGroup):
    """Single item in a material list."""
    material: bpy.props.PointerProperty(
        type=bpy.types.Material,
        name="Material",
    )
    weight: bpy.props.FloatProperty(
        name="Weight",
        default=1.0,
        min=0.0,
        max=1.0,
        subtype='FACTOR',
    )
    enabled: bpy.props.BoolProperty(
        name="Enabled",
        default=True,
    )


class MaterialListSettings(bpy.types.PropertyGroup):
    """Container for the material list."""
    items: bpy.props.CollectionProperty(type=MaterialSlot)
    active_index: bpy.props.IntProperty(name="Active Index")


class MATLIST_OT_add(bpy.types.Operator):
    bl_idname = "matlist.add"
    bl_label = "Add Material Slot"

    def execute(self, context):
        settings = context.scene.matlist_settings
        item = settings.items.add()
        item.name = f"Slot {len(settings.items)}"
        settings.active_index = len(settings.items) - 1
        return {'FINISHED'}


class MATLIST_OT_remove(bpy.types.Operator):
    bl_idname = "matlist.remove"
    bl_label = "Remove Material Slot"

    @classmethod
    def poll(cls, context):
        settings = context.scene.matlist_settings
        return len(settings.items) > 0

    def execute(self, context):
        settings = context.scene.matlist_settings
        settings.items.remove(settings.active_index)
        settings.active_index = max(0, settings.active_index - 1)
        return {'FINISHED'}


class MATLIST_UL_list(bpy.types.UIList):
    def draw_item(self, context, layout, data, item, icon, active_data,
                  active_property, index):
        if self.layout_type in {'DEFAULT', 'COMPACT'}:
            row = layout.row(align=True)
            row.prop(item, "enabled", text="")
            row.prop(item, "material", text="")
            row.prop(item, "weight", text="W")
        elif self.layout_type == 'GRID':
            layout.alignment = 'CENTER'
            layout.label(text=item.name)


class MATLIST_PT_panel(bpy.types.Panel):
    bl_label = "Material List"
    bl_idname = "MATLIST_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "MatList"

    def draw(self, context):
        layout = self.layout
        settings = context.scene.matlist_settings

        row = layout.row()
        row.template_list("MATLIST_UL_list", "", settings, "items",
                          settings, "active_index")

        col = row.column(align=True)
        col.operator("matlist.add", icon='ADD', text="")
        col.operator("matlist.remove", icon='REMOVE', text="")


classes = (
    MaterialSlot,           # Sub-type FIRST
    MaterialListSettings,   # Parent type SECOND
    MATLIST_OT_add,
    MATLIST_OT_remove,
    MATLIST_UL_list,
    MATLIST_PT_panel,
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.matlist_settings = bpy.props.PointerProperty(
        type=MaterialListSettings
    )

def unregister():
    del bpy.types.Scene.matlist_settings
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 3: Dynamic Enum with Safe Caching

```python
# Blender 3.x/4.x/5.x — dynamic enum with garbage collection protection
import bpy

_collection_items = []  # Module-level cache — REQUIRED

def get_collection_items(self, context):
    """Build enum items from scene collections.
    ALWAYS store result in module-level variable to prevent GC crash."""
    global _collection_items
    _collection_items = []
    for i, col in enumerate(bpy.data.collections):
        _collection_items.append(
            (col.name, col.name, f"Collection: {col.name}", 'OUTLINER_COLLECTION', i)
        )
    if not _collection_items:
        _collection_items = [('NONE', "None", "No collections found", 'ERROR', 0)]
    return _collection_items


class CollectionPicker(bpy.types.PropertyGroup):
    target_collection: bpy.props.EnumProperty(
        name="Collection",
        description="Select a collection",
        items=get_collection_items,
        # default MUST be omitted or None when items is a callback
    )

def register():
    bpy.utils.register_class(CollectionPicker)
    bpy.types.Scene.col_picker = bpy.props.PointerProperty(type=CollectionPicker)

def unregister():
    del bpy.types.Scene.col_picker
    bpy.utils.unregister_class(CollectionPicker)
```

---

## Example 4: Update Callbacks with Cross-Property Dependencies

```python
# Blender 3.x/4.x/5.x — safe update callback patterns
import bpy

def on_width_changed(self, context):
    """Update area when width changes. SAFE: modifies OTHER properties."""
    self["_area"] = self.width * self.height

def on_height_changed(self, context):
    """Update area when height changes. SAFE: modifies OTHER properties."""
    self["_area"] = self.width * self.height

def get_area(self):
    """Computed property via getter."""
    return self.get("_area", self.width * self.height)


class DimensionSettings(bpy.types.PropertyGroup):
    width: bpy.props.FloatProperty(
        name="Width",
        default=1.0,
        min=0.001,
        unit='LENGTH',
        update=on_width_changed,
    )
    height: bpy.props.FloatProperty(
        name="Height",
        default=1.0,
        min=0.001,
        unit='LENGTH',
        update=on_height_changed,
    )
    area: bpy.props.FloatProperty(
        name="Area",
        unit='AREA',
        get=get_area,
        # No set = read-only property
    )


def register():
    bpy.utils.register_class(DimensionSettings)
    bpy.types.Scene.dimensions = bpy.props.PointerProperty(type=DimensionSettings)

def unregister():
    del bpy.types.Scene.dimensions
    bpy.utils.unregister_class(DimensionSettings)
```

---

## Example 5: Getter/Setter with EnumProperty

```python
# Blender 3.x/4.x/5.x — EnumProperty with integer-based get/set
import bpy

# Map between string identifiers and integer values
_MODE_MAP = {'WIREFRAME': 0, 'SOLID': 1, 'MATERIAL': 2, 'RENDERED': 3}
_MODE_REVERSE = {v: k for k, v in _MODE_MAP.items()}

def get_shading_mode(self):
    """EnumProperty getter MUST return integer, NOT string."""
    current = self.get("_shading_mode", 1)  # default: SOLID
    return current

def set_shading_mode(self, value):
    """EnumProperty setter receives integer, NOT string."""
    self["_shading_mode"] = value
    # Optionally apply the mode
    mode_name = _MODE_REVERSE.get(value, 'SOLID')
    print(f"Shading mode set to: {mode_name}")


class ViewSettings(bpy.types.PropertyGroup):
    shading_mode: bpy.props.EnumProperty(
        name="Shading",
        items=[
            # 5-tuple REQUIRED when using get/set — number field is the int value
            ('WIREFRAME', "Wireframe", "Wire display", 'SHADING_WIRE', 0),
            ('SOLID', "Solid", "Solid display", 'SHADING_SOLID', 1),
            ('MATERIAL', "Material", "Material preview", 'MATERIAL', 2),
            ('RENDERED', "Rendered", "Rendered preview", 'SHADING_RENDERED', 3),
        ],
        get=get_shading_mode,
        set=set_shading_mode,
    )


def register():
    bpy.utils.register_class(ViewSettings)
    bpy.types.Scene.view_settings = bpy.props.PointerProperty(type=ViewSettings)

def unregister():
    del bpy.types.Scene.view_settings
    bpy.utils.unregister_class(ViewSettings)
```

---

## Example 6: ENUM_FLAG Multi-Select

```python
# Blender 3.x/4.x/5.x — multi-select enum with bitmask values
import bpy

class ExportSettings(bpy.types.PropertyGroup):
    export_axes: bpy.props.EnumProperty(
        name="Export Axes",
        description="Select which axes to export",
        items=[
            ('X', "X", "Export X axis data", 1),   # Power of 2
            ('Y', "Y", "Export Y axis data", 2),   # Power of 2
            ('Z', "Z", "Export Z axis data", 4),   # Power of 2
        ],
        options={'ENUM_FLAG'},
        default={'X', 'Y', 'Z'},  # Default is a SET
    )
    export_types: bpy.props.EnumProperty(
        name="Export Types",
        description="Select data types to include",
        items=[
            ('MESH', "Meshes", "Include mesh data", 'MESH_DATA', 1),
            ('CURVE', "Curves", "Include curve data", 'CURVE_DATA', 2),
            ('LIGHT', "Lights", "Include light data", 'LIGHT_DATA', 4),
            ('CAMERA', "Cameras", "Include camera data", 'CAMERA_DATA', 8),
        ],
        options={'ENUM_FLAG'},
        default={'MESH'},
    )


def register():
    bpy.utils.register_class(ExportSettings)
    bpy.types.Scene.export_settings = bpy.props.PointerProperty(
        type=ExportSettings
    )

def unregister():
    del bpy.types.Scene.export_settings
    bpy.utils.unregister_class(ExportSettings)
```

---

## Example 7: PointerProperty with Poll Filter

```python
# Blender 3.x/4.x/5.x — filtered object selection
import bpy

def poll_armature_objects(self, obj):
    """Only allow armature objects to be selected."""
    return obj.type == 'ARMATURE'

def poll_mesh_with_uv(self, obj):
    """Only allow mesh objects that have UV maps."""
    return (obj.type == 'MESH' and
            obj.data is not None and
            len(obj.data.uv_layers) > 0)


class AnimationTarget(bpy.types.PropertyGroup):
    armature: bpy.props.PointerProperty(
        type=bpy.types.Object,
        name="Armature",
        description="Target armature for animation",
        poll=poll_armature_objects,
    )
    mesh_target: bpy.props.PointerProperty(
        type=bpy.types.Object,
        name="UV Mesh",
        description="Mesh with UV coordinates",
        poll=poll_mesh_with_uv,
    )


def register():
    bpy.utils.register_class(AnimationTarget)
    bpy.types.Scene.anim_target = bpy.props.PointerProperty(
        type=AnimationTarget
    )

def unregister():
    del bpy.types.Scene.anim_target
    bpy.utils.unregister_class(AnimationTarget)
```

---

## Example 8: Color and Vector Properties

```python
# Blender 3.x/4.x/5.x — vector property variants
import bpy

class MaterialSettings(bpy.types.PropertyGroup):
    # RGB color (linear space) — shows color picker
    base_color: bpy.props.FloatVectorProperty(
        name="Base Color",
        subtype='COLOR',
        default=(0.8, 0.8, 0.8),
        min=0.0,
        max=1.0,
        size=3,
    )
    # RGBA color (gamma space) — shows color picker with alpha
    overlay_color: bpy.props.FloatVectorProperty(
        name="Overlay",
        subtype='COLOR_GAMMA',
        default=(1.0, 0.0, 0.0, 0.5),
        min=0.0,
        max=1.0,
        size=4,
    )
    # Position in scene units
    offset: bpy.props.FloatVectorProperty(
        name="Offset",
        subtype='TRANSLATION',
        unit='LENGTH',
        default=(0.0, 0.0, 0.0),
        size=3,
    )
    # Euler rotation (stored as radians, displayed as degrees)
    rotation: bpy.props.FloatVectorProperty(
        name="Rotation",
        subtype='EULER',
        unit='ROTATION',
        default=(0.0, 0.0, 0.0),
        size=3,
    )
    # Direction (auto-normalized by Blender UI)
    direction: bpy.props.FloatVectorProperty(
        name="Direction",
        subtype='DIRECTION',
        default=(0.0, 0.0, 1.0),
        size=3,
    )
    # Axis toggles
    mirror_axes: bpy.props.BoolVectorProperty(
        name="Mirror Axes",
        default=(True, False, False),
        size=3,
        subtype='XYZ',
    )

def register():
    bpy.utils.register_class(MaterialSettings)
    bpy.types.Scene.mat_settings = bpy.props.PointerProperty(
        type=MaterialSettings
    )

def unregister():
    del bpy.types.Scene.mat_settings
    bpy.utils.unregister_class(MaterialSettings)
```

---

## Example 9: Per-Object Properties

```python
# Blender 3.x/4.x/5.x — properties attached to individual objects
import bpy

class ObjectMetadata(bpy.types.PropertyGroup):
    category: bpy.props.EnumProperty(
        name="Category",
        items=[
            ('STRUCTURE', "Structure", "Structural element"),
            ('FURNITURE', "Furniture", "Furniture item"),
            ('FIXTURE', "Fixture", "Fixed installation"),
            ('EQUIPMENT', "Equipment", "Equipment item"),
        ],
        default='STRUCTURE',
    )
    cost: bpy.props.FloatProperty(
        name="Cost",
        description="Estimated cost",
        default=0.0,
        min=0.0,
        precision=2,
    )
    notes: bpy.props.StringProperty(
        name="Notes",
        description="Additional notes",
        default="",
    )


class OBJMETA_PT_panel(bpy.types.Panel):
    bl_label = "Object Metadata"
    bl_idname = "OBJMETA_PT_panel"
    bl_space_type = 'PROPERTIES'
    bl_region_type = 'WINDOW'
    bl_context = "object"

    def draw(self, context):
        obj = context.object
        if obj is None:
            return
        layout = self.layout
        meta = obj.obj_metadata
        layout.prop(meta, "category")
        layout.prop(meta, "cost")
        layout.prop(meta, "notes")


def register():
    bpy.utils.register_class(ObjectMetadata)
    bpy.utils.register_class(OBJMETA_PT_panel)
    # Attach to Object type — each object gets its own instance
    bpy.types.Object.obj_metadata = bpy.props.PointerProperty(
        type=ObjectMetadata
    )

def unregister():
    del bpy.types.Object.obj_metadata
    bpy.utils.unregister_class(OBJMETA_PT_panel)
    bpy.utils.unregister_class(ObjectMetadata)
```

---

## Example 10: WindowManager Properties (Session-Only)

```python
# Blender 3.x/4.x/5.x — temporary properties that reset on restart
import bpy

class SessionState(bpy.types.PropertyGroup):
    is_processing: bpy.props.BoolProperty(
        name="Processing",
        default=False,
    )
    progress: bpy.props.FloatProperty(
        name="Progress",
        default=0.0,
        min=0.0,
        max=1.0,
        subtype='FACTOR',
    )
    status_message: bpy.props.StringProperty(
        name="Status",
        default="Ready",
    )

def register():
    bpy.utils.register_class(SessionState)
    # WindowManager: not saved to .blend, resets each session
    bpy.types.WindowManager.session = bpy.props.PointerProperty(
        type=SessionState
    )

def unregister():
    del bpy.types.WindowManager.session
    bpy.utils.unregister_class(SessionState)
```

---

## Example 11: Blender 5.0 get_transform / set_transform

```python
# Blender 5.0+ ONLY — value transformation with internal storage
import bpy
import math

def angle_get_transform(self, curr_value, is_set):
    """Display stored radians as degrees."""
    return math.degrees(curr_value) if is_set else 0.0

def angle_set_transform(self, new_value, curr_value, is_set):
    """Convert input degrees to radians for storage."""
    return math.radians(new_value)

def path_get_transform(self, curr_value, is_set):
    """Validate path exists on read."""
    import os
    if is_set and os.path.exists(curr_value):
        return curr_value
    return ""

def path_set_transform(self, new_value, curr_value, is_set):
    """Only store valid paths."""
    import os
    return new_value if os.path.isdir(new_value) else curr_value


class TransformSettings(bpy.types.PropertyGroup):
    custom_angle: bpy.props.FloatProperty(
        name="Angle (degrees)",
        get_transform=angle_get_transform,
        set_transform=angle_set_transform,
    )
    validated_path: bpy.props.StringProperty(
        name="Path",
        subtype='DIR_PATH',
        get_transform=path_get_transform,
        set_transform=path_set_transform,
    )

def register():
    bpy.utils.register_class(TransformSettings)
    bpy.types.Scene.transform_settings = bpy.props.PointerProperty(
        type=TransformSettings
    )

def unregister():
    del bpy.types.Scene.transform_settings
    bpy.utils.unregister_class(TransformSettings)
```

---

## Example 12: Blender 5.0 property_unset

```python
# Blender 5.0+ — resetting properties to defaults
import bpy

scene = bpy.context.scene

# Check if a property has been explicitly set
if scene.my_settings.is_property_set("iterations"):
    print("iterations was explicitly set")

# Reset to default value (replaces del obj["prop"] pattern)
scene.my_settings.property_unset("iterations")

# After unset, is_property_set returns False
assert not scene.my_settings.is_property_set("iterations")
```

---

## Example 13: Blender 4.1 Enum ID Properties

```python
# Blender 4.1+ — integer custom properties displayed as enums
import bpy

obj = bpy.context.active_object

# Method 1: Direct id_properties_ui
obj["building_type"] = 0
ui = obj.id_properties_ui("building_type")
ui.update(items=[
    ("RESIDENTIAL", "Residential", "Residential building"),
    ("COMMERCIAL", "Commercial", "Commercial building"),
    ("INDUSTRIAL", "Industrial", "Industrial building"),
    ("MIXED_USE", "Mixed Use", "Mixed-use building"),
])

# Method 2: rna_prop_ui convenience wrapper
from rna_prop_ui import rna_idprop_ui_create
rna_idprop_ui_create(
    obj, "structural_system", default=0,
    items=[
        ("STEEL", "Steel Frame", "Steel structural system"),
        ("CONCRETE", "Concrete", "Reinforced concrete"),
        ("TIMBER", "Timber", "Timber frame construction"),
    ],
)
```
