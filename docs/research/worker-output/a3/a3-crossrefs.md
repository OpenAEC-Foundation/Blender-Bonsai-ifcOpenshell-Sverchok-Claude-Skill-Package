# Cross-References and Academy Content for Vooronderzoek

## Cross-Reference Insertions

### Section 2: File I/O Operations (line 86)
**Insert after section header (line 86):**

> **See also:** For complete function signatures, parameter tables, return values, and extended code examples covering `ifcopenshell.open()`, `ifcopenshell.file()`, `file.write()`, `file.create_entity()`, and auxiliary file methods (add, remove, to_string, transactions), see [fragments/ifcos-core-operations.md](fragments/ifcos-core-operations.md). For deep research on streaming mode, anti-patterns, and version-specific file I/O behavior, see [topic-research/ifcos-core-operations.md](topic-research/ifcos-core-operations.md) (Section 1: FILE I/O OPERATIONS).

---

### Section 3: ifcopenshell.api.run() Complete Category Overview (line 159)
**Insert after section header (line 159):**

> **See also:** For the complete reference of all 35 API modules with individual function signatures, parameters, return values, and code examples for every function, see [fragments/ifcos-api-categories.md](fragments/ifcos-api-categories.md). That document covers every module from `aggregate` through `void`, including the `classification` module relevant to bSDD integration.

---

### Section 4: Element Traversal (line 351)
**Insert after section header (line 351):**

> **See also:** For comprehensive research on element traversal including `by_type()`, `by_id()`, `by_guid()`, inverse attribute traversal, `ifcopenshell.util.element` query functions (`get_psets`, `get_type`, `get_container`, `get_decomposition`, `get_aggregate`), and anti-patterns to avoid, see [topic-research/ifcos-core-operations.md](topic-research/ifcos-core-operations.md) (Section 2: ELEMENT TRAVERSAL).

---

### Section 5: ifcopenshell.util.* Modules (line 415)
**Insert after section header (line 415):**

> **See also:** For deep research on all 29 utility submodules including `ifcopenshell.util.element`, `ifcopenshell.util.unit`, `ifcopenshell.util.placement`, `ifcopenshell.util.shape`, `ifcopenshell.util.selector`, `ifcopenshell.util.date`, and others with working code examples, parameter documentation, and anti-patterns, see [topic-research/ifcos-core-operations.md](topic-research/ifcos-core-operations.md) (Section 3: ifcopenshell.util.* MODULES).

---

### Section 6: ifcopenshell.geom Module (line 556)
**Insert after section header (line 556):**

> **See also:** For detailed research on the geometry module including `ifcopenshell.geom.settings()` configuration options, `create_shape()` vs `iterator` performance comparison, multi-threaded geometry processing, OpenCASCADE vs CGAL geometry engines, and mesh data extraction patterns, see [topic-research/ifcos-core-operations.md](topic-research/ifcos-core-operations.md) (Section 4: ifcopenshell.geom MODULE).

---

### Section 7: IFC Schema Versions (line 695)
**Insert after section header (line 695):**

> **See also:** For a complete schema version reference including entity counts, ISO standard identifiers, and IfcOpenShell schema identifier mappings, see [fragments/ifcos-schema-versions.md](fragments/ifcos-schema-versions.md) (Section 1: Schema Version Overview). For exhaustive schema comparison research including version timelines, entity count breakdowns, Express schema introspection with `ifcopenshell.schema_by_name()`, and attribute-level differences across all three versions, see [topic-research/ifcos-schema-version-comparison.md](topic-research/ifcos-schema-version-comparison.md) (Sections 1, 7, and 14: Schema Overview, Key Attribute Changes, and Entity Existence Matrix).

---

### Section 8: Entity Hierarchy and Inheritance (line 881)
**Insert after section header (line 881):**

> **See also:** For the full entity hierarchy tree diagrams across IFC2x3, IFC4, and IFC4.3, including spatial structure hierarchy changes, IfcBuiltElement subtypes, and IfcElement direct subtypes, see [fragments/ifcos-schema-versions.md](fragments/ifcos-schema-versions.md) (Section 2: Entity Hierarchy and Inheritance). For deeper research on type objects, IfcElement subtype trees, and spatial structure evolution with IfcOpenShell code examples, see [topic-research/ifcos-schema-version-comparison.md](topic-research/ifcos-schema-version-comparison.md) (Sections 2-5: Entity Hierarchy, Spatial Structure, IfcElement Subtypes, and Type Objects).

---

### Section 9: Relationship Types (line 1072)
**Insert after section header (line 1072):**

> **See also:** For the complete IfcRelationship hierarchy tree, detailed documentation of all relationship variants (IfcRelAggregates, IfcRelContainedInSpatialStructure, IfcRelAssociatesMaterial, IfcRelDefinesByProperties, IfcRelDefinesByType, IfcRelVoidsElement, IfcRelFillsElement), and relationship availability across IFC versions, see [fragments/ifcos-schema-versions.md](fragments/ifcos-schema-versions.md) (Section 4: Relationship Types). For in-depth research with code examples for every relationship type and version-specific behavior, see [topic-research/ifcos-schema-version-comparison.md](topic-research/ifcos-schema-version-comparison.md) (Section 6: Relationship Types with code examples).

---

### Section 10: Common Operations (line 1294)
**Insert after section header (line 1294):**

> **See also:** For additional common operation patterns including creating minimal valid IFC files, extracting data from existing files, modifying properties, and geometry processing, see [fragments/ifcos-errors-performance.md](fragments/ifcos-errors-performance.md) (Section 5: Common Operations). For complete workflow examples covering high-level API and low-level approaches, see [fragments/ifcos-core-operations.md](fragments/ifcos-core-operations.md) (Section 8: Complete Workflow Examples).

---

### Section 11: Common Error Patterns (line 1445)
**Insert after section header (line 1445):**

> **See also:** For the comprehensive error pattern reference covering schema mismatch errors, missing required attributes, incorrect relationship creation, geometry processing failures, unit conversion issues, GUID errors, property set errors, type assignment errors, and file corruption patterns with code examples for each, see [fragments/ifcos-errors-performance.md](fragments/ifcos-errors-performance.md) (Section 1: Common Error Patterns). For deep research on error patterns with expanded examples, edge cases, and version-specific error behavior, see [topic-research/ifcos-errors-performance-research.md](topic-research/ifcos-errors-performance-research.md) (Section 1: Common Error Patterns).

---

### Section 12: Performance Considerations (line 1583)
**Insert after section header (line 1583):**

> **See also:** For detailed performance guidance including memory usage tables for files from 10MB to 2GB+, efficient querying strategies, batch vs individual operation patterns, geometry processing optimization with `ifcopenshell.geom.iterator`, and memory management strategies (chunked processing, thread tuning, SQLite conversion, process-level isolation), see [fragments/ifcos-errors-performance.md](fragments/ifcos-errors-performance.md) (Section 2: Performance for Large Models). For extended performance research including profiling techniques, caching strategies, and community-reported optimization patterns, see [topic-research/ifcos-errors-performance-research.md](topic-research/ifcos-errors-performance-research.md) (Section 2: Performance Considerations for Large Models).

---

### Section 13: AI Common Mistakes (line 1670)
**Insert after section header (line 1670):**

> **See also:** For the complete AI mistake reference covering hallucinated API methods, wrong parameter orders, `api.run()` vs direct entity creation confusion, schema version assumptions, missing owner history, incorrect geometry creation, and a quick-reference table of correct vs hallucinated method names, see [fragments/ifcos-errors-performance.md](fragments/ifcos-errors-performance.md) (Section 3: AI Common Mistakes). For deep research on AI pitfalls with expanded examples and a summary of critical rules for AI code generation, see [topic-research/ifcos-errors-performance-research.md](topic-research/ifcos-errors-performance-research.md) (Sections 3, 5, and 6: AI Common Mistakes, Quick Reference API Names, and Summary of Critical Rules).

---

## Academy Reference Insertions

### Section 1: IfcOpenShell Overview (line 27)
**Insert at:** After the "Key Statistics" bullet list (approximately line 42), before the next subsection.
**Text:**

The [IfcOpenShell Academy](https://academy.ifcopenshell.org/) provides hands-on tutorials for learning IfcOpenShell. The tutorial [Using IfcOpenShell to parse IFC files with Python](https://academy.ifcopenshell.org/posts/using-ifcopenshell-to-parse-ifc-files-with-python/) by Dion Moult covers foundational techniques for reading and processing IFC data structures programmatically, and serves as the recommended starting point for new users. The tutorial [Using the parsing functionality of IfcOpenShell interactively](https://academy.ifcopenshell.org/posts/using-the-parsing-functionality-of-ifcopenshell-interactively/) by Thomas Krijnen demonstrates interactive Python and IfcOpenShell workflows using a REPL environment.

---

### Section 2: File I/O Operations (line 86)
**Insert at:** After the File I/O cross-reference block, before the first subsection.
**Text:**

The [IfcOpenShell Academy](https://academy.ifcopenshell.org/) tutorial [Using IfcOpenShell to parse IFC files with Python](https://academy.ifcopenshell.org/posts/using-ifcopenshell-to-parse-ifc-files-with-python/) demonstrates the complete file opening and parsing workflow. For zero-setup experimentation, the tutorial [Using IfcOpenShell in Binder Notebooks with web-viewer and graph visualization](https://academy.ifcopenshell.org/posts/using-ifcopenshell-in-binder-notebook-with-viewer-and-graph/) by Jakob Beetz provides interactive Jupyter notebooks that run directly in the browser without local installation.

---

### Section 6: ifcopenshell.geom Module (line 556)
**Insert at:** After the geom cross-reference block, before the first subsection.
**Text:**

The [IfcOpenShell Academy](https://academy.ifcopenshell.org/) provides several geometry-focused tutorials. [Using IfcOpenShell and pythonOCC to construct new geometry](https://academy.ifcopenshell.org/posts/using-ifcopenshell-and-pythonocc-to-construct-new-geometry/) by Thomas Krijnen demonstrates constructing geometry from bounding elements and calculating volumetric properties. [Using IfcOpenShell and pythonOCC to generate cross sections directly from an IFC file](https://academy.ifcopenshell.org/posts/using-ifcopenshell-and-pythonocc-to-generate-cross-sections-directly-from-an-ifc-file/) by Emiel van Strien covers deriving section information for applications like 3D concrete printing. [Read geometry as Boundary Representation in FreeCAD](https://academy.ifcopenshell.org/posts/read-geometry-as-boundary-representation-in-freecad/) and [Read IFC geometry as triangle meshes in FreeCAD](https://academy.ifcopenshell.org/posts/read-ifc-geometry-as-triangle-meshes-in-freecad/) by Cyril Waechter cover BREP and mesh geometry processing respectively. [Understanding placements in IFC using IfcOpenShell and FreeCAD](https://academy.ifcopenshell.org/posts/understanding-placements-in-ifc-using-ifcopenshell-and-freecad/) by Cyril Waechter provides essential guidance on coordinate systems and placement handling.

---

### Section 7: IFC Schema Versions (line 695)
**Insert at:** After the schema cross-reference block, before the first subsection.
**Text:**

The [IfcOpenShell Academy](https://academy.ifcopenshell.org/) tutorial [Using IfcOpenShell and C++ to generate Alignments through the IFC 4x1 schema](https://academy.ifcopenshell.org/posts/using-ifcopenshell-and-c%2B%2B-to-generate-alignments-through-the-ifc-4x1-schema/) by Francisco Navarrete Mandly demonstrates schema code generation for custom IFC versions, which is relevant for understanding how IfcOpenShell handles schema differences internally.

---

### Section 10: Common Operations (line 1294)
**Insert at:** After the common operations cross-reference block, before the first subsection.
**Text:**

The [IfcOpenShell Academy](https://academy.ifcopenshell.org/) tutorial [Creating a simple wall with property set and quantity information](https://academy.ifcopenshell.org/posts/creating-a-simple-wall-with-property-set-and-quantity-information/) by Kianwee Chen provides a step-by-step walkthrough of procedural IFC wall creation including geometric extrusions, property sets, and quantity data. The [IfcOpenShell Optimizer tutorial](https://academy.ifcopenshell.org/posts/ifcopenshell-optimizer-tutorial/) by Johan Luttun covers reducing IFC file size by eliminating duplicate entity instances. The [Calculate Differences of IFC files with Hashing](https://academy.ifcopenshell.org/posts/calculate-differences-of-ifc-files-with-hashing/) tutorial by Johan Luttun demonstrates comparing successive IFC exports to identify added or removed entity instances.

---

### Section 12: Performance Considerations (line 1583)
**Insert at:** After the performance cross-reference block, before the first subsection.
**Text:**

The [IfcOpenShell Academy](https://academy.ifcopenshell.org/) tutorial [IfcOpenShell Optimizer tutorial](https://academy.ifcopenshell.org/posts/ifcopenshell-optimizer-tutorial/) by Johan Luttun covers file size optimization by eliminating duplicate entity instances, which directly impacts memory usage and loading performance for large models.

---

## Enhanced Sources Section

**Add the following Academy-specific URLs to Section 14 (Sources), after the existing Academy entries:**

- [Using IfcOpenShell to parse IFC files with Python (Academy)](https://academy.ifcopenshell.org/posts/using-ifcopenshell-to-parse-ifc-files-with-python/) - Foundational parsing tutorial by Dion Moult
- [Using IfcOpenShell in Binder Notebooks (Academy)](https://academy.ifcopenshell.org/posts/using-ifcopenshell-in-binder-notebook-with-viewer-and-graph/) - Zero-setup Jupyter notebooks with viewer by Jakob Beetz
- [Using the parsing functionality interactively (Academy)](https://academy.ifcopenshell.org/posts/using-the-parsing-functionality-of-ifcopenshell-interactively/) - Interactive REPL tutorial by Thomas Krijnen
- [IfcOpenShell Optimizer tutorial (Academy)](https://academy.ifcopenshell.org/posts/ifcopenshell-optimizer-tutorial/) - File size reduction by Johan Luttun
- [Calculate Differences of IFC files with Hashing (Academy)](https://academy.ifcopenshell.org/posts/calculate-differences-of-ifc-files-with-hashing/) - IFC diff comparison by Johan Luttun
- [Understanding placements in IFC (Academy)](https://academy.ifcopenshell.org/posts/understanding-placements-in-ifc-using-ifcopenshell-and-freecad/) - Coordinate systems and placement handling by Cyril Waechter
- [Using IfcOpenShell and pythonOCC to construct new geometry (Academy)](https://academy.ifcopenshell.org/posts/using-ifcopenshell-and-pythonocc-to-construct-new-geometry/) - Geometry construction by Thomas Krijnen
- [Using IfcOpenShell and pythonOCC to generate cross sections (Academy)](https://academy.ifcopenshell.org/posts/using-ifcopenshell-and-pythonocc-to-generate-cross-sections-directly-from-an-ifc-file/) - Section generation by Emiel van Strien
- [Read geometry as Boundary Representation in FreeCAD (Academy)](https://academy.ifcopenshell.org/posts/read-geometry-as-boundary-representation-in-freecad/) - BREP geometry processing by Cyril Waechter
- [Read IFC geometry as triangle meshes in FreeCAD (Academy)](https://academy.ifcopenshell.org/posts/read-ifc-geometry-as-triangle-meshes-in-freecad/) - Mesh geometry processing by Cyril Waechter
- [Using IfcOpenShell and C++ for IFC 4x1 Alignments (Academy)](https://academy.ifcopenshell.org/posts/using-ifcopenshell-and-c%2B%2B-to-generate-alignments-through-the-ifc-4x1-schema/) - Schema code generation by Francisco Navarrete Mandly
- [Importing Part module in IfcOpenShell-python (Academy)](https://academy.ifcopenshell.org/posts/Importing%20Part%20module%20in%20ifcOpenShell-python/) - FreeCAD Part module for BREP extraction by Devang Chauhan

---

## Cross-Reference Summary Table

| Main Section | Line | Supporting Files Referenced |
|---|---|---|
| 2. File I/O Operations | 86 | fragments/ifcos-core-operations.md, topic-research/ifcos-core-operations.md |
| 3. API Overview | 159 | fragments/ifcos-api-categories.md |
| 4. Element Traversal | 351 | topic-research/ifcos-core-operations.md |
| 5. util Modules | 415 | topic-research/ifcos-core-operations.md |
| 6. geom Module | 556 | topic-research/ifcos-core-operations.md |
| 7. Schema Versions | 695 | fragments/ifcos-schema-versions.md, topic-research/ifcos-schema-version-comparison.md |
| 8. Entity Hierarchy | 881 | fragments/ifcos-schema-versions.md, topic-research/ifcos-schema-version-comparison.md |
| 9. Relationship Types | 1072 | fragments/ifcos-schema-versions.md, topic-research/ifcos-schema-version-comparison.md |
| 10. Common Operations | 1294 | fragments/ifcos-errors-performance.md, fragments/ifcos-core-operations.md |
| 11. Error Patterns | 1445 | fragments/ifcos-errors-performance.md, topic-research/ifcos-errors-performance-research.md |
| 12. Performance | 1583 | fragments/ifcos-errors-performance.md, topic-research/ifcos-errors-performance-research.md |
| 13. AI Mistakes | 1670 | fragments/ifcos-errors-performance.md, topic-research/ifcos-errors-performance-research.md |

## Academy Reference Summary Table

| Main Section | Line | Academy Tutorials Referenced |
|---|---|---|
| 1. Overview | 27 | Parsing IFC files (Moult), Interactive parsing (Krijnen) |
| 2. File I/O | 86 | Parsing IFC files (Moult), Binder Notebooks (Beetz) |
| 6. geom Module | 556 | pythonOCC geometry (Krijnen), Cross sections (van Strien), BREP (Waechter), Meshes (Waechter), Placements (Waechter) |
| 7. Schema Versions | 695 | IFC 4x1 Alignments (Navarrete Mandly) |
| 10. Common Operations | 1294 | Simple wall creation (Chen), Optimizer (Luttun), IFC diff hashing (Luttun) |
| 12. Performance | 1583 | Optimizer (Luttun) |

## Verification Notes

All cross-reference mappings were verified by reading the actual content of each supporting file:

1. **fragments/ifcos-api-categories.md** (1170 lines): Confirmed contains all 35 API modules with function signatures. Sections: Overview, Complete List, Detailed Module Documentation (aggregate through void), Additional Categories, Complete Minimal IFC File Example.

2. **fragments/ifcos-core-operations.md** (978 lines): Confirmed covers `ifcopenshell.open()`, `ifcopenshell.file()`, `file.write()`, `file.create_entity()`, high-level API entity creation, high-level file creation, auxiliary file methods, and complete workflow examples.

3. **fragments/ifcos-errors-performance.md** (1108 lines): Confirmed covers common error patterns (9 categories), performance for large models (6 strategies), AI common mistakes (7 categories), installation (5 methods), and common operations.

4. **fragments/ifcos-schema-versions.md** (833 lines): Confirmed covers schema version overview, entity hierarchy and inheritance trees, IFC2x3/IFC4/IFC4.3 comparison tables, relationship types hierarchy, migration patterns, and version comparison tables.

5. **topic-research/ifcos-core-operations.md** (1473 lines): Confirmed covers File I/O, Element Traversal, util modules (29 submodules), geom module, supplementary API reference, and version differences.

6. **topic-research/ifcos-errors-performance-research.md** (1518 lines): Confirmed covers error patterns, performance considerations, AI common mistakes, installation, quick reference API names table, and summary of critical rules.

7. **topic-research/ifcos-schema-version-comparison.md** (1484 lines): Confirmed covers schema overview, entity hierarchy, spatial structure, element subtypes, type objects, relationship types with code examples, attribute changes, new IFC4X3 entities, geometry changes, migration patterns, API version handling, schema introspection, pitfalls, and entity existence matrix.

All 13 Academy tutorials from https://academy.ifcopenshell.org/ were cataloged and mapped to relevant sections based on their actual content.
