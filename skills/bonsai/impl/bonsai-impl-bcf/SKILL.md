---
name: bonsai-impl-bcf
description: "Guides implementation of BIM Collaboration Format (BCF) workflows in Bonsai including creating BCF topics, adding viewpoints with camera snapshots, managing comments, importing/exporting BCF files (v2.1 and v3.0), and integrating BCF issue tracking with IFC element references. Activates when working with BCF files, BIM issue tracking, clash report management, or collaboration workflows in Bonsai."
license: MIT
compatibility: "Designed for Claude Code. Requires Blender with Bonsai addon."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# BCF Workflows in Bonsai

> **Version**: Bonsai v0.8.x | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.bim.module.bcf` — NEVER `blenderbim.bim.module.bcf`
> **BCF versions**: v2.1 and v3.0 (via `bcf.bcfxml` library bundled with IfcOpenShell)
> **Dependencies**: bonsai-core-architecture, bonsai-syntax-elements

## Critical Warnings

1. **ALWAYS** set `scene.BCFProperties.author` to a valid email BEFORE creating topics or comments. Default `"john@doe.com"` is a placeholder — replace it.
2. **ALWAYS** use `BcfStore.get_bcfxml()` to access BCF data. NEVER access `BcfStore.bcfxml` directly — `get_bcfxml()` handles lazy initialization and path resolution.
3. **ALWAYS** call `bpy.ops.bim.save_bcf_project()` to persist changes. BCF modifications are in-memory only until saved.
4. **NEVER** mix BCF v2.1 and v3.0 operations on the same project. The version is set at creation/load time and MUST remain consistent.
5. **ALWAYS** ensure an active camera exists before calling `bpy.ops.bim.add_bcf_viewpoint()`. The operator polls for an active scene camera.
6. **NEVER** assume viewpoint coordinates match the loaded IFC model coordinates. Viewpoint activation applies georeference offset corrections.
7. **ALWAYS** use `bonsai.*` imports. NEVER use `blenderbim.*` — renamed in 2024 (v0.8.0+).
8. **BCF v3.0 REQUIRES** non-empty `status` and `type` fields on topics. These are optional in v2.1 but mandatory in v3.0.

---

## Architecture Overview

The BCF module diverges from Bonsai's standard core/tool/operator pattern. At v0.8.x, most domain logic lives directly in operators rather than in a separate `core/bcf.py`. The `tool/bcf.py` provides version-agnostic helper methods for topic data access.

```
+----------------------------------------------------------+
|  DELIVERY LAYER  — bim/module/bcf/                        |
|  operator.py (30+ operators), prop.py, ui.py              |
+----------------------------------------------------------+
|  TOOL LAYER      — tool/bcf.py                            |
|  Version-agnostic getters/setters for topic data          |
+----------------------------------------------------------+
|  DATA LAYER      — bcfstore.py (BcfStore singleton)       |
|  Class-level singleton wrapping bcf.bcfxml.BcfXml         |
+----------------------------------------------------------+
|  LIBRARY         — bcf.v2.bcfxml / bcf.v3.bcfxml          |
|  BCF-XML parser bundled with IfcOpenShell                 |
+----------------------------------------------------------+
```

**Key singleton**: `BcfStore` manages BCF data as a class-level singleton. Only ONE BCF project is loaded at a time. Access via `BcfStore.get_bcfxml()`.

**Properties access**: `bpy.context.scene.BCFProperties` (or `tool.Bcf.get_bcf_props()`) holds all BCF UI state.

---

## Decision Tree: BCF Workflow Selection

```
What BCF operation do you need?
│
├── Start a new BCF project?
│   └── bpy.ops.bim.new_bcf_project()
│       ├── Set props.bcf_version = "2" (v2.1) or "3" (v3.0) FIRST
│       └── Sets up empty BcfXml in BcfStore
│
├── Load an existing BCF file?
│   └── bpy.ops.bim.load_bcf_project(filepath="/path/to/file.bcf")
│       └── Auto-detects version, populates BcfStore + topics
│
├── Create/manage issues?
│   ├── Add topic: bpy.ops.bim.add_bcf_topic()
│   ├── Edit topic: bpy.ops.bim.edit_bcf_topic()
│   ├── View topic: bpy.ops.bim.view_bcf_topic(topic_guid="...")
│   └── Remove topic: bpy.ops.bim.remove_bcf_topic(guid="...")
│
├── Add visual context to an issue?
│   ├── Capture viewpoint: bpy.ops.bim.add_bcf_viewpoint()
│   ├── Restore viewpoint: bpy.ops.bim.activate_bcf_viewpoint()
│   └── Remove viewpoint: bpy.ops.bim.remove_bcf_viewpoint()
│
├── Add discussion to an issue?
│   ├── Add comment: bpy.ops.bim.add_bcf_comment()
│   ├── Edit comment: bpy.ops.bim.edit_bcf_comment()
│   └── Remove comment: bpy.ops.bim.remove_bcf_comment(comment_guid="...")
│
├── Attach documents/references?
│   ├── Reference link: bpy.ops.bim.add_bcf_reference_link()
│   ├── Document ref: bpy.ops.bim.add_bcf_document_reference()
│   ├── BIM snippet: bpy.ops.bim.add_bcf_bim_snippet()
│   └── Header file: bpy.ops.bim.add_bcf_header_file()
│
├── Save BCF to disk?
│   └── bpy.ops.bim.save_bcf_project(filepath="/path/to/output.bcf")
│
└── Close BCF project?
    └── bpy.ops.bim.unload_bcf_project()
```

---

## Essential Patterns

### Pattern 1: BCF Project Lifecycle

```python
# Bonsai v0.8.x — Complete BCF project lifecycle
import bpy

props = bpy.context.scene.BCFProperties

# Step 1: Set author (REQUIRED before any topic/comment creation)
props.author = "architect@example.com"

# Step 2: Choose BCF version and create project
props.bcf_version = "3"  # "2" for v2.1, "3" for v3.0
bpy.ops.bim.new_bcf_project()

# Step 3: Work with topics, viewpoints, comments...
# (see patterns below)

# Step 4: Save to disk
bpy.ops.bim.save_bcf_project(filepath="/path/to/issues.bcf")

# Step 5: Unload when done
bpy.ops.bim.unload_bcf_project()
```

### Pattern 2: Load Existing BCF File

```python
# Bonsai v0.8.x — Load and inspect BCF file
import bpy

# Load BCF file (auto-detects v2.1 or v3.0)
bpy.ops.bim.load_bcf_project(filepath="/path/to/issues.bcf")

# Access loaded data
props = bpy.context.scene.BCFProperties
print(f"Project: {props.name}")
print(f"Version: BCF v{props.bcf_version}")
print(f"Topics: {len(props.topics)}")

# Iterate topics
for topic in props.topics:
    print(f"  [{topic.status}] {topic.title} (GUID: {topic.name})")
```

### Pattern 3: Create Topic with Metadata

```python
# Bonsai v0.8.x — Create and configure a BCF topic
import bpy

props = bpy.context.scene.BCFProperties
props.author = "engineer@example.com"

# Create topic
bpy.ops.bim.add_bcf_topic()

# Get the newly created topic (last in list)
topic = props.topics[props.active_topic_index]

# Set metadata
topic.title = "MEP clash at Level 2 corridor"
topic.type = "Error"
topic.status = "Open"
topic.priority = "Critical"
topic.assigned_to = "mep-coordinator@example.com"
topic.description = "HVAC duct intersects structural beam at grid B-4."

# Persist changes to the BCF data structure
bpy.ops.bim.edit_bcf_topic()
```

### Pattern 4: Capture Viewpoint with Camera Snapshot

```python
# Bonsai v0.8.x — Capture viewpoint for active topic
import bpy

# PREREQUISITE: Active camera MUST exist
if bpy.context.scene.camera is None:
    raise RuntimeError("Set an active camera before capturing a viewpoint.")

# Position camera at the issue location
camera = bpy.context.scene.camera
camera.location = (10.0, -5.0, 3.0)
camera.rotation_euler = (1.22, 0.0, 0.78)

# Capture viewpoint (records camera state + element visibility + snapshot)
bpy.ops.bim.add_bcf_viewpoint()
# This operator:
# 1. Reads camera position/orientation from active scene camera
# 2. Records component visibility and selection state
# 3. Renders a PNG snapshot via bpy.ops.render.opengl()
# 4. Stores clipping planes if present
```

### Pattern 5: Add Comments to a Topic

```python
# Bonsai v0.8.x — Add comments to the active BCF topic
import bpy

props = bpy.context.scene.BCFProperties
props.author = "reviewer@example.com"

# Set comment text
props.comment = "Duct routing needs to be lowered by 200mm to clear beam."

# Optionally link to active viewpoint
props.has_related_viewpoint = True  # Links comment to current viewpoint

# Create comment
bpy.ops.bim.add_bcf_comment()
```

### Pattern 6: Activate (Restore) a Viewpoint

```python
# Bonsai v0.8.x — Restore a saved viewpoint
import bpy

props = bpy.context.scene.BCFProperties
topic = props.active_topic

# Viewpoints are accessed via the topic's viewpoints enum
# Activate the currently selected viewpoint
bpy.ops.bim.activate_bcf_viewpoint()
# This restores:
# - Camera position and orientation
# - Component visibility/selection state
# - Clipping planes
# - Applies georeference offset correction

# To activate a specific viewpoint by GUID:
bpy.ops.bim.activate_bcf_viewpoint(viewpoint_guid="abc-123-def-456")
```

---

## BCF Version Differences

| Feature | BCF v2.1 | BCF v3.0 |
|---------|----------|----------|
| `props.bcf_version` value | `"2"` | `"3"` |
| Topic `status` field | Optional | **REQUIRED** (edit fails without it) |
| Topic `type` field | Optional | **REQUIRED** (edit fails without it) |
| Document storage | Per-topic (`TopicDocumentReference`) | Global registry (`bcfxml.documents`) |
| API support | No | Yes (OpenCDE compliant) |
| Data model module | `bcf.v2.model` | `bcf.v3.model` |
| BcfXml class | `bcf.v2.bcfxml.BcfXml` | `bcf.v3.bcfxml.BcfXml` |

### Version Detection

```python
# Bonsai v0.8.x — Detecting BCF version at runtime
from bonsai.bim.module.bcf.bcfstore import BcfStore

bcfxml = BcfStore.get_bcfxml()
if bcfxml is None:
    raise RuntimeError("No BCF project loaded.")

version_id = bcfxml.version.version_id or ""
is_v2 = version_id.startswith("2")
is_v3 = version_id.startswith("3")
```

---

## Common Operations

### Import BCF from External Tool

```python
# Bonsai v0.8.x — Load BCF from Solibri, Navisworks, etc.
import bpy

# Load external BCF file
bpy.ops.bim.load_bcf_project(filepath="/path/to/coordination.bcf")

props = bpy.context.scene.BCFProperties

# Iterate all topics
for i, topic in enumerate(props.topics):
    print(f"Topic {i}: [{topic.status}] {topic.title}")

    # View topic to load its details
    bpy.ops.bim.view_bcf_topic(topic_guid=topic.name)

    # Load comments for this topic
    bpy.ops.bim.load_bcf_comments(topic_guid=topic.name)
    for comment in topic.comments:
        print(f"  Comment by {comment.author}: {comment.comment}")
```

### Export BCF for External Tools

```python
# Bonsai v0.8.x — Save BCF for sharing with other BIM tools
import bpy

# Save current BCF project
bpy.ops.bim.save_bcf_project(filepath="/path/to/export.bcf")

# Save to same path (overwrite)
bpy.ops.bim.save_bcf_project(save_current_bcf=True)
```

### Link BCF Topic to IFC Elements

```python
# Bonsai v0.8.x — Reference IFC elements in BCF viewpoint
import bpy
import bonsai.tool as tool

model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

# Select the clashing elements in Blender viewport
bpy.ops.object.select_all(action='DESELECT')

# Find elements by GlobalId and select them
for global_id in ["2O2Fr$t4X7Zf8NOew3FLOH", "1kTvXnbbzEWu00n_qpFOat"]:
    entity = model.by_guid(global_id)
    obj = tool.Ifc.get_object(entity)
    if obj:
        obj.select_set(True)

# Capture viewpoint — selected elements are recorded as BCF components
bpy.ops.bim.add_bcf_viewpoint()
```

### Clash Detection to BCF Workflow

```python
# Bonsai v0.8.x — Convert clash results to BCF issues
import bpy

# Step 1: Run clash detection with BCF output
bpy.ops.bim.execute_ifc_clash()
# Save output as .bcf file when prompted

# Step 2: Load the clash BCF into BCF panel
bpy.ops.bim.load_bcf_project(filepath="/path/to/clashes.bcf")

# Each clash becomes a BCF topic with:
# - Viewpoint at clash location
# - Component references for both clashing elements
# - Metadata about clash type and severity
```

### Attach Documents and References

```python
# Bonsai v0.8.x — Add reference links and documents to a topic
import bpy

props = bpy.context.scene.BCFProperties

# Add a reference link (URL)
props.reference_link = "https://project.example.com/issues/MEP-042"
bpy.ops.bim.add_bcf_reference_link()

# Add a document reference
props.document_reference = "/path/to/specification.pdf"
props.document_reference_description = "MEP specification section 4.2"
bpy.ops.bim.add_bcf_document_reference()

# Add a BIM snippet (IFC fragment for context)
props.bim_snippet_reference = "/path/to/fragment.ifc"
props.bim_snippet_type = "IfcSharedBuildingElements"
props.bim_snippet_schema = "IFC4"
bpy.ops.bim.add_bcf_bim_snippet()
```

### Add Labels and Related Topics

```python
# Bonsai v0.8.x — Organize topics with labels and relations
import bpy

props = bpy.context.scene.BCFProperties

# Add labels to active topic
props.label = "MEP"
bpy.ops.bim.add_bcf_label()
props.label = "Structural"
bpy.ops.bim.add_bcf_label()

# Add related topic (requires another topic to exist)
# related_topic is an EnumProperty listing other topics
bpy.ops.bim.add_bcf_related_topic()
```

### Extract Embedded Files

```python
# Bonsai v0.8.x — Extract files embedded in BCF topics
import bpy

# Extract a header file by index
bpy.ops.bim.extract_bcf_file(entity_type="HEADER_FILE", index=0)

# Extract BIM snippet
bpy.ops.bim.extract_bcf_file(entity_type="BIM_SNIPPET", index=0)

# Extract document reference
bpy.ops.bim.extract_bcf_file(entity_type="DOCUMENT_REFERENCE", index=0)
```

---

## BcfStore Singleton

The `BcfStore` class (at `bonsai.bim.module.bcf.bcfstore`) manages BCF session state:

```python
from bonsai.bim.module.bcf.bcfstore import BcfStore

# Get the active BCF project (lazy-loads from props.bcf_file)
bcfxml = BcfStore.get_bcfxml()

# Set BCF data explicitly
BcfStore.set(bcfxml, filepath)

# Load by filepath (triggers lazy load on next get_bcfxml())
BcfStore.set_by_filepath("/path/to/file.bcf")

# Unload BCF session
BcfStore.unload_bcfxml()
```

**Key behavior**:
- `get_bcfxml()` calls `bcf.bcfxml.load(filepath)` from `props.bcf_file` on first access
- `set()` auto-detects BCF version from `bcfxml.version.version_id` and updates `props.bcf_version`
- Silently handles load exceptions (e.g., permission denied) — check return value for `None`
- Resolves relative paths against the `.blend` file directory

---

## BCFProperties Quick Reference

Access via `bpy.context.scene.BCFProperties`:

| Property | Type | Purpose |
|----------|------|---------|
| `author` | `str` | Author email — REQUIRED before creating topics/comments |
| `bcf_version` | `enum` | `"2"` (v2.1) or `"3"` (v3.0) |
| `bcf_file` | `str` | Path to loaded BCF file |
| `name` | `str` | Project name |
| `topics` | `Collection[BcfTopic]` | All loaded topics |
| `active_topic_index` | `int` | Currently selected topic index |
| `comment` | `str` | Input field for new comment text |
| `has_related_viewpoint` | `bool` | Link new comment to active viewpoint |
| `reference_link` | `str` | Input field for reference link URL |
| `label` | `str` | Input field for label text |

**Computed property**: `props.active_topic` returns `BcfTopic` at `active_topic_index` or `None`.

---

## Drag-and-Drop Support

Bonsai registers `BCFFileHandlerOperator` with `bl_file_extensions = ".bcf"`, enabling drag-and-drop import of BCF files directly into Blender. The handler calls `bim.load_bcf_project` with the dropped file path.

---

## Reference Links

- [BCF API Methods](references/methods.md) — Complete operator signatures and BcfStore API
- [Working Examples](references/examples.md) — End-to-end BCF workflow examples
- [Anti-Patterns](references/anti-patterns.md) — What NOT to do when implementing BCF workflows

## Dependencies

- **bonsai-core-architecture** — Three-layer architecture, IfcStore, operator execution pattern
- **bonsai-syntax-elements** — IFC element access for linking BCF issues to IFC entities

## Sources

- Bonsai BCF source: `src/bonsai/bonsai/bim/module/bcf/` (operator.py, prop.py, bcfstore.py)
- Bonsai BCF tool: `src/bonsai/bonsai/tool/bcf.py`
- BCF library: `bcf.v2.bcfxml` / `bcf.v3.bcfxml` (bundled with IfcOpenShell)
- Bonsai Documentation: https://docs.bonsaibim.org
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell
