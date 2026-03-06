# Bonsai Clash Detection — API Reference

> **Version**: Bonsai v0.8.x | IfcClash (IfcOpenShell v0.8.x) | Blender 4.2.0+ | Python 3.11

---

## Bonsai Operators (`bpy.ops.bim.*`)

### AddClashSet

```python
# Bonsai v0.8.x
bpy.ops.bim.add_clash_set()
```

- **bl_idname**: `bim.add_clash_set`
- **Parameters**: None
- **Effect**: Creates a new `ClashSet` entry in `BIMClashProperties.clash_sets` with default name "New Clash Set"
- **Returns**: `{"FINISHED"}`

### RemoveClashSet

```python
# Bonsai v0.8.x
bpy.ops.bim.remove_clash_set(index=0)
```

- **bl_idname**: `bim.remove_clash_set`
- **Parameters**:
  - `index` (IntProperty): Index of clash set to remove
- **Returns**: `{"FINISHED"}`

### AddClashSource

```python
# Bonsai v0.8.x
bpy.ops.bim.add_clash_source(group="a")
```

- **bl_idname**: `bim.add_clash_source`
- **Parameters**:
  - `group` (StringProperty): Target group — `"a"` or `"b"`
- **Effect**: Adds empty `ClashSource` to the specified group of the active clash set
- **Returns**: `{"FINISHED"}`

### RemoveClashSource

```python
# Bonsai v0.8.x
bpy.ops.bim.remove_clash_source(index=0, group="a")
```

- **bl_idname**: `bim.remove_clash_source`
- **Parameters**:
  - `index` (IntProperty): Index of source to remove
  - `group` (StringProperty): Target group — `"a"` or `"b"`
- **Returns**: `{"FINISHED"}`

### SelectClashSource

```python
# Bonsai v0.8.x
bpy.ops.bim.select_clash_source(index=0, group="a")
```

- **bl_idname**: `bim.select_clash_source`
- **Parameters**:
  - `index` (IntProperty): Index of source in group
  - `group` (StringProperty): Target group — `"a"` or `"b"`
  - `filter_glob` (StringProperty): `"*.ifc"` (file dialog filter)
- **Effect**: Opens file dialog, sets `ClashSource.name` to selected IFC file path
- **Returns**: `{"FINISHED"}`

### ExecuteIfcClash

```python
# Bonsai v0.8.x
bpy.ops.bim.execute_ifc_clash()
```

- **bl_idname**: `bim.execute_ifc_clash`
- **Parameters**: None (reads from `BIMClashProperties`)
- **Reads**:
  - `BIMClashProperties.clash_sets` — Clash set configuration
  - `BIMClashProperties.export_path` — Output file path (.json or .bcf)
  - `BIMClashProperties.should_create_clash_snapshots` — Snapshot flag
- **Effect**: Exports clash sets via `tool.Clash.export_clash_sets()`, runs IfcClash engine, writes results
- **Returns**: `{"FINISHED"}`

### SelectIfcClashResults

```python
# Bonsai v0.8.x
bpy.ops.bim.select_ifc_clash_results()
```

- **bl_idname**: `bim.select_ifc_clash_results`
- **Parameters**:
  - `filter_glob` (StringProperty): `"*.json"` (file dialog filter)
- **Effect**: Loads clash results from JSON, populates `ClashSet.clashes` collection, selects conflicting geometry by GlobalId in viewport
- **Returns**: `{"FINISHED"}`

### SelectClash

```python
# Bonsai v0.8.x
bpy.ops.bim.select_clash(index=0)
```

- **bl_idname**: `bim.select_clash`
- **Parameters**:
  - `index` (IntProperty): Index of clash in active clash set
- **Effect**: Selects both clashing elements, installs `ClashDecorator`, sets `p1`/`p2` coordinates, positions viewport camera at clash point with X-ray enabled
- **Returns**: `{"FINISHED"}`

### HideClash

```python
# Bonsai v0.8.x
bpy.ops.bim.hide_clash()
```

- **bl_idname**: `bim.hide_clash`
- **Poll**: Returns `True` only when `ClashDecorator` is installed
- **Effect**: Uninstalls clash decorator, refreshes viewport
- **Returns**: `{"FINISHED"}`

### SmartClashGroup

```python
# Bonsai v0.8.x
bpy.ops.bim.smart_clash_group()
```

- **bl_idname**: `bim.smart_clash_group`
- **Poll**: Returns `True` when `clash_results_path` exists
- **Reads**: `BIMClashProperties.smart_clash_grouping_max_distance`
- **Effect**: Groups spatially proximate clashes using OPTICS algorithm, writes groups to JSON
- **Returns**: `{"FINISHED"}`

### LoadSmartGroupsForActiveClashSet

```python
# Bonsai v0.8.x
bpy.ops.bim.load_smart_groups_for_active_clash_set()
```

- **bl_idname**: `bim.load_smart_groups_for_active_clash_set`
- **Poll**: Returns `True` when active clash set exists
- **Effect**: Populates `BIMClashProperties.smart_clash_groups` from saved JSON
- **Returns**: `{"FINISHED"}`

### SelectSmartGroup

```python
# Bonsai v0.8.x
bpy.ops.bim.select_smart_group(group_index=0)
```

- **bl_idname**: `bim.select_smart_group`
- **Parameters**:
  - `group_index` (IntProperty): Index of smart group
- **Poll**: Validates IFC file loaded, viewport objects exist, active smart group exists
- **Effect**: Selects all elements in group by GlobalId, frames viewport on selection
- **Returns**: `{"FINISHED"}`

### ExportClashSets

```python
# Bonsai v0.8.x
bpy.ops.bim.export_clash_sets()
```

- **bl_idname**: `bim.export_clash_sets`
- **Parameters**:
  - `filter_glob` (StringProperty): `"*.json"`
- **Effect**: Serializes current clash set configuration to JSON file
- **Returns**: `{"FINISHED"}`

### ImportClashSets

```python
# Bonsai v0.8.x
bpy.ops.bim.import_clash_sets()
```

- **bl_idname**: `bim.import_clash_sets`
- **Parameters**:
  - `filter_glob` (StringProperty): `"*.json"`
- **Effect**: Loads clash set configuration from JSON, populates `BIMClashProperties.clash_sets` with all settings (mode, tolerance, clearance, sources, filters)
- **Returns**: `{"FINISHED"}`

### SelectClashResults

```python
# Bonsai v0.8.x
bpy.ops.bim.select_clash_results()
```

- **bl_idname**: `bim.select_clash_results`
- **Effect**: Sets `BIMClashProperties.clash_results_path` via file dialog
- **Returns**: `{"FINISHED"}`

### SelectSmartGroupedClashesPath

```python
# Bonsai v0.8.x
bpy.ops.bim.select_smart_grouped_clashes_path()
```

- **bl_idname**: `bim.select_smart_grouped_clashes_path`
- **Effect**: Sets `BIMClashProperties.smart_grouped_clashes_path` via file dialog
- **Returns**: `{"FINISHED"}`

---

## Bonsai Tool Layer (`bonsai.tool.clash.Clash`)

### get_clash_props

```python
# Bonsai v0.8.x
@classmethod
def get_clash_props(cls) -> BIMClashProperties
```

Returns `bpy.context.scene.BIMClashProperties`.

### export_clash_sets

```python
# Bonsai v0.8.x
@classmethod
def export_clash_sets(cls) -> list[ifcclash.ClashSet]
```

Converts Bonsai `ClashSet` property groups into IfcClash `ClashSet` typed dicts. Iterates all clash sets, processes Group A and Group B sources with their filter configurations, and appends mode-specific parameters (tolerance, allow_touching, clearance).

### get_clash

```python
# Bonsai v0.8.x
@classmethod
def get_clash(cls, clash_set, a_global_id: str, b_global_id: str) -> Union[dict, None]
```

Searches a clash set's results for a specific clash pair using composite key `(a_global_id, b_global_id)`. Returns the clash result dict or `None`.

### get_clash_set

```python
# Bonsai v0.8.x
@classmethod
def get_clash_set(cls, name: str) -> Union[dict, None]
```

Locates a clash set by name within `ClashStore`. Returns the clash set dict or `None`.

### get_clash_sets

```python
# Bonsai v0.8.x
@classmethod
def get_clash_sets(cls) -> list[dict]
```

Returns the complete list of loaded clash sets from `ClashStore`.

### import_active_clashes

```python
# Bonsai v0.8.x
@classmethod
def import_active_clashes(cls) -> None
```

Populates the active clash set's `clashes` collection from stored results. Clears existing data, retrieves clashes, sorts by distance, and creates Blender `Clash` property group entries with metadata (GlobalIds, names, types, status).

### clear_active_clash_set_results

```python
# Bonsai v0.8.x
@classmethod
def clear_active_clash_set_results(cls) -> None
```

Clears all clashes from the active clash set and unsets the `clashes_loaded` property.

### load_clash_sets

```python
# Bonsai v0.8.x
@classmethod
def load_clash_sets(cls, fn: str) -> None
```

Loads clash results from a JSON file path into `ClashStore`. The JSON file is the output of IfcClash execution.

### look_at

```python
# Bonsai v0.8.x
@classmethod
def look_at(cls, target: Vector, location: Vector) -> None
```

Positions the 3D viewport camera to look at `target` from `location`. Sets view location, rotation (quaternion tracking), distance, and enables X-ray shading for clash point visibility.

---

## ClashStore (Static Class)

```python
# Bonsai v0.8.x — bonsai.tool.clash
class ClashStore:
    clash_sets: list = []  # Class-level storage

    @staticmethod
    def purge():
        ClashStore.clash_sets = []
```

Static storage for loaded clash set results. Cleared via `purge()` when reloading or resetting.

---

## IfcClash Engine API (`ifcclash`)

### ClashSettings

```python
# IfcOpenShell v0.8.x — ifcclash.ClashSettings
class ClashSettings:
    def __init__(self):
        self.logger: logging.Logger = None
        self.output: str = "clashes.json"
```

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `logger` | `logging.Logger` | `None` | Logger instance for diagnostic output |
| `output` | `str` | `"clashes.json"` | Output file path (.json or .bcf) |

### Clasher

```python
# IfcOpenShell v0.8.x — ifcclash.Clasher
class Clasher:
    def __init__(self, settings: ClashSettings): ...
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `clash()` | `() -> None` | Execute clash detection across all configured sets |
| `process_clash_set(clash_set)` | `(clash_set: dict) -> None` | Analyze individual clash set |
| `create_group(name)` | `(name: str) -> dict` | Initialize element grouping container |
| `load_ifc(path)` | `(path: str) -> ifcopenshell.file` | Load and cache IFC file |
| `add_collision_objects(...)` | `(...)` | Filter and add geometry to detection tree |
| `export()` | `() -> None` | Write results to output file (delegates to format-specific method) |
| `export_json()` | `() -> None` | Write results as JSON |
| `export_bcfxml()` | `() -> None` | Write results as BCF XML with viewpoints |
| `smart_group_clashes(...)` | `(max_distance: int) -> dict` | Cluster nearby clashes using OPTICS algorithm |
| `get_viewpoint_snapshot(viewpoint)` | `(viewpoint) -> None` | Extensible snapshot generation hook |

| Property | Type | Description |
|----------|------|-------------|
| `clash_sets` | `list[dict]` | Clash set configuration (assign before calling `clash()`) |

### Clash Set Dictionary Schema

```python
# IfcOpenShell v0.8.x — ClashSet TypedDict
{
    "name": str,                    # Human-readable identifier
    "a": [ClashSource, ...],        # Group A sources (required)
    "b": [ClashSource, ...],        # Group B sources (optional)
    "mode": str,                    # "intersection" | "collision" | "clearance"
    "tolerance": float,             # Detection precision threshold
    "allow_touching": bool,         # Accept surface contact
    "clearance": float,             # Proximity distance for clearance mode
    "check_all": bool,              # Check all combinations
    "clashes": dict,                # Results (populated by clash())
}
```

### ClashSource Dictionary Schema

```python
# IfcOpenShell v0.8.x — ClashSource TypedDict
{
    "file": str,                    # Absolute path to .ifc file (required)
    "mode": str,                    # "a" (all) | "i" (include) | "e" (exclude)
    "selector": str,                # IFC class name or query for filtering
    "ifc": ifcopenshell.file,       # Loaded IFC instance (internal, set by engine)
}
```

### ClashResult Dictionary Schema

```python
# IfcOpenShell v0.8.x — ClashResult TypedDict
{
    "a_global_id": str,             # GlobalId of element A
    "b_global_id": str,             # GlobalId of element B
    "a_ifc_class": str,             # IFC class of element A
    "b_ifc_class": str,             # IFC class of element B
    "a_name": str,                  # Name of element A
    "b_name": str,                  # Name of element B
    "p1": list[float],              # Contact point on element A [x, y, z]
    "p2": list[float],              # Contact point on element B [x, y, z]
    "distance": float,              # Distance between p1 and p2
    "type": str,                    # Clash classification
}
```

---

## Bonsai Property Groups

### BIMClashProperties

```python
# Bonsai v0.8.x — bpy.context.scene.BIMClashProperties
class BIMClashProperties(PropertyGroup):
    clash_sets: CollectionProperty(type=ClashSet)
    active_clash_set_index: IntProperty()
    active_clash_index: IntProperty()
    export_path: StringProperty()                        # .bcf or .json output
    clash_results_path: StringProperty()                 # Loaded results path
    should_create_clash_snapshots: BoolProperty(default=False)
    smart_clash_groups: CollectionProperty(type=SmartClashGroup)
    active_smart_group_index: IntProperty()
    smart_clash_grouping_max_distance: IntProperty(default=3, soft_min=1, soft_max=10)
    p1: FloatVectorProperty(default=(0,0,0), subtype="XYZ")
    p2: FloatVectorProperty(default=(0,0,0), subtype="XYZ")
    active_clash_text: StringProperty()
    blender_clash_set_a: CollectionProperty(type=StrProperty)
    blender_clash_set_b: CollectionProperty(type=StrProperty)
    smart_grouped_clashes_path: StringProperty()
```

### ClashSet

```python
# Bonsai v0.8.x
class ClashSet(PropertyGroup):
    mode: EnumProperty(items=[
        ("intersection", "Intersection", "Detect objects that protrude or pierce another object"),
        ("collision", "Collision", "Detect touching objects with any surface collision"),
        ("clearance", "Clearance", "Detect objects within a proximity threshold"),
    ])
    tolerance: FloatProperty(default=0.002, subtype="DISTANCE")
    clearance: FloatProperty(default=0.01, subtype="DISTANCE")
    allow_touching: BoolProperty(default=False)
    check_all: BoolProperty(default=False)
    a: CollectionProperty(type=ClashSource)
    b: CollectionProperty(type=ClashSource)
    clashes: CollectionProperty(type=Clash)
    clashes_loaded: BoolProperty()
```

### ClashSource

```python
# Bonsai v0.8.x
class ClashSource(PropertyGroup):
    name: StringProperty()          # Absolute filepath to .ifc file
    filter_groups: CollectionProperty(type=BIMFilterGroup)
    mode: EnumProperty(items=[
        ("a", "All Elements", ""),
        ("i", "Include", ""),
        ("e", "Exclude", ""),
    ])
```

### Clash

```python
# Bonsai v0.8.x
class Clash(PropertyGroup):
    a_global_id: StringProperty()
    b_global_id: StringProperty()
    a_name: StringProperty()
    b_name: StringProperty()
    clash_type: EnumProperty()      # From CLASH_TYPE_ITEMS
    status: BoolProperty(default=False)
```

### SmartClashGroup

```python
# Bonsai v0.8.x
class SmartClashGroup(PropertyGroup):
    number: StringProperty()
    global_ids: CollectionProperty(type=StrProperty)
```

---

## Sources

- Bonsai source: `src/bonsai/bonsai/bim/module/clash/operator.py`
- Bonsai source: `src/bonsai/bonsai/bim/module/clash/prop.py`
- Bonsai source: `src/bonsai/bonsai/tool/clash.py`
- IfcClash source: `src/ifcclash/ifcclash/ifcclash.py`
- IfcClash docs: https://docs.ifcopenshell.org/ifcclash.html
