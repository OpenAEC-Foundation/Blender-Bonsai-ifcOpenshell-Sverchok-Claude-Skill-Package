# Working Debug and Error Handling Examples

## Example 1: Robust IFC File Opening with Error Handling

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.unit

def open_ifc_safely(filepath):
    """Open an IFC file with comprehensive error handling."""
    try:
        model = ifcopenshell.open(filepath)
    except FileNotFoundError:
        print(f"ERROR: File not found: {filepath}")
        return None
    except ifcopenshell.Error as e:
        print(f"ERROR: Invalid or corrupt IFC file: {e}")
        return None
    except Exception as e:
        print(f"ERROR: Unexpected error opening file: {e}")
        return None

    # Report file info
    print(f"Schema: {model.schema}")
    print(f"Entities: {len(model)}")

    unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
    print(f"Unit scale to meters: {unit_scale}")

    return model
```

## Example 2: Schema-Safe Entity Creation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

def create_wall_safe(model, name, storey):
    """Create a wall entity that works with any IFC schema version."""
    schema = model.schema

    # Choose correct entity class for schema
    if schema == "IFC2X3":
        ifc_class = "IfcWallStandardCase"
    else:
        ifc_class = "IfcWall"

    # Use API to handle OwnerHistory (required in IFC2X3) automatically
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class=ifc_class, name=name)

    # ALWAYS assign spatial containment
    if storey is not None:
        ifcopenshell.api.run("spatial.assign_container", model,
            products=[wall], relating_structure=storey)
    else:
        print(f"WARNING: Wall '{name}' has no spatial container")

    return wall
```

## Example 3: Safe Property Set Access

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

def get_property_safe(element, pset_name, prop_name, default=None):
    """Safely get a property value from an IFC element."""
    psets = ifcopenshell.util.element.get_psets(element)

    pset = psets.get(pset_name)
    if pset is None:
        return default

    value = pset.get(prop_name)
    if value is None:
        return default

    return value


# Usage
fire_rating = get_property_safe(wall, "Pset_WallCommon", "FireRating", default="Unknown")
is_external = get_property_safe(wall, "Pset_WallCommon", "IsExternal", default=False)
```

## Example 4: Safe Geometry Processing with Error Recovery

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import numpy as np

def process_geometry_safe(model, elements, use_world_coords=True):
    """Process geometry for multiple elements with error handling."""
    settings = ifcopenshell.geom.settings()
    settings.set(settings.USE_WORLD_COORDS, use_world_coords)

    results = []
    failures = []

    for element in elements:
        # Skip elements without geometry representation
        if not hasattr(element, "Representation") or element.Representation is None:
            continue

        try:
            shape = ifcopenshell.geom.create_shape(settings, element)
            if shape is None:
                failures.append({
                    "id": element.id(),
                    "type": element.is_a(),
                    "error": "create_shape returned None"
                })
                continue

            verts = np.array(shape.geometry.verts).reshape(-1, 3)
            faces = np.array(shape.geometry.faces).reshape(-1, 3)

            results.append({
                "id": element.id(),
                "guid": element.GlobalId,
                "name": element.Name,
                "vertices": verts,
                "faces": faces
            })
        except RuntimeError as e:
            failures.append({
                "id": element.id(),
                "type": element.is_a(),
                "error": str(e)
            })

    if failures:
        print(f"WARNING: {len(failures)} elements failed geometry processing:")
        for f in failures:
            print(f"  {f['type']} #{f['id']}: {f['error']}")

    return results, failures
```

## Example 5: Batch Geometry Processing with Iterator

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

def process_geometry_batch(model, entity_type="IfcProduct"):
    """Process geometry using iterator for better performance on large files."""
    settings = ifcopenshell.geom.settings()
    settings.set(settings.USE_WORLD_COORDS, True)

    elements = model.by_type(entity_type)
    if not elements:
        print(f"No {entity_type} elements found")
        return []

    iterator = ifcopenshell.geom.iterator(
        settings, model,
        multiprocessing.cpu_count(),
        include=elements
    )

    results = []
    if iterator.initialize():
        while True:
            shape = iterator.get()
            element = model.by_id(shape.id)

            results.append({
                "id": shape.id,
                "guid": element.GlobalId,
                "name": element.Name,
                "vertex_count": len(shape.geometry.verts) // 3,
                "face_count": len(shape.geometry.faces) // 3
            })

            if not iterator.next():
                break

    print(f"Processed geometry for {len(results)} elements")
    return results
```

## Example 6: Safe Bulk Element Removal

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

def remove_elements_safe(model, elements):
    """Safely remove multiple elements with relationship cleanup."""
    # CRITICAL: Collect into list first to avoid iterator invalidation
    elements_to_remove = list(elements)

    # Extract identifiers BEFORE removal (entities become invalid after remove)
    removal_log = []
    for element in elements_to_remove:
        removal_log.append({
            "id": element.id(),
            "type": element.is_a(),
            "guid": element.GlobalId,
            "name": element.Name
        })

    # Remove using API (handles relationship cleanup)
    removed = 0
    for element in elements_to_remove:
        try:
            ifcopenshell.api.run("root.remove_product", model, product=element)
            removed += 1
        except RuntimeError as e:
            print(f"WARNING: Failed to remove {element.is_a()} #{element.id()}: {e}")

    # Clean up orphaned entities
    model.garbage_collect()

    print(f"Removed {removed}/{len(elements_to_remove)} elements")
    return removal_log
```

## Example 7: Property Set Creation Without Duplicates

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

def set_properties_safe(model, element, pset_name, properties):
    """Set properties on an element, creating or updating the property set."""
    existing_psets = ifcopenshell.util.element.get_psets(element)

    if pset_name in existing_psets:
        # Update existing property set
        pset = model.by_id(existing_psets[pset_name]["id"])
    else:
        # Create new property set
        pset = ifcopenshell.api.run("pset.add_pset", model,
            product=element, name=pset_name)

    # Set/update properties
    ifcopenshell.api.run("pset.edit_pset", model,
        pset=pset, properties=properties)

    return pset


# Usage
set_properties_safe(model, wall, "Pset_WallCommon", {
    "IsExternal": True,
    "FireRating": "REI60",
    "ThermalTransmittance": 0.24
})
```

## Example 8: Comprehensive Model Validation

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit

def validate_model(model):
    """Run validation checks on an IFC model and report issues."""
    issues = []

    # Check 1: Schema version
    schema = model.schema
    print(f"Schema: {schema}")

    # Check 2: Project exists
    projects = model.by_type("IfcProject")
    if not projects:
        issues.append("CRITICAL: No IfcProject found")
    elif len(projects) > 1:
        issues.append("WARNING: Multiple IfcProject entities found")

    # Check 3: Units defined
    try:
        unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
        print(f"Unit scale: {unit_scale}")
    except Exception:
        issues.append("CRITICAL: No length units defined")

    # Check 4: Spatial structure exists
    sites = model.by_type("IfcSite")
    buildings = model.by_type("IfcBuilding")
    storeys = model.by_type("IfcBuildingStorey")

    if not sites:
        issues.append("WARNING: No IfcSite found")
    if not buildings:
        issues.append("WARNING: No IfcBuilding found")
    if not storeys:
        issues.append("WARNING: No IfcBuildingStorey found")

    # Check 5: Elements without spatial containment
    products = model.by_type("IfcProduct")
    uncontained = []
    for product in products:
        if product.is_a("IfcSpatialStructureElement"):
            continue  # Spatial elements use aggregation, not containment
        container = ifcopenshell.util.element.get_container(product)
        if container is None and hasattr(product, "Representation"):
            if product.Representation is not None:
                uncontained.append(product)

    if uncontained:
        issues.append(
            f"WARNING: {len(uncontained)} products without spatial containment")
        for p in uncontained[:5]:
            issues.append(f"  - {p.is_a()} '{p.Name}' #{p.id()}")

    # Check 6: Duplicate GUIDs
    guids = {}
    for entity in model.by_type("IfcRoot"):
        guid = entity.GlobalId
        if guid in guids:
            issues.append(
                f"CRITICAL: Duplicate GUID '{guid}' on "
                f"{guids[guid].is_a()} #{guids[guid].id()} and "
                f"{entity.is_a()} #{entity.id()}")
        else:
            guids[guid] = entity

    # Report
    if issues:
        print(f"\n{len(issues)} issues found:")
        for issue in issues:
            print(f"  {issue}")
    else:
        print("\nNo issues found")

    return issues
```

## Example 9: Schema-Agnostic Type Matching

```python
# IfcOpenShell — all schema versions
import ifcopenshell

# Element-to-Type mapping (schema-aware)
ELEMENT_TYPE_MAP = {
    "IfcWall": "IfcWallType",
    "IfcWallStandardCase": "IfcWallType",
    "IfcSlab": "IfcSlabType",
    "IfcColumn": "IfcColumnType",
    "IfcBeam": "IfcBeamType",
    "IfcPlate": "IfcPlateType",
    "IfcMember": "IfcMemberType",
    "IfcCovering": "IfcCoveringType",
}

# IFC2X3-specific type mappings (Style vs Type)
IFC2X3_TYPE_MAP = {
    "IfcDoor": "IfcDoorStyle",
    "IfcWindow": "IfcWindowStyle",
}

# IFC4/IFC4X3 type mappings
IFC4_TYPE_MAP = {
    "IfcDoor": "IfcDoorType",
    "IfcWindow": "IfcWindowType",
}


def get_matching_type_class(element_class, schema):
    """Get the correct type class for an element class in a given schema."""
    # Check schema-specific mappings first
    if schema == "IFC2X3" and element_class in IFC2X3_TYPE_MAP:
        return IFC2X3_TYPE_MAP[element_class]
    elif schema in ("IFC4", "IFC4X3") and element_class in IFC4_TYPE_MAP:
        return IFC4_TYPE_MAP[element_class]

    # Fall back to universal mapping
    return ELEMENT_TYPE_MAP.get(element_class)
```

## Example 10: Unit-Safe Coordinate Extraction

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.unit
import ifcopenshell.util.placement
import numpy as np

def get_element_location_meters(model, element):
    """Get element location in meters, regardless of file units."""
    if element.ObjectPlacement is None:
        return None

    # Get transformation matrix
    matrix = ifcopenshell.util.placement.get_local_placement(
        element.ObjectPlacement)

    # Extract translation (position)
    x, y, z = matrix[:3, 3]

    # Apply unit conversion to get meters
    unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
    x_meters = x * unit_scale
    y_meters = y * unit_scale
    z_meters = z * unit_scale

    return (x_meters, y_meters, z_meters)


# Usage
model = ifcopenshell.open("model.ifc")
for wall in model.by_type("IfcWall"):
    location = get_element_location_meters(model, wall)
    if location:
        print(f"Wall '{wall.Name}': ({location[0]:.3f}, {location[1]:.3f}, {location[2]:.3f}) m")
```

## Example 11: Error-Resilient Bulk Property Extraction

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

def extract_all_wall_properties(model):
    """Extract properties from all walls with error handling."""
    walls = model.by_type("IfcWall")
    results = []
    errors = []

    for wall in walls:
        try:
            psets = ifcopenshell.util.element.get_psets(wall)
            container = ifcopenshell.util.element.get_container(wall)
            wall_type = ifcopenshell.util.element.get_type(wall)

            results.append({
                "guid": wall.GlobalId,
                "name": wall.Name,
                "type_name": wall_type.Name if wall_type else None,
                "storey": container.Name if container else None,
                "is_external": psets.get("Pset_WallCommon", {}).get("IsExternal"),
                "fire_rating": psets.get("Pset_WallCommon", {}).get("FireRating"),
                "all_psets": list(psets.keys())
            })
        except Exception as e:
            errors.append({
                "id": wall.id(),
                "guid": wall.GlobalId,
                "error": str(e)
            })

    print(f"Extracted: {len(results)}, Errors: {len(errors)}")
    return results, errors
```

## Example 12: Transaction-Safe Modifications

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

def modify_with_undo(model, wall, new_name):
    """Modify an entity within a transaction for undo support."""
    # Start transaction
    model.begin_transaction()

    try:
        wall.Name = new_name
        # Perform other modifications...

        # Commit
        model.end_transaction()
        print(f"Changed wall name to '{new_name}'")
        return True

    except Exception as e:
        # Discard on error — NEVER leave transaction open
        model.discard_transaction()
        print(f"ERROR: Modification failed, transaction discarded: {e}")
        return False


# Undo if needed
# model.undo()  # Reverts the last committed transaction
```
