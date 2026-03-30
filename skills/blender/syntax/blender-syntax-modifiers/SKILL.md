---
name: blender-syntax-modifiers
description: >
  Use when working with Blender modifiers via Python -- adding, configuring, or applying
  modifiers, or accessing Geometry Nodes inputs. Prevents the common mistake of reading
  mesh data before applying modifiers (getting unmodified geometry) instead of using
  depsgraph.evaluated_get(). Covers modifier stack, Geometry Nodes input identifiers,
  common AEC modifiers (Array, Boolean, Solidify), and evaluated mesh access.
  Keywords: modifier, Array, Boolean, Solidify, Geometry Nodes, depsgraph, evaluated_get, modifier.apply, modifier stack, input identifier, add modifier from code, apply modifier Python, boolean cut.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-modifiers

## Quick Reference

### Critical Warnings

**NEVER** apply modifiers while the object is in Edit Mode — `modifier_apply` requires Object Mode. Switch with `bpy.ops.object.mode_set(mode='OBJECT')` first.

**NEVER** use dict context overrides for `modifier_apply` in Blender 4.0+ — use `context.temp_override()` instead. Dict overrides raise `TypeError`.

**NEVER** read `obj.data.vertices` expecting post-modifier results — use `obj.evaluated_get(depsgraph).to_mesh()` to access evaluated mesh data.

**NEVER** forget `obj_eval.to_mesh_clear()` after `obj_eval.to_mesh()` — this causes memory leaks. ALWAYS use a `try/finally` block.

**NEVER** assume Geometry Nodes modifier input identifiers match socket names — identifiers are auto-generated as `Socket_N`. ALWAYS look up identifiers via `modifier.node_group.interface.items_tree`.

**NEVER** modify the evaluated mesh — it is read-only. Modify the original object, then let the depsgraph re-evaluate.

### Modifier Type Decision Tree

```
Need to add geometry procedurally?
├── Repeat geometry in a pattern → ARRAY
├── Combine/subtract meshes → BOOLEAN
├── Add thickness to surfaces → SOLIDIFY
├── Subdivide for smoothness → SUBSURF
├── Symmetry across an axis → MIRROR
├── Edge beveling → BEVEL
├── Custom node-based logic → NODES (Geometry Nodes)
└── Reduce polygon count → DECIMATE

Need deformation?
├── Skeleton-based → ARMATURE
├── Surface projection → SHRINKWRAP
├── Volume-based deform → MESH_DEFORM
├── Grid-based deform → LATTICE
└── Along a path → CURVE
```

### AEC-Relevant Modifiers

| Modifier | AEC Use Case | Type String |
|----------|-------------|-------------|
| Array | Repetitive elements (columns, railings, facade panels) | `'ARRAY'` |
| Boolean | Wall openings, structural cutouts, MEP penetrations | `'BOOLEAN'` |
| Solidify | Thickness to imported 2D geometry, wall shells | `'SOLIDIFY'` |
| Geometry Nodes | Parametric components, distribution, rule-based generation | `'NODES'` |
| Mirror | Symmetric buildings, reflected floor plans | `'MIRROR'` |
| Bevel | Edge finishing for architectural details | `'BEVEL'` |

---

## Essential Patterns

### Pattern 1: Add a Modifier

```python
# Blender 3.x/4.x/5.x: add modifier to an object
import bpy

obj = bpy.data.objects.get("Wall")
if obj is None:
    raise RuntimeError("Object 'Wall' not found")

# Add modifier: returns the new Modifier object
mod = obj.modifiers.new(name="MyArray", type='ARRAY')
# name: display name (Blender may append .001 if duplicate)
# type: modifier type enum string (see table above)

# Configure modifier properties
mod.count = 5
mod.relative_offset_displace = (1.0, 0.0, 0.0)
```

### Pattern 2: Remove a Modifier

```python
# Blender 3.x/4.x/5.x: remove modifier by reference or name
import bpy

obj = bpy.data.objects.get("Wall")

# Remove by reference
mod = obj.modifiers.get("MyArray")
if mod is not None:
    obj.modifiers.remove(mod)

# Remove all modifiers
for mod in obj.modifiers[:]:  # Copy list — modifiers change during removal
    obj.modifiers.remove(mod)
```

### Pattern 3: Apply a Modifier (Version-Critical)

Applying a modifier permanently bakes its effect into the mesh. This is an IRREVERSIBLE operation. ALWAYS requires Object Mode.

```python
# Blender 3.2+/4.x/5.x: apply modifier using temp_override
import bpy

obj = bpy.data.objects.get("Wall")

# Ensure Object Mode
if bpy.context.mode != 'OBJECT':
    bpy.ops.object.mode_set(mode='OBJECT')

# Apply a single modifier
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="MyArray")

# Apply ALL modifiers (copy list: stack changes during apply)
with bpy.context.temp_override(object=obj, active_object=obj):
    for mod in obj.modifiers[:]:
        bpy.ops.object.modifier_apply(modifier=mod.name)
```

```python
# Blender < 3.2 ONLY (legacy: do NOT use for new code)
override = bpy.context.copy()
override['object'] = obj
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="MyArray")
```

### Pattern 4: Evaluated Mesh via Depsgraph (Preview Without Applying)

Use this to read the post-modifier mesh WITHOUT permanently applying modifiers.

```python
# Blender 3.x/4.x/5.x: read evaluated mesh data
import bpy

obj = bpy.data.objects.get("Wall")
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()

try:
    print(f"Original verts: {len(obj.data.vertices)}")
    print(f"Evaluated verts: {len(mesh_eval.vertices)}")

    # Read vertex positions in world space
    for v in mesh_eval.vertices:
        world_co = obj_eval.matrix_world @ v.co
        # process world_co...
finally:
    obj_eval.to_mesh_clear()  # ALWAYS clean up
```

### Pattern 5: Geometry Nodes Modifier: Assign and Configure Inputs

```python
# Blender 4.0+: assign node group and set input values
import bpy

obj = bpy.data.objects.get("Building")
mod = obj.modifiers.new(name="GeoNodes", type='NODES')

# Assign an existing node group
node_group = bpy.data.node_groups.get("AEC_WallGenerator")
if node_group is None:
    raise RuntimeError("Node group 'AEC_WallGenerator' not found")
mod.node_group = node_group

# Find input identifiers by name (REQUIRED: identifiers are NOT names)
def get_gn_input_identifier(modifier, input_name):
    """Return the Socket_N identifier for a named GN input."""
    for item in modifier.node_group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == input_name:
                return item.identifier
    raise KeyError(f"Input '{input_name}' not found in node group")

# Set input values using identifiers
height_id = get_gn_input_identifier(mod, "Height")
mod[height_id] = 3.5

thickness_id = get_gn_input_identifier(mod, "Thickness")
mod[thickness_id] = 0.2

# Force scene update after changing GN inputs
bpy.context.view_layer.update()
```

```python
# Blender 4.0+: list all GN modifier inputs
for item in mod.node_group.interface.items_tree:
    if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
        identifier = item.identifier   # e.g., "Socket_2"
        name = item.name               # e.g., "Height"
        socket_type = item.socket_type  # e.g., "NodeSocketFloat"
        current_value = mod.get(identifier)
        print(f"{name} ({socket_type}): {identifier} = {current_value}")
```

### Pattern 6: Reorder Modifiers

```python
# Blender 4.0+/5.x: move modifier to specific position
import bpy

obj = bpy.data.objects.get("Wall")
with bpy.context.temp_override(object=obj):
    bpy.ops.object.modifier_move_to_index(modifier="Boolean", index=0)

# Blender 3.x/4.x/5.x: move up/down by one position
with bpy.context.temp_override(object=obj):
    bpy.ops.object.modifier_move_up(modifier="Boolean")
    bpy.ops.object.modifier_move_down(modifier="Array")
```

---

## Common Operations

### Array Modifier: Repetitive AEC Elements

```python
# Blender 3.x/4.x/5.x: create array of columns
import bpy

obj = bpy.data.objects.get("Column")
mod = obj.modifiers.new(name="ColumnArray", type='ARRAY')

# Fixed count
mod.use_relative_offset = True
mod.relative_offset_displace = (2.0, 0.0, 0.0)  # Spacing in object-relative units
mod.count = 10

# Constant offset (absolute units)
mod.use_constant_offset = True
mod.constant_offset_displace = (3.0, 0.0, 0.0)  # 3m spacing in scene units

# Fit to length
mod.fit_type = 'FIT_LENGTH'
mod.fit_length = 30.0  # Total span in scene units

# Fit to curve
mod.fit_type = 'FIT_CURVE'
mod.curve = bpy.data.objects.get("ArrayPath")
```

### Boolean Modifier: Wall Openings

```python
# Blender 3.x/4.x/5.x: create opening in wall
import bpy

wall = bpy.data.objects.get("Wall")
opening = bpy.data.objects.get("WindowOpening")

mod = wall.modifiers.new(name="WindowCut", type='BOOLEAN')
mod.operation = 'DIFFERENCE'     # 'INTERSECT', 'UNION', 'DIFFERENCE'
mod.object = opening             # Cutter object
mod.solver = 'FAST'              # 'FAST' or 'EXACT' (Blender 3.0+)
# EXACT is slower but handles edge cases better (coplanar faces, thin geometry)

# Hide cutter in viewport (common AEC pattern)
opening.hide_set(True)
opening.hide_render = True
```

### Solidify Modifier: Wall Thickness

```python
# Blender 3.x/4.x/5.x: add thickness to planar geometry
import bpy

obj = bpy.data.objects.get("WallSurface")
mod = obj.modifiers.new(name="WallThickness", type='SOLIDIFY')

mod.thickness = 0.3              # Wall thickness in scene units
mod.offset = -1.0                # -1 = outward, 0 = centered, 1 = inward
mod.use_even_offset = True       # Uniform thickness on angled surfaces
mod.use_quality_normals = True   # Better normals on complex geometry
```

### Copy Modifier Settings Between Objects

```python
# Blender 3.x/4.x/5.x: copy all modifiers from source to target
import bpy

source = bpy.data.objects.get("SourceWall")
target = bpy.data.objects.get("TargetWall")

# Use operator to copy modifiers
with bpy.context.temp_override(
    object=target,
    active_object=source,
    selected_objects=[target]
):
    bpy.ops.object.make_links_data(type='MODIFIERS')
```

### Check if Modifier Exists Before Operating

```python
# Blender 3.x/4.x/5.x: safe modifier access
import bpy

obj = bpy.data.objects.get("Wall")

# Check by name
mod = obj.modifiers.get("MyArray")
if mod is not None:
    print(f"Found modifier: {mod.name}, type: {mod.type}")

# Check by type
has_boolean = any(m.type == 'BOOLEAN' for m in obj.modifiers)

# Iterate with type filter
for mod in obj.modifiers:
    if mod.type == 'ARRAY':
        print(f"Array: {mod.name}, count: {mod.count}")
```

---

## Version-Specific Notes

| Feature | Blender 3.x | Blender 4.0+ |
|---------|-------------|--------------|
| Context override for `modifier_apply` | Dict override (deprecated 3.2) | `context.temp_override()` REQUIRED |
| GN node group sockets API | `node_group.inputs.new(...)` | `node_group.interface.new_socket(...)` |
| GN modifier input identifiers | `Input_N` (Blender 3.x) | `Socket_N` (Blender 4.0+) |
| `modifier_move_to_index` | Not available | Available |
| Boolean solver `'EXACT'` | Available (3.0+) | Available |
| `bmesh.from_object()` depsgraph param | Optional | REQUIRED |
| `BVHTree.FromObject()` depsgraph param | Optional | REQUIRED |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for ObjectModifiers, Modifier base, and specific modifier types
- [references/examples.md](references/examples.md) — Working code examples for all common modifier operations
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with modifiers, with explanations

### Official Sources

- https://docs.blender.org/api/current/bpy.types.ObjectModifiers.html
- https://docs.blender.org/api/current/bpy.types.Modifier.html
- https://docs.blender.org/api/current/bpy.types.NodesModifier.html
- https://docs.blender.org/api/current/bpy.types.ArrayModifier.html
- https://docs.blender.org/api/current/bpy.types.BooleanModifier.html
- https://docs.blender.org/api/current/bpy.types.SolidifyModifier.html
- https://docs.blender.org/api/current/bpy.types.Depsgraph.html
