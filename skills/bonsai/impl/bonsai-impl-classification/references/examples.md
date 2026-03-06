# Classification Workflow Examples

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8.x | Python 3.11+

## Example 1: Classify Elements Using a Library File (Offline)

Load a pre-downloaded IFC classification library and assign codes to elements.

```python
import ifcopenshell
import ifcopenshell.api

# Open model and classification library
model = ifcopenshell.open("building.ifc")
library = ifcopenshell.open("Uniclass2015.ifc")

# Step 1: Copy classification system from library to model
lib_classification = library.by_type("IfcClassification")[0]
classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification=lib_classification
)
# classification.Name == "Uniclass 2015"

# Step 2: Find a specific code in the library
target_code = "Ss_25_10_30"
lib_references = library.by_type("IfcClassificationReference")
lib_ref = next(r for r in lib_references if r.Identification == target_code)

# Step 3: Assign to wall elements (lightweight = no parent hierarchy copied)
walls = model.by_type("IfcWall")
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=walls,
    reference=lib_ref,
    classification=classification,
    is_lightweight=True
)

model.write("building_classified.ifc")
```

---

## Example 2: Classify Using bSDD (Online Lookup)

Search bSDD for classification codes and apply them with associated properties.

```python
from bsdd import Client, apply_ifc_classification_properties
import ifcopenshell
import ifcopenshell.api

client = Client()
model = ifcopenshell.open("building.ifc")

# Step 1: Search NL-SfB for wall codes, filtered by IFC entity type
nlsfb_uri = "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2"
results = client.search_class(
    search_text="buitenwanden",
    dictionary_uris=nlsfb_uri,
    related_ifc_entities=["IfcWall"]
)

# Step 2: Get full class data with properties
class_uri = results["classes"][0]["uri"]
class_data = client.get_class(class_uri, include_class_properties=True)

# Step 3: Add classification system to model
classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="NL-SfB 2005"
)
ifcopenshell.api.run(
    "classification.edit_classification", model,
    classification=classification,
    attributes={
        "Source": "BIM Loket",
        "Edition": "2.2",
    }
)

# Step 4: Assign classification reference
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification=class_data["classCode"],
    name=class_data["name"],
    classification=classification
)

# Step 5: Apply associated properties from bSDD
if class_data.get("classProperties"):
    apply_ifc_classification_properties(
        model, wall, class_data["classProperties"]
    )

model.write("building_classified.ifc")
```

---

## Example 3: Manual/Custom Classification

Create a project-specific classification system without external sources.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

# Step 1: Create custom classification
classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="ProjectPhaseClassification"
)
ifcopenshell.api.run(
    "classification.edit_classification", model,
    classification=classification,
    attributes={
        "Source": "ACME Engineering",
        "Edition": "1.0",
        "Description": "Internal project phase classification"
    }
)

# Step 2: Classify types (preferred over occurrences for shared codes)
for wall_type in model.by_type("IfcWallType"):
    ifcopenshell.api.run(
        "classification.add_reference", model,
        products=[wall_type],
        identification="STR-W-001",
        name="Structural External Wall",
        classification=classification
    )

for slab_type in model.by_type("IfcSlabType"):
    ifcopenshell.api.run(
        "classification.add_reference", model,
        products=[slab_type],
        identification="STR-S-001",
        name="Structural Floor Slab",
        classification=classification
    )

model.write("building_classified.ifc")
```

---

## Example 4: Multiple Classification Systems on One Element

IFC supports assigning codes from multiple classification systems simultaneously.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Add Uniclass
uniclass = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="Uniclass 2015"
)
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=uniclass
)

# Add NL-SfB to the same element
nlsfb = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="NL-SfB 2005"
)
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="21.21",
    name="Buitenwanden",
    classification=nlsfb
)

# Verify both are assigned
import ifcopenshell.util.classification
refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    system = ifcopenshell.util.classification.get_classification(ref)
    print(f"{system.Name}: {ref.Identification} - {ref.Name}")
# Output:
# Uniclass 2015: Ss_25_10_30 - Wall structures
# NL-SfB 2005: 21.21 - Buitenwanden

model.write("building_multi_classified.ifc")
```

---

## Example 5: Read and Validate Classifications

Query existing classifications and verify correctness.

```python
import ifcopenshell
import ifcopenshell.util.classification

model = ifcopenshell.open("building.ifc")

# Iterate all classified elements
for rel in model.by_type("IfcRelAssociatesClassification"):
    ref = rel.RelatingClassification
    if ref.is_a("IfcClassificationReference"):
        system = ifcopenshell.util.classification.get_classification(ref)
        for product in rel.RelatedObjects:
            print(f"{product.is_a()}: {product.Name} -> "
                  f"{system.Name} [{ref.Identification}] {ref.Name}")

# Check if a specific element has a specific classification system
element = model.by_type("IfcWall")[0]
refs = ifcopenshell.util.classification.get_references(element, should_inherit=True)
has_uniclass = any(
    ifcopenshell.util.classification.get_classification(r).Name == "Uniclass 2015"
    for r in refs
)
print(f"Has Uniclass: {has_uniclass}")
```

---

## Example 6: Classification Inheritance (Type vs Occurrence)

Classifications on types are inherited by occurrences unless overridden.

```python
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.classification

model = ifcopenshell.open("building.ifc")

classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="Uniclass 2015"
)

# Classify the TYPE (inherited by all occurrences)
wall_type = model.by_type("IfcWallType")[0]
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall_type],
    identification="Ss_25_10_30",
    name="Wall structures",
    classification=classification
)

# Any IfcWall occurrence of this type inherits the classification
wall = model.by_type("IfcWall")[0]
refs = ifcopenshell.util.classification.get_references(wall, should_inherit=True)
# refs contains the "Ss_25_10_30" reference from the type

# Override on a specific occurrence (overrides type for this system)
ifcopenshell.api.run(
    "classification.add_reference", model,
    products=[wall],
    identification="Ss_25_10_65",
    name="Retaining wall structures",
    classification=classification
)
# Now this wall has "Ss_25_10_65", not "Ss_25_10_30"
```

---

## Example 7: Remove Classifications

Remove individual references or entire classification systems.

```python
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.classification

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Remove a specific reference from one element
refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    system = ifcopenshell.util.classification.get_classification(ref)
    if system.Name == "NL-SfB 2005":
        ifcopenshell.api.run(
            "classification.remove_reference", model,
            reference=ref,
            products=[wall]
        )

# Remove an entire classification system (WARNING: removes ALL references)
for classification in model.by_type("IfcClassification"):
    if classification.Name == "OmniClass":
        ifcopenshell.api.run(
            "classification.remove_classification", model,
            classification=classification
        )

model.write("building_declassified.ifc")
```

---

## Example 8: Bulk Classification by Element Type

Classify all elements of a given IFC class with a single code.

```python
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

classification = ifcopenshell.api.run(
    "classification.add_classification", model,
    classification="Uniclass 2015"
)

# Define classification mapping
type_mapping = {
    "IfcWall": ("Ss_25_10_30", "Wall structures"),
    "IfcSlab": ("Ss_25_30_30", "Floor structures"),
    "IfcColumn": ("Ss_25_12_30", "Column structures"),
    "IfcBeam": ("Ss_25_11_30", "Beam structures"),
    "IfcDoor": ("Pr_30_59_24", "Doors"),
    "IfcWindow": ("Pr_30_59_98", "Windows"),
}

for ifc_class, (code, name) in type_mapping.items():
    elements = model.by_type(ifc_class)
    if elements:
        ifcopenshell.api.run(
            "classification.add_reference", model,
            products=elements,
            identification=code,
            name=name,
            classification=classification
        )
        print(f"Classified {len(elements)} {ifc_class} elements as {code}")

model.write("building_bulk_classified.ifc")
```

---

## Example 9: bSDD Dictionary Browsing

Explore available dictionaries and their class hierarchies.

```python
from bsdd import Client

client = Client()

# List all available dictionaries
response = client.get_dictionary()
for d in response.get("dictionaries", []):
    print(f"{d['name']} v{d.get('version', '?')} by {d.get('organizationNameOwner', '?')}")
    print(f"  URI: {d['uri']}")

# Get class tree for a specific dictionary
nlsfb_uri = "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2"
classes = client.get_classes(
    dictionary_uri=nlsfb_uri,
    related_ifc_entity="IfcWall"
)
for cls in classes.get("classes", []):
    print(f"  {cls.get('referenceCode', '?')}: {cls.get('name', '?')}")

# Get properties for a specific class
class_data = client.get_class(
    "https://identifier.buildingsmart.org/uri/nlsfb/nlsfb2005/2.2/class/21.21",
    include_class_properties=True
)
for prop in class_data.get("classProperties", []):
    print(f"  Property: {prop.get('name')} ({prop.get('propertySet', 'N/A')})")
```

---

## Example 10: Bonsai Operator Usage (Blender Context)

Using Bonsai operators from within Blender's Python console or scripts.

```python
import bpy
from bonsai.bim.ifc import IfcStore

# Load a classification library file
bpy.ops.bim.load_classification_library(filepath="/path/to/Uniclass2015.ifc")

# The library is now in IfcStore.classification_file
lib = IfcStore.classification_file
print(f"Loaded: {lib.by_type('IfcClassification')[0].Name}")

# Add classification from loaded library (uses scene props)
bpy.ops.bim.add_classification()

# Navigate the classification tree
# bpy.ops.bim.change_classification_level(parent_id=<ref_id>)

# Add reference to selected objects
# bpy.ops.bim.add_classification_reference(reference=<ref_id>)

# Using bSDD operators
bpy.ops.bim.load_bsdd_dictionaries()  # Fetches from bSDD API
bpy.ops.bim.search_bsdd_classifications()  # Searches using keyword from props

# Add bSDD classification reference (auto-creates psets)
# bpy.ops.bim.add_classification_reference_from_bsdd()
```
