# Version Error Diagnosis Methods

Functions and utilities for diagnosing version compatibility errors, detecting the active Blender version, and performing compatibility checks.

---

## Version Detection API

### bpy.app.version

Returns Blender version as `tuple[int, int, int]`. ALWAYS use this for version checks.

```python
# Blender 3.x / 4.x / 5.x — Primary version detection
import bpy

major, minor, patch = bpy.app.version  # e.g., (5, 0, 0)

# Tuple comparison — ALWAYS use this pattern
if bpy.app.version >= (4, 0, 0):
    pass  # Blender 4.0+ code path
```

### bpy.app.version_file

Returns the version of the `.blend` file format. Use to detect files saved with older versions.

```python
# Blender 3.x / 4.x / 5.x — Detect outdated .blend files
import bpy

if bpy.app.version_file < (4, 0, 0):
    print("WARNING: File was saved with Blender 3.x — data may need migration")
```

### bpy.app.version_cycle

Returns release stage: `"alpha"`, `"beta"`, `"release candidate"`, or `"release"`.

```python
# Blender 3.x / 4.x / 5.x — Warn on non-release builds
import bpy

if bpy.app.version_cycle != "release":
    print(f"WARNING: Development build ({bpy.app.version_cycle})")
```

---

## Diagnostic Utilities

### Full Diagnostic Report

```python
# Blender 3.x / 4.x / 5.x — Complete diagnostic report for error investigation
import bpy
import sys

def print_blender_diagnostics():
    """Print full version diagnostics for debugging version errors."""
    print("=== Blender Version Diagnostics ===")
    print(f"Blender version: {bpy.app.version}")
    print(f"Version string: {bpy.app.version_string}")
    print(f"Version cycle: {bpy.app.version_cycle}")
    print(f"File version: {bpy.app.version_file}")
    print(f"Python version: {sys.version}")
    print(f"Build hash: {bpy.app.build_hash}")
    print(f"Build date: {bpy.app.build_date}")
    print(f"Binary path: {bpy.app.binary_path}")

    # Blender 4.2+ specific
    if hasattr(bpy.app, 'online_access'):
        print(f"Online access: {bpy.app.online_access}")

    # Blender 4.4+ specific
    if hasattr(bpy.app, 'portable'):
        print(f"Portable: {bpy.app.portable}")
        print(f"Module mode: {bpy.app.module}")
```

### Version Compatibility Check

```python
# Blender 3.x / 4.x / 5.x — Reusable compatibility checker
import bpy

def check_version_compatibility(min_version, max_version=None):
    """Check if current Blender version is within supported range.

    Args:
        min_version: Minimum required version as tuple (major, minor, patch)
        max_version: Maximum supported version as tuple (exclusive). None = no upper limit.

    Returns:
        tuple: (is_compatible: bool, message: str)
    """
    current = bpy.app.version
    if current < min_version:
        return (False, f"Blender {current} is below minimum {min_version}")
    if max_version and current >= max_version:
        return (False, f"Blender {current} is above maximum {max_version}")
    return (True, f"Blender {current} is compatible")


# Usage
ok, msg = check_version_compatibility((4, 0, 0), (6, 0, 0))
if not ok:
    raise RuntimeError(msg)
```

### Feature Detection

ALWAYS prefer `hasattr()` over version comparison when testing for a specific API method or property.

```python
# Blender 3.x / 4.x / 5.x — Feature detection for API availability
import bpy

def has_temp_override():
    """Check if context.temp_override() is available (Blender 4.0+)."""
    return hasattr(bpy.context, 'temp_override')

def has_node_interface():
    """Check if NodeTree.interface API is available (Blender 4.0+)."""
    return hasattr(bpy.types.NodeTree, 'interface')

def has_corner_normals():
    """Check if Mesh.corner_normals is available (Blender 4.1+)."""
    mesh = bpy.data.meshes.new("__test")
    result = hasattr(mesh, 'corner_normals')
    bpy.data.meshes.remove(mesh)
    return result

def has_compositing_node_group():
    """Check if Scene.compositing_node_group is available (Blender 5.0+)."""
    return hasattr(bpy.types.Scene, 'compositing_node_group')

def has_gpu_module():
    """Check if gpu module is available (Blender 3.0+, REQUIRED for 5.0+)."""
    try:
        import gpu
        return hasattr(gpu, 'state')
    except ImportError:
        return False

def has_bgl_module():
    """Check if bgl module is still available (REMOVED in 5.0)."""
    try:
        import bgl
        return True
    except ImportError:
        return False
```

---

## Error Classification Functions

### Classify Version Error

```python
# Blender 3.x / 4.x / 5.x — Classify errors by version cause
import bpy

# Mapping of removed attributes to version and replacement
VERSION_ERROR_MAP = {
    # Blender 4.0 removals
    ("MeshEdge", "bevel_weight"): ((4, 0, 0), "mesh.attributes.get('bevel_weight_edge')"),
    ("MeshEdge", "crease"): ((4, 0, 0), "mesh.attributes.get('crease_edge')"),
    ("Object", "face_maps"): ((4, 0, 0), "Integer face attributes via mesh.attributes"),
    ("Bone", "layers"): ((4, 0, 0), "bone.collections"),
    ("Pose", "bone_groups"): ((4, 0, 0), "bone.collections + bone.color"),
    ("ShaderNodeTree", "inputs"): ((4, 0, 0), "NodeTree.interface.new_socket()"),
    ("ShaderNodeTree", "outputs"): ((4, 0, 0), "NodeTree.interface.new_socket()"),
    ("Mesh", "calc_normals"): ((4, 0, 0), "Removed — normals auto-calculated"),

    # Blender 4.1 removals
    ("Mesh", "use_auto_smooth"): ((4, 1, 0), "mesh.corner_normals + Smooth by Angle modifier"),
    ("Mesh", "auto_smooth_angle"): ((4, 1, 0), "Smooth by Angle modifier"),
    ("Mesh", "calc_normals_split"): ((4, 1, 0), "mesh.corner_normals"),
    ("Mesh", "create_normals_split"): ((4, 1, 0), "mesh.corner_normals"),
    ("Mesh", "free_normals_split"): ((4, 1, 0), "Removed — handled automatically"),
    ("CyclesMaterialSettings", "displacement_method"): ((4, 1, 0), "mat.displacement_method"),

    # Blender 4.3 removals
    ("SceneEEVEE", "use_ssr"): ((4, 3, 0), "Removed — EEVEE Next handles internally"),
    ("SceneEEVEE", "use_bloom"): ((4, 3, 0), "Removed — EEVEE Next handles internally"),
    ("SceneEEVEE", "use_volumetric_lights"): ((4, 3, 0), "Removed — EEVEE Next handles internally"),
    ("SceneEEVEE", "use_gtao"): ((4, 3, 0), "Removed — EEVEE Next handles internally"),

    # Blender 5.0 removals
    ("Brush", "sculpt_tool"): ((5, 0, 0), "brush.sculpt_brush_type"),
    ("Scene", "node_tree"): ((5, 0, 0), "scene.compositing_node_group"),
    ("Action", "fcurves"): ((5, 0, 0), "Slotted Actions channelbag API"),
    ("Sequence", "end_frame"): ((5, 0, 0), "strip.length"),
}

def classify_attribute_error(type_name, attr_name):
    """Classify an AttributeError as a version migration issue.

    Args:
        type_name: The class name that raised the error (e.g., 'MeshEdge')
        attr_name: The attribute that was not found (e.g., 'bevel_weight')

    Returns:
        dict with 'removed_in', 'replacement', or None if not a version issue
    """
    key = (type_name, attr_name)
    if key in VERSION_ERROR_MAP:
        removed_in, replacement = VERSION_ERROR_MAP[key]
        return {
            "removed_in": removed_in,
            "replacement": replacement,
            "current_version": bpy.app.version,
        }
    return None
```

---

## BGL-to-GPU Migration Checker

```python
# Blender 3.x / 4.x / 5.x — Scan script for bgl usage
import re

BGL_PATTERNS = {
    r'import\s+bgl': "Remove 'import bgl'. Use 'import gpu' instead.",
    r'bgl\.glEnable\(bgl\.GL_BLEND\)': "Replace with gpu.state.blend_set('ALPHA')",
    r'bgl\.glDisable\(bgl\.GL_BLEND\)': "Replace with gpu.state.blend_set('NONE')",
    r'bgl\.glLineWidth\(': "Replace with gpu.state.line_width_set()",
    r'bgl\.glEnable\(bgl\.GL_DEPTH_TEST\)': "Replace with gpu.state.depth_test_set('LESS_EQUAL')",
    r'bgl\.glDisable\(bgl\.GL_DEPTH_TEST\)': "Replace with gpu.state.depth_test_set('NONE')",
    r'bgl\.glDepthMask\(': "Replace with gpu.state.depth_mask_set()",
    r'bgl\.glPointSize\(': "Replace with gpu.state.point_size_set()",
    r"from_builtin\(['\"]3D_UNIFORM_COLOR['\"]\)": "Replace with 'POLYLINE_UNIFORM_COLOR'",
    r"from_builtin\(['\"]3D_FLAT_COLOR['\"]\)": "Replace with 'POLYLINE_FLAT_COLOR'",
    r"from_builtin\(['\"]2D_UNIFORM_COLOR['\"]\)": "Replace with 'UNIFORM_COLOR'",
    r'image\.gl_load\(\)': "Replace with gpu.texture.from_image(image)",
    r'image\.bindcode': "Replace with gpu.texture.from_image(image)",
    r'bgl\.glEnable\(bgl\.GL_LINE_SMOOTH\)': "Use POLYLINE_* shaders for antialiased lines (no direct replacement)",
}

def scan_for_bgl_usage(source_code):
    """Scan source code for deprecated bgl usage.

    Args:
        source_code: String of Python source code to scan

    Returns:
        list of dicts with 'line', 'pattern', 'fix'
    """
    issues = []
    for line_num, line in enumerate(source_code.splitlines(), 1):
        for pattern, fix in BGL_PATTERNS.items():
            if re.search(pattern, line):
                issues.append({
                    "line": line_num,
                    "code": line.strip(),
                    "fix": fix,
                })
    return issues
```

---

## Context Override Migration Checker

```python
# Blender 3.x / 4.x / 5.x — Detect legacy context overrides
import re

CONTEXT_OVERRIDE_PATTERNS = [
    r'bpy\.ops\.\w+\.\w+\(\s*\{',             # bpy.ops.foo.bar({...
    r'bpy\.ops\.\w+\.\w+\(\s*override\b',      # bpy.ops.foo.bar(override, ...
    r'bpy\.context\.copy\(\)',                   # context.copy() for override
    r'override\s*=\s*\{["\']object',            # override = {"object": ...
    r'override\s*=\s*bpy\.context\.copy\(\)',    # override = bpy.context.copy()
]

def scan_for_context_overrides(source_code):
    """Scan source code for legacy context override patterns (broken in 4.0+).

    Returns:
        list of dicts with 'line', 'code', 'fix'
    """
    issues = []
    for line_num, line in enumerate(source_code.splitlines(), 1):
        for pattern in CONTEXT_OVERRIDE_PATTERNS:
            if re.search(pattern, line):
                issues.append({
                    "line": line_num,
                    "code": line.strip(),
                    "fix": "Use bpy.context.temp_override() context manager instead",
                })
                break
    return issues
```

---

## Principled BSDF Socket Name Mapper

```python
# Blender 3.x / 4.x / 5.x — Map old Principled BSDF socket names to new names
import bpy

# Blender 4.0 renamed these Principled BSDF sockets
PRINCIPLED_SOCKET_RENAMES = {
    "Subsurface": "Subsurface Weight",
    "Specular": "Specular IOR Level",
    "Transmission": "Transmission Weight",
    "Subsurface Color": "Subsurface Radius",
    "Specular Tint": "Specular Tint",  # kept but type changed to color
    "Sheen": "Sheen Weight",
    "Clearcoat": "Coat Weight",
    "Clearcoat Roughness": "Coat Roughness",
    "Clearcoat Normal": "Coat Normal",
    "Emission": "Emission Color",
}

def get_principled_socket_name(old_name):
    """Return the correct socket name for the current Blender version.

    Args:
        old_name: Socket name as used in Blender 3.x

    Returns:
        Correct socket name for the running Blender version
    """
    if bpy.app.version >= (4, 0, 0):
        return PRINCIPLED_SOCKET_RENAMES.get(old_name, old_name)
    return old_name

def get_principled_socket(node, name):
    """Get a Principled BSDF input socket by version-safe name.

    Args:
        node: The Principled BSDF node
        name: Socket name (3.x or 4.x name accepted)

    Returns:
        The socket, or None if not found
    """
    # Try the name directly first
    if name in node.inputs:
        return node.inputs[name]

    # Try the mapped name
    mapped = get_principled_socket_name(name)
    if mapped in node.inputs:
        return node.inputs[mapped]

    # Try reverse mapping (4.x name given, running 3.x)
    reverse = {v: k for k, v in PRINCIPLED_SOCKET_RENAMES.items()}
    reversed_name = reverse.get(name)
    if reversed_name and reversed_name in node.inputs:
        return node.inputs[reversed_name]

    return None
```

---

## EEVEE Identifier Resolver

```python
# Blender 3.x / 4.x / 5.x — Get correct EEVEE engine identifier
import bpy

def get_eevee_identifier():
    """Return the correct EEVEE render engine identifier for the running version.

    Returns:
        str: 'BLENDER_EEVEE' for 3.x and 5.0+, 'BLENDER_EEVEE_NEXT' for 4.2-4.x
    """
    if bpy.app.version >= (5, 0, 0):
        return 'BLENDER_EEVEE'
    elif bpy.app.version >= (4, 2, 0):
        return 'BLENDER_EEVEE_NEXT'
    return 'BLENDER_EEVEE'

def set_eevee_engine(scene):
    """Set the render engine to EEVEE with the correct identifier."""
    scene.render.engine = get_eevee_identifier()

def is_eevee_active(scene):
    """Check if EEVEE is the active render engine (version-safe)."""
    return scene.render.engine in ('BLENDER_EEVEE', 'BLENDER_EEVEE_NEXT')
```

---

## Compositor Access Helper

```python
# Blender 3.x / 4.x / 5.x — Version-safe compositor node tree access
import bpy

def get_compositor_tree(scene):
    """Return the compositor node tree for the given scene.

    In Blender 5.0+, scene.node_tree is removed. Use scene.compositing_node_group.
    In Blender 3.x-4.x, use scene.node_tree with scene.use_nodes = True.

    Returns:
        The compositor NodeTree, created if necessary
    """
    if bpy.app.version >= (5, 0, 0):
        tree = scene.compositing_node_group
        if tree is None:
            tree = bpy.data.node_groups.new("Compositor", 'CompositorNodeTree')
            scene.compositing_node_group = tree
        return tree
    else:
        scene.use_nodes = True
        return scene.node_tree
```

---

## Sources

- https://docs.blender.org/api/current/bpy.app.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/4.1/python_api/
- https://developer.blender.org/docs/release_notes/4.2/python_api/
- https://developer.blender.org/docs/release_notes/4.3/python_api/
- https://developer.blender.org/docs/release_notes/4.4/python_api/
- https://developer.blender.org/docs/release_notes/5.0/python_api/
- https://developer.blender.org/docs/release_notes/5.1/python_api/
- https://developer.blender.org/docs/release_notes/compatibility/
