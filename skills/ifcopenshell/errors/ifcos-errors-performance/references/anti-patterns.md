# Performance Anti-Patterns

Common performance mistakes when working with IfcOpenShell. Each anti-pattern explains WHAT is wrong, WHY it causes problems, and the CORRECT alternative.

---

## AP-1: Using create_shape() in a Loop for Bulk Processing

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

for wall in model.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)
    # Process shape...
```

### Correct

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

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

### WHY

`create_shape()` is single-threaded and has per-call overhead (settings validation, geometry engine initialization). The iterator uses OpenMP multi-threading and implements geometry caching for identical shapes. For 1000+ elements, the iterator is 5-10x faster. NEVER use `create_shape()` in a loop for more than a handful of elements.

---

## AP-2: Iterating All Entities to Find a Specific Type

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE
model = ifcopenshell.open("model.ifc")

# O(n) iteration over ALL entities in the file
walls = [e for e in model if e.is_a("IfcWall")]
```

### Correct

```python
# IfcOpenShell — all schema versions
model = ifcopenshell.open("model.ifc")

# O(1) lookup using internal class index
walls = model.by_type("IfcWall")
```

### WHY

`for e in model` iterates over every single entity in the file — potentially hundreds of thousands of entities including geometry primitives, coordinate points, and relationships. `by_type()` uses an internal class index that returns matching entities in O(1) time regardless of model size. On a 100MB file with 500,000 entities, the wrong approach is 1000x slower.

---

## AP-3: Calling get_psets() Repeatedly in a Loop

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE
import ifcopenshell.util.element

model = ifcopenshell.open("large_model.ifc")
walls = model.by_type("IfcWall")

# get_psets traverses IsDefinedBy relationships on EVERY call
external_walls = []
for wall in walls:
    psets = ifcopenshell.util.element.get_psets(wall)
    if psets.get("Pset_WallCommon", {}).get("IsExternal"):
        external_walls.append(wall)
        psets_again = ifcopenshell.util.element.get_psets(wall)  # Traverses AGAIN
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell.util.element

model = ifcopenshell.open("large_model.ifc")
walls = model.by_type("IfcWall")

# Cache psets for all walls in one pass
psets_cache = {w.id(): ifcopenshell.util.element.get_psets(w) for w in walls}

external_walls = []
for wall in walls:
    psets = psets_cache[wall.id()]  # O(1) dict lookup
    if psets.get("Pset_WallCommon", {}).get("IsExternal"):
        external_walls.append(wall)
```

### WHY

`get_psets()` traverses `IsDefinedBy` inverse relationships every time it is called. For a model with 10,000 walls and an average of 3 property sets each, calling `get_psets()` twice per wall means 60,000 relationship traversals instead of 10,000. Cache the results when you need to access properties multiple times per element.

---

## AP-4: Per-Element Spatial Containment Assignment

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")
storey = model.by_type("IfcBuildingStorey")[0]

for wall in model.by_type("IfcWall"):
    ifcopenshell.api.run("spatial.assign_container", model,
        relating_structure=storey, products=[wall])
    # Creates N separate IfcRelContainedInSpatialStructure entities!
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell.api

model = ifcopenshell.open("model.ifc")
storey = model.by_type("IfcBuildingStorey")[0]
walls = list(model.by_type("IfcWall"))

# Single call, single relationship entity
ifcopenshell.api.run("spatial.assign_container", model,
    relating_structure=storey, products=walls)
```

### WHY

Each `assign_container` call with a single product creates a separate `IfcRelContainedInSpatialStructure` entity. For 1000 walls, this creates 1000 relationship entities instead of 1. The batch version creates ONE relationship entity with all 1000 walls in its `RelatedElements` attribute. This is both faster (fewer API calls, fewer entity creations) and produces a cleaner, more compliant IFC file.

---

## AP-5: Accumulating All Geometry Shapes in Memory

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE / OOM RISK
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

all_shapes = []  # Accumulating all geometry in memory
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        all_shapes.append(shape)  # Keeps ALL geometry in RAM
        if not iterator.next():
            break

# all_shapes now holds gigabytes of vertex/face data
```

### Correct

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

results = []  # Store only the extracted data you need
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        # Extract ONLY what you need
        results.append({
            "guid": element.GlobalId,
            "vert_count": len(shape.geometry.verts) // 3,
            "face_count": len(shape.geometry.faces) // 3,
        })
        # shape data is not retained — can be garbage collected
        if not iterator.next():
            break
```

### WHY

Each shape object contains vertex arrays, face indices, material data, and transformation matrices. For a building model with 50,000 elements, storing all shapes can easily consume 10-20 GB of RAM. Extract only the data you need into plain Python objects and let the shape objects be garbage collected. If you need to write geometry to a file, stream it directly to disk inside the loop.

---

## AP-6: Using get_info(recursive=True) on Large Files

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE / OOM RISK
import ifcopenshell

model = ifcopenshell.open("large_model.ifc")

# This materializes the ENTIRE entity graph as nested Python dicts
for wall in model.by_type("IfcWall"):
    info = wall.get_info(recursive=True)
    # info contains the wall + ALL referenced entities as nested dicts
    # For a complex wall, this can be thousands of nested objects
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("large_model.ifc")

for wall in model.by_type("IfcWall"):
    # Non-recursive: only direct attributes of this entity
    info = wall.get_info()

    # Or access specific attributes directly
    name = wall.Name
    guid = wall.GlobalId
    type_name = wall.is_a()
```

### WHY

`get_info(recursive=True)` traverses ALL entity references recursively, converting them to Python dicts. A single wall can reference its placement, representation (with all geometry), property sets, type, spatial container, and materials — potentially thousands of entities deep. For 10,000 walls, this can easily exhaust available memory. Use `get_info()` (non-recursive) for direct attributes, or access specific attributes directly for best performance.

---

## AP-7: Opening the Same Large File Multiple Times

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE / WASTEFUL
import ifcopenshell

def get_walls(filepath):
    model = ifcopenshell.open(filepath)  # Loads entire file
    return [w.Name for w in model.by_type("IfcWall")]

def get_slabs(filepath):
    model = ifcopenshell.open(filepath)  # Loads entire file AGAIN
    return [s.Name for s in model.by_type("IfcSlab")]

walls = get_walls("building.ifc")
slabs = get_slabs("building.ifc")
# File loaded twice, doubling load time and peak memory
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell

model = ifcopenshell.open("building.ifc")  # Load ONCE

walls = [w.Name for w in model.by_type("IfcWall")]
slabs = [s.Name for s in model.by_type("IfcSlab")]
# Same model reference, no redundant loading
```

### WHY

`ifcopenshell.open()` parses the entire STEP file and constructs an in-memory object graph. For a 200MB file, this takes 10-30 seconds and 1-2 GB of RAM. Opening the same file twice doubles both time and memory. ALWAYS load the file once and pass the `model` reference to all functions that need it.

---

## AP-8: Using Too Many Threads on Memory-Constrained Systems

### Wrong

```python
# IfcOpenShell — BAD PRACTICE on low-RAM systems
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

# 32 threads on a system with 8GB RAM processing a 500MB file
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()  # Could be 32 threads
)
# Each thread allocates memory for OpenCASCADE operations
# Peak memory: base model (2-5 GB) + per-thread overhead (500MB-1GB each)
# Total: potentially 18-37 GB on a 32-core machine
```

### Correct

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

# Scale threads based on available memory
max_threads = multiprocessing.cpu_count()
# Rule of thumb: each thread needs ~500MB-1GB for geometry processing
# For 8GB system with 2GB used by model: (8-2)/1.0 = 6 threads max
safe_threads = min(max_threads, 4)  # Conservative default for large files

iterator = ifcopenshell.geom.iterator(
    settings, model, safe_threads
)
```

### WHY

The geometry iterator uses OpenMP for multi-threaded processing. Each thread allocates memory for OpenCASCADE boolean operations, tessellation buffers, and intermediate geometry. On a system with limited RAM, using all CPU cores can cause the operating system to start swapping (extremely slow) or trigger an out-of-memory kill. For memory-constrained scenarios, reduce thread count to 2-4 threads.

---

## AP-9: Not Releasing Model After Data Extraction

### Wrong

```python
# IfcOpenShell — BAD PRACTICE for memory-sensitive workflows
import ifcopenshell
import ifcopenshell.util.element

def analyze_model(filepath):
    model = ifcopenshell.open(filepath)

    data = []
    for wall in model.by_type("IfcWall"):
        data.append({
            "name": wall.Name,
            "psets": ifcopenshell.util.element.get_psets(wall)
        })

    # model is still alive, holding hundreds of MB
    # Even though we only need the extracted data
    return data, model  # Returning model keeps it in memory
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element
import gc

def analyze_model(filepath):
    model = ifcopenshell.open(filepath)

    data = []
    for wall in model.by_type("IfcWall"):
        data.append({
            "name": wall.Name,
            "psets": ifcopenshell.util.element.get_psets(wall)
        })

    # Explicitly release the model
    del model
    gc.collect()

    return data  # Only plain Python dicts, minimal memory
```

### WHY

IfcOpenShell's C++ core allocates memory outside Python's garbage collector. Even after all Python references to individual entities are gone, the model object holds the entire C++ object graph in memory. Explicitly deleting the model and calling `gc.collect()` ensures both Python and C++ memory is reclaimed. This is critical when processing multiple large files in sequence.

---

## AP-10: Ignoring the include Parameter on Iterator

### Wrong

```python
# IfcOpenShell — BAD PERFORMANCE
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

# Processes ALL elements with geometry — including furniture, MEP, etc.
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        if element.is_a("IfcWall"):  # Only needed walls!
            # Process wall...
            pass
        if not iterator.next():
            break
```

### Correct

```python
# IfcOpenShell v0.8+ — all schema versions
import ifcopenshell
import ifcopenshell.geom
import multiprocessing

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()

# Only process walls — skips geometry computation for everything else
walls = model.by_type("IfcWall")
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(),
    include=walls
)

if iterator.initialize():
    while True:
        shape = iterator.get()
        # Every shape IS a wall — no post-filtering needed
        if not iterator.next():
            break
```

### WHY

Without the `include` parameter, the iterator processes geometry for ALL elements in the model — potentially thousands of furniture items, MEP components, and spatial elements you don't need. Geometry processing is the most expensive operation in IfcOpenShell. If a model has 50,000 elements but you only need 2,000 walls, using `include` skips 96% of the work. The geometry engine never even touches the excluded elements.

---

## Sources

- [IfcOpenShell Geometry Processing](https://docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html)
- [IfcOpenShell file API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html)
- [GitHub Issue #6905 - Memory usage with geometry iterator](https://github.com/IfcOpenShell/IfcOpenShell/issues/6905)
- [GitHub Issue #2025 - Large IFC dataset strategies](https://github.com/IfcOpenShell/IfcOpenShell/issues/2025)
- [GitHub Issue #5026 - Slow file opening](https://github.com/IfcOpenShell/IfcOpenShell/issues/5026)
