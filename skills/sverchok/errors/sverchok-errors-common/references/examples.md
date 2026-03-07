# sverchok-errors-common: Working Code Examples

All examples based on Sverchok v1.4.0+ source code analysis.
Sources: https://github.com/nortikin/sverchok,
https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcsverchok,
vooronderzoek-sverchok.md §12, §13

---

## Example 1: Correct Data Nesting for All Socket Types

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Demonstrates correct nesting for every common data type

def process(self):
    # NUMBERS: [[value, value, ...]] — depth 2
    self.outputs['Numbers'].sv_set([[1.0, 2.0, 3.0]])

    # VERTICES: [[(x,y,z), (x,y,z), ...]] — depth 3
    self.outputs['Vertices'].sv_set([[(0,0,0), (1,0,0), (1,1,0)]])

    # EDGES: [[(i,j), (i,j), ...]] — depth 3
    self.outputs['Edges'].sv_set([[(0,1), (1,2), (2,0)]])

    # FACES: [[(i,j,k,...), ...]] — depth 3
    self.outputs['Faces'].sv_set([[(0,1,2)]])

    # MATRICES: [Matrix, Matrix, ...] — depth 1 (NOT double-nested)
    from mathutils import Matrix
    self.outputs['Matrix'].sv_set([Matrix.Identity(4)])

    # STRINGS: [["text", "text", ...]] — depth 2
    self.outputs['Text'].sv_set([["hello", "world"]])

    # Multiple objects — outer list has multiple entries:
    self.outputs['MultiVerts'].sv_set([
        [(0,0,0), (1,0,0)],      # Object 0
        [(2,0,0), (3,0,0)],      # Object 1
    ])
```

---

## Example 2: Complete Custom Node with All Error Patterns Avoided

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvCorrectScaleNode(SverchCustomTreeNode, bpy.types.Node):
    """Scale vertices — demonstrates all correct patterns."""
    bl_idname = 'SvCorrectScaleNode'
    bl_label = 'Correct Scale'
    sv_category = 'Transforms'

    # CORRECT: update=updateNode on EVERY property (avoids AI Mistake 5, Error 2)
    factor: bpy.props.FloatProperty(
        name="Factor", default=1.0, update=updateNode
    )

    # CORRECT: Use sv_init, NOT __init__ (avoids AI Mistake 3)
    def sv_init(self, context):
        # CORRECT: Create sockets here, NOT in process() (avoids AI Mistake 7)
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Factor').prop_name = 'factor'
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def process(self):
        # CORRECT: Check output connections first (avoids Error 3)
        if not self.outputs['Vertices'].is_linked:
            return

        # CORRECT: deepcopy=True (default) since we read only (avoids Error 4)
        verts_in = self.inputs['Vertices'].sv_get(default=[[]])
        factor_in = self.inputs['Factor'].sv_get(default=[[self.factor]])

        # CORRECT: Use match_long_repeat, NOT zip (avoids Error 5, AI Mistake 6)
        verts_in, factor_in = match_long_repeat([verts_in, factor_in])

        result = []
        for verts, factors in zip(verts_in, factor_in):
            # Inner level also needs matching
            verts, factors = match_long_repeat([verts, factors])
            scaled = [(v[0]*f, v[1]*f, v[2]*f) for v, f in zip(verts, factors)]
            result.append(scaled)

        # CORRECT: Result is already object-wrapped (avoids Error 1, AI Mistakes 1,2)
        self.outputs['Vertices'].sv_set(result)


def register():
    bpy.utils.register_class(SvCorrectScaleNode)

def unregister():
    bpy.utils.unregister_class(SvCorrectScaleNode)
```

---

## Example 3: Safe Data Mutation Pattern

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
# Demonstrates safe vs unsafe data handling

class SvSafeMutationNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvSafeMutationNode'
    bl_label = 'Safe Mutation'

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def process(self):
        if not self.outputs['Vertices'].is_linked:
            return

        # Option A: deepcopy=True — safe to mutate in-place
        verts = self.inputs['Vertices'].sv_get(deepcopy=True)
        for obj_verts in verts:
            for i, v in enumerate(obj_verts):
                obj_verts[i] = (v[0] + 1, v[1], v[2])  # Safe — copy
        self.outputs['Vertices'].sv_set(verts)

        # Option B: deepcopy=False — build NEW output, do NOT mutate input
        # verts = self.inputs['Vertices'].sv_get(deepcopy=False)
        # result = [[(v[0]+1, v[1], v[2]) for v in obj] for obj in verts]
        # self.outputs['Vertices'].sv_set(result)  # New list, input untouched
```

---

## Example 4: Correct SNLite Script with Proper Aliases

```python
# SNLite node script — Sverchok v1.4.0+
# Demonstrates correct socket type aliases (avoids AI Mistake 4)

"""
in  verts  v    # v = SvVerticesSocket (vertices)
in  scale  s    # s = SvStringsSocket (numbers/strings)
out result v    # v = SvVerticesSocket (vertices)
"""

# Valid single-letter aliases:
# v = SvVerticesSocket
# s = SvStringsSocket
# m = SvMatrixSocket
# c = SvColorSocket

from sverchok.data_structure import match_long_repeat

def sv_main(verts=[[]], scale=[[1.0]]):
    verts, scale = match_long_repeat([verts, scale])
    result = []
    for obj_v, obj_s in zip(verts, scale):
        obj_v, obj_s = match_long_repeat([obj_v, obj_s])
        scaled = [(v[0]*s, v[1]*s, v[2]*s) for v, s in zip(obj_v, obj_s)]
        result.append(scaled)
    return result
```

---

## Example 5: match_long_repeat vs zip Comparison

```python
# Sverchok v1.4.0+ — demonstrating the difference
from sverchok.data_structure import match_long_repeat

# Input data with mismatched lengths
verts = [[(0,0,0), (1,0,0), (2,0,0)]]  # 1 object, 3 vertices
scale = [[2.0]]                          # 1 object, 1 value

# WRONG: zip truncates — only first vertex gets scaled
for v_list, s_list in zip(verts, scale):
    for v, s in zip(v_list, s_list):  # Only 1 iteration!
        print(f"zip: {v} * {s}")
# Output: zip: (0,0,0) * 2.0  — vertices 2 and 3 are LOST

# CORRECT: match_long_repeat extends — all vertices get scaled
verts_m, scale_m = match_long_repeat([verts, scale])
for v_list, s_list in zip(verts_m, scale_m):
    v_list, s_list = match_long_repeat([v_list, s_list])
    for v, s in zip(v_list, s_list):  # 3 iterations
        print(f"mlr: {v} * {s}")
# Output: mlr: (0,0,0) * 2.0, (1,0,0) * 2.0, (2,0,0) * 2.0
```

---

## Example 6: IfcSverchok Correct Data Nesting

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+ and IfcSverchok
# Demonstrates correct nesting for IFC node inputs (avoids AI Mistake 10)

class SvMyIfcNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyIfcNode'
    bl_label = 'My IFC Node'

    def sv_init(self, context):
        self.outputs.new('SvStringsSocket', 'Names')
        self.outputs.new('SvStringsSocket', 'Classes')
        self.outputs.new('SvVerticesSocket', 'Positions')

    def process(self):
        # CORRECT: IfcSverchok expects standard Sverchok double-nested lists
        self.outputs['Names'].sv_set([["Wall_001", "Wall_002"]])
        self.outputs['Classes'].sv_set([["IfcWall", "IfcWall"]])
        self.outputs['Positions'].sv_set([[(0,0,0), (5,0,0)]])

        # WRONG examples that will fail:
        # self.outputs['Names'].sv_set(["Wall_001"])      # Missing outer list
        # self.outputs['Classes'].sv_set("IfcWall")        # Not even a list
        # self.outputs['Positions'].sv_set([(0,0,0)])      # Missing object wrapper
```

---

## Example 7: IfcSverchok Entity Reference by GUID

```python
# Blender 4.0+/5.x with IfcSverchok
# Demonstrates stable entity references (avoids AI Mistake 9)

# WRONG: Storing STEP IDs — these change on every tree re-run
# wall_id = 42
# wall = ifc_file.by_id(wall_id)  # FAILS after re-run — purge invalidated ID

# CORRECT: Query by GlobalId (GUID) — stable across re-runs
wall_guid = "2O2Fr$t4X7Z8jBCV$0Oo9a"
wall = ifc_file.by_guid(wall_guid)

# CORRECT: Query by type — returns all entities of that type
walls = ifc_file.by_type("IfcWall")
for wall in walls:
    print(f"GUID: {wall.GlobalId}, Name: {wall.Name}")

# CORRECT: Query by attribute value
import ifcopenshell.util.selector
walls = ifcopenshell.util.selector.filter_elements(
    ifc_file, "IfcWall[Name='Wall_001']"
)
```

---

## Example 8: Matrix Data Nesting (Not Like Vertices)

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
from mathutils import Matrix

class SvMatrixExampleNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMatrixExampleNode'
    bl_label = 'Matrix Example'

    def sv_init(self, context):
        self.outputs.new('SvMatrixSocket', 'Matrix')

    def process(self):
        if not self.outputs['Matrix'].is_linked:
            return

        # CORRECT: Matrices at level 1 — [M1, M2, ...]
        m1 = Matrix.Identity(4)
        m2 = Matrix.Translation((1, 0, 0))
        self.outputs['Matrix'].sv_set([m1, m2])

        # WRONG: Do NOT double-nest matrices like vertices
        # self.outputs['Matrix'].sv_set([[m1, m2]])  # Over-nested
```
