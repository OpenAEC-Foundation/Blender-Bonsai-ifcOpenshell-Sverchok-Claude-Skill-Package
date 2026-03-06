# blender-errors-data — Error Resolution Examples

## Example 1: ReferenceError After Object Removal

### Problem

```python
# Blender 3.x/4.x/5.x — Crash scenario
import bpy

cube = bpy.data.objects["Cube"]
mesh = cube.data

bpy.data.objects.remove(cube)

# Both lines below raise ReferenceError:
print(cube.name)   # ReferenceError: StructRNA of type Object has been removed
print(mesh.users)  # mesh reference may also be invalid if it was the sole user
```

### Solution

```python
# Blender 3.x/4.x/5.x — Safe removal with cleanup
import bpy

cube = bpy.data.objects.get("Cube")
if cube is None:
    raise RuntimeError("Object 'Cube' not found")

mesh = cube.data
mesh_name = mesh.name  # Store name before removal

bpy.data.objects.remove(cube)
cube = None  # Clear Python reference immediately

# Check if mesh is now orphaned
mesh = bpy.data.meshes.get(mesh_name)
if mesh is not None and mesh.users == 0:
    bpy.data.meshes.remove(mesh)
    mesh = None
```

---

## Example 2: Safe Modal Operator with Undo-Proof References

### Problem

```python
# Blender 3.x/4.x/5.x — Crash scenario
import bpy

class BAD_OT_modal(bpy.types.Operator):
    bl_idname = "bad.modal"
    bl_label = "Bad Modal"

    _obj = None  # Direct bpy reference stored as class variable

    def invoke(self, context, event):
        self._obj = context.active_object
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'LEFTMOUSE':
            self._obj.location.x += 0.1  # CRASH after Ctrl+Z
            return {'RUNNING_MODAL'}
        if event.type == 'ESC':
            return {'CANCELLED'}
        return {'PASS_THROUGH'}
```

### Solution

```python
# Blender 3.x/4.x/5.x — Undo-safe modal operator
import bpy

class SAFE_OT_modal(bpy.types.Operator):
    bl_idname = "safe.modal"
    bl_label = "Safe Modal"

    _target_name: str = ""  # Store name, not reference

    def invoke(self, context, event):
        if context.active_object is None:
            self.report({'WARNING'}, "No active object")
            return {'CANCELLED'}
        self._target_name = context.active_object.name
        context.window_manager.modal_handler_add(self)
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type == 'LEFTMOUSE':
            obj = bpy.data.objects.get(self._target_name)
            if obj is None:
                self.report({'WARNING'}, f"Object '{self._target_name}' was removed")
                return {'CANCELLED'}
            obj.location.x += 0.1  # Safe — fresh reference
            return {'RUNNING_MODAL'}
        if event.type == 'ESC':
            return {'CANCELLED'}
        return {'PASS_THROUGH'}
```

---

## Example 3: SafeObjectRef Wrapper Class

### Pattern

```python
# Blender 3.x/4.x/5.x — Reusable safe reference wrapper
import bpy

class SafeDataRef:
    """Stores data block name and collection, re-fetches on every access.
    Survives undo, redo, and file operations."""

    def __init__(self, data_block, collection):
        """
        Args:
            data_block: A bpy.types.ID instance (object, mesh, material, etc.)
            collection: The bpy.data collection it belongs to (e.g., bpy.data.objects)
        """
        self._name = data_block.name
        self._collection = collection

    @property
    def data(self):
        """Returns the data block, or None if it no longer exists."""
        return self._collection.get(self._name)

    @property
    def name(self):
        return self._name

    def is_valid(self):
        return self._collection.get(self._name) is not None

# Usage:
ref = SafeDataRef(bpy.data.objects["Cube"], bpy.data.objects)

# After any undo/redo/file operation:
obj = ref.data
if obj is not None:
    obj.location.x = 5.0
else:
    print(f"Object '{ref.name}' no longer exists")
```

**Limitation**: If the object is renamed, the reference breaks. For rename-safe references, use a handler that tracks `bpy.msgbus` name change notifications.

---

## Example 4: Name Collision — Capture Return Value

### Problem

```python
# Blender 3.x/4.x/5.x — Name collision mistake
import bpy

bpy.data.meshes.new(name="Floor")
# ... later, or if "Floor" already existed:
bpy.data.meshes.new(name="Floor")

mesh = bpy.data.meshes["Floor"]  # Gets the FIRST one, not the second
# The second mesh is actually named "Floor.001"
```

### Solution

```python
# Blender 3.x/4.x/5.x — ALWAYS capture the return value
import bpy

mesh1 = bpy.data.meshes.new(name="Floor")
print(mesh1.name)  # "Floor"

mesh2 = bpy.data.meshes.new(name="Floor")
print(mesh2.name)  # "Floor.001" — Blender auto-renamed

# Work with the variables, never look up by assumed name
obj1 = bpy.data.objects.new("FloorObj1", mesh1)
obj2 = bpy.data.objects.new("FloorObj2", mesh2)
bpy.context.collection.objects.link(obj1)
bpy.context.collection.objects.link(obj2)
```

---

## Example 5: CollectionProperty Safe Mutation

### Problem

```python
# Blender 3.x/4.x/5.x — Stale pointer after collection mutation
import bpy

class MyItem(bpy.types.PropertyGroup):
    value: bpy.props.IntProperty()

bpy.utils.register_class(MyItem)
bpy.types.Scene.items = bpy.props.CollectionProperty(type=MyItem)

items = bpy.context.scene.items
first = items.add()
first.value = 10

# Add many more items — triggers re-allocation
for i in range(50):
    items.add()

first.value = 20  # CRASH — 'first' pointer is invalid after re-allocation
```

### Solution

```python
# Blender 3.x/4.x/5.x — Safe collection mutation
import bpy

items = bpy.context.scene.items

# Option A: Add all items first, then set values by index
items.add()
for i in range(50):
    items.add()
items[0].value = 10  # Access by index — safe
items[0].value = 20  # Safe — re-fetched from array

# Option B: Track index, not reference
idx = len(items)
items.add()
# Do other add() calls...
items[idx].value = 42  # Access by stored index
```

---

## Example 6: File Load Handler with Safe Re-fetch

### Pattern

```python
# Blender 3.x/4.x/5.x — Safe handler for file operations
import bpy
from bpy.app.handlers import persistent

# Global state stores NAMES, never bpy references
_tracked_objects: list[str] = []

def track_object(obj):
    """Add an object to tracking list by name."""
    _tracked_objects.append(obj.name)

@persistent  # REQUIRED — without this, handler is removed on file load
def on_file_loaded(filepath):
    """Re-validate all tracked objects after file load."""
    valid = []
    for name in _tracked_objects:
        obj = bpy.data.objects.get(name)
        if obj is not None:
            valid.append(name)
        else:
            print(f"Tracked object '{name}' not found in loaded file")
    _tracked_objects.clear()
    _tracked_objects.extend(valid)

@persistent
def on_undo(scene):
    """Re-validate after undo — same invalidation as file load."""
    for name in _tracked_objects:
        obj = bpy.data.objects.get(name)
        if obj is None:
            print(f"Tracked object '{name}' was undone/removed")

bpy.app.handlers.load_post.append(on_file_loaded)
bpy.app.handlers.undo_post.append(on_undo)
```

---

## Example 7: Blender 5.0 IDProperty Migration

### Problem

```python
# Blender 4.x code that breaks in 5.0
import bpy

# Accessing Cycles settings via dict — REMOVED in 5.0
cycles_data = bpy.context.scene["cycles"]

# Resetting RNA property via del — REMOVED in 5.0
del bpy.context.active_object["location"]
```

### Solution

```python
# Blender 3.x/4.x/5.x — Version-safe property access
import bpy

# Version-safe Cycles access
if bpy.app.version >= (5, 0, 0):
    cycles = bpy.context.scene.cycles  # Attribute access only
else:
    cycles = bpy.context.scene.cycles  # Attribute works in all versions

# Version-safe property reset
if bpy.app.version >= (5, 0, 0):
    bpy.context.active_object.property_unset("location")
else:
    # Both work in 3.x/4.x:
    bpy.context.active_object.property_unset("location")  # Preferred
    # del bpy.context.active_object["location"]           # Also works but deprecated

# Custom properties — dict access works in ALL versions
obj = bpy.context.active_object
obj["my_addon_data"] = {"version": 2, "enabled": True}
value = obj.get("my_addon_data", None)
if value is not None:
    del obj["my_addon_data"]  # Works in 3.x/4.x/5.x for custom props
```

---

## Example 8: Batch Object Creation with Safe References

### Pattern

```python
# Blender 3.x/4.x/5.x — Creating multiple objects safely
import bpy

def create_grid(rows: int, cols: int, spacing: float = 2.0):
    """Creates a grid of cubes, returning their names for safe later access."""
    created_names = []

    for row in range(rows):
        for col in range(cols):
            mesh = bpy.data.meshes.new(f"GridMesh_{row}_{col}")
            # Capture return value — name may differ from requested
            obj = bpy.data.objects.new(f"GridCube_{row}_{col}", mesh)

            obj.location = (col * spacing, row * spacing, 0)
            bpy.context.collection.objects.link(obj)

            created_names.append(obj.name)  # Store ACTUAL name

    return created_names

# Usage:
names = create_grid(5, 5)

# Later, safely access created objects:
for name in names:
    obj = bpy.data.objects.get(name)
    if obj is not None:
        obj.scale = (0.5, 0.5, 0.5)
```

---

## Example 9: Detecting Stale References with try/except

### Pattern

```python
# Blender 3.x/4.x/5.x — Graceful stale reference handling
import bpy

def safe_access(reference, fallback_name=None, collection=None):
    """Attempt to access a bpy reference, re-fetch if stale."""
    try:
        # Test if reference is still valid by accessing a property
        _ = reference.name
        return reference
    except ReferenceError:
        if fallback_name and collection:
            return collection.get(fallback_name)
        return None

# Usage:
obj = bpy.data.objects["Cube"]
obj_name = obj.name

# ... undo or other invalidation may occur here ...

obj = safe_access(obj, fallback_name=obj_name, collection=bpy.data.objects)
if obj is not None:
    print(f"Object is valid: {obj.name}")
else:
    print("Object was removed")
```

---

## Official Sources

- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/info_gotchas_internal_data_and_python_objects.html
- https://docs.blender.org/api/current/bpy.types.bpy_prop_collection.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/
