# blender-syntax-modifiers: Anti-Patterns

Common mistakes when working with Blender modifier APIs. Each entry explains WHAT is wrong and WHY it fails.

---

## Anti-Pattern 1: Applying Modifiers in Edit Mode

```python
# WRONG — modifier_apply fails in Edit Mode
bpy.ops.object.mode_set(mode='EDIT')
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="MyArray")  # RuntimeError
```

**WHY:** `bpy.ops.object.modifier_apply()` requires Object Mode. In Edit Mode the mesh data is in an intermediate edit state (BMesh) and cannot accept the baked modifier result. Blender raises a `RuntimeError` with a message indicating the operator requires Object Mode. This applies to all modifier application operators.

```python
# CORRECT — switch to Object Mode before applying
if bpy.context.mode != 'OBJECT':
    bpy.ops.object.mode_set(mode='OBJECT')
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="MyArray")
```

---

## Anti-Pattern 2: Using Dict Context Overrides in Blender 4.0+

```python
# WRONG — dict overrides removed in Blender 4.0+
override = bpy.context.copy()
override['object'] = obj
override['active_object'] = obj
bpy.ops.object.modifier_apply(override, modifier="MyArray")  # TypeError
```

**WHY:** Blender 4.0 removed support for passing dictionary context overrides as the first positional argument to operators. This was deprecated in Blender 3.2 and fully removed in 4.0. In 4.0+, passing a dict raises `TypeError`. The replacement is `bpy.context.temp_override()` which was introduced in Blender 3.2.

```python
# CORRECT — use temp_override (Blender 3.2+/4.x/5.x)
with bpy.context.temp_override(object=obj, active_object=obj):
    bpy.ops.object.modifier_apply(modifier="MyArray")
```

---

## Anti-Pattern 3: Reading `obj.data.vertices` for Post-Modifier Results

```python
# WRONG — obj.data contains original mesh, NOT evaluated mesh
obj = bpy.data.objects["Wall"]
print(len(obj.data.vertices))  # Original vertex count, ignoring modifiers
for v in obj.data.vertices:
    pos = v.co  # Pre-modifier position
```

**WHY:** `obj.data` (the `Mesh` data block) stores only the original, unmodified mesh. Modifiers are evaluated at render/display time via the dependency graph. To access post-modifier geometry, you must use `obj.evaluated_get(depsgraph).to_mesh()`. Reading `obj.data.vertices` directly gives original geometry as if no modifiers exist.

```python
# CORRECT — read evaluated mesh via depsgraph
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
try:
    print(len(mesh_eval.vertices))  # Post-modifier vertex count
    for v in mesh_eval.vertices:
        pos = v.co  # Post-modifier position
finally:
    obj_eval.to_mesh_clear()  # ALWAYS clean up
```

---

## Anti-Pattern 4: Forgetting `to_mesh_clear()` After `to_mesh()`

```python
# WRONG — memory leak, temporary mesh never freed
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
verts = [(v.co.x, v.co.y, v.co.z) for v in mesh_eval.vertices]
# Missing: obj_eval.to_mesh_clear()
```

**WHY:** `to_mesh()` creates a temporary `Mesh` data block in memory. If `to_mesh_clear()` is never called, the temporary mesh persists for the duration of the session, accumulating with each call. In scripts that iterate over many objects or frames, this causes significant memory leaks and eventual performance degradation or out-of-memory errors.

```python
# CORRECT — always use try/finally to guarantee cleanup
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
try:
    verts = [(v.co.x, v.co.y, v.co.z) for v in mesh_eval.vertices]
finally:
    obj_eval.to_mesh_clear()
```

---

## Anti-Pattern 5: Assuming GN Input Identifiers Match Socket Names

```python
# WRONG — identifiers are auto-generated, NOT the display name
mod = obj.modifiers["GeoNodes"]
mod["Height"] = 3.5       # KeyError: 'Height' not found
mod["Thickness"] = 0.2    # KeyError: 'Thickness' not found
```

**WHY:** Geometry Nodes modifier inputs use auto-generated identifiers like `"Socket_0"`, `"Socket_1"`, etc., NOT the human-readable socket names displayed in the UI. The identifier is an internal string assigned when the socket is created in the node group interface. Looking up inputs by display name directly on the modifier raises `KeyError`. You must iterate `modifier.node_group.interface.items_tree` to find the identifier for a given name.

```python
# CORRECT — look up identifier via interface.items_tree
def get_gn_input_id(modifier, input_name):
    for item in modifier.node_group.interface.items_tree:
        if item.item_type == 'SOCKET' and item.in_out == 'INPUT':
            if item.name == input_name:
                return item.identifier
    raise KeyError(f"Input '{input_name}' not found")

height_id = get_gn_input_id(mod, "Height")
mod[height_id] = 3.5
```

---

## Anti-Pattern 6: Modifying the Evaluated Mesh Directly

```python
# WRONG — evaluated mesh is read-only
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
mesh_eval.vertices[0].co.z += 1.0  # Change is lost or raises error
```

**WHY:** The mesh returned by `to_mesh()` on an evaluated object is a temporary snapshot. Any modifications to it are either silently discarded when `to_mesh_clear()` is called or have no effect on the actual object. The evaluated mesh exists only for reading. To modify geometry, change the original `obj.data` or the modifier parameters and let the depsgraph re-evaluate.

```python
# CORRECT — modify the original data, not the evaluated copy
obj.data.vertices[0].co.z += 1.0  # Modifies original mesh
bpy.context.view_layer.update()    # Re-evaluate modifiers
```

---

## Anti-Pattern 7: Iterating Modifier List While Removing Modifiers

```python
# WRONG — collection changes during iteration, skips modifiers
for mod in obj.modifiers:
    obj.modifiers.remove(mod)  # RuntimeError or skips items
```

**WHY:** Removing items from a `bpy_prop_collection` while iterating over it invalidates the iterator. Blender either raises a `RuntimeError` or silently skips elements because the collection indices shift after each removal. This is a standard Python collection-mutation-during-iteration problem.

```python
# CORRECT — copy the list first, then iterate the copy
for mod in obj.modifiers[:]:
    obj.modifiers.remove(mod)

# Alternative: use clear() if available (Blender 3.x+)
obj.modifiers.clear()
```

---

## Anti-Pattern 8: Applying Modifiers Without Copying the Modifier List

```python
# WRONG — modifier stack changes as modifiers are applied
with bpy.context.temp_override(object=obj, active_object=obj):
    for mod in obj.modifiers:
        bpy.ops.object.modifier_apply(modifier=mod.name)  # Skips or errors
```

**WHY:** Applying a modifier removes it from `obj.modifiers`. Iterating over `obj.modifiers` directly while applying causes the same collection-mutation problem as removal: the loop skips modifiers or raises errors because the collection shrinks during iteration.

```python
# CORRECT — snapshot modifier names before applying
modifier_names = [mod.name for mod in obj.modifiers]
with bpy.context.temp_override(object=obj, active_object=obj):
    for name in modifier_names:
        bpy.ops.object.modifier_apply(modifier=name)
```
