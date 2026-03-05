# IfcOpenShell Core Operations - File I/O

> Research fragment covering IfcOpenShell file I/O operations: opening, creating, writing IFC files,
> and creating IFC entities programmatically.

---

## Table of Contents

1. [Opening IFC Files - `ifcopenshell.open()`](#1-opening-ifc-files---ifcopenshellopen)
2. [Creating New IFC Files - `ifcopenshell.file()`](#2-creating-new-ifc-files---ifcopenshellfile)
3. [Writing IFC Files - `file.write()`](#3-writing-ifc-files---filewrite)
4. [Creating Entities - `file.create_entity()`](#4-creating-entities---filecreate_entity)
5. [High-Level API for Entity Creation - `ifcopenshell.api.root.create_entity()`](#5-high-level-api-for-entity-creation)
6. [High-Level API for File Creation - `ifcopenshell.api.project.create_file()`](#6-high-level-api-for-file-creation)
7. [Auxiliary File Methods](#7-auxiliary-file-methods)
8. [Complete Workflow Examples](#8-complete-workflow-examples)
9. [Sources](#9-sources)

---

## 1. Opening IFC Files - `ifcopenshell.open()`

### Function Signature

```python
ifcopenshell.open(
    path: Union[os.PathLike, str],
    format: Optional[str] = None,
    should_stream: bool = False
) -> ifcopenshell.file
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | `os.PathLike \| str` | *required* | Path to the IFC file on disk |
| `format` | `str \| None` | `None` | Force specific format: `".ifc"`, `".ifcZIP"`, `".ifcXML"`, `".ifcJSON"`, `".ifcSQLite"`. Auto-detected from extension when `None` |
| `should_stream` | `bool` | `False` | Enable streaming mode for large files (reduces memory footprint) |

### Return Value

Returns an `ifcopenshell.file` object representing the IFC model loaded into memory.

### Supported Formats

The function automatically detects file format from extension using `ifcopenshell.guess_format()`:
- `.ifc` - IFC-SPF (STEP Physical File, the standard text format)
- `.ifcXML` - IFC in XML serialization
- `.ifcZIP` - Compressed IFC archive (extracts and processes recursively)
- `.ifcJSON` - IFC in JSON format
- `.ifcSQLite` - IFC in SQLite database format

### Error Handling

The module defines specialized exceptions:
- `ifcopenshell.Error` - Generic problem indicator
- `ifcopenshell.SchemaError` - IFC schema-specific issues

Three status constants identify read failures:
- `READ_ERROR` - General file read failure
- `NO_HEADER` - Missing STEP header
- `UNSUPPORTED_SCHEMA` - Schema version not recognized

### Code Examples

```python
import ifcopenshell

# Basic file opening
model = ifcopenshell.open('/path/to/your/model.ifc')

# Access the schema version
print(model.schema)  # Returns: "IFC2X3", "IFC4", or "IFC4X3"

# Access full schema identifier
print(model.schema_identifier)  # e.g., "IFC4_ADD2"

# Access schema version tuple
print(model.schema_version)  # e.g., (4, 0, 2, 1)

# Open a zipped IFC file
model = ifcopenshell.open('/path/to/model.ifcZIP')

# Force a specific format regardless of extension
model = ifcopenshell.open('/path/to/model.dat', format='.ifc')

# Open an IFC XML file
model = ifcopenshell.open('/path/to/model.ifcXML')
```

### Accessing File Header After Opening

```python
model = ifcopenshell.open('model.ifc')

# Access header information
header = model.header
print(header.file_description)
print(header.file_name)
print(header.file_schema)
```

---

## 2. Creating New IFC Files - `ifcopenshell.file()`

### Constructor Signature

```python
ifcopenshell.file(
    f: Optional[ifcopenshell_wrapper.file] = None,
    schema: Optional[str] = None,
    schema_version: Optional[tuple[int, int, int, int]] = None
) -> ifcopenshell.file
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `f` | `ifcopenshell_wrapper.file \| None` | `None` | Underlying IfcOpenShell C++ file object to wrap (internal use only) |
| `schema` | `str \| None` | `"IFC4"` | Which IFC schema to use: `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |
| `schema_version` | `tuple[int,int,int,int] \| None` | `None` | Specific version as `(major, minor, addendum, corrigendum)`, e.g., `(4, 0, 2, 1)` for IFC4 ADD2 TC1 |

### Return Value

Returns a new blank `ifcopenshell.file` object with no entities. All data is stored in memory.

### Internal Initialization

The constructor initializes undo/redo tracking:
- `history_size = 64` (maximum undo steps)
- `history = []` (undo log)
- `future = []` (redo log)
- `transaction = None` (current transaction context)

### Code Examples

```python
import ifcopenshell

# Create a blank IFC4 file (default)
model = ifcopenshell.file()

# Create a blank IFC2X3 file
model = ifcopenshell.file(schema='IFC2X3')

# Create a blank IFC4X3 file
model = ifcopenshell.file(schema='IFC4X3')

# Create with a specific schema version
model = ifcopenshell.file(schema_version=(4, 0, 2, 1))  # IFC4 ADD2 TC1

# Verify the schema
print(model.schema)  # "IFC4"
```

### Important Notes

- `ifcopenshell.file()` creates a **completely blank** model with no header metadata, no project structure, and no units.
- For production use, prefer `ifcopenshell.api.project.create_file()` which sets up header data, timestamps, and MVD defaults automatically (see Section 6).
- The file object supports Python's `len()` function: `len(model)` returns the total number of entities.
- The file object is iterable: you can loop over all entities with `for entity in model:`.

---

## 3. Writing IFC Files - `file.write()`

### Method Signature

```python
file.write(
    path: os.PathLike | str,
    format: str | None = None,
    zipped: bool = False
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | `os.PathLike \| str` | *required* | File output path. Parent directories are created automatically if they don't exist |
| `format` | `str \| None` | `None` | Force specific format: `".ifc"`, `".ifcXML"`, `".ifcZIP"`. Auto-guessed from filename extension when `None` |
| `zipped` | `bool` | `False` | Whether to ZIP-compress the file after writing (uses ZIP_DEFLATED compression) |

### Return Value

`None` - writes the file to disk as a side effect.

### Supported Output Formats

| Extension/Format | Description |
|-----------------|-------------|
| `.ifc` | Standard IFC-SPF (STEP Physical File) text format |
| `.ifcXML` | IFC in XML serialization |
| `.ifcZIP` | Equivalent to `.ifc` with `zipped=True` |

For zipped `.ifcXML`, use `format='.ifcXML'` combined with `zipped=True`.

### Behavior Details

1. **Directory creation**: Automatically creates parent directories using `os.makedirs(exist_ok=True)`
2. **Format guessing**: When `format=None`, guesses from the file extension via `ifcopenshell.guess_format()`
3. **XML handling**: XML serialization is handled via a separate code path
4. **ZIP compression**: When `zipped=True`, wraps the output in a ZIP archive using `ZIP_DEFLATED` compression

### Code Examples

```python
import ifcopenshell

model = ifcopenshell.open('input.ifc')

# Write to standard .ifc format
model.write('/path/to/output/model.ifc')

# Write as XML
model.write('/path/to/output/model.ifcXML')

# Write as compressed ZIP
model.write('/path/to/output/model.ifcZIP')

# Force a specific format regardless of extension
model.write('/path/to/output/model.dat', format='.ifc')

# Write .ifc but also zip it
model.write('/path/to/output/model.ifc', zipped=True)

# Write zipped XML
model.write('/path/to/output/model.ifcXML', format='.ifcXML', zipped=True)
```

### Serialization to String

Instead of writing to disk, you can serialize to a string:

```python
# Get the entire IFC file as a STEP-SPF string
ifc_string = model.to_string()
print(ifc_string)
```

---

## 4. Creating Entities - `file.create_entity()`

### Method Signature

```python
file.create_entity(
    type: str,
    *args: Any,
    **kwargs: Any
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | `str` | **Case-insensitive** name of the IFC class (e.g., `"IfcWall"`, `"IfcPerson"`, `"IFCWALL"`) |
| `*args` | `Any` | Positional arguments matching the IFC class attribute order |
| `**kwargs` | `Any` | Keyword arguments matching IFC class attribute names |

Special keyword argument:
- `id` (int, optional): Explicitly set the STEP entity ID number

### Return Value

Returns an `ifcopenshell.entity_instance` object representing the newly created entity in the file.

### Behavior Details

1. Arguments are mapped to attribute indices of the IFC schema class definition
2. Values are populated while transaction tracking is temporarily disabled for performance
3. The instance is added to the underlying file wrapper
4. If a transaction is active, the creation is recorded in the transaction log (for undo/redo)
5. Unspecified attributes default to `$` (null/unset) in the IFC file

### Code Examples

#### Basic Entity Creation

```python
import ifcopenshell

model = ifcopenshell.file()

# Create a blank entity (all attributes default to $)
person = model.create_entity('IfcPerson')
# Result: #1=IFCPERSON($,$,$,$,$,$,$,$)

# Create with a positional argument (first attribute)
person = model.create_entity('IfcPerson', 'Foobar')
# Result: #2=IFCPERSON('Foobar',$,$,$,$,$,$,$)

# Create with keyword arguments
person = model.create_entity('IfcPerson', Identification='Foobar')
# Result: #3=IFCPERSON('Foobar',$,$,$,$,$,$,$)
```

#### Creating Entities with GlobalId

```python
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file()

# Create a wall with auto-generated GlobalId and a name
wall = model.create_entity('IfcWall',
    GlobalId=ifcopenshell.guid.new(),
    Name='My Wall'
)
# Result: #1=IFCWALL('0EI0MSHbX9gg8Fxwar7lL8',$,'My Wall',$,$,$,$,$,$)
```

#### Using Dictionary Expansion

```python
data = {
    'GlobalId': ifcopenshell.guid.new(),
    'Name': 'Wall Name'
}
wall = model.create_entity('IfcWall', **data)
```

#### Creating Value Types (Wrapped Values)

```python
# Create IFC value type instances for property values
text_value = model.create_entity("IfcText", "Describe the Reference")
bool_value = model.create_entity("IfcBoolean", True)
real_value = model.create_entity("IfcReal", 2.569)
int_value = model.create_entity("IfcInteger", 2)
```

#### Dynamic `createIfc*()` Methods

The file object dynamically generates shorthand methods for every IFC class:

```python
# These are equivalent:
wall = model.create_entity('IfcWall')
wall = model.createIfcWall()

# Dynamic methods also accept positional and keyword arguments
wall = model.createIfcWall(ifcopenshell.guid.new(), None, 'My Wall')

# Create geometric primitives
point = model.createIfcCartesianPoint((0.0, 0.0, 0.0))
direction = model.createIfcDirection((0.0, 0.0, 1.0))
```

#### Assigning Entity References

```python
# Assign one entity as an attribute of another
wall = model.createIfcWall()
owner_history = model.createIfcOwnerHistory()
wall.OwnerHistory = owner_history
```

#### Creating Property Sets

```python
property_values = [
    model.createIfcPropertySingleValue(
        "Reference", "Reference",
        model.create_entity("IfcText", "Some reference"), None
    ),
    model.createIfcPropertySingleValue(
        "IsExternal", "IsExternal",
        model.create_entity("IfcBoolean", True), None
    ),
    model.createIfcPropertySingleValue(
        "ThermalTransmittance", "ThermalTransmittance",
        model.create_entity("IfcReal", 2.569), None
    ),
]

property_set = model.createIfcPropertySet(
    ifcopenshell.guid.new(), None,
    "Pset_WallCommon", None,
    property_values
)
```

---

## 5. High-Level API for Entity Creation

### `ifcopenshell.api.root.create_entity()`

This is the **recommended** approach for creating rooted IFC products and product types. It handles many validation and setup details automatically.

### Function Signature

```python
ifcopenshell.api.root.create_entity(
    file: ifcopenshell.file,
    ifc_class: str = 'IfcBuildingElementProxy',
    predefined_type: str | None = None,
    name: str | None = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `ifcopenshell.file` | *required* | The IFC file object to create the entity in |
| `ifc_class` | `str` | `'IfcBuildingElementProxy'` | The IFC class name to instantiate (any rooted IFC class) |
| `predefined_type` | `str \| None` | `None` | Built-in or user-defined predefined type. Custom types are handled automatically |
| `name` | `str \| None` | `None` | Optional name/identifier for the entity |

### Return Value

Returns an `ifcopenshell.entity_instance` of the specified IFC class.

### Automatic Features

Unlike `file.create_entity()`, this high-level API method automatically:
1. **Generates a valid GlobalId** (22-character IFC GUID)
2. **Stores ownership history** (OwnerHistory)
3. **Validates and stores predefined types** (distinguishes built-in vs. custom types)
4. **Handles schema-dependent defaults** via `handle_2x3_defaults()` and `handle_4_defaults()`
5. **Populates mandatory attributes** that users commonly forget

### Code Examples

```python
import ifcopenshell
import ifcopenshell.api.root
import ifcopenshell.api.project

model = ifcopenshell.api.project.create_file()

# Create a project
project = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcProject", name="My Project")

# Create spatial structure
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="My Site")

building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Building A")

storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# Create building elements
wall = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcWall", name="Exterior Wall")

# Create a wall type with predefined type
wall_type = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcWallType", predefined_type="STANDARD",
    name="Generic Wall Type")

# Create a door with predefined type
door = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcDoor", predefined_type="DOOR",
    name="Main Entrance")
```

### When to Use Which

| Method | Use Case |
|--------|----------|
| `file.create_entity()` | Low-level entity creation, non-rooted entities, value types, geometry primitives |
| `ifcopenshell.api.root.create_entity()` | Rooted products/types (walls, doors, sites, buildings, etc.) |

---

## 6. High-Level API for File Creation

### `ifcopenshell.api.project.create_file()`

### Function Signature

```python
ifcopenshell.api.project.create_file(
    version: str = 'IFC4'
) -> ifcopenshell.file
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `version` | `str` | `'IFC4'` | IFC schema version: `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"`. Custom schema identifiers also accepted if loaded |

### Return Value

Returns an initialized `ifcopenshell.file` object with pre-configured header metadata.

### Differences from `ifcopenshell.file()`

| Feature | `ifcopenshell.file()` | `ifcopenshell.api.project.create_file()` |
|---------|----------------------|------------------------------------------|
| Header metadata | Empty/minimal | Pre-populated with timestamp, application info |
| Preprocessor | Not set | Set to "IfcOpenShell" |
| MVD | Not set | Defaults to DesignTransferView |
| Timestamp | Not set | Current timestamp |
| Schema setup | Basic | Full header with FILE_DESCRIPTION, FILE_NAME, FILE_SCHEMA |

### Code Example

```python
import ifcopenshell.api.project
import ifcopenshell.api.root
import ifcopenshell.api.unit
import ifcopenshell.api.context

# Create file with pre-configured header
model = ifcopenshell.api.project.create_file()

# Create project (required for valid IFC)
project = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcProject", name="My Project")

# Assign default SI units
ifcopenshell.api.unit.assign_unit(model)

# Set up geometry contexts
context = ifcopenshell.api.context.add_context(model, context_type="Model")
body = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=context)
```

---

## 7. Auxiliary File Methods

### `file.add()` - Transfer Entities Between Files

```python
file.add(
    inst: ifcopenshell.entity_instance,
    _id: int = None
) -> ifcopenshell.entity_instance
```

Adds an entity **and all its dependent/referenced entities** to the file. Duplicate additions (checked via `.identity()`) are silently skipped. Returns the added instance.

```python
# Copy a wall from one file to another
source = ifcopenshell.open('source.ifc')
target = ifcopenshell.file(schema=source.schema)

wall = source.by_type('IfcWall')[0]
target.add(wall)  # Recursively adds wall and all referenced entities

target.write('target.ifc')
```

### `file.remove()` - Delete Entities

```python
file.remove(
    inst: ifcopenshell.entity_instance
) -> None
```

Deletes an IFC entity from the file. Any attributes in other entities that reference the deleted entity become null (`$`). References within aggregate types (lists/sets) are removed.

```python
wall = model.by_type('IfcWall')[0]
model.remove(wall)
```

### `file.to_string()` - Serialize to String

```python
file.to_string() -> str
```

Returns the entire IFC model as a STEP-SPF formatted string without writing to disk.

```python
ifc_text = model.to_string()
print(ifc_text[:500])  # Print first 500 characters
```

### File Properties

| Property | Type | Description |
|----------|------|-------------|
| `file.schema` | `str` | General version: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |
| `file.schema_identifier` | `str` | Full version string: `"IFC4_ADD2"`, etc. |
| `file.schema_version` | `tuple` | Version as `(major, minor, addendum, corrigendum)` |
| `file.header` | object | File header metadata access |
| `file.history` | list | Transaction log (oldest to newest) |
| `file.future` | list | Redo history (newest to oldest) |

### Transaction Support

```python
# Begin a transaction for undo/redo support
model.begin_transaction()

# Make changes...
wall = model.create_entity('IfcWall')
wall.Name = "New Wall"

# End the transaction (changes are recorded)
model.end_transaction()

# Undo the last transaction
model.undo()

# Redo the undone transaction
model.redo()

# Discard an active transaction without recording
model.begin_transaction()
# ... changes ...
model.discard_transaction()

# Set maximum undo history size
model.set_history_size(128)
```

---

## 8. Complete Workflow Examples

### Example 1: Create a Simple Model from Scratch (Modern API)

This is the **recommended** approach using the high-level `ifcopenshell.api` modules.

```python
import ifcopenshell
import ifcopenshell.api.root
import ifcopenshell.api.unit
import ifcopenshell.api.context
import ifcopenshell.api.project
import ifcopenshell.api.spatial
import ifcopenshell.api.geometry
import ifcopenshell.api.aggregate

# 1. Create the file with proper header metadata
model = ifcopenshell.api.project.create_file()

# 2. Create the mandatory IfcProject
project = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcProject", name="My Project")

# 3. Assign default SI units
ifcopenshell.api.unit.assign_unit(model)

# 4. Set up geometry representation contexts
context = ifcopenshell.api.context.add_context(model, context_type="Model")
body = ifcopenshell.api.context.add_context(model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=context)

# 5. Create spatial hierarchy
site = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.root.create_entity(model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# 6. Assign spatial aggregation
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model,
    relating_object=building, products=[storey])

# 7. Create a wall element
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall")

# 8. Set its placement
ifcopenshell.api.geometry.edit_object_placement(model, product=wall)

# 9. Add geometry representation
representation = ifcopenshell.api.geometry.add_wall_representation(model,
    context=body, length=5, height=3, thickness=0.2)
ifcopenshell.api.geometry.assign_representation(model,
    product=wall, representation=representation)

# 10. Assign wall to storey
ifcopenshell.api.spatial.assign_container(model,
    relating_structure=storey, products=[wall])

# 11. Write to disk
model.write("output_model.ifc")
```

### Example 2: Create a Wall with Properties (Low-Level API)

This example shows the **low-level** approach using `create_entity()` and `createIfc*()` directly, giving full control over every entity.

```python
import uuid
import time
import tempfile
import ifcopenshell

# Helper functions
O = 0., 0., 0.
X = 1., 0., 0.
Y = 0., 1., 0.
Z = 0., 0., 1.

create_guid = lambda: ifcopenshell.guid.compress(uuid.uuid1().hex)

def create_ifcaxis2placement(ifcfile, point=O, dir1=Z, dir2=X):
    point = ifcfile.createIfcCartesianPoint(point)
    dir1 = ifcfile.createIfcDirection(dir1)
    dir2 = ifcfile.createIfcDirection(dir2)
    axis2placement = ifcfile.createIfcAxis2Placement3D(point, dir1, dir2)
    return axis2placement

def create_ifclocalplacement(ifcfile, point=O, dir1=Z, dir2=X, relative_to=None):
    axis2placement = create_ifcaxis2placement(ifcfile, point, dir1, dir2)
    ifclocalplacement = ifcfile.createIfcLocalPlacement(relative_to, axis2placement)
    return ifclocalplacement

def create_ifcpolyline(ifcfile, point_list):
    ifcpts = [ifcfile.createIfcCartesianPoint(point) for point in point_list]
    polyline = ifcfile.createIfcPolyLine(ifcpts)
    return polyline

def create_ifcextrudedareasolid(ifcfile, point_list, ifcaxis2placement,
                                 extrude_dir, extrusion):
    polyline = create_ifcpolyline(ifcfile, point_list)
    ifcclosedprofile = ifcfile.createIfcArbitraryClosedProfileDef(
        "AREA", None, polyline)
    ifcdir = ifcfile.createIfcDirection(extrude_dir)
    ifcextrudedareasolid = ifcfile.createIfcExtrudedAreaSolid(
        ifcclosedprofile, ifcaxis2placement, ifcdir, extrusion)
    return ifcextrudedareasolid


# --- File setup using a STEP template ---
filename = "hello_wall.ifc"
timestamp = time.time()
timestring = time.strftime("%Y-%m-%dT%H:%M:%S", time.gmtime(timestamp))
creator = "IfcOpenShell User"
organization = "MyOrganization"
application, application_version = "IfcOpenShell", "0.8"
project_globalid, project_name = create_guid(), "Hello Wall"

template = """ISO-10303-21;
HEADER;
FILE_DESCRIPTION(('ViewDefinition [CoordinationView]'),'2;1');
FILE_NAME('%(filename)s','%(timestring)s',('%(creator)s'),('%(organization)s'),'%(application)s','%(application)s','');
FILE_SCHEMA(('IFC2X3'));
ENDSEC;
DATA;
#1=IFCPERSON($,$,'%(creator)s',$,$,$,$,$);
#2=IFCORGANIZATION($,'%(organization)s',$,$,$);
#3=IFCPERSONANDORGANIZATION(#1,#2,$);
#4=IFCAPPLICATION(#2,'%(application_version)s','%(application)s','');
#5=IFCOWNERHISTORY(#3,#4,$,.ADDED.,$,#3,#4,%(timestamp)s);
#6=IFCDIRECTION((1.,0.,0.));
#7=IFCDIRECTION((0.,0.,1.));
#8=IFCCARTESIANPOINT((0.,0.,0.));
#9=IFCAXIS2PLACEMENT3D(#8,#7,#6);
#10=IFCDIRECTION((0.,1.,0.));
#11=IFCGEOMETRICREPRESENTATIONCONTEXT($,'Model',3,1.E-05,#9,#10);
#12=IFCDIMENSIONALEXPONENTS(0,0,0,0,0,0,0);
#13=IFCSIUNIT(*,.LENGTHUNIT.,$,.METRE.);
#14=IFCSIUNIT(*,.AREAUNIT.,$,.SQUARE_METRE.);
#15=IFCSIUNIT(*,.VOLUMEUNIT.,$,.CUBIC_METRE.);
#16=IFCSIUNIT(*,.PLANEANGLEUNIT.,$,.RADIAN.);
#17=IFCMEASUREWITHUNIT(IFCPLANEANGLEMEASURE(0.017453292519943295),#16);
#18=IFCCONVERSIONBASEDUNIT(#12,.PLANEANGLEUNIT.,'DEGREE',#17);
#19=IFCUNITASSIGNMENT((#13,#14,#15,#18));
#20=IFCPROJECT('%(project_globalid)s',#5,'%(project_name)s',$,$,$,$,(#11),#19);
ENDSEC;
END-ISO-10303-21;
""" % locals()

# Write template to temp file and open it
temp_handle, temp_filename = tempfile.mkstemp(suffix=".ifc")
with open(temp_filename, "w") as f:
    f.write(template)

ifcfile = ifcopenshell.open(temp_filename)
owner_history = ifcfile.by_type("IfcOwnerHistory")[0]
project = ifcfile.by_type("IfcProject")[0]
context = ifcfile.by_type("IfcGeometricRepresentationContext")[0]

# --- Create spatial hierarchy ---
site_placement = create_ifclocalplacement(ifcfile)
site = ifcfile.createIfcSite(
    create_guid(), owner_history, "Site", None, None,
    site_placement, None, None, "ELEMENT",
    None, None, None, None, None)

building_placement = create_ifclocalplacement(
    ifcfile, relative_to=site_placement)
building = ifcfile.createIfcBuilding(
    create_guid(), owner_history, 'Building', None, None,
    building_placement, None, None, "ELEMENT", None, None, None)

storey_placement = create_ifclocalplacement(
    ifcfile, relative_to=building_placement)
building_storey = ifcfile.createIfcBuildingStorey(
    create_guid(), owner_history, 'Storey', None, None,
    storey_placement, None, None, "ELEMENT", 0.0)

# Aggregation relationships
ifcfile.createIfcRelAggregates(
    create_guid(), owner_history, "Building Container", None,
    building, [building_storey])
ifcfile.createIfcRelAggregates(
    create_guid(), owner_history, "Site Container", None,
    site, [building])
ifcfile.createIfcRelAggregates(
    create_guid(), owner_history, "Project Container", None,
    project, [site])

# --- Create a wall with geometry ---
wall_placement = create_ifclocalplacement(
    ifcfile, relative_to=storey_placement)

# Axis representation (2D center line)
polyline = create_ifcpolyline(ifcfile, [(0.0, 0.0, 0.0), (5.0, 0.0, 0.0)])
axis_representation = ifcfile.createIfcShapeRepresentation(
    context, "Axis", "Curve2D", [polyline])

# Body representation (3D extrusion)
extrusion_placement = create_ifcaxis2placement(
    ifcfile, (0.0, 0.0, 0.0), (0.0, 0.0, 1.0), (1.0, 0.0, 0.0))
point_list = [(0.0, -0.1, 0.0), (5.0, -0.1, 0.0),
              (5.0, 0.1, 0.0), (0.0, 0.1, 0.0), (0.0, -0.1, 0.0)]
solid = create_ifcextrudedareasolid(
    ifcfile, point_list, extrusion_placement, (0.0, 0.0, 1.0), 3.0)
body_representation = ifcfile.createIfcShapeRepresentation(
    context, "Body", "SweptSolid", [solid])

product_shape = ifcfile.createIfcProductDefinitionShape(
    None, None, [axis_representation, body_representation])

wall = ifcfile.createIfcWallStandardCase(
    create_guid(), owner_history, "Wall", "An awesome wall",
    None, wall_placement, product_shape, None)

# --- Add properties ---
property_values = [
    ifcfile.createIfcPropertySingleValue(
        "Reference", "Reference",
        ifcfile.create_entity("IfcText", "Describe the Reference"), None),
    ifcfile.createIfcPropertySingleValue(
        "IsExternal", "IsExternal",
        ifcfile.create_entity("IfcBoolean", True), None),
    ifcfile.createIfcPropertySingleValue(
        "ThermalTransmittance", "ThermalTransmittance",
        ifcfile.create_entity("IfcReal", 2.569), None),
]
property_set = ifcfile.createIfcPropertySet(
    create_guid(), owner_history, "Pset_WallCommon", None, property_values)
ifcfile.createIfcRelDefinesByProperties(
    create_guid(), owner_history, None, None, [wall], property_set)

# --- Add quantities ---
quantity_values = [
    ifcfile.createIfcQuantityLength("Length", "Length of the wall", None, 5.0),
    ifcfile.createIfcQuantityArea("Area", "Area of the front face", None,
                                   5.0 * solid.Depth),
    ifcfile.createIfcQuantityVolume("Volume", "Volume of the wall", None,
                                     5.0 * solid.Depth * 0.2),
]
element_quantity = ifcfile.createIfcElementQuantity(
    create_guid(), owner_history, "BaseQuantities", None, None, quantity_values)
ifcfile.createIfcRelDefinesByProperties(
    create_guid(), owner_history, None, None, [wall], element_quantity)

# --- Spatial containment ---
ifcfile.createIfcRelContainedInSpatialStructure(
    create_guid(), owner_history, "Building Storey Container", None,
    [wall], building_storey)

# --- Write to disk ---
ifcfile.write(filename)
print(f"IFC file written to {filename}")
```

### Example 3: Open, Modify, and Save

```python
import ifcopenshell

# Open existing file
model = ifcopenshell.open('existing_model.ifc')

# Query elements
walls = model.by_type('IfcWall')
print(f"Found {len(walls)} walls")

# Modify an element
for wall in walls:
    wall.Name = f"Renamed: {wall.Name}"

# Get a specific element by ID
element = model.by_id(42)
print(f"Element #42 is: {element.is_a()}")

# Get element by GlobalId
element = model.by_guid('0EI0MSHbX9gg8Fxwar7lL8')

# Save modifications to a new file
model.write('modified_model.ifc')
```

### Example 4: Copy Elements Between Files

```python
import ifcopenshell

# Open source file
source = ifcopenshell.open('source.ifc')

# Create target file with same schema
target = ifcopenshell.file(schema=source.schema)

# Copy specific elements (and all dependencies)
for wall in source.by_type('IfcWall'):
    target.add(wall)

# Write target file
target.write('walls_only.ifc')
```

### Example 5: Iterate and Inspect All Entities

```python
import ifcopenshell

model = ifcopenshell.open('model.ifc')

# Iterate through all entities
for entity in model:
    print(f"#{entity.id()} = {entity.is_a()}")
    info = entity.get_info()
    print(info)

    # Check for named entities
    if hasattr(entity, "Name") and entity.Name:
        print(f"  Name: {entity.Name}")
```

---

## 9. Sources

- [IfcOpenShell Official Documentation](https://docs.ifcopenshell.org/)
- [Hello World - IfcOpenShell 0.8.4 Documentation](https://docs.ifcopenshell.org/ifcopenshell-python/hello_world.html)
- [Code Examples - IfcOpenShell 0.8.4 Documentation](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)
- [ifcopenshell Module API - IfcOpenShell 0.8.4](https://docs.ifcopenshell.org/autoapi/ifcopenshell/index.html)
- [ifcopenshell.file API - IfcOpenShell 0.8.4](https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html)
- [ifcopenshell.api.root.create_entity API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/root/create_entity/index.html)
- [ifcopenshell.api.project.create_file API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/project/create_file/index.html)
- [IfcOpenShell file.py Source (GitHub)](https://github.com/IfcOpenShell/IfcOpenShell/blob/master/src/ifcopenshell-python/ifcopenshell/file.py)
- [IfcOpenShell __init__.py Source (GitHub)](https://github.com/IfcOpenShell/IfcOpenShell/blob/v0.7.0/src/ifcopenshell-python/ifcopenshell/__init__.py)
- [IfcOpenShell Academy](https://academy.ifcopenshell.org/)
- [Creating a Simple Wall Tutorial (Academy)](https://academy.ifcopenshell.org/posts/creating-a-simple-wall-with-property-set-and-quantity-information/)
- [Using IfcOpenShell to Parse IFC Files (ThinkMoult)](https://thinkmoult.com/using-ifcopenshell-parse-ifc-files-python.html)
- [How to Create Entity (OSArch Community)](https://community.osarch.org/discussion/490/how-to-create-entity-with-ifcopenshell)
- [IfcOpenHouse Tutorial (OSArch Community)](https://community.osarch.org/discussion/1471/ifcopenhouse-step-by-step-tutorial-with-the-ifcopenshell-python-api)
- [IfcOpenShell Code Examples (OSArch Wiki)](https://wiki.osarch.org/index.php?title=IfcOpenShell_code_examples)
