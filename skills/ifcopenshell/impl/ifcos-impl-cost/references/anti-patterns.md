# ifcos-impl-cost: Anti-Patterns

Common mistakes when implementing IFC cost management with IfcOpenShell, and how to avoid them.

---

## AP-01: Creating IfcCostItem Directly with model.create_entity()

### Wrong

```python
# WRONG: Bypasses required relationship setup
cost_item = model.create_entity("IfcCostItem",
    GlobalId=ifcopenshell.guid.new(),
    Name="Foundation Works")
```

### Why It Fails

`IfcCostItem` requires either an `IfcRelAssignsToControl` relationship (linking it to an `IfcCostSchedule`) or an `IfcRelNests` relationship (linking it to a parent `IfcCostItem`). Creating the entity directly skips these relationships, producing an orphaned cost item that no IFC viewer or cost tool will recognize.

### Correct

```python
# CORRECT: API creates the required relationships automatically
cost_item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=cost_item,
    attributes={"Name": "Foundation Works"})
```

---

## AP-02: Passing Both cost_schedule AND cost_item to add_cost_item

### Wrong

```python
# WRONG: Parameters are mutually exclusive
item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule,
    cost_item=parent_item)  # Cannot specify both
```

### Why It Fails

`add_cost_item` uses `cost_schedule` to create a top-level item or `cost_item` to create a nested sub-item. Passing both causes undefined behavior. The function determines the parent context from whichever parameter is provided.

### Correct

```python
# CORRECT: Top-level item in a schedule
top_item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)

# CORRECT: Nested item under a parent
sub_item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=top_item)
```

---

## AP-03: Calling control.assign_control After assign_cost_item_quantity

### Wrong

```python
# WRONG: Duplicate control relationship
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item, products=[slab], prop_name="NetVolume")

# This creates a SECOND IfcRelAssignsToControl -- causes data corruption
ifcopenshell.api.run("control.assign_control", model,
    relating_control=item, related_object=slab)
```

### Why It Fails

`assign_cost_item_quantity` already creates the `IfcRelAssignsToControl` relationship between the cost item and the products. Calling `control.assign_control` separately creates a duplicate relationship, leading to double-counting in cost reports and potential validation errors.

### Correct

```python
# CORRECT: assign_cost_item_quantity handles the control relationship
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item, products=[slab], prop_name="NetVolume")
# No additional control.assign_control call needed.
```

---

## AP-04: Expecting Sub-Values to Sum Without Category="*"

### Wrong

```python
# WRONG: Parent value missing Category="*"
parent_val = ifcopenshell.api.run("cost.add_cost_value", model, parent=item)
# No Category set on parent_val

labor = ifcopenshell.api.run("cost.add_cost_value", model, parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=labor, attributes={"AppliedValue": 2000.0})

material = ifcopenshell.api.run("cost.add_cost_value", model, parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=material, attributes={"AppliedValue": 3000.0})
# Total will NOT be 5000.0 -- sub-values are not aggregated
```

### Why It Fails

The `IfcCostValue.Category` attribute controls how sub-values (components) are combined. The special value `"*"` activates summation. Without it, the sub-values exist as independent components with no defined aggregation behavior.

### Correct

```python
# CORRECT: Set Category="*" on the parent value
parent_val = ifcopenshell.api.run("cost.add_cost_value", model, parent=item)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=parent_val,
    attributes={"Category": "*"})  # Enables summation

labor = ifcopenshell.api.run("cost.add_cost_value", model, parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=labor, attributes={"AppliedValue": 2000.0, "Category": "Labor"})

material = ifcopenshell.api.run("cost.add_cost_value", model, parent=parent_val)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=material, attributes={"AppliedValue": 3000.0, "Category": "Material"})
# Total: 2000 + 3000 = 5000.0
```

---

## AP-05: Linking Quantities Before Products Have Them

### Wrong

```python
# WRONG: Slab has no quantity properties yet
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Slab")

# Linking to a non-existent quantity
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item, products=[slab], prop_name="NetVolume")
# The quantity link is created but NetVolume does not exist on the slab.
# Cost calculations will return 0 or None.
```

### Why It Fails

`assign_cost_item_quantity` creates a parametric link to a named quantity property on the product. If the product does not have a quantity set (`IfcElementQuantity`) containing the named property, the link resolves to nothing.

### Correct

```python
# CORRECT: Create quantity on product FIRST
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Slab")

# Add quantity set with the required property
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=slab, name="Qto_SlabBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model,
    qto=qto, properties={"NetVolume": 45.0})

# THEN link cost item
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item, products=[slab], prop_name="NetVolume")
```

---

## AP-06: Setting Name in add_cost_item Call

### Wrong

```python
# WRONG: add_cost_item does NOT accept name parameter
item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule,
    name="Foundation Works")  # Parameter ignored or raises error
```

### Why It Fails

Unlike some other `add_*` functions, `cost.add_cost_item` only accepts `cost_schedule` or `cost_item` as parameters. The `Name` and `Identification` attributes must be set in a separate `edit_cost_item` call.

### Correct

```python
# CORRECT: Create first, then edit attributes
item = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=item,
    attributes={"Name": "Foundation Works", "Identification": "01.01"})
```

---

## AP-07: Using ifcopenshell.api.run() for Utility/Query Functions

### Wrong

```python
# WRONG: util.cost functions are NOT called via api.run()
items = ifcopenshell.api.run("cost.get_root_cost_items", model,
    cost_schedule=schedule)
```

### Why It Fails

Functions in `ifcopenshell.util.cost` are direct Python function calls, not API mutations. They do not follow the `ifcopenshell.api.run()` convention. Only functions in `ifcopenshell.api.cost` use the `api.run()` pattern.

### Correct

```python
# CORRECT: Call util functions directly
import ifcopenshell.util.cost

items = ifcopenshell.util.cost.get_root_cost_items(schedule)
total = ifcopenshell.util.cost.get_total_quantity(item)
values = ifcopenshell.util.cost.get_cost_values(item)
```

---

## AP-08: Assuming AppliedValue Is Always a Float

### Wrong

```python
# WRONG: Direct float comparison without type checking
value = cost_item.CostValues[0]
total = value.AppliedValue * quantity  # May fail if AppliedValue is not a float
```

### Why It Fails

`IfcCostValue.AppliedValue` is typed as `IfcAppliedValueSelect`, which can be a monetary measure, a ratio, or other value types. Accessing it directly as a float may fail or return unexpected results.

### Correct

```python
# CORRECT: Use utility function for safe extraction
import ifcopenshell.util.cost

value = cost_item.CostValues[0]
applied = ifcopenshell.util.cost.get_primitive_applied_value(value.AppliedValue)
# applied is now a float
```

---

## AP-09: Removing Cost Items Before Their Children

### Wrong

```python
# WRONG: Removing parent while children still reference it
parent = root_items[0]
ifcopenshell.api.run("cost.remove_cost_item", model, cost_item=parent)
# Children may become orphaned with dangling references
```

### Why It Fails

While `remove_cost_item` does handle cascading removal in most cases, it is safer and more predictable to remove children first, then parents, especially in deeply nested structures.

### Correct

```python
# CORRECT: Remove from leaf to root
import ifcopenshell.util.cost

parent = root_items[0]
children = ifcopenshell.util.cost.get_nested_cost_items(parent, is_deep=True)
# Remove children first (deepest first)
for child in reversed(children):
    ifcopenshell.api.run("cost.remove_cost_item", model, cost_item=child)
# Then remove parent
ifcopenshell.api.run("cost.remove_cost_item", model, cost_item=parent)
```

---

## AP-10: Mixing Manual and Parametric Quantities

### Wrong

```python
# WRONG: Adding manual quantity AND parametric link to same cost item
quantity = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=quantity, attributes={"VolumeValue": 100.0})

# Then also linking to products
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item, products=[slab], prop_name="NetVolume")
# The manual 100.0 and parametric value will conflict
```

### Why It Fails

A cost item should use EITHER manual quantities (via `add_cost_item_quantity`) OR parametric quantities (via `assign_cost_item_quantity`), not both. Mixing them creates ambiguity about which quantity value should be used for cost calculations.

### Correct

```python
# CORRECT: Choose ONE approach

# Option A: Manual quantity (when product quantities are not available)
quantity = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=quantity, attributes={"VolumeValue": 100.0})

# Option B: Parametric quantity (5D BIM -- preferred when products exist)
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item, products=[slab], prop_name="NetVolume")
```
