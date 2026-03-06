# IFC Core Concepts — Anti-Patterns

## AP-001: Using String Comparison Instead of is_a() for Type Checking

**WRONG:**
```python
# Schema: IFC2X3, IFC4, IFC4X3
element = model.by_type("IfcWall")[0]

# WRONG — this only matches the exact entity type, NOT parent classes
if element.is_a() == "IfcElement":
    print("This is an element")  # NEVER prints for IfcWall
```

**WHY this is wrong:** `element.is_a()` without arguments returns the exact entity class name (e.g., "IfcWall"). Comparing it to a parent class like "IfcElement" ALWAYS fails because IfcWall is never literally equal to "IfcElement". This misses the entire inheritance chain.

**CORRECT:**
```python
# CORRECT — is_a() with argument traverses the full inheritance chain
if element.is_a("IfcElement"):
    print("This is an element")  # Prints for IfcWall, IfcSlab, IfcColumn, etc.
```

---

## AP-002: Hardcoding IfcBuildingElement Without Schema Awareness

**WRONG:**
```python
# Assumes IFC4X3 model uses IfcBuildingElement
elements = model.by_type("IfcBuildingElement")  # Returns EMPTY tuple in IFC4X3
```

**WHY this is wrong:** `IfcBuildingElement` was renamed to `IfcBuiltElement` in IFC4X3. Querying `IfcBuildingElement` in an IFC4X3 file returns an empty result because the entity does not exist under that name. This is a silent failure — no error is raised.

**CORRECT:**
```python
# CORRECT — version-aware element collection
def get_building_elements(model):
    if model.schema == "IFC4X3":
        return model.by_type("IfcBuiltElement")
    return model.by_type("IfcBuildingElement")

elements = get_building_elements(model)
```

---

## AP-003: Setting Relationships by Direct Attribute Assignment

**WRONG:**
```python
# Schema: IFC2X3, IFC4, IFC4X3
# WRONG — directly assigning an element to a storey attribute
wall.ContainedInStructure = storey  # ERROR or silently ignored

# WRONG — trying to add element to a relationship's related list directly
rel = model.by_type("IfcRelContainedInSpatialStructure")[0]
rel.RelatedElements = rel.RelatedElements + (wall,)  # Fragile and incomplete
```

**WHY this is wrong:** IFC relationships are objectified — they are first-class IfcRoot entities with their own GlobalId, OwnerHistory, and bidirectional references. Direct attribute assignment skips the creation of the relationship entity, does not generate required GUIDs, and does not update inverse references. The `ifcopenshell.api.run()` functions handle all of this correctly.

**CORRECT:**
```python
# CORRECT — use the API to create relationships
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])
```

---

## AP-004: Using create_entity() Instead of api.run() for Element Creation

**WRONG:**
```python
# Schema: IFC2X3, IFC4, IFC4X3
import ifcopenshell.guid

# WRONG — low-level creation misses relationships, owner history, etc.
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),
    Name="My Wall")
# Result: wall exists but has NO spatial containment, NO type, NO properties
```

**WHY this is wrong:** `create_entity()` and `createIfc*()` are low-level methods that create bare entities without establishing any relationships. The created element has no spatial container, no type assignment, no owner history (if required), and no property sets. The `ifcopenshell.api.run("root.create_entity", ...)` function handles GUID generation, OwnerHistory (schema-aware), and returns a properly initialized entity ready for relationship assignment.

**CORRECT:**
```python
# CORRECT — API handles GUID, OwnerHistory, and initialization
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="My Wall")
# Then assign spatial container, type, properties, etc.
```

---

## AP-005: Using IfcWallStandardCase in IFC4X3 Models

**WRONG:**
```python
# Schema: IFC4X3
# WRONG — IfcWallStandardCase was REMOVED in IFC4X3
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallStandardCase", name="Standard Wall")
# Raises RuntimeError: entity does not exist in schema

# WRONG — querying for StandardCase in IFC4X3
standard_walls = model.by_type("IfcWallStandardCase")  # Returns empty or errors
```

**WHY this is wrong:** IFC4 introduced `*StandardCase` subtypes (IfcWallStandardCase, IfcColumnStandardCase, IfcBeamStandardCase, IfcSlabStandardCase, IfcDoorStandardCase, IfcWindowStandardCase, IfcPlateStandardCase, IfcMemberStandardCase, IfcSlabElementedCase). IFC4X3 REMOVED all of them. The StandardCase distinction is now expressed through geometry usage and PredefinedType, not through separate entity types.

**CORRECT:**
```python
# CORRECT for IFC4X3 — use the base entity type
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Standard Wall")
# StandardCase behavior is implied by geometry usage, not entity type

# CORRECT — version-aware StandardCase handling
def get_standard_walls(model):
    if model.schema == "IFC4":
        return model.by_type("IfcWallStandardCase")
    # IFC2X3 and IFC4X3: no StandardCase entity
    return model.by_type("IfcWall")
```

---

## AP-006: Assuming OwnerHistory is Always Required

**WRONG:**
```python
# Schema: IFC4, IFC4X3
# WRONG — creating OwnerHistory when it is not needed
owner_history = ifcopenshell.api.run("owner.create_owner_history", model)
wall = model.createIfcWall(
    ifcopenshell.guid.new(),
    owner_history,  # Unnecessary boilerplate in IFC4+
    "MyWall"
)
```

**WHY this is wrong:** In IFC2X3, OwnerHistory is MANDATORY on all IfcRoot entities. In IFC4 and IFC4X3, OwnerHistory became OPTIONAL. Creating unnecessary OwnerHistory entities bloats the file and adds maintenance overhead. Code that assumes OwnerHistory is always required breaks when encountering IFC4+ files where it is legitimately None.

**CORRECT:**
```python
# CORRECT — let the API handle OwnerHistory based on schema version
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="MyWall")
# The API sets OwnerHistory appropriately for the schema version
```

---

## AP-007: Assuming IfcProject.is_a("IfcObject") is True in All Versions

**WRONG:**
```python
# Schema: IFC4, IFC4X3
project = model.by_type("IfcProject")[0]

# WRONG — this is only True in IFC2X3
if project.is_a("IfcObject"):
    print("Project is an object")  # False in IFC4/IFC4X3
```

**WHY this is wrong:** In IFC2X3, IfcProject inherits from IfcObject. In IFC4 and IFC4X3, IfcProject inherits from the new IfcContext abstract class (which is under IfcObjectDefinition, NOT under IfcObject). Code that checks `project.is_a("IfcObject")` silently returns False in IFC4+, leading to missed logic branches.

**CORRECT:**
```python
# CORRECT — use IfcObjectDefinition (common ancestor in all versions)
if project.is_a("IfcObjectDefinition"):
    print("Project is an object definition")  # True in ALL versions

# Or check version-specifically
if model.schema == "IFC2X3":
    assert project.is_a("IfcObject")    # True
else:
    assert project.is_a("IfcContext")    # True in IFC4+
```

---

## AP-008: Using IfcDoorStyle/IfcWindowStyle in IFC4+ Models

**WRONG:**
```python
# Schema: IFC4, IFC4X3
# WRONG — IfcDoorStyle is an IFC2X3 entity name
door_style = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDoorStyle", name="Standard Door")
# May raise error or create incorrect entity
```

**WHY this is wrong:** In IFC2X3, door and window types are called IfcDoorStyle and IfcWindowStyle. In IFC4 and IFC4X3, they were renamed to IfcDoorType and IfcWindowType. Using the old names in IFC4+ files creates invalid entities or raises errors.

**CORRECT:**
```python
# CORRECT — version-aware type creation
def create_door_type(model, name):
    if model.schema == "IFC2X3":
        return ifcopenshell.api.run("root.create_entity", model,
            ifc_class="IfcDoorStyle", name=name)
    return ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcDoorType", name=name)
```

---

## AP-009: Creating Infrastructure Entities in IFC2X3/IFC4 Models

**WRONG:**
```python
# Schema: IFC2X3 or IFC4
model = ifcopenshell.file(schema="IFC4")

# WRONG — IfcRoad does not exist in IFC4
road = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcRoad", name="Highway A1")
# Raises RuntimeError: entity does not exist in schema
```

**WHY this is wrong:** Infrastructure entities (IfcRoad, IfcBridge, IfcRailway, IfcMarineFacility, IfcFacility, IfcFacilityPart, and their subtypes) were introduced in IFC4X3. They do NOT exist in IFC2X3 or IFC4. ALWAYS check schema version before using infrastructure entities.

**CORRECT:**
```python
# CORRECT — verify schema before using infrastructure entities
if model.schema == "IFC4X3":
    road = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcRoad", name="Highway A1")
else:
    # Use IfcBuilding or IfcBuildingElementProxy as fallback
    pass
```

---

## AP-010: Querying IfcSpatialElement in IFC2X3

**WRONG:**
```python
# Schema: IFC2X3
# WRONG — IfcSpatialElement does not exist in IFC2X3
spatial_elements = model.by_type("IfcSpatialElement")  # Returns empty or errors
```

**WHY this is wrong:** IfcSpatialElement is the abstract supertype introduced in IFC4. In IFC2X3, the equivalent is IfcSpatialStructureElement (which is directly under IfcProduct). Querying IfcSpatialElement in an IFC2X3 model silently returns an empty result.

**CORRECT:**
```python
# CORRECT — version-aware spatial element query
def get_spatial_elements(model):
    if model.schema == "IFC2X3":
        return model.by_type("IfcSpatialStructureElement")
    return model.by_type("IfcSpatialElement")
```

---

## AP-011: Forgetting to Create Representation Context Before Geometry

**WRONG:**
```python
# Schema: IFC2X3, IFC4, IFC4X3
# WRONG — creating geometry without a representation context
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None, length=5.0, height=3.0, thickness=0.2)
# Fails: context is REQUIRED
```

**WHY this is wrong:** Every geometric representation in IFC MUST be assigned to a representation context (IfcGeometricRepresentationContext or its subcontext). The context defines the coordinate space, precision, and purpose of the representation (e.g., "Body" for 3D geometry, "Axis" for center lines). Without a context, the representation is invalid and viewers cannot interpret it.

**CORRECT:**
```python
# CORRECT — create context hierarchy first
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Then create geometry with the context
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

---

## AP-012: Using model.remove() Instead of api.run("root.remove_product")

**WRONG:**
```python
# Schema: IFC2X3, IFC4, IFC4X3
# WRONG — low-level remove does NOT clean up relationships
model.remove(wall)
# Result: orphaned IfcRelContainedInSpatialStructure, IfcRelDefinesByType,
# IfcRelDefinesByProperties, IfcRelAssociatesMaterial entities remain
```

**WHY this is wrong:** `model.remove()` is a low-level method that removes only the entity itself. All relationship entities (IfcRelContainedInSpatialStructure, IfcRelDefinesByType, IfcRelDefinesByProperties, IfcRelAssociatesMaterial, IfcRelVoidsElement, etc.) that reference the removed entity remain as orphaned, invalid references. This corrupts the IFC file.

**CORRECT:**
```python
# CORRECT — API removes entity AND cleans up all relationships
ifcopenshell.api.run("root.remove_product", model, product=wall)
```
