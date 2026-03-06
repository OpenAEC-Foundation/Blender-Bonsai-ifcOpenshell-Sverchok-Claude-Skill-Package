---
name: blender-syntax-materials
description: 'Covers Blender material and shader node API including Principled BSDF input name changes (4.0), material slot assignment, UV mapping, texture node setup, material_slot_add, and node-based material creation. Activates when creating materials programmatically, assigning textures, setting up shaders, or migrating material code between Blender versions.'
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-syntax-materials

## Quick Reference

### Critical Warnings

**NEVER** use Blender 3.x Principled BSDF socket names in 4.0+ code. Sockets were renamed in 4.0 (e.g., `"Subsurface"` -> `"Subsurface Weight"`). Using old names raises `KeyError`.

**NEVER** use `mat.blend_method` in Blender 4.2+. This property was removed. Use `mat.surface_render_method` instead.

**NEVER** access UV data per-vertex. UV coordinates are stored per-loop (per face corner). Access via `uv_layers.data[loop_index]`.

**NEVER** call `bpy.data.images.load()` without `check_existing=True`. Omitting it creates duplicate image data blocks on repeated loads.

**NEVER** set `colorspace_settings.name` incorrectly. Use `'sRGB'` for color/diffuse textures and `'Non-Color'` for normal maps, roughness, metallic, and other data textures.

**ALWAYS** set `mat.use_nodes = True` before accessing `mat.node_tree`. Without this, `node_tree` is `None`.

**ALWAYS** store the reference returned by `bpy.data.materials.new()`. Blender may append `.001` if the name already exists.

### Version Decision Tree

```
Creating material code?
├── Target Blender 4.0+ ONLY?
│   ├── YES → Use 4.0+ Principled BSDF socket names directly
│   └── NO → Use version-safe helper (see Pattern 3)
│
Setting transparency/alpha blending?
├── Blender 4.2+?
│   ├── YES → mat.surface_render_method = 'BLENDED'
│   └── NO (3.x / 4.0-4.1) → mat.blend_method = 'BLEND'
│
Connecting texture to Principled BSDF?
├── Color texture (diffuse, base color)?
│   └── Connect to "Base Color" input (all versions)
├── Normal map?
│   └── Connect via ShaderNodeNormalMap → "Normal" input
└── Data texture (roughness, metallic, AO)?
    └── Set image colorspace to 'Non-Color'
```

---

## Essential Patterns

### Pattern 1: Create and Assign a Material

```python
# Blender 3.x/4.x/5.x — create material and assign to object
import bpy

mat = bpy.data.materials.new(name="MyMaterial")
mat.use_nodes = True  # REQUIRED before accessing node_tree

obj = bpy.context.active_object
if obj.data.materials:
    obj.data.materials[0] = mat   # Replace first slot
else:
    obj.data.materials.append(mat)  # Add new slot
```

### Pattern 2: Build a Complete Node Tree

```python
# Blender 3.x/4.x/5.x — node tree from scratch
mat = bpy.data.materials.new("NodeMaterial")
mat.use_nodes = True
tree = mat.node_tree

# Clear defaults
tree.nodes.clear()

# Create nodes
principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.location = (0, 0)

output = tree.nodes.new('ShaderNodeOutputMaterial')
output.location = (400, 0)

# Link BSDF to output
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
```

### Pattern 3: Version-Safe Principled BSDF Input Access

```python
# Blender 3.x/4.0+ — handle renamed sockets
import bpy

def set_principled_value(node, socket_3x, socket_4x, value):
    """Set Principled BSDF input with version-safe socket name."""
    if bpy.app.version >= (4, 0, 0):
        socket_name = socket_4x
    else:
        socket_name = socket_3x
    if socket_name in node.inputs:
        node.inputs[socket_name].default_value = value

# Usage
set_principled_value(principled, "Subsurface", "Subsurface Weight", 0.1)
set_principled_value(principled, "Specular", "Specular IOR Level", 0.5)
set_principled_value(principled, "Transmission", "Transmission Weight", 1.0)
set_principled_value(principled, "Coat", "Coat Weight", 0.3)
set_principled_value(principled, "Sheen", "Sheen Weight", 0.1)
set_principled_value(principled, "Emission", "Emission Color", (1, 1, 1, 1))
```

### Pattern 4: Texture Node with UV Mapping

```python
# Blender 3.x/4.x/5.x — image texture with UV coordinates
tree = mat.node_tree

tex_coord = tree.nodes.new('ShaderNodeTexCoord')
tex_coord.location = (-800, 0)

mapping = tree.nodes.new('ShaderNodeMapping')
mapping.location = (-600, 0)

tex_node = tree.nodes.new('ShaderNodeTexImage')
tex_node.location = (-400, 0)
img = bpy.data.images.load("/path/to/texture.png", check_existing=True)
tex_node.image = img

# Link: TexCoord → Mapping → Image Texture → Principled BSDF
tree.links.new(tex_coord.outputs["UV"], mapping.inputs["Vector"])
tree.links.new(mapping.outputs["Vector"], tex_node.inputs["Vector"])
tree.links.new(tex_node.outputs["Color"], principled.inputs["Base Color"])
```

### Pattern 5: Alpha/Transparency Setup (Version-Dependent)

```python
# Blender 4.2+ — EEVEE transparency
mat.surface_render_method = 'BLENDED'  # 4.2+ ONLY

# Blender 3.x / 4.0-4.1 — EEVEE transparency
# mat.blend_method = 'BLEND'  # REMOVED in 4.2

# Version-safe pattern
if bpy.app.version >= (4, 2, 0):
    mat.surface_render_method = 'BLENDED'
else:
    mat.blend_method = 'BLEND'
```

---

## Principled BSDF Socket Rename Table (3.x -> 4.0+)

| Blender 3.x Name | Blender 4.0+ Name | Notes |
|---|---|---|
| `Base Color` | `Base Color` | Unchanged |
| `Subsurface` | `Subsurface Weight` | Renamed |
| `Subsurface Color` | **REMOVED** | Use `Base Color` instead |
| `Specular` | `Specular IOR Level` | Renamed |
| `Specular Tint` | `Specular Tint` | Type changed: Float -> Color |
| `Transmission` | `Transmission Weight` | Renamed |
| `Coat` | `Coat Weight` | Renamed |
| `Sheen` | `Sheen Weight` | Renamed |
| `Emission` | `Emission Color` | Renamed |
| `Metallic` | `Metallic` | Unchanged |
| `Roughness` | `Roughness` | Unchanged |
| `IOR` | `IOR` | Unchanged |
| `Alpha` | `Alpha` | Unchanged |
| `Normal` | `Normal` | Unchanged |

---

## Common Operations

### Assign Material to Specific Faces

```python
# Blender 3.x/4.x/5.x — multi-material assignment
import bpy

obj = bpy.context.active_object

# Add two materials to object
mat_a = bpy.data.materials.new("Material_A")
mat_b = bpy.data.materials.new("Material_B")
obj.data.materials.append(mat_a)
obj.data.materials.append(mat_b)

# Assign material index per polygon
for polygon in obj.data.polygons:
    if polygon.center.z > 0:
        polygon.material_index = 1  # Material_B
    else:
        polygon.material_index = 0  # Material_A
```

### Add Material Slot via Operator

```python
# Blender 3.x/4.x/5.x — operator-based slot management
import bpy

obj = bpy.context.active_object
bpy.context.view_layer.objects.active = obj

# Add empty slot
bpy.ops.object.material_slot_add()

# Assign material to active slot
obj.material_slots[obj.active_material_index].material = mat

# Remove active slot
bpy.ops.object.material_slot_remove()
```

### Access Default Principled BSDF

When `use_nodes = True` is set on a new material, Blender creates a default Principled BSDF and Material Output automatically.

```python
# Blender 3.x/4.x/5.x — access default nodes without clearing
mat = bpy.data.materials.new("QuickMaterial")
mat.use_nodes = True
principled = mat.node_tree.nodes.get("Principled BSDF")
# principled is the auto-created ShaderNodeBsdfPrincipled
```

### UV Layer Management

```python
# Blender 3.x/4.x/5.x — UV layer access
mesh = obj.data

# List UV layers
for uv_layer in mesh.uv_layers:
    print(uv_layer.name)

# Get active UV layer
active_uv = mesh.uv_layers.active

# Create new UV layer
new_uv = mesh.uv_layers.new(name="ProjectionUV")

# Read UV coordinates (per-loop, NOT per-vertex)
for loop_idx in range(len(mesh.loops)):
    uv = active_uv.data[loop_idx].uv
    # uv.x, uv.y are the coordinates

# Modify UV coordinates
for loop_idx in range(len(mesh.loops)):
    active_uv.data[loop_idx].uv.x *= 2.0
    active_uv.data[loop_idx].uv.y *= 2.0
```

### Load and Configure Image Textures

```python
# Blender 3.x/4.x/5.x — image loading
import bpy

# Load from file (reuse existing if already loaded)
img = bpy.data.images.load("/path/to/texture.png", check_existing=True)

# Create new blank image (for baking)
img = bpy.data.images.new("BakeTarget", width=2048, height=2048)

# Configure texture node
tex_node = tree.nodes.new('ShaderNodeTexImage')
tex_node.image = img
tex_node.interpolation = 'Linear'   # 'Linear', 'Closest', 'Cubic', 'Smart'
tex_node.projection = 'FLAT'        # 'FLAT', 'BOX', 'SPHERE', 'TUBE'
tex_node.extension = 'REPEAT'       # 'REPEAT', 'EXTEND', 'CLIP'

# Set color space
img.colorspace_settings.name = 'sRGB'       # For diffuse/color textures
# img.colorspace_settings.name = 'Non-Color' # For normal, roughness, metallic
```

### Normal Map Setup

```python
# Blender 3.x/4.x/5.x — normal map node chain
tree = mat.node_tree

normal_tex = tree.nodes.new('ShaderNodeTexImage')
normal_tex.image = bpy.data.images.load("/path/to/normal.png", check_existing=True)
normal_tex.image.colorspace_settings.name = 'Non-Color'  # REQUIRED for normal maps

normal_map = tree.nodes.new('ShaderNodeNormalMap')

tree.links.new(normal_tex.outputs["Color"], normal_map.inputs["Color"])
tree.links.new(normal_map.outputs["Normal"], principled.inputs["Normal"])
```

### Remove All Materials from Object

```python
# Blender 3.x/4.x/5.x — clear all material slots
obj.data.materials.clear()
```

---

## Common Shader Node Types

| Node Type String | Class | Purpose |
|---|---|---|
| `'ShaderNodeBsdfPrincipled'` | Principled BSDF | Main PBR shader |
| `'ShaderNodeOutputMaterial'` | Material Output | Final output |
| `'ShaderNodeTexImage'` | Image Texture | Load image files |
| `'ShaderNodeTexCoord'` | Texture Coordinate | UV, Object, Generated coords |
| `'ShaderNodeMapping'` | Mapping | Transform texture coordinates |
| `'ShaderNodeNormalMap'` | Normal Map | Convert image to normal vector |
| `'ShaderNodeMixShader'` | Mix Shader | Blend two shaders |
| `'ShaderNodeBsdfTransparent'` | Transparent BSDF | Transparency shader |
| `'ShaderNodeBsdfGlass'` | Glass BSDF | Glass shader |
| `'ShaderNodeMath'` | Math | Scalar math operations |
| `'ShaderNodeMixRGB'` | Mix (Color) | Blend colors (3.x name) |
| `'ShaderNodeMix'` | Mix | Blend colors/vectors (4.0+) |
| `'ShaderNodeValToRGB'` | Color Ramp | Map value to color gradient |
| `'ShaderNodeBump'` | Bump | Generate bump from height |
| `'ShaderNodeSeparateXYZ'` | Separate XYZ | Split vector components |
| `'ShaderNodeCombineXYZ'` | Combine XYZ | Merge components to vector |
| `'ShaderNodeTexNoise'` | Noise Texture | Procedural noise |
| `'ShaderNodeTexVoronoi'` | Voronoi Texture | Procedural voronoi |
| `'ShaderNodeGroup'` | Node Group | Reusable node group instance |

---

## EEVEE vs Cycles Material Differences

| Property | EEVEE | Cycles | Notes |
|---|---|---|---|
| Transparency | Requires `surface_render_method` (4.2+) or `blend_method` (3.x-4.1) | Automatic from Alpha | EEVEE needs explicit config |
| Screen Space Refraction | `mat.use_screen_refraction = True` | Not needed | EEVEE-specific |
| Backface Culling | `mat.use_backface_culling` | `mat.use_backface_culling` | Same API |
| Displacement | `mat.cycles.displacement_method` | `mat.cycles.displacement_method` | Cycles-only |
| Shadow Mode | `mat.shadow_method` (3.x-4.1) | Automatic | EEVEE-specific |

---

## Version-Specific Migration Rules

| Feature | Blender 3.x | Blender 4.0+ | Blender 4.2+ |
|---|---|---|---|
| Principled BSDF sockets | Old names | New names (see table) | Same as 4.0 |
| Alpha blending (EEVEE) | `mat.blend_method = 'BLEND'` | `mat.blend_method = 'BLEND'` | `mat.surface_render_method = 'BLENDED'` |
| Shadow method (EEVEE) | `mat.shadow_method` | `mat.shadow_method` | Property restructured |
| Mix node | `'ShaderNodeMixRGB'` | `'ShaderNodeMix'` (preferred) | `'ShaderNodeMix'` |
| Specular Tint type | Float (0.0-1.0) | Color (RGBA) | Color (RGBA) |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for Material, NodeTree, ShaderNode, UV layers, Image
- [references/examples.md](references/examples.md) — Working code examples for common material workflows
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with materials, with WHY explanations

### Official Sources

- https://docs.blender.org/api/current/bpy.types.Material.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeBsdfPrincipled.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeTree.html
- https://docs.blender.org/api/current/bpy.types.UVLoopLayers.html
- https://docs.blender.org/api/current/bpy.types.Image.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
