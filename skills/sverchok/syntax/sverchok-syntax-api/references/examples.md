# sverchok-syntax-api: Working Code Examples

All examples based on Sverchok v1.4.0+ source code analysis.
Source: https://github.com/nortikin/sverchok, vooronderzoek-sverchok.md §9

---

## Example 1: Build a Complete Node Tree from Script

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

# Create a parametric grid tree entirely from script
tree = bpy.data.node_groups.new("ScriptedGrid", 'SverchCustomTreeType')

with tree.init_tree():
    # Number input for grid resolution
    num = tree.nodes.new('SvNumberNode')
    num.location = (0, 0)

    # Plane generator (grid)
    plane = tree.nodes.new('SvPlaneNode')
    plane.location = (200, 0)

    # Noise deformation
    noise = tree.nodes.new('SvNoiseNodeMK3')
    noise.location = (200, -200)

    # Vector math to apply noise as Z displacement
    vec_math = tree.nodes.new('SvVectorMathNodeMK3')
    vec_math.location = (400, 0)

    # Viewer for viewport display
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (600, 0)

    # Connect the chain
    tree.links.new(num.outputs[0], plane.inputs[0])        # Resolution
    tree.links.new(plane.outputs['Vertices'], vec_math.inputs[0])
    tree.links.new(noise.outputs[0], vec_math.inputs[1])
    tree.links.new(vec_math.outputs[0], viewer.inputs['vertices'])
    tree.links.new(plane.outputs['Edges'], viewer.inputs['edges'])
    tree.links.new(plane.outputs['Faces'], viewer.inputs['faces'])

# Force initial evaluation
tree.force_update()
print(f"Tree created: {len(tree.nodes)} nodes, {len(tree.links)} links")
```

---

## Example 2: Parameter Sweep — Export Results to CSV

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
import csv
import os

def parameter_sweep_to_csv(tree_name, sweep_config, output_node_name, output_socket, csv_path):
    """Run a parameter sweep and export results to CSV.

    Args:
        tree_name: Name of the Sverchok tree
        sweep_config: dict mapping param names to lists of values
            e.g., {"NumberNode.int_": [5, 10, 15], "ScaleNode.factor": [1.0, 2.0]}
        output_node_name: Name of the node to read results from
        output_socket: Name of the output socket
        csv_path: Output CSV file path
    """
    tree = bpy.data.node_groups[tree_name]
    tree.sv_process = True
    tree.sv_show = False  # Disable viewport for speed

    # Build param combinations
    import itertools
    param_names = list(sweep_config.keys())
    param_values = list(sweep_config.values())
    combinations = list(itertools.product(*param_values))

    rows = []
    for combo in combinations:
        # Apply parameters
        for param_path, value in zip(param_names, combo):
            node_name, prop_name = param_path.split(".", 1)
            setattr(tree.nodes[node_name], prop_name, value)

        tree.force_update()

        # Read output
        data = tree.nodes[output_node_name].outputs[output_socket].sv_get(deepcopy=False)

        # Record: param values + summary of output
        row = list(combo)
        for obj_data in data:
            row.append(len(obj_data))  # e.g., vertex count per object
        rows.append(row)

    # Write CSV
    with open(csv_path, 'w', newline='') as f:
        writer = csv.writer(f)
        header = param_names + [f"obj_{i}_count" for i in range(len(data))]
        writer.writerow(header)
        writer.writerows(rows)

    tree.sv_show = True
    print(f"Sweep complete: {len(rows)} combinations -> {csv_path}")

# Usage:
# parameter_sweep_to_csv(
#     "MyTree",
#     {"NumberNode.int_": [5, 10, 20], "ScaleNode.factor": [0.5, 1.0, 2.0]},
#     "Box", "Vertices",
#     "/tmp/sweep_results.csv"
# )
```

---

## Example 3: Clone and Modify an Existing Tree

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def clone_tree_with_modifications(source_name, new_name, modifications):
    """Clone a Sverchok tree and apply modifications.

    Args:
        source_name: Name of existing tree to clone
        new_name: Name for the cloned tree
        modifications: dict of {node_name: {prop_name: new_value}}
    """
    source = bpy.data.node_groups.get(source_name)
    if source is None or source.bl_idname != 'SverchCustomTreeType':
        raise RuntimeError(f"Source tree '{source_name}' not found")

    # Copy the tree (bpy.data — see blender-core-api skill)
    clone = source.copy()
    clone.name = new_name

    # Apply modifications
    for node_name, props in modifications.items():
        node = clone.nodes.get(node_name)
        if node is None:
            print(f"WARNING: Node '{node_name}' not found in clone")
            continue
        for prop_name, value in props.items():
            setattr(node, prop_name, value)

    # Force re-evaluation with new parameters
    clone.sv_process = True
    clone.force_update()

    return clone

# Usage:
# variant = clone_tree_with_modifications(
#     "BaseDesign", "Variant_A",
#     {"Subdivisions": {"int_": 8}, "Scale": {"factor": 1.5}}
# )
```

---

## Example 4: Extract Geometry and Create Blender Mesh Objects

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def sverchok_to_mesh(tree_name, node_name, collection_name="SvExport"):
    """Extract geometry from a Sverchok node and create Blender mesh objects.

    Args:
        tree_name: Name of the Sverchok tree
        node_name: Name of the node with geometry output
        collection_name: Target collection for created objects
    """
    tree = bpy.data.node_groups[tree_name]
    tree.sv_process = True
    tree.force_update()

    node = tree.nodes[node_name]
    vertices = node.outputs['Vertices'].sv_get(deepcopy=False)
    edges = node.outputs['Edges'].sv_get(deepcopy=False)
    faces = node.outputs['Faces'].sv_get(deepcopy=False)

    # Ensure target collection exists (bpy.data — see blender-core-api skill)
    coll = bpy.data.collections.get(collection_name)
    if coll is None:
        coll = bpy.data.collections.new(collection_name)
        bpy.context.scene.collection.children.link(coll)

    created = []
    for i, (v, e, f) in enumerate(zip(vertices, edges, faces)):
        mesh = bpy.data.meshes.new(f"Sv_{node_name}_{i}")
        mesh.from_pydata(
            [tuple(vert) for vert in v],
            [tuple(edge) for edge in e],
            [tuple(face) for face in f]
        )
        mesh.update()

        obj = bpy.data.objects.new(f"Sv_{node_name}_{i}", mesh)
        coll.objects.link(obj)
        created.append(obj)

    print(f"Created {len(created)} mesh objects from '{node_name}'")
    return created

# Usage:
# objects = sverchok_to_mesh("MyTree", "Box", "ExportedGeometry")
```

---

## Example 5: Debug a Sverchok Tree

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def debug_tree(tree_name):
    """Print diagnostic information for a Sverchok tree."""
    tree = bpy.data.node_groups.get(tree_name)
    if tree is None or tree.bl_idname != 'SverchCustomTreeType':
        print(f"ERROR: '{tree_name}' is not a Sverchok tree")
        return

    print(f"=== Tree: {tree.name} ===")
    print(f"  sv_process:      {tree.sv_process}")
    print(f"  sv_animate:      {tree.sv_animate}")
    print(f"  sv_draft:        {tree.sv_draft}")
    print(f"  sv_scene_update: {tree.sv_scene_update}")
    print(f"  Nodes: {len(tree.nodes)}, Links: {len(tree.links)}")

    for node in tree.nodes:
        status = "OK" if node.US_is_updated else "STALE"
        error = f" ERROR: {node.US_error}" if node.US_error else ""
        time_ms = node.US_time * 1000 if node.US_time else 0
        print(f"  [{status}] {node.bl_idname}: '{node.name}' ({time_ms:.1f}ms){error}")

# Usage:
# debug_tree("MyParametricTree")
```

---

## Example 6: Animate Parameters via Script

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
import math

def animate_tree_parameter(tree_name, node_name, prop_name, frame_start, frame_end, func):
    """Animate a Sverchok node parameter across a frame range.

    Args:
        tree_name: Name of the Sverchok tree
        node_name: Name of the node to animate
        prop_name: Property name on the node
        frame_start: First frame
        frame_end: Last frame
        func: callable(frame) -> value
    """
    tree = bpy.data.node_groups[tree_name]
    tree.sv_process = True
    tree.sv_animate = True

    node = tree.nodes[node_name]

    for frame in range(frame_start, frame_end + 1):
        bpy.context.scene.frame_set(frame)
        value = func(frame)
        setattr(node, prop_name, value)
        tree.force_update()

    print(f"Animated '{node_name}.{prop_name}' frames {frame_start}-{frame_end}")

# Usage:
# animate_tree_parameter(
#     "MyTree", "NumberNode", "float_",
#     1, 120,
#     lambda frame: math.sin(frame * 0.1) * 5.0
# )
```

---

## Example 7: List Matching Across Multiple Inputs

```python
# Sverchok v1.4.0+
from sverchok.data_structure import match_long_repeat, fullList

# Scenario: 3 objects of vertices but only 1 scale value
verts = [
    [(0,0,0), (1,0,0), (0,1,0)],    # Object 0
    [(2,0,0), (3,0,0), (2,1,0)],    # Object 1
    [(4,0,0), (5,0,0), (4,1,0)],    # Object 2
]
scales = [[2.0]]  # Only 1 object of scale

# match_long_repeat extends scales to match verts length
matched_verts, matched_scales = match_long_repeat([verts, scales])
# matched_scales = [[2.0], [2.0], [2.0]]

# Apply scaling
result = []
for obj_verts, obj_scales in zip(matched_verts, matched_scales):
    # fullList extends scales within each object to match vertex count
    s = list(obj_scales)
    fullList(s, len(obj_verts))
    scaled = [(v[0]*f, v[1]*f, v[2]*f) for v, f in zip(obj_verts, s)]
    result.append(scaled)
```

---

## Example 8: Build a Multi-Tree Workflow

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def build_multi_tree_pipeline():
    """Create multiple linked Sverchok trees for a design pipeline."""
    trees = {}

    # Tree 1: Generate base geometry
    trees['base'] = bpy.data.node_groups.new("Pipeline_Base", 'SverchCustomTreeType')
    with trees['base'].init_tree():
        box = trees['base'].nodes.new('SvBoxNodeMk2')
        box.location = (0, 0)
        steth = trees['base'].nodes.new('SvStethoscopeNodeMK2')
        steth.location = (200, 0)
        trees['base'].links.new(box.outputs['Vertices'], steth.inputs[0])

    # Tree 2: Transform
    trees['transform'] = bpy.data.node_groups.new("Pipeline_Transform", 'SverchCustomTreeType')
    with trees['transform'].init_tree():
        obj_in = trees['transform'].nodes.new('SvObjectNodeMK2')
        obj_in.location = (0, 0)
        transform = trees['transform'].nodes.new('SvMatrixApplyJoinNode')
        transform.location = (200, 0)
        viewer = trees['transform'].nodes.new('SvViewerDrawMk4')
        viewer.location = (400, 0)

    # Force update all trees
    for tree in trees.values():
        tree.sv_process = True
        tree.force_update()

    return trees

# Usage:
# pipeline = build_multi_tree_pipeline()
```

---

## Example 9: Conditional Tree Processing

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def process_trees_selectively(filter_func=None):
    """Process only Sverchok trees matching a filter.

    Args:
        filter_func: callable(tree) -> bool, processes tree if True.
                     If None, processes all trees.
    """
    sv_trees = [
        ng for ng in bpy.data.node_groups
        if ng.bl_idname == 'SverchCustomTreeType'
    ]

    for tree in sv_trees:
        if filter_func and not filter_func(tree):
            continue

        tree.sv_process = True
        tree.force_update()
        print(f"Processed: {tree.name} ({len(tree.nodes)} nodes)")

# Usage: process only trees with "Export" in the name
# process_trees_selectively(lambda t: "Export" in t.name)

# Usage: process only trees in draft mode
# process_trees_selectively(lambda t: t.sv_draft)
```
