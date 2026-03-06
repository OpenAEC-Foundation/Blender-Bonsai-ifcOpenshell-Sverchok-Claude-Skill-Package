---
name: blender-syntax-addons
description: "Defines Blender addon and extension development patterns including legacy bl_info dict, blender_manifest.toml (4.2+), register/unregister lifecycle, multi-file addon structure, AddonPreferences, class naming conventions, and extension packaging for extensions.blender.org. Activates when creating Blender addons, packaging extensions, or migrating from legacy bl_info to manifest format."
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-addons

## Quick Reference

### Version Decision Tree

```
Creating a Blender addon?
│
├── Target Blender ≥ 4.2?
│   ├── YES → Use blender_manifest.toml (extension system)
│   │         NO bl_info dict. NO __init__.py bl_info.
│   │         See: §Extension System
│   │
│   └── Also support < 4.2?
│       └── YES → Provide BOTH bl_info AND blender_manifest.toml
│                 Blender picks the correct one automatically
│
└── Target Blender < 4.2 only?
    └── Use bl_info dict in __init__.py
        See: §Legacy bl_info
```

### Critical Warnings

**NEVER** omit `register()` and `unregister()` functions — Blender REQUIRES both in every addon.

**NEVER** set `bl_idname` on `AddonPreferences` to a hardcoded string — ALWAYS use `__package__`.

**NEVER** register classes in wrong order — PropertyGroups and dependencies MUST register before classes that reference them.

**NEVER** write to the addon's own installation directory — use `bpy.utils.extension_path_user(__package__, create=True)` for writable storage (4.2+).

**NEVER** install pip packages at runtime — bundle wheels in `wheels/` directory instead (4.2+).

**NEVER** use `from my_addon import X` at module level in sub-modules — use `from . import X` (relative imports) for multi-file addons.

**ALWAYS** use the class naming convention `{ADDON}_{TYPE}_{name}` — Blender enforces this pattern for all registered types.

**ALWAYS** unregister classes in reverse order of registration.

---

## Essential Patterns

### Pattern 1: Legacy bl_info Dict (Blender < 4.2)

The `bl_info` dict MUST be at module-level in `__init__.py`. Blender parses it statically (no Python execution), so it MUST NOT use variables or expressions.

```python
# Blender 3.x/4.0/4.1 — REQUIRED in __init__.py (or single-file addon)
bl_info = {
    "name": "My Addon",
    "author": "Developer Name",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),          # Minimum Blender version
    "location": "View3D > Sidebar > My Tab",
    "description": "Short description of addon purpose",
    "warning": "",                  # Optional: shown as warning in prefs
    "doc_url": "https://example.com/docs",
    "tracker_url": "https://example.com/issues",
    "category": "Object",          # Must be a valid Blender category
}
```

Valid `category` values: `3D View`, `Add Mesh`, `Add Curve`, `Animation`, `Compositing`, `Development`, `Game Engine`, `Import-Export`, `Lighting`, `Material`, `Mesh`, `Node`, `Object`, `Paint`, `Physics`, `Render`, `Rigging`, `Scene`, `Sequencer`, `System`, `Text Editor`, `UV`.

### Pattern 2: Extension Manifest (Blender 4.2+)

Extensions use `blender_manifest.toml` instead of `bl_info`. The `__init__.py` contains ONLY `register()`/`unregister()` — no `bl_info` dict.

```toml
# blender_manifest.toml — REQUIRED for extensions (Blender 4.2+)
schema_version = "1.0.0"

id = "my_addon"
version = "1.0.0"
name = "My Addon"
tagline = "Short description without ending punctuation"
maintainer = "Developer Name <dev@example.com>"
type = "add-on"

# Minimum Blender version (must be >= 4.2.0 for extensions)
blender_version_min = "4.2.0"
# blender_version_max = "5.1.0"  # Optional: first UNSUPPORTED version

license = ["SPDX:GPL-3.0-or-later"]
# copyright = ["2024 Developer Name"]  # Optional

# Optional: category tags (from Blender's fixed tag list)
tags = ["Object", "Scene"]

# Optional: bundled Python wheels
# wheels = ["./wheels/some_lib-1.0.0-py3-none-any.whl"]

# Optional: platform restrictions
# platforms = ["windows-x64", "macos-arm64", "linux-x64"]

# Optional: permission declarations (required for network/file access)
# [permissions]
# network = "Downloads data from remote server"
# files = "Reads reference files from disk"
# clipboard = "Copies results to clipboard"
```

### Pattern 3: Register / Unregister Lifecycle

```python
# Blender 3.x/4.x/5.x — REQUIRED functions
import bpy

classes = (
    MyPropertyGroup,     # PropertyGroups FIRST
    MY_OT_operator,      # Then operators
    MY_PT_panel,         # Then UI elements
    MyAddonPreferences,  # Preferences can go anywhere
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    # Attach properties AFTER registering their types
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)

def unregister():
    # Remove properties BEFORE unregistering their types
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

Registration order rules:
1. `PropertyGroup` classes BEFORE any class that references them via `PointerProperty`
2. Parent panels BEFORE sub-panels (if using `bl_parent_id`)
3. `unregister()` MUST reverse the order
4. Property attachments (`bpy.types.Scene.my_prop = ...`) AFTER `register_class()`
5. Property deletions (`del bpy.types.Scene.my_prop`) BEFORE `unregister_class()`

### Pattern 4: Class Naming Convention (Enforced)

Blender enforces: `{ADDON}_{TYPE}_{name}` where `{ADDON}` is uppercase addon identifier, `{TYPE}` is a two-letter type code, and `{name}` is lowercase with underscores.

| Type Code | Base Class | Example |
|-----------|-----------|---------|
| `OT` | `bpy.types.Operator` | `MYADDON_OT_do_something` |
| `PT` | `bpy.types.Panel` | `MYADDON_PT_main_panel` |
| `MT` | `bpy.types.Menu` | `MYADDON_MT_context_menu` |
| `UL` | `bpy.types.UIList` | `MYADDON_UL_item_list` |
| `HT` | `bpy.types.Header` | `MYADDON_HT_header` |
| `KI` | `bpy.types.KeyingSetInfo` | `MYADDON_KI_keying` |

For `bl_idname` on operators: use `"addonname.action_name"` (all lowercase, dot-separated). Example: `bl_idname = "myaddon.export_data"`.

### Pattern 5: Multi-File Addon Structure

```
my_addon/
├── __init__.py          # bl_info (or none for 4.2+) + register/unregister
├── blender_manifest.toml  # 4.2+ only
├── operators.py         # Operator classes
├── panels.py            # Panel classes
├── properties.py        # PropertyGroup classes
├── preferences.py       # AddonPreferences class
├── utils.py             # Helper functions (no bpy.types subclasses)
└── wheels/              # Bundled pip packages (4.2+ only)
    └── some_lib.whl
```

**`__init__.py`** with reload support (legacy addon):

```python
bl_info = {
    "name": "My Addon",
    "blender": (3, 6, 0),
    "category": "Object",
    "version": (1, 0, 0),
}

# Reload support — REQUIRED for F8 / addon toggle without restart
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
    properties.MyPropertyGroup,
    operators.MYADDON_OT_example,
    panels.MYADDON_PT_main,
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(
        type=properties.MyPropertyGroup
    )

def unregister():
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

**`__init__.py`** for extension (4.2+) — NO bl_info, NO reload boilerplate:

```python
from . import operators, panels, properties

import bpy

classes = (
    properties.MyPropertyGroup,
    operators.MYADDON_OT_example,
    panels.MYADDON_PT_main,
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(
        type=properties.MyPropertyGroup
    )

def unregister():
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

### Pattern 6: AddonPreferences

```python
# Blender 3.x/4.x/5.x
class MyAddonPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__  # MUST match addon module name — NEVER hardcode

    api_key: bpy.props.StringProperty(
        name="API Key",
        subtype='PASSWORD',
    )
    debug_mode: bpy.props.BoolProperty(
        name="Debug Mode",
        default=False,
    )
    output_path: bpy.props.StringProperty(
        name="Output Path",
        subtype='DIR_PATH',
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "api_key")
        layout.prop(self, "debug_mode")
        layout.prop(self, "output_path")
```

Access preferences at runtime:

```python
# Blender 3.x/4.x/5.x
prefs = bpy.context.preferences.addons[__package__].preferences
value = prefs.debug_mode
```

### Pattern 7: Extension Packaging (4.2+)

Build and validate an extension for distribution:

```bash
# Validate manifest
blender --command extension validate blender_manifest.toml

# Build .zip for upload to extensions.blender.org
blender --command extension build --source-dir ./my_addon --output-filepath my_addon.zip

# Install locally for testing
blender --command extension install-file my_addon.zip
```

Extension guidelines for extensions.blender.org:
- MUST check `bpy.app.online_access` before any network request
- MUST NOT install pip packages at runtime — bundle in `wheels/`
- MUST NOT modify other addons or extensions
- MUST NOT write to the addon installation directory
- MUST use relative imports only — no `sys.path` manipulation
- MUST load all modules as sub-modules within package namespace
- SHOULD follow PEP 8

### Pattern 8: User-Writable Storage (4.2+)

```python
# Blender 4.2+ — get a writable directory for addon data
import os
user_dir = bpy.utils.extension_path_user(__package__, create=True)
config_path = os.path.join(user_dir, "config.json")
```

---

## Common Operations

### Detect Addon vs Extension Context

```python
# Blender 3.x/4.x/5.x — check if running as extension
import os

def is_extension():
    """Check if this addon is installed as a Blender 4.2+ extension."""
    manifest = os.path.join(os.path.dirname(__file__), "blender_manifest.toml")
    return os.path.exists(manifest)
```

### Version-Conditional Registration

```python
# Support both legacy and extension systems
def register():
    for cls in classes:
        bpy.utils.register_class(cls)

    # Handler registration
    bpy.app.handlers.load_post.append(on_load)

def unregister():
    bpy.app.handlers.load_post.remove(on_load)
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)

# For both: if __name__ == "__main__" block for script testing
if __name__ == "__main__":
    register()
```

### Keymap Registration in Addon

```python
addon_keymaps = []

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

    # Add keymap entry
    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    if kc:
        km = kc.keymaps.new(name='3D View', space_type='VIEW_3D')
        kmi = km.keymap_items.new("myaddon.do_something", 'D', 'PRESS', ctrl=True)
        addon_keymaps.append((km, kmi))

def unregister():
    # Remove keymaps BEFORE unregistering classes
    for km, kmi in addon_keymaps:
        km.keymap_items.remove(kmi)
    addon_keymaps.clear()

    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

### Migration: bl_info to blender_manifest.toml

| bl_info field | Manifest equivalent | Notes |
|---------------|-------------------|-------|
| `"name"` | `name` | Same |
| `"author"` | `maintainer` | Add email: `"Name <email>"` |
| `"version"` | `version` | Tuple → string: `(1,0,0)` → `"1.0.0"` |
| `"blender"` | `blender_version_min` | Tuple → string, min `"4.2.0"` |
| `"description"` | `tagline` | No ending punctuation |
| `"category"` | `tags` | String → array: `"Object"` → `["Object"]` |
| `"doc_url"` | `website` | Same |
| `"warning"` | *(removed)* | No equivalent in manifest |
| `"tracker_url"` | *(removed)* | No equivalent in manifest |
| *(new)* | `id` | Lowercase identifier, no spaces |
| *(new)* | `schema_version` | Always `"1.0.0"` |
| *(new)* | `license` | SPDX format: `["SPDX:GPL-3.0-or-later"]` |
| *(new)* | `type` | `"add-on"` or `"theme"` |
| *(new)* | `maintainer` | `"Name <email>"` format |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for addon registration, preferences, and extension utilities
- [references/examples.md](references/examples.md) — Working code examples for single-file, multi-file, and extension addons
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do when building Blender addons

### Official Sources

- https://docs.blender.org/api/current/info_overview.html
- https://docs.blender.org/manual/en/latest/advanced/scripting/addon_tutorial.html
- https://developer.blender.org/docs/handbook/extensions/
- https://developer.blender.org/docs/handbook/extensions/addon_guidelines/
- https://developer.blender.org/docs/features/extensions/schema/1.0.0/
