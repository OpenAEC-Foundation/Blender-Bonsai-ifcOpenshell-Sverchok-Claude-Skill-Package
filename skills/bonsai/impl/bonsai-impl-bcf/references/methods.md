# BCF API Methods Reference

> **Version**: Bonsai v0.8.x | BCF v2.1 + v3.0
> **Module**: `bonsai.bim.module.bcf`

## BcfStore — BCF Session Manager

**Location**: `bonsai.bim.module.bcf.bcfstore`

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `get_bcfxml` | `@classmethod get_bcfxml(cls)` | `bcf.bcfxml.BcfXml \| None` | Lazy-loads BCF from `props.bcf_file`. Returns cached instance on subsequent calls. |
| `set` | `@classmethod set(cls, bcfxml, filepath)` | `None` | Store BCF data and filepath. Auto-detects version and sets `props.bcf_version`. |
| `set_by_filepath` | `@classmethod set_by_filepath(cls, filepath)` | `None` | Convenience: calls `set(None, filepath)`. Triggers lazy load on next `get_bcfxml()`. |
| `unload_bcfxml` | `@classmethod unload_bcfxml(cls)` | `None` | Calls `set(None, "")` to clear session. |

**Class variable**: `BcfStore.bcfxml: Union[bcf.bcfxml.BcfXml, None]` — NEVER access directly; ALWAYS use `get_bcfxml()`.

---

## tool.Bcf — Version-Agnostic BCF Helpers

**Location**: `bonsai.tool.bcf`

### Properties Access

| Method | Signature | Returns |
|--------|-----------|---------|
| `get_bcf_props` | `@classmethod get_bcf_props(cls)` | `BCFProperties` (`bpy.context.scene.BCFProperties`) |
| `get_path` | `@classmethod get_path(cls)` | `str` (filepath from `props.bcf_file`) |
| `is_list_of` | `@classmethod is_list_of(cls, a: list, t: type)` | `bool` (type guard for list element type) |

### Topic Data Methods

| Method | Signature | Returns |
|--------|-----------|---------|
| `set_topic_bim_snippet` | `(cls, topic: TopicHandler, bim_snippet: BimSnippet \| None, bim_snippet_bytes: bytes \| None)` | `None` |
| `get_topic_document_references` | `(cls, topic: TopicHandler)` | `list[v2.TopicDocumentReference] \| list[v3.DocumentReference]` |
| `set_topic_document_references` | `(cls, topic: TopicHandler, document_references: list)` | `None` |
| `get_topic_header_files` | `(cls, topic: TopicHandler)` | `list[v2.HeaderFile] \| list[v3.File]` |
| `set_topic_header_files` | `(cls, topic: TopicHandler, files: list)` | `None` |
| `get_topic_labels` | `(cls, topic: TopicHandler)` | `list[str]` |
| `set_topic_labels` | `(cls, topic: TopicHandler, labels: list[str])` | `None` |
| `get_topic_reference_links` | `(cls, topic: TopicHandler)` | `list[str]` |
| `set_topic_reference_links` | `(cls, topic: TopicHandler, reference_links: list[str])` | `None` |
| `get_topic_related_topics` | `(cls, topic: TopicHandler)` | `list[v2.TopicRelatedTopic] \| list[v3.TopicRelatedTopicsRelatedTopic]` |
| `set_topic_related_topics` | `(cls, topic: TopicHandler, related_topics: list)` | `None` |
| `get_topic_viewpoints` | `(cls, topic: TopicHandler)` | `list[v2.ViewPoint] \| list[v3.ViewPoint]` |
| `set_topic_viewpoints` | `(cls, topic: TopicHandler, viewpoints: list)` | `None` |

### Viewpoint Data Methods

| Method | Signature | Returns |
|--------|-----------|---------|
| `get_viewpoint_bitmaps` | `(cls, viewpoint: VisualizationInfoHandler)` | `list[v2.VisualizationInfoBitmap] \| list[v3.Bitmap]` |
| `get_viewpoint_view_setup_hints` | `(cls, viewpoint: VisualizationInfoHandler)` | `v2.ViewSetupHints \| v3.ViewSetupHints \| None` |

---

## BCF Operators — Project Management

| Class | `bl_idname` | Properties | Description |
|-------|-------------|------------|-------------|
| `NewBcfProject` | `bim.new_bcf_project` | — | Create new BCF project. Reads `props.bcf_version` (`"2"` or `"3"`). |
| `LoadBcfProject` | `bim.load_bcf_project` | `filepath: StringProperty(subtype="FILE_PATH")`, `filter_glob: StringProperty(default="*.bcf;*.bcfzip")` | Load BCF file from disk. Auto-detects version. |
| `UnloadBcfProject` | `bim.unload_bcf_project` | — | Unload current BCF project, clear BcfStore. |
| `SaveBcfProject` | `bim.save_bcf_project` | `filepath: StringProperty(subtype="FILE_PATH")`, `save_current_bcf: BoolProperty(default=False)` | Save BCF to file. `save_current_bcf=True` overwrites current path. |
| `EditBcfProjectName` | `bim.edit_bcf_project_name` | — | Rename project from `props.name`. |
| `BCFFileHandlerOperator` | `bim.load_bcf_project_file_handler` | `directory: StringProperty`, `files: CollectionProperty` | Drag-and-drop handler for `.bcf` files. |

---

## BCF Operators — Topic Management

| Class | `bl_idname` | Properties | Description |
|-------|-------------|------------|-------------|
| `LoadBcfTopics` | `bim.load_bcf_topics` | — | Load all topics from current project into `props.topics`. |
| `LoadBcfTopic` | `bim.load_bcf_topic` | `topic_guid: StringProperty()`, `topic_index: IntProperty()` | Load single topic by GUID. |
| `AddBcfTopic` | `bim.add_bcf_topic` | — | Create new topic. REQUIRES `props.author` set. Calls `bcfxml.add_topic("New Topic", "", author)`. |
| `RemoveBcfTopic` | `bim.remove_bcf_topic` | `guid: StringProperty()` | Delete topic by GUID. |
| `EditBcfTopic` | `bim.edit_bcf_topic` | — | Persist topic field edits. BCF v3: fails if `status` or `type` is empty. |
| `ViewBcfTopic` | `bim.view_bcf_topic` | `topic_guid: StringProperty()` | Set active topic, load its details. |

---

## BCF Operators — Viewpoint Management

| Class | `bl_idname` | Properties | Description |
|-------|-------------|------------|-------------|
| `AddBcfViewpoint` | `bim.add_bcf_viewpoint` | — | Capture camera position, element visibility/selection, render PNG snapshot. REQUIRES active scene camera. |
| `RemoveBcfViewpoint` | `bim.remove_bcf_viewpoint` | — | Remove active viewpoint from topic. |
| `ActivateBcfViewpoint` | `bim.activate_bcf_viewpoint` | `viewpoint_guid: StringProperty(default="")` | Restore camera, visibility, clipping. Empty GUID uses active viewpoint enum. |

### AddBcfViewpoint Internal Process

1. Read camera position and orientation from `scene.camera`
2. Build `VisualizationInfo` with perspective/orthographic camera data
3. Record selected elements via `vizinfo.set_selected_elements(elements)` — elements from `tool.Ifc.get_entity(obj)` for each selected Blender object
4. Record visible elements via `vizinfo.set_visible_elements(elements)`
5. Capture clipping planes if present
6. Render PNG snapshot via `bpy.ops.render.opengl(write_still=True)` to temp file
7. Store snapshot bytes in viewpoint

### ActivateBcfViewpoint Internal Process

1. Read `VisualizationInfo` from viewpoint
2. Set camera position/orientation (creates camera if needed)
3. Apply component visibility (show/hide elements by GlobalId)
4. Apply component selection
5. Apply component coloring
6. Set clipping planes
7. Apply georeference offset correction between BCF coordinate system and loaded IFC model

---

## BCF Operators — Comment Management

| Class | `bl_idname` | Properties | Description |
|-------|-------------|------------|-------------|
| `LoadBcfComments` | `bim.load_bcf_comments` | `topic_guid: StringProperty()` | Populate `topic.comments` collection for active topic. |
| `AddBcfComment` | `bim.add_bcf_comment` | `comment_guid: StringProperty()` | Create comment. REQUIRES `props.author` and `props.comment`. Optionally links to viewpoint via `props.has_related_viewpoint`. |
| `EditBcfComment` | `bim.edit_bcf_comment` | `comment_guid: StringProperty()` | Update comment text. Sets `modified_date` and `modified_author`. |
| `RemoveBcfComment` | `bim.remove_bcf_comment` | `comment_guid: StringProperty()` | Delete comment by GUID. |

---

## BCF Operators — Metadata & Documents

| Class | `bl_idname` | Properties | Description |
|-------|-------------|------------|-------------|
| `AddBcfReferenceLink` | `bim.add_bcf_reference_link` | — | Add URL from `props.reference_link`. |
| `RemoveBcfReferenceLink` | `bim.remove_bcf_reference_link` | `index: IntProperty()` | Remove reference link by index. |
| `EditBcfReferenceLinks` | `bim.edit_bcf_reference_links` | — | Persist reference link edits. |
| `OpenBcfReferenceLink` | `bim.open_bcf_reference_link` | `index: IntProperty()` | Open URL in system browser. |
| `AddBcfLabel` | `bim.add_bcf_label` | — | Add label from `props.label`. |
| `RemoveBcfLabel` | `bim.remove_bcf_label` | `index: IntProperty()` | Remove label by index. |
| `EditBcfLabels` | `bim.edit_bcf_labels` | — | Persist label edits. |
| `AddBcfDocumentReference` | `bim.add_bcf_document_reference` | — | Add document ref from `props.document_reference`. |
| `RemoveBcfDocumentReference` | `bim.remove_bcf_document_reference` | `index: IntProperty()` | Remove document reference by index. |
| `AddBcfBimSnippet` | `bim.add_bcf_bim_snippet` | — | Add BIM snippet from `props.bim_snippet_*`. |
| `RemoveBcfBimSnippet` | `bim.remove_bcf_bim_snippet` | — | Remove BIM snippet. |
| `AddBcfRelatedTopic` | `bim.add_bcf_related_topic` | — | Add related topic from `props.related_topic` enum. |
| `RemoveBcfRelatedTopic` | `bim.remove_bcf_related_topic` | `index: IntProperty()` | Remove related topic by index. |
| `AddBcfHeaderFile` | `bim.add_bcf_header_file` | — | Add header file from `props.file_reference`. |
| `RemoveBcfFile` | `bim.remove_bcf_file` | `index: IntProperty()` | Remove header file by index. |
| `SelectBcfHeaderFile` | `bim.select_bcf_header_file` | `filter_glob: StringProperty(default="*.ifc;*.ifczip;*.ifcxml;*.ifcjson")` | File picker for header IFC reference. |
| `SelectBcfBimSnippetReference` | `bim.select_bcf_bim_snippet_reference` | — | File picker for BIM snippet. |
| `SelectBcfDocumentReference` | `bim.select_bcf_document_reference` | — | File picker for document reference. |
| `LoadBcfHeaderIfcFile` | `bim.load_bcf_header_ifc_file` | `index: IntProperty()` | Load IFC file referenced in BCF header. |
| `ExtractBcfFile` | `bim.extract_bcf_file` | `entity_type: StringProperty()`, `index: IntProperty()` | Extract embedded file. `entity_type`: `"HEADER_FILE"`, `"BIM_SNIPPET"`, `"DOCUMENT_REFERENCE"`. |

---

## BCFProperties — Scene-Level Property Group

**Access**: `bpy.context.scene.BCFProperties`

### Input/Configuration Properties

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `bcf_file` | `StringProperty` | `""` | Path to loaded BCF file |
| `bcf_version` | `EnumProperty` | `"3"` | `"2"` = v2.1, `"3"` = v3.0 |
| `name` | `StringProperty` | `""` | Project name (update callback syncs to bcfxml) |
| `author` | `StringProperty` | `"john@doe.com"` | Author email — REQUIRED for topics/comments |
| `comment_text_width` | `IntProperty` | `40` | Text wrapping width in UI |

### Topic Input Properties

| Property | Type | Purpose |
|----------|------|---------|
| `file_reference` | `StringProperty` | Input for header file path |
| `file_ifc_project` | `StringProperty` | IFC project reference for header |
| `file_ifc_spatial_structure_element` | `StringProperty` | Spatial structure reference for header |
| `reference_link` | `StringProperty` | Input for reference URL |
| `label` | `StringProperty` | Input for label text |
| `bim_snippet_reference` | `StringProperty` | Input for BIM snippet file path |
| `bim_snippet_type` | `StringProperty` | BIM snippet type identifier |
| `bim_snippet_schema` | `StringProperty` | BIM snippet IFC schema |
| `document_reference` | `StringProperty` | Input for document reference path |
| `document_reference_description` | `StringProperty` | Document reference description |
| `document_description` | `StringProperty` | Document description (v3) |
| `related_topic` | `EnumProperty` | Selection enum for related topics |
| `comment` | `StringProperty` | Input for new comment text |
| `has_related_viewpoint` | `BoolProperty` | Link new comment to active viewpoint |

### Collection Properties

| Property | Type | Purpose |
|----------|------|---------|
| `topics` | `CollectionProperty(BcfTopic)` | All loaded topics |
| `active_topic_index` | `IntProperty` | Currently selected topic (update callback refreshes UI) |

### Computed Properties

| Property | Returns | Description |
|----------|---------|-------------|
| `active_topic` | `BcfTopic \| None` | Returns `topics[active_topic_index]` or `None` if empty |

---

## BcfTopic — Topic Property Group

| Property | Type | Description |
|----------|------|-------------|
| `name` | `StringProperty` | Topic GUID |
| `title` | `StringProperty` | Topic title (update callback syncs to bcfxml) |
| `type` | `StringProperty` | Issue type (search from project `topic_types` extensions) |
| `status` | `StringProperty` | Workflow status (search from project `topic_statuses` extensions) |
| `priority` | `StringProperty` | Priority level (search from project `priorities` extensions) |
| `stage` | `StringProperty` | Project stage (search from project `stages` extensions) |
| `creation_date` | `StringProperty` | ISO timestamp |
| `creation_author` | `StringProperty` | Author email |
| `modified_date` | `StringProperty` | Last modification timestamp |
| `modified_author` | `StringProperty` | Last modifier email |
| `assigned_to` | `StringProperty` | Assignee email (search from project `users` extensions) |
| `due_date` | `StringProperty` | Due date |
| `description` | `StringProperty` | Detailed description |
| `viewpoints` | `EnumProperty` | Viewpoint selector (dynamically populated) |
| `files` | `CollectionProperty(StrProperty)` | Header file list |
| `reference_links` | `CollectionProperty(BcfReferenceLink)` | Reference URLs |
| `labels` | `CollectionProperty(BcfLabel)` | Labels/tags |
| `bim_snippet` | `PointerProperty(BcfBimSnippet)` | BIM snippet reference |
| `document_references` | `CollectionProperty(BcfDocumentReference)` | Document references |
| `related_topics` | `CollectionProperty(StrProperty)` | Related topic GUIDs |
| `comments` | `CollectionProperty(BcfComment)` | Comments list |
| `is_editable` | `BoolProperty` | UI edit mode flag |

---

## BcfComment — Comment Property Group

| Property | Type | Description |
|----------|------|-------------|
| `name` | `StringProperty` | Comment GUID |
| `date` | `StringProperty` | Creation timestamp |
| `author` | `StringProperty` | Author email |
| `comment` | `StringProperty` | Comment text |
| `viewpoint` | `StringProperty` | Linked viewpoint GUID (empty = no viewpoint) |
| `modified_date` | `StringProperty` | Last edit timestamp |
| `modified_author` | `StringProperty` | Last editor email |
| `is_editable` | `BoolProperty` | UI edit mode flag (default: `False`) |

---

## Sources

- Bonsai BCF operator source: `src/bonsai/bonsai/bim/module/bcf/operator.py`
- Bonsai BCF properties: `src/bonsai/bonsai/bim/module/bcf/prop.py`
- Bonsai BCF store: `src/bonsai/bonsai/bim/module/bcf/bcfstore.py`
- Bonsai BCF tool: `src/bonsai/bonsai/tool/bcf.py`
- IfcOpenShell GitHub: https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/bonsai
