# blender-impl-addons — Anti-Patterns

Common mistakes in Blender addon and extension development. Each entry includes the incorrect pattern, the correct alternative, and an explanation of WHY it fails.

---

## Anti-Pattern 1: Using bl_info in Extensions (Blender 4.2+)

**WRONG:**
```python
# __init__.py for a Blender 4.2+ extension
bl_info = {
    "name": "My Extension",
    "version": (1, 0, 0),
    "blender": (4, 2, 0),
    "category": "Object",
}
```

**CORRECT:**
```toml
# blender_manifest.toml
schema_version = "1.0.0"
id = "my_extension"
version = "1.0.0"
name = "My Extension"
tagline = "Short description without ending punctuation"
maintainer = "Developer <dev@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = ["SPDX:GPL-3.0-or-later"]
```

**WHY:** Blender 4.2+ extensions use `blender_manifest.toml` for metadata. The `bl_info` dict is ignored by the extension system and causes validation failures on extensions.blender.org. The extension build command reads only from the manifest file.

---

## Anti-Pattern 2: Hardcoding bl_idname in AddonPreferences

**WRONG:**
```python
class MyPrefs(bpy.types.AddonPreferences):
    bl_idname = "my_addon"  # Hardcoded string
```

**CORRECT:**
```python
class MyPrefs(bpy.types.AddonPreferences):
    bl_idname = __package__  # Dynamically matches addon package name
```

**WHY:** The hardcoded string breaks when the addon directory is renamed, when installed as an extension (where the package name includes a repository prefix), or during development when the module name differs. Using `__package__` guarantees the identifier always matches the actual installed package name.

---

## Anti-Pattern 3: Manipulating sys.path for Dependencies

**WRONG:**
```python
import sys
import os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "libs"))
import some_library
```

**CORRECT:**
```toml
# blender_manifest.toml
wheels = ["./wheels/some_library-1.0.0-py3-none-any.whl"]
```
```python
# __init__.py — just import directly
import some_library
```

**WHY:** Manual `sys.path` manipulation pollutes the global Python path, causes conflicts between addons bundling different versions of the same library, and is explicitly rejected on extensions.blender.org. The `wheels` array in the manifest provides isolated, per-extension dependency resolution.

---

## Anti-Pattern 4: Empty Manifest Fields

**WRONG:**
```toml
# blender_manifest.toml
tags = []
website = ""
copyright = []
```

**CORRECT:**
```toml
# blender_manifest.toml — omit optional fields entirely
# Do NOT include tags, website, or copyright if not needed
```

**WHY:** The manifest schema treats empty arrays and empty strings as validation errors. Optional fields must either contain valid data or be omitted entirely. The `blender --command extension validate` command rejects manifests with empty optional fields.

---

## Anti-Pattern 5: Wrong Registration Order

**WRONG:**
```python
classes = (
    MY_PT_panel,          # Panel registered first
    MY_OT_operator,       # Operator second
    MyPropertyGroup,      # PropertyGroup last
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
```

**CORRECT:**
```python
classes = (
    MyPropertyGroup,      # PropertyGroups FIRST
    MY_OT_operator,       # Operators second
    MY_PT_panel,          # Panels/UI LAST
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)

def unregister():
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

**WHY:** Panels and operators that reference PropertyGroups via `PointerProperty` will fail to register if the PropertyGroup is not yet registered. Blender resolves type references at registration time. Unregistration must happen in reverse order to avoid dangling references.

---

## Anti-Pattern 6: Missing unregister() or Incomplete Cleanup

**WRONG:**
```python
def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
    bpy.app.handlers.load_post.append(on_load)

def unregister():
    for cls in classes:  # Not reversed, and missing property/handler cleanup
        bpy.utils.unregister_class(cls)
```

**CORRECT:**
```python
def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
    bpy.app.handlers.load_post.append(on_load)

def unregister():
    bpy.app.handlers.load_post.remove(on_load)
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

**WHY:** Incomplete cleanup causes errors when toggling the addon on/off, memory leaks from orphaned handlers, and duplicate handler registrations on re-enable. Properties must be deleted before their types are unregistered, and handlers must be removed to prevent callbacks on stale references.

---

## Anti-Pattern 7: Using bpy.utils.register_module()

**WRONG:**
```python
def register():
    bpy.utils.register_module(__name__)

def unregister():
    bpy.utils.unregister_module(__name__)
```

**CORRECT:**
```python
classes = (MyPropertyGroup, MY_OT_operator, MY_PT_panel)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

**WHY:** `bpy.utils.register_module()` was removed in Blender 2.80. It automatically discovered and registered all classes in a module, but this made registration order unpredictable. Explicit class registration with a defined tuple ensures correct dependency ordering.

---

## Anti-Pattern 8: Installing pip Packages at Runtime

**WRONG:**
```python
import subprocess
import sys

def ensure_dependencies():
    try:
        import requests
    except ImportError:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "requests"])
        import requests
```

**CORRECT:**
```toml
# blender_manifest.toml
wheels = ["./wheels/requests-2.31.0-py3-none-any.whl"]
```
```python
# Just import — wheel is available at runtime
import requests
```

**WHY:** Runtime pip installation modifies Blender's Python environment globally, causes version conflicts between addons, requires internet access that users may not have, and is explicitly forbidden for extensions on extensions.blender.org. Bundled wheels provide predictable, isolated dependencies.

---

## Anti-Pattern 9: Writing to the Addon Installation Directory

**WRONG:**
```python
import os
config_dir = os.path.dirname(__file__)
config_path = os.path.join(config_dir, "config.json")
with open(config_path, 'w') as f:
    json.dump(settings, f)
```

**CORRECT:**
```python
import os
import bpy

# Blender 4.2+ — use extension user directory
user_dir = bpy.utils.extension_path_user(__package__, create=True)
config_path = os.path.join(user_dir, "config.json")
with open(config_path, 'w') as f:
    json.dump(settings, f)
```

**WHY:** The addon installation directory may be read-only (system-wide installs, managed repositories), and writing there corrupts the extension package checksum. Extensions on extensions.blender.org are verified against their package contents. `bpy.utils.extension_path_user()` provides a writable, per-user directory that survives extension updates.

---

## Anti-Pattern 10: Not Checking keyconfigs.addon for None

**WRONG:**
```python
def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    km = kc.keymaps.new(name="3D View", space_type='VIEW_3D')  # Crashes in background mode
```

**CORRECT:**
```python
def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    if kc is not None:  # None in background mode
        km = kc.keymaps.new(name="3D View", space_type='VIEW_3D')
        kmi = km.keymap_items.new("myext.action", 'A', 'PRESS', ctrl=True)
        addon_keymaps.append((km, kmi))
```

**WHY:** `wm.keyconfigs.addon` returns `None` when Blender runs in background mode (`blender --background`). Attempting to call `.keymaps.new()` on `None` raises `AttributeError`, which prevents the addon from registering during headless testing and CI/CD pipelines.

---

## Anti-Pattern 11: Using Absolute Imports in Multi-File Addons

**WRONG:**
```python
# operators.py
from my_addon.properties import MyPropertyGroup
from my_addon import utils
```

**CORRECT:**
```python
# operators.py
from .properties import MyPropertyGroup
from . import utils
```

**WHY:** Absolute imports break when the addon directory is renamed, when the addon is installed as an extension (where the package is nested under a repository namespace), or when two addons have similarly named modules. Relative imports always resolve within the addon's own package hierarchy.

---

## Anti-Pattern 12: Skipping Network Access Check in Extensions

**WRONG:**
```python
import urllib.request

def fetch_data(url):
    return urllib.request.urlopen(url).read()
```

**CORRECT:**
```python
import bpy

def fetch_data(url):
    if not bpy.app.online_access:
        raise RuntimeError("Online access is disabled by user preferences")
    import urllib.request
    return urllib.request.urlopen(url).read()
```

**WHY:** Extensions that declare `network` permission MUST check `bpy.app.online_access` before making any network request. This flag reflects the user's explicit choice in Blender preferences. Ignoring it violates user trust and causes rejection on extensions.blender.org during review.
