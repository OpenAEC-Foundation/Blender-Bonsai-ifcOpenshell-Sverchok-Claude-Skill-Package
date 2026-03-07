# IfcSverchok — API Reference

> IfcOpenShell 0.8.x / Sverchok v1.4.0+ / Blender 4.0+/5.x

## SvIfcStore

Central singleton managing the transient IFC file shared across all nodes.

**Location**: `ifcsverchok/helper.py`

### Class Attributes

```python
class SvIfcStore:
    file: Union[ifcopenshell.file, None] = None
    id_map: dict[str, Any] = {}
    schema_identifiers: list[str] = ["IFC4", "IFC2X3"]
    use_bonsai_file: bool = False
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `file` | `ifcopenshell.file \| None` | Single transient IFC file. `None` until first `get_file()` call |
| `id_map` | `dict[str, Any]` | Maps `node_id` string to IFC entity data created by that node |
| `schema_identifiers` | `list[str]` | Supported IFC schemas |
| `use_bonsai_file` | `bool` | When True, `get_file()` returns Bonsai's active file via `tool.Ifc.get()` |

### Static Methods

#### `SvIfcStore.purge()`

```python
@staticmethod
def purge() -> None
```

Resets all state: sets `file = None`, clears `id_map = {}`. Called automatically before each full node tree re-evaluation ("Re-run all nodes").

#### `SvIfcStore.get_file()`

```python
@staticmethod
def get_file() -> ifcopenshell.file
```

Returns the current transient IFC file. Behavior:
- If `use_bonsai_file` is True: returns `tool.Ifc.get()` (Bonsai's active file)
- If `file` is None: calls `create_boilerplate()` to initialize a minimal IFC4 file
- Returns `SvIfcStore.file`

#### `SvIfcStore.create_boilerplate()`

```python
@staticmethod
def create_boilerplate() -> None
```

Creates a minimal IFC4 file with:
- File header (description, implementation_level, name)
- IfcProject with default units
- IfcGeometricRepresentationContext

---

## SvIfcCore

Base class mixed into all IfcSverchok nodes. Provides double-nested input processing.

**Location**: `ifcsverchok/ifc_core.py`

### Attributes

```python
class SvIfcCore:
    sv_input_names: list[str]  # List of input socket names to process
```

### Methods

#### `SvIfcCore.process()`

```python
def process(self) -> None
```

Gathers all inputs listed in `sv_input_names`, applies `zip_long_repeat` twice (double-nested), and calls `self.process_ifc(*sv_input)` for each individual parameter set.

**Double-nested processing logic**:
```python
sv_inputs_nested = [self.inputs[name].sv_get() for name in self.sv_input_names]
for sv_input_nested in zip_long_repeat(*sv_inputs_nested):      # Level 1: objects
    for sv_input in zip_long_repeat(*sv_input_nested):           # Level 2: items
        self.process_ifc(*list(sv_input))
```

#### `SvIfcCore.process_ifc()`

```python
def process_ifc(self, *args) -> None
```

Abstract method. Each node subclass implements this with its specific IFC logic. Arguments correspond to the input sockets in order of `sv_input_names`.

---

## IFC Nodes (24) — Input/Output Signatures

### SvIfcCreateFile

Creates a new empty IFC file with specified schema.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Schema | String | IFC schema identifier ("IFC4" or "IFC2X3") |
| Output | File | IFC File | The created file reference |

### SvIfcReadFile

Opens an existing IFC file from disk.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Path | String | File path to .ifc file |
| Output | File | IFC File | The loaded file reference |

### SvIfcWriteFile

Writes transient IFC file to disk. Calls `ensure_hirarchy()` before writing.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Path | String | Output file path |

### SvIfcCreateEntity

Creates IFC entities with geometry and placement.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | IfcClass | String | IFC class name (e.g., "IfcWall") |
| Input | Names | String | Entity name |
| Input | Representation | IFC Repr | From representation converter node |
| Input | Location | Matrix | 4x4 placement matrix (optional) |
| Output | Entity | IFC Entity | The created entity |

### SvIfcCreateShape

Converts IFC entities to Blender mesh objects.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Entity | IFC Entity | IFC entity to convert |
| Output | Object | Blender Object | Created mesh object |

### SvIfcReadEntity

Reads entity and exposes all attributes as dynamic outputs.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Entity | IFC Entity | Entity to read |
| Output | (dynamic) | Various | One output per entity attribute |

### SvIfcPickIfcClass

UI picker for IFC classes organized by product category.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Output | IfcClass | String | Selected IFC class name |

### SvIfcById

Retrieves IFC entities by STEP file ID.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Id | Integer | STEP file entity ID |
| Output | Entity | IFC Entity | Retrieved entity |

### SvIfcByGuid

Retrieves IFC entities by GlobalId.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Guid | String | IFC GlobalId string |
| Output | Entity | IFC Entity | Retrieved entity |

### SvIfcByType

Queries all entities of a given IFC type.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | IfcClass | String | IFC class name |
| Output | Entities | IFC Entity List | All matching entities |

### SvIfcByQuery

Queries using IfcOpenShell selector syntax.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Query | String | IfcOpenShell selector expression |
| Output | Entities | IFC Entity List | Matching entities |

### SvIfcAdd

Adds an IFC entity to the file.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Entity | IFC Entity | Entity to add |

### SvIfcAddPset

Creates or edits property sets on IFC elements.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Element | IFC Entity | Target element |
| Input | Name | String | Property set name (e.g., "Pset_WallCommon") |
| Input | Properties | String (JSON) | JSON dict of property name-value pairs |

### SvIfcAddSpatialElement

Creates spatial elements and assigns contained elements.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | IfcClass | String | Spatial class (e.g., "IfcBuildingStorey") |
| Input | Elements | IFC Entity List | Elements to contain |
| Output | SpatialElement | IFC Entity | Created spatial element |

### SvIfcRemove

Removes an entity from the IFC file.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Entity | IFC Entity | Entity to remove |

### SvIfcGenerateGuid

Generates a new IFC-compliant GUID.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Output | Guid | String | New IFC GlobalId |

### SvIfcGetProperty

Retrieves a property value from a property set.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Element | IFC Entity | Source element |
| Input | PsetName | String | Property set name |
| Input | PropName | String | Property name |
| Output | Value | Various | Property value |

### SvIfcGetAttribute

Retrieves a direct attribute value from an entity.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Entity | IFC Entity | Source entity |
| Input | Attribute | String | Attribute name |
| Output | Value | Various | Attribute value |

### SvIfcSelectBlenderObjects

Selects Blender objects by matching GlobalId.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Entities | IFC Entity List | Entities whose GlobalIds to match |

### SvIfcApi

Generic node calling any ifcopenshell.api function.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Function | String | API function name (e.g., "root.create_entity") |
| Input | Arguments | String (JSON) | JSON dict of function arguments |
| Output | Result | Various | API function return value |

### SvIfcBMeshToIfcRepr

Converts Blender mesh objects to IFC representations.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | context_type | String | "Model" or "Plan" |
| Input | context_identifier | String | "Body", "Annotation", "Box", or "Axis" |
| Input | target_view | String | Geometric target view |
| Input | blender_objects | Object List | Blender mesh objects |
| Output | Representations | IFC Repr List | IFC representation entity IDs |
| Output | Locations | Matrix List | World matrices (4x4) |

### SvIfcSverchokToIfcRepr

Converts Sverchok geometry to IFC representations.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | context_type | String | "Model" or "Plan" |
| Input | context_identifier | String | "Body", "Annotation", "Box", or "Axis" |
| Input | target_view | String | Geometric target view |
| Input | Vertices | Vertex List | Sverchok vertex data |
| Input | Edges | Edge List | Sverchok edge data |
| Input | Faces | Face List | Sverchok face data |
| Output | Representations | IFC Repr List | IFC representation entity IDs |

### SvIfcCreateProject

Adds IfcProject with units and representation context.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Name | String | Project name |
| Output | Project | IFC Entity | Created IfcProject |

### SvIfcQuickProjectSetup

Creates complete IFC file with project metadata (project + site + building + storey).

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Schema | String | IFC schema identifier |
| Input | ProjectName | String | Project name |
| Output | File | IFC File | Configured file reference |

---

## IFC Shape Builder Nodes (7)

### SvIfcSbRectangle

Creates IfcRectangleProfileDef via ShapeBuilder.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | XDim | Float | Width of rectangle |
| Input | YDim | Float | Height of rectangle |
| Output | Profile | IFC Profile | Rectangle profile entity |

### SvIfcSbExtrude

Extrudes IFC profile along axis.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Profile | IFC Profile | Profile to extrude |
| Input | Magnitude | Float | Extrusion length |
| Output | Shape | IFC Shape | Extruded area solid |

### SvIfcSbRepresentation

Wraps IFC shape items into IfcShapeRepresentation.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Items | IFC Shape List | Shape items to wrap |
| Output | Representation | IFC Repr | Shape representation entity |

### SvSbMesh

Creates IFC mesh from vertices and polygons.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Vertices | Vertex List | Mesh vertices |
| Input | Faces | Face List | Mesh faces/polygons |
| Output | Shape | IFC Shape | Mesh shape entity |

### SvSbPolyline

Creates IFC polyline from vertices.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Vertices | Vertex List | Polyline vertices |
| Output | Shape | IFC Shape | Polyline entity |

### SvSbShapeOutput

Converts IFC shape back to Sverchok geometry.

| Direction | Socket | Type | Description |
|-----------|--------|------|-------------|
| Input | Shape | IFC Shape | IFC shape to convert |
| Output | Vertices | Vertex List | Extracted vertices |
| Output | Faces | Face List | Extracted faces |

### SvIfcSbTest

Test/debug node for ShapeBuilder development.

---

## ensure_hirarchy()

**Location**: `ifcsverchok/helper.py`

```python
def ensure_hirarchy(file: ifcopenshell.file) -> None
```

Called automatically by `SvIfcWriteFile` before exporting. Creates missing spatial structure:

1. Creates default `IfcBuilding` if none exists
2. Assigns orphaned `IfcBuildingStorey` elements to the building
3. Assigns uncontained `IfcElement` instances to the building
4. Creates default `IfcSite` if none exists
5. Links Building → Site → Project via `IfcRelAggregates`

**Note**: Function name contains intentional typo ("hirarchy" not "hierarchy") — preserved from source code.
