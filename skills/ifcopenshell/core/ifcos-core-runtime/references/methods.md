# ifcos-core-runtime — API Method Signatures

## ifcopenshell.entity_instance

The core wrapper class for all IFC entities. Every entity returned by IfcOpenShell queries is an `entity_instance`.

### Attribute Access

```python
entity.AttributeName          # Returns attribute value (PascalCase). Returns None for IFC $null.
entity[index]                 # Returns attribute by positional index (0-based). AVOID — index varies by type/schema.
entity.wrapped_data           # Returns the underlying C++ ifcopenshell_wrapper.entity_instance object.
entity.file                   # Returns the parent ifcopenshell.file object.
```

### Identity and Type Checking

```python
entity.id() -> int
# Returns the STEP file entity ID (e.g., #42 → 42).
# Unique within a single file. Use for identity comparison.

entity.is_a(ifc_class: str = None) -> Union[bool, str]
# With argument: Returns True if entity is an instance of ifc_class or any subclass.
#   entity.is_a("IfcWall")              → True
#   entity.is_a("IfcBuildingElement")   → True (parent class)
#   entity.is_a("IfcSlab")             → False
# Without argument: Returns the entity's IFC class name as a string.
#   entity.is_a()                       → "IfcWall"

entity.is_a() -> str
# Returns the IFC class name. Equivalent to entity.is_a(None).
```

### Equality

```python
entity_a == entity_b -> bool
# Value equality. Compares the underlying wrapped_data C++ objects.
# Returns True if both wrappers refer to the same C++ entity in the same file.
# ALWAYS use == for entity comparison.

entity_a is entity_b -> bool
# Identity check. ALWAYS returns False for separately queried wrappers.
# NEVER use 'is' for entity comparison.
```

### Data Extraction

```python
entity.get_info(
    include_identifier: bool = True,
    recursive: bool = False,
    return_type: type = dict,
    ignore: tuple = (),
    scalar_only: bool = False
) -> dict
# Returns a dictionary of all entity attributes.
# Keys: 'id', 'type', plus all schema-defined attribute names.
#
# Parameters:
#   include_identifier — Include 'id' and 'type' keys (default: True)
#   recursive — Recursively expand referenced entities into dicts (default: False)
#               WARNING: EXTREMELY expensive on large models. Materializes entire entity graph.
#   return_type — Container type for the result (default: dict)
#   ignore — Tuple of attribute names to exclude
#   scalar_only — Only include scalar values, skip entity references (default: False)
#                 Faster than full get_info() for bulk attribute reading.
#
# Example return:
# {"id": 42, "type": "IfcWall", "GlobalId": "2O2Fr$t4X7Zf8NOew3FNr2", "Name": "Wall 001", ...}

entity.get_info_2(
    include_identifier: bool = True,
    recursive: bool = False,
    return_type: type = dict,
    ignore: tuple = ()
) -> dict
# Faster C++ implementation of get_info(). Same signature and behavior.
# Prefer get_info_2() when available (IfcOpenShell 0.7+).

entity.__repr__() -> str
# Returns STEP file representation: '#42=IFCWALL("2O2Fr...",...);'
```

### Inverse References

```python
entity.wrapped_data.get_inverse(
    attribute_name: str
) -> tuple
# Returns entities that reference this entity via the named attribute.
# Low-level; prefer ifcopenshell.util.element functions for common traversals.
```

---

## ifcopenshell.file

The in-memory IFC file object. Owns all entities and manages the C++ data store.

### Querying Entities

```python
model.by_type(
    ifc_class: str,
    include_subtypes: bool = True
) -> tuple
# Returns a TUPLE of all entities matching ifc_class.
# First call for a type builds an internal index (slower).
# Subsequent calls for the same type use the cached index (instant).
#
# Parameters:
#   ifc_class — IFC class name (e.g., "IfcWall", "IfcProduct")
#   include_subtypes — Include entities of subclasses (default: True)
#                      by_type("IfcWall") includes IfcWallStandardCase when True.
#
# Returns: tuple of entity_instance objects. NEVER returns a list.

model.by_id(
    entity_id: int
) -> entity_instance
# Returns the entity with the given STEP file ID.
# Raises RuntimeError if the entity does not exist (e.g., after removal).

model.by_guid(
    guid: str
) -> entity_instance
# Returns the entity with the given GlobalId.
# Raises RuntimeError if no entity has this GUID.

model.__iter__() -> Iterator[entity_instance]
# Iterates over ALL entities in the file. No type filtering.
# Use by_type() instead for type-filtered queries (10-100x faster).

model.__len__() -> int
# Returns the total number of entities in the file.
```

### Entity Lifecycle

```python
model.create_entity(
    ifc_class: str,
    **kwargs
) -> entity_instance
# Creates a new entity directly in the C++ store.
# Low-level: does NOT generate GlobalId, does NOT set OwnerHistory,
# does NOT track for undo, does NOT manage relationships.
# Use ifcopenshell.api.run("root.create_entity", ...) for safe creation.
#
# Parameters:
#   ifc_class — IFC class name (e.g., "IfcWall")
#   **kwargs — Attribute values as keyword arguments (PascalCase names)
#
# Example:
#   wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new(), Name="W1")

model.remove(
    entity: entity_instance
) -> None
# Removes the entity from the C++ store. IMMEDIATELY deallocates the C++ object.
# All Python references to this entity become DANGLING POINTERS.
# Attributes in OTHER entities that referenced the removed entity become null ($).
# Aggregate references (lists/sets) are cleaned up automatically.
#
# WARNING: Does NOT clean up relationships (IfcRelContainedInSpatialStructure, etc.).
# For products, use ifcopenshell.api.run("root.remove_product", ...) instead.
#
# ALWAYS set Python references to None after calling remove().

model.add(
    entity: entity_instance
) -> entity_instance
# Copies an entity (and all its dependencies) from another file into this file.
# Returns the newly created entity in this file.
# The source entity is NOT removed from its original file.
```

### Schema Information

```python
model.schema -> str
# Returns the IFC schema identifier: "IFC2X3", "IFC4", or "IFC4X3".

model.schema_identifier -> str
# Returns the detailed schema identifier, e.g., "IFC4_ADD2".

model.schema_version -> tuple
# Returns the schema version as a tuple, e.g., (4, 0, 2, 1) for IFC4 ADD2 TC1.
```

### Transaction Management

```python
model.begin_transaction() -> None
# Starts recording changes for undo/redo.
# NEVER call undo() or redo() while a transaction is active.

model.end_transaction() -> None
# Ends the current transaction. Changes are recorded for undo.

model.discard_transaction() -> None
# Discards the current transaction without recording.

model.undo() -> None
# Undoes the last completed transaction.
# NEVER call while a transaction is active.

model.redo() -> None
# Redoes the last undone transaction.
# NEVER call while a transaction is active.

model.set_history_size(size: int) -> None
# Sets the maximum number of undo steps. Default: 64.
```

---

## ifcopenshell.guid

```python
ifcopenshell.guid.new() -> str
# Generates a new IFC GlobalId (22-character base64-encoded GUID).
# Use when creating entities via model.create_entity() (not needed with API).

ifcopenshell.guid.expand(guid: str) -> str
# Converts a 22-char IFC GUID to standard 36-char UUID format.

ifcopenshell.guid.compress(uuid: str) -> str
# Converts a 36-char UUID to 22-char IFC GUID format.
```

---

## Key ifcopenshell.api.run() Calls (Runtime-Relevant)

```python
ifcopenshell.api.run("root.create_entity", model,
    ifc_class: str,
    predefined_type: str = None,
    name: str = None
) -> entity_instance
# Creates an entity with automatic GlobalId, OwnerHistory (IFC2X3), and validation.

ifcopenshell.api.run("root.remove_product", model,
    product: entity_instance
) -> None
# Removes a product entity and cleans up all relationships.
# PREFER this over model.remove() for IfcProduct instances.

ifcopenshell.api.run("attribute.edit_attributes", model,
    product: entity_instance,
    attributes: dict
) -> None
# Edits entity attributes with undo tracking.
# Prefer over direct attribute assignment when undo support is needed.
```
