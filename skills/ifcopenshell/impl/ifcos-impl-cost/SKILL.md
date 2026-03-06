---
name: ifcos-impl-cost
description: 'Guides IFC cost management using ifcopenshell.api.cost including cost schedules, cost items, cost values, cost quantities, and 5D BIM workflows. Activates when implementing cost estimation in IFC models, creating cost schedules, or linking quantities to cost items.'
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Cost Management Implementation Guide

## Quick Reference

### Decision Tree: Choosing a Cost Structure

```
Need to manage costs in IFC?
├── Creating a cost estimate or budget?
│   └── Create IfcCostSchedule (predefined_type determines purpose)
│       ├── Budget allocation → predefined_type="BUDGET"
│       ├── Cost plan / estimate → predefined_type="COSTPLAN"
│       ├── Preliminary estimate → predefined_type="ESTIMATE"
│       ├── Bid/tender → predefined_type="TENDER"
│       ├── Priced BoQ → predefined_type="PRICEDBILLOFQUANTITIES"
│       ├── Unpriced BoQ → predefined_type="UNPRICEDBILLOFQUANTITIES"
│       ├── Unit rate schedule → predefined_type="SCHEDULEOFRATES"
│       └── Unknown/other → predefined_type="NOTDEFINED"
│
├── Adding line items to a schedule?
│   └── cost.add_cost_item
│       ├── Top-level item → pass cost_schedule=schedule
│       └── Nested sub-item → pass cost_item=parent_item
│
├── Setting monetary values?
│   └── cost.add_cost_value → then cost.edit_cost_value
│       ├── Fixed lump sum → attributes={"AppliedValue": 42000.0}
│       ├── Unit rate (cost * qty) → set AppliedValue + add quantity
│       ├── Formula-based → use cost.edit_cost_value_formula
│       └── Composite (labor + material) → add sub-values to parent value
│
├── Linking quantities to products?
│   └── cost.assign_cost_item_quantity
│       ├── Named quantity (volume, area) → prop_name="NetVolume"
│       └── Count products → prop_name="" (empty string)
│
└── Querying existing costs?
    └── Use ifcopenshell.util.cost functions (read-only)
```

### Decision Tree: Cost Value Approaches

```
How should the cost be calculated?
├── Fixed total amount (lump sum)?
│   └── Set AppliedValue directly on a single IfcCostValue
│       AppliedValue=42000.0
│
├── Unit rate multiplied by quantity?
│   └── Set AppliedValue as unit rate + attach quantity
│       AppliedValue=85.0 (per m3) + IfcQuantityVolume=120.0
│       Result: 85.0 * 120.0 = 10200.0
│
├── Formula calculation?
│   └── Use edit_cost_value_formula
│       formula="5000 * 1.19" (base * tax)
│       Supports: +, -, *, /, parentheses
│
├── Composite breakdown (labor + material + equipment)?
│   └── Parent IfcCostValue with Category="*" (sum operator)
│       ├── Sub-value: Category="Labor", AppliedValue=2000.0
│       ├── Sub-value: Category="Material", AppliedValue=3000.0
│       └── Sub-value: Category="Equipment", AppliedValue=1500.0
│       Result: 2000 + 3000 + 1500 = 6500.0
│
└── From a schedule of rates (template)?
    └── cost.assign_cost_value(cost_item=item, cost_rate=rate_item)
```

### Critical Warnings

- **ALWAYS** use `ifcopenshell.api.run("cost.add_cost_item", ...)` to create cost items. NEVER use `model.create_entity("IfcCostItem")` directly -- the API sets up required `IfcRelNests` or `IfcRelAssignsToControl` relationships automatically.
- **NEVER** pass both `cost_schedule=` and `cost_item=` to `add_cost_item`. These parameters are mutually exclusive. Exactly one MUST be provided.
- **NEVER** call `control.assign_control` after `assign_cost_item_quantity`. The quantity assignment function creates the `IfcRelAssignsToControl` relationship automatically.
- **NEVER** expect sub-values to sum without setting the parent `IfcCostValue.Category` to `"*"`. Without this, sub-values are NOT aggregated.
- **ALWAYS** ensure products have the named quantity property (via `pset.add_qto` + `pset.edit_qto`) BEFORE calling `assign_cost_item_quantity` with a `prop_name`.
- **ALWAYS** use `cost.edit_cost_item` to set `Name` and `Identification` after creating a cost item. `add_cost_item` does not accept name/identification parameters.
- **NEVER** assume `AppliedValue` is always a float. In IFC, `IfcCostValue.AppliedValue` is an `IfcAppliedValueSelect` which can be a monetary measure or a ratio. When reading values, use `ifcopenshell.util.cost.get_primitive_applied_value()` for safe extraction.

---

## Essential Patterns

### Pattern 1: Create a Cost Schedule

```python
# IfcOpenShell -- IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Cost Demo")
ifcopenshell.api.run("unit.assign_unit", model)

schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Construction Estimate",
    predefined_type="COSTPLAN")
```

### Pattern 2: Build a Cost Breakdown Structure

```python
# IfcOpenShell -- IFC4 / IFC4X3
# Create top-level category
structural = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=structural,
    attributes={"Name": "Structural Works", "Identification": "01"})

# Create nested sub-item
foundations = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=structural)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=foundations,
    attributes={"Name": "Foundations", "Identification": "01.01"})
```

### Pattern 3: Set a Fixed (Lump Sum) Cost Value

```python
# IfcOpenShell -- IFC4 / IFC4X3
value = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=foundations)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=value,
    attributes={"AppliedValue": 42000.0})
```

### Pattern 4: Unit Rate with Manual Quantity

```python
# IfcOpenShell -- IFC4 / IFC4X3
item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item,
    attributes={"Name": "Concrete", "Identification": "01.02"})

# Set unit rate
value = ifcopenshell.api.run("cost.add_cost_value", model, parent=item)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=value,
    attributes={"AppliedValue": 85.0})  # EUR per m3

# Attach manual quantity
quantity = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=quantity,
    attributes={"VolumeValue": 120.0})
# Calculated total: 85.0 * 120.0 = 10200.0
```

### Pattern 5: Link Cost Item to Product Quantities (5D BIM)

```python
# IfcOpenShell -- IFC4 / IFC4X3
# Prerequisites: product with quantity set
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Floor Slab")
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=slab, name="Qto_SlabBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model,
    qto=qto, properties={"NetVolume": 45.0})

# Link cost item to product quantity
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item,
    products=[slab],
    prop_name="NetVolume")
# Cost item now tracks slab's NetVolume parametrically.
# IfcRelAssignsToControl is created automatically.

# Count-based linking (no named property)
door_item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
doors = model.by_type("IfcDoor")
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=door_item,
    products=doors,
    prop_name="")  # Empty string = count products
```

### Pattern 6: Composite Cost Values (Labor + Material)

```python
# IfcOpenShell -- IFC4 / IFC4X3
item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item,
    attributes={"Name": "Brickwork", "Identification": "02.01"})

# Parent value with sum operator
parent_value = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=item)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=parent_value,
    attributes={"Category": "*"})  # "*" = sum sub-values

# Sub-values
labor = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=parent_value)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=labor,
    attributes={"AppliedValue": 2000.0, "Category": "Labor"})

material = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=parent_value)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=material,
    attributes={"AppliedValue": 3000.0, "Category": "Material"})
# Result: 2000 + 3000 = 5000.0
```

### Pattern 7: Formula-Based Cost Values

```python
# IfcOpenShell -- IFC4 / IFC4X3
value = ifcopenshell.api.run("cost.add_cost_value", model, parent=item)
ifcopenshell.api.run("cost.edit_cost_value_formula", model,
    cost_value=value,
    formula="5000 * 1.19")  # base * 19% tax
# Formula syntax: arithmetic expressions parsed by ifcopenshell.util.cost
```

### Pattern 8: Schedule of Rates (Template Reuse)

```python
# IfcOpenShell -- IFC4 / IFC4X3
# Step 1: Create a schedule of rates (reusable template)
rates = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Unit Rate Library",
    predefined_type="SCHEDULEOFRATES")

rate_concrete = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=rates)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=rate_concrete,
    attributes={"Name": "C30 Concrete", "Identification": "R-001"})
rate_val = ifcopenshell.api.run("cost.add_cost_value", model,
    parent=rate_concrete)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=rate_val,
    attributes={"AppliedValue": 95.0})  # EUR/m3

# Step 2: Apply rate to a cost item in another schedule
ifcopenshell.api.run("cost.assign_cost_value", model,
    cost_item=item,
    cost_rate=rate_concrete)
```

---

## Common Operations

### Querying Cost Data

```python
# IfcOpenShell -- IFC4 / IFC4X3
import ifcopenshell.util.cost

# Get all cost schedules
schedules = model.by_type("IfcCostSchedule")

# Get top-level items in a schedule
root_items = ifcopenshell.util.cost.get_root_cost_items(schedule)

# Get all items recursively (generator)
for item in ifcopenshell.util.cost.get_schedule_cost_items(schedule):
    print(item.Name, item.Identification)

# Get nested sub-items of a cost item
children = ifcopenshell.util.cost.get_nested_cost_items(
    structural, is_deep=True)

# Find which cost items reference a product
items = ifcopenshell.util.cost.get_cost_items_for_product(slab)

# Find parent schedule of a cost item
parent_schedule = ifcopenshell.util.cost.get_cost_schedule(item)

# Get total quantity for a cost item
total_qty = ifcopenshell.util.cost.get_total_quantity(item)

# Get cost values as dicts
values = ifcopenshell.util.cost.get_cost_values(item)

# Calculate applied value
total = ifcopenshell.util.cost.calculate_applied_value(
    item, value, category_filter=None)
```

### Editing Existing Cost Data

```python
# IfcOpenShell -- IFC4 / IFC4X3
# Edit cost schedule attributes
ifcopenshell.api.run("cost.edit_cost_schedule", model,
    cost_schedule=schedule,
    attributes={"Name": "Updated Estimate", "Status": "APPROVED"})

# Edit cost item attributes
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item,
    attributes={"Name": "Updated Name", "Description": "Added detail"})

# Edit cost value
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=value,
    attributes={"AppliedValue": 99.50})
```

### Copying and Duplicating

```python
# IfcOpenShell -- IFC4 / IFC4X3
# Copy an entire cost schedule (deep copy)
new_schedule = ifcopenshell.api.run("cost.copy_cost_schedule", model,
    cost_schedule=schedule)

# Copy a single cost item (with nested items)
new_item = ifcopenshell.api.run("cost.copy_cost_item", model,
    cost_item=item)

# Copy values from one item to another
ifcopenshell.api.run("cost.copy_cost_item_values", model,
    source=item, destination=new_item)
```

### Removing Cost Data

```python
# IfcOpenShell -- IFC4 / IFC4X3
# Remove a cost value
ifcopenshell.api.run("cost.remove_cost_value", model,
    parent=item, cost_value=value)

# Remove a quantity from a cost item
ifcopenshell.api.run("cost.remove_cost_item_quantity", model,
    cost_item=item, physical_quantity=quantity)

# Unlink products from cost item
ifcopenshell.api.run("cost.unassign_cost_item_quantity", model,
    cost_item=item, products=[slab])

# Remove a cost item (and its associations)
ifcopenshell.api.run("cost.remove_cost_item", model,
    cost_item=item)

# Remove an entire schedule (and all nested items)
ifcopenshell.api.run("cost.remove_cost_schedule", model,
    cost_schedule=schedule)
```

### Resource-Based Cost Calculation

```python
# IfcOpenShell -- IFC4 / IFC4X3
# When cost items have assigned construction resources,
# calculate the total value from resource costs
ifcopenshell.api.run("cost.calculate_cost_item_resource_value", model,
    cost_item=item)
```

---

## IFC Entity Hierarchy

```
IfcCostSchedule (predefined_type: BUDGET|COSTPLAN|ESTIMATE|TENDER|...)
└── IfcCostItem (via IfcRelAssignsToControl)
    ├── Name, Identification, Description
    ├── IfcCostValue (CostValues attribute)
    │   ├── AppliedValue (float or IfcAppliedValueSelect)
    │   ├── Category ("*" for sum, "Labor", "Material", etc.)
    │   ├── ArithmeticOperator (for formula-based values)
    │   └── Components[] (nested IfcCostValue sub-values)
    ├── IfcPhysicalQuantity (CostQuantities attribute)
    │   ├── IfcQuantityCount (CountValue)
    │   ├── IfcQuantityArea (AreaValue)
    │   ├── IfcQuantityVolume (VolumeValue)
    │   ├── IfcQuantityWeight (WeightValue)
    │   └── IfcQuantityLength (LengthValue)
    ├── IfcRelAssignsToControl → products (parametric link)
    └── IfcRelNests → child IfcCostItems (nesting)
```

---

## Version Notes

### Schema Compatibility

| Feature | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| IfcCostSchedule | Yes | Yes | Yes |
| IfcCostItem | Yes | Yes | Yes |
| IfcCostValue | Yes | Yes (enhanced) | Yes (enhanced) |
| CostScheduleTypeEnum | Limited | Full | Full |
| Formula support | No | Yes | Yes |
| Sub-value components | No | Yes | Yes |

- In **IFC2X3**, `IfcCostValue` has a simpler structure without formula support or nested sub-values.
- In **IFC4** and **IFC4X3**, `IfcCostValue` supports `Components`, `ArithmeticOperator`, and formula-based calculation.
- The `ifcopenshell.api.cost` functions work identically across IFC4 and IFC4X3. For IFC2X3, avoid formula and composite value patterns.

### IfcOpenShell API Consistency

All `ifcopenshell.api.cost.*` functions follow the same calling convention:
```python
ifcopenshell.api.run("cost.<function_name>", model, **kwargs)
```

The `file` parameter is ALWAYS the first positional argument (the IFC model). All other parameters are keyword arguments.

---

## 5D BIM Workflow Summary

5D BIM adds cost information as the fifth dimension (3D geometry + time + cost). The standard workflow:

1. **Model** -- Create or open IFC model with spatial structure and products
2. **Quantify** -- Attach quantities to products via `pset.add_qto` + `pset.edit_qto`
3. **Schedule** -- Create `IfcCostSchedule` with appropriate `predefined_type`
4. **Itemize** -- Build cost breakdown via nested `IfcCostItem` hierarchy
5. **Price** -- Add `IfcCostValue` to each cost item (fixed, unit rate, formula, or composite)
6. **Link** -- Connect cost items to product quantities via `assign_cost_item_quantity`
7. **Calculate** -- Use `ifcopenshell.util.cost` to compute totals and generate reports

---

## Reference Links

- [API Method Signatures](references/methods.md) -- Complete signatures for all cost API functions
- [Working Code Examples](references/examples.md) -- End-to-end cost management examples
- [Anti-Patterns](references/anti-patterns.md) -- Common cost implementation mistakes and how to avoid them
