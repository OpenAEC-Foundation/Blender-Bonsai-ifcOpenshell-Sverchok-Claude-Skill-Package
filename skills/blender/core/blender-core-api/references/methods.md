# blender-core-api: API Method Reference

Sources: https://docs.blender.org/api/current/info_quickstart.html,
https://docs.blender.org/api/current/info_overview.html,
https://docs.blender.org/api/current/bpy.context.html,
https://docs.blender.org/api/current/bpy.types.Depsgraph.html

---

## bpy.data — BlendData

`bpy.data` is a `BlendData` instance providing access to all ID data blocks in the current blend file.

### Data Collections (all versions: 3.x/4.x/5.x)

| Attribute | Type | Contents |
|-----------|------|----------|
| `bpy.data.objects` | `bpy_prop_collection` of `Object` | All scene objects |
| `bpy.data.meshes` | `bpy_prop_collection` of `Mesh` | Mesh data blocks |
| `bpy.data.materials` | `bpy_prop_collection` of `Material` | Materials |
| `bpy.data.scenes` | `bpy_prop_collection` of `Scene` | All scenes |
| `bpy.data.collections` | `bpy_prop_collection` of `Collection` | Collections |
| `bpy.data.node_groups` | `bpy_prop_collection` of `NodeTree` | All node groups |
| `bpy.data.images` | `bpy_prop_collection` of `Image` | Images |
| `bpy.data.textures` | `bpy_prop_collection` of `Texture` | Textures |
| `bpy.data.lights` | `bpy_prop_collection` of `Light` | Lights (was `lamps` in 2.7x) |
| `bpy.data.cameras` | `bpy_prop_collection` of `Camera` | Cameras |
| `bpy.data.curves` | `bpy_prop_collection` of `Curve` | Curve data |
| `bpy.data.armatures` | `bpy_prop_collection` of `Armature` | Armatures |
| `bpy.data.worlds` | `bpy_prop_collection` of `World` | World settings |
| `bpy.data.actions` | `bpy_prop_collection` of `Action` | Animation actions |
| `bpy.data.fonts` | `bpy_prop_collection` of `VectorFont` | Text fonts |
| `bpy.data.sounds` | `bpy_prop_collection` of `Sound` | Sound files |
| `bpy.data.speakers` | `bpy_prop_collection` of `Speaker` | Speakers |
| `bpy.data.shape_keys` | `bpy_prop_collection` of `Key` | Shape keys |
| `bpy.data.lattices` | `bpy_prop_collection` of `Lattice` | Lattice data |
| `bpy.data.particles` | `bpy_prop_collection` of `ParticleSettings` | Particle systems |
| `bpy.data.workspaces` | `bpy_prop_collection` of `WorkSpace` | Workspaces |
| `bpy.data.screens` | `bpy_prop_collection` of `Screen` | Screen layouts |
| `bpy.data.libraries` | `bpy_prop_collection` of `Library` | Linked libraries |
| `bpy.data.filepath` | `str` | Absolute path of current blend file |
| `bpy.data.is_dirty` | `bool` | True if file has unsaved changes |
| `bpy.data.is_saved` | `bool` | True if file has been saved |
| `bpy.data.version` | `tuple[int, int, int]` | Blender version that saved the file |

### Collection Methods (apply to all bpy.data.* collections)

```python
# Access by name — Blender 3.x/4.x/5.x
obj = bpy.data.objects["Cube"]          # KeyError if not found
obj = bpy.data.objects.get("Cube")      # None if not found (PREFERRED)
obj = bpy.data.objects.get("Cube", default_value)  # With default

# Create new data block — Blender 3.x/4.x/5.x
mesh = bpy.data.meshes.new(name="MyMesh")
# NOTE: Blender may append ".001" if name already exists.
# ALWAYS store the returned reference, not the name.

# Remove data block — Blender 3.x/4.x/5.x
bpy.data.meshes.remove(mesh)
bpy.data.meshes.remove(mesh, do_unlink=True)  # Also unlink from objects

# Iterate — Blender 3.x/4.x/5.x
for obj in bpy.data.objects:
    print(obj.name, obj.type)

# Check existence — Blender 3.x/4.x/5.x
if "Cube" in bpy.data.objects:
    pass

# Count — Blender 3.x/4.x/5.x
count = len(bpy.data.objects)
```

---

## bpy.context — Context Access

`bpy.context` is a read-only snapshot of the current Blender state. Values change depending on the active editor, mode, and selection.

### Global Context Members (always available)

| Attribute | Type | Description |
|-----------|------|-------------|
| `bpy.context.scene` | `Scene` | Active scene |
| `bpy.context.view_layer` | `ViewLayer` | Active view layer |
| `bpy.context.collection` | `Collection` | Active collection |
| `bpy.context.preferences` | `Preferences` | User preferences |
| `bpy.context.window_manager` | `WindowManager` | Window manager |
| `bpy.context.window` | `Window` | Active window |
| `bpy.context.screen` | `Screen` | Active screen layout |
| `bpy.context.workspace` | `WorkSpace` | Active workspace |
| `bpy.context.mode` | `str` | Current mode enum |

### Object Context Members (available in 3D Viewport and most editors)

| Attribute | Type | Description |
|-----------|------|-------------|
| `bpy.context.object` | `Object \| None` | Active object (alias for `active_object`) |
| `bpy.context.active_object` | `Object \| None` | Active object |
| `bpy.context.selected_objects` | `list[Object]` | All selected objects |
| `bpy.context.editable_objects` | `list[Object]` | Selected and editable objects |
| `bpy.context.selectable_objects` | `list[Object]` | All selectable objects in scene |
| `bpy.context.visible_objects` | `list[Object]` | All visible objects in scene |
| `bpy.context.selected_editable_objects` | `list[Object]` | Selected + editable |

### Area/Region Members (available when running inside an editor)

| Attribute | Type | Description |
|-----------|------|-------------|
| `bpy.context.area` | `Area \| None` | Current editor area |
| `bpy.context.region` | `Region \| None` | Current region within area |
| `bpy.context.region_data` | `RegionView3D \| None` | 3D viewport region data |
| `bpy.context.space_data` | `Space \| None` | Editor-specific space data |

### Mode Values for bpy.context.mode

| Mode String | Active When |
|------------|-------------|
| `'OBJECT'` | Object Mode |
| `'EDIT_MESH'` | Edit Mode (mesh) |
| `'EDIT_CURVE'` | Edit Mode (curve) |
| `'EDIT_ARMATURE'` | Edit Mode (armature) |
| `'POSE'` | Pose Mode |
| `'SCULPT'` | Sculpt Mode |
| `'PAINT_VERTEX'` | Vertex Paint |
| `'PAINT_WEIGHT'` | Weight Paint |
| `'PAINT_TEXTURE'` | Texture Paint |
| `'PARTICLE'` | Particle Edit |

### Context.temp_override() — Blender 3.2+, REQUIRED in 4.0+

```python
# Signature (Blender 3.2+/4.x/5.x)
Context.temp_override(**keywords)
# Returns a context manager. Use as: with bpy.context.temp_override(key=value):

# Common override keys:
# window, screen, area, region, space_data
# active_object, object, selected_objects
# edit_object, active_bone, selected_bones

# Example — override active object
with bpy.context.temp_override(active_object=obj, object=obj):
    bpy.ops.object.shade_smooth()

# Example — override area type
with bpy.context.temp_override(area=view3d_area):
    bpy.ops.view3d.snap_cursor_to_center()
```

---

## bpy.ops — Operator Invocation

Operators are accessed via `bpy.ops.<module>.<operator_name>()`. Module and name are lowercased.

### Operator Return Values

| Return Set | Meaning |
|-----------|---------|
| `{'FINISHED'}` | Completed successfully |
| `{'CANCELLED'}` | Cancelled (no undo step created) |
| `{'RUNNING_MODAL'}` | Running modally (receiving events) |
| `{'PASS_THROUGH'}` | Event not consumed |
| `{'INTERFACE'}` | Popup displayed |

### Common Operator Modules

| Module | Operators cover |
|--------|----------------|
| `bpy.ops.object` | Object transformations, visibility, linking |
| `bpy.ops.mesh` | Mesh editing, primitives |
| `bpy.ops.armature` | Armature and bone operations |
| `bpy.ops.transform` | Translation, rotation, scale |
| `bpy.ops.render` | Render and animation |
| `bpy.ops.wm` | File I/O, window management |
| `bpy.ops.view3d` | 3D Viewport operations |
| `bpy.ops.uv` | UV mapping operations |

### Checking poll() Before Calling

```python
# Blender 3.x/4.x/5.x — check if operator can run
if bpy.ops.object.modifier_apply.poll():
    bpy.ops.object.modifier_apply(modifier="Subsurf")
```

---

## bpy.types — Type System

All Blender types are subclasses of `bpy.types.bpy_struct`. Top-level data types are subclasses of `bpy.types.ID`.

### Key Base Types

| Type | Description |
|------|-------------|
| `bpy.types.bpy_struct` | Base for all RNA types |
| `bpy.types.ID` | Base for all ID data blocks (named, reference-counted) |
| `bpy.types.Object` | Scene object (any type) |
| `bpy.types.Mesh` | Mesh data |
| `bpy.types.Material` | Material |
| `bpy.types.Scene` | Scene |
| `bpy.types.Collection` | Collection of objects |
| `bpy.types.Operator` | Base for custom operators |
| `bpy.types.Panel` | Base for custom UI panels |
| `bpy.types.PropertyGroup` | Base for grouped properties |
| `bpy.types.AddonPreferences` | Base for addon preferences |

### ID Data Block Shared Properties

All `bpy.types.ID` subclasses have:

| Property | Type | Description |
|----------|------|-------------|
| `.name` | `str` | Data block name (unique within its type collection) |
| `.name_full` | `str` | Name with library prefix if linked |
| `.users` | `int` | Number of users (references to this data block) |
| `.use_fake_user` | `bool` | Keep data even when users == 0 |
| `.is_library_indirect` | `bool` | True if linked from a library |
| `.library` | `Library \| None` | Source library if linked |
| `.session_uid` | `int` | Unique ID for this session (Blender 4.1+) |

### ID.rename() — Blender 4.3+

```python
# Blender 4.3+ — complex rename behavior
obj.rename("NewName", mode='NEVER')  # Mode: 'NEVER', 'ALWAYS', 'AUTO'
# Direct name assignment: obj.name = "NewName" (unchanged behavior, all versions)
```

### RNA Introspection

```python
# Blender 3.x/4.x/5.x — access type metadata
obj = bpy.context.active_object
rna = obj.bl_rna  # RNA type descriptor

# List all properties
for prop in rna.properties:
    print(f"{prop.identifier} ({prop.type}): {prop.description}")

# Get specific property info
prop = rna.properties['location']
print(prop.array_length)  # 3
print(prop.subtype)       # 'TRANSLATION'
print(prop.unit)          # 'LENGTH'

# List all functions/methods
for func in rna.functions:
    print(f"{func.identifier}({[p.identifier for p in func.parameters]})")
```

---

## bpy.types.Depsgraph — Dependency Graph

The `Depsgraph` class manages evaluation order and caching of scene data.

### Obtaining a Depsgraph

```python
# Blender 3.x/4.x/5.x — from context (most common)
depsgraph = bpy.context.evaluated_depsgraph_get()

# Blender 3.x/4.x/5.x — from scene (for background scripts)
depsgraph = bpy.context.scene.view_layers[0].depsgraph
```

### Depsgraph Properties

| Property | Type | Description |
|----------|------|-------------|
| `depsgraph.scene` | `Scene` | The scene this depsgraph belongs to |
| `depsgraph.scene_eval` | `Scene` | Evaluated scene |
| `depsgraph.view_layer` | `ViewLayer` | Associated view layer |
| `depsgraph.view_layer_eval` | `ViewLayer` | Evaluated view layer |
| `depsgraph.updates` | `DepsgraphUpdate collection` | IDs updated in last evaluation |
| `depsgraph.object_instances` | `DepsgraphObjectInstance iterator` | All object instances |
| `depsgraph.objects` | `Object iterator` | All original objects being evaluated |
| `depsgraph.ids` | `ID iterator` | All ID data blocks being evaluated |

### Depsgraph Methods

```python
# Blender 3.x/4.x/5.x
depsgraph.update()
# Trigger a full re-evaluation of the dependency graph.
# Use after programmatically changing data that needs evaluation.

depsgraph.id_eval_get(id_orig)
# -> ID | None
# Returns the evaluated version of an ID data block.
# Equivalent to: id_orig.evaluated_get(depsgraph)
```

### Object.evaluated_get() — Accessing Evaluated Data

```python
# Blender 3.x/4.x/5.x
obj_original = bpy.data.objects["Cube"]
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj_original.evaluated_get(depsgraph)  # -> Object (evaluated)

# Get evaluated mesh (with all modifiers applied)
mesh_eval = obj_eval.to_mesh()          # -> Mesh (temporary, read-only)
# ... read data ...
obj_eval.to_mesh_clear()               # REQUIRED: free temporary mesh

# Navigate back to original
original = obj_eval.original           # -> Object (original)
```

### DepsgraphObjectInstance Properties

Accessed via `depsgraph.object_instances`:

| Property | Type | Description |
|----------|------|-------------|
| `.object` | `Object` | The evaluated object |
| `.matrix_world` | `Matrix` | World-space transform matrix |
| `.is_instance` | `bool` | True if this is a particle/collection instance |
| `.parent` | `Object \| None` | Parent object creating this instance |
| `.persistent_id` | `tuple[int]` | Stable identifier for this instance |
| `.random_id` | `int` | Random ID for shading (particle variation) |
| `.show_self` | `bool` | Show the instanced object itself |
| `.show_particles` | `bool` | Show particle children |

### DepsgraphUpdate Properties

Accessed via `depsgraph.updates` in handlers:

| Property | Type | Description |
|----------|------|-------------|
| `.id` | `ID` | The data block that was updated |
| `.is_updated_transform` | `bool` | Location/rotation/scale changed |
| `.is_updated_geometry` | `bool` | Mesh/curve geometry changed |
| `.is_updated_shading` | `bool` | Material or shader changed |

---

## bpy.app — Application Info

| Attribute | Type | Description |
|-----------|------|-------------|
| `bpy.app.version` | `tuple[int, int, int]` | Blender version, e.g., `(4, 2, 0)` |
| `bpy.app.version_string` | `str` | Version as string, e.g., `"4.2.0"` |
| `bpy.app.binary_path` | `str` | Path to Blender executable |
| `bpy.app.tempdir` | `str` | Temporary directory path |
| `bpy.app.background` | `bool` | True when running headless (`--background`) |
| `bpy.app.handlers` | `module` | Application event handlers |
| `bpy.app.timers` | `module` | Timer registration |

### bpy.app.handlers

```python
# Available handlers (all versions)
bpy.app.handlers.load_pre           # Before file load
bpy.app.handlers.load_post          # After file load
bpy.app.handlers.save_pre           # Before file save
bpy.app.handlers.save_post          # After file save
bpy.app.handlers.render_pre         # Before render
bpy.app.handlers.render_post        # After render
bpy.app.handlers.depsgraph_update_pre   # Before depsgraph update
bpy.app.handlers.depsgraph_update_post  # After depsgraph update
bpy.app.handlers.frame_change_pre   # Before frame change
bpy.app.handlers.frame_change_post  # After frame change
```

### bpy.app.timers

```python
# Blender 3.x/4.x/5.x — deferred execution on main thread
bpy.app.timers.register(callback, first_interval=0.0)
# callback: callable, returns float (next delay in seconds) or None (stop timer)

bpy.app.timers.unregister(callback)
bpy.app.timers.is_registered(callback)  # -> bool
```
