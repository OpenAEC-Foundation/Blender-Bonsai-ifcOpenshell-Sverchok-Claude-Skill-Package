# blender-syntax-data: Anti-Patterns

These are confirmed error patterns for Blender data management. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://docs.blender.org/api/current/info_gotcha.html
- https://docs.blender.org/api/current/bpy.types.Collection.html
- https://docs.blender.org/api/current/bpy.types.BlendDataLibraries.html
- https://docs.blender.org/api/current/bpy.types.IDOverrideLibrary.html
- supplementary-blender-gaps.md §6.7

---

## AP-001: Renaming the Scene Collection

**WHY this is wrong**: The root Scene Collection (`bpy.context.scene.collection`) is a special system object. Its name cannot be changed and it cannot be deleted. Attempting to rename it silently fails or raises an error.

```python
# WRONG — scene.collection name is read-only
bpy.context.scene.collection.name = "My Project"
```

```python
# CORRECT — create a child collection for organization
root = bpy.data.collections.new("My Project")
bpy.context.scene.collection.children.link(root)
# Organize everything under root
```

NEVER attempt to rename or delete `scene.collection`. ALWAYS create child collections for organization.

---

## AP-002: Forgetting link=True When Linking

**WHY this is wrong**: `bpy.data.libraries.load()` defaults to `link=False`, which **appends** (creates a local copy). Omitting `link=True` gives you an editable copy instead of a linked reference. This is the opposite of what "link" means semantically.

```python
# WRONG — this APPENDS, not links (link=False is default)
with bpy.data.libraries.load(filepath) as (data_src, data_dst):
    data_dst.collections = ["Module_A"]
# data_dst.collections[0] is a LOCAL copy, not a linked reference
```

```python
# CORRECT — explicitly set link=True
with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
    data_dst.collections = ["Module_A"]
# data_dst.collections[0] is a LINKED reference
```

ALWAYS set `link=True` explicitly when linking. NEVER rely on the default for linking operations.

---

## AP-003: Not Checking for None After libraries.load()

**WHY this is wrong**: If a requested name does not exist in the source file, `libraries.load()` silently returns `None` in the corresponding position. Attempting to use `None` as a data block causes `AttributeError` or `TypeError`.

```python
# WRONG — no None check
with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
    data_dst.collections = ["NonExistent_Collection"]

col = data_dst.collections[0]
bpy.context.scene.collection.children.link(col)  # TypeError: expected Collection, got NoneType
```

```python
# CORRECT — always check for None
with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
    data_dst.collections = ["NonExistent_Collection"]

for col in data_dst.collections:
    if col is not None:
        bpy.context.scene.collection.children.link(col)
    else:
        print("WARNING: Collection not found in library")
```

ALWAYS check that returned data blocks are not `None` after `libraries.load()`.

---

## AP-004: Using Proxies in Blender 4.0+

**WHY this is wrong**: Object proxies (`bpy.ops.object.proxy_make()`) were the legacy method for making linked objects editable. Proxies were deprecated in 3.x and **completely removed in Blender 4.0**. Calling proxy-related APIs in 4.0+ raises `AttributeError`.

```python
# WRONG — Blender 3.x only, REMOVED in 4.0
bpy.ops.object.proxy_make(object="LinkedArmature")
```

```python
# CORRECT — Blender 4.0+
bpy.ops.object.make_override_library()
```

NEVER use proxy APIs in Blender 4.0+ code. ALWAYS use library overrides (`make_override_library()`).

---

## AP-005: Using asset_file_handle in Blender 5.0+

**WHY this is wrong**: `bpy.context.asset_file_handle` (returning `AssetHandle`) was the Blender 3.x API for accessing the active asset. It was deprecated in 4.0 and **removed in Blender 5.0**. Using it in 5.0+ raises `AttributeError`.

```python
# WRONG — Blender 3.x only, REMOVED in 5.0
asset = bpy.context.asset_file_handle  # AttributeError in 5.0+
```

```python
# CORRECT — Blender 4.0+/5.x
asset = bpy.context.asset  # AssetRepresentation
if asset:
    print(asset.name)
    meta = asset.metadata
```

ALWAYS use `context.asset` (returning `AssetRepresentation`) in Blender 4.0+. NEVER use `asset_file_handle` in any new code.

---

## AP-006: Automating Catalogs via Operators

**WHY this is wrong**: `bpy.ops.asset.catalog_new()` creates a new catalog but provides **no parameters** to set the catalog name, path, or UUID. The operator only works interactively. For automation, the catalog ends up with a default name that cannot be controlled.

```python
# WRONG — no way to set name/UUID
bpy.ops.asset.catalog_new()  # Creates catalog with uncontrollable name
```

```python
# CORRECT — edit the catalog file directly
from pathlib import Path
import uuid

cats_file = Path("/path/to/asset_library") / "blender_assets.cats.txt"
with open(cats_file, "a") as f:
    cat_uuid = str(uuid.uuid4())
    f.write(f"{cat_uuid}:Structural/Columns:Columns\n")
```

NEVER use `bpy.ops.asset.catalog_new()` for programmatic catalog creation. ALWAYS edit `blender_assets.cats.txt` directly for automation.

---

## AP-007: Object in Multiple Collections Without Unlinking

**WHY this is wrong**: When moving an object from one collection to another, calling `link()` on the target without `unlink()` on the source leaves the object in **both** collections. This causes confusion in the Outliner and potential issues with visibility and selection.

```python
# WRONG — object ends up in BOTH collections
new_col.objects.link(obj)
# obj is now in old_col AND new_col
```

```python
# CORRECT — unlink from source after linking to target
new_col.objects.link(obj)
old_col.objects.unlink(obj)
```

ALWAYS unlink objects from the source collection after linking to the target when moving (not duplicating) objects between collections.

---

## AP-008: Iterating and Removing Data Blocks Simultaneously

**WHY this is wrong**: Removing items from a `bpy_prop_collection` while iterating over it invalidates the iterator. This can cause skipped items, crashes, or `RuntimeError`.

```python
# WRONG — iterator invalidated by removal
for mat in bpy.data.materials:
    if mat.users == 0:
        bpy.data.materials.remove(mat)  # Invalidates iterator
```

```python
# CORRECT — collect first, then remove
orphans = [m for m in bpy.data.materials if m.users == 0 and not m.use_fake_user]
for mat in orphans:
    bpy.data.materials.remove(mat)
```

NEVER remove data blocks while iterating over their collection. ALWAYS collect references into a list first, then remove in a separate loop.

---

## AP-009: Using Python del to Remove Data Blocks

**WHY this is wrong**: `del variable` only removes the Python reference to the data block. The actual Blender data block remains in `bpy.data` and continues to consume memory. It becomes an orphan and is only cleaned up on file save (if it has zero users and no fake user).

```python
# WRONG — only removes Python variable, not the data block
mesh = bpy.data.meshes.new("TempMesh")
del mesh  # mesh data block STILL exists in bpy.data.meshes
```

```python
# CORRECT — use remove() to actually delete the data block
mesh = bpy.data.meshes.new("TempMesh")
bpy.data.meshes.remove(mesh)  # Data block is actually removed
```

ALWAYS use `bpy.data.<collection>.remove()` to delete data blocks. NEVER rely on Python `del` for data block cleanup.

---

## AP-010: Linking Objects Without Linking to a Collection

**WHY this is wrong**: Creating an object with `bpy.data.objects.new()` only creates the data block. Without linking to a collection, the object is invisible in the viewport, excluded from renders, and will be treated as an orphan on save.

```python
# WRONG — object created but never visible
mesh = bpy.data.meshes.new("MyMesh")
obj = bpy.data.objects.new("MyObject", mesh)
# Object exists in bpy.data.objects but is invisible
```

```python
# CORRECT — link to collection immediately after creation
mesh = bpy.data.meshes.new("MyMesh")
obj = bpy.data.objects.new("MyObject", mesh)
bpy.context.collection.objects.link(obj)  # NOW visible
```

ALWAYS link new objects to a collection immediately after creation. Objects without collection links are invisible orphans.

---

## AP-011: Assuming LayerCollection Has Dict-Style Access

**WHY this is wrong**: `view_layer.layer_collection` is a tree structure, not a flat dictionary. There is no `layer_collection["name"]` access at the root level. Only `.children` provides named access for direct children. Nested collections require recursive search.

```python
# WRONG — no dict access on root layer_collection
lc = bpy.context.view_layer.layer_collection["Building_A"]  # TypeError
```

```python
# CORRECT — recursive search
def find_layer_collection(layer_col, name):
    if layer_col.name == name:
        return layer_col
    for child in layer_col.children:
        result = find_layer_collection(child, name)
        if result:
            return result
    return None

lc = find_layer_collection(bpy.context.view_layer.layer_collection, "Building_A")
```

ALWAYS use recursive search to find `LayerCollection` by name. NEVER assume flat dict-style access on `view_layer.layer_collection`.

---

## AP-012: Forgetting to Set fake_user on Template Data Blocks

**WHY this is wrong**: Data blocks with zero users (no objects reference them) are automatically purged when saving the .blend file. Template materials, meshes, or node groups that are meant to be used later will be lost.

```python
# WRONG — material disappears after save if nothing uses it
template_mat = bpy.data.materials.new("Template_Concrete")
# ... configure material ...
# If no object has this material assigned, it gets purged on save
```

```python
# CORRECT — set fake_user to preserve
template_mat = bpy.data.materials.new("Template_Concrete")
template_mat.use_fake_user = True
# Material survives save even with zero real users
```

ALWAYS set `use_fake_user = True` on data blocks that should persist without active users (templates, libraries, reusable assets).

---

## AP-013: Overriding System Override Properties

**WHY this is wrong**: Library overrides include both user-created overrides and system overrides (auto-generated for internal bookkeeping). Modifying system override properties can corrupt the override hierarchy and cause sync failures.

```python
# WRONG — modifying without checking override type
obj = bpy.data.objects["OverriddenObj"]
if obj.override_library:
    # Blindly iterating and modifying all override properties
    for prop in obj.override_library.properties:
        # Modifying system-generated properties corrupts the override
        pass
```

```python
# CORRECT — check is_system_override before modifying
obj = bpy.data.objects["OverriddenObj"]
if obj.override_library and not obj.override_library.is_system_override:
    # Safe to inspect and work with user overrides
    for prop in obj.override_library.properties:
        print(f"User override: {prop.rna_path}")
```

ALWAYS check `is_system_override` before programmatically working with override properties. NEVER modify system overrides.

---

## AP-014: Using context.active_file for Asset Shelf in 5.0+

**WHY this is wrong**: `bpy.context.active_file` was used in Blender 4.x for the asset shelf context. It was **removed in Blender 5.0**. The unified access point is `context.asset`.

```python
# WRONG — Blender 4.x only, REMOVED in 5.0
active = bpy.context.active_file  # AttributeError in 5.0+
```

```python
# CORRECT — Blender 4.0+/5.x
asset = bpy.context.asset  # Works everywhere 4.0+
```

ALWAYS use `context.asset` for asset access in Blender 4.0+. NEVER use `context.active_file` in code targeting Blender 5.0+.
