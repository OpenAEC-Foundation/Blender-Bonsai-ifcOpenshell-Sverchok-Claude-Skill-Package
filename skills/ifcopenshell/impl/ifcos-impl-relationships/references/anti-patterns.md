# IFC Relationship Management — Anti-Patterns

Common mistakes when managing IFC relationships with IfcOpenShell. Each entry explains WHAT is wrong and WHY it fails.

---

## Anti-Pattern 1: Creating Relationship Entities Directly with `model.create_entity()`

```python
# WRONG — bypasses API logic, missing GlobalId and cleanup
rel = model.create_entity("IfcRelAggregates",
    RelatingObject=building,
    RelatedObjects=[storey])
```

**WHY:** `model.create_entity()` creates a raw IFC entity without generating a GlobalId, without attaching OwnerHistory (required in IFC2X3), without removing prior conflicting relationships, and without recalculating placements. The `ifcopenshell.api.run()` functions handle all of these automatically. For example, `aggregate.assign_object` removes prior aggregation/containment from products before reassigning, and recalculates placements relative to the new parent. Direct creation skips all of this and produces potentially invalid or inconsistent models.

```python
# CORRECT — use the API
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)
```

---

## Anti-Pattern 2: Containing an Element in Multiple Spatial Structures

```python
# WRONG — element can only have ONE spatial container
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_ground, products=[wall])
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_first, products=[wall])
# Second call REMOVES the first containment — wall is only in storey_first
```

**WHY:** The IFC schema constrains each element to exactly one `IfcRelContainedInSpatialStructure`. The `spatial.assign_container` API enforces this by removing any prior containment before assigning the new one. The second call does not add a second container — it replaces the first. If an element physically spans multiple storeys, use `spatial.assign_container` for the primary container and `spatial.reference_structure` for secondary references.

```python
# CORRECT — primary containment + secondary reference
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey_ground, products=[wall])
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[wall], relating_structure=storey_first)
```

---

## Anti-Pattern 3: Using IFC2X3 Inverse Attributes in IFC4 Models (or Vice Versa)

```python
# WRONG — IsTypedBy does not exist in IFC2X3
# (IFC2X3 model)
for rel in wall.IsTypedBy:  # AttributeError in IFC2X3
    print(rel.RelatingType.Name)
```

**WHY:** Inverse attribute names changed between IFC2X3 and IFC4. In IFC2X3, type assignment is accessed through `IsDefinedBy` (filtered for `IfcRelDefinesByType`). In IFC4+, a dedicated `IsTypedBy` inverse was added. Similarly, `IsNestedBy` exists only in IFC4+; in IFC2X3, nesting is accessed through `IsDecomposedBy` filtered for `IfcRelNests`. Using the wrong inverse attribute raises `AttributeError`.

```python
# CORRECT — use ifcopenshell.util.element for version-safe queries
import ifcopenshell.util.element
elem_type = ifcopenshell.util.element.get_type(wall)  # Works in all versions

# Or check schema explicitly
schema = wall.wrapped_data.file.schema
if schema == "IFC2X3":
    for rel in wall.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByType"):
            print(rel.RelatingType.Name)
else:
    for rel in wall.IsTypedBy:
        print(rel.RelatingType.Name)
```

---

## Anti-Pattern 4: Assigning Properties by Direct Attribute Modification

```python
# WRONG — IFC uses objectified relationships for properties
wall.IsExternal = True           # AttributeError: no such attribute
wall.FireRating = "REI90"        # AttributeError: no such attribute
```

**WHY:** IFC does not store properties as direct attributes on elements. Properties are stored in `IfcPropertySet` entities linked to elements via `IfcRelDefinesByProperties` relationships. This is the "objectified relationship" pattern — relationships are first-class entities in the IFC schema. To set properties, create a property set and edit its properties through the API.

```python
# CORRECT — use property set API
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "IsExternal": True,
    "FireRating": "REI90"
})
```

---

## Anti-Pattern 5: Forgetting to Remove Prior Relationships Before Reassignment

```python
# WRONG — manually creating a new IfcRelAggregates without cleanup
# Element already aggregated under building_A, now moving to building_B
new_rel = model.create_entity("IfcRelAggregates",
    RelatingObject=building_b,
    RelatedObjects=[storey])
# Old IfcRelAggregates under building_A still exists — dual aggregation
```

**WHY:** When using `model.create_entity()` directly, prior relationships are not cleaned up. The element ends up in two aggregation relationships simultaneously, which violates the IFC schema (an element can have at most one aggregate parent). The `ifcopenshell.api.run("aggregate.assign_object", ...)` function automatically removes prior aggregation, containment, and nesting before assigning the new parent.

```python
# CORRECT — API handles prior relationship cleanup
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building_b)
# Prior aggregation under building_A is automatically removed
```

---

## Anti-Pattern 6: Using `model.remove()` Instead of API Removal Functions

```python
# WRONG — leaves orphaned relationships
model.remove(wall)
# IfcRelContainedInSpatialStructure still references the deleted wall
# IfcRelDefinesByType still references the deleted wall
# IfcRelDefinesByProperties still references the deleted wall
```

**WHY:** `model.remove()` deletes only the specified entity without cleaning up its relationships. This leaves orphaned `IfcRelationship` entities that reference a now-deleted element, producing an invalid IFC file. The `root.remove_product` API recursively cleans up all associated relationships (containment, aggregation, type assignment, property sets, materials, geometry, placement, nesting, and voids).

```python
# CORRECT — use root.remove_product for safe deletion
ifcopenshell.api.run("root.remove_product", model, product=wall)
```

---

## Anti-Pattern 7: Using `spatial.reference_structure` in IFC2X3

```python
# WRONG — IfcRelReferencedInSpatialStructure does not exist in IFC2X3
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
# ... create spatial hierarchy ...
ifcopenshell.api.run("spatial.reference_structure", model,
    products=[column], relating_structure=storey_first)  # Schema error
```

**WHY:** `IfcRelReferencedInSpatialStructure` was introduced in IFC4. It does not exist in the IFC2X3 schema. Attempting to create this relationship in an IFC2X3 model raises a schema validation error. In IFC2X3, an element can only be contained in one spatial structure — there is no mechanism for secondary references.

```python
# CORRECT — check schema before using IFC4+ features
if model.schema == "IFC4" or model.schema == "IFC4X3":
    ifcopenshell.api.run("spatial.reference_structure", model,
        products=[column], relating_structure=storey_first)
```

---

## Anti-Pattern 8: Querying Relationships Without Using `ifcopenshell.util.element`

```python
# WRONG — version-dependent, verbose, error-prone
def get_wall_type(wall):
    for rel in wall.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByType"):
            return rel.RelatingType
    return None

# Breaks in IFC4+ where IsTypedBy is the correct inverse
```

**WHY:** Writing manual inverse-attribute traversal code is verbose, fragile, and version-dependent. The inverse attribute names and entity hierarchies changed between IFC2X3 and IFC4. `ifcopenshell.util.element` provides version-safe query functions that handle these differences internally: `get_type()`, `get_container()`, `get_material()`, `get_psets()`, `get_decomposition()`, `get_aggregate()`, and `get_nest()`.

```python
# CORRECT — use ifcopenshell.util.element
import ifcopenshell.util.element

elem_type = ifcopenshell.util.element.get_type(wall)
container = ifcopenshell.util.element.get_container(wall)
material = ifcopenshell.util.element.get_material(wall)
psets = ifcopenshell.util.element.get_psets(wall)
```

---

## Anti-Pattern 9: Nesting Elements That Should Be Aggregated

```python
# WRONG — stair flights should be aggregated, not nested
stair = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcStair", name="Main Stair")
flight = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcStairFlight", name="Flight 1")
ifcopenshell.api.run("nest.assign_object", model,
    related_objects=[flight], relating_object=stair)  # Wrong relationship
```

**WHY:** IFC distinguishes between aggregation (whole/part decomposition) and nesting (host/component attachment). Physical assemblies like stairs (= flights + landings + railings), curtain walls (= panels + mullions), and roofs (= slabs + beams) use `IfcRelAggregates`. Nesting (`IfcRelNests`) is for components attached at connection points, like a faucet on a sink or a sensor on equipment. Using the wrong relationship type produces a semantically incorrect model that BIM tools may misinterpret.

```python
# CORRECT — use aggregation for physical assemblies
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[flight], relating_object=stair)
```
