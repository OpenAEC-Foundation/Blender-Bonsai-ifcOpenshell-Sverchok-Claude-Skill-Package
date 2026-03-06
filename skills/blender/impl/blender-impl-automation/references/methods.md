# blender-impl-automation — Method Reference

Complete API signatures for I/O operators, render operators, scene assembly, and command-line arguments.

---

## Command-Line Arguments

### Rendering

| Argument | Long Form | Description |
|----------|-----------|-------------|
| `-b` | `--background` | Run without UI (headless mode) |
| `-o` | `--render-output` | Set output path. `#` replaced by frame number |
| `-f` | `--render-frame` | Render specific frame(s). `-f 1`, `-f 1..10`, `-f 1,3,5` |
| `-a` | `--render-anim` | Render full animation (frame_start to frame_end) |
| `-s` | `--frame-start` | Override start frame |
| `-e` | `--frame-end` | Override end frame |
| `-j` | `--frame-jump` | Frame step (render every Nth frame) |
| `-E` | `--engine` | Set render engine: `CYCLES`, `BLENDER_EEVEE_NEXT` (4.0+), `BLENDER_EEVEE` (3.x) |
| `-t` | `--threads` | Thread count (0 = auto-detect) |
| `-F` | `--render-format` | Output format: `PNG`, `JPEG`, `OPEN_EXR`, `TIFF`, etc. |

### Script Execution

| Argument | Long Form | Description |
|----------|-----------|-------------|
| `-P` | `--python` | Run a Python script file |
| | `--python-expr` | Execute a Python expression string |
| | `--python-console` | Start an interactive console |
| | `--python-exit-code N` | Set exit code on Python exception (Blender 3.0+) |
| | `--python-use-system-env` | Use system Python environment |
| `--` | | Separator — all args after this go to `sys.argv` for the script |

### File Operations

| Argument | Description |
|----------|-------------|
| (positional) | Open .blend file: `blender file.blend` |
| `--factory-startup` | Ignore user preferences and addons |
| `--addons` | Comma-separated addon list to enable |
| `-noaudio` | Disable audio system |
| `--debug` | Enable debug mode (verbose logging) |
| `--debug-python` | Enable Python debug logging |
| `--log` | Enable specific log categories |

---

## I/O Operators: OBJ (Blender 4.0+)

### bpy.ops.wm.obj_export()

```python
bpy.ops.wm.obj_export(
    filepath="",                      # str — Output file path
    check_existing=True,              # bool — Check for existing file
    export_animation=False,           # bool — Export animation frames as separate files
    start_frame=-2147483648,          # int — Start frame
    end_frame=2147483647,             # int — End frame
    forward_axis='NEGATIVE_Z',       # enum — Forward axis
    up_axis='Y',                      # enum — Up axis
    global_scale=1.0,                 # float — Global scale factor
    apply_modifiers=True,             # bool — Apply modifiers to mesh
    export_eval_mode='DAG_EVAL_VIEWPORT',  # enum — Evaluation mode
    export_selected_objects=False,    # bool — Export only selected
    export_uv=True,                   # bool — Export UV coordinates
    export_normals=True,              # bool — Export normals
    export_colors=False,              # bool — Export vertex colors
    export_materials=True,            # bool — Export material assignments
    export_triangulated_mesh=False,   # bool — Triangulate mesh
    export_curves_as_nurbs=False,     # bool — Export curves as NURBS
    export_object_groups=False,       # bool — Export object groups
    export_material_groups=False,     # bool — Export material groups
    export_vertex_groups=False,       # bool — Export vertex groups
    export_smooth_groups=False,       # bool — Export smooth groups
    smooth_group_bitflags=False,      # bool — Use bitflags for smooth groups
    filter_glob="*.obj;*.mtl",        # str — File type filter
    path_mode='AUTO',                 # enum — Path mode for textures
)
```

### bpy.ops.wm.obj_import()

```python
bpy.ops.wm.obj_import(
    filepath="",                      # str — Input file path
    forward_axis='NEGATIVE_Z',       # enum — Forward axis
    up_axis='Y',                      # enum — Up axis
    global_scale=1.0,                 # float — Global scale
    clamp_size=0.0,                   # float — Clamp bounding box (0=disabled)
    use_split_objects=True,           # bool — Split by object
    use_split_groups=False,           # bool — Split by group
    import_vertex_groups=False,       # bool — Import vertex groups
    validate_meshes=False,            # bool — Validate mesh data
    filter_glob="*.obj;*.mtl",        # str — File type filter
)
```

---

## I/O Operators: STL (Blender 4.0+)

### bpy.ops.wm.stl_export()

```python
bpy.ops.wm.stl_export(
    filepath="",                      # str — Output file path
    check_existing=True,              # bool — Check existing file
    export_selected_objects=False,    # bool — Export only selected
    global_scale=1.0,                 # float — Global scale factor
    use_scene_unit=False,             # bool — Apply scene unit scale
    ascii_format=False,               # bool — ASCII vs binary STL
    apply_modifiers=True,             # bool — Apply modifiers
    forward_axis='Y',                 # enum — Forward axis
    up_axis='Z',                      # enum — Up axis
    batch_mode='OFF',                 # enum — Batch export: 'OFF', 'OBJECT'
    filter_glob="*.stl",              # str — File type filter
)
```

### bpy.ops.wm.stl_import()

```python
bpy.ops.wm.stl_import(
    filepath="",                      # str — Input file path
    global_scale=1.0,                 # float — Global scale
    use_scene_unit=False,             # bool — Apply scene unit
    use_facet_normal=False,           # bool — Use facet normals
    forward_axis='Y',                 # enum — Forward axis
    up_axis='Z',                      # enum — Up axis
    filter_glob="*.stl",              # str — File type filter
)
```

---

## I/O Operators: FBX (All Versions)

### bpy.ops.export_scene.fbx()

```python
bpy.ops.export_scene.fbx(
    filepath="",                      # str — Output file path
    check_existing=True,              # bool
    use_selection=False,              # bool — Export only selected
    use_visible=False,                # bool — Export only visible
    use_active_collection=False,      # bool — Export active collection
    global_scale=1.0,                 # float
    apply_unit_scale=True,            # bool — Apply unit scale
    apply_scale_options='FBX_SCALE_NONE',  # enum — ALWAYS set explicitly
        # 'FBX_SCALE_NONE' — All local
        # 'FBX_SCALE_UNITS' — FBX units scale
        # 'FBX_SCALE_CUSTOM' — Custom scale
        # 'FBX_SCALE_ALL' — All global (recommended for game engines)
    axis_forward='-Z',                # enum — Forward axis
    axis_up='Y',                      # enum — Up axis
    object_types={'ARMATURE', 'CAMERA', 'EMPTY', 'LIGHT', 'MESH', 'OTHER'},
    use_mesh_modifiers=True,          # bool — Apply modifiers
    use_mesh_modifiers_render=True,   # bool — Use render modifiers
    mesh_smooth_type='OFF',           # enum — 'OFF', 'FACE', 'EDGE'
    use_subsurf=False,                # bool — Export subdivision surface
    use_mesh_edges=False,             # bool — Export loose edges
    use_triangles=False,              # bool — Triangulate
    use_custom_props=False,           # bool — Export custom properties
    add_leaf_bones=True,              # bool — Add leaf bones to armature
    primary_bone_axis='Y',            # enum
    secondary_bone_axis='X',          # enum
    use_armature_deform_only=False,   # bool
    bake_anim=True,                   # bool — Export animation
    bake_anim_use_all_bones=True,     # bool
    bake_anim_use_nla_strips=True,    # bool
    bake_anim_use_all_actions=True,   # bool
    bake_anim_force_startend_keying=True, # bool
    bake_anim_step=1.0,              # float — Frame step
    bake_anim_simplify_factor=1.0,   # float — Keyframe reduction
    embed_textures=False,             # bool — Embed textures in FBX
    batch_mode='OFF',                 # enum — 'OFF', 'SCENE', 'COLLECTION', 'ACTIVE_SCENE_COLLECTION'
    path_mode='AUTO',                 # enum — Texture path mode
)
```

### bpy.ops.import_scene.fbx()

```python
bpy.ops.import_scene.fbx(
    filepath="",                      # str — Input file path
    global_scale=1.0,                 # float
    decal_offset=0.0,                 # float
    use_manual_orientation=False,     # bool
    axis_forward='-Z',                # enum
    axis_up='Y',                      # enum
    use_custom_normals=True,          # bool
    use_image_search=True,            # bool — Search for textures
    use_alpha_decals=False,           # bool
    decal_offset=0.0,                 # float
    use_anim=True,                    # bool — Import animation
    anim_offset=1.0,                  # float
    use_subsurf=False,                # bool
    use_custom_props=True,            # bool — Import custom properties
    use_custom_props_enum_as_string=True,  # bool
    ignore_leaf_bones=False,          # bool
    force_connect_children=False,     # bool
    automatic_bone_orientation=False, # bool
    primary_bone_axis='Y',            # enum
    secondary_bone_axis='X',          # enum
    use_prepost_rot=True,             # bool
)
```

---

## I/O Operators: glTF/GLB (All Versions)

### bpy.ops.export_scene.gltf()

```python
bpy.ops.export_scene.gltf(
    filepath="",                      # str — Output file path
    check_existing=True,              # bool
    export_format='GLB',              # enum — 'GLB', 'GLTF_SEPARATE', 'GLTF_EMBEDDED'
    export_copyright="",              # str — Copyright string
    export_image_format='AUTO',       # enum — 'AUTO', 'JPEG', 'NONE'
    export_texture_dir="",            # str — Texture output directory
    export_keep_originals=False,      # bool — Keep original textures
    export_texcoords=True,            # bool — UV coordinates
    export_normals=True,              # bool
    export_draco_mesh_compression_enable=False,  # bool — Draco compression
    export_draco_mesh_compression_level=6,       # int
    export_tangents=False,            # bool
    export_materials='EXPORT',        # enum — 'EXPORT', 'PLACEHOLDER', 'NONE'
    export_colors=True,               # bool — Vertex colors
    use_selection=False,              # bool — Export only selected
    export_extras=False,              # bool — Export custom properties
    export_yup=True,                  # bool — +Y up axis
    export_apply=False,               # bool — Apply modifiers
    export_animations=True,           # bool
    export_frame_range=True,          # bool — Limit to frame range
    export_frame_step=1,              # int
    export_cameras=False,             # bool
    export_lights=False,              # bool
    export_nla_strips=True,           # bool — Group by NLA strips
)
```

### bpy.ops.import_scene.gltf()

```python
bpy.ops.import_scene.gltf(
    filepath="",                      # str — Input file path
    merge_vertices=False,             # bool
    import_shading='NORMALS',         # enum — 'NORMALS', 'FLAT', 'SMOOTH'
    bone_heuristic='TEMPERANCE',      # enum
    guess_original_bind_pose=True,    # bool
)
```

---

## I/O Operators: USD (2.9+)

### bpy.ops.wm.usd_export()

```python
bpy.ops.wm.usd_export(
    filepath="",                      # str — Output file path (.usdc or .usda)
    check_existing=True,              # bool
    selected_objects_only=False,      # bool
    visible_objects_only=True,        # bool
    export_animation=False,           # bool
    export_hair=False,                # bool
    export_uvmaps=True,               # bool
    export_normals=True,              # bool
    export_materials=True,            # bool
    use_instancing=True,              # bool — Optimize instances
    evaluation_mode='RENDER',         # enum — 'RENDER', 'VIEWPORT'
    generate_preview_surface=True,    # bool — USD preview surface
    export_textures=True,             # bool
    overwrite_textures=False,         # bool
    relative_paths=True,              # bool
)
```

### bpy.ops.wm.usd_import()

```python
bpy.ops.wm.usd_import(
    filepath="",                      # str — Input file path
    scale=1.0,                        # float
    set_frame_range=True,             # bool
    import_cameras=True,              # bool
    import_curves=True,               # bool
    import_lights=True,               # bool
    import_materials=True,            # bool
    import_meshes=True,               # bool
    import_volumes=True,              # bool
    import_subdiv=False,              # bool — Import subdivision scheme
    import_instance_proxies=True,     # bool
    import_visible_only=True,         # bool
    create_collection=False,          # bool
    read_mesh_uvs=True,               # bool
    read_mesh_colors=True,            # bool
)
```

---

## I/O Operators: Alembic (2.78+)

### bpy.ops.wm.alembic_export()

```python
bpy.ops.wm.alembic_export(
    filepath="",                      # str — Output file path (.abc)
    check_existing=True,              # bool
    start=bpy.context.scene.frame_start,  # int — Start frame
    end=bpy.context.scene.frame_end,      # int — End frame
    xsamples=1,                       # int — Transform samples per frame
    gsamples=1,                       # int — Geometry samples per frame
    sh_open=0.0,                      # float — Shutter open time
    sh_close=1.0,                     # float — Shutter close time
    selected=False,                   # bool — Export only selected
    visible_objects_only=True,        # bool
    flatten=False,                    # bool — Flatten hierarchy
    uvs=True,                         # bool — Export UVs
    normals=True,                     # bool — Export normals
    vcolors=False,                    # bool — Export vertex colors
    face_sets=False,                  # bool — Export face sets
    subdiv_schema=False,              # bool — Use subdivision schema
    apply_subdiv=False,               # bool — Apply subdivision modifier
    curves_as_mesh=False,             # bool — Export curves as mesh
    use_instancing=True,              # bool — Export instances
    global_scale=1.0,                 # float
    triangulate=False,                # bool
    export_custom_properties=True,    # bool — IMPORTANT for AEC data
    as_background_job=False,          # bool
)
```

### bpy.ops.wm.alembic_import()

```python
bpy.ops.wm.alembic_import(
    filepath="",                      # str — Input file path
    scale=1.0,                        # float
    set_frame_range=True,             # bool — Adjust scene frame range
    validate_meshes=False,            # bool
    always_add_cache_reader=False,    # bool
    is_sequence=False,                # bool — File is animation sequence
)
```

---

## Render Operators

### bpy.ops.render.render()

```python
bpy.ops.render.render(
    animation=False,    # bool — Render full animation (frame_start to frame_end)
    write_still=False,  # bool — Write rendered image to file
    use_viewport=False, # bool — Use viewport render settings
    layer="",           # str — Render specific render layer
    scene="",           # str — Render specific scene
)
```

**Rules:**
- `animation=True` renders all frames and writes to `scene.render.filepath`
- `write_still=True` renders current frame and writes single image
- NEVER set both `animation=True` and `write_still=True`
- In background mode: ALWAYS works (no UI required)
- Output path `#` symbols replaced by frame number: `/tmp/frame_####` → `/tmp/frame_0001`

---

## Scene Assembly

### bpy.data.libraries.load()

```python
with bpy.data.libraries.load(filepath, link=False, relative=False) as (data_from, data_to):
    # data_from: available data (read-only lists)
    # data_to: data to import (assign lists)
    data_to.objects = data_from.objects          # All objects
    data_to.objects = ["ObjectName"]             # Specific object
    data_to.collections = data_from.collections  # All collections
    data_to.materials = data_from.materials      # All materials
```

**Args:**
- `filepath`: str — Path to .blend file
- `link`: bool — True=link (reference), False=append (full copy)
- `relative`: bool — Use relative paths for linked libraries

**Available data categories:** `objects`, `meshes`, `materials`, `textures`, `images`, `collections`, `node_groups`, `worlds`, `scenes`, `actions`

### bpy.ops.wm.append()

```python
bpy.ops.wm.append(
    filepath="",     # str — Full path: "/path/file.blend/Object/Name"
    directory="",    # str — Data block dir: "/path/file.blend/Object"
    filename="",     # str — Data block name
    link=False,      # bool — Link instead of append
    autoselect=True, # bool — Select imported objects
    active_collection=True,  # bool — Put in active collection
    instance_collections=False,  # bool — Instance linked collections
)
```

### bpy.ops.wm.link()

Same parameters as `append()` but with `link=True` default behavior.

---

## bpy.app Attributes for Automation

| Attribute | Type | Description |
|-----------|------|-------------|
| `bpy.app.background` | bool | True when running in `--background` mode |
| `bpy.app.binary_path` | str | Path to Blender executable |
| `bpy.app.version` | tuple(int,int,int) | Blender version, e.g., `(4, 2, 0)` |
| `bpy.app.version_string` | str | Human-readable version, e.g., `"4.2.0"` |
| `bpy.app.version_file` | tuple | Version of currently open .blend file |
| `bpy.app.tempdir` | str | Blender's temp directory |
| `bpy.app.is_job_running('RENDER')` | bool | Check if render job is active |

---

## Sources

- https://docs.blender.org/api/current/bpy.ops.wm.html
- https://docs.blender.org/api/current/bpy.ops.export_scene.html
- https://docs.blender.org/api/current/bpy.ops.import_scene.html
- https://docs.blender.org/api/current/bpy.ops.render.html
- https://docs.blender.org/api/current/bpy.data.html
- https://docs.blender.org/api/current/bpy.app.html
- https://docs.blender.org/manual/en/latest/advanced/command_line/arguments.html
