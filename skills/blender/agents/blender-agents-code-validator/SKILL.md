---
name: blender-agents-code-validator
description: >
  Use when reviewing, validating, or auditing Blender Python code for correctness. Runs
  systematic checks for deprecated API usage, context errors, version compatibility issues,
  threading violations, data reference invalidation, incorrect operator calls, and addon
  structure compliance. Prevents shipping code with silent version-dependent failures.
  Keywords: code review, validation, audit, deprecated API, context error, version
  compatibility, threading, addon structure, code quality, Blender Python.
license: MIT
compatibility: "Designed for Claude Code. Requires Python 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
dependencies:
  - blender-core-api
  - blender-core-versions
  - blender-core-runtime
  - blender-errors-context
  - blender-errors-data
  - blender-errors-version
---

# Blender Code Validator Agent

Systematic validation checklist for reviewing Blender Python code. Run this checklist against any Blender Python code to identify errors, deprecations, and anti-patterns.

## Quick Reference — When to Activate

Activate this validator when:
- Reviewing, auditing, or validating Blender Python scripts
- Checking addon/extension code before distribution
- Migrating code between Blender versions (3.x → 4.x → 5.x)
- Investigating runtime crashes, context errors, or stale references
- Generating new Blender Python code (run validator on output)

## Validation Checklist

Run each check in order. Each check has a severity level:
- **BLOCKER**: Code will crash, produce data corruption, or cause undefined behavior
- **WARNING**: Code has a bug, performance issue, or deprecated usage that will break in a future version
- **INFO**: Code works but does not follow best practices

---

### CHECK 1: Determine Target Version

**Severity if missing: WARNING**

Every Blender Python file MUST declare or imply a target Blender version. Without this, validation cannot determine which APIs are valid.

**Detection patterns:**
1. Look for `bpy.app.version` checks
2. Look for `bl_info["blender"]` tuple in addon code
3. Look for `blender_manifest.toml` in extension code (implies 4.2+)
4. If no version indicator exists → flag WARNING: "No target Blender version declared"

**Decision tree:**

```
Has bl_info with "blender" key? → Target = bl_info["blender"] tuple
Has blender_manifest.toml?      → Target = 4.2+
Has bpy.app.version checks?     → Target = checked range
None of the above?               → WARNING: Undeclared target version
```

---

### CHECK 2: Context Safety

**Severity: BLOCKER**

Scan for context-related errors — the #1 source of Blender Python crashes.

#### 2.1 Restricted Context Access

Flag BLOCKER if code calls `bpy.ops.*` or modifies `bpy.data.*` inside:
- `draw()` method of any Panel, Header, Menu, UIList, or Gizmo
- Application handler callbacks (frame_change_pre/post, depsgraph_update_post, load_post, render_*)
- Timer callbacks registered via `bpy.app.timers.register()`
- Draw callbacks registered via `SpaceView3D.draw_handler_add()`
- Any background thread (threading.Thread, concurrent.futures)

**Detection**: Identify all function definitions. Check parent class. If parent inherits from `bpy.types.Panel/Header/Menu/UIList/Gizmo`, flag any `bpy.ops` call or data mutation inside `draw()`.

#### 2.2 Operator poll() Not Checked

Flag WARNING if code calls `bpy.ops.*` without verifying prerequisites:
- `bpy.ops.object.mode_set(mode='EDIT')` without checking `context.active_object is not None`
- `bpy.ops.mesh.*` without verifying `context.mode == 'EDIT_MESH'`
- `bpy.ops.object.modifier_apply()` without checking object mode and modifier existence
- Any `bpy.ops.view3d.*` without verifying area type is `'VIEW_3D'`

#### 2.3 Context Override Errors (Version-Critical)

Flag BLOCKER if code uses dict-style context overrides on Blender 4.0+:
```python
# BLOCKER on 4.0+: Dict overrides removed
override = bpy.context.copy()
bpy.ops.something(override, ...)
```

Flag WARNING if code uses dict overrides without version gating (no `bpy.app.version` check).

**Version matrix:**
| Blender Version | Dict Override | temp_override |
|----------------|---------------|---------------|
| 3.0 – 3.1      | YES           | NO            |
| 3.2 – 3.6      | YES (deprecated) | YES        |
| 4.0+            | REMOVED       | YES           |

---

### CHECK 3: API Correctness

**Severity: BLOCKER for removed APIs, WARNING for deprecated APIs**

#### 3.1 Removed API Usage

Scan for APIs removed in each version. Flag BLOCKER if used without version guard:

**Removed in 4.0:**
- `mesh.calc_normals()` → normals are auto-calculated
- `bone.layers` → use `bone.collections`
- `armature.bone_groups` → removed
- `obj.face_maps` → use integer face attributes
- `MeshEdge.bevel_weight` → use `mesh.attributes` with float layer
- `MeshEdge.crease` → use `mesh.attributes` with float layer
- `NodeTree.inputs.new()` / `NodeTree.outputs.new()` → use `NodeTree.interface.new_socket()`
- Dict-style context overrides → use `context.temp_override()`

**Removed in 5.0:**
- `bgl` module → use `gpu` module
- `del obj["custom_prop"]` for RNA properties → use `obj.property_unset("prop")`
- `scene["cycles"]` dict access → use `scene.cycles` attribute access
- `image.bindcode` → use `gpu.texture.from_image()`

**Removed in 5.1:**
- `Sequence.frame_final_start` → use `Sequence.frame_start`
- `Sequence.frame_final_end` → use `Sequence.frame_end`

#### 3.2 Deprecated API Usage

Flag WARNING for deprecated APIs with known removal schedule:
- `bpy.utils.register_module(__name__)` → removed in 2.80, register classes individually
- `obj.select = True` → use `obj.select_set(True)` (removed in 2.80)
- `bpy.data.lamps` → use `bpy.data.lights` (removed in 2.80)
- `mesh.uv_textures` → use `mesh.uv_layers` (removed in 2.80)
- `bl_info` dict in addon → use `blender_manifest.toml` for Blender 4.2+ extensions

#### 3.3 Incorrect EEVEE Identifier

Flag BLOCKER if EEVEE identifier does not match target version:
| Version | Correct Identifier |
|---------|-------------------|
| 3.x     | `'BLENDER_EEVEE'` |
| 4.2–4.x | `'BLENDER_EEVEE_NEXT'` |
| 5.0+    | `'BLENDER_EEVEE'` |

---

### CHECK 4: Data Safety

**Severity: BLOCKER**

#### 4.1 Stale bpy.data References

Flag BLOCKER if code stores `bpy.data.*` references and uses them after:
- Any `bpy.ops.*` call (operators can trigger undo)
- Any `bpy.data.*.remove()` call
- Any `bpy.ops.wm.open_mainfile()` / `bpy.ops.wm.read_homefile()`
- Mode switches via `bpy.ops.object.mode_set()`
- Collection property `.add()` / `.remove()` (re-allocates C arrays)

**Detection**: Track variable assignments from `bpy.data.*`. If the variable is used after any of the above operations without re-fetching → BLOCKER.

#### 4.2 Name Collision on Data Creation

Flag WARNING if code does:
```python
bpy.data.meshes.new("Name")
mesh = bpy.data.meshes["Name"]  # May be "Name.001"
```
Correct pattern: `mesh = bpy.data.meshes.new("Name")` — capture the return value.

#### 4.3 Missing Object-Collection Link

Flag BLOCKER if code creates a data block and object but never calls `collection.objects.link(obj)`:
```python
mesh = bpy.data.meshes.new("Mesh")
obj = bpy.data.objects.new("Object", mesh)
# MISSING: bpy.context.collection.objects.link(obj)
```

#### 4.4 BMesh Lifecycle Errors

Flag BLOCKER if:
- `bm.verts[i]` accessed without prior `bm.verts.ensure_lookup_table()` call
- `bmesh.new()` used without corresponding `bm.free()` (except edit-mode BMesh from `bmesh.from_edit_mesh()`)
- `bm.to_mesh(mesh)` called without `mesh.update()` afterward

#### 4.5 Missing mesh.update() After from_pydata()

Flag BLOCKER if `mesh.from_pydata()` is called without `mesh.update()` afterward.

---

### CHECK 5: Threading Safety

**Severity: BLOCKER**

#### 5.1 bpy Access from Background Threads

Flag BLOCKER if any `bpy.*` call occurs inside:
- `threading.Thread` target functions
- `concurrent.futures.ThreadPoolExecutor` / `ProcessPoolExecutor` submitted callables
- Any function passed to thread creation

**Correct pattern**: Use `queue.Queue` + `bpy.app.timers.register()` to relay results to main thread.

#### 5.2 Unsafe Timer Callbacks

Flag WARNING if a timer callback performs heavy computation (loops over large data sets, file I/O). Timers run on the main thread and block the UI.

---

### CHECK 6: Addon/Extension Structure

**Severity: WARNING for structural issues, BLOCKER for registration errors**

#### 6.1 Registration Order

Flag BLOCKER if `register()` function:
- Registers operators/panels before their dependent PropertyGroup classes
- Creates PointerProperty referencing a type not yet registered
- Does not delete PointerProperty in `unregister()` before unregistering the type

Correct order: Register dependencies first (PropertyGroup → Operator → Panel). Unregister in reverse order.

#### 6.2 Handler Cleanup

Flag WARNING if addon registers handlers in `register()` but does not remove them in `unregister()`.

#### 6.3 Missing @persistent on Handlers

Flag BLOCKER if a handler callback is appended to `bpy.app.handlers.*` without the `@bpy.app.handlers.persistent` decorator. Without it, the handler is removed on file load.

#### 6.4 Dynamic EnumProperty Items GC

Flag BLOCKER if `EnumProperty(items=callback_function)` where the callback returns a freshly created list without caching it in a persistent variable. The list gets garbage-collected → crash or empty dropdown.

#### 6.5 Extension vs Legacy Addon Format

Flag INFO if target is Blender 4.2+ and code uses `bl_info` dict instead of `blender_manifest.toml`.

---

### CHECK 7: Operator Best Practices

**Severity: WARNING**

#### 7.1 Unnecessary bpy.ops Usage

Flag WARNING if `bpy.ops.*` is used where direct data access achieves the same result:
- `bpy.ops.object.location_clear()` → `obj.location = (0, 0, 0)`
- `bpy.ops.object.select_all(action='DESELECT')` then individual select → `obj.select_set()`
- `bpy.ops.transform.translate(value=...)` → `obj.location += Vector(...)`

#### 7.2 Missing Undo Support in Custom Operators

Flag WARNING if a custom Operator modifies data but does not set `bl_options = {'REGISTER', 'UNDO'}`.

---

### CHECK 8: GPU / Drawing Code (Blender 5.0+)

**Severity: BLOCKER**

Flag BLOCKER if code imports `bgl` on Blender 5.0+. The `bgl` module is removed.

Flag WARNING if code uses `bgl` on Blender 4.x without version guard — `bgl` is deprecated since 3.x.

All drawing code MUST use the `gpu` module (`gpu.shader`, `gpu.types.GPUBatch`, `gpu.state`).

---

## Severity Summary Decision Tree

```
Code will crash or corrupt data?
├── YES → BLOCKER
│   ├── Context access in restricted callback
│   ├── bpy access from thread
│   ├── Stale reference after undo/remove/mode-switch
│   ├── Removed API without version guard
│   ├── Missing ensure_lookup_table() on BMesh
│   ├── Missing bm.free() on standalone BMesh
│   ├── Missing collection.objects.link()
│   ├── Wrong registration order (PointerProperty before type)
│   ├── Missing @persistent on handler
│   ├── EnumProperty items GC (no cache)
│   ├── bgl import on Blender 5.0+
│   └── Missing mesh.update() after from_pydata()
├── NO → Code uses deprecated API?
│   ├── YES → WARNING
│   │   ├── Deprecated API with known removal version
│   │   ├── Dict context override on 3.2–3.6
│   │   ├── Operator without poll() pre-check
│   │   ├── Unnecessary bpy.ops usage
│   │   ├── Missing handler cleanup in unregister()
│   │   ├── Heavy computation in timer callback
│   │   └── Missing undo support in operator
│   └── NO → Best practice issue?
│       ├── YES → INFO
│       │   ├── No target version declared
│       │   ├── bl_info on 4.2+ (use blender_manifest.toml)
│       │   └── Style/naming conventions
│       └── NO → PASS
```

---

## Auto-Fix Patterns

When a check fails, apply these fixes automatically where safe:

| Issue | Auto-Fix |
|-------|----------|
| Dict context override | Replace with `context.temp_override()` |
| `obj.select = True` | Replace with `obj.select_set(True)` |
| `bpy.data.lamps` | Replace with `bpy.data.lights` |
| `mesh.uv_textures` | Replace with `mesh.uv_layers` |
| Missing `mesh.update()` after `from_pydata()` | Insert `mesh.update()` after `from_pydata()` call |
| Missing `bm.free()` | Insert `bm.free()` after last `bm.to_mesh()` call |
| Missing `ensure_lookup_table()` | Insert `bm.verts.ensure_lookup_table()` before first index access |
| Missing `collection.objects.link()` | Insert `bpy.context.collection.objects.link(obj)` after `objects.new()` |
| `bgl` import on 5.0+ | Replace with `gpu` module equivalents |
| `scene["cycles"]` on 5.0+ | Replace with `scene.cycles` attribute access |
| `del obj["prop"]` on 5.0+ | Replace with `obj.property_unset("prop")` |
| `BLENDER_EEVEE` on 4.2–4.x | Replace with `BLENDER_EEVEE_NEXT` |
| `bone.layers` on 4.0+ | Replace with `bone.collections` usage |
| `NodeTree.inputs.new()` on 4.0+ | Replace with `NodeTree.interface.new_socket()` |
| Missing `@persistent` on handler | Add `@bpy.app.handlers.persistent` decorator |

---

## Validation Report Format

After running all checks, produce a report:

```
## Validation Report
Target Version: Blender X.Y
Total Issues: N

### BLOCKERS (N)
- [CHECK X.Y] Description — file:line

### WARNINGS (N)
- [CHECK X.Y] Description — file:line

### INFO (N)
- [CHECK X.Y] Description — file:line

### Auto-Fixes Applied (N)
- Description — file:line
```

---

## Reference Links

- **Validation Rules**: [references/methods.md](references/methods.md)
- **Before/After Examples**: [references/examples.md](references/examples.md)
- **Anti-Patterns Catalog**: [references/anti-patterns.md](references/anti-patterns.md)

### Dependency Skills
- `blender-core-api` — bpy module structure and data access patterns
- `blender-core-versions` — Version matrix and breaking changes
- `blender-core-runtime` — Threading, handlers, timers, mathutils
- `blender-errors-context` — Context error diagnosis and resolution
- `blender-errors-data` — Data reference errors and safe access patterns
- `blender-errors-version` — Version compatibility error diagnosis

### Official Documentation
- Blender Python API: https://docs.blender.org/api/current/
- Blender Gotchas: https://docs.blender.org/api/current/info_gotcha.html
- Blender 4.0 Release Notes: https://wiki.blender.org/wiki/Reference/Release_Notes/4.0
- Blender 5.0 Release Notes: https://wiki.blender.org/wiki/Reference/Release_Notes/5.0
