# Relationship API Method Signatures

Complete API reference for IfcOpenShell relationship management methods. All signatures verified against the [official IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/).

---

## ifcopenshell.api.spatial Module

### spatial.assign_container()

Assigns physical elements to be contained in a spatial structure element. Creates an `IfcRelContainedInSpatialStructure` relationship.

```python
ifcopenshell.api.run("spatial.assign_container", file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of IfcElements to contain |
| `relating_structure` | `entity_instance` | The IfcSpatialStructureElement container (IfcSite, IfcBuilding, IfcBuildingStorey, IfcSpace, or IFC4X3 facility types) |

**Returns:** `IfcRelContainedInSpatialStructure` instance, or `None` if products list is empty.

**Behavior:** Removes any prior spatial containment from the products before assigning the new container.

---

### spatial.unassign_container()

Removes spatial containment relationships from products.

```python
ifcopenshell.api.run("spatial.unassign_container", file,
    products: list[ifcopenshell.entity_instance]
) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of IfcProducts to remove containment from |

---

### spatial.reference_structure()

Establishes non-hierarchical spatial references for elements that span multiple spatial structures. Creates an `IfcRelReferencedInSpatialStructure` relationship. **IFC4+ only.**

```python
ifcopenshell.api.run("spatial.reference_structure", file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of IfcElements to reference |
| `relating_structure` | `entity_instance` | The IfcSpatialStructureElement to reference |

**Returns:** `IfcRelReferencedInSpatialStructure` instance, or `None` if products list is empty.

---

### spatial.dereference_structure()

Removes spatial references between products and a spatial structure element.

```python
ifcopenshell.api.run("spatial.dereference_structure", file,
    products: list[ifcopenshell.entity_instance],
    relating_structure: ifcopenshell.entity_instance
) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of IfcElements to dereference |
| `relating_structure` | `entity_instance` | The IfcSpatialStructureElement to dereference from |

---

## ifcopenshell.api.aggregate Module

### aggregate.assign_object()

Establishes an aggregate (whole/part) relationship. Creates an `IfcRelAggregates` relationship.

```python
ifcopenshell.api.run("aggregate.assign_object", file,
    products: list[ifcopenshell.entity_instance],
    relating_object: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of child parts (IfcElement or IfcSpatialStructureElement subtypes) |
| `relating_object` | `entity_instance` | The parent whole element |

**Returns:** `IfcRelAggregates` instance, or `None` if products list is empty.

**Behavior:**
- Recalculates product placements to be relative to the parent
- Removes any prior aggregation, containment, or nesting relationships from the products

---

### aggregate.unassign_object()

Removes aggregation relationships from specified products.

```python
ifcopenshell.api.run("aggregate.unassign_object", file,
    products: list[ifcopenshell.entity_instance]
) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of elements to remove from aggregation |

---

## ifcopenshell.api.type Module

### type.assign_type()

Assigns a type definition to element occurrences. Creates an `IfcRelDefinesByType` relationship.

```python
ifcopenshell.api.run("type.assign_type", file,
    related_objects: list[ifcopenshell.entity_instance],
    relating_type: ifcopenshell.entity_instance,
    should_map_representations: bool = True
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `related_objects` | `list[entity_instance]` | *required* | List of IfcElement occurrences to assign |
| `relating_type` | `entity_instance` | *required* | The IfcElementType to assign |
| `should_map_representations` | `bool` | `True` | Whether to map representations from type to occurrences |

**Returns:** `IfcRelDefinesByType` instance, or `None` if related_objects is empty.

**Behavior:** Occurrences inherit properties, materials, and geometric representations from their assigned type.

---

### type.unassign_type()

Removes type assignment from occurrences.

```python
ifcopenshell.api.run("type.unassign_type", file,
    related_objects: list[ifcopenshell.entity_instance]
) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `related_objects` | `list[entity_instance]` | List of IfcElement occurrences to unassign |

**Note:** Unassignment does NOT automatically remove mapped representations or material usages previously associated with the type.

---

### type.map_type_representations()

Synchronizes occurrence representations with their type's representation.

```python
ifcopenshell.api.run("type.map_type_representations", file,
    related_object: ifcopenshell.entity_instance,
    relating_type: ifcopenshell.entity_instance
) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `related_object` | `entity_instance` | Single IfcElement occurrence |
| `relating_type` | `entity_instance` | The IfcElementType reference |

---

## ifcopenshell.api.pset Module

### pset.add_pset()

Creates a new property set attached to a product. Automatically creates the `IfcRelDefinesByProperties` relationship.

```python
ifcopenshell.api.run("pset.add_pset", file,
    product: ifcopenshell.entity_instance,
    name: str,
    ifc2x3_subclass: str | None = None
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `product` | `entity_instance` | *required* | The IfcObject or IfcTypeObject receiving the property set |
| `name` | `str` | *required* | Property set name (standardized: "Pset_*" prefix) |
| `ifc2x3_subclass` | `str \| None` | `None` | Optional IFC2X3 property set subclass |

**Returns:** The created `IfcPropertySet` instance.

---

### pset.edit_pset()

Modifies property values within a property set.

```python
ifcopenshell.api.run("pset.edit_pset", file,
    pset: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict | None = None,
    pset_template: ifcopenshell.entity_instance | None = None,
    should_purge: bool = True
) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `pset` | `entity_instance` | *required* | The IfcPropertySet to edit |
| `name` | `str \| None` | `None` | New name for the property set |
| `properties` | `dict \| None` | `None` | Dict of property name → value. `None` values delete the property if `should_purge=True` |
| `pset_template` | `entity_instance \| None` | `None` | Optional template for type validation |
| `should_purge` | `bool` | `True` | Whether to remove properties set to `None` |

---

### pset.assign_pset()

Assigns an existing property set to additional products.

```python
ifcopenshell.api.run("pset.assign_pset", file,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | `list[entity_instance]` | List of IfcObjects to assign the property set to |
| `pset` | `entity_instance` | The existing IfcPropertySet |

**Returns:** `IfcRelDefinesByProperties` instance, or `None`.

---

### pset.remove_pset()

Deletes a property set and its relationship from a product.

```python
ifcopenshell.api.run("pset.remove_pset", file,
    product: ifcopenshell.entity_instance,
    pset: ifcopenshell.entity_instance
) -> None
```

---

### pset.add_qto()

Creates a new quantity set attached to a product.

```python
ifcopenshell.api.run("pset.add_qto", file,
    product: ifcopenshell.entity_instance,
    name: str
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `product` | `entity_instance` | The IfcObject receiving the quantity set |
| `name` | `str` | Quantity set name (standardized: "Qto_*" prefix) |

**Returns:** The created `IfcElementQuantity` instance.

---

### pset.edit_qto()

Modifies quantity values within a quantity set.

```python
ifcopenshell.api.run("pset.edit_qto", file,
    qto: ifcopenshell.entity_instance,
    name: str | None = None,
    properties: dict | None = None,
    pset_template: ifcopenshell.entity_instance | None = None
) -> None
```

---

## ifcopenshell.api.material Module

### material.add_material()

Creates a new material entity.

```python
ifcopenshell.api.run("material.add_material", file,
    name: str = "Unnamed",
    category: str | None = None,
    description: str | None = None
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | `"Unnamed"` | Material name |
| `category` | `str \| None` | `None` | Material category (IFC4+ only) |
| `description` | `str \| None` | `None` | Material description (IFC4+ only) |

**Returns:** `IfcMaterial` instance.

---

### material.assign_material()

Associates a material with products. Creates an `IfcRelAssociatesMaterial` relationship.

```python
ifcopenshell.api.run("material.assign_material", file,
    products: list[ifcopenshell.entity_instance],
    type: str = "IfcMaterial",
    material: ifcopenshell.entity_instance | None = None
) -> ifcopenshell.entity_instance | list | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `products` | `list[entity_instance]` | *required* | List of IfcElements or IfcElementTypes |
| `type` | `str` | `"IfcMaterial"` | Material type to assign |
| `material` | `entity_instance \| None` | `None` | Existing material entity to assign |

**Returns:** `IfcRelAssociatesMaterial` instance(s), or `None`.

---

### material.unassign_material()

Removes material assignments from products.

```python
ifcopenshell.api.run("material.unassign_material", file,
    products: list[ifcopenshell.entity_instance]
) -> None
```

---

### material.add_material_set()

Creates a material set (layer set, profile set, or constituent set).

```python
ifcopenshell.api.run("material.add_material_set", file,
    name: str = "Unnamed",
    set_type: str = "IfcMaterialConstituentSet"
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | `"Unnamed"` | Set name |
| `set_type` | `str` | `"IfcMaterialConstituentSet"` | One of: `"IfcMaterialLayerSet"`, `"IfcMaterialProfileSet"`, `"IfcMaterialConstituentSet"` |

---

### material.add_layer()

Adds a layer to a material layer set.

```python
ifcopenshell.api.run("material.add_layer", file,
    layer_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

**Returns:** `IfcMaterialLayer` instance.

---

### material.add_profile()

Adds a profile to a material profile set. **IFC4+ only.**

```python
ifcopenshell.api.run("material.add_profile", file,
    profile_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance | None = None,
    profile: ifcopenshell.entity_instance | None = None,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

**Returns:** `IfcMaterialProfile` instance.

---

### material.add_constituent()

Adds a constituent to a material constituent set. **IFC4+ only.**

```python
ifcopenshell.api.run("material.add_constituent", file,
    constituent_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

**Returns:** `IfcMaterialConstituent` instance.

---

## ifcopenshell.api.void Module

### void.add_opening()

Creates a void in an element. Creates an `IfcRelVoidsElement` relationship.

```python
ifcopenshell.api.run("void.add_opening", file,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `opening` | `entity_instance` | The IfcOpeningElement defining the void shape |
| `element` | `entity_instance` | The host IfcElement to cut the void in |

**Returns:** `IfcRelVoidsElement` instance.

---

### void.add_filling()

Places an element into an opening. Creates an `IfcRelFillsElement` relationship.

```python
ifcopenshell.api.run("void.add_filling", file,
    opening: ifcopenshell.entity_instance,
    element: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `opening` | `entity_instance` | The IfcOpeningElement to fill |
| `element` | `entity_instance` | The IfcElement filling the opening (e.g., IfcDoor, IfcWindow) |

**Returns:** `IfcRelFillsElement` instance.

---

### void.remove_opening()

Removes an opening from its host element. Deletes the `IfcRelVoidsElement` and the `IfcOpeningElement`.

```python
ifcopenshell.api.run("void.remove_opening", file,
    opening: ifcopenshell.entity_instance
) -> None
```

---

### void.remove_filling()

Removes a filling from its opening. Deletes the `IfcRelFillsElement`.

```python
ifcopenshell.api.run("void.remove_filling", file,
    element: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.nest Module

### nest.assign_object()

Creates a nesting relationship where child objects attach to a parent host. Creates an `IfcRelNests` relationship.

```python
ifcopenshell.api.run("nest.assign_object", file,
    related_objects: list[ifcopenshell.entity_instance],
    relating_object: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `related_objects` | `list[entity_instance]` | List of child objects to nest |
| `relating_object` | `entity_instance` | The parent host element |

**Returns:** `IfcRelNests` instance, or `None` if related_objects is empty.

**Behavior:**
- Removes prior aggregation, containment, or nesting relationships from the children
- Recalculates placement relative to parent hierarchy
- When the parent moves, nested children move with it

---

### nest.unassign_object()

Removes nesting relationships from child objects.

```python
ifcopenshell.api.run("nest.unassign_object", file,
    related_objects: list[ifcopenshell.entity_instance]
) -> None
```

---

## ifcopenshell.api.group Module

### group.add_group()

Creates a new logical group.

```python
ifcopenshell.api.run("group.add_group", file,
    name: str = "Unnamed",
    description: str | None = None
) -> ifcopenshell.entity_instance
```

**Returns:** `IfcGroup` instance.

---

### group.assign_group()

Assigns products to a group. Creates an `IfcRelAssignsToGroup` relationship.

```python
ifcopenshell.api.run("group.assign_group", file,
    products: list[ifcopenshell.entity_instance],
    group: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```

---

### group.unassign_group()

Removes products from a group.

```python
ifcopenshell.api.run("group.unassign_group", file,
    products: list[ifcopenshell.entity_instance],
    group: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.util.element — Relationship Query Utilities

Version-safe query functions that handle IFC2X3/IFC4/IFC4X3 differences internally.

| Function | Returns | Description |
|----------|---------|-------------|
| `get_container(element)` | `entity_instance \| None` | Spatial container of an element |
| `get_type(element)` | `entity_instance \| None` | Type definition of an element |
| `get_types(type_element)` | `list[entity_instance]` | All occurrences of a type |
| `get_material(element)` | `entity_instance \| None` | Material assigned to an element |
| `get_psets(element)` | `dict` | All property sets as `{name: {prop: value}}` |
| `get_decomposition(element)` | `list[entity_instance]` | All children in decomposition hierarchy |
| `get_aggregate(element)` | `entity_instance \| None` | Parent aggregate of an element |
| `get_nest(element)` | `entity_instance \| None` | Parent nesting host of an element |
