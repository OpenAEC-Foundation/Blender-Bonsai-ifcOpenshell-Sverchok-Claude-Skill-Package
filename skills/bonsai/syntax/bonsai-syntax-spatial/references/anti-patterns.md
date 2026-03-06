# Bonsai Spatial Structure — Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+
> Common mistakes when working with spatial structure in Bonsai and IfcOpenShell.

---

## AP-1: Passing Single Element Instead of List (v0.8+ Breaking Change)

### Wrong

```python
# WRONG — IfcOpenShell v0.8+ expects products as a list
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)
# Raises: TypeError or KeyError: 'product'

ifcopenshell.api.run("aggregate.assign_object", model,
    product=site, relating_object=project)
# Raises: TypeError or KeyError: 'product'
```

### Correct

```python
# CORRECT — ALWAYS pass products as a list in v0.8+
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
```

### Why

IfcOpenShell v0.8+ changed parameter names from singular (`product`) to plural (`products`) accepting a list. This affects `spatial.assign_container`, `aggregate.assign_object`, `type.assign_type`, `material.assign_material`, `spatial.reference_structure`, and many other relationship functions.

---

## AP-2: Using Containment Instead of Reference for Multi-Storey Elements

### Wrong

```python
# WRONG — An element can only be contained in ONE container
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")

# Assign to ground floor
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=ground_floor)

# This MOVES the column from ground floor to first floor (not what was intended)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=first_floor)
```

### Correct

```python
# CORRECT — Use containment for primary location, reference for additional
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C1")

# Primary containment in ground floor
ifcopenshell.api.run("spatial.assign_container", model,
    products=[column], relating_structure=ground_floor)

# Reference in first floor (non-exclusive, multi-storey)
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[column], relating_structure=first_floor)
```

### Why

`IfcRelContainedInSpatialStructure` is exclusive — an element can only be in ONE container. Assigning to a new container silently removes it from the old one. Use `IfcRelReferencedInSpatialStructure` for elements that span or are relevant to multiple spatial zones.

---

## AP-3: Creating Spatial Elements Without Connecting to Hierarchy

### Wrong

```python
# WRONG — Creates orphan spatial elements with no hierarchy connection
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="My Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# Elements exist but are NOT connected — invalid IFC model
# No IfcRelAggregates relationships created
```

### Correct

```python
# CORRECT — ALWAYS connect spatial elements via IfcRelAggregates
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="My Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# Connect the hierarchy
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
```

### Why

IFC requires a complete spatial decomposition tree rooted at `IfcProject`. Orphan spatial elements (not connected via `IfcRelAggregates`) result in invalid IFC files that fail validation and cannot be properly navigated in BIM viewers.

---

## AP-4: Using model.create_entity() Instead of API for Spatial Elements

### Wrong

```python
# WRONG — model.create_entity() skips GlobalId, ownership, and validation
site = model.create_entity("IfcSite", Name="My Site")
# No GlobalId generated, no OwnerHistory set, no predefined type validation
```

### Correct

```python
# CORRECT — ALWAYS use root.create_entity for spatial elements
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="My Site")
# Automatically generates GlobalId, sets OwnerHistory, validates predefined type
```

### Why

`model.create_entity()` is a low-level function that creates a raw IFC entity without setting required attributes like `GlobalId` and `OwnerHistory`. The `root.create_entity` API function handles all of this automatically. Using the low-level function produces non-compliant IFC files.

---

## AP-5: Directly Manipulating Spatial Attributes Instead of Using API

### Wrong

```python
# WRONG — Direct attribute manipulation bypasses relationship management
storey.Name = "Renamed Floor"
storey.Elevation = 3.5
```

### Correct

```python
# CORRECT — Use attribute.edit_attributes API
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=storey, attributes={"Name": "Renamed Floor", "Elevation": 3.5})
```

### Why

Direct attribute manipulation bypasses the IfcOpenShell API's relationship management, event system, and undo support. In Bonsai, direct changes are NOT tracked by `IfcStore` and will NOT be synced to the Blender UI.

---

## AP-6: Deleting Spatial Elements with model.remove() Instead of API

### Wrong

```python
# WRONG — model.remove() leaves dangling relationships
model.remove(storey)
# IfcRelAggregates, IfcRelContainedInSpatialStructure still reference
# the deleted storey — corrupted IFC graph
```

### Correct

```python
# CORRECT — Use root.remove_product to clean up all relationships
ifcopenshell.api.run("root.remove_product", model, product=storey)
# Removes IfcRelAggregates, IfcRelContainedInSpatialStructure, etc.
```

### Why

`model.remove()` only removes the entity itself, leaving all inverse relationships pointing to a deleted entity. `root.remove_product` cleans up ALL relationships before removing the entity.

---

## AP-7: Trying to Delete IfcProject

### Wrong

```python
# WRONG — IfcProject is the root of the model and cannot be deleted
ifcopenshell.api.run("root.remove_product", model, product=project)
# OR
bpy.ops.bim.delete_container(container=project.id())
# Bonsai operator poll() returns False — nothing happens
```

### Correct

```python
# CORRECT — IfcProject must always exist. Delete children, not the root.
# To delete a building:
bpy.ops.bim.delete_container(container=building.id())
```

### Why

Every valid IFC file requires exactly one `IfcProject`. Bonsai's operator poll method explicitly blocks deletion of `IfcProject`. If you need to start over, create a new IFC file instead.

---

## AP-8: Placing Elements Without Setting a Default Container First

### Wrong (Bonsai context)

```python
# WRONG — No default container set, element has no spatial context
bpy.ops.bim.generate_space()
# Raises SpaceGenerationError: "Set a default container to generate spaces"
```

### Correct

```python
# CORRECT — ALWAYS set default container before creating elements
storey = model.by_type("IfcBuildingStorey")[0]
bpy.ops.bim.set_default_container(container=storey.id())

# Now space generation works
bpy.ops.bim.generate_space()
```

### Why

Bonsai requires a default container (typically an `IfcBuildingStorey`) to know where new elements belong. Space generation specifically requires this context to determine the cutting plane elevation and boundary detection.

---

## AP-9: Skipping Hierarchy Levels (e.g., Building Directly Under Project)

### Wrong

```python
# WRONG — Skipping IfcSite (technically valid but poor practice)
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="My Building")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=project)
# Some IFC validators and viewers expect the standard hierarchy
```

### Correct

```python
# CORRECT — Follow the standard hierarchy: Project > Site > Building > Storey
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Default Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="My Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
```

### Why

While IFC technically allows flexible aggregation, the standard hierarchy (Project > Site > Building > Storey) is expected by most BIM tools and validators. Skipping levels causes interoperability issues. Bonsai's `guess_default_container()` traverses this specific chain and will fail to find a default container if levels are skipped.

---

## AP-10: Using blenderbim.* Imports Instead of bonsai.*

### Wrong

```python
# WRONG — blenderbim was renamed to bonsai in v0.8.0 (2024)
from blenderbim.bim.ifc import IfcStore
import blenderbim.core.spatial as core_spatial
# ModuleNotFoundError in Bonsai v0.8.x
```

### Correct

```python
# CORRECT — Use bonsai.* for all imports (v0.8.0+)
from bonsai.bim.ifc import IfcStore
import bonsai.core.spatial as core_spatial
```

### Why

Bonsai was renamed from BlenderBIM in 2024 (v0.8.0). All module paths changed from `blenderbim.*` to `bonsai.*`. The old paths no longer exist and will raise `ModuleNotFoundError`.
