# blender-syntax-data: API Method Reference

Sources:
- https://docs.blender.org/api/current/bpy.types.Collection.html
- https://docs.blender.org/api/current/bpy.types.BlendDataLibraries.html
- https://docs.blender.org/api/current/bpy.types.AssetMetaData.html
- https://docs.blender.org/api/current/bpy.types.IDOverrideLibrary.html
- https://docs.blender.org/api/current/bpy.ops.asset.html

---

## bpy.data.collections — BlendDataCollections

Access: `bpy.data.collections` (read-only `bpy_prop_collection` of `Collection`)

### BlendDataCollections Methods

```python
# Blender 3.x/4.x/5.x
bpy.data.collections.new(name: str) -> Collection
    # Create a new collection data block.
    # name: display name for the collection
    # Returns: the new Collection

bpy.data.collections.remove(collection: Collection, do_unlink: bool = True, do_id_user: bool = True, do_ui_user: bool = True) -> None
    # Remove a collection from the blend file.
    # do_unlink: unlink from all usages before removal
    # do_id_user: decrement user count of associated data
    # do_ui_user: clear UI references

bpy.data.collections.get(name: str, default=None) -> Collection | None
    # Safe access by name.

bpy.data.collections.find(name: str) -> int
    # Return index of named collection, or -1 if not found.

bpy.data.collections.tag(value: bool) -> None
    # Tag all collections (for batch operations).
```

---

## bpy.types.Collection (ID)

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `name` | `str` | R/W | Collection name |
| `children` | `CollectionChildren` | R | Child collections |
| `objects` | `CollectionObjects` | R | Directly contained objects |
| `all_objects` | `bpy_prop_collection` of `Object` | R | All objects including nested collections |
| `hide_viewport` | `bool` | R/W | Global viewport visibility |
| `hide_render` | `bool` | R/W | Global render visibility |
| `hide_select` | `bool` | R/W | Disable selection in viewport |
| `color_tag` | `str` | R/W | Color tag for outliner (`'NONE'`, `'COLOR_01'`..`'COLOR_08'`) |
| `lineart_usage` | `str` | R/W | Line Art usage |
| `use_fake_user` | `bool` | R/W | Fake user to prevent orphan cleanup |
| `users` | `int` | R | Number of users |
| `asset_data` | `AssetMetaData` or `None` | R | Asset metadata (if marked as asset) |
| `library` | `Library` or `None` | R | Library this collection belongs to (if linked) |
| `override_library` | `IDOverrideLibrary` or `None` | R | Override data (if library override) |

### CollectionChildren Methods

```python
# Blender 3.x/4.x/5.x
collection.children.link(child: Collection) -> None
    # Add child collection. Raises RuntimeError if already linked or would create cycle.

collection.children.unlink(child: Collection) -> None
    # Remove child collection.
```

### CollectionObjects Methods

```python
# Blender 3.x/4.x/5.x
collection.objects.link(object: Object) -> None
    # Add object to collection. Object can be in multiple collections.
    # Raises RuntimeError if already in this collection.

collection.objects.unlink(object: Object) -> None
    # Remove object from collection.
    # Raises RuntimeError if object not in this collection.
```

---

## bpy.types.LayerCollection

Access: `bpy.context.view_layer.layer_collection` (root) and `.children` (recursive)

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `name` | `str` | R | Matches linked Collection name |
| `collection` | `Collection` | R | The underlying Collection data block |
| `children` | collection of `LayerCollection` | R | Child layer collections |
| `exclude` | `bool` | R/W | Completely exclude from view layer |
| `hide_viewport` | `bool` | R/W | Hide in viewport for this view layer |
| `holdout` | `bool` | R/W | Mask out in render |
| `indirect_only` | `bool` | R/W | Only use for indirect lighting (Cycles) |
| `is_visible` | `bool` | R | Computed visibility (accounts for parent state) |

---

## bpy.data.libraries — BlendDataLibraries

### load() Context Manager

```python
# Blender 3.x/4.x/5.x
bpy.data.libraries.load(
    filepath: str,
    link: bool = False,
    relative: bool = False,
    assets_only: bool = False
) -> context_manager
    # Returns context manager yielding (data_src, data_dst).
    # data_src: read-only; data_src.<type> returns list of available names (str).
    # data_dst: writable; assign list of names to data_dst.<type> to load them.
    # After context exits, data_dst.<type> contains loaded data blocks (or None for missing).
    #
    # Parameters:
    #   filepath: absolute path to .blend file
    #   link: True = linked reference (read-only), False = appended copy (local)
    #   relative: store filepath as relative
    #   assets_only: only load data blocks marked as assets
```

### write()

```python
# Blender 3.x/4.x/5.x
bpy.data.libraries.write(
    filepath: str,
    datablocks: set[ID],
    path_remap: str = "NONE",
    fake_user: bool = False,
    compress: bool = False
) -> None
    # Write data blocks to a .blend file.
    #
    # Parameters:
    #   filepath: destination .blend file path
    #   datablocks: set of ID data blocks to write
    #   path_remap: "NONE", "RELATIVE", "RELATIVE_ALL", "STRIP"
    #   fake_user: set fake user on written data blocks
    #   compress: compress the output file
```

### Available Data Types for load()

Both `data_src` and `data_dst` expose these attributes:

| Attribute | Data Block Type |
|-----------|----------------|
| `.actions` | `Action` |
| `.armatures` | `Armature` |
| `.brushes` | `Brush` |
| `.cameras` | `Camera` |
| `.collections` | `Collection` |
| `.curves` | `Curve` |
| `.fonts` | `VectorFont` |
| `.grease_pencils` | `GreasePencil` |
| `.images` | `Image` |
| `.lattices` | `Lattice` |
| `.lights` | `Light` |
| `.linestyles` | `FreestyleLineStyle` |
| `.masks` | `Mask` |
| `.materials` | `Material` |
| `.meshes` | `Mesh` |
| `.metaballs` | `MetaBall` |
| `.movieclips` | `MovieClip` |
| `.node_groups` | `NodeTree` |
| `.objects` | `Object` |
| `.paint_curves` | `PaintCurve` |
| `.palettes` | `Palette` |
| `.particles` | `ParticleSettings` |
| `.scenes` | `Scene` |
| `.sounds` | `Sound` |
| `.speakers` | `Speaker` |
| `.texts` | `Text` |
| `.textures` | `Texture` |
| `.volumes` | `Volume` |
| `.worlds` | `World` |
| `.workspaces` | `WorkSpace` |

---

## bpy.types.Library

Access: `bpy.data.libraries` or `data_block.library`

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `name` | `str` | R | Library name |
| `filepath` | `str` | R/W | Path to the linked .blend file |
| `parent` | `Library` or `None` | R | Parent library (for indirect links) |
| `packed_file` | `PackedFile` or `None` | R | Packed file data (if packed) |
| `version` | `tuple[int,int,int]` | R | Blender version that saved the library |
| `users_id` | collection of `ID` | R | All data blocks from this library |

### Methods

```python
# Blender 3.x/4.x/5.x
library.reload() -> None
    # Reload this library and all its data blocks from disk.
```

---

## bpy.types.IDOverrideLibrary

Access: `data_block.override_library` (None if not a library override)

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `reference` | `ID` | R | The linked source data block |
| `hierarchy_root` | `ID` | R | Root of the override hierarchy |
| `is_in_hierarchy` | `bool` | R | Part of a hierarchy override |
| `is_system_override` | `bool` | R | Auto-generated (not user-created) |
| `properties` | collection of `IDOverrideLibraryProperty` | R | Overridden properties |

### IDOverrideLibraryProperty

| Property | Type | Description |
|----------|------|-------------|
| `rna_path` | `str` | RNA path of the overridden property |
| `operations` | collection of `IDOverrideLibraryPropertyOperation` | Operations on this property |

### IDOverrideLibraryPropertyOperation

| Property | Type | Description |
|----------|------|-------------|
| `operation` | `str` | `'NOOP'`, `'REPLACE'`, `'DIFF_ADD'`, `'DIFF_SUB'`, `'FACT_MULTIPLY'`, `'INSERT_AFTER'`, `'INSERT_BEFORE'` |
| `flag` | set | Operation flags |
| `subitem_reference_name` | `str` | Subitem name in reference |
| `subitem_local_name` | `str` | Subitem name in local |
| `subitem_reference_index` | `int` | Subitem index in reference |
| `subitem_local_index` | `int` | Subitem index in local |

---

## Override Operators

```python
# Blender 4.0+
bpy.ops.object.make_override_library(collection: str = "")
    # Create library override for active object or collection.
    # Requires: linked object selected and active.

bpy.ops.object.override_library_resync()
    # Resync overrides after source library changes.

bpy.ops.object.override_library_reset()
    # Reset override to match library reference.
```

---

## bpy.types.AssetMetaData

Access: `data_block.asset_data` (None until `asset_mark()` is called)

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `active_tag` | `int` | R/W | Index of active tag |
| `author` | `str` | R/W | Asset author name |
| `catalog_id` | `str` | R/W | UUID of the catalog this asset belongs to |
| `catalog_simple_name` | `str` | R | Simple (display) name of the assigned catalog |
| `copyright` | `str` | R/W | Copyright notice |
| `description` | `str` | R/W | Asset description |
| `license` | `str` | R/W | License identifier |
| `tags` | `AssetTags` | R | Tag collection |

### AssetTags Methods

```python
# Blender 3.x/4.x/5.x
asset_data.tags.new(name: str, skip_if_exists: bool = False) -> AssetTag
    # Add a tag. skip_if_exists=True prevents duplicates.

asset_data.tags.remove(tag: AssetTag) -> None
    # Remove a specific tag.
```

### Asset ID Methods (on any ID data block)

```python
# Blender 3.x/4.x/5.x
data_block.asset_mark() -> None
    # Mark this data block as an asset. Creates asset_data (AssetMetaData).

data_block.asset_clear() -> None
    # Remove asset status. Clears asset_data.

data_block.asset_generate_preview() -> None
    # Generate asset preview image.
```

---

## Asset Operators

```python
# Blender 3.x/4.x/5.x
bpy.ops.asset.mark() -> None
    # Mark selected data blocks as assets.

bpy.ops.asset.clear(set_fake_user: bool = False) -> None
    # Remove asset status from selected.

bpy.ops.ed.lib_id_generate_preview(asset_id: str = "") -> None
    # Generate preview for asset.

# Blender 4.x/5.x — catalog operators
bpy.ops.asset.catalog_new() -> None
    # Create new catalog (no name/UUID parameter).

bpy.ops.asset.catalog_delete() -> None
    # Delete active catalog.

bpy.ops.asset.catalogs_save() -> None
    # Save catalog changes to disk.
```

---

## bpy.types.AssetRepresentation (4.0+)

Access: `bpy.context.asset`

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `name` | `str` | R | Asset name |
| `full_library_path` | `str` | R | Full path to the library .blend file |
| `full_path` | `str` | R | Full path including data block |
| `id_type` | `str` | R | Type of the data block (`'OBJECT'`, `'MATERIAL'`, etc.) |
| `local_id` | `ID` or `None` | R | Local data block (if asset is local) |
| `metadata` | `AssetMetaData` | R | Asset metadata |

---

## Data Block Common Properties (bpy.types.ID)

All data blocks in `bpy.data.*` inherit from `ID`:

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `name` | `str` | R/W | Data block name |
| `name_full` | `str` | R | Full name including library |
| `users` | `int` | R | Number of users |
| `use_fake_user` | `bool` | R/W | Prevent orphan cleanup |
| `is_library_indirect` | `bool` | R | Indirectly linked from library |
| `library` | `Library` or `None` | R | Source library (None if local) |
| `library_weak_reference` | `LibraryWeakReference` or `None` | R | Weak reference to library |
| `override_library` | `IDOverrideLibrary` or `None` | R | Override data |
| `asset_data` | `AssetMetaData` or `None` | R | Asset metadata |
| `preview` | `ImagePreview` or `None` | R | Preview image |
| `tag` | `bool` | R/W | Tag for batch operations |
| `is_evaluated` | `bool` | R | True if this is an evaluated copy |
| `original` | `ID` | R | Original data block (if evaluated) |
