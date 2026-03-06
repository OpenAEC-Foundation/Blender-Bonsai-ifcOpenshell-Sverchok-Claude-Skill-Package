# ifcos-impl-cost: Working Code Examples

End-to-end examples for IFC cost management with IfcOpenShell.

---

## Example 1: Complete Cost Estimate for a Building

Creates a full cost breakdown structure with multiple categories, unit rates, and quantities.

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

# Create IFC model with project structure
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Office Building")
ifcopenshell.api.run("unit.assign_unit", model)

# Create cost schedule
schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Preliminary Estimate Q1 2026",
    predefined_type="ESTIMATE")

# --- Category 1: Structural Works ---
cat_structural = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=cat_structural,
    attributes={"Name": "Structural Works", "Identification": "01"})

# Foundations (lump sum)
item_foundations = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=cat_structural)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_foundations,
    attributes={"Name": "Foundations", "Identification": "01.01"})
val_found = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item_foundations)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_found,
    attributes={"AppliedValue": 125000.0})

# Concrete frame (unit rate)
item_concrete = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=cat_structural)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_concrete,
    attributes={"Name": "Concrete Frame", "Identification": "01.02"})
val_concrete = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item_concrete)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_concrete,
    attributes={"AppliedValue": 95.0})  # EUR/m3
qty_concrete = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item_concrete, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=qty_concrete,
    attributes={"VolumeValue": 340.0})

# Steel reinforcement (unit rate by weight)
item_steel = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=cat_structural)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_steel,
    attributes={"Name": "Reinforcement Steel", "Identification": "01.03"})
val_steel = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item_steel)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_steel,
    attributes={"AppliedValue": 1.20})  # EUR/kg
qty_steel = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item_steel, ifc_class="IfcQuantityWeight")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=qty_steel,
    attributes={"WeightValue": 45000.0})

# --- Category 2: Finishing Works ---
cat_finishing = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=cat_finishing,
    attributes={"Name": "Finishing Works", "Identification": "02"})

# Internal painting (unit rate by area)
item_paint = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=cat_finishing)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_paint,
    attributes={"Name": "Internal Painting", "Identification": "02.01"})
val_paint = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item_paint)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_paint,
    attributes={"AppliedValue": 12.50})  # EUR/m2
qty_paint = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item_paint, ifc_class="IfcQuantityArea")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=qty_paint,
    attributes={"AreaValue": 2800.0})

model.write("building_estimate.ifc")
print("Cost estimate saved.")
```

---

## Example 2: 5D BIM -- Linking Costs to Model Quantities

Demonstrates parametric cost linking where cost items track product quantities automatically.

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="5D BIM Demo")
ifcopenshell.api.run("unit.assign_unit", model)
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW")

# Create building elements
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Main Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Create slabs with quantities
slab_1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Slab Zone A")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[slab_1], relating_structure=storey)
qto_1 = ifcopenshell.api.run("pset.add_qto", model,
    product=slab_1, name="Qto_SlabBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model,
    qto=qto_1, properties={"NetVolume": 32.5, "NetArea": 130.0})

slab_2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Slab Zone B")
ifcopenshell.api.run("spatial.assign_container", model,
    products=[slab_2], relating_structure=storey)
qto_2 = ifcopenshell.api.run("pset.add_qto", model,
    product=slab_2, name="Qto_SlabBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model,
    qto=qto_2, properties={"NetVolume": 28.0, "NetArea": 112.0})

# Create cost schedule and items
schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="5D Cost Estimate",
    predefined_type="COSTPLAN")

# Concrete cost item linked to slab volumes
item_concrete = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_concrete,
    attributes={"Name": "Ground Slab Concrete", "Identification": "C-001"})
val_concrete = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item_concrete)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_concrete,
    attributes={"AppliedValue": 95.0})  # EUR/m3

# Link cost item to BOTH slabs' NetVolume
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item_concrete,
    products=[slab_1, slab_2],
    prop_name="NetVolume")
# Total quantity: 32.5 + 28.0 = 60.5 m3
# Total cost: 60.5 * 95.0 = 5747.5 EUR

model.write("5d_bim_demo.ifc")
print("5D BIM cost model saved.")
```

---

## Example 3: Composite Cost Values (Labor + Material + Equipment)

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Composite Cost Demo")
ifcopenshell.api.run("unit.assign_unit", model)

schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Detailed Estimate",
    predefined_type="PRICEDBILLOFQUANTITIES")

# Create cost item for brickwork
item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item,
    attributes={"Name": "External Brickwork", "Identification": "E-001"})

# Parent value with sum operator
parent_val = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=parent_val,
    attributes={"Category": "*"})  # CRITICAL: "*" enables summation

# Labor sub-value
labor = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=labor,
    attributes={"AppliedValue": 35.0, "Category": "Labor"})

# Material sub-value
material = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=material,
    attributes={"AppliedValue": 48.0, "Category": "Material"})

# Equipment sub-value
equipment = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=equipment,
    attributes={"AppliedValue": 7.50, "Category": "Equipment"})

# Add quantity
qty = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item, ifc_class="IfcQuantityArea")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=qty,
    attributes={"AreaValue": 450.0})
# Unit rate: 35 + 48 + 7.50 = 90.50 EUR/m2
# Total: 90.50 * 450 = 40725.0 EUR

model.write("composite_cost.ifc")
```

---

## Example 4: Schedule of Rates (Reusable Rate Library)

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Rate Library Demo")
ifcopenshell.api.run("unit.assign_unit", model)

# Step 1: Create schedule of rates (template)
rate_schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="2026 Unit Rate Library",
    predefined_type="SCHEDULEOFRATES")

# Define rate items
rate_c30 = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=rate_schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=rate_c30,
    attributes={"Name": "C30 Concrete Supply & Place", "Identification": "R-C30"})
val_c30 = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=rate_c30)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_c30,
    attributes={"AppliedValue": 110.0})  # EUR/m3

rate_rebar = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=rate_schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=rate_rebar,
    attributes={"Name": "B500B Reinforcement", "Identification": "R-RB"})
val_rebar = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=rate_rebar)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_rebar,
    attributes={"AppliedValue": 1.35})  # EUR/kg

# Step 2: Create project estimate that uses the rates
estimate = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Project Estimate",
    predefined_type="COSTPLAN")

item_slab = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=estimate)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_slab,
    attributes={"Name": "Ground Floor Slab", "Identification": "S-001"})

# Apply rate from the library
ifcopenshell.api.run("cost.assign_cost_value", model,
    cost_item=item_slab,
    cost_rate=rate_c30)

# Add quantity
qty_slab = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item_slab, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=qty_slab,
    attributes={"VolumeValue": 55.0})
# Total: 110.0 * 55.0 = 6050.0 EUR

model.write("rate_library.ifc")
```

---

## Example 5: Querying and Reporting on Cost Data

```python
# IfcOpenShell -- IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.util.cost

model = ifcopenshell.open("building_estimate.ifc")

# List all cost schedules
for schedule in model.by_type("IfcCostSchedule"):
    print(f"\nSchedule: {schedule.Name} ({schedule.PredefinedType})")

    # Iterate all items (including nested)
    for item in ifcopenshell.util.cost.get_schedule_cost_items(schedule):
        indent = "  "
        # Determine nesting level by checking parent
        parent_schedule = ifcopenshell.util.cost.get_cost_schedule(item)

        # Get quantity
        total_qty = ifcopenshell.util.cost.get_total_quantity(item)

        # Get cost values
        values = ifcopenshell.util.cost.get_cost_values(item)

        print(f"{indent}{item.Identification}: {item.Name}")
        if total_qty is not None:
            print(f"{indent}  Quantity: {total_qty}")
        for v in values:
            print(f"{indent}  Value: {v}")

    # Find cost items for a specific product
    slabs = model.by_type("IfcSlab")
    for slab in slabs:
        linked_items = ifcopenshell.util.cost.get_cost_items_for_product(slab)
        if linked_items:
            print(f"\n{slab.Name} is linked to:")
            for li in linked_items:
                print(f"  - {li.Identification}: {li.Name}")
```

---

## Example 6: Formula-Based Cost with Tax

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Formula Demo")
ifcopenshell.api.run("unit.assign_unit", model)

schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Formula Estimate", predefined_type="COSTPLAN")

item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item,
    attributes={"Name": "Professional Fees", "Identification": "PF-001"})

# Base cost with formula
value = ifcopenshell.api.run("cost.add_cost_value", model, parent=item)
ifcopenshell.api.run("cost.edit_cost_value_formula", model,
    cost_value=value,
    formula="25000 * 1.21")  # base * 21% VAT
# Result: 30250.0

model.write("formula_cost.ifc")
```

---

## Example 7: Copy and Modify a Cost Schedule

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building_estimate.ifc")
schedules = model.by_type("IfcCostSchedule")
original = schedules[0]

# Deep-copy the entire schedule
revised = ifcopenshell.api.run("cost.copy_cost_schedule", model,
    cost_schedule=original)
ifcopenshell.api.run("cost.edit_cost_schedule", model,
    cost_schedule=revised,
    attributes={"Name": "Revised Estimate v2", "Status": "DRAFT"})

# Modify a specific item in the copy
import ifcopenshell.util.cost
root_items = ifcopenshell.util.cost.get_root_cost_items(revised)
for item in root_items:
    if item.Identification == "01":
        children = ifcopenshell.util.cost.get_nested_cost_items(item)
        for child in children:
            if child.Identification == "01.02":
                # Update concrete unit rate
                values = child.CostValues
                if values:
                    ifcopenshell.api.run("cost.edit_cost_value", model,
                        cost_value=values[0],
                        attributes={"AppliedValue": 105.0})  # price increase
                break

model.write("revised_estimate.ifc")
```

---

## Example 8: Count-Based Costing (Doors, Windows)

```python
# IfcOpenShell -- IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Doors & Windows Estimate",
    predefined_type="ESTIMATE")

# Cost per door (count-based)
item_doors = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item_doors,
    attributes={"Name": "Internal Doors", "Identification": "D-001"})
val_doors = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item_doors)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=val_doors,
    attributes={"AppliedValue": 450.0})  # EUR per door

# Link to all doors (count mode -- prop_name is empty string)
doors = model.by_type("IfcDoor")
if doors:
    ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
        cost_item=item_doors,
        products=doors,
        prop_name="")  # empty = count the products
    # Total: 450.0 * len(doors) EUR

model.write("door_costs.ifc")
```
