# sverchok-impl-topologic: Anti-Patterns

Confirmed error patterns for TopologicSverchok. Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- https://github.com/wassimj/TopologicSverchok (v0.8.3.0)
- https://topologicpy.readthedocs.io
- vooronderzoek-sverchok.md §11

---

## AP-001: Treating Non-Manifold Faces as Exterior Boundary

**WHY this is wrong**: In a CellComplex, internal faces are shared between two adjacent cells. These non-manifold faces represent the shared walls, floors, and ceilings between rooms. They are NOT part of the building's exterior envelope. Treating all CellComplex faces as exterior surfaces produces incorrect envelope area calculations and wrong energy model inputs.

```python
# WRONG — includes internal shared faces in envelope calculation
from topologicpy.CellComplex import CellComplex
from topologicpy.Face import Face

cc = CellComplex.ByFaces(all_building_faces, tolerance=0.001)
all_faces = CellComplex.Faces(cc)  # Returns ALL faces including internal
total_area = sum(Face.Area(f) for f in all_faces)  # Overcounts by including internal faces
```

```python
# CORRECT — use Decompose to separate external from internal faces
from topologicpy.CellComplex import CellComplex
from topologicpy.Face import Face

cc = CellComplex.ByFaces(all_building_faces, tolerance=0.001)
decomposed = CellComplex.Decompose(cc, tiltAngle=70.0)

# Only external faces for envelope area
external_vertical = decomposed.get('externalVerticalFaces', [])
external_top = decomposed.get('externalTopHorizontalFaces', [])
external_bottom = decomposed.get('externalBottomHorizontalFaces', [])

envelope_faces = external_vertical + external_top + external_bottom
envelope_area = sum(Face.Area(f) for f in envelope_faces)
```

ALWAYS use `CellComplex.Decompose()` or `CellComplexExternalBoundary` to get the outer shell before computing envelope metrics. NEVER use `CellComplex.Faces()` directly for envelope analysis.

---

## AP-002: Passing Raw Blender Coordinates to Topologic API

**WHY this is wrong**: Topologic nodes and the topologicpy API operate on Topologic topology objects (`topologic.Vertex`, `topologic.Edge`, etc.), not on raw Python tuples or Blender vertex coordinates. Passing a Python tuple `(x, y, z)` where a `topologic.Vertex` is expected causes a TypeError or produces None silently.

```python
# WRONG — passing raw coordinates to topology functions
from topologicpy.Cell import Cell

raw_verts = [(0,0,0), (1,0,0), (1,1,0), (0,1,0)]  # Blender-style tuples
# This will fail — Cell.ByFaces expects topologic.Face objects, not coordinate tuples
cell = Cell.ByFaces(raw_verts)  # TypeError or None
```

```python
# CORRECT — convert to Topologic objects first
from topologicpy.Vertex import Vertex
from topologicpy.Face import Face
from topologicpy.Cell import Cell
from topologicpy.Topology import Topology

# Option 1: use ByGeometry for batch conversion
topology = Topology.ByGeometry(
    vertices=[(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,1), (1,0,1), (1,1,1), (0,1,1)],
    faces=[[0,1,2,3], [4,5,6,7], [0,1,5,4], [1,2,6,5], [2,3,7,6], [3,0,4,7]]
)

# Option 2: create individual Topologic Vertex objects
v0 = Vertex.ByCoordinates(0, 0, 0)
v1 = Vertex.ByCoordinates(1, 0, 0)
face = Face.ByVertices([v0, v1, ...])
```

In Sverchok node trees, ALWAYS connect the `TopologyByGeometry` node between Object In data and any TopologicSverchok node. NEVER pipe raw vertex sockets directly to topology class nodes.

---

## AP-003: Using CellComplexByFaces on Non-Planar Geometry

**WHY this is wrong**: TopologicSverchok requires planar faces for CellComplex construction. Non-planar (warped) faces cannot be properly integrated into the non-manifold topology structure and will cause `CellComplexByFaces` to return None or produce an incomplete topology with missing cells.

```python
# WRONG — non-planar face (warped quad) as input
from topologicpy.Vertex import Vertex
from topologicpy.Face import Face
from topologicpy.CellComplex import CellComplex

v0 = Vertex.ByCoordinates(0, 0, 0)
v1 = Vertex.ByCoordinates(1, 0, 0)
v2 = Vertex.ByCoordinates(1, 1, 0.5)  # z=0.5 makes this non-planar
v3 = Vertex.ByCoordinates(0, 1, 0)

non_planar_face = Face.ByVertices([v0, v1, v2, v3])
cc = CellComplex.ByFaces([non_planar_face, ...])  # Returns None or broken CellComplex
```

```python
# CORRECT — verify planarity before building CellComplex
from topologicpy.Topology import Topology
from topologicpy.Face import Face
from topologicpy.CellComplex import CellComplex

faces = [...]  # Input faces

# Check planarity
planar_faces = []
for f in faces:
    if Topology.IsPlanar(f, tolerance=0.001):
        planar_faces.append(f)
    else:
        # Triangulate non-planar face into planar triangles
        triangles = Face.Triangulate(f, mode=0)
        planar_faces.extend(triangles)

cc = CellComplex.ByFaces(planar_faces, tolerance=0.001)
```

ALWAYS verify planarity with `Topology.IsPlanar()` before building CellComplex. ALWAYS triangulate non-planar quads/polygons with `Face.Triangulate()` before passing to `CellComplexByFaces`. NEVER assume imported geometry is planar.

---

## AP-004: Using Optional Integration Nodes Without Installing Dependencies

**WHY this is wrong**: IFC nodes require ifcopenshell, Energy nodes require OpenStudio and honeybee-energy, Speckle nodes require specklepy, Neo4j nodes require py2neo, and DGL nodes require DGL and PyTorch. Using these nodes without installing the required packages causes an import error at node registration time or a runtime AttributeError when the node processes data. The error may appear as a generic "node not found" or a Python traceback, not a clear dependency message.

```python
# WRONG — using EnergyModelByTopology without OpenStudio installed
# The node appears in the list but will fail when it processes:
# ImportError: No module named 'openstudio'
```

```python
# CORRECT — use the InstallDependencies node first
# In Sverchok node editor:
# 1. Add node: InstallDependencies
# 2. Check which optional packages are available
# 3. Install required packages via the node's UI button
# 4. Restart Blender after installation

# Alternatively, test in Python console before building node tree:
try:
    import openstudio
    print("OpenStudio available — Energy nodes will work")
except ImportError:
    print("OpenStudio not installed — use: pip install openstudio")

try:
    import ifcopenshell
    print("ifcopenshell available — IFC nodes will work")
except ImportError:
    print("ifcopenshell not installed — install via BlenderBIM or pip")
```

ALWAYS use the `InstallDependencies` node when first setting up TopologicSverchok. NEVER add Energy, IFC, Speckle, or DGL nodes to a production node tree without verifying the required packages are installed.

---

## AP-005: Wrong Tolerance Values for CellComplex Construction

**WHY this is wrong**: If the tolerance is too small (e.g., 0.0000001), faces that should share edges or vertices will not be recognized as connected, resulting in a CellComplex with too many cells or missing connections. If the tolerance is too large (e.g., 0.1), distinct faces may be incorrectly merged, losing room boundaries. BIM geometry is typically modeled in meters with millimeter precision.

```python
# WRONG — tolerance too small for BIM geometry in meters
from topologicpy.CellComplex import CellComplex
cc = CellComplex.ByFaces(faces, tolerance=0.0000001)  # Gaps not bridged
# Result: walls don't connect, cells not formed

# WRONG — tolerance too large
cc = CellComplex.ByFaces(faces, tolerance=0.5)  # 50cm tolerance
# Result: thin walls (0.2m thick) merged incorrectly
```

```python
# CORRECT — use appropriate tolerance for the model's unit and precision
# BIM models in meters: 1mm = 0.001m
cc = CellComplex.ByFaces(faces, tolerance=0.001)  # 1mm — standard BIM tolerance

# BIM models in millimeters: 1mm tolerance
cc = CellComplex.ByFaces(faces, tolerance=1.0)  # 1mm unit in mm model

# Check model units before setting tolerance
# In Blender: Scene Properties > Units > Unit Scale
```

ALWAYS match tolerance to the model's unit system. Use `tolerance=0.001` for meter-scale models (1mm precision). Use `tolerance=0.01` for centimeter-scale models (1cm precision). NEVER use the default `0.0001` for IFC/BIM models without verifying unit scale.

---

## AP-006: Modifying Topologic Objects In-Place

**WHY this is wrong**: Topologic topology objects are immutable — methods that appear to "modify" them actually return new topology objects. Assigning the result of a transformation to the same variable name without using the returned value results in the original unmodified topology being used downstream.

```python
# WRONG — ignoring the returned modified topology
from topologicpy.Topology import Topology

my_cell = Cell.Box(width=5.0, length=4.0, height=3.0)
Topology.Translate(my_cell, x=10.0, y=0.0, z=0.0)  # Returns new topology — IGNORED
# my_cell is still at the original position (0, 0, 0)
```

```python
# CORRECT — always capture the returned topology
from topologicpy.Topology import Topology

my_cell = Cell.Box(width=5.0, length=4.0, height=3.0)
my_cell_moved = Topology.Translate(my_cell, x=10.0, y=0.0, z=0.0)  # Capture result
# my_cell_moved is at (10, 0, 0)
# my_cell is unchanged at (0, 0, 0)
```

ALWAYS assign the return value of Topologic transformation methods (`Translate`, `Rotate`, `Scale`, `Place`, `Transform`, `Boolean`, etc.) to a new variable. NEVER assume Topologic methods modify objects in-place.

---

## AP-007: Using GraphByTopology on a Cluster Instead of CellComplex

**WHY this is wrong**: `GraphByTopology` with `direct=True` builds a room adjacency graph by finding Cells that share Faces. A Cluster of disconnected Cells has no shared Faces — each Cell is spatially isolated. The resulting graph will have vertices (one per cell) but no edges, so no adjacency information is available.

```python
# WRONG — input is a Cluster of separate, non-adjacent cells
from topologicpy.Cluster import Cluster
from topologicpy.Graph import Graph

cell_a = Cell.Box(origin=Vertex.ByCoordinates(0, 0, 0), width=3, length=4, height=3)
cell_b = Cell.Box(origin=Vertex.ByCoordinates(10, 0, 0), width=3, length=4, height=3)  # 10m gap
cluster = Cluster.ByTopologies([cell_a, cell_b])

graph = Graph.ByTopology(cluster, direct=True)
# Graph has 2 vertices but 0 edges — cells are not adjacent in topology
```

```python
# CORRECT — merge cells into a CellComplex using shared face geometry
from topologicpy.CellComplex import CellComplex
from topologicpy.Graph import Graph

# Cells must be topologically adjacent (sharing faces) before CellComplex construction
# Create wall faces between cells and build CellComplex from all faces:
all_faces = []
all_faces.extend(Cell.Faces(cell_a))
all_faces.extend(Cell.Faces(cell_b))
all_faces.extend(shared_wall_faces)  # Faces at the boundary between cells

cc = CellComplex.ByFaces(all_faces, tolerance=0.001)
graph = Graph.ByTopology(cc, direct=True)
# Now graph correctly shows cell_a and cell_b as adjacent
```

ALWAYS build a CellComplex from face geometry (not from pre-built isolated cells) to ensure shared boundaries are recognized. NEVER pass a Cluster to `GraphByTopology` and expect adjacency edges between cells.

---

## AP-008: Forgetting to Export Topologic Results for Blender Visualization

**WHY this is wrong**: Topologic topology objects are Python objects managed by the topologicpy library — they are invisible in Blender's viewport. Without converting back to Blender geometry, all the topology analysis results remain inaccessible to Blender's rendering, viewport display, and other Sverchok nodes.

```python
# WRONG — Topologic topology is never converted back to Blender geometry
# The node tree ends at a Topology node — nothing appears in viewport
[CellComplexByFaces] -> [GraphByTopology] -> [GraphShortestPath]
# Result (Wire) is a Topologic Wire object — invisible in viewport
```

```python
# CORRECT — convert Topologic output back to Blender geometry for visualization
[GraphShortestPath]
  OUT: path (Topologic Wire)
  ▼
[TopologyBlenderGeometry]
  IN: topology = path Wire
  OUT: vertices (SvVerticesSocket)
  OUT: edges (SvStringsSocket)
  ▼
[Viewer Draw Mk4]
  IN: vertices, edges
  -> Visible in viewport as path line

# Or use TopologyGeometry node which outputs Sverchok-compatible vertex/edge/face lists
```

ALWAYS add `TopologyBlenderGeometry` (or `TopologyGeometry`) at the end of any Topologic analysis chain when visualization in Blender's viewport is needed. NEVER assume Topologic objects render automatically in Blender.

---

## AP-009: Using Wrong Node for Blender-to-Topologic Conversion

**WHY this is wrong**: TopologicSverchok has two similar-looking nodes: `TopologyByGeometry` converts Blender mesh data (vertex list + edge/face index lists) into a Topologic Topology. `TopologyBlenderGeometry` converts in the OPPOSITE direction (Topologic to Blender). Using the wrong direction node produces either a TypeError (wrong input types) or a None output with no error message.

```python
# WRONG node direction
[Object In (Blender mesh)] -> [TopologyBlenderGeometry]
# TopologyBlenderGeometry expects a Topologic Topology as input, not Blender mesh data
# Result: None or TypeError

# WRONG direction (reverse)
[CellComplexByFaces (Topologic)] -> [TopologyByGeometry]
# TopologyByGeometry expects vertex/edge/face index lists, not Topologic objects
# Result: None or TypeError
```

```python
# CORRECT — Blender to Topologic
[Object In] ──vertices/edges/faces──► [TopologyByGeometry] ──► Topologic Topology

# CORRECT — Topologic to Blender
[CellComplexByFaces] ──cellcomplex──► [TopologyBlenderGeometry] ──► vertices/edges/faces ──► [Viewer Draw]
```

ALWAYS use `TopologyByGeometry` for Blender-to-Topologic conversion. ALWAYS use `TopologyBlenderGeometry` for Topologic-to-Blender conversion. The naming is directional: "ByGeometry" = constructed from Blender geometry; "BlenderGeometry" = returns Blender geometry.

---

## AP-010: Using CellComplexByFaces with Inconsistent Face Normals

**WHY this is wrong**: CellComplex construction relies on face normal directions to determine which side of a face is "inside" a cell versus "outside". Inconsistent normals (some faces pointing inward, others outward for the same room) cause `CellComplexByFaces` to misidentify which cells are formed, produce phantom cells, or return None.

```python
# WRONG — mixing inward and outward normals in face list
faces = [
    face_north_wall_normal_out,   # normal points north (outward) — correct
    face_south_wall_normal_in,    # normal points north (inward) — inconsistent
    floor_face_normal_up,         # normal points up (outward for floor) — correct
    roof_face_normal_up,          # normal points up — incorrect for roof (should point up/out = up is correct for roof)
]
cc = CellComplex.ByFaces(faces)  # May produce None or wrong cell count
```

```python
# CORRECT — ensure all face normals point outward from cell interior
from topologicpy.Face import Face
from topologicpy.Topology import Topology

def normalize_face_normals_outward(faces, interior_point):
    """Flip any face whose normal points toward the interior point."""
    corrected = []
    for face in faces:
        normal = Face.Normal(face)
        face_center = Face.InternalVertex(face)

        # Vector from face center to interior point
        from topologicpy.Vertex import Vertex
        dx = Vertex.X(interior_point) - Vertex.X(face_center)
        dy = Vertex.Y(interior_point) - Vertex.Y(face_center)
        dz = Vertex.Z(interior_point) - Vertex.Z(face_center)

        # Dot product: if positive, normal points toward interior — flip it
        dot = normal[0]*dx + normal[1]*dy + normal[2]*dz
        if dot > 0:
            face = Topology.Rotate(face, direction=[0,0,1], degree=180)  # Invert normal
        corrected.append(face)
    return corrected
```

ALWAYS ensure consistent face normal orientation before calling `CellComplexByFaces`. For Blender geometry, use the Recalculate Normals operator (Mesh > Normals > Recalculate Outside) before exporting to Topologic. NEVER assume imported geometry has consistent normals.
