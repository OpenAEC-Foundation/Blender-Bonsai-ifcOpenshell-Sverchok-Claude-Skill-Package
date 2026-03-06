---
name: bonsai-syntax-geometry
description: "Provides Bonsai geometry creation and editing syntax including IFC representation management, profile extrusion, boolean operations, mesh-to-IFC geometry conversion, and Bonsai-specific geometry tools. Covers IfcExtrudedAreaSolid creation, IfcShapeRepresentation assignment, geometry context setup, and the relationship between Blender mesh data and IFC geometric representations."
license: MIT
compatibility: Designed for Claude Code. Requires Python 3.x.
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Bonsai Geometry Syntax

> **Version**: Bonsai v0.8.x | IfcOpenShell v0.8+ | Blender 4.2.0+ | Python 3.11
> **Module path**: `bonsai.bim.module.geometry` — NEVER `blenderbim.*`
> **Dependencies**: bonsai-core-architecture, ifcos-syntax-api

## Critical Warnings

1. **ALWAYS** create a Body context before adding geometry. Call `ifcopenshell.util.representation.get_context(model, "Model", "Body", "MODEL_VIEW")` and create the context if it returns `None`.
2. **ALWAYS** use `ifcopenshell.api.run("geometry.assign_representation", ...)` to connect a representation to an element. NEVER set `product.Representation` directly.
3. **ALWAYS** call `bpy.ops.bim.edit_object_placement()` (or `ifcopenshell.api.run("geometry.edit_object_placement", ...)`) after moving a Blender object. Changing `obj.location` or `obj.matrix_world` does NOT update `IfcLocalPlacement`.
4. **ALWAYS** call `geometry.unassign_representation` before `geometry.remove_representation`. Removing an assigned representation leaves dangling references.
5. **NEVER** create `IfcExtrudedAreaSolid` via `model.create_entity()`. Use `geometry.add_wall_representation()`, `geometry.add_profile_representation()`, or `ShapeBuilder.extrude()` — they handle units, context assignment, and `RepresentationType` automatically.
6. **NEVER** pass a single element to `vertices` in `add_mesh_representation()`. The parameter expects `[[item1_verts], [item2_verts]]` — a list of lists, parallel to `faces`.
7. **ALWAYS** use `feature.add_feature()` for openings in Bonsai v0.8.0+. The older `void.add_opening()` still works in standalone IfcOpenShell but is superseded in Bonsai.
8. **NEVER** assume `RepresentationType` is set automatically by `model.create_entity("IfcShapeRepresentation", ...)`. Use API functions or `ShapeBuilder.get_representation()` which auto-detect the correct type.

## Decision Tree: How to Create Geometry

```
What geometry do you need?
|
+-- Parametric wall (rectangle cross-section)?
|   --> geometry.add_wall_representation(context, length, height, thickness)
|
+-- Parametric slab (extruded footprint)?
|   --> geometry.add_slab_representation(context, depth, polyline=[(x,y),...])
|
+-- Standard profile extrusion (I-beam, rectangle, circle, L, T, U, Z, C)?
|   --> 1. profile.add_parameterized_profile(ifc_class="IfcRectangleProfileDef")
|   --> 2. attribute.edit_attributes(product=profile, attributes={"XDim": 0.4, "YDim": 0.4})
|   --> 3. geometry.add_profile_representation(context, profile=profile, depth=3.0)
|
+-- Custom profile extrusion (arbitrary shape)?
|   --> 1. profile.add_arbitrary_profile(profile=[(x,y), ...])
|   --> 2. geometry.add_profile_representation(context, profile=profile, depth=3.0)
|
+-- Arbitrary mesh from Blender (tessellated)?
|   --> geometry.add_mesh_representation(context, vertices=[[...]], faces=[[...]])
|
+-- Complex parametric solid (sphere, block, swept disk)?
|   --> ShapeBuilder: builder.sphere(), builder.block(), builder.create_swept_disk_solid()
|   --> Then: builder.get_representation(context, [solid])
|
+-- Geometry with boolean cuts (clipped walls, angled tops)?
|   --> Option A: clippings parameter in add_wall/slab/profile_representation
|   --> Option B: geometry.add_boolean(first_item, second_items, operator="DIFFERENCE")
|
+-- 2D axis line (wall centerline, beam axis)?
|   --> geometry.add_axis_representation(context, axis=[(x1,y1), (x2,y2)])
|
+-- Blender mesh -> IFC in Bonsai (live editing)?
|   --> bpy.ops.bim.update_representation(obj="ObjectName")
```

## Representation Contexts

### Two-Step Context Setup

```python
import ifcopenshell.api
import ifcopenshell.util.representation

# Step 1: Check for existing context
body = ifcopenshell.util.representation.get_context(
    model, "Model", "Body", "MODEL_VIEW")

# Step 2: Create if missing
if not body:
    model3d = ifcopenshell.api.run("context.add_context", model,
        context_type="Model")
    body = ifcopenshell.api.run("context.add_context", model,
        context_type="Model", context_identifier="Body",
        target_view="MODEL_VIEW", parent=model3d)
```

### Context Types and Sub-Contexts

| Parent Context | Sub-Context Identifier | Target View | Purpose |
|---|---|---|---|
| Model | Body | MODEL_VIEW | Main 3D solid geometry |
| Model | Axis | GRAPH_VIEW | Centerline/axis (walls, beams) |
| Model | Box | MODEL_VIEW | Bounding box (clash detection) |
| Model | FootPrint | MODEL_VIEW | 2D floor projection |
| Model | Profile | ELEVATION_VIEW | Cross-section profile |
| Model | Clearance | MODEL_VIEW | Clearance zones |
| Plan | Annotation | PLAN_VIEW | 2D annotations, dimensions |
| Plan | Axis | PLAN_VIEW | 2D centerline for plan views |

**ALWAYS** create the root context first (no `parent`), then create subcontexts with `parent=root_context`.

## Representation Types Reference

| RepresentationType | IFC Items | Typical Use |
|---|---|---|
| `"SweptSolid"` | `IfcExtrudedAreaSolid`, `IfcRevolvedAreaSolid` | Walls, columns, beams, slabs |
| `"Clipping"` | `IfcBooleanClippingResult` | Walls/slabs with angled cuts |
| `"Brep"` | `IfcFacetedBrep` | Arbitrary mesh (IFC2X3 or forced) |
| `"Tessellation"` | `IfcPolygonalFaceSet`, `IfcTriangulatedFaceSet` | Arbitrary mesh (IFC4+) |
| `"AdvancedBrep"` | `IfcAdvancedBrep` | NURBS/curved geometry |
| `"SurfaceModel"` | `IfcFaceBasedSurfaceModel` | Open shells |
| `"Curve2D"` | `IfcPolyline`, `IfcIndexedPolyCurve` | 2D axis lines |
| `"Curve3D"` | `IfcPolyline`, `IfcIndexedPolyCurve` | 3D axis lines |
| `"MappedRepresentation"` | `IfcMappedItem` | Type-shared geometry |
| `"BoundingBox"` | `IfcBoundingBox` | Box context only |

## Common Operations

### Create Wall Geometry

```python
import ifcopenshell.api

# Requires: body context, wall element already created
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall, representation=representation)
ifcopenshell.api.run("geometry.edit_object_placement", model, product=wall)
```

### Create Profile Extrusion (Column)

```python
import ifcopenshell.api

profile = ifcopenshell.api.run("profile.add_parameterized_profile", model,
    ifc_class="IfcRectangleProfileDef")
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=profile, attributes={"XDim": 0.4, "YDim": 0.4})

representation = ifcopenshell.api.run("geometry.add_profile_representation", model,
    context=body, profile=profile, depth=3.0)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
```

### Create Mesh Representation

```python
import ifcopenshell.api

# Vertices and faces as list-of-lists (one sub-list per representation item)
vertices = [[(0.,0.,0.), (1.,0.,0.), (1.,1.,0.), (0.,1.,0.),
             (0.,0.,1.), (1.,0.,1.), (1.,1.,1.), (0.,1.,1.)]]
faces = [[(0,1,2,3), (4,5,6,7), (0,1,5,4), (1,2,6,5), (2,3,7,6), (3,0,4,7)]]

representation = ifcopenshell.api.run("geometry.add_mesh_representation", model,
    context=body, vertices=vertices, faces=faces)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=element, representation=representation)
```

### Boolean Clipping (Angled Wall Top)

```python
import ifcopenshell.api

# Option A: Via clippings parameter (preferred for simple cuts)
representation = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=5.0, height=3.0, thickness=0.2,
    clippings=[{"location": (0.0, 0.0, 2.5), "normal": (0.0, -0.3, 1.0)}])

# Option B: Via add_boolean (for complex operations)
# first_item = existing IfcExtrudedAreaSolid from representation.Items[0]
# second_item = IfcHalfSpaceSolid
results = ifcopenshell.api.run("geometry.add_boolean", model,
    first_item=first_item, second_items=[second_item], operator="DIFFERENCE")
```

### Create Opening (Void)

```python
import ifcopenshell.api

opening = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOpeningElement", name="Door Opening")

# Create opening geometry (box shape)
opening_repr = ifcopenshell.api.run("geometry.add_wall_representation", model,
    context=body, length=0.9, height=2.1, thickness=0.3)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=opening, representation=opening_repr)

# Attach opening to wall (Bonsai v0.8+: use feature.add_feature)
ifcopenshell.api.run("feature.add_feature", model,
    feature=opening, element=wall)

# Optionally fill with door
ifcopenshell.api.run("feature.add_filling", model,
    opening=opening, element=door)
```

### Remove Representation (Correct Order)

```python
import ifcopenshell.api

# Step 1: Unassign from product
ifcopenshell.api.run("geometry.unassign_representation", model,
    product=element, representation=representation)

# Step 2: Remove the representation entity
ifcopenshell.api.run("geometry.remove_representation", model,
    representation=representation)
```

### ShapeBuilder — Parametric Geometry

```python
from ifcopenshell.util.shape_builder import ShapeBuilder

builder = ShapeBuilder(model)

# Extrude a rectangle
rect = builder.rectangle(size=(0.4, 0.4))
profile = builder.profile(rect, name="Column Profile")
solid = builder.extrude(profile, magnitude=3.0)
representation = builder.get_representation(body, [solid])

# Assign to element
ifcopenshell.api.run("geometry.assign_representation", model,
    product=column, representation=representation)
```

## Bonsai Operators (Blender Context)

| Operator | Purpose |
|---|---|
| `bpy.ops.bim.edit_object_placement()` | Sync Blender transform to IFC |
| `bpy.ops.bim.update_representation(obj="Name")` | Sync Blender mesh to IFC representation |
| `bpy.ops.bim.add_representation()` | Add new IFC representation to object |
| `bpy.ops.bim.switch_representation()` | Switch active representation |
| `bpy.ops.bim.remove_representation()` | Remove a representation |
| `bpy.ops.bim.copy_representation()` | Copy representation from active to selected |
| `bpy.ops.bim.override_mode_set_edit()` | Enter edit mode (IFC-aware) |
| `bpy.ops.bim.override_mode_set_object()` | Return to object mode (saves IFC) |
| `bpy.ops.bim.enable_editing_representation_items()` | Enable per-item editing |
| `bpy.ops.bim.remove_representation_item()` | Remove single representation item |

### Representation Class Hints for `update_representation`

When calling `bpy.ops.bim.update_representation(ifc_representation_class="...")`, valid values:

| Value | Effect |
|---|---|
| `"IfcExtrudedAreaSolid/IfcRectangleProfileDef"` | Auto-detect rectangular extrusion |
| `"IfcExtrudedAreaSolid/IfcCircleProfileDef"` | Auto-detect cylindrical extrusion |
| `"IfcExtrudedAreaSolid/IfcArbitraryClosedProfileDef"` | Auto-detect arbitrary profile extrusion |
| `"IfcExtrudedAreaSolid/IfcArbitraryProfileDefWithVoids"` | Profile extrusion with holes |
| `"IfcExtrudedAreaSolid/IfcMaterialProfileSetUsage"` | Profile from material set |
| `""` (empty/omitted) | Tessellation/Brep (mesh as-is) |

## MappedItems — Type Geometry Sharing

When `type.assign_type` is called, geometry on the `IfcTypeProduct` is shared to all instances via `IfcMappedItem`. Each instance references the type's `IfcRepresentationMap` with its own transformation.

**ALWAYS** assign geometry to the type, not individual occurrences, when using typed elements. This ensures geometry is shared efficiently.

```python
# Add geometry to type (not occurrence)
ifcopenshell.api.run("geometry.assign_representation", model,
    product=wall_type, representation=representation)

# Assign type to occurrence — geometry is automatically mapped
ifcopenshell.api.run("type.assign_type", model,
    related_objects=[wall], relating_type=wall_type)
```

## Version Notes

### IFC2X3 vs IFC4+ Geometry Differences

| Feature | IFC2X3 | IFC4+ |
|---|---|---|
| Mesh representation | `IfcFacetedBrep` only | `IfcPolygonalFaceSet` (preferred), `IfcTriangulatedFaceSet` |
| Curve types | `IfcPolyline` | `IfcIndexedPolyCurve` (preferred), `IfcPolyline` |
| Boolean result | `IfcBooleanResult` | `IfcBooleanClippingResult` (for DIFFERENCE + SweptSolid) |
| Triangulated mesh | Not available | `IfcTriangulatedFaceSet` |

### Bonsai v0.8.0+ Changes

- `void.add_opening()` superseded by `feature.add_feature()` for all feature types
- `feature.add_feature()` handles subtraction (`IfcRelVoidsElement`), addition (`IfcRelProjectsElement`), and surface features (`IfcRelAdheresToElement`)
- Relationship API functions now require **list** parameters: `products=[element]`, not `product=element`

## References

- [methods.md](references/methods.md) — Complete geometry API signatures
- [examples.md](references/examples.md) — Working geometry code examples
- [anti-patterns.md](references/anti-patterns.md) — Geometry mistakes to avoid

## Sources

- Bonsai source: `bonsai/bim/module/geometry/` (v0.8.x)
- IfcOpenShell API: `ifcopenshell/api/geometry/` (v0.8+)
- IfcOpenShell ShapeBuilder: `ifcopenshell/util/shape_builder.py`
- Bonsai Documentation: https://docs.bonsaibim.org
- IfcOpenShell Docs: https://docs.ifcopenshell.org
