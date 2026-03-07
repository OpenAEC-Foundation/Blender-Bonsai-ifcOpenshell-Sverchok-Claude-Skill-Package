# AEC Python Development — Blender, Bonsai, IfcOpenShell

## What This Project Uses
- **Blender Python API** (`bpy`) — 3D modeling, rendering, animation for AEC
- **IfcOpenShell** — IFC file parsing, creation, geometry processing (C++ core with Python bindings)
- **Bonsai** (formerly BlenderBIM) — Native IFC BIM authoring inside Blender
- **Sverchok** (optional) — Parametric/generative design via node trees in Blender

## Critical Python Runtime Rules

### Blender Embedded Python
- Blender runs an **embedded CPython**. All `bpy` calls MUST happen on the **main thread**
- `bpy.data` references are **invalidated by undo/redo**. Never cache ID references across operations
- In operator `execute()`, `modal()`, and draw callbacks: **context is restricted**. Only read the specific context attributes documented for that handler
- Use `bpy.app.timers.register()` for deferred execution, NEVER `threading.Timer`
- `bpy.msgbus` for property change notifications, NEVER polling loops
- Background mode (`blender --background`): no UI, no OpenGL context, no viewport operators

### IfcOpenShell C++ Bindings
- Entity objects are **wrappers around C++ pointers**. If the underlying file is modified or garbage-collected, references become invalid
- Always use `ifcopenshell.api.run()` for mutations, never modify attributes directly on entity wrappers
- Schema matters: IFC2X3, IFC4, IFC4X3 have different entity names and property sets
- `ifcopenshell.util` modules provide safe extraction patterns — prefer these over manual traversal

### Bonsai Context
- Bonsai stores ALL data as native IFC. There is no "export to IFC" — saving IS the IFC file
- Access BIM data via `tool.Ifc.get()` to get the IfcOpenShell file object
- Bonsai operators live under `bpy.ops.bim.*` — check `poll()` before calling
- Bonsai extends Blender's property system with IFC-backed custom properties

### Sverchok Runtime (when addon is enabled)
- Sverchok uses its own node tree type (`SverchCustomTreeType`) — access via `bpy.data.node_groups` filtered by `bl_idname`
- **CRITICAL**: Data nesting levels must be correct — vertices=level 3 `[[[x,y,z]]]`, edges/strings=level 2, matrices=level 1. Wrong nesting = silent wrong results
- **ALWAYS** use `updateNode` callback on node `bpy.props` properties — without it, property changes won't trigger node re-evaluation
- IfcSverchok nodes share a single transient IFC file via `SvIfcStore` — purged on every full tree update. Use `use_bonsai_file` to persist to Bonsai's file
- Socket data is cached in `socket_data_cache` — **ALWAYS** use `deepcopy=True` on `sv_get()` if you mutate data, or you corrupt upstream nodes
- Sverchok availability check: `addon_utils.check("sverchok")` returns `(is_enabled, is_loaded)`

## Version Detection
```python
# Blender version
import bpy
major, minor, patch = bpy.app.version  # e.g., (4, 2, 0)

# IfcOpenShell version
import ifcopenshell
ifcopenshell.version  # e.g., "0.8.1"

# IFC schema of an open file
ifc_file = ifcopenshell.open("model.ifc")
schema = ifc_file.schema  # "IFC2X3", "IFC4", "IFC4X3"
```

## Key Breaking Changes (Blender)
| Change | Removed In | Migration |
|--------|-----------|-----------|
| `bgl` module | 5.0 | Use `gpu` module |
| `3D_UNIFORM_COLOR` shader | 4.0 | Use `UNIFORM_COLOR` |
| `register_class()` on non-RNA types | 4.0 | Only register bpy.types subclasses |
| `Object.hide` | 4.0 | Use `Object.hide_set()` / `Object.hide_get()` |
| Legacy addon `bl_info` | 5.0 | Use extension manifest `blender_manifest.toml` |
| `bpy.ops.export_scene.obj()` | 4.0 | Use `bpy.ops.wm.obj_export()` |

## Testing and Verification

### Blender Scripts (headless)
```bash
blender --background --python my_script.py
blender --background model.blend --python my_script.py
```

### IfcOpenShell Scripts (standalone)
```bash
python my_ifc_script.py  # No Blender needed
```

### Bonsai Scripts (requires Blender + Bonsai addon)
```bash
blender --background --python my_bonsai_script.py  # Bonsai must be enabled
```

### When Blender MCP Is Connected
If the `blender-mcp` MCP server is available, use its tools for live interaction:
- `get_scene_info` — inspect current scene state
- `execute_blender_code` — run Python code directly in Blender
- `get_viewport_screenshot` — visual verification of results
- Always call `get_scene_info` first to understand current state before making changes

## IFC Property Extraction (Universal Pattern)
```python
# This pattern works everywhere: IfcOpenShell, Bonsai, web-ifc
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    name = prop.Name
                    value = prop.NominalValue.wrappedValue
```

## Common Workflow: Create IFC Element with IfcOpenShell API
```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.api.run("project.create_file", schema="IFC4")
project = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", ifc)
site = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcSite", name="Site")
ifcopenshell.api.run("aggregate.assign_object", ifc, products=[site], relating_object=project)
ifc.write("output.ifc")
```

## File Conventions
- IFC files: `.ifc` (STEP format), `.ifcJSON`, `.ifcXML`
- Blender files: `.blend`
- Bonsai native files: `.ifc` (Bonsai saves directly to IFC, not .blend)

## Do NOT
- Use `threading` module for `bpy` operations (crashes Blender)
- Cache `bpy.data` references across undo boundaries
- Hardcode IFC schema — always check `ifc_file.schema`
- Use `bgl` in Blender 4.0+ code (deprecated, removed in 5.0)
- Call `bpy.ops.*` without verifying `poll()` returns True
- Assume `bpy.context.object` is always available (it can be None)
