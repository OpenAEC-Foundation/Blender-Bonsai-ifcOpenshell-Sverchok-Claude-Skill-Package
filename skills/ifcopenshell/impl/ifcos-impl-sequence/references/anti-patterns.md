# ifcos-impl-sequence — Anti-Patterns

Common mistakes when implementing IFC scheduling with `ifcopenshell.api.sequence`. Each anti-pattern includes the wrong code, the correct approach, and the reason.

---

## AP-01: Creating Tasks with model.create_entity()

### WRONG

```python
# NEVER create scheduling entities directly
task = model.create_entity("IfcTask", Name="Foundation Work")
schedule.IsDecomposedBy[0].RelatedObjects = (task,)
```

### CORRECT

```python
# ALWAYS use the API — it creates required relationships automatically
task = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Foundation Work")
```

### Why

`add_task` automatically creates the `IfcRelNests` (for subtasks) or `IfcRelAssignsToControl` (for root tasks) relationship. Direct entity creation leaves tasks orphaned with no connection to the schedule, producing an invalid IFC file.

---

## AP-02: Passing Both work_schedule and parent_task to add_task

### WRONG

```python
# NEVER provide both — they are mutually exclusive
task = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    parent_task=parent,
    name="Subtask")
```

### CORRECT

```python
# Root task: use work_schedule
root_task = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Phase 1")

# Subtask: use parent_task
sub_task = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=root_task,
    name="Subtask A")
```

### Why

`work_schedule` creates a root-level task controlled by the schedule (via `IfcRelAssignsToControl`). `parent_task` creates a nested subtask (via `IfcRelNests`). Providing both creates conflicting relationships.

---

## AP-03: Assigning Time Data to Summary/Parent Tasks

### WRONG

```python
# NEVER add IfcTaskTime to parent tasks
parent = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Structural Works")
child = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=parent, name="Formwork")

# BAD: Adding time to parent task
tt = ifcopenshell.api.run("sequence.add_task_time", model, task=parent)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={"ScheduleStart": "2026-04-01", "ScheduleDuration": "P20D"})
```

### CORRECT

```python
# ONLY add time data to LEAF tasks (no subtasks)
parent = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Structural Works")  # Summary — no time

child = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=parent, name="Formwork")
tt = ifcopenshell.api.run("sequence.add_task_time", model, task=child)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={"ScheduleStart": "2026-04-01", "ScheduleDuration": "P5D"})
```

### Why

In standard scheduling practice (and IFC convention), parent tasks derive their dates from child task rollup. Assigning explicit dates to parent tasks creates conflicts when `cascade_schedule` or `recalculate_schedule` runs. Summary task dates should be inferred, not set.

---

## AP-04: Editing Task Time Before Creating It

### WRONG

```python
task = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Excavation")

# BAD: task.TaskTime is None — this will crash
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=task.TaskTime,  # None!
    attributes={"ScheduleStart": "2026-04-01"})
```

### CORRECT

```python
task = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule, name="Excavation")

# Step 1: Create the IfcTaskTime entity
tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)

# Step 2: Now edit it
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={"ScheduleStart": "2026-04-01", "ScheduleDuration": "P4D"})
```

### Why

`add_task` creates an `IfcTask` with `TaskTime = None`. The `IfcTaskTime` entity must be explicitly created with `add_task_time` before any time attributes can be set.

---

## AP-05: Using Python datetime Objects Instead of ISO 8601 Strings

### WRONG (IFC4+)

```python
from datetime import datetime, timedelta

tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)

# BAD: Python datetime objects cause errors in IFC4+
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={
        "ScheduleStart": datetime(2026, 4, 1),      # Wrong type!
        "ScheduleDuration": timedelta(days=5),        # Wrong type!
    })
```

### CORRECT (IFC4+)

```python
tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)

# ALWAYS use ISO 8601 strings for IFC4+
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={
        "ScheduleStart": "2026-04-01",     # ISO 8601 date
        "ScheduleDuration": "P5D",          # ISO 8601 duration
    })
```

### Why

IFC4+ uses ISO 8601 string representations for dates and durations (`IfcDate`, `IfcDuration`). Python `datetime` and `timedelta` objects are not automatically converted and will raise type errors. Use `add_date_time` to convert if needed.

---

## AP-06: Creating Cyclical Task Dependencies

### WRONG

```python
# NEVER create circular dependencies
seq1 = ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_a, related_process=task_b)
seq2 = ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_b, related_process=task_c)
seq3 = ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_c, related_process=task_a)  # Cycle!

# This WILL crash:
ifcopenshell.api.run("sequence.cascade_schedule", model, task=task_a)
# RecursionError: maximum recursion depth exceeded
```

### CORRECT

```python
# ALWAYS ensure dependency graph is a DAG (directed acyclic graph)
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_a, related_process=task_b)
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_b, related_process=task_c)
# task_c has no successor back to task_a — no cycle
```

### Why

Both `cascade_schedule` and `recalculate_schedule` traverse the task dependency graph recursively. Cycles cause infinite recursion leading to `RecursionError`. Always verify your dependency network is acyclic before cascading.

---

## AP-07: Forgetting to Cascade After Schedule Changes

### WRONG

```python
# Create tasks and sequences...
tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task_a)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={"ScheduleStart": "2026-04-01", "ScheduleDuration": "P5D"})

ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_a, related_process=task_b)

# BAD: task_b.TaskTime.ScheduleStart is still None!
# Dates do NOT propagate automatically
```

### CORRECT

```python
# After setting up tasks, times, and sequences:
ifcopenshell.api.run("sequence.cascade_schedule", model, task=task_a)

# NOW task_b.TaskTime.ScheduleStart is computed from task_a's finish
```

### Why

IfcOpenShell does NOT automatically propagate dates when tasks or sequences change. You MUST explicitly call `cascade_schedule` to forward-propagate dates through the dependency network.

---

## AP-08: Using Wrong Sequence Direction

### WRONG

```python
# Confusing predecessor/successor direction
# Intent: "Formwork must finish before Rebar starts"
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_rebar,      # This is the PREDECESSOR
    related_process=task_formwork)     # This is the SUCCESSOR
# Result: Rebar must finish before Formwork — BACKWARDS!
```

### CORRECT

```python
# relating_process = PREDECESSOR (must happen first)
# related_process = SUCCESSOR (happens after)
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_formwork,    # Predecessor
    related_process=task_rebar)        # Successor
```

### Why

The naming follows IFC convention: `RelatingProcess` is the predecessor (the process that relates TO the dependency), `RelatedProcess` is the successor (the process that IS related). Think: "relating causes related."

---

## AP-09: Missing Project Bootstrap for Standalone Schedule Files

### WRONG

```python
# NEVER create schedules without a valid IFC project
model = ifcopenshell.file(schema="IFC4")

# BAD: No IfcProject, no units — invalid file
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="My Schedule")
model.write("schedule.ifc")  # Invalid IFC: no IfcProject
```

### CORRECT

```python
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# NOW create schedule — file has valid project context
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="My Schedule")
model.write("schedule.ifc")
```

### Why

Every valid IFC file requires an `IfcProject` with assigned units. `IfcWorkSchedule` is linked to the project via `IfcRelDeclares`. Without this structure, IFC validators will reject the file.

---

## AP-10: Confusing assign_process and assign_product

### WRONG

```python
# Intent: Wall is BUILT BY the construction task
# BAD: Using assign_product with wrong direction
ifcopenshell.api.run("sequence.assign_product", model,
    relating_product=wall,
    related_object=task)
# This says: "wall controls task" — semantically wrong for 4D BIM
```

### CORRECT

```python
# For 4D BIM: "task constructs/modifies product"
# Use assign_process to link products as inputs/outputs of tasks
ifcopenshell.api.run("sequence.assign_process", model,
    relating_process=task,
    related_object=wall)
# Creates IfcRelAssignsToProcess — the standard 4D BIM pattern
```

### Why

For 4D BIM visualization, the standard relationship is `IfcRelAssignsToProcess` (task operates on product). `assign_product` creates `IfcRelAssignsToProduct` which has a different semantic meaning (product is the output/deliverable). Most 4D tools expect `IfcRelAssignsToProcess`.

---

## AP-11: Not Checking Schema Before Date Format

### WRONG

```python
# Assuming IFC4+ date format without checking schema
model = ifcopenshell.open("legacy_model.ifc")  # Could be IFC2X3!

tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={"ScheduleStart": "2026-04-01"})  # Fails on IFC2X3!
```

### CORRECT

```python
model = ifcopenshell.open("model.ifc")

# ALWAYS check schema for date handling
if model.schema == "IFC2X3":
    # IFC2X3 uses IfcDateAndTime entities
    import datetime
    dt = datetime.datetime(2026, 4, 1)
    date_value = ifcopenshell.api.run("sequence.add_date_time", model, dt=dt)
    # Returns an IfcDateAndTime entity
else:
    # IFC4+ uses ISO 8601 strings
    date_value = "2026-04-01"
```

### Why

IFC2X3 represents dates as `IfcDateAndTime` entities (complex structured data). IFC4+ uses simple ISO 8601 strings. The `add_date_time` helper handles the conversion, but you must know which schema you're targeting.

---

## AP-12: Editing Schedule Attributes Directly on Entity

### WRONG

```python
# NEVER modify entity attributes directly
task.Name = "Updated Name"
task.Identification = "B.1"
task.TaskTime.ScheduleStart = "2026-05-01"
```

### CORRECT

```python
# ALWAYS use the API functions
ifcopenshell.api.run("sequence.edit_task", model,
    task=task,
    attributes={"Name": "Updated Name", "Identification": "B.1"})

ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=task.TaskTime,
    attributes={"ScheduleStart": "2026-05-01"})
```

### Why

Direct attribute modification bypasses ownership tracking, transaction management, and internal consistency logic (e.g., `edit_task_time` auto-calculates finish dates from start + duration). Always use the API for mutations.
