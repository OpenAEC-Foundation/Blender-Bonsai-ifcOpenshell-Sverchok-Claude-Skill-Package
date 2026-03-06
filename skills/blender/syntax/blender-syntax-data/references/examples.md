# blender-syntax-data: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/bpy.types.Collection.html
- https://docs.blender.org/api/current/bpy.types.BlendDataLibraries.html
- https://docs.blender.org/api/current/bpy.types.AssetMetaData.html
- https://docs.blender.org/api/current/bpy.types.IDOverrideLibrary.html

---

## Example 1: Building a Collection Hierarchy for AEC Projects

```python
# Blender 3.x/4.x/5.x — organize a building model with collections
import bpy

def create_building_hierarchy(building_name):
    """Create a standard AEC collection hierarchy."""
    # Create root building collection
    root = bpy.data.collections.new(building_name)
    bpy.context.scene.collection.children.link(root)

    # Create standard sub-collections
    subcollections = ["Structure", "Architecture", "MEP", "Site"]
    for name in subcollections:
        sub = bpy.data.collections.new(f"{building_name}_{name}")
        root.children.link(sub)

    # Create nested structure collections
    structure = bpy.data.collections.get(f"{building_name}_Structure")
    for element_type in ["Columns", "Beams", "Slabs", "Walls"]:
        element_col = bpy.data.collections.new(f"{building_name}_{element_type}")
        structure.children.link(element_col)

    return root

building = create_building_hierarchy("Tower_A")
print(f"Created hierarchy with {len(building.children)} sub-collections")
```

---

## Example 2: Move Objects Between Collections

```python
# Blender 3.x/4.x/5.x — move objects between collections safely
import bpy

def move_objects_to_collection(objects, target_collection_name):
    """Move objects to target collection, removing from all other collections."""
    target = bpy.data.collections.get(target_collection_name)
    if target is None:
        raise ValueError(f"Collection '{target_collection_name}' not found")

    for obj in objects:
        # Find all collections currently containing this object
        source_collections = [c for c in bpy.data.collections if obj.name in c.objects]

        # Also check scene root collection
        if obj.name in bpy.context.scene.collection.objects:
            source_collections.append(bpy.context.scene.collection)

        # Link to target first (ensures object always has at least one collection)
        if obj.name not in target.objects:
            target.objects.link(obj)

        # Then unlink from all source collections (except target)
        for col in source_collections:
            if col != target:
                col.objects.unlink(obj)

# Usage
selected = [obj for obj in bpy.data.objects if obj.select_get()]
move_objects_to_collection(selected, "Structure_Columns")
```

---

## Example 3: Collection Visibility Management

```python
# Blender 3.x/4.x/5.x — manage visibility per view layer
import bpy

def find_layer_collection(layer_col, name):
    """Recursively find a LayerCollection by name."""
    if layer_col.name == name:
        return layer_col
    for child in layer_col.children:
        result = find_layer_collection(child, name)
        if result:
            return result
    return None

def set_collection_visible(collection_name, visible, view_layer=None):
    """Set collection visibility in a specific view layer."""
    if view_layer is None:
        view_layer = bpy.context.view_layer

    lc = find_layer_collection(view_layer.layer_collection, collection_name)
    if lc is None:
        raise ValueError(f"LayerCollection '{collection_name}' not found")

    lc.exclude = not visible

def isolate_collection(collection_name, view_layer=None):
    """Show only this collection, hide all others at the same level."""
    if view_layer is None:
        view_layer = bpy.context.view_layer

    lc = find_layer_collection(view_layer.layer_collection, collection_name)
    if lc is None:
        raise ValueError(f"LayerCollection '{collection_name}' not found")

    # Hide all siblings
    parent = None
    for child in view_layer.layer_collection.children:
        if find_layer_collection(child, collection_name):
            parent = child
            break

    if parent is None:
        parent = view_layer.layer_collection

    for sibling in parent.children:
        sibling.exclude = (sibling.name != collection_name)

# Usage
set_collection_visible("MEP", False)
isolate_collection("Structure")
```

---

## Example 4: Link Collections from External Library

```python
# Blender 3.x/4.x/5.x — link collections from another .blend file
import bpy

def link_collections_from_library(filepath, collection_names):
    """Link specific collections from an external .blend file."""
    linked = []

    with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
        # Validate requested collections exist
        available = set(data_src.collections)
        for name in collection_names:
            if name not in available:
                print(f"WARNING: Collection '{name}' not found in {filepath}")

        # Request only collections that exist
        data_dst.collections = [n for n in collection_names if n in available]

    # Link loaded collections to scene
    for col in data_dst.collections:
        if col is not None:
            bpy.context.scene.collection.children.link(col)
            linked.append(col.name)
            print(f"Linked: {col.name}")

    return linked

# Usage
linked = link_collections_from_library(
    "/path/to/structural_library.blend",
    ["Foundation_Module", "Column_Grid_A", "Beam_System_01"]
)
print(f"Successfully linked {len(linked)} collections")
```

---

## Example 5: Append and Modify Objects

```python
# Blender 3.x/4.x/5.x — append objects and modify locally
import bpy

def append_and_place(filepath, object_names, target_collection=None, offset=(0, 0, 0)):
    """Append objects from library and place in scene."""
    if target_collection is None:
        target_collection = bpy.context.collection

    with bpy.data.libraries.load(filepath, link=False) as (data_src, data_dst):
        data_dst.objects = [n for n in object_names if n in data_src.objects]

    placed = []
    for i, obj in enumerate(data_dst.objects):
        if obj is not None:
            target_collection.objects.link(obj)
            # Apply offset
            obj.location.x += offset[0] + (i * 3.0)  # Space objects apart
            obj.location.y += offset[1]
            obj.location.z += offset[2]
            placed.append(obj)

    return placed

# Usage
columns = append_and_place(
    "/path/to/library.blend",
    ["Column_300x300", "Column_400x400"],
    offset=(10.0, 0.0, 0.0)
)
```

---

## Example 6: Library Override Workflow

```python
# Blender 4.0+ — link, override, and modify
import bpy

def create_override_from_library(filepath, collection_name):
    """Link a collection and create library overrides for editing."""
    # Step 1: Link the collection
    with bpy.data.libraries.load(filepath, link=True) as (data_src, data_dst):
        if collection_name not in data_src.collections:
            raise ValueError(f"Collection '{collection_name}' not in library")
        data_dst.collections = [collection_name]

    col = data_dst.collections[0]
    if col is None:
        raise RuntimeError(f"Failed to link '{collection_name}'")

    bpy.context.scene.collection.children.link(col)

    # Step 2: Select objects and create overrides
    bpy.ops.object.select_all(action='DESELECT')

    # Find linked objects
    for obj in col.all_objects:
        if obj.library is not None:
            obj.select_set(True)
            bpy.context.view_layer.objects.active = obj

    # Create override for hierarchy
    bpy.ops.object.make_override_library()

    # Step 3: Find overridden objects
    overridden = [
        obj for obj in bpy.data.objects
        if obj.override_library and obj.override_library.reference.name.startswith(collection_name)
    ]

    return overridden

# Usage
overrides = create_override_from_library(
    "/path/to/building_modules.blend",
    "Module_Facade_A"
)
for obj in overrides:
    print(f"Override: {obj.name}, source: {obj.override_library.reference.name}")
```

---

## Example 7: Inspect Override Differences

```python
# Blender 4.0+ — inspect what properties are overridden
import bpy

def report_overrides():
    """Report all library overrides in the file."""
    for obj in bpy.data.objects:
        if obj.override_library is None:
            continue

        ovr = obj.override_library
        print(f"\n=== {obj.name} ===")
        print(f"  Source: {ovr.reference.name}")
        print(f"  Library: {ovr.reference.library.filepath if ovr.reference.library else 'local'}")
        print(f"  System override: {ovr.is_system_override}")

        if ovr.properties:
            print(f"  Overridden properties ({len(ovr.properties)}):")
            for prop in ovr.properties:
                print(f"    {prop.rna_path}")
                for op in prop.operations:
                    print(f"      Operation: {op.operation}")
        else:
            print("  No property overrides")

report_overrides()
```

---

## Example 8: Asset Library Management

```python
# Blender 3.x/4.x/5.x — mark objects as assets with full metadata
import bpy

def mark_as_asset(data_block, description="", author="", tags=None, catalog_id=""):
    """Mark a data block as an asset with metadata."""
    data_block.asset_mark()

    meta = data_block.asset_data
    if description:
        meta.description = description
    if author:
        meta.author = author
    if catalog_id:
        meta.catalog_id = catalog_id
    if tags:
        for tag in tags:
            meta.tags.new(tag, skip_if_exists=True)

    return meta

# Usage — mark multiple objects as assets
asset_configs = [
    ("Column_300x300", "RC column 300x300mm, L=3000mm", ["structural", "concrete", "column"]),
    ("Beam_IPE300", "Steel beam IPE300", ["structural", "steel", "beam"]),
    ("Slab_200mm", "RC slab 200mm thick", ["structural", "concrete", "slab"]),
]

for obj_name, desc, tags in asset_configs:
    obj = bpy.data.objects.get(obj_name)
    if obj:
        mark_as_asset(
            obj,
            description=desc,
            author="AEC Library",
            tags=tags,
            catalog_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890"
        )
        print(f"Marked as asset: {obj_name}")
```

---

## Example 9: Batch Asset Catalog File Generation

```python
# Blender 4.x/5.x — generate blender_assets.cats.txt programmatically
import uuid
from pathlib import Path

def generate_catalog_file(library_path, catalog_tree):
    """Generate blender_assets.cats.txt for an asset library.

    Args:
        library_path: path to the asset library directory
        catalog_tree: dict mapping catalog paths to simple names
            e.g. {"Structural/Columns": "Columns", "Structural/Beams": "Beams"}
    """
    cats_file = Path(library_path) / "blender_assets.cats.txt"

    lines = [
        "# This is an Asset Catalog Definition file for Blender.",
        "#",
        "# Empty lines and lines starting with `#` will be ignored.",
        "# The first non-ignored line should be the version indicator.",
        "# Other lines are of the format \"UUID:catalog/path/for/assets:Simple Catalog Name\"",
        "",
        "VERSION 1",
        "",
    ]

    for catalog_path, simple_name in catalog_tree.items():
        cat_uuid = str(uuid.uuid4())
        lines.append(f"{cat_uuid}:{catalog_path}:{simple_name}")

    cats_file.write_text("\n".join(lines) + "\n")
    print(f"Written catalog file: {cats_file}")

# Usage
generate_catalog_file(
    "/path/to/asset_library",
    {
        "Structural": "Structural",
        "Structural/Columns": "Columns",
        "Structural/Beams": "Beams",
        "Structural/Slabs": "Slabs",
        "Architecture/Walls": "Walls",
        "Architecture/Doors": "Doors",
        "Architecture/Windows": "Windows",
        "MEP/HVAC": "HVAC",
        "MEP/Plumbing": "Plumbing",
    }
)
```

---

## Example 10: Data Block Lifecycle and Cleanup

```python
# Blender 3.x/4.x/5.x — manage data block creation and cleanup
import bpy

def create_material_library(materials_config):
    """Create a set of materials and protect with fake users.

    Args:
        materials_config: list of dicts with 'name' and 'color' keys
    Returns:
        list of created Material data blocks
    """
    created = []
    for config in materials_config:
        mat = bpy.data.materials.new(config["name"])
        mat.use_nodes = True
        mat.use_fake_user = True  # Prevent orphan cleanup

        # Set base color on Principled BSDF
        bsdf = mat.node_tree.nodes.get("Principled BSDF")
        if bsdf:
            bsdf.inputs["Base Color"].default_value = config["color"]

        created.append(mat)

    return created

def cleanup_unused_data():
    """Remove all orphaned data blocks."""
    # Method 1: Targeted cleanup (safer — you control what gets removed)
    orphan_meshes = [m for m in bpy.data.meshes if m.users == 0 and not m.use_fake_user]
    for mesh in orphan_meshes:
        bpy.data.meshes.remove(mesh)

    orphan_mats = [m for m in bpy.data.materials if m.users == 0 and not m.use_fake_user]
    for mat in orphan_mats:
        bpy.data.materials.remove(mat)

    # Method 2: Nuclear option (removes ALL orphans recursively)
    bpy.ops.outliner.orphans_purge(do_local_ids=True, do_linked_ids=True, do_recursive=True)

# Usage
materials = create_material_library([
    {"name": "Concrete_Exposed", "color": (0.6, 0.58, 0.55, 1.0)},
    {"name": "Steel_Painted", "color": (0.15, 0.18, 0.22, 1.0)},
    {"name": "Glass_Clear", "color": (0.85, 0.9, 0.95, 1.0)},
])
print(f"Created {len(materials)} materials with fake users")
```

---

## Example 11: Export Specific Data Blocks to Library File

```python
# Blender 3.x/4.x/5.x — write data blocks to external .blend
import bpy

def export_to_library(filepath, collection_name=None, include_materials=True):
    """Export objects (and optionally materials) to a library .blend file."""
    data_blocks = set()

    if collection_name:
        col = bpy.data.collections.get(collection_name)
        if col is None:
            raise ValueError(f"Collection '{collection_name}' not found")

        # Add collection and all objects
        data_blocks.add(col)
        for obj in col.all_objects:
            data_blocks.add(obj)
            if obj.data:
                data_blocks.add(obj.data)

            # Include materials
            if include_materials:
                for slot in obj.material_slots:
                    if slot.material:
                        data_blocks.add(slot.material)
    else:
        # Export all selected objects
        for obj in bpy.context.selected_objects:
            data_blocks.add(obj)
            if obj.data:
                data_blocks.add(obj.data)

    bpy.data.libraries.write(
        filepath,
        data_blocks,
        path_remap="RELATIVE",
        compress=True
    )
    print(f"Exported {len(data_blocks)} data blocks to {filepath}")

# Usage
export_to_library(
    "/path/to/structural_library.blend",
    collection_name="Structure_Columns",
    include_materials=True
)
```

---

## Example 12: Reload Linked Libraries

```python
# Blender 3.x/4.x/5.x — reload all linked libraries
import bpy

def reload_all_libraries():
    """Reload all linked libraries from disk."""
    for lib in bpy.data.libraries:
        print(f"Reloading: {lib.filepath}")
        lib.reload()

    # Force depsgraph update after reload
    bpy.context.view_layer.update()
    print(f"Reloaded {len(bpy.data.libraries)} libraries")

def list_library_contents():
    """List all linked libraries and their data blocks."""
    for lib in bpy.data.libraries:
        print(f"\nLibrary: {lib.filepath}")
        print(f"  Version: {lib.version}")
        for data_block in lib.users_id:
            override_status = " [OVERRIDE]" if data_block.override_library else ""
            print(f"  {data_block.bl_rna.name}: {data_block.name}{override_status}")

# Usage
list_library_contents()
reload_all_libraries()
```
