# OpenAEC Foundation IFC & Infrastructure Ecosystem Research

**Date:** 2026-03-05
**Researcher:** research-openaec-ifc (automated agent)
**Scope:** 3 OpenAEC Foundation repositories focused on IFC patterns, infrastructure, and collaboration

---

## 1. Monty IFC Viewer

**Repository:** https://github.com/OpenAEC-Foundation/monty-ifc-viewer
**Purpose:** Web-based IFC model viewer with desktop wrapper (Tauri)
**Tech Stack:** Solid.js, Three.js, web-ifc (v0.0.66), Vite, Tauri v2

### 1.1 IFC File Loading

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

### 1.2 IFC API Initialization (web-ifc)

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

### 1.3 Property Extraction Patterns

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

**Skill Relevance:** `bonsai-syntax-properties` — The property traversal pattern (IfcRelDefinesByProperties → IfcPropertySet → HasProperties) is identical in IfcOpenShell but uses different API syntax.

### 1.4 Geometry Extraction

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

**Skill Relevance:** `bonsai-impl-geometry` — The vertex unpacking pattern (6 floats: position + normal) and matrix transformation application are analogous to IfcOpenShell's `ifcopenshell.geom.create_shape()` output.

### 1.5 Web-Based Architecture Patterns

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

### 1.6 Anti-Patterns Found

| Anti-Pattern | Location | Description |
|-------------|----------|-------------|
| Silent error swallowing | geometry-service.ts, property-service.ts | `catch (_) {}` blocks hide all errors |
| Non-reactive data mutation | state/ifc-store.ts | `export let elements: IFCElement[] = []` bypasses Solid.js signals |
| Magic opacity threshold | raycaster-service.ts | Hardcoded `opacity > 0.2` for selection filtering |
| Case-sensitive material matching | material-classifier.ts | `matLower.includes('holz')` does not cover all casing variants |
| Animation loop leak risk | useThreeCanvas.ts | `requestAnimationFrame` not guaranteed to be cleaned up |
| O(n) material resolution | property-service.ts | Loops through all relationships per element |

---

## 2. INB Template

**Repository:** https://github.com/OpenAEC-Foundation/inb-template
**Purpose:** IFC template and library for the Dutch construction industry (Ifc NL Bouw)
**Version:** Template 0.6, Library 0.1
**Schema:** IFC4 (IFC4x3 ready in development)
**Tool:** IfcOpenShell 0.8.4-alpha250721 + Bonsai 0.8.4-alpha250721

### 2.1 Repository Structure

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

### 2.2 Property Set Definitions

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

### 2.3 Classification System (NL-SfB)

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

### 2.4 Material and Element Type Organization

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

### 2.5 ODS-Based Data Definition Pattern

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

### 2.6 Anti-Patterns Found

| Anti-Pattern | Location | Description |
|-------------|----------|-------------|
| Hardcoded Windows paths | CreateLibrary.py | `path = "C:/Users/Gebruiker/Documents/GitHub/..."` not portable |
| bpy dependency | Multiple scripts | Scripts depend on Blender runtime; cannot run standalone |
| Non-standard property set naming | Template IFC | `EPset_*` instead of standard `Pset_*` reduces interoperability |
| No Quantity Take-Off sets | Template IFC | No `Qto_*` property sets for cost estimation |
| Incomplete classification integration | 04_development | Classification libraries exist but are not assigned to elements in the main template |
| RGB scaling factor | CreateLibrary.py | `/ 51` instead of `/ 255`; produces 0-5 range instead of 0-1 |

---

## 3. Nextcloud Check-in/Check-out Feature + IfcGit-4-Nextcloud

**Original target:** https://github.com/OpenAEC-Foundation/Nextcloud-Check-in-Check-out-feature (PRIVATE — inaccessible)
**Alternative researched:** https://github.com/OpenAEC-Foundation/ifcgit-4-nextcloud (PUBLIC)
**Purpose:** IFC version control and collaboration platform with file locking
**Tech Stack:** FastAPI, pygit2, IfcOpenShell, PostgreSQL, Redis, Vue.js 3, Three.js

The Nextcloud Check-in/Check-out repository is private and could not be cloned. Its description states: *"With this feature you can check out and check in files or directories like Autodesk Vault. It makes it possible for mech. engineers to work on parts and assemblies or just view them. This repo will mainly be focussed on shared structured Claude Code projects."*

The public **ifcgit-4-nextcloud** repository implements related patterns and was researched as a substitute.

### 3.1 File Locking Patterns

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

### 3.2 Concurrent File Access Patterns

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

### 3.3 IFC Merge with Semantic Conflict Resolution

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

### 3.4 IfcOpenShell Property Extraction (Server-Side)

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

**Skill Relevance:** `bonsai-syntax-properties` — This is the standard IfcOpenShell pattern for property set extraction: `element.IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties → IfcPropertySingleValue.NominalValue.wrappedValue`.

### 3.5 WebDAV Integration for CAD Tools

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

### 3.6 Anti-Patterns Found

| Anti-Pattern | Location | Description |
|-------------|----------|-------------|
| Incomplete ifcmerge integration | merge_service.py:73 | `# TODO` — resolved conflicts not written back to index |
| FileLock model unused | models.py:37-45 | Lock acquisition/release endpoints not implemented |
| No conflict detection at API level | git/routes.py | Simultaneous commits on same branch may fail silently |
| Full file in-memory via WebDAV | webdav/handler.py | Large IFC files cause high memory usage |
| No fragment cache cleanup | projects/service.py | Orphaned cache files accumulate on project deletion |
| No retry logic | fragments/service.py | Failed fragment generation not automatically retried |

---

## Summary Table

| Repository | Key Findings | Impacted Skills |
|-----------|-------------|-----------------|
| **Monty IFC Viewer** | Web-based IFC viewer using web-ifc (WASM), not IfcOpenShell. Solid.js + Three.js architecture. Property traversal via IfcRelDefinesByProperties. Vertex format: 6 floats (pos+normal). Raycasting for selection. | `bonsai-syntax-properties`, `bonsai-impl-geometry`, `bonsai-impl-classification` |
| **INB Template** | IFC4 schema. Dutch construction template with 113 element types. Custom EPset_* property sets (non-standard). NL-SfB (2005/2019), STABU, BB-SfB classification libraries as separate IFC ProjectLibrary files. ODS-driven data definition. IfcMaterialLayerSet + IfcMaterialProfileSet patterns. | `bonsai-syntax-properties`, `bonsai-impl-classification`, `bonsai-impl-materials` |
| **IfcGit-4-Nextcloud** | Git-backed IFC versioning with FastAPI. FileLock model (DB-backed, with expiry). Branch-based concurrent isolation. Redis job queue (max 4 jobs). ifcmerge for semantic IFC conflict resolution (incomplete). WebDAV for native CAD access. IfcOpenShell property extraction server-side. | Multi-agent workflow, `bonsai-syntax-properties`, concurrent file strategy |
| **Nextcloud Check-in/out** (private) | Autodesk Vault-like check-in/out for files and directories. Focus on shared Claude Code projects. Could not be accessed (private repository). | Multi-agent workflow, file locking strategy |

---

## Cross-Cutting Observations

1. **Property extraction patterns are consistent** across all three repos: `element.IsDefinedBy → IfcRelDefinesByProperties → IfcPropertySet → HasProperties`. This is the canonical IfcOpenShell/web-ifc pattern.

2. **Classification is modular** in the Dutch ecosystem: separate IFC ProjectLibrary files per classification system, imported into projects via IfcRelAssociatesClassification.

3. **Concurrent file access** favors Git branch isolation over pessimistic locking. File locks exist as a model but are not enforced at the API level.

4. **IFC merge** is an unsolved problem: ifcmerge (Rust binary) attempts semantic 3-way merge but the integration is incomplete.

5. **No standard Pset_ usage** in the INB Template — all custom EPset_* sets. This reduces interoperability with IFC validation tools that expect standard property set names.

6. **Schema version convergence:** All repos target IFC4, with IFC4x3 readiness in development.
