# Bonsai Element Access — Code Examples

> **Version**: Bonsai v0.8.4 | Blender 4.2.0+ | Python 3.11
> ALL examples require Blender context. Run via `blender --python script.py` or from Blender's Python console.

---

## Example 1: Get Active Object's IFC Data

```python
"""
Get the IFC class, name, GlobalId, and properties of the active Blender object.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bpy
import bonsai.tool as tool
import ifcopenshell.util.element

obj = bpy.context.active_object
if obj is None:
    print("No active object")
else:
    entity = tool.Ifc.get_entity(obj)
    if entity is None:
        print(f"{obj.name} is not linked to IFC")
    else:
        print(f"Blender name:  {obj.name}")
        print(f"IFC class:     {entity.is_a()}")
        print(f"IFC Name:      {entity.Name}")
        print(f"GlobalId:      {entity.GlobalId}")
        print(f"Step ID:       {entity.id()}")

        # Get all property sets
        psets = ifcopenshell.util.element.get_psets(entity)
        for pset_name, props in psets.items():
            print(f"\n  {pset_name}:")
            for prop_name, value in props.items():
                if prop_name != "id":  # Skip internal id key
                    print(f"    {prop_name}: {value}")
```

---

## Example 2: List All Walls with Their Storeys

```python
"""
List all IfcWall entities in the model with their spatial container (storey).
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool
import ifcopenshell.util.element

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

walls = model.by_type("IfcWall")
print(f"Found {len(walls)} walls:\n")

for wall in walls:
    container = ifcopenshell.util.element.get_container(wall)
    container_name = container.Name if container else "(unassigned)"

    wall_type = ifcopenshell.util.element.get_type(wall)
    type_name = wall_type.Name if wall_type else "(no type)"

    obj = tool.Ifc.get_object(wall)
    blender_name = obj.name if obj else "(no Blender object)"

    print(f"  {wall.Name or '(unnamed)'}")
    print(f"    Type:    {type_name}")
    print(f"    Storey:  {container_name}")
    print(f"    Blender: {blender_name}")
    print(f"    GID:     {wall.GlobalId}")
    print()
```

---

## Example 3: Select All Elements of a Specific IFC Class

```python
"""
Select all IfcDoor objects in the Blender viewport.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bpy
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Deselect everything
bpy.ops.object.select_all(action='DESELECT')

# Select all doors
ifc_class = "IfcDoor"
count = 0
for entity in model.by_type(ifc_class):
    obj = tool.Ifc.get_object(entity)
    if obj:
        obj.select_set(True)
        count += 1

print(f"Selected {count} {ifc_class} objects")

# Optionally set the first one as active
if count > 0:
    bpy.context.view_layer.objects.active = bpy.context.selected_objects[0]
```

---

## Example 4: Build a Storey Element Report

```python
"""
Report all elements per storey with IFC class and property summary.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool
import ifcopenshell.util.element

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

storeys = model.by_type("IfcBuildingStorey")
print(f"Model has {len(storeys)} storeys\n")

for storey in storeys:
    elements = ifcopenshell.util.element.get_decomposition(storey)
    print(f"=== {storey.Name} ({len(elements)} elements) ===")

    # Count by class
    class_counts = {}
    for el in elements:
        cls = el.is_a()
        class_counts[cls] = class_counts.get(cls, 0) + 1

    for cls, count in sorted(class_counts.items()):
        print(f"  {cls}: {count}")
    print()
```

---

## Example 5: Find Element by GlobalId and Access Its Data

```python
"""
Look up an element by GlobalId and print its full data.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool
import ifcopenshell.util.element
from bonsai.bim.ifc import IfcStore

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

target_guid = "2O2Fr$t4X7Zf8NOew3FLOH"  # Replace with actual GlobalId

try:
    entity = model.by_guid(target_guid)
except RuntimeError:
    print(f"No entity with GlobalId {target_guid}")
    entity = None

if entity:
    print(f"Found: {entity.is_a()} — {entity.Name}")
    print(f"Step ID: {entity.id()}")

    # Get Blender object
    obj = tool.Ifc.get_object(entity)
    if obj:
        print(f"Blender object: {obj.name}")
        print(f"Location: {tuple(obj.location)}")
    else:
        print("No Blender representation")

    # Get container
    container = ifcopenshell.util.element.get_container(entity)
    if container:
        print(f"Container: {container.Name} ({container.is_a()})")

    # Get type
    etype = ifcopenshell.util.element.get_type(entity)
    if etype:
        print(f"Type: {etype.Name}")

    # Get material
    material = ifcopenshell.util.element.get_material(entity)
    if material:
        print(f"Material: {material.Name} ({material.is_a()})")
```

---

## Example 6: Iterate IfcStore.id_map for Batch Processing

```python
"""
Process all IFC-linked Blender objects using IfcStore.id_map.
Fastest method for iterating all linked objects.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool
from bonsai.bim.ifc import IfcStore

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

print(f"Total linked objects: {len(IfcStore.id_map)}\n")

# Build summary by IFC class
class_summary = {}
for ifc_id, obj in IfcStore.id_map.items():
    try:
        entity = model.by_id(ifc_id)
        cls = entity.is_a()
        if cls not in class_summary:
            class_summary[cls] = []
        class_summary[cls].append((entity.Name, obj.name))
    except RuntimeError:
        print(f"  WARNING: stale id_map entry #{ifc_id} → {obj.name}")

for cls in sorted(class_summary.keys()):
    items = class_summary[cls]
    print(f"{cls} ({len(items)}):")
    for ifc_name, bl_name in items[:5]:  # Show first 5
        print(f"  IFC: {ifc_name or '(unnamed)'} → Blender: {bl_name}")
    if len(items) > 5:
        print(f"  ... and {len(items) - 5} more")
    print()
```

---

## Example 7: Check Which Selected Objects Are IFC-Linked

```python
"""
Check the IFC link status of all selected Blender objects.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bpy
import bonsai.tool as tool

linked = []
unlinked = []

for obj in bpy.context.selected_objects:
    entity = tool.Ifc.get_entity(obj)
    if entity:
        linked.append((obj.name, entity.is_a(), entity.Name))
    else:
        unlinked.append(obj.name)

print(f"IFC-linked objects ({len(linked)}):")
for bl_name, ifc_class, ifc_name in linked:
    print(f"  {bl_name} → {ifc_class}: {ifc_name}")

if unlinked:
    print(f"\nNon-IFC objects ({len(unlinked)}):")
    for name in unlinked:
        print(f"  {name}")
```

---

## Example 8: Navigate Spatial Hierarchy

```python
"""
Print the full spatial hierarchy tree of the loaded IFC model.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool
import ifcopenshell.util.element

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

def print_hierarchy(element, indent=0):
    prefix = "  " * indent
    obj = tool.Ifc.get_object(element)
    bl_name = f" [{obj.name}]" if obj else ""
    print(f"{prefix}{element.is_a()}: {element.Name}{bl_name}")

    # Get children via aggregation (spatial decomposition)
    if hasattr(element, "IsDecomposedBy"):
        for rel in element.IsDecomposedBy:
            for child in rel.RelatedObjects:
                print_hierarchy(child, indent + 1)

    # Get contained elements (physical elements in this spatial zone)
    if hasattr(element, "ContainsElements"):
        for rel in element.ContainsElements:
            for el in rel.RelatedElements:
                obj = tool.Ifc.get_object(el)
                bl_name = f" [{obj.name}]" if obj else ""
                print(f"{'  ' * (indent + 1)}├── {el.is_a()}: {el.Name}{bl_name}")

# Start from IfcProject
projects = model.by_type("IfcProject")
for project in projects:
    print_hierarchy(project)
```

---

## Example 9: Get Property Value Using tool.Pset

```python
"""
Read a specific property value from an element using tool.Pset.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bpy
import bonsai.tool as tool

obj = bpy.context.active_object
entity = tool.Ifc.get_entity(obj)

if entity is None:
    print("Active object is not linked to IFC")
else:
    # Method A: Using tool.Pset (Bonsai tool layer)
    pset = tool.Pset.get_element_pset(entity, "Pset_WallCommon")
    if pset:
        print(f"IsExternal:  {pset.get('IsExternal')}")
        print(f"LoadBearing: {pset.get('LoadBearing')}")
        print(f"FireRating:  {pset.get('FireRating')}")
    else:
        print("Pset_WallCommon not found on this element")

    # Method B: Using ifcopenshell.util.element (standalone-compatible)
    import ifcopenshell.util.element
    all_psets = ifcopenshell.util.element.get_psets(entity)
    wall_common = all_psets.get("Pset_WallCommon", {})
    print(f"FireRating (via util): {wall_common.get('FireRating')}")
```

---

## Example 10: Safe Element Lookup with Error Handling

```python
"""
Safely look up elements by various identifiers with proper error handling.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool
from bonsai.bim.ifc import IfcStore

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Safe lookup by step ID
def get_entity_by_id(step_id):
    """Returns entity or None. NEVER raises."""
    try:
        return model.by_id(step_id)
    except RuntimeError:
        return None

# Safe lookup by GlobalId
def get_entity_by_guid(guid):
    """Returns entity or None. NEVER raises."""
    try:
        return model.by_guid(guid)
    except RuntimeError:
        return None

# Safe lookup of Blender object for entity
def get_blender_object(entity):
    """Returns Blender object or None."""
    return tool.Ifc.get_object(entity)

# Safe lookup of entity for Blender object
def get_ifc_entity(obj):
    """Returns IFC entity or None."""
    return tool.Ifc.get_entity(obj)

# Usage
entity = get_entity_by_id(42)
if entity:
    obj = get_blender_object(entity)
    print(f"Entity #{entity.id()} → {entity.is_a()}")
    if obj:
        print(f"  Blender object: {obj.name}")
else:
    print("Entity #42 not found")
```

---

## Example 11: Find Elements Without Blender Representation

```python
"""
Find IFC elements that exist in the model but have no Blender scene object.
Useful for diagnosing missing geometry or unlinked elements.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

orphaned = []
for element in model.by_type("IfcElement"):
    obj = tool.Ifc.get_object(element)
    if obj is None:
        orphaned.append(element)

print(f"Elements without Blender representation: {len(orphaned)}")
for el in orphaned[:20]:
    container_info = ""
    if hasattr(el, "ContainedInStructure") and el.ContainedInStructure:
        container = el.ContainedInStructure[0].RelatingStructure
        container_info = f" in {container.Name}"
    print(f"  {el.is_a()}: {el.Name or '(unnamed)'}{container_info} (#{el.id()})")

if len(orphaned) > 20:
    print(f"  ... and {len(orphaned) - 20} more")
```

---

## Example 12: Cross-Reference Entities Using Inverse Attributes

```python
"""
Navigate IFC inverse relationships to find related entities.
Bonsai v0.8.x | Blender 4.2.0+
"""
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

wall = model.by_type("IfcWall")[0]
print(f"Wall: {wall.Name} (#{wall.id()})")

# Openings (voids) in this wall
if hasattr(wall, "HasOpenings"):
    for rel in wall.HasOpenings:
        opening = rel.RelatedOpeningElement
        print(f"  Opening: {opening.Name} (#{opening.id()})")

        # Check if opening is filled (e.g., with a door/window)
        if hasattr(opening, "HasFillings"):
            for fill_rel in opening.HasFillings:
                filling = fill_rel.RelatedBuildingElement
                print(f"    Filling: {filling.is_a()} — {filling.Name}")

# Spatial containment
if hasattr(wall, "ContainedInStructure") and wall.ContainedInStructure:
    container = wall.ContainedInStructure[0].RelatingStructure
    print(f"  Contained in: {container.is_a()} — {container.Name}")

# Type definition
if hasattr(wall, "IsTypedBy") and wall.IsTypedBy:
    wall_type = wall.IsTypedBy[0].RelatingType
    print(f"  Type: {wall_type.Name}")

# Material association
if hasattr(wall, "HasAssociations"):
    for assoc in wall.HasAssociations:
        if assoc.is_a("IfcRelAssociatesMaterial"):
            mat = assoc.RelatingMaterial
            print(f"  Material: {mat.is_a()} — {mat.Name}")
```
