# Bonsai Clash Detection — Working Code Examples

> **Version**: Bonsai v0.8.x | IfcClash (IfcOpenShell v0.8.x) | Blender 4.2.0+ | Python 3.11

---

## Example 1: Basic Structural vs MEP Clash Detection (Bonsai)

Detects intersections between structural elements and MEP (mechanical/electrical/plumbing) elements from the same IFC file.

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy

props = bpy.context.scene.BIMClashProperties

# Step 1: Create clash set
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]
clash_set.mode = "intersection"
clash_set.tolerance = 0.001  # 1mm precision

# Step 2: Add structural elements to Group A
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/absolute/path/to/model.ifc"
clash_set.a[0].mode = "i"  # Include only filtered elements
# Note: Filter configuration via Bonsai UI or filter_groups collection

# Step 3: Add MEP elements to Group B
bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/absolute/path/to/model.ifc"
clash_set.b[0].mode = "i"

# Step 4: Set output path and execute
props.export_path = "/absolute/path/to/clashes.json"
bpy.ops.bim.execute_ifc_clash()

# Step 5: Load and review results
props.clash_results_path = "/absolute/path/to/clashes.json"
bpy.ops.bim.select_ifc_clash_results()

# Step 6: Navigate to first clash
if len(clash_set.clashes) > 0:
    bpy.ops.bim.select_clash(index=0)
    print(f"Clash: {clash_set.clashes[0].a_name} vs {clash_set.clashes[0].b_name}")
```

---

## Example 2: Multi-File Clearance Check

Checks whether elements from separate discipline models maintain required clearance distances.

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy

props = bpy.context.scene.BIMClashProperties

# Create clearance clash set
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]
clash_set.mode = "clearance"
clash_set.clearance = 0.3  # 300mm maintenance clearance

# Group A: Electrical equipment (from electrical model)
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/electrical.ifc"
clash_set.a[0].mode = "a"  # All elements

# Group B: Structural elements (from structural model)
bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/path/to/structural.ifc"
clash_set.b[0].mode = "a"  # All elements

# Execute with BCF output for stakeholder sharing
props.export_path = "/path/to/clearance_issues.bcf"
props.should_create_clash_snapshots = True
bpy.ops.bim.execute_ifc_clash()
```

---

## Example 3: Smart Grouping Workflow

Groups large clash result sets by spatial proximity to identify systemic issues.

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy

props = bpy.context.scene.BIMClashProperties

# Prerequisite: clash detection has been run, results exist
props.clash_results_path = "/path/to/clashes.json"

# Configure grouping distance (meters)
props.smart_clash_grouping_max_distance = 5  # 5m clustering radius

# Run smart grouping
bpy.ops.bim.smart_clash_group()

# Load groups for the active clash set
bpy.ops.bim.load_smart_groups_for_active_clash_set()

# Iterate all smart groups
for i, group in enumerate(props.smart_clash_groups):
    element_count = len(group.global_ids)
    print(f"Group {group.number}: {element_count} elements")

# Navigate to a specific group (selects elements, frames viewport)
if len(props.smart_clash_groups) > 0:
    bpy.ops.bim.select_smart_group(group_index=0)
```

---

## Example 4: Complete Clash-to-BCF Pipeline

End-to-end workflow from clash detection through BCF issue tracking.

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy

props = bpy.context.scene.BIMClashProperties

# === Phase 1: Define clash sets ===
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]
clash_set.mode = "intersection"
clash_set.tolerance = 0.002  # 2mm

# Group A: HVAC ductwork
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/hvac.ifc"
clash_set.a[0].mode = "a"

# Group B: Structural frame
bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/path/to/structural.ifc"
clash_set.b[0].mode = "a"

# === Phase 2: Execute with BCF output ===
props.export_path = "/path/to/hvac_structural_clashes.bcf"
props.should_create_clash_snapshots = True
bpy.ops.bim.execute_ifc_clash()

# === Phase 3: Load BCF into Bonsai BCF panel ===
bpy.ops.bim.load_bcf_project(filepath="/path/to/hvac_structural_clashes.bcf")

# Each clash is now a BCF topic with:
# - Viewpoint positioned at clash location
# - Component references for both clashing elements
# - Snapshot image (if should_create_clash_snapshots was True)
```

---

## Example 5: IfcClash Standalone (Without Blender)

Run clash detection from command line or standalone Python script.

### Python API

```python
# IfcOpenShell v0.8.x — Standalone Python (no Blender required)
import ifcclash
import json
import logging

# Configure logging
logger = logging.getLogger("IfcClash")
logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))
logger.addHandler(handler)

# Configure settings
settings = ifcclash.ClashSettings()
settings.output = "/path/to/results.json"
settings.logger = logger

# Define clash sets
clash_sets = [
    {
        "name": "Pipes vs Walls",
        "mode": "intersection",
        "tolerance": 0.001,
        "a": [
            {
                "file": "/path/to/plumbing.ifc",
                "mode": "i",
                "selector": "IfcFlowSegment",
            }
        ],
        "b": [
            {
                "file": "/path/to/architectural.ifc",
                "mode": "i",
                "selector": "IfcWall",
            }
        ],
    },
    {
        "name": "Duct Clearance Check",
        "mode": "clearance",
        "clearance": 0.15,  # 150mm
        "a": [
            {
                "file": "/path/to/hvac.ifc",
                "mode": "i",
                "selector": "IfcDuctSegment",
            }
        ],
        "b": [
            {
                "file": "/path/to/structural.ifc",
                "mode": "i",
                "selector": "IfcBeam",
            }
        ],
    }
]

# Execute
clasher = ifcclash.Clasher(settings)
clasher.clash_sets = clash_sets
clasher.clash()
clasher.export()

# Parse results
with open("/path/to/results.json") as f:
    results = json.load(f)

for clash_set_result in results:
    name = clash_set_result["name"]
    clashes = clash_set_result.get("clashes", {})
    print(f"\n{name}: {len(clashes)} clashes found")
    for key, clash in clashes.items():
        print(f"  {clash['a_name']} ({clash['a_ifc_class']}) "
              f"vs {clash['b_name']} ({clash['b_ifc_class']})")
        print(f"  Distance: {clash['distance']:.4f}m")
```

### CLI Usage

```bash
# IfcOpenShell v0.8.x — Command line
# Create clash_config.json first (see JSON format below)
python -m ifcclash clash_config.json -o results.json
```

### JSON Configuration File Format

```json
[
    {
        "name": "Structural vs MEP",
        "mode": "intersection",
        "tolerance": 0.001,
        "a": [
            {
                "file": "/absolute/path/to/structural.ifc",
                "mode": "i",
                "selector": "IfcBeam"
            },
            {
                "file": "/absolute/path/to/structural.ifc",
                "mode": "i",
                "selector": "IfcColumn"
            }
        ],
        "b": [
            {
                "file": "/absolute/path/to/mep.ifc",
                "mode": "i",
                "selector": "IfcFlowSegment"
            }
        ]
    }
]
```

---

## Example 6: Export and Reimport Clash Set Configuration

Save clash set definitions for reuse across projects or team members.

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy

# Export current clash sets to JSON (opens file dialog)
bpy.ops.bim.export_clash_sets()

# Import clash sets from previously saved JSON (opens file dialog)
bpy.ops.bim.import_clash_sets()
```

---

## Example 7: Iterate All Clashes and Generate Custom Report

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy
import csv

props = bpy.context.scene.BIMClashProperties

# Prerequisite: clash results loaded
clash_set = props.clash_sets[props.active_clash_set_index]

# Generate CSV report
with open("/path/to/clash_report.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["Index", "Element A", "A GlobalId", "Element B", "B GlobalId", "Type"])
    for i, clash in enumerate(clash_set.clashes):
        writer.writerow([
            i,
            clash.a_name,
            clash.a_global_id,
            clash.b_name,
            clash.b_global_id,
            clash.clash_type,
        ])

print(f"Report generated: {len(clash_set.clashes)} clashes exported")
```

---

## Example 8: Programmatic Clash Review with Element Selection

```python
# Bonsai v0.8.x — Run inside Blender via blender --python
import bpy
import bonsai.tool as tool
from bonsai.bim.ifc import IfcStore

props = bpy.context.scene.BIMClashProperties
model = tool.Ifc.get()

if model is None:
    raise RuntimeError("No IFC project loaded.")

# Load clash results
props.clash_results_path = "/path/to/clashes.json"
bpy.ops.bim.select_ifc_clash_results()

clash_set = props.clash_sets[props.active_clash_set_index]

# Review each clash: look up elements, check properties
for i, clash in enumerate(clash_set.clashes):
    # Look up Blender objects via GlobalId
    obj_a = IfcStore.guid_map.get(clash.a_global_id)
    obj_b = IfcStore.guid_map.get(clash.b_global_id)

    if obj_a and obj_b:
        # Get IFC entities for property inspection
        entity_a = tool.Ifc.get_entity(obj_a)
        entity_b = tool.Ifc.get_entity(obj_b)

        print(f"Clash {i}:")
        print(f"  A: {entity_a.is_a()} '{entity_a.Name}' (#{entity_a.id()})")
        print(f"  B: {entity_b.is_a()} '{entity_b.Name}' (#{entity_b.id()})")

        # Select clash for viewport inspection
        bpy.ops.bim.select_clash(index=i)

# Clean up visualization
bpy.ops.bim.hide_clash()
```

---

## Sources

- Bonsai source: `src/bonsai/bonsai/bim/module/clash/operator.py`
- Bonsai source: `src/bonsai/bonsai/tool/clash.py`
- IfcClash source: `src/ifcclash/ifcclash/ifcclash.py`
- IfcClash docs: https://docs.ifcopenshell.org/ifcclash.html
