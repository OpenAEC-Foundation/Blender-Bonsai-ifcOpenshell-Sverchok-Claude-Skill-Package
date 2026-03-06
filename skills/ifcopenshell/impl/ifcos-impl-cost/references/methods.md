# ifcos-impl-cost: API Method Signatures

Complete method signatures for `ifcopenshell.api.cost` and `ifcopenshell.util.cost`.

---

## ifcopenshell.api.cost -- Mutation Functions

All functions are called via `ifcopenshell.api.run("cost.<name>", file, **kwargs)`.

### Schedule Management

```python
# Create a cost schedule
ifcopenshell.api.run("cost.add_cost_schedule", file,
    name: str | None = None,
    predefined_type: str = "NOTDEFINED"
) -> IfcCostSchedule

# Edit cost schedule attributes
ifcopenshell.api.run("cost.edit_cost_schedule", file,
    cost_schedule: IfcCostSchedule,
    attributes: dict[str, Any]
) -> None

# Copy an entire cost schedule (deep copy including all items)
ifcopenshell.api.run("cost.copy_cost_schedule", file,
    cost_schedule: IfcCostSchedule
) -> IfcCostSchedule

# Remove a cost schedule and all nested items
ifcopenshell.api.run("cost.remove_cost_schedule", file,
    cost_schedule: IfcCostSchedule
) -> None
```

**`predefined_type` values** (from `IfcCostScheduleTypeEnum`):
- `"BUDGET"` -- Budget allocation
- `"COSTPLAN"` -- Cost plan / cost estimate
- `"ESTIMATE"` -- Preliminary cost estimate
- `"TENDER"` -- Bid/tender submission
- `"PRICEDBILLOFQUANTITIES"` -- Priced bill of quantities
- `"UNPRICEDBILLOFQUANTITIES"` -- Unpriced bill of quantities
- `"SCHEDULEOFRATES"` -- Schedule of unit rates (template for reuse)
- `"USERDEFINED"` -- Custom type (set `ObjectType` attribute via edit)
- `"NOTDEFINED"` -- Default / unspecified

**Editable `IfcCostSchedule` attributes:**
- `Name` (str) -- Schedule name
- `Description` (str) -- Schedule description
- `Status` (str) -- e.g., `"DRAFT"`, `"APPROVED"`, `"SUBMITTED"`
- `SubmittedOn` (str) -- Submission date
- `UpdateDate` (str) -- Last update date

### Cost Item Management

```python
# Create a cost item (top-level or nested)
# IMPORTANT: Exactly ONE of cost_schedule or cost_item must be provided
ifcopenshell.api.run("cost.add_cost_item", file,
    cost_schedule: IfcCostSchedule | None = None,
    cost_item: IfcCostItem | None = None
) -> IfcCostItem

# Edit cost item attributes
ifcopenshell.api.run("cost.edit_cost_item", file,
    cost_item: IfcCostItem,
    attributes: dict[str, Any]
) -> None

# Copy a cost item (with nested items and relationships)
ifcopenshell.api.run("cost.copy_cost_item", file,
    cost_item: IfcCostItem
) -> IfcCostItem | list[IfcCostItem]

# Copy cost values from one item to another
ifcopenshell.api.run("cost.copy_cost_item_values", file,
    source: IfcCostItem,
    destination: IfcCostItem
) -> None

# Remove a cost item and its associations
ifcopenshell.api.run("cost.remove_cost_item", file,
    cost_item: IfcCostItem
) -> None
```

**Editable `IfcCostItem` attributes:**
- `Name` (str) -- Item name (e.g., "Structural Works")
- `Identification` (str) -- Item code (e.g., "01.01")
- `Description` (str) -- Detailed description

### Cost Value Management

```python
# Add a cost value (or sub-value when parent is another IfcCostValue)
ifcopenshell.api.run("cost.add_cost_value", file,
    parent: IfcCostItem | IfcCostValue
) -> IfcCostValue

# Edit cost value attributes
ifcopenshell.api.run("cost.edit_cost_value", file,
    cost_value: IfcCostValue,
    attributes: dict[str, Any]
) -> None

# Set a formula on a cost value
ifcopenshell.api.run("cost.edit_cost_value_formula", file,
    cost_value: IfcCostValue,
    formula: str
) -> None

# Assign cost values from a schedule of rates
ifcopenshell.api.run("cost.assign_cost_value", file,
    cost_item: IfcCostItem,
    cost_rate: IfcCostItem  # from a SCHEDULEOFRATES schedule
) -> None

# Remove a cost value
ifcopenshell.api.run("cost.remove_cost_value", file,
    parent: IfcCostItem | IfcCostValue,
    cost_value: IfcCostValue
) -> None
```

**Editable `IfcCostValue` attributes:**
- `AppliedValue` (float) -- Monetary value or unit rate
- `Category` (str) -- `"*"` for sum operator, or label like `"Labor"`, `"Material"`
- `UnitBasis` -- Unit of measurement basis
- `ArithmeticOperator` -- Operator for sub-value computation

### Quantity Management

```python
# Add a quantity to a cost item (manual quantity)
ifcopenshell.api.run("cost.add_cost_item_quantity", file,
    cost_item: IfcCostItem,
    ifc_class: str = "IfcQuantityCount"
) -> IfcPhysicalQuantity

# Edit quantity attributes
ifcopenshell.api.run("cost.edit_cost_item_quantity", file,
    physical_quantity: IfcPhysicalQuantity,
    attributes: dict[str, Any]
) -> None

# Remove a quantity from a cost item
ifcopenshell.api.run("cost.remove_cost_item_quantity", file,
    cost_item: IfcCostItem,
    physical_quantity: IfcPhysicalQuantity
) -> None
```

**`ifc_class` values for quantities:**
- `"IfcQuantityCount"` -- Count (attribute: `CountValue`)
- `"IfcQuantityArea"` -- Area in m2 (attribute: `AreaValue`)
- `"IfcQuantityVolume"` -- Volume in m3 (attribute: `VolumeValue`)
- `"IfcQuantityWeight"` -- Weight in kg (attribute: `WeightValue`)
- `"IfcQuantityLength"` -- Length in m (attribute: `LengthValue`)

### Parametric Quantity Assignment

```python
# Link cost item quantities to product quantities (5D BIM)
# Creates IfcRelAssignsToControl automatically
ifcopenshell.api.run("cost.assign_cost_item_quantity", file,
    cost_item: IfcCostItem,
    products: list[IfcProduct],
    prop_name: str = ""  # empty = count products; named = read quantity
) -> None

# Remove parametric link between cost item and products
ifcopenshell.api.run("cost.unassign_cost_item_quantity", file,
    cost_item: IfcCostItem,
    products: list[IfcProduct]
) -> None
```

### Resource-Based Calculation

```python
# Calculate cost from assigned construction resources
ifcopenshell.api.run("cost.calculate_cost_item_resource_value", file,
    cost_item: IfcCostItem
) -> None
```

---

## ifcopenshell.util.cost -- Read-Only Query Functions

These are direct function calls, NOT via `ifcopenshell.api.run()`.

### Cost Item Traversal

```python
import ifcopenshell.util.cost

# Get top-level items in a schedule
get_root_cost_items(cost_schedule: IfcCostSchedule) -> list[IfcCostItem]

# Get child items of a cost item
get_nested_cost_items(cost_item: IfcCostItem, is_deep: bool = False) -> list[IfcCostItem]

# Generator: all nested items recursively
get_all_nested_cost_items(cost_item: IfcCostItem) -> Generator[IfcCostItem]

# Generator: all items in a schedule (including nested)
get_schedule_cost_items(cost_schedule: IfcCostSchedule) -> Generator[IfcCostItem]
```

### Cost Item Lookups

```python
# Find parent schedule of a cost item
get_cost_schedule(cost_item: IfcCostItem) -> IfcCostSchedule

# Find cost items linked to a product
get_cost_items_for_product(product: IfcProduct) -> list[IfcCostItem]

# Get assigned elements for a cost item
get_cost_item_assignments(
    cost_item: IfcCostItem,
    filter_by_type: str | None = None,
    is_deep: bool = False
) -> list

# Filter assignments by type
get_cost_assignments_by_type(
    cost_item: IfcCostItem,
    filter_by_type: str | None = None
) -> list

# Get associated rate item
get_assigned_rate_cost_item(cost_item: IfcCostItem) -> IfcCostItem | None
```

### Value and Quantity Functions

```python
# Get cost values as list of dicts
get_cost_values(cost_item: IfcCostItem) -> list[dict]

# Get cost rate information
get_cost_rate(file: ifcopenshell.file, cost_item: IfcCostItem) -> Any

# Calculate total quantity for a cost item
get_total_quantity(root_element: IfcCostItem) -> float | None

# Calculate applied value with optional category filter
calculate_applied_value(
    root_element: IfcCostItem,
    cost_value: IfcCostValue,
    category_filter: str | None = None
) -> float

# Extract primitive numeric value from AppliedValue
get_primitive_applied_value(applied_value: Any) -> float

# Sum costs from child elements
sum_child_root_elements(
    root_element: IfcCostItem,
    category_filter: str | None = None
) -> float

# Get available quantity property names from products
get_product_quantity_names(elements: list) -> list[str]

# List cost schedule types in a file
get_cost_schedule_types(file: ifcopenshell.file) -> list[dict]
```

### Serialization

```python
# Convert cost value to string representation
serialise_cost_value(cost_value: IfcCostValue) -> str

# Convert applied value to string
serialise_applied_value(applied_value: Any) -> str

# Parse formula string into structured data
unserialise_cost_value(formula: str, cost_value: IfcCostValue) -> dict
```

### CostValueUnserialiser Class

```python
class CostValueUnserialiser:
    def parse(formula: str) -> dict
    def get_category(category: str) -> str
    def get_formula(formula: str) -> str
    def get_operand(operand: str) -> Any
    def get_operator(operator: str) -> str
    def get_value(value: str) -> float
```
