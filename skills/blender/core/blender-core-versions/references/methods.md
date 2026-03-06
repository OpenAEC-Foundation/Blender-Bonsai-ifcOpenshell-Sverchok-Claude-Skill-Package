# Version Detection API Reference

## bpy.app.version

Returns the Blender version as a tuple of three integers: `(major, minor, patch)`.

**Type**: `tuple[int, int, int]` (read-only)

```python
# Blender 3.x / 4.x / 5.x — Primary version detection method
import bpy

version = bpy.app.version  # e.g., (4, 2, 0)
major = bpy.app.version[0]  # 4
minor = bpy.app.version[1]  # 2
patch = bpy.app.version[2]  # 0

# Tuple unpacking
major, minor, patch = bpy.app.version
```

### Tuple Comparison

ALWAYS use tuple comparison for version checks. Python compares tuples element-by-element, making this reliable and readable.

```python
# Blender 3.x / 4.x / 5.x — Correct version comparison
import bpy

# Check minimum version
if bpy.app.version >= (4, 0, 0):
    print("Blender 4.0 or later")

# Check exact version
if bpy.app.version == (4, 2, 0):
    print("Exactly Blender 4.2.0")

# Check version range
if (4, 0, 0) <= bpy.app.version < (5, 0, 0):
    print("Blender 4.x series")

# Multi-branch version check
if bpy.app.version >= (5, 0, 0):
    # Blender 5.0+ path
    pass
elif bpy.app.version >= (4, 2, 0):
    # Blender 4.2–4.x path
    pass
elif bpy.app.version >= (4, 0, 0):
    # Blender 4.0–4.1 path
    pass
else:
    # Blender 3.x path
    pass
```

---

## bpy.app.version_string

Returns the Blender version as a human-readable string.

**Type**: `str` (read-only)

```python
# Blender 3.x / 4.x / 5.x — Version string for display
import bpy

version_str = bpy.app.version_string  # e.g., "4.2.0"
print(f"Running Blender {version_str}")
```

NEVER parse `bpy.app.version_string` for version comparison. ALWAYS use `bpy.app.version` tuple instead.

```python
# WRONG — NEVER do this
if bpy.app.version_string.startswith("4"):  # Fragile string parsing
    pass

# CORRECT — ALWAYS do this
if bpy.app.version >= (4, 0, 0):  # Reliable tuple comparison
    pass
```

---

## bpy.app.version_cycle

Returns the release cycle stage as a string.

**Type**: `str` (read-only)

| Value | Meaning |
|-------|---------|
| `"alpha"` | Alpha development build |
| `"beta"` | Beta development build |
| `"release candidate"` | Release candidate |
| `"release"` | Stable release |

```python
# Blender 3.x / 4.x / 5.x — Check release stage
import bpy

cycle = bpy.app.version_cycle  # e.g., "release"

if bpy.app.version_cycle != "release":
    print("WARNING: Running a development build of Blender")
```

---

## bpy.app.version_file

Returns the version of the blend file format. This differs from the application version when opening files saved with older Blender versions.

**Type**: `tuple[int, int, int]` (read-only)

```python
# Blender 3.x / 4.x / 5.x — Check file format version
import bpy

file_version = bpy.app.version_file  # e.g., (3, 6, 0) if opened old file
app_version = bpy.app.version        # e.g., (4, 2, 0) current Blender

if file_version < (4, 0, 0):
    print("File was saved with Blender 3.x — data may need migration")
```

---

## Additional Version Properties

| Property | Type | Description | Since |
|----------|------|-------------|-------|
| `bpy.app.version` | `tuple[int, int, int]` | Application version | All |
| `bpy.app.version_string` | `str` | Human-readable version | All |
| `bpy.app.version_cycle` | `str` | Release stage | All |
| `bpy.app.version_file` | `tuple[int, int, int]` | Blend file format version | All |
| `bpy.app.binary_path` | `str` | Path to Blender executable | All |
| `bpy.app.build_date` | `bytes` | Build date | All |
| `bpy.app.build_hash` | `str` | Git commit hash | All |
| `bpy.app.online_access` | `bool` | Online access permitted | 4.2+ |
| `bpy.app.portable` | `bool` | Portable installation | 4.4+ |
| `bpy.app.module` | `bool` | Running as Python module | 4.4+ |

**Removed properties:**

| Property | Removed In | Notes |
|----------|-----------|-------|
| `bpy.app.version_char` | 4.0 | No replacement; was unused |

---

## Version-Safe Utility Function

```python
# Blender 3.x / 4.x / 5.x — Reusable version utility
import bpy

def blender_version_at_least(major, minor=0, patch=0):
    """Return True if running Blender is at least the given version."""
    return bpy.app.version >= (major, minor, patch)

def blender_version_range(min_version, max_version):
    """Return True if running Blender is within the given range (inclusive min, exclusive max)."""
    return min_version <= bpy.app.version < max_version

# Usage
if blender_version_at_least(4, 2):
    # Extension manifest path
    pass

if blender_version_range((4, 0, 0), (5, 0, 0)):
    # Blender 4.x only
    pass
```

---

## Sources

- https://docs.blender.org/api/current/bpy.app.html
- https://developer.blender.org/docs/release_notes/4.0/python_api/
- https://developer.blender.org/docs/release_notes/compatibility/
