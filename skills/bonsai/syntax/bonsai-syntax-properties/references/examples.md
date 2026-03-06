# Property Set Management — Code Examples

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | IFC4
> All examples verified against IfcOpenShell source code.

---

## Example 1: Complete Property Workflow — Wall with Standard Psets and Qtos

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

# --- Setup ---
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Demo Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# --- Create wall ---
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall-001")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[wall], relating_structure=storey)

# --- Add standard property set ---
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")

ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "Reference": "W-001",
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI90",
    "ThermalTransmittance": 0.24,
    "AcousticRating": "45dB",
    "Combustible": False,
})

# --- Add standard quantity set ---
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=wall, name="Qto_WallBaseQuantities")

ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 5.0,          # IfcQuantityLength (keyword: "length")
    "Height": 3.0,          # IfcQuantityLength (keyword: "height")
    "Width": 0.2,           # IfcQuantityLength (keyword: "width")
    "GrossFootprintArea": 1.0,  # IfcQuantityArea (keyword: "area")
    "NetSideArea": 15.0,    # IfcQuantityArea (keyword: "area")
    "NetVolume": 3.0,       # IfcQuantityVolume (keyword: "volume")
    "GrossSideArea": 15.0,  # IfcQuantityArea (keyword: "area")
    "GrossVolume": 3.0,     # IfcQuantityVolume (keyword: "volume")
})

# --- Read properties back ---
all_psets = ifcopenshell.util.element.get_psets(wall)
print(all_psets)
# {"Pset_WallCommon": {"id": 42, "Reference": "W-001", "IsExternal": True, ...},
#  "Qto_WallBaseQuantities": {"id": 43, "Length": 5.0, ...}}

fire_rating = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon", "FireRating")
print(fire_rating)  # "REI90"

model.write("wall_with_properties.ifc")
```

---

## Example 2: Custom Property Set with Company Prefix

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")
wall = model.by_type("IfcWall")[0]

# Custom pset — use company prefix, NEVER "Pset_" prefix
custom_pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Acme_StructuralData")

ifcopenshell.api.run("pset.edit_pset", model, pset=custom_pset, properties={
    "LoadCapacity": 500.0,          # float -> IfcReal
    "SustainableSource": True,      # bool  -> IfcBoolean
    "CertificationCode": "EN-1234", # str   -> IfcLabel
    "InspectionYear": 2024,         # int   -> IfcInteger
})
```

---

## Example 3: Property Set on Type vs Occurrence

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.open("project.ifc")

# --- Pset on type (shared by all instances) ---
wall_type = model.by_type("IfcWallType")[0]
type_pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall_type, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=type_pset, properties={
    "IsExternal": True,
    "LoadBearing": True,
    "FireRating": "REI60",
})

# --- Pset on occurrence (overrides type for this specific wall) ---
wall = model.by_type("IfcWall")[0]
occ_pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=occ_pset, properties={
    "FireRating": "REI90",  # Override: this wall has higher fire rating
})

# --- Reading with inheritance ---
# Includes type-inherited properties (default)
all_props = ifcopenshell.util.element.get_psets(wall)
print(all_props["Pset_WallCommon"]["FireRating"])  # "REI90" (occurrence overrides)
print(all_props["Pset_WallCommon"]["LoadBearing"])  # True (inherited from type)

# Own properties only (skip type-inherited)
own_props = ifcopenshell.util.element.get_psets(wall, should_inherit=False)
print(own_props["Pset_WallCommon"]["FireRating"])   # "REI90"
print("LoadBearing" in own_props.get("Pset_WallCommon", {}))  # True only if also on occurrence
```

---

## Example 4: Delete Properties and Psets

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")
wall = model.by_type("IfcWall")[0]

# --- Get existing pset ---
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Pset_WallCommon")  # Idempotent: returns existing

# --- Delete individual properties ---
# With should_purge=True (default): property entity removed entirely
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "OldProperty": None,
    "DeprecatedField": None,
})

# With should_purge=False: property entity kept but value cleared
ifcopenshell.api.run("pset.edit_pset", model, pset=pset,
    properties={"OptionalProp": None}, should_purge=False)

# --- Remove entire pset (with all its properties) ---
ifcopenshell.api.run("pset.remove_pset", model, product=wall, pset=pset)
```

---

## Example 5: Quantity Set with Calculated Values

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")
slab = model.by_type("IfcSlab")[0]

# --- Add quantity set ---
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=slab, name="Qto_SlabBaseQuantities")

# --- Calculated quantities ---
length = 10.0
width = 8.0
thickness = 0.25
area = length * width
volume = area * thickness
perimeter = 2 * (length + width)

ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Perimeter": perimeter,       # float + no keyword -> IfcQuantityLength (default)
    "GrossArea": area,            # float + "Area"     -> IfcQuantityArea
    "NetArea": area * 0.95,       # float + "Area"     -> IfcQuantityArea
    "GrossVolume": volume,        # float + "Volume"   -> IfcQuantityVolume
    "NetVolume": volume * 0.95,   # float + "Volume"   -> IfcQuantityVolume
    "GrossWeight": volume * 2400, # float + "Weight"   -> IfcQuantityWeight
    "Width": thickness,           # float + "Width"    -> IfcQuantityLength
})

# --- Remove a quantity (always purged, no should_purge parameter) ---
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Perimeter": None,  # Removed from model
})
```

---

## Example 6: Property Set Templates

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")

# --- Create custom pset template ---
template = ifcopenshell.api.run("pset_template.add_pset_template", model,
    name="Acme_RiskAssessment",
    template_type="PSET_TYPEDRIVENOVERRIDE",
    applicable_entity="IfcWall,IfcWallType,IfcSlab,IfcSlabType")

# --- Add single-value property templates ---
ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=template,
    name="HighVoltageRisk",
    description="Presence of high voltage hazard",
    template_type="P_SINGLEVALUE",
    primary_measure_type="IfcBoolean")

ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=template,
    name="MaxOccupancy",
    description="Maximum allowed occupancy count",
    template_type="P_SINGLEVALUE",
    primary_measure_type="IfcInteger")

# --- Add enumerated-value property template ---
enum_prop = ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=template,
    name="RiskLevel",
    description="Risk assessment category",
    template_type="P_ENUMERATEDVALUE",
    primary_measure_type="IfcLabel")

# Set enumeration values via edit_prop_template
ifcopenshell.api.run("pset_template.edit_prop_template", model,
    prop_template=enum_prop,
    attributes={"Enumerators": ["LOW", "MEDIUM", "HIGH", "CRITICAL"]})

# --- Use template when editing pset ---
wall = model.by_type("IfcWall")[0]
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=wall, name="Acme_RiskAssessment")

# Pass template for correct type resolution
ifcopenshell.api.run("pset.edit_pset", model, pset=pset,
    pset_template=template,
    properties={
        "HighVoltageRisk": True,
        "MaxOccupancy": 50,
        "RiskLevel": ["HIGH"],  # Enum values as list
    })
```

---

## Example 7: Quantity Set Template

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")

# --- Create QTO template ---
qto_template = ifcopenshell.api.run("pset_template.add_pset_template", model,
    name="Acme_ConcreteQuantities",
    template_type="QTO_TYPEDRIVENOVERRIDE",
    applicable_entity="IfcWall,IfcSlab,IfcColumn")

# Template type defaults auto-detect for QTO templates
ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=qto_template,
    name="ExposedSurfaceArea",
    description="Total exposed surface area",
    template_type="Q_AREA")

ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=qto_template,
    name="ConcreteVolume",
    description="Volume of concrete used",
    template_type="Q_VOLUME")

ifcopenshell.api.run("pset_template.add_prop_template", model,
    pset_template=qto_template,
    name="RebarCount",
    description="Number of rebar elements",
    template_type="Q_COUNT")
```

---

## Example 8: Sharing Psets Between Elements

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.element

model = ifcopenshell.open("project.ifc")
walls = model.by_type("IfcWall")[:4]

# --- Create pset on first wall ---
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=walls[0], name="Acme_FireProtection")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "FireResistanceClass": "F90",
    "CoatingThickness": 0.015,
})

# --- Share pset with additional walls ---
ifcopenshell.api.run("pset.assign_pset", model,
    products=[walls[1], walls[2], walls[3]], pset=pset)

# All 4 walls now share the same pset entity
elements = ifcopenshell.util.element.get_elements_by_pset(pset)
assert len(elements) == 4

# --- Unshare for specific walls (creates independent copies) ---
copies = ifcopenshell.api.run("pset.unshare_pset", model,
    products=[walls[2], walls[3]], pset=pset)

# walls[0] and walls[1] still share original pset
# walls[2] and walls[3] each have independent copies
assert len(copies) == 2
```

---

## Example 9: Pset on Material

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("project.ifc")

# --- Add material and its properties ---
concrete = ifcopenshell.api.run("material.add_material", model,
    name="Concrete C30/37", category="concrete")

# add_pset works on IfcMaterialDefinition
mat_pset = ifcopenshell.api.run("pset.add_pset", model,
    product=concrete, name="Pset_MaterialConcrete")

ifcopenshell.api.run("pset.edit_pset", model, pset=mat_pset, properties={
    "CompressiveStrength": 30.0,
    "MaxAggregateSize": 0.032,
    "AdmixturesDescription": "Plasticizer Type A",
})
```

---

## Example 10: Bonsai Operator Workflow (Inside Blender)

```python
# Bonsai v0.8.x — Run inside Blender with Bonsai active
import bpy
from bonsai.bim.ifc import IfcStore
import ifcopenshell.api

# --- Guard: ensure IFC project is loaded ---
model = IfcStore.get_file()
if model is None:
    raise RuntimeError("No IFC project loaded. Open an IFC file first.")

# --- Get active object's IFC entity ---
obj = bpy.context.active_object
element = IfcStore.get_element(obj.BIMObjectProperties.ifc_definition_id)
if element is None:
    raise RuntimeError("Active object is not linked to an IFC entity.")

# --- Add and edit property set via API ---
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=element, name="Acme_QualityControl")

ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "InspectionDate": "2024-06-15",
    "InspectorName": "J. Smith",
    "PassedQC": True,
})

# --- Or use Bonsai operators (handles undo, UI refresh) ---
# Add pset via operator
bpy.ops.bim.add_pset(obj=obj.name, obj_type="Object")

# Copy property to all selected objects
bpy.ops.bim.copy_property_to_selection(name="PassedQC")

# Calculate quantities from mesh geometry
bpy.ops.bim.calculate_face_areas()
bpy.ops.bim.calculate_object_volumes()

# Perform full quantity take-off
bpy.ops.bim.perform_quantity_take_off()
```

---

## Example 11: Batch Property Assignment to Multiple Elements

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

# --- Assign FireRating to all external walls ---
for wall in model.by_type("IfcWall"):
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=wall, name="Pset_WallCommon")

    # Read existing to check IsExternal
    existing = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon")
    if existing and existing.get("IsExternal") is True:
        ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
            "FireRating": "REI120",
        })

model.write("building_updated.ifc")
```

---

## Example 12: Reading Verbose Property Data

```python
# Bonsai v0.8.x / IfcOpenShell v0.8+
import ifcopenshell.util.element

wall = model.by_type("IfcWall")[0]

# --- Verbose mode: includes IFC class info per property ---
verbose_data = ifcopenshell.util.element.get_psets(wall, verbose=True)
# Returns:
# {
#   "Pset_WallCommon": {
#     "id": 42,
#     "FireRating": {"value": "REI90", "class": "IfcPropertySingleValue",
#                    "type": "IfcLabel"},
#     ...
#   }
# }

# --- Check if element has a specific quantity ---
has_length = ifcopenshell.util.element.has_property(wall, "Length")
# Note: has_property only checks quantity sets (qtos_only=True)
```

---

## Sources

- IfcOpenShell API source: `src/ifcopenshell-python/ifcopenshell/api/pset/`
- IfcOpenShell util source: `src/ifcopenshell-python/ifcopenshell/util/element.py`
- Bonsai pset module: `src/bonsai/bonsai/bim/module/pset/`
- Bonsai qto module: `src/bonsai/bonsai/bim/module/qto/`
