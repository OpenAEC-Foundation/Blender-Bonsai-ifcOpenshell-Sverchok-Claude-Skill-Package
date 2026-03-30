---
name: bonsai-impl-clash
description: >
  Use when performing clash detection, interference checking, or spatial coordination in
  BIM models using Bonsai and IfcClash. Prevents the common mistake of running clash detection
  without proper element group filters (testing everything against everything). Covers clash
  set definition, element filtering, smart grouping of results, and BCF integration for
  issue tracking.
  Keywords: clash detection, IfcClash, interference, collision, spatial coordination,
  clash set, element filter, BCF integration, clash report.
license: MIT
compatibility: "Designed for Claude Code. Requires Blender with Bonsai addon."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Clash Detection Implementation

> **Version**: Bonsai v0.8.x | IfcClash (from IfcOpenShell v0.8.x) | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.bim.module.clash` — NEVER `blenderbim.bim.module.clash`
> **Engine**: IfcClash — frontend for FCL (Flexible Collision Library)

## Critical Warnings

1. **ALWAYS** save IFC files to disk BEFORE running clash detection. IfcClash reads files from disk, NOT from Blender's in-memory model.
2. **ALWAYS** define BOTH Group A and Group B in every clash set, even for intra-model checks (e.g., structural vs. MEP in the same file).
3. **ALWAYS** use `filter_groups` to limit clash scope. Unfiltered detection on large models produces excessive runtime and noise.
4. **ALWAYS** use `bim.smart_clash_group` for result sets exceeding 50 clashes. Manual review of hundreds of individual clashes is impractical.
5. **NEVER** expect real-time clash checking. Clash detection is a batch operation — there is NO live monitoring.
6. **NEVER** use `collision` mode with `allow_touching=True` for detecting hard overlaps — this generates false positives where elements share surfaces. Use `intersection` mode instead.
7. **ALWAYS** specify an output path (`.bcf` or `.json`) before executing clash detection.
8. **ALWAYS** use `bonsai.*` imports. NEVER use `blenderbim.*` — renamed in 2024 (v0.8.0+).

---

## Decision Tree: Clash Detection Mode Selection

```
What type of geometric conflict are you detecting?
│
├── Physical overlap (pipe through wall, duct through beam)?
│   └── Use mode = "intersection"
│       └── tolerance = 0.001 (1mm) for architectural
│       └── tolerance = 0.0001 (0.1mm) for mechanical
│
├── Surface contact (elements touching)?
│   └── Use mode = "collision"
│       └── allow_touching = False (default) for strict contact detection
│       └── allow_touching = True ONLY when surface sharing is acceptable
│
└── Proximity clearance (maintenance access, fire safety)?
    └── Use mode = "clearance"
        └── clearance = required_distance_in_meters
        └── Example: clearance = 0.3 for 300mm maintenance gap
```

## Decision Tree: Output Format Selection

```
What will you do with clash results?
│
├── Share with external stakeholders (Navisworks, Solibri, BIMcollab)?
│   └── Export as .bcf
│       └── Each clash becomes a BCF topic with viewpoint
│       └── Set should_create_clash_snapshots = True for visual context
│
├── Programmatic analysis or custom reporting?
│   └── Export as .json
│       └── Contains coordinates, GlobalIds, distances
│       └── Parseable by any JSON library
│
└── Internal review within Bonsai?
    └── Export as .json
        └── Load with bim.select_ifc_clash_results
        └── Navigate clashes in 3D viewport
```

---

## Essential Patterns

### Pattern 1: Define a Clash Set

```python
# Bonsai v0.8.x: Create and configure a clash set
import bpy

props = bpy.context.scene.BIMClashProperties

# Create a new clash set
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]

# Set detection mode
clash_set.mode = "intersection"    # "intersection" | "collision" | "clearance"
clash_set.tolerance = 0.002        # Default: 0.002m (2mm)

# Add Group A source
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/structural.ifc"
clash_set.a[0].mode = "i"  # "a"=all, "i"=include filtered, "e"=exclude filtered

# Add Group B source
bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/path/to/mep.ifc"
clash_set.b[0].mode = "i"
```

### Pattern 2: Run Clash Detection

```python
# Bonsai v0.8.x: Execute clash detection
import bpy

props = bpy.context.scene.BIMClashProperties

# Set output path BEFORE execution
props.export_path = "/path/to/results.json"

# Execute clash detection
bpy.ops.bim.execute_ifc_clash()

# For BCF output with snapshots
props.should_create_clash_snapshots = True
props.export_path = "/path/to/results.bcf"
bpy.ops.bim.execute_ifc_clash()
```

### Pattern 3: Review Clash Results

```python
# Bonsai v0.8.x: Load and navigate clash results
import bpy

props = bpy.context.scene.BIMClashProperties

# Load results from JSON file
props.clash_results_path = "/path/to/results.json"
bpy.ops.bim.select_ifc_clash_results()

# Navigate to a specific clash (highlights pair, positions viewport)
bpy.ops.bim.select_clash(index=0)

# Access clash data
clash_set = props.clash_sets[props.active_clash_set_index]
for i, clash in enumerate(clash_set.clashes):
    print(f"Clash {i}: {clash.a_name} vs {clash.b_name}")
    print(f"  A GlobalId: {clash.a_global_id}")
    print(f"  B GlobalId: {clash.b_global_id}")
    print(f"  Type: {clash.clash_type}")

# Remove clash visualization overlay
bpy.ops.bim.hide_clash()
```

### Pattern 4: Use IfcClash Standalone (Without Bonsai)

```python
# IfcClash standalone: Bonsai v0.8.x / IfcOpenShell v0.8.x
import ifcclash
import json
import logging

# Configure settings
settings = ifcclash.ClashSettings()
settings.output = "/path/to/results.json"
settings.logger = logging.getLogger("IfcClash")
settings.logger.setLevel(logging.INFO)

# Define clash sets
clash_sets = [
    {
        "name": "Structural vs MEP",
        "a": [{"file": "/path/to/structural.ifc", "mode": "i", "selector": "IfcBeam"}],
        "b": [{"file": "/path/to/mep.ifc", "mode": "i", "selector": "IfcFlowSegment"}],
        "tolerance": 0.001,
        "mode": "intersection",
    }
]

# Run clash detection
clasher = ifcclash.Clasher(settings)
clasher.clash_sets = clash_sets
clasher.clash()
clasher.export()
```

---

## Common Operations

### Filter Configuration

Element filters control which elements participate in clash detection. Each `ClashSource` has a `mode` and optional `filter_groups`:

| Filter Mode | Value | Behavior |
|-------------|-------|----------|
| All Elements | `"a"` | Include every element from the IFC file |
| Include | `"i"` | Include ONLY elements matching the selector/filter |
| Exclude | `"e"` | Include all elements EXCEPT those matching the selector/filter |

**Standalone IfcClash selector syntax** uses IFC class names as filters:

```json
{
    "file": "/path/to/model.ifc",
    "mode": "i",
    "selector": "IfcWall"
}
```

**Bonsai UI filter groups** use `BIMFilterGroup` collections on `ClashSource.filter_groups` for more granular control (filter by class, property, attribute).

### Smart Grouping

Smart grouping clusters spatially proximate clashes using the OPTICS algorithm to reduce noise and identify systemic patterns:

```python
# Bonsai v0.8.x: Smart clash grouping
import bpy

props = bpy.context.scene.BIMClashProperties

# Set maximum clustering distance (meters, default: 3)
props.smart_clash_grouping_max_distance = 3

# Run grouping (requires clash_results_path to be set)
bpy.ops.bim.smart_clash_group()

# Load smart groups for the active clash set
bpy.ops.bim.load_smart_groups_for_active_clash_set()

# Navigate to a specific group (selects all elements, frames viewport)
bpy.ops.bim.select_smart_group(group_index=0)

# Access group data
for group in props.smart_clash_groups:
    print(f"Group {group.number}: {len(group.global_ids)} elements")
```

### BCF Integration

Clash results integrate directly with BCF (BIM Collaboration Format) for issue tracking:

```python
# Bonsai v0.8.x: Clash-to-BCF workflow
import bpy

props = bpy.context.scene.BIMClashProperties

# Step 1: Run clash detection with BCF output
props.export_path = "/path/to/clashes.bcf"
props.should_create_clash_snapshots = True
bpy.ops.bim.execute_ifc_clash()

# Step 2: Load BCF results in Bonsai BCF panel
bpy.ops.bim.load_bcf_project(filepath="/path/to/clashes.bcf")

# Each clash becomes a BCF topic containing:
# - Viewpoint positioned at the clash location
# - Component references for both clashing elements (GlobalIds)
# - Metadata: clash type, severity classification
```

### Export and Import Clash Set Configurations

```python
# Bonsai v0.8.x: Save/restore clash set definitions
import bpy

# Export current clash set configuration to JSON
bpy.ops.bim.export_clash_sets()

# Import clash set configuration from JSON
bpy.ops.bim.import_clash_sets()
```

### Clash Visualization in 3D Viewport

The `ClashDecorator` renders clash points as a viewport overlay:

- **Two colored points** at P1 and P2 (clash contact endpoints)
- **Connecting line** between P1 and P2 (when points differ)
- **Text label** at midpoint with clash identification
- Colors configurable via addon preferences (selected/unselected/special)
- Uses GPU shaders (`UNIFORM_COLOR`, `POLYLINE_UNIFORM_COLOR`)

```python
# Bonsai v0.8.x: Clash visualization properties
props = bpy.context.scene.BIMClashProperties

# P1 and P2 coordinates (set by select_clash operator)
print(f"P1: {tuple(props.p1)}")
print(f"P2: {tuple(props.p2)}")
print(f"Label: {props.active_clash_text}")
```

---

## Workflow: Complete Clash Detection Pipeline

```
1. Prepare Models
   ├── Save all IFC files to disk
   └── Verify file paths are absolute

2. Define Clash Sets
   ├── bpy.ops.bim.add_clash_set()
   ├── Configure mode (intersection/collision/clearance)
   ├── Set tolerance/clearance values
   ├── Add Group A sources with filters
   └── Add Group B sources with filters

3. Execute Detection
   ├── Set export_path (.json or .bcf)
   ├── Set should_create_clash_snapshots (for BCF)
   └── bpy.ops.bim.execute_ifc_clash()

4. Review Results
   ├── Load results: bpy.ops.bim.select_ifc_clash_results()
   ├── Navigate: bpy.ops.bim.select_clash(index=N)
   └── Hide overlay: bpy.ops.bim.hide_clash()

5. Group and Analyze
   ├── bpy.ops.bim.smart_clash_group()
   ├── bpy.ops.bim.load_smart_groups_for_active_clash_set()
   └── bpy.ops.bim.select_smart_group(group_index=N)

6. Report (optional)
   ├── BCF export for external collaboration
   └── JSON export for programmatic analysis
```

---

## Operator Quick Reference

| Operator | bl_idname | Purpose |
|----------|-----------|---------|
| AddClashSet | `bim.add_clash_set` | Create new clash set |
| RemoveClashSet | `bim.remove_clash_set` | Delete clash set by index |
| AddClashSource | `bim.add_clash_source` | Add IFC source to group (a/b) |
| RemoveClashSource | `bim.remove_clash_source` | Remove source from group |
| SelectClashSource | `bim.select_clash_source` | File picker for IFC source |
| ExecuteIfcClash | `bim.execute_ifc_clash` | Run clash detection engine |
| SelectIfcClashResults | `bim.select_ifc_clash_results` | Load clash results from JSON |
| SelectClash | `bim.select_clash` | Navigate to individual clash |
| HideClash | `bim.hide_clash` | Remove clash visualization |
| SmartClashGroup | `bim.smart_clash_group` | Group clashes by proximity |
| LoadSmartGroupsForActiveClashSet | `bim.load_smart_groups_for_active_clash_set` | Load saved smart groups |
| SelectSmartGroup | `bim.select_smart_group` | Select elements in smart group |
| ExportClashSets | `bim.export_clash_sets` | Export configuration to JSON |
| ImportClashSets | `bim.import_clash_sets` | Import configuration from JSON |
| SelectClashResults | `bim.select_clash_results` | Set clash results file path |
| SelectSmartGroupedClashesPath | `bim.select_smart_grouped_clashes_path` | Set smart groups file path |

---

## Property Reference

### BIMClashProperties (on `bpy.context.scene`)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `clash_sets` | Collection[ClashSet] | — | All defined clash sets |
| `active_clash_set_index` | Int | 0 | Currently selected clash set |
| `active_clash_index` | Int | 0 | Currently selected clash |
| `export_path` | String | `""` | Output path for .bcf or .json |
| `clash_results_path` | String | `""` | Path to loaded results file |
| `should_create_clash_snapshots` | Bool | False | Generate BCF viewpoint snapshots |
| `smart_clash_groups` | Collection[SmartClashGroup] | — | Grouped clash results |
| `active_smart_group_index` | Int | 0 | Currently selected smart group |
| `smart_clash_grouping_max_distance` | Int | 3 | Max clustering distance (meters) |
| `p1` | FloatVector | (0,0,0) | First clash contact point |
| `p2` | FloatVector | (0,0,0) | Second clash contact point |
| `active_clash_text` | String | `""` | Label text for active clash |

---

## Dependencies

- **bonsai-core-architecture** — Three-layer architecture, IfcStore, operator pattern
- **bonsai-syntax-elements** — IFC element access via tool.Ifc, entity↔object mapping

## Reference Links

- [Complete API Signatures](references/methods.md) — All method signatures for Bonsai clash module and IfcClash engine
- [Working Code Examples](references/examples.md) — End-to-end clash detection workflow examples
- [Anti-Patterns](references/anti-patterns.md) — What NOT to do when implementing clash detection

## Sources

- Bonsai source: `src/bonsai/bonsai/bim/module/clash/` (operators, props, UI, decorator)
- Bonsai source: `src/bonsai/bonsai/tool/clash.py` (tool layer implementation)
- IfcClash source: `src/ifcclash/ifcclash/ifcclash.py` (engine)
- IfcClash docs: https://docs.ifcopenshell.org/ifcclash.html
- Bonsai docs: https://docs.bonsaibim.org
