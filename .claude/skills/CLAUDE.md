---
name: AEC Python Development — Blender, Bonsai, IfcOpenShell, Sverchok
description: Root skill for all AEC Python work. Triggers on Blender bpy scripting, Bonsai BIM authoring, IfcOpenShell IFC manipulation, Sverchok parametric design, and Blender MCP live interaction.
---

# AEC Python — Quick Reference

## 1. Blender MCP Workflow

ALWAYS follow this cycle when Blender MCP is connected:

```
get_scene_info → execute_blender_code → get_viewport_screenshot
```

| Tool | Purpose | When |
|------|---------|------|
| `get_scene_info` | Read current scene state | ALWAYS call first |
| `execute_blender_code` | Run Python in Blender | Every code action |
| `get_viewport_screenshot` | Visual verification | ALWAYS call after execution |

NEVER skip `get_scene_info` — you need scene state before making changes.
NEVER skip `get_viewport_screenshot` — visual confirmation catches silent failures.

## 2. Python Runtime Rules

### Blender Embedded Python
- `bpy` runs on the **main thread only** — NEVER use `threading` for bpy operations
- `bpy.data` references are **invalidated by undo/redo** — NEVER cache ID references across operations
- `bpy.ops.*` ALWAYS check `poll()` before calling — operators can fail silently
- `bpy.context.object` can be `None` — ALWAYS check before use
- Use `bpy.app.timers.register()` for deferred execution, NEVER `threading.Timer`
- Use `bpy.msgbus` for property change notifications, NEVER polling loops
- **No `bgl` module** — removed in Blender 5.0, use `gpu` module instead

### IfcOpenShell C++ Bindings
- Entity objects wrap C++ pointers — references invalidate when the file is modified or GC'd
- ALWAYS use `ifcopenshell.api.run()` for mutations — NEVER modify attributes directly
- Entity identity: use `is` for same-object check, `==` for value equality
- ALWAYS check schema: `ifc_file.schema` returns `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"`

## 3. IfcOpenShell Quick Patterns

ALWAYS import ifcopenshell inside `execute_blender_code` — it runs in Blender's Python:

```python
import ifcopenshell
import ifcopenshell.api

# Create a new IFC file
ifc = ifcopenshell.api.run("project.create_file", schema="IFC4")

# Create entities — ALWAYS use api.run(), NEVER construct directly
wall = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcWall", name="Wall_01")

# Write to disk
ifc.write("/path/to/output.ifc")
```

### Universal Property Extraction Pattern
```python
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    name = prop.Name
                    value = prop.NominalValue.wrappedValue
```

### Schema-Aware Coding
```python
schema = ifc_file.schema
if schema == "IFC2X3":
    # Use IfcRelContainedInSpatialStructure
elif schema in ("IFC4", "IFC4X3"):
    # Additional entity types available
```

## 4. Bonsai Quick Patterns

### IFC Access
```python
import bpy
# Method 1: via tool (preferred in Bonsai context)
import blenderbim.tool as tool
ifc_file = tool.Ifc.get()

# Method 2: via IfcStore
from blenderbim.bim.ifc import IfcStore
ifc_file = IfcStore.get_file()
```

### Core Rules
- Everything **IS IFC** — there is no "export to IFC". Saving IS the IFC file
- Operators live under `bpy.ops.bim.*` — ALWAYS check `poll()` first
- Spatial hierarchy: use `assign_container` for physical elements, `aggregate` for spatial elements
- Materials: ALWAYS assign to **type**, NEVER to occurrence
- Bonsai extends Blender's properties with IFC-backed custom properties

### Spatial Structure
```python
import ifcopenshell.api

# Physical element → spatial container
ifcopenshell.api.run("spatial.assign_container", ifc,
    products=[wall], relating_structure=storey)

# Spatial element → spatial parent
ifcopenshell.api.run("aggregate.assign_object", ifc,
    products=[storey], relating_object=building)
```

## 5. Sverchok Quick Patterns

### Data Nesting Convention
```python
vertices = [[[x, y, z], [x, y, z]]]   # List of lists of [x,y,z] — triple nested
edges    = [[[0, 1], [1, 2]]]          # List of lists of [i,j] — double nested
faces    = [[[0, 1, 2, 3]]]            # List of lists of indices — double nested
scalars  = [[1.0, 2.0, 3.0]]          # List of lists of values — double nested
```

### Programmatic Node Trees
```python
import sverchok
from sverchok.utils.testing import create_node_tree

tree = create_node_tree()
# Or via bpy:
tree = bpy.data.node_groups.new("MyTree", "SverchCustomTreeType")

node_a = tree.nodes.new("SvNumberNode")
node_b = tree.nodes.new("SvGenVectorsNodeMk2")
tree.links.new(node_a.outputs[0], node_b.inputs[0])
```

### Key Rules
- `updateNode()` callback is REQUIRED after parameter changes
- Use **SNLite** node for custom Python logic inside node trees
- **IfcSverchok** exports parametric geometry to IFC from node trees
- NEVER modify node tree data outside Blender's main thread

## 6. Version Detection

ALWAYS run this check at the start of any AEC session:

```python
import bpy
print(f"Blender: {'.'.join(str(v) for v in bpy.app.version)}")

import ifcopenshell
print(f"IfcOpenShell: {ifcopenshell.version}")

try:
    import blenderbim
    from blenderbim.bim.ifc import IfcStore
    print(f"Bonsai: {blenderbim.get_version()}")
except ImportError:
    print("Bonsai: not installed")

try:
    import sverchok
    print(f"Sverchok: {sverchok.bl_info.get('version', 'unknown')}")
except ImportError:
    print("Sverchok: not installed")

# If IFC file is open:
try:
    import blenderbim.tool as tool
    ifc_file = tool.Ifc.get()
    if ifc_file:
        print(f"IFC Schema: {ifc_file.schema}")
except:
    pass
```

## 7. Common Workflow Recipes

### Recipe A: Create IFC Model from Scratch
```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.api.run("project.create_file", schema="IFC4")
project = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcProject", name="Demo")
ifcopenshell.api.run("unit.assign_unit", ifc)
site = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcBuildingStorey", name="Ground Floor")
ifcopenshell.api.run("aggregate.assign_object", ifc, products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", ifc, products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", ifc, products=[storey], relating_object=building)
ifc.write("/tmp/demo.ifc")
```

### Recipe B: Assign Material in Blender
```python
import bpy

mat = bpy.data.materials.new(name="Concrete")
mat.use_nodes = True
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs["Base Color"].default_value = (0.6, 0.6, 0.6, 1.0)
bsdf.inputs["Roughness"].default_value = 0.8

obj = bpy.context.active_object
if obj and obj.data:
    if obj.data.materials:
        obj.data.materials[0] = mat
    else:
        obj.data.materials.append(mat)
```

### Recipe C: Build Sverchok Node Tree from Python
```python
import bpy

tree = bpy.data.node_groups.new("Parametric", "SverchCustomTreeType")

# Create nodes
num = tree.nodes.new("SvNumberNode")
num.location = (0, 0)
num.int_ = 10

grid = tree.nodes.new("SvPlaneNodeMk3")
grid.location = (200, 0)

viewer = tree.nodes.new("SvViewerDrawMk4")
viewer.location = (400, 0)

# Connect
tree.links.new(num.outputs[0], grid.inputs[0])
tree.links.new(grid.outputs[0], viewer.inputs[0])
tree.links.new(grid.outputs[1], viewer.inputs[1])
```

## 8. Breaking Changes Reference

| Change | Version | Migration |
|--------|---------|-----------|
| `bgl` module removed | Blender 5.0 | Use `gpu` module |
| `3D_UNIFORM_COLOR` shader | Blender 4.0 | Use `UNIFORM_COLOR` |
| `Object.hide` attribute | Blender 4.0 | Use `hide_set()` / `hide_get()` |
| Legacy `bl_info` addon format | Blender 5.0 | Use `blender_manifest.toml` |
| `bpy.ops.export_scene.obj()` | Blender 4.0 | Use `bpy.ops.wm.obj_export()` |

## 9. Do NOT

- Use `threading` for `bpy` operations — crashes Blender
- Cache `bpy.data` references across undo boundaries — they invalidate
- Hardcode IFC schema — ALWAYS check `ifc_file.schema`
- Use `bgl` in Blender 4.0+ — deprecated, removed in 5.0
- Call `bpy.ops.*` without `poll()` check — silent failures
- Assume `bpy.context.object` is set — it can be `None`
- Modify entity attributes directly — ALWAYS use `ifcopenshell.api.run()`
- Assign materials to occurrences in Bonsai — ALWAYS assign to type
- Skip `get_scene_info` before MCP execution — you need current state
- Skip `get_viewport_screenshot` after MCP execution — visual verification is mandatory
