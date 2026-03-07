---
name: bonsai-core-architecture
description: 'Explains Bonsai (formerly BlenderBIM) addon architecture including native IFC workflow, tool.Ifc.get() for IfcOpenShell file access, bpy.ops.bim.* operator namespace, BIM property panels, IFC-backed custom properties, and the relationship between Blender objects and IFC entities. Activates when working with Bonsai addon, developing Bonsai extensions, or scripting BIM workflows in Blender.'
version: 'Bonsai v0.8.4 | Minimum Blender: 4.2.0 | Python 3.11'
---

# Bonsai Core Architecture

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.*` — NEVER `blenderbim.*`
> **Repository**: `IfcOpenShell/IfcOpenShell` monorepo → `src/bonsai/`

## Critical Warnings

1. **ALWAYS** use `bonsai.*` imports. NEVER use `blenderbim.*` — renamed in 2024 (v0.8.0+).
2. **ALWAYS** check `IfcStore.get_file()` for `None` before any IFC operation.
3. **NEVER** describe Bonsai workflow as "import/export". The IFC file IS the document.
4. **ALWAYS** run Bonsai scripts via `blender --python`. NEVER import `bonsai.*` from system Python.
5. **ALWAYS** call `bpy.ops.bim.edit_object_placement()` after direct matrix/location changes.
6. **ALWAYS** use `ifcopenshell.api.run()` for IFC mutations, NEVER modify entity attributes directly.
7. **NEVER** use `void.add_opening()` — use `feature.add_feature()` (changed in v0.8.0+).

## Three-Layer Architecture

```
+-----------------------------------------------------+
|  DELIVERY LAYER  — bim/module/                       |
|  Blender UI: operators, properties, panels, gizmos   |
+-----------------------------------------------------+
|  DOMAIN LAYER   — core/ + tool/                      |
|  Abstract use-case logic (core) + concrete impls     |
|  (tool), injected via @interface contract             |
+-----------------------------------------------------+
|  DATA LAYER     — IfcStore + IFC in-memory graph     |
|  Live ifcopenshell.file instance, id_map, guid_map   |
+-----------------------------------------------------+
```

### Layer Rules

| Layer | Location | Imports `bpy`? | Imports `ifcopenshell`? | Responsibility |
|-------|----------|---------------|------------------------|----------------|
| Delivery | `bim/module/<name>/` | YES | NO (via tool) | Blender UI, operators, panels |
| Core | `core/<name>.py` | NEVER | Via injected tool | Abstract use-case logic |
| Tool | `tool/<name>.py` | YES | YES | Concrete Blender + IFC operations |

**Key principle**: Core NEVER imports `bpy`. All Blender-specific code lives in Tool implementations. This keeps business logic independently testable.

## Module Structure Pattern

Every functional area follows a three-part split:

| Sub-layer | Location | Purpose |
|-----------|----------|---------|
| UI (delivery) | `bim/module/<name>/ui.py`, `operator.py`, `prop.py` | Panels, operators, properties |
| Core (abstract) | `core/<name>.py` | Use-case flow, no Blender imports |
| Tool (concrete) | `tool/<name>.py` | Blender + IFC implementation |

### Module File Structure

| File | Purpose |
|------|---------|
| `__init__.py` | Package init |
| `operator.py` | `bpy.types.Operator` subclasses (`bpy.ops.bim.*`) |
| `prop.py` | `bpy.types.PropertyGroup` subclasses |
| `ui.py` | Panel `draw()` methods |
| `data.py` | Cached/computed UI data (refresh patterns) |
| `decorator.py` | `bpy.types.SpaceView3D` draw callbacks (overlays) |
| `gizmo.py` | `bpy.types.Gizmo` / `GizmoGroup` subclasses |

### Registered Modules (~51-56)

Bonsai registers approximately 51-56 functional modules, each corresponding to an IFC domain area:

```
aggregate    alignment    attribute    bcf         boundary
brick        bsdd         cad          clash       classification
constraint   context      cost         covering    csv
debug        demo         diff         document    drawing
fm           geometry     georeference gis         ifcgit
layer        library      material     nest        owner
patch        profile      project      pset        pset_template
qto          resource     search       sequence    spatial
structural   style        system       tester      type
unit         void         web
```

### Demo Module — Reference Implementation

The `demo` module (`bim/module/demo/`) is a heavily commented reference implementation for new contributors. It demonstrates the full module pattern including all optional files. Enable it in `bonsai/bim/__init__.py`.

## The `@interface` Decorator Pattern

`core/tool.py` defines tool interfaces using the `@interface` decorator — a design-by-contract enforcement mechanism:

```python
# bonsai/core/tool.py (simplified)
@interface
class Ifc:
    def get(self): ...          # Returns ifcopenshell.file or None
    def set(self, ifc): ...     # Sets active IFC file
    def run(self, command, **kwargs): ...  # Runs ifcopenshell.api command
    def get_entity(self, obj): ...        # Blender obj -> IFC entity
    def get_object(self, element): ...    # IFC entity -> Blender obj
    def link(self, element, obj): ...     # Bidirectional link

@interface
class Spatial:
    def assign_container(self, element, structure): ...
```

Concrete implementations live in `tool/ifc.py`, `tool/spatial.py`, etc. Operators import them as `tool.Ifc`, `tool.Spatial`.

## Operator Execution Pattern

### Standard Pattern

```python
# bim/module/spatial/operator.py (Delivery Layer)
class BIM_OT_assign_container(bpy.types.Operator):
    bl_idname = "bim.assign_container"
    bl_label = "Assign Container"
    bl_options = {"REGISTER", "UNDO"}

    def execute(self, context):
        return IfcStore.execute_ifc_operator(self, context)

    def _execute(self, context):
        core.spatial.assign_container(
            tool.Ifc,        # injected concrete tool
            tool.Spatial,
            structure=self.structure,
        )
        return {"FINISHED"}
```

### Key Rules for Operators

1. **ALWAYS** use `IfcStore.execute_ifc_operator(self, context)` — it wraps IFC operations with undo support.
2. Put actual logic in `_execute()`, NOT in `execute()`.
3. Pass `tool.*` classes as dependency injection parameters to `core.*` functions.
4. **ALWAYS** include `"UNDO"` in `bl_options` for IFC-modifying operators.
5. Operator namespace is `bpy.ops.bim.*` — ALWAYS use this prefix.

## IfcStore — Central State Manager

`IfcStore` (`bonsai/bim/ifc.py`) is a **static class** — the single source of truth for all IFC state:

```python
from bonsai.bim.ifc import IfcStore

# Key static attributes
IfcStore.path       # str: absolute path to .ifc file
IfcStore.file       # ifcopenshell.file | None: live in-memory IFC
IfcStore.schema     # str: "IFC2X3", "IFC4", or "IFC4X3"
IfcStore.id_map     # dict: {ifc_id (int) -> blender_object}
IfcStore.guid_map   # dict: {GlobalId (str) -> blender_object}
IfcStore.edited_objs  # set: objects with pending IFC changes

# Key static methods
model = IfcStore.get_file()        # Returns ifcopenshell.file or None
schema = IfcStore.get_schema()     # Returns schema string
obj = IfcStore.get_element(ifc_id) # Blender object by IFC id
```

### ALWAYS Guard Against None

```python
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")
walls = model.by_type("IfcWall")
```

## `tool.Ifc` — High-Level IFC Access

| Method | Purpose | Returns |
|--------|---------|---------|
| `tool.Ifc.get()` | Live IFC file instance | `ifcopenshell.file` or `None` |
| `tool.Ifc.set(ifc)` | Set active IFC file + post-load hooks | None |
| `tool.Ifc.run(command, **kwargs)` | Execute `ifcopenshell.api` command | Varies |
| `tool.Ifc.get_entity(obj)` | Blender object -> IFC entity | `entity_instance` or `None` |
| `tool.Ifc.get_object(element)` | IFC entity -> Blender object | `bpy.types.Object` or `None` |
| `tool.Ifc.link(element, obj)` | Create bidirectional link | None |

## Blender Object <-> IFC Entity Mapping

```
Blender Object                        IFC Entity
-----------------                     ----------
BIMObjectProperties                   ifcopenshell.entity_instance
  .ifc_definition_id  ------------->  .id()
                      <------------- IfcStore.id_map
  .name                               .Name
  Matrix (loc/rot/scale)              IfcLocalPlacement
  Mesh geometry                       IfcShapeRepresentation
```

### Decision Tree: Accessing IFC Data

```
Need IFC data from Blender object?
|
+--> Use tool.Ifc.get_entity(obj)
     |
     +--> Returns entity_instance? --> Access .GlobalId, .Name, .is_a()
     +--> Returns None? --> Object is not linked to IFC

Need Blender object from IFC entity?
|
+--> Use tool.Ifc.get_object(entity)
     |
     +--> Returns bpy.types.Object? --> Manipulate via Blender API
     +--> Returns None? --> Entity has no Blender representation

Need to run IFC API command?
|
+--> Inside Bonsai (bpy available)?
|    +--> Use tool.Ifc.run("module.command", **kwargs)
|
+--> Standalone (no bpy)?
     +--> Use ifcopenshell.api.run("module.command", ifc_file, **kwargs)
```

## Native IFC Workflow

The IFC file IS the document. There is NO import/export step.

| Action | What Happens |
|--------|-------------|
| Create new project | Empty `ifcopenshell.file` created in memory |
| Open IFC file | `ifcopenshell.open()` loads directly into `IfcStore` |
| Create element | IFC entity created + Blender mesh generated + bidirectional link |
| Modify element | IFC graph updated via `ifcopenshell.api` + Blender mesh regenerated |
| Save project | `model.write(path)` — writes live IFC directly to disk |

### File Conventions

- Bonsai native files: `.ifc` (saves directly to IFC, NOT `.blend`)
- Schema versions: `IFC2X3` (legacy), `IFC4` (recommended), `IFC4X3` (latest)

## Major `tool.*` Classes

| Tool Class | Purpose | Key Methods |
|-----------|---------|-------------|
| `tool.Ifc` | IFC model access, entity<->object | `get()`, `get_entity()`, `get_object()`, `run()` |
| `tool.Spatial` | Spatial container hierarchy | `get_container()`, `get_decomposed_elements()` |
| `tool.Geometry` | Mesh/representation management | `get_active_representation()`, `edit_object_placement()` |
| `tool.Blender` | Blender scene utilities | `get_obj_ifc_definition_id()`, `set_active_object()` |
| `tool.Pset` | Property sets management | `get_element_pset()`, `get_pset_props()` |
| `tool.Type` | IFC type management | `get_type()`, `assign_type()` |
| `tool.Material` | Material/layer sets | `get_material()`, `get_style()` |
| `tool.Context` | Representation contexts | `get_context()` |
| `tool.Style` | Visual/surface styles | `get_style()`, `get_surface_style()` |
| `tool.Drawing` | 2D drawing/documentation | `get_active_drawing()`, `get_drawing_elements()` |

## BIM Property Panels

Bonsai extends Blender's property system with IFC-backed custom properties via `BIMObjectProperties`:

- **Object Metadata**: `Properties > Object Information > Object Metadata` — IFC class, type
- **Spatial Container**: `Properties > Object Information > Spatial Container` — containing storey/space
- **Property Sets**: `Properties > Object Information > Property Sets` — IFC psets
- **Quantity Sets**: `Properties > Object Information > Quantity Sets` — IFC qtos

### IFC-Backed Custom Properties

Bonsai registers `BIMObjectProperties` on every Blender object:

```python
# Key property: ifc_definition_id
obj.BIMProperties.ifc_definition_id  # int: IFC entity step ID (0 = not linked)
```

Property modifications in Blender UI panels are synced to IFC via `pset.edit_pset()`. NEVER assume modifying a Blender property alone updates the IFC data.

## IfcOpenShell API Integration

Bonsai uses three abstraction levels for IFC operations:

| Level | Module | Use When |
|-------|--------|----------|
| Core | `ifcopenshell.file` | Low-level entity creation/read |
| Utility | `ifcopenshell.util.*` | Data extraction, conversions |
| API | `ifcopenshell.api.run()` | ALL mutations (ALWAYS use this) |

### Preferred: `ifcopenshell.api.run()`

```python
# ALWAYS use the API for mutations — maintains IFC graph integrity
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")

# NEVER modify attributes directly
# wall.Name = "Bad"  # WRONG — bypasses relationship management
```

## Blender Registration

```python
# bonsai/__init__.py (schematic)
bl_info = {
    "name": "Bonsai",
    "version": (0, 8, 4),
    "blender": (4, 2, 0),
    "category": "System",
}

def register():
    register_classes(classes)
    for mod in modules.values():
        mod.register()
    bpy.types.Scene.BIMProperties = PointerProperty(type=BIMProperties)

def unregister():
    for mod in reversed(list(modules.values())):
        mod.unregister()
    unregister_classes(classes)
```

## IfcSverchok Integration (`use_bonsai_file`)

IfcSverchok (Sverchok's IFC bridge) can operate on Bonsai's active IFC file instead of its own transient `SvIfcStore`:

- **Toggle**: `use_bonsai_file` on `SvIfcCreateProject` node
- **When enabled**: `SvIfcStore.get_file()` returns `tool.Ifc.get()` (Bonsai's live IFC)
- **Effect**: Sverchok node tree writes IFC entities directly into Bonsai's model
- **Dependency**: Bonsai addon MUST be enabled and have an active IFC project loaded
- **Warning**: Undo/redo with simultaneous Sverchok + Bonsai modifications can crash. Save before combining.

Refer to: `sverchok-impl-ifcsverchok`

## Source Code Paths (IfcOpenShell Monorepo)

| Component | Path |
|-----------|------|
| Bonsai root | `src/bonsai/bonsai/` |
| IfcStore | `src/bonsai/bonsai/bim/ifc.py` |
| Tool classes | `src/bonsai/bonsai/tool/` |
| Core functions | `src/bonsai/bonsai/core/` |
| Module operators/UI | `src/bonsai/bonsai/bim/module/` |
| Interface definitions | `src/bonsai/bonsai/core/tool.py` |
| Demo module | `src/bonsai/bonsai/bim/module/demo/` |
| IfcOpenShell API | `src/ifcopenshell-python/ifcopenshell/api/` |

## References

- [methods.md](references/methods.md) — Complete API signatures for tool.*, IfcStore, core functions
- [examples.md](references/examples.md) — Working code examples for common Bonsai operations
- [anti-patterns.md](references/anti-patterns.md) — What NOT to do when scripting Bonsai

## Sources

- Bonsai Documentation: https://docs.bonsaibim.org
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell
- OSArch Community: https://community.osarch.org
- IfcOpenShell API Docs: https://docs.ifcopenshell.org
