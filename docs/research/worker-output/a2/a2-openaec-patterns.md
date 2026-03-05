## Real-World Usage: OpenAEC Projects

The OpenAEC Foundation maintains several repositories relevant to the Blender-Bonsai-IfcOpenShell-Sverchok skill package. Two repositories are directly referenced: GIS-to-Blender and aec-scripts. A third related repository, building-py, provides additional context for Blender integration within the OpenAEC ecosystem.

### GIS-to-Blender (3D Environment Automation)

**Repository**: https://github.com/OpenAEC-Foundation/GIS-to-Blender_3DEnvironment_Automation
**Purpose**: Prompt LLMs (primarily Claude Code) to build 3D environments in Blender from GIS data.
**Blender Version**: Not yet specified (repository is in early stage).
**Status**: Pre-development. The repository contains only a README and LICENSE (LGPL 3.0) as of February 2026. No Python code exists yet.

#### Key Findings

The repository's README states: *"With this repo you can prompt LLM's to build a 3D environment in Blender. Mainly built for Claude Code."*

This repository is the **direct target use case** for the skill package being developed. It represents the intended consumer of Claude skills that automate Blender via bpy. The fact that it contains no code yet confirms that the skill package MUST provide the foundational bpy patterns that this project will rely on.

#### Architecture Notes

- The repository is designed as an LLM-promptable project, NOT a traditional addon.
- LGPL 3.0 license is consistent with the Blender ecosystem licensing requirements.
- Created January 2026, last pushed February 2026 -- active but early stage.

---

### AEC Scripts

**Repository**: https://github.com/OpenAEC-Foundation/aec-scripts
**Purpose**: Python scripts and Revit addins for AEC (Architecture, Engineering, Construction) workflows.

#### Key Finding: No bpy Usage

The aec-scripts repository contains **zero Blender/bpy code**. All Python scripts are pyRevit pushbutton scripts targeting Autodesk Revit via IronPython. The repository uses:

- `clr` (Common Language Runtime) for .NET interop
- `Autodesk.Revit.DB` for Revit API access
- `pyrevit` framework for script hosting
- `System.Windows.Forms` for UI

The `.pushbutton` directory structure is a pyRevit convention, NOT a Blender addon pattern.

#### Relevant GIS/Mesh Patterns (Transferable to Blender)

Despite being Revit-targeted, the GIS2BIM pushbutton contains **CityJSON parsing and mesh creation patterns** that are directly transferable to Blender/bpy. These patterns represent real-world AEC geometry workflows.

##### Pattern 1: CityJSON Vertex Parsing and Coordinate Transformation

The `cityjson_parser.py` module and `parse_cityjson_lod22_geometry()` function demonstrate how to parse 3D BAG (Dutch building registry) CityJSON data into vertices and faces. This is directly applicable to Blender mesh creation.

```python
# Revit version (from aec-scripts GIS2BIM.pushbutton/cityjson_parser.py)
def transform_vertices(vertices, scale, translate, rd_x, rd_y):
    """Transform CityJSON integer vertices to relative meters."""
    converted = []
    for vx, vy, vz in vertices:
        x = float(vx) * scale[0] + translate[0]
        y = float(vy) * scale[1] + translate[1]
        z = float(vz) * scale[2] + translate[2]
        converted.append((x - rd_x, y - rd_y, z))
    return converted
```

The equivalent Blender pattern ALWAYS uses `bpy.data.meshes.new()` and `from_pydata()`:

```python
# Blender equivalent pattern (Blender 4.x+)
import bpy

def cityjson_to_blender_mesh(name, vertices, faces):
    """Create a Blender mesh from CityJSON vertices and faces.

    vertices: list of (x, y, z) tuples in meters
    faces: list of vertex index tuples (triangulated)
    """
    mesh = bpy.data.meshes.new(name)
    mesh.from_pydata(vertices, [], faces)  # Blender 2.80+
    mesh.update()
    mesh.validate()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj
```

##### Pattern 2: Fan Triangulation for CityJSON Polygons

The `triangulate_polygon()` function in `cityjson_parser.py` implements fan triangulation, which is necessary because CityJSON polygons can have more than 3 vertices.

```python
# From aec-scripts GIS2BIM.pushbutton/cityjson_parser.py
def triangulate_polygon(vertex_indices):
    """Fan triangulation for polygons with >3 vertices."""
    if len(vertex_indices) < 3:
        return []
    if len(vertex_indices) == 3:
        return [tuple(vertex_indices)]
    triangles = []
    v0 = vertex_indices[0]
    for i in range(1, len(vertex_indices) - 1):
        triangles.append((v0, vertex_indices[i], vertex_indices[i + 1]))
    return triangles
```

In Blender, this triangulation is NEVER necessary when using `from_pydata()` because Blender natively handles n-gons. ALWAYS pass the polygon directly:

```python
# Blender 4.x+ - n-gons are natively supported
mesh = bpy.data.meshes.new("building")
# Blender handles quads and n-gons directly - no manual triangulation needed
mesh.from_pydata(vertices, [], polygon_faces)  # polygon_faces can have 3+ vertices
mesh.update()
```

##### Pattern 3: CityJSON LOD Selection Strategy

The GIS2BIM script implements a LOD (Level of Detail) preference hierarchy for 3D BAG data: LOD 2.2 > 1.3 > 1.2. This strategy is directly applicable when importing Dutch 3D building data into Blender.

```python
# From aec-scripts GIS2BIM.pushbutton/script.py (line 1474)
# LOD preference: 2.2 > 1.3 > 1.2
LOD_PREFERENCE = ['2.2', '1.3', '1.2']

# This pattern ALWAYS applies regardless of target application (Revit or Blender)
for geom in obj.get('geometry', []):
    lod_str = str(geom.get('lod', ''))
    if lod_str in LOD_PREFERENCE:
        if best_lod is None or LOD_PREFERENCE.index(lod_str) < LOD_PREFERENCE.index(best_lod):
            best_geom = geom
            best_lod = lod_str
```

##### Pattern 4: Batch Building Import with Error Isolation

The GIS2BIM script processes buildings in a loop with per-building error isolation (try/except around each building). This is a critical pattern for batch GIS imports:

```python
# Revit version (from aec-scripts GIS2BIM.pushbutton/script.py lines 1613-1716)
for building in buildings:
    try:
        vertices = building['vertices']
        polygon_faces = building.get('polygon_faces', [])
        # ... create geometry ...
        count += 1
    except Exception as e:
        skipped += 1  # Continue with next building
```

The Blender equivalent ALWAYS follows the same isolation pattern:

```python
# Blender 4.x+ batch import pattern
import bpy

def import_buildings_batch(buildings, collection_name="3D BAG"):
    """Import multiple CityJSON buildings with per-building error isolation."""
    collection = bpy.data.collections.new(collection_name)
    bpy.context.scene.collection.children.link(collection)

    count = 0
    skipped = 0
    for building in buildings:
        try:
            verts = building['vertices']
            faces = building['polygon_faces']
            obj_id = building.get('id', 'unknown')

            mesh = bpy.data.meshes.new(f"3DBAG_{obj_id}")
            mesh.from_pydata(verts, [], faces)
            mesh.update()

            obj = bpy.data.objects.new(f"3DBAG_{obj_id}", mesh)
            collection.objects.link(obj)
            count += 1
        except Exception as e:
            skipped += 1
            print(f"Skipped {building.get('id', '?')}: {e}")

    return count, skipped
```

#### Addon Structure Notes

The aec-scripts repository uses the **pyRevit pushbutton convention** (`.pushbutton` directories with `script.py` + `icon.png` + `bundle.yaml`). This is NOT applicable to Blender addons, which ALWAYS use either:
- `bl_info` dictionary (Blender 2.80 - 4.1, legacy extensions)
- `blender_manifest.toml` (Blender 4.2+, extension platform)

---

### Related: building-py Library

**Repository**: https://github.com/OpenAEC-Foundation/building-py
**Purpose**: Python library for creating buildings, building systems, and objects with export to multiple programs including Blender, Revit, FreeCAD, and Speckle.

While not one of the two repositories specified in the task, building-py is directly related to the OpenAEC Blender workflow:

- The README explicitly lists **Blender** as a target export platform.
- The document tree (`docs/documenttree.md`) includes a planned `exchange/Blender` module alongside FreeCAD, Revit, and Speckle.
- As of March 2026, the Blender exchange module does **NOT yet exist** (only FreeCAD, Revit, Speckle, IFC, DXF, and Struct4U exchange modules are implemented).
- The library provides a Mesh geometry class that could serve as an intermediary format for Blender export.

This confirms that Blender integration is an **active goal** within the OpenAEC ecosystem but has not yet been implemented. The skill package being developed will directly enable this planned integration.

---

### Cross-Cutting Patterns

#### 1. GIS-to-3D Pipeline Architecture

Both repositories demonstrate a consistent GIS-to-3D pipeline that is directly applicable to Blender:

1. **Data acquisition**: HTTP requests to Dutch GIS APIs (3D BAG, PDOK, WFS/WMS services)
2. **Coordinate transformation**: RD (Rijksdriehoekscoordinaten) to relative project coordinates
3. **Geometry parsing**: CityJSON/GeoJSON to vertices + faces
4. **Mesh creation**: Vertices + faces to application-specific mesh objects
5. **Error-tolerant batch processing**: Per-object try/except with skip-and-continue

The Blender equivalent of step 4 ALWAYS uses `bpy.data.meshes.new()` + `mesh.from_pydata()`.

#### 2. No Existing bpy Code in Either Repository

Neither GIS-to-Blender nor aec-scripts contains any `import bpy` statements or Blender Python API usage. This means:
- The skill package MUST NOT assume any existing patterns to follow from these repos.
- The skill package MUST define the canonical bpy patterns for the OpenAEC ecosystem from scratch.
- The GIS-to-Blender repo is explicitly waiting for LLM-generated Blender code (Claude Code).

#### 3. Licensing Consistency

Both repositories use **LGPL 3.0**, which is compatible with Blender's GPL licensing. Any bpy code generated by the skill package MUST also be GPL/LGPL compatible.

#### 4. Dutch AEC Domain Context

The OpenAEC projects focus on Dutch AEC workflows with specific data sources:
- **3D BAG** (3D Basisregistratie Adressen en Gebouwen): Dutch building registry with LOD 2.2 CityJSON
- **PDOK** (Publieke Dienstverlening Op de Kaart): Dutch geodata platform
- **BGT** (Basisregistratie Grootschalige Topografie): Large-scale topography
- **Kadaster**: Dutch land registry

The skill package SHOULD understand these data sources because Claude Code will be asked to create Blender scripts that import this Dutch GIS data.

#### 5. Claude Code Integration Pattern

Both repos contain `.claude/` directories and `tmpclaude-*` temporary files, confirming active Claude Code usage for development. The GIS-to-Blender repo README explicitly states it is "Mainly built for Claude Code." This confirms the skill package is building for a real, active use case.
