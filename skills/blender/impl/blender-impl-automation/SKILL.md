---
name: blender-impl-automation
description: >
  Use when automating Blender workflows -- batch rendering, headless processing via
  blender --background, file format conversion, or pipeline integration. Prevents the
  common mistake of using viewport operators in background mode (no UI context available).
  Covers command-line rendering, file I/O automation (OBJ/FBX/STL/USD/glTF), scene
  assembly, and CI/CD pipeline integration.
  Keywords: batch rendering, headless, blender --background, automation, file conversion,
  OBJ, FBX, STL, USD, glTF, pipeline, command-line.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender 3.x/4.x/5.x with Python."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# blender-impl-automation

## Quick Reference

### Critical Warnings

**NEVER** use `bpy.ops.export_scene.obj()` or `bpy.ops.export_mesh.stl()` in Blender 4.0+ — these Python exporters are REMOVED. Use `bpy.ops.wm.obj_export()` and `bpy.ops.wm.stl_export()`.

**NEVER** call viewport operators (`bpy.ops.view3d.*`, `bpy.ops.screen.*`) in `--background` mode — no UI context exists. Check `bpy.app.background` before any UI-dependent operator.

**NEVER** use `bpy.ops.wm.invoke_*` in background mode — modal/interactive operators require a window event loop.

**NEVER** hardcode absolute file paths — use `os.path`, `pathlib.Path`, or command-line arguments.

**NEVER** omit `sys.exit()` in CI/CD scripts — Blender returns 0 by default even on script errors. Use `--python-exit-code 1` (3.0+) or explicit `sys.exit(1)`.

**ALWAYS** set `apply_scale_options` explicitly in FBX export — default scale causes unit mismatch.

**ALWAYS** verify axis conventions (`axis_forward`, `axis_up`) match the target application for FBX/OBJ.

**ALWAYS** set `export_custom_properties=True` for Alembic when preserving AEC metadata.

**ALWAYS** parse custom arguments AFTER the `--` separator in command-line scripts.

### Decision Tree: Automation Approach

```
What kind of automation?
├─ Render frames/animation?
│  ├─ Single frame → blender -b file.blend -o /path/frame_ -f FRAME
│  ├─ Animation   → blender -b file.blend -o /path/frame_ -a
│  └─ Custom settings → blender -b file.blend -P render_script.py
├─ Convert file formats?
│  ├─ Single file → blender -b -P convert.py -- --input f --output f
│  └─ Batch dir   → blender -b -P batch_convert.py -- --dir /path
├─ Assemble scene from sources?
│  ├─ Full copies → bpy.ops.wm.append() or bpy.data.libraries.load()
│  ├─ Linked refs → bpy.ops.wm.link()
│  └─ Overridable → Library Overrides (4.0+)
├─ Process data without rendering?
│  └─ blender -b [-file.blend] -P script.py
└─ CI/CD pipeline?
   └─ blender -b --python-exit-code 1 -P script.py
```

### Decision Tree: Export Format

```
Target?
├─ Game engines → FBX (animation) or glTF/GLB (static)
├─ Web/WebGL   → glTF/GLB (prefer GLB binary)
├─ 3D printing → STL (bpy.ops.wm.stl_export, 4.0+)
├─ CAD interchange → OBJ (bpy.ops.wm.obj_export, 4.0+)
├─ VFX pipeline → USD or Alembic (animated geometry)
└─ Archival     → USD (.usdc/.usda)
```

### I/O Operator Version Matrix

| Format | Blender 3.x | Blender 4.0+ |
|--------|-------------|--------------|
| OBJ | `bpy.ops.export_scene.obj()` | REMOVED → `bpy.ops.wm.obj_export()` |
| STL | `bpy.ops.export_mesh.stl()` | REMOVED → `bpy.ops.wm.stl_export()` |
| PLY | `bpy.ops.export_mesh.ply()` | REMOVED → `bpy.ops.wm.ply_export()` |
| FBX | `bpy.ops.export_scene.fbx()` | Same (unchanged) |
| glTF | `bpy.ops.export_scene.gltf()` | Same (unchanged) |
| USD | `bpy.ops.wm.usd_export()` | Same (unchanged) |
| Alembic | `bpy.ops.wm.alembic_export()` | Same (unchanged) |

---

## Section 1: Command-Line Interface

```bash
# Background mode
blender -b -P script.py
blender -b scene.blend -P script.py

# Custom arguments (after -- separator)
blender -b scene.blend -P script.py -- --my-arg value

# Render single frame / animation / frame range
blender -b scene.blend -o /tmp/render_ -f 1
blender -b scene.blend -o /tmp/render_ -a
blender -b scene.blend -o /tmp/render_ -s 1 -e 250 -a

# Set engine / threads
blender -b scene.blend -E CYCLES -o /tmp/render_ -f 1
blender -b scene.blend -t 8 -o /tmp/render_ -a

# Python expression
blender -b --python-expr "import bpy; print(bpy.app.version)"

# CI/CD exit code propagation (Blender 3.0+)
blender -b --python-exit-code 1 -P script.py
```

### Argument Parsing Pattern

```python
# Blender 3.x/4.x/5.x: Parse arguments after '--' separator
import sys, argparse

def get_args():
    argv = sys.argv
    if "--" not in argv:
        return argparse.Namespace()
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True)
    parser.add_argument("--output", required=True)
    parser.add_argument("--scale", type=float, default=1.0)
    return parser.parse_args(argv[argv.index("--") + 1:])
```

---

## Section 2: Background Mode

```python
# Blender 3.x/4.x/5.x: Detect and guard
import bpy

if bpy.app.background:
    # SAFE: bpy.data, bpy.ops.wm.*, file I/O, render
    # UNSAFE: viewport ops, UI drawing, modal operators, GPU
    pass
```

| Category | Background Mode |
|----------|----------------|
| `bpy.data.*` access | YES |
| File I/O (import/export) | YES |
| `bpy.ops.render.render()` | YES |
| Non-UI operators | YES |
| Viewport ops (`view3d.*`) | NO |
| Modal operators | NO |
| `context.area` | None — guard access |
| GPU drawing | NO |

### bpy as a Module (Alternative)

```python
# Outside Blender: pip install bpy (match Blender's Python version)
import bpy
bpy.ops.wm.read_homefile(use_empty=True)
# Same capabilities and limitations as --background
```

---

## Section 3: File Format I/O

### Version-Safe I/O Pattern

```python
# Blender 3.x/4.x/5.x: Version-safe OBJ/STL
import bpy

def export_obj(filepath, **kwargs):
    if bpy.app.version >= (4, 0, 0):
        bpy.ops.wm.obj_export(filepath=filepath, **kwargs)
    else:
        bpy.ops.export_scene.obj(filepath=filepath, **kwargs)

def export_stl(filepath, **kwargs):
    if bpy.app.version >= (4, 0, 0):
        bpy.ops.wm.stl_export(filepath=filepath, **kwargs)
    else:
        bpy.ops.export_mesh.stl(filepath=filepath, **kwargs)
```

### Key Parameters per Format

**FBX** (`bpy.ops.export_scene.fbx()` — all versions):
- `apply_scale_options='FBX_SCALE_ALL'` — ALWAYS set
- `axis_forward='-Z'`, `axis_up='Y'` — verify against target
- `use_mesh_modifiers=True`, `use_custom_props=True`

**glTF/GLB** (`bpy.ops.export_scene.gltf()` — all versions):
- `export_format='GLB'` — prefer binary single-file
- `export_apply=True`, `export_materials='EXPORT'`

**USD** (`bpy.ops.wm.usd_export()` — 2.9+):
- `selected_objects_only=True`, `export_materials=True`
- `use_instancing=True`

**Alembic** (`bpy.ops.wm.alembic_export()` — 2.78+):
- `export_custom_properties=True` — REQUIRED for AEC metadata
- `selected=True`, `use_instancing=True`

**OBJ 4.0+** (`bpy.ops.wm.obj_export()`):
- `export_selected_objects=True`, `apply_modifiers=True`
- `forward_axis='NEGATIVE_Z'`, `up_axis='Y'`

**STL 4.0+** (`bpy.ops.wm.stl_export()`):
- `export_selected_objects=True`, `ascii_format=False`
- `apply_modifiers=True`

---

## Section 4: Batch Rendering

```python
# Blender 3.x/4.x/5.x: Scripted render
import bpy

scene = bpy.context.scene
scene.render.engine = 'CYCLES'
scene.render.resolution_x = 1920
scene.render.resolution_y = 1080
scene.render.image_settings.file_format = 'PNG'

# Single frame
scene.render.filepath = "/tmp/render_output"
bpy.ops.render.render(write_still=True)

# Animation
scene.frame_start = 1
scene.frame_end = 250
scene.render.filepath = "/tmp/anim/frame_"
bpy.ops.render.render(animation=True)
```

### Multi-Camera Render

```python
# Blender 3.x/4.x/5.x
import bpy, os

output_dir = "/tmp/renders"
os.makedirs(output_dir, exist_ok=True)
scene = bpy.context.scene

for cam in [o for o in bpy.data.objects if o.type == 'CAMERA']:
    scene.camera = cam
    scene.render.filepath = os.path.join(output_dir, f"{cam.name}_")
    bpy.ops.render.render(write_still=True)
```

### EEVEE Engine Version Detection

```python
# Blender 3.x → 'BLENDER_EEVEE', Blender 4.0+ → 'BLENDER_EEVEE_NEXT'
if bpy.app.version >= (4, 0, 0):
    bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    bpy.context.scene.render.engine = 'BLENDER_EEVEE'
```

---

## Section 5: Scene Assembly

### Append vs Link vs libraries.load()

| | `bpy.data.libraries.load()` | `bpy.ops.wm.append()` | `bpy.ops.wm.link()` |
|--|------|--------|------|
| Returns imported data | YES | NO | NO |
| Operator context needed | NO | YES | YES |
| Background mode safe | YES | YES | YES |
| Recommended for scripts | YES | Simple cases | Linked refs |

### bpy.data.libraries.load() (Preferred for Scripts)

```python
# Blender 3.x/4.x/5.x: Batch append from .blend files
import bpy, os, glob

for blend_path in glob.glob("/path/to/assets/*.blend"):
    with bpy.data.libraries.load(blend_path, link=False) as (data_from, data_to):
        data_to.objects = data_from.objects  # Import all objects
    for obj in data_to.objects:
        if obj is not None:
            bpy.context.collection.objects.link(obj)
```

### Append/Link via Operators

```python
# Blender 3.x/4.x/5.x: Append single object
import bpy, os

source = "/path/to/library.blend"
bpy.ops.wm.append(
    filepath=os.path.join(source, "Object", "MyObject"),
    directory=os.path.join(source, "Object"),
    filename="MyObject",
    link=False,
)
```

---

## Section 6: Pipeline Integration

### Exit Code Pattern

```python
# ALWAYS use in CI/CD scripts
import sys

try:
    do_work()
except Exception as e:
    print(f"FAILED: {e}", file=sys.stderr)
    sys.exit(1)
```

```bash
blender -b --python-exit-code 1 -P script.py
echo $?  # Non-zero on failure
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `BLENDER_USER_SCRIPTS` | Custom script search path |
| `BLENDER_SYSTEM_SCRIPTS` | System-wide script path |
| `BLENDER_USER_CONFIG` | User config directory |
| `OCIO` | OpenColorIO config |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for I/O, render, and assembly operators
- [references/examples.md](references/examples.md) — Full working automation scripts
- [references/anti-patterns.md](references/anti-patterns.md) — Common automation mistakes with fixes

### Official Sources

- https://docs.blender.org/api/current/bpy.ops.wm.html
- https://docs.blender.org/api/current/bpy.ops.export_scene.html
- https://docs.blender.org/api/current/bpy.ops.import_scene.html
- https://docs.blender.org/api/current/bpy.ops.render.html
- https://docs.blender.org/api/current/bpy.app.html
- https://docs.blender.org/manual/en/latest/advanced/command_line/arguments.html

### Related Skills

- [blender-impl-operators](../blender-impl-operators/SKILL.md) — Modal operators, batch processing, progress reporting
- [blender-core-runtime](../../core/blender-core-runtime/SKILL.md) — Background mode, threading, handlers
- [blender-core-versions](../../core/blender-core-versions/SKILL.md) — Breaking changes across versions
