# blender-impl-automation — Working Examples

All examples verified against Blender 4.x API. Version-specific notes included where applicable.

---

## Example 1: Batch File Format Converter

Converts all files in a directory from one format to another. Handles the 4.0 OBJ/STL operator migration.

```python
#!/usr/bin/env python3
"""Batch file format converter for Blender.
Usage: blender -b --python-exit-code 1 -P batch_convert.py -- --input-dir /path --output-dir /path --format GLB
"""
# Blender 3.x/4.x/5.x
import bpy
import sys
import os
import glob
import argparse

SUPPORTED_IMPORT = {
    ".obj": "OBJ",
    ".fbx": "FBX",
    ".stl": "STL",
    ".glb": "GLTF",
    ".gltf": "GLTF",
    ".usd": "USD",
    ".usdc": "USD",
    ".usda": "USD",
    ".abc": "ALEMBIC",
}

def parse_args():
    argv = sys.argv
    if "--" not in argv:
        print("ERROR: Use -- separator for script arguments", file=sys.stderr)
        sys.exit(1)
    parser = argparse.ArgumentParser(description="Batch format converter")
    parser.add_argument("--input-dir", required=True, help="Input directory")
    parser.add_argument("--output-dir", required=True, help="Output directory")
    parser.add_argument("--format", required=True,
                        choices=["GLB", "FBX", "OBJ", "USD", "STL"],
                        help="Target export format")
    parser.add_argument("--scale", type=float, default=1.0)
    return parser.parse_args(argv[argv.index("--") + 1:])

def clear_scene():
    """Remove all objects from the scene."""
    bpy.ops.object.select_all(action='SELECT')
    bpy.ops.object.delete(use_global=False)
    # Purge orphan data
    for block_type in ('meshes', 'materials', 'textures', 'images'):
        collection = getattr(bpy.data, block_type)
        for block in list(collection):
            if block.users == 0:
                collection.remove(block)

def import_file(filepath):
    ext = os.path.splitext(filepath)[1].lower()
    if ext == ".obj":
        if bpy.app.version >= (4, 0, 0):
            bpy.ops.wm.obj_import(filepath=filepath)
        else:
            bpy.ops.import_scene.obj(filepath=filepath)
    elif ext == ".fbx":
        bpy.ops.import_scene.fbx(filepath=filepath)
    elif ext in (".glb", ".gltf"):
        bpy.ops.import_scene.gltf(filepath=filepath)
    elif ext == ".stl":
        if bpy.app.version >= (4, 0, 0):
            bpy.ops.wm.stl_import(filepath=filepath)
        else:
            bpy.ops.import_mesh.stl(filepath=filepath)
    elif ext in (".usd", ".usdc", ".usda"):
        bpy.ops.wm.usd_import(filepath=filepath)
    elif ext == ".abc":
        bpy.ops.wm.alembic_import(filepath=filepath)
    else:
        raise ValueError(f"Unsupported import format: {ext}")

def export_file(filepath, fmt):
    if fmt == "GLB":
        bpy.ops.export_scene.gltf(
            filepath=filepath,
            export_format='GLB',
            export_apply=True,
        )
    elif fmt == "FBX":
        bpy.ops.export_scene.fbx(
            filepath=filepath,
            apply_scale_options='FBX_SCALE_ALL',
            use_mesh_modifiers=True,
        )
    elif fmt == "OBJ":
        if bpy.app.version >= (4, 0, 0):
            bpy.ops.wm.obj_export(filepath=filepath, apply_modifiers=True)
        else:
            bpy.ops.export_scene.obj(filepath=filepath, use_mesh_modifiers=True)
    elif fmt == "USD":
        bpy.ops.wm.usd_export(filepath=filepath, export_materials=True)
    elif fmt == "STL":
        if bpy.app.version >= (4, 0, 0):
            bpy.ops.wm.stl_export(filepath=filepath, apply_modifiers=True)
        else:
            bpy.ops.export_mesh.stl(filepath=filepath, use_mesh_modifiers=True)

FORMAT_EXT = {"GLB": ".glb", "FBX": ".fbx", "OBJ": ".obj", "USD": ".usdc", "STL": ".stl"}

def main():
    args = parse_args()
    os.makedirs(args.output_dir, exist_ok=True)

    extensions = list(SUPPORTED_IMPORT.keys())
    input_files = []
    for ext in extensions:
        input_files.extend(glob.glob(os.path.join(args.input_dir, f"*{ext}")))

    if not input_files:
        print(f"No supported files found in {args.input_dir}", file=sys.stderr)
        sys.exit(1)

    success = 0
    failed = 0
    for filepath in sorted(input_files):
        basename = os.path.splitext(os.path.basename(filepath))[0]
        output_path = os.path.join(args.output_dir, basename + FORMAT_EXT[args.format])

        try:
            clear_scene()
            import_file(filepath)

            if args.scale != 1.0:
                for obj in bpy.context.scene.objects:
                    obj.scale *= args.scale

            export_file(output_path, args.format)
            print(f"OK: {os.path.basename(filepath)} -> {os.path.basename(output_path)}")
            success += 1
        except Exception as e:
            print(f"FAIL: {os.path.basename(filepath)}: {e}", file=sys.stderr)
            failed += 1

    print(f"\nComplete: {success} converted, {failed} failed")
    if failed > 0:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## Example 2: Batch Render Pipeline

Renders multiple scenes or camera angles from command line with configurable output.

```python
#!/usr/bin/env python3
"""Batch render pipeline.
Usage: blender -b scene.blend --python-exit-code 1 -P batch_render.py -- --output-dir /renders --cameras all --engine CYCLES
"""
# Blender 3.x/4.x/5.x
import bpy
import sys
import os
import argparse

def parse_args():
    argv = sys.argv
    if "--" not in argv:
        print("ERROR: Use -- separator", file=sys.stderr)
        sys.exit(1)
    parser = argparse.ArgumentParser()
    parser.add_argument("--output-dir", required=True)
    parser.add_argument("--cameras", default="all",
                        help="Comma-separated camera names or 'all'")
    parser.add_argument("--engine", default=None,
                        choices=["CYCLES", "EEVEE"])
    parser.add_argument("--resolution", default="1920x1080")
    parser.add_argument("--samples", type=int, default=None)
    parser.add_argument("--format", default="PNG",
                        choices=["PNG", "JPEG", "OPEN_EXR", "TIFF"])
    parser.add_argument("--animation", action="store_true")
    return parser.parse_args(argv[argv.index("--") + 1:])

def set_engine(engine_name):
    if engine_name == "CYCLES":
        bpy.context.scene.render.engine = 'CYCLES'
    elif engine_name == "EEVEE":
        if bpy.app.version >= (4, 0, 0):
            bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'
        else:
            bpy.context.scene.render.engine = 'BLENDER_EEVEE'

def main():
    args = parse_args()
    os.makedirs(args.output_dir, exist_ok=True)
    scene = bpy.context.scene

    # Configure render settings
    res_x, res_y = args.resolution.split("x")
    scene.render.resolution_x = int(res_x)
    scene.render.resolution_y = int(res_y)
    scene.render.resolution_percentage = 100
    scene.render.image_settings.file_format = args.format

    if args.engine:
        set_engine(args.engine)

    if args.samples and scene.render.engine == 'CYCLES':
        scene.cycles.samples = args.samples

    # Get cameras
    if args.cameras == "all":
        cameras = [obj for obj in bpy.data.objects if obj.type == 'CAMERA']
    else:
        cam_names = [n.strip() for n in args.cameras.split(",")]
        cameras = []
        for name in cam_names:
            cam = bpy.data.objects.get(name)
            if cam is None or cam.type != 'CAMERA':
                print(f"WARNING: Camera '{name}' not found", file=sys.stderr)
                continue
            cameras.append(cam)

    if not cameras:
        print("ERROR: No cameras found", file=sys.stderr)
        sys.exit(1)

    # Render each camera
    for cam in cameras:
        scene.camera = cam
        output_path = os.path.join(args.output_dir, f"{cam.name}_")
        scene.render.filepath = output_path

        print(f"Rendering: {cam.name} ({'animation' if args.animation else 'still'})")

        if args.animation:
            bpy.ops.render.render(animation=True)
        else:
            bpy.ops.render.render(write_still=True)

        print(f"  Output: {output_path}")

    print(f"\nRendered {len(cameras)} camera(s) to {args.output_dir}")

if __name__ == "__main__":
    main()
```

---

## Example 3: Scene Assembly from Asset Library

Assembles a scene by importing objects from multiple .blend library files.

```python
#!/usr/bin/env python3
"""Scene assembly from .blend asset libraries.
Usage: blender -b --python-exit-code 1 -P assemble_scene.py -- --config scene_config.json --output assembled.blend
"""
# Blender 3.x/4.x/5.x
import bpy
import sys
import os
import json
import argparse
from mathutils import Vector, Euler
import math

def parse_args():
    argv = sys.argv
    if "--" not in argv:
        sys.exit(1)
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", required=True, help="JSON config file")
    parser.add_argument("--output", required=True, help="Output .blend file")
    return parser.parse_args(argv[argv.index("--") + 1:])

def load_config(path):
    """Expected config format:
    {
        "assets": [
            {
                "library": "/path/to/library.blend",
                "object": "Chair",
                "location": [0, 0, 0],
                "rotation": [0, 0, 45],
                "scale": [1, 1, 1],
                "collection": "Furniture"
            }
        ]
    }
    """
    with open(path, 'r') as f:
        return json.load(f)

def ensure_collection(name):
    """Get or create a collection by name."""
    coll = bpy.data.collections.get(name)
    if coll is None:
        coll = bpy.data.collections.new(name)
        bpy.context.scene.collection.children.link(coll)
    return coll

def main():
    args = parse_args()

    if not os.path.isfile(args.config):
        print(f"ERROR: Config not found: {args.config}", file=sys.stderr)
        sys.exit(1)

    config = load_config(args.config)
    bpy.ops.wm.read_homefile(use_empty=True)

    imported = 0
    failed = 0

    for asset in config.get("assets", []):
        lib_path = asset["library"]
        obj_name = asset["object"]

        if not os.path.isfile(lib_path):
            print(f"SKIP: Library not found: {lib_path}", file=sys.stderr)
            failed += 1
            continue

        # Import using bpy.data.libraries.load (preferred for scripts)
        with bpy.data.libraries.load(lib_path, link=False) as (data_from, data_to):
            if obj_name in data_from.objects:
                data_to.objects = [obj_name]
            else:
                print(f"SKIP: '{obj_name}' not found in {lib_path}", file=sys.stderr)
                failed += 1
                continue

        # Link to target collection
        target_coll_name = asset.get("collection", "Scene Collection")
        target_coll = ensure_collection(target_coll_name)

        for obj in data_to.objects:
            if obj is None:
                continue
            target_coll.objects.link(obj)

            # Apply transforms
            if "location" in asset:
                obj.location = Vector(asset["location"])
            if "rotation" in asset:
                r = asset["rotation"]
                obj.rotation_euler = Euler(
                    (math.radians(r[0]), math.radians(r[1]), math.radians(r[2])), 'XYZ'
                )
            if "scale" in asset:
                obj.scale = Vector(asset["scale"])

            imported += 1
            print(f"  Placed: {obj.name} at {list(obj.location)}")

    # Save assembled scene
    os.makedirs(os.path.dirname(os.path.abspath(args.output)), exist_ok=True)
    bpy.ops.wm.save_as_mainfile(filepath=os.path.abspath(args.output))

    print(f"\nAssembled: {imported} objects ({failed} failed)")
    print(f"Saved to: {args.output}")

    if failed > 0:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## Example 4: Headless Data Processing

Processes mesh data without rendering — extracts vertex data, applies modifiers, exports statistics.

```python
#!/usr/bin/env python3
"""Headless mesh data extraction.
Usage: blender -b model.blend --python-exit-code 1 -P extract_data.py -- --output stats.json
"""
# Blender 3.x/4.x/5.x
import bpy
import sys
import os
import json
import argparse

def parse_args():
    argv = sys.argv
    if "--" not in argv:
        sys.exit(1)
    parser = argparse.ArgumentParser()
    parser.add_argument("--output", required=True, help="Output JSON file")
    parser.add_argument("--apply-modifiers", action="store_true")
    return parser.parse_args(argv[argv.index("--") + 1:])

def get_mesh_stats(obj, apply_modifiers=False):
    """Extract mesh statistics from an object."""
    if apply_modifiers:
        depsgraph = bpy.context.evaluated_depsgraph_get()
        obj_eval = obj.evaluated_get(depsgraph)
        mesh = obj_eval.to_mesh()
    else:
        mesh = obj.data

    stats = {
        "name": obj.name,
        "vertices": len(mesh.vertices),
        "edges": len(mesh.edges),
        "polygons": len(mesh.polygons),
        "location": list(obj.location),
        "dimensions": list(obj.dimensions),
        "modifiers": [mod.name for mod in obj.modifiers],
    }

    # Bounding box
    bbox = [list(v) for v in obj.bound_box]
    stats["bounding_box"] = {
        "min": [min(v[i] for v in bbox) for i in range(3)],
        "max": [max(v[i] for v in bbox) for i in range(3)],
    }

    if apply_modifiers:
        obj_eval.to_mesh_clear()

    return stats

def main():
    args = parse_args()

    if not bpy.app.background:
        print("WARNING: Running in foreground mode — use -b for headless")

    mesh_objects = [obj for obj in bpy.data.objects if obj.type == 'MESH']
    if not mesh_objects:
        print("ERROR: No mesh objects found", file=sys.stderr)
        sys.exit(1)

    results = {
        "file": bpy.data.filepath or "(unsaved)",
        "blender_version": list(bpy.app.version),
        "object_count": len(mesh_objects),
        "objects": [],
    }

    for obj in mesh_objects:
        stats = get_mesh_stats(obj, apply_modifiers=args.apply_modifiers)
        results["objects"].append(stats)
        print(f"  {obj.name}: {stats['vertices']} verts, {stats['polygons']} faces")

    total_verts = sum(o["vertices"] for o in results["objects"])
    total_faces = sum(o["polygons"] for o in results["objects"])
    results["totals"] = {"vertices": total_verts, "polygons": total_faces}

    os.makedirs(os.path.dirname(os.path.abspath(args.output)), exist_ok=True)
    with open(args.output, 'w') as f:
        json.dump(results, f, indent=2)

    print(f"\nExported stats for {len(mesh_objects)} objects to {args.output}")
    print(f"Totals: {total_verts} vertices, {total_faces} polygons")

if __name__ == "__main__":
    main()
```

---

## Example 5: Multi-Format Export Pipeline

Exports the current scene to multiple formats in one pass.

```python
#!/usr/bin/env python3
"""Export scene to multiple formats simultaneously.
Usage: blender -b scene.blend --python-exit-code 1 -P multi_export.py -- --output-dir /exports --formats GLB,FBX,USD
"""
# Blender 3.x/4.x/5.x
import bpy
import sys
import os
import argparse

def parse_args():
    argv = sys.argv
    if "--" not in argv:
        sys.exit(1)
    parser = argparse.ArgumentParser()
    parser.add_argument("--output-dir", required=True)
    parser.add_argument("--formats", required=True,
                        help="Comma-separated: GLB,FBX,OBJ,USD,STL,ABC")
    parser.add_argument("--selected-only", action="store_true")
    parser.add_argument("--name", default=None,
                        help="Base filename (default: blend file name)")
    return parser.parse_args(argv[argv.index("--") + 1:])

EXPORTERS = {
    "GLB": {
        "ext": ".glb",
        "fn": lambda fp, sel: bpy.ops.export_scene.gltf(
            filepath=fp, export_format='GLB',
            use_selection=sel, export_apply=True,
            export_materials='EXPORT',
        ),
    },
    "FBX": {
        "ext": ".fbx",
        "fn": lambda fp, sel: bpy.ops.export_scene.fbx(
            filepath=fp, use_selection=sel,
            apply_scale_options='FBX_SCALE_ALL',
            use_mesh_modifiers=True, use_custom_props=True,
        ),
    },
    "OBJ": {
        "ext": ".obj",
        "fn": lambda fp, sel: (
            bpy.ops.wm.obj_export(filepath=fp, export_selected_objects=sel,
                                   apply_modifiers=True)
            if bpy.app.version >= (4, 0, 0) else
            bpy.ops.export_scene.obj(filepath=fp, use_selection=sel,
                                      use_mesh_modifiers=True)
        ),
    },
    "USD": {
        "ext": ".usdc",
        "fn": lambda fp, sel: bpy.ops.wm.usd_export(
            filepath=fp, selected_objects_only=sel,
            export_materials=True, use_instancing=True,
        ),
    },
    "STL": {
        "ext": ".stl",
        "fn": lambda fp, sel: (
            bpy.ops.wm.stl_export(filepath=fp, export_selected_objects=sel,
                                   apply_modifiers=True)
            if bpy.app.version >= (4, 0, 0) else
            bpy.ops.export_mesh.stl(filepath=fp, use_selection=sel,
                                     use_mesh_modifiers=True)
        ),
    },
    "ABC": {
        "ext": ".abc",
        "fn": lambda fp, sel: bpy.ops.wm.alembic_export(
            filepath=fp, selected=sel,
            export_custom_properties=True,
        ),
    },
}

def main():
    args = parse_args()
    os.makedirs(args.output_dir, exist_ok=True)

    base_name = args.name or os.path.splitext(
        os.path.basename(bpy.data.filepath)
    )[0] or "export"

    formats = [f.strip().upper() for f in args.formats.split(",")]

    for fmt in formats:
        if fmt not in EXPORTERS:
            print(f"SKIP: Unknown format '{fmt}'", file=sys.stderr)
            continue

        exporter = EXPORTERS[fmt]
        output_path = os.path.join(args.output_dir, base_name + exporter["ext"])

        try:
            exporter["fn"](output_path, args.selected_only)
            size = os.path.getsize(output_path)
            print(f"OK: {fmt} -> {output_path} ({size:,} bytes)")
        except Exception as e:
            print(f"FAIL: {fmt}: {e}", file=sys.stderr)

if __name__ == "__main__":
    main()
```

---

## Example 6: CI/CD Validation Script

Validates a .blend file meets quality requirements (triangle count, naming, materials).

```python
#!/usr/bin/env python3
"""Validate .blend file for production pipeline.
Usage: blender -b model.blend --python-exit-code 1 -P validate.py -- --max-tris 100000
"""
# Blender 3.x/4.x/5.x
import bpy
import sys
import argparse

def parse_args():
    argv = sys.argv
    if "--" not in argv:
        sys.exit(1)
    parser = argparse.ArgumentParser()
    parser.add_argument("--max-tris", type=int, default=100000)
    parser.add_argument("--require-materials", action="store_true")
    parser.add_argument("--naming-prefix", default=None)
    return parser.parse_args(argv[argv.index("--") + 1:])

def count_triangles(obj):
    depsgraph = bpy.context.evaluated_depsgraph_get()
    obj_eval = obj.evaluated_get(depsgraph)
    mesh = obj_eval.to_mesh()
    tri_count = sum(len(p.vertices) - 2 for p in mesh.polygons)
    obj_eval.to_mesh_clear()
    return tri_count

def main():
    args = parse_args()
    errors = []
    warnings = []

    mesh_objects = [o for o in bpy.data.objects if o.type == 'MESH']

    # Check triangle count
    total_tris = 0
    for obj in mesh_objects:
        tris = count_triangles(obj)
        total_tris += tris
        if tris > args.max_tris // 2:
            warnings.append(f"{obj.name}: {tris:,} triangles (high)")

    if total_tris > args.max_tris:
        errors.append(f"Total triangles {total_tris:,} exceeds limit {args.max_tris:,}")

    # Check materials
    if args.require_materials:
        for obj in mesh_objects:
            if not obj.data.materials:
                errors.append(f"{obj.name}: no materials assigned")

    # Check naming convention
    if args.naming_prefix:
        for obj in mesh_objects:
            if not obj.name.startswith(args.naming_prefix):
                errors.append(f"{obj.name}: does not match prefix '{args.naming_prefix}'")

    # Report results
    print(f"File: {bpy.data.filepath}")
    print(f"Objects: {len(mesh_objects)}, Triangles: {total_tris:,}")

    for w in warnings:
        print(f"  WARNING: {w}")
    for e in errors:
        print(f"  ERROR: {e}")

    if errors:
        print(f"\nVALIDATION FAILED: {len(errors)} error(s)")
        sys.exit(1)
    else:
        print("\nVALIDATION PASSED")

if __name__ == "__main__":
    main()
```

---

## Example 7: Append and Link with Library Overrides

```python
# Blender 4.0+ — Link collection and create library override
import bpy
import os

source_blend = "/path/to/furniture_library.blend"
collection_name = "DiningSet"

# Step 1: Link the collection
bpy.ops.wm.link(
    filepath=os.path.join(source_blend, "Collection", collection_name),
    directory=os.path.join(source_blend, "Collection"),
    filename=collection_name,
)

# Step 2: Find the linked collection instance
linked_obj = None
for obj in bpy.context.selected_objects:
    if obj.instance_collection and obj.instance_collection.name == collection_name:
        linked_obj = obj
        break

# Step 3: Create library override (makes linked data locally editable)
if linked_obj:
    # Blender 4.0+ — Library Overrides replace the old Make Local
    with bpy.context.temp_override(active_object=linked_obj, object=linked_obj):
        bpy.ops.object.make_override_library()
    print(f"Created library override for: {collection_name}")
```

---

## Example 8: Render with Compositing Output Nodes

```python
# Blender 3.x/4.x/5.x — Render with multiple output passes
import bpy
import os

scene = bpy.context.scene
scene.render.engine = 'CYCLES'
scene.use_nodes = True  # Enable compositing

# Enable render passes
view_layer = scene.view_layers[0]
view_layer.use_pass_z = True           # Depth
view_layer.use_pass_normal = True      # Normals
view_layer.use_pass_diffuse_color = True  # Albedo

# Set up file output node
tree = scene.node_tree
tree.nodes.clear()

rl_node = tree.nodes.new('CompositorNodeRLayers')
composite = tree.nodes.new('CompositorNodeComposite')
file_output = tree.nodes.new('CompositorNodeOutputFile')

output_dir = "/tmp/render_passes"
os.makedirs(output_dir, exist_ok=True)
file_output.base_path = output_dir
file_output.format.file_format = 'OPEN_EXR'

# Connect passes
tree.links.new(rl_node.outputs['Image'], composite.inputs['Image'])
tree.links.new(rl_node.outputs['Image'], file_output.inputs[0])

# Add slots for additional passes
file_output.file_slots.new('Depth')
tree.links.new(rl_node.outputs['Depth'], file_output.inputs['Depth'])

file_output.file_slots.new('Normal')
tree.links.new(rl_node.outputs['Normal'], file_output.inputs['Normal'])

# Render
scene.render.filepath = os.path.join(output_dir, "beauty_")
bpy.ops.render.render(write_still=True)
print(f"Rendered passes to: {output_dir}")
```

---

## Sources

- https://docs.blender.org/api/current/bpy.ops.wm.html
- https://docs.blender.org/api/current/bpy.ops.export_scene.html
- https://docs.blender.org/api/current/bpy.ops.import_scene.html
- https://docs.blender.org/api/current/bpy.ops.render.html
- https://docs.blender.org/api/current/bpy.data.html
- https://docs.blender.org/api/current/bpy.app.html
- https://docs.blender.org/manual/en/latest/advanced/command_line/arguments.html
