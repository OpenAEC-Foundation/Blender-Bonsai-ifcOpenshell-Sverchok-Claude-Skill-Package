# blender-syntax-addons — Working Examples

## Example 1: Minimal Single-File Addon (Legacy)

Smallest complete addon for Blender < 4.2.

```python
# my_addon.py — single-file legacy addon
bl_info = {
    "name": "Minimal Addon",
    "author": "Developer",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),
    "category": "Object",
    "description": "Demonstrates minimal addon structure",
}

import bpy


class MINIMAL_OT_hello(bpy.types.Operator):
    """Print a greeting to the console"""
    bl_idname = "minimal.hello"
    bl_label = "Say Hello"

    def execute(self, context):
        self.report({'INFO'}, "Hello from Minimal Addon!")
        return {'FINISHED'}


class MINIMAL_PT_panel(bpy.types.Panel):
    bl_label = "Minimal Addon"
    bl_idname = "MINIMAL_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Minimal"

    def draw(self, context):
        self.layout.operator("minimal.hello")


classes = (MINIMAL_OT_hello, MINIMAL_PT_panel)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)


def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)


if __name__ == "__main__":
    register()
```

---

## Example 2: Minimal Extension (Blender 4.2+)

Smallest complete extension with `blender_manifest.toml`.

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "minimal_extension"
version = "1.0.0"
name = "Minimal Extension"
tagline = "Demonstrates minimal extension structure"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Object"]
```

**__init__.py:**
```python
import bpy


class MINEXT_OT_hello(bpy.types.Operator):
    """Print a greeting to the console"""
    bl_idname = "minext.hello"
    bl_label = "Say Hello"

    def execute(self, context):
        self.report({'INFO'}, "Hello from Extension!")
        return {'FINISHED'}


class MINEXT_PT_panel(bpy.types.Panel):
    bl_label = "Minimal Extension"
    bl_idname = "MINEXT_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "MinExt"

    def draw(self, context):
        self.layout.operator("minext.hello")


classes = (MINEXT_OT_hello, MINEXT_PT_panel)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)


def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 3: Multi-File Addon with PropertyGroup and Preferences

Complete multi-file addon with properties, preferences, and operators.

**Directory structure:**
```
my_toolkit/
├── __init__.py
├── blender_manifest.toml   # Include for 4.2+ support
├── operators.py
├── panels.py
├── properties.py
└── preferences.py
```

**__init__.py:**
```python
bl_info = {
    "name": "My Toolkit",
    "author": "Developer",
    "version": (2, 0, 0),
    "blender": (3, 6, 0),
    "location": "View3D > Sidebar > Toolkit",
    "description": "Collection of modeling utilities",
    "category": "Mesh",
}

if "bpy" in locals():
    import importlib
    importlib.reload(properties)
    importlib.reload(operators)
    importlib.reload(panels)
    importlib.reload(preferences)
else:
    from . import properties
    from . import operators
    from . import panels
    from . import preferences

import bpy

classes = (
    # PropertyGroups first
    properties.TOOLKIT_PG_settings,
    # Then preferences
    preferences.ToolkitPreferences,
    # Then operators
    operators.TOOLKIT_OT_align_objects,
    operators.TOOLKIT_OT_batch_rename,
    # Then UI
    panels.TOOLKIT_PT_main,
    panels.TOOLKIT_PT_align,
    panels.TOOLKIT_PT_rename,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.toolkit_settings = bpy.props.PointerProperty(
        type=properties.TOOLKIT_PG_settings
    )


def unregister():
    del bpy.types.Scene.toolkit_settings
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)


if __name__ == "__main__":
    register()
```

**properties.py:**
```python
import bpy


class TOOLKIT_PG_settings(bpy.types.PropertyGroup):
    align_axis: bpy.props.EnumProperty(
        name="Axis",
        items=[
            ('X', "X", "Align along X axis"),
            ('Y', "Y", "Align along Y axis"),
            ('Z', "Z", "Align along Z axis"),
        ],
        default='X',
    )
    rename_prefix: bpy.props.StringProperty(
        name="Prefix",
        default="obj_",
    )
    rename_start_number: bpy.props.IntProperty(
        name="Start Number",
        default=1,
        min=0,
    )
```

**operators.py:**
```python
import bpy


class TOOLKIT_OT_align_objects(bpy.types.Operator):
    """Align selected objects along an axis"""
    bl_idname = "toolkit.align_objects"
    bl_label = "Align Objects"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return len(context.selected_objects) > 1

    def execute(self, context):
        settings = context.scene.toolkit_settings
        axis_idx = {'X': 0, 'Y': 1, 'Z': 2}[settings.align_axis]

        # Get average position along axis
        objects = context.selected_objects
        avg = sum(obj.location[axis_idx] for obj in objects) / len(objects)

        # Align all selected objects
        for obj in objects:
            obj.location[axis_idx] = avg

        self.report({'INFO'}, f"Aligned {len(objects)} objects on {settings.align_axis}")
        return {'FINISHED'}


class TOOLKIT_OT_batch_rename(bpy.types.Operator):
    """Rename selected objects with prefix and sequential numbering"""
    bl_idname = "toolkit.batch_rename"
    bl_label = "Batch Rename"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return len(context.selected_objects) > 0

    def execute(self, context):
        settings = context.scene.toolkit_settings
        prefix = settings.rename_prefix
        start = settings.rename_start_number

        for i, obj in enumerate(context.selected_objects):
            obj.name = f"{prefix}{start + i:03d}"

        self.report({'INFO'}, f"Renamed {len(context.selected_objects)} objects")
        return {'FINISHED'}
```

**panels.py:**
```python
import bpy


class TOOLKIT_PT_main(bpy.types.Panel):
    bl_label = "My Toolkit"
    bl_idname = "TOOLKIT_PT_main"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Toolkit"

    def draw(self, context):
        layout = self.layout
        layout.label(text="Modeling Utilities")


class TOOLKIT_PT_align(bpy.types.Panel):
    bl_label = "Align"
    bl_idname = "TOOLKIT_PT_align"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Toolkit"
    bl_parent_id = "TOOLKIT_PT_main"

    def draw(self, context):
        layout = self.layout
        settings = context.scene.toolkit_settings
        layout.prop(settings, "align_axis")
        layout.operator("toolkit.align_objects")


class TOOLKIT_PT_rename(bpy.types.Panel):
    bl_label = "Rename"
    bl_idname = "TOOLKIT_PT_rename"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Toolkit"
    bl_parent_id = "TOOLKIT_PT_main"
    bl_options = {'DEFAULT_CLOSED'}

    def draw(self, context):
        layout = self.layout
        settings = context.scene.toolkit_settings
        layout.prop(settings, "rename_prefix")
        layout.prop(settings, "rename_start_number")
        layout.operator("toolkit.batch_rename")
```

**preferences.py:**
```python
import bpy


class ToolkitPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__

    auto_save: bpy.props.BoolProperty(
        name="Auto Save Before Operations",
        default=True,
    )
    log_level: bpy.props.EnumProperty(
        name="Log Level",
        items=[
            ('ERROR', "Error", "Only errors"),
            ('WARNING', "Warning", "Warnings and errors"),
            ('INFO', "Info", "All messages"),
        ],
        default='WARNING',
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "auto_save")
        layout.prop(self, "log_level")
```

---

## Example 4: Extension with Bundled Wheels and Network Access

Extension that bundles a third-party library and accesses the network.

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "data_fetcher"
version = "1.0.0"
name = "Data Fetcher"
tagline = "Fetches and imports data from remote APIs"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Import-Export"]

wheels = ["./wheels/requests-2.31.0-py3-none-any.whl"]

[permissions]
network = "Downloads data from configured API endpoint"
files = "Caches downloaded data locally"
```

**__init__.py:**
```python
import bpy
import os


class FETCHER_OT_download(bpy.types.Operator):
    """Download data from configured endpoint"""
    bl_idname = "fetcher.download"
    bl_label = "Download Data"

    def execute(self, context):
        # MUST check online_access before network requests
        if not bpy.app.online_access:
            self.report({'ERROR'}, "Online access is disabled in Blender preferences")
            return {'CANCELLED'}

        import requests  # Bundled in wheels/

        prefs = context.preferences.addons[__package__].preferences
        try:
            response = requests.get(prefs.api_url, timeout=10)
            response.raise_for_status()
        except requests.RequestException as e:
            self.report({'ERROR'}, f"Download failed: {e}")
            return {'CANCELLED'}

        # Write to user-writable directory (NOT addon directory)
        user_dir = bpy.utils.extension_path_user(__package__, path="cache", create=True)
        filepath = os.path.join(user_dir, "data.json")
        with open(filepath, 'w') as f:
            f.write(response.text)

        self.report({'INFO'}, f"Data saved to {filepath}")
        return {'FINISHED'}


class FetcherPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__

    api_url: bpy.props.StringProperty(
        name="API URL",
        default="https://api.example.com/data",
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "api_url")


class FETCHER_PT_panel(bpy.types.Panel):
    bl_label = "Data Fetcher"
    bl_idname = "FETCHER_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Fetcher"

    def draw(self, context):
        layout = self.layout
        layout.operator("fetcher.download")


classes = (FetcherPreferences, FETCHER_OT_download, FETCHER_PT_panel)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)


def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 5: Keymap Registration

Addon that registers keyboard shortcuts.

```python
import bpy

addon_keymaps = []


class SHORTCUT_OT_action(bpy.types.Operator):
    """Operator with keyboard shortcut"""
    bl_idname = "shortcut.action"
    bl_label = "Shortcut Action"
    bl_options = {'REGISTER', 'UNDO'}

    def execute(self, context):
        self.report({'INFO'}, "Shortcut triggered!")
        return {'FINISHED'}


classes = (SHORTCUT_OT_action,)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)

    # Register keymaps
    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    if kc:  # None in background mode
        km = kc.keymaps.new(name='3D View', space_type='VIEW_3D')
        kmi = km.keymap_items.new(
            "shortcut.action",
            type='D',
            value='PRESS',
            ctrl=True,
            shift=True,
        )
        addon_keymaps.append((km, kmi))


def unregister():
    # Remove keymaps FIRST
    for km, kmi in addon_keymaps:
        km.keymap_items.remove(kmi)
    addon_keymaps.clear()

    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 6: Migration from bl_info to Extension

Before (legacy addon `__init__.py`):

```python
bl_info = {
    "name": "My Tool",
    "author": "Jane Dev",
    "version": (2, 1, 0),
    "blender": (3, 6, 0),
    "location": "View3D > Sidebar",
    "description": "A useful modeling tool",
    "doc_url": "https://example.com/docs",
    "tracker_url": "https://example.com/issues",
    "category": "Mesh",
}

import bpy
# ... rest of addon code
```

After (extension with `blender_manifest.toml`):

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "my_tool"
version = "2.1.0"
name = "My Tool"
tagline = "A useful modeling tool"
maintainer = "Jane Dev <jane@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Mesh"]
website = "https://example.com/docs"
```

**__init__.py** (simplified — no `bl_info`, no reload boilerplate):
```python
import bpy
# ... rest of addon code (register/unregister unchanged)
```

Key changes:
1. Remove `bl_info` dict entirely from `__init__.py`
2. Create `blender_manifest.toml` with equivalent fields
3. Remove reload boilerplate (`if "bpy" in locals()` block) — extensions handle reloading automatically
4. Add `[permissions]` table if addon uses network, files, etc.
5. Bundle any pip dependencies in `wheels/` directory

---

## Example 7: Dual-Format Addon (Legacy + Extension)

Support both Blender < 4.2 and >= 4.2 from the same codebase.

```
my_dual_addon/
├── __init__.py              # Contains bl_info for legacy support
├── blender_manifest.toml    # For 4.2+ extension support
├── operators.py
└── panels.py
```

**__init__.py:**
```python
# bl_info for legacy Blender (< 4.2) — ignored when loaded as extension
bl_info = {
    "name": "Dual Format Addon",
    "author": "Developer",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),
    "description": "Works in both legacy and extension mode",
    "category": "Object",
}

# Reload support for legacy mode
if "bpy" in locals():
    import importlib
    importlib.reload(operators)
    importlib.reload(panels)
else:
    from . import operators
    from . import panels

import bpy

classes = (
    operators.DUAL_OT_example,
    panels.DUAL_PT_panel,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)


def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)


if __name__ == "__main__":
    register()
```

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "dual_format_addon"
version = "1.0.0"
name = "Dual Format Addon"
tagline = "Works in both legacy and extension mode"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Object"]
```

When Blender 4.2+ loads this as an extension, `bl_info` is simply ignored. When older Blender loads it as a legacy addon, `blender_manifest.toml` is ignored.
