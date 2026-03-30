---
name: blender-impl-addons
description: >
  Use when building a production Blender addon -- project structure, testing, CI/CD, or
  packaging for extensions.blender.org. Prevents the common mistake of hardcoding paths
  or skipping the extension manifest for Blender 4.2+. Covers multi-file addon structure,
  testing strategies, dependency management, and distribution workflows.
  Keywords: addon development, extension packaging, CI/CD, testing, multi-file addon,
  blender_manifest.toml, distribution, extensions.blender.org.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 4.2+ for extensions, 3.x+ for legacy addons."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-impl-addons

## Quick Reference

### Decision Tree: Extension vs Legacy Addon

```
Target Blender version?
├── 4.2+ ──▶ Use Extension System (blender_manifest.toml)
│   ├── Distribute via extensions.blender.org? ──▶ YES: Follow hosted extension rules
│   └── Self-distribute? ──▶ Build .zip with `blender --command extension build`
├── 3.x–4.1 ──▶ Use Legacy Addon (bl_info dict)
└── Support both? ──▶ Use Extension System, set blender_version_min = "4.2.0"
    Legacy users must install manually
```

### Decision Tree: Single-File vs Multi-File Addon

```
Addon complexity?
├── < 200 lines, no external deps ──▶ Single-file addon (__init__.py only)
├── Multiple operators/panels ──▶ Multi-file addon (package directory)
├── External Python deps ──▶ Multi-file extension with wheels/ directory
└── Platform-specific binaries ──▶ Multi-platform extension with platforms field
```

### Critical Warnings

**NEVER** use `bl_info` in extensions targeting Blender 4.2+ — use `blender_manifest.toml` instead.

**NEVER** leave `blender_manifest.toml` fields empty — omit optional fields entirely rather than setting them to `""` or `[]`.

**NEVER** manipulate `sys.path` manually in extensions — use `wheels` array in manifest for dependencies.

**NEVER** use `bpy.utils.register_module(__name__)` — removed in Blender 2.80. Register classes individually.

**ALWAYS** test extensions headless: `blender --background --python-exit-code 1 --python test_script.py`.

**ALWAYS** implement both `register()` and `unregister()` — unregister MUST reverse ALL registration in reverse order.

**ALWAYS** use `__package__` (not `__name__`) for `AddonPreferences.bl_idname` in multi-file addons.

---

## Essential Patterns

### Pattern 1: Extension Project Structure (Blender 4.2+)

```
my_extension/
├── blender_manifest.toml    # REQUIRED: replaces bl_info
├── __init__.py              # register() / unregister() — NO bl_info
├── operators.py             # Operator classes
├── panels.py                # Panel/Menu/Header classes
├── properties.py            # PropertyGroup classes
├── preferences.py           # AddonPreferences class
├── utils.py                 # Internal helpers
└── wheels/                  # Optional: bundled Python packages
    └── some_lib-1.0-py3-none-any.whl
```

### Pattern 2: blender_manifest.toml (Complete Template)

```toml
schema_version = "1.0.0"

# REQUIRED fields
id = "my_extension_name"
version = "1.0.0"
name = "My Extension Name"
tagline = "One-line description without ending punctuation"
maintainer = "Developer Name <email@example.com>"
type = "add-on"
blender_version_min = "4.2.0"
license = [
    "SPDX:GPL-3.0-or-later",
]

# OPTIONAL fields: omit if not needed, NEVER leave empty
# blender_version_max = "5.1.0"
# website = "https://extensions.blender.org/add-ons/my-extension/"
# copyright = ["2024-2026 Developer Name"]
# tags = ["Object", "Import-Export"]

# OPTIONAL: Python wheel dependencies
# wheels = ["./wheels/requests-2.31.0-py3-none-any.whl"]

# OPTIONAL: platform restrictions (omit for all-platform support)
# platforms = ["windows-x64", "macos-arm64", "linux-x64"]

# OPTIONAL: permission declarations (each with explanation, max 64 chars, no period)
# [permissions]
# network = "Downloads assets from remote server"
# files = "Reads and writes project files"
# clipboard = "Copies generated code to clipboard"

# Build configuration
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

### Pattern 3: Extension __init__.py (Blender 4.2+)

```python
# NO bl_info dict: metadata lives in blender_manifest.toml

# Reload support for development
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
    properties.MyProperties,
    operators.MY_OT_example,
    panels.MY_PT_main_panel,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_ext = bpy.props.PointerProperty(type=properties.MyProperties)


def unregister():
    del bpy.types.Scene.my_ext
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

### Pattern 4: Legacy Multi-File Addon (Blender 3.x–4.1)

```python
# __init__.py: legacy addon with bl_info
bl_info = {
    "name": "My Addon",
    "author": "Developer Name",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),
    "location": "View3D > Sidebar > My Tab",
    "description": "Short description of functionality",
    "warning": "",
    "doc_url": "https://example.com/docs",
    "category": "Object",
}

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
    properties.MyProperties,
    operators.MY_OT_example,
    panels.MY_PT_main_panel,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=properties.MyProperties)


def unregister():
    del bpy.types.Scene.my_props
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)
```

### Pattern 5: Class Naming Convention (REQUIRED: All Versions)

Blender enforces: `{ADDON}_{TYPE}_{name}` with uppercase prefix and type code.

| Type Code | Base Class | Example |
|-----------|-----------|---------|
| `OT` | `bpy.types.Operator` | `MYEXT_OT_do_something` |
| `PT` | `bpy.types.Panel` | `MYEXT_PT_main_panel` |
| `MT` | `bpy.types.Menu` | `MYEXT_MT_context_menu` |
| `UL` | `bpy.types.UIList` | `MYEXT_UL_item_list` |
| `HT` | `bpy.types.Header` | `MYEXT_HT_header` |
| `KI` | `bpy.types.KeyingSetInfo` | `MYEXT_KI_keying_set` |

### Pattern 6: Registration Order

Register in dependency order: PropertyGroups first, then Operators, then Panels/Menus. Unregister in reverse order.

```python
classes = (
    # 1. PropertyGroups FIRST (operators and panels depend on them)
    MyProperties,
    MyItemProperties,
    # 2. Operators
    MYEXT_OT_add_item,
    MYEXT_OT_remove_item,
    # 3. Panels/Menus/UILists LAST (depend on operators and properties)
    MYEXT_UL_item_list,
    MYEXT_PT_main_panel,
    MYEXT_MT_context_menu,
)
```

---

## Common Operations

### Building and Installing Extensions

```bash
# Build extension package (creates .zip in current directory)
blender --command extension build --source-dir ./my_extension --output-dir ./dist

# Validate manifest without building
blender --command extension validate ./my_extension

# Install extension from file (for testing)
blender --command extension install-file ./dist/my_extension-1.0.0.zip

# Install from local repo for development
blender --command extension install-file ./dist/my_extension-1.0.0.zip --enable
```

### Testing Addon Headless

```bash
# Run test script with exit code propagation (for CI)
blender --background --python-exit-code 1 --python tests/test_addon.py

# Run with specific blend file
blender --background test_scene.blend --python-exit-code 1 --python tests/test_addon.py

# Run with addon enabled
blender --background --python-expr "import bpy; bpy.ops.preferences.addon_enable(module='my_extension')" --python tests/test_addon.py
```

### Headless Test Script Pattern

```python
# tests/test_addon.py: runs via blender --background --python
import sys
import bpy

def test_register():
    """Verify addon registers without errors."""
    # For extensions, enable via preferences
    bpy.ops.preferences.addon_enable(module="my_extension")
    assert "my_extension" in bpy.context.preferences.addons
    print("PASS: register")

def test_operator_exists():
    """Verify operators are registered."""
    assert hasattr(bpy.ops.myext, "do_something")
    print("PASS: operator exists")

def test_operator_runs():
    """Verify operator executes successfully."""
    bpy.ops.mesh.primitive_cube_add()
    result = bpy.ops.myext.do_something()
    assert result == {'FINISHED'}
    print("PASS: operator runs")

if __name__ == "__main__":
    failures = 0
    for test_fn in [test_register, test_operator_exists, test_operator_runs]:
        try:
            test_fn()
        except Exception as e:
            print(f"FAIL: {test_fn.__name__}: {e}")
            failures += 1
    if failures > 0:
        sys.exit(1)
    print(f"All tests passed")
```

### Bundling Python Wheels

```bash
# Download platform-independent wheels
pip download some_package==1.0.0 --dest ./my_extension/wheels/ \
    --only-binary=:all: --python-version 3.11 --no-deps

# For platform-specific wheels, build per-platform
pip download some_package==1.0.0 --dest ./wheels_win/ \
    --only-binary=:all: --platform win_amd64 --python-version 3.11 --no-deps
pip download some_package==1.0.0 --dest ./wheels_linux/ \
    --only-binary=:all: --platform manylinux2014_x86_64 --python-version 3.11 --no-deps
```

Then reference in `blender_manifest.toml`:
```toml
wheels = [
    "./wheels/some_package-1.0.0-py3-none-any.whl",
]
```

### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/blender-extension.yml
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

### AddonPreferences

```python
class MYEXT_AddonPreferences(bpy.types.AddonPreferences):
    bl_idname = __package__  # MUST match package name

    api_key: bpy.props.StringProperty(
        name="API Key",
        subtype='PASSWORD',
    )
    debug_mode: bpy.props.BoolProperty(
        name="Debug Mode",
        default=False,
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "api_key")
        layout.prop(self, "debug_mode")

# Access preferences at runtime:
# prefs = bpy.context.preferences.addons[__package__].preferences
# value = prefs.debug_mode
```

### Network Access (Extension Requirement)

Extensions declaring `network` permission MUST check `bpy.app.online_access` before connecting:

```python
import bpy

def fetch_remote_data(url):
    if not bpy.app.online_access:
        raise RuntimeError("Online access is disabled by user preferences")
    import urllib.request
    return urllib.request.urlopen(url).read()
```

### Addon Keymaps

```python
addon_keymaps = []

def register_keymaps():
    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    if kc is None:
        return  # Background mode — no keyconfigs available
    km = kc.keymaps.new(name="3D View", space_type='VIEW_3D')
    kmi = km.keymap_items.new("myext.do_something", 'D', 'PRESS', ctrl=True)
    addon_keymaps.append((km, kmi))

def unregister_keymaps():
    for km, kmi in addon_keymaps:
        km.keymap_items.remove(kmi)
    addon_keymaps.clear()
```

**ALWAYS** check `kc is not None` — `keyconfigs.addon` returns `None` in background mode.

---

## Publishing to extensions.blender.org

### Submission Checklist

1. Extension MUST be functional — test with `blender --background`
2. `blender_manifest.toml` MUST pass validation: `blender --command extension validate`
3. All permissions MUST be declared with explanations (max 64 chars, no period)
4. Python dependencies MUST be bundled as wheels — no runtime pip installs
5. Network access MUST check `bpy.app.online_access` before connecting
6. Extension MUST NOT contain auto-updater code
7. Extension MUST be self-contained — no cross-extension dependencies
8. License MUST use SPDX identifiers: `"SPDX:GPL-3.0-or-later"`
9. Run linter: `find . -iname "*.py" | xargs ruff check`
10. Use `bpy.utils.extension_path_user()` for writing to valid file locations

### Review Rejection Reasons

- Hardcoded datablock names (`bpy.data.objects["Cube"]`)
- Backslash path separators outside escape sequences
- Manual `sys.path` manipulation
- Empty manifest fields (omit optional fields instead)
- Using `__file__` for import path construction
- Missing `bpy.utils.escape_identifier()` for bone name data-paths
- Unused imports or syntax issues (run `ruff check`)

---

## Version Migration: Legacy Addon to Extension

### Step-by-Step Conversion

1. Create `blender_manifest.toml` with metadata from `bl_info`
2. Remove `bl_info` dict from `__init__.py`
3. Keep `register()` / `unregister()` unchanged
4. Move any pip dependencies to `wheels/` directory
5. Add `[build]` section with exclude patterns
6. Test: `blender --command extension validate ./my_extension`
7. Build: `blender --command extension build --source-dir ./my_extension`

### bl_info to Manifest Field Mapping

| bl_info Key | Manifest Field | Notes |
|-------------|---------------|-------|
| `"name"` | `name` | Direct mapping |
| `"author"` | `maintainer` | Add email: `"Name <email>"` |
| `"version"` | `version` | Tuple `(1, 0, 0)` → string `"1.0.0"` |
| `"blender"` | `blender_version_min` | Tuple → string, minimum `"4.2.0"` |
| `"description"` | `tagline` | Max 64 chars, no ending punctuation |
| `"doc_url"` | `website` | Direct mapping |
| `"category"` | `tags` | Array format: `["Object"]` |
| `"warning"` | — | No equivalent; remove |

---

## Reference Links
- [references/methods.md](references/methods.md) -- Registration API, manifest schema, build commands
- [references/examples.md](references/examples.md) -- Addon and extension examples
- [references/anti-patterns.md](references/anti-patterns.md) -- Common addon development mistakes
- Official: [Extension Schema](https://developer.blender.org/docs/features/extensions/schema/1.0.0/), [Guidelines](https://developer.blender.org/docs/features/extensions/moderation/guidelines/), [API Overview](https://docs.blender.org/api/current/info_overview.html)
