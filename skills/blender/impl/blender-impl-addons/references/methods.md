# blender-impl-addons — Method Reference

Complete API signatures for addon registration, extension packaging, manifest validation, and build commands.

---

## Registration API

### bpy.utils.register_class(cls)

Registers a single class with Blender's RNA system. The class MUST subclass a `bpy.types` base class.

```python
# Signature — Blender 3.x/4.x/5.x
bpy.utils.register_class(cls: type) -> None
```

- Raises `ValueError` if the class is already registered
- Raises `TypeError` if the class is not a valid RNA subclass
- In Blender 4.0+: only `bpy.types` subclasses are accepted

### bpy.utils.unregister_class(cls)

Unregisters a previously registered class from Blender's RNA system.

```python
# Signature — Blender 3.x/4.x/5.x
bpy.utils.unregister_class(cls: type) -> None
```

- Raises `RuntimeError` if the class is not currently registered
- ALWAYS call in reverse order of registration

### bpy.utils.register_classes_factory(classes)

Returns a `(register, unregister)` function pair for batch class registration.

```python
# Signature — Blender 3.x/4.x/5.x
bpy.utils.register_classes_factory(
    classes: tuple[type, ...]
) -> tuple[callable, callable]

# Usage
classes = (MyProperties, MY_OT_action, MY_PT_panel)
register, unregister = bpy.utils.register_classes_factory(classes)
```

- `register()` registers all classes in tuple order
- `unregister()` unregisters all classes in reverse order
- Does NOT handle PointerProperty attachment — add that separately

### bpy.utils.register_submodule_factory(module_name, submodule_names)

Returns register/unregister pair that imports submodules and registers their classes.

```python
# Signature — Blender 3.x/4.x/5.x
bpy.utils.register_submodule_factory(
    module_name: str,
    submodule_names: tuple[str, ...]
) -> tuple[callable, callable]

# Usage
register, unregister = bpy.utils.register_submodule_factory(
    __name__,
    ("operators", "panels", "properties")
)
```

---

## AddonPreferences API

### bpy.types.AddonPreferences

Base class for addon preference panels accessible in Edit > Preferences > Add-ons.

```python
# Class definition — Blender 3.x/4.x/5.x
class MyPrefs(bpy.types.AddonPreferences):
    bl_idname = __package__  # MUST match addon package name — NEVER hardcode

    my_setting: bpy.props.BoolProperty(name="My Setting", default=False)

    def draw(self, context):
        self.layout.prop(self, "my_setting")
```

### Accessing Preferences at Runtime

```python
# Blender 3.x/4.x/5.x
prefs = bpy.context.preferences.addons[__package__].preferences
value = prefs.my_setting
```

- Raises `KeyError` if the addon is not enabled
- Use `__package__` (not `__name__`) in multi-file addons

---

## Extension Build Commands (Blender 4.2+)

### blender --command extension build

Builds an extension package (.zip) from source directory.

```bash
blender --command extension build \
    --source-dir ./my_extension \
    --output-dir ./dist
```

| Argument | Description |
|----------|-------------|
| `--source-dir PATH` | Source directory containing `blender_manifest.toml` |
| `--output-dir PATH` | Output directory for the built .zip file |
| `--output-filepath PATH` | Explicit output file path (overrides `--output-dir`) |
| `--split-platforms` | Build separate packages per platform (when `platforms` is set) |
| `--verbose` | Verbose build output |

### blender --command extension validate

Validates an extension manifest and structure without building.

```bash
blender --command extension validate ./my_extension
blender --command extension validate blender_manifest.toml
```

- Returns exit code 0 on success, non-zero on validation failure
- Checks: required fields, SPDX license format, version format, wheel paths

### blender --command extension install-file

Installs an extension from a local .zip file.

```bash
blender --command extension install-file ./dist/my_extension-1.0.0.zip
blender --command extension install-file ./dist/my_extension-1.0.0.zip --enable
```

| Argument | Description |
|----------|-------------|
| `--enable` | Enable the extension immediately after installation |
| `--repo` | Target repository (default: user repository) |

---

## blender_manifest.toml Schema (v1.0.0)

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `schema_version` | string | Always `"1.0.0"` |
| `id` | string | Lowercase identifier, no spaces. Must match directory name |
| `version` | string | SemVer: `"1.0.0"` |
| `name` | string | Human-readable display name |
| `tagline` | string | One-line description, max 64 chars, no ending punctuation |
| `maintainer` | string | `"Name <email>"` format |
| `type` | string | `"add-on"` or `"theme"` |
| `blender_version_min` | string | Minimum Blender version, at least `"4.2.0"` |
| `license` | array[string] | SPDX identifiers: `["SPDX:GPL-3.0-or-later"]` |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `blender_version_max` | string | First unsupported Blender version |
| `website` | string | URL to extension homepage or documentation |
| `copyright` | array[string] | Copyright notices: `["2024-2026 Developer Name"]` |
| `tags` | array[string] | Category tags from Blender's fixed list |
| `wheels` | array[string] | Paths to bundled Python wheel files |
| `platforms` | array[string] | Platform restrictions: `"windows-x64"`, `"macos-arm64"`, `"linux-x64"` |

### [permissions] Table

| Permission | Description |
|------------|-------------|
| `network` | Explanation string, max 64 chars, no period |
| `files` | Explanation string, max 64 chars, no period |
| `clipboard` | Explanation string, max 64 chars, no period |

### [build] Table

```toml
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

---

## Keymap Registration API

### bpy.context.window_manager.keyconfigs.addon

Returns the addon keyconfig. Returns `None` in background mode.

```python
# Blender 3.x/4.x/5.x
wm = bpy.context.window_manager
kc = wm.keyconfigs.addon
if kc is None:
    return  # Background mode — no keyconfigs available
```

### KeyMap.keymap_items.new()

```python
# Signature — Blender 3.x/4.x/5.x
km.keymap_items.new(
    idname: str,          # Operator bl_idname
    type: str,            # Key: 'A', 'B', ..., 'F1', 'SPACE', etc.
    value: str,           # 'PRESS', 'RELEASE', 'CLICK', 'DOUBLE_CLICK'
    ctrl: bool = False,
    shift: bool = False,
    alt: bool = False,
    oskey: bool = False,
) -> KeyMapItem
```

---

## Headless Testing API

### Running Tests via Command Line

```bash
# Basic test execution
blender --background --python-exit-code 1 --python tests/test_addon.py

# With specific blend file
blender --background test_scene.blend --python-exit-code 1 --python tests/test_addon.py

# Enable addon before test
blender --background --python-expr \
    "import bpy; bpy.ops.preferences.addon_enable(module='my_extension')" \
    --python tests/test_addon.py
```

### bpy.ops.preferences.addon_enable()

```python
# Signature — Blender 3.x/4.x/5.x
bpy.ops.preferences.addon_enable(module: str) -> set[str]
```

- `module`: Addon module name (matches package directory name)
- Returns `{'FINISHED'}` on success
- Raises `RuntimeError` if addon cannot be found or loaded

### bpy.ops.preferences.addon_disable()

```python
# Signature — Blender 3.x/4.x/5.x
bpy.ops.preferences.addon_disable(module: str) -> set[str]
```

---

## Wheel Bundling (pip download)

```bash
# Platform-independent wheel
pip download PACKAGE==VERSION \
    --dest ./my_extension/wheels/ \
    --only-binary=:all: \
    --python-version 3.11 \
    --no-deps

# Platform-specific wheels
pip download PACKAGE==VERSION \
    --dest ./wheels_win/ \
    --only-binary=:all: \
    --platform win_amd64 \
    --python-version 3.11 \
    --no-deps

pip download PACKAGE==VERSION \
    --dest ./wheels_linux/ \
    --only-binary=:all: \
    --platform manylinux2014_x86_64 \
    --python-version 3.11 \
    --no-deps

pip download PACKAGE==VERSION \
    --dest ./wheels_mac/ \
    --only-binary=:all: \
    --platform macosx_11_0_arm64 \
    --python-version 3.11 \
    --no-deps
```

- Blender 4.2 uses Python 3.11; Blender 4.3+/5.x may use Python 3.12+
- Match `--python-version` to the target Blender's Python version
- Use `--no-deps` and download each dependency separately for full control

---

## Extension User-Writable Storage (Blender 4.2+)

### bpy.utils.extension_path_user()

```python
# Signature — Blender 4.2+
bpy.utils.extension_path_user(
    package: str,
    path: str = "",
    create: bool = False
) -> str
```

- `package`: Use `__package__`
- `path`: Optional subdirectory within the user path
- `create`: If `True`, creates the directory if it does not exist
- Returns absolute path to the writable user directory for the extension

```python
# Usage
import os
user_dir = bpy.utils.extension_path_user(__package__, create=True)
config_path = os.path.join(user_dir, "config.json")
```

---

## Network Access Guard (Blender 4.2+)

### bpy.app.online_access

```python
# Blender 4.2+ — Read-only boolean
if bpy.app.online_access:
    # User has enabled online access in preferences
    import urllib.request
    data = urllib.request.urlopen(url).read()
else:
    raise RuntimeError("Online access is disabled by user preferences")
```

- Returns `True` if the user has enabled online access
- Extensions declaring `network` permission MUST check this before any network request
- Accessing the network without checking causes rejection on extensions.blender.org
