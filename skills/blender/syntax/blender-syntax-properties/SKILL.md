---
name: blender-syntax-properties
description: >
  Use when adding custom properties to Blender objects -- BoolProperty, IntProperty,
  FloatProperty, EnumProperty, PointerProperty, or PropertyGroup. Prevents the common
  mistake of not using update callbacks (changes not reflected in UI) or returning stale
  items from dynamic EnumProperty. Covers all bpy.props types, subtypes, units, getters/
  setters, and CollectionProperty patterns.
  Keywords: bpy.props, PropertyGroup, BoolProperty, IntProperty, FloatProperty, EnumProperty, PointerProperty, CollectionProperty, update callback, dynamic enum, add custom property, user input, settings.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-properties

## Quick Reference

### Property Type Selection

| Need | Property Type | Python Type |
|------|---------------|-------------|
| Toggle/flag | `BoolProperty` | `bool` |
| Count/index | `IntProperty` | `int` |
| Size/factor | `FloatProperty` | `float` |
| Name/path | `StringProperty` | `str` |
| Dropdown/mode | `EnumProperty` | `str` (identifier) |
| Axis toggles (X, Y, Z) | `BoolVectorProperty` | `tuple[bool, ...]` |
| Dimensions | `IntVectorProperty` | `tuple[int, ...]` |
| Color/position/rotation | `FloatVectorProperty` | `tuple[float, ...]` |
| Link to another type | `PointerProperty` | Reference to PropertyGroup or ID |
| Dynamic list | `CollectionProperty` | List of PropertyGroup items |

### Critical Warnings

**ALWAYS** use Python annotation syntax (`:` not `=`) for property declarations in classes.

**ALWAYS** register sub-PropertyGroups BEFORE parent PropertyGroups — registration order matters.

**ALWAYS** store dynamic enum callback results in a module-level variable to prevent garbage collection crashes.

**ALWAYS** set explicit unique integer values in enum items used with `ENUM_FLAG` or getter/setter patterns.

**NEVER** modify the property that triggered an `update` callback — causes infinite recursion.

**NEVER** define `set` without a matching `get` callback — Blender raises an error.

**NEVER** call operators inside `update` callbacks — they trigger undo and corrupt state.

**NEVER** set `default` on `EnumProperty` when `items` is a callback — use `default=None` or omit it.

---

## Version-Critical Changes

| Version | Change | Impact |
|---------|--------|--------|
| 4.1 | Enum ID properties via `id_properties_ui` | Integer custom properties can display as dropdowns |
| 5.0 | `property_unset()` replaces `del obj["prop"]` | RNA properties use separate storage from custom properties |
| 5.0 | `get_transform` / `set_transform` callbacks | Faster alternative to `get`/`set` using internal storage |
| 5.0 | IDProperty storage split | `bpy.props`-defined properties no longer accessible via dict syntax |

### Blender 5.0 Property Storage Split

```python
# Blender <5.0: dict access to RNA properties worked
value = obj["my_rna_prop"]  # WORKED in 4.x

# Blender 5.0+: dict access BROKEN for bpy.props-defined properties
value = obj["my_rna_prop"]  # Returns None or KeyError

# CORRECT in 5.0+: use RNA path
value = obj.my_rna_prop

# Reset property to default (replaces del obj["prop"]):
obj.property_unset("my_rna_prop")
```

---

## Essential Patterns

### Pattern 1: Basic Property Declaration

```python
# Blender 3.x/4.x/5.x: annotation syntax REQUIRED
class MyOperator(bpy.types.Operator):
    bl_idname = "object.my_operator"
    bl_label = "My Operator"

    count: bpy.props.IntProperty(name="Count", default=5, min=1, max=100)
    factor: bpy.props.FloatProperty(name="Factor", default=1.0, subtype='FACTOR')
    name_input: bpy.props.StringProperty(name="Name", default="Object")
```

### Pattern 2: PropertyGroup Registration

```python
# Blender 3.x/4.x/5.x
class MySettings(bpy.types.PropertyGroup):
    enabled: bpy.props.BoolProperty(name="Enabled", default=True)
    count: bpy.props.IntProperty(name="Count", default=5, min=1)
    scale: bpy.props.FloatProperty(name="Scale", default=1.0, min=0.01)

def register():
    bpy.utils.register_class(MySettings)
    bpy.types.Scene.my_settings = bpy.props.PointerProperty(type=MySettings)

def unregister():
    del bpy.types.Scene.my_settings
    bpy.utils.unregister_class(MySettings)
```

### Pattern 3: Nested PropertyGroups

```python
# Blender 3.x/4.x/5.x: registration order is CRITICAL
class SubSettings(bpy.types.PropertyGroup):
    value: bpy.props.FloatProperty(name="Value", default=0.0)

class MainSettings(bpy.types.PropertyGroup):
    sub: bpy.props.PointerProperty(type=SubSettings)
    items: bpy.props.CollectionProperty(type=SubSettings)
    active_index: bpy.props.IntProperty()

def register():
    bpy.utils.register_class(SubSettings)    # FIRST: sub-types
    bpy.utils.register_class(MainSettings)   # SECOND: parent type
    bpy.types.Scene.settings = bpy.props.PointerProperty(type=MainSettings)

def unregister():
    del bpy.types.Scene.settings
    bpy.utils.unregister_class(MainSettings)  # Reverse order
    bpy.utils.unregister_class(SubSettings)
```

### Pattern 4: Static EnumProperty

```python
# Blender 3.x/4.x/5.x
# Items: (identifier, name, description[, icon[, number]])
my_mode: bpy.props.EnumProperty(
    name="Mode",
    items=[
        ('FAST', "Fast", "Low quality, fast render", 'PLAY', 0),
        ('MEDIUM', "Medium", "Balanced quality", 'PAUSE', 1),
        ('HIGH', "High", "High quality, slow render", 'RENDER_STILL', 2),
    ],
    default='MEDIUM',
)
```

### Pattern 5: Dynamic EnumProperty

```python
# Blender 3.x/4.x/5.x
# WARNING: MUST store results to prevent garbage collection crash

_enum_items_cache = []  # Module-level storage REQUIRED

def get_object_items(self, context):
    global _enum_items_cache
    _enum_items_cache = [
        (obj.name, obj.name, f"Select {obj.name}", 'OBJECT_DATA', i)
        for i, obj in enumerate(bpy.data.objects)
    ]
    if not _enum_items_cache:
        _enum_items_cache = [('NONE', "None", "No objects available", 'ERROR', 0)]
    return _enum_items_cache

my_object: bpy.props.EnumProperty(
    name="Object",
    items=get_object_items,
    # default MUST be None or omitted when items is a callback
)
```

### Pattern 6: Update Callback

```python
# Blender 3.x/4.x/5.x
def on_count_changed(self, context):
    # SAFE: modify OTHER properties on self
    if self.count > 10:
        self.warning = "High count may impact performance"
    # UNSAFE: do NOT modify self.count here (infinite recursion)
    # UNSAFE: do NOT call bpy.ops.* here (undo corruption)

class MySettings(bpy.types.PropertyGroup):
    count: bpy.props.IntProperty(
        name="Count", default=5, update=on_count_changed,
    )
    warning: bpy.props.StringProperty()
```

### Pattern 7: Getter/Setter

```python
# Blender 3.x/4.x/5.x
def get_computed(self):
    # Compute value on read — no internal storage
    return len(bpy.data.objects)

def set_computed(self, value):
    # Handle write — must store somewhere if persistence needed
    self["_cached_count"] = value

class MySettings(bpy.types.PropertyGroup):
    object_count: bpy.props.IntProperty(
        name="Object Count",
        get=get_computed,
        set=set_computed,
    )
```

### Pattern 8: Get/Set Transform (Blender 5.0+)

```python
# Blender 5.0+ ONLY: faster than get/set, uses internal storage
def clamp_transform(self, new_value, curr_value, is_set):
    """Transform value before storing."""
    return max(0.0, min(new_value, 100.0))

def display_transform(self, curr_value, is_set):
    """Transform value on read."""
    return curr_value * 2.0 if is_set else 0.0

class MySettings(bpy.types.PropertyGroup):
    clamped: bpy.props.FloatProperty(
        name="Clamped",
        set_transform=clamp_transform,
        get_transform=display_transform,
    )
```

### Pattern 9: ENUM_FLAG (Multi-Select)

```python
# Blender 3.x/4.x/5.x: numbers MUST be powers of 2
my_flags: bpy.props.EnumProperty(
    name="Axes",
    items=[
        ('X', "X Axis", "Include X axis", 1),
        ('Y', "Y Axis", "Include Y axis", 2),
        ('Z', "Z Axis", "Include Z axis", 4),
    ],
    options={'ENUM_FLAG'},
    default={'X', 'Z'},  # Default is a SET, not a string
)
```

### Pattern 10: PointerProperty with Poll

```python
# Blender 3.x/4.x/5.x: filter selectable objects
def filter_mesh_objects(self, obj):
    return obj.type == 'MESH'

class MySettings(bpy.types.PropertyGroup):
    target: bpy.props.PointerProperty(
        type=bpy.types.Object,
        name="Target",
        poll=filter_mesh_objects,
    )
```

### Pattern 11: CollectionProperty Operations

```python
# Blender 3.x/4.x/5.x
scene = bpy.context.scene

# Add item
item = scene.my_settings.items.add()
item.name = "New Item"
item.value = 42.0

# Remove by index
scene.my_settings.items.remove(0)

# Move item (from_index, to_index)
scene.my_settings.items.move(0, 2)

# Clear all
scene.my_settings.items.clear()

# Iterate
for item in scene.my_settings.items:
    print(item.name, item.value)

# Access by index
first = scene.my_settings.items[0]

# Find by name (returns index or -1)
idx = scene.my_settings.items.find("New Item")
```

---

## Subtype and Unit Quick Reference

### Numeric Subtypes

| Subtype | Applies To | Effect |
|---------|-----------|--------|
| `'NONE'` | All | Default display |
| `'PIXEL'` | Int/Float | Pixel unit display |
| `'UNSIGNED'` | Int | Unsigned display |
| `'PERCENTAGE'` | Float | 0-100% display |
| `'FACTOR'` | Float | 0.0-1.0 slider |
| `'ANGLE'` | Float | Radians with degree display |
| `'TIME'` | Float | Scene-relative time (frames) |
| `'TIME_ABSOLUTE'` | Float | Absolute time (seconds) |
| `'DISTANCE'` | Float | Distance in scene units |
| `'POWER'` | Float | Power unit display |
| `'TEMPERATURE'` | Float | Temperature display |

### Vector Subtypes

| Subtype | Applies To | Effect |
|---------|-----------|--------|
| `'COLOR'` | FloatVector | Linear RGB color picker |
| `'COLOR_GAMMA'` | FloatVector | Gamma-space color picker |
| `'TRANSLATION'` | FloatVector | Position in scene units |
| `'DIRECTION'` | FloatVector | Normalized direction |
| `'VELOCITY'` | FloatVector | Velocity vector |
| `'ACCELERATION'` | FloatVector | Acceleration vector |
| `'EULER'` | FloatVector | Euler rotation (radians) |
| `'QUATERNION'` | FloatVector | Quaternion rotation (size=4) |
| `'AXISANGLE'` | FloatVector | Axis-angle rotation (size=4) |
| `'XYZ'` | FloatVector | Generic XYZ coordinates |
| `'MATRIX'` | FloatVector | Matrix representation |

### String Subtypes

| Subtype | Effect |
|---------|--------|
| `'NONE'` | Plain text input |
| `'FILE_PATH'` | File browser button |
| `'DIR_PATH'` | Directory browser button |
| `'FILE_NAME'` | File name field |
| `'BYTE_STRING'` | Byte string |
| `'PASSWORD'` | Masked input (***) |

### Unit Values (FloatProperty / FloatVectorProperty only)

| Unit | Display |
|------|---------|
| `'NONE'` | No unit |
| `'LENGTH'` | Scene length unit |
| `'AREA'` | Area (length²) |
| `'VOLUME'` | Volume (length³) |
| `'ROTATION'` | Degrees/radians |
| `'TIME'` | Scene-relative time |
| `'TIME_ABSOLUTE'` | Absolute seconds |
| `'VELOCITY'` | Speed |
| `'ACCELERATION'` | Acceleration |
| `'MASS'` | Mass |
| `'CAMERA'` | Camera distance |
| `'POWER'` | Power |
| `'TEMPERATURE'` | Temperature |

### Options Flags

| Flag | Applies To | Effect |
|------|-----------|--------|
| `'HIDDEN'` | All | Hidden from UI, accessible via Python |
| `'SKIP_SAVE'` | All | Value not saved between operator invocations |
| `'ANIMATABLE'` | All (default) | Can be keyframed |
| `'LIBRARY_EDITABLE'` | All | Editable on linked data-blocks |
| `'PROPORTIONAL'` | Numeric | Proportional editing support |
| `'TEXTEDIT_UPDATE'` | String | Updates on each keystroke |
| `'ENUM_FLAG'` | Enum only | Multi-select with power-of-2 values |

---

## Decision Trees

### When to Use PointerProperty vs CollectionProperty

```
Need to reference ONE item?
├── YES → PointerProperty
│   ├── Reference to another PropertyGroup? → type=MyPropertyGroup
│   └── Reference to an ID type? → type=bpy.types.Object (or Material, etc.)
└── NO → Need a LIST of items?
    └── YES → CollectionProperty(type=MyPropertyGroup)
        └── Track active selection? → Add IntProperty for active_index
```

### When to Use get/set vs get_transform/set_transform

```
Blender version?
├── <5.0 → Use get/set (only option)
└── >=5.0
    ├── Need custom storage logic? → Use get/set
    └── Only need value transformation?
        └── Use get_transform/set_transform (faster, uses internal storage)
```

### When to Use update vs msgbus

```
Property changes on THIS class?
├── YES → Use update callback on the property
└── NO → Watching EXTERNAL property changes?
    └── YES → Use bpy.msgbus.subscribe_rna()
```

---

## Attachment Points

Properties can be attached to any Blender ID type:

```python
# Per-scene settings
bpy.types.Scene.my_prop = bpy.props.PointerProperty(type=MySettings)

# Per-object settings
bpy.types.Object.my_prop = bpy.props.PointerProperty(type=MySettings)

# Per-material settings
bpy.types.Material.my_prop = bpy.props.FloatProperty(name="Custom")

# Per-mesh settings
bpy.types.Mesh.my_prop = bpy.props.IntProperty(name="Custom")

# Per-bone settings
bpy.types.Bone.my_prop = bpy.props.StringProperty(name="Custom")

# Per-window-manager (session-only, not saved)
bpy.types.WindowManager.my_prop = bpy.props.BoolProperty(name="Temp")
```

**WindowManager** properties are session-only — they reset when Blender restarts. Use for temporary UI state.

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for all bpy.props types
- [references/examples.md](references/examples.md) — Working code examples for every property pattern
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do, with explanations

### Official Sources

- https://docs.blender.org/api/current/bpy.props.html
- https://docs.blender.org/api/current/bpy.types.PropertyGroup.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
