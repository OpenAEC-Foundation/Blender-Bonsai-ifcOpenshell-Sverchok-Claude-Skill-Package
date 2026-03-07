# Sverchok Code Validator — Validation Methods Reference

Detailed detection logic and resolution for each of the 19 validation checks.

---

## CHECK 1: Data Nesting Correctness

**Detection method:**
1. Find all `self.outputs[...].sv_set(expr)` calls
2. Resolve `expr` to its construction site
3. Count list nesting depth from outermost `[` to innermost data element
4. Compare against required level for socket type

**Socket type resolution:**
- Look up socket type from `sv_init()`: `self.outputs.new('SvVerticesSocket', 'Name')` → vertices → level 3
- If socket type is ambiguous, check the socket class name in the `new()` call

**Nesting level counting rules:**
```
[Matrix(), Matrix()]                    → level 1 (list of matrices)
[[1, 2, 3]]                            → level 2 (list of list of scalars)
[[(0,0,0), (1,0,0)]]                   → level 3 (list of list of tuples)
```

**Resolution:** Wrap or unwrap data to match the required nesting level.

---

## CHECK 2: updateNode Callback

**Detection method:**
1. Find all class-level annotations matching pattern `: *Property(`
2. For each property, check if `update=updateNode` is present in the arguments
3. Verify `updateNode` is imported from `sverchok.data_structure`

**Resolution:** Add `update=updateNode` to the property declaration. Add the import if missing.

---

## CHECK 3: Output Connection Check

**Detection method:**
1. Locate the `process(self)` method
2. Check the first 5 lines for an early-exit pattern checking output connections
3. Valid patterns:
   - `if not any(s.is_linked for s in self.outputs): return`
   - `if not self.outputs['Name'].is_linked: return` (single output)
   - `if not self.outputs[0].is_linked: return`

**Resolution:** Insert at the start of `process()`:
```python
if not any(s.is_linked for s in self.outputs):
    return
```

---

## CHECK 4: match_long_repeat Before zip

**Detection method:**
1. Find all `zip(a, b, ...)` calls inside `process()`
2. For each zip argument, trace back to determine if it originates from `sv_get()`
3. If multiple arguments originate from different socket inputs, verify `match_long_repeat()` was called on those variables before the `zip()`

**Resolution:** Insert `a, b = match_long_repeat([a, b])` before the `zip()` call.

---

## CHECK 5: Socket Creation Location

**Detection method:**
1. Find all `.inputs.new(` and `.outputs.new(` calls
2. Determine the enclosing method name
3. Valid enclosing methods: `sv_init`, `sv_update`
4. Invalid: `process`, `draw`, or any other method

**Resolution:** Move socket creation to `sv_init()`.

---

## CHECK 6: deepcopy for Input Data

**Detection method:**
1. Find `sv_get()` calls and their assigned variables
2. Track all subsequent operations on those variables
3. Flag if any mutating operation is performed and `deepcopy=False` was passed

**Mutating operations:** `.append()`, `.extend()`, `.insert()`, `.pop()`, `.remove()`, `.sort()`, `.reverse()`, `del var[i]`, `var[i] = ...`

**Resolution:** Change `sv_get(deepcopy=False)` to `sv_get(deepcopy=True)` or `sv_get()`.

---

## CHECK 7: SNLite Alias Validation

**Detection method:**
1. Detect SNLite scripts by the presence of a header docstring with `in`/`out` declarations
2. Parse each `in`/`out` line: `in name type [options]`
3. Validate `type` against the allowed alias set: `s`, `v`, `m`, `o`, `C`, `S`, `So`, `SF`, `VF`, `D`, `FP`

**Resolution:** Replace invalid alias with the correct single-letter code.

---

## CHECK 8: Docstring Format

**Detection method:**
1. Find class definition inheriting from `SverchCustomTreeNode`
2. Check for immediate docstring after class declaration
3. Parse docstring for `Triggers:` and `Tooltip:` lines

**Resolution:** Add or fix docstring with correct format.

---

## CHECK 9: Standard Process Pattern

**Detection method:**
1. Locate `process(self)` method
2. Check for the 5 standard steps in order:
   - Output connection check (CHECK 3)
   - `sv_get()` calls with defaults
   - `match_long_repeat()` call
   - `for ... in zip(...)` iteration loop
   - `sv_set()` output calls

**Resolution:** Restructure `process()` to follow the standard pattern.

---

## CHECK 10: IfcSverchok Double-Nesting

**Detection method:**
1. Determine if the node tree context involves IfcSverchok (imports from `ifcsverchok`, bl_idname contains `SvIfc`, or node interacts with IFC nodes)
2. Verify all string/name data output follows level 2 nesting: `[["value"]]` not `["value"]`

**Resolution:** Wrap single-nested data in an additional list level.

---

## CHECK 11: BMesh Cleanup

**Detection method:**
1. Find all `bmesh.new()` and `bmesh_from_pydata()` calls
2. Track the assigned variable name
3. Verify `var.free()` is called after all operations
4. Exclude BMesh from `bmesh.from_edit_mesh()` (managed by Blender)

**Resolution:** Insert `bm.free()` after the last BMesh operation.

---

## CHECK 12: NumPy Optimization

**Detection method:**
1. Find `for` loops inside `process()` that iterate over vertex/numeric data
2. Check if loop body performs element-wise arithmetic on tuple components
3. Pattern: `(v[0] * s, v[1] * s, v[2] * s)` or similar

**Resolution:** Suggest NumPy vectorized alternative: `(np.array(data) * scalar).tolist()`.

---

## CHECK 13: Import Validation

**Detection method:**
1. Collect all identifiers used from Sverchok modules
2. Verify each has a matching import statement
3. Verify import paths are valid Sverchok module paths

**Known valid import paths:**
- `sverchok.data_structure` → `updateNode`, `match_long_repeat`, `fullList`, `repeat_last`
- `sverchok.node_tree` → `SverchCustomTreeNode`
- `sverchok.utils.sv_bmesh_utils` → `bmesh_from_pydata`, `pydata_from_bmesh`
- `sverchok.core.sv_custom_exceptions` → `SvNoDataError`

**Resolution:** Add the correct import statement.

---

## CHECK 14: Socket Name Consistency

**Detection method:**
1. Parse `sv_init()` for all `self.inputs.new('Type', 'Name')` and `self.outputs.new('Type', 'Name')` calls
2. Build a set of valid input and output socket names
3. Parse `process()` for all `self.inputs['Name']` and `self.outputs['Name']` references
4. Flag any name in process() not found in sv_init()

**Resolution:** Fix the socket name to match the one defined in `sv_init()`.

---

## CHECK 15: match_long_repeat Unpacking

**Detection method:**
1. Find `match_long_repeat([...])` calls
2. Count the number of elements in the input list
3. Verify the unpacking target has the same count

**Flag if:** `result = match_long_repeat([a, b, c])` (should be `a, b, c = ...`)

**Resolution:** Unpack to the correct number of variables.

---

## CHECK 16: sv_get Default Pattern

**Detection method:**
1. Find all `sv_get()` calls
2. Check if `default=` parameter is present
3. For mandatory inputs (marked with `is_mandatory=True`), default is not required
4. For optional inputs, flag if default is missing

**Recommended defaults by socket type:**
- Vertices: `default=[[]]`
- Strings/numbers: `default=[[0]]` or `default=[[1.0]]`
- Edges/faces: `default=[[]]`
- Matrices: `default=[]`

**Resolution:** Add appropriate `default=` parameter.

---

## CHECK 17: Registration Completeness

**Detection method:**
1. Find class definition inheriting from `SverchCustomTreeNode`
2. Check for `bl_idname` class attribute (string starting with `Sv`)
3. Check for `bl_label` class attribute
4. Check for `sv_category` class attribute
5. Check for `bl_icon` class attribute

**Resolution:** Add missing attributes.

---

## CHECK 18: Property Type Validation

**Detection method:**
1. Find all property annotations (`: *Property(`)
2. Verify annotation syntax uses `:` not `=`
3. Check property type appropriateness (IntProperty for counts, FloatProperty for continuous values)

**Resolution:** Fix syntax or property type.

---

## CHECK 19: Edge/Face Data Format

**Detection method:**
1. Find edge and face data construction (list comprehensions, explicit lists)
2. Verify edge tuples have exactly 2 elements
3. Verify face tuples have 3 or more elements
4. Verify all elements are integers, not floats

**Resolution:** Fix tuple sizes and ensure integer indices.
