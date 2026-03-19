---
name: blender-core-api
description: >
  Use when writing Blender Python scripts that access bpy module structure, RNA data,
  the context system, dependency graph, or operator invocation. Prevents the common mistake
  of accessing context attributes outside their valid scope or calling operators without
  checking poll(). Covers bpy.data, bpy.context, bpy.ops, depsgraph evaluation, and
  RNA property access patterns across Blender 3.x/4.x/5.x.
  Keywords: bpy, Blender Python, RNA, context, depsgraph, operator, bpy.data, bpy.ops.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-core-api

## Quick Reference

### bpy Module Hierarchy

| Module | Purpose | Example |
|--------|---------|---------|
| `bpy.data` | All blend-file data (ID data blocks) | `bpy.data.objects["Cube"]` |
| `bpy.context` | Current state (active object, mode, selection) | `bpy.context.active_object` |
| `bpy.ops` | Operators with undo support | `bpy.ops.mesh.primitive_cube_add()` |
| `bpy.types` | Type definitions for all Blender structs | `class MyOp(bpy.types.Operator):` |
| `bpy.props` | Property definitions for custom data | `bpy.props.FloatProperty()` |
| `bpy.utils` | Utility functions (registration, paths) | `bpy.utils.register_class(cls)` |
| `bpy.app` | Application info (version, handlers) | `bpy.app.version` |
| `bpy.path` | Blender-aware path utilities | `bpy.path.abspath("//file.png")` |
| `bpy.msgbus` | Message bus for property notifications | `bpy.msgbus.subscribe_rna()` |

### bpy.data Collections (all versions)

```
bpy.data
├── .objects      ├── .meshes       ├── .materials    ├── .scenes
├── .collections  ├── .node_groups  ├── .images       ├── .textures
├── .lights       ├── .cameras      ├── .curves       ├── .armatures
├── .worlds       ├── .actions      └── ... (30+ total collection types)
```

### Critical Warnings

**NEVER** cache `bpy.data` object references across undo operations — undo invalidates all pointers.

**NEVER** call `bpy.ops.*` from a background thread — all bpy calls MUST run on the main thread.

**NEVER** use dict context overrides in Blender 4.0+ — use `context.temp_override()` instead.

**NEVER** modify scene data inside `Panel.draw()` — draw callbacks are read-only.

**NEVER** access `mesh.vertices` while in Edit Mode — the data is not synchronized. Use `bmesh.from_edit_mesh()` instead.

---

## Essential Patterns

### Pattern 1: Data Access vs. Operators

Use `bpy.data` for direct data manipulation (fast, no undo). Use `bpy.ops` only when undo support or user-facing actions are required.

```python
# Blender 3.x/4.x/5.x — CORRECT: direct data access (no context required)
obj = bpy.data.objects.get("Cube")
if obj:
    obj.location = (1.0, 0.0, 0.0)

# Blender 3.x/4.x/5.x — CORRECT: operator when undo is needed
bpy.ops.object.location_clear()  # Requires correct context
```

### Pattern 2: Context Overrides (version-critical)

```python
# Blender 3.x ONLY — BROKEN in 4.0+
override = bpy.context.copy()
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="Subsurf")  # REMOVED in 4.0

# Blender 3.2+ and 4.x/5.x — CORRECT
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="Subsurf")

# Blender 4.x/5.x — Override area for viewport operators
for window in bpy.context.window_manager.windows:
    for area in window.screen.areas:
        if area.type == 'VIEW_3D':
            with bpy.context.temp_override(window=window, area=area):
                bpy.ops.view3d.snap_cursor_to_center()
            break
```

### Pattern 3: ID Data Block Lifecycle

```python
# Blender 3.x/4.x/5.x — create, link, manage user count
mesh = bpy.data.meshes.new("MyMesh")         # Creates data block
obj = bpy.data.objects.new("MyObject", mesh) # Creates object referencing mesh
bpy.context.collection.objects.link(obj)     # REQUIRED: link to scene collection

# Prevent orphaned data from being purged on save
mesh.use_fake_user = True  # Keeps data even with zero users

# Remove data block
bpy.data.meshes.remove(mesh)
```

### Pattern 4: Dependency Graph

```python
# Blender 3.x/4.x/5.x — get evaluated (post-modifier) data
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)       # Evaluated object
mesh_eval = obj_eval.to_mesh()                # Evaluated mesh (read-only)
# ... read mesh_eval ...
obj_eval.to_mesh_clear()                      # ALWAYS clean up

# Iterate ALL instances (including collection/particle instances)
for instance in depsgraph.object_instances:
    obj = instance.object          # Evaluated object
    matrix = instance.matrix_world # World transform
    is_inst = instance.is_instance # True for non-base instances
```

---

## Common Operations

### Object Selection and Activation

```python
# Blender 3.x/4.x/5.x — correct API (2.80+ style)
obj.select_set(True)                              # NOT obj.select = True
bpy.context.view_layer.objects.active = obj       # NOT scene.objects.active
```

### RNA Introspection

```python
# Blender 3.x/4.x/5.x — inspect properties via RNA
obj = bpy.context.active_object
for prop in obj.bl_rna.properties:
    print(f"{prop.identifier}: {prop.type}")

# Get type info on a specific property
prop = obj.bl_rna.properties['location']
print(prop.array_length)  # 3
print(prop.subtype)       # 'TRANSLATION'
```

### Application Handlers

```python
# Blender 3.x/4.x/5.x
from bpy.app.handlers import persistent

@persistent  # Decorator required to survive file load
def on_depsgraph_update(scene, depsgraph):
    for update in depsgraph.updates:
        if update.is_updated_geometry:
            print(f"Geometry changed: {update.id.name}")

bpy.app.handlers.depsgraph_update_post.append(on_depsgraph_update)

# Unregister in addon unregister():
bpy.app.handlers.depsgraph_update_post.remove(on_depsgraph_update)
```

### Version Detection

```python
# Blender 3.x/4.x/5.x
major, minor, patch = bpy.app.version  # e.g., (4, 2, 0)

if bpy.app.version >= (4, 0, 0):
    # Use temp_override, NodeTree.interface, bone.collections
    pass
if bpy.app.version >= (5, 0, 0):
    # Use gpu module exclusively for drawing
    pass
```

### Creating and Linking Objects

```python
# Blender 3.x/4.x/5.x — complete pattern
mesh = bpy.data.meshes.new("MyMesh")
verts = [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]
faces = [(0, 1, 2, 3)]
mesh.from_pydata(verts, [], faces)
mesh.update()  # ALWAYS call after from_pydata

obj = bpy.data.objects.new("MyObject", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED — objects without links are invisible
```

### Restricted Contexts

| Callback | Forbidden Operations |
|----------|---------------------|
| `Panel.draw()` | `bpy.ops.*`, property modification |
| `depsgraph_update_post` | Adding/removing objects, scene mutation |
| `render_pre/post` | Most `bpy.ops.*` |
| Background mode | All viewport operators, OpenGL |

---

## Version-Specific Migration Rules

| Feature | Blender 3.x | Blender 4.0+ |
|---------|-------------|--------------|
| Context override | `bpy.ops.foo(override_dict, ...)` | `context.temp_override(...)` |
| Mesh bevel weight | `edge.bevel_weight` | `mesh.attributes["bevel_weight_edge"]` |
| Mesh crease | `edge.crease` | `mesh.attributes["crease_edge"]` |
| Node group sockets | `node_group.inputs.new(...)` | `node_group.interface.new_socket(...)` |
| Bone layers | `bone.layers[i]` | `bone.collections` |

| Feature | Blender 4.x | Blender 5.0+ |
|---------|-------------|--------------|
| Drawing | `bgl` module (deprecated) | `gpu` module (REQUIRED) |
| Compositing | `scene.node_tree` | `scene.compositing_node_group` |
| EEVEE identifier | `BLENDER_EEVEE` or `BLENDER_EEVEE_NEXT` | `BLENDER_EEVEE` |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for bpy.data, bpy.context, bpy.ops, bpy.types, Depsgraph
- [references/examples.md](references/examples.md) — Working code examples verified against official documentation
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do, with WHY explanations

### Official Sources

- https://docs.blender.org/api/current/info_quickstart.html
- https://docs.blender.org/api/current/info_overview.html
- https://docs.blender.org/api/current/bpy.context.html
- https://docs.blender.org/api/current/bpy.types.Depsgraph.html
- https://docs.blender.org/api/current/bpy.types.Context.html
