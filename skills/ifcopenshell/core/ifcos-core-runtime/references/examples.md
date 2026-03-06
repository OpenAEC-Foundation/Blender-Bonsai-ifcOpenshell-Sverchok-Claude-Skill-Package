# ifcos-core-runtime — Working Code Examples

## Example 1: Safe Entity Reference Handling

```python
# Schema-agnostic
import ifcopenshell

model = ifcopenshell.open("building.ifc")

# SAFE: Store STEP IDs for long-lived references
wall_ids = [w.id() for w in model.by_type("IfcWall")]

# Process walls by re-fetching from ID
for wid in wall_ids:
    try:
        wall = model.by_id(wid)
        print(f"Wall #{wid}: {wall.Name}")
    except RuntimeError:
        print(f"Wall #{wid} no longer exists")
```

## Example 2: Entity Comparison

```python
# Schema-agnostic
import ifcopenshell

model = ifcopenshell.open("building.ifc")
walls = model.by_type("IfcWall")

if len(walls) >= 2:
    wall_a = walls[0]
    wall_b = model.by_id(walls[0].id())

    # CORRECT: Value equality
    assert wall_a == wall_b        # True — same underlying entity
    assert wall_a.id() == wall_b.id()  # True — same STEP ID

    # WRONG: Identity check (always False for separately queried wrappers)
    assert not (wall_a is wall_b)  # True — different Python objects

    # CORRECT: Membership check in a collection
    wall_set = set()
    wall_set.add(wall_a.id())
    assert wall_b.id() in wall_set  # Use IDs for set membership
```

## Example 3: Safe Entity Removal

```python
# Schema-agnostic
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("building.ifc")

# Collect IDs of walls to remove (do NOT remove during iteration)
walls_to_remove = [w.id() for w in model.by_type("IfcWall") if w.Name == "Temporary"]

# Remove walls using API (cleans up relationships)
for wid in walls_to_remove:
    try:
        wall = model.by_id(wid)
        ifcopenshell.api.run("root.remove_product", model, product=wall)
        wall = None  # Nullify reference — C++ object is gone
    except RuntimeError:
        pass  # Entity already removed (e.g., cascading removal)

model.write("cleaned.ifc")
```

## Example 4: Efficient by_type() Usage

```python
# Schema-agnostic
import ifcopenshell

model = ifcopenshell.open("large_building.ifc")

# by_type() returns a TUPLE (not a list)
walls = model.by_type("IfcWall")
print(type(walls))  # <class 'tuple'>
print(f"Found {len(walls)} walls")

# First call builds the index; second call is instant
slabs = model.by_type("IfcSlab")   # Builds index for IfcSlab
slabs2 = model.by_type("IfcSlab")  # Uses cached index

# Include vs exclude subtypes
all_walls = model.by_type("IfcWall")                          # Includes IfcWallStandardCase
only_walls = model.by_type("IfcWall", include_subtypes=False)  # Excludes subtypes

# Convert to list only if mutation is needed
wall_list = list(walls)
wall_list.sort(key=lambda w: w.Name or "")
```

## Example 5: get_info() for Bulk Attribute Reading

```python
# Schema-agnostic
import ifcopenshell

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

# Full attribute extraction (one C++ round-trip)
info = wall.get_info()
# info = {"id": 42, "type": "IfcWall", "GlobalId": "2O2Fr$...", "Name": "Wall 001", ...}

print(f"Entity: #{info['id']} ({info['type']})")
print(f"Name: {info.get('Name')}")

# Scalar-only extraction (faster — skips entity references)
scalar_info = wall.get_info(scalar_only=True)

# AVOID on large models:
# deep_info = wall.get_info(recursive=True)  # Materializes entire graph
```

## Example 6: Thread-Safe Parallel Processing

```python
# Schema-agnostic
import ifcopenshell
import concurrent.futures

def analyze_file(filepath):
    """Each thread opens its own file instance — safe."""
    model = ifcopenshell.open(filepath)
    walls = model.by_type("IfcWall")
    slabs = model.by_type("IfcSlab")
    return {
        "file": filepath,
        "walls": len(walls),
        "slabs": len(slabs),
        "total": len(model)
    }

files = ["building_a.ifc", "building_b.ifc", "building_c.ifc"]

with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    results = list(executor.map(analyze_file, files))

for r in results:
    print(f"{r['file']}: {r['walls']} walls, {r['slabs']} slabs, {r['total']} total")
```

## Example 7: Thread-Safe Parallel Write (Separate Files)

```python
# Schema-agnostic
import ifcopenshell
import ifcopenshell.api
import concurrent.futures

def process_and_write(input_path, output_path, prefix):
    """Each thread works on its own independent model instance."""
    model = ifcopenshell.open(input_path)
    for wall in model.by_type("IfcWall"):
        ifcopenshell.api.run("attribute.edit_attributes", model,
            product=wall, attributes={"Name": f"{prefix}_{wall.Name}"})
    model.write(output_path)
    return output_path

tasks = [
    ("model.ifc", "output_a.ifc", "A"),
    ("model.ifc", "output_b.ifc", "B"),
    ("model.ifc", "output_c.ifc", "C"),
]

with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(process_and_write, *t) for t in tasks]
    for f in concurrent.futures.as_completed(futures):
        print(f"Written: {f.result()}")
```

## Example 8: Memory Management for Large Models

```python
# Schema-agnostic
import ifcopenshell
import gc

def process_large_model(filepath):
    model = ifcopenshell.open(filepath)

    # Process in batches to limit peak memory
    wall_ids = [w.id() for w in model.by_type("IfcWall")]

    results = []
    for wid in wall_ids:
        wall = model.by_id(wid)
        # Use direct attribute access, NOT get_info(recursive=True)
        results.append({
            "id": wid,
            "name": wall.Name,
            "type": wall.is_a(),
        })

    # Explicitly release the model
    del model
    gc.collect()

    return results
```

## Example 9: Safe File Lifecycle Pattern

```python
# Schema-agnostic
import ifcopenshell

class IfcModelManager:
    """Keeps file reference alive as long as manager exists."""

    def __init__(self, filepath):
        self._model = ifcopenshell.open(filepath)

    @property
    def model(self):
        return self._model

    def get_walls(self):
        return self._model.by_type("IfcWall")

    def get_entity(self, entity_id):
        try:
            return self._model.by_id(entity_id)
        except RuntimeError:
            return None

    def close(self):
        del self._model
        self._model = None

# Usage
manager = IfcModelManager("building.ifc")
walls = manager.get_walls()
print(walls[0].Name)  # Safe — manager keeps model reference alive

manager.close()
# walls[0].Name  # UNSAFE after close — entities are invalidated
```

## Example 10: Schema-Aware Code

```python
# Schema-agnostic
import ifcopenshell

model = ifcopenshell.open("unknown_file.ifc")
schema = model.schema  # "IFC2X3", "IFC4", or "IFC4X3"

# Schema-aware entity selection
def get_wall_class(schema):
    if schema == "IFC2X3":
        return "IfcWallStandardCase"
    else:
        return "IfcWall"

def get_door_type_class(schema):
    if schema == "IFC2X3":
        return "IfcDoorStyle"
    else:
        return "IfcDoorType"

# Schema-aware attribute access
wall = model.by_type(get_wall_class(schema))[0]

if schema in ("IFC4", "IFC4X3"):
    predefined_type = wall.PredefinedType
else:
    predefined_type = None  # IFC2X3 IfcWallStandardCase has no PredefinedType

print(f"Schema: {schema}, Wall class: {wall.is_a()}, PredefinedType: {predefined_type}")
```

## Example 11: Version Detection

```python
# Schema-agnostic
import ifcopenshell

# Library version
print(f"IfcOpenShell version: {ifcopenshell.version}")  # e.g., "0.8.1"

# File schema detection
model = ifcopenshell.open("model.ifc")
print(f"Schema: {model.schema}")                # "IFC2X3", "IFC4", "IFC4X3"
print(f"Schema ID: {model.schema_identifier}")   # e.g., "IFC4_ADD2"
print(f"Schema version: {model.schema_version}") # e.g., (4, 0, 2, 1)
```

## Example 12: Installation Verification

```bash
# pip installation
pip install ifcopenshell
python -c "import ifcopenshell; print(ifcopenshell.version)"

# conda installation
conda install -c conda-forge ifcopenshell
python -c "import ifcopenshell; print(ifcopenshell.version)"

# Blender integration check
/path/to/blender/python/bin/python -c "import ifcopenshell; print(ifcopenshell.version)"

# Docker minimal
# Dockerfile:
# FROM python:3.11-slim
# RUN pip install ifcopenshell
# CMD ["python", "-c", "import ifcopenshell; print(ifcopenshell.version)"]
```
