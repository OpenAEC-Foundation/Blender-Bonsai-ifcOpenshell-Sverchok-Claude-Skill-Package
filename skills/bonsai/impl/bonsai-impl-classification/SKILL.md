---
name: bonsai-impl-classification
description: "Implements Bonsai classification workflows including applying classification systems (Uniclass 2015, OmniClass, NL-SfB, CCI), bSDD (buildingSMART Data Dictionary) integration for property lookups, managing IfcClassificationReference assignments, and bulk classification operations. Covers classification library import, element-to-classification mapping, and cross-referencing between classification systems."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Classification Implementation

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x | Python 3.11+
> **Module paths**: `bonsai.bim.module.classification`, `bonsai.bim.module.bsdd`
> **IFC schemas**: IFC2X3, IFC4, IFC4X3

## Critical Warnings

- ALWAYS use `ifcopenshell.api.run("classification.add_reference", ...)` with `products` (list), NOT `product` (singular). The API requires a list even for single elements.
- NEVER use identifier URIs (`identifier.buildingsmart.org`) for bSDD API calls. ALWAYS use `api.bsdd.buildingsmart.org` endpoints.
- ALWAYS filter bSDD searches by `related_ifc_entity` to prevent incorrect classification assignments.
- NEVER hardcode classification codes without verifying them against the source (bSDD or library file).
- In IFC2X3, the attribute is `ItemReference`, NOT `Identification`. The IfcOpenShell API handles this migration automatically when using `add_reference`.

## Decision Tree: Classification Source Selection

```
Need to classify an IFC element?
├── Have internet access?
│   ├── YES → Need live, up-to-date codes?
│   │   ├── YES → Use bSDD Client (Path 1)
│   │   └── NO  → Use Library File (Path 2)
│   └── NO  → Use Library File (Path 2)
├── Need associated properties auto-populated?
│   └── YES → Use bSDD Client (Path 1) with apply_ifc_classification_properties()
├── Batch processing large model?
│   └── YES → Use Library File (Path 2) for performance
└── Custom project-specific codes?
    └── YES → Use Manual Classification (Path 3)
```

## Three Classification Paths

### Path 1: bSDD Client (Online Lookup)

Use `bsdd.Client` for real-time classification lookup from the buildingSMART Data Dictionary.

```python
from bsdd import Client
import ifcopenshell
import ifcopenshell.api

client = Client()
model = ifcopenshell.open("model.ifc")

# Search for wall classifications in NL-SfB
results = client.search_class(
    search_text="external wall",
    dictionary_uris="https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2",
    related_ifc_entities=["IfcWall"]
)

# Add classification system to model
classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="NL-SfB 2005"
)

# Assign reference to element
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="21.21",
    name="Buitenwanden",
    classification=classification
)
```

### Path 2: Classification Library File (Offline)

Use pre-downloaded IFC classification library files from `https://github.com/Moult/IfcClassification`.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")
library = ifcopenshell.open("Uniclass2015.ifc")

# Copy classification from library to model
lib_classification = library.by_type("IfcClassification")[0]
classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification=lib_classification
)

# Find and assign a specific reference
lib_ref = [r for r in library.by_type("IfcClassificationReference")
           if r.Identification == "Ss_25_10_30"][0]
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    reference=lib_ref,
    classification=classification,
    is_lightweight=True
)
```

### Path 3: Manual/Custom Classification

Create project-specific classification codes without external sources.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")

classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="ProjectClassification"
)

wall_type = model.by_type("IfcWallType")[0]
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall_type],
    identification="W-001",
    name="Load Bearing External Wall",
    classification=classification
)
```

## Bonsai Operator Reference (Blender Context)

### Classification Operators

| Operator | bl_idname | Purpose |
|----------|-----------|---------|
| `LoadClassificationLibrary` | `bim.load_classification_library` | Load `.ifc/.ifczip/.ifcxml` classification file |
| `AddClassification` | `bim.add_classification` | Add classification from loaded library |
| `AddClassificationFromBSDD` | `bim.add_classification_from_bsdd` | Add classification from active bSDD dictionaries |
| `AddClassificationReference` | `bim.add_classification_reference` | Assign library reference to selected elements |
| `AddClassificationReferenceFromBSDD` | `bim.add_classification_reference_from_bsdd` | Assign bSDD class + auto-create property sets |
| `AddManualClassification` | `bim.add_manual_classification` | Create custom classification with user attributes |
| `AddManualClassificationReference` | `bim.add_manual_classification_reference` | Create custom reference for selected elements |
| `EditClassification` | `bim.edit_classification` | Modify classification attributes |
| `EditClassificationReference` | `bim.edit_classification_reference` | Modify reference attributes |
| `RemoveClassification` | `bim.remove_classification` | Remove classification AND all child references |
| `RemoveClassificationReference` | `bim.remove_classification_reference` | Unassign reference from selected elements |
| `ChangeClassificationLevel` | `bim.change_classification_level` | Navigate classification tree hierarchy |

### bSDD Operators

| Operator | bl_idname | Purpose |
|----------|-----------|---------|
| `LoadBSDDDictionaries` | `bim.load_bsdd_dictionaries` | Fetch all dictionaries from bSDD API |
| `SearchBSDDClassifications` | `bim.search_bsdd_classifications` | Search classes by keyword |
| `ImportBSDDClasses` | `bim.import_bsdd_classes` | Load bSDD classes for current element |
| `SearchBSDDProperties` | `bim.search_bsdd_properties` | Search properties by keyword |
| `AddBSDDProperties` | `bim.add_bsdd_properties` | Add selected bSDD properties as psets |

## IFC Classification Entity Model

```
IfcClassification                          (the classification system)
  .Name            = "Uniclass 2015"
  .Source           = "NBS"
  .Edition          = "1.8"
  .Specification    = "<URI>"              (IFC4X3 only)
  .Location         = "<URI>"              (IFC4 — mapped to [5] internally)
  └── HasReferences → [IfcClassificationReference, ...]

IfcClassificationReference                 (a specific code assignment)
  .Identification   = "Ss_25_10_30"        (IFC4+; "ItemReference" in IFC2X3)
  .Name             = "Wall structures"
  .Location         = "<class URI>"
  .ReferencedSource → IfcClassification    (or parent IfcClassificationReference)
  └── HasReferences → [child refs...]      (hierarchical codes)

IfcRelAssociatesClassification             (links elements to references)
  .RelatingClassification → IfcClassificationReference
  .RelatedObjects   → [IfcWall, IfcWallType, ...]
```

## Schema Version Differences

| Feature | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| Classification code attribute | `ItemReference` | `Identification` | `Identification` |
| Classification URI attribute | N/A | `Location` (attr index [5]) | `Specification` (attr index [5]) |
| Non-rooted element support | No | `IfcExternalReferenceRelationship` | `IfcExternalReferenceRelationship` |
| Multiple classifications per element | Yes | Yes | Yes |

## bSDD Integration Details

### Active Dictionary Persistence

Bonsai persists active bSDD dictionary selections in the IFC file as `IfcLibraryInformation` entities named `"BBIM_Active_bSDD"`. This makes selections portable across sessions.

```python
# How Bonsai stores active bSDD state in IFC
# IfcLibraryInformation.Name = "BBIM_Active_bSDD"
# └── IfcLibraryReference per active dictionary
#     .Name = dictionary["name"]
#     .Location = dictionary["uri"]
```

### bSDD-to-IFC Mapping

| bSDD Concept | IFC Entity | IFC Attribute |
|--------------|------------|---------------|
| Dictionary name | `IfcClassification` | `.Name` |
| Dictionary URI | `IfcClassification` | `.Specification` (IFC4X3) / `.Location` (IFC4) |
| Dictionary version | `IfcClassification` | `.Edition` |
| Organization | `IfcClassification` | `.Source` |
| Class code | `IfcClassificationReference` | `.Identification` |
| Class name | `IfcClassificationReference` | `.Name` |
| Class URI | `IfcClassificationReference` | `.Location` |
| Property | `IfcPropertySingleValue` | `.Name` |
| Allowed values | `IfcPropertyEnumeratedValue` | `.EnumerationValues` |

### bSDD Search with IFC Entity Filtering

ALWAYS pass `related_ifc_entities` when searching bSDD. This maps bSDD classes to their corresponding IFC entity types and prevents assigning wall codes to doors.

```python
from bsdd import Client

client = Client()

# Correct: filtered by IFC entity type
results = client.search_class(
    search_text="external wall",
    dictionary_uris="https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2",
    related_ifc_entities=["IfcWall"]
)

# WRONG: unfiltered search returns irrelevant results
results = client.search_class(
    search_text="external wall",
    dictionary_uris="https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2"
)
```

## Multi-Object Classification

Bonsai operators `AddClassificationReference`, `AddClassificationReferenceFromBSDD`, `AddManualClassificationReference`, and `RemoveClassificationReference` support Blender multi-selection via `context.selected_objects`.

For scripted bulk classification:

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")
walls = model.by_type("IfcWall")

classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="Uniclass 2015"
)

# Classify all walls at once (products accepts a list)
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=walls,
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=classification
)
```

## Reading Classifications

### Using ifcopenshell.util.classification

```python
import ifcopenshell.util.classification

element = model.by_type("IfcWall")[0]

# Get all classification references (includes inherited from type)
refs = ifcopenshell.util.classification.get_references(element, should_inherit=True)

for ref in refs:
    system = ifcopenshell.util.classification.get_classification(ref)
    print(f"{system.Name}: {ref.Identification} - {ref.Name}")
```

### Direct IFC Traversal

```python
element = model.by_type("IfcWall")[0]

for assoc in element.HasAssociations:
    if assoc.is_a("IfcRelAssociatesClassification"):
        ref = assoc.RelatingClassification
        print(f"Code: {ref.Identification}, Name: {ref.Name}")
```

## Bonsai Architecture Pattern

Classification operators bypass the standard three-layer architecture:

- **Delivery layer** (`bim/module/classification/operator.py`): Operators call `ifcopenshell.api.classification.*` directly.
- **Tool layer** (`tool/classification.py`): Minimal — only `get_location()`/`set_location()` for schema-aware URI handling.
- **Core layer**: No `core/classification.py` exists.

bSDD operators follow the full three-layer pattern:

- **Delivery layer** (`bim/module/bsdd/operator.py`): Delegates to `core.bsdd.*`.
- **Core layer** (`core/bsdd.py`): 4 pure functions with `tool.Bsdd` dependency injection.
- **Tool layer** (`tool/bsdd.py`): Full bSDD client integration, caching, IFC persistence.

## Common Classification Systems

| System | Region | bSDD URI (when available) |
|--------|--------|---------------------------|
| Uniclass 2015 | UK | Published on bSDD |
| OmniClass | North America | Published on bSDD |
| NL-SfB 2005 | Netherlands | `https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2` |
| CCI | International | Published on bSDD |
| ETIM | Europe | Published on bSDD |
| MasterFormat | North America | Published on bSDD |
| UniFormat | North America | Published on bSDD |

Multiple classification systems CAN be assigned simultaneously to a single element. IFC supports multiple `IfcRelAssociatesClassification` relationships per product.

## Reference Files

- [API Signatures](references/methods.md)
- [Workflow Examples](references/examples.md)
- [Anti-Patterns](references/anti-patterns.md)
