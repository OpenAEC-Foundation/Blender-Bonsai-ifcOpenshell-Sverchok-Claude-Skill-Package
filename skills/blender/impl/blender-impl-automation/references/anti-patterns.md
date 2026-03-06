# blender-impl-automation — Anti-Patterns

Common mistakes in Blender automation, batch processing, and headless workflows. Each entry includes the incorrect pattern, the correct alternative, and an explanation of WHY it fails.

---

## Anti-Pattern 1: Using Removed I/O Operators in Blender 4.0+

**WRONG:**
```python
# Blender 4.0+ — these operators are REMOVED
bpy.ops.export_scene.obj(filepath="/tmp/model.obj")
bpy.ops.export_mesh.stl(filepath="/tmp/model.stl")
bpy.ops.export_mesh.ply(filepath="/tmp/model.ply")
```

**CORRECT:**
```python
# Blender 4.0+ — use wm.* operators
bpy.ops.wm.obj_export(filepath="/tmp/model.obj")
bpy.ops.wm.stl_export(filepath="/tmp/model.stl")
bpy.ops.wm.ply_export(filepath="/tmp/model.ply")
```

**Version-safe wrapper:**
```python
def export_obj(filepath, **kwargs):
    if bpy.app.version >= (4, 0, 0):
        bpy.ops.wm.obj_export(filepath=filepath, **kwargs)
    else:
        bpy.ops.export_scene.obj(filepath=filepath, **kwargs)
```

**WHY:** Blender 4.0 replaced the Python-based OBJ, STL, and PLY exporters with C++ implementations under `bpy.ops.wm.*`. The old `export_scene.obj`, `export_mesh.stl`, and `export_mesh.ply` operators no longer exist. Calling them raises `AttributeError`, which silently breaks batch processing pipelines.

---

## Anti-Pattern 2: Calling Viewport Operators in Background Mode

**WRONG:**
```python
# Runs in blender --background — crashes or does nothing
bpy.ops.view3d.snap_cursor_to_selected()
bpy.ops.screen.animation_play()
bpy.ops.wm.invoke_props_dialog(operator)
```

**CORRECT:**
```python
import bpy

if bpy.app.background:
    # Use data-level manipulation instead of viewport operators
    bpy.context.scene.cursor.location = obj.location
else:
    bpy.ops.view3d.snap_cursor_to_selected()
```

**WHY:** Background mode (`blender --background`) has no UI context — no windows, no areas, no regions. Viewport operators (`view3d.*`, `screen.*`) and modal/interactive operators (`wm.invoke_*`) require an active window event loop. Calling them raises `RuntimeError: Operator bpy.ops.view3d.* has no default context` or silently fails.

---

## Anti-Pattern 3: Omitting Exit Codes in CI/CD Scripts

**WRONG:**
```python
# script.py — no explicit exit code
import bpy

try:
    do_work()
except Exception as e:
    print(f"Error: {e}")
    # Script ends — Blender returns 0 regardless
```

```bash
blender -b -P script.py
echo $?  # Always 0, even on failure
```

**CORRECT:**
```python
import sys

try:
    do_work()
except Exception as e:
    print(f"FAILED: {e}", file=sys.stderr)
    sys.exit(1)
```

```bash
blender -b --python-exit-code 1 -P script.py
echo $?  # Non-zero on Python exception
```

**WHY:** Blender returns exit code 0 by default, even when a Python script raises an unhandled exception. Without `--python-exit-code 1` (Blender 3.0+) or an explicit `sys.exit(1)`, CI/CD pipelines report success on failures. Both approaches should be combined: `--python-exit-code` catches unhandled exceptions, and `sys.exit(1)` handles caught errors.

---

## Anti-Pattern 4: Hardcoding Absolute File Paths

**WRONG:**
```python
bpy.ops.wm.obj_export(filepath="C:/Users/dev/output/model.obj")
bpy.ops.render.render(write_still=True)
bpy.context.scene.render.filepath = "/home/dev/renders/frame_"
```

**CORRECT:**
```python
import sys
import argparse
import os

def get_args():
    argv = sys.argv
    if "--" not in argv:
        return argparse.Namespace(output="/tmp/output")
    parser = argparse.ArgumentParser()
    parser.add_argument("--output", required=True)
    return parser.parse_args(argv[argv.index("--") + 1:])

args = get_args()
os.makedirs(args.output, exist_ok=True)
filepath = os.path.join(args.output, "model.obj")
bpy.ops.wm.obj_export(filepath=filepath)
```

**WHY:** Hardcoded paths break when scripts run on different machines, operating systems, or CI/CD runners. Use command-line arguments (parsed after `--` separator), environment variables, or relative paths. This makes automation scripts portable across development, staging, and production environments.

---

## Anti-Pattern 5: Not Setting FBX Scale Options Explicitly

**WRONG:**
```python
# Default scale causes unit mismatch in game engines
bpy.ops.export_scene.fbx(filepath="/tmp/model.fbx")
```

**CORRECT:**
```python
bpy.ops.export_scene.fbx(
    filepath="/tmp/model.fbx",
    apply_scale_options='FBX_SCALE_ALL',
    axis_forward='-Z',
    axis_up='Y',
    use_mesh_modifiers=True,
)
```

**WHY:** FBX default scale behavior results in a 100x scale factor mismatch between Blender (1 unit = 1 meter) and FBX convention (1 unit = 1 centimeter). Without `apply_scale_options='FBX_SCALE_ALL'`, imported models in Unity, Unreal Engine, or other DCC tools appear 100x too large or too small. Axis conventions also differ between applications and must be set explicitly.

---

## Anti-Pattern 6: Parsing Arguments Before the -- Separator

**WRONG:**
```python
import sys
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--input", required=True)
args = parser.parse_args()  # Parses ALL sys.argv including Blender's own arguments
```

**CORRECT:**
```python
import sys
import argparse

def get_args():
    argv = sys.argv
    if "--" not in argv:
        return argparse.Namespace()
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True)
    return parser.parse_args(argv[argv.index("--") + 1:])

args = get_args()
```

**WHY:** `sys.argv` contains Blender's own command-line arguments (`-b`, `-P`, etc.) before the `--` separator. Parsing the full `sys.argv` with argparse raises `error: unrecognized arguments` for Blender's flags. The `--` separator convention reserves everything after it for the script's own arguments.

---

## Anti-Pattern 7: Using Wrong EEVEE Engine Name Across Versions

**WRONG:**
```python
# Breaks on Blender 4.0+ or 3.x depending on which name is used
bpy.context.scene.render.engine = 'BLENDER_EEVEE'       # 3.x only
bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'  # 4.0+ only
```

**CORRECT:**
```python
if bpy.app.version >= (4, 0, 0):
    bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'
else:
    bpy.context.scene.render.engine = 'BLENDER_EEVEE'
```

**WHY:** Blender 4.0 renamed the EEVEE engine from `BLENDER_EEVEE` to `BLENDER_EEVEE_NEXT` as part of the EEVEE rewrite. Using the wrong name raises `ValueError: bpy_struct: item.attr = val: enum "BLENDER_EEVEE" not found`. Automation scripts that target multiple Blender versions must check `bpy.app.version` and select the correct engine string.

---

## Anti-Pattern 8: Not Creating Output Directories Before Rendering

**WRONG:**
```python
bpy.context.scene.render.filepath = "/tmp/renders/project/frame_"
bpy.ops.render.render(animation=True)  # Fails if /tmp/renders/project/ does not exist
```

**CORRECT:**
```python
import os

output_dir = "/tmp/renders/project"
os.makedirs(output_dir, exist_ok=True)
bpy.context.scene.render.filepath = os.path.join(output_dir, "frame_")
bpy.ops.render.render(animation=True)
```

**WHY:** Blender does not create intermediate directories for render output paths. If the parent directory does not exist, the render completes but the output file is silently not written (or raises `OSError` depending on the output format handler). Always use `os.makedirs(dir, exist_ok=True)` before setting `render.filepath`.

---

## Anti-Pattern 9: Using bpy.data.libraries.load() Without Linking Objects to Scene

**WRONG:**
```python
with bpy.data.libraries.load(blend_path, link=False) as (data_from, data_to):
    data_to.objects = data_from.objects
# Objects are in bpy.data but NOT visible in any scene
```

**CORRECT:**
```python
with bpy.data.libraries.load(blend_path, link=False) as (data_from, data_to):
    data_to.objects = data_from.objects

for obj in data_to.objects:
    if obj is not None:
        bpy.context.collection.objects.link(obj)
```

**WHY:** `bpy.data.libraries.load()` imports data-blocks into `bpy.data` but does not add them to any scene or collection. Without explicitly calling `collection.objects.link(obj)`, the imported objects exist only as orphan data-blocks — invisible in the viewport, excluded from renders, and garbage-collected on file save. The `if obj is not None` guard handles data-blocks that fail to load.

---

## Anti-Pattern 10: Ignoring context.area Being None in Background Mode

**WRONG:**
```python
# Runs in blender --background
area = bpy.context.area
area.type = 'VIEW_3D'  # AttributeError: 'NoneType' has no attribute 'type'
```

**CORRECT:**
```python
import bpy

if not bpy.app.background:
    area = bpy.context.area
    if area is not None:
        area.type = 'VIEW_3D'
else:
    # Background mode — manipulate data directly without area context
    pass
```

**WHY:** In background mode, `bpy.context.area` is `None` because no UI areas exist. Any script that accesses area properties without guarding for `None` raises `AttributeError`. Automation scripts should use `bpy.app.background` to detect headless mode and fall back to data-level operations that do not require a UI context.
