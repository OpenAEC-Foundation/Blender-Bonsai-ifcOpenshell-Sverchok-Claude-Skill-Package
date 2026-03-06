# blender-syntax-properties: Anti-Patterns

Sources: https://docs.blender.org/api/current/bpy.props.html,
https://docs.blender.org/api/current/bpy.types.PropertyGroup.html,
Blender Python API gotchas, common forum issues

---

## AP-01: Assignment Syntax Instead of Annotation

**WRONG:**
```python
class MySettings(bpy.types.PropertyGroup):
    count = bpy.props.IntProperty(name="Count")  # WRONG: = assignment
```

**CORRECT:**
```python
class MySettings(bpy.types.PropertyGroup):
    count: bpy.props.IntProperty(name="Count")   # CORRECT: : annotation
```

**Why:** Blender's RNA system requires Python annotation syntax (PEP 526) for property declarations on classes. Assignment syntax (`=`) does not register the property with the RNA system — the property will silently not appear in the UI and will not be saved.

---

## AP-02: Wrong Registration Order for Nested PropertyGroups

**WRONG:**
```python
def register():
    bpy.utils.register_class(ParentSettings)  # WRONG: parent first
    bpy.utils.register_class(SubSettings)      # Sub-type not yet known
```

**CORRECT:**
```python
def register():
    bpy.utils.register_class(SubSettings)      # FIRST: register dependencies
    bpy.utils.register_class(ParentSettings)   # SECOND: parent can now find SubSettings

def unregister():
    del bpy.types.Scene.my_settings
    bpy.utils.unregister_class(ParentSettings)  # Reverse order
    bpy.utils.unregister_class(SubSettings)
```

**Why:** When `ParentSettings` references `SubSettings` via `PointerProperty(type=SubSettings)` or `CollectionProperty(type=SubSettings)`, Blender must resolve the type at registration time. If the sub-type is not yet registered, Blender raises `RuntimeError: Error: Registering operator class: ... could not register`.

---

## AP-03: Dynamic Enum Without Module-Level Cache

**WRONG:**
```python
def get_items(self, context):
    return [  # Returned list is garbage-collected after function returns
        (obj.name, obj.name, "", i)
        for i, obj in enumerate(bpy.data.objects)
    ]
```

**CORRECT:**
```python
_cached_items = []  # Module-level variable prevents GC

def get_items(self, context):
    global _cached_items
    _cached_items = [
        (obj.name, obj.name, "", i)
        for i, obj in enumerate(bpy.data.objects)
    ]
    return _cached_items
```

**Why:** Python garbage-collects the returned list after the callback returns. Blender holds C-level pointers to the string data inside the tuples. When Python frees those strings, Blender reads freed memory — causing crashes, garbled text, or undefined behavior. ALWAYS store the list in a module-level variable.

---

## AP-04: Modifying the Triggering Property in an Update Callback

**WRONG:**
```python
def on_count_changed(self, context):
    if self.count > 100:
        self.count = 100  # WRONG: modifies the property that triggered this callback
```

**CORRECT:**
```python
# Option A: Use min/max instead of callback clamping
count: bpy.props.IntProperty(name="Count", default=5, min=1, max=100)

# Option B: Modify a DIFFERENT property
def on_count_changed(self, context):
    if self.count > 50:
        self.warning_text = "High count may be slow"  # Different property: safe
```

**Why:** Blender has no recursion protection for update callbacks. Modifying the triggering property calls the update callback again, which modifies the property again, causing infinite recursion until Python's recursion limit crashes the script.

---

## AP-05: Calling Operators Inside Update Callbacks

**WRONG:**
```python
def on_enabled_changed(self, context):
    if self.enabled:
        bpy.ops.object.select_all(action='SELECT')  # WRONG: operator in update
```

**CORRECT:**
```python
def on_enabled_changed(self, context):
    if self.enabled:
        # Direct data manipulation — no undo step, no operator
        for obj in bpy.data.objects:
            obj.select_set(True)
```

**Why:** Operators push undo steps. Update callbacks are called during property changes, which may already be inside an undo step. Calling operators here corrupts the undo stack and can cause crashes or data loss. Use direct data access (`bpy.data`) instead of operators (`bpy.ops`) inside callbacks.

---

## AP-06: Setting Default on Dynamic EnumProperty

**WRONG:**
```python
my_enum: bpy.props.EnumProperty(
    items=get_items_callback,
    default='FIRST_ITEM',  # WRONG: default with dynamic items
)
```

**CORRECT:**
```python
my_enum: bpy.props.EnumProperty(
    items=get_items_callback,
    # default MUST be None or omitted when items is a callback
)
```

**Why:** When `items` is a callback, the available items are not known at definition time. Setting a static `default` that does not exist in the dynamically-generated list causes `TypeError` or silent fallback to an invalid value. ALWAYS omit `default` or set it to `None` when using a callback.

---

## AP-07: EnumProperty Getter Returning String Instead of Integer

**WRONG:**
```python
def get_mode(self):
    return 'FAST'  # WRONG: returns string identifier

def set_mode(self, value):
    print(value)   # Receives integer, not string
```

**CORRECT:**
```python
def get_mode(self):
    return self.get("_mode", 0)  # CORRECT: returns integer (item number)

def set_mode(self, value):
    self["_mode"] = value  # value is integer

my_mode: bpy.props.EnumProperty(
    items=[
        ('FAST', "Fast", "", 0),    # Number field = the integer value
        ('MEDIUM', "Medium", "", 1),
        ('HIGH', "High", "", 2),
    ],
    get=get_mode,
    set=set_mode,
)
```

**Why:** When `get` and `set` are defined on an `EnumProperty`, the get callback MUST return the integer value (the `number` field from the items tuple), and the set callback receives an integer. This is different from normal enum usage where string identifiers are used. Returning a string from the getter causes `TypeError` or silent failures.

---

## AP-08: Defining set Without get

**WRONG:**
```python
def my_setter(self, value):
    self["_data"] = value * 2

my_prop: bpy.props.FloatProperty(
    name="Value",
    set=my_setter,  # WRONG: set without get
)
```

**CORRECT:**
```python
def my_getter(self):
    return self.get("_data", 0.0)

def my_setter(self, value):
    self["_data"] = value * 2

my_prop: bpy.props.FloatProperty(
    name="Value",
    get=my_getter,  # REQUIRED when set is defined
    set=my_setter,
)
```

**Why:** Blender raises `ValueError` if `set` is defined without a matching `get`. The same rule applies to `set_transform` without `get_transform` in Blender 5.0+. However, `get` without `set` IS valid — it creates a read-only property.

---

## AP-09: Forgetting to Delete PointerProperty Before Unregister

**WRONG:**
```python
def unregister():
    bpy.utils.unregister_class(MySettings)  # WRONG: pointer still references class
    # bpy.types.Scene.my_settings still exists, pointing to unregistered class
```

**CORRECT:**
```python
def unregister():
    del bpy.types.Scene.my_settings         # FIRST: remove the pointer
    bpy.utils.unregister_class(MySettings)  # SECOND: unregister the class
```

**Why:** If the PointerProperty still exists when the class is unregistered, Blender holds a dangling reference to the unregistered type. This causes errors on file save, property access, and re-registration. ALWAYS delete the pointer BEFORE unregistering the class.

---

## AP-10: Using ENUM_FLAG Without Powers of 2

**WRONG:**
```python
my_flags: bpy.props.EnumProperty(
    items=[
        ('A', "A", "", 0),  # WRONG: 0 is not a power of 2
        ('B', "B", "", 1),
        ('C', "C", "", 2),
        ('D', "D", "", 3),  # WRONG: 3 is not a power of 2
    ],
    options={'ENUM_FLAG'},
)
```

**CORRECT:**
```python
my_flags: bpy.props.EnumProperty(
    items=[
        ('A', "A", "", 1),   # 2^0
        ('B', "B", "", 2),   # 2^1
        ('C', "C", "", 4),   # 2^2
        ('D', "D", "", 8),   # 2^3
    ],
    options={'ENUM_FLAG'},
    default={'A'},  # Default is a SET
)
```

**Why:** `ENUM_FLAG` uses a bitmask internally. Values that are not powers of 2 overlap with combinations of other flags, causing incorrect multi-select behavior. Value 3 (binary 11) is indistinguishable from selecting both 1 and 2. Value 0 means "nothing selected" and cannot be individually toggled.

---

## AP-11: Dict-Style Access to RNA Properties in Blender 5.0+

**WRONG (Blender 5.0+):**
```python
# This worked in 4.x but is BROKEN in 5.0
value = obj["my_rna_property"]  # Returns None or KeyError in 5.0
del obj["my_rna_property"]      # Does not reset RNA property in 5.0
```

**CORRECT (Blender 5.0+):**
```python
value = obj.my_rna_property               # Use RNA attribute access
obj.property_unset("my_rna_property")     # Reset to default value
```

**Why:** Blender 5.0 separated RNA property storage from ID property (dict-style) storage. Properties defined with `bpy.props.*` are stored in a separate internal system, not in the ID property dictionary. Dict-style access (`obj["prop"]`) only accesses custom ID properties, not RNA-registered properties.

---

## AP-12: Empty Dynamic Enum Callback Return

**WRONG:**
```python
def get_items(self, context):
    items = [(obj.name, obj.name, "") for obj in bpy.data.objects]
    return items  # WRONG: returns empty list when no objects exist
```

**CORRECT:**
```python
_cached = []

def get_items(self, context):
    global _cached
    _cached = [(obj.name, obj.name, "", i) for i, obj in enumerate(bpy.data.objects)]
    if not _cached:
        _cached = [('NONE', "None", "No items available", 0)]
    return _cached
```

**Why:** An empty items list causes `ValueError` in Blender — enum properties MUST always have at least one item. ALWAYS include a fallback item when the dynamic list might be empty.

---

## AP-13: Using step=1 on FloatProperty Expecting Normal Increments

**WRONG assumption:**
```python
scale: bpy.props.FloatProperty(
    name="Scale",
    step=1,  # Expects UI increment of 1.0 — actually increments by 0.01
)
```

**CORRECT:**
```python
scale: bpy.props.FloatProperty(
    name="Scale",
    step=100,  # UI increment: 100 / 100 = 1.0 per click
)
```

**Why:** For `FloatProperty`, the `step` parameter is divided by 100 for the actual UI increment. `step=3` (the default) means the UI increments by 0.03 per click. For `IntProperty`, `step` is used directly (step=1 means increment by 1). This inconsistency is a common source of confusion.

---

## AP-14: CollectionProperty with ID Type

**WRONG:**
```python
class MySettings(bpy.types.PropertyGroup):
    objects: bpy.props.CollectionProperty(type=bpy.types.Object)  # WRONG
```

**CORRECT:**
```python
class ObjectRef(bpy.types.PropertyGroup):
    obj: bpy.props.PointerProperty(type=bpy.types.Object)

class MySettings(bpy.types.PropertyGroup):
    objects: bpy.props.CollectionProperty(type=ObjectRef)  # PropertyGroup wrapper
```

**Why:** `CollectionProperty` ONLY accepts `PropertyGroup` subclasses as its `type` parameter. It does NOT accept ID types (Object, Material, etc.). To store a collection of references to ID types, create a PropertyGroup wrapper containing a PointerProperty.

---

## AP-15: Assuming update Callback self is the Operator

**WRONG:**
```python
class MY_OT_example(bpy.types.Operator):
    bl_idname = "my.example"
    bl_label = "Example"

    count: bpy.props.IntProperty(
        update=lambda self, ctx: self.report({'INFO'}, "Changed"),
        # WRONG: self is OperatorProperties, not the Operator
        # self.report() does not exist on OperatorProperties
    )
```

**CORRECT:**
```python
class MY_OT_example(bpy.types.Operator):
    bl_idname = "my.example"
    bl_label = "Example"

    count: bpy.props.IntProperty(
        update=lambda self, ctx: print(f"Count: {self.count}"),
        # self is OperatorProperties — only property access is safe
    )
```

**Why:** When an `update` callback is defined on an Operator property, `self` is an `OperatorProperties` instance, NOT the Operator itself. Operator methods like `self.report()`, `self.execute()`, and internal attributes are inaccessible from the callback.

---

## AP-16: Modifying Scene Data in Panel.draw()

**WRONG:**
```python
class MY_PT_panel(bpy.types.Panel):
    def draw(self, context):
        if context.scene.my_settings.needs_update:
            context.scene.my_settings.needs_update = False  # WRONG: write in draw
```

**CORRECT:**
```python
class MY_PT_panel(bpy.types.Panel):
    def draw(self, context):
        layout = self.layout
        settings = context.scene.my_settings
        # READ-ONLY: only use layout.prop(), layout.label(), etc.
        layout.prop(settings, "needs_update")
```

**Why:** `Panel.draw()` is a read-only context. Modifying properties triggers dependency graph updates, which trigger redraws, which call `draw()` again — causing infinite loops or Blender freezing. Panel draw callbacks MUST only read data and construct UI layout.
