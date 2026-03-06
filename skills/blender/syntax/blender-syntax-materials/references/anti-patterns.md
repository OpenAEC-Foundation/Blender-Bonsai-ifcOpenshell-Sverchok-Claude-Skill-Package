# blender-syntax-materials: Anti-Patterns

These are confirmed error patterns for Blender material and shader node scripting. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://docs.blender.org/api/current/bpy.types.Material.html
- https://docs.blender.org/api/current/bpy.types.ShaderNodeBsdfPrincipled.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.2/python_api/
- vooronderzoek-blender.md §3.7 (Materials & Shading Anti-Patterns)

---

## AP-MAT-001: Using 3.x Principled BSDF Socket Names in 4.0+

**WHY this is wrong**: Blender 4.0 renamed multiple Principled BSDF input sockets. Using old names raises `KeyError` because the socket lookup fails. This is one of the most common material script breakages when upgrading from 3.x to 4.0+.

```python
# WRONG — Blender 3.x socket names, BROKEN in 4.0+
principled.inputs["Subsurface"].default_value = 0.5            # KeyError
principled.inputs["Specular"].default_value = 0.5              # KeyError
principled.inputs["Transmission"].default_value = 1.0          # KeyError
principled.inputs["Coat"].default_value = 0.3                  # KeyError
principled.inputs["Sheen"].default_value = 0.1                 # KeyError
principled.inputs["Emission"].default_value = (1, 1, 1, 1)    # KeyError
```

```python
# CORRECT — Blender 4.0+ socket names
principled.inputs["Subsurface Weight"].default_value = 0.5
principled.inputs["Specular IOR Level"].default_value = 0.5
principled.inputs["Transmission Weight"].default_value = 1.0
principled.inputs["Coat Weight"].default_value = 0.3
principled.inputs["Sheen Weight"].default_value = 0.1
principled.inputs["Emission Color"].default_value = (1, 1, 1, 1)
```

```python
# CORRECT — version-safe pattern (supports both 3.x and 4.0+)
def set_principled_value(node, socket_3x, socket_4x, value):
    if bpy.app.version >= (4, 0, 0):
        socket_name = socket_4x
    else:
        socket_name = socket_3x
    if socket_name in node.inputs:
        node.inputs[socket_name].default_value = value
```

ALWAYS use 4.0+ socket names for Blender 4.0+ code. For cross-version scripts, ALWAYS use the version-safe helper pattern.

---

## AP-MAT-002: Using blend_method in Blender 4.2+

**WHY this is wrong**: The `Material.blend_method` property was removed in Blender 4.2. Accessing it raises `AttributeError`. The replacement is `Material.surface_render_method` with different enum values.

```python
# WRONG — Blender 3.x / 4.0-4.1 only, BROKEN in 4.2+
mat.blend_method = 'BLEND'     # AttributeError in 4.2+
mat.blend_method = 'CLIP'      # AttributeError in 4.2+
mat.blend_method = 'HASHED'    # AttributeError in 4.2+
mat.shadow_method = 'CLIP'     # May also be removed/restructured
```

```python
# CORRECT — Blender 4.2+
mat.surface_render_method = 'BLENDED'   # Replaces 'BLEND'
mat.surface_render_method = 'DITHERED'  # Replaces 'OPAQUE', 'CLIP', 'HASHED'

# CORRECT — version-safe pattern
if bpy.app.version >= (4, 2, 0):
    mat.surface_render_method = 'BLENDED'
else:
    mat.blend_method = 'BLEND'
```

ALWAYS check `bpy.app.version` before setting transparency method. NEVER use `blend_method` in code targeting Blender 4.2+.

---

## AP-MAT-003: Accessing node_tree Without use_nodes = True

**WHY this is wrong**: When `Material.use_nodes` is `False` (the default for non-node materials), `Material.node_tree` is `None`. Attempting to access `.nodes` on `None` raises `AttributeError`. Setting `use_nodes = True` creates the default node tree with a Principled BSDF and Material Output.

```python
# WRONG — node_tree is None
mat = bpy.data.materials.new("MyMat")
tree = mat.node_tree          # None
tree.nodes.clear()            # AttributeError: 'NoneType' has no attribute 'nodes'
```

```python
# CORRECT — enable nodes first
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True           # Creates default node tree
tree = mat.node_tree           # ShaderNodeTree (not None)
tree.nodes.clear()             # Works
```

ALWAYS set `mat.use_nodes = True` before accessing `mat.node_tree`.

---

## AP-MAT-004: Incorrect Color Space for Non-Color Textures

**WHY this is wrong**: Normal maps, roughness maps, metallic maps, and other data textures store numeric data, not colors. Loading them with `'sRGB'` color space applies gamma correction, which distorts the data values and produces incorrect shading results. Normal maps will look "washed out" and bump directions will be wrong.

```python
# WRONG — sRGB for a normal map (applies unwanted gamma correction)
normal_img = bpy.data.images.load("/path/to/normal.png", check_existing=True)
# colorspace_settings.name defaults to 'sRGB'
tex_node.image = normal_img
# Result: incorrect normals, washed-out bump effect
```

```python
# CORRECT — Non-Color for data textures
normal_img = bpy.data.images.load("/path/to/normal.png", check_existing=True)
normal_img.colorspace_settings.name = 'Non-Color'  # REQUIRED for data textures
tex_node.image = normal_img
# Result: correct normal interpretation
```

**Color space rules:**
| Texture Type | Color Space |
|---|---|
| Diffuse / Base Color / Albedo | `'sRGB'` |
| Normal map | `'Non-Color'` |
| Roughness map | `'Non-Color'` |
| Metallic map | `'Non-Color'` |
| Ambient Occlusion | `'Non-Color'` |
| Displacement / Height | `'Non-Color'` |
| Emission color texture | `'sRGB'` |

ALWAYS set `colorspace_settings.name = 'Non-Color'` for normal, roughness, metallic, AO, and displacement maps. NEVER leave data textures in the default `'sRGB'` space.

---

## AP-MAT-005: Accessing UV Data Per-Vertex Instead of Per-Loop

**WHY this is wrong**: UV coordinates in Blender are stored per-loop (per face corner), NOT per-vertex. A vertex shared by N faces has N separate UV coordinates (one for each face corner using that vertex). Attempting to access UV by vertex index produces wrong results or index errors.

```python
# WRONG — accessing UV by vertex index
mesh = obj.data
uv_layer = mesh.uv_layers.active
for vert_idx, vert in enumerate(mesh.vertices):
    uv = uv_layer.data[vert_idx].uv  # WRONG: uses vertex index, not loop index
    # Result: wrong UV mapping, may exceed array bounds
```

```python
# CORRECT — access UV by loop index (face corner)
mesh = obj.data
uv_layer = mesh.uv_layers.active
for poly in mesh.polygons:
    for loop_idx in poly.loop_indices:
        uv = uv_layer.data[loop_idx].uv
        vert_idx = mesh.loops[loop_idx].vertex_index
        # Now you have both the UV and the vertex it belongs to
```

ALWAYS access UV data via `uv_layer.data[loop_index]`. NEVER use vertex index to access UV data.

---

## AP-MAT-006: Loading Images Without check_existing=True

**WHY this is wrong**: Each call to `bpy.data.images.load()` without `check_existing=True` creates a new image data block, even if the same file is already loaded. This causes duplicate data blocks (`texture.png`, `texture.png.001`, `texture.png.002`, etc.), wastes memory, and makes material management confusing.

```python
# WRONG — creates duplicates on repeated calls
img = bpy.data.images.load("/path/to/texture.png")
# If called again, creates "texture.png.001"
img2 = bpy.data.images.load("/path/to/texture.png")
# img and img2 are DIFFERENT data blocks for the SAME file
```

```python
# CORRECT — reuses existing image data block
img = bpy.data.images.load("/path/to/texture.png", check_existing=True)
# If called again, returns the same data block
img2 = bpy.data.images.load("/path/to/texture.png", check_existing=True)
# img and img2 are the SAME data block
```

ALWAYS pass `check_existing=True` to `bpy.data.images.load()`. NEVER load images without this flag unless intentionally creating duplicates.

---

## AP-MAT-007: Using Subsurface Color Socket in 4.0+

**WHY this is wrong**: The `"Subsurface Color"` input was completely removed from the Principled BSDF in Blender 4.0. It does not exist as a socket. The base color now drives both the surface color and the subsurface scattering color. Attempting to access it raises `KeyError`.

```python
# WRONG — Blender 3.x only, REMOVED in 4.0+
principled.inputs["Subsurface Color"].default_value = (0.8, 0.2, 0.1, 1.0)
# KeyError: 'Subsurface Color'
```

```python
# CORRECT — Blender 4.0+
# Subsurface color is now derived from Base Color
principled.inputs["Base Color"].default_value = (0.8, 0.2, 0.1, 1.0)
principled.inputs["Subsurface Weight"].default_value = 0.5
# The base color automatically influences the subsurface appearance
```

NEVER reference `"Subsurface Color"` in Blender 4.0+ code. ALWAYS use `"Base Color"` to control both surface and subsurface color.

---

## AP-MAT-008: Assuming Default Principled BSDF Exists After nodes.clear()

**WHY this is wrong**: When you call `tree.nodes.clear()`, ALL nodes are removed including the default Principled BSDF and Material Output. After clearing, `tree.nodes.get("Principled BSDF")` returns `None`. You must create new nodes manually.

```python
# WRONG — clearing nodes then trying to access defaults
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

principled = tree.nodes.get("Principled BSDF")
principled.inputs["Base Color"].default_value = (1, 0, 0, 1)
# AttributeError: 'NoneType' has no attribute 'inputs'
```

```python
# CORRECT option 1 — don't clear, modify default nodes
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
principled = tree.nodes.get("Principled BSDF")  # Exists in default tree
principled.inputs["Base Color"].default_value = (1, 0, 0, 1)

# CORRECT option 2 — clear and recreate all nodes
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
output = tree.nodes.new('ShaderNodeOutputMaterial')
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
principled.inputs["Base Color"].default_value = (1, 0, 0, 1)
```

ALWAYS either keep default nodes intact OR recreate both Principled BSDF and Material Output after clearing.

---

## AP-MAT-009: Specular Tint Type Mismatch (3.x vs 4.0+)

**WHY this is wrong**: In Blender 3.x, the `"Specular Tint"` input is a Float (0.0 to 1.0). In Blender 4.0+, it was changed to a Color (RGBA). Passing a float to the 4.0+ color socket or a color tuple to the 3.x float socket produces incorrect results or type errors.

```python
# WRONG — using float value in Blender 4.0+ (expects Color)
principled.inputs["Specular Tint"].default_value = 0.5  # TypeError in 4.0+

# WRONG — using color value in Blender 3.x (expects Float)
principled.inputs["Specular Tint"].default_value = (1, 0.9, 0.8, 1)  # TypeError in 3.x
```

```python
# CORRECT — version-safe approach
if bpy.app.version >= (4, 0, 0):
    # 4.0+: Specular Tint is a Color
    principled.inputs["Specular Tint"].default_value = (1.0, 0.9, 0.8, 1.0)
else:
    # 3.x: Specular Tint is a Float
    principled.inputs["Specular Tint"].default_value = 0.5
```

ALWAYS check Blender version before setting Specular Tint. The socket type changed from Float to Color in 4.0.

---

## AP-MAT-010: Forgetting to Link Material Output

**WHY this is wrong**: A material node tree without a `ShaderNodeOutputMaterial` connected to the BSDF produces no visible shading. The object appears as solid pink (missing material) or uses default shading. This is a common oversight when building node trees from scratch.

```python
# WRONG — no Material Output node, or output not linked
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.inputs["Base Color"].default_value = (1, 0, 0, 1)
# Missing: Material Output node and link
# Result: pink/default material in viewport
```

```python
# CORRECT — always create and link Material Output
mat = bpy.data.materials.new("MyMat")
mat.use_nodes = True
tree = mat.node_tree
tree.nodes.clear()

principled = tree.nodes.new('ShaderNodeBsdfPrincipled')
principled.inputs["Base Color"].default_value = (1, 0, 0, 1)

output = tree.nodes.new('ShaderNodeOutputMaterial')
tree.links.new(principled.outputs["BSDF"], output.inputs["Surface"])
```

ALWAYS create a `ShaderNodeOutputMaterial` and link it to the shader's BSDF output when building node trees from scratch.

---

## AP-MAT-011: Using ShaderNodeMixRGB in Blender 4.0+

**WHY this is wrong**: `ShaderNodeMixRGB` was replaced by `ShaderNodeMix` in Blender 4.0. While `ShaderNodeMixRGB` may still work for backward compatibility, it is deprecated and may be removed. The new `ShaderNodeMix` node supports mixing colors, floats, and vectors.

```python
# WRONG — deprecated node type in 4.0+
mix = tree.nodes.new('ShaderNodeMixRGB')  # Deprecated in 4.0+
```

```python
# CORRECT — Blender 4.0+
mix = tree.nodes.new('ShaderNodeMix')
mix.data_type = 'RGBA'  # For color mixing

# CORRECT — version-safe
if bpy.app.version >= (4, 0, 0):
    mix = tree.nodes.new('ShaderNodeMix')
    mix.data_type = 'RGBA'
else:
    mix = tree.nodes.new('ShaderNodeMixRGB')
```

ALWAYS use `ShaderNodeMix` for Blender 4.0+ code. For cross-version scripts, check `bpy.app.version` before selecting the node type.

---

## AP-MAT-012: Assigning Material Without Object Data Check

**WHY this is wrong**: Not all objects have `data.materials`. Empties, cameras, lights, and other non-mesh objects do not support material slots. Attempting to assign materials to them raises `AttributeError`.

```python
# WRONG — no type check
obj = bpy.context.active_object
obj.data.materials.append(mat)
# AttributeError if obj is an Empty, Camera, Light, etc.
```

```python
# CORRECT — check object type first
obj = bpy.context.active_object
if obj is not None and obj.type in {'MESH', 'CURVE', 'SURFACE', 'FONT', 'META'}:
    if obj.data.materials:
        obj.data.materials[0] = mat
    else:
        obj.data.materials.append(mat)
```

ALWAYS verify `obj.type` supports materials before assigning. Object types that support materials: `MESH`, `CURVE`, `SURFACE`, `FONT`, `META`, `GPENCIL`, `VOLUME`.
