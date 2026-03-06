# blender-syntax-materials: Working Code Examples

All examples verified against official Blender Python API documentation:
- https://docs.blender.org/api/current/bpy.types.Material.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeBsdfPrincipled.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeTree.html
- https://docs.blender.org/api/current/bpy.types.Image.html

---

## Example 1: Create Basic Material and Assign to Object

```python
# Blender 3.x/4.x/5.x — minimal material creation
import bpy

# Create material
mat = bpy.data.materials.new(name="BasicRed")
mat.use_nodes = True

# Set color on the default Principled BSDF
principled = mat.node_tree.nodes.get("Principled BSDF")
if principled:
    principled.inputs["Base Color"].default_value = (0.8, 0.1, 0.1, 1.0)
    principled.inputs["Roughness"].default_value = 0.4

# Assign to active object
obj = bpy.context.active_object
if obj is not None and hasattr(obj.data, 'materials'):
    if obj.data.materials:
        obj.data.materials[0] = mat
    else:
        obj.data.materials.append(mat)
```

---

## Example 2: Full PBR Material with Textures (Blender 4.0+)

```python
# Blender 4.0+ — complete PBR material with diffuse, roughness, normal
import bpy
import os

def create_pbr_material(name, texture_dir):
    """Create a PBR material from texture files in a directory.

    Expects files: diffuse.png, roughness.png, normal.png in texture_dir.
    """
    mat = bpy.data.materials.new(name=name)
    mat.use_nodes = True
    tree = mat.node_tree
    tree.nodes.clear()

    # Create nodes
    principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
    principled.location = (0, 0)

    output = tree.nodes.new('ShaderNodeOutputMaterial')
    output.location = (400, 0)

    tex_coord = tree.nodes.new('ShaderNodeTexCoord')
    tex_coord.location = (-1000, 0)

    mapping = tree.nodes.new('ShaderNodeMapping')
    mapping.location = (-800, 0)

    # Link UV → Mapping
    tree.links.new(tex_coord.outputs["UV"], mapping.inputs["Vector"])

    # Diffuse texture
    diffuse_path = os.path.join(texture_dir, "diffuse.png")
    if os.path.exists(diffuse_path):
        diffuse_tex = tree.nodes.new('ShaderNodeTexImage')
        diffuse_tex.location = (-400, 200)
        diffuse_tex.image = bpy.data.images.load(diffuse_path, check_existing=True)
        diffuse_tex.image.colorspace_settings.name = 'sRGB'
        tree.links.new(mapping.outputs["Vector"], diffuse_tex.inputs["Vector"])
        tree.links.new(diffuse_tex.outputs["Color"], principled.inputs["Base Color"])

    # Roughness texture
    roughness_path = os.path.join(texture_dir, "roughness.png")
    if os.path.exists(roughness_path):
        roughness_tex = tree.nodes.new('ShaderNodeTexImage')
        roughness_tex.location = (-400, -100)
        roughness_tex.image = bpy.data.images.load(roughness_path, check_existing=True)
        roughness_tex.image.colorspace_settings.name = 'Non-Color'
        tree.links.new(mapping.outputs["Vector"], roughness_tex.inputs["Vector"])
        tree.links.new(roughness_tex.outputs["Color"], principled.inputs["Roughness"])

    # Normal map
    normal_path = os.path.join(texture_dir, "normal.png")
    if os.path.exists(normal_path):
        normal_tex = tree.nodes.new('ShaderNodeTexImage')
        normal_tex.location = (-400, -400)
        normal_tex.image = bpy.data.images.load(normal_path, check_existing=True)
        normal_tex.image.colorspace_settings.name = 'Non-Color'

        normal_map = tree.nodes.new('ShaderNodeNormalMap')
        normal_map.location = (-150, -400)

        tree.links.new(mapping.outputs["Vector"], normal_tex.inputs["Vector"])
        tree.links.new(normal_tex.outputs["Color"], normal_map.inputs["Color"])
        tree.links.new(normal_map.outputs["Normal"], principled.inputs["Normal"])

    # Link BSDF → Output
    tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])

    return mat
```

---

## Example 3: AEC Glass Material (Blender 4.2+)

```python
# Blender 4.2+ — glass material for AEC visualization
import bpy

def create_glass_material(name="AEC_Glass", ior=1.45, transmission=1.0,
                          roughness=0.0, color=(0.8, 0.9, 1.0, 1.0)):
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    tree = mat.node_tree
    principled = tree.nodes.get("Principled BSDF")

    principled.inputs["Base Color"].default_value = color
    principled.inputs["Transmission Weight"].default_value = transmission
    principled.inputs["IOR"].default_value = ior
    principled.inputs["Roughness"].default_value = roughness
    principled.inputs["Alpha"].default_value = 0.3

    # EEVEE transparency — Blender 4.2+
    mat.surface_render_method = 'BLENDED'

    return mat


def create_glass_material_versionsafe(name="AEC_Glass", ior=1.45):
    """Version-safe glass material for Blender 3.x through 5.x."""
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    principled = mat.node_tree.nodes.get("Principled BSDF")

    principled.inputs["Base Color"].default_value = (0.8, 0.9, 1.0, 1.0)
    principled.inputs["IOR"].default_value = ior
    principled.inputs["Roughness"].default_value = 0.0
    principled.inputs["Alpha"].default_value = 0.3

    # Handle renamed sockets
    if bpy.app.version >= (4, 0, 0):
        principled.inputs["Transmission Weight"].default_value = 1.0
    else:
        principled.inputs["Transmission"].default_value = 1.0

    # Handle EEVEE transparency changes
    if bpy.app.version >= (4, 2, 0):
        mat.surface_render_method = 'BLENDED'
    else:
        mat.blend_method = 'BLEND'

    return mat
```

---

## Example 4: Multi-Material Object

```python
# Blender 3.x/4.x/5.x — assign different materials to different faces
import bpy

obj = bpy.context.active_object
if obj is None or obj.type != 'MESH':
    raise RuntimeError("Select a mesh object")

# Create materials
mat_wall = bpy.data.materials.new("AEC_Wall")
mat_wall.use_nodes = True
mat_wall.node_tree.nodes.get("Principled BSDF").inputs["Base Color"].default_value = (0.9, 0.85, 0.8, 1.0)

mat_floor = bpy.data.materials.new("AEC_Floor")
mat_floor.use_nodes = True
mat_floor.node_tree.nodes.get("Principled BSDF").inputs["Base Color"].default_value = (0.4, 0.35, 0.3, 1.0)

# Assign materials to object
obj.data.materials.append(mat_wall)   # Index 0
obj.data.materials.append(mat_floor)  # Index 1

# Assign based on face normal direction
for polygon in obj.data.polygons:
    if abs(polygon.normal.z) > 0.9:
        # Horizontal face (floor/ceiling)
        polygon.material_index = 1
    else:
        # Vertical face (wall)
        polygon.material_index = 0
```

---

## Example 5: Procedural Material with Noise Texture

```python
# Blender 3.x/4.x/5.x — procedural noise-based material
import bpy

mat = bpy.data.materials.new("ProceduralStone")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

# Create nodes
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (200, 0)

output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (600, 0)

noise = tree.nodes.new('ShaderNodeTexNoise')
noise.location = (-400, 100)
noise.inputs["Scale"].default_value = 5.0
noise.inputs["Detail"].default_value = 8.0

color_ramp = tree.nodes.new('ShaderNodeValToRGB')
color_ramp.location = (-100, 100)
# Configure color ramp stops
cr = color_ramp.color_ramp
cr.elements[0].color = (0.3, 0.28, 0.25, 1.0)
cr.elements[1].color = (0.6, 0.55, 0.5, 1.0)

tex_coord = tree.nodes.new('ShaderNodeTexCoord')
tex_coord.location = (-600, 0)

# Link nodes
tree.links.new(tex_coord.outputs["Object"], noise.inputs["Vector"])
tree.links.new(noise.outputs["Fac"], color_ramp.inputs["Fac"])
tree.links.new(color_ramp.outputs["Color"], principled.inputs["Base Color"])
tree.links.new(noise.outputs["Fac"], principled.inputs["Roughness"])
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
```

---

## Example 6: Emission Material

```python
# Blender 4.0+ — emissive material (e.g., for light fixtures)
import bpy

def create_emission_material(name="LightEmit", color=(1.0, 0.95, 0.8, 1.0),
                              strength=10.0):
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    principled = mat.node_tree.nodes.get("Principled BSDF")

    principled.inputs["Emission Color"].default_value = color
    principled.inputs["Emission Strength"].default_value = strength

    return mat
```

---

## Example 7: UV Projection via Operator

```python
# Blender 3.x/4.x/5.x — project UVs on selected object
import bpy

obj = bpy.context.active_object
if obj is None or obj.type != 'MESH':
    raise RuntimeError("Select a mesh object")

# Ensure object mode, then switch to edit mode
bpy.ops.object.mode_set(mode='OBJECT')
bpy.context.view_layer.objects.active = obj
obj.select_set(True)

bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')

# Apply cube projection
bpy.ops.uv.cube_project(cube_size=1.0)

# Return to object mode
bpy.ops.object.mode_set(mode='OBJECT')
```

---

## Example 8: Copy Material Between Objects

```python
# Blender 3.x/4.x/5.x — copy materials from source to target
import bpy

def copy_materials(source_obj, target_obj):
    """Copy all materials from source to target, replacing target's materials."""
    target_obj.data.materials.clear()
    for mat_slot in source_obj.material_slots:
        if mat_slot.material:
            target_obj.data.materials.append(mat_slot.material)
        else:
            target_obj.data.materials.append(None)

# Usage
source = bpy.data.objects.get("SourceObject")
target = bpy.data.objects.get("TargetObject")
if source and target:
    copy_materials(source, target)
```

---

## Example 9: Batch Material Assignment

```python
# Blender 3.x/4.x/5.x — assign material to multiple objects
import bpy

mat = bpy.data.materials.get("SharedMaterial")
if mat is None:
    mat = bpy.data.materials.new("SharedMaterial")
    mat.use_nodes = True

# Assign to all selected mesh objects
for obj in bpy.context.selected_objects:
    if obj.type == 'MESH':
        if obj.data.materials:
            obj.data.materials[0] = mat
        else:
            obj.data.materials.append(mat)
```

---

## Example 10: Material with Node Group

```python
# Blender 3.x/4.x/5.x — use a reusable node group in a material
import bpy

mat = bpy.data.materials.new("GroupMaterial")
mat.use_nodes = True
tree = mat.node_tree

# Reference existing node group
group_name = "PBR_Setup"
node_group = bpy.data.node_groups.get(group_name)
if node_group is None:
    raise RuntimeError(f"Node group '{group_name}' not found")

# Create group node instance
group_node = tree.nodes.new('ShaderNodeGroup')
group_node.node_tree = node_group
group_node.location = (-200, 0)

# Connect group output to Principled BSDF (depends on group interface)
principled = tree.nodes.get("Principled BSDF")
if "Color" in group_node.outputs and principled:
    tree.links.new(group_node.outputs["Color"], principled.inputs["Base Color"])
```

---

## Example 11: Read Material Properties from Object

```python
# Blender 3.x/4.x/5.x — inspect materials on an object
import bpy

obj = bpy.context.active_object
if obj is None or not hasattr(obj.data, 'materials'):
    raise RuntimeError("Select an object with materials")

for i, mat_slot in enumerate(obj.material_slots):
    mat = mat_slot.material
    if mat is None:
        print(f"Slot {i}: Empty")
        continue

    print(f"Slot {i}: {mat.name}")
    print(f"  use_nodes: {mat.use_nodes}")
    print(f"  users: {mat.users}")

    if mat.use_nodes and mat.node_tree:
        for node in mat.node_tree.nodes:
            print(f"  Node: {node.bl_idname} ({node.name})")
            if node.bl_idname == 'ShaderNodeBsdfPrincipled':
                bc = node.inputs["Base Color"].default_value
                print(f"    Base Color: ({bc[0]:.2f}, {bc[1]:.2f}, {bc[2]:.2f})")
```

---

## Example 12: Version-Safe Complete Material Setup

```python
# Blender 3.x/4.x/5.x — version-safe material with all common settings
import bpy


def set_principled_value(node, socket_3x, socket_4x, value):
    """Set Principled BSDF input, handling 3.x vs 4.0+ name differences."""
    if bpy.app.version >= (4, 0, 0):
        socket_name = socket_4x
    else:
        socket_name = socket_3x
    if socket_name in node.inputs:
        node.inputs[socket_name].default_value = value


def create_material_versionsafe(name, base_color=(0.8, 0.8, 0.8, 1.0),
                                 metallic=0.0, roughness=0.5,
                                 transmission=0.0, emission_strength=0.0):
    """Create material that works across Blender 3.x, 4.x, and 5.x."""
    mat = bpy.data.materials.new(name=name)
    mat.use_nodes = True
    principled = mat.node_tree.nodes.get("Principled BSDF")

    if principled is None:
        raise RuntimeError("Default Principled BSDF not found")

    # Unchanged sockets (same name in all versions)
    principled.inputs["Base Color"].default_value = base_color
    principled.inputs["Metallic"].default_value = metallic
    principled.inputs["Roughness"].default_value = roughness

    # Renamed sockets (version-safe)
    set_principled_value(principled, "Transmission", "Transmission Weight", transmission)
    set_principled_value(principled, "Emission", "Emission Color", (1, 1, 1, 1))

    if emission_strength > 0:
        principled.inputs["Emission Strength"].default_value = emission_strength

    return mat
```
