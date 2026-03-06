# blender-errors-data — Data Access API Signatures

## bpy.data Collection Methods

All `bpy.data` collections (`bpy.data.objects`, `bpy.data.meshes`, `bpy.data.materials`, etc.) share these methods. Collections are instances of `bpy.types.bpy_prop_collection`.

### Lookup Methods

```python
# Blender 3.x/4.x/5.x

bpy.data.objects[name: str] -> bpy.types.Object
# Returns the data block with exact name match.
# Raises KeyError if not found.
# DANGER: Name may have been auto-renamed (e.g., "Cube.001").

bpy.data.objects[index: int] -> bpy.types.Object
# Returns the data block at the given index.
# Raises IndexError if out of range.
# WARNING: Indices are NOT stable — they change when data blocks are added/removed.

bpy.data.objects.get(name: str, default=None) -> bpy.types.Object | None
# Returns the data block with exact name match, or default if not found.
# PREFERRED over [] access — avoids KeyError on missing data.

bpy.data.objects.find(name: str) -> int
# Returns the index of the data block with the given name.
# Returns -1 if not found.
```

### Creation Methods

```python
# Blender 3.x/4.x/5.x

bpy.data.objects.new(name: str, object_data) -> bpy.types.Object
# Creates a new object. object_data is Mesh, Curve, etc., or None for Empty.
# ALWAYS capture the return value — the actual name may differ from the requested name.
# The object is NOT linked to any scene/collection after creation.

bpy.data.meshes.new(name: str) -> bpy.types.Mesh
# Creates a new empty mesh data block.
# ALWAYS capture the return value.

bpy.data.materials.new(name: str) -> bpy.types.Material
# Creates a new material data block.
# ALWAYS capture the return value.

bpy.data.collections.new(name: str) -> bpy.types.Collection
# Creates a new collection data block.
# ALWAYS capture the return value.

# Pattern applies to ALL bpy.data.<collection>.new() calls.
```

### Removal Methods

```python
# Blender 3.x/4.x/5.x

bpy.data.objects.remove(
    object: bpy.types.Object,
    do_unlink: bool = True,      # Unlink from all collections/scenes
    do_id_user: bool = True,     # Decrement user count on referenced data
    do_ui_user: bool = True      # Clear UI references
) -> None
# Removes the data block from the blend file.
# After this call, ANY Python variable pointing to this data raises ReferenceError.
# Set variable to None after removal to avoid accidental access.

bpy.data.meshes.remove(mesh: bpy.types.Mesh, ...) -> None
bpy.data.materials.remove(material: bpy.types.Material, ...) -> None
# Same signature pattern for all collection types.
```

### Iteration and Membership

```python
# Blender 3.x/4.x/5.x

len(bpy.data.objects) -> int
# Returns number of data blocks in the collection.

for obj in bpy.data.objects:
    # Iterates all data blocks. Safe as long as you don't modify the collection during iteration.
    pass

"Cube" in bpy.data.objects -> bool
# Returns True if a data block with that exact name exists.

list(bpy.data.objects) -> list[bpy.types.Object]
# Creates a Python list snapshot. References in the list are still bpy pointers
# and subject to the same invalidation rules.
```

---

## ID Data Block Properties

All data blocks inherit from `bpy.types.ID` and share these properties.

```python
# Blender 3.x/4.x/5.x

obj.name -> str                    # Read/write. Unique within its collection.
obj.name_full -> str               # Read-only. Includes library path for linked data.
obj.users -> int                   # Read-only. Number of users (references from other data).
obj.use_fake_user -> bool          # Read/write. If True, data block is never auto-purged.
obj.is_library_indirect -> bool    # Read-only. True if indirectly linked from another .blend.
obj.library -> bpy.types.Library   # Read-only. None if local, Library if linked.
obj.is_evaluated -> bool           # Read-only. True if this is a depsgraph-evaluated copy.
obj.original -> bpy.types.ID       # Read-only. Points to original if evaluated, else self.
```

### User Count Management

```python
# Blender 3.x/4.x/5.x

obj.user_clear()
# Sets user count to 0. Data block will be purged on next save/load.
# WARNING: Use only when you explicitly want to orphan the data block.

obj.user_remap(new_id: bpy.types.ID)
# Remaps all users of this data block to point to new_id instead.
# The old data block becomes zero-user and will be purged.

obj.user_of_id(id: bpy.types.ID) -> bool
# Returns True if this data block uses the given ID.
```

---

## CollectionProperty Methods

`bpy.props.CollectionProperty` items stored on operators, panels, or ID types.

```python
# Blender 3.x/4.x/5.x

collection.add() -> PropertyGroup
# Appends a new item to the end. Returns the new item.
# WARNING: May re-allocate the entire underlying C array.
# ALL prior Python references to elements in this collection become INVALID.

collection.remove(index: int) -> None
# Removes the item at the given index.
# WARNING: Same re-allocation risk as add(). Re-fetch all references.

collection.clear() -> None
# Removes all items. ALL element references become invalid.

collection.move(from_index: int, to_index: int) -> None
# Moves an item from one index to another. Does NOT re-allocate.

collection[index: int] -> PropertyGroup
# Access by index. Safe after re-allocation (re-fetches from array).

collection[name: str] -> PropertyGroup
# Access by the item's name property (if it has one).

len(collection) -> int
# Returns number of items.

for item in collection:
    # Iteration. Do NOT add/remove during iteration.
    pass
```

---

## Property Access Methods (Version-Critical)

### Blender 3.x/4.x — Dict-like Access for All Properties

```python
# Blender 3.x/4.x — Dict-like access works for BOTH custom and system properties
obj["my_custom_prop"] = 42          # Set custom property
value = obj["my_custom_prop"]       # Get custom property
del obj["my_custom_prop"]           # Delete custom property

# Also works for RNA-defined (system) properties in 3.x/4.x:
value = scene["cycles"]             # Access Cycles settings (DEPRECATED in 4.x)
del obj["rna_prop"]                 # Reset RNA property to default
```

### Blender 5.0+ — Separated Access

```python
# Blender 5.0+ — Custom properties: dict access STILL works
obj["my_custom_prop"] = 42          # Set — works
value = obj["my_custom_prop"]       # Get — works
del obj["my_custom_prop"]           # Delete — works

# Blender 5.0+ — System (RNA) properties: dict access REMOVED
value = scene.cycles                # Access via attribute — REQUIRED
obj.property_unset("my_rna_prop")   # Reset to default — replaces del obj["prop"]

# Blender 5.0+ — Introspection for migration
sys_props = obj.bl_system_properties_get()  # Access system IDProperties for versioning
```

### Property Existence Checks

```python
# Blender 3.x/4.x/5.x — Check custom property existence
if "my_prop" in obj:                     # True if custom property exists
    pass

if "my_prop" in obj.keys():              # Same, explicit keys
    pass

# Blender 3.x/4.x/5.x — Check RNA property existence
if hasattr(obj, "my_rna_prop"):          # True if RNA property is defined
    pass

# Blender 3.x/4.x/5.x — Get with default
value = obj.get("my_prop", default_value)  # Returns default if not found
```

---

## Data Block Linking and Unlinking

```python
# Blender 3.x/4.x/5.x — Link object to collection (REQUIRED for visibility)
bpy.context.collection.objects.link(obj)
# Raises RuntimeError if object is already linked to this collection.

bpy.context.collection.objects.unlink(obj)
# Removes from collection. Object still exists in bpy.data.objects.

# Safe link pattern:
if obj.name not in bpy.context.collection.objects:
    bpy.context.collection.objects.link(obj)
```

---

## Orphan Data Management

```python
# Blender 3.x/4.x/5.x — Purge orphaned data blocks
bpy.ops.outliner.orphans_purge(
    do_local_ids=True,      # Purge local orphans
    do_linked_ids=True,     # Purge linked orphans
    do_recursive=True       # Repeat until no more orphans
)

# Blender 3.x/4.x/5.x — Manual check
for mesh in bpy.data.meshes:
    if mesh.users == 0 and not mesh.use_fake_user:
        bpy.data.meshes.remove(mesh)
```

---

## Official Sources

- https://docs.blender.org/api/current/bpy.types.bpy_prop_collection.html
- https://docs.blender.org/api/current/bpy.types.ID.html
- https://docs.blender.org/api/current/bpy.props.html
- https://docs.blender.org/api/current/info_gotchas_internal_data_and_python_objects.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/
