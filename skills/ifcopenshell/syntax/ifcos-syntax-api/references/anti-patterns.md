# ifcos-syntax-api — Anti-Patterns and Common Mistakes

Mistakes documented from real issues, forums, and known AI hallucination patterns.

---

## Anti-Pattern 1: Hallucinated API Module Names

**WRONG — These modules do NOT exist:**
```python
# WRONG: "element" module does not exist
ifcopenshell.api.run("element.create", model, ifc_class="IfcWall")

# WRONG: "wall" module does not exist
ifcopenshell.api.run("wall.add", model, name="Wall 1")

# WRONG: "property" module does not exist
ifcopenshell.api.run("property.add_set", model, element=wall, name="Pset_WallCommon")

# WRONG: "space" module does not exist
ifcopenshell.api.run("space.assign", model, element=wall, space=storey)

# WRONG: "create" module does not exist
ifcopenshell.api.run("create.wall", model, name="Wall 1")

# WRONG: "ifc" module does not exist
ifcopenshell.api.run("ifc.create_entity", model, ifc_class="IfcWall")
```

**CORRECT equivalents:**
```python
# root.create_entity — the ONLY way to create IFC entities via API
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")

# pset.add_pset — property sets use the "pset" module
pset = ifcopenshell.api.run("pset.add_pset", model, product=wall, name="Pset_WallCommon")

# spatial.assign_container — spatial placement uses the "spatial" module
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

**Rule:** ALWAYS check the 35-module table in the SKILL.md before writing an api.run() call.

---

## Anti-Pattern 2: Direct Attribute Modification

**WRONG — Direct attribute assignment bypasses validation:**
```python
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")

# WRONG: Direct assignment does NOT update OwnerHistory or validate values
wall.Name = "Wall 001"
wall.Description = "External wall"
wall.PredefinedType = "SOLIDWALL"
```

**CORRECT — Use attribute.edit_attributes:**
```python
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")

# CORRECT: API validates and tracks changes
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=wall, attributes={
        "Name": "Wall 001",
        "Description": "External wall",
        "PredefinedType": "SOLIDWALL",
    })
```

**Exception:** The `name=` parameter in `root.create_entity` sets the name at creation time — that IS correct.

---

## Anti-Pattern 3: Single Element Instead of List (v0.8+ Breaking Change)

**WRONG — Passing single element where list is expected:**
```python
# WRONG (v0.8+): product= is not a valid parameter
ifcopenshell.api.run("spatial.assign_container", model,
    product=wall, relating_structure=storey)

# WRONG (v0.8+): related_object= expects a list
ifcopenshell.api.run("type.assign_type", model,
    related_object=wall, relating_type=wall_type)

# WRONG (v0.8+): product= should be products= (list)
ifcopenshell.api.run("material.assign_material", model,
    product=wall, material=concrete)
```

**CORRECT — ALWAYS use list parameters:**
```python
# CORRECT: products= is a list
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# CORRECT: related_objects= is a list
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)

# CORRECT: products= is a list
ifcopenshell.api.run("material.assign_material", model,
    products=[wall], material=concrete)
```

**Rule:** If the parameter name is plural (`products`, `related_objects`, `items`), it ALWAYS expects a list.

---

## Anti-Pattern 4: Skipping Project Bootstrap

**WRONG — Creating elements without proper project setup:**
```python
model = ifcopenshell.file(schema="IFC4")
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
model.write("output.ifc")  # Invalid IFC: no project, no units, no hierarchy
```

**CORRECT — ALWAYS set up the complete project first:**
```python
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

site = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSite")
building = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuilding")
storey = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcBuildingStorey")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# NOW create elements
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
```

**Required bootstrap order:** file → project → units → contexts → spatial hierarchy.

---

## Anti-Pattern 5: Using model.remove() for Products

**WRONG — model.remove() leaves dangling references:**
```python
wall = model.by_type("IfcWall")[0]
model.remove(wall)
# Danger: psets, material associations, containment, type associations
# are NOT cleaned up. References become null ($).
```

**CORRECT — Use root.remove_product for safe deletion:**
```python
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run("root.remove_product", model, product=wall)
# Safely removes: geometry, placement, properties, materials,
# containment, aggregation, nesting, type associations.
```

**Rule:** `model.remove()` is ONLY safe for non-product entities (e.g., removing a single property). For IfcElement, IfcElementType, IfcSpatialElement — ALWAYS use `root.remove_product`.

---

## Anti-Pattern 6: Wrong Parameter Name for project.create_file

**WRONG — Using schema= instead of version=:**
```python
# WRONG: parameter is "version", not "schema"
model = ifcopenshell.api.run("project.create_file", schema="IFC4")

# WRONG: confusing ifcopenshell.file() with project.create_file()
model = ifcopenshell.api.run("project.create_file", schema_version=(4, 0, 2, 1))
```

**CORRECT:**
```python
# CORRECT: project.create_file uses "version="
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# For bare files (testing only): ifcopenshell.file() uses "schema="
model = ifcopenshell.file(schema="IFC4")
```

---

## Anti-Pattern 7: Missing Context Before Geometry

**WRONG — Adding geometry without a context:**
```python
# WRONG: no context exists yet
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None, length=5.0, height=3.0, thickness=0.2)  # Error!
```

**CORRECT — Create root context and subcontext first:**
```python
# Step 1: Root context
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")

# Step 2: Subcontext (Body for 3D geometry)
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Step 3: NOW create geometry using the subcontext
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

**Rule:** ALWAYS create `context.add_context(context_type="Model")` first (root), then create subcontexts with `context_identifier` and `parent`.

---

## Anti-Pattern 8: Assigning Materials to Occurrences Instead of Types

**WRONG — Per-occurrence material assignment:**
```python
wall1 = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
wall2 = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")

# WRONG: Assigning material to each occurrence separately
ifcopenshell.api.run("material.assign_material", model,
    products=[wall1], material=concrete)
ifcopenshell.api.run("material.assign_material", model,
    products=[wall2], material=concrete)
```

**CORRECT — Assign material to the type, then assign type to occurrences:**
```python
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="WT-200")

# Assign material to type ONCE
ifcopenshell.api.run("material.assign_material", model,
    products=[wall_type], material=concrete)

# Create occurrences and assign type
wall1 = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
wall2 = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall1, wall2], relating_type=wall_type)
# Both walls now inherit the material from their type
```

**Rule:** Materials, property sets, and representations SHOULD be assigned to types. Occurrences inherit from their type.

---

## Anti-Pattern 9: Using model.create_entity() for Production Code

**WRONG — Low-level entity creation:**
```python
import ifcopenshell.guid

# WRONG: model.create_entity() does NOT generate GlobalId, OwnerHistory, or validate types
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),
    Name="Wall 001")
```

**CORRECT — Use the API:**
```python
# CORRECT: api creates GlobalId, OwnerHistory, validates predefined_type
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 001", predefined_type="SOLIDWALL")
```

**Exception:** `model.create_entity()` is acceptable in unit tests or when creating non-rooted helper entities. For any IfcRoot subclass (products, types, spatial elements), ALWAYS use `root.create_entity`.

---

## Anti-Pattern 10: Forgetting to assign_representation After Creating Geometry

**WRONG — Creating geometry but not connecting it to the element:**
```python
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
# Geometry created but wall has NO geometry — invisible in viewers!
```

**CORRECT — ALWAYS assign and place:**
```python
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)

# Step 1: Connect geometry to element
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)

# Step 2: Set placement (at minimum, identity matrix for origin)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
```

**Rule:** Creating geometry has THREE steps: (1) create representation, (2) assign to product, (3) set placement.

---

## Anti-Pattern 11: Wrong Aggregation Direction

**WRONG — Reversed parent-child relationship:**
```python
# WRONG: project is the parent, not the child
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[project], relating_object=site)  # Project under site!
```

**CORRECT — Parent (whole) is relating_object, children (parts) are products:**
```python
# CORRECT: Site is a part of Project
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)

# CORRECT hierarchy:
# Project → Site → Building → Storey → Space
```

**Rule:** `relating_object` = parent/whole, `products` = children/parts. The hierarchy ALWAYS flows: IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace.

---

## Anti-Pattern 12: Mixing Containment and Aggregation

**WRONG — Using spatial.assign_container for spatial hierarchy:**
```python
# WRONG: spatial.assign_container is for elements IN spaces, not for spaces IN buildings
ifcopenshell.api.run("spatial.assign_container", model,
    products=[storey], relating_structure=building)
```

**CORRECT — Use aggregate for hierarchy, spatial for containment:**
```python
# CORRECT: Spatial hierarchy uses AGGREGATION
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# CORRECT: Physical elements IN spaces uses CONTAINMENT
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

**Rule:**
- `aggregate.assign_object` = "is part of" (spatial hierarchy: Project→Site→Building→Storey)
- `spatial.assign_container` = "is located in" (elements inside spatial structures)
- `spatial.reference_structure` = "is also visible in" (multi-storey elements)

---

## Anti-Pattern 13: Forgetting to Import ifcopenshell.api

**WRONG — Calling api functions without importing the api module:**
```python
import ifcopenshell
# Missing: import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
# AttributeError: module 'ifcopenshell' has no attribute 'api'
```

**CORRECT:**
```python
import ifcopenshell
import ifcopenshell.api  # REQUIRED for api.run() to work

model = ifcopenshell.file(schema="IFC4")
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
```

---

## Anti-Pattern 14: Ignoring Return Values

**WRONG — Not capturing the returned entity:**
```python
# WRONG: Return value ignored — cannot reference the wall later
ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall", name="Wall 1")

# Now how do you assign it to a storey? You've lost the reference!
```

**CORRECT:**
```python
# CORRECT: Capture the return value
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall 1")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)
```

**Rule:** ALWAYS capture return values from functions that create entities. You will need these references for subsequent API calls.

---

## Sources

- IfcOpenShell GitHub Issues
- IfcOpenShell community forums
- docs.ifcopenshell.org changelog
- Observed AI hallucination patterns in IFC code generation
