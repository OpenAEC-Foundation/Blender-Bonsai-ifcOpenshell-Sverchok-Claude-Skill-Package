# BCF Anti-Patterns

> **Version**: Bonsai v0.8.x | BCF v2.1 + v3.0
> Each entry explains WHAT not to do, WHY it fails, and the CORRECT approach.

---

## Anti-Pattern 1: Accessing BcfStore.bcfxml Directly

### WRONG

```python
# Bonsai v0.8.x — WRONG: direct attribute access
from bonsai.bim.module.bcf.bcfstore import BcfStore

bcfxml = BcfStore.bcfxml  # May be None even when a BCF file is configured
if bcfxml:
    topics = bcfxml.topics
```

### WHY This Fails

`BcfStore.bcfxml` is a class variable that starts as `None`. The `get_bcfxml()` method handles lazy initialization: it reads the filepath from `props.bcf_file`, resolves relative paths against the `.blend` file directory, and loads the BCF data on first access. Accessing the class variable directly skips all of this logic, resulting in `None` even when a valid BCF file path is configured.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: use get_bcfxml()
from bonsai.bim.module.bcf.bcfstore import BcfStore

bcfxml = BcfStore.get_bcfxml()  # Handles lazy init + path resolution
if bcfxml is None:
    raise RuntimeError("No BCF project loaded.")
topics = bcfxml.topics
```

---

## Anti-Pattern 2: Creating Topics Without Setting Author

### WRONG

```python
# Bonsai v0.8.x — WRONG: author not set
import bpy

bpy.ops.bim.new_bcf_project()
bpy.ops.bim.add_bcf_topic()  # Uses default "john@doe.com"
```

### WHY This Fails

The default author is `"john@doe.com"` — a placeholder value. Topics and comments created with this value are attributed to a non-existent person, making issue tracking meaningless. External BCF tools that import the file display this fake email, confusing project stakeholders. The author field propagates to `creation_author`, `modified_author`, and comment `author` fields throughout the BCF XML data.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: set author before creating topics
import bpy

props = bpy.context.scene.BCFProperties
props.author = "actual-user@company.com"  # Set real email

bpy.ops.bim.new_bcf_project()
bpy.ops.bim.add_bcf_topic()
```

---

## Anti-Pattern 3: Adding Viewpoint Without Active Camera

### WRONG

```python
# Bonsai v0.8.x — WRONG: no camera check
import bpy

bpy.ops.bim.add_bcf_viewpoint()  # Fails or produces empty viewpoint
```

### WHY This Fails

`AddBcfViewpoint` reads the camera position and orientation from `bpy.context.scene.camera`. If no camera exists, the operator has no spatial reference for the viewpoint. The operator's `poll()` method checks for an active camera. Without one, the operator returns `{'CANCELLED'}` or produces a viewpoint with no useful camera data. The PNG snapshot capture (`bpy.ops.render.opengl()`) also requires a camera for rendering.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: verify camera before viewpoint
import bpy

if bpy.context.scene.camera is None:
    bpy.ops.object.camera_add(location=(0, 0, 5))
    bpy.context.scene.camera = bpy.context.active_object

# Position camera at the issue location
bpy.context.scene.camera.location = (10.0, -5.0, 3.5)
bpy.context.scene.camera.rotation_euler = (1.22, 0.0, 0.78)

bpy.ops.bim.add_bcf_viewpoint()
```

---

## Anti-Pattern 4: Forgetting to Save BCF Changes

### WRONG

```python
# Bonsai v0.8.x — WRONG: changes lost on exit
import bpy

bpy.ops.bim.new_bcf_project()
bpy.ops.bim.add_bcf_topic()
# ... make many edits ...
# Exit Blender without saving BCF
```

### WHY This Fails

All BCF modifications are stored in-memory in the `BcfStore.bcfxml` object. The BCF data is NOT written to disk automatically. When Blender closes or the BCF project is unloaded, all unsaved changes are permanently lost. Unlike Bonsai's native IFC workflow (which tracks changes to `.ifc` files), BCF has no auto-save mechanism.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: save after modifications
import bpy

bpy.ops.bim.new_bcf_project()
bpy.ops.bim.add_bcf_topic()
# ... make many edits ...

# Save to disk
bpy.ops.bim.save_bcf_project(filepath="/path/to/output.bcf")

# Or save to the current file path
bpy.ops.bim.save_bcf_project(save_current_bcf=True)
```

---

## Anti-Pattern 5: Mixing BCF v2.1 and v3.0 Operations

### WRONG

```python
# Bonsai v0.8.x — WRONG: version mismatch
import bpy
from bcf.v3.model import Comment  # v3 model class

# Load a v2.1 BCF file
bpy.ops.bim.load_bcf_project(filepath="/path/to/v2_issues.bcf")

# Attempt to create v3-style objects on v2 project
# This causes type mismatches and XML serialization errors
```

### WHY This Fails

BCF v2.1 and v3.0 use different data model classes (`bcf.v2.model` vs `bcf.v3.model`), different XML schemas, and different document storage patterns. The version is set at project creation or file load time and embedded in the BCF XML structure. Mixing v2 and v3 model objects causes serialization failures when saving. The `BcfStore.set()` method auto-detects version from `bcfxml.version.version_id` — this detection is one-way and immutable.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: use version-appropriate operations
import bpy
from bonsai.bim.module.bcf.bcfstore import BcfStore

# Load BCF (version auto-detected)
bpy.ops.bim.load_bcf_project(filepath="/path/to/issues.bcf")

# Check version before version-specific operations
bcfxml = BcfStore.get_bcfxml()
version_id = bcfxml.version.version_id or ""
is_v2 = version_id.startswith("2")

# Use Bonsai operators — they handle version branching internally
bpy.ops.bim.add_bcf_topic()
bpy.ops.bim.add_bcf_comment()
```

---

## Anti-Pattern 6: Omitting Required Fields in BCF v3.0

### WRONG

```python
# Bonsai v0.8.x — WRONG: missing required v3.0 fields
import bpy

props = bpy.context.scene.BCFProperties
props.bcf_version = "3"
bpy.ops.bim.new_bcf_project()

bpy.ops.bim.add_bcf_topic()
topic = props.topics[props.active_topic_index]
topic.title = "Issue without status or type"
# topic.status = ""  # Empty — REQUIRED in v3.0
# topic.type = ""    # Empty — REQUIRED in v3.0
bpy.ops.bim.edit_bcf_topic()  # Returns CANCELLED with error
```

### WHY This Fails

BCF v3.0 schema mandates `status` and `type` fields on topics. The `EditBcfTopic` operator validates these fields and returns `{'CANCELLED'}` with an error message when either is empty in v3.0 mode. In v2.1, these fields are optional, so the same code would succeed — creating a version-dependent bug that only appears when using v3.0.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: set required fields for v3.0
import bpy

props = bpy.context.scene.BCFProperties
props.bcf_version = "3"
bpy.ops.bim.new_bcf_project()

bpy.ops.bim.add_bcf_topic()
topic = props.topics[props.active_topic_index]
topic.title = "Issue with all required fields"
topic.status = "Open"    # REQUIRED in v3.0
topic.type = "Error"     # REQUIRED in v3.0
bpy.ops.bim.edit_bcf_topic()  # Succeeds
```

---

## Anti-Pattern 7: Assuming Viewpoint Coordinates Match Model

### WRONG

```python
# Bonsai v0.8.x — WRONG: direct coordinate comparison
from bonsai.bim.module.bcf.bcfstore import BcfStore

bcfxml = BcfStore.get_bcfxml()
topic = list(bcfxml.topics.values())[0]
viewpoint = list(topic.viewpoints.values())[0]
vizinfo = viewpoint.visualization_info

# Directly use viewpoint camera coordinates
camera_pos = vizinfo.perspective_camera.camera_view_point
# These coordinates may NOT match the loaded IFC model's coordinate system
```

### WHY This Fails

BCF viewpoints store camera coordinates in the coordinate system of the originating model. When viewpoints come from a different BIM tool (Solibri, Navisworks) or a federated model with a different georeference origin, the coordinates do not match the loaded Bonsai model. The `ActivateBcfViewpoint` operator applies georeference offset corrections to account for this difference. Bypassing the operator and reading coordinates directly results in incorrect camera placement.

### CORRECT

```python
# Bonsai v0.8.x — CORRECT: use the operator for coordinate correction
import bpy

# The operator handles georeference offsets automatically
bpy.ops.bim.activate_bcf_viewpoint(viewpoint_guid="abc-123-def")
```

---

## Anti-Pattern 8: Concurrent BCF Access from Multiple Processes

### WRONG

```python
# WRONG: multiple Blender instances accessing same BCF file
# Process 1:
bpy.ops.bim.load_bcf_project(filepath="/shared/coordination.bcf")
bpy.ops.bim.add_bcf_topic()
bpy.ops.bim.save_bcf_project(save_current_bcf=True)

# Process 2 (concurrent):
bpy.ops.bim.load_bcf_project(filepath="/shared/coordination.bcf")  # Stale data
bpy.ops.bim.add_bcf_topic()
bpy.ops.bim.save_bcf_project(save_current_bcf=True)  # Overwrites Process 1's changes
```

### WHY This Fails

`BcfStore` is a class-level singleton within a single Blender process. It does NOT handle concurrent access from multiple processes. Two Blender instances loading and saving the same BCF file cause data races — the last save wins, silently overwriting the other process's changes. Additionally, `get_bcfxml()` silently handles load exceptions (such as file locks), returning `None` without raising an error.

### CORRECT

Use file locking or sequential access patterns. Assign BCF files per-user or per-session, then merge manually:

```python
# Bonsai v0.8.x — CORRECT: per-user BCF files
import bpy
import os

username = os.getenv("USER", "unknown")
filepath = f"/shared/coordination_{username}.bcf"

bpy.ops.bim.load_bcf_project(filepath=filepath)
# ... work with BCF ...
bpy.ops.bim.save_bcf_project(save_current_bcf=True)
# Merge BCF files in a coordination meeting
```

---

## Sources

- Bonsai BCF operator source: `src/bonsai/bonsai/bim/module/bcf/operator.py`
- Bonsai BCF store: `src/bonsai/bonsai/bim/module/bcf/bcfstore.py`
- Bonsai BCF properties: `src/bonsai/bonsai/bim/module/bcf/prop.py`
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
