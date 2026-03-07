---
name: sverchok-impl-topologic
description: 'Covers TopologicSverchok for building topology analysis — non-manifold topology operations, CellComplex workflows, space adjacency graphs, dual graphs, and energy simulation integration. Activates when working with TopologicSverchok nodes, building envelope analysis, space adjacency graphs, room connectivity, CellComplex-based BIM workflows, or energy simulation preparation in Blender/Sverchok.'
license: MIT
compatibility: 'Designed for Claude Code. Requires Blender 4.0+/5.x with Sverchok v1.2.0+ and TopologicSverchok v0.8.3+.'
metadata:
  author: OpenAEC-Foundation
  version: '1.0'
---

# sverchok-impl-topologic

## Quick Reference

### What Is TopologicSverchok

TopologicSverchok integrates the **topologicpy** non-manifold topology (NMT) library with Sverchok's parametric node system. It enables hierarchical and topological representations of architectural spaces, buildings, and artefacts through 326 nodes organized by topology class.

- **326 nodes** in the `nodes/Topologic/` directory
- **Non-manifold topology**: lines, surfaces, and volumes coexist simultaneously
- **AEC workflows**: envelope analysis, space adjacency, dual graphs, energy simulation
- **Integrations**: IFC (ifcopenshell), OpenStudio/EnergyPlus, Speckle, Neo4j, DGL
- **License**: AGPL-3.0
- **Repository**: https://github.com/wassimj/TopologicSverchok
- **Python API**: https://topologicpy.readthedocs.io

### Topology Class Hierarchy

| Class | Dimension | AEC Example | Key Nodes |
|-------|-----------|-------------|-----------|
| **Vertex** | 0D | Column insertion point | `VertexByCoordinates`, `VertexEnclosingCell` |
| **Edge** | 1D | Structural beam axis | `EdgeByVertices`, `EdgeLength`, `EdgeDirection` |
| **Wire** | 1D composite | Room boundary outline | `WireByEdges`, `WireRectangle`, `WireIsClosed` |
| **Face** | 2D | Wall surface, floor slab | `FaceByEdges`, `FaceArea`, `FaceTrimByWire` |
| **Shell** | 2D composite | Building envelope | `ShellByFaces`, `ShellByLoft`, `ShellIsClosed` |
| **Cell** | 3D | Room volume | `CellByFaces`, `CellVolume`, `CellInternalVertex` |
| **CellComplex** | 3D non-manifold | Multi-room building | `CellComplexByFaces`, `CellComplexDecompose` |
| **Cluster** | Mixed | Building group | `ClusterByTopologies`, `ClusterType` |

### Critical Warnings

**NEVER** confuse Topologic topology objects with Blender mesh data — they are completely different representations. Convert with `TopologyByGeometry` (Blender to Topologic) or `TopologyBlenderGeometry` (Topologic to Blender).

**NEVER** pass raw Blender vertex coordinates directly to Topologic nodes — always use `VertexByCoordinates` to create Topologic Vertex objects first.

**NEVER** assume all faces in a CellComplex are planar — TopologicSverchok requires planar geometry; use `TopologyIsPlanar` to verify before building CellComplex structures.

**ALWAYS** check optional dependency availability before using IFC, Energy, Speckle, or Neo4j nodes — these require separate installations of ifcopenshell, OpenStudio, specklepy, or py2neo.

**ALWAYS** use `CellComplexDecompose` to categorize faces as external/internal/non-manifold before querying building envelope — non-manifold faces are shared between cells and should not be treated as exterior surfaces.

### Decision Tree

```
Need to analyze building spaces topologically?
├── From Blender geometry → TopologyByGeometry -> CellComplexByFaces
├── From IFC file → IFCReadFile -> TopologyByImportedIFC
├── Room adjacency → CellComplexByFaces -> CellAdjacentCells
├── Dual graph (rooms=nodes, walls=edges) → GraphByTopology
└── Envelope faces → CellComplexDecompose -> filter external faces

Need to run energy simulation?
├── From topology → EnergyModelByTopology -> EnergyModelRunSimulation
├── From IFC → EnergyModelByImportedIFC -> EnergyModelExportToOSM
├── Export for EnergyPlus → EnergyModelExportToIDF
└── Query results → EnergyModelQuery (reports, tables, rows, columns)

Need to analyze space connectivity?
├── Space graph → GraphByTopology (CellComplex input)
├── Shortest path between rooms → GraphShortestPath
├── Centrality analysis → GraphBetweennessCentrality / GraphDegreeCentrality
└── Export graph → GraphExportToCSV

Need to work with IFC?
├── Read IFC → IFCReadFile
├── Extract elements → IFCBuildingElements
├── Create spaces → IFCCreateSpaces
├── Clash detection → IFCClashDetection
└── Connect elements → IFCConnectBuildingElements
```

---

## Installation

### Requirements

| Dependency | Version | Required |
|------------|---------|----------|
| Blender | >= 4.0 | Yes |
| Sverchok | >= 1.2.0 | Yes |
| topologicpy | latest | Yes (auto-installed) |
| NumPy | >= 1.22.4 | Yes |
| ifcopenshell | latest | For IFC nodes |
| OpenStudio | >= 3.4.0 | For Energy nodes |
| honeybee-energy | >= 1.91.49 | For HB nodes |
| specklepy | >= 2.7.6 | For Speckle nodes |
| py2neo | >= 2021.2.3 | For Neo4j nodes |
| DGL | latest | For DGL/ML nodes |

### Installation Steps

1. Download TopologicSverchok ZIP from https://github.com/wassimj/TopologicSverchok
2. In Blender: Edit > Preferences > Add-ons > Install
3. Select the ZIP file — do NOT extract first
4. Enable the add-on in the list
5. Use the `InstallDependencies` node on first use to install topologicpy and optional packages

---

## Essential Patterns

### Pattern 1: Build a CellComplex from Blender Geometry

```
Sverchok node tree setup:
Object In (Blender object) -> [vertices, edges, faces]
   -> TopologyByGeometry (Topology object)
   -> CellComplexByFaces (CellComplex)
   -> CellComplexDecompose (external faces, internal faces, non-manifold faces)
```

The `TopologyByGeometry` node converts Blender mesh data (vertex coordinates + face indices) into a Topologic Topology. Planar geometry only — non-planar faces will cause errors. `CellComplexByFaces` merges all faces into a non-manifold structure where shared faces become internal boundaries between cells.

### Pattern 2: Room Adjacency Graph

```
CellComplex
   -> GraphByTopology (direct=True, viaSharedTopologies=True, toExteriorTopologies=False)
   -> Graph object
      -> GraphVertices -> list of room vertices
      -> GraphEdges -> list of shared-wall edges
      -> GraphShortestPath (vertexA, vertexB) -> Wire path
      -> GraphDepthMap (startVertex) -> distance map
```

`GraphByTopology` creates a dual graph where each Cell becomes a vertex and shared Faces (internal walls/slabs) become edges. Set `direct=True` for cell-to-cell edges via shared faces. The output Graph supports all standard graph operations (shortest path, centrality, community detection).

### Pattern 3: Building Envelope Analysis

```
CellComplex
   -> CellComplexDecompose
      -> externalFaces -> FaceArea (total envelope area)
      -> internalFaces -> FaceArea (total internal surface area)
      -> nonManifoldFaces -> (shared between cells — zero-thickness boundaries)
   -> filter externalFaces by FaceFacingToward (direction=[0,0,1]) -> roof faces
   -> filter externalFaces by FaceFacingToward (direction=[0,0,-1]) -> floor faces
   -> remaining externalFaces -> wall faces
```

### Pattern 4: Energy Simulation from Topology

```
CellComplex (building geometry)
   -> EnergyModelByTopology (
        building=cellcomplex,
        weatherFilePath="path/to/weather.epw",
        floorLevels=[0, 3, 6, 9],
        glazingRatio=0.4,
        coolingTemp=25.0,
        heatingTemp=20.0
      )
   -> EnergyModel
      -> EnergyModelRunSimulation (osBinaryPath, outputFolder)
      -> EnergyModelQuery (reportName, tableName, columnName)
      -> simulation results (heating/cooling loads)
```

### Pattern 5: IFC to Topologic Workflow

```
IFCReadFile (path) -> IFC file object
   -> IFCBuildingElements (ifc, elementType="IfcWall") -> wall topologies
   -> TopologyByImportedIFC -> Topology objects
   -> CellComplexByFaces -> building topology
   -> GraphByTopology -> adjacency graph
```

Alternatively use `EnergyModelByImportedIFC` for direct IFC-to-energy-model conversion without intermediate topology steps.

---

## Common Operations

### Node Categories Overview

| Category | Node Count | Purpose |
|----------|-----------|---------|
| Vertex | 10 | Point creation, distance, containment |
| Edge | 12 | Linear elements, direction, length |
| Wire | 16 | Connected edge paths, primitives |
| Face | 26 | Surface elements, area, normals |
| Shell | 10 | Connected face collections |
| Cell | 20 | Volumetric elements, primitives |
| CellComplex | 10 | Non-manifold building models |
| Cluster | 2 | Mixed topology collections |
| Topology | 50 | Base operations (boolean, transform, I/O) |
| Graph | 36 | Adjacency, pathfinding, analysis |
| EnergyModel | 20 | OpenStudio/EnergyPlus simulation |
| IFC | 10 | IFC file operations |
| HB (Honeybee) | 8 | Honeybee energy model |
| DGL | 18 | Deep graph learning |
| Speckle | 17 | Speckle interoperability |
| Dictionary | 8 | Metadata key-value storage |
| Color | 5 | Visualization coloring |
| Neo4j | 5 | Graph database export |
| Matrix | 4 | Transformation matrices |

### Key Node Reference

#### CellComplex Nodes

| Node | Inputs | Outputs | Description |
|------|--------|---------|-------------|
| `CellComplexByFaces` | faces, tolerance | cellcomplex | Merge faces into non-manifold CellComplex |
| `CellComplexByCells` | cells | cellcomplex | Merge cells by shared faces |
| `CellComplexDecompose` | cellcomplex | externalFaces, internalFaces, nonManifoldFaces, externalVertices, internalVertices | Decompose by location/type |
| `CellComplexExternalBoundary` | cellcomplex | shell | Get outer envelope shell |
| `CellComplexInternalBoundaries` | cellcomplex | faces | Get internal boundary faces |
| `CellComplexNonManifoldFaces` | cellcomplex | faces | Get non-manifold (shared) faces |
| `CellComplexPrism` | origin, width, length, height, uSides, vSides, wSides | cellcomplex | Subdivided box |

#### Graph Nodes

| Node | Inputs | Outputs | Description |
|------|--------|---------|-------------|
| `GraphByTopology` | topology, direct, viaSharedTopologies, toExteriorTopologies | graph | Build dual graph from topology |
| `GraphShortestPath` | graph, vertexA, vertexB | path (Wire) | Dijkstra shortest path |
| `GraphAllPaths` | graph, vertexA, vertexB | paths | All paths between vertices |
| `GraphDepthMap` | graph, startVertex | depthMap | Distance from start vertex |
| `GraphAdjacentVertices` | graph, vertex | vertices | Direct neighbors |
| `GraphVertices` | graph | vertices | All graph vertices |
| `GraphEdges` | graph | edges | All graph edges |
| `GraphDensity` | graph | density | Edge density ratio |
| `GraphDiameter` | graph | diameter | Maximum shortest path |
| `GraphMST` | graph | mst | Minimum spanning tree |

#### Topology Base Nodes

| Node | Inputs | Outputs | Description |
|------|--------|---------|-------------|
| `TopologyByGeometry` | vertices, edges, faces | topology | Blender mesh to Topologic |
| `TopologyBlenderGeometry` | topology | vertices, edges, faces | Topologic to Blender mesh |
| `TopologyBoolean` | topologyA, topologyB, operation | topology | Union/Difference/Intersection |
| `TopologyAdjacentTopologies` | topology, topologyType | topologies | Find adjacent entities |
| `TopologySubTopologies` | topology, subTopologyType | topologies | Extract sub-entities |
| `TopologySuperTopologies` | topology, superTopologyType | topologies | Find containing entities |
| `TopologySharedTopologies` | topologyA, topologyB, topologyType | topologies | Find shared entities |
| `TopologyDictionary` | topology | dictionary | Get attached metadata |
| `TopologySetDictionary` | topology, dictionary | topology | Attach metadata |
| `TopologyAnalyze` | topology | string | Describe topology structure |
| `TopologyExportToJSONMK2` | topology, path | success | Export to JSON file |
| `TopologyByImportedJSONMK2` | path | topology | Import from JSON file |

### Dictionary Nodes for Metadata

Topologic entities carry attached dictionaries for BIM data:

| Node | Purpose |
|------|---------|
| `DictionaryByKeysValues` | Create dictionary from key-value lists |
| `DictionaryValueAtKey` | Read value for a specific key |
| `DictionarySetValueAtKey` | Write/update a value |
| `DictionaryKeys` | Get all keys |
| `DictionaryValues` | Get all values |
| `DictionaryByObjectProperties` | Extract Blender object custom properties |

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for all topology classes (topologicpy)
- [references/examples.md](references/examples.md) — Working node tree patterns for AEC workflows
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do with TopologicSverchok

### Cross-References

- `ifcos-impl-geometry` — IFC geometry input for TopologicSverchok workflows
- `sverchok-impl-custom-nodes` — Custom node development for TopologicSverchok extension

### Official Sources

- https://github.com/wassimj/TopologicSverchok
- https://topologic.app/software/
- https://topologicpy.readthedocs.io
- https://github.com/wassimj/topologicpy
