# sverchok-impl-topologic: Working Examples

Sources:
- https://github.com/wassimj/TopologicSverchok (v0.8.3.0)
- https://topologicpy.readthedocs.io
- vooronderzoek-sverchok.md §11

---

## Example 1: Room Adjacency from BIM Geometry

**Goal**: Build a CellComplex from building geometry and compute which rooms are adjacent.

**Node tree setup** (Sverchok + TopologicSverchok):

```
[Object In] ─────────────────────────────────────────────────┐
  object: building_walls_object                               │
  ▼                                                           │
[TopologyByGeometry]                                          │
  IN: vertices (SvVerticesSocket)  ◄── Object In: vertices   │
  IN: edges (SvStringsSocket)      ◄── Object In: edges       │
  IN: faces (SvStringsSocket)      ◄── Object In: faces       │
  OUT: topology ──────────────────────────────────────────────┘
  ▼
[CellComplexByFaces]
  IN: topology (SvStringsSocket - carries Topologic object)
  IN: tolerance = 0.001
  OUT: cellcomplex
  ▼
[CellComplexDecompose]
  IN: cellcomplex
  OUT: externalFaces ──► [FaceArea] ──► total envelope area
  OUT: internalFaces ──► [FaceArea] ──► internal surface area
  OUT: cells ──────────► [cell count via List Length]
```

**For adjacency**: Connect `cellcomplex` to `GraphByTopology`:

```
[CellComplexByFaces]
  OUT: cellcomplex
  ▼
[GraphByTopology]
  IN: topology = cellcomplex
  IN: direct = True
  IN: viaSharedTopologies = False
  IN: toExteriorTopologies = False
  OUT: graph
  ▼
[GraphVertices] ──► list of room vertices (one per cell/room)
[GraphEdges]    ──► list of wall edges (one per shared face between cells)
```

**Python equivalent** (in a SNLite scripting node):

```python
# SNLite node - requires TopologicSverchok installed
import topologicpy
from topologicpy.Topology import Topology
from topologicpy.CellComplex import CellComplex
from topologicpy.Graph import Graph

def process(verts_in, edges_in, faces_in):
    """Build adjacency graph from building geometry."""
    # Convert Sverchok geometry to Topologic topology
    topology = Topology.ByGeometry(
        vertices=verts_in[0],
        edges=edges_in[0],
        faces=faces_in[0],
        tolerance=0.001
    )

    # Build CellComplex (non-manifold building model)
    all_faces = Topology.SubTopologies(topology, subTopologyType='face')
    cc = CellComplex.ByFaces(all_faces, tolerance=0.001)

    if cc is None:
        return None, None

    # Get all rooms (cells)
    cells = CellComplex.Cells(cc)
    print(f"Building has {len(cells)} rooms")

    # Build dual graph (rooms=vertices, shared walls=edges)
    graph = Graph.ByTopology(cc, direct=True, tolerance=0.001)

    # Find adjacent rooms for each room
    g_vertices = Graph.Vertices(graph)
    for v in g_vertices:
        adjacent = Graph.AdjacentVertices(graph, v)
        print(f"Room at {v}: {len(adjacent)} adjacent rooms")

    return cells, graph
```

---

## Example 2: Building Envelope Analysis

**Goal**: Decompose a building CellComplex into roof, wall, and floor faces with area calculations.

**Node tree**:

```
[CellComplexByFaces]
  OUT: cellcomplex
  ▼
[CellComplexDecompose]
  OUT: externalFaces ──────────────────────────────────────┐
                                                            │
  ▼ (pipe externalFaces through FaceFacingToward filter)  │
[FaceFacingToward]                                          │
  IN: faces = externalFaces                                 │
  IN: direction = [0, 0, 1]    (upward-facing = roof)       │
  OUT: facingFaces ──► roof faces                           │
  OUT: notFacingFaces ──────────────────────────────────── ┘
  ▼
[FaceFacingToward]
  IN: faces = notFacingFaces
  IN: direction = [0, 0, -1]   (downward-facing = ground floor)
  OUT: facingFaces ──► ground floor faces
  OUT: notFacingFaces ──► vertical wall faces
```

**Area calculation per face type**:

```
[FaceArea]
  IN: face = (each external face)
  OUT: area ──► [List Statistics: Sum] ──► total roof/wall/floor area
```

**Python equivalent**:

```python
from topologicpy.CellComplex import CellComplex
from topologicpy.Face import Face

def analyze_envelope(cellcomplex):
    """Decompose building envelope into face types."""
    decomposed = CellComplex.Decompose(cellcomplex, tiltAngle=70.0)

    # Extract face categories
    ext_vertical = decomposed.get('externalVerticalFaces', [])
    ext_top_horiz = decomposed.get('externalTopHorizontalFaces', [])
    ext_bot_horiz = decomposed.get('externalBottomHorizontalFaces', [])
    ext_inclined = decomposed.get('externalInclinedFaces', [])

    # Calculate areas
    wall_area = sum(Face.Area(f) for f in ext_vertical)
    roof_area = sum(Face.Area(f) for f in ext_top_horiz)
    floor_area = sum(Face.Area(f) for f in ext_bot_horiz)
    inclined_area = sum(Face.Area(f) for f in ext_inclined)

    total_envelope = wall_area + roof_area + floor_area + inclined_area

    results = {
        'wall_count': len(ext_vertical),
        'wall_area': round(wall_area, 2),
        'roof_count': len(ext_top_horiz),
        'roof_area': round(roof_area, 2),
        'floor_count': len(ext_bot_horiz),
        'floor_area': round(floor_area, 2),
        'inclined_count': len(ext_inclined),
        'inclined_area': round(inclined_area, 2),
        'total_envelope_area': round(total_envelope, 2),
    }
    return results
```

---

## Example 3: Space Connectivity Graph with Shortest Path

**Goal**: Find the shortest walking path between two rooms in a building.

**Node tree**:

```
[CellComplexByFaces]
  OUT: cellcomplex
  ▼
[GraphByTopology]
  IN: topology = cellcomplex
  IN: direct = True
  OUT: graph
  ▼
[GraphVertices]
  OUT: vertices ──► select vertexA (room A) and vertexB (room B)
  ▼
[GraphShortestPath]
  IN: graph = graph
  IN: vertexA = selected room A vertex
  IN: vertexB = selected room B vertex
  OUT: path (Wire) ──► [TopologyBlenderGeometry] ──► visualize in Blender

[GraphDepthMap]
  IN: graph = graph
  IN: startVertex = main entrance vertex
  OUT: depthMap ──► [Dictionary] ──► color rooms by distance from entrance
```

**Python equivalent**:

```python
from topologicpy.Graph import Graph
from topologicpy.Vertex import Vertex

def find_shortest_path(graph, room_a_cell, room_b_cell):
    """Find shortest path between two rooms in building graph."""
    # Get internal vertices for each room (guaranteed interior points)
    from topologicpy.Cell import Cell
    vertex_a = Cell.InternalVertex(room_a_cell, tolerance=0.001)
    vertex_b = Cell.InternalVertex(room_b_cell, tolerance=0.001)

    # Find nearest graph vertices
    graph_vertex_a = Graph.NearestVertex(graph, vertex_a)
    graph_vertex_b = Graph.NearestVertex(graph, vertex_b)

    # Compute shortest path
    path = Graph.ShortestPath(graph, graph_vertex_a, graph_vertex_b)

    if path is None:
        print("No path found — rooms may not be connected")
        return None

    # Topological distance (number of rooms to traverse)
    distance = Graph.TopologicalDistance(graph, graph_vertex_a, graph_vertex_b)
    print(f"Path traverses {distance} rooms")

    return path
```

---

## Example 4: Energy Simulation Preparation

**Goal**: Convert a building CellComplex to an OpenStudio energy model and run a simulation.

**Node tree**:

```
[CellComplexByFaces]
  OUT: cellcomplex (building topology with rooms as cells)
  ▼
[EnergyModelByTopology]
  IN: building = cellcomplex
  IN: weatherFilePath = "/path/to/weather.epw"
  IN: designDayFilePath = "/path/to/design.ddy"
  IN: floorLevels = [0.0, 3.0, 6.0, 9.0]
  IN: buildingName = "MyBuilding"
  IN: buildingType = "Commercial"
  IN: northAxis = 0.0
  IN: glazingRatio = 0.4
  IN: coolingTemp = 25.0
  IN: heatingTemp = 20.0
  OUT: energyModel
  ▼
[EnergyModelRunSimulation]
  IN: model = energyModel
  IN: weatherFilePath = "/path/to/weather.epw"
  IN: osBinaryPath = "/usr/bin/openstudio"
  IN: outputFolder = "/tmp/simulation_results"
  OUT: simulatedModel
  ▼
[EnergyModelQuery]
  IN: model = simulatedModel
  IN: reportName = "HVACSizingSummary"
  IN: tableName = "Zone Sensible Cooling"
  IN: columnName = "Calculated Design Load"
  OUT: results ──► cooling loads per zone (room)
```

**Python equivalent**:

```python
from topologicpy.EnergyModel import EnergyModel

def run_energy_simulation(cellcomplex, weather_path, osm_binary, output_dir):
    """Create and run energy simulation from building topology."""
    # Create OpenStudio model from CellComplex
    model = EnergyModel.ByTopology(
        building=cellcomplex,
        weatherFilePath=weather_path,
        designDayFilePath=weather_path.replace('.epw', '.ddy'),
        floorLevels=[0.0, 3.0, 6.0, 9.0],
        buildingType='Commercial',
        glazingRatio=0.4,
        coolingTemp=25.0,
        heatingTemp=20.0
    )

    if model is None:
        raise RuntimeError("Failed to create energy model")

    # Export to inspect before simulation
    EnergyModel.ExportToOSM(model, output_dir + "/building.osm", overwrite=True)

    # Run EnergyPlus simulation
    simulated = EnergyModel.Run(
        model=model,
        weatherFilePath=weather_path,
        osBinaryPath=osm_binary,
        outputFolder=output_dir
    )

    # Query results
    report_names = EnergyModel.ReportNames(simulated)
    cooling_loads = EnergyModel.Query(
        simulated,
        reportName='HVACSizingSummary',
        reportForString='Entire Facility',
        tableName='Zone Sensible Cooling',
        columnName='Calculated Design Load',
        units='W'
    )

    return cooling_loads
```

---

## Example 5: IFC to Topologic to Graph Workflow

**Goal**: Load an IFC file, extract building spaces, and compute connectivity.

**Node tree**:

```
[IFCReadFile]
  IN: path = "/path/to/building.ifc"
  OUT: ifc
  ▼
[IFCBuildingElements]
  IN: ifc = ifc
  IN: elementType = "IfcSpace"
  OUT: topologies (list of Face/Cell topologies per space)
  ▼
[CellComplexByCells]
  IN: cells = topologies
  OUT: cellcomplex
  ▼
[GraphByTopology]
  IN: topology = cellcomplex
  IN: direct = True
  OUT: graph
  ▼
[GraphExportToCSV]
  IN: graph = graph
  IN: path = "/path/to/output/"
  OUT: success (exports vertices.csv + edges.csv)
```

**Python equivalent**:

```python
from topologicpy.Topology import Topology
from topologicpy.CellComplex import CellComplex
from topologicpy.Graph import Graph

def ifc_to_graph(ifc_path, output_csv_path):
    """Load IFC and build space adjacency graph."""
    # Load IFC as Topologic topology
    topology = Topology.ByIFCPath(ifc_path, transferDictionaries=True)

    if topology is None:
        raise FileNotFoundError(f"Cannot load IFC: {ifc_path}")

    # Extract all cells (spaces)
    cells = Topology.SubTopologies(topology, subTopologyType='cell')
    print(f"Found {len(cells)} spaces in IFC")

    if len(cells) < 2:
        raise ValueError("Need at least 2 spaces for adjacency graph")

    # Build CellComplex from cells
    cc = CellComplex.ByCells(cells, transferDictionaries=True)

    # Build adjacency graph
    graph = Graph.ByTopology(cc, direct=True, tolerance=0.001)

    # Export for downstream analysis
    Graph.ExportToCSV(graph, output_csv_path, overwrite=True)

    # Print space names from IFC dictionaries
    for v in Graph.Vertices(graph):
        d = Topology.Dictionary(v)
        if d:
            from topologicpy.Dictionary import Dictionary
            name = Dictionary.ValueAtKey(d, 'Name')
            print(f"Space: {name}")

    return graph
```

---

## Example 6: Attach BIM Data to Topology

**Goal**: Attach room program/type data to CellComplex cells and query it.

```python
from topologicpy.CellComplex import CellComplex
from topologicpy.Cell import Cell
from topologicpy.Topology import Topology
from topologicpy.Dictionary import Dictionary

def attach_room_data(cellcomplex, room_data_list):
    """Attach program data dictionary to each cell in CellComplex.

    Args:
        cellcomplex: CellComplex topology
        room_data_list: list of dicts, one per cell, e.g.:
            [{'Name': 'Office_01', 'Type': 'Office', 'Area': 25.0}, ...]
    """
    cells = CellComplex.Cells(cellcomplex)

    if len(cells) != len(room_data_list):
        raise ValueError(f"Mismatch: {len(cells)} cells but {len(room_data_list)} data entries")

    for cell, room_data in zip(cells, room_data_list):
        keys = list(room_data.keys())
        values = list(room_data.values())
        d = Dictionary.ByKeysValues(keys, values)
        Topology.SetDictionary(cell, d)

    return cellcomplex

def query_room_type(cellcomplex, room_type='Office'):
    """Return all cells with a specific room type."""
    cells = CellComplex.Cells(cellcomplex)
    matching = []

    for cell in cells:
        d = Topology.Dictionary(cell)
        if d is None:
            continue
        cell_type = Dictionary.ValueAtKey(d, 'Type')
        if cell_type == room_type:
            matching.append(cell)

    return matching
```

---

## Example 7: Dual Graph for Circulation Analysis

**Goal**: Compute space syntax metrics (integration, depth) for a floor plan.

```python
from topologicpy.CellComplex import CellComplex
from topologicpy.Graph import Graph
from topologicpy.Topology import Topology
from topologicpy.Dictionary import Dictionary

def space_syntax_analysis(cellcomplex):
    """Compute basic space syntax metrics for building floor plan."""
    # Build adjacency graph (dual graph)
    graph = Graph.ByTopology(
        cellcomplex,
        direct=True,
        viaSharedTopologies=False,
        toExteriorTopologies=False,
        tolerance=0.001
    )

    vertices = Graph.Vertices(graph)
    n = Graph.Order(graph)

    if n < 2:
        return {}

    metrics = {}
    for v in vertices:
        # Depth from this vertex to all others
        depth_map = Graph.DepthMap(graph, v)

        # Mean depth (average topological distance)
        total_depth = sum(depth_map.values())
        mean_depth = total_depth / (n - 1) if n > 1 else 0

        # Relative Asymmetry (RA) — space syntax integration measure
        # RA = 2(MD - 1) / (n - 2)
        ra = 2 * (mean_depth - 1) / (n - 2) if n > 2 else 0

        # Real Relative Asymmetry (RRA) — normalized
        # Using D_n value from space syntax lookup tables (simplified)
        d_n = 1.0  # Simplified; proper implementation uses lookup table
        rra = ra / d_n if d_n > 0 else 0

        # Integration = 1/RRA (higher = more integrated/accessible)
        integration = 1 / rra if rra > 0 else float('inf')

        # Degree (number of direct neighbors)
        adj = Graph.AdjacentVertices(graph, v)
        degree = len(adj)

        # Store metrics in vertex dictionary
        d = Dictionary.ByKeysValues(
            ['mean_depth', 'integration', 'degree', 'ra'],
            [round(mean_depth, 3), round(integration, 3), degree, round(ra, 3)]
        )
        Topology.SetDictionary(v, d)
        metrics[id(v)] = {'mean_depth': mean_depth, 'integration': integration, 'degree': degree}

    return metrics
```
