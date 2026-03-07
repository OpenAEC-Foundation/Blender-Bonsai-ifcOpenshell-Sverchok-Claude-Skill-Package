# Sverchok Code Validator — Before/After Examples

Concrete before/after examples for each validation check.

---

## CHECK 1: Data Nesting Correctness

### Vertices — Level 3

```python
# BEFORE (BLOCKER): Missing object wrapper — level 2
verts = [(0,0,0), (1,0,0), (1,1,0)]
self.outputs['Vertices'].sv_set(verts)

# AFTER: Correct level 3
verts = [[(0,0,0), (1,0,0), (1,1,0)]]
self.outputs['Vertices'].sv_set(verts)
```

### Strings/Numbers — Level 2

```python
# BEFORE (BLOCKER): Flat list — level 1
numbers = [1, 2, 3]
self.outputs['Numbers'].sv_set(numbers)

# AFTER: Correct level 2
numbers = [[1, 2, 3]]
self.outputs['Numbers'].sv_set(numbers)
```

### Matrices — Level 1

```python
# BEFORE (BLOCKER): Double-wrapped — level 2
matrices = [[Matrix(), Matrix()]]
self.outputs['Matrices'].sv_set(matrices)

# AFTER: Correct level 1
matrices = [Matrix(), Matrix()]
self.outputs['Matrices'].sv_set(matrices)
```

---

## CHECK 2: updateNode Callback

```python
# BEFORE (BLOCKER): No update callback
scale: FloatProperty(name='Scale', default=1.0)

# AFTER: With updateNode
scale: FloatProperty(name='Scale', default=1.0, update=updateNode)
```

---

## CHECK 3: Output Connection Check

```python
# BEFORE (WARNING): No early exit
def process(self):
    data = expensive_computation()
    self.outputs['Result'].sv_set(data)

# AFTER: Early exit when unconnected
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return
    data = expensive_computation()
    self.outputs['Result'].sv_set(data)
```

---

## CHECK 4: match_long_repeat Before zip

```python
# BEFORE (BLOCKER): Direct zip truncates data
verts = self.inputs['Vertices'].sv_get()
scale = self.inputs['Scale'].sv_get()
for v, s in zip(verts, scale):
    ...

# AFTER: Match lengths first
verts = self.inputs['Vertices'].sv_get()
scale = self.inputs['Scale'].sv_get()
verts, scale = match_long_repeat([verts, scale])
for v, s in zip(verts, scale):
    ...
```

---

## CHECK 5: Socket Creation Location

```python
# BEFORE (BLOCKER): Socket created in process()
def process(self):
    self.inputs.new('SvStringsSocket', 'Extra')
    data = self.inputs['Extra'].sv_get()
    ...

# AFTER: Socket created in sv_init()
def sv_init(self, context):
    self.inputs.new('SvStringsSocket', 'Extra')

def process(self):
    data = self.inputs['Extra'].sv_get()
    ...
```

---

## CHECK 6: deepcopy for Input Data

```python
# BEFORE (WARNING): Mutating cached input data
data = self.inputs['Data'].sv_get(deepcopy=False)
data[0].append(999)  # Corrupts cache

# AFTER: Safe copy
data = self.inputs['Data'].sv_get(deepcopy=True)
data[0].append(999)  # Safe
```

---

## CHECK 7: SNLite Aliases

```python
# BEFORE (BLOCKER): Invalid aliases
"""
in  verts  vertices
out result string
"""

# AFTER: Correct single-letter aliases
"""
in  verts  v
out result s
"""
```

---

## CHECK 8: Node Docstring

```python
# BEFORE (INFO): Missing Triggers/Tooltip
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    """My custom node."""

# AFTER: Proper docstring format
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: scale transform multiply
    Tooltip: Scales vertices by a factor
    """
```

---

## CHECK 9: Standard Process Pattern

```python
# BEFORE (WARNING): Non-standard process
def process(self):
    data = self.inputs['Data'].sv_get()
    result = data[0][0] * 2
    self.outputs['Result'].sv_set(result)

# AFTER: Standard 5-step pattern
def process(self):
    if not any(s.is_linked for s in self.outputs):
        return
    data = self.inputs['Data'].sv_get(default=[[0]])
    result = []
    for obj in data:
        result.append([v * 2 for v in obj])
    self.outputs['Result'].sv_set(result)
```

---

## CHECK 10: IfcSverchok Double-Nesting

```python
# BEFORE (BLOCKER): Single-nested IFC data
self.outputs['Names'].sv_set(["Wall_001"])

# AFTER: Correct Sverchok nesting for IFC
self.outputs['Names'].sv_set([["Wall_001"]])
```

---

## CHECK 11: BMesh Cleanup

```python
# BEFORE (BLOCKER): Missing bm.free()
bm = bmesh_from_pydata(verts, edges, faces)
bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=1)
new_v, new_e, new_f = pydata_from_bmesh(bm)
# bm.free() is missing — memory leak

# AFTER: Proper cleanup
bm = bmesh_from_pydata(verts, edges, faces)
bmesh.ops.subdivide_edges(bm, edges=bm.edges[:], cuts=1)
new_v, new_e, new_f = pydata_from_bmesh(bm)
bm.free()
```

---

## CHECK 12: NumPy Optimization

```python
# BEFORE (INFO): Python loop
result = []
for v in vertices:
    result.append((v[0] * 2, v[1] * 2, v[2] * 2))

# AFTER: NumPy vectorized
import numpy as np
result = (np.array(vertices) * 2).tolist()
```

---

## CHECK 13: Import Validation

```python
# BEFORE (BLOCKER): Missing imports
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    scale: FloatProperty(update=updateNode)  # updateNode not imported
    def process(self):
        a, b = match_long_repeat([a, b])  # match_long_repeat not imported

# AFTER: Correct imports
from sverchok.data_structure import updateNode, match_long_repeat
from sverchok.node_tree import SverchCustomTreeNode

class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    scale: FloatProperty(update=updateNode)
    def process(self):
        a, b = match_long_repeat([a, b])
```

---

## CHECK 14: Socket Name Consistency

```python
# BEFORE (BLOCKER): Name mismatch
def sv_init(self, context):
    self.inputs.new('SvVerticesSocket', 'Vertices')

def process(self):
    verts = self.inputs['Verts'].sv_get()  # KeyError: 'Verts'

# AFTER: Matching names
def sv_init(self, context):
    self.inputs.new('SvVerticesSocket', 'Vertices')

def process(self):
    verts = self.inputs['Vertices'].sv_get()
```

---

## CHECK 15: match_long_repeat Unpacking

```python
# BEFORE (WARNING): Not unpacked
result = match_long_repeat([verts, scale])
for v, s in zip(result[0], result[1]):
    ...

# AFTER: Properly unpacked
verts, scale = match_long_repeat([verts, scale])
for v, s in zip(verts, scale):
    ...
```

---

## CHECK 16: sv_get Default

```python
# BEFORE (WARNING): No default — crashes when unconnected
scale = self.inputs['Scale'].sv_get()

# AFTER: With default value
scale = self.inputs['Scale'].sv_get(default=[[1.0]])
```

---

## CHECK 17: Registration Completeness

```python
# BEFORE (BLOCKER): Missing bl_idname
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_label = 'My Node'

# AFTER: Complete registration
class SvMyNode(SverchCustomTreeNode, bpy.types.Node):
    bl_idname = 'SvMyNode'
    bl_label = 'My Node'
    bl_icon = 'NONE'
    sv_category = 'Custom'
```

---

## CHECK 18: Property Type Validation

```python
# BEFORE (BLOCKER): Old assignment syntax
count = IntProperty(name='Count', default=5, update=updateNode)

# AFTER: Annotation syntax (Blender 2.80+)
count: IntProperty(name='Count', default=5, update=updateNode)
```

---

## CHECK 19: Edge/Face Data Format

```python
# BEFORE (BLOCKER): Float indices and wrong tuple size
edges = [[(0.0, 1.0, 2.0)]]  # 3 elements, floats
faces = [[(0, 1)]]            # Only 2 elements

# AFTER: Integer indices, correct sizes
edges = [[(0, 1), (1, 2)]]    # 2-element tuples, integers
faces = [[(0, 1, 2)]]         # 3+ element tuples, integers
```

---

## Complete Correct Node Example

```python
import bpy
from bpy.props import FloatProperty
from sverchok.node_tree import SverchCustomTreeNode
from sverchok.data_structure import updateNode, match_long_repeat

class SvScaleNode(SverchCustomTreeNode, bpy.types.Node):
    """
    Triggers: scale transform multiply
    Tooltip: Scales vertices by a factor
    """
    bl_idname = 'SvScaleNode'
    bl_label = 'Scale Vertices'
    bl_icon = 'NONE'
    sv_category = 'Transforms'

    scale_factor: FloatProperty(
        name='Scale', default=1.0, update=updateNode
    )

    def sv_init(self, context):
        self.inputs.new('SvVerticesSocket', 'Vertices')
        self.inputs.new('SvStringsSocket', 'Scale').prop_name = 'scale_factor'
        self.outputs.new('SvVerticesSocket', 'Vertices')

    def process(self):
        if not any(s.is_linked for s in self.outputs):
            return

        verts = self.inputs['Vertices'].sv_get(default=[[]])
        scale = self.inputs['Scale'].sv_get()

        verts, scale = match_long_repeat([verts, scale])

        result = []
        for vert_list, scale_list in zip(verts, scale):
            vert_list, scale_list = match_long_repeat([vert_list, scale_list])
            scaled = []
            for v, s in zip(vert_list, scale_list):
                scaled.append((v[0] * s, v[1] * s, v[2] * s))
            result.append(scaled)

        self.outputs['Vertices'].sv_set(result)

def register():
    bpy.utils.register_class(SvScaleNode)

def unregister():
    bpy.utils.unregister_class(SvScaleNode)
```
