# blender-impl-addons — Working Examples

## Example 1: Minimal Single-File Extension (Blender 4.2+)

The smallest complete extension with manifest and single `__init__.py`.

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "hello_extension"
version = "1.0.0"
name = "Hello Extension"
tagline = "Minimal extension demonstrating project structure"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Object"]

[build]
paths_exclude_pattern = [
    "__pycache__/",
    "*.pyc",
    ".git/",
    "tests/",
]
```

**__init__.py:**
```python
# NO bl_info — metadata lives in blender_manifest.toml
import bpy


class HELLO_OT_greet(bpy.types.Operator):
    """Print a greeting to the info area"""
    bl_idname = "hello.greet"
    bl_label = "Say Hello"

    def execute(self, context):
        self.report({'INFO'}, "Hello from extension!")
        return {'FINISHED'}


class HELLO_PT_panel(bpy.types.Panel):
    bl_label = "Hello Extension"
    bl_idname = "HELLO_PT_panel"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Hello"

    def draw(self, context):
        self.layout.operator("hello.greet")


classes = (HELLO_OT_greet, HELLO_PT_panel)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)


def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 2: Multi-File Extension with Properties (Blender 4.2+)

Full multi-file extension with operators, panels, and property groups.

**Project structure:**
```
my_toolbox/
├── blender_manifest.toml
├── __init__.py
├── operators.py
├── panels.py
└── properties.py
```

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "my_toolbox"
version = "1.2.0"
name = "My Toolbox"
tagline = "Collection of mesh utility operators"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Mesh", "Object"]

[build]
paths_exclude_pattern = [
    "__pycache__/",
    "*.pyc",
    ".git/",
    ".gitignore",
    "tests/",
    "*.blend1",
]
```

**properties.py:**
```python
import bpy


class TOOLBOX_Properties(bpy.types.PropertyGroup):
    scale_factor: bpy.props.FloatProperty(
        name="Scale Factor",
        description="Uniform scale multiplier",
        default=1.0,
        min=0.01,
        max=100.0,
    )
    apply_to_selected: bpy.props.BoolProperty(
        name="Apply to Selected",
        description="Apply operation to all selected objects",
        default=True,
    )
```

**operators.py:**
```python
import bpy


class TOOLBOX_OT_uniform_scale(bpy.types.Operator):
    """Scale selected objects by the configured factor"""
    bl_idname = "toolbox.uniform_scale"
    bl_label = "Uniform Scale"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return context.selected_objects

    def execute(self, context):
        props = context.scene.toolbox_props
        factor = props.scale_factor
        targets = context.selected_objects if props.apply_to_selected else [context.active_object]
        for obj in targets:
            if obj is not None:
                obj.scale *= factor
        self.report({'INFO'}, f"Scaled {len(targets)} object(s) by {factor}")
        return {'FINISHED'}


class TOOLBOX_OT_reset_scale(bpy.types.Operator):
    """Reset scale of selected objects to (1, 1, 1)"""
    bl_idname = "toolbox.reset_scale"
    bl_label = "Reset Scale"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return context.selected_objects

    def execute(self, context):
        for obj in context.selected_objects:
            obj.scale = (1.0, 1.0, 1.0)
        self.report({'INFO'}, f"Reset scale on {len(context.selected_objects)} object(s)")
        return {'FINISHED'}
```

**panels.py:**
```python
import bpy


class TOOLBOX_PT_main(bpy.types.Panel):
    bl_label = "My Toolbox"
    bl_idname = "TOOLBOX_PT_main"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Toolbox"

    def draw(self, context):
        layout = self.layout
        props = context.scene.toolbox_props

        layout.prop(props, "scale_factor")
        layout.prop(props, "apply_to_selected")
        layout.separator()
        layout.operator("toolbox.uniform_scale")
        layout.operator("toolbox.reset_scale")
```

**__init__.py:**
```python
from . import operators, panels, properties

import bpy

classes = (
    properties.TOOLBOX_Properties,
    operators.TOOLBOX_OT_uniform_scale,
    operators.TOOLBOX_OT_reset_scale,
    panels.TOOLBOX_PT_main,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.toolbox_props = bpy.props.PointerProperty(
        type=properties.TOOLBOX_Properties
    )


def unregister():
    del bpy.types.Scene.toolbox_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 3: Legacy Multi-File Addon (Blender 3.x–4.1)

Legacy addon with `bl_info`, reload support, and AddonPreferences.

**__init__.py:**
```python
bl_info = {
    "name": "Legacy Toolbox",
    "author": "Developer",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),
    "location": "View3D > Sidebar > Legacy",
    "description": "Demonstrates legacy multi-file addon structure",
    "category": "Object",
}

# Reload support — required for F8 / addon toggle without restart
if "bpy" in locals():
    import importlib
    importlib.reload(operators)
    importlib.reload(panels)
    importlib.reload(properties)
else:
    from . import operators
    from . import panels
    from . import properties

import bpy

classes = (
    properties.LEGACY_Properties,
    operators.LEGACY_OT_action,
    panels.LEGACY_PT_panel,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.legacy_props = bpy.props.PointerProperty(
        type=properties.LEGACY_Properties
    )


def unregister():
    del bpy.types.Scene.legacy_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 4: Extension with AddonPreferences and Keymaps

Extension with user preferences, keyboard shortcuts, and writable storage.

**__init__.py:**
```python
import bpy
import os

from . import operators, panels

addon_keymaps = []


class MYEXT_AddonPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__  # MUST use __package__, NEVER hardcode

    auto_save: bpy.props.BoolProperty(
        name="Auto Save Results",
        default=True,
    )
    output_dir: bpy.props.StringProperty(
        name="Output Directory",
        subtype='DIR_PATH',
        default="//output/",
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "auto_save")
        layout.prop(self, "output_dir")


classes = (
    MYEXT_AddonPreferences,
    operators.MYEXT_OT_process,
    panels.MYEXT_PT_main,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)

    # Register keymap
    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    if kc is not None:
        km = kc.keymaps.new(name="3D View", space_type='VIEW_3D')
        kmi = km.keymap_items.new("myext.process", 'P', 'PRESS', ctrl=True, shift=True)
        addon_keymaps.append((km, kmi))


def unregister():
    # Remove keymaps BEFORE unregistering classes
    for km, kmi in addon_keymaps:
        km.keymap_items.remove(kmi)
    addon_keymaps.clear()

    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

---

## Example 5: Extension with Bundled Python Wheels

Extension that bundles a third-party Python library.

**Project structure:**
```
asset_downloader/
├── blender_manifest.toml
├── __init__.py
├── downloader.py
└── wheels/
    └── requests-2.31.0-py3-none-any.whl
```

**blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "asset_downloader"
version = "1.0.0"
name = "Asset Downloader"
tagline = "Download assets from remote repositories"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Import-Export"]

wheels = ["./wheels/requests-2.31.0-py3-none-any.whl"]

[permissions]
network = "Downloads assets from configured remote server"
files = "Saves downloaded assets to project directory"

[build]
paths_exclude_pattern = [
    "__pycache__/",
    "*.pyc",
    ".git/",
    "tests/",
]
```

**downloader.py:**
```python
import bpy
import os


def download_asset(url, output_path):
    """Download a file, respecting Blender's online access setting."""
    if not bpy.app.online_access:
        raise RuntimeError("Online access is disabled by user preferences")

    import requests  # Bundled via wheels/

    response = requests.get(url, timeout=30)
    response.raise_for_status()

    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    with open(output_path, 'wb') as f:
        f.write(response.content)

    return output_path
```

---

## Example 6: CI/CD Pipeline for Extension (GitHub Actions)

Complete GitHub Actions workflow for validating, testing, and building an extension.

**.github/workflows/blender-extension.yml:**
```yaml
name: Build & Test Extension
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        blender-version: ["4.2", "4.3", "5.0"]
    steps:
      - uses: actions/checkout@v4

      - name: Install Blender
        run: |
          sudo snap install blender --channel=${{ matrix.blender-version }}/stable --classic

      - name: Validate manifest
        run: blender --command extension validate ./my_extension

      - name: Run tests
        run: |
          blender --background --python-exit-code 1 --python tests/test_addon.py

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Blender
        run: sudo snap install blender --channel=4.2/stable --classic

      - name: Build extension
        run: |
          blender --command extension build \
            --source-dir ./my_extension \
            --output-dir ./dist

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: extension-package
          path: ./dist/*.zip
```

---

## Example 7: Headless Test Script

Test script for validating addon registration and operator execution.

**tests/test_addon.py:**
```python
import sys
import bpy


def test_register():
    """Verify addon registers without errors."""
    bpy.ops.preferences.addon_enable(module="my_extension")
    assert "my_extension" in bpy.context.preferences.addons, \
        "Addon not found in preferences after enabling"
    print("PASS: register")


def test_operator_exists():
    """Verify operators are registered."""
    assert hasattr(bpy.ops.myext, "process"), \
        "Operator 'myext.process' not found"
    print("PASS: operator exists")


def test_operator_runs():
    """Verify operator executes on a test object."""
    bpy.ops.mesh.primitive_cube_add()
    bpy.context.view_layer.objects.active = bpy.context.selected_objects[0]
    result = bpy.ops.myext.process()
    assert result == {'FINISHED'}, f"Expected FINISHED, got {result}"
    print("PASS: operator runs")


def test_unregister():
    """Verify addon unregisters cleanly."""
    bpy.ops.preferences.addon_disable(module="my_extension")
    assert "my_extension" not in bpy.context.preferences.addons, \
        "Addon still present after disabling"
    print("PASS: unregister")


if __name__ == "__main__":
    failures = 0
    for test_fn in [test_register, test_operator_exists, test_operator_runs, test_unregister]:
        try:
            test_fn()
        except Exception as e:
            print(f"FAIL: {test_fn.__name__}: {e}")
            failures += 1
    if failures > 0:
        print(f"{failures} test(s) failed")
        sys.exit(1)
    print("All tests passed")
```

---

## Example 8: Migration — Legacy Addon to Extension

Converting a `bl_info` addon to the extension system.

**Before (legacy __init__.py):**
```python
bl_info = {
    "name": "My Tool",
    "author": "Dev",
    "version": (2, 1, 0),
    "blender": (3, 6, 0),
    "description": "Automates mesh cleanup",
    "category": "Mesh",
    "doc_url": "https://example.com/docs",
}

import bpy

# ... classes and register/unregister ...
```

**After — step 1: Create blender_manifest.toml:**
```toml
schema_version = "1.0.0"

id = "my_tool"
version = "2.1.0"
name = "My Tool"
tagline = "Automates mesh cleanup"
maintainer = "Dev <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
tags = ["Mesh"]
website = "https://example.com/docs"

[build]
paths_exclude_pattern = [
    "__pycache__/",
    "*.pyc",
    ".git/",
    "tests/",
]
```

**After — step 2: Remove bl_info from __init__.py:**
```python
# bl_info REMOVED — metadata is in blender_manifest.toml

from . import operators, panels, properties

import bpy

# ... classes and register/unregister remain unchanged ...
```

**After — step 3: Validate and build:**
```bash
blender --command extension validate ./my_tool
blender --command extension build --source-dir ./my_tool --output-dir ./dist
```
