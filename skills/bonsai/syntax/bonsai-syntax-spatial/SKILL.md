---
name: bonsai-syntax-spatial
description: "Provides Bonsai spatial structure syntax for creating and managing IFC spatial hierarchy including IfcSite, IfcBuilding, IfcBuildingStorey, and IfcSpace elements. Covers spatial containment (IfcRelContainedInSpatialStructure), spatial decomposition (IfcRelAggregates), and Bonsai-specific spatial tools for navigating and modifying the building structure tree."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Spatial Structure Syntax

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.*` — NEVER `blenderbim.*`
> **Source files**: `core/spatial.py`, `tool/spatial.py`, `bim/module/spatial/`

## Critical Warnings

1. **ALWAYS** use `bonsai.*` imports. NEVER use `blenderbim.*` — renamed in 2024 (v0.8.0+).
2. **ALWAYS** build spatial hierarchy via `IfcRelAggregates` using `ifcopenshell.api.run("aggregate.assign_object", ...)`. NEVER create spatial elements without connecting them to the hierarchy.
3. **ALWAYS** use `ifcopenshell.api.run("spatial.assign_container", ...)` to place physical elements in spatial containers. NEVER set containment via direct attribute manipulation.
4. **ALWAYS** pass `products` as a **list** (v0.8+ breaking change): `products=[wall]`, NOT `product=wall`.
5. **NEVER** delete `IfcProject` — it is the root of every IFC model. Bonsai operators enforce this constraint.
6. **ALWAYS** set a default container (typically `IfcBuildingStorey`) before creating elements. Use `bpy.ops.bim.set_default_container(container=storey_id)`.
7. **NEVER** assign an element to multiple containers via `IfcRelContainedInSpatialStructure` — an element can only be contained in ONE spatial structure. Use `IfcRelReferencedInSpatialStructure` for multi-storey references.
8. **ALWAYS** use `ifcopenshell.api.run("root.create_entity", ...)` to create spatial elements. NEVER use `model.create_entity()` — it skips GlobalId generation and ownership assignment.

## IFC Spatial Hierarchy

```
IfcProject                          (exactly 1 per file, root node)
  └── IfcSite                       (via IfcRelAggregates)
        └── IfcBuilding             (via IfcRelAggregates)
              └── IfcBuildingStorey  (via IfcRelAggregates)
                    ├── IfcSpace    (via IfcRelAggregates)
                    ├── IfcWall     (via IfcRelContainedInSpatialStructure)
                    ├── IfcDoor     (via IfcRelContainedInSpatialStructure)
                    └── IfcColumn   (via IfcRelContainedInSpatialStructure)
```

### Spatial Elements

| IFC Class | Purpose | Required? | Typical Count |
|-----------|---------|-----------|---------------|
| `IfcProject` | Root of model | YES (exactly 1) | 1 |
| `IfcSite` | Geographic land parcel | Recommended | 1+ |
| `IfcBuilding` | Physical building structure | Recommended | 1+ |
| `IfcBuildingStorey` | Floor level | Recommended | 1+ per building |
| `IfcSpace` | Room, corridor, void | Optional | 0+ per storey |

### Three Key Relationships

| Relationship | IFC Class | Purpose | Cardinality |
|-------------|-----------|---------|-------------|
| Spatial decomposition | `IfcRelAggregates` | Parent-child between spatial elements | 1:N |
| Spatial containment | `IfcRelContainedInSpatialStructure` | Places physical elements in a spatial zone | 1:N (exclusive) |
| Spatial reference | `IfcRelReferencedInSpatialStructure` | Non-exclusive reference (multi-storey elements) | M:N |

## Decision Tree: Spatial Operations

```
What spatial operation do you need?
│
├── Create spatial hierarchy (Project/Site/Building/Storey)?
│   ├── Create elements: root.create_entity with ifc_class="IfcSite" etc.
│   └── Connect hierarchy: aggregate.assign_object
│
├── Place physical element (wall/door/column) in a storey or space?
│   └── spatial.assign_container (products=[element], relating_structure=storey)
│
├── Remove element from container?
│   └── spatial.unassign_container (products=[element])
│
├── Reference element in additional storey (multi-storey column)?
│   └── spatial.reference_structure (products=[element], relating_structure=storey2)
│
├── Remove spatial reference?
│   └── spatial.dereference_structure (products=[element], relating_structure=storey2)
│
├── Query spatial containment?
│   └── ifcopenshell.util.element.get_container(element)
│
├── Query spatial decomposition?
│   └── ifcopenshell.util.element.get_decomposition(container)
│
├── Delete spatial container?
│   └── bpy.ops.bim.delete_container(container=container_id)
│   NOTE: Cannot delete IfcProject
│
└── Generate IfcSpace from walls?
    ├── From cursor/object position: bpy.ops.bim.generate_space()
    └── From selected walls: bpy.ops.bim.generate_spaces_from_walls()
```

## Essential Patterns

### Pattern 1: Build Complete Spatial Hierarchy

```python
# Bonsai v0.8.x — Build spatial hierarchy from scratch
# Requires: active Bonsai project in Blender
import bpy
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Step 1: Create project (root)
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")

# Step 2: Assign units (required before geometry)
ifcopenshell.api.run("unit.assign_unit", model)

# Step 3: Create spatial elements
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Default Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
ground_floor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
first_floor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")

# Step 4: Connect hierarchy via IfcRelAggregates
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[ground_floor, first_floor], relating_object=building)
```

### Pattern 2: Assign Element to Container

```python
# Bonsai v0.8.x — Place wall in a storey
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001")

# Assign to ground floor (IfcRelContainedInSpatialStructure)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=ground_floor)
```

### Pattern 3: Multi-Storey Element Reference

```python
# Bonsai v0.8.x — Column spanning two storeys
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")

# Primary containment in ground floor
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=ground_floor)

# Additional reference in first floor (IfcRelReferencedInSpatialStructure)
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[column], relating_structure=first_floor)
```

### Pattern 4: Query Spatial Information

```python
# Bonsai v0.8.x — Query spatial relationships
import ifcopenshell.util.element

# Get container of an element
container = ifcopenshell.util.element.get_container(wall)
# Returns: IfcBuildingStorey or None

# Get all elements in a container
elements = ifcopenshell.util.element.get_decomposition(building)
# Returns: list of child spatial elements

# Get all parts (direct children via IfcRelAggregates)
parts = ifcopenshell.util.element.get_parts(building)
# Returns: list of IfcBuildingStorey instances
```

### Pattern 5: Bonsai Operators for Spatial Operations

```python
# Bonsai v0.8.x — Using Bonsai operators (inside Blender)
import bpy

# Set default container for new elements
bpy.ops.bim.set_default_container(container=storey.id())

# Assign selected objects to a container
bpy.ops.bim.assign_container(container=storey.id())

# Remove container assignment from selected objects
bpy.ops.bim.remove_container()

# Reference selected objects from a structure
bpy.ops.bim.reference_structure()

# Delete a spatial container (NOT IfcProject)
bpy.ops.bim.delete_container(container=storey.id())

# Copy selected objects to another container
bpy.ops.bim.copy_to_container(container=target_storey.id())

# Generate IfcSpace from surrounding walls
bpy.ops.bim.generate_space()

# Generate spaces from selected walls
bpy.ops.bim.generate_spaces_from_walls()

# Load spatial decomposition tree in UI
bpy.ops.bim.import_spatial_decomposition()

# Select all objects in a container
bpy.ops.bim.select_similar_container(is_recursive=True)

# Toggle IfcSpace visibility
bpy.ops.bim.toggle_spatial_elements(is_visible=True)
```

## Bonsai Three-Layer Architecture (Spatial Module)

```
DELIVERY LAYER — bim/module/spatial/operator.py
  BIM_OT_assign_container, BIM_OT_delete_container, etc.
  Calls: core.spatial.assign_container(tool.Ifc, tool.Spatial, ...)
  ─────────────────────────────────────────────────────────
DOMAIN LAYER — core/spatial.py
  assign_container(), reference_structure(), generate_space(), etc.
  Validates via: spatial.can_contain(), spatial.can_reference()
  Calls: ifc.run("spatial.assign_container", ...)
  ─────────────────────────────────────────────────────────
TOOL LAYER — tool/spatial.py
  Spatial.get_container(), Spatial.can_contain(), etc.
  Concrete implementation using bpy + ifcopenshell
```

### Operator Execution Pattern

```python
# bim/module/spatial/operator.py (Delivery Layer)
class BIM_OT_assign_container(bpy.types.Operator):
    bl_idname = "bim.assign_container"
    bl_label = "Assign Container"
    bl_options = {"REGISTER", "UNDO"}
    container: bpy.props.IntProperty()

    def execute(self, context):
        return IfcStore.execute_ifc_operator(self, context)

    def _execute(self, context):
        # Dependency injection: pass tool implementations to core
        core.spatial.assign_container(
            tool.Ifc, tool.Collector, tool.Spatial,
            container=tool.Ifc.get().by_id(self.container),
            objs=context.selected_objects,
        )
        return {"FINISHED"}
```

### Key Rules

1. Operators call `IfcStore.execute_ifc_operator(self, context)` — wraps IFC ops with undo.
2. Actual logic goes in `_execute()`, NOT `execute()`.
3. Tool classes (`tool.Ifc`, `tool.Spatial`) are passed as dependency injection.
4. `bl_options` ALWAYS includes `"UNDO"` for IFC-modifying operators.

## Spatial Validation Rules

### Containment Validation (`can_contain`)

An element can be contained in a spatial structure element ONLY if:
1. The container is an `IfcSpatialStructureElement` (IFC2X3) or also `IfcExternalSpatialStructureElement` (IFC4+)
2. The element has a `ContainedInStructure` inverse attribute

### Reference Validation (`can_reference`)

An element can reference a spatial structure ONLY if:
1. The structure is an `IfcSpatialStructureElement` (IFC2X3) or `IfcSpatialElement` (IFC4+)
2. The element has a `ReferencedInStructures` inverse attribute

## Default Project Structure

When creating a new IFC project via Bonsai UI (**File > New IFC Project**), Bonsai generates:

```
IfcProject  "My Project"
  └── IfcSite  "My Site"
        └── IfcBuilding  "My Building"
              └── IfcBuildingStorey  "My Storey"  (Elevation: 0.0)
```

The default container is set to `IfcBuildingStorey "My Storey"`.

## Space Generation

Bonsai provides two automated space generation methods:

### From Context (Surrounding Walls)
1. Set default container (an `IfcBuildingStorey`)
2. Position cursor or select an object at the space location
3. Call `bpy.ops.bim.generate_space()`
4. Bonsai bisects visible walls/columns at storey elevation, polygonizes boundaries, finds the enclosing polygon, and extrudes it to create an `IfcSpace`

### From Selected Walls
1. Select wall objects that enclose spaces
2. Call `bpy.ops.bim.generate_spaces_from_walls()`
3. Bonsai computes the union of wall footprints and creates `IfcSpace` elements from interior holes

## IFC Schema Version Differences

| Entity / Feature | IFC2X3 | IFC4 | IFC4X3 |
|-----------------|--------|------|--------|
| `IfcSite` | Yes | Yes | Yes |
| `IfcBuilding` | Yes | Yes | Yes |
| `IfcBuildingStorey` | Yes | Yes | Yes |
| `IfcSpace` | Yes | Yes | Yes |
| `IfcFacility` | No | No | Yes |
| `IfcFacilityPart` | No | No | Yes |
| `IfcSpatialElement` (base class) | No | Yes | Yes |
| `IfcExternalSpatialStructureElement` | No | Yes | Yes |
| Containment validation class | `IfcSpatialStructureElement` | + `IfcExternalSpatialStructureElement` | + `IfcExternalSpatialStructureElement` |
| Reference validation class | `IfcSpatialStructureElement` | `IfcSpatialElement` | `IfcSpatialElement` |

## Source Code Paths

| Component | Path in IfcOpenShell Monorepo |
|-----------|------------------------------|
| Core logic | `src/bonsai/bonsai/core/spatial.py` |
| Tool implementation | `src/bonsai/bonsai/tool/spatial.py` |
| Operators | `src/bonsai/bonsai/bim/module/spatial/operator.py` |
| UI panels | `src/bonsai/bonsai/bim/module/spatial/ui.py` |
| Properties | `src/bonsai/bonsai/bim/module/spatial/prop.py` |
| Interface contract | `src/bonsai/bonsai/core/tool.py` (class `Spatial`) |

## Dependencies

- **ifcos-syntax-api** — For `ifcopenshell.api.run()` invocation patterns and module reference
- **bonsai-core-architecture** — For three-layer architecture, IfcStore, tool.Ifc patterns

## Reference Links

- [Spatial API Method Signatures](references/methods.md) — Complete signatures for core, tool, and operator methods
- [Working Code Examples](references/examples.md) — End-to-end spatial structure examples
- [Anti-Patterns](references/anti-patterns.md) — Common spatial structure mistakes and how to avoid them
