# Project Management API Reference

> **Version**: IfcOpenShell v0.8.x | IFC2X3 / IFC4 / IFC4X3
> **Call pattern**: `ifcopenshell.api.<module>.<function>(model, **kwargs)`

## ifcopenshell.api.project

### create_file

Creates a new IFC file with required IfcProject entity, default units, and representation contexts.

```python
ifcopenshell.api.project.create_file(version="IFC4") -> ifcopenshell.file
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `version` | `str` | `"IFC4"` | IFC schema version: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |

**Returns**: `ifcopenshell.file` — New IFC file instance with IfcProject pre-created.

**Notes**:
- ALWAYS prefer this over `ifcopenshell.file(schema=...)` — the low-level constructor does NOT create IfcProject, units, or contexts.
- The created file includes: IfcProject entity, IfcUnitAssignment (default SI), IfcGeometricRepresentationContext.

---

### assign_declaration

Associates definitions (types, materials, profiles) with an IfcProject or IfcProjectLibrary context.

```python
ifcopenshell.api.project.assign_declaration(
    model,
    definitions,          # list[entity] — objects to declare
    relating_context      # entity — IfcProject or IfcProjectLibrary
) -> entity | None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `definitions` | `list[entity]` | Objects to associate with the context |
| `relating_context` | `entity` | IfcProject or IfcProjectLibrary instance |

**Returns**: `IfcRelDeclares` entity, or `None` if already declared.

**Note**: IFC4+ only. IFC2X3 does not have `IfcRelDeclares`.

---

### unassign_declaration

Removes definitions from an IfcProject or IfcProjectLibrary context.

```python
ifcopenshell.api.project.unassign_declaration(
    model,
    definitions,          # list[entity]
    relating_context      # entity
) -> None
```

---

### append_asset

Transfers an asset from a library file to the current project file.

```python
ifcopenshell.api.project.append_asset(
    model,
    library,                              # ifcopenshell.file — source library
    element,                              # entity — element in library to copy
    reuse_identities=None,                # dict | None — mapping of old->new IDs
    assume_asset_uniqueness_by_name=True  # bool
) -> entity
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `library` | `ifcopenshell.file` | — | Source library file |
| `element` | `entity` | — | Element to copy from library |
| `reuse_identities` | `dict \| None` | `None` | Identity reuse mapping for batch operations |
| `assume_asset_uniqueness_by_name` | `bool` | `True` | Skip duplicates if name matches |

**Returns**: The newly created entity in the project file.

**Supported asset types**: IfcTypeProduct, IfcProduct, IfcMaterial, IfcProfileDef, IfcCostSchedule, IfcGroup.

---

### edit_header

Edits the IFC file header metadata (STEP format header section).

```python
ifcopenshell.api.project.edit_header(
    model,
    editor=None,              # str — person/entity editing the file
    organization=None,        # str — organization name
    application=None,         # str — application name
    application_version=None  # str — application version
) -> None
```

---

## ifcopenshell.api.unit

### assign_unit

Assigns default measurement units to the project (IfcUnitAssignment).

```python
ifcopenshell.api.unit.assign_unit(
    model,
    units=None,     # list[entity] | None — explicit unit entities
    length=None,    # dict | None — {"is_metric": True, "raw": "MILLIMETRE"}
    area=None,      # dict | None — {"is_metric": True, "raw": "SQUARE_METRE"}
    volume=None     # dict | None — {"is_metric": True, "raw": "CUBIC_METRE"}
) -> IfcUnitAssignment
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `units` | `list[entity] \| None` | `None` | Pre-created unit entities to assign |
| `length` | `dict \| None` | `None` | Shorthand length unit spec |
| `area` | `dict \| None` | `None` | Shorthand area unit spec |
| `volume` | `dict \| None` | `None` | Shorthand volume unit spec |

**Returns**: `IfcUnitAssignment` entity.

**Behaviour when all parameters are None**: Creates default SI units (MILLIMETRE length, SQUARE_METRE area, CUBIC_METRE volume, RADIAN plane angle, STERADIAN solid angle).

---

### add_si_unit

Creates an SI measurement unit.

```python
ifcopenshell.api.unit.add_si_unit(
    model,
    unit_type="LENGTHUNIT",  # str — IfcUnitEnum value
    prefix=None              # str | None — SI prefix
) -> IfcSIUnit
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `unit_type` | `str` | `"LENGTHUNIT"` | One of: `LENGTHUNIT`, `AREAUNIT`, `VOLUMEUNIT`, `PLANEANGLEUNIT`, `SOLIDANGLEUNIT`, `MASSUNIT`, `TIMEUNIT`, `THERMODYNAMICTEMPERATUREUNIT`, `LUMINOUSINTENSITYUNIT`, `FREQUENCYUNIT`, `FORCEUNIT`, `PRESSUREUNIT`, `ENERGYUNIT`, `POWERUNIT`, `ELECTRICCURRENTUNIT` |
| `prefix` | `str \| None` | `None` | One of: `"EXA"`, `"PETA"`, `"TERA"`, `"GIGA"`, `"MEGA"`, `"KILO"`, `"HECTO"`, `"DECA"`, `None` (base), `"DECI"`, `"CENTI"`, `"MILLI"`, `"MICRO"`, `"NANO"`, `"PICO"`, `"FEMTO"`, `"ATTO"` |

**Returns**: `IfcSIUnit` entity.

---

### add_conversion_based_unit

Creates a conversion-based (non-SI) unit.

```python
ifcopenshell.api.unit.add_conversion_based_unit(
    model,
    name="foot",              # str — standard conversion name
    conversion_offset=None    # float | None — offset for temperature scales
) -> IfcConversionBasedUnit
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | `"foot"` | Recognized names: `"foot"`, `"inch"`, `"yard"`, `"mile"`, `"square foot"`, `"square inch"`, `"square yard"`, `"cubic foot"`, `"cubic inch"`, `"cubic yard"`, `"degree"` |
| `conversion_offset` | `float \| None` | `None` | Non-zero only for Fahrenheit/Rankine |

**Returns**: `IfcConversionBasedUnit` entity.

---

### add_monetary_unit

Creates a monetary (currency) unit.

```python
ifcopenshell.api.unit.add_monetary_unit(
    model,
    currency="DOLLARYDOO"  # str — ISO 4217 currency code
) -> IfcMonetaryUnit
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `currency` | `str` | `"DOLLARYDOO"` | ISO 4217 code (e.g., `"EUR"`, `"USD"`, `"GBP"`, `"CHF"`, `"JPY"`) |

**Returns**: `IfcMonetaryUnit` entity.

---

### add_context_dependent_unit

Creates a custom project-specific unit.

```python
ifcopenshell.api.unit.add_context_dependent_unit(
    model,
    unit_type,     # str — IfcUnitEnum value
    name,          # str — custom unit name
    dimensions     # tuple — dimensional exponents (7 ints)
) -> IfcContextDependentUnit
```

---

### edit_named_unit

Modifies a named unit's attributes.

```python
ifcopenshell.api.unit.edit_named_unit(
    model,
    unit,        # entity — the unit to edit
    attributes   # dict — attribute-value pairs
) -> None
```

---

### edit_monetary_unit

Modifies a monetary unit's attributes.

```python
ifcopenshell.api.unit.edit_monetary_unit(
    model,
    unit,        # entity — IfcMonetaryUnit
    attributes   # dict — attribute-value pairs
) -> None
```

---

### remove_unit

Deletes a unit entity from the model.

```python
ifcopenshell.api.unit.remove_unit(model, unit) -> None
```

---

### unassign_unit

Removes units from the project's IfcUnitAssignment without deleting the unit entities.

```python
ifcopenshell.api.unit.unassign_unit(model, units) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `units` | `list[entity]` | Unit entities to unassign |

---

## ifcopenshell.api.georeference

### add_georeferencing

Creates empty georeferencing entities on the model.

```python
ifcopenshell.api.georeference.add_georeferencing(
    model,
    ifc_class="IfcMapConversion",  # str — "IfcMapConversion" or "IfcMapConversionScaled" (IFC4X3 only)
    name="EPSG:3857"               # str — initial CRS name
) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifc_class` | `str` | `"IfcMapConversion"` | Map conversion entity class. `"IfcMapConversionScaled"` only valid for IFC4X3. |
| `name` | `str` | `"EPSG:3857"` | Initial CRS name (will be overridden by `edit_georeferencing`) |

**IFC2X3 behaviour**: Creates `ePSet_MapConversion` and `ePSet_ProjectedCRS` property sets on IfcProject instead of entities.

---

### edit_georeferencing

Modifies map conversion parameters and projected CRS attributes.

```python
ifcopenshell.api.georeference.edit_georeferencing(
    model,
    coordinate_operation=None,  # dict | None
    projected_crs=None          # dict | None
) -> None
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `coordinate_operation` | `dict \| None` | Map conversion attributes (see below) |
| `projected_crs` | `dict \| None` | CRS attributes (see below) |

**coordinate_operation dict keys**:

| Key | Type | Description |
|-----|------|-------------|
| `"Eastings"` | `float` | False origin easting (map X offset) |
| `"Northings"` | `float` | False origin northing (map Y offset) |
| `"OrthogonalHeight"` | `float` | Height above reference datum |
| `"XAxisAbscissa"` | `float` | cos(rotation_angle) — Project North rotation |
| `"XAxisOrdinate"` | `float` | sin(rotation_angle) — Project North rotation |
| `"Scale"` | `float` | Scale factor (usually 1.0) |

**projected_crs dict keys**:

| Key | Type | Description |
|-----|------|-------------|
| `"Name"` | `str` | EPSG code (e.g., `"EPSG:28992"`) |
| `"Description"` | `str` | Human-readable name |
| `"GeodeticDatum"` | `str` | Datum name |
| `"VerticalDatum"` | `str` | Vertical datum name |
| `"MapProjection"` | `str` | Projection method name |
| `"MapZone"` | `str` | UTM zone or grid zone |

---

### edit_true_north

Sets the true north direction relative to project Y-axis.

```python
ifcopenshell.api.georeference.edit_true_north(
    model,
    true_north=0.0  # float | tuple — angle in degrees or (x, y) vector
) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `true_north` | `float \| tuple` | `0.0` | Angle in degrees anticlockwise from Y-axis, or `(x, y)` direction vector |

---

### edit_wcs

Adjusts the World Coordinate System origin and rotation.

```python
ifcopenshell.api.georeference.edit_wcs(
    model,
    x=0.0,          # float — WCS origin X
    y=0.0,          # float — WCS origin Y
    z=0.0,          # float — WCS origin Z
    rotation=0.0,   # float — rotation in degrees
    is_si=True      # bool — coordinates in SI (meters)
) -> None
```

**IMPORTANT**: Keep WCS at origin (0,0,0) unless there is a specific surveying requirement. Moving WCS affects ALL local placements in the model.

---

### remove_georeferencing

Removes all georeferencing data from the model.

```python
ifcopenshell.api.georeference.remove_georeferencing(model) -> None
```

---

## ifcopenshell.api.context

### add_context

Creates a geometric representation context or sub-context.

```python
ifcopenshell.api.context.add_context(
    model,
    context_type="Model",           # str — "Model" or "Plan"
    context_identifier=None,        # str | None — sub-context identifier
    target_view=None,               # str | None — target view type
    target_scale=None,              # float | None — target scale factor
    parent=None                     # entity | None — parent context
) -> IfcGeometricRepresentationContext | IfcGeometricRepresentationSubContext
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context_type` | `str` | `"Model"` | `"Model"` (3D) or `"Plan"` (2D) |
| `context_identifier` | `str \| None` | `None` | `"Body"`, `"Axis"`, `"Box"`, `"FootPrint"`, `"Clearance"`, `"Profile"`, `"Annotation"` |
| `target_view` | `str \| None` | `None` | `"MODEL_VIEW"`, `"PLAN_VIEW"`, `"GRAPH_VIEW"`, `"SKETCH_VIEW"`, `"REFLECTED_PLAN_VIEW"`, `"SECTION_VIEW"`, `"ELEVATION_VIEW"` |
| `target_scale` | `float \| None` | `None` | Scale factor for the view |
| `parent` | `entity \| None` | `None` | Parent context (required for sub-contexts) |

**Returns**: `IfcGeometricRepresentationContext` (if no parent) or `IfcGeometricRepresentationSubContext` (if parent specified).

---

### edit_context

Modifies attributes of a representation context.

```python
ifcopenshell.api.context.edit_context(
    model,
    context,      # entity — context to edit
    attributes    # dict — attribute-value pairs
) -> None
```

---

### remove_context

Removes a representation context and all geometry assigned to it.

```python
ifcopenshell.api.context.remove_context(model, context) -> None
```

**WARNING**: Removing a context deletes ALL representations using that context. Use with extreme caution.

---

## ifcopenshell.api.owner

### add_person

Creates an IfcPerson entity.

```python
ifcopenshell.api.owner.add_person(
    model,
    identification="HArtiworker",  # str — unique ID
    family_name="Hartworker",       # str
    given_name="Harry"              # str
) -> IfcPerson
```

---

### add_organisation

Creates an IfcOrganization entity.

```python
ifcopenshell.api.owner.add_organisation(
    model,
    identification="ACME",  # str — unique ID
    name="ACME Corp"         # str — organization name
) -> IfcOrganization
```

---

### add_person_and_organisation

Links a person to an organisation.

```python
ifcopenshell.api.owner.add_person_and_organisation(
    model,
    person,        # entity — IfcPerson
    organisation   # entity — IfcOrganization
) -> IfcPersonAndOrganization
```

---

### add_application

Registers an application identity.

```python
ifcopenshell.api.owner.add_application(
    model,
    application_developer=None,         # entity | None — IfcOrganization
    version=None,                       # str | None — app version
    application_full_name="IfcOpenShell",  # str
    application_identifier="IfcOpenShell"  # str
) -> IfcApplication
```

---

### create_owner_history

Creates an IfcOwnerHistory entity. Typically called automatically when creating entities.

```python
ifcopenshell.api.owner.create_owner_history(model) -> IfcOwnerHistory | None
```

**Note**: Returns `None` if `settings.get_user` or `settings.get_application` callbacks are not configured.

---

### settings (module-level configuration)

```python
# Configure owner history auto-creation
ifcopenshell.api.owner.settings.get_user = lambda ifc: person_and_org_entity
ifcopenshell.api.owner.settings.get_application = lambda ifc: application_entity
```

**IMPORTANT**: Owner history is MANDATORY in IFC2X3. In IFC4+, it is optional but strongly recommended for production deliverables. Configure these callbacks before creating any entities.

---

## ifcopenshell.api.root

### create_entity

Creates a new rooted IFC entity with auto-generated GlobalId and owner history.

```python
ifcopenshell.api.root.create_entity(
    model,
    ifc_class="IfcBuildingElementProxy",  # str — IFC entity class
    predefined_type=None,                 # str | None — PredefinedType enum
    name=None                             # str | None — entity name
) -> entity
```

---

### remove_product

Removes a product and all its dependent relationships.

```python
ifcopenshell.api.root.remove_product(model, product) -> None
```

---

## ifcopenshell.api.aggregate

### assign_object

Assigns products to an aggregate parent (spatial containment hierarchy).

```python
ifcopenshell.api.aggregate.assign_object(
    model,
    relating_object,  # entity — parent container
    products          # list[entity] — children to assign
) -> IfcRelAggregates | None
```

---

### unassign_object

Removes products from their aggregate parent.

```python
ifcopenshell.api.aggregate.unassign_object(
    model,
    products  # list[entity] — children to unassign
) -> None
```

---

## ifcopenshell Low-Level File I/O

### ifcopenshell.open

Opens an existing IFC file from disk.

```python
ifcopenshell.open(path) -> ifcopenshell.file
```

### ifcopenshell.file (constructor)

Creates a blank IFC file WITHOUT project setup.

```python
ifcopenshell.file(schema="IFC4") -> ifcopenshell.file
```

**WARNING**: NEVER use this for project creation. Use `ifcopenshell.api.project.create_file()` instead.

### ifcopenshell.file.write

Writes the IFC model to disk.

```python
model.write(path) -> None
```

### ifcopenshell.file.schema

Returns the schema identifier string.

```python
model.schema  # "IFC2X3", "IFC4", or "IFC4X3"
```

### ifcopenshell.file.by_type

Returns all entities of a given IFC class.

```python
model.by_type("IfcProject") -> list[entity]
```

---

## Bonsai Operators (Inside Blender)

### bpy.ops.bim.create_project

Creates a new IFC project within Blender.

```python
bpy.ops.bim.create_project(template="metric_m")
```

| Template Value | Description |
|---------------|-------------|
| `"metric_m"` | Metric project with metres |
| `"metric_mm"` | Metric project with millimetres |
| `"imperial_ft"` | Imperial project with feet |
| `"demo"` | Demo project with pre-populated content |

---

### bpy.ops.bim.load_project

Opens an IFC file in Blender.

```python
bpy.ops.bim.load_project(filepath="/path/to/project.ifc")
```

---

### bpy.ops.bim.save_project

Saves the current IFC project.

```python
bpy.ops.bim.save_project(filepath="/path/to/project.ifc")
```

If `filepath` is omitted, saves to the current `IfcStore.path`.

---

## Schema Migration

### ifcopenshell.util.schema.Migrator

Migrates IFC files between schema versions.

```python
import ifcopenshell.util.schema

migrator = ifcopenshell.util.schema.Migrator()
migrated_model = migrator.migrate(source_model, target_schema)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `source_model` | `ifcopenshell.file` | Source IFC file |
| `target_schema` | `str` | Target schema: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |

**Returns**: New `ifcopenshell.file` instance in the target schema.
