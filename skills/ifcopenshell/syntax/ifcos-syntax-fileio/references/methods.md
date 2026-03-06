# File I/O Method Signatures

Complete API reference for IfcOpenShell file I/O operations. All signatures verified against the [official IfcOpenShell documentation](https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html).

---

## ifcopenshell.open()

Opens an existing IFC file from disk and returns an `ifcopenshell.file` object.

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
| `format` | `str \| None` | `None` | Force format: `".ifc"`, `".ifcZIP"`, `".ifcXML"`, `".ifcJSON"`, `".ifcSQLite"`. Auto-detected from extension when `None` |
| `should_stream` | `bool` | `False` | Enable streaming mode for large files (reduces memory, sequential access only) |

### Return Value

`ifcopenshell.file` — The loaded IFC model.

### Exceptions

| Exception | Condition |
|-----------|-----------|
| `FileNotFoundError` | File does not exist at the given path |
| `ifcopenshell.Error` | Generic IFC parsing error |
| `ifcopenshell.SchemaError` | IFC schema not recognized |

### Supported Formats

| Extension | Description |
|-----------|-------------|
| `.ifc` | IFC-SPF (STEP Physical File) — standard text format |
| `.ifcXML` | IFC in XML serialization |
| `.ifcZIP` | Compressed IFC archive |
| `.ifcJSON` | IFC in JSON format |
| `.ifcSQLite` | IFC in SQLite database format |

---

## ifcopenshell.file()

Creates a new, empty IFC file in memory.

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
| `f` | `ifcopenshell_wrapper.file \| None` | `None` | Internal C++ file wrapper (internal use only) |
| `schema` | `str \| None` | `"IFC4"` | IFC schema: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |
| `schema_version` | `tuple[int,int,int,int] \| None` | `None` | Specific version as `(major, minor, addendum, corrigendum)` |

### Return Value

`ifcopenshell.file` — A new blank file with no entities.

### Internal Initialization

The constructor initializes transaction tracking:
- `history_size = 64` (maximum undo steps)
- `history = []` (undo log)
- `future = []` (redo log)
- `transaction = None` (current transaction context)

**WARNING:** The created file has NO header metadata, NO project, NO units. For production use, ALWAYS use `ifcopenshell.api.project.create_file()` instead.

---

## ifcopenshell.api.project.create_file()

Creates a new IFC file with pre-configured header metadata. This is the **recommended** method for production IFC creation.

```python
ifcopenshell.api.project.create_file(
    version: str = "IFC4"
) -> ifcopenshell.file
```

**Alternative call syntax via `api.run()`:**

```python
ifcopenshell.api.run("project.create_file", version="IFC4")
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `version` | `str` | `"IFC4"` | IFC schema version: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |

### Return Value

`ifcopenshell.file` — An initialized file with pre-configured header metadata.

### Auto-Configured Header Fields

| Field | Value |
|-------|-------|
| Preprocessor | `"IfcOpenShell"` |
| MVD | `"DesignTransferView"` (default) |
| Timestamp | Current timestamp |
| FILE_DESCRIPTION | Pre-populated |
| FILE_NAME | Pre-populated |
| FILE_SCHEMA | Matches `version` parameter |

---

## file.write()

Exports the IFC model to a file on disk.

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
| `path` | `os.PathLike \| str` | *required* | Output file path. Parent directories are created automatically |
| `format` | `str \| None` | `None` | Force format: `".ifc"`, `".ifcXML"`, `".ifcZIP"`. Auto-detected from extension when `None` |
| `zipped` | `bool` | `False` | Compress the output file using ZIP_DEFLATED |

### Return Value

`None` — Writes the file to disk as a side effect.

### Behavior

1. Parent directories are created automatically (`os.makedirs(exist_ok=True)`)
2. Format is guessed from extension when `format=None`
3. When `zipped=True`, wraps output in a ZIP archive

---

## file.to_string()

Serializes the entire IFC model to a STEP-SPF formatted string.

```python
file.to_string() -> str
```

### Parameters

None.

### Return Value

`str` — The complete IFC file content as a STEP-SPF formatted string.

---

## file.add()

Incorporates an entity and all its referenced/dependent entities into the file.

```python
file.add(
    inst: ifcopenshell.entity_instance,
    _id: int = None
) -> ifcopenshell.entity_instance
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `inst` | `ifcopenshell.entity_instance` | *required* | The entity to add (from another file) |
| `_id` | `int \| None` | `None` | Explicit STEP entity ID to assign (auto-assigned when `None`) |

### Return Value

`ifcopenshell.entity_instance` — The added entity instance in the target file.

### Behavior

- Recursively adds all referenced entities
- Duplicate additions (checked by `.identity()`) are silently skipped
- The entity and its entire dependency graph are copied

---

## file.remove()

Deletes an IFC entity from the file.

```python
file.remove(
    inst: ifcopenshell.entity_instance
) -> None
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `inst` | `ifcopenshell.entity_instance` | *required* | The entity to remove |

### Return Value

`None`

### Behavior

- Attributes in other entities referencing the deleted entity become null (`$`)
- References within aggregate types (lists/sets) are removed automatically
- Does NOT clean up relationships (spatial containment, property sets, etc.)

**WARNING:** For safe removal of products with relationship cleanup, use `ifcopenshell.api.run("root.remove_product", model, product=entity)` instead.

---

## Transaction Methods

### file.begin_transaction()

Starts a new transaction block for tracking modifications.

```python
file.begin_transaction() -> None
```

**NEVER** nest transactions. ALWAYS call `end_transaction()` or `discard_transaction()` before starting a new one.

### file.end_transaction()

Finalizes the current transaction and adds it to the undo history.

```python
file.end_transaction() -> None
```

### file.discard_transaction()

Cancels the active transaction without applying changes to the undo history.

```python
file.discard_transaction() -> None
```

### file.undo()

Reverts the most recent transaction from the history.

```python
file.undo() -> None
```

**NEVER** call `undo()` while a transaction is active. ALWAYS call `end_transaction()` first.

### file.redo()

Reapplies a previously undone transaction.

```python
file.redo() -> None
```

**NEVER** call `redo()` while a transaction is active.

### file.set_history_size()

Configures the maximum number of transaction states retained.

```python
file.set_history_size(size: int) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `size` | `int` | `64` | Maximum number of undo steps to retain |

---

## File Properties

| Property | Type | Description |
|----------|------|-------------|
| `file.schema` | `str` | General schema: `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |
| `file.schema_identifier` | `str` | Full version string (e.g., `"IFC4_ADD2"`) |
| `file.schema_version` | `tuple[int,int,int,int]` | Numeric version (major, minor, addendum, corrigendum) |
| `file.header` | `file_header` | Access to file header metadata |
| `file.history` | `list[Transaction]` | Undo history (oldest to newest) |
| `file.future` | `list[Transaction]` | Redo stack (newest to oldest) |

---

## Batch Operations

### file.batch()

Initiates a low-level batching mechanism that accelerates deletion of large entity hierarchies.

```python
file.batch() -> None
```

### file.unbatch()

Concludes the batching operation.

```python
file.unbatch() -> None
```

**ALWAYS** pair `batch()` with `unbatch()`. Use batch mode only when deleting large numbers of entities for performance optimization.
