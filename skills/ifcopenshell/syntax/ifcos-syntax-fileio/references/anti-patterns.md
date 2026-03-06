# File I/O Anti-Patterns

Common mistakes when working with IfcOpenShell file I/O operations. Each anti-pattern explains WHAT is wrong and WHY it causes problems.

---

## AP-1: Using ifcopenshell.file() Without Explicit Schema

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
model = ifcopenshell.file()
```

### Correct

```python
# IfcOpenShell — IFC4 (or "IFC2X3", "IFC4X3")
model = ifcopenshell.file(schema="IFC4")
```

### WHY

`ifcopenshell.file()` defaults to `"IFC4"`, but relying on implicit defaults makes the code fragile. If the default changes in a future IfcOpenShell version, or if a reader misunderstands the intended schema, the code silently produces the wrong output. ALWAYS specify the `schema=` parameter explicitly to make the intent clear and the code self-documenting.

---

## AP-2: Using ifcopenshell.file() for Production IFC Creation

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")
# File has NO header metadata, NO timestamps, NO MVD
model.write("production_model.ifc")
```

### Correct

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
# File has proper header, timestamps, preprocessor info, MVD
```

### WHY

`ifcopenshell.file()` creates a completely blank file with no header metadata, no preprocessor info, no timestamp, and no MVD (Model View Definition). IFC viewers and validators expect these header fields to be populated. Files without proper headers are flagged as invalid by IFC compliance checkers. `ifcopenshell.api.project.create_file()` automatically populates FILE_DESCRIPTION, FILE_NAME, FILE_SCHEMA, timestamps, and sets the MVD to DesignTransferView.

---

## AP-3: Creating a File Without IfcProject, Units, or Context

### Wrong

```python
# IfcOpenShell — IFC4 — BAD PRACTICE
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())
model.write("incomplete.ifc")
# File has no project, no units, no geometry context — invalid IFC
```

### Correct

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")

# REQUIRED: IfcProject
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")

# REQUIRED: Unit assignment
ifcopenshell.api.run("unit.assign_unit", model)

# REQUIRED: Geometry context
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW")
```

### WHY

A valid IFC file requires at minimum: an IfcProject, unit assignments, and a geometric representation context. Without these, IFC viewers fail to load the model, geometry cannot be rendered, and measurements have no defined units. Validators reject files missing these mandatory elements.

---

## AP-4: Not Checking Schema After Opening a File

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.open("unknown_model.ifc")
# Assumes IFC4 entities exist — crashes on IFC2X3 files
spaces = model.by_type("IfcSpatialZone")  # IfcSpatialZone does not exist in IFC2X3
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("unknown_model.ifc")
print(model.schema)  # Check first: "IFC2X3", "IFC4", or "IFC4X3"

if model.schema == "IFC2X3":
    spatials = model.by_type("IfcSpatialStructureElement")
elif model.schema in ("IFC4", "IFC4X3"):
    spatials = model.by_type("IfcSpatialElement")
```

### WHY

IFC2X3, IFC4, and IFC4X3 have different entity types, different attribute names, and different inheritance hierarchies. Code that assumes a specific schema crashes or returns empty results on files with a different schema. ALWAYS check `model.schema` before using schema-specific entity types or attributes.

---

## AP-5: Leaving Transactions Open

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")
model.begin_transaction()
wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())
# Transaction never closed — undo/redo state is corrupted
model.write("output.ifc")
```

### Correct

```python
# IfcOpenShell — IFC4
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")
model.begin_transaction()
wall = model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())
model.end_transaction()
model.write("output.ifc")
```

### WHY

An unclosed transaction leaves the file's internal state in an inconsistent condition. Subsequent calls to `begin_transaction()`, `undo()`, or `redo()` produce undefined behavior. ALWAYS pair `begin_transaction()` with either `end_transaction()` or `discard_transaction()`.

---

## AP-6: Calling undo() or redo() During an Active Transaction

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
model.begin_transaction()
model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())
model.undo()  # Called while transaction is still open
model.end_transaction()
```

### Correct

```python
# IfcOpenShell — IFC4
model.begin_transaction()
model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())
model.end_transaction()
model.undo()  # Called after transaction is closed
```

### WHY

`undo()` and `redo()` operate on completed transactions in the history. Calling them while a transaction is active interferes with the transaction tracking system and produces unpredictable results. ALWAYS close the current transaction before calling `undo()` or `redo()`.

---

## AP-7: Using file.remove() for Products Without Cleanup

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]
model.remove(wall)
# Spatial containment, property set links, type assignments are now broken
# Other entities still reference a deleted entity (null pointers)
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]
ifcopenshell.api.run("root.remove_product", model, product=wall)
# Cleans up spatial containment, property sets, type assignments, geometry
```

### WHY

`file.remove()` is a low-level operation that deletes the entity but does NOT clean up relationships. The deleted entity's references in other entities (IfcRelContainedInSpatialStructure, IfcRelDefinesByProperties, IfcRelDefinesByType, etc.) become null (`$`), leaving orphaned relationships and broken references. The high-level API `root.remove_product` handles all relationship cleanup automatically.

---

## AP-8: Writing to the Same File While Streaming

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.open("model.ifc", should_stream=True)
# Streaming mode reads sequentially from disk
model.write("model.ifc")  # Overwriting the file being read from
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import shutil

model = ifcopenshell.open("model.ifc")  # Non-streaming: fully loaded into memory
# ... modifications ...
model.write("model_temp.ifc")
shutil.move("model_temp.ifc", "model.ifc")
```

### WHY

In streaming mode, the file is read sequentially from disk — it is not fully loaded into memory. Writing to the same file path while streaming corrupts both the read and write operations. ALWAYS write to a separate file path first, then rename/move if overwriting the original is needed.

---

## AP-9: Mixing Up schema= and version= Parameters

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
# Wrong parameter name for create_file
model = ifcopenshell.api.run("project.create_file", schema="IFC4")

# Wrong parameter name for file()
model = ifcopenshell.file(version="IFC4")
```

### Correct

```python
# IfcOpenShell — IFC4
# file() uses schema=
model = ifcopenshell.file(schema="IFC4")

# project.create_file() uses version=
model = ifcopenshell.api.run("project.create_file", version="IFC4")
```

### WHY

`ifcopenshell.file()` accepts `schema=` as the parameter name, while `ifcopenshell.api.project.create_file()` accepts `version=`. Using the wrong parameter name passes it as an unexpected keyword argument, which is silently ignored, causing the file to be created with the default schema instead of the intended one. This produces files with the wrong schema that fail validation downstream.

---

## AP-10: Using create_entity() for Complex Operations

### Wrong

```python
# IfcOpenShell — IFC4 — BAD PRACTICE
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file(schema="IFC4")
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="My Wall")
# Wall has no spatial containment, no property sets, no type, no geometry
# It is a floating, unconnected entity in the IFC file
```

### Correct

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="My Wall")
# Wall has auto-generated GlobalId, OwnerHistory, and proper defaults
```

### WHY

`create_entity()` is a low-level operation that creates a bare entity without any relationships. For rooted IFC products (walls, doors, slabs, etc.), the high-level `ifcopenshell.api.root.create_entity()` automatically generates GlobalIds, sets OwnerHistory, validates predefined types, and handles schema-dependent defaults. Using `create_entity()` for products results in incomplete entities that fail IFC validation.
