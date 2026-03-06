# blender-syntax-properties: API Method Reference

Sources: https://docs.blender.org/api/current/bpy.props.html,
https://docs.blender.org/api/current/bpy.types.PropertyGroup.html

---

## Property Function Signatures

All parameters MUST be passed as keyword arguments. Properties can be assigned to subclasses of `ID`, `Bone`, and `PoseBone`.

### BoolProperty

```python
bpy.props.BoolProperty(
    name="",                    # str: UI display name
    description="",             # str: tooltip text
    translation_context="*",    # str: i18n context
    default=False,              # bool: initial value
    options={'ANIMATABLE'},     # set: property flags
    override=set(),             # set: library override flags
    tags=set(),                 # set: custom tags
    subtype='NONE',             # str: 'NONE' only
    update=None,                # callable(self, context) -> None
    get=None,                   # callable(self) -> bool
    set=None,                   # callable(self, value) -> None
    get_transform=None,         # 5.0+: callable(self, curr_value, is_set) -> bool
    set_transform=None,         # 5.0+: callable(self, new_value, curr_value, is_set) -> bool
)
```

### BoolVectorProperty

```python
bpy.props.BoolVectorProperty(
    name="",
    description="",
    translation_context="*",
    default=(False, False, False),  # tuple[bool, ...]: initial values
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    subtype='NONE',             # str: 'NONE', 'LAYER', 'LAYER_MEMBER'
    size=3,                     # int: 1 to 32
    update=None,
    get=None,                   # callable(self) -> tuple[bool, ...]
    set=None,                   # callable(self, value) -> None
    get_transform=None,         # 5.0+
    set_transform=None,         # 5.0+
)
```

### IntProperty

```python
bpy.props.IntProperty(
    name="",
    description="",
    translation_context="*",
    default=0,                  # int: initial value
    min=-(2**31),               # int: hard minimum (clamped on assignment)
    max=(2**31) - 1,            # int: hard maximum
    soft_min=-(2**31),          # int: UI slider minimum (does NOT clamp)
    soft_max=(2**31) - 1,       # int: UI slider maximum (does NOT clamp)
    step=1,                     # int: UI increment per click
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    subtype='NONE',             # str: 'NONE', 'PIXEL', 'UNSIGNED', 'PERCENTAGE',
                                #      'FACTOR', 'ANGLE', 'TIME', 'TIME_ABSOLUTE',
                                #      'DISTANCE', 'DISTANCE_CAMERA', 'POWER',
                                #      'TEMPERATURE'
    update=None,
    get=None,                   # callable(self) -> int
    set=None,                   # callable(self, value) -> None
    get_transform=None,         # 5.0+
    set_transform=None,         # 5.0+
)
```

### IntVectorProperty

```python
bpy.props.IntVectorProperty(
    name="",
    description="",
    translation_context="*",
    default=(0, 0, 0),          # tuple[int, ...]: initial values
    min=-(2**31),
    max=(2**31) - 1,
    soft_min=-(2**31),
    soft_max=(2**31) - 1,
    step=1,
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    subtype='NONE',             # str: same as IntProperty + vector subtypes
    size=3,                     # int: 1 to 32
    update=None,
    get=None,                   # callable(self) -> tuple[int, ...]
    set=None,
    get_transform=None,         # 5.0+
    set_transform=None,         # 5.0+
)
```

### FloatProperty

```python
bpy.props.FloatProperty(
    name="",
    description="",
    translation_context="*",
    default=0.0,                # float: initial value
    min=-3.402823e+38,          # float: hard minimum
    max=3.402823e+38,           # float: hard maximum
    soft_min=-3.402823e+38,     # float: UI slider minimum
    soft_max=3.402823e+38,      # float: UI slider maximum
    step=3,                     # int: UI increment (1-100, divided by 100)
    precision=2,                # int: decimal places displayed (0-6)
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    subtype='NONE',             # str: 'NONE', 'PIXEL', 'UNSIGNED', 'PERCENTAGE',
                                #      'FACTOR', 'ANGLE', 'TIME', 'TIME_ABSOLUTE',
                                #      'DISTANCE', 'DISTANCE_CAMERA', 'POWER',
                                #      'TEMPERATURE'
    unit='NONE',                # str: 'NONE', 'LENGTH', 'AREA', 'VOLUME',
                                #      'ROTATION', 'TIME', 'TIME_ABSOLUTE',
                                #      'VELOCITY', 'ACCELERATION', 'MASS',
                                #      'CAMERA', 'POWER', 'TEMPERATURE'
    update=None,
    get=None,                   # callable(self) -> float
    set=None,                   # callable(self, value) -> None
    get_transform=None,         # 5.0+
    set_transform=None,         # 5.0+
)
```

### FloatVectorProperty

```python
bpy.props.FloatVectorProperty(
    name="",
    description="",
    translation_context="*",
    default=(0.0, 0.0, 0.0),   # tuple[float, ...]: initial values
    min=-3.402823e+38,
    max=3.402823e+38,
    soft_min=-3.402823e+38,
    soft_max=3.402823e+38,
    step=3,
    precision=2,
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    subtype='NONE',             # str: 'NONE' + numeric subtypes + vector subtypes:
                                #      'COLOR', 'COLOR_GAMMA', 'TRANSLATION',
                                #      'DIRECTION', 'VELOCITY', 'ACCELERATION',
                                #      'EULER', 'QUATERNION', 'AXISANGLE',
                                #      'XYZ', 'XYZ_LENGTH', 'MATRIX',
                                #      'COORDINATES', 'LAYER', 'LAYER_MEMBER'
    unit='NONE',                # str: same as FloatProperty
    size=3,                     # int: 1 to 32
    update=None,
    get=None,                   # callable(self) -> tuple[float, ...]
    set=None,
    get_transform=None,         # 5.0+
    set_transform=None,         # 5.0+
)
```

### StringProperty

```python
bpy.props.StringProperty(
    name="",
    description="",
    translation_context="*",
    default="",                 # str: initial value
    maxlen=0,                   # int: max length (0 = unlimited)
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    subtype='NONE',             # str: 'NONE', 'FILE_PATH', 'DIR_PATH',
                                #      'FILE_NAME', 'BYTE_STRING', 'PASSWORD'
    update=None,
    get=None,                   # callable(self) -> str
    set=None,                   # callable(self, value) -> None
    get_transform=None,         # 5.0+
    set_transform=None,         # 5.0+
)
```

### EnumProperty

```python
bpy.props.EnumProperty(
    items,                      # REQUIRED: list[tuple] or callable(self, context)
    name="",
    description="",
    translation_context="*",
    default=None,               # str or set: MUST be None when items is a callback
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    update=None,
    get=None,                   # callable(self) -> int (item number, NOT string)
    set=None,                   # callable(self, value: int) -> None
    # NOTE: No subtype, no unit
    # NOTE: No get_transform / set_transform
)
```

**Items tuple format:**

```python
# Minimum (3-tuple):
("IDENTIFIER", "Display Name", "Tooltip")

# With icon (4-tuple):
("IDENTIFIER", "Display Name", "Tooltip", 'ICON_NAME')
("IDENTIFIER", "Display Name", "Tooltip", icon_int_value)

# Full (5-tuple):
("IDENTIFIER", "Display Name", "Tooltip", 'ICON_NAME', unique_int)
```

**Rules for item numbers:**
- MUST be unique per item
- MUST be explicitly set when using ENUM_FLAG (powers of 2: 1, 2, 4, 8, ...)
- MUST be explicitly set when using get/set callbacks
- Used for stable file serialization — changing numbers breaks saved values

### PointerProperty

```python
bpy.props.PointerProperty(
    type=None,                  # REQUIRED: PropertyGroup subclass or ID subclass
    name="",
    description="",
    translation_context="*",
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    poll=None,                  # callable(self, object) -> bool: filter candidates
    update=None,                # callable(self, context) -> None
    # NOTE: No get/set, no subtype, no unit
    # NOTE: No get_transform / set_transform
)
```

### CollectionProperty

```python
bpy.props.CollectionProperty(
    type=None,                  # REQUIRED: PropertyGroup subclass ONLY (not ID types)
    name="",
    description="",
    translation_context="*",
    options={'ANIMATABLE'},
    override=set(),
    tags=set(),
    # NOTE: No update, get, set, poll callbacks
    # NOTE: No get_transform / set_transform
)
```

### RemoveProperty

```python
bpy.props.RemoveProperty(
    cls,                        # class: the class to remove the property from
    attr="",                    # str: name of the property attribute
)
```

---

## CollectionProperty Instance Methods

```python
collection = scene.my_settings.items  # bpy_prop_collection

# Add new item (returns the added item)
item = collection.add()

# Remove item at index
collection.remove(index)        # int: index to remove

# Move item from one index to another
collection.move(from_index, to_index)

# Clear all items
collection.clear()

# Find item by name (returns index, -1 if not found)
index = collection.find("name")

# Access by index
item = collection[0]

# Access by name
item = collection["name"]      # KeyError if not found
item = collection.get("name")  # None if not found

# Length
count = len(collection)

# Iteration
for item in collection:
    print(item.name)
```

---

## PropertyGroup Inherited Methods (from bpy_struct)

```python
# Property state (Blender 3.x/4.x/5.x)
pg.is_property_set(property)           # bool: True if explicitly set
pg.is_property_hidden(property)        # bool: True if hidden from UI
pg.is_property_readonly(property)      # bool: True if read-only
pg.property_unset(property)            # Reset to default value (5.0+ preferred)

# RNA paths
pg.path_from_id(property="")           # str: RNA path from ID data
pg.path_resolve(path, coerce=True)     # Resolve RNA path to value

# Animation
pg.keyframe_insert(data_path, index=-1, frame=bpy.context.scene.frame_current)
pg.keyframe_delete(data_path, index=-1, frame=bpy.context.scene.frame_current)
pg.driver_add(path, index=-1)          # Add driver
pg.driver_remove(path, index=-1)       # Remove driver
```

---

## Callback Signatures Reference

### update(self, context)

```python
def update_func(self, context):
    """Called when property value changes.

    Args:
        self: the data structure (PropertyGroup, Operator, etc.)
              WARNING: For operator properties, self is OperatorProperties,
              NOT the Operator instance.
        context: bpy.context (may be None in some edge cases)

    Rules:
        - NEVER modify the property that triggered this callback
        - NEVER call bpy.ops.* (triggers undo, corrupts state)
        - May be called from threaded context
        - Not available on CollectionProperty
    """
    pass
```

### get(self) / set(self, value)

```python
def getter(self):
    """Custom getter — replaces internal storage.

    Args:
        self: the data structure
    Returns:
        Value matching the property type (bool/int/float/str/tuple)
        For EnumProperty: return INTEGER (item number), NOT string
    """
    return self.get("_internal_key", default_value)

def setter(self, value):
    """Custom setter — replaces internal storage.

    Args:
        self: the data structure
        value: the new value being assigned
               For EnumProperty: receives INTEGER (item number), NOT string
    Rules:
        - NEVER define set without a matching get
        - Property is NOT stored automatically when get/set are defined
        - Store manually via self["key"] if persistence is needed
    """
    self["_internal_key"] = value
```

### get_transform / set_transform (Blender 5.0+)

```python
def get_transform_func(self, curr_value, is_set):
    """Transform stored value on read — uses internal storage.

    Args:
        self: the data structure
        curr_value: current internally-stored value
        is_set: bool — True if property has been explicitly set
    Returns:
        Transformed value (same type as property)
    Notes:
        - Faster than get/set because internal storage is preserved
        - Can be used without set_transform (read-only transform)
    """
    return curr_value

def set_transform_func(self, new_value, curr_value, is_set):
    """Transform value before storing — uses internal storage.

    Args:
        self: the data structure
        new_value: the value being assigned
        curr_value: current stored value
        is_set: bool — True if property has been explicitly set
    Returns:
        Value to store (MUST return a value, not None)
    Notes:
        - REQUIRES matching get_transform
        - Cannot be used without get_transform
        - May be called from threaded context
    """
    return new_value
```

### poll(self, object) — PointerProperty only

```python
def poll_func(self, object):
    """Filter which objects can be selected for a PointerProperty.

    Args:
        self: the struct the PointerProperty is defined on
        object: the candidate object being tested
    Returns:
        True to allow selection, False to reject
    """
    return object.type == 'MESH'
```

### Dynamic enum items callback

```python
def enum_items_func(self, context):
    """Generate enum items dynamically.

    Args:
        self: the data structure
        context: bpy.context (may be None)
    Returns:
        list of tuples: 3-tuple, 4-tuple, or 5-tuple format

    CRITICAL: Store the returned list in a module-level variable.
    If Python garbage-collects the list while Blender holds string
    pointers, Blender will crash or display garbage text.
    """
    global _cached_items
    _cached_items = [("ID", "Name", "Desc", i) for i in range(5)]
    return _cached_items
```

---

## Blender 4.1: Enum ID Properties

Integer custom properties (dict-style, NOT bpy.props) can display as dropdowns:

```python
# Set integer custom property
obj["my_mode"] = 0

# Add enum display via id_properties_ui
ui = obj.id_properties_ui("my_mode")
ui.update(items=[
    ("MODE_A", "Mode A", "First mode"),
    ("MODE_B", "Mode B", "Second mode"),
    ("MODE_C", "Mode C", "Third mode"),
])

# Alternative: rna_prop_ui convenience function
from rna_prop_ui import rna_idprop_ui_create
rna_idprop_ui_create(obj, "my_mode", default=0, items=[
    ("MODE_A", "Mode A", "First mode"),
    ("MODE_B", "Mode B", "Second mode"),
])
```

This is DIFFERENT from `bpy.props.EnumProperty` — it applies to raw ID properties only.
