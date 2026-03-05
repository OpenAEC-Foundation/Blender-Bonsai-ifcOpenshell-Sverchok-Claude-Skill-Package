# bSDD Integration with IfcOpenShell Classification System

## What is bSDD?

The **buildingSMART Data Dictionary (bSDD)** is an online, centralized RESTful API operated by buildingSMART International. It serves as a standardized repository for construction industry classifications, properties, and material definitions. Organizations worldwide publish their classification systems (called "dictionaries") to bSDD, making them programmatically accessible via a single API.

bSDD solves a critical interoperability problem: instead of downloading classification spreadsheets or PDFs and manually entering codes, software ALWAYS retrieves up-to-date classification data directly from the API. This ensures codes are entered correctly and that the latest edition is used.

### Core Data Model

bSDD organizes data into these entities:

| Entity | Description | IFC Mapping |
|--------|-------------|-------------|
| **Dictionary** | A classification system published by an organization (e.g., NL-SfB, Uniclass) | `IfcClassification` |
| **Class** | A set of objects sharing characteristics (e.g., "External Wall", "Concrete Column") | `IfcClassificationReference` |
| **Property** | An inherent feature of a Class (e.g., thermal conductivity, fire rating) | `IfcPropertySingleValue` |
| **ClassProperty** | A Property instantiated within a specific Class, with context-specific constraints | `IfcPropertySingleValue` with constraints |
| **ClassRelation** | Links between Classes: IsParentOf, IsChildOf, HasPart, IsPartOf, HasMaterial, IsEqualTo, IsSimilarTo, HasReference | N/A (structural) |
| **AllowedValue** | Enumerated value restrictions for properties (e.g., fire ratings: REI30, REI60, REI90) | `IfcPropertyEnumeratedValue` |

### Class Types in bSDD

bSDD Classes are categorized into four types:
- **Standard Class** (most common) -- represents physical objects, abstract concepts, or processes
- **Material** -- physical substances, mapped to `IfcMaterial` via `IfcExternalReferenceRelationship`
- **GroupOfProperties** -- organized property collections
- **AlternativeUse** -- catch-all category

## bSDD API

### Base URLs

| Environment | URL | Authentication |
|-------------|-----|----------------|
| **Production** | `https://api.bsdd.buildingsmart.org/` | OAuth2 (Azure AD B2C) for write; read is open |
| **Test** | `https://test.bsdd.buildingsmart.org/` | Open for unsecured endpoints |
| **Identifiers** | `https://identifier.buildingsmart.org/uri/...` | Open (browser/JSON) |

**IMPORTANT**: Identifier URIs (`https://identifier.buildingsmart.org/...`) MUST NOT be used for system-to-system communication. ALWAYS use the API endpoints instead.

### REST Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/Dictionary/v1` | GET | List all available dictionaries |
| `/api/Class/v1` | GET | Retrieve class details including properties |
| `/api/SearchList/v1` | GET | Search for classes by text within a dictionary |
| `/api/Property/v1` | GET | Retrieve individual property details |

### GraphQL Endpoints

| Endpoint | Authentication |
|----------|----------------|
| `https://api.bsdd.buildingsmart.org/graphqls` | Secured (OAuth2) |
| `https://test.bsdd.buildingsmart.org/graphql` | Unsecured |
| `https://test.bsdd.buildingsmart.org/graphiql` | Interactive playground |

### Authentication

bSDD uses Azure Active Directory B2C for authentication. The test environment credentials are:

- **Client ID**: `4aba821f-d4ff-498b-a462-c2837dbbba70`
- **Tenant**: `buildingsmartservices.onmicrosoft.com`
- **API Scope**: `https://buildingsmartservices.onmicrosoft.com/bsddapi/read`

For production use, organizations MUST request a custom Client ID via buildingSMART's contact form. The MSAL (Microsoft Authentication Library) is the recommended authentication library for Python, JavaScript, Java, and .NET.

### Query Examples (Raw REST)

**List all dictionaries:**
```
GET https://api.bsdd.buildingsmart.org/api/Dictionary/v1
Accept: application/json
```

**Search for wall-related classes in NL-SfB:**
```
GET https://api.bsdd.buildingsmart.org/api/SearchList/v1?DictionaryUri=https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2&SearchText=wall&RelatedIfcEntity=IfcWall
Accept: application/json
```

**Get a specific class with properties:**
```
GET https://api.bsdd.buildingsmart.org/api/Class/v1?Uri=https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2/class/21.21&IncludeClassProperties=true
Accept: application/json
```

## IfcOpenShell Classification Module

### Available Functions

The `ifcopenshell.api.classification` module provides six functions for managing IFC classifications:

#### 1. `add_classification(file, classification)`
Creates a new classification system in the IFC project.
- `classification`: Either a string (custom name) or an `IfcClassification` entity from a library file
- **Returns**: `IfcClassification` entity

#### 2. `add_reference(file, products, reference, identification, name, classification, is_lightweight)`
Associates a classification reference with one or more IFC products.
- `products`: List of IFC elements to classify
- `reference`: An `IfcClassificationReference` from a library (optional)
- `identification`: Classification code string, e.g., `"Pr_12_23_34"` (for custom approach)
- `name`: Human-readable label (for custom approach)
- `classification`: The `IfcClassification` entity in the model
- `is_lightweight`: Boolean; when True, NEVER includes parent references in the hierarchy
- **Returns**: `IfcClassificationReference` or None

#### 3. `edit_classification(file, classification, attributes)`
Modifies attributes of an existing `IfcClassification`.

#### 4. `edit_reference(file, reference, attributes)`
Modifies attributes of an existing `IfcClassificationReference`.

#### 5. `remove_classification(file, classification)`
Deletes a classification system and ALL associated references.

#### 6. `remove_reference(file, reference, products)`
Unassigns a classification reference from specified products. Deletes the reference entity if no products remain associated.

### IFC Classification Entities

The IFC schema defines three core classification entities:

```
IfcClassification
    .Name              = "NL-SfB 2005"          (dictionary name)
    .Source             = "BIM Loket"            (organization)
    .Edition            = "2.2"                  (version)
    .EditionDate        = "2005-01-01"           (release date)
    .Specification      = "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2"  (IFC4x3)
    .Location           = "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2"  (IFC4)

IfcClassificationReference
    .Identification     = "21.21"                (class code)
    .Name               = "Buitenwanden"         (class name)
    .Location           = "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2/class/21.21"
    .ReferencedSource   -> IfcClassification     (link to parent system)

IfcRelAssociatesClassification
    .RelatingClassification -> IfcClassificationReference
    .RelatedObjects     -> [IfcWall, IfcWallType, ...]  (classified elements)
```

### Utility Functions for Retrieval

```python
import ifcopenshell.util.classification

# Get all classification references assigned to an element
references = ifcopenshell.util.classification.get_references(element)

# Get the classification system for a reference
system = ifcopenshell.util.classification.get_classification(reference)
```

### Two Approaches to Adding Classifications

**Approach 1: From a classification library file**

Classification libraries are available at https://github.com/Moult/IfcClassification as `.ifc` files. This approach provides the full hierarchy of codes.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("my_model.ifc")
library = ifcopenshell.open("Uniclass2015.ifc")

# Get the classification entity from the library
lib_classification = library.by_type("IfcClassification")[0]

# Add it to the model
classification = ifcopenshell.api.run(
    "classification.add_classification",
    model,
    classification=lib_classification
)

# Get a specific reference from the library
lib_reference = [r for r in library.by_type("IfcClassificationReference")
                 if r.Identification == "Pr_30_59_99_02"][0]

# Assign to an element
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run(
    "classification.add_reference",
    model,
    products=[wall],
    reference=lib_reference,
    classification=classification,
    is_lightweight=True
)
```

**Approach 2: Custom/manual classification**

```python
classification = ifcopenshell.api.run(
    "classification.add_classification",
    model,
    classification="MyProjectClassification"
)

wall_type = model.by_type("IfcWallType")[0]
ifcopenshell.api.run(
    "classification.add_reference",
    model,
    products=[wall_type],
    identification="W-001",
    name="Load Bearing External Wall",
    classification=classification
)
```

## Integration: bSDD -> IfcOpenShell

### The IfcOpenShell `bsdd` Module

IfcOpenShell includes a dedicated `bsdd` Python package (located at `src/bsdd/` in the repository). This package was developed as a Google Summer of Code (GSoC) project and completed in February 2023. It is a standalone library that requires only Python to run -- it does NOT depend on IfcOpenShell core.

**Installation**: The module is included with IfcOpenShell installations. It can also be used independently.

**Source files**:
- `bsdd.py` -- Main `Client` class and `apply_ifc_classification_properties()` utility
- `bsdd_json.py` -- JSON handling
- `type_hints.py` -- TypedDict contracts for API responses (`DictionaryContractV1`, `ClassContractV1`, `PropertyContractV5`, etc.)

### Client Class

```python
from bsdd import Client

client = Client()
# client.baseurl = "https://api.bsdd.buildingsmart.org/api/"
# Authentication is handled via OAuth2 B2C (login() method)
```

### Client Methods

| Method | Purpose | Parameters |
|--------|---------|------------|
| `get_dictionary()` | List all available dictionaries | None |
| `get_classes(uri)` | Get class tree for a dictionary | Dictionary URI |
| `get_properties(uri)` | Get properties for a dictionary | Dictionary URI |
| `get_class(uri)` | Get a specific class with properties | Class URI |
| `get_class_relations(uri)` | Get class relationships (paginated) | Class URI |
| `get_class_properties(uri)` | Get class properties (paginated) | Class URI |
| `search_text(text)` | Free text search across all dictionaries | Search text |
| `search_in_dictionary(uri, related_ifc_entity)` | Search within a specific dictionary | Dictionary URI, IFC entity name |
| `search_class(search_text, dictionary_uris, related_ifc_entities)` | Search for classes by text | Search text, list of dictionary URIs, list of IFC entities |
| `get_countries()` | List all countries | None |
| `get_languages()` | List all languages | None |
| `get_reference_documents()` | Get reference documents | None |
| `get_units()` | Get available units | None |
| `get_property(uri)` | Get a single property | Property URI |
| `get_property_value(uri)` | Get property value details | Property value URI |

### Complete Workflow: bSDD Lookup + IfcOpenShell Classification

This is the canonical workflow for looking up a classification in bSDD and applying it to an IFC model using IfcOpenShell:

```python
from bsdd import Client, apply_ifc_classification_properties
import ifcopenshell
import ifcopenshell.api

# ----- STEP 1: Connect to bSDD -----
client = Client()

# ----- STEP 2: Browse available dictionaries -----
dictionaries = client.get_dictionary()
# Each dictionary has: uri, name, version, organizationCode, etc.

# ----- STEP 3: Search for classification codes -----
# Example: search the Dutch NL-SfB 2005 for wall-related codes
results = client.search_in_dictionary(
    "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2",
    related_ifc_entity="IfcWall"
)
# Returns list of matching classes with their URIs and codes

# ----- STEP 4: Search by text within a dictionary -----
results = client.search_class(
    search_text="buitenwanden",  # "external walls" in Dutch
    dictionary_uris="https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2",
    related_ifc_entities=["IfcWall"]
)

# ----- STEP 5: Retrieve full class data including properties -----
class_data = client.get_class(
    "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2/class/21.21"
)
# class_data contains: classCode, name, classProperties, relatedIfcEntityNames, etc.

# ----- STEP 6: Open or create IFC model -----
model = ifcopenshell.open("my_building.ifc")
wall = model.by_type("IfcWall")[0]

# ----- STEP 7: Add the classification system to the IFC model -----
classification = ifcopenshell.api.run(
    "classification.add_classification",
    model,
    classification="NL-SfB 2005"
)

# Optionally set metadata to match bSDD dictionary info
ifcopenshell.api.run(
    "classification.edit_classification",
    model,
    classification=classification,
    attributes={
        "Source": "BIM Loket",
        "Edition": "2.2",
        "Location": "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2"
    }
)

# ----- STEP 8: Add the classification reference to the element -----
ifcopenshell.api.run(
    "classification.add_reference",
    model,
    products=[wall],
    identification="21.21",
    name="Buitenwanden",
    classification=classification
)

# ----- STEP 9: Apply bSDD properties to the element -----
# This utility AUTOMATICALLY creates property sets from bSDD class data
if class_data.get("classProperties"):
    apply_ifc_classification_properties(
        model, wall, class_data["classProperties"]
    )

# ----- STEP 10: Save the model -----
model.write("my_building_classified.ifc")
```

### What `apply_ifc_classification_properties()` Does

This utility function bridges the gap between bSDD property definitions and IFC property sets. When a bSDD class defines required properties (e.g., a "Load Bearing External Wall" classification requires a `LoadBearing` property set to `True`), this function:

1. Reads the `classProperties` array from the bSDD API response
2. Groups properties by their property set name
3. Creates `IfcPropertySet` entities in the IFC model
4. Creates `IfcPropertySingleValue` or `IfcPropertyEnumeratedValue` entities
5. Associates property sets with the target element via `IfcRelDefinesByProperties`

**Note on non-IFC property sets**: As of commit `4f64d04` (see IfcOpenShell issue #5404), Bonsai added support for assigning non-IFC bSDD properties. Previously, only bSDD classes that contained IFC-specific property sets would work. Now, custom property sets from any bSDD dictionary can be applied.

### bSDD-to-IFC Entity Mapping Reference

| bSDD Concept | IFC Entity | IFC Attribute |
|--------------|------------|---------------|
| Dictionary name | `IfcClassification` | `.Name` |
| Dictionary URI | `IfcClassification` | `.Specification` (IFC4x3) or `.Location` (IFC4) |
| Dictionary version | `IfcClassification` | `.Edition` |
| Organization code | `IfcClassification` | `.Source` |
| Release date | `IfcClassification` | `.EditionDate` |
| Class name | `IfcClassificationReference` | `.Name` |
| Class code | `IfcClassificationReference` | `.Identification` (IFC4+) |
| Class URI | `IfcClassificationReference` | `.Location` |
| Property code | `IfcPropertySingleValue` | `.Name` |
| Property URI | `IfcPropertySingleValue` | `.Specification` (IFC4x3) |
| Predefined value | `IfcPropertySingleValue` | `.NominalValue` |
| Allowed values | `IfcPropertyEnumeratedValue` | `.EnumerationValues` |
| Material | `IfcMaterial` | via `IfcExternalReferenceRelationship` |

## Common Classification Systems Available via bSDD

The following classification systems are among those published on bSDD. Each has a unique dictionary URI:

### By Region

**Netherlands:**
- **NL-SfB 2005** -- `https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2` -- The Dutch construction classification, widely used for element coding in BIM models. Based on the Swedish SfB system.

**United Kingdom:**
- **Uniclass 2015** -- The unified classification for the UK construction industry covering all sectors (buildings, infrastructure, landscape). Organized into tables: Complexes (Co), Entities (En), Activities (Ac), Spaces/Locations (SL), Elements (EF/Ss), Systems (Ss), Products (Pr).

**North America:**
- **OmniClass** -- The OmniClass Construction Classification System, organized into 15 tables covering the full construction lifecycle. Commonly used in the US and Canada.
- **MasterFormat** -- CSI's specification-focused classification, widely used for organizing construction specifications.
- **UniFormat** -- ASTM's element-based classification used for preliminary cost estimation and performance specification.

**Scandinavia:**
- **SfB** / **BSAB** -- Swedish classification systems that form the basis for several European national systems.

**Finland:**
- **Building 90** -- Finnish construction classification system.

### International / Cross-Region

- **IFC (buildingSMART)** -- `https://identifier.buildingsmart.org/uri/buildingsmart/ifc/4.3` -- The IFC schema itself is published as a bSDD dictionary, enabling cross-referencing between IFC entity types and other classification systems.
- **ETIM** -- International classification for technical products, commonly used in electrical and HVAC engineering.
- **CCI (Construction Classification International)** -- An emerging international classification standard.

### Filtering by IFC Entity

A key feature of bSDD is the `RelatedIfcEntityNamesList` field on each class. This field explicitly maps bSDD classes to their corresponding IFC entities. When searching bSDD, you ALWAYS filter by `related_ifc_entity` to narrow results to classifications relevant to the specific element type being classified. For example:

- Searching with `related_ifc_entity="IfcWall"` returns only wall-related codes
- Searching with `related_ifc_entity="IfcDoor"` returns only door-related codes

This prevents incorrect classification assignments and speeds up the lookup process.

## Relevance for Claude Skill Package

### Why bSDD Integration Matters

1. **Automated classification**: A Claude skill can look up the correct classification code for any IFC element by querying bSDD, eliminating manual code entry errors.
2. **Property population**: bSDD stores required properties per classification. A skill can AUTOMATICALLY populate mandatory property sets when a classification is assigned.
3. **Multi-standard support**: A single workflow supports NL-SfB (Netherlands), Uniclass (UK), OmniClass (US), and any other dictionary published on bSDD.
4. **Always up-to-date**: Unlike static classification files, bSDD API responses ALWAYS reflect the latest published version.

### Recommended Skill Architecture

The Claude skill package MUST document two integration paths:

**Path 1: bSDD Client (online, live lookup)**
- Use `from bsdd import Client` for real-time classification lookup
- Best for: interactive classification assignment, property discovery, validation
- Requires: internet connection

**Path 2: Classification Library Files (offline, pre-downloaded)**
- Use `ifcopenshell.open("Uniclass2015.ifc")` with pre-downloaded IFC libraries from https://github.com/Moult/IfcClassification
- Best for: batch processing, offline environments, performance-critical operations
- NEVER requires internet connection

### Key Code Patterns to Document

1. **Lookup pattern**: `Client().search_class()` -> `get_class()` -> extract identification/name -> `add_reference()`
2. **Property application pattern**: `get_class()` -> extract `classProperties` -> `apply_ifc_classification_properties()`
3. **Validation pattern**: `get_references(element)` -> verify classification code exists in bSDD -> flag mismatches
4. **Bulk classification pattern**: iterate `model.by_type("IfcWall")` -> batch `search_in_dictionary()` -> assign references

### Anti-patterns to Document

- NEVER use `identifier.buildingsmart.org` URIs for API calls -- use `api.bsdd.buildingsmart.org` endpoints
- NEVER hardcode classification codes without verifying them against bSDD
- NEVER skip the `related_ifc_entity` filter when searching -- it prevents incorrect classifications
- NEVER assume a bSDD class has IFC-specific property sets -- check `classProperties` first and handle non-IFC psets

## Sources

- [IfcOpenShell bSDD Documentation](https://docs.ifcopenshell.org/bsdd.html)
- [IfcOpenShell Classification API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/classification/index.html)
- [bSDD API Usage Guide](https://technical.buildingsmart.org/services/bsdd/using-the-bsdd-api/)
- [bSDD Data Structure](https://technical.buildingsmart.org/services/bsdd/data-structure/)
- [bSDD GitHub Repository](https://github.com/buildingSMART/bSDD)
- [bSDD API Documentation (GitHub)](https://github.com/buildingSMART/bSDD/blob/master/Documentation/bSDD%20API.md)
- [bSDD-IFC Mapping Documentation](https://github.com/buildingSMART/bSDD/blob/master/Documentation/bSDD-IFC%20documentation.md)
- [IfcOpenShell bSDD Source Code](https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bsdd)
- [GSoC bSDD Library Proposal (Completed)](https://github.com/opencax/GSoC/issues/40)
- [IfcOpenShell Issue #5404: Non-IFC bSDD Properties](https://github.com/IfcOpenShell/IfcOpenShell/issues/5404)
- [IfcOpenShell Code Examples](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)
- [buildingSMART Data Dictionary Overview](https://www.buildingsmart.org/users/services/buildingsmart-data-dictionary/)
- [IfcClassification IFC Standard](https://standards.buildingsmart.org/IFC/DEV/IFC4_2/FINAL/HTML/schema/ifcexternalreferenceresource/lexical/ifcclassification.htm)
- [IFC Classification Libraries](https://github.com/Moult/IfcClassification)
