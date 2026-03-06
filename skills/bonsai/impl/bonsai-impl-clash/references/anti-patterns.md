# Bonsai Clash Detection — Anti-Patterns

> **Version**: Bonsai v0.8.x | IfcClash (IfcOpenShell v0.8.x) | Blender 4.2.0+ | Python 3.11

---

## AP-1: Running Clash Detection on Unsaved IFC Files

### What happens

IfcClash reads IFC files **from disk**, not from Blender's in-memory model. If the IFC file has unsaved modifications, clash detection runs against the stale on-disk version. Results miss recent changes and report false positives/negatives.

### Wrong

```python
# Bonsai v0.8.x — WRONG: IFC file has unsaved changes
import bpy

# User modified walls in Blender but did NOT save the IFC file
bpy.ops.bim.add_clash_set()
clash_set = bpy.context.scene.BIMClashProperties.clash_sets[0]
clash_set.a[0].name = "/path/to/model.ifc"  # Stale on disk
bpy.ops.bim.execute_ifc_clash()  # Detects against OLD geometry
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Save IFC file before clash detection
import bpy
from bonsai.bim.ifc import IfcStore

# Save the IFC file to disk first
model = IfcStore.get_file()
if model is not None and IfcStore.path:
    model.write(IfcStore.path)

# Now run clash detection against the current state
bpy.ops.bim.execute_ifc_clash()
```

### WHY

IfcClash is a file-based engine. It opens IFC files independently of Blender's scene state. The `ClashSource.name` property points to a filesystem path. The engine calls `ifcopenshell.open(path)` on that path — it has no access to `IfcStore.file` or any in-memory modifications.

---

## AP-2: Using Collision Mode with allow_touching for Hard Clash Detection

### What happens

`collision` mode with `allow_touching=True` flags every surface contact as a clash. In BIM models, many elements intentionally share surfaces (wall on slab, beam on column). This generates hundreds of false positives that obscure real conflicts.

### Wrong

```python
# Bonsai v0.8.x — WRONG: False positives from shared surfaces
import bpy

props = bpy.context.scene.BIMClashProperties
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]
clash_set.mode = "collision"
clash_set.allow_touching = True  # Flags all touching surfaces
bpy.ops.bim.execute_ifc_clash()
# Result: 500+ false positives from walls sitting on slabs
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Use intersection mode for hard clashes
import bpy

props = bpy.context.scene.BIMClashProperties
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]
clash_set.mode = "intersection"  # Only detects actual penetrations
clash_set.tolerance = 0.002      # 2mm precision
bpy.ops.bim.execute_ifc_clash()
# Result: Only true geometric overlaps reported
```

### WHY

`intersection` mode detects geometry that protrudes into or pierces another element's volume. This is what "hard clash" means in BIM coordination. `collision` mode detects ANY contact including intentional surface adjacency. Use `collision` only when detecting unintended touching is the specific goal.

---

## AP-3: Running Unfiltered Clash Detection on Large Models

### What happens

Checking all elements against all elements without filtering creates an O(n²) comparison space. A model with 10,000 elements generates up to 50 million pairwise checks. Runtime increases from seconds to hours, and results contain irrelevant clashes between unrelated disciplines.

### Wrong

```python
# Bonsai v0.8.x — WRONG: No filtering on large model
import bpy

props = bpy.context.scene.BIMClashProperties
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]

bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/full_building.ifc"
clash_set.a[0].mode = "a"  # ALL elements — no filtering

bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/path/to/full_building.ifc"
clash_set.b[0].mode = "a"  # ALL elements — no filtering

bpy.ops.bim.execute_ifc_clash()
# Result: Hours of runtime, thousands of irrelevant clashes
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Filter by discipline/element class
import bpy

props = bpy.context.scene.BIMClashProperties
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]

# Group A: Only structural beams and columns
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/full_building.ifc"
clash_set.a[0].mode = "i"  # Include filtered elements only

# Group B: Only MEP segments
bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/path/to/full_building.ifc"
clash_set.b[0].mode = "i"  # Include filtered elements only

bpy.ops.bim.execute_ifc_clash()
# Result: Targeted check, fast execution, relevant results
```

For standalone IfcClash, use the `selector` field:

```python
# IfcOpenShell v0.8.x — Standalone with selectors
clash_sets = [{
    "name": "Beams vs Ducts",
    "a": [{"file": "/path/to/model.ifc", "mode": "i", "selector": "IfcBeam"}],
    "b": [{"file": "/path/to/model.ifc", "mode": "i", "selector": "IfcDuctSegment"}],
}]
```

### WHY

The FCL collision library performs broadphase + narrowphase detection. Without filtering, the broadphase tree contains ALL geometric representations. Even with spatial acceleration, the number of narrowphase collision checks scales quadratically with element count. Filtering reduces the element sets to only relevant disciplines, cutting both runtime and noise.

---

## AP-4: Reviewing Hundreds of Clashes Without Smart Grouping

### What happens

Large clash result sets (100+ clashes) are impractical to review individually. Many clashes cluster around the same spatial conflict (e.g., 20 pipe segments all clashing with the same beam). Without grouping, reviewers waste time on redundant findings and miss the systemic pattern.

### Wrong

```python
# Bonsai v0.8.x — WRONG: Manual iteration of 500 clashes
import bpy

props = bpy.context.scene.BIMClashProperties
clash_set = props.clash_sets[props.active_clash_set_index]

# One-by-one review of every clash
for i in range(len(clash_set.clashes)):
    bpy.ops.bim.select_clash(index=i)
    # Inspect... 500 times...
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Group first, then review by group
import bpy

props = bpy.context.scene.BIMClashProperties

# Group spatially proximate clashes
props.smart_clash_grouping_max_distance = 3  # 3m radius
bpy.ops.bim.smart_clash_group()
bpy.ops.bim.load_smart_groups_for_active_clash_set()

# Review by group (each group represents one systemic issue)
for i, group in enumerate(props.smart_clash_groups):
    print(f"Group {group.number}: {len(group.global_ids)} related elements")
    bpy.ops.bim.select_smart_group(group_index=i)
```

### WHY

Smart grouping uses the OPTICS clustering algorithm to identify spatial clusters. A single rerouted duct running through 15 beams generates 15 individual clashes but ONE smart group. Reviewing 20 groups instead of 300 clashes reduces coordination time by an order of magnitude and surfaces the root cause (the duct routing) instead of the symptoms (individual intersections).

---

## AP-5: Defining Only Group A Without Group B

### What happens

IfcClash supports omitting Group B (checking within Group A only), but Bonsai's module expects BOTH groups to be defined. Missing Group B produces empty results or runtime errors in the Bonsai workflow. Even for intra-model checks, both groups must exist.

### Wrong

```python
# Bonsai v0.8.x — WRONG: Missing Group B
import bpy

props = bpy.context.scene.BIMClashProperties
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]

# Only Group A defined
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/model.ifc"
# Group B is empty — Bonsai does not handle this case

bpy.ops.bim.execute_ifc_clash()
# Result: No clashes detected or error
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Both groups defined for intra-model check
import bpy

props = bpy.context.scene.BIMClashProperties
bpy.ops.bim.add_clash_set()
clash_set = props.clash_sets[props.active_clash_set_index]

# Group A: Structural elements from the same model
bpy.ops.bim.add_clash_source(group="a")
clash_set.a[0].name = "/path/to/model.ifc"
clash_set.a[0].mode = "i"  # Filter to structural

# Group B: MEP elements from the same model
bpy.ops.bim.add_clash_source(group="b")
clash_set.b[0].name = "/path/to/model.ifc"  # Same file is valid
clash_set.b[0].mode = "i"  # Filter to MEP

bpy.ops.bim.execute_ifc_clash()
```

### WHY

Bonsai's `tool.Clash.export_clash_sets()` iterates both `clash_set.a` and `clash_set.b` collections. When Group B is empty, the exported configuration lacks a comparison target. While standalone IfcClash supports within-group-A detection, Bonsai's operator layer requires both groups to produce valid clash set JSON for the engine.

---

## AP-6: Using Relative File Paths in ClashSource

### What happens

`ClashSource.name` stores the filepath to an IFC file on disk. IfcClash opens this file with `ifcopenshell.open(path)`. Relative paths resolve against the current working directory, which varies between Blender sessions, scripts, and operating systems. Clash detection fails silently or loads the wrong file.

### Wrong

```python
# Bonsai v0.8.x — WRONG: Relative path
clash_set.a[0].name = "models/structural.ifc"  # Relative — fragile
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Absolute path
clash_set.a[0].name = "/home/user/project/models/structural.ifc"  # Absolute
```

### WHY

Blender's working directory depends on how it was launched (terminal, shortcut, file association). A relative path that works from one terminal session breaks from another. IfcClash runs file operations independently and does not resolve paths relative to the .blend file. Absolute paths guarantee consistent file resolution across all execution contexts.

---

## AP-7: Expecting Real-Time Clash Monitoring

### What happens

Developers attempt to implement live clash checking that detects conflicts as elements move in the viewport. Bonsai has NO real-time clash detection capability. Clash detection is a batch operation that reads IFC files from disk.

### Wrong

```python
# Bonsai v0.8.x — WRONG: Attempting live clash monitoring
import bpy

def on_transform(scene, depsgraph):
    """Hypothetical live clash check — DOES NOT EXIST"""
    bpy.ops.bim.execute_ifc_clash()  # Called every frame change

bpy.app.handlers.depsgraph_update_post.append(on_transform)
```

### Correct

```python
# Bonsai v0.8.x — CORRECT: Batch clash detection at coordination milestones
import bpy
from bonsai.bim.ifc import IfcStore

# Save current state
model = IfcStore.get_file()
if model is not None and IfcStore.path:
    model.write(IfcStore.path)

# Run clash detection as an explicit batch operation
bpy.ops.bim.execute_ifc_clash()
```

### WHY

Clash detection involves loading IFC files, building collision trees (BVH), and running geometric intersection tests. This process takes seconds to minutes depending on model size. Attaching it to `depsgraph_update_post` would freeze Blender on every frame update. The architecture is designed for checkpoint-based coordination: model, save, detect, review, resolve.

---

## Sources

- Bonsai source: `src/bonsai/bonsai/bim/module/clash/operator.py`
- Bonsai source: `src/bonsai/bonsai/tool/clash.py`
- IfcClash source: `src/ifcclash/ifcclash/ifcclash.py`
- IfcClash docs: https://docs.ifcopenshell.org/ifcclash.html
