# BCF Workflow Examples

> **Version**: Bonsai v0.8.x | BCF v2.1 + v3.0
> All examples run inside Blender with Bonsai addon active (`blender --python script.py`).

---

## Example 1: Complete BCF Issue Creation Workflow

Create a new BCF project, add a topic with viewpoint and comments, then save.

```python
# Bonsai v0.8.x — Complete BCF issue creation
import bpy

# ---- Step 1: Configure BCF project ----
props = bpy.context.scene.BCFProperties
props.author = "architect@example.com"
props.bcf_version = "3"  # Use BCF v3.0

# Create new project
bpy.ops.bim.new_bcf_project()

# ---- Step 2: Create a topic ----
bpy.ops.bim.add_bcf_topic()

# Configure the topic
topic = props.topics[props.active_topic_index]
topic.title = "Structural beam clashes with HVAC duct at Level 2"
topic.type = "Error"
topic.status = "Open"
topic.priority = "Critical"
topic.assigned_to = "structural@example.com"
topic.description = (
    "Steel beam W12x26 at grid intersection B-4 intersects "
    "HVAC supply duct. Duct routing needs adjustment."
)

# Persist topic edits
bpy.ops.bim.edit_bcf_topic()

# ---- Step 3: Add labels ----
props.label = "Structural"
bpy.ops.bim.add_bcf_label()
props.label = "MEP"
bpy.ops.bim.add_bcf_label()

# ---- Step 4: Capture viewpoint ----
# Ensure camera is positioned at the issue location
camera = bpy.context.scene.camera
if camera is None:
    bpy.ops.object.camera_add(location=(10.0, -5.0, 3.5))
    bpy.context.scene.camera = bpy.context.active_object
    camera = bpy.context.scene.camera

camera.location = (10.0, -5.0, 3.5)
camera.rotation_euler = (1.22, 0.0, 0.78)

# Capture viewpoint (includes snapshot PNG)
bpy.ops.bim.add_bcf_viewpoint()

# ---- Step 5: Add comments ----
props.comment = "Initial report: duct must be lowered 200mm to clear beam."
props.has_related_viewpoint = True
bpy.ops.bim.add_bcf_comment()

props.comment = "Coordination meeting scheduled for 2026-03-10."
props.has_related_viewpoint = False
bpy.ops.bim.add_bcf_comment()

# ---- Step 6: Add reference link ----
props.reference_link = "https://project.example.com/issues/STRUCT-042"
bpy.ops.bim.add_bcf_reference_link()

# ---- Step 7: Save BCF file ----
bpy.ops.bim.save_bcf_project(filepath="/tmp/coordination_issues.bcf")

print("BCF project saved with 1 topic, 1 viewpoint, and 2 comments.")
```

---

## Example 2: Load and Review BCF from External Tool

Import a BCF file from Solibri/Navisworks and iterate through issues.

```python
# Bonsai v0.8.x — Load and review external BCF
import bpy

# Load BCF file (auto-detects version)
bpy.ops.bim.load_bcf_project(filepath="/path/to/coordination.bcf")

props = bpy.context.scene.BCFProperties
print(f"Project: {props.name}")
print(f"BCF Version: v{props.bcf_version}")
print(f"Total topics: {len(props.topics)}")
print("---")

# Review all topics
for i, topic in enumerate(props.topics):
    print(f"\nTopic {i + 1}: {topic.title}")
    print(f"  GUID:     {topic.name}")
    print(f"  Status:   {topic.status}")
    print(f"  Type:     {topic.type}")
    print(f"  Priority: {topic.priority}")
    print(f"  Assigned: {topic.assigned_to}")
    print(f"  Created:  {topic.creation_date} by {topic.creation_author}")

    # Load comments for this topic
    bpy.ops.bim.view_bcf_topic(topic_guid=topic.name)
    bpy.ops.bim.load_bcf_comments(topic_guid=topic.name)

    for comment in topic.comments:
        print(f"  Comment ({comment.date}): {comment.comment}")
        print(f"    Author: {comment.author}")
        if comment.viewpoint:
            print(f"    Viewpoint: {comment.viewpoint}")
```

---

## Example 3: Navigate Viewpoints

Restore saved viewpoints to review issue locations in the 3D viewport.

```python
# Bonsai v0.8.x — Navigate BCF viewpoints
import bpy

props = bpy.context.scene.BCFProperties

# Ensure a topic is selected
if not props.topics:
    raise RuntimeError("No BCF topics loaded.")

topic = props.active_topic
if topic is None:
    bpy.ops.bim.view_bcf_topic(topic_guid=props.topics[0].name)
    topic = props.active_topic

# Activate the currently selected viewpoint
# This restores camera position, element visibility, and clipping
bpy.ops.bim.activate_bcf_viewpoint()

# To cycle through viewpoints programmatically:
# The viewpoints property is an EnumProperty — iterate via BCF library
from bonsai.bim.module.bcf.bcfstore import BcfStore

bcfxml = BcfStore.get_bcfxml()
if bcfxml is not None:
    bcf_topic = bcfxml.topics.get(topic.name)
    if bcf_topic:
        viewpoints = bcf_topic.viewpoints
        for vp_guid, vp in viewpoints.items():
            print(f"Viewpoint: {vp_guid}")
            bpy.ops.bim.activate_bcf_viewpoint(viewpoint_guid=vp_guid)
```

---

## Example 4: Link BCF Topics to IFC Elements

Select IFC elements and create viewpoints that reference them.

```python
# Bonsai v0.8.x — Link IFC elements to BCF viewpoint
import bpy
import bonsai.tool as tool

# Verify IFC model is loaded
model = tool.Ifc.get()
if model is None:
    raise RuntimeError("No IFC project loaded.")

props = bpy.context.scene.BCFProperties
props.author = "coordinator@example.com"

# Deselect all objects first
bpy.ops.object.select_all(action='DESELECT')

# Select specific IFC elements by GlobalId
target_ids = [
    "2O2Fr$t4X7Zf8NOew3FLOH",  # Structural beam
    "1kTvXnbbzEWu00n_qpFOat",  # HVAC duct
]

for global_id in target_ids:
    try:
        entity = model.by_guid(global_id)
        obj = tool.Ifc.get_object(entity)
        if obj:
            obj.select_set(True)
            print(f"Selected: {entity.is_a()} — {entity.Name}")
    except RuntimeError:
        print(f"Element not found: {global_id}")

# Ensure camera is set
if bpy.context.scene.camera is None:
    raise RuntimeError("Set an active camera before capturing viewpoint.")

# Capture viewpoint — selected elements are recorded as BCF components
bpy.ops.bim.add_bcf_viewpoint()
# The viewpoint stores:
# - Selected elements (GlobalIds) → component selection
# - Visible elements → component visibility
# - Camera position → visualization info
```

---

## Example 5: BCF v2.1 vs v3.0 Project Creation

Create BCF projects in both versions.

```python
# Bonsai v0.8.x — BCF version-specific creation

import bpy

props = bpy.context.scene.BCFProperties
props.author = "engineer@example.com"

# ---- BCF v2.1 project ----
props.bcf_version = "2"
bpy.ops.bim.new_bcf_project()
bpy.ops.bim.add_bcf_topic()

topic_v2 = props.topics[props.active_topic_index]
topic_v2.title = "V2.1 issue — status and type are optional"
# In v2.1, status and type fields are optional
bpy.ops.bim.edit_bcf_topic()

bpy.ops.bim.save_bcf_project(filepath="/tmp/issues_v2.bcf")
bpy.ops.bim.unload_bcf_project()

# ---- BCF v3.0 project ----
props.bcf_version = "3"
bpy.ops.bim.new_bcf_project()
bpy.ops.bim.add_bcf_topic()

topic_v3 = props.topics[props.active_topic_index]
topic_v3.title = "V3.0 issue — status and type REQUIRED"
topic_v3.status = "Open"   # REQUIRED in v3.0
topic_v3.type = "Error"    # REQUIRED in v3.0
bpy.ops.bim.edit_bcf_topic()

bpy.ops.bim.save_bcf_project(filepath="/tmp/issues_v3.bcf")
bpy.ops.bim.unload_bcf_project()
```

---

## Example 6: Batch Topic Creation from Clash Results

Create multiple BCF topics from a list of clash data.

```python
# Bonsai v0.8.x — Batch BCF topic creation from clash data
import bpy

props = bpy.context.scene.BCFProperties
props.author = "clash-coordinator@example.com"
props.bcf_version = "3"
bpy.ops.bim.new_bcf_project()

# Clash data (from external analysis or manual list)
clashes = [
    {
        "title": "Pipe penetrates floor slab at grid A-3",
        "type": "Error",
        "priority": "Critical",
        "assigned_to": "mep@example.com",
        "description": "DN150 hot water pipe passes through 200mm concrete slab without sleeve.",
    },
    {
        "title": "Cable tray conflicts with beam flange at B-7",
        "type": "Warning",
        "priority": "Normal",
        "assigned_to": "electrical@example.com",
        "description": "Cable tray 300mm wide needs routing adjustment.",
    },
    {
        "title": "Clearance violation: sprinkler head too close to wall",
        "type": "Warning",
        "priority": "Normal",
        "assigned_to": "fire-safety@example.com",
        "description": "Sprinkler head within 100mm of partition wall. Minimum 150mm required.",
    },
]

for clash in clashes:
    bpy.ops.bim.add_bcf_topic()
    topic = props.topics[props.active_topic_index]
    topic.title = clash["title"]
    topic.type = clash["type"]
    topic.status = "Open"
    topic.priority = clash["priority"]
    topic.assigned_to = clash["assigned_to"]
    topic.description = clash["description"]
    bpy.ops.bim.edit_bcf_topic()

bpy.ops.bim.save_bcf_project(filepath="/tmp/clash_report.bcf")
print(f"Created {len(clashes)} BCF topics.")
```

---

## Example 7: Add Header File Reference

Reference the source IFC file in a BCF topic header.

```python
# Bonsai v0.8.x — Add IFC file reference to BCF topic header
import bpy

props = bpy.context.scene.BCFProperties

# Set header file details
props.file_reference = "/path/to/model.ifc"
props.file_ifc_project = "0YvctVUKr0kugbFTf53O9L"  # IfcProject GlobalId
props.file_ifc_spatial_structure_element = "3$cshxZO95gv7Mu_048wUj"  # IfcSite GlobalId

# Add header file to active topic
bpy.ops.bim.add_bcf_header_file()

# Load IFC file from BCF header (opens the referenced IFC in Bonsai)
bpy.ops.bim.load_bcf_header_ifc_file(index=0)
```

---

## Example 8: Comment Workflow with Viewpoint Links

Create a discussion thread with viewpoint-linked and standalone comments.

```python
# Bonsai v0.8.x — Comment thread with viewpoint references
import bpy

props = bpy.context.scene.BCFProperties
props.author = "reviewer@example.com"

# Ensure active topic exists
if props.active_topic is None:
    raise RuntimeError("Select a BCF topic first.")

# Comment 1: Linked to a viewpoint (visual reference)
# First, capture a viewpoint
if bpy.context.scene.camera:
    bpy.ops.bim.add_bcf_viewpoint()

    # Now add comment linked to the viewpoint
    props.comment = "See attached viewpoint — duct routing conflicts with beam."
    props.has_related_viewpoint = True
    bpy.ops.bim.add_bcf_comment()

# Comment 2: Standalone (no viewpoint)
props.comment = "Proposed solution: lower duct by 200mm and add transition piece."
props.has_related_viewpoint = False
bpy.ops.bim.add_bcf_comment()

# Comment 3: Resolution note
props.author = "mep-engineer@example.com"
props.comment = "Duct rerouted in revision 4. Ready for re-check."
props.has_related_viewpoint = False
bpy.ops.bim.add_bcf_comment()

# Save changes
bpy.ops.bim.save_bcf_project(save_current_bcf=True)
```

---

## Example 9: Extract Embedded Files from BCF

Extract BIM snippets, header files, and document references from a loaded BCF.

```python
# Bonsai v0.8.x — Extract embedded BCF files
import bpy

# Load BCF with embedded files
bpy.ops.bim.load_bcf_project(filepath="/path/to/issues_with_attachments.bcf")

props = bpy.context.scene.BCFProperties

# View first topic
if props.topics:
    bpy.ops.bim.view_bcf_topic(topic_guid=props.topics[0].name)

    topic = props.active_topic

    # Extract header files
    for i, f in enumerate(topic.files):
        bpy.ops.bim.extract_bcf_file(entity_type="HEADER_FILE", index=i)
        print(f"Extracted header file {i}: {f.name}")

    # Extract BIM snippet (if present)
    if topic.bim_snippet.reference:
        bpy.ops.bim.extract_bcf_file(entity_type="BIM_SNIPPET", index=0)
        print("Extracted BIM snippet")

    # Extract document references
    for i, doc in enumerate(topic.document_references):
        if not doc.is_external:
            bpy.ops.bim.extract_bcf_file(entity_type="DOCUMENT_REFERENCE", index=i)
            print(f"Extracted document: {doc.description}")
```

---

## Example 10: Unload and Cleanup

Properly close a BCF session.

```python
# Bonsai v0.8.x — Clean BCF session shutdown
import bpy
from bonsai.bim.module.bcf.bcfstore import BcfStore

# Check if a BCF project is loaded
bcfxml = BcfStore.get_bcfxml()
if bcfxml is not None:
    # Save if needed
    bpy.ops.bim.save_bcf_project(save_current_bcf=True)

    # Unload project (clears BcfStore and props)
    bpy.ops.bim.unload_bcf_project()

    # Verify cleanup
    assert BcfStore.get_bcfxml() is None
    print("BCF project unloaded.")
else:
    print("No BCF project loaded.")
```

---

## Sources

- Bonsai BCF operator source: `src/bonsai/bonsai/bim/module/bcf/operator.py`
- Bonsai BCF properties: `src/bonsai/bonsai/bim/module/bcf/prop.py`
- Bonsai BCF store: `src/bonsai/bonsai/bim/module/bcf/bcfstore.py`
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
