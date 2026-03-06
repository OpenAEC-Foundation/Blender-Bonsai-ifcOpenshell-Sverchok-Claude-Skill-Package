# blender-syntax-addons — Anti-Patterns

Common mistakes in Blender addon syntax and structure. Each entry includes the incorrect pattern, the correct alternative, and an explanation of WHY it fails.

---

## Anti-Pattern 1: Using Variables or Expressions in bl_info

**WRONG:**
```python
VERSION = (1, 0, 0)
bl_info = {
    "name": "My Addon",
    "version": VERSION,  # Variable reference
    "blender": (3, 6, 0),
    "category": "Object",
}
```

**CORRECT:**
```python
bl_info = {
    "name": "My Addon",
    "version": (1, 0, 0),  # Literal values only
    "blender": (3, 6, 0),
    "category": "Object",
}
```

**WHY:** Blender parses `bl_info` statically using AST evaluation without executing the Python file. Variable references, function calls, and expressions are not resolved during static parsing. The addon will appear with missing or default metadata in Edit > Preferences > Add-ons, or fail to be recognized entirely.

---

## Anti-Pattern 2: Wrong Class Naming Convention

**WRONG:**
```python
class MyOperator(bpy.types.Operator):
    bl_idname = "object.my_operator"
    bl_label = "My Operator"

class my_panel(bpy.types.Panel):
    bl_idname = "my_panel"
    bl_label = "My Panel"
```

**CORRECT:**
```python
class MYADDON_OT_my_operator(bpy.types.Operator):
    bl_idname = "myaddon.my_operator"
    bl_label = "My Operator"

class MYADDON_PT_my_panel(bpy.types.Panel):
    bl_idname = "MYADDON_PT_my_panel"
    bl_label = "My Panel"
```

**WHY:** Blender enforces the `{ADDON}_{TYPE}_{name}` naming convention for all registered types. Classes that do not follow this pattern raise `Warning: 'MyOperator' does not contain '_PT_' / '_OT_'` and may fail to register in Blender 4.0+. The `bl_idname` for operators must be lowercase with a single dot separator (`addon.action`). Panel `bl_idname` must match the class name.

---

## Anti-Pattern 3: Registering Classes in Wrong Order

**WRONG:**
```python
classes = (
    MYADDON_PT_panel,       # Panel depends on PropertyGroup but registered first
    MYADDON_OT_operator,
    MyPropertyGroup,         # PropertyGroup registered last
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
```

**CORRECT:**
```python
classes = (
    MyPropertyGroup,         # PropertyGroups FIRST
    MYADDON_OT_operator,     # Then operators
    MYADDON_PT_panel,        # UI elements LAST
)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
```

**WHY:** `bpy.props.PointerProperty(type=MyPropertyGroup)` requires `MyPropertyGroup` to be registered before any class that references it. Blender resolves type references at registration time, not at class definition time. Out-of-order registration raises `RuntimeError: PointerProperty type unregistered`.

---

## Anti-Pattern 4: Not Reversing Unregistration Order

**WRONG:**
```python
def unregister():
    del bpy.types.Scene.my_props
    for cls in classes:  # Same order as registration
        bpy.utils.unregister_class(cls)
```

**CORRECT:**
```python
def unregister():
    del bpy.types.Scene.my_props
    for cls in reversed(classes):  # Reverse order
        bpy.utils.unregister_class(cls)
```

**WHY:** Unregistering in forward order removes PropertyGroups before the panels and operators that reference them. This creates dangling type references that cause `RuntimeError` or segfaults in some Blender versions. Reverse order ensures dependent classes are removed before their dependencies.

---

## Anti-Pattern 5: Using __name__ Instead of __package__ for bl_idname

**WRONG:**
```python
class MyPrefs(bpy.types.AddonPreferences):
    bl_idname = __name__  # Returns "my_addon.__init__" in multi-file addons
```

**CORRECT:**
```python
class MyPrefs(bpy.types.AddonPreferences):
    bl_idname = __package__  # Returns "my_addon" — matches addon module name
```

**WHY:** In a multi-file addon, `__name__` evaluates to `"my_addon.__init__"` inside `__init__.py`, which does not match the addon's module name (`"my_addon"`). This causes `KeyError` when accessing preferences via `bpy.context.preferences.addons[__package__].preferences` and prevents the preferences panel from appearing in Edit > Preferences.

---

## Anti-Pattern 6: Missing Reload Support in Legacy Addons

**WRONG:**
```python
# __init__.py — legacy addon
bl_info = { ... }

from . import operators
from . import panels

import bpy
# ... register/unregister ...
```

**CORRECT:**
```python
# __init__.py — legacy addon with reload support
bl_info = { ... }

if "bpy" in locals():
    import importlib
    importlib.reload(operators)
    importlib.reload(panels)
else:
    from . import operators
    from . import panels

import bpy
# ... register/unregister ...
```

**WHY:** Without the `if "bpy" in locals()` reload guard, toggling the addon off/on in preferences or pressing F8 uses stale cached module objects. Changes to submodules during development are not reflected until Blender restarts. The reload pattern forces Python to re-execute submodule code on re-enable.

---

## Anti-Pattern 7: Using Absolute Imports in Sub-Modules

**WRONG:**
```python
# operators.py
from my_addon.properties import MyPropertyGroup
import my_addon.utils as utils
```

**CORRECT:**
```python
# operators.py
from .properties import MyPropertyGroup
from . import utils
```

**WHY:** Absolute imports resolve against `sys.path`, which depends on the installation method and Blender version. When the addon directory is renamed or installed as an extension (where the package is nested under a repository namespace), absolute imports fail with `ModuleNotFoundError`. Relative imports always resolve within the addon's own package.

---

## Anti-Pattern 8: Attaching PointerProperty Before Registering the Type

**WRONG:**
```python
def register():
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
    for cls in classes:
        bpy.utils.register_class(cls)  # MyPropertyGroup registered after PointerProperty
```

**CORRECT:**
```python
def register():
    for cls in classes:
        bpy.utils.register_class(cls)  # Register types first
    bpy.types.Scene.my_props = bpy.props.PointerProperty(type=MyPropertyGroup)
```

**WHY:** `PointerProperty(type=MyPropertyGroup)` validates that `MyPropertyGroup` is a registered RNA type at the time of attachment. Calling it before `register_class(MyPropertyGroup)` raises `TypeError: expected an RNA type, got 'MyPropertyGroup'`. Property attachments must always follow class registration.

---

## Anti-Pattern 9: Writing bl_info AND blender_manifest.toml for 4.2+ Only

**WRONG:**
```python
# __init__.py for 4.2+-only extension
bl_info = {
    "name": "My Extension",
    "version": (1, 0, 0),
    "blender": (4, 2, 0),
    "category": "Object",
}
```
```toml
# blender_manifest.toml
id = "my_extension"
# ... full manifest ...
```

**CORRECT — if targeting only 4.2+:**
```python
# __init__.py — NO bl_info
import bpy
# ... register/unregister only ...
```
```toml
# blender_manifest.toml — sole source of metadata
id = "my_extension"
# ... full manifest ...
```

**CORRECT — if supporting both legacy AND extension:**
```python
# __init__.py — bl_info for legacy, manifest for extension
bl_info = {
    "name": "My Extension",
    "version": (1, 0, 0),
    "blender": (3, 6, 0),
    "category": "Object",
}
```
```toml
# blender_manifest.toml — used by 4.2+
blender_version_min = "4.2.0"
```

**WHY:** When targeting 4.2+ exclusively, including `bl_info` is misleading and causes linter warnings. Blender 4.2+ ignores `bl_info` entirely for extensions. If supporting both legacy and extension systems, `bl_info` must target the legacy version range (< 4.2) while the manifest handles 4.2+.

---

## Anti-Pattern 10: Invalid Category in bl_info

**WRONG:**
```python
bl_info = {
    "name": "My Addon",
    "category": "Tools",  # Not a valid Blender category
}
```

**CORRECT:**
```python
bl_info = {
    "name": "My Addon",
    "category": "Object",  # Must be from Blender's fixed category list
}
```

**WHY:** Blender has a fixed list of valid categories. Using an invalid category places the addon in an "Unknown" section in preferences, making it harder for users to find. Valid categories include: `3D View`, `Add Mesh`, `Add Curve`, `Animation`, `Compositing`, `Development`, `Game Engine`, `Import-Export`, `Lighting`, `Material`, `Mesh`, `Node`, `Object`, `Paint`, `Physics`, `Render`, `Rigging`, `Scene`, `Sequencer`, `System`, `Text Editor`, `UV`.
