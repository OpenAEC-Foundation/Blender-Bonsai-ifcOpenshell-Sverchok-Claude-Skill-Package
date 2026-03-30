---
name: ifcos-syntax-fileio
description: >
  Use when opening, creating, writing, or serializing IFC files with IfcOpenShell.
  Prevents the common mistake of not using transactions for multi-step operations
  (no undo on failure). Covers ifcopenshell.open(), file.create(), file.write(),
  transaction management with undo/redo, and schema selection.
  Keywords: ifcopenshell.open, file.write, create file, IFC file, transaction, undo, redo, serialize, IFC I/O, file operations, open IFC file, save IFC, read IFC in Python.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell File I/O Operations

## Quick Reference

### Decision Tree: Opening vs Creating IFC Files

```
Need an IFC model?
├── Existing file on disk?
│   └── YES → ifcopenshell.open("path/to/file.ifc")
│       ├── Large file (100MB+)? → use should_stream=True
│       └── Non-standard extension? → use format=".ifc"
│
└── NO → Create new file
    ├── Need header metadata, timestamps, MVD? (production use)
    │   └── YES → ifcopenshell.api.project.create_file(version="IFC4")
    │
    └── Need bare-minimum empty file? (testing, prototyping)
        └── ifcopenshell.file(schema="IFC4")
```

### Decision Tree: Writing Output

```
Need to output IFC data?
├── Write to disk?
│   ├── Standard .ifc → model.write("output.ifc")
│   ├── XML format → model.write("output.ifcXML")
│   ├── Compressed → model.write("output.ifc", zipped=True)
│   └── ZIP archive → model.write("output.ifcZIP")
│
└── Get as string? → model.to_string()
```

### Critical Warnings

- **ALWAYS** specify `schema=` explicitly when calling `ifcopenshell.file()`. The default is `"IFC4"` but relying on implicit defaults is error-prone.
- **ALWAYS** check `model.schema` after opening a file before using schema-specific entities.
- **NEVER** use `ifcopenshell.file()` for production IFC creation. Use `ifcopenshell.api.project.create_file()` instead — it sets up header metadata, timestamps, and MVD automatically.
- **NEVER** call `model.remove()` without understanding that references to the removed entity become null. Use `ifcopenshell.api.run("root.remove_product", ...)` for safe removal of products.
- **ALWAYS** wrap undo-able operations in `begin_transaction()` / `end_transaction()` pairs. NEVER leave a transaction open.
- **NEVER** call `undo()` or `redo()` while a transaction is active.

---

## Essential Patterns

### Pattern 1: Open an Existing IFC File

```python
# IfcOpenShell: all schema versions
import ifcopenshell

model = ifcopenshell.open("/path/to/model.ifc")
print(model.schema)  # "IFC2X3", "IFC4", or "IFC4X3"
print(f"Entities: {len(model)}")
```

### Pattern 2: Create a New IFC File (Production)

```python
# IfcOpenShell: IFC4 (change version= for IFC2X3 or IFC4X3)
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW")
```

### Pattern 3: Create a New IFC File (Bare Minimum)

```python
# IfcOpenShell: IFC4 (specify schema= for other versions)
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")
# WARNING: This file has NO header, NO project, NO units.
# ALWAYS add at minimum: IfcProject, units, and a geometric context.
```

### Pattern 4: Write IFC to Disk

```python
# IfcOpenShell: all schema versions
model.write("/path/to/output.ifc")

# Write as compressed ZIP
model.write("/path/to/output.ifc", zipped=True)

# Write as XML format
model.write("/path/to/output.ifcXML")
```

### Pattern 5: Serialize IFC to String

```python
# IfcOpenShell: all schema versions
ifc_string = model.to_string()
```

### Pattern 6: Transaction Management (Undo/Redo)

```python
# IfcOpenShell: all schema versions
model.begin_transaction()

wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Test Wall")

model.end_transaction()

# Undo the transaction
model.undo()

# Redo the transaction
model.redo()
```

### Pattern 7: Transfer Entity Between Files

```python
# IfcOpenShell: all schema versions
source = ifcopenshell.open("source.ifc")
target = ifcopenshell.file(schema=source.schema)

wall = source.by_type("IfcWall")[0]
target.add(wall)  # Recursively adds wall and all referenced entities
target.write("target.ifc")
```

### Pattern 8: Remove Entity from File

```python
# IfcOpenShell: all schema versions
wall = model.by_type("IfcWall")[0]
model.remove(wall)
# WARNING: All attributes referencing this entity become null ($).
# Aggregate references (lists/sets) are cleaned up automatically.
```

---

## Common Operations

### Opening Files

```python
# IfcOpenShell: all schema versions
import ifcopenshell

# Standard file open
model = ifcopenshell.open("building.ifc")

# Open ZIP-compressed IFC
model = ifcopenshell.open("building.ifcZIP")

# Open XML-format IFC
model = ifcopenshell.open("building.ifcXML")

# Force format when extension is non-standard
model = ifcopenshell.open("building.dat", format=".ifc")

# Streaming mode for large files (100MB+)
# WARNING: Only supports sequential iteration, NOT random access
model = ifcopenshell.open("huge_model.ifc", should_stream=True)
for entity in model:
    if entity.is_a("IfcWall"):
        print(entity.Name)
```

### Creating Files

```python
# IfcOpenShell: IFC4 (change version/schema for other versions)
import ifcopenshell
import ifcopenshell.api

# RECOMMENDED: Production file with proper header
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Alternative: Bare empty file (testing only)
model = ifcopenshell.file(schema="IFC4")

# IFC2X3 file
model_legacy = ifcopenshell.file(schema="IFC2X3")

# IFC4X3 file (infrastructure projects)
model_infra = ifcopenshell.file(schema="IFC4X3")

# Specific schema sub-version
model_specific = ifcopenshell.file(schema_version=(4, 0, 2, 1))  # IFC4 ADD2 TC1
```

### Writing Files

```python
# IfcOpenShell: all schema versions
# Standard STEP format
model.write("output.ifc")

# XML serialization
model.write("output.ifcXML")

# ZIP-compressed STEP
model.write("output.ifcZIP")

# STEP with additional ZIP compression
model.write("output.ifc", zipped=True)

# ZIP-compressed XML
model.write("output.ifcXML", format=".ifcXML", zipped=True)

# Serialize to string (no file written)
ifc_text = model.to_string()
```

### Adding and Removing Entities

```python
# IfcOpenShell: all schema versions
import ifcopenshell

# Add entity from another file (copies entity + all dependencies)
source = ifcopenshell.open("source.ifc")
target = ifcopenshell.file(schema=source.schema)
wall = source.by_type("IfcWall")[0]
target.add(wall)

# Remove entity from file
model.remove(wall)
```

### Transaction Management

```python
# IfcOpenShell: all schema versions

# Basic transaction
model.begin_transaction()
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="New Wall")
model.end_transaction()

# Undo last transaction
model.undo()

# Redo undone transaction
model.redo()

# Discard transaction without recording
model.begin_transaction()
# ... experimental changes ...
model.discard_transaction()

# Set maximum undo history size (default: 64)
model.set_history_size(128)
```

### Error Handling

```python
# IfcOpenShell: all schema versions
import ifcopenshell

try:
    model = ifcopenshell.open("model.ifc")
except FileNotFoundError:
    print("File not found")
except ifcopenshell.Error:
    print("Invalid or corrupt IFC file")
```

### Accessing File Metadata

```python
# IfcOpenShell: all schema versions
model = ifcopenshell.open("model.ifc")

# Schema information
print(model.schema)             # "IFC2X3", "IFC4", or "IFC4X3"
print(model.schema_identifier)  # e.g., "IFC4_ADD2"
print(model.schema_version)     # e.g., (4, 0, 2, 1)

# Header information
print(model.header.file_description)
print(model.header.file_name)
print(model.header.file_schema)

# Entity count
print(len(model))
```

---

## Version Notes

### Schema Parameter Differences

| Method | Parameter | Values |
|--------|-----------|--------|
| `ifcopenshell.file()` | `schema=` | `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |
| `ifcopenshell.api.project.create_file()` | `version=` | `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |

**ALWAYS** use the correct parameter name: `schema=` for `file()`, `version=` for `create_file()`.

### `ifcopenshell.file()` vs `ifcopenshell.api.project.create_file()`

| Feature | `ifcopenshell.file()` | `api.project.create_file()` |
|---------|----------------------|----------------------------|
| Header metadata | Empty/minimal | Pre-populated (timestamp, app info) |
| Preprocessor | Not set | Set to "IfcOpenShell" |
| MVD | Not set | DesignTransferView default |
| Timestamp | Not set | Current timestamp |
| Use case | Testing, prototyping | Production IFC creation |

---

## Reference Links

- [API Method Signatures](references/methods.md) — Complete signatures for all file I/O methods
- [Working Code Examples](references/examples.md) — End-to-end examples for all file operations
- [Anti-Patterns](references/anti-patterns.md) — Common file I/O mistakes and how to avoid them
