# blender-core-runtime: API Method Reference

Sources:
- https://docs.blender.org/api/current/mathutils.html
- https://docs.blender.org/api/current/mathutils.kdtree.html
- https://docs.blender.org/api/current/mathutils.bvhtree.html
- https://docs.blender.org/api/current/bl_math.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy.msgbus.html
- https://docs.blender.org/api/current/bpy.app.handlers.html

---

## mathutils.Vector (all versions: 3.x/4.x/5.x)

`from mathutils import Vector`

### Constructors

```python
Vector((x, y))               # 2D vector
Vector((x, y, z))            # 3D vector
Vector((x, y, z, w))         # 4D vector (homogeneous)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.x`, `.y`, `.z`, `.w` | `float` | Component access |
| `.length` | `float` | Euclidean length (involves sqrt) |
| `.length_squared` | `float` | Squared length (faster, use for comparisons) |
| `.xy`, `.xz`, `.yz` | `Vector` | 2D swizzle |
| `.xzy`, `.yxz`, etc. | `Vector` | 3D swizzle (rearranged) |
| `.is_frozen` | `bool` | True if vector is immutable |
| `.is_valid` | `bool` | True if memory is valid |
| `.is_wrapped` | `bool` | True if wrapping C data |
| `.owner` | `object` | C data owner (if wrapped) |

### Methods

```python
# Returns NEW vector — does NOT modify original
v.normalized()                   # → Vector (unit length)
v.resized(size)                  # → Vector with different dimension
v.to_2d()                       # → Vector 2D (drops z)
v.to_3d()                       # → Vector 3D (z=0)
v.to_4d()                       # → Vector 4D (w=1)
v.to_track_quat(track, up)      # → Quaternion (track='X','Y','Z','-X','-Y','-Z')
v.to_tuple(precision=None)      # → tuple

# Mutates vector IN PLACE — returns None
v.normalize()                    # Normalizes to unit length
v.negate()                       # Negates all components
v.zero()                         # Sets all components to 0
v.freeze()                       # Makes immutable (hashable)
v.resize(size)                   # Resize dimension in place
v.resize_2d()                    # Resize to 2D in place
v.resize_3d()                    # Resize to 3D in place
v.resize_4d()                    # Resize to 4D in place

# Computation methods — return values
v.dot(other)                     # → float (dot product)
v.cross(other)                   # → Vector (3D cross product)
v.angle(other, fallback=None)    # → float (radians)
v.angle_signed(other, fallback)  # → float (signed angle around axis, 2D only)
v.project(other)                 # → Vector (projection onto other)
v.reflect(mirror)                # → Vector (reflection across plane)
v.lerp(other, factor)            # → Vector (linear interpolation)
v.slerp(other, factor, fallback=None)  # → Vector (spherical linear interp)
v.rotation_difference(other)     # → Quaternion (rotation from self to other)
v.orthogonal()                   # → Vector (perpendicular, arbitrary)
v.copy()                         # → Vector (deep copy)
```

### Operators

```python
v1 + v2          # Vector addition
v1 - v2          # Vector subtraction
v * scalar       # Scalar multiplication
v / scalar       # Scalar division
-v               # Negation
v @ matrix       # Matrix-vector multiplication (transform point)
matrix @ v       # Matrix-vector multiplication (transform point)
v1 @ v2          # Dot product (same as v1.dot(v2))
```

---

## mathutils.Matrix (all versions: 3.x/4.x/5.x)

`from mathutils import Matrix`

### Constructors (Class Methods)

```python
Matrix()                              # 4x4 identity
Matrix.Identity(size)                 # NxN identity (2, 3, or 4)
Matrix.Translation(vector)            # 4x4 translation
Matrix.Rotation(angle, size, axis)    # angle: radians; size: 3 or 4; axis: 'X','Y','Z' or Vector
Matrix.Scale(factor, size, axis=None) # Uniform or axis-specific scale
Matrix.Diagonal(vector)               # Diagonal matrix from vector
Matrix.Shear(plane, size, factor)     # Shear matrix
Matrix.OrthoProjection(axis, size)    # Orthogonal projection
Matrix.LocRotScale(loc, rot, scale)   # Combined TRS matrix (Blender 3.4+)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.row[i]` | `Vector` | Row access |
| `.col[i]` | `Vector` | Column access |
| `.translation` | `Vector` | Translation component (4x4 only) |
| `.median_scale` | `float` | Average scale factor |
| `.is_identity` | `bool` | True if identity matrix |
| `.is_negative` | `bool` | True if negative determinant (mirrored) |
| `.is_orthogonal` | `bool` | True if orthogonal |
| `.is_orthogonal_axis_vectors` | `bool` | True if axes are orthogonal |

### Methods

```python
# Returns NEW matrix — does NOT modify original
m.decompose()                    # → (Vector_loc, Quaternion_rot, Vector_scale)
m.inverted()                     # → Matrix (raises ValueError if singular)
m.inverted_safe()                # → Matrix (returns identity if singular)
m.transposed()                   # → Matrix
m.normalized()                   # → Matrix (orthonormal)
m.adjugated()                    # → Matrix (adjugate/classical adjoint)
m.to_2x2()                      # → 2x2 Matrix
m.to_3x3()                      # → 3x3 Matrix
m.to_4x4()                      # → 4x4 Matrix
m.to_euler(order='XYZ', compatible=None)  # → Euler
m.to_quaternion()                # → Quaternion
m.to_scale()                     # → Vector (scale component)
m.to_translation()               # → Vector (translation)
m.copy()                         # → Matrix (deep copy)
m.lerp(other, factor)            # → Matrix (linear interpolation)

# Mutates matrix IN PLACE — returns None
m.identity()                     # Reset to identity
m.invert()                       # Invert in place
m.invert_safe()                  # Invert in place (identity if singular)
m.normalize()                    # Normalize in place
m.transpose()                    # Transpose in place
m.zero()                         # Set all elements to 0
m.resize_4x4()                   # Resize to 4x4 in place
m.freeze()                       # Make immutable

# Computation
m.determinant()                  # → float
```

### Operators

```python
m1 @ m2          # Matrix multiplication (ALWAYS use @, NEVER *)
m @ v            # Transform vector by matrix
v @ m            # NOT equivalent to m @ v (transposes semantics)
```

---

## mathutils.Quaternion (all versions: 3.x/4.x/5.x)

`from mathutils import Quaternion`

### Constructors

```python
Quaternion()                           # Identity (1, 0, 0, 0)
Quaternion((w, x, y, z))              # From components
Quaternion(axis_vector, angle)         # From axis-angle (angle in radians)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.w`, `.x`, `.y`, `.z` | `float` | Components |
| `.axis` | `Vector` | Rotation axis |
| `.angle` | `float` | Rotation angle (radians) |
| `.magnitude` | `float` | Length of quaternion |

### Methods

```python
# Returns NEW quaternion
q.conjugated()                   # → Quaternion (conjugate)
q.inverted()                     # → Quaternion (inverse)
q.normalized()                   # → Quaternion (unit quaternion)
q.negated()                      # → Quaternion (negated)
q.copy()                         # → Quaternion
q.slerp(other, factor)           # → Quaternion (spherical linear interp)
q.rotation_difference(other)     # → Quaternion (rotation from self to other)
q.to_axis_angle()                # → (Vector, float)
q.to_euler(order='XYZ', compatible=None)  # → Euler
q.to_matrix()                    # → 3x3 Matrix
q.to_exponential_map()           # → Vector (exponential map representation)
q.to_swing_twist(axis)           # → (Quaternion_swing, float_twist)

# Mutates IN PLACE
q.conjugate()
q.invert()
q.normalize()
q.negate()
q.identity()
q.rotate(other_euler_or_quat)
q.make_compatible(other)
```

### Operators

```python
q1 @ q2          # Combine rotations
q @ v            # Rotate vector by quaternion
```

---

## mathutils.Euler (all versions: 3.x/4.x/5.x)

`from mathutils import Euler`

### Constructor

```python
Euler((x, y, z), order='XYZ')   # x, y, z in radians; order: 'XYZ', 'XZY', 'YXZ', 'YZX', 'ZXY', 'ZYX'
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.x`, `.y`, `.z` | `float` | Rotation angles (radians) |
| `.order` | `str` | Rotation order string |

### Methods

```python
e.to_quaternion()                # → Quaternion
e.to_matrix()                    # → 3x3 Matrix
e.copy()                         # → Euler
e.zero()                         # Reset to (0, 0, 0) in place
e.rotate(other)                  # Combine rotations in place
e.rotate_axis(axis, angle)       # Rotate around single axis in place ('X', 'Y', 'Z')
e.make_compatible(other)         # Adjust to avoid gimbal jumps (animation)
e.freeze()                       # Make immutable
```

---

## mathutils.kdtree.KDTree (all versions: 3.x/4.x/5.x)

`from mathutils.kdtree import KDTree`

### Constructor

```python
KDTree(size)     # size: maximum number of points
```

### Methods

```python
kd.insert(co, index)            # Insert point (co: 3D tuple/Vector, index: int)
kd.balance()                     # MANDATORY: build tree after all inserts

# Query methods — MUST call balance() first
kd.find(co, filter=None)        # → (co, index, dist) nearest point
kd.find_n(co, n)                # → list of (co, index, dist) N nearest
kd.find_range(co, radius)       # → list of (co, index, dist) within radius
```

---

## mathutils.bvhtree.BVHTree (all versions: 3.x/4.x/5.x)

`from mathutils.bvhtree import BVHTree`

### Constructors (Class Methods)

```python
BVHTree.FromObject(object, depsgraph, deform=True, render=False, cage=False)
BVHTree.FromBMesh(bmesh, epsilon=0.0)
BVHTree.FromPolygons(vertices, polygons, all_triangles=False, epsilon=0.0)
```

### Methods

```python
bvh.ray_cast(origin, direction, distance=sys.float_info.max)
# → (location, normal, index, distance) or (None, None, None, None)

bvh.find_nearest(origin, distance=1.84467e+19)
# → (location, normal, index, distance) or (None, None, None, None)

bvh.find_nearest_range(origin, distance=1.84467e+19)
# → list of (location, normal, index, distance)

bvh.overlap(other_tree)
# → list of (index_self, index_other) pairs of overlapping polygons
```

---

## bl_math (all versions: 3.x/4.x/5.x)

`import bl_math`

```python
bl_math.lerp(from_val, to_val, factor)     # → float (linear interpolation)
bl_math.clamp(value, min=0.0, max=1.0)     # → float (clamped value)
bl_math.smoothstep(from_val, to_val, value) # → float (cubic Hermite S-curve)
```

All functions operate on `float` only (NOT vectors). Available in driver expressions.

---

## bpy.app.timers (all versions: 3.x/4.x/5.x)

```python
import bpy

bpy.app.timers.register(
    function,                    # Callable — MUST return None or float
    first_interval=0.0,         # Delay before first call (seconds)
    persistent=False,           # Survive file load if True
)

bpy.app.timers.unregister(function)        # Remove timer
bpy.app.timers.is_registered(function)     # → bool
```

### Timer Return Values

| Return | Behavior |
|--------|----------|
| `None` | Timer unregisters (one-shot) |
| `float` | Timer reschedules after N seconds |

---

## bpy.app.handlers (all versions: 3.x/4.x/5.x)

```python
from bpy.app.handlers import persistent

@persistent
def my_handler(scene):
    pass

bpy.app.handlers.<handler_name>.append(my_handler)
bpy.app.handlers.<handler_name>.remove(my_handler)
```

### Available Handlers

| Handler | Signature | Trigger |
|---------|-----------|---------|
| `load_pre` | `(filepath)` | Before file load |
| `load_post` | `(filepath)` | After file load |
| `save_pre` | `(filepath)` | Before file save |
| `save_post` | `(filepath)` | After file save |
| `undo_pre` | `(scene)` | Before undo |
| `undo_post` | `(scene)` | After undo |
| `redo_pre` | `(scene)` | Before redo |
| `redo_post` | `(scene)` | After redo |
| `depsgraph_update_pre` | `(scene, depsgraph)` | Before depsgraph evaluation |
| `depsgraph_update_post` | `(scene, depsgraph)` | After depsgraph evaluation |
| `frame_change_pre` | `(scene)` | Before frame change |
| `frame_change_post` | `(scene, depsgraph)` | After frame change |
| `render_pre` | `(scene)` | Before render starts |
| `render_post` | `(scene)` | After render completes |
| `render_init` | `(engine)` | Render engine init |
| `render_complete` | `(scene)` | Render fully done |
| `render_cancel` | `(scene)` | Render cancelled |

---

## bpy.msgbus (all versions: 3.x/4.x/5.x)

```python
import bpy

# Subscribe to property changes
bpy.msgbus.subscribe_rna(
    key=property_path_or_tuple,  # Instance: obj.location; Type: (bpy.types.Object, "location")
    owner=owner_object,          # Any Python object (identity-compared for cleanup)
    args=tuple_of_args,          # Passed to notify callback
    notify=callback_function,    # Called when property changes
    options=set(),               # Empty set or {'PERSISTENT'}
)

# Clear all subscriptions for an owner
bpy.msgbus.clear_by_owner(owner)

# Force notification (manual publish)
bpy.msgbus.publish_rna(key=property_path)
```

### msgbus vs Property Update Callbacks

| Aspect | `bpy.props` update callback | `bpy.msgbus` |
|--------|---------------------------|--------------|
| Timing | Immediate on property write | Postponed until operators finish |
| Frequency | Every write | Once per update cycle |
| File load survival | Inherent (part of property def) | Only with `PERSISTENT` option |
| Scope | Own property only | Any RNA property |

---

## bpy.app — Runtime Information (all versions: 3.x/4.x/5.x)

| Attribute | Type | Description |
|-----------|------|-------------|
| `bpy.app.version` | `tuple(int, int, int)` | Blender version `(major, minor, patch)` |
| `bpy.app.version_string` | `str` | Human-readable version |
| `bpy.app.background` | `bool` | True if running in background mode |
| `bpy.app.binary_path` | `str` | Path to Blender executable |
| `bpy.app.tempdir` | `str` | Temporary directory path |
| `bpy.app.debug` | `bool` | True if `--debug` flag passed |
| `bpy.app.is_job_running(job_type)` | method → `bool` | Check if job is running (e.g., `'RENDER'`) |
