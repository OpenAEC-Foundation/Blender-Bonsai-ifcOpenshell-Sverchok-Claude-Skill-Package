# Classification API Signatures

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x | Python 3.11+
> **Source**: `ifcopenshell.api.classification`, `ifcopenshell.util.classification`, `bsdd.Client`

## ifcopenshell.api.classification

### add_classification

Adds a classification system to the IFC project.

```python
ifcopenshell.api.run(
    "classification.add_classification",
    file: ifcopenshell.file,
    classification: Union[str, ifcopenshell.entity_instance]
) -> ifcopenshell.entity_instance  # IfcClassification
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `file` | `ifcopenshell.file` | The target IFC model |
| `classification` | `str` or `IfcClassification` | String creates custom; entity copies from library file using `ifcopenshell.util.schema.Migrator` |

**Behavior:**
- From `str`: Creates `IfcClassification(Name=classification)` + `IfcRelAssociatesClassification` linked to `IfcProject`
- From `IfcClassification` entity: Migrates the entity (handles schema differences like `EditionDate` format changes between IFC2X3 and IFC4)
- ALWAYS creates an `IfcRelAssociatesClassification` linking the classification to the project

**Returns:** The new `IfcClassification` entity in the model.

---

### add_reference

Associates a classification reference with one or more IFC products.

```python
ifcopenshell.api.run(
    "classification.add_reference",
    file: ifcopenshell.file,
    products: list[ifcopenshell.entity_instance],
    reference: Optional[ifcopenshell.entity_instance] = None,
    identification: Optional[str] = None,
    name: Optional[str] = None,
    classification: Optional[ifcopenshell.entity_instance] = None,
    is_lightweight: bool = True
) -> Optional[ifcopenshell.entity_instance]  # IfcClassificationReference or None
```

**Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | — | The target IFC model |
| `products` | `list` | — | List of IFC elements to classify. MUST be a list, even for single elements |
| `reference` | `IfcClassificationReference` | `None` | From library file. Mutually exclusive with `identification`/`name` |
| `identification` | `str` | `None` | Classification code (e.g., `"Ss_25_10_30"`). For custom references |
| `name` | `str` | `None` | Human-readable label. For custom references |
| `classification` | `IfcClassification` | `None` | The classification system in the model |
| `is_lightweight` | `bool` | `True` | When `True`, detaches parent references from hierarchy |

**Behavior:**
- Custom approach (identification/name): Creates `IfcClassificationReference` with `ReferencedSource` → `IfcClassification`
- Library approach (reference): Uses `Migrator` to copy reference entity. Lightweight mode strips parent chain.
- Checks for existing references by identification to avoid duplicates
- For rooted products (`IfcRoot` subclasses): Uses `IfcRelAssociatesClassification`
- For non-rooted products (IFC4+): Uses `IfcExternalReferenceRelationship`
- IFC2X3: Maps `Identification` to `ItemReference` automatically

**Returns:** The `IfcClassificationReference` entity, or `None` if reference already exists on all products.

---

### edit_classification

Modifies attributes of an existing `IfcClassification`.

```python
ifcopenshell.api.run(
    "classification.edit_classification",
    file: ifcopenshell.file,
    classification: ifcopenshell.entity_instance,
    attributes: dict[str, Any]
) -> None
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `file` | `ifcopenshell.file` | The target IFC model |
| `classification` | `IfcClassification` | The entity to modify |
| `attributes` | `dict` | Key-value pairs of attributes to set (e.g., `{"Name": "...", "Edition": "..."}`) |

**Common attributes:** `Name`, `Source`, `Edition`, `EditionDate`, `Description`, `Location` (IFC4), `Specification` (IFC4X3)

---

### edit_reference

Modifies attributes of an existing `IfcClassificationReference`.

```python
ifcopenshell.api.run(
    "classification.edit_reference",
    file: ifcopenshell.file,
    reference: ifcopenshell.entity_instance,
    attributes: dict[str, Any]
) -> None
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `file` | `ifcopenshell.file` | The target IFC model |
| `reference` | `IfcClassificationReference` | The entity to modify |
| `attributes` | `dict` | Key-value pairs (e.g., `{"Identification": "...", "Name": "..."}`) |

**Common attributes:** `Identification` (IFC4+), `ItemReference` (IFC2X3), `Name`, `Location`, `Description`

---

### remove_classification

Deletes a classification system and ALL associated references.

```python
ifcopenshell.api.run(
    "classification.remove_classification",
    file: ifcopenshell.file,
    classification: ifcopenshell.entity_instance
) -> None
```

**Behavior:**
- Recursively collects ALL child references via `HasReferences`
- Removes all references first, then the classification entity
- Removes orphaned `IfcRelAssociatesClassification` relationships
- **WARNING**: This is destructive and removes ALL references in the classification system

---

### remove_reference

Unassigns a classification reference from specified products.

```python
ifcopenshell.api.run(
    "classification.remove_reference",
    file: ifcopenshell.file,
    reference: ifcopenshell.entity_instance,
    products: list[ifcopenshell.entity_instance]
) -> None
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `file` | `ifcopenshell.file` | The target IFC model |
| `reference` | `IfcClassificationReference` | The reference to unassign |
| `products` | `list` | Elements to unassign from |

**Behavior:**
- Removes products from the `IfcRelAssociatesClassification.RelatedObjects`
- If no products remain associated, deletes the reference entity itself

---

## ifcopenshell.util.classification

### get_references

Returns all classification references assigned to an element.

```python
ifcopenshell.util.classification.get_references(
    element: ifcopenshell.entity_instance,
    should_inherit: bool = True
) -> set[ifcopenshell.entity_instance]
```

**Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `element` | Entity | — | Any IFC element |
| `should_inherit` | `bool` | `True` | Include references inherited from type element |

**Behavior:**
- For `IfcRoot` subclasses: Traverses `HasAssociations` → `IfcRelAssociatesClassification`
- For non-`IfcRoot`: Traverses `HasExternalReferences` → `IfcExternalReferenceRelationship`
- When `should_inherit=True`: Merges occurrence + type references. Occurrence references override type references per classification system.

---

### get_classification

Returns the parent `IfcClassification` for a reference.

```python
ifcopenshell.util.classification.get_classification(
    reference: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance  # IfcClassification
```

**Behavior:** Walks up the `ReferencedSource` chain until it reaches an `IfcClassification` entity (not an `IfcClassificationReference`).

---

### get_inherited_references

Returns parent references in the hierarchy chain.

```python
ifcopenshell.util.classification.get_inherited_references(
    reference: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]
```

**Behavior:** Collects parent `IfcClassificationReference` entities by walking up `ReferencedSource`.

---

### get_classification_data

Returns the classification tree structure as a dict.

```python
ifcopenshell.util.classification.get_classification_data(
    file: ifcopenshell.file
) -> tuple[list[dict], str]
```

**Returns:** Tuple of (tree structure as list of dicts, classification system name).

---

## bsdd.Client

### Constructor

```python
from bsdd import Client

client = Client()
# client.baseurl = "https://api.bsdd.buildingsmart.org/api/"
```

### search_class

Search for classes across dictionaries.

```python
client.search_class(
    search_text: str,
    dictionary_uris: Optional[str] = None,
    related_ifc_entities: Optional[list[str]] = None
) -> dict  # ClassSearchResponseContractV1
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `search_text` | `str` | Keyword to search |
| `dictionary_uris` | `str` | Comma-separated dictionary URIs to search within |
| `related_ifc_entities` | `list[str]` | IFC entity filter (e.g., `["IfcWall"]`). ALWAYS use this. |

---

### get_dictionary

List all available bSDD dictionaries.

```python
client.get_dictionary(
    dictionary_uri: Optional[str] = None,
    include_test_dictionaries: bool = False
) -> dict  # DictionaryResponseContractV1
```

---

### get_class

Get full class details including properties.

```python
client.get_class(
    uri: str,
    include_class_properties: bool = True,
    include_child_class_reference: bool = False,
    include_class_relations: bool = False
) -> dict  # ClassContractV1
```

---

### get_classes

Get class tree for a dictionary.

```python
client.get_classes(
    dictionary_uri: str,
    use_nested_classes: bool = True,
    class_type: Optional[str] = None,
    search_text: Optional[str] = None,
    related_ifc_entity: Optional[str] = None
) -> dict  # DictionaryClassesResponseContractV1
```

---

### search_in_dictionary

Search within a specific dictionary.

```python
client.search_in_dictionary(
    dictionary_uri: str,
    search_text: Optional[str] = None,
    related_ifc_entity: Optional[str] = None
) -> dict  # SearchInDictionaryResponseContractV1
```

---

### get_class_properties

Get properties for a class (paginated).

```python
client.get_class_properties(
    class_uri: str,
    property_set: Optional[str] = None,
    property_code: Optional[str] = None
) -> dict
```

---

### get_property

Get a single property's details.

```python
client.get_property(
    uri: str
) -> dict  # PropertyContractV5
```

---

## bsdd.apply_ifc_classification_properties

Standalone utility that bridges bSDD properties to IFC property sets.

```python
from bsdd import apply_ifc_classification_properties

apply_ifc_classification_properties(
    ifc_file: ifcopenshell.file,
    element: ifcopenshell.entity_instance,
    classificationProperties: list[dict]
) -> None
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `ifc_file` | `ifcopenshell.file` | Target IFC model |
| `element` | Entity | Element to receive property sets |
| `classificationProperties` | `list[dict]` | From `class_data["classProperties"]` (bSDD API response) |

**Behavior:**
1. Groups properties by property set name
2. Creates `IfcPropertySet` entities
3. Creates `IfcPropertySingleValue` or `IfcPropertyEnumeratedValue` per property
4. Associates via `IfcRelDefinesByProperties`

---

## Bonsai Tool Layer

### tool.Classification

```python
class Classification:
    @classmethod
    def get_classification_props(cls) -> BIMClassificationProperties: ...

    @classmethod
    def get_classification_reference_props(cls) -> BIMClassificationReferenceProperties: ...

    @classmethod
    def get_location(cls, classification) -> Optional[str]:
        """IFC4/IFC4X3: returns classification[5] (Specification/Location).
        IFC2X3: returns None (no Location attribute)."""

    @classmethod
    def set_location(cls, classification, location) -> None:
        """IFC4/IFC4X3: sets classification[5].
        IFC2X3: no-op."""
```

### tool.Bsdd

```python
class Bsdd:
    identifier_url = "https://identifier.buildingsmart.org"
    client = bsdd.Client()
    bsdd_classes: dict[str, dict] = {}      # URI -> class data cache
    bsdd_properties: dict[str, dict] = {}   # URI -> property data cache

    @classmethod
    def get_dictionaries(cls, client) -> list[dict]: ...
    @classmethod
    def search_class(cls, client, keyword, dictionary_uris, related_ifc_entities) -> list: ...
    @classmethod
    def get_active_dictionary_uri(cls) -> str: ...
    @classmethod
    def get_related_ifc_entities(cls, keyword) -> list[str]: ...
    @classmethod
    def create_dictionaries(cls, dictionaries) -> None: ...
    @classmethod
    def create_classes(cls, class_dict) -> None: ...
    @classmethod
    def create_class_psets(cls, pset_dict) -> None: ...
    @classmethod
    def clear_domains(cls) -> None: ...
    @classmethod
    def clear_classes(cls) -> None: ...
    @classmethod
    def clear_class_psets(cls) -> None: ...
    @classmethod
    def set_active_bsdd(cls, name, uri) -> None: ...
    @classmethod
    def save_active_bsdd_to_ifc(cls) -> None: ...
    @classmethod
    def get_active_bsdd_ifc(cls) -> list[dict]: ...
```
