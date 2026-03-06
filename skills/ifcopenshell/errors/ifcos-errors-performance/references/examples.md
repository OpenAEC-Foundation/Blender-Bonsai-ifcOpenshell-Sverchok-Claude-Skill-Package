# Performance Working Code Examples

All examples verified against the [IfcOpenShell documentation](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html) and [code examples](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html).

---

## 1. Bulk Geometry Export with Iterator

Extract mesh data from all elements using the high-performance iterator.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing
import json

model = ifcopenshell.open("building.ifc")

settings = ifcopenshell.geom.settings()
settings.set("use-world-coords", True)
settings.set("weld-vertices", True)

iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)

results = []
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)

        verts = shape.geometry.verts
        faces = shape.geometry.faces

        vertices = [(verts[i], verts[i+1], verts[i+2])
                    for i in range(0, len(verts), 3)]
        triangles = [(faces[i], faces[i+1], faces[i+2])
                     for i in range(0, len(faces), 3)]

        results.append({
            "guid": element.GlobalId,
            "name": element.Name,
            "type": element.is_a(),
            "num_vertices": len(vertices),
            "num_triangles": len(triangles),
        })

        if not iterator.next():
            break

print(f"Processed {len(results)} elements")
```

---

## 2. Type-Filtered Geometry Processing

Process only specific element types to reduce processing time and memory.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")

settings = ifcopenshell.geom.settings()
settings.set("use-world-coords", True)

# Process structural elements only
structural_types = ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam"]
for element_type in structural_types:
    elements = model.by_type(element_type)
    if not elements:
        print(f"No {element_type} found, skipping")
        continue

    iterator = ifcopenshell.geom.iterator(
        settings, model, multiprocessing.cpu_count(),
        include=elements
    )

    count = 0
    if iterator.initialize():
        while True:
            shape = iterator.get()
            count += 1
            # Process geometry for this type...
            if not iterator.next():
                break

    print(f"Processed {count} {element_type} elements")
```

---

## 3. Fast Property Extraction Without Geometry

Extract element data without the overhead of geometry processing.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit
import csv

model = ifcopenshell.open("building.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

# Build property cache for all walls at once
walls = model.by_type("IfcWall")
wall_data = []

for wall in walls:
    psets = ifcopenshell.util.element.get_psets(wall)
    wall_type = ifcopenshell.util.element.get_type(wall)
    container = ifcopenshell.util.element.get_container(wall)

    wall_data.append({
        "guid": wall.GlobalId,
        "name": wall.Name or "",
        "type_name": wall_type.Name if wall_type else "",
        "storey": container.Name if container else "",
        "is_external": psets.get("Pset_WallCommon", {}).get("IsExternal", ""),
        "fire_rating": psets.get("Pset_WallCommon", {}).get("FireRating", ""),
    })

# Write to CSV
with open("wall_report.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=wall_data[0].keys())
    writer.writeheader()
    writer.writerows(wall_data)

print(f"Exported {len(wall_data)} walls to wall_report.csv")
```

---

## 4. High-Performance Property Index

Build a property index from relationship entities for O(1) lookups.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
from collections import defaultdict

model = ifcopenshell.open("large_model.ifc")

# Strategy: Build index from IfcRelDefinesByProperties directly
# This is faster than calling get_psets() per element for large models
pset_index = defaultdict(dict)

for rel in model.by_type("IfcRelDefinesByProperties"):
    pdef = rel.RelatingPropertyDefinition
    if not pdef.is_a("IfcPropertySet"):
        continue

    pset_name = pdef.Name
    props = {}
    for prop in pdef.HasProperties:
        if prop.is_a("IfcPropertySingleValue") and prop.NominalValue:
            props[prop.Name] = prop.NominalValue.wrappedValue

    for obj in rel.RelatedObjects:
        pset_index[obj.id()][pset_name] = props

# Now O(1) lookup per element
for wall in model.by_type("IfcWall"):
    wall_props = pset_index.get(wall.id(), {})
    fire_rating = wall_props.get("Pset_WallCommon", {}).get("FireRating")
    if fire_rating:
        print(f"{wall.Name}: FireRating = {fire_rating}")
```

---

## 5. Spatial Index for Element Lookups

Build a spatial containment index for fast element-by-storey queries.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
from collections import defaultdict

model = ifcopenshell.open("building.ifc")

# Build spatial containment index
elements_by_container = defaultdict(list)
for rel in model.by_type("IfcRelContainedInSpatialStructure"):
    container = rel.RelatingStructure
    for element in rel.RelatedElements:
        elements_by_container[container.id()].append(element)

# Query: all elements on a specific storey
for storey in model.by_type("IfcBuildingStorey"):
    elements = elements_by_container.get(storey.id(), [])
    wall_count = sum(1 for e in elements if e.is_a("IfcWall"))
    print(f"{storey.Name}: {len(elements)} elements, {wall_count} walls")
```

---

## 6. Memory-Controlled Large File Processing

Process a very large file with controlled memory usage.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.element
import gc
import sys

filepath = "huge_model.ifc"

# Phase 1: Extract metadata (low memory)
model = ifcopenshell.open(filepath)
print(f"Schema: {model.schema}")
print(f"Total entities: {len(model)}")

wall_count = len(model.by_type("IfcWall"))
slab_count = len(model.by_type("IfcSlab"))
print(f"Walls: {wall_count}, Slabs: {slab_count}")

# Phase 2: Extract property data to plain dicts, then release model
wall_data = []
for wall in model.by_type("IfcWall"):
    wall_data.append({
        "id": wall.id(),
        "guid": wall.GlobalId,
        "name": wall.Name,
        "psets": ifcopenshell.util.element.get_psets(wall)
    })

# Phase 3: Process geometry by type to limit peak memory
settings = ifcopenshell.geom.settings()
settings.set("use-world-coords", True)

geometry_data = []
for etype in ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam"]:
    elements = model.by_type(etype)
    if not elements:
        continue

    # Use fewer threads to reduce peak memory
    iterator = ifcopenshell.geom.iterator(
        settings, model, 2, include=elements
    )

    if iterator.initialize():
        while True:
            shape = iterator.get()
            geometry_data.append({
                "id": shape.id,
                "vert_count": len(shape.geometry.verts) // 3,
                "face_count": len(shape.geometry.faces) // 3,
            })
            if not iterator.next():
                break

    gc.collect()
    print(f"Processed {etype}: {len(elements)} elements")

# Phase 4: Release model
del model
gc.collect()

# Continue working with plain Python data structures
print(f"Total geometry records: {len(geometry_data)}")
```

---

## 7. Batch Entity Creation with Performance

Create many elements efficiently using batch operations.

```python
# IfcOpenShell v0.8+ — IFC4
import ifcopenshell
import ifcopenshell.api

# Create file with project structure
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="Performance Test")
ifcopenshell.api.run("unit.assign_unit", model)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Create walls — collect ALL, then batch-assign
walls = []
for i in range(100):
    wall = ifcopenshell.api.run("root.create_entity", model,
        ifc_class="IfcWall", name=f"Wall-{i:03d}")
    walls.append(wall)

# BATCH: Single spatial containment call for all walls
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=walls)

# BATCH: Single type assignment for all walls
wall_type = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcWallType", name="Standard Wall 200mm")
ifcopenshell.api.run("type.assign_type", model,
    related_objects=walls, relating_type=wall_type)

model.write("batch_created.ifc")
print(f"Created {len(walls)} walls with batch operations")
```

---

## 8. Performance Benchmarking Script

Measure and compare different processing strategies.

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.element
import multiprocessing
import time

filepath = "model.ifc"

# Benchmark 1: File loading
t0 = time.perf_counter()
model = ifcopenshell.open(filepath)
t_load = time.perf_counter() - t0
print(f"File load: {t_load:.3f}s ({len(model)} entities)")

# Benchmark 2: by_type queries
t0 = time.perf_counter()
walls = model.by_type("IfcWall")
t_query = time.perf_counter() - t0
print(f"by_type('IfcWall'): {t_query:.6f}s ({len(walls)} results)")

# Benchmark 3: Property extraction (per-element)
t0 = time.perf_counter()
for wall in walls:
    psets = ifcopenshell.util.element.get_psets(wall)
t_psets = time.perf_counter() - t0
print(f"get_psets (per-element): {t_psets:.3f}s "
      f"({len(walls)/t_psets:.0f} elements/sec)")

# Benchmark 4: Property extraction (indexed)
t0 = time.perf_counter()
pset_rels = model.by_type("IfcRelDefinesByProperties")
pset_cache = {}
for rel in pset_rels:
    for obj in rel.RelatedObjects:
        pset_cache[obj.id()] = True  # Simplified index
t_indexed = time.perf_counter() - t0
print(f"Property index build: {t_indexed:.3f}s "
      f"({len(pset_rels)} relationships)")

# Benchmark 5: Geometry with create_shape (sequential)
settings = ifcopenshell.geom.settings()
test_walls = walls[:20]  # Only first 20 for comparison

t0 = time.perf_counter()
for wall in test_walls:
    try:
        shape = ifcopenshell.geom.create_shape(settings, wall)
    except RuntimeError:
        pass
t_sequential = time.perf_counter() - t0
print(f"create_shape (sequential, {len(test_walls)}): {t_sequential:.3f}s")

# Benchmark 6: Geometry with iterator (parallel)
t0 = time.perf_counter()
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(),
    include=test_walls
)
count = 0
if iterator.initialize():
    while True:
        shape = iterator.get()
        count += 1
        if not iterator.next():
            break
t_parallel = time.perf_counter() - t0
print(f"iterator (parallel, {count}): {t_parallel:.3f}s")

if t_sequential > 0 and t_parallel > 0:
    speedup = t_sequential / t_parallel
    print(f"Iterator speedup: {speedup:.1f}x")
```

---

## 9. Subprocess Isolation for Guaranteed Memory Cleanup

Use a subprocess to ensure all memory is reclaimed after processing.

```python
# IfcOpenShell v0.8+ — all schema versions
import subprocess
import json
import sys

def process_in_subprocess(filepath, element_type="IfcWall"):
    """Process geometry in isolated subprocess for memory safety."""
    script = f"""
import ifcopenshell
import ifcopenshell.geom
import json

model = ifcopenshell.open("{filepath}")
settings = ifcopenshell.geom.settings()
settings.set("use-world-coords", True)

elements = model.by_type("{element_type}")
iterator = ifcopenshell.geom.iterator(settings, model, 4, include=elements)

data = []
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        data.append({{
            "guid": element.GlobalId,
            "name": element.Name or "",
            "vert_count": len(shape.geometry.verts) // 3,
            "face_count": len(shape.geometry.faces) // 3,
        }})
        if not iterator.next():
            break

print(json.dumps(data))
"""
    result = subprocess.run(
        [sys.executable, "-c", script],
        capture_output=True, text=True, timeout=300
    )
    if result.returncode != 0:
        raise RuntimeError(f"Subprocess failed: {result.stderr}")
    return json.loads(result.stdout)

# Usage — all memory is reclaimed when subprocess exits
wall_data = process_in_subprocess("huge_model.ifc", "IfcWall")
slab_data = process_in_subprocess("huge_model.ifc", "IfcSlab")
print(f"Walls: {len(wall_data)}, Slabs: {len(slab_data)}")
```

---

## Sources

- [IfcOpenShell Geometry Processing](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html)
- [IfcOpenShell Code Examples](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)
- [IfcOpenShell Academy - Optimizer Tutorial](https://academy.ifcopenshell.org/posts/ifcopenshell-optimizer-tutorial/)
- [GitHub Issue #6905 - Memory usage with geometry iterator](https://github.com/IfcOpenShell/IfcOpenShell/issues/6905)
- [GitHub Issue #2025 - Large IFC dataset strategies](https://github.com/IfcOpenShell/IfcOpenShell/issues/2025)
