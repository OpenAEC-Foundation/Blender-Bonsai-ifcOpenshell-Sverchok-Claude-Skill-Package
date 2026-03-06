# OpenAEC Foundation Projects: IfcOpenShell Usage Patterns

## Overview

The OpenAEC Foundation (https://github.com/OpenAEC-Foundation) maintains 35+ repositories spanning construction calculation tools, IFC format utilities, and open-source AEC software. Four repositories were analyzed for IfcOpenShell usage patterns. The findings reveal two distinct integration approaches: **building.py** and **INB Template** demonstrate extensive IfcOpenShell usage with complementary patterns (abstraction-layer vs. direct-API), while **Monty IFC Viewer** uses web-ifc (JavaScript) instead, and **AEC Scripts** does not exist as a public repository.

---

## building.py

**Repository:** https://github.com/OpenAEC-Foundation/building-py
**Language:** Python (requires 3.10+)
**Status:** Beta (v0.1), actively maintained, 30 stars
**License:** Open source

### Project Description

building.py is a Python library for creating buildings, building systems, and objects, then exporting them to multiple platforms including Blender, Revit, Speckle, FreeCAD, and IFC (via IfcOpenShell). The core philosophy treats nearly all building elements as beams or panels, providing 40+ geometric and structural component types with multi-format export.

### IfcOpenShell Modules Used

| Module | Usage |
|--------|-------|
| `ifcopenshell` (core) | `ifcopenshell.open()` for loading IFC files, `ifcopenshell.file()` for creating new models |
| `ifcopenshell.api` / `ifcopenshell.api.run` | High-level API for project creation, spatial structure, geometry, materials, types |
| `ifcopenshell.geom` | `ifcopenshell.geom.create_shape()` for geometry processing during IFC import |
| `ifcopenshell.util.element` | Element utility functions (imported as `util`) |

**Dependency declaration:** `ifcopenshell` is listed in `requirements.txt` as a direct dependency.

### Code Patterns

#### Pattern 1: Wrapper-Class Abstraction Layer

building.py wraps all IfcOpenShell operations behind two main classes in `exchange/IFC.py`:

**`CreateIFC` class** — Builder pattern for IFC file creation:
```python
ifc_project = CreateIFC()
ifc_project.add_project("My Project")
ifc_project.add_site("My Site")
ifc_project.add_building("Building A")
ifc_project.add_storey("Ground Floor")
ifc_project.export("output.ifc")
```

Each method internally calls `ifcopenshell.api.run()` but the consumer NEVER interacts with IfcOpenShell directly. This is a deliberate design choice for developer ergonomics.

**`LoadIFC` class** — Reader pattern for IFC import:
```python
loader = LoadIFC(filename, project, filter_type)
loader.load()        # calls ifcopenshell.open()
loader.getobject()   # calls ifc_file.by_type()
loader.convertobject()  # transforms IFC geometry to building-py objects
```

#### Pattern 2: `ifcopenshell.api.run()` as Primary API

The `exchange/IFC.py` module uses the `run()` function from `ifcopenshell.api` as the primary interface. Key API categories used:

```python
from ifcopenshell.api import run

# Unit management
run("unit.assign_unit", model)

# Context setup
run("context.add_context", model, context_type="Model")

# Spatial hierarchy
run("root.create_entity", model, ifc_class="IfcProject", name="Project")
run("aggregate.assign_object", model, relating_object=project, product=site)

# Geometry (multiple approaches)
run("geometry.add_wall_representation", model, context=ctx, ...)
run("geometry.add_profile_representation", model, context=ctx, profile=profile, depth=length)
```

#### Pattern 3: Mixed High-Level and Low-Level API Usage

The `translateObjectsToIFC` function demonstrates mixing `ifcopenshell.api.run()` with direct entity creation:

- **High-level:** `run("root.create_entity", ...)` for standard operations
- **Low-level:** `model.create_entity()` and `model.createIfc*()` for custom geometry (arbitrary profiles, clipping planes)
- Surfaces/Faces become `IfcColumnType` with arbitrary profiles
- Frames become `IfcBeamType` with cross-section profiles
- Floors become `IfcSlab` with extruded solid geometry
- Grids become `IfcGrid` with axes

#### Pattern 4: Geometry Conversion Pipeline

For IFC import, the geometry pipeline uses:
```python
settings = ifcopenshell.geom.settings()
shape = ifcopenshell.geom.create_shape(settings, ifc_element)
# Then converts to building-py internal geometry (Points, Vectors)
```

### Key Takeaways

1. **Abstraction is the dominant pattern.** End users NEVER write `ifcopenshell.api.run()` calls directly; they use `CreateIFC` / `LoadIFC` wrappers.
2. **The `ifcopenshell.api.run()` function is the backbone** of all write operations inside the wrapper classes.
3. **Geometry processing uses `ifcopenshell.geom`** for reading, but manual entity creation for writing complex shapes.
4. **building.py demonstrates that IfcOpenShell can serve as a backend** behind a domain-specific API — users create "beams" and "panels," not "IfcBeamType" entities.

---

## Monty IFC Viewer

**Repository:** https://github.com/OpenAEC-Foundation/monty-ifc-viewer
**Language:** TypeScript (43.2%), HTML (43.0%), CSS (11.0%), Python (2.5%), Rust (0.3%)
**Status:** Active development

### Project Description

Monty IFC Viewer is a web-based and desktop (Tauri) IFC viewer with construction sequence visualization. Users can drag-and-drop IFC files, select sequencing parameters, and animate the building construction process step by step. It supports element selection, property inspection, and camera controls.

### IfcOpenShell Modules Used

**None.** Monty IFC Viewer does NOT use IfcOpenShell. It uses `web-ifc` (v0.0.66), a JavaScript/WebAssembly library for IFC parsing directly in the browser/desktop environment.

The technology stack is:
- **web-ifc** — IFC file parsing (JavaScript/WASM)
- **Three.js** (v0.128.0) — 3D rendering
- **Solid.js** — Reactive UI framework
- **Tauri** — Desktop application wrapper (Rust backend)

The only Python file in the repository (`scripts/generate-icons.py`) generates application icons using Pillow and has zero IFC-related functionality.

### Key Takeaways

1. **web-ifc is the JavaScript/WASM alternative to IfcOpenShell** for browser-based IFC processing.
2. **No Python or IfcOpenShell dependency exists** in this project — it is a purely client-side viewer.
3. **Relevant contrast:** This project demonstrates the boundary where IfcOpenShell is NOT the right tool — browser-based IFC viewers ALWAYS use web-ifc instead.

---

## INB Template

**Repository:** https://github.com/OpenAEC-Foundation/inb-template
**Language:** Python (with IfcOpenShell + Bonsai/BlenderBIM)
**Status:** Experimental, active development
**IFC Version:** IFC4x3

### Project Description

INB Template (Ifc NL Bouw / IFC Netherlands Construction) is an experimental IFC template and component library for the Dutch construction sector. It provides standardized Dutch building components (walls, floors, roofs, steel profiles), NEN47-compliant hatch patterns, and material libraries — all generated programmatically via Python/IfcOpenShell scripts from spreadsheet data.

### IfcOpenShell Modules Used

| Module | Usage |
|--------|-------|
| `ifcopenshell` (core) | `ifcopenshell.open()` for file I/O, `ifc_file.write()` for output |
| `ifcopenshell.api.run` | Full CRUD operations on IFC entities |
| `ifcopenshell.api.owner` | Owner history and application settings |
| `ifcopenshell.api.material` | Material creation and assignment |
| `ifcopenshell.api.geometry` | Geometry representation creation |
| `ifcopenshell.util.element` | Element utility functions |
| `ifcopenshell.util.representation` | Representation utilities |
| `ifcopenshell.util.unit` | Unit conversion/management |
| `ifcopenshell.util.placement` | `get_local_placement()` for coordinate extraction |
| `ifcopenshell.validate` | IFC model validation |
| `bonsai.bim` | Bonsai (BlenderBIM) integration |
| `bonsai.tool` | Bonsai tool access |

### Code Patterns

#### Pattern 1: Spreadsheet-Driven IFC Generation (CreateLibrary.py)

The primary script reads construction data from an ODS spreadsheet (`Base_Library.ods`) and generates a complete IFC library file:

```python
# Project and library creation
ifcopenshell.api.run("project.create_file")
# Creates IfcProject + IfcProjectLibrary

# Unit assignment
# Uses MILLIMETERS as base unit

# Context setup
# Model, Plan, Body, and Annotation contexts

# Material creation from spreadsheet data
ifcopenshell.api.run("material.add_material", file, name=name, category=category)
ifcopenshell.api.run("material.assign_material", file, product=element, type="IfcMaterialLayerSet")
```

This pattern demonstrates **data-driven IFC generation** — the IFC model structure is NOT hardcoded but derived from external data sources.

#### Pattern 2: Comprehensive `ifcopenshell.api.run()` Category Coverage

The CreateLibrary.py script uses an exceptionally wide range of API categories:

```python
# Entity lifecycle
ifcopenshell.api.run("root.create_entity", file, ifc_class="IfcWall", name=name)

# Material system
ifcopenshell.api.run("material.add_material", file, name=name, category=category)
ifcopenshell.api.run("material.assign_material", file, product=element, type="IfcMaterialLayerSet")

# Geometry representations
ifcopenshell.api.run("geometry.add_wall_representation", file, context=model_body,
                     length=length, height=height, thickness=thickness)
ifcopenshell.api.run("geometry.add_profile_representation", file, context=model_body,
                     profile=profile, depth=length)

# Spatial structure
ifcopenshell.api.run("spatial.assign_container", file, relating_structure=storey, product=element)
ifcopenshell.api.run("aggregate.assign_object", file, relating_object=project, product=site)

# Type system
ifcopenshell.api.run("type.assign_type", file, related_object=instance, relating_type=type_obj)
```

#### Pattern 3: Bonsai/BlenderBIM Integration

The CreateLibrary.py script imports `bonsai.bim` and `bonsai.tool`, indicating it runs INSIDE Blender with the Bonsai add-on active. This means:
- The script can leverage Blender's viewport for visual verification
- `bpy` (Blender Python) is available alongside IfcOpenShell
- Templates are designed for direct use in the Bonsai workflow

#### Pattern 4: Direct Attribute Access for Reading (Pile Scripts)

The pile-related scripts (`Get_IfcPile_Height.py`, `Number_IfcPile_topleft_to_bottomright.py`) demonstrate a simpler, read-focused pattern:

```python
import ifcopenshell
import ifcopenshell.util.placement

ifc_file = ifcopenshell.open(ifc_file_path)
piles = ifc_file.by_type('IfcPile')

for pile in piles:
    matrix = ifcopenshell.util.placement.get_local_placement(pile.ObjectPlacement)
    z_coordinate = matrix[2][3]     # Z from 4x4 transformation matrix
    name = pile.Name                # Direct attribute access
    global_id = pile.GlobalId       # Direct attribute access
    tag = pile.Tag                  # Read/write attribute
```

Key characteristics:
- Uses `ifcopenshell.util.placement.get_local_placement()` for coordinate extraction from 4x4 matrices
- Direct attribute access pattern (`entity.Name`, `entity.GlobalId`, `entity.Tag`)
- `ifc_file.by_type()` for entity querying
- `ifc_file.write()` for saving modifications
- Uses `hasattr()` to check attribute existence before writing

#### Pattern 5: IFC Validation

The IfcOpenHouse.py script imports `ifcopenshell.validate`, indicating the project includes model validation as part of its workflow. This is a best practice rarely seen in other projects.

### Key Takeaways

1. **Data-driven IFC generation is a powerful pattern.** Reading element definitions from spreadsheets and generating IFC programmatically enables non-programmers to define building libraries.
2. **The project demonstrates the WIDEST range of `ifcopenshell.api.run()` categories** of any analyzed repo: root, material, geometry, spatial, aggregate, type, project, owner.
3. **Bonsai integration is native.** The scripts are designed to run inside Blender, bridging the gap between programmatic IFC creation and visual BIM authoring.
4. **Dutch construction standards (NEN47, NL-SFB, STABU) drive the template design**, showing how IfcOpenShell serves as the engine for national BIM standards compliance.
5. **The pile scripts are excellent examples of focused utility scripts** — small, single-purpose IfcOpenShell programs that solve specific construction site problems (pile height extraction, spatial numbering).

---

## AEC Scripts

**Repository:** https://github.com/OpenAEC-Foundation/aec-scripts
**Status:** Does NOT exist (HTTP 404)

### Finding

The repository `https://github.com/OpenAEC-Foundation/aec-scripts` returns a 404 error. It either:
- Has been deleted or renamed
- Is a private repository
- Never existed as a public repository

No IfcOpenShell patterns can be extracted from a non-existent repository.

---

## Cross-Cutting Patterns

### Pattern Summary Table

| Pattern | building.py | INB Template | Monty Viewer |
|---------|------------|--------------|--------------|
| `ifcopenshell.api.run()` | Primary write API | Primary write API | Not used |
| `ifcopenshell.open()` | For loading | For loading | Not used (uses web-ifc) |
| `ifcopenshell.geom` | Shape conversion | Not observed | Not used |
| `ifcopenshell.util.placement` | Not observed | Coordinate extraction | Not used |
| `ifcopenshell.util.element` | Element utilities | Element utilities | Not used |
| `ifcopenshell.validate` | Not observed | Model validation | Not used |
| Bonsai/BlenderBIM integration | Export target | Native script environment | Not applicable |
| Abstraction layer | Yes (CreateIFC/LoadIFC) | No (direct API calls) | Not applicable |
| Data-driven generation | No (programmatic) | Yes (spreadsheet-driven) | Not applicable |

### Common Patterns Across IfcOpenShell-Using Repos

1. **`ifcopenshell.api.run()` is the universal write interface.** Both building.py and INB Template use it as the primary mechanism for creating and modifying IFC entities. The function signature `run(api_category.operation, model, **kwargs)` is the de facto standard.

2. **`ifc_file.by_type()` is the universal read/query interface.** Every project that reads IFC files uses `by_type()` to retrieve entities by their IFC class name.

3. **Direct attribute access (`entity.Name`, `entity.GlobalId`) is standard** for reading entity properties. No project uses getter methods — direct attribute access is ALWAYS the pattern.

4. **Spatial hierarchy follows a consistent creation order:** Project -> Site -> Building -> Storey -> Elements. Both building.py and INB Template follow this exact sequence.

5. **Unit assignment happens immediately after project creation.** Both projects call unit assignment as one of the first operations after creating the IFC file.

### Divergent Approaches

1. **Abstraction vs. Direct API:** building.py wraps IfcOpenShell behind domain-specific classes; INB Template calls `ifcopenshell.api.run()` directly. Both approaches are valid — the choice depends on whether the target audience is developers (abstraction) or BIM specialists (direct API).

2. **Geometry creation:** building.py uses `ifcopenshell.geom.create_shape()` for import and manual entity creation for export. INB Template uses `ifcopenshell.api.run("geometry.add_*_representation")` exclusively. The high-level geometry API is simpler but less flexible.

3. **Browser vs. Desktop:** Monty IFC Viewer demonstrates that IfcOpenShell is NOT suitable for browser-based applications — web-ifc fills that niche. IfcOpenShell is ALWAYS the choice for server-side or desktop Python applications.

---

## Relevance for Claude Skill Package

### Direct Implications

1. **The `ifcopenshell.api.run()` pattern MUST be the primary teaching pattern** in any IfcOpenShell skill documentation. Every real-world project uses it as the backbone of IFC write operations.

2. **Two abstraction levels are valid and should be documented:**
   - **Direct API usage** (as in INB Template) — suitable for BIM automation scripts
   - **Wrapper classes** (as in building.py) — suitable for library/framework development

3. **The spatial hierarchy creation sequence is universal** and MUST be presented as a standard pattern: Project -> Site -> Building -> Storey -> Elements, with unit assignment immediately after project creation.

4. **Data-driven IFC generation** (spreadsheet -> Python -> IFC) is a powerful real-world pattern that SHOULD be documented as an advanced use case.

5. **Utility scripts for construction-specific tasks** (pile height, spatial numbering) demonstrate that IfcOpenShell is used for focused, single-purpose automation — not just full-model creation.

6. **The `ifcopenshell.util.*` modules are essential for practical work:**
   - `util.placement` for coordinate extraction
   - `util.element` for element traversal
   - `util.unit` for unit management
   - `util.representation` for representation access

7. **IFC validation (`ifcopenshell.validate`) is a best practice** that SHOULD be recommended in skill documentation.

### What NOT to Include

- web-ifc patterns (JavaScript, not Python, outside IfcOpenShell scope)
- AEC Scripts patterns (repository does not exist)
- Bonsai-specific `bpy` patterns (separate from pure IfcOpenShell usage)
