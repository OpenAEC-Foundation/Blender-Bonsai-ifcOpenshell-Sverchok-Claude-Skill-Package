# Supplementary IfcOpenShell Research -- Gap Domains

**Date:** 2026-03-05
**Status:** COMPLETE
**Subject:** Supplementary research covering IfcOpenShell API domains NOT covered in vooronderzoek-ifcopenshell.md
**Scope:** Cost management, 4D scheduling, MEP systems, drawing/2D, profiles, validation & georeferencing, Python runtime quirks

---

## Table of Contents

1. [Cost Management (ifcopenshell.api.cost)](#1-cost-management-ifcopenshellapicost)
2. [4D Scheduling (ifcopenshell.api.sequence)](#2-4d-scheduling-ifcopenshellapisequence)
3. [MEP Systems (ifcopenshell.api.system)](#3-mep-systems-ifcopenshellapisystem)
4. [Drawing / 2D (ifcopenshell.api.drawing)](#4-drawing--2d-ifcopenshellapidrawing)
5. [Profiles (ifcopenshell.api.profile)](#5-profiles-ifcopenshellapiprofile)
6. [Validation & Georeferencing](#6-validation--georeferencing)
7. [Python Runtime Quirks](#7-python-runtime-quirks)
8. [Sources](#8-sources)

---

## 1. Cost Management (ifcopenshell.api.cost)

### 1.1 Overview

The `ifcopenshell.api.cost` module provides 20 functions for creating, editing, and managing cost schedules and cost items within IFC models. Cost management in IFC follows a hierarchical structure: a **cost schedule** contains **cost items**, which hold **cost values** and reference **quantities** from products.

**Key IFC Entities:**
- `IfcCostSchedule` -- top-level container for cost data (estimate, bid, tender, actual)
- `IfcCostItem` -- a line item in a cost schedule, supporting nesting for breakdown structures
- `IfcCostValue` -- monetary value attached to a cost item (supports formulas and sub-values)
- `IfcPhysicalQuantity` -- quantity measure linked to a cost item (count, area, volume, weight, length)

### 1.2 Complete API Function Reference

| Function | Parameters | Returns | Purpose |
|----------|-----------|---------|---------|
| `add_cost_schedule` | `file`, `name=None`, `predefined_type='NOTDEFINED'` | `IfcCostSchedule` | Create a new cost schedule container |
| `add_cost_item` | `file`, `cost_schedule=None`, `cost_item=None` | `IfcCostItem` | Create a cost item (top-level or nested) |
| `add_cost_item_quantity` | `file`, `cost_item`, `ifc_class='IfcQuantityCount'` | `IfcPhysicalQuantity` | Attach a quantity to a cost item |
| `add_cost_value` | `file`, `parent` | `IfcCostValue` | Create a cost value or sub-value |
| `assign_cost_item_quantity` | `file`, `cost_item`, `products`, `prop_name=''` | `None` | Link cost item quantities parametrically to products |
| `assign_cost_value` | `file`, `cost_item`, `cost_rate` | `None` | Associate cost values from a rate schedule |
| `calculate_cost_item_resource_value` | `file`, `cost_item` | `None` | Compute total cost from assigned construction resources |
| `copy_cost_item` | `file`, `cost_item` | `IfcCostItem` | Duplicate a cost item with relationships |
| `copy_cost_item_values` | `file`, `source`, `destination` | `None` | Transfer cost values between cost items |
| `copy_cost_schedule` | `file`, `cost_schedule` | `IfcCostSchedule` | Duplicate an entire cost schedule |
| `edit_cost_item` | `file`, `cost_item`, `attributes` | `None` | Modify cost item attributes |
| `edit_cost_item_quantity` | `file`, `physical_quantity`, `attributes` | `None` | Update quantity attributes |
| `edit_cost_schedule` | `file`, `cost_schedule`, `attributes` | `None` | Change cost schedule attributes |
| `edit_cost_value` | `file`, `cost_value`, `attributes` | `None` | Modify cost value attributes |
| `edit_cost_value_formula` | `file`, `cost_value`, `formula` | `None` | Assign a spreadsheet-like formula |
| `remove_cost_item` | `file`, `cost_item` | `None` | Delete a cost item and associations |
| `remove_cost_item_quantity` | `file`, `cost_item`, `physical_quantity` | `None` | Remove a quantity from a cost item |
| `remove_cost_schedule` | `file`, `cost_schedule` | `None` | Remove a schedule and all nested items |
| `remove_cost_value` | `file`, `parent`, `cost_value` | `None` | Delete a cost value |
| `unassign_cost_item_quantity` | `file`, `cost_item`, `products` | `None` | Sever parametric quantity connections |

### 1.3 Cost Schedule Types

The `predefined_type` parameter for `add_cost_schedule` accepts values from `IfcCostScheduleTypeEnum`:

| Type | Description |
|------|-------------|
| `BUDGET` | Budget allocation |
| `COSTPLAN` | Cost plan / cost estimate |
| `ESTIMATE` | Preliminary cost estimate |
| `TENDER` | Bid/tender submission |
| `PRICEDBILLOFQUANTITIES` | Priced bill of quantities |
| `UNPRICEDBILLOFQUANTITIES` | Unpriced bill of quantities |
| `SCHEDULEOFRATES` | Schedule of unit rates (template) |
| `USERDEFINED` | Custom type (set `ObjectType` attribute) |
| `NOTDEFINED` | Default / unspecified |

### 1.4 Creating a Cost Breakdown Structure

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Step 1: Create cost schedule
schedule = ifcopenshell.api.run("cost.add_cost_schedule", model,
    name="Construction Estimate",
    predefined_type="COSTPLAN")

# Step 2: Create top-level cost items (WBS categories)
structural = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_schedule=schedule)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=structural,
    attributes={"Name": "Structural Works", "Identification": "01"})

# Step 3: Create nested cost items
foundations = ifcopenshell.api.run("cost.add_cost_item", model,
    cost_item=structural)
ifcopenshell.api.run("cost.edit_cost_item", model,
    cost_item=foundations,
    attributes={"Name": "Foundations", "Identification": "01.01"})
```

### 1.5 Cost Values: Direct, Unit-Based, and Formula

```python
# IFC4 / IFC4X3
# --- Approach 1: Direct fixed cost ---
item = ifcopenshell.api.run("cost.add_cost_item", model, cost_schedule=schedule)
value = ifcopenshell.api.run("cost.add_cost_value", model, parent=item)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=value,
    attributes={"AppliedValue": 42000.0})

# --- Approach 2: Unit cost x quantity ---
item2 = ifcopenshell.api.run("cost.add_cost_item", model, cost_schedule=schedule)
value2 = ifcopenshell.api.run("cost.add_cost_value", model, parent=item2)
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=value2,
    attributes={"AppliedValue": 85.0})  # EUR per m3

quantity = ifcopenshell.api.run("cost.add_cost_item_quantity", model,
    cost_item=item2, ifc_class="IfcQuantityVolume")
ifcopenshell.api.run("cost.edit_cost_item_quantity", model,
    physical_quantity=quantity,
    attributes={"VolumeValue": 120.0})
# Result: 85.0 * 120.0 = 10200.0

# --- Approach 3: Formula-based cost ---
item3 = ifcopenshell.api.run("cost.add_cost_item", model, cost_schedule=schedule)
value3 = ifcopenshell.api.run("cost.add_cost_value", model, parent=item3)
ifcopenshell.api.run("cost.edit_cost_value_formula", model,
    cost_value=value3,
    formula="5000 * 1.19")  # base cost * 19% tax
# Formula syntax follows ifcopenshell.util.cost conventions

# --- Approach 4: Composite sub-values ---
item4 = ifcopenshell.api.run("cost.add_cost_item", model, cost_schedule=schedule)
value4 = ifcopenshell.api.run("cost.add_cost_value", model, parent=item4)
sub_labor = ifcopenshell.api.run("cost.add_cost_value", model, parent=value4)
sub_material = ifcopenshell.api.run("cost.add_cost_value", model, parent=value4)

# Set parent category to "*" to sum sub-values
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=value4, attributes={"Category": "*"})
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=sub_labor, attributes={"AppliedValue": 2000.0, "Category": "Labor"})
ifcopenshell.api.run("cost.edit_cost_value", model,
    cost_value=sub_material, attributes={"AppliedValue": 3000.0, "Category": "Material"})
# Result: 2000 + 3000 = 5000.0
```

### 1.6 Parametric Quantity Linking

The `assign_cost_item_quantity` function creates a parametric connection between a cost item and product quantities. When product quantities change, cost calculations update automatically.

```python
# IFC4 / IFC4X3
# Create a slab with a volume quantity
slab = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcSlab", name="Ground Slab")
qto = ifcopenshell.api.run("pset.add_qto", model, product=slab, name="Qto_SlabBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={"NetVolume": 45.0})

# Link cost item to slab's NetVolume
ifcopenshell.api.run("cost.assign_cost_item_quantity", model,
    cost_item=item2,
    products=[slab],
    prop_name="NetVolume")
# The cost item now tracks the slab's volume parametrically.
# IMPORTANT: This ALSO creates a control relationship (IfcRelAssignsToControl)
# automatically -- do NOT call control.assign_control separately.

# Unlink when needed
ifcopenshell.api.run("cost.unassign_cost_item_quantity", model,
    cost_item=item2,
    products=[slab])
```

**IMPORTANT:** When `prop_name` is empty string `""`, the function counts the number of products instead of reading a named quantity. This is useful for item-based costing (e.g., cost per door, cost per window).

### 1.7 Utility Functions (ifcopenshell.util.cost)

The `ifcopenshell.util.cost` module provides 20+ read-only query and calculation functions:

| Function | Purpose |
|----------|---------|
| `get_root_cost_items(cost_schedule)` | Get top-level items in a schedule |
| `get_nested_cost_items(cost_item, is_deep)` | Get child items (shallow or deep) |
| `get_all_nested_cost_items(cost_item)` | Generator for all nested items recursively |
| `get_schedule_cost_items(cost_schedule)` | Generator for ALL items including nested |
| `get_cost_schedule(cost_item)` | Find parent schedule of a cost item |
| `get_cost_items_for_product(product)` | Find all cost items linked to a product |
| `get_cost_values(cost_item)` | Extract cost value data as dicts |
| `get_cost_rate(file, cost_item)` | Get associated rate |
| `get_total_quantity(root_element)` | Aggregate total quantity |
| `calculate_applied_value(root_element, cost_value, category_filter)` | Calculate the applied value with optional filtering |
| `serialise_cost_value(cost_value)` | Convert to string representation |
| `unserialise_cost_value(formula, cost_value)` | Parse formula string to structured data |

### 1.8 Anti-Patterns

- **NEVER** create `IfcCostItem` directly with `model.create_entity()`. ALWAYS use `cost.add_cost_item` which sets up the required `IfcRelNests` or `IfcRelAssignsToControl` relationship.
- **NEVER** pass both `cost_schedule` and `cost_item` to `add_cost_item`. These parameters are mutually exclusive. Exactly one MUST be provided.
- **NEVER** call `control.assign_control` separately after `assign_cost_item_quantity` -- the function handles the control relationship automatically.
- **NEVER** assume cost value arithmetic without setting the parent `Category` to `"*"`. Without this, sub-values are NOT summed.
- **ALWAYS** verify that products have the named quantity (via `pset.add_qto` / `pset.edit_qto`) before calling `assign_cost_item_quantity` with a `prop_name`.

---

## 2. 4D Scheduling (ifcopenshell.api.sequence)

### 2.1 Overview

The `ifcopenshell.api.sequence` module provides 40 functions for creating and managing construction schedules within IFC models. It supports hierarchical work breakdown structures, task sequencing, calendar-based scheduling, and automatic date cascading.

**Key IFC Entities:**
- `IfcWorkPlan` -- groups related work schedules (e.g., for baseline comparison)
- `IfcWorkSchedule` -- container for tasks (construction schedule or maintenance plan)
- `IfcTask` -- a single activity with optional time data, inputs, outputs, and resources
- `IfcTaskTime` -- temporal properties (start, finish, duration) for a task
- `IfcRelSequence` -- predecessor/successor relationship between tasks
- `IfcLagTime` -- delay between sequenced tasks
- `IfcWorkCalendar` -- defines working days, holidays, and exceptions
- `IfcWorkTime` -- working or exception time rules within a calendar
- `IfcRecurrencePattern` -- recurring intervals (daily, weekly, monthly, yearly)

### 2.2 Complete API Function Reference

| Function | Key Parameters | Returns | Purpose |
|----------|---------------|---------|---------|
| `add_work_plan` | `name`, `predefined_type`, `start_time` | `IfcWorkPlan` | Create a work plan container |
| `add_work_schedule` | `name`, `predefined_type`, `object_type`, `start_time`, `work_plan` | `IfcWorkSchedule` | Create a work schedule |
| `add_task` | `work_schedule`, `parent_task`, `name`, `description`, `identification`, `predefined_type` | `IfcTask` | Create a task (root or nested) |
| `add_task_time` | `task`, `is_recurring` | `IfcTaskTime` | Attach temporal data to a task |
| `add_work_calendar` | `name`, `predefined_type` | `IfcWorkCalendar` | Create a work calendar |
| `add_work_time` | `work_calendar`, `time_type` | `IfcWorkTime` | Add working/exception time to calendar |
| `add_time_period` | `recurrence_pattern`, `start_time`, `end_time` | Entity | Define time windows in recurrence |
| `add_date_time` | `dt` | `str` or Entity | Convert datetime to IFC format |
| `assign_sequence` | `relating_process`, `related_process`, `sequence_type='FINISH_START'` | `IfcRelSequence` | Create predecessor-successor link |
| `assign_lag_time` | `rel_sequence`, `lag_value`, `duration_type` | Entity | Add delay between tasks |
| `assign_process` | `relating_process`, `related_object` | Entity | Link inputs/controls/resources to task |
| `assign_product` | `relating_product`, `related_object` | Entity | Designate products as task outputs |
| `assign_recurrence_pattern` | `parent`, `recurrence_type` | Entity | Set recurring intervals |
| `assign_work_plan` | `work_schedule`, `work_plan` | Entity | Associate schedule with plan |
| `cascade_schedule` | `task` | `None` | Propagate dates through sequences |
| `recalculate_schedule` | -- | `None` | Recompute all schedule dates |
| `calculate_task_duration` | -- | -- | Compute duration from constraints |
| `create_baseline` | -- | -- | Create baseline for comparison |
| `copy_work_schedule` | -- | -- | Duplicate a schedule |
| `duplicate_task` | -- | -- | Replicate task with properties |
| `edit_task` | `task`, `attributes` | `None` | Modify task attributes |
| `edit_task_time` | `task_time`, `attributes` | `None` | Update temporal properties |
| `edit_sequence` | -- | `None` | Alter task relationships |
| `edit_lag_time` | -- | `None` | Modify lag durations |
| `edit_work_calendar` | -- | `None` | Change calendar definitions |
| `edit_work_plan` | -- | `None` | Modify work plan attributes |
| `edit_work_schedule` | -- | `None` | Update schedule properties |
| `edit_work_time` | -- | `None` | Revise working/exception times |
| `edit_recurrence_pattern` | -- | `None` | Update recurrence definitions |
| `remove_task` | `task` | `None` | Delete task and relationships |
| `remove_work_calendar` | -- | `None` | Delete calendar |
| `remove_work_plan` | -- | `None` | Remove work plan |
| `remove_work_schedule` | -- | `None` | Delete schedule |
| `remove_work_time` | -- | `None` | Remove working/exception time |
| `remove_time_period` | -- | `None` | Remove time period |
| `unassign_lag_time` | -- | `None` | Remove lag from sequence |
| `unassign_process` | -- | `None` | Detach inputs/resources |
| `unassign_product` | -- | `None` | Dissociate product outputs |
| `unassign_recurrence_pattern` | -- | `None` | Remove recurrence |
| `unassign_sequence` | -- | `None` | Remove task dependency |

### 2.3 Building a Construction Schedule

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Step 1: Create work plan (optional top-level grouping)
work_plan = ifcopenshell.api.run("sequence.add_work_plan", model,
    name="Main Construction Plan")

# Step 2: Create work schedule within the plan
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Phase 1 - Structure",
    predefined_type="PLANNED",
    work_plan=work_plan)

# Step 3: Create root tasks (top-level WBS)
task_foundations = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Foundations",
    identification="A")

# Step 4: Create subtasks
task_formwork = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=task_foundations,
    name="Formwork",
    identification="A.1")

task_rebar = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=task_foundations,
    name="Reinforcement",
    identification="A.2")

task_pour = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=task_foundations,
    name="Concrete Pour",
    identification="A.3")
```

### 2.4 Task Time and Duration

Task time data is blank by default. ALWAYS add time data with `add_task_time`, then configure with `edit_task_time`.

```python
# IFC4 / IFC4X3
# Add time to leaf tasks only (parent tasks are structural containers)
time_formwork = ifcopenshell.api.run("sequence.add_task_time", model,
    task=task_formwork)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=time_formwork,
    attributes={
        "ScheduleStart": "2026-04-01",      # ISO 8601 date string
        "ScheduleDuration": "P5D"            # ISO 8601 duration: 5 days
    })

time_rebar = ifcopenshell.api.run("sequence.add_task_time", model,
    task=task_rebar)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=time_rebar,
    attributes={"ScheduleDuration": "P3D"})

time_pour = ifcopenshell.api.run("sequence.add_task_time", model,
    task=task_pour)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=time_pour,
    attributes={"ScheduleDuration": "P1D"})
```

**Key `IfcTaskTime` Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `ScheduleStart` | ISO date string | Planned start date |
| `ScheduleFinish` | ISO date string | Planned finish date |
| `ScheduleDuration` | ISO duration | Planned duration (e.g., `"P5D"`, `"P2W"`) |
| `ActualStart` | ISO date string | Actual start date |
| `ActualFinish` | ISO date string | Actual finish date |
| `ActualDuration` | ISO duration | Actual duration |
| `EarlyStart` | ISO date string | CPM early start |
| `EarlyFinish` | ISO date string | CPM early finish |
| `LateStart` | ISO date string | CPM late start |
| `LateFinish` | ISO date string | CPM late finish |
| `FreeFloat` | ISO duration | Free float |
| `TotalFloat` | ISO duration | Total float |
| `Completion` | float | Percentage complete (0.0 to 1.0) |
| `DurationType` | string | `ELAPSEDDAYS`, `WORKTIME`, `CALENDARTIME` |

### 2.5 Task Sequencing and Dependencies

```python
# IFC4 / IFC4X3
# Create Finish-to-Start sequences (most common in construction)
seq1 = ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_formwork,
    related_process=task_rebar,
    sequence_type="FINISH_START")

seq2 = ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_rebar,
    related_process=task_pour,
    sequence_type="FINISH_START")

# Add lag time (e.g., 1 day curing before next activity)
ifcopenshell.api.run("sequence.assign_lag_time", model,
    rel_sequence=seq2,
    lag_value="P1D",
    duration_type="WORKTIME")

# Cascade dates: propagates from task_formwork through successors
ifcopenshell.api.run("sequence.cascade_schedule", model,
    task=task_formwork)
# After cascading:
# - task_rebar.ScheduleStart = task_formwork.ScheduleFinish
# - task_pour.ScheduleStart = task_rebar.ScheduleFinish + 1 day lag
```

**Sequence Types:**

| Type | Code | Description |
|------|------|-------------|
| Finish-to-Start | `FINISH_START` | Predecessor MUST finish before successor starts (default, most common) |
| Finish-to-Finish | `FINISH_FINISH` | Predecessor finish constrains successor finish |
| Start-to-Start | `START_START` | Predecessor start constrains successor start |
| Start-to-Finish | `START_FINISH` | Predecessor start constrains successor finish (rare) |

### 2.6 Work Calendars

```python
# IFC4 / IFC4X3
# Create a standard work calendar
calendar = ifcopenshell.api.run("sequence.add_work_calendar", model,
    name="Standard 5-Day Week")

# Add standard working time (Monday-Friday)
work_time = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="WorkingTimes")

# Set recurrence pattern: weekly on Mon-Fri
pattern = ifcopenshell.api.run("sequence.assign_recurrence_pattern", model,
    parent=work_time,
    recurrence_type="WEEKLY")
ifcopenshell.api.run("sequence.edit_recurrence_pattern", model,
    recurrence_pattern=pattern,
    attributes={"WeekdayComponent": [1, 2, 3, 4, 5]})  # Mon=1 through Fri=5

# Add working hours (9am-5pm)
ifcopenshell.api.run("sequence.add_time_period", model,
    recurrence_pattern=pattern,
    start_time="09:00:00",
    end_time="17:00:00")

# Add exception time (e.g., public holiday)
exception = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="ExceptionTimes")
```

### 2.7 Task-Product Relationships (4D BIM)

```python
# IFC4 / IFC4X3
# Link a task to the products it constructs (IfcRelAssignsToProcess)
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Exterior Wall A")

ifcopenshell.api.run("sequence.assign_process", model,
    relating_process=task_pour,
    related_object=wall)

# Query products for a task (outputs)
ifcopenshell.api.run("sequence.assign_product", model,
    relating_product=wall,
    related_object=task_pour)
```

### 2.8 Utility Functions (ifcopenshell.util.sequence)

| Function | Purpose |
|----------|---------|
| `get_root_tasks(work_schedule)` | Get top-level tasks in a schedule |
| `get_nested_tasks(task)` | Get immediate child tasks |
| `get_all_nested_tasks(task)` | Generator for all nested tasks recursively |
| `get_work_schedule_tasks(work_schedule)` | Generator for ALL tasks including nested |
| `get_parent_task(task)` | Find parent task in hierarchy |
| `get_task_work_schedule(task)` | Find which schedule contains a task |
| `get_calendar(task)` / `derive_calendar(task)` | Get work calendar for a task |
| `get_task_inputs(task, is_recursive)` | Get materials/resources consumed |
| `get_task_outputs(task, is_recursive)` | Get products generated |
| `get_task_resources(task, is_recursive)` | Get assigned resources |
| `get_tasks_for_product(product, schedule)` | Find tasks for a product |
| `get_related_products(relating_product, related_object)` | Get products output by task |
| `is_working_day(day, calendar)` | Check if a date is a work day |
| `count_working_days(start, finish, calendar)` | Count working days in range |
| `offset_date(start, duration, duration_type, calendar)` | Advance date by duration |
| `guess_date_range(work_schedule)` | Estimate schedule start/end from tasks |

### 2.9 IFC2X3 vs IFC4+ Differences

| Feature | IFC2X3 | IFC4 / IFC4X3 |
|---------|--------|----------------|
| Date format | `IfcDateAndTime` entity | ISO 8601 string |
| Task predefined types | Limited | `CONSTRUCTION`, `DEMOLITION`, `MAINTENANCE`, `MOVE`, `OPERATION`, `USERDEFINED`, `NOTDEFINED` |
| Work schedule predefined types | Limited | `ACTUAL`, `BASELINE`, `PLANNED`, `USERDEFINED`, `NOTDEFINED` |
| `add_date_time` return | `IfcDateAndTime` entity | Formatted string |

### 2.10 Anti-Patterns

- **NEVER** assign time data to parent tasks in a work breakdown structure. ONLY leaf tasks (tasks with no subtasks) receive `IfcTaskTime`. Parent tasks exist for organizational grouping only.
- **NEVER** pass both `work_schedule` and `parent_task` to `add_task`. These are mutually exclusive. A root task uses `work_schedule`; a subtask uses `parent_task`.
- **NEVER** create cyclical sequence relationships (A depends on B depends on A). `cascade_schedule` will recurse infinitely and crash.
- **ALWAYS** call `add_task_time` before `edit_task_time`. The task time entity MUST exist before editing its attributes.
- **ALWAYS** use ISO 8601 format for dates (`"2026-04-01"`) and durations (`"P5D"`) in IFC4+. Using Python `datetime` objects directly will cause errors.
- **NEVER** forget to call `cascade_schedule` after modifying task durations or sequences. Dates do NOT propagate automatically.

---

## 3. MEP Systems (ifcopenshell.api.system)

### 3.1 Overview

The `ifcopenshell.api.system` module provides 12 functions for creating and managing building distribution systems (HVAC, plumbing, electrical, fire protection). MEP systems in IFC use a port-based connection model where distribution elements connect through ports.

**Key IFC Entities:**
- `IfcDistributionSystem` -- logical grouping of distribution elements (IFC4+)
- `IfcSystem` -- generic system grouping (IFC2X3)
- `IfcBuildingSystem` -- facade and building envelope systems
- `IfcDistributionElement` -- base class for all MEP elements
- `IfcDistributionPort` -- connection point on an element (inlet/outlet)
- `IfcFlowSegment` -- duct segment, pipe segment, cable segment
- `IfcFlowFitting` -- elbow, tee, junction, transition
- `IfcFlowTerminal` -- air terminal, fixture, outlet, lamp
- `IfcFlowController` -- valve, damper, switch
- `IfcFlowMovingDevice` -- pump, fan, compressor
- `IfcFlowStorageDevice` -- tank, vessel
- `IfcFlowTreatmentDevice` -- filter, interceptor

### 3.2 Complete API Function Reference

| Function | Parameters | Returns | Purpose |
|----------|-----------|---------|---------|
| `add_system` | `file`, `ifc_class='IfcDistributionSystem'` | Entity | Create a distribution system |
| `edit_system` | `file`, `system`, `attributes` | `None` | Modify system attributes |
| `remove_system` | `file`, `system` | `None` | Delete system (keeps elements) |
| `assign_system` | `file`, `products`, `system` | Entity or `None` | Add elements to a system |
| `unassign_system` | `file`, `products`, `system` | `None` | Remove elements from system |
| `add_port` | `file`, `element=None` | `IfcDistributionPort` | Create a connection port |
| `assign_port` | `file`, `element`, `port` | Entity | Assign orphaned port to element |
| `unassign_port` | `file`, `element`, `port` | `None` | Remove port from element |
| `connect_port` | `file`, `port1`, `port2`, `direction='NOTDEFINED'`, `element=None` | `None` | Connect two ports |
| `disconnect_port` | `file`, `port` | `None` | Disconnect a port |
| `assign_flow_control` | `file`, `relating_flow_element`, `related_flow_control` | Entity or `None` | Assign control to flow element |
| `unassign_flow_control` | `file`, `relating_flow_element`, `related_flow_control` | `None` | Remove control assignment |

### 3.3 Creating an HVAC System

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Step 1: Create the system
hvac_system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=hvac_system,
    attributes={"Name": "HVAC Supply Air", "PredefinedType": "VENTILATION"})

# Step 2: Create distribution elements
ahu = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAirToAirHeatRecovery", name="AHU-01")
duct1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Supply Duct 01")
duct2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Supply Duct 02")
terminal = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAirTerminal", name="Supply Diffuser 01")

# Step 3: Assign elements to the system
ifcopenshell.api.run("system.assign_system", model,
    products=[ahu, duct1, duct2, terminal],
    system=hvac_system)
```

### 3.4 Port-Based Connections

Every distribution element MUST have ports to participate in flow connections. Ports define the connection points, flow direction, and system type.

```python
# IFC4 / IFC4X3
# Create ports for duct segments (each segment needs 2 ports: inlet + outlet)
duct1_inlet = ifcopenshell.api.run("system.add_port", model, element=duct1)
duct1_outlet = ifcopenshell.api.run("system.add_port", model, element=duct1)

duct2_inlet = ifcopenshell.api.run("system.add_port", model, element=duct2)
duct2_outlet = ifcopenshell.api.run("system.add_port", model, element=duct2)

# Create port for AHU outlet
ahu_outlet = ifcopenshell.api.run("system.add_port", model, element=ahu)

# Create port for terminal inlet
terminal_inlet = ifcopenshell.api.run("system.add_port", model, element=terminal)

# Connect ports in sequence: AHU -> Duct1 -> Duct2 -> Terminal
ifcopenshell.api.run("system.connect_port", model,
    port1=ahu_outlet, port2=duct1_inlet, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=duct1_outlet, port2=duct2_inlet, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=duct2_outlet, port2=terminal_inlet, direction="SOURCE")
```

**Port Flow Directions:**

| Direction | Meaning |
|-----------|---------|
| `SOURCE` | Flow exits through this port (outlet) |
| `SINK` | Flow enters through this port (inlet) |
| `SOURCEANDSINK` | Bidirectional flow |
| `NOTDEFINED` | Direction not specified |

**IMPORTANT:** For two ports to connect successfully, they MUST have:
1. Compatible system types
2. Opposite flow directions (one `SOURCE`, one `SINK`)
3. Aligned placement data (position and orientation)

### 3.5 Flow Control Assignment

```python
# IFC4 / IFC4X3
# Create a damper that controls airflow in a duct
damper = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDamper", name="Volume Damper 01")

ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct1,
    related_flow_control=damper)
```

### 3.6 System Traversal (ifcopenshell.util.system)

The `ifcopenshell.util.system` module provides functions for querying system topology:

```python
# IFC4 / IFC4X3
import ifcopenshell.util.system

# Get all elements in a system
elements = ifcopenshell.util.system.get_system_elements(hvac_system)

# Get all systems an element belongs to
systems = ifcopenshell.util.system.get_element_systems(duct1)

# Get ports on an element
ports = ifcopenshell.util.system.get_ports(duct1)
source_ports = ifcopenshell.util.system.get_ports(duct1, flow_direction="SOURCE")

# Traverse connections: what is connected downstream?
downstream = ifcopenshell.util.system.get_connected_to(duct1)

# Traverse connections: what is connected upstream?
upstream = ifcopenshell.util.system.get_connected_from(duct1)

# Get the element that owns a port
element = ifcopenshell.util.system.get_port_element(duct1_outlet)

# Get the port connected to this port
connected = ifcopenshell.util.system.get_connected_port(duct1_outlet)

# Check if a product can be assigned to a system
can_assign = ifcopenshell.util.system.is_assignable(duct1, hvac_system)
```

### 3.7 IFC2X3 vs IFC4+ Differences

| Feature | IFC2X3 | IFC4 / IFC4X3 |
|---------|--------|----------------|
| System class | `IfcSystem` | `IfcDistributionSystem` (preferred) |
| System types | Limited predefined types | Extensive: `VENTILATION`, `HEATING`, `COOLING`, `PLUMBING`, `ELECTRICAL`, `FIREPROTECTION`, etc. |
| Port nesting | `IfcRelNests` | `IfcRelNests` |
| Building systems | Not available | `IfcBuildingSystem` for facades |

### 3.8 Anti-Patterns

- **NEVER** connect two ports with the same flow direction. SOURCE MUST connect to SINK.
- **NEVER** forget to create ports before connecting elements. Elements without ports cannot participate in flow networks.
- **NEVER** use `IfcDistributionSystem` in IFC2X3 models. Use `IfcSystem` instead (pass `ifc_class="IfcSystem"` to `add_system`).
- **ALWAYS** create at least two ports per segment element (inlet and outlet).
- **ALWAYS** assign elements to a system after creating the port connections. The system is a logical grouping; ports define the physical topology.
- **NEVER** assume `remove_system` deletes contained elements. It ONLY removes the system grouping; elements remain in the model.

---

## 4. Drawing / 2D (ifcopenshell.api.drawing)

### 4.1 Overview

The `ifcopenshell.api.drawing` module is a minimal API with only 3 functions. It handles the association between 2D annotations and 3D products in IFC models. The actual 2D drawing generation is primarily handled by the Bonsai BIM tool (not the IfcOpenShell API directly).

**Key IFC Entities:**
- `IfcAnnotation` -- 2D annotation element (text, dimension, leader, symbol)
- `IfcTextLiteral` -- text content within an annotation
- `IfcRelAssignsToProduct` -- links annotation to a product for "smart" annotations

### 4.2 Complete API Function Reference

| Function | Parameters | Returns | Purpose |
|----------|-----------|---------|---------|
| `assign_product` | `file`, `relating_product`, `related_object` | Entity | Link annotation to a product |
| `unassign_product` | `file`, `relating_product`, `related_object` | `None` | Unlink annotation from product |
| `edit_text_literal` | `file`, `text_literal`, `attributes` | `None` | Modify text annotation properties |

### 4.3 Smart Annotations

"Smart" annotations reference product attributes, properties, and relationships. When the product changes, the annotation can update accordingly.

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Create a wall and an annotation
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Wall A-01")
annotation = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAnnotation", name="Wall Label")

# Link annotation to wall (makes it "smart")
ifcopenshell.api.run("drawing.assign_product", model,
    relating_product=wall,
    related_object=annotation)

# Later, unlink if annotation should become standalone
ifcopenshell.api.run("drawing.unassign_product", model,
    relating_product=wall,
    related_object=annotation)
```

### 4.4 Text Literals

```python
# IFC4 / IFC4X3
# Edit text content of an annotation's text literal
# First, find or create the text literal entity
text_literal = model.create_entity("IfcTextLiteralWithExtent",
    Literal="Wall Type: RC200",
    Placement=model.create_entity("IfcAxis2Placement2D"),
    Path="RIGHT")

ifcopenshell.api.run("drawing.edit_text_literal", model,
    text_literal=text_literal,
    attributes={"Literal": "Wall Type: RC250 (Updated)"})
```

### 4.5 Relationship to Bonsai Drawing Module

**IMPORTANT:** The `ifcopenshell.api.drawing` module is intentionally minimal. The comprehensive 2D drawing generation workflow (creating drawing views, sheets, SVG output, dimension annotations, section cuts) is implemented in the **Bonsai BIM tool** (`bonsai.bim.module.drawing`), which uses this API internally. For full drawing generation capabilities, use Bonsai's drawing module or operator calls.

### 4.6 Anti-Patterns

- **NEVER** expect `ifcopenshell.api.drawing` to generate complete 2D drawings from 3D models. That functionality lives in Bonsai.
- **NEVER** use `drawing.assign_product` for non-annotation objects. The `related_object` MUST be an `IfcAnnotation`.
- **ALWAYS** use `drawing.edit_text_literal` instead of directly modifying text literal attributes with `attribute.edit_attributes` to ensure proper update handling.

---

## 5. Profiles (ifcopenshell.api.profile)

### 5.1 Overview

The `ifcopenshell.api.profile` module provides 6 functions for creating and managing cross-section profiles used in extruded geometry (beams, columns, walls, slabs). Profiles define the 2D shape that is swept along a path to create 3D geometry.

**Key IFC Entities:**
- `IfcProfileDef` -- abstract base class for all profiles
- `IfcParameterizedProfileDef` -- profiles defined by dimensional parameters
- `IfcArbitraryClosedProfileDef` -- profiles defined by an arbitrary polyline
- `IfcArbitraryProfileDefWithVoids` -- arbitrary profiles with holes
- `IfcCompositeProfileDef` -- multiple profiles combined into one

### 5.2 Complete API Function Reference

| Function | Parameters | Returns | Purpose |
|----------|-----------|---------|---------|
| `add_parameterised_profile` | `file`, `ifc_class`, `profile_type='AREA'` | Entity | Create a standard parametric profile |
| `add_arbitrary_profile` | `file`, `profile` (coordinates), `name=None` | Entity | Create a custom polyline profile |
| `add_arbitrary_profile_with_voids` | `file`, `outer_profile`, `inner_profiles`, `name=None` | Entity | Create a profile with holes |
| `edit_profile` | `file`, `profile`, `attributes` | `None` | Modify profile attributes |
| `copy_profile` | `file`, `profile` | Entity | Duplicate a profile |
| `remove_profile` | `file`, `profile` | `None` | Delete a profile |

**IMPORTANT:** The function name in the API is `add_parameterised_profile` (British spelling). Using `add_parameterized_profile` (American spelling) will cause an error.

### 5.3 Parametric Profile Types

All parametric profiles inherit from `IfcParameterizedProfileDef` and are defined by standard dimensional parameters:

| IFC Class | Shape | Key Parameters |
|-----------|-------|----------------|
| `IfcRectangleProfileDef` | Rectangle | `XDim`, `YDim` |
| `IfcRectangleHollowProfileDef` | Hollow rectangle | `XDim`, `YDim`, `WallThickness` |
| `IfcRoundedRectangleProfileDef` | Rounded rectangle | `XDim`, `YDim`, `RoundingRadius` |
| `IfcCircleProfileDef` | Circle | `Radius` |
| `IfcCircleHollowProfileDef` | Hollow circle / tube | `Radius`, `WallThickness` |
| `IfcEllipseProfileDef` | Ellipse | `SemiAxis1`, `SemiAxis2` |
| `IfcIShapeProfileDef` | I-beam / H-beam | `OverallWidth`, `OverallDepth`, `WebThickness`, `FlangeThickness`, `FilletRadius` |
| `IfcAsymmetricIShapeProfileDef` | Asymmetric I-beam | Top and bottom flange dimensions independently |
| `IfcTShapeProfileDef` | T-shape | `Depth`, `FlangeWidth`, `WebThickness`, `FlangeThickness` |
| `IfcLShapeProfileDef` | L-shape / angle | `Depth`, `Width`, `Thickness` |
| `IfcUShapeProfileDef` | U-shape / channel | `Depth`, `FlangeWidth`, `WebThickness`, `FlangeThickness` |
| `IfcCShapeProfileDef` | C-shape | `Depth`, `Width`, `WallThickness`, `Girth` |
| `IfcZShapeProfileDef` | Z-shape | `Depth`, `FlangeWidth`, `WebThickness`, `FlangeThickness` |
| `IfcTrapeziumProfileDef` | Trapezoid | `BottomXDim`, `TopXDim`, `YDim`, `TopXOffset` |

### 5.4 Creating Parametric Profiles

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# I-beam profile (e.g., HEA 200)
i_profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcIShapeProfileDef")
ifcopenshell.api.run("profile.edit_profile", model,
    profile=i_profile,
    attributes={
        "ProfileName": "HEA 200",
        "OverallWidth": 0.200,      # meters (ALWAYS SI units)
        "OverallDepth": 0.190,
        "WebThickness": 0.0065,
        "FlangeThickness": 0.010,
        "FilletRadius": 0.018
    })

# Circular hollow section (e.g., CHS 168.3x8)
pipe_profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcCircleHollowProfileDef")
ifcopenshell.api.run("profile.edit_profile", model,
    profile=pipe_profile,
    attributes={
        "ProfileName": "CHS 168.3x8",
        "Radius": 0.08415,         # outer radius in meters
        "WallThickness": 0.008
    })

# Rectangle profile (e.g., concrete column 400x600)
rect_profile = ifcopenshell.api.run("profile.add_parameterised_profile", model,
    ifc_class="IfcRectangleProfileDef")
ifcopenshell.api.run("profile.edit_profile", model,
    profile=rect_profile,
    attributes={
        "ProfileName": "RC 400x600",
        "XDim": 0.400,
        "YDim": 0.600
    })
```

### 5.5 Creating Arbitrary Profiles

For non-standard shapes, use `add_arbitrary_profile` with a list of 2D coordinate tuples. Coordinates MUST be in SI meters.

```python
# IFC4 / IFC4X3
# Custom L-shaped profile (non-standard dimensions)
custom_profile = ifcopenshell.api.run("profile.add_arbitrary_profile", model,
    profile=[
        (0.0, 0.0),
        (0.3, 0.0),
        (0.3, 0.05),
        (0.05, 0.05),
        (0.05, 0.2),
        (0.0, 0.2)
    ],
    name="Custom L 300x200x50")

# Profile with voids (e.g., hollow section with internal opening)
hollow_profile = ifcopenshell.api.run("profile.add_arbitrary_profile_with_voids", model,
    outer_profile=[
        (0.0, 0.0), (0.5, 0.0), (0.5, 0.5), (0.0, 0.5)
    ],
    inner_profiles=[
        [(0.05, 0.05), (0.45, 0.05), (0.45, 0.45), (0.05, 0.45)]
    ],
    name="Hollow Box 500x500")
```

### 5.6 Using Profiles in Geometry

Profiles are used with `geometry.add_profile_representation` to create extruded solids:

```python
# IFC4 / IFC4X3
# Create representation context
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

# Create a column with the I-profile
column = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcColumn", name="Column C-01")
representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body,
    profile=i_profile,
    depth=3.5)  # extrusion height in meters
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
```

### 5.7 Profile Type Parameter

The `profile_type` parameter in `add_parameterised_profile` accepts:

| Value | Meaning |
|-------|---------|
| `AREA` | Used for solid extrusions (default, most common) |
| `CURVE` | Used for curve-based representations (centerline) |

### 5.8 Anti-Patterns

- **NEVER** use millimeters for profile dimensions. IfcOpenShell expects SI meters. A 200mm flange width MUST be specified as `0.200`.
- **NEVER** create profiles with `model.create_entity()` directly. ALWAYS use the `profile.add_*` functions which handle proper schema setup.
- **NEVER** leave the profile polyline open (first point != last point) in `add_arbitrary_profile`. The API closes the loop automatically, but providing an explicit closure is clearer.
- **NEVER** misspell `add_parameterised_profile` as `add_parameterized_profile`. The API uses British spelling.
- **ALWAYS** set `ProfileName` via `edit_profile` after creation for identification in BIM viewers.
- **ALWAYS** verify that the `ifc_class` string exactly matches an IFC schema class name (case-sensitive).

---

## 6. Validation & Georeferencing

### 6.1 Validation (ifcopenshell.validate)

#### 6.1.1 Overview

The `ifcopenshell.validate` module provides schema-level validation of IFC files. It checks entity attributes against the EXPRESS schema definition, verifying types, cardinality, enumerations, and optionally WHERE rules.

#### 6.1.2 Core Validation Function

```python
import ifcopenshell
import ifcopenshell.validate
import logging

model = ifcopenshell.open("building.ifc")

# Basic validation with default logger
logger = logging.getLogger("ifcopenshell.validate")
logger.setLevel(logging.WARNING)
handler = logging.StreamHandler()
logger.addHandler(handler)

ifcopenshell.validate.validate(model, logger)
```

#### 6.1.3 Validation Function Signature

```python
ifcopenshell.validate.validate(
    f,                    # ifcopenshell.file or filepath string
    logger,               # logging.Logger instance
    express_rules=False   # Enable EXPRESS WHERE rule checking (slower)
)
```

**Validation checks performed:**
- Entity attribute type correctness (string, integer, float, entity reference)
- Inverse attribute cardinality (min/max counts of referencing entities)
- Simple type validation (IfcLabel, IfcLengthMeasure, etc.)
- Select type validation (value is a valid member of the select)
- Enumeration validation (value is a valid enum member)
- Aggregation validation (list/set size constraints)
- GUID format validation (22-character base64 format)
- File header structure validation
- Application reference validation

#### 6.1.4 JSON Logger for Programmatic Access

```python
# IFC4 / IFC4X3
import ifcopenshell.validate

model = ifcopenshell.open("building.ifc")

# Use json_logger for structured output
json_log = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, json_log)

# Access validation results programmatically
for statement in json_log.statements:
    print(f"Level: {statement['level']}, Message: {statement['message']}")
```

#### 6.1.5 Individual Validation Functions

| Function | Purpose |
|----------|---------|
| `validate(f, logger, express_rules)` | Full model validation |
| `validate_guid(guid)` | Check GUID format (returns `None` if valid, error string otherwise) |
| `validate_ifc_header(f, logger)` | Validate file header structure |
| `validate_ifc_applications(f, logger)` | Validate application references |
| `assert_valid(attr_type, val, schema, no_throw, attr)` | Validate a single attribute value |
| `assert_valid_inverse(attr, val, schema)` | Validate inverse attribute cardinality |

#### 6.1.6 EXPRESS WHERE Rules

```python
# IFC4 / IFC4X3
# Enable WHERE rule checking (significantly slower)
ifcopenshell.validate.validate(model, logger, express_rules=True)
# WHERE rules check semantic constraints beyond type checking, e.g.:
# - IfcWall.OverallHeight > 0
# - IfcSite.RefLatitude has valid range
# - IfcProject has exactly one IfcUnitAssignment
```

**IMPORTANT:** EXPRESS WHERE rule validation requires the C++ backend to evaluate EXPRESS expressions. This is significantly slower than basic schema validation (10-100x slower on large models). ALWAYS run basic validation first, fix those issues, then enable EXPRESS rules.

#### 6.1.7 LogDetectionHandler

```python
# IFC4 / IFC4X3
# Detect if ANY validation issues were logged
detection_handler = ifcopenshell.validate.LogDetectionHandler()
logger.addHandler(detection_handler)

ifcopenshell.validate.validate(model, logger)

if detection_handler.message_logged:
    print("Validation issues found")
else:
    print("Model is valid")
```

### 6.2 Georeferencing (ifcopenshell.api.georeference)

#### 6.2.1 Overview

The `ifcopenshell.api.georeference` module provides 5 functions for managing map coordinate systems and true north orientation in IFC models. Georeferencing translates between local model coordinates and real-world map coordinates.

**Key IFC Entities:**
- `IfcMapConversion` -- transformation from local to map coordinates (IFC4+)
- `IfcMapConversionScaled` -- map conversion with scale factor (IFC4X3)
- `IfcProjectedCRS` -- projected coordinate reference system definition
- `IfcGeometricRepresentationContext` -- contains WCS and true north

#### 6.2.2 Complete API Function Reference

| Function | Parameters | Returns | Purpose |
|----------|-----------|---------|---------|
| `add_georeferencing` | `file`, `ifc_class='IfcMapConversion'`, `name='EPSG:3857'` | `None` | Create empty georeferencing entities |
| `edit_georeferencing` | `file`, `coordinate_operation=None`, `projected_crs=None` | `None` | Modify map conversion and CRS |
| `edit_true_north` | `file`, `true_north=0.0` | `None` | Set true north orientation |
| `edit_wcs` | `file`, `x=0.0`, `y=0.0`, `z=0.0`, `rotation=0.0`, `is_si=True` | `None` | Adjust world coordinate system |
| `remove_georeferencing` | `file` | `None` | Remove all georeferencing data |

#### 6.2.3 Setting Up Georeferencing

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api
from math import cos, sin, radians

model = ifcopenshell.file(schema="IFC4")

# Step 1: Add empty georeferencing entities
ifcopenshell.api.run("georeference.add_georeferencing", model)

# Step 2: Configure the coordinate reference system and map conversion
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    projected_crs={"Name": "EPSG:28992"},  # Amersfoort / RD New (Netherlands)
    coordinate_operation={
        "Eastings": 155000.0,          # False origin easting
        "Northings": 463000.0,         # False origin northing
        "OrthogonalHeight": 0.0,       # Height above reference
        "XAxisAbscissa": cos(radians(-5.0)),  # Project North rotation
        "XAxisOrdinate": sin(radians(-5.0)),
        "Scale": 1.0
    })

# Step 3: Set true north (optional, for solar analysis)
ifcopenshell.api.run("georeference.edit_true_north", model,
    true_north=5.0)  # Degrees anticlockwise from Y-axis
```

#### 6.2.4 Minimal Georeferencing (Horizontal Construction)

```python
# IFC4X3 -- For infrastructure projects where model origin = map origin
ifcopenshell.api.run("georeference.add_georeferencing", model)
ifcopenshell.api.run("georeference.edit_georeferencing", model,
    projected_crs={"Name": "EPSG:7856"})
```

#### 6.2.5 World Coordinate System Adjustment

```python
# IFC4 / IFC4X3
# Move the WCS origin (affects all geometric contexts)
ifcopenshell.api.run("georeference.edit_wcs", model,
    x=10.0, y=20.0, z=0.0,
    rotation=0.0,
    is_si=True)  # Coordinates in SI meters

# IMPORTANT: Keep WCS at origin (0,0,0) unless there is a specific
# surveying requirement. Moving WCS affects ALL local placements.
```

#### 6.2.6 IFC2X3 vs IFC4+ Differences

| Feature | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| Map conversion | Property set on IfcProject | `IfcMapConversion` | `IfcMapConversion` or `IfcMapConversionScaled` |
| CRS definition | Property set | `IfcProjectedCRS` | `IfcProjectedCRS` |
| MapUnit attribute | String (full unit name) | `IfcNamedUnit` object | `IfcNamedUnit` object |
| Removal | Removes property sets | Removes entities | Removes entities |

#### 6.2.7 Coordinate Conversion Utilities

The `ifcopenshell.util.geolocation` module (not covered in detail here) provides utility functions for converting between local model coordinates and map coordinates.

#### 6.2.8 Anti-Patterns

- **NEVER** set georeferencing without consulting the project surveyor. Incorrect Eastings/Northings will place the building in the wrong location on Earth.
- **NEVER** confuse Project North with True North. `XAxisAbscissa` and `XAxisOrdinate` in the coordinate operation use **Project North** (the angle from Grid North to Project North). True North is set separately.
- **NEVER** use `IfcMapConversionScaled` in IFC4 models. The scaled variant is only available in IFC4X3.
- **ALWAYS** call `add_georeferencing` before `edit_georeferencing`. The entities MUST exist before editing.
- **ALWAYS** use EPSG codes in the format `"EPSG:XXXXX"` for the CRS Name attribute.
- **NEVER** move the WCS from origin without a specific surveying reason. This affects all geometry in the model.

---

## 7. Python Runtime Quirks

### 7.1 C++ Binding Behavior

IfcOpenShell's Python interface wraps a C++ core engine. The `entity_instance` class delegates attribute access and operations to C++ objects via `wrapped_data`.

```python
# The Python wrapper delegates to C++ bindings
wall = model.by_type("IfcWall")[0]
type(wall)                    # <class 'ifcopenshell.entity_instance'>
type(wall.wrapped_data)       # <class 'ifcopenshell_wrapper.entity_instance'>

# The file property returns the parent ifcopenshell.file
wall.file  # Returns the model object
```

**IMPORTANT:** The wrapper maintains a reference to the parent file object. If the file object is garbage collected, accessing `wrapped_data` on any entity from that file causes a segmentation fault or undefined behavior. ALWAYS maintain a reference to the file object as long as any entities from it are in use.

### 7.2 Entity Identity: `is` vs `==`

```python
# IFC4 / IFC4X3
wall = model.by_type("IfcWall")[0]

# Two separate queries return DIFFERENT Python wrapper objects
wall_a = model.by_id(wall.id())
wall_b = model.by_id(wall.id())

wall_a == wall_b    # True  (value equality via wrapped_data comparison)
wall_a is wall_b    # False (different Python wrapper objects)

# ALWAYS use == for entity comparison, NEVER use 'is'
# For identity checks, use .id():
wall_a.id() == wall_b.id()  # True (same STEP ID)
```

### 7.3 File Lifecycle and Entity Invalidation

```python
# IFC4 / IFC4X3
wall = model.by_type("IfcWall")[0]

# After removing an entity, ALL references to it become INVALID
model.remove(wall)  # Low-level removal

# Accessing wall after removal causes undefined behavior:
# wall.Name  # CRASH or garbage data -- the C++ object is deallocated

# ALWAYS nullify Python references after removal
wall = None

# The API's root.remove_product is safer (cleans up relationships first):
ifcopenshell.api.run("root.remove_product", model, product=other_wall)
# But even here, the Python reference becomes invalid afterward
```

**IMPORTANT:** This behavior is fundamentally different from pure Python objects. In Python, objects persist as long as references exist. In IfcOpenShell, the C++ backend owns the entity data. When `model.remove()` is called, the C++ object is deallocated regardless of Python reference count.

### 7.4 Schema-Specific Attribute Errors

```python
# IFC2X3 vs IFC4 -- Attribute availability differs by schema

# IfcWallStandardCase exists in IFC2X3/IFC4 but NOT in IFC4X3
# In IFC4X3, use IfcWall with PredefinedType instead

# Some attributes exist in one schema but not another:
# IFC4: IfcTask.PredefinedType exists
# IFC2X3: IfcTask has no PredefinedType attribute

# Accessing a non-existent attribute raises AttributeError:
try:
    task = model.by_type("IfcTask")[0]
    pt = task.PredefinedType  # Raises AttributeError in IFC2X3
except AttributeError:
    pass

# ALWAYS check the schema before accessing schema-specific attributes:
if model.schema == "IFC4" or model.schema == "IFC4X3":
    predefined_type = task.PredefinedType
```

### 7.5 Performance: by_type() Index Behavior

```python
# IFC4 / IFC4X3
# by_type() builds an internal index on FIRST call
# Subsequent calls with the SAME type are fast (index lookup)

walls = model.by_type("IfcWall")       # First call: builds index (slow for large models)
walls2 = model.by_type("IfcWall")      # Second call: instant (cached index)

# by_type() returns a TUPLE, not a list
type(walls)  # <class 'tuple'>

# For large models (100k+ entities), by_type() is ALWAYS faster than
# manual iteration:

# GOOD: Use by_type for repeated queries
walls = model.by_type("IfcWall")
for wall in walls:
    process(wall)

# BAD: Do NOT iterate all entities and filter manually
# This is 10-100x slower than by_type():
for entity in model:              # Iterates ALL entities
    if entity.is_a("IfcWall"):    # Manual type check
        process(entity)
```

### 7.6 Batch Operations and Performance

```python
# IFC4 / IFC4X3
# For bulk modifications, minimize API calls where possible

# SLOW: Individual API calls in a loop
for i in range(1000):
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=f"Wall {i}")

# FASTER: Use batch_mode context manager (if available in your version)
# Note: batch_mode reduces intermediate validation overhead

# For reading data, use get_info() for bulk attribute extraction:
wall = model.by_type("IfcWall")[0]
info = wall.get_info()  # Returns dict of ALL attributes at once
# info = {"id": 42, "type": "IfcWall", "GlobalId": "...", "Name": "...", ...}

# For recursive data extraction (expensive, use sparingly):
deep_info = wall.get_info(recursive=True)

# For scalar-only extraction (faster, skips entity references):
scalar_info = wall.get_info(scalar_only=True)
```

### 7.7 Thread Safety

**IfcOpenShell is NOT thread-safe for write operations.** The C++ backend does not use locks or atomic operations for entity creation, modification, or removal.

```python
# DANGEROUS: Concurrent writes from multiple threads
import threading

def create_walls(model, start, count):
    for i in range(start, start + count):
        # This WILL corrupt the model or crash
        ifcopenshell.api.run("root.create_entity", model,
            ifc_class="IfcWall", name=f"Wall {i}")

# NEVER do this:
# t1 = threading.Thread(target=create_walls, args=(model, 0, 100))
# t2 = threading.Thread(target=create_walls, args=(model, 100, 100))
# t1.start(); t2.start(); t1.join(); t2.join()

# SAFE: Read operations from multiple threads are generally safe
# as long as no thread is writing simultaneously

# SAFE: Use separate model instances per thread
def process_model_copy(filepath, output_path):
    local_model = ifcopenshell.open(filepath)
    # Modify local_model freely -- it is an independent instance
    local_model.write(output_path)

# For parallel processing of large datasets:
# 1. Open separate file instances per thread
# 2. Process independently
# 3. Merge results in a single thread
```

### 7.8 Memory Management for Large Models

```python
# IFC4 / IFC4X3
# Large models (500MB+ IFC files) require careful memory management

# Standard open loads ENTIRE file into memory:
model = ifcopenshell.open("large_building.ifc")  # May use 4-8GB RAM

# For read-only analysis, consider streaming/chunked processing:
# Process elements in batches rather than loading all relationships at once

# get_info(recursive=True) is EXTREMELY expensive on large models
# because it materializes the entire entity graph:
# wall.get_info(recursive=True)  # Can trigger massive memory allocation

# INSTEAD, access specific attributes directly:
name = wall.Name                    # Accesses one attribute
material = ifcopenshell.util.element.get_material(wall)  # Targeted query

# Garbage collection: Python GC does NOT control C++ memory
# The C++ backend allocates memory independently
# model.remove(entity) frees C++ memory but Python wrappers may linger

# To free a large model from memory:
del model  # Releases the C++ file object
import gc
gc.collect()  # Ensures Python wrappers are cleaned up
```

### 7.9 Attribute Access Patterns

```python
# IFC4 / IFC4X3
wall = model.by_type("IfcWall")[0]

# Named attribute access (Pythonic, preferred)
name = wall.Name
global_id = wall.GlobalId

# Positional attribute access (index-based)
name = wall[2]  # Name is the 3rd attribute (0-indexed)

# IMPORTANT: Positional indices vary by entity type and schema version
# ALWAYS use named access for maintainability

# Check if an attribute has a value (vs $null in IFC)
if wall.Description is not None:
    print(wall.Description)

# is_a() checks class hierarchy (includes parent classes)
wall.is_a("IfcWall")             # True
wall.is_a("IfcBuildingElement")  # True (parent class)
wall.is_a("IfcProduct")          # True (grandparent class)
wall.is_a("IfcSlab")             # False

# get_info() returns a dict with all attributes
info = wall.get_info()
# Keys include: 'id', 'type', plus all schema attributes
```

### 7.10 Common Gotchas Summary

| Gotcha | Description | Solution |
|--------|-------------|----------|
| Entity invalidation | Removed entities cause crashes when accessed | Set references to `None` after removal |
| `is` vs `==` | Different wrapper objects for same entity | ALWAYS use `==` or compare `.id()` |
| File GC | Deleting file object invalidates all entities | Keep file reference alive |
| Schema attributes | Accessing absent attributes raises `AttributeError` | Check `model.schema` first |
| Thread writes | Concurrent writes corrupt the model | Use single-threaded writes or separate files |
| `by_type` returns tuple | Cannot `.append()` to result | Convert to list if mutation needed |
| Unit assumptions | API expects SI meters | ALWAYS convert to meters before passing values |
| `get_info(recursive=True)` | Massive memory allocation on large models | Use direct attribute access instead |
| Positional vs named access | Index varies by schema and type | ALWAYS use named attribute access |
| `model.remove()` vs API | Low-level remove does NOT clean relationships | Use `root.remove_product` instead |

---

## 8. Sources

- IfcOpenShell API Documentation -- Cost Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/cost/index.html
- IfcOpenShell API Documentation -- Sequence Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/sequence/index.html
- IfcOpenShell API Documentation -- System Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/system/index.html
- IfcOpenShell API Documentation -- Drawing Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/drawing/index.html
- IfcOpenShell API Documentation -- Profile Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/profile/index.html
- IfcOpenShell API Documentation -- Georeference Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/georeference/index.html
- IfcOpenShell API Documentation -- Validate Module: https://docs.ifcopenshell.org/autoapi/ifcopenshell/validate/index.html
- IfcOpenShell Utility Documentation -- Cost: https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/cost/index.html
- IfcOpenShell Utility Documentation -- Sequence: https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/sequence/index.html
- IfcOpenShell Utility Documentation -- System: https://docs.ifcopenshell.org/autoapi/ifcopenshell/util/system/index.html
- IfcOpenShell Entity Instance Documentation: https://docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html
- IfcOpenShell Python Documentation: https://docs.ifcopenshell.org/ifcopenshell-python.html
- IFC 4.3 Documentation -- Profile Resource: https://ifc43-docs.standards.buildingsmart.org/IFC/RELEASE/IFC4x3/HTML/ifcprofileresource/content.html
- IFC 4.3 Documentation -- IfcDistributionPort: https://ifc43-docs.standards.buildingsmart.org/IFC/RELEASE/IFC4x3/HTML/lexical/IfcDistributionPort.htm
- IFC 4.3 Documentation -- IfcIShapeProfileDef: https://ifc43-docs.standards.buildingsmart.org/IFC/RELEASE/IFC4x3/HTML/lexical/IfcIShapeProfileDef.htm
- IfcOpenShell GitHub Repository: https://github.com/IfcOpenShell/IfcOpenShell
