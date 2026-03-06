# blender-syntax-modifiers: API Method Reference

Sources:
- https://docs.blender.org/api/current/bpy.types.ObjectModifiers.html
- https://docs.blender.org/api/current/bpy.types.Modifier.html
- https://docs.blender.org/api/current/bpy.types.NodesModifier.html
- https://docs.blender.org/api/current/bpy.types.ArrayModifier.html
- https://docs.blender.org/api/current/bpy.types.BooleanModifier.html
- https://docs.blender.org/api/current/bpy.types.SolidifyModifier.html

---

## ObjectModifiers Collection — `obj.modifiers`

`bpy.types.ObjectModifiers` is the collection of modifiers on an `Object`. Accessed via `obj.modifiers`.

### Methods

```python
# Blender 3.x/4.x/5.x
obj.modifiers.new(name: str, type: str) -> Modifier
# Add a new modifier to the object.
# name: Display name for the modifier (Blender may append .001 if duplicate)
# type: Modifier type enum string (see Modifier Type Enum below)
# Returns: The newly created Modifier subclass instance

obj.modifiers.remove(modifier: Modifier) -> None
# Remove an existing modifier from the object.
# modifier: The Modifier instance to remove (NOT a string name)
# Raises: RuntimeError if modifier is not on this object

obj.modifiers.clear() -> None
# Remove all modifiers from the object.
# Blender 3.x/4.x/5.x

obj.modifiers.get(name: str, default=None) -> Modifier | None
# Get modifier by name. Returns default if not found.
# Inherited from bpy_prop_collection.

obj.modifiers[name: str] -> Modifier
# Access by name. Raises KeyError if not found.

obj.modifiers[index: int] -> Modifier
# Access by index. Raises IndexError if out of range.

len(obj.modifiers) -> int
# Number of modifiers on the object.

# Iteration
for mod in obj.modifiers:
    print(mod.name, mod.type)
```

### Modifier Type Enum (type parameter for `new()`)

#### Generate Modifiers

| Type String | Modifier Name | Description |
|-------------|--------------|-------------|
| `'ARRAY'` | Array | Duplicate geometry in a pattern |
| `'BEVEL'` | Bevel | Bevel edges or vertices |
| `'BOOLEAN'` | Boolean | Combine/subtract/intersect meshes |
| `'BUILD'` | Build | Animate face appearance |
| `'DECIMATE'` | Decimate | Reduce polygon count |
| `'EDGE_SPLIT'` | Edge Split | Split edges by angle |
| `'MASK'` | Mask | Hide geometry by vertex group |
| `'MIRROR'` | Mirror | Mirror geometry across axis |
| `'MULTIRES'` | Multiresolution | Subdivide with sculpt support |
| `'NODES'` | Geometry Nodes | Custom node-based modifier |
| `'REMESH'` | Remesh | Retopologize mesh |
| `'SCREW'` | Screw | Revolve profile |
| `'SKIN'` | Skin | Generate mesh around edges |
| `'SOLIDIFY'` | Solidify | Add thickness |
| `'SUBSURF'` | Subdivision Surface | Smooth subdivision |
| `'TRIANGULATE'` | Triangulate | Convert to triangles |
| `'VOLUME_TO_MESH'` | Volume to Mesh | Convert volume to mesh |
| `'WELD'` | Weld | Merge close vertices |
| `'WIREFRAME'` | Wireframe | Generate wireframe mesh |

#### Deform Modifiers

| Type String | Modifier Name | Description |
|-------------|--------------|-------------|
| `'ARMATURE'` | Armature | Skeleton deformation |
| `'CAST'` | Cast | Deform toward geometric shape |
| `'CURVE'` | Curve | Deform along a curve |
| `'DISPLACE'` | Displace | Offset by texture/vertex group |
| `'HOOK'` | Hook | Deform with empty/bone |
| `'LAPLACIANDEFORM'` | Laplacian Deform | Shape preservation deform |
| `'LAPLACIANSMOOTH'` | Laplacian Smooth | Noise-reducing smooth |
| `'LATTICE'` | Lattice | Grid-based deformation |
| `'MESH_DEFORM'` | Mesh Deform | Cage-based deformation |
| `'SHRINKWRAP'` | Shrinkwrap | Project onto target surface |
| `'SIMPLE_DEFORM'` | Simple Deform | Twist, bend, taper, stretch |
| `'SMOOTH'` | Smooth | Vertex smoothing |
| `'CORRECTIVE_SMOOTH'` | Corrective Smooth | Smooth preserving volume |
| `'SURFACE_DEFORM'` | Surface Deform | Bind to target surface |
| `'WARP'` | Warp | Deform between two points |
| `'WAVE'` | Wave | Animated wave deformation |

#### Physics Modifiers (added via physics systems, not `new()`)

| Type String | Modifier Name |
|-------------|--------------|
| `'CLOTH'` | Cloth |
| `'COLLISION'` | Collision |
| `'DYNAMIC_PAINT'` | Dynamic Paint |
| `'FLUID'` | Fluid |
| `'OCEAN'` | Ocean |
| `'PARTICLE_SYSTEM'` | Particle System |
| `'SOFT_BODY'` | Soft Body |

---

## Modifier Base — `bpy.types.Modifier`

All modifiers inherit from `bpy.types.Modifier`. These properties are available on every modifier.

### Base Properties

| Property | Type | Description |
|----------|------|-------------|
| `.name` | `str` | Modifier display name |
| `.type` | `str` | Read-only modifier type enum (e.g., `'ARRAY'`) |
| `.show_viewport` | `bool` | Show modifier effect in viewport (default: `True`) |
| `.show_render` | `bool` | Show modifier effect in render (default: `True`) |
| `.show_in_editmode` | `bool` | Show modifier in Edit Mode (default: `True`) |
| `.show_on_cage` | `bool` | Adjust edit cage to modifier result (default: `False`) |
| `.show_expanded` | `bool` | Expanded in properties panel |
| `.is_active` | `bool` | Whether this is the active modifier (Blender 4.0+) |
| `.use_apply_on_spline` | `bool` | Apply on spline control points (curve modifiers) |

---

## ArrayModifier — `bpy.types.ArrayModifier`

### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.count` | `int` | `2` | Number of duplicates (when `fit_type='FIXED_COUNT'`) |
| `.fit_type` | `enum` | `'FIXED_COUNT'` | `'FIXED_COUNT'`, `'FIT_LENGTH'`, `'FIT_CURVE'` |
| `.fit_length` | `float` | `0.0` | Total length to fill (when `fit_type='FIT_LENGTH'`) |
| `.curve` | `Object \| None` | `None` | Curve to fit (when `fit_type='FIT_CURVE'`) |
| `.use_relative_offset` | `bool` | `True` | Use relative offset |
| `.relative_offset_displace` | `Vector[3]` | `(1,0,0)` | Relative offset per axis (1.0 = one object width) |
| `.use_constant_offset` | `bool` | `False` | Use constant (absolute) offset |
| `.constant_offset_displace` | `Vector[3]` | `(0,0,0)` | Constant offset in scene units |
| `.use_object_offset` | `bool` | `False` | Use an empty's transform as offset |
| `.offset_object` | `Object \| None` | `None` | Object for transform offset |
| `.use_merge_vertices` | `bool` | `False` | Merge vertices of adjacent copies |
| `.merge_threshold` | `float` | `0.01` | Distance for vertex merging |
| `.use_merge_vertices_cap` | `bool` | `False` | Merge first/last copy vertices |
| `.start_cap` | `Object \| None` | `None` | Object as start cap |
| `.end_cap` | `Object \| None` | `None` | Object as end cap |

---

## BooleanModifier — `bpy.types.BooleanModifier`

### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.operation` | `enum` | `'DIFFERENCE'` | `'INTERSECT'`, `'UNION'`, `'DIFFERENCE'` |
| `.operand_type` | `enum` | `'OBJECT'` | `'OBJECT'` or `'COLLECTION'` (Blender 3.0+) |
| `.object` | `Object \| None` | `None` | Target object (when `operand_type='OBJECT'`) |
| `.collection` | `Collection \| None` | `None` | Target collection (when `operand_type='COLLECTION'`) |
| `.solver` | `enum` | `'EXACT'` | `'FAST'` or `'EXACT'` (Blender 3.0+) |
| `.use_self` | `bool` | `False` | Allow self-intersection (EXACT solver only) |
| `.use_hole_tolerant` | `bool` | `False` | Better handling of non-manifold (EXACT solver only) |
| `.double_threshold` | `float` | `0.0` | Distance for merging coincident geometry |

### Solver Notes

- **FAST**: Uses BMesh boolean. Fast but fails on coplanar faces and thin geometry.
- **EXACT**: Uses exact arithmetic. Slower but handles edge cases (coplanar, self-intersecting, non-manifold). ALWAYS prefer EXACT for AEC geometry which frequently has coplanar faces.

---

## SolidifyModifier — `bpy.types.SolidifyModifier`

### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.thickness` | `float` | `0.01` | Shell thickness |
| `.offset` | `float` | `-1.0` | Offset direction: -1=outward, 0=centered, 1=inward |
| `.use_even_offset` | `bool` | `False` | Maintain uniform thickness on angled geometry |
| `.use_quality_normals` | `bool` | `False` | Better normals for complex geometry |
| `.use_rim` | `bool` | `True` | Fill edges at open boundaries |
| `.use_rim_only` | `bool` | `False` | Only generate rim, no shell |
| `.use_flip_normals` | `bool` | `False` | Flip normals of inner surface |
| `.solidify_mode` | `enum` | `'EXTRUDE'` | `'EXTRUDE'` (simple) or `'NON_MANIFOLD'` (complex) |
| `.nonmanifold_thickness_mode` | `enum` | `'FIXED'` | `'FIXED'`, `'EVEN'`, `'CONSTRAINTS'` (NON_MANIFOLD mode) |
| `.nonmanifold_boundary_mode` | `enum` | `'NONE'` | `'NONE'`, `'ROUND'`, `'FLAT'` (NON_MANIFOLD mode) |
| `.thickness_clamp` | `float` | `0.0` | Clamp thickness (0=no clamp) |
| `.vertex_group` | `str` | `""` | Vertex group for variable thickness |
| `.thickness_vertex_group` | `float` | `0.0` | Factor for vertex group thickness |
| `.material_offset` | `int` | `0` | Material index offset for new faces |
| `.material_offset_rim` | `int` | `0` | Material index offset for rim faces |

---

## NodesModifier — `bpy.types.NodesModifier`

### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.node_group` | `NodeTree \| None` | `None` | The Geometry Nodes group assigned |
| `.show_group_selector` | `bool` | `True` | Show the node group selector in UI |
| `.bake_directory` | `str` | `""` | Directory for baked simulation data |

### Accessing GN Modifier Inputs (Blender 4.0+)

GN modifier inputs are accessed as custom properties on the modifier using identifier keys.

```python
# Blender 4.0+ — access via node_group.interface.items_tree
modifier.node_group  # -> NodeTree
modifier.node_group.interface  # -> NodeTreeInterface
modifier.node_group.interface.items_tree  # -> Collection of NodeTreeInterfaceItem

# Each item has:
# .item_type: 'SOCKET' or 'PANEL'
# .in_out: 'INPUT' or 'OUTPUT' (only for SOCKETs)
# .name: Human-readable name (e.g., "Height")
# .identifier: Internal identifier (e.g., "Socket_2")
# .socket_type: Socket type string (e.g., "NodeSocketFloat")

# Read input value:
value = modifier[item.identifier]

# Write input value:
modifier[item.identifier] = new_value
```

### GN Input Socket Types

| Socket Type String | Python Type | Example Value |
|-------------------|-------------|---------------|
| `NodeSocketFloat` | `float` | `3.5` |
| `NodeSocketInt` | `int` | `10` |
| `NodeSocketBool` | `bool` / `int` | `True` or `1` |
| `NodeSocketVector` | `tuple[float, float, float]` | `(1.0, 2.0, 3.0)` |
| `NodeSocketColor` | `tuple[float, float, float, float]` | `(1.0, 0.5, 0.0, 1.0)` |
| `NodeSocketString` | `str` | `"text"` |
| `NodeSocketObject` | Pointer → set via `modifier[id]` | `bpy.data.objects["Cube"]` |
| `NodeSocketCollection` | Pointer → set via `modifier[id]` | `bpy.data.collections["Col"]` |
| `NodeSocketMaterial` | Pointer → set via `modifier[id]` | `bpy.data.materials["Mat"]` |
| `NodeSocketImage` | Pointer → set via `modifier[id]` | `bpy.data.images["Img"]` |
| `NodeSocketGeometry` | N/A (internal connection only) | — |

---

## Modifier Operators — `bpy.ops.object`

All modifier operators require correct context. In Blender 4.0+, use `context.temp_override()`.

### Apply

```python
# Blender 3.2+/4.x/5.x
bpy.ops.object.modifier_apply(
    modifier: str = "",         # Modifier name to apply
    report: bool = False,       # Report to info area
) -> set[str]  # {'FINISHED'} or {'CANCELLED'}
# REQUIRES: Object mode. Fails in Edit Mode.
# REQUIRES: context.object set to the target object.
```

### Apply as Shape Key

```python
# Blender 3.x/4.x/5.x
bpy.ops.object.modifier_apply_as_shapekey(
    keep_modifier: bool = False,  # Keep modifier after applying
    modifier: str = "",           # Modifier name
    report: bool = False,
) -> set[str]
```

### Move

```python
# Blender 3.x/4.x/5.x
bpy.ops.object.modifier_move_up(modifier: str = "") -> set[str]
bpy.ops.object.modifier_move_down(modifier: str = "") -> set[str]

# Blender 4.0+
bpy.ops.object.modifier_move_to_index(
    modifier: str = "",
    index: int = 0,
) -> set[str]
```

### Copy

```python
# Blender 3.x/4.x/5.x
bpy.ops.object.modifier_copy(modifier: str = "") -> set[str]
```

### Set Active

```python
# Blender 4.0+
bpy.ops.object.modifier_set_active(modifier: str = "") -> set[str]
```

---

## Depsgraph — Evaluated Mesh Access

### Key Methods

```python
# Blender 3.x/4.x/5.x
depsgraph = bpy.context.evaluated_depsgraph_get()
# Returns: Depsgraph — may trigger evaluation if scene has pending changes

obj_eval = obj.evaluated_get(depsgraph)
# Returns: Object — evaluated version with all modifiers applied (read-only)

mesh_eval = obj_eval.to_mesh()
# Returns: Mesh — temporary mesh from evaluated state
# MUST be freed with obj_eval.to_mesh_clear()

obj_eval.to_mesh_clear()
# Frees the temporary mesh. ALWAYS call after to_mesh().

original = obj_eval.original
# Returns: Object — back-reference to the original data block
```

### BVHTree and BMesh with Depsgraph (Blender 4.0+ — depsgraph parameter REQUIRED)

```python
# Blender 4.0+
from mathutils.bvhtree import BVHTree
import bmesh

depsgraph = bpy.context.evaluated_depsgraph_get()

# BVHTree from evaluated object
bvh = BVHTree.FromObject(obj, depsgraph)  # depsgraph is REQUIRED in 4.0+

# BMesh from evaluated object
bm = bmesh.new()
bm.from_object(obj, depsgraph)  # depsgraph is REQUIRED in 4.0+
bm.free()
```
