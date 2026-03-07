# sverchok-impl-topologic: API Method Reference

Sources:
- https://github.com/wassimj/TopologicSverchok (v0.8.3.0)
- https://topologicpy.readthedocs.io
- https://github.com/wassimj/topologicpy

---

## Vertex Class — topologicpy.Vertex

All methods are static class methods called as `Vertex.MethodName(...)`.

```python
Vertex.ByCoordinates(x, y, z)
# Returns: topologic.Vertex
# Create a Vertex at given coordinates.

Vertex.Coordinates(vertex, outputType='xyz', mantissa=6)
# Returns: list [x, y, z] or dict
# Get x, y, z coordinates. outputType: 'xyz', 'x', 'y', 'z', 'matrix'

Vertex.X(vertex, mantissa=6)
Vertex.Y(vertex, mantissa=6)
Vertex.Z(vertex, mantissa=6)
# Returns: float — individual coordinate values

Vertex.Distance(vertex, topology, includeCentroid=True, mantissa=6)
# Returns: float — distance to nearest point on topology

Vertex.EnclosingCells(vertex, topology, exclusive=False, mantissa=6, tolerance=0.0001)
# Returns: list[Cell] — Cells that contain this vertex

Vertex.NearestVertex(vertex, topology, useKDTree=True, mantissa=6)
# Returns: Vertex — nearest vertex in topology

Vertex.IsCoincident(vertexA, vertexB, tolerance=0.0001, silent=False)
# Returns: bool

Vertex.IsInternal(vertex, topology, tolerance=0.0001, silent=False)
# Returns: bool — True if vertex is strictly inside topology

Vertex.Project(vertex, face, direction=None, mantissa=6)
# Returns: Vertex — vertex projected onto face along direction

Vertex.AreCollinear(vertices, mantissa=6, tolerance=0.0001)
# Returns: bool

Vertex.AreCoplanar(vertices, mantissa=6, tolerance=0.0001, silent=False)
# Returns: bool

Vertex.Centroid(vertices, mantissa=6)
# Returns: Vertex — centroid of vertex list

Vertex.Normal(vertices, mantissa=6, tolerance=0.0001, silent=False)
# Returns: list [nx, ny, nz] — normal vector of vertex plane

Vertex.Degree(vertex, hostTopology, topologyType)
# Returns: int — number of connections to adjacent topologies of given type

Vertex.Fuse(vertices, mantissa=6, tolerance=0.0001)
# Returns: list[Vertex] — deduplicated vertices within tolerance
```

---

## Edge Class — topologicpy.Edge

```python
Edge.ByStartVertexEndVertex(vertexA, vertexB, tolerance=0.0001, silent=False)
# Returns: topologic.Edge

Edge.ByVertices(*vertices, tolerance=0.0001, silent=False)
# Returns: topologic.Edge — edge between first and last vertex

Edge.StartVertex(edge, silent=False)
Edge.EndVertex(edge, silent=False)
# Returns: Vertex

Edge.Vertices(edge, silent=False)
# Returns: list[Vertex] — [startVertex, endVertex]

Edge.Length(edge, mantissa=6)
# Returns: float

Edge.Direction(edge, mantissa=6)
# Returns: list [dx, dy, dz] — unit direction vector

Edge.Angle(edgeA, edgeB, mantissa=6, bracket=False)
# Returns: float — angle in degrees

Edge.IsCollinear(edgeA, edgeB, mantissa=6, tolerance=0.0001)
# Returns: bool

Edge.IsParallel(edgeA, edgeB, mantissa=6, tolerance=0.0001)
# Returns: bool

Edge.VertexByDistance(edge, distance=0.0, origin=None, mantissa=6, tolerance=0.0001)
# Returns: Vertex — point at given distance along edge from origin

Edge.VertexByParameter(edge, u=0.0)
# Returns: Vertex — point at parameter u (0.0=start, 1.0=end)

Edge.ParameterAtVertex(edge, vertex, mantissa=6, silent=False)
# Returns: float — parameter u at vertex location

Edge.Extend(edge, distance=1.0, bothSides=True, reverse=False, tolerance=0.0001, silent=False)
# Returns: Edge — edge extended by distance

Edge.Trim(edge, distance=0.0, bothSides=True, reverse=False, tolerance=0.0001, silent=False)
# Returns: Edge — edge trimmed by distance

Edge.NormalEdge(edge, length=1.0, u=0.5, angle=0.0, tolerance=0.0001, silent=False)
# Returns: Edge — edge perpendicular to input edge at parameter u

Edge.Line(origin=None, length=1, direction=[1,0,0], placement='center', tolerance=0.0001)
# Returns: Edge — straight line primitive
```

---

## Wire Class — topologicpy.Wire

```python
Wire.ByEdges(edges, orient=False, tolerance=0.0001, silent=False)
# Returns: topologic.Wire

Wire.ByVertices(vertices, close=False, tolerance=0.0001, silent=False)
# Returns: topologic.Wire — connect vertices sequentially

Wire.Circle(origin=None, radius=0.5, sides=16, spokes=0, direction=[0,0,1], placement='center')
# Returns: Wire — circular wire

Wire.Rectangle(origin=None, width=1.0, length=1.0, diagonals=False, direction=[0,0,1], placement='center')
# Returns: Wire — rectangular wire

Wire.Ellipse(origin=None, inputMode=1, width=2.0, length=1.0, sides=32, direction=[0,0,1], placement='center')
# Returns: Wire

Wire.Star(origin=None, radiusA=0.5, radiusB=0.2, rays=8, direction=[0,0,1], placement='center')
# Returns: Wire — star-shaped wire

Wire.IsClosed(wire)
# Returns: bool

Wire.IsManifold(wire, silent=False)
# Returns: bool

Wire.Length(wire, mantissa=6)
# Returns: float — total length of all edges

Wire.Vertices(wire)
# Returns: list[Vertex]

Wire.Edges(wire)
# Returns: list[Edge]

Wire.Normal(wire, outputType='xyz', mantissa=6)
# Returns: list [nx, ny, nz]

Wire.Project(wire, face, direction=None, mantissa=6, tolerance=0.0001)
# Returns: Wire — wire projected onto face

Wire.ByOffset(wire, offset=1.0, offsetKey='offset', stepOffsetA=0.0, stepOffsetB=0.0)
# Returns: Wire — offset wire (inward negative, outward positive)

Wire.Fillet(wire, radius=0.0, sides=16, radiusKey='radius', tolerance=0.0001)
# Returns: Wire — filleted wire corners

Wire.Simplify(wire, method='douglas-peucker', tolerance=0.0001, silent=False)
# Returns: Wire — simplified wire with fewer vertices

Wire.ConvexHull(topology, mantissa=6, tolerance=0.0001)
# Returns: Wire — convex hull of topology

Wire.IsSimilar(wireA, wireB, tolerance=0.0001)
# Returns: bool

Wire.WireSplit(wire, vertex)
# Returns: list[Wire] — wire split at vertex
```

---

## Face Class — topologicpy.Face

```python
Face.ByWire(wire, tolerance=0.0001, silent=False)
# Returns: topologic.Face — face from closed wire boundary

Face.ByWires(externalBoundary, internalBoundaries=[], tolerance=0.0001)
# Returns: Face — face with holes

Face.ByEdges(edges, tolerance=0.0001, silent=False)
# Returns: Face

Face.ByVertices(vertices, tolerance=0.0001, silent=False)
# Returns: Face

Face.AddInternalBoundaries(face, wires)
# Returns: Face — face with added internal boundary wires (holes)

Face.ExternalBoundary(face, tolerance=0.0001)
# Returns: Wire — outer perimeter

Face.InternalBoundaries(face)
# Returns: list[Wire] — holes

Face.Area(face, mantissa=6)
# Returns: float

Face.Normal(face, outputType='xyz', mantissa=6)
# Returns: list [nx, ny, nz] — face normal vector

Face.Angle(faceA, faceB, mantissa=6)
# Returns: float — angle between face normals in degrees

Face.FacingToward(face, direction=[0,0,-1], asym=False, tolerance=0.0001)
# Returns: bool — True if face normal points toward direction

Face.CompassAngle(face, north=None, mantissa=6)
# Returns: float — horizontal orientation angle from north (0-360)

Face.IsCoplanar(faceA, faceB, mantissa=6, tolerance=0.0001)
# Returns: bool

Face.IsConvex(face, mantissa=6, silent=False)
# Returns: bool

Face.IsInside(face, vertex, tolerance=0.0001)
# Returns: bool — True if vertex is inside face boundary

Face.InternalVertex(face, tolerance=0.0001)
# Returns: Vertex — guaranteed interior point

Face.VertexByParameters(face, u=0.5, v=0.5)
# Returns: Vertex — point at parametric coordinates

Face.TrimByWire(face, wire, reverse=False)
# Returns: Face — face trimmed by wire

Face.Flatten(face, origin=None, direction=None)
# Returns: Face — face projected to XY plane

Face.Triangulate(face, mode=0, meshSize=None)
# Returns: list[Face] — triangulated sub-faces

Face.MedialAxis(face, resolution=0)
# Returns: Wire — centerline skeleton

Face.Isovist(face, vertex, obstacles=[])
# Returns: Face — visible area polygon from viewpoint

Face.Rectangle(origin=None, width=1.0, length=1.0, direction=[0,0,1], placement='center')
Face.Circle(origin=None, radius=0.5, sides=16, direction=[0,0,1], placement='center')
Face.Square(origin=None, size=1.0, direction=[0,0,1], placement='center')
# Primitive face creation methods

Face.SharedEdges(faceA, faceB)
# Returns: list[Edge] — edges shared between two faces
```

---

## Shell Class — topologicpy.Shell

```python
Shell.ByFaces(faces, tolerance=0.0001, silent=False)
# Returns: topologic.Shell — connected shell from faces

Shell.ByFacesCluster(cluster, tolerance=0.0001)
# Returns: Shell

Shell.ByWires(wires, triangulate=True, tolerance=0.0001)
# Returns: Shell — lofted shell through wire sequence

Shell.IsClosed(shell)
# Returns: bool — True if shell has no boundary edges

Shell.ExternalBoundary(shell)
# Returns: Wire — outer boundary (if not closed)

Shell.InternalBoundaries(shell)
# Returns: list[Wire]

Shell.Faces(shell)
# Returns: list[Face]

Shell.Edges(shell)
# Returns: list[Edge]

Shell.Vertices(shell)
# Returns: list[Vertex]

Shell.Roof(shell, angle=45.0, epsilon=0.01, tolerance=0.0001)
# Returns: Shell — hipped roof via straight skeleton

Shell.Skeleton(shell, boundary=True, tolerance=0.001)
# Returns: Wire — straight skeleton of shell boundary

Shell.SelfMerge(shell, angTolerance=0.1, tolerance=0.0001)
# Returns: Shell — merged coplanar faces

Shell.RemoveCollinearEdges(shell, angTolerance=0.1, tolerance=0.0001)
# Returns: Shell

Shell.Voronoi(shell, vertices=None, tolerance=0.0001)
# Returns: Shell — Voronoi-partitioned shell

Shell.Pie(shell, innerRadius=0.0, outerRadius=1.0, sides=32, rings=1)
# Returns: Shell — pie-shaped shell
```

---

## Cell Class — topologicpy.Cell

```python
Cell.ByFaces(faces, planarize=False, tolerance=0.0001, silent=False)
# Returns: topologic.Cell — closed cell from face list

Cell.ByShell(shell, planarize=False, tolerance=0.0001, silent=False)
# Returns: Cell

Cell.ByThickenedFace(face, thickness=1.0, bothSides=False, vSides=1, reverse=False, tolerance=0.0001)
# Returns: Cell — extruded face

Cell.ByWires(wires, close=True, triangulate=True, tolerance=0.0001)
# Returns: Cell — lofted cell through wire sequence

Cell.Volume(cell, mantissa=6)
# Returns: float

Cell.SurfaceArea(cell, mantissa=6)
# Returns: float

Cell.Compactness(cell, reference='sphere', mantissa=6)
# Returns: float — 0.0 to 1.0 (1.0 = perfect sphere or cube)

Cell.ExternalBoundary(cell, tolerance=0.0001, silent=False)
# Returns: Shell — outer boundary shell

Cell.InternalBoundaries(cell)
# Returns: list[Shell] — void/hole shells

Cell.InternalVertex(cell, tolerance=0.0001, silent=False)
# Returns: Vertex — guaranteed interior point

Cell.ContainmentStatus(cell, vertex, tolerance=0.0001)
# Returns: int — 0=inside, 1=on boundary, 2=outside

Cell.IsInside(cell, vertex, tolerance=0.0001)
# Returns: bool

Cell.Faces(cell)
Cell.Edges(cell)
Cell.Vertices(cell)
Cell.Wires(cell)
Cell.Shells(cell)
# Returns: list of respective topology type

Cell.Decompose(cell, tiltAngle=70.0, tolerance=0.0001)
# Returns: dict — keys: 'verticalFaces', 'horizontalFaces', 'inclinedFaces', 'internalHorizontalFaces'

Cell.Sets(cells, tolerance=0.0001)
# Returns: list — cells classified by enclosure status

# Primitive creation:
Cell.Box(origin=None, width=1.0, length=1.0, height=1.0, uSides=1, vSides=1, wSides=1, direction=[0,0,1], placement='center', tolerance=0.0001)
Cell.Prism(origin=None, width=1.0, length=1.0, height=1.0, uSides=1, vSides=1, wSides=1, direction=[0,0,1], placement='center')
Cell.Cylinder(origin=None, radius=0.5, height=1.0, uSides=16, vSides=1, direction=[0,0,1], placement='center')
Cell.Cone(origin=None, baseRadius=0.5, topRadius=0.0, height=1.0, sides=16, direction=[0,0,1], placement='center')
Cell.Sphere(origin=None, radius=0.5, uSides=16, vSides=8, direction=[0,0,1], placement='center')
Cell.Torus(origin=None, majorRadius=1.0, minorRadius=0.25, uSides=16, vSides=8, direction=[0,0,1], placement='center')
Cell.Pipe(edge, profiles=None, radius=0.25, sides=8, startOffset=0.0, endOffset=0.0, endcapA=False, endcapB=False)
Cell.Roof(face, angle=45.0, epsilon=0.01, tolerance=0.0001)
```

---

## CellComplex Class — topologicpy.CellComplex

```python
CellComplex.ByFaces(faces, tolerance=0.0001, silent=False)
# Returns: topologic.CellComplex — non-manifold structure from face list
# Key AEC use: input all wall/floor/ceiling faces; shared faces become internal boundaries

CellComplex.ByCells(cells, transferDictionaries=False, silent=False)
# Returns: CellComplex — merge cells that share faces

CellComplex.ByWires(wires, triangulate=False, tolerance=0.0001)
# Returns: CellComplex — lofted through wires

CellComplex.ExternalBoundary(cellcomplex)
# Returns: Shell — outer envelope shell

CellComplex.ExternalFaces(cellcomplex)
# Returns: list[Face] — faces on the exterior boundary

CellComplex.InternalFaces(cellcomplex)
# Returns: list[Face] — faces inside the CellComplex

CellComplex.NonManifoldFaces(cellcomplex)
# Returns: list[Face] — faces shared by more than two cells

CellComplex.Decompose(cellcomplex, tiltAngle=70.0, tolerance=0.0001)
# Returns: dict with keys:
#   'externalVerticalFaces' — exterior walls
#   'externalTopHorizontalFaces' — roof faces
#   'externalBottomHorizontalFaces' — ground floor faces
#   'externalInclinedFaces' — sloped exterior faces
#   'internalVerticalFaces' — internal walls
#   'internalHorizontalFaces' — internal floors/ceilings
#   'internalInclinedFaces' — internal sloped faces

CellComplex.Cells(cellcomplex)
# Returns: list[Cell] — all cells (rooms)

CellComplex.Faces(cellcomplex)
# Returns: list[Face] — all faces

CellComplex.Edges(cellcomplex)
# Returns: list[Edge]

CellComplex.Vertices(cellcomplex)
# Returns: list[Vertex]

CellComplex.Volume(cellcomplex, mantissa=6)
# Returns: float — total volume

CellComplex.RemoveCollinearEdges(cellcomplex, angTolerance=0.1, tolerance=0.0001)
# Returns: CellComplex — simplified geometry

# Primitives:
CellComplex.Box(origin=None, width=1.0, length=1.0, height=1.0, uSides=2, vSides=2, wSides=2, direction=[0,0,1], placement='center')
CellComplex.Prism(origin=None, width=1.0, length=1.0, height=1.0, uSides=2, vSides=2, wSides=2)
CellComplex.Tetrahedron(origin=None, radius=0.5, depth=3)
```

---

## Graph Class — topologicpy.Graph

```python
Graph.ByTopology(topology, direct=True, directApertures=False, viaSharedTopologies=False,
                 viaSharedApertures=False, toExteriorTopologies=False,
                 toExteriorApertures=False, useInternalVertex=True, tolerance=0.0001)
# Returns: topologic.Graph — dual graph from topology
# direct=True: edges between cells that share faces (room adjacency)
# viaSharedTopologies=True: also connect via lower-dimension shared entities
# toExteriorTopologies=True: connect cells to exterior faces/edges

Graph.ByVerticesEdges(vertices, edges)
# Returns: Graph

Graph.ShortestPath(graph, vertexA, vertexB, vertexKey='', edgeKey='')
# Returns: Wire — shortest path as wire between vertices

Graph.ShortestPaths(graph, vertexA, vertexB, timeLimit=10)
# Returns: list[Wire]

Graph.AllPaths(graph, vertexA, vertexB, timeLimit=10)
# Returns: list[Wire] — all possible paths

Graph.AdjacentVertices(graph, vertex)
# Returns: list[Vertex] — directly connected vertices

Graph.Vertices(graph)
# Returns: list[Vertex]

Graph.Edges(graph)
# Returns: list[Edge]

Graph.Order(graph)
# Returns: int — number of vertices

Graph.Size(graph)
# Returns: int — number of edges

Graph.Density(graph)
# Returns: float — edge density (0.0-1.0)

Graph.Diameter(graph)
# Returns: int — maximum shortest path length

Graph.DegreeSequence(graph)
# Returns: list[int] — sorted vertex degrees

Graph.DepthMap(graph, startVertex)
# Returns: dict — {vertex: distance_from_start}

Graph.MST(graph)
# Returns: Graph — minimum spanning tree

Graph.Tree(graph, vertex, tolerance=0.0001)
# Returns: Graph — spanning tree from root vertex

Graph.VisibilityGraph(face, obstacles=[], tolerance=0.0001)
# Returns: Graph — visibility graph for pathfinding

Graph.AddVertex(graph, vertex, tolerance=0.0001)
# Returns: Graph — with added vertex

Graph.AddEdge(graph, edge, tolerance=0.0001)
# Returns: Graph — with added edge

Graph.RemoveVertex(graph, vertex)
# Returns: Graph — with vertex removed

Graph.RemoveEdge(graph, edge)
# Returns: Graph — with edge removed

Graph.ContainsVertex(graph, vertex, tolerance=0.0001)
# Returns: bool

Graph.ContainsEdge(graph, edge, tolerance=0.0001)
# Returns: bool

Graph.TopologicalDistance(graph, vertexA, vertexB)
# Returns: int — number of hops

Graph.NearestVertex(graph, vertex)
# Returns: Vertex — nearest graph vertex

Graph.IsComplete(graph)
# Returns: bool

Graph.IsolatedVertices(graph)
# Returns: list[Vertex] — vertices with no edges

Graph.ExportToCSV(graph, path, overwrite=False)
# Exports graph as CSV files (vertex list + edge list)

Graph.ByIFCPath(path, tolerance=0.0001)
# Returns: Graph — graph built from IFC file space relationships

Graph.HasseDiagram(topology, types=None)
# Returns: Graph — directed Hasse diagram of topology hierarchy

Graph.Quotient(topology, topologyType, key)
# Returns: Graph — quotient graph grouping by dictionary key value
```

---

## Topology Class — topologicpy.Topology (Base)

```python
Topology.ByGeometry(vertices, edges=[], faces=[], name='', dictionary=None, tolerance=0.0001)
# Returns: topologic.Topology — convert Blender-style geometry to Topologic

Topology.Geometry(topology, transferDictionaries=False)
# Returns: dict {'vertices': [...], 'edges': [...], 'faces': [...]}

Topology.SubTopologies(topology, subTopologyType='vertex')
# Returns: list — sub-entities of given type
# subTopologyType: 'vertex', 'edge', 'wire', 'face', 'shell', 'cell', 'cellcomplex'

Topology.SuperTopologies(topology, hostTopology, superTopologyType='face')
# Returns: list — parent entities containing this topology

Topology.AdjacentTopologies(topology, hostTopology, topologyType='face')
# Returns: list — adjacent entities of given type sharing a sub-entity

Topology.SharedTopologies(topologyA, topologyB, topologyType='vertex')
# Returns: list — entities shared between two topologies

Topology.Boolean(topologyA, topologyB, operation='union', tolerance=0.0001)
# Returns: Topology
# operation: 'union', 'difference', 'intersect', 'symdif', 'merge', 'slice', 'impose', 'imprint'

Topology.Merge(topologies, tolerance=0.0001)
# Returns: Topology — merge list of topologies

Topology.Union(topologyA, topologyB, tolerance=0.0001)
Topology.Difference(topologyA, topologyB, tolerance=0.0001)
Topology.Intersect(topologyA, topologyB, tolerance=0.0001)
Topology.SymmetricDifference(topologyA, topologyB, tolerance=0.0001)
# Boolean operation shortcuts

Topology.Filter(topologies, topologyType='face', filterByDictionary=False, key='', value='')
# Returns: list — filtered by type or dictionary value

Topology.Dimensionality(topology)
# Returns: int — 0=Vertex, 1=Edge/Wire, 2=Face/Shell, 3=Cell/CellComplex

Topology.Type(topology)
# Returns: int — type ID (1=Vertex, 2=Edge, 4=Wire, 8=Face, 16=Shell, 32=Cell, 64=CellComplex, 128=Cluster)

Topology.TypeAsString(topology)
# Returns: str — 'Vertex', 'Edge', 'Wire', 'Face', 'Shell', 'Cell', 'CellComplex', 'Cluster'

Topology.IsPlanar(topology, tolerance=0.0001)
# Returns: bool — True if all vertices are coplanar

Topology.IsSame(topologyA, topologyB)
# Returns: bool

Topology.CenterOfMass(topology, mantissa=6)
# Returns: Vertex

Topology.Centroid(topology, mantissa=6)
# Returns: Vertex

Topology.BoundingBox(topology, optimize=0, mantissa=6)
# Returns: Cell — axis-aligned or optimized bounding box

Topology.ConvexHull(topology)
# Returns: Topology — convex hull

Topology.Triangulate(topology, mode=0, meshSize=None)
# Returns: Topology — triangulated version

Topology.RemoveCollinearEdges(topology, angTolerance=0.1, tolerance=0.0001)
# Returns: Topology — simplified

Topology.Translate(topology, x=0, y=0, z=0)
# Returns: Topology — translated copy

Topology.Rotate(topology, origin=None, direction=[0,0,1], degree=0.0)
# Returns: Topology — rotated copy

Topology.Scale(topology, origin=None, x=1.0, y=1.0, z=1.0)
# Returns: Topology — scaled copy

Topology.Transform(topology, matrix)
# Returns: Topology — transformed copy

Topology.Place(topology, originA, originB)
# Returns: Topology — repositioned copy

Topology.Dictionary(topology)
# Returns: dict — attached metadata dictionary

Topology.SetDictionary(topology, dictionary)
# Returns: Topology — topology with dictionary attached

Topology.TransferDictionaries(sources, sinks, tolerance=0.0001)
# Transfers dictionary data from source topologies to matching sinks

Topology.Analyze(topology)
# Returns: str — human-readable topology description

Topology.ExportToBREP(topology, path, overwrite=False)
Topology.ExportToJSON(topology, path, overwrite=False)
Topology.ExportToOBJ(topology, path, overwrite=False)
Topology.ByBREPPath(path)
Topology.ByJSONPath(path)
Topology.ByOBJPath(path)
Topology.ByIFCPath(path, transferDictionaries=True, tolerance=0.0001)
# I/O methods for various formats
```

---

## EnergyModel Class — topologicpy.EnergyModel

Requires: OpenStudio >= 3.4.0, honeybee-energy >= 1.91.49

```python
EnergyModel.ByTopology(building, shadingSurfaces=None, osModelPath=None,
                        weatherFilePath=None, designDayFilePath=None,
                        floorLevels=None, buildingName='TopologicBuilding',
                        buildingType='Commercial', northAxis=0.0,
                        glazingRatio=0.0, coolingTemp=25.0, heatingTemp=20.0,
                        defaultSpaceType=None, spaceNameKey='TOPOLOGIC_name',
                        spaceTypeKey='TOPOLOGIC_type', mantissa=6, tolerance=0.0001)
# Returns: openstudio.Model
# building: CellComplex or Cell representing the building
# floorLevels: list[float] — z-coordinates of floor levels

EnergyModel.ByImportedIFC(path, weatherFilePath=None, designDayFilePath=None,
                           floorLevels=None, buildingName='', glazingRatio=0.0)
# Returns: openstudio.Model — direct IFC to energy model

EnergyModel.ByImportedOSM(path)
# Returns: openstudio.Model — load existing OpenStudio model

EnergyModel.ExportToOSM(model, path, overwrite=False)
# Returns: bool

EnergyModel.ExportToGBXML(model, path, overwrite=False)
# Returns: bool

EnergyModel.ExportToIDF(model, path, overwrite=False)
# Returns: bool — export to EnergyPlus IDF format

EnergyModel.ExportToHBJSON(model, path, overwrite=False)
# Returns: bool — export to Honeybee JSON

EnergyModel.Run(model, weatherFilePath, osBinaryPath, outputFolder, removeFiles=False)
# Returns: openstudio.Model — with simulation results attached

EnergyModel.Query(model, reportName='HVACSizingSummary',
                  reportForString='Entire Facility',
                  tableName='Zone Sensible Cooling',
                  columnName='Calculated Design Load',
                  rowNames=[], units='W')
# Returns: list — query results from SQL output

EnergyModel.ReportNames(model)   # Returns: list[str]
EnergyModel.TableNames(model, reportName)   # Returns: list[str]
EnergyModel.RowNames(model, reportName, tableName)   # Returns: list[str]
EnergyModel.ColumnNames(model, reportName, tableName)   # Returns: list[str]
EnergyModel.Units(model, reportName, tableName, columnName)   # Returns: list[str]

EnergyModel.Topologies(model, tolerance=0.0001)
# Returns: dict — {'cells': [...], 'apertures': [...], 'shadingFaces': [...]}

EnergyModel.DefaultConstructionSets(model)   # Returns: list
EnergyModel.DefaultScheduleSets(model)       # Returns: list
EnergyModel.SpaceTypes(model)                # Returns: list
EnergyModel.SqlFile(model)                   # Returns: str — path to SQL results file
```

---

## IFC Integration — TopologicSverchok IFC Nodes

These nodes use ifcopenshell internally:

```
IFCReadFile(path) -> ifc
IFCBuildingElements(ifc, elementType='IfcWall') -> topologies
IFCClashDetection(ifc, tolerance=0.001) -> clashes
IFCConnectBuildingElements(ifc) -> ifc (with connections added)
IFCCreateSpaces(ifc) -> ifc (with spaces detected)
IFCAdd2ndLevelBoundaries(ifc) -> ifc (with 2nd level space boundaries)
IFCExportToHBJSON(ifc, path) -> hbjson
IFCWriteFile(ifc, path) -> success
```

`TopologyByImportedIFC(path)` uses `Topology.ByIFCPath()` to load IFC geometry as Topologic entities with dictionaries containing IFC attributes (GlobalId, Name, type).

---

## Dictionary Class — topologicpy.Dictionary

Topologic dictionaries attach metadata to any topology entity.

```python
Dictionary.ByKeysValues(keys, values)
# Returns: dict-like topologic.Dictionary

Dictionary.ValueAtKey(dictionary, key)
# Returns: value (string, float, int, or list)

Dictionary.SetValueAtKey(dictionary, key, value)
# Returns: updated Dictionary

Dictionary.Keys(dictionary)
# Returns: list[str]

Dictionary.Values(dictionary)
# Returns: list

Dictionary.ByMergedDictionaries(dictionaries)
# Returns: merged Dictionary

Dictionary.ByObjectProperties(obj)
# Returns: Dictionary from Blender object custom properties
```
