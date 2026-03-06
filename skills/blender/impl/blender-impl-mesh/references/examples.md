# blender-impl-mesh — Examples Reference

Working code examples for AEC mesh workflows. All examples verified against Blender Python API documentation.

---

## Example 1: Create a Wall from Footprint

Creates a wall mesh from a 2D polyline (start/end points) with specified height and thickness.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh
from mathutils import Vector

def create_wall(name, start, end, height=3.0, thickness=0.2):
    """Create a wall mesh from two 2D points.

    Args:
        name: Object name
        start: (x, y) start point
        end: (x, y) end point
        height: Wall height in meters
        thickness: Wall thickness in meters
    Returns:
        bpy.types.Object
    """
    sx, sy = start
    ex, ey = end

    # Calculate perpendicular offset for thickness
    dx, dy = ex - sx, ey - sy
    length = (dx**2 + dy**2) ** 0.5
    nx, ny = -dy / length * thickness / 2, dx / length * thickness / 2

    verts = [
        (sx + nx, sy + ny, 0),      # 0: bottom-left outer
        (sx - nx, sy - ny, 0),      # 1: bottom-left inner
        (ex - nx, ey - ny, 0),      # 2: bottom-right inner
        (ex + nx, ey + ny, 0),      # 3: bottom-right outer
        (sx + nx, sy + ny, height), # 4: top-left outer
        (sx - nx, sy - ny, height), # 5: top-left inner
        (ex - nx, ey - ny, height), # 6: top-right inner
        (ex + nx, ey + ny, height), # 7: top-right outer
    ]

    faces = [
        (0, 3, 7, 4),  # Outer face
        (1, 5, 6, 2),  # Inner face
        (0, 4, 5, 1),  # Left cap
        (3, 2, 6, 7),  # Right cap
        (0, 1, 2, 3),  # Bottom
        (4, 7, 6, 5),  # Top
    ]

    mesh = bpy.data.meshes.new(name)
    mesh.from_pydata(verts, [], faces)
    mesh.update()
    mesh.validate()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj

# Usage
wall = create_wall("Wall_001", (0, 0), (5, 0), height=3.0, thickness=0.2)
```

---

## Example 2: Create a Slab from Boundary Polygon

Creates a horizontal slab (floor/roof) from an arbitrary polygon outline.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh

def create_slab(name, outline, thickness=0.3, z_offset=0.0):
    """Create a slab mesh from 2D outline coordinates.

    Args:
        name: Object name
        outline: List of (x, y) tuples defining slab boundary
        thickness: Slab thickness in meters
        z_offset: Z position of slab bottom
    Returns:
        bpy.types.Object
    """
    bm = bmesh.new()

    # Create bottom face
    bottom_verts = [bm.verts.new((x, y, z_offset)) for x, y in outline]
    bm.verts.ensure_lookup_table()
    bottom_face = bm.faces.new(bottom_verts)

    # Extrude upward for thickness
    result = bmesh.ops.extrude_face_region(bm, geom=[bottom_face])
    top_verts = [e for e in result['geom'] if isinstance(e, bmesh.types.BMVert)]
    bmesh.ops.translate(bm, vec=(0, 0, thickness), verts=top_verts)

    bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])

    mesh = bpy.data.meshes.new(name)
    bm.to_mesh(mesh)
    bm.free()
    mesh.update()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj

# Usage — L-shaped floor slab
outline = [(0,0), (10,0), (10,5), (6,5), (6,8), (0,8)]
slab = create_slab("FloorSlab_001", outline, thickness=0.3, z_offset=0.0)
```

---

## Example 3: Batch IFC Model Visualization

Loads all geometric IFC elements and creates Blender meshes using foreach_set for performance.

```python
# Blender 3.x/4.x/5.x — requires ifcopenshell
import bpy
import numpy as np
import ifcopenshell
import ifcopenshell.geom

def visualize_ifc_model(ifc_path, target_classes=None):
    """Create Blender meshes for all geometric elements in an IFC file.

    Args:
        ifc_path: Path to .ifc file
        target_classes: List of IFC class names to import, e.g. ['IfcWall', 'IfcSlab'].
                       Default: all IfcProduct subclasses with geometry.
    Returns:
        List of created bpy.types.Object
    """
    ifc_file = ifcopenshell.open(ifc_path)
    settings = ifcopenshell.geom.settings()
    settings.set("use-world-coords", True)

    # Create a collection for the IFC model
    collection = bpy.data.collections.new(ifc_path.split("/")[-1])
    bpy.context.scene.collection.children.link(collection)

    if target_classes is None:
        elements = ifc_file.by_type("IfcProduct")
    else:
        elements = []
        for cls in target_classes:
            elements.extend(ifc_file.by_type(cls))

    created_objects = []

    for element in elements:
        try:
            shape = ifcopenshell.geom.create_shape(settings, element)
        except RuntimeError:
            continue  # No geometry representation

        verts_flat = shape.geometry.verts
        faces_flat = shape.geometry.faces

        if len(verts_flat) == 0:
            continue

        coords = np.array(verts_flat, dtype=np.float32).reshape(-1, 3)
        indices = np.array(faces_flat, dtype=np.int32)

        vert_count = len(coords)
        tri_count = len(indices) // 3

        element_name = element.Name or f"IFC_{element.id()}"
        mesh = bpy.data.meshes.new(element_name)
        mesh.vertices.add(vert_count)
        mesh.loops.add(tri_count * 3)
        mesh.polygons.add(tri_count)

        mesh.vertices.foreach_set("co", coords.flatten())
        mesh.loops.foreach_set("vertex_index", indices)

        loop_starts = np.arange(0, tri_count * 3, 3, dtype=np.int32)
        loop_totals = np.full(tri_count, 3, dtype=np.int32)
        mesh.polygons.foreach_set("loop_start", loop_starts)
        mesh.polygons.foreach_set("loop_total", loop_totals)

        mesh.update()
        mesh.validate()

        obj = bpy.data.objects.new(element_name, mesh)
        collection.objects.link(obj)

        # Store IFC metadata as custom properties
        obj["ifc_class"] = element.is_a()
        obj["ifc_global_id"] = element.GlobalId
        obj["ifc_id"] = element.id()

        created_objects.append(obj)

    return created_objects

# Usage
objects = visualize_ifc_model("/path/to/model.ifc",
                               target_classes=["IfcWall", "IfcSlab", "IfcColumn"])
print(f"Created {len(objects)} mesh objects")
```

---

## Example 4: Mesh Analysis for Quantity Takeoff

Analyzes mesh objects to extract AEC quantities (area, volume, dimensions).

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh
from mathutils import Vector

def quantity_takeoff(objects):
    """Calculate AEC quantities for a list of mesh objects.

    Args:
        objects: List of bpy.types.Object with mesh data
    Returns:
        List of dicts with 'name', 'gross_area', 'volume',
        'length', 'width', 'height'
    """
    results = []

    for obj in objects:
        if obj.type != 'MESH':
            continue

        bm = bmesh.new()
        bm.from_mesh(obj.data)
        bm.transform(obj.matrix_world)  # Apply world transform

        total_area = sum(f.calc_area() for f in bm.faces)
        volume = abs(bm.calc_volume())

        # Bounding box dimensions
        if bm.verts:
            xs = [v.co.x for v in bm.verts]
            ys = [v.co.y for v in bm.verts]
            zs = [v.co.z for v in bm.verts]
            length = max(xs) - min(xs)
            width = max(ys) - min(ys)
            height = max(zs) - min(zs)
        else:
            length = width = height = 0.0

        results.append({
            'name': obj.name,
            'gross_area': round(total_area, 4),
            'volume': round(volume, 4),
            'length': round(length, 4),
            'width': round(width, 4),
            'height': round(height, 4),
        })

        bm.free()

    return results

# Usage
selected = bpy.context.selected_objects
quantities = quantity_takeoff(selected)
for q in quantities:
    print(f"{q['name']}: area={q['gross_area']}m², vol={q['volume']}m³, "
          f"L={q['length']}m W={q['width']}m H={q['height']}m")
```

---

## Example 5: Parametric Column Generator

Creates a column mesh with configurable cross-section and height using BMesh.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh
import math

def create_column(name, cross_section='CIRCLE', width=0.4, depth=0.4,
                  height=3.0, segments=16, location=(0, 0, 0)):
    """Create a parametric column mesh.

    Args:
        name: Object name
        cross_section: 'CIRCLE', 'SQUARE', or 'RECTANGLE'
        width: Column width (X dimension) in meters
        depth: Column depth (Y dimension) in meters
        height: Column height (Z dimension) in meters
        segments: Number of segments for circular cross-section
        location: (x, y, z) placement
    Returns:
        bpy.types.Object
    """
    bm = bmesh.new()

    if cross_section == 'CIRCLE':
        radius = width / 2
        base_verts = []
        for i in range(segments):
            angle = 2 * math.pi * i / segments
            x = radius * math.cos(angle)
            y = radius * math.sin(angle)
            base_verts.append(bm.verts.new((x, y, 0)))
    elif cross_section == 'SQUARE':
        hw = width / 2
        base_verts = [
            bm.verts.new((-hw, -hw, 0)),
            bm.verts.new(( hw, -hw, 0)),
            bm.verts.new(( hw,  hw, 0)),
            bm.verts.new((-hw,  hw, 0)),
        ]
    else:  # RECTANGLE
        hw, hd = width / 2, depth / 2
        base_verts = [
            bm.verts.new((-hw, -hd, 0)),
            bm.verts.new(( hw, -hd, 0)),
            bm.verts.new(( hw,  hd, 0)),
            bm.verts.new((-hw,  hd, 0)),
        ]

    bm.verts.ensure_lookup_table()
    base_face = bm.faces.new(base_verts)

    # Extrude to height
    result = bmesh.ops.extrude_face_region(bm, geom=[base_face])
    top_verts = [e for e in result['geom'] if isinstance(e, bmesh.types.BMVert)]
    bmesh.ops.translate(bm, vec=(0, 0, height), verts=top_verts)

    bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])

    mesh = bpy.data.meshes.new(name)
    bm.to_mesh(mesh)
    bm.free()
    mesh.update()

    obj = bpy.data.objects.new(name, mesh)
    obj.location = location
    bpy.context.collection.objects.link(obj)
    return obj

# Usage
col1 = create_column("Column_C01", 'CIRCLE', width=0.5, height=3.6, location=(0, 0, 0))
col2 = create_column("Column_R01", 'RECTANGLE', width=0.4, depth=0.6, height=3.6, location=(5, 0, 0))
```

---

## Example 6: Merge Duplicate Vertices

Cleans up mesh geometry by merging vertices within a tolerance distance.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh

def merge_close_vertices(obj, distance=0.001):
    """Remove duplicate vertices within distance threshold.

    Args:
        obj: bpy.types.Object with mesh data
        distance: Merge distance threshold in meters
    Returns:
        Number of vertices removed
    """
    bm = bmesh.new()
    bm.from_mesh(obj.data)

    original_count = len(bm.verts)
    bmesh.ops.remove_doubles(bm, verts=bm.verts[:], dist=distance)
    removed = original_count - len(bm.verts)

    bm.to_mesh(obj.data)
    bm.free()
    obj.data.update()

    return removed

# Usage
obj = bpy.context.active_object
removed = merge_close_vertices(obj, distance=0.001)
print(f"Removed {removed} duplicate vertices")
```

---

## Example 7: Set Smooth Shading per Face via foreach_set

Applies smooth shading to all faces using bulk operation.

```python
# Blender 3.x/4.x/5.x
import bpy
import numpy as np

def set_all_faces_smooth(obj, smooth=True):
    """Set smooth shading on all faces using foreach_set.

    Args:
        obj: bpy.types.Object with mesh data
        smooth: True for smooth, False for flat
    """
    mesh = obj.data
    count = len(mesh.polygons)
    values = np.full(count, smooth, dtype=bool)
    mesh.polygons.foreach_set("use_smooth", values)
    mesh.update()

# Usage
set_all_faces_smooth(bpy.context.active_object, smooth=True)
```

---

## Example 8: Extract World-Space Vertices with foreach_get

Reads vertex positions in world space using numpy for performance.

```python
# Blender 3.x/4.x/5.x
import bpy
import numpy as np

def get_world_vertices(obj):
    """Get all vertex positions in world coordinates.

    Args:
        obj: bpy.types.Object with mesh data
    Returns:
        numpy array of shape (N, 3) in world space
    """
    mesh = obj.data
    count = len(mesh.vertices)

    # Read local coordinates
    coords = np.empty(count * 3, dtype=np.float64)
    mesh.vertices.foreach_get("co", coords)
    coords = coords.reshape(-1, 3)

    # Apply world matrix
    mat = np.array(obj.matrix_world)  # 4x4 matrix
    # Add homogeneous coordinate
    ones = np.ones((count, 1), dtype=np.float64)
    coords_h = np.hstack([coords, ones])  # (N, 4)

    # Transform
    world_coords = (mat @ coords_h.T).T[:, :3]
    return world_coords

# Usage
obj = bpy.context.active_object
world_verts = get_world_vertices(obj)
print(f"Bounding box: min={world_verts.min(axis=0)}, max={world_verts.max(axis=0)}")
```

---

## Example 9: Create Window Opening with Boolean

Creates a window opening in a wall using BMesh inset operation.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh
from mathutils import Vector

def create_window_opening(wall_obj, face_index, inset=0.3, depth=0.0):
    """Create a window opening by insetting and deleting a face.

    Args:
        wall_obj: bpy.types.Object — the wall mesh
        face_index: Index of the face to cut opening in
        inset: Inset distance from face edges
        depth: Depth of the inset (0 = flat)
    """
    bm = bmesh.new()
    bm.from_mesh(wall_obj.data)
    bm.faces.ensure_lookup_table()

    target_face = bm.faces[face_index]

    # Inset the face
    result = bmesh.ops.inset_individual(
        bm, faces=[target_face],
        thickness=inset, depth=depth,
        use_even_offset=True
    )

    # The inner face (opening) — delete it
    # After inset, the original face becomes the inner face
    bm.faces.ensure_lookup_table()
    # Find the inner face (smallest area among faces sharing the same vertices)
    inner_face = target_face  # inset_individual modifies the original face to be the inner face
    bmesh.ops.delete(bm, geom=[inner_face], context='FACES')

    bm.to_mesh(wall_obj.data)
    bm.free()
    wall_obj.data.update()

# Usage
wall = bpy.data.objects["Wall_001"]
create_window_opening(wall, face_index=0, inset=0.5)
```

---

## Example 10: Add UV Coordinates to Custom Mesh

Assigns planar UV mapping to a mesh using foreach_set.

```python
# Blender 3.x/4.x/5.x
import bpy
import numpy as np

def add_planar_uv(obj, uv_name="UVMap"):
    """Add planar UV mapping projected from Z axis.

    Args:
        obj: bpy.types.Object with mesh data
        uv_name: Name for the UV layer
    """
    mesh = obj.data

    if uv_name not in mesh.uv_layers:
        mesh.uv_layers.new(name=uv_name)

    uv_layer = mesh.uv_layers[uv_name]
    loop_count = len(mesh.loops)

    # Get vertex coordinates for each loop
    loop_vert_indices = np.empty(loop_count, dtype=np.int32)
    mesh.loops.foreach_get("vertex_index", loop_vert_indices)

    vert_count = len(mesh.vertices)
    all_coords = np.empty(vert_count * 3, dtype=np.float64)
    mesh.vertices.foreach_get("co", all_coords)
    all_coords = all_coords.reshape(-1, 3)

    # Use X,Y as UV coordinates (planar projection from top)
    uv_data = np.zeros(loop_count * 2, dtype=np.float32)
    for i, vi in enumerate(loop_vert_indices):
        uv_data[i * 2] = all_coords[vi][0]      # U = X
        uv_data[i * 2 + 1] = all_coords[vi][1]  # V = Y

    uv_layer.data.foreach_set("uv", uv_data)
    mesh.update()

# Usage
add_planar_uv(bpy.context.active_object)
```

---

## Example 11: Building Floor Plan from Coordinates

Creates an extruded floor plan from 2D outline coordinates using BMesh.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh

def create_floor_plan(name, outline_coords, height=3.0):
    """Create extruded floor plan from 2D outline coordinates.

    Args:
        name: Building name
        outline_coords: List of (x, y) tuples defining the floor outline
        height: Wall height in meters
    Returns:
        The created bpy.types.Object
    """
    bm = bmesh.new()
    base_verts = [bm.verts.new((x, y, 0.0)) for x, y in outline_coords]
    bm.verts.ensure_lookup_table()

    # Create floor face
    floor = bm.faces.new(base_verts)

    # Extrude walls
    result = bmesh.ops.extrude_face_region(bm, geom=[floor])
    top_verts = [e for e in result['geom'] if isinstance(e, bmesh.types.BMVert)]
    bmesh.ops.translate(bm, vec=(0, 0, height), verts=top_verts)

    # Recalculate normals (outward)
    bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])

    mesh = bpy.data.meshes.new(name)
    bm.to_mesh(mesh)
    bm.free()
    mesh.update()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj

# Usage
plan = create_floor_plan("Building_001", [(0,0), (10,0), (10,8), (0,8)], height=3.0)
```

---

## Example 12: Mesh Analysis — Area, Volume, Bounds

Calculates mesh statistics for AEC quantity analysis.

```python
# Blender 3.x/4.x/5.x
import bpy
import bmesh
from mathutils import Vector

def analyze_mesh(obj):
    """Calculate mesh statistics for AEC analysis.

    Args:
        obj: bpy.types.Object with mesh data
    Returns:
        Dict with 'total_area', 'volume', 'bounds_min', 'bounds_max',
        'dimensions', 'vert_count', 'face_count'
    """
    bm = bmesh.new()
    bm.from_mesh(obj.data)
    bm.transform(obj.matrix_world)  # Apply world transform

    total_area = sum(f.calc_area() for f in bm.faces)
    volume = bm.calc_volume()  # Only valid for closed (watertight) meshes

    all_coords = [v.co for v in bm.verts]
    bounds_min = Vector((
        min(c.x for c in all_coords),
        min(c.y for c in all_coords),
        min(c.z for c in all_coords),
    ))
    bounds_max = Vector((
        max(c.x for c in all_coords),
        max(c.y for c in all_coords),
        max(c.z for c in all_coords),
    ))

    result = {
        'total_area': total_area,
        'volume': volume,
        'bounds_min': bounds_min,
        'bounds_max': bounds_max,
        'dimensions': bounds_max - bounds_min,
        'vert_count': len(bm.verts),
        'face_count': len(bm.faces),
    }

    bm.free()
    return result

# Usage
stats = analyze_mesh(bpy.context.active_object)
print(f"Area: {stats['total_area']:.2f}m², Volume: {stats['volume']:.2f}m³")
```

---

## Example 13: Bulk Vertex Read with foreach_get

Reads vertex coordinates and normals using foreach_get for high performance.

```python
# Blender 3.x/4.x/5.x
import bpy
import numpy as np

def get_mesh_vertices_fast(mesh):
    """Read all vertex coordinates using foreach_get.

    Args:
        mesh: bpy.types.Mesh
    Returns:
        numpy array of shape (N, 3), dtype float64
    """
    count = len(mesh.vertices)
    coords = np.empty(count * 3, dtype=np.float64)
    mesh.vertices.foreach_get("co", coords)
    return coords.reshape(-1, 3)

def get_mesh_normals_fast(mesh):
    """Read all vertex normals using foreach_get.

    Args:
        mesh: bpy.types.Mesh
    Returns:
        numpy array of shape (N, 3), dtype float64
    """
    count = len(mesh.vertices)
    normals = np.empty(count * 3, dtype=np.float64)
    mesh.vertices.foreach_get("normal", normals)
    return normals.reshape(-1, 3)

# Usage
mesh = bpy.context.active_object.data
coords = get_mesh_vertices_fast(mesh)
normals = get_mesh_normals_fast(mesh)
print(f"Read {len(coords)} vertices, bounding box: {coords.min(axis=0)} to {coords.max(axis=0)}")
```

---

## Example 14: Custom Float Attributes (Blender 3.2+/4.x/5.x)

Adds and sets custom float attributes on mesh elements.

```python
# Blender 3.2+/4.x/5.x
import bpy

def add_custom_float_attribute(mesh, attr_name, domain, values):
    """Add a custom float attribute to mesh.

    Args:
        mesh: bpy.types.Mesh
        attr_name: Attribute name string
        domain: 'POINT', 'EDGE', 'FACE', or 'CORNER'
        values: Flat list/array of float values
    """
    if attr_name not in mesh.attributes:
        mesh.attributes.new(name=attr_name, type='FLOAT', domain=domain)
    attr = mesh.attributes[attr_name]
    attr.data.foreach_set("value", values)
    mesh.update()

# Usage — assign per-face thermal values
mesh = bpy.context.active_object.data
face_count = len(mesh.polygons)
thermal_values = [20.0 + i * 0.5 for i in range(face_count)]
add_custom_float_attribute(mesh, "thermal_zone", "FACE", thermal_values)
```

---

## Example 15: Material Assignment to Specific Faces

Assigns a material to selected faces by polygon index.

```python
# Blender 3.x/4.x/5.x
import bpy

def assign_material_to_faces(obj, material, face_indices):
    """Assign a material to specific faces by index.

    Args:
        obj: bpy.types.Object with mesh data
        material: bpy.types.Material
        face_indices: List of polygon indices
    """
    mesh = obj.data

    # Add material slot if not present
    if material.name not in [s.material.name for s in obj.material_slots if s.material]:
        obj.data.materials.append(material)

    # Find the material index
    mat_idx = list(obj.data.materials).index(material)

    # Assign to faces
    for fi in face_indices:
        mesh.polygons[fi].material_index = mat_idx

# Usage
obj = bpy.context.active_object
mat = bpy.data.materials.new("Wall_Exterior")
mat.diffuse_color = (0.8, 0.75, 0.65, 1.0)
assign_material_to_faces(obj, mat, [0, 1, 2, 3])  # Assign to first 4 faces
```
