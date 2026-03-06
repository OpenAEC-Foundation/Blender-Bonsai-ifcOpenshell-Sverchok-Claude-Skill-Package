# blender-syntax-addons — Method Reference

## Registration API

### bpy.utils.register_class(cls)

Registers a class with Blender's RNA system. The class MUST be a subclass of a `bpy.types` base class.

```python
# Signature
bpy.utils.register_class(cls: type) -> None

# Usage — Blender 3.x/4.x/5.x
bpy.utils.register_class(MY_OT_example)
```

- Raises `ValueError` if class is already registered
- Raises `TypeError` if class is not a valid RNA type
- In Blender 4.0+: ONLY `bpy.types` subclasses can be registered (non-RNA types raise error)

### bpy.utils.unregister_class(cls)

Unregisters a previously registered class.

```python
# Signature
bpy.utils.unregister_class(cls: type) -> None

# Usage — Blender 3.x/4.x/5.x
bpy.utils.unregister_class(MY_OT_example)
```

- Raises `RuntimeError` if class is not registered
- ALWAYS unregister in reverse order of registration

### bpy.utils.register_classes_factory(classes)

Returns a `(register, unregister)` function pair for a tuple of classes.

```python
# Signature — Blender 3.x/4.x/5.x
bpy.utils.register_classes_factory(classes: tuple) -> tuple[callable, callable]

# Usage
classes = (MY_OT_example, MY_PT_panel)
register, unregister = bpy.utils.register_classes_factory(classes)
# register() registers all classes in order
# unregister() unregisters all classes in reverse order
```

### bpy.utils.register_submodule_factory(module_name, submodule_names)

Returns register/unregister pair that handles submodule importing and class registration.

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
    bl_idname = __package__  # MUST match addon module name

    # Properties defined as class annotations
    prop_name: bpy.props.StringProperty(name="Label", default="value")

    def draw(self, context):
        """Draw the preferences panel. Called by Blender."""
        self.layout.prop(self, "prop_name")
```

Required attributes:
- `bl_idname` — MUST be `__package__` (the addon module name)

Optional methods:
- `draw(self, context)` — Custom UI layout for preferences panel

### Accessing Preferences at Runtime

```python
# Blender 3.x/4.x/5.x
prefs = bpy.context.preferences.addons[__package__].preferences

# Access individual properties
value = prefs.prop_name

# From outside the addon (must know module name)
prefs = bpy.context.preferences.addons["my_addon_module"].preferences
```

- Raises `KeyError` if addon is not enabled
- `bpy.context.preferences.addons` is a dict-like `bpy.types.bpy_prop_collection`

---

## Extension Utilities (Blender 4.2+)

### bpy.utils.extension_path_user(package, *, path="", create=False)

Returns a user-writable directory path for extension data storage.

```python
# Signature — Blender 4.2+
bpy.utils.extension_path_user(
    package: str,      # Typically __package__
    *,
    path: str = "",    # Optional subdirectory
    create: bool = False  # Create directory if it doesn't exist
) -> str

# Usage
user_dir = bpy.utils.extension_path_user(__package__, create=True)
cache_dir = bpy.utils.extension_path_user(__package__, path="cache", create=True)
```

- Returns path under user's Blender config directory
- ALWAYS use this instead of writing to addon installation directory
- Not available in Blender < 4.2

### bpy.app.online_access

```python
# Blender 4.2+ — read-only property
can_network = bpy.app.online_access  # bool

# MUST check before any network request in extensions
if bpy.app.online_access:
    import urllib.request
    response = urllib.request.urlopen(url)
```

---

## bl_info Dict Fields (Legacy)

Complete field reference for `bl_info` dictionary (Blender < 4.2, works through 5.x):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `"name"` | `str` | YES | Display name in preferences |
| `"author"` | `str` | YES | Author name(s) |
| `"version"` | `tuple[int, ...]` | YES | Addon version, e.g. `(1, 0, 0)` |
| `"blender"` | `tuple[int, int, int]` | YES | Minimum Blender version |
| `"location"` | `str` | NO | Where to find in UI |
| `"description"` | `str` | YES | Short description |
| `"warning"` | `str` | NO | Warning text shown in prefs |
| `"doc_url"` | `str` | NO | Documentation URL |
| `"tracker_url"` | `str` | NO | Bug tracker URL |
| `"support"` | `str` | NO | `"OFFICIAL"`, `"COMMUNITY"`, `"TESTING"` |
| `"category"` | `str` | YES | Category from fixed list |

---

## blender_manifest.toml Fields (Extension)

Complete field reference for `blender_manifest.toml` (Blender 4.2+):

### Required Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `schema_version` | String | Manifest format version | `"1.0.0"` |
| `id` | String | Unique identifier (lowercase, underscores) | `"my_addon"` |
| `version` | String | Semver version | `"1.0.0"` |
| `name` | String | Display name | `"My Addon"` |
| `tagline` | String | One-line description (no trailing period) | `"Does something useful"` |
| `maintainer` | String | Maintainer with email | `"Name <email@example.com>"` |
| `type` | String | `"add-on"` or `"theme"` | `"add-on"` |
| `blender_version_min` | String | Minimum version (≥ `"4.2.0"`) | `"4.2.0"` |
| `license` | Array of String | SPDX identifiers | `["SPDX:GPL-3.0-or-later"]` |

### Optional Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `blender_version_max` | String | First unsupported version | `"5.1.0"` |
| `website` | String | Homepage/docs URL | `"https://..."` |
| `copyright` | Array of String | Copyright notices | `["2024 Name"]` |
| `tags` | Array of String | Category tags | `["Object", "Scene"]` |
| `platforms` | Array of String | OS/arch restrictions | `["windows-x64", "linux-x64"]` |
| `wheels` | Array of String | Bundled Python wheels | `["./wheels/lib.whl"]` |

### Permissions Table (Optional)

```toml
[permissions]
network = "Justification text without ending period"
files = "Justification text without ending period"
clipboard = "Justification text without ending period"
camera = "Justification text without ending period"
microphone = "Justification text without ending period"
```

Valid permission keys: `network`, `files`, `clipboard`, `camera`, `microphone`.

Each value is a single-sentence justification string without ending punctuation.

---

## CLI Commands (Blender 4.2+)

### Extension Management

```bash
# Validate manifest
blender --command extension validate blender_manifest.toml

# Build extension package
blender --command extension build \
    --source-dir ./my_addon \
    --output-filepath my_addon.zip

# Install from file
blender --command extension install-file my_addon.zip

# Server-side: generate repo index
blender --command extension server-generate --repo-dir ./repo
```

---

## Class Registration Requirements

### Registerable bpy.types Base Classes

| Base Class | Type Code | bl_idname Format |
|-----------|-----------|-----------------|
| `bpy.types.Operator` | `OT` | `"addon.action_name"` (lowercase, dot-separated) |
| `bpy.types.Panel` | `PT` | `"ADDON_PT_name"` (matches class name) |
| `bpy.types.Menu` | `MT` | `"ADDON_MT_name"` |
| `bpy.types.Header` | `HT` | `"ADDON_HT_name"` |
| `bpy.types.UIList` | `UL` | `"ADDON_UL_name"` |
| `bpy.types.PropertyGroup` | *(none)* | No bl_idname, registered by class |
| `bpy.types.AddonPreferences` | *(none)* | `bl_idname = __package__` |
| `bpy.types.KeyingSetInfo` | `KI` | `"ADDON_KI_name"` |
| `bpy.types.RenderEngine` | *(none)* | `bl_idname = "ADDON_RENDER"` |
| `bpy.types.NodeTree` | *(none)* | `bl_idname = "AddonNodeTree"` |

### Panel Required Attributes

```python
class ADDON_PT_panel(bpy.types.Panel):
    bl_label = "Panel Title"              # REQUIRED
    bl_idname = "ADDON_PT_panel"          # REQUIRED (matches class name)
    bl_space_type = 'VIEW_3D'             # REQUIRED
    bl_region_type = 'UI'                 # REQUIRED
    bl_category = "Tab Name"              # REQUIRED for 'UI' region
    # bl_parent_id = "ADDON_PT_parent"    # Optional: creates sub-panel
    # bl_options = {'DEFAULT_CLOSED'}     # Optional: collapsed by default
```

Valid `bl_space_type`: `'VIEW_3D'`, `'IMAGE_EDITOR'`, `'NODE_EDITOR'`, `'SEQUENCE_EDITOR'`, `'CLIP_EDITOR'`, `'DOPESHEET_EDITOR'`, `'GRAPH_EDITOR'`, `'NLA_EDITOR'`, `'TEXT_EDITOR'`, `'PROPERTIES'`, `'OUTLINER'`, `'PREFERENCES'`, `'FILE_BROWSER'`.

Valid `bl_region_type`: `'UI'` (sidebar), `'TOOLS'` (toolbar), `'HEADER'`, `'WINDOW'`, `'TOOL_PROPS'`.
