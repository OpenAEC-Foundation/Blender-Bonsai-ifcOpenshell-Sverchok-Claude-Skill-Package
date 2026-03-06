# ifcos-impl-geometry — Anti-Patterns

Common geometry mistakes and how to avoid them.

---

## Anti-Pattern 1: No Error Handling on create_shape()

**Wrong:**
```python
shape = ifcopenshell.geom.create_shape(settings, element)
verts = shape.geometry.verts  # Crashes if element has no geometry
```

**Correct:**
```python
try:
    shape = ifcopenshell.geom.create_shape(settings, element)
except RuntimeError:
    shape = None  # IfcBuildingStorey, IfcProject, some IfcSite have no geometry

if shape:
    verts = np.array(shape.geometry.verts).reshape(-1, 3)
```

**Reason:** Not all IfcProduct subtypes have geometry. `create_shape()` raises `RuntimeError` for elements without processable geometry (IfcProject, IfcBuildingStorey, some IfcSite instances).

---

## Anti-Pattern 2: Using create_shape() in a Loop for Large Sets

**Wrong:**
```python
for wall in model.by_type("IfcWall"):
    shape = ifcopenshell.geom.create_shape(settings, wall)  # 5-10x slower
```

**Correct:**
```python
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count(),
    include=model.by_type("IfcWall"))

if iterator.initialize():
    while True:
        shape = iterator.get()
        if not iterator.next():
            break
```

**Reason:** The iterator uses internal caching, multithreading, and batch processing. For 100+ elements it is 5-10x faster than individual `create_shape()` calls.

---

## Anti-Pattern 3: Assuming Meters

**Wrong:**
```python
verts = np.array(shape.geometry.verts).reshape(-1, 3)
volume = calculate_volume(verts)  # Wrong if model uses millimeters
```

**Correct:**
```python
import ifcopenshell.util.unit
scale = ifcopenshell.util.unit.calculate_unit_scale(model)
verts = np.array(shape.geometry.verts).reshape(-1, 3) * scale
```

**Reason:** IFC files can use any length unit (millimeters, inches, feet). Always check and apply the unit scale factor.

---

## Anti-Pattern 4: Creating Geometry Without a Context

**Wrong:**
```python
model = ifcopenshell.api.run("project.create_file", version="IFC4")
# Skipping context creation...
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=None, length=5.0, height=3.0, thickness=0.2)  # Fails
```

**Correct:**
```python
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="My Project")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model,
    context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

**Reason:** All geometry representations require an IfcGeometricRepresentationSubContext. Create the root "Model" context first, then a "Body" subcontext.

---

## Anti-Pattern 5: Forgetting Object Placement

**Wrong:**
```python
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)
# Missing placement — element has no defined position
```

**Correct:**
```python
wall = ifcopenshell.api.run("root.create_entity", model, ifc_class="IfcWall")
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=wall_rep)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
```

**Reason:** Elements without an IfcLocalPlacement have undefined position. Always call `edit_object_placement()` even for elements at the origin (pass no matrix for identity placement).

---

## Anti-Pattern 6: Using Low-Level Entity Creation Instead of API

**Wrong:**
```python
# Manual extrusion creation — error-prone, misses validation
direction = model.create_entity("IfcDirection", DirectionRatios=[0.0, 0.0, 1.0])
profile = model.create_entity("IfcRectangleProfileDef",
    ProfileType="AREA", XDim=5.0, YDim=0.2)
solid = model.create_entity("IfcExtrudedAreaSolid",
    SweptArea=profile, Depth=3.0, ExtrudedDirection=direction)
# Still need to create representation, assign context, etc.
```

**Correct:**
```python
wall_rep = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
```

**Reason:** The `geometry.add_*_representation()` API functions handle context assignment, representation type selection, and validation automatically. Only use `model.create_entity()` when no API function exists for your specific geometry type.

---

## Anti-Pattern 7: Comparing Positions Without World Coordinates

**Wrong:**
```python
settings = ifcopenshell.geom.settings()
# USE_WORLD_COORDS defaults to False
shape_a = ifcopenshell.geom.create_shape(settings, element_a)
shape_b = ifcopenshell.geom.create_shape(settings, element_b)
# Comparing local-space coordinates — meaningless across elements
distance = np.linalg.norm(verts_a[0] - verts_b[0])
```

**Correct:**
```python
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)
shape_a = ifcopenshell.geom.create_shape(settings, element_a)
shape_b = ifcopenshell.geom.create_shape(settings, element_b)
distance = np.linalg.norm(verts_a[0] - verts_b[0])
```

**Reason:** Without `USE_WORLD_COORDS`, vertex positions are relative to each element's local origin. Cross-element spatial comparisons require world-space coordinates.
