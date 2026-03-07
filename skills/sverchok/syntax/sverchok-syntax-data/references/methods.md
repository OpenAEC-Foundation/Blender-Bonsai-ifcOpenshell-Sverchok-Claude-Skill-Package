# sverchok-syntax-data: API Reference

> Complete method signatures and implementation details for Sverchok data nesting and list matching.
> All signatures verified against Sverchok v1.4.0+ source code. Blender 4.0+/5.x.

---

## 1. List Matching Functions (data_structure.py)

### match_long_repeat(lsts)

```python
def match_long_repeat(lsts):
    """Extend shorter lists by repeating their last element to match the longest list.

    Args:
        lsts: List of lists to match.

    Returns:
        List of lists, all with equal length.

    Example:
        match_long_repeat([[1,2,3,4,5], [10,11]])
        # -> [[1,2,3,4,5], [10,11,11,11,11]]

    Raises:
        TypeError: If any input is not a list/tuple (atomic object).

    Notes:
        - Does NOT modify input lists; produces non-deep copies.
        - Uses repeat_last() internally as an infinite iterator.
        - This is the DEFAULT matching mode in Sverchok.
    """
```

### match_long_cycle(lsts)

```python
def match_long_cycle(lsts):
    """Extend shorter lists by cycling their elements to match the longest list.

    Args:
        lsts: List of lists to match.

    Returns:
        List of lists, all with equal length.

    Example:
        match_long_cycle([[1,2,3,4,5], [10,11]])
        # -> [[1,2,3,4,5], [10,11,10,11,10]]

    Notes:
        - Uses itertools.cycle() internally.
    """
```

### match_short(lsts)

```python
def match_short(lsts):
    """Truncate all lists to the length of the shortest list.

    Args:
        lsts: List of lists to match.

    Returns:
        List of lists, all truncated to shortest length.

    Example:
        match_short([[1,2,3,4,5], [10,11]])
        # -> [[1,2], [10,11]]
    """
```

### match_cross(lsts)

```python
def match_cross(lsts):
    """Cross-reference (Cartesian product) of all input lists.

    Args:
        lsts: List of lists to cross-reference.

    Returns:
        List of lists with all combinations (fast cycle of long).

    Example:
        match_cross([[1,2], [5,6,7]])
        # -> [[1,1,1,2,2,2], [5,6,7,5,6,7]]

    Notes:
        - Uses itertools.product() internally.
        - Output size = product of all input lengths.
        - WARNING: Causes combinatorial explosion with large inputs.
    """
```

### match_cross2(lsts)

```python
def match_cross2(lsts):
    """Cross-reference variant (Cartesian product, fast cycle of short).

    Args:
        lsts: List of lists to cross-reference.

    Returns:
        List of lists with all combinations (reversed nesting order).

    Example:
        match_cross2([[1,2], [5,6,7]])
        # -> [[1,2,1,2,1,2], [5,5,6,6,7,7]]

    Notes:
        - Uses reversed(product(*reversed(lsts))) internally.
    """
```

### list_match_func

```python
list_match_func = {
    "SHORT":  match_short,
    "CYCLE":  match_long_cycle,
    "REPEAT": match_long_repeat,
    "XREF":   match_cross,
    "XREF2":  match_cross2,
}
```

### list_match_modes (EnumProperty items)

```python
list_match_modes = [
    ("SHORT",  "Short",       "Match shortest List",                    1),
    ("CYCLE",  "Cycle",       "Match longest List by cycling",          2),
    ("REPEAT", "Repeat Last", "Match longest List by repeating last",   3),
    ("XREF",   "X-Ref",      "Cross reference (fast cycle of long)",   4),
    ("XREF2",  "X-Ref 2",    "Cross reference (fast cycle of short)",  5),
]
```

### numpy_list_match_modes

```python
# Only the first 3 modes (no cross-reference for NumPy)
numpy_list_match_modes = list_match_modes[:3]  # SHORT, CYCLE, REPEAT
```

### zip_long_repeat(*lists)

```python
def zip_long_repeat(*lists):
    """Convenience wrapper: match_long_repeat + zip for easy iteration.

    Args:
        *lists: Multiple lists to match and iterate.

    Returns:
        zip iterator over matched lists.

    Example:
        for v, s in zip_long_repeat(verts, scales):
            process(v, s)

    Notes:
        - Equivalent to: zip(*match_long_repeat(lists))
    """
```

---

## 2. List Extension Helpers (data_structure.py)

### fullList(l, count)

```python
def fullList(l, count):
    """Extend list in-place by repeating the last element until len(l) >= count.

    Args:
        l: List to extend (MODIFIED IN-PLACE).
        count: Target length.

    Returns:
        None (modifies l in-place).

    Example:
        data = [1, 2, 3]
        fullList(data, 6)
        # data is now [1, 2, 3, 3, 3, 3]

    Notes:
        - No-op if len(l) == count.
        - Does NOT deep-copy the repeated element.
    """
```

### fullList_deep_copy(l, count)

```python
def fullList_deep_copy(l, count):
    """Same as fullList but deep-copies the last element. Safe for mutable objects.

    Args:
        l: List to extend (MODIFIED IN-PLACE).
        count: Target length.

    Returns:
        None (modifies l in-place).

    Notes:
        - Uses copy.deepcopy(l[-1]) for each repeated element.
        - Use this when l contains mutable objects (lists, dicts, etc.).
    """
```

### repeat_last(lst)

```python
def repeat_last(lst):
    """Infinite iterator: yields all elements of lst, then repeats last element forever.

    Args:
        lst: Source list.

    Yields:
        Elements from lst, then lst[-1] indefinitely.

    Example:
        gen = repeat_last([1, 2, 3])
        [next(gen) for _ in range(6)]  # [1, 2, 3, 3, 3, 3]

    Notes:
        - ALWAYS use with a terminating consumer (zip, islice).
        - Handles numpy arrays (uses len() check).
        - Implementation: chain(lst, cycle([lst[-1]]))
    """
```

### repeat_last_for_length(lst, count, deepcopy=False)

```python
def repeat_last_for_length(lst, count, deepcopy=False):
    """Return a new list of exactly `count` elements by repeating the last item.

    Args:
        lst: Source list. None and [] return as-is.
        count: Desired output length.
        deepcopy: If True, deep-copy the repeated element.

    Returns:
        New list of length `count`, or original if lst is empty/None.

    Examples:
        repeat_last_for_length(None, 5)     # None
        repeat_last_for_length([], 5)       # []
        repeat_last_for_length([1,2], 4)    # [1, 2, 2, 2]
        repeat_last_for_length([1,2,3], 2)  # [1, 2]  (truncates if longer)
    """
```

### fixed_iter(data, iter_number, fill_value=0)

```python
def fixed_iter(data, iter_number, fill_value=0):
    """Iterator yielding exactly iter_number elements from data, padding with last value.

    Args:
        data: Source iterable.
        iter_number: Exact number of elements to yield.
        fill_value: Default value if data is empty.

    Yields:
        Elements from data, then cycles last value until iter_number reached.

    Notes:
        - Used internally by match_sockets() for object-level padding.
        - If data is empty, yields fill_value repeated iter_number times.
    """
```

---

## 3. Nesting Level Detection (data_structure.py)

### SIMPLE_DATA_TYPES

```python
from numpy import float64, int32, int64
from mathutils import Matrix

SIMPLE_DATA_TYPES = (float, int, float64, int32, int64, str, Matrix)
```

These types are treated as level-0 atoms (leaf values) by all nesting-level functions.

### get_data_nesting_level(data, data_types=SIMPLE_DATA_TYPES, search_first_data=False)

```python
def get_data_nesting_level(data, data_types=SIMPLE_DATA_TYPES, search_first_data=False):
    """Detect nesting depth of data by locating the first actual data element.

    Args:
        data: Any nested structure.
        data_types: Types considered as leaf-level data (level 0).
        search_first_data: If True, searches all branches for actual data
                          (not just data[0]). Useful for sparse structures.

    Returns:
        int: Nesting level.

    Examples:
        get_data_nesting_level(1)                    # 0
        get_data_nesting_level([])                   # 1
        get_data_nesting_level([1])                  # 1
        get_data_nesting_level([[(1,2,3)]])          # 3
        get_data_nesting_level([[1, 2], [3, 4]])     # 2

    Raises:
        TypeError: If None is encountered at any nesting level.

    Notes:
        - Only inspects first element by default (search_first_data=False).
        - Does NOT support heterogeneous nesting in a single list.
        - Empty lists return level 1.
        - Unknown types (not in data_types, not list/tuple/ndarray) return level 0.
    """
```

### describe_data_shape(data)

```python
def describe_data_shape(data):
    """Human-readable description of data shape for debugging.

    Args:
        data: Any nested data structure.

    Returns:
        str: Description like "Level 3: list [1] of list [4] of tuple [3] of float"

    Examples:
        describe_data_shape(None)              # "Level 0: NoneType"
        describe_data_shape(1)                 # "Level 0: int"
        describe_data_shape([])                # "Level 1: list [0]"
        describe_data_shape([[(1,2,3)]])       # "Level 3: list [1] of list [1] of tuple [3] of int"

    Notes:
        - Only inspects first element (assumes homogeneous data).
        - Useful for debugging nesting issues in process() methods.
    """
```

### get_max_data_nesting_level(data, data_types=SIMPLE_DATA_TYPES)

```python
def get_max_data_nesting_level(data, data_types=SIMPLE_DATA_TYPES):
    """Return maximum nesting depth across ALL branches (not just first element).

    Args:
        data: Any nested structure.
        data_types: Types considered as leaf data.

    Returns:
        int: Maximum nesting level found in any branch.

    Notes:
        - More thorough but slower than get_data_nesting_level().
        - Inspects every element at every level.
    """
```

### levels_of_list_or_np(lst)

```python
def levels_of_list_or_np(lst):
    """Calculate containment nesting level (integer) for lists and numpy arrays.

    Args:
        lst: A list, tuple, or numpy array.

    Returns:
        int: Nesting level. Returns 0 for empty lists.

    Notes:
        - For numpy arrays, uses len(n.shape) instead of recursion.
        - Only inspects first element of each list (like get_data_nesting_level).
        - Used internally by process_matched() and DataWalker.
        - WARNING: Does NOT count tuples-of-numbers as a nesting level.
          Use get_data_nesting_level() for user-facing nesting detection.
    """
```

---

## 4. Nesting Level Adjustment (data_structure.py)

### ensure_nesting_level(data, target_level, data_types=SIMPLE_DATA_TYPES, input_name=None)

```python
def ensure_nesting_level(data, target_level, data_types=SIMPLE_DATA_TYPES, input_name=None):
    """Wrap data in [] to reach target nesting level. RAISES if already too deep.

    Args:
        data: Input data of any nesting level.
        target_level: Required nesting level.
        data_types: Types treated as level-0 atoms.
        input_name: Socket name for error messages (optional).

    Returns:
        Data wrapped to exactly target_level.

    Raises:
        TypeError: If current nesting > target_level.

    Examples:
        ensure_nesting_level(17, 0)        # 17
        ensure_nesting_level(17, 1)        # [17]
        ensure_nesting_level([17], 1)      # [17]
        ensure_nesting_level([17], 2)      # [[17]]
        ensure_nesting_level([(1,2,3)], 3) # [[(1,2,3)]]
        ensure_nesting_level([[[17]]], 1)  # TypeError!
    """
```

### ensure_min_nesting(data, target_level, data_types=SIMPLE_DATA_TYPES, input_name=None)

```python
def ensure_min_nesting(data, target_level, data_types=SIMPLE_DATA_TYPES, input_name=None):
    """Wrap data in [] to reach MINIMUM target nesting. Does NOT raise if deeper.

    Args:
        data: Input data of any nesting level.
        target_level: Minimum required nesting level.
        data_types: Types treated as level-0 atoms.
        input_name: Socket name for error messages (optional).

    Returns:
        Data wrapped to at least target_level. If already deeper, returns as-is.

    Examples:
        ensure_min_nesting(17, 0)          # 17
        ensure_min_nesting(17, 1)          # [17]
        ensure_min_nesting([17], 2)        # [[17]]
        ensure_min_nesting([[[17]]], 1)    # [[[17]]]  (no change, already deeper)
    """
```

### flatten_data(data, target_level=1, data_types=SIMPLE_DATA_TYPES)

```python
def flatten_data(data, target_level=1, data_types=SIMPLE_DATA_TYPES):
    """Reduce nesting to target_level by concatenating sublists.

    Args:
        data: Nested data structure.
        target_level: Desired output nesting level.
        data_types: Types treated as level-0 atoms.

    Returns:
        Data with nesting reduced to target_level.

    Raises:
        TypeError: If current nesting < target_level (cannot flatten further).

    Notes:
        - Recursively extends result lists with sub-items.
        - Used by socket preprocessing when use_flatten is True.
    """
```

### graft_data(data, item_level=1, wrap_level=1, data_types=SIMPLE_DATA_TYPES)

```python
def graft_data(data, item_level=1, wrap_level=1, data_types=SIMPLE_DATA_TYPES):
    """Wrap each nested item at item_level with wrap_level additional [].

    Args:
        data: Nested data structure.
        item_level: Nesting level at which to apply wrapping.
        wrap_level: How many layers of [] to add.
        data_types: Types treated as level-0 atoms.

    Returns:
        Data with extra wrapping at specified depth.

    Notes:
        - Used by socket preprocessing when use_graft is True.
    """
```

### map_recursive(fn, data, data_types=SIMPLE_DATA_TYPES)

```python
def map_recursive(fn, data, data_types=SIMPLE_DATA_TYPES):
    """Apply fn to each leaf item, preserving nesting structure.

    Args:
        fn: Callable to apply to each leaf value.
        data: Nested data structure.
        data_types: Types treated as leaf-level atoms.

    Returns:
        Same nesting structure with fn applied to every leaf.

    Notes:
        - Similar to recurse_fx but uses SIMPLE_DATA_TYPES for leaf detection.
    """
```

### map_at_level(function, data, item_level=0, data_types=SIMPLE_DATA_TYPES)

```python
def map_at_level(function, data, item_level=0, data_types=SIMPLE_DATA_TYPES):
    """Apply function to sub-lists at specified nesting level.

    Args:
        function: Callable to apply.
        data: Nested data structure.
        item_level: The nesting level at which to apply the function.
        data_types: Types treated as level-0 atoms.

    Returns:
        Transformed data with function applied at the specified level.

    Notes:
        - Result nesting is SIMPLER than input (item_level levels eliminated).
    """
```

---

## 5. NumPy Matching Functions (data_structure.py)

### numpy_list_match_func

```python
numpy_list_match_func = {
    "SHORT":  numpy_match_short,
    "CYCLE":  numpy_match_long_cycle,
    "REPEAT": numpy_match_long_repeat,
}
```

**Note:** XREF and XREF2 are NOT available for NumPy arrays.

### numpy_match_long_repeat(list_of_arrays)

```python
def numpy_match_long_repeat(list_of_arrays):
    """Match numpy array lengths by repeating the last row.

    Uses np.repeat on the last row and np.concatenate.
    """
```

### numpy_match_long_cycle(list_of_arrays)

```python
def numpy_match_long_cycle(list_of_arrays):
    """Match numpy array lengths by cycling (np.tile + concatenate)."""
```

### numpy_match_short(list_of_arrays)

```python
def numpy_match_short(list_of_arrays):
    """Match numpy array lengths by slicing to the shortest."""
```

### numpy_full_list(array, desired_length)

```python
def numpy_full_list(array, desired_length):
    """Extend numpy array to desired_length by repeating last row.

    Converts non-ndarray input to ndarray automatically.
    """
```

### numpy_full_list_cycle(array, desired_length)

```python
def numpy_full_list_cycle(array, desired_length):
    """Extend numpy array to desired_length by cycling (np.tile)."""
```

---

## 6. SvRecursiveNode Mixin (utils/nodes_mixins/recursive_nodes.py)

### Class Definition

```python
class SvRecursiveNode():
    """Mixin for automatic node vectorization. Handles data matching and
    recursive descent through nesting levels automatically.

    Provides:
        - list_match: EnumProperty with 3 matching modes (default: REPEAT)
        - process(): Automatic input reading, matching, and output writing
        - pre_setup(): Override hook called before input data is read
        - process_data(params): Override to implement node logic

    Socket properties to configure in sv_init():
        s.is_mandatory:    bool (default False) — skip if unconnected
        s.nesting_level:   int  (default 2; 3 for SvVerticesSocket)
        s.default_mode:    str  (default 'EMPTY_LIST')
        s.pre_processing:  str  (default 'NONE')
    """
```

### Properties

```python
# EnumProperty for matching mode selection
list_match: EnumProperty(
    name="List Match",
    description="Behavior on different list lengths",
    items=numpy_list_match_modes,  # SHORT, CYCLE, REPEAT only — NO XREF/XREF2
    default="REPEAT",
    update=updateNode
)

# BMesh support (set in node class)
build_bmesh = False              # Set True to auto-build bmesh from verts/edges/faces
bmesh_inputs = [0, 1, 2]        # Input socket indices for verts, edges, faces
```

**Important:** The `list_match` EnumProperty uses `numpy_list_match_modes` which only includes SHORT, CYCLE, and REPEAT. XREF and XREF2 are NOT available through the SvRecursiveNode mixin.

### process() Method

```python
def process(self):
    """Main execution method. Called by the update system.

    Workflow:
        1. Calls self.pre_setup() for dynamic socket configuration
        2. Checks all is_mandatory sockets are connected; returns if not
        3. Checks at least one output is connected; returns if not
        4. For each input socket:
           a. Reads nesting_level from socket
           b. Gets default from DEFAULT_TYPES[s.default_mode]
           c. Reads data with sv_get(deepcopy=False, default=default)
           d. If pre_processing == 'ONE_ITEM': applies one_item_list(ensure_min_nesting(data, 2))
           e. Else: applies ensure_min_nesting(data, s.nesting_level)
        5. If build_bmesh: converts vertex/edge/face inputs to bmesh objects
        6. Calls process_matched(params, self.process_data, self.list_match,
                                 input_nesting, len(self.outputs))
        7. Sets output socket data via sv_set()
    """
```

### pre_setup() Method

```python
def pre_setup(self):
    """Override this to dynamically configure socket properties before data is read.

    Called at the start of process(), before any sv_get() calls.
    Use to change nesting_level, default_mode, etc. based on node properties.

    Example:
        def pre_setup(self):
            if self.mode == 'FLAT':
                self.inputs[0].nesting_level = 2
            else:
                self.inputs[0].nesting_level = 3
    """
```

### process_data(params) Method

```python
def process_data(self, params):
    """Override this to implement node logic.

    Args:
        params: List of matched input data, one entry per input socket.
                Data is ALREADY matched to equal object counts and at the
                correct nesting level for a single "object".

    Returns:
        If single output: return the result directly (not wrapped in list)
        If multiple outputs: return tuple/list of results, one per output socket

    Example (single output):
        def process_data(self, params):
            verts, scale = params
            return [(v[0]*scale, v[1]*scale, v[2]*scale) for v in verts]

    Example (multiple outputs):
        def process_data(self, params):
            verts, = params
            centers = [sum(v)/len(v) for v in zip(*verts)]
            count = len(verts)
            return [centers], [count]
    """
```

### DEFAULT_TYPES

```python
DEFAULT_TYPES = {
    'NONE':       ...,          # Ellipsis — no default, socket is optional
    'EMPTY_LIST': [[]],         # Empty nested list
    'MATRIX':     [Matrix()],   # Identity matrix
    'MASK':       [[True]],     # Boolean mask (all selected)
}
```

### one_item_list(data) (helper)

```python
def one_item_list(data):
    """Collapse data to one value per object for pre_processing='ONE_ITEM'.

    Logic:
        - If data has 1 element: return data[0]
        - If any sub-list has length > 1: return data as-is (complex input)
        - Otherwise: unwrap single-element sub-lists -> [d[0] for d in data]

    Examples:
        one_item_list([[5]])         # [5]       -> single value
        one_item_list([[1], [2]])    # [1, 2]    -> one per object
        one_item_list([[1, 2]])      # [1, 2]    -> unwrapped
        one_item_list([[1, 2, 3]])   # [1, 2, 3] -> returned as-is (>1 elements)
    """
```

---

## 7. Vectorize Decorator (utils/vectorize.py)

### vectorize(func=None, *, match_mode="REPEAT")

```python
def vectorize(func=None, *, match_mode="REPEAT"):
    """Decorator that transforms a function operating on single values into
    one that operates on nested lists of arbitrary depth.

    Args:
        func: The function to vectorize. Must use keyword-only arguments.
        match_mode: Matching mode string ("REPEAT", "CYCLE", "SHORT", "XREF", "XREF2")

    Returns:
        Wrapped function that handles nested data automatically.

    Requirements:
        - Decorated function MUST use keyword-only arguments (no positional args).
        - Type annotations control nesting level detection:
            float, int, bool, str, Matrix  -> level 0
            list, tuple                     -> level 1
            List[float]                     -> level 1
            List[Tuple[float, float, float]] -> level 2
            List[List[float]]               -> level 2
        - Return annotation Tuple[list, list] signals multiple output lists.
        - Return annotation without Tuple signals single output.

    Raises:
        TypeError: If positional arguments are passed to the wrapped function.

    Notes:
        - Can be used as @vectorize or @vectorize(match_mode="CYCLE").
        - None or [] inputs are wrapped in EmptyDataWalker (produce empty output).
        - Short-circuit: if all inputs are at VALUE level, calls function directly.
    """
```

### Sverchok Type Aliases (for vectorize annotations)

```python
# Commonly used type aliases in Sverchok source:
SvVerts = List[Tuple[float, float, float]]    # Nesting level 2 (one object's verts)
SvEdges = List[Tuple[int, int]]               # Nesting level 2 (one object's edges)
SvPolys = List[List[int]]                     # Nesting level 2 (one object's faces)
```

### match_sockets(*sockets_data)

```python
def match_sockets(*sockets_data):
    """Generator that iterates over object-level data, matching within each object.

    Args:
        *sockets_data: Multiple socket data lists (each at object level).

    Yields:
        List of matched data for one object iteration.

    Example:
        data1 = [[1,2,3]]
        data2 = [[4,5], [6,7]]
        data3 = [[8]]
        for d1, d2, d3 in match_sockets(data1, data2, data3):
            # iter 1: d1=[1,2,3], d2=[4,5,5], d3=[8]
            # iter 2: d1=[1,2,3], d2=[6,7,7], d3=[8]

    Notes:
        - Object count matched by fixed_iter (repeats last object).
        - Within each object, data length matched by fixed_iter (repeats last item).
        - NumPy arrays matched via numpy_full_list.
        - Items with length 1 are NOT padded (broadcast as single value).
    """
```

### DataWalker Class

```python
class DataWalker:
    """Tree walker for nested data traversal used by vectorize().

    Match modes (class constants):
        SHORT, CYCLE, REPEAT, XREF, XREF2

    Node types (class constants):
        VALUE   - Current position is at a leaf value
        END     - Current sub-tree is exhausted
        SUB_TREE - Current position contains nested data

    Constructor:
        DataWalker(data, output_nesting=0, mode=REPEAT, data_name=None)

        Args:
            data: The nested data to walk.
            output_nesting: Nesting level at which data is treated as a leaf VALUE.
            mode: Match mode for step_down_matching.
            data_name: Optional name for debug representation.

    Key methods:
        what_is_next()                       -> VALUE | END | SUB_TREE
        step_down_matching(match_len, mode)  -> descend into sub-tree
        step_up()                            -> ascend after END
        pop_next_value()                     -> retrieve leaf value

    Properties:
        next_values_number  -> int: count of immediate children
        is_exhausted        -> bool: True when stack is empty

    Notes:
        - Uses a stack-based traversal with EXIT_VALUE sentinel.
        - what_is_next() is cached for performance (keyed by id(stack[-1])).
        - Leaf detection compares levels_of_list_or_np(data) against output_nesting.
    """
```

### walk_data(walkers, out_list)

```python
def walk_data(walkers: List[DataWalker], out_list: List[list]):
    """Drive multiple DataWalkers in sync, yielding matched leaf values.

    Args:
        walkers: List of DataWalker instances (one per input).
        out_list: List of empty lists for output collection.

    Yields:
        Tuple of (matched_values_list, output_containers_list) at each leaf.

    Algorithm:
        1. Initial step_down on all walkers (extra wrapping layer)
        2. Loop while any walker is not exhausted:
           - If all at VALUE: yield values and output containers
           - If any at END: step_up all walkers and output tree generators
           - If any at SUB_TREE: step_down all walkers (with max-length matching)
    """
```

### EmptyDataWalker

```python
class EmptyDataWalker:
    """Null-object walker for None or [] inputs. All operations are no-ops.

    Used when an input to vectorize() is None or empty.
    what_is_next() always returns VALUE, is_exhausted is always True.
    pop_next_value() returns the original empty data.
    """
```

### devectorize(func=None, *, match_mode="REPEAT")

```python
def devectorize(func=None, *, match_mode="REPEAT"):
    """Inverse of vectorize: flattens nested data before passing to function.

    Creates DataWalker with output_nesting = annotation_level - 1.
    Collects all walked data into flat structures, then calls function once.
    """
```

---

## 8. Recursive Processing (utils/sv_itertools.py)

### process_matched(params, main_func, matching_mode, input_nesting, outputs_num)

```python
def process_matched(params, main_func, matching_mode, input_nesting, outputs_num):
    """Recursively descend through nesting levels, matching and processing at each level.

    This is the core engine used by SvRecursiveNode.process().

    Args:
        params: List of input data lists.
        main_func: Function to call at the target nesting level.
                   Called as main_func(matching_f(params)).
        matching_mode: String key for list_match_func ("REPEAT", "CYCLE", etc.)
        input_nesting: List of int, desired nesting level per parameter.
        outputs_num: Number of output lists (int).

    Returns:
        If outputs_num == 1: single result list.
        If outputs_num > 1: list of result lists.

    Algorithm:
        1. Measure current nesting of each param (levels_of_list_or_np).
        2. Compare to desired input_nesting.
        3. If any param has nesting > desired:
           a. Wrap under-nested params in [].
           b. Apply matching function to equalize object counts.
           c. Recurse for each matched group.
        4. If all params at desired nesting: call main_func(matched_params).
    """
```

### recurse_fx(l, f)

```python
def recurse_fx(l, f):
    """Apply function f to every leaf element in nested structure l.

    Args:
        l: Nested lists/tuples or a leaf value.
        f: Function to apply to each leaf.

    Returns:
        Same structure as l, with f applied to every leaf.

    Example:
        recurse_fx([[1, 2], [3, 4]], lambda x: x * 2)
        # [[2, 4], [6, 8]]

    Notes:
        - Leaves are any non-list, non-tuple values.
        - Preserves nesting structure exactly.
    """
```

### recurse_fxy(l1, l2, f)

```python
def recurse_fxy(l1, l2, f):
    """Binary recursive application of f to paired leaf elements.

    Args:
        l1, l2: Nested structures or leaf values.
        f: Binary function f(x, y) to apply at leaves.

    Returns:
        Matched structure with f applied to paired leaves.

    Matching behavior:
        - Both lists: zip_longest with last-element fill (REPEAT-last).
        - One list, one scalar: broadcast scalar to each element.
        - Both scalars: apply f directly.

    Example:
        recurse_fxy([1, 2, 3], [10, 20], lambda x, y: x + y)
        # [11, 22, 33]  (20 repeated for element 3)
    """
```

### recurse_f_level_control(params, constant, main_func, matching_f, desired_levels, concatenate="APPEND")

```python
def recurse_f_level_control(params, constant, main_func, matching_f,
                            desired_levels, concatenate="APPEND"):
    """Level-controlled recursive processing with constant parameters.

    Args:
        params: List of input parameters (will be spread).
        constant: Constant parameter (NOT spread across recursion).
        main_func: Function to call at target level: main_func(params, constant, matching_f).
        matching_f: Matching function (e.g., match_long_repeat).
        desired_levels: List of int, one desired level per param.
        concatenate: "APPEND" or "EXTEND" — how to collect results.

    Returns:
        Collected results from all recursive calls.

    Notes:
        - Unlike process_matched, this passes constant and matching_f to main_func.
        - Used by nodes that need a non-varying parameter across recursion levels.
    """
```

### recurse_f_multipar(params, f, matching_f)

```python
def recurse_f_multipar(params, f, matching_f):
    """Multi-parameter recursive application with matching.

    Args:
        params: List of parameters (spread using matching_f).
        f: Function applied to leaf-level param tuples: f(params).
        matching_f: Matching function for list equalization.

    Returns:
        Recursively matched and processed results.

    Notes:
        - Generalizes recurse_fxy to N parameters.
        - Scalar params are auto-wrapped in [].
    """
```

### recurse_f_multipar_const(params, const, f, matching_f)

```python
def recurse_f_multipar_const(params, const, f, matching_f):
    """Like recurse_f_multipar but with a constant parameter that is not spread.

    Args:
        params: Parameters to spread recursively.
        const: Constant passed through unchanged.
        f: Function applied as f(params, const) at leaf level.
        matching_f: Matching function.
    """
```

### append_result(result, local_result, one_output) / extend_result(...)

```python
def append_result(result, local_result, one_output):
    """Helper: append sub-results to result list(s).
    If one_output: result.append(local_result)
    If multi-output: result[i].append(local_result[i]) for each output."""

def extend_result(result, local_result, one_output):
    """Same as append_result but uses extend instead of append (flattens one level)."""
```
