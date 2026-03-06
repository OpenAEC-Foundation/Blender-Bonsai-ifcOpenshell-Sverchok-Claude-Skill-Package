# Schema Error Methods Reference

Complete method signatures for schema introspection, validation, and migration in IfcOpenShell.

---

## Schema Introspection Methods

### ifcopenshell.ifcopenshell_wrapper.schema_by_name()

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schema_name` | str | YES | `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |

**Returns**: Schema object for entity introspection.

**Raises**: `RuntimeError` if schema_name is not recognized.

```python
import ifcopenshell

schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
print(type(schema))  # <class 'ifcopenshell.ifcopenshell_wrapper.schema_definition'>
```

---

### schema.declaration_by_name()

```python
entity_decl = schema.declaration_by_name(entity_name)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entity_name` | str | YES | IFC entity name (e.g., `"IfcWall"`) |

**Returns**: Entity declaration object.

**Raises**: `RuntimeError` if entity does not exist in this schema.

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4X3")

# Check if entity exists
try:
    decl = schema.declaration_by_name("IfcBuiltElement")
    print("IfcBuiltElement exists in IFC4X3")
except RuntimeError:
    print("IfcBuiltElement does NOT exist in IFC4X3")

# This would raise RuntimeError:
# schema.declaration_by_name("IfcBuildingElement")  # Not in IFC4X3
```

---

### entity.all_attributes()

```python
attributes = entity_decl.all_attributes()
```

**Returns**: List of attribute declaration objects (includes inherited attributes).

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
entity = schema.declaration_by_name("IfcWall")
for attr in entity.all_attributes():
    print(f"  {attr.name()}: optional={attr.optional()}")

# Output includes: GlobalId, OwnerHistory, Name, Description,
#                  ObjectType, ObjectPlacement, Representation, Tag, PredefinedType
```

---

### entity.is_abstract()

```python
is_abstract = entity_decl.is_abstract()
```

**Returns**: `bool` — True if entity is abstract and cannot be instantiated.

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4")
print(schema.declaration_by_name("IfcElement").is_abstract())    # True
print(schema.declaration_by_name("IfcWall").is_abstract())       # False
```

---

### entity.supertype()

```python
parent = entity_decl.supertype()
```

**Returns**: Parent entity declaration, or None for root entities.

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4X3")
entity = schema.declaration_by_name("IfcBuiltElement")
parent = entity.supertype()
print(parent.name())  # "IfcElement"
```

---

### entity.subtypes()

```python
children = entity_decl.subtypes()
```

**Returns**: List of direct subtype entity declarations.

```python
schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name("IFC4X3")
entity = schema.declaration_by_name("IfcFacility")
for sub in entity.subtypes():
    print(sub.name())
# Output: IfcBridge, IfcBuilding, IfcMarineFacility, IfcRailway, IfcRoad
```

---

## File Schema Properties

### model.schema

```python
schema_id = model.schema
```

**Returns**: `str` — `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"`.

```python
model = ifcopenshell.open("building.ifc")
print(model.schema)  # "IFC4"
```

---

### model.schema_identifier

```python
schema_id = model.schema_identifier
```

**Returns**: `str` — More specific schema identifier (e.g., `"IFC4_ADD2"`).

---

### model.schema_version

```python
version_tuple = model.schema_version
```

**Returns**: `tuple` — Numeric version (e.g., `(4, 0, 2, 1)` for IFC4 ADD2 TC1).

---

## File Creation with Schema

### ifcopenshell.file()

```python
model = ifcopenshell.file(schema=schema_name)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `schema` | str | NO | `"IFC4"` | `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |
| `schema_version` | tuple | NO | None | Specific sub-version, e.g. `(4, 0, 2, 1)` |

**ALWAYS** specify `schema=` explicitly. NEVER rely on the default.

---

### ifcopenshell.api.run("project.create_file")

```python
model = ifcopenshell.api.run("project.create_file", version=schema_name)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `version` | str | YES | `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |

**Note**: The parameter is `version=` (not `schema=`). This is different from `ifcopenshell.file()`.

---

## model.by_type()

```python
entities = model.by_type(ifc_type, include_subtypes=True)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `ifc_type` | str | YES | — | IFC entity name |
| `include_subtypes` | bool | NO | True | Include subtypes in results |

**Raises**: `RuntimeError` if `ifc_type` does not exist in the file's schema.

**ALWAYS** verify the entity exists in the target schema before calling `by_type()`.

---

## Migration Methods (ifcpatch)

### ifcpatch.execute()

```python
output = ifcpatch.execute(patch_config)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `patch_config` | dict | YES | Configuration dictionary (see below) |

**patch_config keys:**

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `"input"` | str | YES | Input file path |
| `"file"` | ifcopenshell.file | YES | Opened IFC file object |
| `"recipe"` | str | YES | Recipe name (e.g., `"Migrate"`) |
| `"arguments"` | list | YES | Recipe-specific arguments |

**Migrate recipe arguments**: `["IFC4"]` or `["IFC4X3"]` — the target schema version.

**Returns**: Modified `ifcopenshell.file` object.

```python
import ifcopenshell
import ifcpatch

model = ifcopenshell.open("input.ifc")
output = ifcpatch.execute({
    "input": "input.ifc",
    "file": model,
    "recipe": "Migrate",
    "arguments": ["IFC4"]
})
```

---

### ifcpatch.write()

```python
ifcpatch.write(output, output_path)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `output` | ifcopenshell.file | YES | Result from `ifcpatch.execute()` |
| `output_path` | str | YES | Output file path |

```python
ifcpatch.write(output, "migrated_output.ifc")
```

---

## Validation Methods

### ifcopenshell.validate.validate()

```python
ifcopenshell.validate.validate(model, logger)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `model` | ifcopenshell.file | YES | IFC file to validate |
| `logger` | logger object | YES | Logger to collect validation results |

```python
import ifcopenshell.validate

model = ifcopenshell.open("model.ifc")
logger = ifcopenshell.validate.json_logger()
ifcopenshell.validate.validate(model, logger)

for error in logger.statements:
    print(f"{error['severity']}: {error['message']}")
```

---

### ifcopenshell.validate.json_logger()

```python
logger = ifcopenshell.validate.json_logger()
```

**Returns**: Logger object that collects validation errors as structured data.

**logger.statements**: List of dicts with keys:
- `"severity"`: `"Error"` or `"Warning"`
- `"message"`: Human-readable error description
- `"instance"`: Entity reference (if applicable)

---

## Entity Existence Check (Helper)

This is a utility pattern, not a built-in method:

```python
def entity_exists_in_schema(schema_name, entity_name):
    """Check if an IFC entity exists in a given schema version.

    Parameters:
        schema_name: "IFC2X3", "IFC4", or "IFC4X3"
        entity_name: IFC entity name (e.g., "IfcWall")

    Returns:
        bool: True if entity exists in the schema
    """
    schema = ifcopenshell.ifcopenshell_wrapper.schema_by_name(schema_name)
    try:
        schema.declaration_by_name(entity_name)
        return True
    except RuntimeError:
        return False
```

---

## Schema Identifier Constants

| Schema | model.schema | file() param | create_file() param |
|--------|-------------|-------------|-------------------|
| IFC 2x3 TC1 | `"IFC2X3"` | `schema="IFC2X3"` | `version="IFC2X3"` |
| IFC4 ADD2 TC1 | `"IFC4"` | `schema="IFC4"` | `version="IFC4"` |
| IFC4.3 (IFC4x3) | `"IFC4X3"` | `schema="IFC4X3"` | `version="IFC4X3"` |

**ALWAYS** use uppercase identifiers. `"ifc4"` and `"IFC4"` may not be interchangeable depending on the IfcOpenShell version.
