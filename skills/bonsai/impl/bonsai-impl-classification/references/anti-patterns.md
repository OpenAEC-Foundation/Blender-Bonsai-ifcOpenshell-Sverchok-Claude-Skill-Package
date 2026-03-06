# Classification Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x | Python 3.11+

## Anti-Pattern 1: Using `product` Instead of `products`

**Wrong:**
```python
# WRONG: 'product' is not a valid parameter
ifcopenshell.api.run(
    "classification.add_reference", model,
    product=wall,  # WRONG — singular
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=classification
)
```

**Correct:**
```python
# CORRECT: ALWAYS use 'products' (list)
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],  # CORRECT — list, even for single element
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=classification
)
```

**Why:** The `add_reference` API signature requires `products` as a list. Passing `product` (singular) silently fails — no error is raised, but no classification is assigned. This is the most common AI mistake in classification code.

---

## Anti-Pattern 2: Using Identifier URIs for API Calls

**Wrong:**
```python
# WRONG: identifier.buildingsmart.org is for browser viewing, NOT API calls
import requests
response = requests.get(
    "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2/class/21.21"
)
```

**Correct:**
```python
# CORRECT: Use the bSDD Client which calls api.bsdd.buildingsmart.org
from bsdd import Client

client = Client()
class_data = client.get_class(
    "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2/class/21.21",
    include_class_properties=True
)
```

**Why:** `identifier.buildingsmart.org` URIs are permanent identifiers, not API endpoints. They return HTML pages for browsers, not JSON for code. The `bsdd.Client` correctly routes requests to `api.bsdd.buildingsmart.org` while accepting identifier URIs as parameters.

---

## Anti-Pattern 3: Omitting IFC Entity Filter in bSDD Searches

**Wrong:**
```python
# WRONG: unfiltered search returns irrelevant results
results = client.search_class(
    search_text="concrete",
    dictionary_uris=nlsfb_uri
    # Missing related_ifc_entities!
)
# Returns concrete walls, concrete columns, concrete beams, concrete materials...
```

**Correct:**
```python
# CORRECT: ALWAYS filter by IFC entity type
results = client.search_class(
    search_text="concrete",
    dictionary_uris=nlsfb_uri,
    related_ifc_entities=["IfcColumn"]  # Only column-related codes
)
```

**Why:** Without the `related_ifc_entities` filter, bSDD returns ALL matching classes across all entity types. This leads to incorrect classification assignments — e.g., assigning a wall code to a column. The filter maps bSDD classes to their `RelatedIfcEntityNamesList`, ensuring only applicable codes are returned.

---

## Anti-Pattern 4: Hardcoding Classification Codes Without Verification

**Wrong:**
```python
# WRONG: hardcoded code may not exist or may be outdated
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="EF_25_10",  # Guessed code — may not exist
    name="Wall systems",
    classification=classification
)
```

**Correct:**
```python
# CORRECT: Verify code exists in library or bSDD first
library = ifcopenshell.open("Uniclass2015.ifc")
valid_refs = library.by_type("IfcClassificationReference")
valid_codes = {r.Identification for r in valid_refs}

code = "EF_25_10"
if code in valid_codes:
    lib_ref = next(r for r in valid_refs if r.Identification == code)
    ifcopenshell.api.run(
        "classification.add_reference", model,
        products=[wall],
        reference=lib_ref,
        classification=classification,
        is_lightweight=True
    )
else:
    print(f"WARNING: Code '{code}' not found in Uniclass 2015 library")
```

**Why:** Classification codes differ between editions (e.g., Uniclass table codes changed between versions). Hardcoding unverified codes creates invalid IFC data that fails BIM validation checks. ALWAYS verify against the source library or bSDD API.

---

## Anti-Pattern 5: Forgetting to Link Classification to Project

**Wrong:**
```python
# WRONG: Creating IfcClassification directly without project association
classification = model.create_entity("IfcClassification", Name="Uniclass 2015")
# No IfcRelAssociatesClassification → orphaned entity
```

**Correct:**
```python
# CORRECT: Use the API which creates the relationship automatically
classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="Uniclass 2015"
)
# API creates IfcRelAssociatesClassification linking to IfcProject
```

**Why:** `ifcopenshell.api.run("classification.add_classification", ...)` creates both the `IfcClassification` entity AND an `IfcRelAssociatesClassification` linking it to the `IfcProject`. Creating the entity directly with `model.create_entity()` produces an orphaned classification that is invisible to viewers and validators. NEVER bypass the API for classification operations.

---

## Anti-Pattern 6: Assuming IFC2X3 Has the Same Attributes as IFC4

**Wrong:**
```python
# WRONG: 'Identification' does not exist in IFC2X3
ref = model.create_entity(
    "IfcClassificationReference",
    Identification="21.21",  # WRONG for IFC2X3
    Name="Buitenwanden"
)
```

**Correct:**
```python
# CORRECT: Use the API which handles schema migration automatically
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="21.21",  # API maps to ItemReference for IFC2X3
    name="Buitenwanden",
    classification=classification
)
```

**Why:** In IFC2X3, the code attribute is `ItemReference`, not `Identification`. In IFC4+, it is `Identification`. The IfcOpenShell API handles this mapping transparently. Similarly, `IfcClassification.Location` in IFC4 corresponds to `IfcClassification.Specification` in IFC4X3. ALWAYS use the API rather than direct entity creation to avoid schema-specific bugs.

---

## Anti-Pattern 7: Removing Classification System When Only Reference Removal Is Needed

**Wrong:**
```python
# WRONG: Removes the ENTIRE classification system and ALL references
ifcopenshell.api.run(
    "classification.remove_classification", model,
    classification=classification
)
# All elements in the model lose their Uniclass codes!
```

**Correct:**
```python
# CORRECT: Remove only the specific reference from specific products
ifcopenshell.api.run(
    "classification.remove_reference", model,
    reference=ref,
    products=[wall]
)
```

**Why:** `remove_classification` recursively collects and deletes ALL child `IfcClassificationReference` entities before removing the `IfcClassification` itself. This is a destructive operation that declassifies every element in the model. Use `remove_reference` to unassign individual codes from specific elements.

---

## Anti-Pattern 8: Classifying Occurrences When Types Should Be Classified

**Wrong:**
```python
# WRONG: Classifying each wall occurrence individually
for wall in model.by_type("IfcWall"):
    ifcopenshell.api.run(
        "classification.add_reference", model,
        products=[wall],
        identification="Ss_25_10_30",
        name="Wall structures",
        classification=classification
    )
# Creates N separate IfcClassificationReference entities
```

**Correct:**
```python
# CORRECT: Classify the type — occurrences inherit via should_inherit=True
wall_types = model.by_type("IfcWallType")
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=wall_types,
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=classification
)
# All IfcWall occurrences of these types inherit the classification
```

**Why:** In IFC, classifications assigned to an `IfcTypeObject` (e.g., `IfcWallType`) are inherited by all occurrences (e.g., `IfcWall`) of that type. Classifying occurrences individually creates redundant `IfcClassificationReference` entities and inflates file size. ALWAYS classify types when all occurrences share the same code. Only classify individual occurrences to override the type-level classification.

---

## Anti-Pattern 9: Mixing Library References and Custom Identification

**Wrong:**
```python
# WRONG: Providing both a library reference AND identification/name
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    reference=lib_ref,           # From library file
    identification="Ss_25_10_30",  # Custom — conflicts with reference
    classification=classification
)
```

**Correct:**
```python
# CORRECT: Use EITHER library reference OR custom identification, not both
# Option A: From library
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    reference=lib_ref,
    classification=classification,
    is_lightweight=True
)

# Option B: Custom
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=classification
)
```

**Why:** The `add_reference` API has two mutually exclusive paths: library-based (using `reference` parameter) and custom (using `identification`/`name` parameters). Providing both creates undefined behavior where the library reference is migrated but then custom attributes may be ignored or partially applied.

---

## Anti-Pattern 10: Not Checking for Duplicate Classifications

**Wrong:**
```python
# WRONG: Adding the same classification system multiple times
ifcopenshell.api.run("classification.add_classification", model, classification="Uniclass 2015")
ifcopenshell.api.run("classification.add_classification", model, classification="Uniclass 2015")
# Model now has TWO IfcClassification entities named "Uniclass 2015"
```

**Correct:**
```python
# CORRECT: Check if the classification already exists
existing = [c for c in model.by_type("IfcClassification") if c.Name == "Uniclass 2015"]
if existing:
    classification = existing[0]
else:
    classification = ifcopenshell.api.run(
        "classification.add_classification", model,
        classification="Uniclass 2015"
    )
```

**Why:** The API does not check for duplicate classification systems. Calling `add_classification` multiple times with the same name creates duplicate `IfcClassification` entities. This causes confusion in Bonsai's UI (duplicate entries in dropdowns) and ambiguity when assigning references. ALWAYS check for existing classifications before adding.
