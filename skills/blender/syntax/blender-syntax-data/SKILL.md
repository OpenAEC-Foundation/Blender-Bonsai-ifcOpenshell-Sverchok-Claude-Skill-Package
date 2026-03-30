---
name: blender-syntax-data
description: >
  Use when managing Blender data blocks -- collections, linked libraries, library overrides,
  or the asset system. Prevents the common mistake of not handling fake users when removing
  data blocks, or using append when link+override is needed. Covers bpy.data collections,
  BlendDataLibraries, library overrides, asset system, and data transfer between files.
  Keywords: bpy.data, collections, library overrides, linked library, asset system, append, link, fake user, data block, BlendDataLibraries, organize objects, link from other file, asset browser.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-data

## Quick Reference

### Scope

This skill covers Blender data management:
- **Collections**: hierarchy, visibility, object membership
- **Library linking/appending**: `bpy.data.libraries.load()`
- **Library overrides**: local editable copies of linked data (4.0+)
- **Asset system**: marking, metadata, catalogs, `AssetRepresentation`
- **Data blocks**: creation, removal, fake users, orphan cleanup
- **Data transfer**: moving data between .blend files

### Critical Warnings

**NEVER** rename `scene.collection` — the root Scene Collection is read-only for name and cannot be deleted.

**NEVER** use proxies in Blender 4.0+ — proxies were completely removed. Use library overrides instead.

**NEVER** use `context.asset_file_handle` in Blender 5.0+ — it was removed. Use `context.asset` (`AssetRepresentation`) instead.

**NEVER** assume `bpy.data.libraries.load()` defaults to linking — the default is `link=False` (append). ALWAYS set `link=True` explicitly when linking.

**NEVER** automate asset catalogs via `bpy.ops.asset.catalog_new()` when you need specific names or UUIDs — the operator provides no parameter control. Edit `blender_assets.cats.txt` directly instead.

**ALWAYS** check that returned data blocks are not `None` after `libraries.load()` — missing names silently return `None`.

**ALWAYS** call `bpy.data.meshes.remove()` (or equivalent) to delete data blocks — Python `del` only removes the reference, not the data block.

---

## Decision Tree: Which Data Operation?

```
Need to organize objects in scene?
├── Group objects visually → Collection (§1)
├── Control per-view-layer visibility → LayerCollection (§1)
└── Control render/viewport visibility → Collection properties (§1)

Need data from another .blend file?
├── Read-only reference that stays synced → Link (§2, link=True)
├── Full local editable copy → Append (§2, link=False)
└── Editable copy that tracks source → Library Override (§3)

Need to manage assets?
├── Mark data block as asset → asset_mark() (§4)
├── Set metadata (tags, description) → AssetMetaData (§4)
├── Organize into catalogs → blender_assets.cats.txt (§4)
└── Access active asset in UI → context.asset (4.0+) (§4)

Need to manage data block lifecycle?
├── Create new data block → bpy.data.<collection>.new() (§5)
├── Prevent orphan cleanup → use_fake_user = True (§5)
├── Remove data block → bpy.data.<collection>.remove() (§5)
└── Clean all orphans → bpy.ops.outliner.orphans_purge() (§5)
```

---

## §1 Collections

Collections organize objects in a tree hierarchy. Every scene has exactly one root `scene.collection` (the "Scene Collection").

### Create and Nest Collections

```python
# Blender 3.x/4.x/5.x
import bpy

# Create a collection
building_col = bpy.data.collections.new("Building_A")
bpy.context.scene.collection.children.link(building_col)

# Nest collections
floors_col = bpy.data.collections.new("Floors")
building_col.children.link(floors_col)
```

### Object Membership

Objects can exist in multiple collections simultaneously. ALWAYS unlink from the source collection if you want exclusive membership.

```python
# Blender 3.x/4.x/5.x
obj = bpy.data.objects["Wall_01"]
walls_col.objects.link(obj)                        # Add to target
bpy.context.scene.collection.objects.unlink(obj)   # Remove from source
```

### Collection Visibility

Two independent visibility systems exist:

| Property | Scope | Effect |
|----------|-------|--------|
| `collection.hide_viewport` | Global | Hidden in ALL viewports |
| `collection.hide_render` | Global | Hidden in ALL renders |
| `layer_collection.exclude` | Per view layer | Completely excluded from evaluation |
| `layer_collection.hide_viewport` | Per view layer | Hidden in viewport only |

```python
# Blender 3.x/4.x/5.x: per-view-layer visibility
view_layer = bpy.context.view_layer
layer_col = view_layer.layer_collection.children["Building_A"]
layer_col.exclude = True        # Exclude from this view layer entirely
layer_col.hide_viewport = True  # Hide in viewport, keep in evaluation
```

### Traverse Collection Tree

```python
# Blender 3.x/4.x/5.x
def walk_collection_tree(collection, depth=0):
    print("  " * depth + collection.name)
    for child in collection.children:
        walk_collection_tree(child, depth + 1)
    for obj in collection.objects:
        print("  " * (depth + 1) + f"[{obj.type}] {obj.name}")

walk_collection_tree(bpy.context.scene.collection)
```

### Find LayerCollection by Name

`view_layer.layer_collection` mirrors the collection hierarchy but is NOT indexed by name at the top level. Recursive search is required.

```python
# Blender 3.x/4.x/5.x
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

---

## §2 Library Linking and Appending

### bpy.data.libraries.load(): Recommended Method

```python
# Blender 3.x/4.x/5.x: Link (read-only reference)
filepath = "/path/to/library.blend"

with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
    # data_src.<type> lists available names (strings)
    # data_dst.<type> accepts list of names to load
    data_dst.collections = ["BuildingModule_A"]

# ALWAYS check for None: missing names return None
for col in data_dst.collections:
    if col is not None:
        bpy.context.scene.collection.children.link(col)
```

```python
# Blender 3.x/4.x/5.x: Append (full local copy)
with bpy.data.libraries.load(filepath, link=False) as (data_src, data_dst):
    data_dst.objects = ["Column_Type_A"]
    data_dst.materials = ["Concrete_Exposed"]

for obj in data_dst.objects:
    if obj is not None:
        bpy.context.collection.objects.link(obj)
```

### Operator Method (Alternative)

```python
# Blender 3.x/4.x/5.x: link via operator
bpy.ops.wm.link(
    filepath=filepath + "/Collection/BuildingModule_A",
    directory=filepath + "/Collection/",
    filename="BuildingModule_A"
)

# Append via operator
bpy.ops.wm.append(
    filepath=filepath + "/Object/Column_Type_A",
    directory=filepath + "/Object/",
    filename="Column_Type_A"
)
```

### Key Parameters for libraries.load()

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `filepath` | str | required | Path to .blend file |
| `link` | bool | `False` | `True` = link (read-only), `False` = append (local copy) |
| `relative` | bool | `False` | Store path as relative |
| `assets_only` | bool | `False` | Only load data blocks marked as assets |

### Available Data Types in libraries.load()

Access via `data_src.<type>` and `data_dst.<type>`:
`actions`, `armatures`, `brushes`, `cameras`, `collections`, `curves`, `fonts`, `grease_pencils`, `images`, `lattices`, `lights`, `linestyles`, `masks`, `materials`, `meshes`, `metaballs`, `movieclips`, `node_groups`, `objects`, `paint_curves`, `palettes`, `particles`, `scenes`, `sounds`, `speakers`, `texts`, `textures`, `volumes`, `worlds`, `workspaces`

---

## §3 Library Overrides (4.0+)

Library overrides replace proxies (removed in 4.0). They create local editable copies of linked data that track the source — non-overridden properties sync automatically when the source updates.

### Create Override

```python
# Blender 4.0+: create library override from linked object
import bpy

# Step 1: Link data
with bpy.data.libraries.load("/path/to/library.blend", link=True) as (data_src, data_dst):
    data_dst.collections = ["LinkedBuilding"]

for col in data_dst.collections:
    if col is not None:
        bpy.context.scene.collection.children.link(col)

# Step 2: Select linked object and create override
linked_obj = bpy.data.objects["LinkedBuilding"]
bpy.context.view_layer.objects.active = linked_obj
linked_obj.select_set(True)

# Creates local override: object is now editable
bpy.ops.object.make_override_library()
```

### Inspect Overrides

```python
# Blender 4.0+
obj = bpy.data.objects["LinkedBuilding"]
if obj.override_library:
    ovr = obj.override_library
    print(f"Source: {ovr.reference.name}")
    print(f"Is system override: {ovr.is_system_override}")
    for prop in ovr.properties:
        print(f"  Overridden path: {prop.rna_path}")
        for op in prop.operations:
            print(f"    Operation: {op.operation}")
```

### Override Properties

| Property | Type | Description |
|----------|------|-------------|
| `obj.override_library` | `IDOverrideLibrary` or `None` | Override data (None if not an override) |
| `.reference` | `ID` | The linked data block being overridden |
| `.is_system_override` | `bool` | True if auto-generated (not user-created) |
| `.is_in_hierarchy` | `bool` | True if part of hierarchy override |
| `.hierarchy_root` | `ID` | Root of the override hierarchy |
| `.properties` | collection | Overridden properties list |

### Resync Overrides

```python
# Blender 4.0+: resync after source library changes
bpy.ops.object.override_library_resync()
```

---

## §4 Asset System

### Mark and Configure Assets

```python
# Blender 3.x/4.x/5.x: mark any ID data block as asset
import bpy

obj = bpy.data.objects["Column_Type_A"]
obj.asset_mark()

# Access metadata
meta = obj.asset_data  # bpy.types.AssetMetaData
meta.description = "Reinforced concrete column, 300x300mm"
meta.author = "AEC Library"
meta.license = "CC-BY-4.0"
meta.copyright = "2024 AEC Corp"

# Tags
meta.tags.new("concrete", skip_if_exists=True)
meta.tags.new("structural", skip_if_exists=True)

# Catalog assignment (requires UUID)
meta.catalog_id = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"

# Generate preview
bpy.ops.ed.lib_id_generate_preview(asset_id=obj.name)

# Clear asset status
obj.asset_clear()
```

### Asset Context Access (Version-Critical)

| Version | Access Pattern | Notes |
|---------|---------------|-------|
| Blender 3.x | `context.asset_file_handle` | `AssetHandle` — REMOVED in 5.0 |
| Blender 4.0+ | `context.asset` | `AssetRepresentation` — use this |
| Blender 5.0+ | `context.asset` | `AssetRepresentation` only — `asset_file_handle` gone |

```python
# Blender 4.0+: access active asset in asset browser
asset = bpy.context.asset  # AssetRepresentation or None
if asset:
    print(asset.name)
    print(asset.full_library_path)
    meta = asset.metadata  # AssetMetaData
```

### Asset Catalogs

Catalog management has NO direct data API. Two approaches:

```python
# Approach 1: Operators (limited: no name/UUID control)
bpy.ops.asset.catalog_new()
bpy.ops.asset.catalog_delete()
bpy.ops.asset.catalogs_save()
```

```python
# Approach 2: Direct file edit (RECOMMENDED for automation)
# File: <asset_library_path>/blender_assets.cats.txt
# Format: UUID:path/to/catalog:Simple Name
# Example line: a1b2c3d4-e5f6-7890-abcd-ef1234567890:Structural/Columns:Columns
```

---

## §5 Data Block Lifecycle

### Create Data Blocks

```python
# Blender 3.x/4.x/5.x
mesh = bpy.data.meshes.new("MyMesh")
obj = bpy.data.objects.new("MyObject", mesh)
mat = bpy.data.materials.new("MyMaterial")

# REQUIRED: link object to a collection to make it visible
bpy.context.collection.objects.link(obj)
```

### Fake Users: Prevent Orphan Cleanup

Data blocks with zero users are removed on file save. Use `use_fake_user` to prevent this.

```python
# Blender 3.x/4.x/5.x
mat = bpy.data.materials.new("Template_Material")
mat.use_fake_user = True  # Survives save even with zero real users

# Check user count
print(mat.users)           # Number of real users
print(mat.use_fake_user)   # True if fake user is set
```

### Remove Data Blocks

```python
# Blender 3.x/4.x/5.x
# Remove specific data block
bpy.data.meshes.remove(mesh)
bpy.data.objects.remove(obj)
bpy.data.materials.remove(mat)

# Purge ALL orphaned data blocks
bpy.ops.outliner.orphans_purge(do_local_ids=True, do_linked_ids=True, do_recursive=True)
```

### Batch Data Block Operations

```python
# Blender 3.x/4.x/5.x: iterate and filter data blocks
for mat in bpy.data.materials:
    if mat.users == 0 and not mat.use_fake_user:
        bpy.data.materials.remove(mat)

# SAFER: collect first, then remove (avoids iterator invalidation)
orphans = [m for m in bpy.data.materials if m.users == 0 and not m.use_fake_user]
for mat in orphans:
    bpy.data.materials.remove(mat)
```

### Write Data to External File

```python
# Blender 3.x/4.x/5.x: save specific data blocks to a new .blend
data_blocks = {bpy.data.objects["Column_A"], bpy.data.materials["Concrete"]}
bpy.data.libraries.write("/path/to/output.blend", data_blocks)

# With options
bpy.data.libraries.write(
    "/path/to/output.blend",
    data_blocks,
    path_remap="RELATIVE",  # "NONE", "RELATIVE", "RELATIVE_ALL", "STRIP"
    fake_user=False,
    compress=True
)
```

---

## §6 Version Migration Summary

| Feature | Blender 3.x | Blender 4.0+ | Blender 5.0+ |
|---------|-------------|--------------|--------------|
| Object proxies | Available (deprecated) | REMOVED | REMOVED |
| Library overrides | Available | REQUIRED (replaces proxies) | REQUIRED |
| `context.asset_file_handle` | Available | Deprecated | REMOVED |
| `context.asset` | Not available | Available | REQUIRED |
| `AssetHandle` type | Available | Deprecated | REMOVED |
| `AssetRepresentation` | Not available | Available | REQUIRED |
| `context.active_file` (asset shelf) | N/A | Available | REMOVED |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for collections, libraries, assets, overrides
- [references/examples.md](references/examples.md) — Working code examples for all data management operations
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with data management

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Collection.html
- https://docs.blender.org/api/current/bpy.types.BlendDataLibraries.html
- https://docs.blender.org/api/current/bpy.types.AssetMetaData.html
- https://docs.blender.org/api/current/bpy.types.IDOverrideLibrary.html
- https://docs.blender.org/api/current/bpy.ops.asset.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
