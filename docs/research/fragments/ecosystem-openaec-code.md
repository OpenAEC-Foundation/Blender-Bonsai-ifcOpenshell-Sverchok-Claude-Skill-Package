# OpenAEC Foundation Code Repository Research

Research date: 2026-03-05
Repositories analyzed: 3
Method: Full clone and source code analysis

---

## 1. building.py

**Repository**: https://github.com/OpenAEC-Foundation/building-py
**What it is**: Python library for creating buildings, building systems, and objects with export to IFC, Speckle, FreeCAD, Revit, and DXF.
**Language**: Python 3.10+
**Dependencies**: ifcopenshell, specklepy, ezdxf, pythonnet, pillow

### Architecture

```
building-py/
├── abstract/          # Vector, Matrix, Color, Interval, Serializable
├── geometry/          # Point, Line, Arc, PolyCurve, Mesh, Extrusion
├── construction/      # Beam, Column, Panel, Wall, Floor, Door, Level
├── exchange/          # IFC, Speckle, DXF, CSV, GIS2BIM, FreeCAD, Revit
├── library/           # Steel profiles, materials, fill patterns
├── examples/          # Usage examples
└── BuildingPy.py      # Core project container
```

### IfcOpenShell API Usage Patterns

**File: exchange/IFC.py**

The library uses IfcOpenShell in two modes:

**A. Reading IFC (LoadIFC class)**:
```python
# exchange/IFC.py - Loading and geometry extraction
import ifcopenshell.geom
import ifcopenshell.util.element as util

settings = ifcopenshell.geom.settings()
settings.set(settings.SEW_SHELLS, True)

ifc_file = ifcopenshell.open(filename)
elements = ifc_file.by_type("IfcWall")  # Query by type

# Shape extraction
shape = ifcopenshell.geom.create_shape(settings, element)

# Property extraction via IsDefinedBy traversal
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
```

**Skill Relevance**: ifc-read, ifc-query, ifc-geometry-extraction

**B. Writing IFC (CreateIFC class)**:
```python
# exchange/IFC.py - IFC hierarchy creation
from ifcopenshell.api import run

class CreateIFC:
    def __init__(self):
        self.model = ifcopenshell.file()

    def add_project(self, name):
        self.project = self.model.createIfcProject(
            GlobalId=ifcopenshell.guid.new(),
            OwnerHistory=None,
            Name=name,
            RepresentationContexts=[],
            UnitsInContext=None
        )
        # Unit and context setup via run() API
        run("unit.assign_unit", self.model)
        self.context = run("context.add_context", self.model, context_type="Model")
        self.body = run("context.add_context", self.model,
            context_type="Model",
            context_identifier="Body",
            target_view="MODEL_VIEW",
            parent=self.context)

    def add_site(self, name):
        self.site = self.model.createIfcSite(
            GlobalId=ifcopenshell.guid.new(),
            Name=name
        )
        self.model.createIfcRelAggregates(
            GlobalId=ifcopenshell.guid.new(),
            RelatingObject=self.project,
            RelatedObjects=[self.site]
        )
```

**Skill Relevance**: ifc-create, ifc-hierarchy, ifc-export

### IFC Entities Created

| Entity | Purpose | File Location |
|--------|---------|---------------|
| IfcProject | Project container | exchange/IFC.py:171-181 |
| IfcSite | Site definition | exchange/IFC.py:190-213 |
| IfcBuilding | Building definition | exchange/IFC.py:216-237 |
| IfcBuildingStorey | Floor levels | exchange/IFC.py:240-259 |
| IfcBeam / IfcBeamType | Structural beams with profiles | exchange/IFC.py:342-371 |
| IfcSlab | Floor slabs with extrusion geometry | exchange/IFC.py:385-409 |
| IfcGrid | Grid systems with axes | exchange/IFC.py:426-455 |
| IfcBuildingElementProxy | Generic surface elements | exchange/IFC.py:325-335 |

### IFC Geometry Representation Patterns

**Profile-based geometry (beams/columns)**:
```python
# exchange/IFC.py:289-331 - Custom profiles with voids
externe_ifc_punten = [
    ifc_creator.model.create_entity('IfcCartesianPoint', Coordinates=p)
    for p in externe_punten
]
externe_polyline = ifc_creator.model.create_entity('IfcPolyline', Points=externe_ifc_punten)
custom_profile = ifc_creator.model.create_entity(
    'IfcArbitraryProfileDefWithVoids',
    ProfileType='AREA',
    ProfileName=object_type.name,
    OuterCurve=externe_polyline,
    InnerCurves=interne_polyline_lists
)
```

**Entity creation via run() API**:
```python
# exchange/IFC.py - Beam type creation and assignment
beam_type = run("root.create_entity", model, ifc_class="IfcBeamType", name=name)
material_set = run("material.add_material_set", model,
    name=name, set_type="IfcMaterialProfileSet")
material = run("material.add_material", model, name="", category="steel")
run("type.assign_type", model, related_object=instance, relating_type=beam_type)
run("material.assign_material", model, product=instance, material=material_set)
run("geometry.assign_representation", model, product=instance, representation=shape)
run("spatial.assign_container", model, relating_structure=storey, product=instance)
```

**Skill Relevance**: ifc-geometry, ifc-profiles, ifc-materials, ifc-spatial

### Blender (bpy) Patterns

**Finding: No Blender (bpy) integration exists in this repository.** The exchange/ directory exports to Speckle, FreeCAD, Revit, DXF, and IFC — but not to Blender directly. No `import bpy` found anywhere.

**Skill Relevance**: This confirms a gap that our skill package fills — building.py creates the geometry model but has no Blender export path.

### Core Geometry Model

**Serializable base class** (abstract/serializable.py):
All objects inherit from `Serializable` enabling JSON persistence.

**Key construction objects**:

```python
# construction/beam.py - Beam with profile-based extrusion
class Beam(Serializable, Meshable):
    def __init__(self, start: Point, end: Point, profile: Profile | str,
                 name: str = "Beam", material: Material = BaseSteel,
                 angle: float = 0, justification: list[str] | Vector = Vector()):
        if isinstance(profile, str):
            profile = profile_by_name(profile)  # Lookup by name
        self.start = start
        self.end = end
        self.profile = profile

    @property
    def extrusion(self) -> Extrusion:
        return Extrusion.by_2d_polycurve_vector(
            self.profile.curve, self.start, self.end - self.start)
```

```python
# construction/panel.py - Panel from polycurve + thickness
class Panel(Serializable, Meshable):
    @classmethod
    def by_polycurve_thickness(cls, polycurve, thickness, offset=0,
                                name=None, material=BaseTimber):
        return Panel(
            Extrusion.by_polycurve_height(polycurve, thickness, offset),
            material, name)

    @classmethod
    def by_baseline_height(cls, baseline, height, thickness,
                           name=None, material=BaseTimber):
        polycurve = PolyCurve.by_points([
            baseline.start, baseline.end,
            baseline.end + Vector(0, 0, height),
            baseline.start + Vector(0, 0, height)])
        return Panel(
            Extrusion.by_polycurve_height(polycurve, thickness, 0),
            material, name)
```

**Skill Relevance**: blender-geometry, parametric-modeling, mesh-generation

### Design Patterns

1. **Factory Method Pattern**: `Line.by_start_end()`, `Arc.by_start_mid_end()`, `Extrusion.by_polycurve_height()`, `Panel.by_polycurve_thickness()`
2. **Multi-Target Export**: Each exchange module translates the same internal objects to a different format
3. **Serializable Base**: JSON save/load for all geometric objects
4. **Profile Lookup**: Beams accept profile names as strings, resolved via `profile_by_name()`

### Anti-Patterns Found

| Issue | Location | Description |
|-------|----------|-------------|
| Mixed API styles | exchange/IFC.py | Both `run()` API and direct `create_entity()` used interchangeably — inconsistent |
| Debug prints in production | exchange/IFC.py:104-112 | `print(a["Name"])`, `print(shape)`, `print(mesh)` |
| Silent exception handling | exchange/speckle.py:157-158 | `except Exception as e: print(e)` — failures swallowed |
| Hard-coded values | exchange/IFC.py:400 | `thickness = 1` — no parameter |
| Missing type hints | construction/wall.py, door.py | No type annotations on methods |
| Incomplete implementations | construction/wall.py, door.py | Skeleton classes with `by_mesh()` stubs |
| 18+ TODO items | Across codebase | Unfinished rotation math, duplicate point removal, coordinate bugs |
| Z-axis bug | BuildingPy.py:3203 | "z axis is pointing the other way when angle > PI" |
| Polygon winding issues | BuildingPy-gis2bim.py:5270 | "TODO: correct winding" — unresolved |

---

## 2. GIS-to-Blender_3DEnvironment_Automation

**Repository**: https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
**Status**: Template/placeholder repository — contains no implementation code.

### Contents

The repository contains only:
- `README.md`
- `LICENSE.md`

### README Content

```
With this repo you can prompt LLM's to build a 3D environment in Blender
Mainly built for Claude Code
```

### Analysis

This repository establishes intent for LLM-driven Blender automation using Claude Code. It is a skeleton repository without actual Python code, Blender scripts, or IFC integration.

**No code patterns to extract.** The repository serves as a placeholder for a future workflow where:
- LLMs (specifically Claude Code) generate Blender Python scripts
- GIS data feeds into Blender 3D environment creation
- The workflow operates via headless/CLI Blender

**Skill Relevance**: The intent aligns directly with our skill package goals — Claude Code generating Blender/IFC automation scripts. Our skills would provide the structured knowledge this repo's approach requires.

### Anti-Patterns

- Empty repository published as a "tool" — no implementation behind the README
- No examples, no tests, no code structure

---

## 3. AEC Scripts

**Repository**: https://github.com/OpenAEC-Foundation/aec-scripts
**What it is**: Collection of 15 Revit/pyRevit automation scripts for AEC workflows.
**Language**: IronPython (Revit), C# (viewers)
**Total size**: ~14,825 lines of Python across 16 files

### Repository Structure

```
aec-scripts/
├── GIS2BIM.pushbutton/          # 3,178 lines — Dutch GIS → Revit
│   ├── script.py
│   └── cityjson_parser.py       # CityJSON parsing
├── SCAN2BIM.pushbutton/         # 993 lines — Point cloud → Revit
├── 3BMkozijn.pushbutton/        # 897 lines — Parametric window frames
├── trapgenerator.pushbutton/    # 1,585 lines — Parametric staircases
├── SheetGenerator.pushbutton/   # 1,547 lines — Document automation
├── FilledRegionPicker.pushbutton/ # 1,869 lines — Hatch selection UI
├── ToolbarManager.pushbutton/   # 846 lines — UI configuration
├── familymanager.pushbutton/    # 761 lines — Family library management
├── autodimensionering.pushbutton/ # 690 lines — Auto-dimensioning
├── legendgenerator.pushbutton/  # 638 lines — Legend creation
├── importparameters.pushbutton/ # 265 lines — CSV → Revit parameters
├── exportparameters.pushbutton/ # 201 lines — Revit → CSV parameters
├── 3BM opname tool/             # Survey tool
├── 3BM viewer/                  # C# Revit Add-in
└── RevitAddinConfigurator/      # C# .NET configuration
```

### IfcOpenShell and Blender Usage

**Finding: This repository contains zero imports of ifcopenshell or bpy.** It is exclusively Revit API / IronPython based. All geometry operations use the Revit API (`Autodesk.Revit.DB`).

### Relevant Patterns for AEC Skill Package

#### CityJSON Parsing and Triangulation

```python
# GIS2BIM.pushbutton/cityjson_parser.py — CityJSON geometry handling

def parse_cityjson_vertices(raw_vertices):
    """Parse raw vertices from 3D BAG API (string or array format)"""
    vertices = []
    for v in raw_vertices:
        if isinstance(v, str):
            parts = v.split()
            if len(parts) >= 3:
                vertices.append([int(parts[0]), int(parts[1]), int(parts[2])])
        elif isinstance(v, (list, tuple)) and len(v) >= 3:
            vertices.append([v[0], v[1], v[2]])
    return vertices

def triangulate_polygon(vertex_indices):
    """Fan triangulation for non-triangle polygons"""
    if len(vertex_indices) < 3:
        return []
    if len(vertex_indices) == 3:
        return [tuple(vertex_indices)]
    triangles = []
    v0 = vertex_indices[0]
    for i in range(1, len(vertex_indices) - 1):
        triangles.append((v0, vertex_indices[i], vertex_indices[i + 1]))
    return triangles

def extract_lod22_faces(geometry):
    """Extract faces from CityJSON geometry (LOD 2.2)
    Handles both Solid and MultiSurface types"""
    geom_type = geometry.get('type', '')
    boundaries = geometry.get('boundaries', [])
    faces = []
    if geom_type == 'Solid':
        for shell in boundaries:
            for surface in shell:
                if surface and len(surface) > 0:
                    ring = surface[0] if isinstance(surface[0], list) else surface
                    if len(ring) >= 3:
                        faces.extend(triangulate_polygon(ring))
    elif geom_type == 'MultiSurface':
        for surface in boundaries:
            if surface and len(surface) > 0:
                ring = surface[0] if isinstance(surface[0], list) else surface
                if len(ring) >= 3:
                    faces.extend(triangulate_polygon(ring))
    return faces

def transform_vertices(vertices, scale, translate, rd_x, rd_y):
    """Apply CityJSON transform and project-relative coordinates"""
    converted = []
    for vx, vy, vz in vertices:
        x = float(vx) * scale[0] + translate[0]
        y = float(vy) * scale[1] + translate[1]
        z = float(vz) * scale[2] + translate[2]
        converted.append((x - rd_x, y - rd_y, z))
    return converted
```

**Skill Relevance**: blender-mesh-import, cityjson-to-blender, gis-coordinate-transform

#### DirectShape Mesh Generation (Revit equivalent of Blender mesh creation)

```python
# GIS2BIM.pushbutton/script.py — Mesh from CityJSON data

def create_directshapes_from_buildings(doc, buildings):
    """Create Revit DirectShape elements from 3D BAG buildings"""
    for building in buildings:
        vertices = building['vertices']
        polygon_faces = building.get('polygon_faces', [])

        # Create vertex list
        xyz_verts = [XYZ(meters_to_internal(v[0]),
                        meters_to_internal(v[1]),
                        meters_to_internal(v[2])) for v in vertices]

        # Create triangulated mesh
        for tri_indices in polygon_faces:
            face_verts = List[XYZ]()
            for idx in tri_indices:
                face_verts.Add(xyz_verts[idx])

        mesh = Mesh.CreateTriangleMesh(face_verts, triangle_indices)
        ds = DirectShape.CreateElement(doc, category_id)
        ds.SetShape([mesh])
```

**Skill Relevance**: The vertex-face mesh creation pattern maps directly to Blender's `bpy.types.Mesh.from_pydata(vertices, edges, faces)` pattern. CityJSON → mesh is a reusable workflow.

#### Parametric Geometry (Extrusion from CurveLoop)

```python
# 3BMkozijn.pushbutton/script.py — Rectangular extrusion

def _create_rectangular_extrusion(self, origin, width, height, depth):
    """Create rectangular extrusion geometry via CurveLoop"""
    profile = CurveLoop()
    p1 = origin
    p2 = origin + XYZ(width, 0, 0)
    p3 = origin + XYZ(width, height, 0)
    p4 = origin + XYZ(0, height, 0)

    profile.Append(Line.CreateBound(p1, p2))
    profile.Append(Line.CreateBound(p2, p3))
    profile.Append(Line.CreateBound(p3, p4))
    profile.Append(Line.CreateBound(p4, p1))

    solid = GeometryCreationUtilities.CreateExtrusionGeometry(
        [profile], XYZ(0, 0, 1), depth)
```

**Skill Relevance**: Maps to Blender's `bmesh.ops.extrude_face_region()` or `bpy.ops.mesh.extrude_region_move()` patterns. The CurveLoop → extrusion workflow is equivalent to PolyCurve → Extrusion in building.py.

#### GIS Web Service Integration

```python
# GIS2BIM.pushbutton/script.py — 60+ Dutch GIS data sources

GIS_DATA_LAYERS = {
    'bag_3d_cityjson': {
        'name': '3D BAG LOD2.2 (CityJSON)',
        'category': '3D Data',
        'type': '3dbag_cityjson',
        'url': 'https://api.3dbag.nl'
    },
    'bgt_wegdelen': {
        'name': 'BGT Wegdelen',
        'category': '2D Vectordata',
        'type': 'ogcapi',
        'url': 'https://api.pdok.nl/lv/bgt/ogc/v1',
        'collection': 'wegdeel'
    },
    # ... 60+ more layers (WMTS, WMS, WFS, OGC API)
}
```

**Skill Relevance**: gis-to-blender, dutch-gis-integration

#### Revit Transaction Pattern (comparable to Blender undo)

```python
# Common Revit pattern across all scripts
from pyrevit import revit

with revit.Transaction("Create elements"):
    # All modifications inside transaction
    element = create_element(doc, params)
    element.set_parameter(value)
# Auto-commit on exit, auto-rollback on exception
```

**Skill Relevance**: Maps to Blender's `bpy.ops` context and undo system. The transaction pattern is a best practice for atomic operations.

### Anti-Patterns Found

| Issue | Location | Description |
|-------|----------|-------------|
| Bare except clauses | GIS2BIM script.py:75-94 | Failure handler catches everything silently |
| Hard-coded limits | SCAN2BIM script.py:82 | `max_points=100000` — no configuration |
| Memory concerns | GIS2BIM script.py:1095-1106 | Large bitmap tiles combined without streaming |
| Geometry validation gaps | GIS2BIM script.py:1642 | Distance check but no co-linearity validation |
| IronPython workarounds | GIS2BIM script.py:53-68 | Manual struct construction instead of proper API |

---

## Summary Table

| Repository | Key Findings | Impacted Skills |
|------------|-------------|-----------------|
| **building.py** | IfcOpenShell `run()` API for entity creation; IFC hierarchy (Project→Site→Building→Storey); Profile-based beam geometry via `IfcArbitraryProfileDefWithVoids`; Extrusion-based panels/slabs; Factory method pattern for geometry; No Blender export (gap our skills fill) | ifc-create, ifc-hierarchy, ifc-geometry, ifc-profiles, ifc-materials, parametric-modeling |
| **building.py** | `ifcopenshell.geom.create_shape()` for mesh extraction; Property extraction via `IsDefinedBy` traversal; `ifcopenshell.geom.settings()` with `SEW_SHELLS` | ifc-read, ifc-query, ifc-geometry-extraction |
| **building.py** | Mixed `run()` vs `create_entity()` API usage; Debug prints; Silent exceptions; 18+ TODO items | Anti-pattern: API consistency, error handling |
| **GIS-to-Blender** | Empty repository — placeholder for LLM-driven Blender automation via Claude Code | Validates our skill package approach |
| **aec-scripts** | CityJSON parsing with triangulation; Coordinate transforms (RD→local); 60+ Dutch GIS data sources | cityjson-import, gis-coordinate-transform, mesh-from-external-data |
| **aec-scripts** | DirectShape mesh from vertices+faces (maps to `Mesh.from_pydata()`); Extrusion from CurveLoop profiles | blender-mesh-creation, blender-extrusion |
| **aec-scripts** | Parametric window frames (kozijn) and staircases (trap) with configurable dimensions | parametric-modeling, construction-automation |
| **aec-scripts** | Zero IfcOpenShell/bpy usage — purely Revit API | Gap: skills bridge Revit patterns to Blender/IFC |

## Cross-Repository Pattern Analysis

### Common IFC Creation Pattern (from building.py)

The canonical pattern for creating IFC files from Python:

```
1. ifcopenshell.file()                           → Create empty model
2. run("unit.assign_unit", model)                → Set units
3. run("context.add_context", model, ...)        → Add representation context
4. createIfcProject() + createIfcSite() + ...    → Build spatial hierarchy
5. run("root.create_entity", model, ...)         → Create typed elements
6. run("material.assign_material", ...)          → Assign materials
7. run("geometry.assign_representation", ...)    → Assign geometry
8. run("spatial.assign_container", ...)          → Place in hierarchy
9. model.write(filename)                         → Export
```

**Skill Relevance**: This 9-step pattern is the reference implementation for our ifc-create skill.

### Common Mesh Pattern (from aec-scripts, applicable to Blender)

The CityJSON → mesh pipeline from aec-scripts translates directly to Blender:

```
Revit:   XYZ(x,y,z) → Mesh.CreateTriangleMesh(verts, faces) → DirectShape.SetShape([mesh])
Blender: (x,y,z)    → mesh.from_pydata(verts, edges, faces)  → object.data = mesh
```

**Skill Relevance**: blender-mesh-import, cityjson-to-blender

### Geometry Creation Strategy

Both building.py and aec-scripts use the same core strategy:
1. Define a 2D profile (PolyCurve / CurveLoop)
2. Extrude along a direction vector
3. Assign material/properties
4. Place in spatial hierarchy

This maps to Blender as:
1. Create `bmesh` face from vertices
2. `bmesh.ops.extrude_face_region()` along normal
3. Assign material slot
4. Parent to collection

**Skill Relevance**: blender-geometry, blender-extrusion, parametric-modeling
