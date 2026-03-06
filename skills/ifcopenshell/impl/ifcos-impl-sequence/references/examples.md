# ifcos-impl-sequence — Working Code Examples

End-to-end examples for IFC scheduling and 4D BIM scenarios. All examples verified against IfcOpenShell documentation.

---

## Example 1: Complete Residential Construction Schedule

Creates a full construction schedule with WBS, sequencing, and date cascading.

```python
# IFC4 — Full residential construction schedule
import ifcopenshell
import ifcopenshell.api

# Bootstrap
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Residential Villa")
ifcopenshell.api.run("unit.assign_unit", model)

# Create work plan and schedule
plan = ifcopenshell.api.run("sequence.add_work_plan", model,
    name="Villa Construction Plan")
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Main Schedule",
    predefined_type="PLANNED",
    work_plan=plan)

# --- WBS Level 1: Summary tasks (NO time data) ---
phase_site = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Site Preparation",
    identification="1",
    predefined_type="CONSTRUCTION")

phase_foundation = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Foundations",
    identification="2",
    predefined_type="CONSTRUCTION")

phase_structure = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="Superstructure",
    identification="3",
    predefined_type="CONSTRUCTION")

# --- WBS Level 2: Leaf tasks (GET time data) ---
# Site preparation subtasks
task_clearing = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_site,
    name="Site Clearing",
    identification="1.1")

task_grading = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_site,
    name="Grading & Leveling",
    identification="1.2")

# Foundation subtasks
task_excavation = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_foundation,
    name="Excavation",
    identification="2.1")

task_formwork = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_foundation,
    name="Foundation Formwork",
    identification="2.2")

task_rebar = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_foundation,
    name="Reinforcement",
    identification="2.3")

task_pour = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_foundation,
    name="Concrete Pour",
    identification="2.4")

task_cure = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_foundation,
    name="Curing",
    identification="2.5")

# Structure subtasks
task_columns = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_structure,
    name="Column Construction",
    identification="3.1")

task_beams = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_structure,
    name="Beam Construction",
    identification="3.2")

task_slab = ifcopenshell.api.run("sequence.add_task", model,
    parent_task=phase_structure,
    name="Floor Slab",
    identification="3.3")

# --- Assign durations to leaf tasks ---
leaf_tasks = [
    (task_clearing, "P3D", "2026-04-01"),
    (task_grading, "P2D", None),
    (task_excavation, "P4D", None),
    (task_formwork, "P5D", None),
    (task_rebar, "P3D", None),
    (task_pour, "P1D", None),
    (task_cure, "P7D", None),
    (task_columns, "P10D", None),
    (task_beams, "P8D", None),
    (task_slab, "P5D", None),
]

for task, duration, start in leaf_tasks:
    tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)
    attrs = {"ScheduleDuration": duration}
    if start:
        attrs["ScheduleStart"] = start
    ifcopenshell.api.run("sequence.edit_task_time", model,
        task_time=tt, attributes=attrs)

# --- Define sequences ---
sequences = [
    (task_clearing, task_grading, "FINISH_START", None),
    (task_grading, task_excavation, "FINISH_START", None),
    (task_excavation, task_formwork, "FINISH_START", None),
    (task_formwork, task_rebar, "START_START", None),     # rebar starts when formwork starts
    (task_rebar, task_pour, "FINISH_START", None),
    (task_pour, task_cure, "FINISH_START", None),
    (task_cure, task_columns, "FINISH_START", "P2D"),     # 2-day lag after curing
    (task_columns, task_beams, "FINISH_START", None),
    (task_beams, task_slab, "FINISH_START", None),
]

for pred, succ, seq_type, lag in sequences:
    rel = ifcopenshell.api.run("sequence.assign_sequence", model,
        relating_process=pred,
        related_process=succ,
        sequence_type=seq_type)
    if lag:
        ifcopenshell.api.run("sequence.assign_lag_time", model,
            rel_sequence=rel,
            lag_value=lag,
            duration_type="WORKTIME")

# --- Cascade dates from first task ---
ifcopenshell.api.run("sequence.cascade_schedule", model, task=task_clearing)

# --- Calculate critical path ---
ifcopenshell.api.run("sequence.recalculate_schedule", model,
    work_schedule=schedule)

model.write("residential_schedule.ifc")
```

---

## Example 2: 4D BIM — Task-Product Linking for Visualization

Links building elements to construction tasks for 4D timeline animation.

```python
# IFC4 — 4D BIM with task-product relationships
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Office Block")
ifcopenshell.api.run("unit.assign_unit", model)

# Create spatial structure
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Main Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Block A")
storey_gf = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")
storey_1f = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="First Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey_gf, storey_1f], relating_object=building)

# Create building elements
gf_slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="GF Slab")
gf_walls = [
    ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=f"GF Wall {i}")
    for i in range(1, 5)
]
ff_slab = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSlab", name="1F Slab")

# Place elements in storeys
ifcopenshell.api.run("spatial.assign_container", model,
    products=[gf_slab] + gf_walls, relating_structure=storey_gf)
ifcopenshell.api.run("spatial.assign_container", model,
    products=[ff_slab], relating_structure=storey_1f)

# Create construction schedule
schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Construction Timeline", predefined_type="PLANNED")

task_gf_slab = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="GF Slab Construction",
    identification="A")

task_gf_walls = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="GF Wall Construction",
    identification="B")

task_ff_slab = ifcopenshell.api.run("sequence.add_task", model,
    work_schedule=schedule,
    name="1F Slab Construction",
    identification="C")

# Add time data
for task, dur, start in [
    (task_gf_slab, "P5D", "2026-06-01"),
    (task_gf_walls, "P10D", None),
    (task_ff_slab, "P5D", None),
]:
    tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)
    attrs = {"ScheduleDuration": dur}
    if start:
        attrs["ScheduleStart"] = start
    ifcopenshell.api.run("sequence.edit_task_time", model,
        task_time=tt, attributes=attrs)

# Sequence: slab → walls → upper slab
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_gf_slab, related_process=task_gf_walls)
ifcopenshell.api.run("sequence.assign_sequence", model,
    relating_process=task_gf_walls, related_process=task_ff_slab)

# --- 4D BIM: Link tasks to products ---
# GF slab is built by task_gf_slab
ifcopenshell.api.run("sequence.assign_process", model,
    relating_process=task_gf_slab,
    related_object=gf_slab)

# All GF walls are built by task_gf_walls
for wall in gf_walls:
    ifcopenshell.api.run("sequence.assign_process", model,
        relating_process=task_gf_walls,
        related_object=wall)

# 1F slab is built by task_ff_slab
ifcopenshell.api.run("sequence.assign_process", model,
    relating_process=task_ff_slab,
    related_object=ff_slab)

# Cascade and save
ifcopenshell.api.run("sequence.cascade_schedule", model, task=task_gf_slab)
model.write("4d_bim_schedule.ifc")
```

---

## Example 3: Work Calendar with Holidays

Creates a calendar with standard working hours and exception days.

```python
# IFC4 — Work calendar with holidays and custom hours
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.file(schema="IFC4")

# Create standard 5-day calendar
calendar = ifcopenshell.api.run("sequence.add_work_calendar", model,
    name="NL Standard Construction Calendar")

# Working times: Monday-Friday
work_time = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="WorkingTimes")

pattern = ifcopenshell.api.run("sequence.assign_recurrence_pattern", model,
    parent=work_time,
    recurrence_type="WEEKLY")
ifcopenshell.api.run("sequence.edit_recurrence_pattern", model,
    recurrence_pattern=pattern,
    attributes={"WeekdayComponent": [1, 2, 3, 4, 5]})  # Mon-Fri

# Morning shift: 07:00-12:00
ifcopenshell.api.run("sequence.add_time_period", model,
    recurrence_pattern=pattern,
    start_time="07:00:00",
    end_time="12:00:00")

# Afternoon shift: 13:00-16:30
ifcopenshell.api.run("sequence.add_time_period", model,
    recurrence_pattern=pattern,
    start_time="13:00:00",
    end_time="16:30:00")

# Exception: Christmas period (non-working)
christmas = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="ExceptionTimes")

# Exception: Summer Fridays (half day — working exception)
summer_fri = ifcopenshell.api.run("sequence.add_work_time", model,
    work_calendar=calendar,
    time_type="WorkingTimes")
summer_pattern = ifcopenshell.api.run("sequence.assign_recurrence_pattern", model,
    parent=summer_fri,
    recurrence_type="WEEKLY")
ifcopenshell.api.run("sequence.edit_recurrence_pattern", model,
    recurrence_pattern=summer_pattern,
    attributes={"WeekdayComponent": [5]})  # Fridays only
ifcopenshell.api.run("sequence.add_time_period", model,
    recurrence_pattern=summer_pattern,
    start_time="07:00:00",
    end_time="12:00:00")  # Half day only
```

---

## Example 4: Gantt Chart Data Extraction

Extracts complete scheduling data for rendering in external tools.

```python
# IFC4 — Extract data for Gantt chart rendering
import ifcopenshell
import ifcopenshell.util.sequence
import json

model = ifcopenshell.open("construction_schedule.ifc")

def extract_gantt_data(model):
    """Extract all scheduling data for Gantt chart rendering."""
    schedules = model.by_type("IfcWorkSchedule")
    result = []

    for schedule in schedules:
        schedule_data = {
            "name": schedule.Name,
            "type": schedule.PredefinedType,
            "tasks": [],
        }

        for task in ifcopenshell.util.sequence.get_root_tasks(schedule):
            schedule_data["tasks"].append(extract_task_tree(task))

        result.append(schedule_data)

    return result

def extract_task_tree(task, depth=0):
    """Recursively extract task data."""
    tt = task.TaskTime
    task_data = {
        "id": task.Identification or task.GlobalId,
        "name": task.Name,
        "depth": depth,
        "predefined_type": task.PredefinedType,
        "start": tt.ScheduleStart if tt else None,
        "finish": tt.ScheduleFinish if tt else None,
        "duration": tt.ScheduleDuration if tt else None,
        "completion": tt.Completion if tt else 0.0,
        "is_critical": getattr(tt, "IsCritical", None) if tt else None,
        "total_float": tt.TotalFloat if tt else None,
        "free_float": tt.FreeFloat if tt else None,
        "actual_start": tt.ActualStart if tt else None,
        "actual_finish": tt.ActualFinish if tt else None,
        "predecessors": [],
        "products": [],
        "subtasks": [],
    }

    # Extract predecessors (for dependency arrows)
    for rel in task.IsSuccessorFrom or []:
        pred = rel.RelatingProcess
        lag = None
        if rel.TimeLag:
            lag = rel.TimeLag.LagValue.wrappedValue if hasattr(rel.TimeLag.LagValue, 'wrappedValue') else str(rel.TimeLag.LagValue)
        task_data["predecessors"].append({
            "task_id": pred.Identification or pred.GlobalId,
            "type": rel.SequenceType,
            "lag": lag,
        })

    # Extract linked products (for 4D visualization)
    outputs = ifcopenshell.util.sequence.get_task_outputs(task)
    for product in outputs:
        task_data["products"].append({
            "global_id": product.GlobalId,
            "class": product.is_a(),
            "name": product.Name,
        })

    # Recurse into subtasks
    for subtask in ifcopenshell.util.sequence.get_nested_tasks(task):
        task_data["subtasks"].append(extract_task_tree(subtask, depth + 1))

    return task_data

# Extract and export
gantt_data = extract_gantt_data(model)
print(json.dumps(gantt_data, indent=2, default=str))
```

---

## Example 5: Progress Tracking with Actual Dates

Updates tasks with actual progress data.

```python
# IFC4 — Record actual progress against planned schedule
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("planned_schedule.ifc")

# Find a task by identification
tasks = model.by_type("IfcTask")
task_a1 = next(t for t in tasks if t.Identification == "1.1")

# Update with actual dates
tt = task_a1.TaskTime
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt,
    attributes={
        "ActualStart": "2026-04-02",       # Started 1 day late
        "ActualDuration": "P4D",           # Took 4 days instead of 3
        "Completion": 1.0,                  # 100% complete
        "StatusTime": "2026-04-06",        # Date of this update
    })

# Mark a task as partially complete
task_a2 = next(t for t in tasks if t.Identification == "1.2")
tt2 = task_a2.TaskTime
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=tt2,
    attributes={
        "ActualStart": "2026-04-06",
        "Completion": 0.6,                  # 60% complete
        "RemainingTime": "P2D",            # 2 days remaining
        "StatusTime": "2026-04-09",
    })

model.write("updated_schedule.ifc")
```

---

## Example 6: Baseline Comparison

Creates and compares schedule baselines.

```python
# IFC4 — Create baseline for earned value analysis
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.util.sequence

model = ifcopenshell.open("project_schedule.ifc")
schedule = model.by_type("IfcWorkSchedule")[0]

# Create a baseline snapshot
baseline = ifcopenshell.api.run("sequence.create_baseline", model,
    work_schedule=schedule,
    name="Baseline Rev 1")

# The baseline is a copy with predefined_type = BASELINE
# Compare planned vs baseline dates
baseline_tasks = {
    t.Identification: t
    for t in ifcopenshell.util.sequence.get_root_tasks(baseline)
}

for task in ifcopenshell.util.sequence.get_root_tasks(schedule):
    tid = task.Identification
    if tid in baseline_tasks:
        bt = baseline_tasks[tid]
        planned_start = task.TaskTime.ScheduleStart if task.TaskTime else "N/A"
        baseline_start = bt.TaskTime.ScheduleStart if bt.TaskTime else "N/A"
        print(f"{tid} {task.Name}: "
              f"Planned={planned_start}, Baseline={baseline_start}")
```

---

## Example 7: Per-Storey Construction Sequence (Multi-Storey 4D)

Creates a repeating schedule pattern across building storeys.

```python
# IFC4 — Per-storey schedule with automatic sequencing
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Tower Block")
ifcopenshell.api.run("unit.assign_unit", model)

schedule = ifcopenshell.api.run("sequence.add_work_schedule", model,
    name="Tower Construction", predefined_type="PLANNED")

storey_names = ["Ground Floor", "First Floor", "Second Floor", "Third Floor"]
previous_last_task = None

for i, storey_name in enumerate(storey_names):
    # Summary task per storey
    storey_task = ifcopenshell.api.run("sequence.add_task", model,
        work_schedule=schedule,
        name=f"{storey_name} Works",
        identification=str(i + 1))

    # Subtasks per storey
    activities = [
        ("Formwork", "P3D"),
        ("Reinforcement", "P2D"),
        ("Concrete Pour", "P1D"),
        ("Curing", "P7D"),
    ]

    prev_task = None
    first_task = None
    for j, (act_name, dur) in enumerate(activities):
        task = ifcopenshell.api.run("sequence.add_task", model,
            parent_task=storey_task,
            name=f"{storey_name} - {act_name}",
            identification=f"{i+1}.{j+1}")

        tt = ifcopenshell.api.run("sequence.add_task_time", model, task=task)
        ifcopenshell.api.run("sequence.edit_task_time", model,
            task_time=tt,
            attributes={"ScheduleDuration": dur})

        if first_task is None:
            first_task = task

        if prev_task:
            ifcopenshell.api.run("sequence.assign_sequence", model,
                relating_process=prev_task, related_process=task)

        prev_task = task

    # Link previous storey's last task to this storey's first task
    if previous_last_task and first_task:
        ifcopenshell.api.run("sequence.assign_sequence", model,
            relating_process=previous_last_task,
            related_process=first_task)

    previous_last_task = prev_task

# Set start date on very first leaf task and cascade
all_tasks = model.by_type("IfcTask")
first_leaf = next(t for t in all_tasks if t.Identification == "1.1")
ifcopenshell.api.run("sequence.edit_task_time", model,
    task_time=first_leaf.TaskTime,
    attributes={"ScheduleStart": "2026-05-01"})
ifcopenshell.api.run("sequence.cascade_schedule", model, task=first_leaf)

model.write("tower_schedule.ifc")
```
