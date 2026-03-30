---
name: ifcos-errors-performance
description: >
  Use when processing large IFC files (100MB+) or optimizing slow IfcOpenShell scripts.
  Prevents the #1 performance mistake: calling create_shape() per element instead of using
  the geometry iterator for batch processing. Covers geometry iterator, by_type caching,
  batch patterns, memory management, multiprocessing strategies, and profiling.
  Keywords: performance, large IFC, geometry iterator, create_shape, batch processing, memory, multiprocessing, optimization, slow, 100MB, profiling, IFC file too large, takes too long, out of memory.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# IfcOpenShell Performance Optimization

## Quick Reference

### Decision Tree: Geometry Processing Strategy

```
Processing IFC geometry?
├── Single element (interactive/debug)?
│   └── ifcopenshell.geom.create_shape(settings, element)
│
├── Multiple elements (10+)?
│   └── ALWAYS use ifcopenshell.geom.iterator
│       ├── Need all elements? → iterator(settings, model, cpu_count())
│       └── Need specific types? → iterator(settings, model, cpu_count(), include=filtered)
│
└── No geometry needed (data extraction only)?
    └── Skip geometry entirely — use by_type() + get_psets()
```

### Decision Tree: Large File Strategy

```
File size?
├── < 10 MB (small) → Standard ifcopenshell.open(), no special handling
│
├── 10-200 MB (medium) → Cache by_type() results, batch API calls
│
├── 200 MB - 2 GB (large)
│   ├── Data only? → Load, extract to plain dicts, del model, gc.collect()
│   ├── Geometry? → Use iterator with include filter, limit threads on low-RAM
│   └── Repeated access? → Extract once, cache in external format
│
└── > 2 GB (very large)
    ├── Needs full model? → 32+ GB RAM required
    ├── Needs subset? → Extract IDs first, process in chunks
    └── Geometry? → Process by type in sequence, gc.collect() between types
```

### Critical Warnings

- **ALWAYS** use `ifcopenshell.geom.iterator` for batch geometry processing (10+ elements). NEVER call `create_shape()` in a loop for bulk operations — iterator is 5-10x faster.
- **ALWAYS** pass `multiprocessing.cpu_count()` to the iterator for optimal parallelism. Reduce thread count only on memory-constrained systems.
- **ALWAYS** cache `by_type()` results when accessing the same type multiple times. The call is fast (O(1) internal index), but repeated calls add overhead in tight loops.
- **ALWAYS** batch spatial containment and type assignments. Pass a list of products to a single API call instead of calling per-element.
- **NEVER** store all geometry shapes in memory simultaneously. Process each shape and discard immediately.
- **NEVER** use `get_info(recursive=True)` on large files — it materializes the entire entity graph into Python dicts.
- **NEVER** open the same large file multiple times. Open once and pass the `model` reference.
- **ALWAYS** call `gc.collect()` after releasing large models or between geometry processing batches.

---

## Essential Patterns

### Pattern 1: Geometry Iterator (Batch Processing)

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()

iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        verts = shape.geometry.verts   # Flat: [x1,y1,z1, x2,y2,z2, ...]
        faces = shape.geometry.faces   # Flat: [i1,i2,i3, ...]
        # Process immediately, do NOT accumulate shapes
        if not iterator.next():
            break
```

### Pattern 2: Filtered Geometry Processing

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

# Process only walls: reduces memory and time
walls = model.by_type("IfcWall")
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(),
    include=walls
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        # Process shape...
        if not iterator.next():
            break
```

### Pattern 3: Efficient Property Extraction (No Geometry)

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("large_model.ifc")

# Build pset index ONCE from relationship entities
pset_rels = model.by_type("IfcRelDefinesByProperties")
pset_map = {}
for rel in pset_rels:
    for obj in rel.RelatedObjects:
        if obj.id() not in pset_map:
            pset_map[obj.id()] = []
        pset_map[obj.id()].append(rel.RelatingPropertyDefinition)

# Now O(1) lookup per element instead of traversing relationships each time
```

### Pattern 4: Batch API Operations

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")
storey = model.by_type("IfcBuildingStorey")[0]
walls = list(model.by_type("IfcWall"))

# CORRECT: Single API call for all elements
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=walls)
# Creates ONE IfcRelContainedInSpatialStructure for all walls

# CORRECT: Batch type assignment
wall_type = model.by_type("IfcWallType")[0]
ifcopenshell.api.run("type.assign_type", model,
    related_objects=walls, relating_type=wall_type)
```

### Pattern 5: Memory Management for Large Files

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.util.element
import gc

def extract_wall_data(filepath):
    """Extract wall data from large file, then release model."""
    model = ifcopenshell.open(filepath)

    data = []
    for wall in model.by_type("IfcWall"):
        psets = ifcopenshell.util.element.get_psets(wall)
        data.append({
            "guid": wall.GlobalId,
            "name": wall.Name,
            "properties": psets
        })

    # Release model and force garbage collection
    del model
    gc.collect()

    return data  # Work with plain Python dicts from here
```

### Pattern 6: Sequential Type Processing for Very Large Files

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.geom
import gc

model = ifcopenshell.open("huge_model.ifc")

element_types = ["IfcWall", "IfcSlab", "IfcColumn", "IfcBeam"]
for etype in element_types:
    elements = model.by_type(etype)
    if not elements:
        continue

    settings = ifcopenshell.geom.settings()
    iterator = ifcopenshell.geom.iterator(
        settings, model, 4, include=elements
    )

    if iterator.initialize():
        while True:
            shape = iterator.get()
            # Process and store results immediately
            if not iterator.next():
                break

    # Force GC between types to control peak memory
    gc.collect()
```

---

## Performance Reference Tables

### File Size vs Resource Usage

| File Size | Approx Elements | RAM Usage | Load Time | Notes |
|-----------|----------------|-----------|-----------|-------|
| < 10 MB | < 1,000 | < 200 MB | < 1s | No special handling needed |
| 10-200 MB | 1,000-50,000 | 200 MB-2 GB | 1-10s | Cache query results |
| 200 MB-2 GB | 50,000-500,000 | 2-16 GB | 10-60s | Filter and batch everything |
| > 2 GB | > 500,000 | 16+ GB | 60s+ | Process by type, subprocess isolation |

### Geometry Iterator vs create_shape()

| Aspect | `create_shape()` | `geom.iterator` |
|--------|-----------------|-----------------|
| Use case | Single element, interactive | Batch processing, export |
| Multi-threading | No | Yes (OpenMP, multi-core) |
| Geometry caching | No | Yes (reuses identical geometry) |
| Error handling | Exception per element | Skips failed elements automatically |
| Speed (1000 elements) | ~60s (sequential) | ~8s (8 cores) |
| Memory per call | Lower overhead | Better amortized for many elements |

### Geometry Settings for Performance

| Setting | Effect | Performance Impact |
|---------|--------|-------------------|
| `disable-opening-subtractions` | Skips boolean CSG operations | Major speedup, less accurate geometry |
| `use-world-coords` | Applies global transforms | Slight overhead, but avoids manual transform |
| `weld-vertices` | Merges duplicate vertices | Smaller output, slight processing cost |
| `apply-default-materials` | Adds material data | Required for glTF, adds overhead |
| `dimensionality` | Controls output complexity | CURVES_SURFACES_AND_SOLIDS is slowest |

### Query Performance

| Method | Complexity | Notes |
|--------|-----------|-------|
| `model.by_type("IfcWall")` | O(1) | Uses internal class index |
| `model.by_id(42)` | O(1) | Uses internal ID map |
| `model.by_guid("3Oe$...")` | O(1) | Uses GUID index |
| `for e in model if e.is_a("IfcWall")` | O(n) | NEVER use — iterates ALL entities |
| `ifcopenshell.util.element.get_psets(wall)` | O(k) | Traverses k relationships per call |
| `ifcopenshell.util.selector.filter_elements(model, query)` | O(n) | Full scan, but expressive queries |

---

## Common Operations

### Profiling IFC Operations

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import time

model = ifcopenshell.open("model.ifc")

# Time file loading
start = time.perf_counter()
model = ifcopenshell.open("model.ifc")
load_time = time.perf_counter() - start
print(f"Load time: {load_time:.2f}s")

# Time by_type queries
start = time.perf_counter()
walls = model.by_type("IfcWall")
query_time = time.perf_counter() - start
print(f"by_type query: {query_time:.6f}s for {len(walls)} walls")

# Time geometry processing
import ifcopenshell.geom
import multiprocessing

settings = ifcopenshell.geom.settings()
start = time.perf_counter()
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)
count = 0
if iterator.initialize():
    while True:
        shape = iterator.get()
        count += 1
        if not iterator.next():
            break
geom_time = time.perf_counter() - start
print(f"Geometry: {count} shapes in {geom_time:.2f}s "
      f"({count/geom_time:.0f} shapes/sec)")
```

### Controlling Thread Count for Memory

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing
import os

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

# Check available memory (Linux/macOS)
try:
    import psutil
    available_gb = psutil.virtual_memory().available / (1024**3)
except ImportError:
    available_gb = 8  # Conservative fallback

# Scale threads to available memory
# Each thread can use 500MB-1GB for geometry processing
max_threads = multiprocessing.cpu_count()
safe_threads = min(max_threads, max(1, int(available_gb / 1.0)))

iterator = ifcopenshell.geom.iterator(
    settings, model, safe_threads
)
```

### Subprocess Isolation for Very Large Files

```python
# IfcOpenShell v0.8+: all schema versions
import subprocess
import json

# Process geometry in a subprocess to guarantee memory cleanup
# Python's GC may not release all C++ allocated memory
result = subprocess.run(
    ["python", "-c", """
import ifcopenshell
import ifcopenshell.geom
import json

model = ifcopenshell.open("huge_model.ifc")
settings = ifcopenshell.geom.settings()
walls = model.by_type("IfcWall")
iterator = ifcopenshell.geom.iterator(settings, model, 4, include=walls)

data = []
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        data.append({"guid": element.GlobalId, "verts": len(shape.geometry.verts)})
        if not iterator.next():
            break

print(json.dumps(data))
"""],
    capture_output=True, text=True
)
data = json.loads(result.stdout)
# Subprocess memory is fully reclaimed by OS on exit
```

### Disabling Expensive Geometry Operations

```python
# IfcOpenShell v0.8+: all schema versions
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()

# Skip boolean operations (opening subtractions) for speed
# Doors/windows won't create holes in walls, but processing is much faster
settings.set("disable-opening-subtractions", True)

# Use world coordinates to avoid manual transform calculations
settings.set("use-world-coords", True)
```

---

## Version Notes

### Schema Sensitivity

This skill has **low schema sensitivity**. Performance patterns apply equally to IFC2X3, IFC4, and IFC4X3 files. The geometry iterator, caching strategies, and memory management techniques are schema-independent.

### IfcOpenShell Version Notes

| Feature | Version | Notes |
|---------|---------|-------|
| `geom.iterator` | All versions | Core performance feature since early releases |
| `geom.settings()` | All versions | String-based setting names in v0.8+ |
| `by_type()` class index | All versions | O(1) lookup, always available |
| `include` filter on iterator | v0.7+ | Filter elements before geometry processing |
| Subprocess isolation | Any | Python-level pattern, not IfcOpenShell-specific |

---

## Reference Links

- [Performance Method Signatures](references/methods.md) — Complete API signatures for geometry processing and querying
- [Working Performance Examples](references/examples.md) — End-to-end optimization examples for real scenarios
- [Performance Anti-Patterns](references/anti-patterns.md) — Common performance mistakes and how to avoid them
