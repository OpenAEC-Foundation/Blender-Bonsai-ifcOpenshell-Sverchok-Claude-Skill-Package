# Material API Method Signatures

Complete API reference for IfcOpenShell material operations. All signatures verified against the [official IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/material/index.html).

---

## ifcopenshell.api.material.add_material()

Creates an IfcMaterial definition representing a physical material.

```python
ifcopenshell.api.material.add_material(
    file: ifcopenshell.file,
    name: str | None = None,
    category: str | None = None,
    description: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `name` | `str \| None` | `None` | Material name (e.g., "Concrete C30/37") |
| `category` | `str \| None` | `None` | Standard category: concrete, steel, aluminium, block, brick, stone, wood, glass, gypsum, plastic, earth |
| `description` | `str \| None` | `None` | Human-readable description (IFC4+ only, ignored in IFC2X3) |

### Return Value

`ifcopenshell.entity_instance` — The created IfcMaterial entity.

### Notes

- Category and description attributes exist only in IFC4+. They are silently ignored in IFC2X3.
- Material names SHOULD be unique within a file for clarity but IFC does not enforce uniqueness.

---

## ifcopenshell.api.material.assign_material()

Associates a material or material set with one or more products via IfcRelAssociatesMaterial.

```python
ifcopenshell.api.material.assign_material(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    type: str = "IfcMaterial",
    material: ifcopenshell.entity_instance | None = None
) -> ifcopenshell.entity_instance | list | None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `products` | `list[entity_instance]` | *required* | Elements or types to assign material to |
| `type` | `str` | `"IfcMaterial"` | Material type: `"IfcMaterial"`, `"IfcMaterialLayerSet"`, `"IfcMaterialProfileSet"`, `"IfcMaterialConstituentSet"`, `"IfcMaterialList"` |
| `material` | `entity_instance \| None` | `None` | The material or material set to assign. When `None` with a set type, creates an empty set |

### Return Value

`ifcopenshell.entity_instance | list | None` — The IfcRelAssociatesMaterial relationship, or a list of relationships, or None.

### Behavior

- Automatically unassigns any previously assigned material from the products.
- When `type` is a set type and `material` is `None`, creates an empty material set and assigns it.
- When assigning to element types, occurrences of that type inherit the material automatically.

---

## ifcopenshell.api.material.add_material_set()

Creates a material set container for layered, profiled, or constituent materials.

```python
ifcopenshell.api.material.add_material_set(
    file: ifcopenshell.file,
    name: str = "Unnamed",
    set_type: str = "IfcMaterialConstituentSet"
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `name` | `str` | `"Unnamed"` | Name for the material set |
| `set_type` | `str` | `"IfcMaterialConstituentSet"` | One of: `"IfcMaterialLayerSet"`, `"IfcMaterialProfileSet"`, `"IfcMaterialConstituentSet"`, `"IfcMaterialList"` |

### Return Value

`ifcopenshell.entity_instance` — The created material set entity.

### Notes

- `IfcMaterialConstituentSet` is NOT available in IFC2X3. Use `IfcMaterialList` for IFC2X3.
- The default `set_type` is `"IfcMaterialConstituentSet"`. ALWAYS specify `set_type` explicitly.

---

## ifcopenshell.api.material.add_layer()

Adds a material layer to an IfcMaterialLayerSet.

```python
ifcopenshell.api.material.add_layer(
    file: ifcopenshell.file,
    layer_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `layer_set` | `entity_instance` | *required* | The IfcMaterialLayerSet to add the layer to |
| `material` | `entity_instance` | *required* | The IfcMaterial for this layer |
| `name` | `str \| None` | `None` | Optional layer name |

### Return Value

`ifcopenshell.entity_instance` — The created IfcMaterialLayer entity.

### Notes

- The layer is created with `LayerThickness = 0.0`. ALWAYS call `edit_layer()` to set the thickness.
- Layers are added in order. The first layer added is the outermost layer.

---

## ifcopenshell.api.material.edit_layer()

Modifies properties of an existing IfcMaterialLayer.

```python
ifcopenshell.api.material.edit_layer(
    file: ifcopenshell.file,
    layer: ifcopenshell.entity_instance,
    attributes: dict | None = None,
    material: ifcopenshell.entity_instance | None = None
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `layer` | `entity_instance` | *required* | The IfcMaterialLayer to edit |
| `attributes` | `dict \| None` | `None` | Attributes to update: `{"LayerThickness": float, "IsVentilated": bool, "Name": str, "Description": str, "Priority": int}` |
| `material` | `entity_instance \| None` | `None` | New IfcMaterial to assign to this layer |

### Return Value

`None`

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `LayerThickness` | `float` | Thickness in model units (meters for SI). REQUIRED. |
| `IsVentilated` | `bool` | Whether the layer is ventilated (air gap) |
| `Name` | `str` | Layer name |
| `Description` | `str` | Layer description |
| `Priority` | `int` | Priority for layer connectivity |

---

## ifcopenshell.api.material.remove_layer()

Removes a layer from its IfcMaterialLayerSet.

```python
ifcopenshell.api.material.remove_layer(
    file: ifcopenshell.file,
    layer: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.material.add_constituent()

Adds a named material constituent to an IfcMaterialConstituentSet. IFC4+ only.

```python
ifcopenshell.api.material.add_constituent(
    file: ifcopenshell.file,
    constituent_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `constituent_set` | `entity_instance` | *required* | The IfcMaterialConstituentSet to add to |
| `material` | `entity_instance` | *required* | The IfcMaterial for this constituent |
| `name` | `str \| None` | `None` | Constituent name (e.g., "Frame", "Glazing") |

### Return Value

`ifcopenshell.entity_instance` — The created IfcMaterialConstituent entity.

### Notes

- NOT available in IFC2X3. Calling this on an IFC2X3 file raises an error.
- Names SHOULD be descriptive of the component part (e.g., "Frame", "Glazing", "Panel").

---

## ifcopenshell.api.material.edit_constituent()

Modifies properties of an existing IfcMaterialConstituent.

```python
ifcopenshell.api.material.edit_constituent(
    file: ifcopenshell.file,
    constituent: ifcopenshell.entity_instance,
    attributes: dict | None = None,
    material: ifcopenshell.entity_instance | None = None
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `constituent` | `entity_instance` | *required* | The IfcMaterialConstituent to edit |
| `attributes` | `dict \| None` | `None` | Attributes to update: `{"Name": str, "Description": str, "Fraction": float, "Category": str}` |
| `material` | `entity_instance \| None` | `None` | New IfcMaterial to assign |

---

## ifcopenshell.api.material.remove_constituent()

Removes a constituent from its IfcMaterialConstituentSet.

```python
ifcopenshell.api.material.remove_constituent(
    file: ifcopenshell.file,
    constituent: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.material.add_profile()

Adds a material profile to an IfcMaterialProfileSet.

```python
ifcopenshell.api.material.add_profile(
    file: ifcopenshell.file,
    profile_set: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance | None = None,
    profile: ifcopenshell.entity_instance | None = None,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `profile_set` | `entity_instance` | *required* | The IfcMaterialProfileSet to add to |
| `material` | `entity_instance \| None` | `None` | The IfcMaterial for this profile |
| `profile` | `entity_instance \| None` | `None` | The IfcProfileDef defining the cross-section |
| `name` | `str \| None` | `None` | Profile item name |

### Return Value

`ifcopenshell.entity_instance` — The created IfcMaterialProfile entity.

### Notes

- ALWAYS provide a `profile` (IfcProfileDef). A material profile without a cross-section definition is incomplete.
- Common profile types: `IfcIShapeProfileDef`, `IfcRectangleProfileDef`, `IfcCircleProfileDef`, `IfcArbitraryClosedProfileDef`.

---

## ifcopenshell.api.material.edit_profile()

Modifies properties of an existing IfcMaterialProfile.

```python
ifcopenshell.api.material.edit_profile(
    file: ifcopenshell.file,
    profile: ifcopenshell.entity_instance,
    attributes: dict | None = None,
    material: ifcopenshell.entity_instance | None = None
) -> None
```

---

## ifcopenshell.api.material.remove_profile()

Removes a profile from its IfcMaterialProfileSet.

```python
ifcopenshell.api.material.remove_profile(
    file: ifcopenshell.file,
    profile: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.material.assign_profile()

Changes the profile curve of a material profile item and updates geometry.

```python
ifcopenshell.api.material.assign_profile(
    file: ifcopenshell.file,
    material_profile: ifcopenshell.entity_instance,
    profile: ifcopenshell.entity_instance
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `material_profile` | `entity_instance` | *required* | The IfcMaterialProfile to update |
| `profile` | `entity_instance` | *required* | The new IfcProfileDef to assign |

### Behavior

- Replaces the existing profile definition.
- Automatically updates body representation extrusions that reference this profile.

---

## ifcopenshell.api.material.edit_layer_usage()

Edits the IfcMaterialLayerSetUsage properties controlling offset from the reference line.

```python
ifcopenshell.api.material.edit_layer_usage(
    file: ifcopenshell.file,
    usage: ifcopenshell.entity_instance,
    attributes: dict
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `usage` | `entity_instance` | *required* | The IfcMaterialLayerSetUsage to edit |
| `attributes` | `dict` | *required* | Attributes: `{"OffsetFromReferenceLine": float, "DirectionSense": str, "LayerSetDirection": str}` |

### Key Attributes

| Attribute | Type | Values | Description |
|-----------|------|--------|-------------|
| `OffsetFromReferenceLine` | `float` | Any | Offset in model units from reference line |
| `DirectionSense` | `str` | `"POSITIVE"`, `"NEGATIVE"` | Direction of layer build-up |
| `LayerSetDirection` | `str` | `"AXIS1"`, `"AXIS2"`, `"AXIS3"` | Axis for layer stacking |

---

## ifcopenshell.api.material.edit_profile_usage()

Edits the IfcMaterialProfileSetUsage properties.

```python
ifcopenshell.api.material.edit_profile_usage(
    file: ifcopenshell.file,
    usage: ifcopenshell.entity_instance,
    attributes: dict
) -> None
```

---

## ifcopenshell.api.material.copy_material()

Duplicates a material or material set with all associated psets and styles.

```python
ifcopenshell.api.material.copy_material(
    file: ifcopenshell.file,
    material: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `material` | `entity_instance` | *required* | The IfcMaterialDefinition to copy |

### Return Value

`ifcopenshell.entity_instance` — The copied material.

### Behavior

- Copies property sets and surface styles.
- For material sets, set items (layers, profiles, constituents) are copied but underlying IfcMaterial entities are reused (not duplicated).

---

## ifcopenshell.api.material.edit_material()

Updates attributes of an IfcMaterial.

```python
ifcopenshell.api.material.edit_material(
    file: ifcopenshell.file,
    material: ifcopenshell.entity_instance,
    attributes: dict
) -> None
```

### Key Attributes

| Attribute | Type | Schema | Description |
|-----------|------|--------|-------------|
| `Name` | `str` | All | Material name |
| `Category` | `str` | IFC4+ | Material category |
| `Description` | `str` | IFC4+ | Material description |

---

## ifcopenshell.api.material.remove_material()

Deletes an IfcMaterial from the file.

```python
ifcopenshell.api.material.remove_material(
    file: ifcopenshell.file,
    material: ifcopenshell.entity_instance
) -> None
```

**WARNING:** Ensure the material is not referenced by any material set or relationship before removing.

---

## ifcopenshell.api.material.remove_material_set()

Deletes a material set from the file.

```python
ifcopenshell.api.material.remove_material_set(
    file: ifcopenshell.file,
    material_set: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.material.unassign_material()

Removes material associations from products.

```python
ifcopenshell.api.material.unassign_material(
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance]
) -> None
```

---

## ifcopenshell.api.material.edit_assigned_material()

Edits properties of an assigned material.

```python
ifcopenshell.api.material.edit_assigned_material(
    file: ifcopenshell.file,
    element: ifcopenshell.entity_instance,
    attributes: dict
) -> None
```

---

## ifcopenshell.api.material.add_list_item()

Adds a material to an IfcMaterialList (IFC2X3 legacy).

```python
ifcopenshell.api.material.add_list_item(
    file: ifcopenshell.file,
    material_list: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.material.remove_list_item()

Removes a material from an IfcMaterialList.

```python
ifcopenshell.api.material.remove_list_item(
    file: ifcopenshell.file,
    material_list: ifcopenshell.entity_instance,
    material: ifcopenshell.entity_instance
) -> None
```

---

## ifcopenshell.api.material.reorder_set_item()

Reorders items within a material set.

```python
ifcopenshell.api.material.reorder_set_item(
    file: ifcopenshell.file,
    material_set: ifcopenshell.entity_instance,
    old_index: int,
    new_index: int
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC model |
| `material_set` | `entity_instance` | *required* | The material set containing the item |
| `old_index` | `int` | *required* | Current 0-based index of the item |
| `new_index` | `int` | *required* | Target 0-based index for the item |

---

## ifcopenshell.api.material.set_shape_aspect_constituents()

Associates material constituents with shape aspects of an element.

```python
ifcopenshell.api.material.set_shape_aspect_constituents(
    file: ifcopenshell.file,
    element: ifcopenshell.entity_instance,
    constituents: list[ifcopenshell.entity_instance]
) -> None
```

---

## Utility Functions (ifcopenshell.util.element)

### get_material()

Retrieves the material assigned to an element.

```python
ifcopenshell.util.element.get_material(
    element: ifcopenshell.entity_instance,
    should_skip_usage: bool = False,
    should_inherit: bool = True
) -> ifcopenshell.entity_instance | None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | `entity_instance` | *required* | The IFC element to query |
| `should_skip_usage` | `bool` | `False` | When True, returns the underlying set instead of the usage entity |
| `should_inherit` | `bool` | `True` | When True, checks the element's type if no direct material is assigned |

### Return Value

Returns the material definition: `IfcMaterial`, `IfcMaterialLayerSet`, `IfcMaterialLayerSetUsage`, `IfcMaterialProfileSet`, `IfcMaterialProfileSetUsage`, `IfcMaterialConstituentSet`, `IfcMaterialList`, or `None`.
