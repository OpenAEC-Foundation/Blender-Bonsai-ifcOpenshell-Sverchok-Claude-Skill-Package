# blender-syntax-materials: API Method Reference

Sources:
- https://docs.blender.org/api/current/bpy.types.Material.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeBsdfPrincipled.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeTree.html
- https://docs.blender.org/api/current/bpy.types.Image.html
- https://docs.blender.org/api/current/bpy.types.UVLoopLayers.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/

---

## bpy.data.materials — Material Collection

```python
# Blender 3.x/4.x/5.x
bpy.data.materials                          # bpy_prop_collection of Material
bpy.data.materials.new(name: str)           # -> Material (create new)
bpy.data.materials.remove(material: Material, do_unlink: bool = True)  # Delete
bpy.data.materials.get(name: str)           # -> Material | None
bpy.data.materials[name: str]               # -> Material (KeyError if not found)
len(bpy.data.materials)                     # -> int
```

---

## bpy.types.Material — Material Properties

### Core Properties (all versions: 3.x/4.x/5.x)

| Property | Type | Description |
|---|---|---|
| `mat.name` | `str` | Material name (unique within bpy.data.materials) |
| `mat.use_nodes` | `bool` | Enable shader node tree. MUST be True to access node_tree |
| `mat.node_tree` | `ShaderNodeTree \| None` | Shader node tree. None if use_nodes is False |
| `mat.diffuse_color` | `float[4]` (RGBA) | Viewport display color (not render color) |
| `mat.specular_intensity` | `float` | Viewport specular intensity |
| `mat.roughness` | `float` | Viewport roughness |
| `mat.metallic` | `float` | Viewport metallic (4.0+) |
| `mat.use_backface_culling` | `bool` | Cull backfaces in viewport and EEVEE |
| `mat.use_fake_user` | `bool` | Prevent purge when users == 0 |
| `mat.users` | `int` (read-only) | Number of users referencing this material |
| `mat.pass_index` | `int` | Pass index for compositing |

### EEVEE-Specific Properties

```python
# Blender 3.x / 4.0-4.1 ONLY — REMOVED in 4.2
mat.blend_method           # str enum: 'OPAQUE', 'CLIP', 'HASHED', 'BLEND'
mat.shadow_method          # str enum: 'NONE', 'OPAQUE', 'CLIP', 'HASHED'
mat.alpha_threshold        # float: alpha clip threshold (for 'CLIP' method)
mat.use_screen_refraction  # bool: enable screen-space refraction

# Blender 4.2+ — replaces blend_method
mat.surface_render_method  # str enum: 'DITHERED', 'BLENDED'
# 'DITHERED' = default opaque (replaces OPAQUE/CLIP/HASHED)
# 'BLENDED' = alpha blending (replaces BLEND)
```

### Cycles-Specific Properties

```python
# Blender 3.x/4.x/5.x — via mat.cycles
mat.cycles.displacement_method  # str enum: 'BUMP', 'DISPLACEMENT', 'BOTH'
mat.cycles.volume_sampling      # str enum: 'DISTANCE', 'EQUIANGULAR', 'MULTIPLE_IMPORTANCE'
mat.cycles.volume_interpolation # str enum: 'LINEAR', 'CUBIC'
mat.cycles.homogeneous_volume   # bool
```

---

## bpy.types.ShaderNodeTree — Shader Node Tree

### Properties

| Property | Type | Description |
|---|---|---|
| `tree.nodes` | `Nodes` collection | All shader nodes in this tree |
| `tree.links` | `NodeLinks` collection | All links between node sockets |
| `tree.type` | `str` (read-only) | Always `'SHADER'` for material trees |

### Node Management

```python
# Blender 3.x/4.x/5.x
tree = mat.node_tree

# Create node
node = tree.nodes.new(type: str)              # -> ShaderNode
# type: 'ShaderNodeBsdfPrincipled', 'ShaderNodeTexImage', etc.

# Remove node
tree.nodes.remove(node: ShaderNode)

# Clear all nodes
tree.nodes.clear()

# Access by name
node = tree.nodes.get("Principled BSDF")      # -> ShaderNode | None
node = tree.nodes["Principled BSDF"]           # KeyError if not found

# Active node
tree.nodes.active = node                       # Set active node
active = tree.nodes.active                     # Get active node
```

### Link Management

```python
# Blender 3.x/4.x/5.x
# Create link between output socket and input socket
link = tree.links.new(
    input: NodeSocket,     # The OUTPUT socket (source)
    output: NodeSocket     # The INPUT socket (destination)
)
# NOTE: Parameter names are counterintuitive.
# First arg is the source (output), second is the destination (input).

# Remove link
tree.links.remove(link: NodeLink)

# Clear all links
tree.links.clear()
```

### Node Common Properties

```python
# Blender 3.x/4.x/5.x — all ShaderNode subclasses share these
node.location          # float[2]: (x, y) position in editor
node.name              # str: node name (unique within tree)
node.label             # str: display label (can be empty)
node.hide              # bool: collapse node in editor
node.mute              # bool: bypass node (pass through)
node.inputs            # NodeInputs collection of NodeSocket
node.outputs           # NodeOutputs collection of NodeSocket
node.bl_idname         # str (read-only): node type identifier
```

---

## ShaderNodeBsdfPrincipled — Principled BSDF

### Input Sockets (Blender 4.0+)

| Socket Name | Type | Default | Description |
|---|---|---|---|
| `Base Color` | Color | (0.8, 0.8, 0.8, 1.0) | Surface base color |
| `Metallic` | Float | 0.0 | Metallic factor (0=dielectric, 1=metal) |
| `Roughness` | Float | 0.5 | Microsurface roughness |
| `IOR` | Float | 1.45 | Index of refraction |
| `Alpha` | Float | 1.0 | Surface transparency |
| `Normal` | Vector | — | Normal perturbation input |
| `Subsurface Weight` | Float | 0.0 | Subsurface scattering amount |
| `Subsurface Radius` | Vector | (1.0, 0.2, 0.1) | SSS per-channel radius |
| `Subsurface Scale` | Float | 0.05 | SSS overall scale |
| `Subsurface Anisotropy` | Float | 0.0 | SSS directionality |
| `Specular IOR Level` | Float | 0.5 | Specular reflection level |
| `Specular Tint` | Color | (1.0, 1.0, 1.0, 1.0) | Specular tint color (was Float in 3.x) |
| `Anisotropic` | Float | 0.0 | Anisotropic reflection amount |
| `Anisotropic Rotation` | Float | 0.0 | Anisotropic rotation angle |
| `Tangent` | Vector | — | Tangent vector for anisotropy |
| `Transmission Weight` | Float | 0.0 | Glass-like transmission |
| `Coat Weight` | Float | 0.0 | Clear coat layer intensity |
| `Coat Roughness` | Float | 0.03 | Clear coat roughness |
| `Coat IOR` | Float | 1.5 | Clear coat IOR |
| `Coat Tint` | Color | (1.0, 1.0, 1.0, 1.0) | Clear coat tint |
| `Coat Normal` | Vector | — | Clear coat normal |
| `Sheen Weight` | Float | 0.0 | Sheen layer intensity |
| `Sheen Roughness` | Float | 0.5 | Sheen roughness |
| `Sheen Tint` | Color | (1.0, 1.0, 1.0, 1.0) | Sheen tint color |
| `Emission Color` | Color | (1.0, 1.0, 1.0, 1.0) | Emission color |
| `Emission Strength` | Float | 0.0 | Emission intensity |
| `Weight` | Float | — | Mix weight (for shader mixing) |

### Output Sockets

| Socket Name | Type | Description |
|---|---|---|
| `BSDF` | Shader | Combined shader output |

### Accessing Sockets

```python
# Blender 3.x/4.x/5.x — access by name or index
node = tree.nodes["Principled BSDF"]

# By name (PREFERRED — version-dependent, see rename table in SKILL.md)
node.inputs["Base Color"].default_value = (0.8, 0.2, 0.1, 1.0)
node.inputs["Metallic"].default_value = 1.0
node.inputs["Roughness"].default_value = 0.3

# By index (fragile — index changes between versions, AVOID)
node.inputs[0].default_value = (0.8, 0.2, 0.1, 1.0)  # NOT RECOMMENDED

# Check if socket exists before access
if "Subsurface Weight" in node.inputs:
    node.inputs["Subsurface Weight"].default_value = 0.1
```

---

## ShaderNodeTexImage — Image Texture

### Properties

| Property | Type | Description |
|---|---|---|
| `tex_node.image` | `Image \| None` | Assigned image data block |
| `tex_node.interpolation` | `str` | `'Linear'`, `'Closest'`, `'Cubic'`, `'Smart'` |
| `tex_node.projection` | `str` | `'FLAT'`, `'BOX'`, `'SPHERE'`, `'TUBE'` |
| `tex_node.projection_blend` | `float` | Blend factor for BOX projection |
| `tex_node.extension` | `str` | `'REPEAT'`, `'EXTEND'`, `'CLIP'` |

### Output Sockets

| Socket Name | Type | Description |
|---|---|---|
| `Color` | Color | Image color output |
| `Alpha` | Float | Image alpha channel |

### Input Sockets

| Socket Name | Type | Description |
|---|---|---|
| `Vector` | Vector | Texture coordinate input |

---

## bpy.types.Image — Image Data

```python
# Blender 3.x/4.x/5.x — load image
img = bpy.data.images.load(
    filepath: str,            # Absolute or Blender-relative path
    check_existing: bool = False  # Reuse if already loaded (ALWAYS use True)
)  # -> Image

# Create new blank image
img = bpy.data.images.new(
    name: str,
    width: int,
    height: int,
    alpha: bool = False,
    float_buffer: bool = False,
    is_data: bool = False
)  # -> Image
```

### Image Properties

| Property | Type | Description |
|---|---|---|
| `img.filepath` | `str` | File path (may be Blender-relative `//`) |
| `img.filepath_raw` | `str` | Raw file path |
| `img.name` | `str` | Image data block name |
| `img.size` | `int[2]` (read-only) | Width, height in pixels |
| `img.colorspace_settings.name` | `str` | Color space: `'sRGB'`, `'Non-Color'`, `'Linear'`, etc. |
| `img.alpha_mode` | `str` | `'STRAIGHT'`, `'PREMUL'`, `'CHANNEL_PACKED'`, `'NONE'` |
| `img.source` | `str` | `'FILE'`, `'GENERATED'`, `'MOVIE'`, `'SEQUENCE'` |
| `img.type` | `str` (read-only) | `'IMAGE'`, `'UV_TEST'`, `'RENDER_RESULT'`, `'COMPOSITING'` |
| `img.pixels` | `float[]` | Raw pixel data (RGBA floats, flattened) |
| `img.is_dirty` | `bool` (read-only) | True if modified and unsaved |

### Image Methods

```python
# Blender 3.x/4.x/5.x
img.pack()                  # Pack image into .blend file
img.unpack(method='USE_LOCAL')  # Unpack to disk
img.save()                  # Save to filepath
img.save_render(filepath: str)  # Save render result
img.reload()                # Reload from disk
img.update()                # Update image from pixels array
img.gl_free()               # Free OpenGL image texture (deprecated in 5.0)
```

---

## UV Layer Access — bpy.types.MeshUVLoopLayer

### Collection Access

```python
# Blender 3.x/4.x/5.x
mesh = obj.data

mesh.uv_layers                    # MeshUVLoopLayer collection
mesh.uv_layers.active             # Active UV layer (MeshUVLoopLayer | None)
mesh.uv_layers.active_index       # int: active layer index
mesh.uv_layers.new(name: str = "UVMap")  # -> MeshUVLoopLayer (create new)
mesh.uv_layers.remove(layer: MeshUVLoopLayer)  # Remove UV layer
```

### UV Data Access

```python
# Blender 3.x/4.x/5.x
uv_layer = mesh.uv_layers.active

# CRITICAL: UV data is per-loop (per face corner), NOT per-vertex
# A vertex shared by N faces has N separate UV coordinates

# Access individual UV coordinate
uv_data = uv_layer.data[loop_index]  # MeshUVLoop
uv_data.uv                           # float[2]: (u, v) coordinates
uv_data.uv.x                         # float: U coordinate
uv_data.uv.y                         # float: V coordinate

# Total UV entries = total loops (face corners)
count = len(uv_layer.data)  # == len(mesh.loops)

# Map polygon to UV coordinates
for poly in mesh.polygons:
    for loop_idx in poly.loop_indices:
        uv = uv_layer.data[loop_idx].uv
        # This UV belongs to this polygon's corner
```

---

## Object Material Slots

```python
# Blender 3.x/4.x/5.x
obj.data.materials                         # IDMaterials collection
obj.data.materials.append(mat: Material)   # Add material to end
obj.data.materials.pop(index: int = -1)    # Remove by index
obj.data.materials.clear()                 # Remove all materials
obj.data.materials[index] = mat            # Replace at index
len(obj.data.materials)                    # Number of material slots

# Material slot properties
obj.material_slots                          # MaterialSlot collection (read-only)
obj.material_slots[index].material          # Material | None
obj.material_slots[index].link              # 'OBJECT' or 'DATA'
obj.active_material                         # Active material (shortcut)
obj.active_material_index                   # int: active slot index

# Operator-based management
bpy.ops.object.material_slot_add()          # Add empty slot
bpy.ops.object.material_slot_remove()       # Remove active slot
bpy.ops.object.material_slot_move(direction='UP')   # Reorder
bpy.ops.object.material_slot_move(direction='DOWN')
```

---

## Additional Shader Node Types

### ShaderNodeNormalMap

```python
# Blender 3.x/4.x/5.x
normal_map = tree.nodes.new('ShaderNodeNormalMap')
normal_map.space           # str: 'TANGENT', 'OBJECT', 'WORLD', 'BLENDER_OBJECT', 'BLENDER_WORLD'
normal_map.uv_map          # str: name of UV map to use (for TANGENT space)
# Inputs: "Strength" (Float), "Color" (Color)
# Outputs: "Normal" (Vector)
```

### ShaderNodeMapping

```python
# Blender 3.x/4.x/5.x
mapping = tree.nodes.new('ShaderNodeMapping')
mapping.vector_type        # str: 'POINT', 'TEXTURE', 'VECTOR', 'NORMAL'
# Inputs: "Vector", "Location", "Rotation", "Scale"
# Outputs: "Vector"
```

### ShaderNodeTexCoord

```python
# Blender 3.x/4.x/5.x
tex_coord = tree.nodes.new('ShaderNodeTexCoord')
tex_coord.object           # Object | None: reference object for Object output
# Outputs: "Generated", "Normal", "UV", "Object", "Camera", "Window", "Reflection"
```

### ShaderNodeGroup

```python
# Blender 3.x/4.x/5.x
group_node = tree.nodes.new('ShaderNodeGroup')
group_node.node_tree = bpy.data.node_groups["MyNodeGroup"]
# Inputs/outputs defined by the node group's interface
```
