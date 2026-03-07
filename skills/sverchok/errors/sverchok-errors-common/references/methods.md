# sverchok-errors-common: Error Detection Methods

Sources: https://github.com/nortikin/sverchok (v1.4.0+),
vooronderzoek-sverchok.md §12, §13

---

## Diagnostic Function: Detect Nesting Level Issues

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
def check_nesting_level(data, expected_depth, label="data"):
    """Verify data nesting matches Sverchok conventions.
    Parameters:
        data: the socket data to check
        expected_depth: int — expected nesting depth (2 for numbers, 3 for vertices)
        label: str — name for error messages
    Returns:
        bool — True if nesting is correct
    """
    current = data
    for level in range(expected_depth):
        if not isinstance(current, (list, tuple)):
            print(f"ERROR: {label} has wrong nesting at level {level}. "
                  f"Expected depth {expected_depth}, got non-list at level {level}.")
            return False
        if len(current) == 0:
            return True  # Empty is valid
        current = current[0]
    return True

# Usage in process():
# check_nesting_level(verts, 3, "vertices")  # [[( x,y,z)]] = depth 3
# check_nesting_level(numbers, 2, "numbers")  # [[1,2,3]] = depth 2
```

---

## Diagnostic Function: Detect Socket Data Mutation

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import copy

def safe_sv_get(socket, label="input"):
    """Get socket data with mutation detection.
    Wraps sv_get to track whether the original cache is accidentally mutated.
    Use during development/debugging only — adds overhead.
    """
    data = socket.sv_get(deepcopy=True)
    # Store a reference copy to detect accidental mutation
    _original = copy.deepcopy(data)

    class MutationGuard:
        def __init__(self, data, original, label):
            self.data = data
            self._original = original
            self._label = label

        def check(self):
            """Call after processing to verify no unintended mutation."""
            raw = socket.sv_get(deepcopy=False)
            if raw != self._original:
                print(f"WARNING: {self._label} socket cache was mutated!")
                return False
            return True

    return data, MutationGuard(data, _original, label)
```

---

## Diagnostic Function: Detect Missing updateNode

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
def check_node_properties(node):
    """Check if all properties on a custom node have update=updateNode.
    Parameters:
        node: SverchCustomTreeNode — the node to inspect
    Prints warnings for properties missing the updateNode callback.
    """
    from sverchok.data_structure import updateNode

    node_class = type(node)
    annotations = getattr(node_class, '__annotations__', {})

    for prop_name in annotations:
        # Get the property definition from the RNA
        rna_prop = node.bl_rna.properties.get(prop_name)
        if rna_prop is None:
            continue
        # Check if it's a Blender property with an update callback
        if hasattr(rna_prop, 'update') and rna_prop.update is None:
            print(f"WARNING: Property '{prop_name}' on {node.bl_idname} "
                  f"has no update callback. Add update=updateNode.")
```

---

## Diagnostic Function: Detect Sockets Created in process()

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
def check_socket_count_stability(node, expected_inputs, expected_outputs):
    """Verify socket counts match expectations after process().
    Call before and after process to detect socket creation in process().
    Parameters:
        node: SverchCustomTreeNode — the node to check
        expected_inputs: int — expected input socket count
        expected_outputs: int — expected output socket count
    """
    actual_in = len(node.inputs)
    actual_out = len(node.outputs)

    if actual_in != expected_inputs:
        print(f"ERROR: {node.bl_idname} has {actual_in} inputs, "
              f"expected {expected_inputs}. Sockets may be created in process().")
    if actual_out != expected_outputs:
        print(f"ERROR: {node.bl_idname} has {actual_out} outputs, "
              f"expected {expected_outputs}. Sockets may be created in process().")
```

---

## Diagnostic Function: Full Node Tree Health Check

```python
# Blender 4.0+/5.x with Sverchok v1.4.0+
import bpy

def diagnose_sverchok_tree(tree_name):
    """Run comprehensive diagnostics on a Sverchok node tree.
    Checks for common error patterns documented in this skill.
    Parameters:
        tree_name: str — name of the Sverchok node tree
    """
    tree = bpy.data.node_groups.get(tree_name)
    if tree is None or tree.bl_idname != 'SverchCustomTreeType':
        print(f"ERROR: '{tree_name}' is not a Sverchok tree")
        return

    print(f"=== Diagnostics: {tree.name} ===")
    print(f"  sv_process: {tree.sv_process}")

    if not tree.sv_process:
        print("  WARNING: Tree processing is DISABLED. No nodes will execute.")

    for node in tree.nodes:
        # Check for errors
        if node.US_error:
            print(f"  ERROR in '{node.name}': {node.US_error}")

        # Check execution time for performance issues
        if node.US_time and node.US_time > 1.0:
            print(f"  SLOW: '{node.name}' took {node.US_time:.2f}s")

        # Check for unconnected outputs with expensive nodes
        has_linked_output = any(out.is_linked for out in node.outputs)
        if not has_linked_output and len(node.outputs) > 0:
            print(f"  NOTE: '{node.name}' has no connected outputs "
                  f"(consider if computation is wasted)")

    print(f"=== End diagnostics ===")
```

---

## Key API Methods for Error Handling

### sv_get() — Socket Data Retrieval

```python
socket.sv_get(default=None, deepcopy=True)
# Parameters:
#   default: value returned if socket is unconnected and has no data
#   deepcopy: bool — True returns independent copy (safe to mutate)
#                    False returns cached reference (read-only)
# Raises:
#   SvNoDataError — if no data and no default provided
```

### updateNode — Property Change Callback

```python
from sverchok.data_structure import updateNode
# Signature: updateNode(self, context)
# Use as: update=updateNode on all bpy.props in Sverchok nodes
# Effect: Dispatches PropertyEvent -> marks node outdated -> triggers process()
```

### match_long_repeat — List Length Matching

```python
from sverchok.data_structure import match_long_repeat
# Signature: match_long_repeat(list_of_lists) -> list[list]
# Repeats the last element of shorter lists to match the longest
# Example: match_long_repeat([[1,2,3], [10]]) -> [[1,2,3], [10,10,10]]
```

### SvIfcStore.purge() — IFC Data Reset

```python
# Called automatically before every full IfcSverchok tree update
# Clears all IFC entity data from the store
# STEP IDs are invalidated — only GUIDs persist
```
