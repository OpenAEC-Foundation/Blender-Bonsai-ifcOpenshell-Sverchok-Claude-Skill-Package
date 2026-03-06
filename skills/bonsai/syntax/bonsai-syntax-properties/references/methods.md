# Property Set API Method Signatures

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+
> **Source**: `ifcopenshell/api/pset/`, `ifcopenshell/api/pset_template/`, `ifcopenshell/util/element.py`, `bonsai/tool/pset.py`, `bonsai/core/pset.py`

---

## ifcopenshell.api.pset Module

### add_pset

```python
ifcopenshell.api.run("pset.add_pset", model,
    product: ifcopenshell.entity_instance,
    name: str,
    ifc2x3_subclass: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

Creates an `IfcPropertySet` and links it to the product. **Idempotent**: returns existing pset if one with the same name already exists on the product.

**Product type behavior**:

| Product Type | Behavior |
|-------------|----------|
| `IfcObject` / `IfcContext` | Creates `IfcPropertySet` + `IfcRelDefinesByProperties` link |
| `IfcTypeObject` | Creates `IfcPropertySet`, appends to `HasPropertySets` |
| `IfcMaterialDefinition` | IFC2X3: creates `ifc2x3_subclass` (default `IfcExtendedMaterialProperties`). IFC4+: creates `IfcMaterialProperties` |
| `IfcProfileDef` | IFC2X3: creates `ifc2x3_subclass` (default `IfcGeneralProfileProperties`). IFC4+: creates `IfcProfileProperties` |
| Other | Raises `TypeError` |

---

### add_qto

```python
ifcopenshell.api.run("pset.add_qto", model,
    product: ifcopenshell.entity_instance,
    name: str,
) -> ifcopenshell.entity_instance
```

Creates an `IfcElementQuantity` and links it to the product. **Idempotent**: returns existing qto if one with the same name already exists.

Sets `MethodOfMeasurement = "BaseQuantities"` automatically when name ends with `"BaseQuantities"`.

**Product type behavior**:

| Product Type | Behavior |
|-------------|----------|
| `IfcObject` / `IfcContext` | Creates `IfcElementQuantity` + `IfcRelDefinesByProperties` link |
| `IfcTypeObject` | Creates `IfcElementQuantity`, appends to `HasPropertySets` |

---

### edit_pset

```python
ifcopenshell.api.run("pset.edit_pset", model,
    pset: ifcopenshell.entity_instance,
    name: Optional[str] = None,
    properties: Optional[dict[str, Any]] = None,
    pset_template: Optional[ifcopenshell.entity_instance] = None,
    should_purge: bool = True,
) -> None
```

Edits properties on an existing pset. Handles `IfcPropertySet`, `IfcMaterialProperties`, and `IfcProfileProperties`.

**Parameters**:

| Parameter | Description |
|-----------|-------------|
| `pset` | The pset entity to edit |
| `name` | New name for the pset (optional rename) |
| `properties` | Dict of `{name: value}` pairs to set/update/delete |
| `pset_template` | Custom template to use for type resolution. If `None`, built-in buildingSMART template is auto-loaded by pset name |
| `should_purge` | When `True` (default): properties with `None` values are removed. When `False`: properties remain with `NominalValue=None` |

**Value handling**:

| Value | Behavior |
|-------|----------|
| Python primitive (`str`, `float`, `int`, `bool`) | Auto-mapped to IFC type (see type mapping) |
| `None` + `should_purge=True` | Property entity removed from model |
| `None` + `should_purge=False` | Property kept with `NominalValue=None` |
| `list` / `tuple` | Creates `IfcPropertyEnumeratedValue` or `IfcPropertyListValue` (requires template) |
| `ifcopenshell.entity_instance` | Set directly as `NominalValue` |
| `dict` with `"Unit"` and `"NominalValue"` keys | Sets custom unit alongside value |
| `datetime.date` / `datetime.datetime` | Converted via `.isoformat()` to string |

**Shared property handling**: When a property entity is shared (`get_total_inverses > 1`), it is treated as a new property to avoid affecting other psets.

---

### edit_qto

```python
ifcopenshell.api.run("pset.edit_qto", model,
    qto: ifcopenshell.entity_instance,
    name: Optional[str] = None,
    properties: Optional[dict[str, Any]] = None,
    pset_template: Optional[ifcopenshell.entity_instance] = None,
) -> None
```

Edits quantities on an existing `IfcElementQuantity` or `IfcPhysicalComplexQuantity`.

**No `should_purge` parameter**: quantities set to `None` are ALWAYS removed. IFC does not allow `None` quantities.

**Value handling**:

| Value | Behavior |
|-------|----------|
| `float` | Creates `IfcQuantity{Type}` based on name keyword inference |
| `int` | Creates `IfcQuantityCount` |
| `None` | Quantity entity always removed |
| `dict` | Creates `IfcPhysicalComplexQuantity` with `Discrimination` key, recurses for nested quantities |
| `ifcopenshell.entity_instance` | Type extracted from entity class |

**Name-based type inference** (`infer_property_type`):

```python
FLOAT_TYPE_KEYWORDS = (
    ("Area",   ("area",)),
    ("Volume", ("volume",)),
    ("Weight", ("weight", "mass")),
    ("Length", ("length", "width", "height", "depth", "distance")),
    ("Time",   ("time", "duration")),
)
```

Default for unrecognized float names: `"Length"`.

---

### assign_pset

```python
ifcopenshell.api.run("pset.assign_pset", model,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance,
) -> Union[ifcopenshell.entity_instance, None]
```

Assigns an existing pset to additional products. For occurrences: creates or reuses `IfcRelDefinesByProperties`. For types: appends to `HasPropertySets`.

Returns the `IfcRelDefinesByProperties` for occurrences, `None` for types.

---

### unassign_pset

```python
ifcopenshell.api.run("pset.unassign_pset", model,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance,
) -> None
```

Removes the pset association from products without deleting the pset entity itself.

---

### unshare_pset

```python
ifcopenshell.api.run("pset.unshare_pset", model,
    products: list[ifcopenshell.entity_instance],
    pset: ifcopenshell.entity_instance,
) -> list[ifcopenshell.entity_instance]
```

Creates independent shallow copies of a shared pset for the specified products. Unassigns the shared pset from those products first, then assigns the copy.

Returns list of newly created pset copies.

---

### remove_pset

```python
ifcopenshell.api.run("pset.remove_pset", model,
    product: ifcopenshell.entity_instance,
    pset: ifcopenshell.entity_instance,
) -> None
```

Removes a pset from a product. If the pset is no longer used by any other product, the pset entity and all its properties are also deleted from the model. Handles cleanup of `IfcRelDefinesByProperties`, `OwnerHistory`, `IfcPropertyEnumeration`, and shared properties.

---

## ifcopenshell.api.pset_template Module

### add_pset_template

```python
ifcopenshell.api.run("pset_template.add_pset_template", model,
    name: str = "New_Pset",
    template_type: str = "PSET_TYPEDRIVENOVERRIDE",
    applicable_entity: str = "IfcObject,IfcTypeObject",
) -> ifcopenshell.entity_instance
```

Creates an `IfcPropertySetTemplate`.

**Valid `template_type` values**:

| Value | Description |
|-------|-------------|
| `PSET_TYPEDRIVENONLY` | Assigned only to types |
| `PSET_TYPEDRIVENOVERRIDE` | Assigned to types or occurrences; occurrence overrides type |
| `PSET_OCCURRENCEDRIVEN` | Assigned to occurrences only |
| `PSET_PERFORMANCEDRIVEN` | Timeseries data range |
| `QTO_TYPEDRIVENONLY` | Quantities on types only |
| `QTO_TYPEDRIVENOVERRIDE` | Quantities on types or occurrences |
| `QTO_OCCURRENCEDRIVEN` | Quantities on occurrences only |
| `NOTDEFINED` | No restriction |

---

### add_prop_template

```python
ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template: ifcopenshell.entity_instance,
    name: str = "NewProperty",
    description: Optional[str] = None,
    template_type: Optional[str] = None,
    primary_measure_type: Optional[str] = None,
) -> ifcopenshell.entity_instance
```

Creates an `IfcSimplePropertyTemplate`. Appended to `pset_template.HasPropertyTemplates` in alphabetical order.

**Auto-detection**: If `template_type` is `None`: defaults to `"Q_LENGTH"` for QTO templates, `"P_SINGLEVALUE"` for PSET templates. If PSET and `primary_measure_type` is `None`: defaults to `"IfcLabel"`.

**Valid property `template_type` values**: `P_SINGLEVALUE`, `P_ENUMERATEDVALUE`, `P_BOUNDEDVALUE`, `P_LISTVALUE`, `P_TABLEVALUE`, `P_REFERENCEVALUE`

**Valid quantity `template_type` values**: `Q_LENGTH`, `Q_AREA`, `Q_VOLUME`, `Q_COUNT`, `Q_WEIGHT`, `Q_TIME`

---

### edit_pset_template

```python
ifcopenshell.api.run("pset_template.edit_pset_template", model,
    pset_template: ifcopenshell.entity_instance,
    attributes: dict[str, Any],
) -> None
```

Edits pset template attributes directly via `setattr`.

---

### edit_prop_template

```python
ifcopenshell.api.run("pset_template.edit_prop_template", model,
    prop_template: ifcopenshell.entity_instance,
    attributes: dict[str, Any],
) -> None
```

Edits property template attributes. Special handling for `"Enumerators"` key: accepts a list of plain values and creates/updates an `IfcPropertyEnumeration` entity.

---

### remove_pset_template

```python
ifcopenshell.api.run("pset_template.remove_pset_template", model,
    pset_template: ifcopenshell.entity_instance,
) -> None
```

Recursively removes pset template and all its property templates via `remove_deep`.

---

### remove_prop_template

```python
ifcopenshell.api.run("pset_template.remove_prop_template", model,
    prop_template: ifcopenshell.entity_instance,
) -> None
```

Removes property template from parent's `HasPropertyTemplates` and calls `remove_deep`.

---

## ifcopenshell.util.element — Property Reading Functions

### get_psets

```python
ifcopenshell.util.element.get_psets(
    element: ifcopenshell.entity_instance,
    psets_only: bool = False,
    qtos_only: bool = False,
    should_inherit: bool = True,
    verbose: bool = False,
) -> dict[str, dict[str, Any]]
```

Returns all psets and/or qtos as a nested dict. Each inner dict always includes an `"id"` key with the pset entity step ID.

| Parameter | Default | Effect |
|-----------|---------|--------|
| `psets_only` | `False` | Only return `IfcPropertySet` |
| `qtos_only` | `False` | Only return `IfcElementQuantity` |
| `should_inherit` | `True` | Include psets inherited from type |
| `verbose` | `False` | Include IFC class info per property (e.g., `"class": "IfcPropertySingleValue"`) |

---

### get_pset

```python
ifcopenshell.util.element.get_pset(
    element: ifcopenshell.entity_instance,
    name: str,
    prop: Optional[str] = None,
    psets_only: bool = False,
    qtos_only: bool = False,
    should_inherit: bool = True,
    verbose: bool = False,
) -> Union[Any, dict[str, Any]]
```

Returns a single pset/qto by name. If `prop` is specified, returns that single property value directly instead of the full dict.

---

### get_property_definition

```python
ifcopenshell.util.element.get_property_definition(
    definition: Optional[ifcopenshell.entity_instance],
    prop: Optional[str] = None,
    verbose: bool = False,
) -> Union[Any, dict[str, Any]]
```

Extracts property values from a pset/qto entity directly.

---

### get_elements_by_pset

```python
ifcopenshell.util.element.get_elements_by_pset(
    pset: ifcopenshell.entity_instance,
) -> set[ifcopenshell.entity_instance]
```

Returns all elements that reference the given pset entity.

---

## Bonsai tool.Pset Methods

All methods are `@classmethod` on `bonsai.tool.pset.Pset`.

```python
# Property retrieval
tool.Pset.get_element_pset(element, pset_name) -> Union[entity, None]
tool.Pset.get_pset_props(obj_name, obj_type) -> PsetProperties
tool.Pset.get_pset_name(obj_name, obj_type, pset_type="PSET") -> str
tool.Pset.get_pset_template(name) -> Union[entity, None]

# Validation
tool.Pset.is_pset_applicable(element, pset_name) -> bool
tool.Pset.is_pset_empty(pset) -> bool

# Type resolution
tool.Pset.get_special_type_for_prop(prop_or_template) -> Literal["LENGTH", "AREA", "VOLUME", "URI", ""]
tool.Pset.get_prop_template_primitive_type(prop_template) -> str
tool.Pset.cast_string_to_primitive(value) -> Any

# UI operations
tool.Pset.enable_pset_editing(pset_id, pset_name, pset_type, obj, obj_type) -> None
tool.Pset.clear_blender_pset_properties(props) -> None
tool.Pset.set_active_pset(props, pset, has_template) -> None
tool.Pset.enable_proposed_pset(props, pset_name, pset_type, has_template) -> None
tool.Pset.import_pset_from_template(pset_template, pset, props) -> None
tool.Pset.import_pset_from_existing(pset, props, pset_template) -> None
tool.Pset.add_proposed_property(name, value, props) -> Union[None, str]
tool.Pset.reset_proposed_property_fields(props) -> None

# Sharing
tool.Pset.get_selected_pset_elements(obj_name, obj_type, pset) -> list[entity]

# Bulk operations
tool.Pset.get_global_pset_props() -> GlobalPsetProperties
tool.Pset.get_bulk_operation_collection(operation_type) -> bpy_prop_collection
```

---

## Bonsai core.pset Functions

```python
# Copy a property from active object to all selected objects
core.pset.copy_property_to_selection(
    ifc, pset, obj, pset_name, prop_name, prop_value, is_pset=True) -> None

# Add a new pset to element(s)
core.pset.add_pset(ifc, pset, blender, obj_name, obj_type) -> None

# Enable pset editing mode in Blender UI
core.pset.enable_pset_editing(
    pset_tool, pset, pset_name, pset_type, obj_name, obj_type) -> None

# Add a custom property to a non-template pset
core.pset.add_proposed_prop(pset, obj_name, obj_type, name, value) -> Union[None, str]

# Unshare a pset for selected elements
core.pset.unshare_pset(ifc, pset_tool, obj_type, obj_name, pset_id) -> None
```

---

## Bonsai Operators — Complete bl_idname List

### pset module (`bim/module/pset/`)

| bl_idname | Purpose |
|-----------|---------|
| `bim.toggle_pset_expansion` | Toggle pset expand/collapse in UI |
| `bim.enable_pset_editing` | Enter pset editing mode |
| `bim.disable_pset_editing` | Exit pset editing mode |
| `bim.edit_pset` | Save pset edits to IFC |
| `bim.remove_pset` | Remove pset from element(s) |
| `bim.add_pset` | Add property set |
| `bim.unshare_pset` | Unshare (copy) pset for selected elements |
| `bim.add_qto` | Add quantity set |
| `bim.copy_property_to_selection` | Copy property to selected objects |
| `bim.add_property_to_edit` | Add entry to bulk operation list |
| `bim.remove_property_to_edit` | Remove entry from bulk operation list |
| `bim.pset_bulk_edit_clear_list` | Clear bulk operation list |
| `bim.pset_bulk_rename_parameters` | Bulk rename properties across elements |
| `bim.add_edit_custom_property` | Bulk add/edit custom properties |
| `bim.bulk_remove_psets` | Bulk remove psets from selected objects |
| `bim.add_proposed_prop` | Add custom property to non-template pset |
| `bim.save_pset_as_template` | Save pset as reusable template |

### pset_template module (`bim/module/pset_template/`)

| bl_idname | Purpose |
|-----------|---------|
| `bim.add_pset_template_file` | Create new .ifc template file |
| `bim.add_pset_template` | Add pset template to template file |
| `bim.remove_pset_template` | Remove pset template |
| `bim.enable_editing_pset_template` | Enter pset template editing mode |
| `bim.disable_editing_pset_template` | Exit pset template editing mode |
| `bim.enable_editing_prop_template` | Enter property template editing mode |
| `bim.disable_editing_prop_template` | Exit property template editing mode |
| `bim.edit_pset_template` | Save pset template edits |
| `bim.edit_prop_template` | Save property template edits |
| `bim.add_prop_template` | Add property template to pset template |
| `bim.remove_prop_template` | Remove property template |
| `bim.add_prop_enum` | Add enumeration value |
| `bim.delete_prop_enum` | Remove enumeration value |
| `bim.save_pset_template_file` | Write template file to disk |
| `bim.remove_pset_template_file` | Delete template file from disk |
| `bim.pset_templates_ui_select` | Select matching template in UI |

### qto module (`bim/module/qto/`)

| bl_idname | Purpose |
|-----------|---------|
| `bim.calculate_circle_radius` | Calculate circle radius from vertices |
| `bim.calculate_edge_lengths` | Calculate total edge lengths |
| `bim.calculate_face_areas` | Calculate total face areas |
| `bim.calculate_object_volumes` | Calculate object volumes |
| `bim.calculate_formwork_area` | Calculate formwork area |
| `bim.calculate_side_formwork_area` | Calculate side formwork area |
| `bim.calculate_single_quantity` | Calculate single quantity via ifc5d |
| `bim.perform_quantity_take_off` | Full QTO on selected objects |

---

## Sources

- IfcOpenShell API source: `src/ifcopenshell-python/ifcopenshell/api/pset/`
- IfcOpenShell API source: `src/ifcopenshell-python/ifcopenshell/api/pset_template/`
- IfcOpenShell util source: `src/ifcopenshell-python/ifcopenshell/util/element.py`
- Bonsai pset module: `src/bonsai/bonsai/bim/module/pset/`
- Bonsai pset_template module: `src/bonsai/bonsai/bim/module/pset_template/`
- Bonsai qto module: `src/bonsai/bonsai/bim/module/qto/`
- Bonsai core/pset: `src/bonsai/bonsai/core/pset.py`
- Bonsai tool/pset: `src/bonsai/bonsai/tool/pset.py`
