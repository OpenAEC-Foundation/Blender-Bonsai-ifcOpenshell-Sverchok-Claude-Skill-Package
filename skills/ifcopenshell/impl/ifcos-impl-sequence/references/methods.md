# ifcos-impl-sequence — API Method Signatures

Complete signatures for all `ifcopenshell.api.sequence` functions verified against the official IfcOpenShell documentation.

**Source:** https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/sequence/

---

## Work Plan Functions

### add_work_plan

```python
ifcopenshell.api.run("sequence.add_work_plan", file,
    name: str | None = None,
    predefined_type: str = "NOTDEFINED",
    start_time: str | datetime.time | None = None
) -> IfcWorkPlan
```

Creates a top-level work plan that groups related work schedules (e.g., for baseline comparison).

### edit_work_plan

```python
ifcopenshell.api.run("sequence.edit_work_plan", file,
    work_plan: IfcWorkPlan,
    attributes: dict[str, Any]
) -> None
```

Modifies attributes of an `IfcWorkPlan`.

### remove_work_plan

```python
ifcopenshell.api.run("sequence.remove_work_plan", file,
    work_plan: IfcWorkPlan
) -> None
```

Deletes a work plan and disassociates its schedules.

### assign_work_plan

```python
ifcopenshell.api.run("sequence.assign_work_plan", file,
    work_schedule: IfcWorkSchedule,
    work_plan: IfcWorkPlan
) -> IfcRelAggregates
```

Associates a work schedule with a work plan.

---

## Work Schedule Functions

### add_work_schedule

```python
ifcopenshell.api.run("sequence.add_work_schedule", file,
    name: str = "Unnamed",
    predefined_type: str = "NOTDEFINED",
    object_type: str | None = None,
    start_time: str | datetime.time | None = None,
    work_plan: IfcWorkPlan | None = None
) -> IfcWorkSchedule
```

Creates a work schedule. If `work_plan` is provided, the schedule is automatically associated.

**Predefined types (IFC4+):** `ACTUAL`, `BASELINE`, `PLANNED`, `USERDEFINED`, `NOTDEFINED`

### edit_work_schedule

```python
ifcopenshell.api.run("sequence.edit_work_schedule", file,
    work_schedule: IfcWorkSchedule,
    attributes: dict[str, Any]
) -> None
```

Modifies attributes of an `IfcWorkSchedule`.

### remove_work_schedule

```python
ifcopenshell.api.run("sequence.remove_work_schedule", file,
    work_schedule: IfcWorkSchedule
) -> None
```

Deletes a work schedule and all contained tasks.

### copy_work_schedule

```python
ifcopenshell.api.run("sequence.copy_work_schedule", file,
    work_schedule: IfcWorkSchedule
) -> IfcWorkSchedule
```

Duplicates an entire work schedule with all tasks and relationships.

### create_baseline

```python
ifcopenshell.api.run("sequence.create_baseline", file,
    work_schedule: IfcWorkSchedule,
    name: str | None = None
) -> IfcWorkSchedule
```

Creates a baseline copy of a schedule for progress comparison.

---

## Task Functions

### add_task

```python
ifcopenshell.api.run("sequence.add_task", file,
    work_schedule: IfcWorkSchedule | None = None,
    parent_task: IfcTask | None = None,
    name: str | None = None,
    description: str | None = None,
    identification: str | None = None,
    predefined_type: str = "NOTDEFINED"
) -> IfcTask
```

Creates a task. **EXACTLY ONE** of `work_schedule` or `parent_task` MUST be provided:
- `work_schedule`: Creates a root-level task in the schedule
- `parent_task`: Creates a subtask nested under the parent

**Predefined types (IFC4+):** `CONSTRUCTION`, `DEMOLITION`, `MAINTENANCE`, `MOVE`, `OPERATION`, `USERDEFINED`, `NOTDEFINED`

### edit_task

```python
ifcopenshell.api.run("sequence.edit_task", file,
    task: IfcTask,
    attributes: dict[str, Any]
) -> None
```

Modifies task attributes (Name, Identification, Description, PredefinedType, etc.).

### remove_task

```python
ifcopenshell.api.run("sequence.remove_task", file,
    task: IfcTask
) -> None
```

Deletes a task, **all subtasks recursively**, and all associated relationships (sequences, controls).

### duplicate_task

```python
ifcopenshell.api.run("sequence.duplicate_task", file,
    task: IfcTask
) -> IfcTask
```

Replicates a task with its properties and time data.

---

## Task Time Functions

### add_task_time

```python
ifcopenshell.api.run("sequence.add_task_time", file,
    task: IfcTask,
    is_recurring: bool = False
) -> IfcTaskTime | IfcTaskTimeRecurring
```

Attaches temporal data to a task. Set `is_recurring=True` for repeating tasks (creates `IfcTaskTimeRecurring`).

**ALWAYS** call this BEFORE `edit_task_time`.

### edit_task_time

```python
ifcopenshell.api.run("sequence.edit_task_time", file,
    task_time: IfcTaskTime,
    attributes: dict[str, Any]
) -> None
```

Modifies temporal properties. Internally auto-calculates:
- If `ScheduleStart` + `ScheduleDuration` → computes `ScheduleFinish`
- If `ScheduleStart` + `ScheduleFinish` → computes `ScheduleDuration`

**Key attributes:**

| Attribute | Type | Example |
|-----------|------|---------|
| `ScheduleStart` | ISO date | `"2026-04-01"` |
| `ScheduleFinish` | ISO date | `"2026-04-15"` |
| `ScheduleDuration` | ISO duration | `"P10D"` |
| `ActualStart` | ISO date | `"2026-04-02"` |
| `ActualFinish` | ISO date | `"2026-04-14"` |
| `ActualDuration` | ISO duration | `"P9D"` |
| `Completion` | float | `0.75` |
| `DurationType` | enum | `"WORKTIME"` |
| `RemainingTime` | ISO duration | `"P3D"` |

### add_date_time

```python
ifcopenshell.api.run("sequence.add_date_time", file,
    dt: datetime.datetime
) -> str | IfcDateAndTime
```

Converts a Python `datetime` to IFC format:
- **IFC4+**: Returns ISO 8601 string
- **IFC2X3**: Returns `IfcDateAndTime` entity

---

## Sequence Functions

### assign_sequence

```python
ifcopenshell.api.run("sequence.assign_sequence", file,
    relating_process: IfcTask,
    related_process: IfcTask,
    sequence_type: str = "FINISH_START"
) -> IfcRelSequence
```

Creates a predecessor-successor dependency. `relating_process` is the predecessor; `related_process` is the successor.

**Sequence types:** `FINISH_START`, `FINISH_FINISH`, `START_START`, `START_FINISH`

### edit_sequence

```python
ifcopenshell.api.run("sequence.edit_sequence", file,
    rel_sequence: IfcRelSequence,
    attributes: dict[str, Any]
) -> None
```

Modifies a sequence relationship (e.g., change `SequenceType`).

### unassign_sequence

```python
ifcopenshell.api.run("sequence.unassign_sequence", file,
    rel_sequence: IfcRelSequence
) -> None
```

Removes a sequence relationship between two tasks.

### assign_lag_time

```python
ifcopenshell.api.run("sequence.assign_lag_time", file,
    rel_sequence: IfcRelSequence,
    lag_value: str,
    duration_type: str = "WORKTIME"
) -> IfcLagTime
```

Adds a delay between sequenced tasks. `lag_value` uses ISO 8601 duration format.

**Duration types:** `ELAPSEDDAYS`, `WORKTIME`, `CALENDARTIME`

### edit_lag_time

```python
ifcopenshell.api.run("sequence.edit_lag_time", file,
    lag_time: IfcLagTime,
    attributes: dict[str, Any]
) -> None
```

Modifies lag time attributes.

### unassign_lag_time

```python
ifcopenshell.api.run("sequence.unassign_lag_time", file,
    rel_sequence: IfcRelSequence
) -> None
```

Removes lag time from a sequence relationship.

---

## Schedule Calculation Functions

### cascade_schedule

```python
ifcopenshell.api.run("sequence.cascade_schedule", file,
    task: IfcTask
) -> None
```

Propagates dates forward from a starting task through its successors. Computes `ScheduleFinish` from `ScheduleStart` + `ScheduleDuration`, then sets successor `ScheduleStart` based on sequence type and lag.

**IMPORTANT:** Does NOT run backward pass. Use `recalculate_schedule` for full CPM analysis.

### recalculate_schedule

```python
ifcopenshell.api.run("sequence.recalculate_schedule", file,
    work_schedule: IfcWorkSchedule
) -> None
```

Performs full Critical Path Method (CPM) analysis:
1. Forward pass → computes `EarlyStart`, `EarlyFinish`
2. Backward pass → computes `LateStart`, `LateFinish`
3. Floats → computes `TotalFloat`, `FreeFloat`
4. Critical tasks → marks tasks with zero float as critical (`IsCritical`)

Cyclical relationships trigger a `RecursionError`.

### calculate_task_duration

```python
ifcopenshell.api.run("sequence.calculate_task_duration", file,
    task: IfcTask
) -> None
```

Computes duration from task constraints and calendar.

---

## Process and Product Assignment Functions

### assign_process

```python
ifcopenshell.api.run("sequence.assign_process", file,
    relating_process: IfcTask,
    related_object: IfcProduct | IfcResource
) -> IfcRelAssignsToProcess
```

Links an object (product or resource) as an input to a task. Creates an `IfcRelAssignsToProcess` relationship. Use for 4D BIM task-to-product linking.

### unassign_process

```python
ifcopenshell.api.run("sequence.unassign_process", file,
    relating_process: IfcTask,
    related_object: IfcProduct | IfcResource
) -> None
```

Removes a process assignment.

### assign_product

```python
ifcopenshell.api.run("sequence.assign_product", file,
    relating_product: IfcProduct,
    related_object: IfcTask
) -> IfcRelAssignsToProduct
```

Designates a product as the output of a task. Creates an `IfcRelAssignsToProduct` relationship.

### unassign_product

```python
ifcopenshell.api.run("sequence.unassign_product", file,
    relating_product: IfcProduct,
    related_object: IfcTask
) -> None
```

Removes a product assignment.

---

## Work Calendar Functions

### add_work_calendar

```python
ifcopenshell.api.run("sequence.add_work_calendar", file,
    name: str = "Unnamed",
    predefined_type: str = "NOTDEFINED"
) -> IfcWorkCalendar
```

Creates a work calendar defining working days and hours.

### edit_work_calendar

```python
ifcopenshell.api.run("sequence.edit_work_calendar", file,
    work_calendar: IfcWorkCalendar,
    attributes: dict[str, Any]
) -> None
```

Modifies calendar attributes.

### remove_work_calendar

```python
ifcopenshell.api.run("sequence.remove_work_calendar", file,
    work_calendar: IfcWorkCalendar
) -> None
```

Deletes a work calendar.

### add_work_time

```python
ifcopenshell.api.run("sequence.add_work_time", file,
    work_calendar: IfcWorkCalendar,
    time_type: Literal["WorkingTimes", "ExceptionTimes"] = "WorkingTimes"
) -> IfcWorkTime
```

Adds a working time or exception time definition to a calendar.

### edit_work_time

```python
ifcopenshell.api.run("sequence.edit_work_time", file,
    work_time: IfcWorkTime,
    attributes: dict[str, Any]
) -> None
```

Modifies work time attributes.

### remove_work_time

```python
ifcopenshell.api.run("sequence.remove_work_time", file,
    work_time: IfcWorkTime
) -> None
```

Removes a work time from a calendar.

---

## Recurrence Pattern Functions

### assign_recurrence_pattern

```python
ifcopenshell.api.run("sequence.assign_recurrence_pattern", file,
    parent: IfcWorkTime | IfcTaskTimeRecurring,
    recurrence_type: str = "WEEKLY"
) -> IfcRecurrencePattern
```

Sets a recurring interval on a work time or recurring task time.

**Recurrence types:** `DAILY`, `WEEKLY`, `MONTHLY_BY_DAY_OF_MONTH`, `MONTHLY_BY_POSITION`, `BY_DAY_COUNT`, `BY_WEEKDAY_COUNT`, `YEARLY_BY_DAY_OF_MONTH`, `YEARLY_BY_POSITION`

### edit_recurrence_pattern

```python
ifcopenshell.api.run("sequence.edit_recurrence_pattern", file,
    recurrence_pattern: IfcRecurrencePattern,
    attributes: dict[str, Any]
) -> None
```

Modifies recurrence pattern attributes. Key attributes:
- `WeekdayComponent`: list of ints (1=Mon through 7=Sun)
- `DayComponent`: list of ints (day of month)
- `MonthComponent`: list of ints (month number)
- `Interval`: int (repeat every N periods)
- `Occurrences`: int (max occurrences)

### unassign_recurrence_pattern

```python
ifcopenshell.api.run("sequence.unassign_recurrence_pattern", file,
    recurrence_pattern: IfcRecurrencePattern
) -> None
```

Removes a recurrence pattern.

---

## Time Period Functions

### add_time_period

```python
ifcopenshell.api.run("sequence.add_time_period", file,
    recurrence_pattern: IfcRecurrencePattern,
    start_time: str | datetime.time | None = None,
    end_time: str | datetime.time | None = None
) -> IfcTimePeriod
```

Defines a time window within a recurrence pattern (e.g., working hours 08:00-17:00).

### remove_time_period

```python
ifcopenshell.api.run("sequence.remove_time_period", file,
    time_period: IfcTimePeriod
) -> None
```

Removes a time period from a recurrence pattern.

---

## Utility Functions (ifcopenshell.util.sequence)

These are read-only query functions. Import with `import ifcopenshell.util.sequence`.

| Function | Signature | Returns |
|----------|-----------|---------|
| `get_root_tasks` | `(work_schedule)` | `list[IfcTask]` |
| `get_nested_tasks` | `(task)` | `list[IfcTask]` |
| `get_all_nested_tasks` | `(task)` | `Generator[IfcTask]` |
| `get_work_schedule_tasks` | `(work_schedule)` | `Generator[IfcTask]` |
| `get_parent_task` | `(task)` | `IfcTask | None` |
| `get_task_work_schedule` | `(task)` | `IfcWorkSchedule | None` |
| `get_calendar` | `(task)` | `IfcWorkCalendar | None` |
| `derive_calendar` | `(task)` | `IfcWorkCalendar | None` |
| `get_task_inputs` | `(task, is_recursive=False)` | `list` |
| `get_task_outputs` | `(task, is_recursive=False)` | `list` |
| `get_task_resources` | `(task, is_recursive=False)` | `list` |
| `get_tasks_for_product` | `(product, schedule=None)` | `list[IfcTask]` |
| `get_related_products` | `(relating_product, related_object)` | `list` |
| `is_working_day` | `(day, calendar)` | `bool` |
| `count_working_days` | `(start, finish, calendar)` | `int` |
| `offset_date` | `(start, duration, duration_type, calendar)` | `datetime` |
| `guess_date_range` | `(work_schedule)` | `tuple[datetime, datetime]` |
