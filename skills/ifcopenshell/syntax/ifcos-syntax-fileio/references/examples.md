# File I/O Working Code Examples

All examples verified against the [IfcOpenShell documentation](https://docs.ifcopenshell.org/ifcopenshell-python/hello_world.html) and [file API reference](https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html).

---

## 1. Opening IFC Files

### Open a Standard IFC File

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("/path/to/model.ifc")

# Inspect the loaded model
print(model.schema)             # "IFC2X3", "IFC4", or "IFC4X3"
print(model.schema_identifier)  # e.g., "IFC4_ADD2"
print(model.schema_version)     # e.g., (4, 0, 2, 1)
print(f"Total entities: {len(model)}")
```

### Open Different File Formats

```python
# IfcOpenShell — all schema versions
import ifcopenshell

# ZIP-compressed IFC
model = ifcopenshell.open("/path/to/model.ifcZIP")

# XML-format IFC
model = ifcopenshell.open("/path/to/model.ifcXML")

# Force format when extension is non-standard
model = ifcopenshell.open("/path/to/model.dat", format=".ifc")
```

### Open Large Files with Streaming

```python
# IfcOpenShell — all schema versions
import ifcopenshell

# Streaming mode: lower memory footprint, sequential access only
model = ifcopenshell.open("/path/to/huge_model.ifc", should_stream=True)
for entity in model:
    if entity.is_a("IfcWall"):
        print(entity.Name)
# WARNING: by_type(), by_id(), by_guid() are NOT available in streaming mode.
```

### Access File Header

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")

header = model.header
print(header.file_description)
print(header.file_name)
print(header.file_schema)
```

### Error Handling When Opening Files

```python
# IfcOpenShell — all schema versions
import ifcopenshell

try:
    model = ifcopenshell.open("model.ifc")
except FileNotFoundError:
    print("File not found")
except ifcopenshell.Error:
    print("Invalid or corrupt IFC file")
```

---

## 2. Creating New IFC Files

### Production File (Recommended)

```python
# IfcOpenShell — IFC4 (change version= for IFC2X3 or IFC4X3)
import ifcopenshell
import ifcopenshell.api

# Create file with proper header metadata
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Set up required project structure
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")

# Assign default SI units
ifcopenshell.api.run("unit.assign_unit", model)

# Set up geometry representation contexts
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=context)

model.write("new_project.ifc")
```

### Bare Empty File (Testing Only)

```python
# IfcOpenShell — IFC4 (change schema= for other versions)
import ifcopenshell

# IFC4 file
model = ifcopenshell.file(schema="IFC4")

# IFC2X3 file (legacy compatibility)
model_legacy = ifcopenshell.file(schema="IFC2X3")

# IFC4X3 file (infrastructure projects)
model_infra = ifcopenshell.file(schema="IFC4X3")

# Specific sub-version
model_specific = ifcopenshell.file(schema_version=(4, 0, 2, 1))  # IFC4 ADD2 TC1

print(model.schema)  # "IFC4"
```

### Direct Module Call vs api.run()

```python
# IfcOpenShell — IFC4
import ifcopenshell
import ifcopenshell.api
import ifcopenshell.api.project

# These two calls are equivalent:
model_a = ifcopenshell.api.run("project.create_file", version="IFC4")
model_b = ifcopenshell.api.project.create_file(version="IFC4")
```

---

## 3. Writing IFC Files

### Write to Different Formats

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("input.ifc")

# Standard STEP format (.ifc)
model.write("/path/to/output.ifc")

# XML serialization (.ifcXML)
model.write("/path/to/output.ifcXML")

# ZIP-compressed archive (.ifcZIP)
model.write("/path/to/output.ifcZIP")

# Force specific format regardless of extension
model.write("/path/to/output.dat", format=".ifc")

# STEP with additional ZIP compression
model.write("/path/to/output.ifc", zipped=True)

# ZIP-compressed XML
model.write("/path/to/output.ifcXML", format=".ifcXML", zipped=True)
```

### Serialize to String

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")
ifc_string = model.to_string()
print(ifc_string[:500])  # Print first 500 characters
```

### Safe Overwrite Pattern

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import shutil

model = ifcopenshell.open("model.ifc")
# ... modifications ...
model.write("model_temp.ifc")
shutil.move("model_temp.ifc", "model.ifc")
```

---

## 4. Adding and Removing Entities

### Transfer Entity Between Files

```python
# IfcOpenShell — all schema versions
import ifcopenshell

source = ifcopenshell.open("source.ifc")
target = ifcopenshell.file(schema=source.schema)

# Add copies a wall and ALL its referenced entities recursively
wall = source.by_type("IfcWall")[0]
target.add(wall)

target.write("target.ifc")
```

### Remove Entity from File

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")

wall = model.by_type("IfcWall")[0]
print(f"Before removal: {len(model.by_type('IfcWall'))} walls")

model.remove(wall)
print(f"After removal: {len(model.by_type('IfcWall'))} walls")

model.write("modified.ifc")
```

---

## 5. Transaction Management

### Basic Transaction with Undo/Redo

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file(schema="IFC4")

# Start tracking changes
model.begin_transaction()

wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(),
    Name="Transaction Wall")

model.end_transaction()

print(f"After creation: {len(model.by_type('IfcWall'))} walls")

# Undo the creation
model.undo()
print(f"After undo: {len(model.by_type('IfcWall'))} walls")

# Redo the creation
model.redo()
print(f"After redo: {len(model.by_type('IfcWall'))} walls")
```

### Discard a Transaction

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")

# Start a transaction for experimental changes
model.begin_transaction()

# Make changes that may not be wanted
model.create_entity("IfcWall", GlobalId=ifcopenshell.guid.new())

# Discard all changes — nothing is recorded in undo history
model.discard_transaction()

print(f"Walls after discard: {len(model.by_type('IfcWall'))}")  # 0
```

### Multiple Transactions with History

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.guid

model = ifcopenshell.file(schema="IFC4")

# Transaction 1: Create a wall
model.begin_transaction()
wall = model.create_entity("IfcWall",
    GlobalId=ifcopenshell.guid.new(), Name="Wall 1")
model.end_transaction()

# Transaction 2: Create a slab
model.begin_transaction()
slab = model.create_entity("IfcSlab",
    GlobalId=ifcopenshell.guid.new(), Name="Slab 1")
model.end_transaction()

# Undo Transaction 2 (removes slab)
model.undo()

# Undo Transaction 1 (removes wall)
model.undo()

# Redo Transaction 1 (restores wall)
model.redo()
```

### Set History Size

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.file(schema="IFC4")

# Increase undo history from default 64 to 128
model.set_history_size(128)
```

---

## 6. Complete Workflow: Create a Model from Scratch

```python
# IfcOpenShell — IFC4 (recommended production workflow)
import ifcopenshell
import ifcopenshell.api

# 1. Create file with proper header
model = ifcopenshell.api.run("project.create_file", version="IFC4")

# 2. Create project (mandatory)
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Building Project")

# 3. Assign default SI units
ifcopenshell.api.run("unit.assign_unit", model)

# 4. Set up geometry contexts
context = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model",
    context_identifier="Body",
    target_view="MODEL_VIEW",
    parent=context)

# 5. Create spatial hierarchy
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Construction Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

# 6. Assign spatial aggregation
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=project, products=[site])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=site, products=[building])
ifcopenshell.api.run("aggregate.assign_object", model,
    relating_object=building, products=[storey])

# 7. Create a wall
wall = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWall", name="Exterior Wall")

# 8. Assign wall to storey
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=[wall])

# 9. Write to disk
model.write("complete_model.ifc")
```

---

## 7. Complete Workflow: Read, Modify, Save

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import shutil

# Open existing file
model = ifcopenshell.open("input_model.ifc")

# Check schema before doing schema-specific work
print(f"Schema: {model.schema}")

# Find and rename all walls
for wall in model.by_type("IfcWall"):
    if wall.Name:
        wall.Name = f"RENAMED_{wall.Name}"

# Safe save (write to temp, then move)
model.write("input_model_temp.ifc")
shutil.move("input_model_temp.ifc", "output_model.ifc")
```
