# sverchok-core-concepts: Working Code Examples

All examples based on Sverchok v1.4.0+ source code analysis.
Source: https://github.com/nortikin/sverchok, vooronderzoek-sverchok.md §1, §2

---

## Example 1: Create and Populate a Sverchok Node Tree

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

# Create a new Sverchok node tree
tree = bpy.data.node_groups.new("MyParametricTree", 'SverchCustomTreeType')

# ALWAYS use init_tree() to suppress updates during construction
with tree.init_tree():
    # Add a Number node (input)
    num = tree.nodes.new('SvNumberNode')
    num.location = (0, 0)

    # Add a Box generator
    box = tree.nodes.new('SvBoxNodeMk2')
    box.location = (200, 0)

    # Add a Viewer Draw node (output/visualization)
    viewer = tree.nodes.new('SvViewerDrawMk4')
    viewer.location = (400, 0)

    # Connect: Number -> Box size, Box verts -> Viewer
    tree.links.new(num.outputs[0], box.inputs['Size'])
    tree.links.new(box.outputs['Vertices'], viewer.inputs['vertices'])
    tree.links.new(box.outputs['Edges'], viewer.inputs['edges'])
    tree.links.new(box.outputs['Faces'], viewer.inputs['faces'])

# Force initial evaluation
tree.force_update()
print(f"Tree '{tree.name}' created with {len(tree.nodes)} nodes")
```

---

## Example 2: Read Output Data from a Node

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups.get("MyParametricTree")
if tree is None or tree.bl_idname != 'SverchCustomTreeType':
    raise RuntimeError("Sverchok tree not found")

# Ensure the tree is processed
tree.sv_process = True
tree.force_update()

# Access a node's output data
box_node = tree.nodes.get("Box")
if box_node is None:
    raise RuntimeError("Box node not found")

# sv_get() returns nested lists: [[object1_data], [object2_data], ...]
try:
    vertices = box_node.outputs['Vertices'].sv_get(deepcopy=False)
    edges = box_node.outputs['Edges'].sv_get(deepcopy=False)
    faces = box_node.outputs['Faces'].sv_get(deepcopy=False)

    for obj_idx, (verts, edgs, facs) in enumerate(zip(vertices, edges, faces)):
        print(f"Object {obj_idx}: {len(verts)} verts, {len(edgs)} edges, {len(facs)} faces")
except Exception as e:
    print(f"No data: {e}")
```

---

## Example 3: Convert Sverchok Output to Blender Mesh

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

tree = bpy.data.node_groups["MyParametricTree"]
tree.force_update()

node = tree.nodes["Box"]
vertices = node.outputs['Vertices'].sv_get(deepcopy=False)
edges = node.outputs['Edges'].sv_get(deepcopy=False)
faces = node.outputs['Faces'].sv_get(deepcopy=False)

# Create Blender mesh objects from Sverchok output
for obj_idx, (verts, edgs, facs) in enumerate(zip(vertices, edges, faces)):
    mesh = bpy.data.meshes.new(f"SvOutput_{obj_idx}")
    # Convert tuples to the format from_pydata expects
    mesh.from_pydata(
        [tuple(v) for v in verts],
        [tuple(e) for e in edgs],
        [tuple(f) for f in facs]
    )
    mesh.update()  # ALWAYS call after from_pydata

    obj = bpy.data.objects.new(f"SvObject_{obj_idx}", mesh)
    bpy.context.collection.objects.link(obj)  # REQUIRED to appear in viewport

print(f"Created {len(vertices)} objects from Sverchok output")
```

---

## Example 4: Minimal Custom Sverchok Node

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvMyScaleNode(SverchCustomTreeNode, bpy.types.Node):
    """Scale vertices by a factor — minimal custom node example."""
    bl_idname = 'SvMyScaleNode'
    bl_label = 'My Scale'
    sv_category = 'Transforms'  # Appears in Shift+A > Transforms

    factor: bpy.props.FloatProperty(
        name="Factor",
        default=1.0,
        update=updateNode  # ALWAYS use updateNode for property callbacks
    )

    def sv_init(self, context):
        """Define input/output sockets."""
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Factor').prop_name = 'factor'
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def process(self):
        """Main computation — called by update system, NEVER directly."""
        if not self.outputs['Vertices'].is_linked:
            return  # Skip computation if output is not connected

        verts_in = self.inputs['Vertices'].sv_get(default=[[]])
        factor_in = self.inputs['Factor'].sv_get(default=[[self.factor]])

        # Match list lengths across inputs
        matched = match_long_repeat([verts_in, factor_in])

        result = []
        for verts, factors in zip(*matched):
            # match_long_repeat for inner lists too
            factors_matched = factors
            if len(factors) < len(verts):
                factors_matched = factors + [factors[-1]] * (len(verts) - len(factors))

            scaled = []
            for v, f in zip(verts, factors_matched):
                scaled.append((v[0] * f, v[1] * f, v[2] * f))
            result.append(scaled)

        self.outputs['Vertices'].sv_set(result)


def register():
    bpy.utils.register_class(SvMyScaleNode)

def unregister():
    bpy.utils.unregister_class(SvMyScaleNode)
```

---

## Example 5: Inspect Node Tree State and Debug

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def debug_sverchok_tree(tree_name):
    """Print diagnostic information for a Sverchok node tree."""
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
        status = "OK" if node.US_is_updated else "NOT UPDATED"
        error = f" ERROR: {node.US_error}" if node.US_error else ""
        time_ms = node.US_time * 1000 if node.US_time else 0
        print(f"  [{status}] {node.bl_idname}: '{node.name}' ({time_ms:.1f}ms){error}")

        # Show connected sockets
        for inp in node.inputs:
            linked = "LINKED" if inp.is_linked else "unlinked"
            print(f"    IN:  {inp.name} ({inp.bl_idname}) [{linked}]")
        for out in node.outputs:
            linked = "LINKED" if out.is_linked else "unlinked"
            print(f"    OUT: {out.name} ({out.bl_idname}) [{linked}]")

# Usage:
# debug_sverchok_tree("MyParametricTree")
```

---

## Example 6: Control Tree Properties Programmatically

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def setup_animation_tree(tree_name):
    """Configure a Sverchok tree for animation playback."""
    tree = bpy.data.node_groups.get(tree_name)
    if tree is None:
        raise RuntimeError(f"Tree '{tree_name}' not found")

    # Enable animation mode — tree will update on every frame change
    tree.sv_animate = True
    tree.sv_process = True

    # Disable draft mode for final quality
    tree.sv_draft = False

    # Enable viewport display
    tree.sv_show = True

    # Force initial evaluation
    tree.force_update()
    print(f"Tree '{tree_name}' configured for animation")


def toggle_draft_mode(tree_name):
    """Toggle draft mode for faster viewport interaction."""
    tree = bpy.data.node_groups.get(tree_name)
    if tree is None:
        raise RuntimeError(f"Tree '{tree_name}' not found")

    tree.sv_draft = not tree.sv_draft
    tree.force_update()
    print(f"Draft mode: {'ON' if tree.sv_draft else 'OFF'}")
```

---

## Example 7: Iterate All Sverchok Trees in a File

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def list_all_sverchok_trees():
    """Find and list all Sverchok node trees in the current file."""
    sv_trees = [
        ng for ng in bpy.data.node_groups
        if ng.bl_idname == 'SverchCustomTreeType'
    ]

    if not sv_trees:
        print("No Sverchok trees found")
        return

    for tree in sv_trees:
        node_count = len(tree.nodes)
        link_count = len(tree.links)
        active = "ACTIVE" if tree.sv_process else "PAUSED"
        print(f"  {tree.name}: {node_count} nodes, {link_count} links [{active}]")

    return sv_trees

# Usage:
# trees = list_all_sverchok_trees()
```

---

## Example 8: Data Nesting and match_long_repeat

```python
# Sverchok v1.4.0+ — understanding data nesting
from sverchok.data_structure import match_long_repeat

# Sverchok data is ALWAYS nested lists.
# Level 0: list of objects
# Level 1: data per object

# Single object with 3 vertices:
single_object_verts = [[(0, 0, 0), (1, 0, 0), (0, 1, 0)]]

# Two objects with different vertex counts:
two_objects_verts = [
    [(0, 0, 0), (1, 0, 0), (0, 1, 0)],      # Object 0: triangle
    [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)]  # Object 1: quad
]

# When inputs have different object counts, match_long_repeat aligns them:
verts = [[(0, 0, 0), (1, 0, 0)], [(2, 0, 0), (3, 0, 0), (4, 0, 0)]]
scales = [[2.0]]  # Only 1 object worth of scale data

matched = match_long_repeat([verts, scales])
# Result: verts unchanged (2 objects)
#         scales becomes [[2.0], [2.0]] (repeated to match 2 objects)

# Numbers follow the same convention:
numbers = [[1.0, 2.0, 3.0]]  # 1 object, 3 values
matrices = [[matrix_a, matrix_b]]  # 1 object, 2 matrices
```

---

## Example 9: Working with Socket Type Conversion

```python
# Sverchok v1.4.0+ — socket data flows and automatic conversion
# When connecting sockets of different types, Sverchok applies implicit conversion.

# Example: SvVerticesSocket -> SvStringsSocket
# Vertices [(x,y,z), ...] are converted to nested number lists [[x,y,z], ...]

# Example: SvStringsSocket -> SvVerticesSocket
# Number triples [[x,y,z], ...] are interpreted as vertex coordinates

# In a custom node, handle type checking explicitly:
class SvMyFlexibleNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyFlexibleNode'
    bl_label = 'Flexible Node'

    def sv_init(self, context):
        # SvStringsSocket accepts any data type
        self.inputs.new('SvStringsSocket', 'Data')
        self.outputs.new('SvStringsSocket', 'Result')

    def process(self):
        data = self.inputs['Data'].sv_get(default=[[]])
        # Data may be vertices, numbers, matrices depending on what's connected
        # ALWAYS validate the structure before processing
        result = []
        for obj_data in data:
            if obj_data and isinstance(obj_data[0], (list, tuple)) and len(obj_data[0]) == 3:
                # Looks like vertex data
                result.append([sum(v) for v in obj_data])
            else:
                # Treat as scalar data
                result.append(obj_data)
        self.outputs['Result'].sv_set(result)
```
