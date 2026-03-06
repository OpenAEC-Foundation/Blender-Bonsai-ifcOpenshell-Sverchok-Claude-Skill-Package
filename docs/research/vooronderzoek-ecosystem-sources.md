# Vooronderzoek F: Ecosystem Sources

**Date:** 2026-03-06
**Scope:** OpenAEC Foundation projects, Claude/Anthropic skill development platform, and cross-technology patterns
**Purpose:** Consolidate ecosystem research from three parallel research agents into a single reference document for the Blender-Bonsai-IfcOpenShell-Sverchok Claude Skill Package
**Status:** Complete (consolidated from 3 fragments)

**Research agents:**
1. `research-openaec-code` — building.py, GIS-to-Blender, AEC Scripts (2026-03-05)
2. `research-openaec-ifc` — Monty IFC Viewer, INB Template, Nextcloud/IfcGit (2026-03-05)
3. `research-claude-platform` — Claude skill specification, YAML format, triggering (2026-03-05)

---

## Table of Contents

- [Part 1: OpenAEC Foundation Related Projects](#part-1-openaec-foundation-related-projects)
  - [1.1 building.py](#11-buildingpy)
  - [1.2 GIS-to-Blender 3D Environment Automation](#12-gis-to-blender-3denvironment-automation)
  - [1.3 AEC Scripts](#13-aec-scripts)
  - [1.4 Monty IFC Viewer](#14-monty-ifc-viewer)
  - [1.5 INB Template](#15-inb-template)
  - [1.6 Nextcloud Check-in/Check-out & IfcGit-4-Nextcloud](#16-nextcloud-check-incheck-out--ifcgit-4-nextcloud)
- [Part 2: Claude/Anthropic Skill Development Platform](#part-2-claudeanthropic-skill-development-platform)
  - [2.1 Official Skill Specification](#21-official-skill-specification-agent-skills-open-standard)
  - [2.2 Skill Discovery and Loading](#22-skill-discovery-and-loading-mechanism)
  - [2.3 Content Best Practices](#23-skill-content-best-practices)
  - [2.4 Gap Analysis](#24-gap-analysis-our-format-vs-official-specification)
  - [2.5 Optimization Opportunities](#25-optimization-opportunities)
  - [2.6 ERPNext Comparison](#26-comparison-with-erpnext-skill-package)
  - [2.7 Recommendations](#27-recommendations-for-our-skill-package)
- [Part 3: Cross-Technology Patterns](#part-3-cross-technology-patterns)
  - [3.1 IFC Property Extraction](#31-ifc-property-extraction-pattern)
  - [3.2 IFC Creation Pipeline](#32-ifc-creation-pipeline)
  - [3.3 Mesh Generation Pipeline](#33-mesh-generation-pipeline)
  - [3.4 Geometry Strategy](#34-geometry-creation-strategy)
  - [3.5 Classification Architecture](#35-classification-architecture)
  - [3.6 Concurrent File Access](#36-concurrent-file-access-patterns)
  - [3.7 Schema Version Convergence](#37-schema-version-convergence)
  - [3.8 Anti-Pattern Summary](#38-anti-pattern-summary-across-all-repositories)
- [Summary Table](#summary-table)

---

# Part 1: OpenAEC Foundation Related Projects

## 1.1 building.py

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

**Skill Relevance**: `ifcos-syntax-elements`, `ifcos-syntax-geometry`, `ifcos-impl-extraction`

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

**Skill Relevance**: `ifcos-syntax-api`, `ifcos-impl-creation`, `bonsai-syntax-spatial`

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

**Skill Relevance**: `ifcos-syntax-api`, `ifcos-syntax-geometry`, `bonsai-impl-modeling`

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

**Skill Relevance**: `blender-impl-mesh`, `blender-syntax-mesh`

### Design Patterns

1. **Factory Method Pattern**: `Line.by_start_end()`, `Arc.by_start_mid_end()`, `Extrusion.by_polycurve_height()`, `Panel.by_polycurve_thickness()`
2. **Multi-Target Export**: Each exchange module translates the same internal objects to a different format
3. **Serializable Base**: JSON save/load for all geometric objects
4. **Profile Lookup**: Beams accept profile names as strings, resolved via `profile_by_name()`

### Blender (bpy) Integration

**Finding: No Blender (bpy) integration exists in this repository.** The exchange/ directory exports to Speckle, FreeCAD, Revit, DXF, and IFC — but not to Blender directly. No `import bpy` found anywhere.

**Skill Relevance**: This confirms a gap that our skill package fills — building.py creates the geometry model but has no Blender export path.

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

## 1.2 GIS-to-Blender 3DEnvironment Automation

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

**Skill Relevance**: The intent aligns directly with our skill package goals — Claude Code generating Blender/IFC automation scripts. Our skills would provide the structured knowledge this repo's approach requires. Relevant skills: `blender-impl-automation`, `blender-impl-mesh`

### Anti-Patterns

- Empty repository published as a "tool" — no implementation behind the README
- No examples, no tests, no code structure

---

## 1.3 AEC Scripts

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

**Skill Relevance**: `blender-impl-mesh`, `blender-syntax-mesh` — CityJSON vertex/face parsing maps directly to `bpy.types.Mesh.from_pydata()`

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

**Skill Relevance**: `blender-impl-mesh` — The vertex-face mesh creation pattern maps directly to Blender's `bpy.types.Mesh.from_pydata(vertices, edges, faces)` pattern. CityJSON to mesh is a reusable workflow.

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

**Skill Relevance**: `blender-syntax-mesh`, `blender-impl-mesh` — Maps to Blender's `bmesh.ops.extrude_face_region()` or `bpy.ops.mesh.extrude_region_move()` patterns. The CurveLoop to extrusion workflow is equivalent to PolyCurve to Extrusion in building.py.

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

**Skill Relevance**: `blender-impl-automation` — GIS data source catalog is reusable for Blender-based GIS workflows.

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

**Skill Relevance**: `blender-core-api` — Maps to Blender's `bpy.ops` context and undo system. The transaction pattern is a best practice for atomic operations.

### Anti-Patterns Found

| Issue | Location | Description |
|-------|----------|-------------|
| Bare except clauses | GIS2BIM script.py:75-94 | Failure handler catches everything silently |
| Hard-coded limits | SCAN2BIM script.py:82 | `max_points=100000` — no configuration |
| Memory concerns | GIS2BIM script.py:1095-1106 | Large bitmap tiles combined without streaming |
| Geometry validation gaps | GIS2BIM script.py:1642 | Distance check but no co-linearity validation |
| IronPython workarounds | GIS2BIM script.py:53-68 | Manual struct construction instead of proper API |

---

## 1.4 Monty IFC Viewer

**Repository:** https://github.com/OpenAEC-Foundation/monty-ifc-viewer
**Purpose:** Web-based IFC model viewer with desktop wrapper (Tauri)
**Tech Stack:** Solid.js, Three.js, web-ifc (v0.0.66), Vite, Tauri v2

### IFC File Loading

The viewer uses **web-ifc** (not IfcOpenShell) for client-side IFC parsing. Two loading methods exist:

1. **File Drop/Input** — drag-and-drop or file picker for `.ifc` and `.ifczip`
2. **URL Loading** — query parameter `?model=<url>` for remote files

**Loading pipeline:**
```
File Input → Uint8Array → web-ifc OpenModel() → Geometry Extraction → Three.js Scene
```

```typescript
// src/actions/load-file.ts
const buffer = await file.arrayBuffer();
const data = new Uint8Array(buffer);
const modelId = openModel(data);
```

### IFC API Initialization (web-ifc)

```typescript
// src/services/ifc-service.ts
export async function initIFC(): Promise<void> {
  ifcApi = new WebIFC.IfcAPI();
  ifcApi.SetWasmPath('/');
  await ifcApi.Init();
}
```

**Core API methods used:**
- `OpenModel(data)` — Opens IFC file from Uint8Array, returns modelId
- `CloseModel(modelId)` — Cleanup
- `GetLine(modelId, expressId)` — Retrieve entity by Express ID
- `GetLineIDsWithType(modelId, type)` — Query all entities of a specific type
- `GetFlatMesh(modelId, expressId)` — Get mesh geometry for element

### Property Extraction Patterns

The viewer traverses these IFC relationship types for property extraction:

- `IFCRELASSOCIATESMATERIAL` — Material assignments
- `IFCRELDEFINESBYTYPE` — Type definitions
- `IFCRELDEFINESBYPROPERTIES` — Property set relationships

```typescript
// src/services/property-service.ts — Material Resolution
const matRels = getLineIDsWithType(modelId, WebIFC.IFCRELASSOCIATESMATERIAL);
for (let i = 0; i < matRels.size(); i++) {
  const relId = matRels.get(i);
  const rel = getLine(modelId, relId);
  if (rel.RelatingMaterial?.value) {
    const mat = getLine(modelId, rel.RelatingMaterial.value);
    materialName = mat.Name?.value;
  }
}
```

**Skill Relevance:** `bonsai-syntax-properties` — The property traversal pattern (IfcRelDefinesByProperties to IfcPropertySet to HasProperties) is identical in IfcOpenShell but uses different API syntax.

### Geometry Extraction

Geometry is extracted only for a defined set of BIM product types:

```typescript
// src/services/geometry-service.ts
const PRODUCT_TYPES = [
  WebIFC.IFCWALL, WebIFC.IFCWALLSTANDARDCASE,
  WebIFC.IFCSLAB, WebIFC.IFCSLABELEMENTEDCASE, WebIFC.IFCSLABSTANDARDCASE,
  WebIFC.IFCCOLUMN, WebIFC.IFCCOLUMNSTANDARDCASE,
  WebIFC.IFCBEAM, WebIFC.IFCBEAMSTANDARDCASE,
  WebIFC.IFCWINDOW, WebIFC.IFCDOOR,
  WebIFC.IFCROOF, WebIFC.IFCSTAIR, WebIFC.IFCSTAIRFLIGHT,
  // ... additional types
];
```

**Vertex extraction pipeline:**
```typescript
// src/services/geometry-service.ts (lines 104-147)
const geometry = getFlatMesh(modelId, expressId);
const placedGeometries = geometry.geometries;

for (let j = 0; j < placedGeometries.size(); j++) {
  const pg = placedGeometries.get(j);
  const geomData = getGeometry(modelId, pg.geometryExpressID);

  // Vertex format: 6 floats per vertex (x, y, z, nx, ny, nz)
  const verts = getVertexArray(geomData.GetVertexData(), geomData.GetVertexDataSize());
  const indices = getIndexArray(geomData.GetIndexData(), geomData.GetIndexDataSize());

  for (let k = 0; k < verts.length; k += 6) {
    const idx = (k / 6) * 3;
    positions[idx] = verts[k];          // x
    positions[idx + 1] = verts[k + 1];  // y
    positions[idx + 2] = verts[k + 2];  // z
    normals[idx] = verts[k + 3];        // normal x
    normals[idx + 1] = verts[k + 4];    // normal y
    normals[idx + 2] = verts[k + 5];    // normal z
  }

  const threeGeom = new THREE.BufferGeometry();
  threeGeom.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  threeGeom.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
  threeGeom.setIndex(new THREE.BufferAttribute(indices, 1));

  // Apply transformation matrix from IFC placement
  const matrix = new THREE.Matrix4();
  matrix.fromArray(pg.flatTransformation);
  mesh.applyMatrix4(matrix);
}
```

**Skill Relevance:** `bonsai-syntax-geometry`, `ifcos-syntax-geometry` — The vertex unpacking pattern (6 floats: position + normal) and matrix transformation application are analogous to IfcOpenShell's `ifcopenshell.geom.create_shape()` output.

### Web-Based Architecture Patterns

| Component | Technology | Pattern |
|-----------|-----------|---------|
| UI Framework | Solid.js | Fine-grained reactivity via signals |
| 3D Rendering | Three.js 0.128.0 | BufferGeometry + MeshLambertMaterial |
| IFC Parsing | web-ifc 0.0.66 | WASM-based client-side parsing |
| Desktop Wrapper | Tauri v2 | Cross-platform native shell |
| Build Tool | Vite + SWC | Fast HMR development |

**State management uses Solid.js signals:**
```typescript
// src/state/ifc-store.ts, viewer-store.ts, selection-store.ts, player-store.ts
export const [currentIndex, setCurrentIndex] = createSignal(-1);
```

**Material classification (fragile):**
```typescript
// src/utils/material-classifier.ts
if (matLower.includes('holz') || matLower.includes('wood') || ...) {
  return 'wood';
}
```

### Anti-Patterns Found

| Anti-Pattern | Location | Description |
|-------------|----------|-------------|
| Silent error swallowing | geometry-service.ts, property-service.ts | `catch (_) {}` blocks hide all errors |
| Non-reactive data mutation | state/ifc-store.ts | `export let elements: IFCElement[] = []` bypasses Solid.js signals |
| Magic opacity threshold | raycaster-service.ts | Hardcoded `opacity > 0.2` for selection filtering |
| Case-sensitive material matching | material-classifier.ts | `matLower.includes('holz')` does not cover all casing variants |
| Animation loop leak risk | useThreeCanvas.ts | `requestAnimationFrame` not guaranteed to be cleaned up |
| O(n) material resolution | property-service.ts | Loops through all relationships per element |

---

## 1.5 INB Template

**Repository:** https://github.com/OpenAEC-Foundation/inb-template
**Purpose:** IFC template and library for the Dutch construction industry (Ifc NL Bouw)
**Version:** Template 0.6, Library 0.1
**Schema:** IFC4 (IFC4x3 ready in development)
**Tool:** IfcOpenShell 0.8.4-alpha250721 + Bonsai 0.8.4-alpha250721

### Repository Structure

```
inb-template/
├── 02_scripts/Python_Scripts/
│   ├── CreateLibrary.py (876 lines)         # Main library generation script
│   ├── CreatePatternCSSFiles.py              # CSS pattern generation
│   ├── Get_IfcPile_Height.py                 # Pile height calculations
│   ├── Number_IfcPile_topleft_to_bottomright.py  # Pile numbering
│   └── IfcOpenHouse.py                       # Example project
├── 03_user example projects/
│   ├── 001_CLT gebouwdeel/
│   ├── 002_Bibliotheek CLT gebouwdeel/
│   ├── 003_Steel Structure/
│   └── 004_Wood Framing Element/
├── 04_development projects/
│   ├── 001_classifications/
│   ├── 002_Property Sets/                    # IN PROGRESS
│   └── 003_kozijnen/                         # IN PROGRESS
├── 05_classifications/
│   ├── NL_SfB_4_cijfers_2005.ifc            # 939 classification items
│   ├── NL_SfB_tabel_1_2019.ifc              # Updated NL-SfB table
│   ├── BB_SfB_4_cijfers.ifc                  # Belgian variant
│   └── STABU-Element (6 Cijfers).ifc         # Deprecated STABU coding
├── INB-Template 0.6.ifc (65,850 lines)       # Main template file
├── INB-Library 0.1.ifc (25,340 lines)        # Reusable library
├── layouts/ (SVG templates)
├── sheets/ (Drawing sheets in SVG)
└── Base_Library.ods                           # Data definition spreadsheet
```

### Property Set Definitions

The template uses **custom EPset_* property sets** (not standard Pset_*):

**EPset_Parametric** — Layer set directionality:
```ifc
#53465=IFCPROPERTYSET('0u$QVS3sLBPRQNW_989oDl',$,'EPset_Parametric',$,(#53466));
#53466=IFCPROPERTYSINGLEVALUE('LayerSetDirection',$,IFCLABEL('AXIS3'),$);
```

**EPset_Annotation** — Drawing and styling:
```ifc
#135424=IFCPROPERTYSET('2aQ87NK896pQ7XxCCAVhrF',$,'EPset_Annotation',$,(#135422,#135423));
#135422=IFCPROPERTYSINGLEVALUE('Symbol',$,IFCLABEL('kol_ond'),$);
#135423=IFCPROPERTYSINGLEVALUE('Classes',$,IFCLABEL('small fill-bg'),$);
```

**EPset_Drawing** — Drawing metadata:
```ifc
#564906=IFCPROPERTYSET('1YyNY7W7v9cuELh8Ol4nxc',$,'EPset_Drawing',$,(#564908...));
#564908=IFCPROPERTYSINGLEVALUE('TargetView',$,IFCLABEL('PLAN_VIEW'),$);
#564909=IFCPROPERTYSINGLEVALUE('Scale',$,IFCLABEL('1/50'),$);
#564910=IFCPROPERTYSINGLEVALUE('HumanScale',$,IFCLABEL('1:50'),$);
#564911=IFCPROPERTYSINGLEVALUE('HasUnderlay',$,IFCBOOLEAN(.T.),$);
#564912=IFCPROPERTYSINGLEVALUE('HasLinework',$,IFCBOOLEAN(.T.),$);
#564913=IFCPROPERTYSINGLEVALUE('HasAnnotation',$,IFCBOOLEAN(.T.),$);
#564915=IFCPROPERTYSINGLEVALUE('Stylesheet',$,IFCTEXT('drawings/assets/default.css'),$);
#564916=IFCPROPERTYSINGLEVALUE('Markers',$,IFCTEXT('drawings/assets/markers.svg'),$);
#564917=IFCPROPERTYSINGLEVALUE('Symbols',$,IFCTEXT('drawings/assets/symbols.svg'),$);
#564918=IFCPROPERTYSINGLEVALUE('Patterns',$,IFCTEXT('drawings/assets/patterns.svg'),$);
```

**Skill Relevance:** `bonsai-syntax-properties` — The EPset_* naming convention is non-standard. Standard IFC uses Pset_* for standardized property sets and custom project-specific property sets. The template demonstrates that Bonsai supports arbitrary property set naming, but interoperability tools may not recognize EPset_* prefixed sets.

### Classification System (NL-SfB)

Four classification systems are provided as separate IFC ProjectLibrary files:

#### NL-SfB (4 digits) — 2005 Version
```ifc
// 05_classifications/NL_SfB_4_cijfers_2005.ifc
#1=IFCPROJECTLIBRARY($,$,'NL/SfB (4 cijfers)',$,$,$,$,$,$);
#2=IFCCLASSIFICATION('BIMLoket','2005','2017-08-31',
    'NL/SfB (4 cijfers)',
    'Kies hieronder een NL/SfB codering...',
    'http://bimloket.nl/NL-SfB',('.'));
#3=IFCRELASSOCIATESCLASSIFICATION('3W_Zw1XTz8hOPhQmQSZp_N',$,$,$,(#1),#2);
```

**Key categories:**
- `0-`: Indirect project provisions (INDIRECTE PROJECTVOORZIENINGEN)
- `1-`: Foundations (FUNDERINGEN) — 11: Ground provisions, 13: Floor on grade, 16: Foundation constructions, 17: Pile foundations
- `2-`: Rough construction (RUWBOUW) — 21: External walls, 22: Internal walls, 23: Floors, 27: Roofs, 28: Stairs
- Total: 939 classification reference items

#### NL-SfB Table 1 — 2019 Update
```ifc
// 05_classifications/NL_SfB_tabel_1_2019.ifc
#1807935=IFCCLASSIFICATION('BIMLoket','december 2019','2023-01-13',
    'NL-SfB tabel 1 Classification', ...);
```

#### STABU Element (6 digits) — Deprecated
```ifc
// 05_classifications/STABU-Element (6 Cijfers).ifc
#2=IFCCLASSIFICATION('STABU','(6 cijfers)','1991-01-01',
    'STABU-Element',
    'Deze 6 cijferige codering wordt niet meer ondersteund...',
    'www.stabu.org',('.'));
```

#### BB/SfB (Belgian variant)
```ifc
// 05_classifications/BB_SfB_4_cijfers.ifc
#2=IFCCLASSIFICATION('Regie der Gebouwen','1990','2017-08-31',
    'BB/SfB (3/4 cijfers)', ...);
```

**Skill Relevance:** `bonsai-impl-classification` — The classification hierarchy uses `IfcClassificationReference` with parent references. Each classification is a self-contained IFC ProjectLibrary that projects import. The dot-separated notation (`'.'`) is configured via the `NotationFacet` parameter of `IfcClassification`.

### Material and Element Type Organization

The template implements sophisticated material layer sets and profile sets:

```python
# 02_scripts/Python_Scripts/CreateLibrary.py — Layer type creation
def create_layer_type(self, ifc_class, name, description, thickness,
                      material_element, material_layerset_name):
    element = ifcopenshell.api.run("root.create_entity", self.file,
                                  ifc_class=ifc_class, name=name)
    rel = ifcopenshell.api.run("material.assign_material", self.file,
                               product=element, type="IfcMaterialLayerSet")
    layer_set = rel.RelatingMaterial
    layer_set.LayerSetName = material_layerset_name

    layer = ifcopenshell.api.run("material.add_layer", self.file,
                                 layer_set=layer_set, material=material_element)
    ifcopenshell.api.run("project.assign_declaration", self.file,
                         definition=element, relating_context=self.library)
```

```python
# Profile type creation (steel sections)
def create_profile_type(self, ifc_class, name, profile, material_element):
    element = ifcopenshell.api.run("root.create_entity", self.file,
                                  ifc_class=ifc_class, name=name)
    rel = ifcopenshell.api.run("material.assign_material", self.file,
                               product=element, type="IfcMaterialProfileSet")
    profile_set = rel.RelatingMaterial

    material_profile = ifcopenshell.api.run(
        "material.add_profile", self.file, profile_set=profile_set,
        material=material_element)
    ifcopenshell.api.run("material.assign_profile", self.file,
                         material_profile=material_profile, profile=profile)
```

**Material examples in INB-Template 0.6.ifc:**
```ifc
#53462=IFCMATERIALLAYERSET((#53464),'gewapend_beton_prefab',$);
#53464=IFCMATERIALLAYER(#1795990,200.,$,$,$,$,$);

#1795705=IFCMATERIAL('naaldhoutvurenCLT',$,'Hout');
#1795706=IFCMATERIALLAYER(#1795705,140.,$,$,$,$,$);
#1795707=IFCMATERIALLAYERSET((#1795706),'naaldhout_vuren_CLT',$);
```

**Element types:** 113 total — 20+ IfcSlabType, 20+ IfcWallType, IfcBeamType, IfcColumnType, plus steel profiles (HEA, HEB, HEM, IPE, AA, HD, DIN, DIE, DIL, DIR, Box, Pipes, T-sections, L-sections).

### ODS-Based Data Definition Pattern

Data-driven template generation reads from LibreOffice Calc (ODS) spreadsheets:

```python
# 02_scripts/Python_Scripts/CreateLibrary.py
from pandas_ods_reader import read_ods

path = "C:/Users/.../INB-Template/02_scripts/Base_Library.ods"
sheet_name = "materials"
library_mat = read_ods(path, sheet_name)

for ind in library_mat.index:
    material_name = library_mat["IfcElementType"][ind]
    material_category = library_mat["Category"][ind]
    rgb = library_mat["RGB"][ind]
    red = float(rgb.split(',')[0]) / 51
    green = float(rgb.split(',')[1]) / 51
    blue = float(rgb.split(',')[2]) / 51
```

**Sheets in Base_Library.ods:** `building_storey`, `grids`, `materials`, walls, floors, profiles.

### Anti-Patterns Found

| Anti-Pattern | Location | Description |
|-------------|----------|-------------|
| Hardcoded Windows paths | CreateLibrary.py | `path = "C:/Users/Gebruiker/Documents/GitHub/..."` not portable |
| bpy dependency | Multiple scripts | Scripts depend on Blender runtime; cannot run standalone |
| Non-standard property set naming | Template IFC | `EPset_*` instead of standard `Pset_*` reduces interoperability |
| No Quantity Take-Off sets | Template IFC | No `Qto_*` property sets for cost estimation |
| Incomplete classification integration | 04_development | Classification libraries exist but are not assigned to elements in the main template |
| RGB scaling factor | CreateLibrary.py | `/ 51` instead of `/ 255`; produces 0-5 range instead of 0-1 |

---

## 1.6 Nextcloud Check-in/Check-out & IfcGit-4-Nextcloud

**Original target:** https://github.com/OpenAEC-Foundation/Nextcloud-Check-in-Check-out-feature (PRIVATE — inaccessible)
**Alternative researched:** https://github.com/OpenAEC-Foundation/ifcgit-4-nextcloud (PUBLIC)
**Purpose:** IFC version control and collaboration platform with file locking
**Tech Stack:** FastAPI, pygit2, IfcOpenShell, PostgreSQL, Redis, Vue.js 3, Three.js

The Nextcloud Check-in/Check-out repository is private and could not be cloned. Its description states: *"With this feature you can check out and check in files or directories like Autodesk Vault. It makes it possible for mech. engineers to work on parts and assemblies or just view them. This repo will mainly be focussed on shared structured Claude Code projects."*

The public **ifcgit-4-nextcloud** repository implements related patterns and was researched as a substitute.

### File Locking Patterns

Database-backed file locking with expiration semantics:

```python
# server/src/projects/models.py (lines 37-45)
class FileLock(Base):
    __tablename__ = "file_locks"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("projects.id"), nullable=False)
    file_path: Mapped[str] = mapped_column(String(500), nullable=False)
    user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    locked_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    expires_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

**Design decisions:**
- Locks stored in PostgreSQL (not filesystem-based)
- Each lock tracks: project, file path, user, timestamp, optional expiration
- Expiration prevents stale locks (deadlock prevention)
- Lock model is defined but lock acquisition/release endpoints are not implemented yet (Phase 2)

**Skill Relevance:** Multi-agent workflow — The file locking model with expiration is directly applicable to multi-agent batch strategies. Agents could acquire locks before writing, with automatic expiry preventing deadlocks.

### Concurrent File Access Patterns

**Upload/commit flow:**
```python
# server/src/git/routes.py (lines 61-97)
@router.post("/{slug}/files", response_model=FileUploadResponse)
async def upload_file(
    slug: str,
    file: UploadFile = File(...),
    path: str = Query("", description="Subdirectory path"),
    branch: str = Query("main"),
    message: str = Query("", description="Commit message"),
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    project = await require_project_access(slug, user, db, required_role="editor")
    file_data = await file.read()
    file_path = f"{path}/{file.filename}".strip("/") if path else file.filename

    commit_hash = git_svc.commit_file(
        repo_path=project.git_repo_path,
        file_path=file_path,
        file_data=file_data,
        message=message,
        author_name=user.username,
        author_email=user.email,
        branch=branch,
    )

    # Queue fragment generation if it's an IFC file
    if file_path.lower().endswith(".ifc"):
        from src.workers.queue import enqueue_fragment_generation
        await enqueue_fragment_generation(str(project.id), file_path, commit_hash)

    return FileUploadResponse(commit_hash=commit_hash, file_path=file_path, message=message)
```

**Concurrency strategy:**
1. **Non-blocking writes** — Each user upload results in an immediate Git commit
2. **Background processing** — Fragment generation queued via Redis (async)
3. **Branch-based isolation** — Users work on separate branches simultaneously
4. **Redis + ARQ job queue** — Background jobs for CPU-intensive IFC tasks

**Worker configuration:**
```python
# server/src/workers/queue.py (lines 158-169)
class WorkerSettings:
    functions = [
        generate_fragment_job,
        run_clash_detection_job,
        run_validation_job,
        run_graph_import_job,
    ]
    redis_settings = parse_redis_url(settings.redis_url)
    max_jobs = 4  # Max 4 concurrent background jobs
    job_timeout = 7200  # 2 hours for large IFC imports
```

**Skill Relevance:** Multi-agent batch strategy — The branch-based isolation + background job queue pattern is directly applicable. Each agent could work on a branch, with a merge step at the end.

### IFC Merge with Semantic Conflict Resolution

```python
# server/src/git/merge_service.py (lines 60-75)
if index.conflicts:
    ifc_conflicts = [
        path for path in _get_conflict_paths(index)
        if path.lower().endswith(".ifc")
    ]

    if ifc_conflicts:
        resolved = _try_ifcmerge(repo, merge_base, target_commit, source_commit, ifc_conflicts)
        if resolved:
            for path in ifc_conflicts:
                if path in resolved:
                    # TODO: Update index with resolved content
                    pass

    remaining_conflicts = [
        path for path in _get_conflict_paths(index)
        if path not in (ifc_conflicts if ifc_conflicts else [])
    ]

    if remaining_conflicts:
        return {"status": "conflict", "conflicts": remaining_conflicts}
```

**Key insight:** For IFC merge conflicts, the system delegates to `ifcmerge` (external Rust binary) for semantic 3-way merge — treating IFC as structured data, not text lines. This integration is incomplete (TODO on line 73).

### IfcOpenShell Property Extraction (Server-Side)

```python
# server/src/fragments/service.py (lines 161-214)
async def _generate_properties(ifc_content: bytes, output_path: str):
    import ifcopenshell

    model = ifcopenshell.open(tmp_path)
    properties = {}

    for element in model.by_type("IfcProduct"):
        express_id = element.id()
        props = {
            "expressID": express_id,
            "GlobalId": element.GlobalId,
            "Class": element.is_a(),
            "Name": getattr(element, "Name", None),
            "ObjectType": getattr(element, "ObjectType", None),
        }

        psets = {}
        if hasattr(element, "IsDefinedBy"):
            for rel in element.IsDefinedBy:
                if rel.is_a("IfcRelDefinesByProperties"):
                    pset = rel.RelatingPropertyDefinition
                    if pset.is_a("IfcPropertySet"):
                        pset_props = {}
                        for prop in pset.HasProperties:
                            if prop.is_a("IfcPropertySingleValue"):
                                val = prop.NominalValue
                                pset_props[prop.Name] = val.wrappedValue if val else None
                        psets[pset.Name] = pset_props

        props["propertySets"] = psets
        properties[str(express_id)] = props
```

**Skill Relevance:** `bonsai-syntax-properties`, `ifcos-syntax-elements` — This is the standard IfcOpenShell pattern for property set extraction: `element.IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties → IfcPropertySingleValue.NominalValue.wrappedValue`.

### WebDAV Integration for CAD Tools

```python
# server/src/webdav/handler.py
class IfcGitDAVProvider(DAVProvider):
    """WebDAV provider backed by IfcGit Git repositories."""

    def get_resource_inst(self, path, environ):
        parts = path.strip("/").split("/")

        if len(parts) == 1:
            slug = parts[0]
            repo_path = os.path.join(settings.repos_dir, f"{slug}.git")
            if os.path.exists(repo_path):
                return ProjectFilesCollection(path, environ, slug, repo_path)

        if len(parts) == 2:
            slug, filename = parts[0], parts[1]
            repo_path = os.path.join(settings.repos_dir, f"{slug}.git")
            content = get_file_content(repo_path, filename, branch="main")
            if content is not None:
                return IfcGitFile(path, environ, content, filename)
```

This enables native file access from Revit, Bonsai/BlenderBIM, and FreeCAD via `dav://` URLs.

### Anti-Patterns Found

| Anti-Pattern | Location | Description |
|-------------|----------|-------------|
| Incomplete ifcmerge integration | merge_service.py:73 | `# TODO` — resolved conflicts not written back to index |
| FileLock model unused | models.py:37-45 | Lock acquisition/release endpoints not implemented |
| No conflict detection at API level | git/routes.py | Simultaneous commits on same branch may fail silently |
| Full file in-memory via WebDAV | webdav/handler.py | Large IFC files cause high memory usage |
| No fragment cache cleanup | projects/service.py | Orphaned cache files accumulate on project deletion |
| No retry logic | fragments/service.py | Failed fragment generation not automatically retried |

---

# Part 2: Claude/Anthropic Skill Development Platform

## 2.1 Official Skill Specification (Agent Skills Open Standard)

Claude Code skills follow the [Agent Skills](https://agentskills.io) open standard, which works across multiple AI tools. Claude Code extends the standard with additional features.

### Directory Structure (Required)

```
skill-name/
├── SKILL.md              # Required - main instructions
├── scripts/              # Optional - executable code
├── references/           # Optional - documentation loaded on demand
└── assets/               # Optional - templates, images, data files
```

The directory name MUST match the `name` field in the YAML frontmatter.

### SKILL.md Format

The file MUST contain YAML frontmatter followed by Markdown content.

#### Required Frontmatter Fields

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | Yes | Max 64 chars. Lowercase letters, numbers, hyphens only. Must not start/end with hyphen. No consecutive hyphens. Must match parent directory name. |
| `description` | Yes (recommended in Claude Code) | Max 1024 chars. Non-empty. Describes what the skill does AND when to use it. No XML tags. |

#### Optional Frontmatter Fields (Agent Skills Standard)

| Field | Constraints |
|-------|-------------|
| `license` | License name or reference to bundled license file |
| `compatibility` | Max 500 chars. Environment requirements (product, packages, network) |
| `metadata` | Arbitrary key-value mapping (e.g., author, version) |
| `allowed-tools` | Space-delimited list of pre-approved tools (experimental) |

#### Claude Code-Specific Frontmatter Fields

These fields extend the open standard and are specific to Claude Code:

| Field | Default | Purpose |
|-------|---------|---------|
| `disable-model-invocation` | `false` | Set `true` to prevent Claude from auto-loading this skill. User must invoke with `/name`. |
| `user-invocable` | `true` | Set `false` to hide from `/` menu. Skill remains available for Claude to auto-invoke. |
| `argument-hint` | - | Hint shown during autocomplete (e.g., `[issue-number]`). |
| `model` | - | Model to use when skill is active. |
| `context` | - | Set to `fork` to run in a forked subagent context. |
| `agent` | `general-purpose` | Which subagent type when `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom. |
| `hooks` | - | Hooks scoped to this skill's lifecycle. |

#### Invocation Control Matrix

| Frontmatter | User can invoke | Claude can invoke | Context behavior |
|-------------|----------------|-------------------|------------------|
| (default) | Yes | Yes | Description always in context; full skill loads when invoked |
| `disable-model-invocation: true` | Yes | No | Description NOT in context; full skill loads when user invokes |
| `user-invocable: false` | No | Yes | Description always in context; full skill loads when invoked |

**Critical distinction**: `user-invocable: false` is a UI setting only. It does NOT prevent Claude from triggering the skill. To prevent model invocation, MUST use `disable-model-invocation: true`.

### Markdown Body Content

No format restrictions from the specification. Recommended sections:
- Step-by-step instructions
- Examples of inputs and outputs
- Common edge cases

#### String Substitutions (Claude Code)

| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All arguments passed when invoking |
| `$ARGUMENTS[N]` or `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing SKILL.md |

#### Dynamic Context Injection (Claude Code)

The `` !`command` `` syntax runs shell commands before skill content is sent to Claude. Output replaces the placeholder.

---

## 2.2 Skill Discovery and Loading Mechanism

### Three-Level Progressive Disclosure

Skills use a three-level loading system to minimize context window consumption:

| Level | When Loaded | Token Cost | Content |
|-------|------------|------------|---------|
| **Level 1: Metadata** | Always (at startup) | ~100 tokens per skill | `name` and `description` from YAML frontmatter |
| **Level 2: Instructions** | When skill is triggered | < 5000 tokens recommended | SKILL.md body |
| **Level 3: Resources** | As needed | Effectively unlimited | Bundled files (scripts, references, assets) |

### Description-Driven Discovery

Claude uses the `description` field to decide when to invoke a skill. The description is injected into the system prompt at session startup. This is the PRIMARY trigger mechanism.

**Description budget**: All skill descriptions share a budget of 2% of the context window, with a fallback of 16,000 characters. Override with `SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable.

**Description writing rules**:
- ALWAYS write in third person ("Processes Excel files", not "I can help you" or "You can use this")
- Include BOTH what the skill does AND when to use it
- Include specific keywords that match user queries
- Be specific, not vague
- Max 1024 characters

### Skill Location Hierarchy

| Priority | Location | Scope |
|----------|----------|-------|
| 1 (highest) | Enterprise managed settings | All org users |
| 2 | Personal `~/.claude/skills/<name>/SKILL.md` | All user's projects |
| 3 | Project `.claude/skills/<name>/SKILL.md` | This project only |
| 4 | Plugin `<plugin>/skills/<name>/SKILL.md` | Where plugin enabled |

When skills share the same name across levels, higher-priority locations win. Plugin skills use `plugin-name:skill-name` namespace (no conflicts).

### Automatic Discovery

- **Nested directories**: Claude Code discovers skills from nested `.claude/skills/` (e.g., `packages/frontend/.claude/skills/`) — supports monorepos.
- **Additional directories**: Skills in `--add-dir` directories are loaded automatically with live change detection.
- **Backward compatibility**: `.claude/commands/` files still work and support the same frontmatter. If a skill and command share a name, the skill takes precedence.

---

## 2.3 Skill Content Best Practices

### Core Principles (Official Anthropic Guidance)

1. **Concise is key**: Context window is a shared resource. Challenge every token: "Does Claude really need this explanation?"
2. **Set appropriate degrees of freedom**: Match specificity to task fragility (high freedom for flexible tasks, low freedom for fragile operations).
3. **Test with all models**: Haiku needs more guidance, Opus needs less. Aim for instructions that work across models.

### SKILL.md Body Recommendations

- Keep SKILL.md body **under 500 lines**
- Move detailed reference material to separate files
- Use **progressive disclosure**: SKILL.md = overview + navigation; details in referenced files
- Keep file references **one level deep** (no nested references)
- Include **table of contents** for reference files over 100 lines
- Use **forward slashes** in file paths (never backslashes)
- Name files descriptively (`form_validation_rules.md`, not `doc2.md`)

### Description Optimization

The description is the MOST critical field for skill activation.

**Effective pattern**:
```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Anti-patterns**:
```yaml
description: Helps with documents       # Too vague
description: Processes data              # Too vague
description: Does stuff with files       # Useless
```

**Naming conventions** (from best practices):
- Prefer gerund form: `processing-pdfs`, `analyzing-spreadsheets`
- Acceptable: noun phrases (`pdf-processing`) or action-oriented (`process-pdfs`)
- Avoid: vague names (`helper`, `utils`), generic (`documents`, `data`), reserved words (`anthropic-*`, `claude-*`)

### Content Patterns

**Template pattern**: Provide output format templates with appropriate strictness.

**Examples pattern**: Include input/output pairs for quality-dependent tasks.

**Conditional workflow pattern**: Guide through decision points.

**Feedback loop pattern**: Run validator → fix errors → repeat. Greatly improves output quality.

**Workflow checklist pattern**: For complex multi-step tasks, provide a checklist Claude can track.

### Anti-Patterns to Avoid

- Windows-style paths (`scripts\helper.py`)
- Offering too many options without a default
- Deeply nested file references
- Time-sensitive information (use "old patterns" section instead)
- Inconsistent terminology
- Assuming tools/packages are installed
- Voodoo constants (undocumented magic numbers)
- Punting errors to Claude instead of handling them

---

## 2.4 Gap Analysis: Our Format vs. Official Specification

### What We Have (WAY_OF_WORK.md / REQUIREMENTS.md)

Our current skill format:
```yaml
---
name: {tech}-{category}-{topic}
description: "Deterministic [description]. Use this skill when Claude needs to [trigger scenario]..."
---
```

Content sections:
1. Quick Reference (critical warnings, decision trees)
2. Essential Patterns (with version annotations)
3. Common Operations (code snippets)
4. Reference Links (to references/ files)

Directory layout:
```
skill-name/
├── SKILL.md              # Main file, < 500 lines
└── references/
    ├── methods.md        # Complete API signatures
    ├── examples.md       # Working code examples
    └── anti-patterns.md  # What NOT to do
```

### Gaps Identified

#### A. Missing Frontmatter Fields

| Field | Status | Action Required |
|-------|--------|----------------|
| `name` | Present | Validate: must match directory name, lowercase+hyphens only, max 64 chars |
| `description` | Present | Verify third-person, max 1024 chars, includes trigger words |
| `license` | Missing | Add `license: MIT` (or project license) |
| `compatibility` | Missing | Add `compatibility: Designed for Claude Code. Requires Python 3.x.` |
| `metadata` | Missing | Consider adding `author`, `version` fields |
| `disable-model-invocation` | Not used | Correct — our skills should auto-invoke |
| `user-invocable` | Not used | Consider for reference-only skills (e.g., core/ category) |
| `allowed-tools` | Not used | Consider for safety on certain skills |

#### B. Naming Convention Mismatch

**Our convention**: `{tech}-{category}-{topic}` (e.g., `blender-syntax-operators`)
**Official convention**: Lowercase, hyphens, max 64 chars, gerund form preferred

**Issue**: Our naming scheme is valid but differs from the recommended gerund pattern. However, our `{tech}-{category}-{topic}` pattern provides better discoverability for domain-specific skills. This is an acceptable deviation.

**Action**: Ensure all names pass validation (no uppercase, no consecutive hyphens, no leading/trailing hyphens, max 64 chars).

#### C. Description Format

**Our pattern**: `"Deterministic [description]. Use this skill when Claude needs to [trigger scenario]..."`
**Official guidance**: Third-person, includes what it does AND when to use it, specific keywords.

**Issue**: Our "Deterministic" prefix wastes description tokens. The word has no trigger value.

**Action**: Remove "Deterministic" prefix. Lead with action verbs in third person. Include domain-specific trigger keywords.

**Before**: `"Deterministic guide for IfcOpenShell API categories. Use this skill when Claude needs to use ifcopenshell.api.run()..."`

**After**: `"Navigates IfcOpenShell API categories (root, spatial, geometry, material, type, pset, classification). Use when writing ifcopenshell.api.run() calls, creating IFC elements, or working with IFC spatial structures."`

#### D. Missing Directory Types

**Our structure**: Only `references/` subdirectory.
**Official structure**: `scripts/`, `references/`, `assets/`

**Action**: Consider adding `scripts/` for validation scripts (e.g., version-checking scripts, IFC schema validators).

#### E. Reference File Organization

**Our approach**: `methods.md`, `examples.md`, `anti-patterns.md`
**Official approach**: Domain-organized files, table of contents for files > 100 lines

**Assessment**: Our structure is valid and well-organized. The three-file pattern (methods, examples, anti-patterns) aligns with the progressive disclosure model. No change needed.

#### F. Content Language Style

**Our standard**: Imperative, deterministic (ALWAYS/NEVER)
**Official guidance**: "Avoid heavy-handed MUSTs and ALWAYS statements. Explain the reasoning so Claude understands the 'why'."

**Issue**: Direct conflict. The official skill-creator skill explicitly recommends AGAINST overuse of ALWAYS/NEVER/MUST. Instead, it recommends explaining the reasoning.

**Recommended action**: Keep deterministic language for truly critical rules (version-specific breaking changes, known API traps). Add brief "why" explanations. Relax ALWAYS/NEVER for preferences vs. requirements.

**Before**: `"ALWAYS use bpy.context.view_layer.objects.active instead of bpy.context.active_object"`
**After**: `"Use bpy.context.view_layer.objects.active instead of bpy.context.active_object — the latter is read-only and raises AttributeError on assignment in Blender 4.x."`

---

## 2.5 Optimization Opportunities

### Description Trigger Optimization

Based on the skill-creator skill's description optimization workflow:

1. Generate 20 trigger eval queries (10 should-trigger, 10 should-not-trigger)
2. Test description activation accuracy
3. Iterate on description wording

**For our domain skills, recommended trigger patterns**:

```yaml
# IfcOpenShell skills
description: "Provides IfcOpenShell API method signatures and usage patterns for ifcopenshell.api.run(), ifcopenshell.file, and ifcopenshell.util. Use when writing Python code that creates, reads, modifies, or validates IFC files, BIM models, or OpenBIM data."

# Blender skills
description: "Guides Blender Python API (bpy) operator registration, property definitions, and panel creation for Blender 3.x and 4.x. Use when writing Blender addons, extensions, operators, or scripts using bpy.types, bpy.props, or bpy.ops."

# Bonsai skills
description: "Covers Bonsai BIM addon patterns including tool.Ifc, spatial structure creation, and property set management. Use when writing Bonsai-specific code, working with IFC in Blender, or extending the Bonsai BIM addon."
```

**Key trigger keywords to include per technology**:
- **Blender**: bpy, addon, extension, operator, modifier, bmesh, mesh, context, panel, property
- **IfcOpenShell**: ifc, ifcopenshell, ifc2x3, ifc4, ifc4.3, bim, openbim, pset, spatial structure
- **Bonsai**: bonsai, blenderbim, tool.Ifc, bim addon, ifc in blender
- **Sverchok**: sverchok, node, parametric, visual programming, data flow

### Progressive Disclosure Optimization

Current SKILL.md structure loads ALL content at once. Optimize by:

1. Keep SKILL.md under 500 lines (current requirement matches)
2. Move method signatures entirely to `references/methods.md`
3. Move examples entirely to `references/examples.md`
4. SKILL.md becomes: quick-reference + decision trees + links to reference files
5. Add table of contents to any reference file > 100 lines

### Evaluation Framework

Adopt the official evaluation pattern from the skill-creator skill:

```json
{
  "skill_name": "ifcos-api-categories",
  "evals": [
    {
      "id": 1,
      "prompt": "Create an IFC wall element using IfcOpenShell",
      "expected_output": "Code using ifcopenshell.api.run('root.create_entity'...)",
      "files": []
    }
  ]
}
```

Save to `evals/evals.json` per skill for regression testing.

### Metadata Fields

Add to all skills:
```yaml
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
  technologies: "blender,ifcopenshell,bonsai,sverchok"
```

### Compatibility Field

Add where relevant:
```yaml
compatibility: Designed for Claude Code. Requires Python 3.x and knowledge of Blender/IfcOpenShell APIs.
```

---

## 2.6 Comparison with ERPNext Skill Package

The ERPNext Skill Package (28 skills) is the reference implementation from which our WAY_OF_WORK.md derives.

### ERPNext Patterns We Should Keep

- **Research-first methodology**: 7-phase approach (proven effective)
- **Category system**: syntax, core, impl, errors, agents
- **Deterministic language**: Critical for preventing API hallucination
- **Version matrix**: Essential for multi-version technologies
- **Anti-pattern documentation**: Primary differentiator from generic AI output

### ERPNext Patterns to Update Based on New Research

| ERPNext Pattern | Official Guidance | Recommended Update |
|----------------|-------------------|-------------------|
| Heavy ALWAYS/NEVER | Explain the "why" | Add reasoning to critical rules |
| No `license` field | Include license | Add `license: MIT` |
| No `metadata` | Include author/version | Add metadata block |
| No `compatibility` | Include when needed | Add for Claude Code targeting |
| No evaluation framework | Build evals first | Add `evals/evals.json` per skill |
| Fixed description pattern | Optimized trigger words | Rewrite descriptions per §2.5 |

---

## 2.7 Recommendations for Our Skill Package

### Immediate Actions (Before Skill Creation)

1. **Update SKILL.md template** to include all official frontmatter fields:
   ```yaml
   ---
   name: {tech}-{category}-{topic}
   description: "[Third-person action verb] [specific capability]. Use when [specific trigger scenarios with domain keywords]."
   license: MIT
   compatibility: Designed for Claude Code. Requires Python 3.x.
   metadata:
     author: OpenAEC-Foundation
     version: "1.0"
   ---
   ```

2. **Rewrite description pattern**: Remove "Deterministic" prefix. Lead with verbs. Include technology-specific keywords. Max 1024 chars.

3. **Add evaluation framework**: Create `evals/evals.json` for each skill with 3+ test scenarios.

4. **Validate names**: Ensure all skill names pass: lowercase, hyphens only, max 64 chars, match directory name, no consecutive hyphens.

### Structural Changes

1. **Add table of contents** to reference files over 100 lines.
2. **Consider `scripts/` directory** for skills that benefit from validation scripts.
3. **Soften language where appropriate**: Use ALWAYS/NEVER only for genuinely critical rules. Add "because [reason]" to each constraint.

### Quality Assurance

1. **Test with multiple models**: At minimum Sonnet and Haiku (Opus has better recall, Haiku needs more guidance).
2. **Run trigger evaluations**: 20 queries per skill (10 should-trigger, 10 should-not-trigger).
3. **Monitor context budget**: With 20+ skills, descriptions may exceed the 16k character budget. Keep descriptions concise.

### Distribution Strategy

- **Claude Code**: Commit `.claude/skills/` to version control (primary distribution)
- **Plugin format**: Package as Claude Code plugin for easy installation
- **Claude.ai**: Provide ZIP download instructions per skill
- **API**: Document skill_id usage for programmatic access

---

## 2.8 Sources (Claude Platform Research)

- [Agent Skills Open Standard Specification](https://agentskills.io/specification)
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [Agent Skills Overview (Anthropic Platform)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Skill Authoring Best Practices (Anthropic Platform)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Anthropic Skills Repository](https://github.com/anthropics/skills)
- [Skill Creator SKILL.md](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [Claude Custom Skills Help Center](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
- [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package)
- [Claude Agent Skills Deep Dive (Lee Han Chung)](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)

---

# Part 3: Cross-Technology Patterns

This section synthesizes patterns observed across all three research fragments: OpenAEC code repositories, IFC/infrastructure projects, and the Claude skill platform.

## 3.1 IFC Property Extraction Pattern

**Observed in:** building.py, Monty IFC Viewer, IfcGit-4-Nextcloud, INB Template

The canonical pattern for extracting property sets from IFC elements is consistent across all repositories:

```
element.IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties → IfcPropertySingleValue.NominalValue.wrappedValue
```

**IfcOpenShell (Python)** — building.py, IfcGit-4-Nextcloud:
```python
for rel in element.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                if prop.is_a("IfcPropertySingleValue"):
                    value = prop.NominalValue.wrappedValue
```

**web-ifc (TypeScript)** — Monty IFC Viewer:
```typescript
const matRels = getLineIDsWithType(modelId, WebIFC.IFCRELDEFINESBYPROPERTIES);
// Same traversal, different API surface
```

**Skill Relevance:** `bonsai-syntax-properties`, `ifcos-syntax-elements`, `ifcos-impl-extraction` — This pattern MUST be documented as the canonical approach. All three skills should reference the same traversal chain.

## 3.2 IFC Creation Pipeline

**Observed in:** building.py (primary), INB Template (library generation)

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

**Skill Relevance:** `ifcos-impl-creation`, `ifcos-syntax-api` — This 9-step pattern is the reference implementation for our IFC creation skills.

## 3.3 Mesh Generation Pipeline

**Observed in:** aec-scripts (CityJSON to Revit), Monty IFC Viewer (IFC to Three.js)

The CityJSON to mesh pipeline from aec-scripts translates directly to Blender:

```
Revit:   XYZ(x,y,z) → Mesh.CreateTriangleMesh(verts, faces) → DirectShape.SetShape([mesh])
Three.js: Float32Array → BufferGeometry.setAttribute('position',...) → THREE.Mesh
Blender: (x,y,z)    → mesh.from_pydata(verts, edges, faces)  → object.data = mesh
```

All three platforms share the same fundamental pattern: vertices + face indices = renderable mesh. The data format differs but the concept is identical.

**Skill Relevance:** `blender-impl-mesh`, `blender-syntax-mesh` — Document the Blender equivalent alongside the Revit/Three.js patterns for users migrating workflows.

## 3.4 Geometry Creation Strategy

**Observed in:** building.py, aec-scripts

Both building.py and aec-scripts use the same core geometry strategy:

| Step | building.py | aec-scripts (Revit) | Blender equivalent |
|------|-------------|--------------------|--------------------|
| 1. Define 2D profile | PolyCurve | CurveLoop | bmesh face from vertices |
| 2. Extrude along vector | Extrusion.by_2d_polycurve_vector() | GeometryCreationUtilities.CreateExtrusionGeometry() | bmesh.ops.extrude_face_region() |
| 3. Assign material | Material object | Revit Material | material slot assignment |
| 4. Place in hierarchy | IFC spatial container | Revit Level/Phase | Blender Collection |

**Skill Relevance:** `blender-impl-mesh`, `blender-syntax-mesh`, `ifcos-syntax-geometry` — The profile-extrude-assign-place workflow is universal across AEC platforms.

## 3.5 Classification Architecture

**Observed in:** INB Template

The Dutch construction ecosystem uses modular classification via separate IFC ProjectLibrary files:

- **NL-SfB (4 digits, 2005)** — 939 items, primary Dutch classification
- **NL-SfB Table 1 (2019)** — Updated version
- **BB/SfB** — Belgian variant
- **STABU Element (6 digits)** — Deprecated

Each classification is a self-contained IFC file using `IfcProjectLibrary` + `IfcClassification` + `IfcClassificationReference` hierarchy. Projects import these libraries and associate elements via `IfcRelAssociatesClassification`.

**Skill Relevance:** `bonsai-impl-classification` — Document both the NL-SfB system (Dutch context) and the general pattern (Uniclass, OmniClass) for international use.

## 3.6 Concurrent File Access Patterns

**Observed in:** IfcGit-4-Nextcloud

Three strategies for concurrent IFC file access:

1. **Branch-based isolation** — Each user/agent works on a separate Git branch, merge at completion
2. **Database-backed file locks** — PostgreSQL FileLock model with expiry timestamps (prevents deadlocks)
3. **Background job queues** — Redis + ARQ for CPU-intensive IFC processing (max 4 concurrent jobs, 2-hour timeout)

**IFC-specific insight:** Semantic 3-way merge via `ifcmerge` (Rust binary) treats IFC as structured data rather than text lines. This is incomplete but represents the correct approach for IFC conflict resolution.

**Skill Relevance:** Multi-agent workflow patterns — The branch isolation + merge pattern is directly applicable to our parallel agent execution strategy described in the masterplan.

## 3.7 Schema Version Convergence

**Observed across:** all IFC-related repositories

All repositories target **IFC4** as the primary schema, with IFC4x3 readiness in development:

| Repository | Schema | Notes |
|-----------|--------|-------|
| building.py | IFC4 (implicit) | Uses run() API which defaults to IFC4 |
| INB Template | IFC4 (explicit) | IFC4x3 ready in development |
| Monty IFC Viewer | Schema-agnostic | web-ifc handles multiple schemas |
| IfcGit-4-Nextcloud | Schema-agnostic | Stores raw IFC files |

**Skill Relevance:** `ifcos-core-schemas`, `ifcos-errors-schema` — Skills must support IFC4 as primary with IFC2x3 backward compatibility and IFC4x3 forward readiness.

## 3.8 Anti-Pattern Summary Across All Repositories

| Anti-Pattern | Occurrences | Repositories | Impact on Skills |
|-------------|-------------|-------------|-----------------|
| Silent exception handling | 4 | building.py, Monty, IfcGit, aec-scripts | `ifcos-errors-schema`: Document proper error handling |
| Mixed IfcOpenShell API styles | 1 | building.py | `ifcos-syntax-api`: Clarify when to use `run()` vs `create_entity()` |
| Hardcoded paths/values | 3 | building.py, INB, aec-scripts | General: Anti-pattern for all skills |
| Non-standard property naming | 1 | INB Template | `bonsai-syntax-properties`: Document EPset_* vs Pset_* |
| Missing type hints | 2 | building.py, aec-scripts | General: Skills should promote typed code |
| Incomplete implementations | 3 | building.py, IfcGit, INB | General: Skills should provide complete patterns |

---

# Summary Table

| Source | Key Findings | Impacted Skills |
|--------|-------------|-----------------|
| **building.py** | IfcOpenShell `run()` API for entity creation; IFC hierarchy (Project→Site→Building→Storey); Profile-based beam geometry via `IfcArbitraryProfileDefWithVoids`; Extrusion-based panels/slabs; Factory method pattern for geometry; No Blender export (gap our skills fill) | `ifcos-syntax-api`, `ifcos-impl-creation`, `ifcos-syntax-geometry`, `bonsai-syntax-spatial`, `bonsai-impl-modeling` |
| **building.py** | `ifcopenshell.geom.create_shape()` for mesh extraction; Property extraction via `IsDefinedBy` traversal; `ifcopenshell.geom.settings()` with `SEW_SHELLS` | `ifcos-syntax-elements`, `ifcos-syntax-geometry`, `ifcos-impl-extraction` |
| **building.py** | Mixed `run()` vs `create_entity()` API usage; Debug prints; Silent exceptions; 18+ TODO items | Anti-pattern documentation for `ifcos-syntax-api` |
| **GIS-to-Blender** | Empty repository — placeholder for LLM-driven Blender automation via Claude Code | Validates our skill package approach; `blender-impl-automation` |
| **aec-scripts** | CityJSON parsing with triangulation; Coordinate transforms (RD→local); 60+ Dutch GIS data sources | `blender-impl-mesh`, `blender-syntax-mesh` |
| **aec-scripts** | DirectShape mesh from vertices+faces (maps to `Mesh.from_pydata()`); Extrusion from CurveLoop profiles | `blender-impl-mesh`, `blender-syntax-mesh` |
| **aec-scripts** | Parametric window frames (kozijn) and staircases (trap) with configurable dimensions; Zero IfcOpenShell/bpy usage | `blender-impl-mesh`, `blender-impl-automation` |
| **Monty IFC Viewer** | Web-based IFC viewer using web-ifc (WASM), not IfcOpenShell. Solid.js + Three.js. Property traversal via IfcRelDefinesByProperties. Vertex format: 6 floats (pos+normal). | `bonsai-syntax-properties`, `bonsai-syntax-geometry`, `ifcos-syntax-geometry` |
| **INB Template** | IFC4 schema. Dutch construction template with 113 element types. Custom EPset_* property sets. NL-SfB (2005/2019), STABU, BB-SfB classification libraries. ODS-driven data definition. IfcMaterialLayerSet + IfcMaterialProfileSet patterns. | `bonsai-syntax-properties`, `bonsai-impl-classification`, `bonsai-impl-modeling` |
| **IfcGit-4-Nextcloud** | Git-backed IFC versioning with FastAPI. FileLock model (DB-backed, with expiry). Branch-based concurrent isolation. Redis job queue. ifcmerge for semantic IFC conflict resolution (incomplete). WebDAV for CAD access. | Multi-agent workflow, `bonsai-syntax-properties`, `ifcos-syntax-elements` |
| **Nextcloud Check-in/out** (private) | Autodesk Vault-like check-in/out. Focus on shared Claude Code projects. Could not be accessed. | Multi-agent workflow, file locking strategy |
| **Claude Skill Specification** | YAML frontmatter format; 3-level progressive disclosure; Description-driven discovery (2% context budget); 4-level location hierarchy; String substitutions and dynamic context injection | All skills (format compliance) |
| **Claude Skill Best Practices** | Under 500 lines; Third-person descriptions; Forward slashes; Explain the "why" behind constraints; Feedback loop pattern; Evaluation framework | All skills (quality assurance) |
| **Gap Analysis** | Missing: `license`, `compatibility`, `metadata` fields; "Deterministic" prefix wastes tokens; ALWAYS/NEVER overuse conflicts with official guidance; Missing `scripts/` directory | All skills (template update) |
| **Cross-Tech: Property Extraction** | Consistent pattern across all IFC repos: IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties | `bonsai-syntax-properties`, `ifcos-syntax-elements`, `ifcos-impl-extraction` |
| **Cross-Tech: Mesh Pipeline** | Universal vertices+faces pattern maps across Revit, Three.js, and Blender (`from_pydata`) | `blender-impl-mesh`, `blender-syntax-mesh` |
| **Cross-Tech: Concurrent Access** | Branch isolation + file locks + background queues for multi-agent IFC workflows | Multi-agent execution strategy |

---

*Document generated: 2026-03-06*
*Consolidation of: ecosystem-claude-platform.md, ecosystem-openaec-code.md, ecosystem-openaec-ifc.md*
