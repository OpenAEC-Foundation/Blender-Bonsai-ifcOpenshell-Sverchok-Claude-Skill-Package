# IfcOpenShell Skill Inventory Briefing

**Date**: 2026-03-06
**Status**: COMPLETE
**Author**: interpret-ifcos agent
**Purpose**: Condensed briefing for masterplan agent — definitive skill list, recommendations, patterns

---

## 1. Definitive Skill List (18 Skills)

### 1.1 Syntax Skills (4)

#### ifcos-syntax-fileio
- **Category**: syntax
- **Scope**: File I/O operations — open, create, write, serialize IFC files
- **Key API surface**: `ifcopenshell.open()`, `ifcopenshell.file()`, `ifcopenshell.api.project.create_file()`, `file.write()`, `file.to_string()`, `file.add()`, `file.remove()`, transactions (`begin_transaction`/`end_transaction`/`undo`/`redo`)
- **Source sections**: vooronderzoek §2 (L90-165), fragments/core-operations §1-3 (L22-245), topic-research/core-operations §1 (L10-290)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html, docs.ifcopenshell.org/ifcopenshell-python/hello_world.html
- **Schema sensitivity**: All (IFC2X3/IFC4/IFC4X3) — schema string in `file()` constructor
- **Dependencies**: None
- **Estimated complexity**: S

#### ifcos-syntax-api
- **Category**: syntax
- **Scope**: `ifcopenshell.api.run()` / direct module calls — all 30+ API modules, parameter conventions, invocation patterns
- **Key API surface**: `ifcopenshell.api.run("module.function", model, **kwargs)` and direct `ifcopenshell.api.module.function(model, ...)`. Modules: root, spatial, aggregate, geometry, type, pset, material, context, unit, owner, classification, cost, sequence, system, group, void, boundary, structural, document, drawing, nest, layer, profile, style, constraint, resource, feature, alignment, georeference, project, attribute, pset_template
- **Source sections**: vooronderzoek §3 (L167-358), fragments/api-categories full file (L1-1170), topic-research/errors §5 (L1393-1498)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/index.html, docs.ifcopenshell.org/autoapi/ifcopenshell/api/root/index.html (and all sub-module URLs)
- **Schema sensitivity**: Medium — most modules work across schemas; `api.run()` handles schema differences internally
- **Dependencies**: ifcos-syntax-fileio
- **Estimated complexity**: M

#### ifcos-syntax-elements
- **Category**: syntax
- **Scope**: Element traversal and querying — by_type, by_id, by_guid, inverse references, entity attributes, get_info(), is_a(), GUID utilities
- **Key API surface**: `model.by_type()`, `model.by_id()`, `model.by_guid()`, `model.get_inverse()`, `model.traverse()`, `entity.get_info()`, `entity.is_a()`, `entity.id()`, `ifcopenshell.guid.new()`, `ifcopenshell.guid.expand()`, `ifcopenshell.guid.compress()`
- **Source sections**: vooronderzoek §4 (L361-424), §8 (L903-1093), topic-research/core-operations §2 (L292-629)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html
- **Schema sensitivity**: Low — query API is schema-agnostic; entity class names differ per schema
- **Dependencies**: ifcos-syntax-fileio
- **Estimated complexity**: S

#### ifcos-syntax-util
- **Category**: syntax
- **Scope**: All `ifcopenshell.util.*` modules — element, unit, placement, selector, date, shape_builder, schema, classification, geolocation, representation, attribute, cost, sequence, system
- **Key API surface**: `util.element.get_psets()`, `get_type()`, `get_container()`, `get_material()`, `get_decomposition()`, `get_aggregate()`; `util.unit.calculate_unit_scale()`; `util.placement.get_local_placement()`; `util.selector.filter_elements()`; `util.date.ifc2datetime()`/`datetime2ifc()`; `util.shape_builder.ShapeBuilder`; `util.schema.get_entity_attributes()`
- **Source sections**: vooronderzoek §5 (L427-565), topic-research/core-operations §3 (L631-1086)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/util/index.html
- **Schema sensitivity**: Medium — `util.element` abstracts schema differences (recommended approach)
- **Dependencies**: ifcos-syntax-fileio
- **Estimated complexity**: M

### 1.2 Implementation Skills (8)

#### ifcos-impl-creation
- **Category**: impl
- **Scope**: Creating valid IFC files from scratch — minimal valid file pattern, spatial hierarchy setup (Project→Site→Building→Storey), element creation with geometry, property assignment
- **Key API surface**: `root.create_entity`, `unit.assign_unit`, `context.add_context`, `aggregate.assign_object`, `spatial.assign_container`, `geometry.add_wall_representation`, `geometry.assign_representation`, `geometry.edit_object_placement`, `pset.add_pset`, `pset.edit_pset`
- **Source sections**: vooronderzoek §10 (L1326-1471), fragments/api-categories §Complete Example (L1109-1154)
- **SOURCES.md URLs**: docs.ifcopenshell.org/ifcopenshell-python/code_examples.html, academy.ifcopenshell.org/posts/creating-a-simple-wall-with-property-set-and-quantity-information/
- **Schema sensitivity**: HIGH — OwnerHistory mandatory in IFC2X3; StandardCase entities differ per schema
- **Dependencies**: ifcos-syntax-fileio, ifcos-syntax-api
- **Estimated complexity**: M

#### ifcos-impl-geometry
- **Category**: impl
- **Scope**: Geometry creation and processing — wall/slab/profile/mesh representations, geometry settings, create_shape, iterator, BRep vs tessellation, ShapeBuilder
- **Key API surface**: `ifcopenshell.geom.settings()`, `ifcopenshell.geom.create_shape()`, `ifcopenshell.geom.iterator()`, `geometry.add_wall_representation`, `geometry.add_mesh_representation`, `geometry.add_profile_representation`, `geometry.add_slab_representation`, `geometry.add_door_representation`, `geometry.add_window_representation`, `geometry.add_boolean`
- **Source sections**: vooronderzoek §6 (L570-710), fragments/api-categories §3 geometry (L241-425), topic-research/core-operations §4 (L1088-1363)
- **SOURCES.md URLs**: docs.ifcopenshell.org/ifcopenshell-python/geometry_settings.html, geometry_processing.html, geometry_creation.html
- **Schema sensitivity**: Low — geometry API is mostly schema-agnostic
- **Dependencies**: ifcos-syntax-api, ifcos-impl-creation
- **Estimated complexity**: L

#### ifcos-impl-materials
- **Category**: impl
- **Scope**: Material assignment — single materials, layer sets (walls/slabs), profile sets (beams/columns), constituent sets (doors/windows), material lists
- **Key API surface**: `material.add_material`, `material.add_material_set`, `material.assign_material`, `material.add_layer`, `material.edit_layer`, `material.add_constituent`, `material.add_profile`, `material.edit_layer_usage`, `material.edit_profile_usage`
- **Source sections**: vooronderzoek §3 material (L312-341), fragments/api-categories §6 (L639-752)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/material/index.html
- **Schema sensitivity**: Medium — `IfcMaterialConstituentSet` is IFC4+; `IfcMaterialList` is legacy IFC2X3
- **Dependencies**: ifcos-syntax-api
- **Estimated complexity**: M

#### ifcos-impl-relationships
- **Category**: impl
- **Scope**: IFC relationship types — aggregation, containment, type assignment, property assignment, material association, voids/fillings, nesting, grouping
- **Key API surface**: `aggregate.assign_object`, `spatial.assign_container`, `spatial.reference_structure`, `type.assign_type`, `pset.add_pset`/`assign_pset`, `void.add_opening`, `void.add_filling`, `nest.assign_object`, `group.add_group`/`assign_group`
- **Source sections**: vooronderzoek §9 (L1096-1315), fragments/schema-versions §4 (L299-435), topic-research/schema §6 (L460-885)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/spatial/index.html, .../aggregate/index.html, .../type/index.html
- **Schema sensitivity**: HIGH — `IfcRelDecomposes` split in IFC4; `IfcRelPositions` new in IFC4X3
- **Dependencies**: ifcos-syntax-api, ifcos-syntax-elements
- **Estimated complexity**: M

#### ifcos-impl-cost
- **Category**: impl
- **Scope**: Cost management — cost schedules, cost items (WBS), cost values (direct/unit/formula), parametric quantity linking
- **Key API surface**: `ifcopenshell.api.cost.*` (20 functions), `ifcopenshell.util.cost.*` (12 functions)
- **Source sections**: supplementary-ifcos-gaps §1 (L23-209)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/cost/index.html, .../util/cost/index.html
- **Schema sensitivity**: Low — supported across all schemas
- **Dependencies**: ifcos-syntax-api
- **Estimated complexity**: M

#### ifcos-impl-sequence
- **Category**: impl
- **Scope**: 4D scheduling — work schedules, tasks (WBS), task time/duration, dependencies (FS/SS/FF/SF), work calendars, task-product relationships, cascade_schedule
- **Key API surface**: `ifcopenshell.api.sequence.*` (40 functions), `ifcopenshell.util.sequence.*` (16 functions)
- **Source sections**: supplementary-ifcos-gaps §2 (L211-491)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/sequence/index.html, .../util/sequence/index.html
- **Schema sensitivity**: Medium — IFC2X3 uses `IfcDateAndTime` entities; IFC4+ uses ISO 8601 strings
- **Dependencies**: ifcos-syntax-api
- **Estimated complexity**: L

#### ifcos-impl-mep
- **Category**: impl
- **Scope**: MEP systems — distribution systems, port-based connectivity, flow segments/fittings/terminals, system traversal
- **Key API surface**: `ifcopenshell.api.system.*` (12 functions), `ifcopenshell.util.system.*` (8 functions)
- **Source sections**: supplementary-ifcos-gaps §3 (L493-668)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/system/index.html, .../util/system/index.html
- **Schema sensitivity**: Medium — IFC2X3 uses `IfcSystem`; IFC4+ uses `IfcDistributionSystem`
- **Dependencies**: ifcos-syntax-api, ifcos-impl-relationships
- **Estimated complexity**: M

#### ifcos-impl-profiles
- **Category**: impl
- **Scope**: Cross-section profiles — 14 parametric profile types (I-shape, rectangle, circle, etc.), arbitrary profiles, profiles in geometry
- **Key API surface**: `ifcopenshell.api.profile.*` (6 functions): `add_parameterised_profile`, `add_arbitrary_profile`, `add_arbitrary_profile_with_voids`, `edit_profile`, `remove_profile`
- **Source sections**: supplementary-ifcos-gaps §5 (L746-908)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/api/profile/index.html
- **Schema sensitivity**: Low — profiles work across all schemas
- **Dependencies**: ifcos-syntax-api, ifcos-impl-geometry
- **Estimated complexity**: S

### 1.3 Error/Performance Skills (3)

#### ifcos-errors-schema
- **Category**: errors
- **Scope**: Schema version pitfalls — entity availability per schema, attribute changes, StandardCase removal in IFC4X3, OwnerHistory requirements, entity renames, migration patterns with ifcpatch
- **Key API surface**: `model.schema`, `ifcopenshell.ifcopenshell_wrapper.schema_by_name()`, `ifcpatch.execute()`, schema introspection helpers
- **Source sections**: vooronderzoek §7 (L713-900), §11 errors 1-2 (L1479-1503), fragments/schema-versions full file (L1-834), topic-research/schema full file (L1-1485)
- **SOURCES.md URLs**: technical.buildingsmart.org/standards/ifc/ifc-schema-specifications/, ifc43-docs.standards.buildingsmart.org/, docs.ifcopenshell.org/ifcpatch.html
- **Schema sensitivity**: THIS IS the schema skill — covers all three schemas
- **Dependencies**: ifcos-syntax-fileio
- **Estimated complexity**: L

#### ifcos-errors-patterns
- **Category**: errors
- **Scope**: Common error patterns and anti-patterns — 10 documented errors (relationship errors, unit conversion, GUID errors, pset errors, geometry failures, file corruption), correct vs incorrect code pairs
- **Key API surface**: Diagnostic: `model.schema`, `util.unit.calculate_unit_scale()`, `util.element.get_psets()`, `ifcopenshell.guid.new()`. Recovery: `root.remove_product` (not `model.remove`), batch `assign_container` (not loops)
- **Source sections**: vooronderzoek §11 (L1475-1611), fragments/errors-performance §1 (L18-451), topic-research/errors §1 (L9-639)
- **SOURCES.md URLs**: docs.ifcopenshell.org/ifcopenshell-python.html
- **Schema sensitivity**: Medium — several errors are schema-specific
- **Dependencies**: ifcos-syntax-api, ifcos-syntax-elements
- **Estimated complexity**: M

#### ifcos-errors-performance
- **Category**: errors
- **Scope**: Performance optimization for large models — memory usage patterns (RAM = 10-20x file size), batch vs individual operations, iterator vs create_shape (5-10x speedup), memory management for 100MB+ files, threading constraints
- **Key API surface**: `ifcopenshell.geom.iterator()` with multiprocessing, `model.by_type(include_subtypes=False)`, batch `assign_container(products=[...])`, `gc.collect()`, `model.garbage_collect()`
- **Source sections**: vooronderzoek §12 (L1615-1703), fragments/errors-performance §2 (L454-665), topic-research/errors §2 (L643-944)
- **SOURCES.md URLs**: docs.ifcopenshell.org/ifcopenshell-python/geometry_processing.html
- **Schema sensitivity**: Low
- **Dependencies**: ifcos-syntax-fileio, ifcos-impl-geometry
- **Estimated complexity**: M

### 1.4 Core Skills (2)

#### ifcos-core-concepts
- **Category**: core
- **Scope**: IFC fundamentals — entity hierarchy (IfcRoot→IfcObjectDefinition→IfcObject→IfcProduct→IfcElement), spatial structure, ownership model, placement system, representation system, relationship model
- **Key API surface**: `entity.is_a()`, inheritance chains, `IfcProject/IfcSite/IfcBuilding/IfcBuildingStorey` hierarchy, `IfcFacility` (IFC4X3)
- **Source sections**: vooronderzoek §8 (L903-1093), fragments/schema-versions §2 (L45-220), topic-research/schema §2-4 (L50-421)
- **SOURCES.md URLs**: ifc43-docs.standards.buildingsmart.org/, technical.buildingsmart.org/standards/ifc/
- **Schema sensitivity**: HIGH — entity hierarchy differs significantly between versions
- **Dependencies**: None
- **Estimated complexity**: M

#### ifcos-core-runtime
- **Category**: core
- **Scope**: Python runtime quirks — C++ binding behavior, entity identity (`is` vs `==`), entity invalidation after removal, `by_type()` returns tuple, thread safety (single-threaded writes only), memory management, attribute access patterns (PascalCase), installation
- **Key API surface**: `ifcopenshell.entity_instance`, `model.by_type()` index behavior, `model.remove()` invalidation, `model.schema`
- **Source sections**: supplementary-ifcos-gaps §7 (L1115-1360), topic-research/errors §3-4 (L949-1390)
- **SOURCES.md URLs**: docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html, pypi.org/project/ifcopenshell/
- **Schema sensitivity**: Low — runtime behavior is schema-agnostic
- **Dependencies**: ifcos-syntax-fileio
- **Estimated complexity**: M

### 1.5 Agent Skills (1)

#### ifcos-code-validator
- **Category**: agents
- **Scope**: Validate IfcOpenShell scripts — check for hallucinated APIs, wrong parameter names, schema compatibility, missing spatial hierarchy, missing context, missing unit setup, correct use of api.run() vs create_entity
- **Key API surface**: All — validation checklist references all API surfaces
- **Source sections**: vooronderzoek §13 AI Mistakes (L1706-1735), fragments/errors-performance §3 (L668-828), topic-research/errors §3 (L949-1257), §5 Quick Reference (L1393-1498), §6 Critical Rules (L1502-1519)
- **SOURCES.md URLs**: All IfcOpenShell URLs
- **Schema sensitivity**: HIGH — must validate schema-appropriate entity/attribute usage
- **Dependencies**: All other ifcos skills
- **Estimated complexity**: L

---

## 2. Merge/Split/Add/Remove Recommendations

### vs. Raw Masterplan (13 skills proposed)

| Raw Masterplan Skill | Recommendation | Rationale |
|---|---|---|
| ifcos-syntax-fileio | KEEP | Well-scoped, clear API boundary |
| ifcos-syntax-api | KEEP | Essential — 30+ modules need a reference skill |
| ifcos-syntax-elements | KEEP | Clear scope: traversal + querying |
| ifcos-syntax-geometry | MERGE → ifcos-impl-geometry | Geometry syntax is inseparable from geometry implementation; geom settings + create_shape + iterator belong with geometry creation |
| ifcos-syntax-util | KEEP | 17+ util modules warrant dedicated syntax skill |
| ifcos-impl-creation | KEEP | The "hello world" workflow — critical standalone skill |
| ifcos-impl-extraction | MERGE → ifcos-syntax-elements + ifcos-syntax-util | Extraction is just traversal + utility functions; no unique API surface |
| ifcos-impl-modification | MERGE → ifcos-impl-creation | Modification uses same API as creation (edit_pset, edit_attributes); not enough unique content |
| ifcos-errors-schema | KEEP | Critical — most complex error domain |
| ifcos-errors-geometry | MERGE → ifcos-errors-patterns | Geometry errors are a subset of the broader error pattern collection |
| ifcos-core-schemas | MERGE → ifcos-errors-schema + ifcos-core-concepts | Schema overview splits into two concerns: entity hierarchy (concepts) and version differences (errors-schema) |
| ifcos-core-concepts | KEEP | IFC fundamentals need a dedicated skill |
| ifcos-code-validator | KEEP | AI hallucination prevention is critical |

### ADD (5 new skills from gap analysis)

| New Skill | Rationale |
|---|---|
| ifcos-impl-cost | 20 API functions + 12 util functions; distinct BIM domain (supplementary-gaps §1) |
| ifcos-impl-sequence | 40 API functions; largest API module; critical for 4D BIM (supplementary-gaps §2) |
| ifcos-impl-mep | 12 API functions; distinct domain (HVAC, plumbing, electrical) (supplementary-gaps §3) |
| ifcos-impl-profiles | 6 API functions; supports geometry creation for structural elements (supplementary-gaps §5) |
| ifcos-core-runtime | Python/C++ binding quirks, thread safety, memory — cross-cutting concern (supplementary-gaps §7) |

### REMOVE/DEFER (3 raw masterplan skills merged away)

| Removed | Merged Into |
|---|---|
| ifcos-syntax-geometry | ifcos-impl-geometry |
| ifcos-impl-extraction | ifcos-syntax-elements + ifcos-syntax-util |
| ifcos-impl-modification | ifcos-impl-creation |

### CONSIDER BUT DEFER (3 potential skills with thin API surface)

| Potential Skill | Why Defer |
|---|---|
| ifcos-impl-drawing | Only 3 API functions; full drawing lives in Bonsai, not IfcOpenShell |
| ifcos-impl-validation | `ifcopenshell.validate` is useful but small; could be a section in ifcos-core-runtime |
| ifcos-impl-georeference | 5 API functions; include as section in ifcos-impl-creation or separate later if demand warrants |
| ifcos-impl-bsdd | bSDD integration is specialized; could be section in a classification skill |

---

## 3. Key Patterns Summary

### Top 10 IfcOpenShell Python Patterns for AEC

1. **Minimal Valid IFC File** (7-step bootstrap):
   ```
   create_file → create IfcProject → assign_unit → add_context (Model/Body/MODEL_VIEW)
   → spatial hierarchy (Site→Building→Storey) → create element + geometry → assign_container
   ```
   Source: vooronderzoek L1326-1384

2. **Schema-Aware Entity Creation**:
   ```python
   if model.schema == "IFC4X3":
       cls = "IfcBuiltElement"
   else:
       cls = "IfcBuildingElement"
   ```
   Source: vooronderzoek L843-870

3. **Batch Spatial Assignment** (NOT per-element loops):
   ```python
   ifcopenshell.api.spatial.assign_container(model, relating_structure=storey, products=walls)
   ```
   Source: vooronderzoek L1693-1695

4. **Unit Scale Conversion** (ALWAYS before using coordinates):
   ```python
   scale = ifcopenshell.util.unit.calculate_unit_scale(model)
   x_meters = raw_x * scale
   ```
   Source: vooronderzoek L1524-1529

5. **Geometry Iterator for Bulk Processing** (5-10x faster than create_shape loop):
   ```python
   iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
   if iterator.initialize():
       while True:
           shape = iterator.get()
           # process shape
           if not iterator.next(): break
   ```
   Source: topic-research/core-operations L1258-1318

6. **Property Set Check-Before-Create** (prevent duplicates):
   ```python
   existing = ifcopenshell.util.element.get_psets(element)
   if "Pset_WallCommon" not in existing:
       pset = ifcopenshell.api.pset.add_pset(model, product=element, name="Pset_WallCommon")
   ```
   Source: vooronderzoek L1588-1595

7. **CSS-Like Element Selection** (modern API):
   ```python
   walls = ifcopenshell.util.selector.filter_elements(model, 'IfcWall, /Pset_WallCommon/.IsExternal=True')
   ```
   Source: topic-research/core-operations L832-882

8. **Safe Product Removal** (ALWAYS use API, never model.remove):
   ```python
   ifcopenshell.api.root.remove_product(model, product=wall)  # cleans up all relationships
   ```
   Source: vooronderzoek L1556-1566

9. **Type-Occurrence Pattern** (define once, reuse):
   ```python
   wall_type = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWallType", name="Standard")
   ifcopenshell.api.type.assign_type(model, related_objects=[wall], relating_type=wall_type)
   ```
   Source: fragments/api-categories L429-503

10. **Data-Driven IFC Generation** (spreadsheet → IFC):
    ```
    Read ODS/CSV → iterate rows → api.run() per element/material/pset → write IFC
    ```
    Source: vooronderzoek L1967-1985

### Top 5 Schema Version Pitfalls

1. **IfcWallStandardCase in IFC4X3**: Does not exist. Use `IfcWall` with `PredefinedType`. Causes `RuntimeError` at runtime.
   Source: vooronderzoek L1479-1491, fragments/schema-versions L293-294

2. **OwnerHistory mandatory in IFC2X3**: Must create `IfcPerson`, `IfcOrganization`, `IfcPersonAndOrganization`, `IfcApplication` before any entity. IFC4+ makes it optional.
   Source: fragments/errors-performance L62-108

3. **IfcBuildingElement → IfcBuiltElement rename in IFC4X3**: Code using old name silently fails or errors.
   Source: fragments/schema-versions L474-475

4. **Date/time entities removed in IFC4**: `IfcCalendarDate`, `IfcLocalTime`, `IfcDateAndTime` → ISO 8601 strings. Affects scheduling and cost modules.
   Source: fragments/schema-versions L282-283

5. **IfcFacility hierarchy insertion in IFC4X3**: `IfcBuilding` is now a subtype of `IfcFacility`. Spatial hierarchy traversal code that counts levels will break for infrastructure projects.
   Source: fragments/schema-versions L492-494

### Top 5 Anti-Patterns (from Errors Research)

1. **Direct relationship assignment**: `wall.ContainedInStructure = storey` — NEVER works. Inverse attributes are read-only. ALWAYS use `spatial.assign_container()`.
   Source: vooronderzoek L1532-1543

2. **model.remove() without cleanup**: Leaves dangling references in all relationships. ALWAYS use `root.remove_product()`.
   Source: vooronderzoek L1556-1566, fragments/errors-performance L407-451

3. **create_shape() in a loop**: 5-10x slower than `geom.iterator` for 100+ elements. Memory also multiplies.
   Source: fragments/errors-performance L581-599

4. **Hallucinated API methods**: `model.get_all_walls()`, `wall.get_properties()`, `ifcopenshell.api.run("wall.create")` — NONE of these exist. Claude must use documented API only.
   Source: fragments/errors-performance L671-692, topic-research/errors L951-1021

5. **Missing geometric context**: Creating geometry without `context.add_context("Model", "Body", "MODEL_VIEW")` first produces orphaned representations invisible in viewers.
   Source: topic-research/errors L1193-1255

---

## 4. Source Coverage Map

| Skill | Research Docs | SOURCES.md URLs |
|---|---|---|
| ifcos-syntax-fileio | vooronderzoek §2, frag/core-ops §1-3, topic/core-ops §1 | docs.ifcopenshell.org/.../file/, .../hello_world.html |
| ifcos-syntax-api | vooronderzoek §3, frag/api-categories (full), topic/errors §5 | docs.ifcopenshell.org/.../api/index.html + all module URLs |
| ifcos-syntax-elements | vooronderzoek §4,8, topic/core-ops §2 | docs.ifcopenshell.org/.../entity_instance/ |
| ifcos-syntax-util | vooronderzoek §5, topic/core-ops §3 | docs.ifcopenshell.org/.../util/index.html |
| ifcos-impl-creation | vooronderzoek §10, frag/api-categories §Example | docs.ifcopenshell.org/.../code_examples.html, academy...wall |
| ifcos-impl-geometry | vooronderzoek §6, frag/api-categories §3, topic/core-ops §4 | docs.ifcopenshell.org/.../geometry_settings.html, geometry_processing.html, geometry_creation.html |
| ifcos-impl-materials | vooronderzoek §3 material, frag/api-categories §6 | docs.ifcopenshell.org/.../api/material/ |
| ifcos-impl-relationships | vooronderzoek §9, frag/schema-versions §4, topic/schema §6 | docs.ifcopenshell.org/.../api/spatial/, .../aggregate/, .../type/ |
| ifcos-impl-cost | supp-gaps §1 | docs.ifcopenshell.org/.../api/cost/, .../util/cost/ |
| ifcos-impl-sequence | supp-gaps §2 | docs.ifcopenshell.org/.../api/sequence/, .../util/sequence/ |
| ifcos-impl-mep | supp-gaps §3 | docs.ifcopenshell.org/.../api/system/, .../util/system/ |
| ifcos-impl-profiles | supp-gaps §5 | docs.ifcopenshell.org/.../api/profile/ |
| ifcos-errors-schema | vooronderzoek §7,§11(1-2), frag/schema-versions (full), topic/schema (full) | technical.buildingsmart.org/standards/ifc/, ifc43-docs..., ifcpatch docs |
| ifcos-errors-patterns | vooronderzoek §11, frag/errors-perf §1, topic/errors §1 | docs.ifcopenshell.org/ifcopenshell-python.html |
| ifcos-errors-performance | vooronderzoek §12, frag/errors-perf §2, topic/errors §2 | docs.ifcopenshell.org/.../geometry_processing.html |
| ifcos-core-concepts | vooronderzoek §8, frag/schema-versions §2, topic/schema §2-4 | ifc43-docs..., technical.buildingsmart.org/standards/ |
| ifcos-core-runtime | supp-gaps §7, topic/errors §3-4 | docs.ifcopenshell.org/.../entity_instance/, pypi.org/project/ifcopenshell/ |
| ifcos-code-validator | vooronderzoek §13, frag/errors-perf §3, topic/errors §3,5,6 | All IfcOpenShell URLs |

---

## 5. Build Order Recommendation

```
Layer 1 (Foundation — no dependencies):
  ifcos-core-concepts, ifcos-core-runtime, ifcos-syntax-fileio

Layer 2 (API Reference — depends on Layer 1):
  ifcos-syntax-api, ifcos-syntax-elements, ifcos-syntax-util

Layer 3 (Implementation — depends on Layer 2):
  ifcos-impl-creation, ifcos-impl-geometry, ifcos-impl-materials,
  ifcos-impl-relationships, ifcos-impl-profiles

Layer 4 (Domain Skills — depends on Layer 3):
  ifcos-impl-cost, ifcos-impl-sequence, ifcos-impl-mep

Layer 5 (Error Handling — references all layers):
  ifcos-errors-schema, ifcos-errors-patterns, ifcos-errors-performance

Layer 6 (Orchestration — depends on all):
  ifcos-code-validator
```

---

## 6. Research Coverage Assessment

| Research File | Lines | Skills Fed |
|---|---|---|
| vooronderzoek-ifcopenshell.md | ~2030 | ALL 18 skills |
| supplementary-ifcos-gaps.md | 1381 | 5 skills (cost, sequence, mep, profiles, runtime) |
| fragments/ifcos-api-categories.md | 1171 | 4 skills (syntax-api, impl-creation, impl-geometry, impl-materials) |
| fragments/ifcos-core-operations.md | 979 | 2 skills (syntax-fileio, syntax-elements) |
| fragments/ifcos-errors-performance.md | 1109 | 3 skills (errors-patterns, errors-performance, code-validator) |
| fragments/ifcos-schema-versions.md | 834 | 3 skills (errors-schema, core-concepts, impl-relationships) |
| topic-research/ifcos-core-operations.md | 1474 | 4 skills (syntax-util, impl-geometry, syntax-elements, syntax-fileio) |
| topic-research/ifcos-errors-performance-research.md | 1519 | 3 skills (errors-patterns, errors-performance, code-validator) |
| topic-research/ifcos-schema-version-comparison.md | 1485 | 3 skills (errors-schema, core-concepts, impl-relationships) |
| scope-analysis.md | 484 | Scope boundaries — all skills |
| raw-masterplan.md | 229 | Structural template — all skills |
| REQUIREMENTS.md | 108 | Quality criteria — all skills |
| DECISIONS.md | 73 | Process constraints — all skills |
| SOURCES.md | 274 | URL verification — all skills |

**Total research input**: ~11,150 lines across 14 files → 18 skills

---

*End of briefing. This document is input for the masterplan writer agent.*
