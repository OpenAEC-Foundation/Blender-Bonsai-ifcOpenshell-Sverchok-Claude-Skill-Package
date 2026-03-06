# blender-impl-operators — Working Examples

All examples verified against Blender 4.x API. Version-specific notes included where applicable.

---

## Example 1: Modal Timer with Progress Reporting

Processes selected objects incrementally using a modal timer with full progress reporting and cleanup.

```python
# Blender 3.x/4.x/5.x
import bpy

class BATCH_OT_process_objects(bpy.types.Operator):
    """Process all selected objects with progress"""
    bl_idname = "batch.process_objects"
    bl_label = "Batch Process Objects"
    bl_options = {'REGISTER', 'UNDO'}

    _timer = None
    _objects: list = None
    _index: int = 0

    @classmethod
    def poll(cls, context):
        return len(context.selected_objects) > 0

    def invoke(self, context, event):
        self._objects = list(context.selected_objects)
        self._index = 0

        wm = context.window_manager
        wm.progress_begin(0, len(self._objects))
        self._timer = wm.event_timer_add(0.01, window=context.window)
        wm.modal_handler_add(self)

        context.area.header_text_set(f"Processing: 0/{len(self._objects)}")
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type in {'RIGHTMOUSE', 'ESC'}:
            self._cleanup(context)
            self.report({'WARNING'}, "Processing cancelled")
            return {'CANCELLED'}

        if event.type == 'TIMER':
            if self._index >= len(self._objects):
                self._cleanup(context)
                self.report({'INFO'}, f"Processed {len(self._objects)} objects")
                return {'FINISHED'}

            # Process one item per timer tick
            obj = self._objects[self._index]
            obj.location.z += 0.1  # Example work
            self._index += 1

            # Update progress
            context.window_manager.progress_update(self._index)
            context.area.header_text_set(
                f"Processing: {self._index}/{len(self._objects)}"
            )

        return {'PASS_THROUGH'}

    def _cleanup(self, context):
        context.area.header_text_set(None)
        wm = context.window_manager
        wm.progress_end()
        if self._timer is not None:
            wm.event_timer_remove(self._timer)
            self._timer = None

    def cancel(self, context):
        self._cleanup(context)


def register():
    bpy.utils.register_class(BATCH_OT_process_objects)

def unregister():
    bpy.utils.unregister_class(BATCH_OT_process_objects)
```

---

## Example 2: File Import with ImportHelper

Complete import operator using `bpy_extras.io_utils.ImportHelper` with file filtering and custom options.

```python
# Blender 3.x/4.x/5.x
import bpy
import os
import json
from bpy_extras.io_utils import ImportHelper

class DATA_OT_import_json(bpy.types.Operator, ImportHelper):
    """Import point data from JSON file"""
    bl_idname = "data.import_json"
    bl_label = "Import JSON Points"
    bl_options = {'REGISTER', 'UNDO'}

    filter_glob: bpy.props.StringProperty(
        default="*.json",
        options={'HIDDEN'},
    )
    scale: bpy.props.FloatProperty(
        name="Scale Factor",
        default=1.0,
        min=0.001,
        description="Scale factor for imported coordinates",
    )
    create_mesh: bpy.props.BoolProperty(
        name="Create Mesh",
        default=True,
        description="Create mesh object from points (otherwise empties)",
    )

    def execute(self, context):
        if not os.path.isfile(self.filepath):
            self.report({'ERROR'}, f"File not found: {self.filepath}")
            return {'CANCELLED'}

        try:
            with open(self.filepath, 'r') as f:
                data = json.load(f)
        except (json.JSONDecodeError, IOError) as e:
            self.report({'ERROR'}, f"Cannot read file: {e}")
            return {'CANCELLED'}

        points = data.get("points", [])
        if not points:
            self.report({'WARNING'}, "No points found in file")
            return {'CANCELLED'}

        if self.create_mesh:
            mesh = bpy.data.meshes.new("ImportedPoints")
            verts = [(p[0] * self.scale, p[1] * self.scale, p[2] * self.scale)
                     for p in points]
            mesh.from_pydata(verts, [], [])
            mesh.update()
            obj = bpy.data.objects.new("ImportedPoints", mesh)
            context.collection.objects.link(obj)
        else:
            for i, p in enumerate(points):
                empty = bpy.data.objects.new(f"Point_{i:04d}", None)
                empty.location = (p[0] * self.scale, p[1] * self.scale, p[2] * self.scale)
                empty.empty_display_type = 'PLAIN_AXES'
                empty.empty_display_size = 0.1
                context.collection.objects.link(empty)

        self.report({'INFO'}, f"Imported {len(points)} points from {os.path.basename(self.filepath)}")
        return {'FINISHED'}


def menu_func_import(self, context):
    self.layout.operator(DATA_OT_import_json.bl_idname, text="JSON Points (.json)")

def register():
    bpy.utils.register_class(DATA_OT_import_json)
    bpy.types.TOPBAR_MT_file_import.append(menu_func_import)

def unregister():
    bpy.types.TOPBAR_MT_file_import.remove(menu_func_import)
    bpy.utils.unregister_class(DATA_OT_import_json)
```

---

## Example 3: File Export with ExportHelper

```python
# Blender 3.x/4.x/5.x
import bpy
import json
from bpy_extras.io_utils import ExportHelper

class DATA_OT_export_json(bpy.types.Operator, ExportHelper):
    """Export selected object vertices as JSON"""
    bl_idname = "data.export_json"
    bl_label = "Export JSON Points"

    filename_ext = ".json"  # REQUIRED for ExportHelper

    filter_glob: bpy.props.StringProperty(default="*.json", options={'HIDDEN'})

    apply_modifiers: bpy.props.BoolProperty(
        name="Apply Modifiers",
        default=True,
        description="Export evaluated mesh with modifiers applied",
    )

    @classmethod
    def poll(cls, context):
        return (context.active_object is not None
                and context.active_object.type == 'MESH')

    def execute(self, context):
        obj = context.active_object

        if self.apply_modifiers:
            depsgraph = context.evaluated_depsgraph_get()
            obj_eval = obj.evaluated_get(depsgraph)
            mesh = obj_eval.to_mesh()
        else:
            mesh = obj.data

        points = [[v.co.x, v.co.y, v.co.z] for v in mesh.vertices]

        if self.apply_modifiers:
            obj_eval.to_mesh_clear()

        data = {"object": obj.name, "vertex_count": len(points), "points": points}

        try:
            with open(self.filepath, 'w') as f:
                json.dump(data, f, indent=2)
        except IOError as e:
            self.report({'ERROR'}, f"Cannot write file: {e}")
            return {'CANCELLED'}

        self.report({'INFO'}, f"Exported {len(points)} vertices to {self.filepath}")
        return {'FINISHED'}


def menu_func_export(self, context):
    self.layout.operator(DATA_OT_export_json.bl_idname, text="JSON Points (.json)")

def register():
    bpy.utils.register_class(DATA_OT_export_json)
    bpy.types.TOPBAR_MT_file_export.append(menu_func_export)

def unregister():
    bpy.types.TOPBAR_MT_file_export.remove(menu_func_export)
    bpy.utils.unregister_class(DATA_OT_export_json)
```

---

## Example 4: Batch Apply Modifiers with Context Override

```python
# Blender 4.0+ — context.temp_override() is REQUIRED
import bpy

class OBJECT_OT_batch_apply_modifiers(bpy.types.Operator):
    """Apply all modifiers on selected objects"""
    bl_idname = "object.batch_apply_modifiers"
    bl_label = "Batch Apply Modifiers"
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        return any(obj.modifiers for obj in context.selected_objects)

    def execute(self, context):
        objects = list(context.selected_objects)
        applied = 0
        failed = 0

        for obj in objects:
            with bpy.context.temp_override(active_object=obj, object=obj):
                for mod in obj.modifiers[:]:  # Copy — modifiers removed during apply
                    try:
                        bpy.ops.object.modifier_apply(modifier=mod.name)
                        applied += 1
                    except RuntimeError:
                        failed += 1

        self.report({'INFO'}, f"Applied {applied} modifiers ({failed} failed)")
        return {'FINISHED'}
```

---

## Example 5: Multi-Step State Machine Modal Operator

Interactive tool that requires two clicks to define a line, then creates geometry.

```python
# Blender 3.x/4.x/5.x
import bpy
from mathutils import Vector

class TOOLS_OT_line_tool(bpy.types.Operator):
    """Click two points to create an edge"""
    bl_idname = "tools.line_tool"
    bl_label = "Line Tool"
    bl_options = {'REGISTER', 'UNDO'}

    _state: str = 'PICK_START'
    _start: Vector = None

    def invoke(self, context, event):
        self._state = 'PICK_START'
        self._start = None
        context.window_manager.modal_handler_add(self)
        context.area.header_text_set("Line Tool: click start point (ESC to cancel)")
        return {'RUNNING_MODAL'}

    def modal(self, context, event):
        if event.type in {'RIGHTMOUSE', 'ESC'}:
            context.area.header_text_set(None)
            return {'CANCELLED'}

        if event.type == 'LEFTMOUSE' and event.value == 'PRESS':
            if self._state == 'PICK_START':
                self._start = self._region_to_3d(context, event)
                if self._start is None:
                    self.report({'WARNING'}, "Cannot determine 3D position")
                    return {'RUNNING_MODAL'}
                self._state = 'PICK_END'
                context.area.header_text_set("Line Tool: click end point (ESC to cancel)")
                return {'RUNNING_MODAL'}

            elif self._state == 'PICK_END':
                end = self._region_to_3d(context, event)
                if end is None:
                    return {'RUNNING_MODAL'}
                self._create_edge(context, self._start, end)
                context.area.header_text_set(None)
                return {'FINISHED'}

        return {'PASS_THROUGH'}

    def _region_to_3d(self, context, event):
        """Convert 2D mouse position to 3D using view ray on XY plane."""
        from bpy_extras.view3d_utils import region_2d_to_origin_3d, region_2d_to_vector_3d
        region = context.region
        rv3d = context.region_data
        coord = (event.mouse_region_x, event.mouse_region_y)
        origin = region_2d_to_origin_3d(region, rv3d, coord)
        direction = region_2d_to_vector_3d(region, rv3d, coord)
        # Intersect with XY plane (Z=0)
        if abs(direction.z) < 1e-6:
            return None
        t = -origin.z / direction.z
        return origin + direction * t

    def _create_edge(self, context, start, end):
        mesh = bpy.data.meshes.new("Line")
        mesh.from_pydata([start, end], [(0, 1)], [])
        mesh.update()
        obj = bpy.data.objects.new("Line", mesh)
        context.collection.objects.link(obj)
```

---

## Example 6: Parameter Dialog with Dynamic Check

Operator that gathers parameters via dialog, with dynamic validation using `check()`.

```python
# Blender 3.x/4.x/5.x
import bpy

class OBJECT_OT_array_create(bpy.types.Operator):
    """Create an array of the active object with parameters"""
    bl_idname = "object.array_create"
    bl_label = "Create Array"
    bl_options = {'REGISTER', 'UNDO'}

    count: bpy.props.IntProperty(name="Count", default=5, min=1, max=1000)
    spacing: bpy.props.FloatProperty(name="Spacing", default=2.0, min=0.01)
    axis: bpy.props.EnumProperty(
        name="Axis",
        items=[('X', "X", ""), ('Y', "Y", ""), ('Z', "Z", "")],
        default='X',
    )
    linked: bpy.props.BoolProperty(
        name="Linked Copies",
        default=True,
        description="Share mesh data between copies",
    )

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self, width=300)

    def check(self, context):
        # Clamp count to prevent accidental massive arrays
        if self.count > 100 and not self.linked:
            self.count = 100
        return True

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "count")
        layout.prop(self, "spacing")
        layout.prop(self, "axis", expand=True)
        layout.prop(self, "linked")
        if self.count > 50:
            layout.label(text=f"Will create {self.count} objects", icon='INFO')
        if self.count > 100 and not self.linked:
            layout.label(text="Linked copies required for >100", icon='ERROR')

    def execute(self, context):
        source = context.active_object
        axis_map = {'X': 0, 'Y': 1, 'Z': 2}
        axis_idx = axis_map[self.axis]

        for i in range(1, self.count):
            if self.linked:
                new_obj = source.copy()  # Linked copy (shared mesh)
            else:
                new_obj = source.copy()
                new_obj.data = source.data.copy()  # Deep copy mesh

            offset = [0.0, 0.0, 0.0]
            offset[axis_idx] = i * self.spacing
            new_obj.location = (
                source.location.x + offset[0],
                source.location.y + offset[1],
                source.location.z + offset[2],
            )
            context.collection.objects.link(new_obj)

        self.report({'INFO'}, f"Created {self.count - 1} copies")
        return {'FINISHED'}
```

---

## Example 7: Thread-Safe Background Work with bpy.app.timers

Pattern for running long computation in a background thread and updating Blender safely.

```python
# Blender 3.x/4.x/5.x
import bpy
import queue
import threading

_result_queue = queue.Queue()

def _process_results():
    """Timer callback — runs on Blender's main thread."""
    while not _result_queue.empty():
        fn = _result_queue.get()
        fn()
    return 0.5  # Check every 0.5 seconds

def _heavy_computation(filepath):
    """Runs in background thread — NEVER touch bpy here."""
    import time
    time.sleep(3)  # Simulate long computation
    result = f"Processed {filepath}"

    def apply_result():
        """This closure runs on the main thread via the queue."""
        bpy.context.scene.name = result[:63]
        print(f"Result applied: {result}")

    _result_queue.put(apply_result)


class TOOLS_OT_background_process(bpy.types.Operator):
    """Run heavy computation in background thread"""
    bl_idname = "tools.background_process"
    bl_label = "Background Process"

    filepath: bpy.props.StringProperty(name="File", default="/tmp/data.txt")

    def execute(self, context):
        # Ensure timer is running
        if not bpy.app.timers.is_registered(_process_results):
            bpy.app.timers.register(_process_results, persistent=True)

        # Start background thread
        thread = threading.Thread(
            target=_heavy_computation,
            args=(self.filepath,),
            daemon=True,
        )
        thread.start()

        self.report({'INFO'}, "Background processing started")
        return {'FINISHED'}
```

---

## Example 8: Undo-Grouped Operator for Rapid Calls

Operator designed to be called repeatedly (e.g., from a timer or UI slider) with consolidated undo.

```python
# Blender 3.x/4.x/5.x
import bpy

class OBJECT_OT_incremental_move(bpy.types.Operator):
    """Move active object incrementally — repeated calls produce one undo step"""
    bl_idname = "object.incremental_move"
    bl_label = "Incremental Move"
    bl_options = {'REGISTER', 'UNDO_GROUPED'}  # Key: UNDO_GROUPED

    delta: bpy.props.FloatProperty(name="Delta", default=0.1)

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def execute(self, context):
        context.active_object.location.z += self.delta
        return {'FINISHED'}
```

---

## Example 9: Confirmation Dialog for Destructive Operations

```python
# Blender 3.x/4.x/5.x
import bpy

class OBJECT_OT_purge_orphans(bpy.types.Operator):
    """Remove all orphaned data blocks"""
    bl_idname = "object.purge_orphans"
    bl_label = "Purge Orphan Data"
    bl_options = {'REGISTER', 'UNDO'}

    def invoke(self, context, event):
        return context.window_manager.invoke_confirm(self, event)

    def execute(self, context):
        removed = 0
        for collection_name in ('meshes', 'materials', 'textures', 'images'):
            collection = getattr(bpy.data, collection_name)
            for item in list(collection):
                if item.users == 0:
                    collection.remove(item)
                    removed += 1
        self.report({'INFO'}, f"Removed {removed} orphaned data blocks")
        return {'FINISHED'}
```

---

## Example 10: Single-Shot Deferred Timer

```python
# Blender 3.x/4.x/5.x
import bpy
import functools

def _deferred_action(scene_name, message):
    """Runs once on main thread after delay."""
    scene = bpy.data.scenes.get(scene_name)
    if scene:
        print(f"[Deferred] {message} on scene: {scene.name}")
    return None  # Run once — return None to unregister

# Schedule to run after 2 seconds:
bpy.app.timers.register(
    functools.partial(_deferred_action, "Scene", "Hello"),
    first_interval=2.0,
)
```

---

## Sources

- https://docs.blender.org/api/current/bpy.types.Operator.html
- https://docs.blender.org/api/current/bpy.app.timers.html
- https://docs.blender.org/api/current/bpy_extras.io_utils.html
- https://docs.blender.org/api/current/bpy.types.WindowManager.html
- https://docs.blender.org/api/current/bpy_extras.view3d_utils.html
