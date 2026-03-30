---
name: ifcos-impl-sequence
description: >
  Use when implementing construction schedules or 4D BIM timelines in IFC -- work schedules,
  tasks, task dependencies, and Gantt chart data extraction. Prevents the common mistake of
  not linking tasks to elements (no 4D visualization possible). Covers ifcopenshell.api.sequence,
  work schedules, task time relationships, and construction sequence modeling.
  Keywords: schedule, 4D BIM, work schedule, task, Gantt, timeline, construction sequence,
  IfcWorkSchedule, IfcTask, task dependency.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IFC Scheduling and 4D BIM Implementation

## Quick Reference

### Critical Warnings

- **ALWAYS** use `ifcopenshell.api.run("sequence.*", ...)` for schedule mutations. NEVER create `IfcTask`, `IfcWorkSchedule`, or `IfcRelSequence` directly with `model.create_entity()`.
- **ALWAYS** call `add_task_time` BEFORE `edit_task_time`. The `IfcTaskTime` entity MUST exist before editing.
- **ALWAYS** use ISO 8601 strings for dates (`"2026-04-01"`) and durations (`"P5D"`) in IFC4+. NEVER pass Python `datetime` objects directly to `edit_task_time`.
- **NEVER** assign `IfcTaskTime` to parent/summary tasks. ONLY leaf tasks (no subtasks) receive time data.
- **NEVER** pass both `work_schedule` and `parent_task` to `add_task`. These are mutually exclusive.
- **NEVER** create cyclical sequence relationships. `cascade_schedule` will recurse infinitely.
- **ALWAYS** call `cascade_schedule` after modifying task durations or sequences. Dates do NOT propagate automatically.
- **ALWAYS** pass `products` as a **list** in v0.8+ relationship functions (e.g., `assign_process`).
- **NEVER** forget to create a project bootstrap (IfcProject, units, contexts) before creating schedules.

### Version Differences: IFC2X3 vs IFC4+

| Feature | IFC2X3 | IFC4 / IFC4X3 |
|---------|--------|----------------|
| Date format | `IfcDateAndTime` entity | ISO 8601 string (`"2026-04-01"`) |
| Duration format | `IfcDateAndTime` entity | ISO 8601 duration (`"P5D"`, `"P2W"`) |
| `add_date_time` returns | `IfcDateAndTime` entity | Formatted string |
| Task predefined types | Limited | `CONSTRUCTION`, `DEMOLITION`, `MAINTENANCE`, `MOVE`, `OPERATION`, `USERDEFINED`, `NOTDEFINED` |
| Schedule predefined types | Limited | `ACTUAL`, `BASELINE`, `PLANNED`, `USERDEFINED`, `NOTDEFINED` |
| `IfcWorkPlan` | Not available | Available — groups related schedules |
| Lag time support | Basic | Full ISO 8601 duration with duration types |
| Recurrence patterns | Limited | Full `IfcRecurrencePattern` support |

**ALWAYS** check `model.schema` before writing scheduling code. Use the IFC4+ patterns unless targeting legacy IFC2X3 files.

### Decision Tree: What Do You Need?

```
What scheduling operation do you need?
├── Create a construction schedule from scratch?
│   └── Follow: Full Schedule Bootstrap (Pattern 1)
│
├── Add tasks to an existing schedule?
│   ├── Top-level task?
│   │   └── add_task(work_schedule=schedule, ...)
│   └── Subtask of existing task?
│       └── add_task(parent_task=parent, ...)
│
├── Set task dates and durations?
│   ├── Task has no IfcTaskTime yet?
│   │   └── add_task_time(task=task) → then edit_task_time(...)
│   └── Task already has IfcTaskTime?
│       └── edit_task_time(task_time=task.TaskTime, ...)
│
├── Define task dependencies?
│   ├── Simple finish-to-start?
│   │   └── assign_sequence(sequence_type="FINISH_START")
│   ├── With lag/delay between tasks?
│   │   └── assign_sequence → assign_lag_time
│   └── Other dependency types?
│       └── assign_sequence with FINISH_FINISH, START_START, or START_FINISH
│
├── Create a work calendar (working days/hours)?
│   └── Follow: Calendar Setup (Pattern 3)
│
├── Link tasks to building products (4D BIM)?
│   ├── Product is INPUT to task (consumed/used)?
│   │   └── assign_process(relating_process=task, related_object=product)
│   └── Product is OUTPUT of task (created)?
│       └── assign_product(relating_product=product, related_object=task)
│
├── Propagate dates through task network?
│   └── cascade_schedule(task=starting_task)
│
├── Calculate critical path and floats?
│   └── recalculate_schedule(work_schedule=schedule)
│
├── Extract Gantt chart data?
│   └── Follow: Gantt Data Extraction (Pattern 5)
│
└── Compare schedule baselines?
    └── create_baseline → compare schedules
```

---

## Essential Patterns

### Pattern 1: Full Schedule Bootstrap

```python
# IFC4: Complete construction schedule from scratch
import ifcopenshell
import ifcopenshell.api

# Step 1: Create IFC file with project (REQUIRED before any scheduling)
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

# Step 2: Create work schedule (directly, or within a work plan)
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Construction Schedule",
    predefined_type="PLANNED")

# Step 3: Create tasks
task_a = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Foundations",
    identification="A")

task_b = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Superstructure",
    identification="B")

# Step 4: Add time data to tasks
time_a = ifcopenshell.api.run("sequence.add_task_time", model, task=task_a)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=time_a,
    attributes={
        "ScheduleStart": "2026-04-01",
        "ScheduleDuration": "P10D",
    })

time_b = ifcopenshell.api.run("sequence.add_task_time", model, task=task_b)
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=time_b,
    attributes={"ScheduleDuration": "P20D"})

# Step 5: Create dependency
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_a,
    related_process=task_b,
    sequence_type="FINISH_START")

# Step 6: Cascade dates (task_b start computed from task_a finish)
ifcopenshell.api.run("sequence.cascade_schedule", model, task=task_a)

model.write("schedule.ifc")
```

### Pattern 2: Hierarchical Work Breakdown Structure

```python
# IFC4: WBS with parent/child task nesting
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Phase 1", predefined_type="PLANNED")

# Root task (summary: NO time data)
phase_struct = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Structural Works",
    identification="1")

# Subtasks (leaf tasks: GET time data)
formwork = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_struct,
    name="Formwork",
    identification="1.1")

rebar = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_struct,
    name="Reinforcement",
    identification="1.2")

pour = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_struct,
    name="Concrete Pour",
    identification="1.3")

# Add time data ONLY to leaf tasks
for task, dur in [(formwork, "P5D"), (rebar, "P3D"), (pour, "P1D")]:
    tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)
    ifcopenshell.api.run("sequence.edit_task_time", model,
        task_time=tt,
        attributes={"ScheduleDuration": dur})

# Sequence the leaf tasks
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=formwork, related_process=rebar)
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=rebar, related_process=pour)
```

### Pattern 3: Work Calendar Setup

```python
# IFC4: Standard 5-day work week with holidays
calendar = ifcopenshell.api.run("sequence.add_work_calendar", model,
    name="Standard 5-Day Week")

# Define working times (Monday-Friday)
work_time = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="WorkingTimes")

pattern = ifcopenshell.api.run("sequence.assign_recurrence_pattern", model,
    parent=work_time,
    recurrence_type="WEEKLY")
ifcopenshell.api.run("sequence.edit_recurrence_pattern", model,
    recurrence_pattern=pattern,
    attributes={"WeekdayComponent": [1, 2, 3, 4, 5]})  # Mon=1..Fri=5

# Add working hours
ifcopenshell.api.run("sequence.add_time_period", model,
    recurrence_pattern=pattern,
    start_time="08:00:00",
    end_time="17:00:00")

# Add exception (holiday)
exception = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="ExceptionTimes")
```

### Pattern 4: 4D BIM: Linking Tasks to Products

```python
# IFC4: Connect tasks to building elements for 4D visualization
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Exterior Wall A")
slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="Ground Floor Slab")

# Link products as outputs of tasks (IfcRelAssignsToProcess)
ifcopenshell.api.run("sequence.assign_process", model,
    relating_process=pour,
    related_object=slab)
ifcopenshell.api.run("sequence.assign_process", model,
    relating_process=formwork,
    related_object=wall)

# Query: which products does a task build?
# Use ifcopenshell.util.sequence.get_task_outputs(task)
```

### Pattern 5: Gantt Chart Data Extraction

```python
# IFC4: Extract scheduling data for Gantt chart rendering
import ifcopenshell
import ifcopenshell.util.sequence

model = ifcopenshell.open("schedule.ifc")
schedules = model.by_type("IfcWorkSchedule")

for schedule in schedules:
    print(f"Schedule: {schedule.Name}")
    for task in ifcopenshell.util.sequence.get_root_tasks(schedule):
        extract_task_data(task, indent=0)

def extract_task_data(task, indent=0):
    """Recursively extract task data for Gantt rendering."""
    prefix = "  " * indent
    task_time = task.TaskTime

    row = {
        "id": task.Identification,
        "name": task.Name,
        "start": task_time.ScheduleStart if task_time else None,
        "finish": task_time.ScheduleFinish if task_time else None,
        "duration": task_time.ScheduleDuration if task_time else None,
        "completion": task_time.Completion if task_time else None,
        "is_critical": task_time.IsCritical if task_time else None,
    }
    print(f"{prefix}{row['id']}: {row['name']} "
          f"({row['start']} → {row['finish']}, {row['duration']})")

    # Get predecessors for dependency arrows
    for rel in task.IsSuccessorFrom or []:
        pred = rel.RelatingProcess
        print(f"{prefix}  ← depends on: {pred.Identification} "
              f"({rel.SequenceType})")

    # Recurse into subtasks
    for subtask in ifcopenshell.util.sequence.get_nested_tasks(task):
        extract_task_data(subtask, indent + 1)
```

### Pattern 6: Critical Path Analysis

```python
# IFC4: Calculate critical path and floats
ifcopenshell.api.run("sequence.recalculate_schedule", model,
    work_schedule=schedule)

# After recalculation, check task floats
for task in model.by_type("IfcTask"):
    tt = task.TaskTime
    if tt:
        is_critical = (tt.TotalFloat == "P0D") if tt.TotalFloat else False
        print(f"{task.Name}: TotalFloat={tt.TotalFloat}, "
              f"FreeFloat={tt.FreeFloat}, Critical={is_critical}")
```

---

## Common Operations

### Add Lag Time Between Tasks

```python
# IFC4: Add 2-day lag after concrete pour (curing time)
seq = ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=pour_task,
    related_process=next_task,
    sequence_type="FINISH_START")

ifcopenshell.api.run("sequence.assign_lag_time", model,
    rel_sequence=seq,
    lag_value="P2D",
    duration_type="WORKTIME")
```

### Modify Existing Task

```python
# IFC4: Update task attributes
ifcopenshell.api.run("sequence.edit_task", model,
    task=task,
    attributes={
        "Name": "Updated Task Name",
        "Identification": "B.1",
        "Description": "Revised scope",
        "PredefinedType": "CONSTRUCTION",
    })
```

### Remove Task (Cascading Delete)

```python
# IFC4: Removes task, all subtasks, and all relationships
ifcopenshell.api.run("sequence.remove_task", model, task=task)
```

### Work Plan with Multiple Schedules

```python
# IFC4: Group schedules under a work plan
plan = ifcopenshell.api.run("sequence.add_work_plan", model,
    name="Master Construction Plan")

planned = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Planned Schedule",
    predefined_type="PLANNED",
    work_plan=plan)

actual = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Actual Schedule",
    predefined_type="ACTUAL",
    work_plan=plan)
```

### Query Schedule Data with Utilities

```python
import ifcopenshell.util.sequence as seq_util

# Navigate task hierarchy
root_tasks = seq_util.get_root_tasks(schedule)
children = seq_util.get_nested_tasks(parent_task)
all_tasks = list(seq_util.get_all_nested_tasks(parent_task))

# Find task context
parent = seq_util.get_parent_task(task)
owning_schedule = seq_util.get_task_work_schedule(task)

# 4D BIM queries
products = seq_util.get_task_outputs(task)
inputs = seq_util.get_task_inputs(task)
resources = seq_util.get_task_resources(task)
tasks_for_wall = seq_util.get_tasks_for_product(wall, schedule)

# Calendar queries
is_work = seq_util.is_working_day(date, calendar)
work_days = seq_util.count_working_days(start, finish, calendar)
new_date = seq_util.offset_date(start, duration, "WORKTIME", calendar)

# Schedule overview
start, end = seq_util.guess_date_range(schedule)
```

---

## IfcTaskTime Attribute Reference

| Attribute | Type (IFC4+) | Description |
|-----------|-------------|-------------|
| `ScheduleStart` | ISO date string | Planned start date |
| `ScheduleFinish` | ISO date string | Planned finish date (auto-calculated if duration set) |
| `ScheduleDuration` | ISO duration | Planned duration (`"P5D"`, `"P2W"`, `"P1M"`) |
| `ActualStart` | ISO date string | Actual start date |
| `ActualFinish` | ISO date string | Actual finish date |
| `ActualDuration` | ISO duration | Actual duration |
| `EarlyStart` | ISO date string | CPM forward pass early start |
| `EarlyFinish` | ISO date string | CPM forward pass early finish |
| `LateStart` | ISO date string | CPM backward pass late start |
| `LateFinish` | ISO date string | CPM backward pass late finish |
| `FreeFloat` | ISO duration | Free float (populated by `recalculate_schedule`) |
| `TotalFloat` | ISO duration | Total float (populated by `recalculate_schedule`) |
| `IsCritical` | bool | On critical path (populated by `recalculate_schedule`) |
| `Completion` | float | Percentage complete (0.0 to 1.0) |
| `DurationType` | enum | `ELAPSEDDAYS`, `WORKTIME`, `CALENDARTIME` |
| `StatusTime` | ISO date string | Date of last status update |
| `RemainingTime` | ISO duration | Remaining duration |

---

## Sequence Types Reference

| Type | Constant | Description | Construction Example |
|------|----------|-------------|---------------------|
| Finish-to-Start | `FINISH_START` | Predecessor MUST finish before successor starts | Pour concrete → cure → strip formwork |
| Finish-to-Finish | `FINISH_FINISH` | Predecessor finish constrains successor finish | Painting must finish when inspection finishes |
| Start-to-Start | `START_START` | Predecessor start constrains successor start | Excavation starts → hauling starts simultaneously |
| Start-to-Finish | `START_FINISH` | Predecessor start constrains successor finish | Rare — used for just-in-time delivery |

**ALWAYS** default to `FINISH_START` unless there is a specific reason for another type. It represents 90%+ of construction dependencies.

---

## Dependencies

- **ifcos-syntax-api** — API invocation patterns (`api.run()` vs direct calls), module table, parameter conventions
- **ifcos-syntax-fileio** — File I/O (`ifcopenshell.open()`, `model.write()`, transactions)

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all 40 sequence functions
- [Working Code Examples](references/examples.md) — End-to-end scheduling scenarios
- [Anti-Patterns](references/anti-patterns.md) — Common scheduling mistakes and corrections
