# Utility Module Anti-Patterns

Common mistakes when using `ifcopenshell.util.*` modules. Each anti-pattern explains WHAT is wrong and WHY it causes problems.

---

## AP-1: Manual Inverse Traversal Instead of Using util.element

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Manual traversal to get property sets
psets = {}
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset_def = rel.RelatingPropertyDefinition
        if pset_def.is_a("IfcPropertySet"):
            props = {}
            for prop in pset_def.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    props[prop.Name] = prop.NominalValue.wrappedValue if prop.NominalValue else None
            psets[pset_def.Name] = props
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

psets = ifcopenshell.util.element.get_psets(wall)
```

### WHY

Manual inverse traversal is verbose, error-prone, and schema-dependent. It does not handle `IfcPropertyEnumeratedValue`, `IfcPropertyListValue`, `IfcPropertyBoundedValue`, `IfcComplexProperty`, or quantity sets. It also misses type-inherited properties. `get_psets()` handles all property types, all schema versions, and type inheritance in a single call.

---

## AP-2: Using Deprecated Selector().parse() Instead of filter_elements()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE (DEPRECATED)
import ifcopenshell
from ifcopenshell.util.selector import Selector

model = ifcopenshell.open("model.ifc")
selector = Selector()
walls = selector.parse(model, ".IfcWall")
external = selector.parse(model, '.IfcWall[Pset_WallCommon.IsExternal = "True"]')
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")
external = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')
```

### WHY

`Selector().parse()` is deprecated. The query syntax is different: `filter_elements()` does NOT use the `.` prefix for class names and uses `, /PsetName/.PropName = value` for property filtering. The old syntax used `[PsetName.PropName = "value"]`. Using the deprecated API produces warnings and may break in future IfcOpenShell versions.

---

## AP-3: Ignoring Unit Scale When Reading Coordinates

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.placement

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
x, y, z = matrix[0][3], matrix[1][3], matrix[2][3]
print(f"Position: ({x}, {y}, {z}) metres")  # WRONG — values are in project units
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.placement
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

wall = model.by_type("IfcWall")[0]
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
x = matrix[0][3] * unit_scale
y = matrix[1][3] * unit_scale
z = matrix[2][3] * unit_scale
print(f"Position: ({x}, {y}, {z}) metres")
```

### WHY

IFC files store coordinates in project units, which are defined by the file's `IfcUnitAssignment`. A file using millimetres stores `5000.0` for a 5-metre wall. Without multiplying by `calculate_unit_scale()`, coordinate values are interpreted incorrectly. This is the most common source of scale errors in IFC data extraction.

---

## AP-4: Calling shape Utilities on Raw IFC Entities

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.shape

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Passing raw entity — CRASHES
volume = ifcopenshell.util.shape.get_volume(wall)
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.shape

model = ifcopenshell.open("model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

wall = model.by_type("IfcWall")[0]

# Process geometry first
shape = ifcopenshell.geom.create_shape(settings, wall)

# Pass processed geometry, NOT raw entity
volume = ifcopenshell.util.shape.get_volume(shape.geometry)
```

### WHY

`ifcopenshell.util.shape` functions operate on processed geometry objects produced by `ifcopenshell.geom.create_shape()`. Raw IFC entities contain geometry definitions (IfcExtrudedAreaSolid, IfcBooleanResult, etc.) that must be evaluated by the geometry kernel (OpenCASCADE) before metrics like volume or area can be calculated. Passing a raw entity raises an `AttributeError` or produces incorrect results.

---

## AP-5: Not Checking for None Returns

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Assuming type always exists — crashes on untyped elements
wall_type = ifcopenshell.util.element.get_type(wall)
print(wall_type.Name)  # AttributeError: 'NoneType' has no attribute 'Name'

# Assuming container always exists
container = ifcopenshell.util.element.get_container(wall)
print(container.Name)  # Same error
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    print(f"Type: {wall_type.Name}")
else:
    print("No type assigned")

container = ifcopenshell.util.element.get_container(wall)
if container:
    print(f"Container: {container.Name}")
```

### WHY

IFC elements are NOT guaranteed to have types, containers, materials, or classifications. Untyped elements return `None` from `get_type()`. Uncontained elements return `None` from `get_container()`. Elements without material assignments return `None` from `get_material()`. ALWAYS check for `None` before accessing attributes on returned entities.

---

## AP-6: Assuming Property Set Names Are Universal

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Assuming Pset_WallCommon always exists
pset = ifcopenshell.util.element.get_pset(wall, "Pset_WallCommon")
is_external = pset["IsExternal"]  # KeyError or TypeError if pset is None
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Safe access with fallback
is_external = ifcopenshell.util.element.get_pset(
    wall, "Pset_WallCommon", "IsExternal")
if is_external is not None:
    print(f"IsExternal: {is_external}")
else:
    print("Property not available")
```

### WHY

IFC property sets are NOT guaranteed to exist. `Pset_WallCommon` is a standard property set defined by the IFC specification, but it is optional. Many IFC authoring tools do not populate standard property sets, or use custom property set names. ALWAYS handle the case where `get_pset()` returns `None`.

---

## AP-7: Using Storey.Elevation Instead of get_storey_elevation()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell

model = ifcopenshell.open("model.ifc")
storey = model.by_type("IfcBuildingStorey")[0]

# Elevation attribute may be None or inconsistent with placement
elevation = storey.Elevation
print(f"Storey at {elevation}")  # Could be None
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.placement

model = ifcopenshell.open("model.ifc")
storey = model.by_type("IfcBuildingStorey")[0]

elevation = ifcopenshell.util.placement.get_storey_elevation(storey)
print(f"Storey at {elevation}")
```

### WHY

The `Elevation` attribute on `IfcBuildingStorey` is optional and may be `None`. Even when present, it may be inconsistent with the storey's actual placement in the spatial hierarchy. `get_storey_elevation()` checks the placement Z coordinate first (which reflects the actual model geometry) and falls back to the `Elevation` attribute only if the placement is not available. This produces more reliable results.

---

## AP-8: Forgetting should_inherit=True When Querying Properties

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Explicitly disabling inheritance without understanding the consequence
psets = ifcopenshell.util.element.get_psets(wall, should_inherit=False)
# Missing type-level properties like Pset_WallCommon.IsExternal
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Default: should_inherit=True — includes type-level psets
psets = ifcopenshell.util.element.get_psets(wall)

# Only use should_inherit=False when you specifically need
# to distinguish occurrence vs type properties
own = ifcopenshell.util.element.get_psets(wall, should_inherit=False)
inherited = ifcopenshell.util.element.get_psets(
    ifcopenshell.util.element.get_type(wall)) if ifcopenshell.util.element.get_type(wall) else {}
```

### WHY

In IFC, properties can be defined at both the occurrence level (on the element itself) and the type level (on the IfcTypeObject). Standard practice is to define common properties on the type. If you disable inheritance with `should_inherit=False`, you miss properties defined on the type, which are the majority of standard properties in most IFC files.

---

## AP-9: Mixing Up get_material() and get_materials()

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# get_material returns the material SET entity, not individual materials
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material: {material.Name}")  # May crash — IfcMaterialLayerSetUsage has no Name
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# get_material() returns the material association entity
# (IfcMaterial, IfcMaterialLayerSetUsage, IfcMaterialProfileSet, etc.)
material = ifcopenshell.util.element.get_material(wall)
if material:
    print(f"Material type: {material.is_a()}")

# get_materials() returns a flat list of individual IfcMaterial entities
materials = ifcopenshell.util.element.get_materials(wall)
for mat in materials:
    print(f"Material: {mat.Name}")  # Safe — IfcMaterial always has Name
```

### WHY

`get_material()` returns the material association entity, which can be `IfcMaterial`, `IfcMaterialLayerSet`, `IfcMaterialLayerSetUsage`, `IfcMaterialProfileSet`, or `IfcMaterialConstituentSet`. Not all of these have a `Name` attribute. `get_materials()` returns a flat list of individual `IfcMaterial` entities extracted from any material structure, which always have `Name`.

---

## AP-10: Not Using calculate_unit_scale() for Area/Volume Units

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

# Only getting length scale
length_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

# Wrong: applying length scale to area or volume
raw_area = 25000000.0  # mm²
area_m2 = raw_area * length_scale  # This gives mm * 0.001 = mm, not m²
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.unit

model = ifcopenshell.open("model.ifc")

length_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

# For area: square the length scale
area_scale = length_scale ** 2
raw_area = 25000000.0  # mm²
area_m2 = raw_area * area_scale  # 25.0 m²

# For volume: cube the length scale
volume_scale = length_scale ** 3
raw_volume = 5000000000.0  # mm³
volume_m3 = raw_volume * volume_scale  # 5.0 m³

# OR use calculate_unit_scale with specific unit type
area_scale_direct = ifcopenshell.util.unit.calculate_unit_scale(
    model, "AREAUNIT")
volume_scale_direct = ifcopenshell.util.unit.calculate_unit_scale(
    model, "VOLUMEUNIT")
```

### WHY

`calculate_unit_scale(model)` defaults to `"LENGTHUNIT"`, returning a linear scale factor (e.g., 0.001 for millimetres). For area, the factor must be squared (0.001² = 0.000001). For volume, it must be cubed (0.001³ = 0.000000001). Applying the linear scale to area or volume values produces results off by orders of magnitude. Either square/cube the linear scale or call `calculate_unit_scale()` with `"AREAUNIT"` or `"VOLUMEUNIT"` explicitly.

---

## AP-11: Using filter_elements() with Wrong Query Syntax

### Wrong

```python
# IfcOpenShell — BAD PRACTICE
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# Wrong: using dot prefix (old Selector().parse() syntax)
walls = ifcopenshell.util.selector.filter_elements(model, ".IfcWall")

# Wrong: using bracket syntax (old Selector().parse() syntax)
external = ifcopenshell.util.selector.filter_elements(
    model, '.IfcWall[Pset_WallCommon.IsExternal = "True"]')
```

### Correct

```python
# IfcOpenShell — all schema versions
import ifcopenshell
import ifcopenshell.util.selector

model = ifcopenshell.open("model.ifc")

# No dot prefix for class names
walls = ifcopenshell.util.selector.filter_elements(model, "IfcWall")

# Use comma-separated facets and /PsetName/.PropName syntax
external = ifcopenshell.util.selector.filter_elements(
    model, 'IfcWall, /Pset_WallCommon/.IsExternal = True')
```

### WHY

`filter_elements()` uses a different query syntax than the deprecated `Selector().parse()`. Class names do NOT use dot prefix. Property filtering uses `, /PsetName/.PropName = value` with comma separation, NOT `[PsetName.PropName = "value"]` with bracket notation. Using the wrong syntax produces empty results or errors without clear error messages.
