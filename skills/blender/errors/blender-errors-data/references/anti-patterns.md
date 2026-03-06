# blender-errors-data — Anti-Patterns

## Anti-Pattern 1: Caching bpy.data References in Global Variables

### The Mistake

```python
# Blender 3.x/4.x/5.x — WRONG
import bpy

# Module-level cache of Blender data
_cached_objects = {}

def cache_active_object():
    obj = bpy.context.active_object
    _cached_objects["main"] = obj  # Stored as direct bpy reference

def use_cached_object():
    obj = _cached_objects["main"]
    obj.location.x += 1  # CRASH after undo, file load, or object deletion
```

### Why It Fails

Global variables persist across the entire Python session. Blender data pointers are invalidated by undo, redo, file load, and file revert. The Python variable survives, but the C data it points to is freed. Accessing it causes `ReferenceError` or a segfault.

### The Fix

```python
# Blender 3.x/4.x/5.x — CORRECT
import bpy

_cached_names = {}

def cache_active_object():
    obj = bpy.context.active_object
    if obj is not None:
        _cached_names["main"] = obj.name  # Store NAME, not reference

def use_cached_object():
    name = _cached_names.get("main")
    if name is None:
        return
    obj = bpy.data.objects.get(name)
    if obj is None:
        del _cached_names["main"]  # Clean up stale entry
        return
    obj.location.x += 1  # Safe — freshly fetched
```

---

## Anti-Pattern 2: Looking Up by Name After new() Instead of Capturing Return Value

### The Mistake

```python
# Blender 3.x/4.x/5.x — WRONG
import bpy

bpy.data.materials.new(name="Glass")
mat = bpy.data.materials["Glass"]  # May get wrong material if "Glass" already existed

# The new material could be "Glass.001" — this line gets the OLD "Glass"
mat.use_nodes = True  # Modifies the WRONG material
```

### Why It Fails

`bpy.data.<collection>.new(name)` does NOT guarantee the returned data block has the requested name. If a data block with that name already exists, Blender appends `.001`, `.002`, etc. to the NEW data block. Looking up by the original name retrieves the PRE-EXISTING data block, not the one you just created.

### The Fix

```python
# Blender 3.x/4.x/5.x — CORRECT
import bpy

mat = bpy.data.materials.new(name="Glass")  # Capture return value
mat.use_nodes = True  # ALWAYS correct — operates on the newly created material
print(mat.name)  # Shows actual name: "Glass" or "Glass.001"
```

---

## Anti-Pattern 3: Holding CollectionProperty Element References Across Mutations

### The Mistake

```python
# Blender 3.x/4.x/5.x — WRONG
import bpy

items = bpy.context.scene.my_list  # CollectionProperty
refs = []

for i in range(10):
    item = items.add()
    item.name = f"Item_{i}"
    refs.append(item)  # Stores pointer to C array element

# Later:
for ref in refs:
    ref.value = 42  # CRASH — all pointers invalidated by add() re-allocations
```

### Why It Fails

`CollectionProperty` stores data in a contiguous C array. When `.add()` or `.remove()` is called, Blender may re-allocate the entire array to a new memory location. All Python references to elements obtained before the re-allocation point to freed memory.

### The Fix

```python
# Blender 3.x/4.x/5.x — CORRECT: use indices, not references
import bpy

items = bpy.context.scene.my_list

# Create all items first
for i in range(10):
    item = items.add()
    item.name = f"Item_{i}"

# Then access by index
for i in range(len(items)):
    items[i].value = 42  # Safe — index-based access re-fetches from array
```

---

## Anti-Pattern 4: Using del obj["prop"] for RNA Properties in Blender 5.0+

### The Mistake

```python
# Blender 5.0+ — WRONG
import bpy

# Trying to reset an RNA-defined property using dict deletion
del bpy.context.active_object["location"]      # Fails — not a custom property
del bpy.context.scene["cycles"]                 # Fails — system property

# Trying to read system properties via dict access
render_engine = bpy.context.scene["render"]     # KeyError in 5.0+
```

### Why It Fails

Blender 5.0 separates system-defined IDProperties (RNA properties registered via `bpy.props` or built into Blender's C code) from user-defined custom properties. Dict-like access (`obj["key"]`) only reaches custom properties. System properties are accessible exclusively through attribute access (`obj.key`).

### The Fix

```python
# Blender 5.0+ — CORRECT
import bpy

# Reset RNA property to its default value
bpy.context.active_object.property_unset("location")

# Access system properties via attributes
render = bpy.context.scene.render
cycles = bpy.context.scene.cycles

# Version-safe pattern:
def reset_property(data_block, prop_name):
    """Reset a property to its default value across all Blender versions."""
    data_block.property_unset(prop_name)  # Works in 3.x/4.x/5.x
```

---

## Anti-Pattern 5: Iterating bpy.data Collections While Modifying Them

### The Mistake

```python
# Blender 3.x/4.x/5.x — WRONG
import bpy

# Remove all meshes with no users
for mesh in bpy.data.meshes:
    if mesh.users == 0:
        bpy.data.meshes.remove(mesh)  # Modifies collection during iteration — CRASH
```

### Why It Fails

Removing an item from a `bpy.data` collection while iterating over it modifies the underlying C array. The iterator's internal index becomes invalid, causing skipped items, double-processing, or a crash.

### The Fix

```python
# Blender 3.x/4.x/5.x — CORRECT: snapshot to list first
import bpy

# Option A: Collect targets first, then remove
orphans = [m for m in bpy.data.meshes if m.users == 0]
for mesh in orphans:
    bpy.data.meshes.remove(mesh)

# Option B: Iterate in reverse by index
for i in range(len(bpy.data.meshes) - 1, -1, -1):
    mesh = bpy.data.meshes[i]
    if mesh.users == 0:
        bpy.data.meshes.remove(mesh)

# Option C: Use built-in purge operator
bpy.ops.outliner.orphans_purge(do_local_ids=True, do_linked_ids=True, do_recursive=True)
```

---

## Anti-Pattern 6: Assuming Object Names Are Stable After Rename Operations

### The Mistake

```python
# Blender 3.x/4.x/5.x — WRONG
import bpy

# Store object name as identifier
target_name = "Wall_01"
obj = bpy.data.objects.get(target_name)
obj.name = "Wall_Front"  # Rename succeeds

# But if "Wall_Front" already existed, the OTHER object was renamed to "Wall_Front.001"
# Code elsewhere that stores "Wall_Front" as a reference to the other object now breaks

# Later in another part of the code:
other_obj = bpy.data.objects.get("Wall_Front")
# This returns the RENAMED object, not the original "Wall_Front" which is now "Wall_Front.001"
```

### Why It Fails

When you rename a data block to a name that already exists, Blender renames the EXISTING data block by appending `.001` to make room. This cascade affects ANY code that stored the old name of the displaced data block.

### The Fix

```python
# Blender 3.x/4.x/5.x — CORRECT: check for collision before rename
import bpy

def safe_rename(obj, new_name):
    """Rename object, returning the actual assigned name."""
    if new_name in bpy.data.objects and bpy.data.objects[new_name] != obj:
        print(f"WARNING: '{new_name}' already exists, "
              f"existing object will be renamed to '{new_name}.001'")
    obj.name = new_name
    return obj.name  # Return ACTUAL name (may differ if collision occurred)

# Or avoid collisions entirely:
def unique_rename(obj, desired_name):
    """Rename with guaranteed unique name, no collisions."""
    if desired_name not in bpy.data.objects:
        obj.name = desired_name
    else:
        # Find available name
        i = 1
        while f"{desired_name}.{i:03d}" in bpy.data.objects:
            i += 1
        obj.name = f"{desired_name}.{i:03d}"
    return obj.name
```

---

## Anti-Pattern 7: Accessing Mesh Data in Edit Mode via mesh.vertices

### The Mistake

```python
# Blender 3.x/4.x/5.x — WRONG
import bpy

bpy.ops.object.mode_set(mode='EDIT')
mesh = bpy.context.active_object.data
for v in mesh.vertices:
    print(v.co)  # Data is NOT synchronized — values are stale or access crashes
```

### Why It Fails

In Edit Mode, the canonical mesh data lives in the BMesh edit structure, not in `mesh.vertices`. The `mesh.vertices` array is a snapshot from when Edit Mode was entered and is not updated until you return to Object Mode. Modifying it has no effect; reading it returns stale data.

### The Fix

```python
# Blender 3.x/4.x/5.x — CORRECT: use BMesh in Edit Mode
import bpy
import bmesh

bpy.ops.object.mode_set(mode='EDIT')
bm = bmesh.from_edit_mesh(bpy.context.edit_object.data)
for v in bm.verts:
    print(v.co)  # Live data — safe and current

# After BMesh modifications:
bmesh.update_edit_mesh(bpy.context.edit_object.data)

# OR: switch to Object Mode first
bpy.ops.object.mode_set(mode='OBJECT')
mesh = bpy.context.active_object.data
for v in mesh.vertices:
    print(v.co)  # Now synchronized — safe
```

---

## Summary Table

| # | Anti-Pattern | Error Type | Fix |
|---|-------------|-----------|-----|
| 1 | Caching bpy references in globals | `ReferenceError` / crash | Store names, re-fetch on access |
| 2 | Looking up by name after `new()` | Wrong data block selected | Capture return value of `new()` |
| 3 | Holding CollectionProperty refs across mutations | Crash / segfault | Use index-based access after mutations |
| 4 | `del obj["prop"]` for RNA props in 5.0+ | `KeyError` / no effect | Use `property_unset()` |
| 5 | Iterating collection while removing | Crash / skipped items | Snapshot to list first, then remove |
| 6 | Assuming names are stable after rename | Wrong object referenced | Check collision, store actual name |
| 7 | Reading `mesh.vertices` in Edit Mode | Stale / incorrect data | Use `bmesh.from_edit_mesh()` or switch to Object Mode |

---

## Official Sources

- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/info_gotchas_internal_data_and_python_objects.html
- https://docs.blender.org/api/current/info_gotchas_meshes.html
- https://developer.blender.org/docs/release_notes/5.0/python_api/
