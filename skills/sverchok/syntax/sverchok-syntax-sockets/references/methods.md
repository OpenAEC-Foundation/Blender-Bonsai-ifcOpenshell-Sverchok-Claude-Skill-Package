# sverchok-syntax-sockets — API Reference: Methods and Properties

Sverchok v1.4.0, Blender 4.0+/5.x
Sources: `core/sockets.py`, `core/socket_conversions.py`, `core/socket_data.py`

---

## SvSocketProcessing (base of SvSocketCommon)

Defines all data processing flag properties. Inherits from Blender's `PropertyGroup`.

```python
class SvSocketProcessing:
    # Permission flags — set in sv_init() to show flag UI controls
    allow_flatten: BoolProperty(default=False)
    allow_flatten_topology: BoolProperty(default=False)
    allow_simplify: BoolProperty(default=False)
    allow_graft: BoolProperty(default=False)
    allow_unwrap: BoolProperty(default=False)
    allow_wrap: BoolProperty(default=False)

    # Internal update control flags
    skip_simplify_mode_update: BoolProperty(default=False)
    skip_wrap_mode_update: BoolProperty(default=False)

    # Active user-controlled flags (set via UI or code)
    use_graft: BoolProperty(name="Graft", default=False, update=process_from_socket)
    use_unwrap: BoolProperty(name="Unwrap", default=False, update=update_unwrap_flag)
    use_wrap: BoolProperty(name="Wrap", default=False, update=update_wrap_flag)
    use_flatten_topology: BoolProperty(name="Flatten Topology", default=False, update=process_from_socket)
    use_flatten: BoolProperty(name="Flatten", default=False, update=update_flatten_flag)
    use_simplify: BoolProperty(name="Simplify", default=False, update=update_simplify_flag)
```

---

## SvSocketCommon

Inherits from `SvSocketProcessing`. All Sverchok socket classes inherit from both `NodeSocket` and `SvSocketCommon`.

### Properties

```python
class SvSocketCommon(SvSocketProcessing):
    # Display
    color: tuple                      # RGBA — defined per socket class, not a BPY prop
    label: StringProperty(default='') # Overrides socket name in UI

    # Property binding
    use_prop: BoolProperty(default=False)    # Show embedded node property
    prop_name: StringProperty(default='')   # Node attribute name for property display
    custom_draw: StringProperty(default='') # Custom draw method name on node

    # Data shape tracking
    objects_number: IntProperty(min=0)      # Number of data objects
    is_mandatory: BoolProperty(default=False)

    # Data structure hints
    nesting_level: IntProperty(default=2)   # Expected list depth
    default_mode: EnumProperty(            # Fallback when no data
        items=['NONE', 'EMPTY_LIST', 'MATRIX', 'MASK'],
        default='EMPTY_LIST'
    )
    pre_processing: EnumProperty(          # Pre-processing mode
        items=['NONE', 'ONE_ITEM'],
        default='NONE'
    )

    # Internal cache key
    s_id: StringProperty(options={'SKIP_SAVE'})  # Stores computed socket_id

    # Class-level attributes (not BPY props, defined per socket class)
    quick_link_to_node: str   # bl_idname of node for quick-link button
    default_conversion_name: str  # Name of ConversionPolicy class
```

### Methods

```python
def sv_get(self, default=SvNoDataError, deepcopy=True, implicit_conversions=None) -> list:
    """
    Retrieve data from this socket.

    Fallback chain (input sockets):
      1. Data written by connected upstream node (from socket data cache)
      2. Node property named by prop_name (if prop_name is set)
      3. Socket's own property (if use_prop=True)
      4. default parameter value
      5. Raises SvNoDataError (if default is not provided)

    For output sockets: reads from socket data cache.

    Args:
        default: Value returned when no data exists. Omit to raise SvNoDataError.
        deepcopy: If True (default), returns independent copy — safe to mutate.
                  If False, returns reference — NEVER mutate this.
        implicit_conversions: Override the socket's default_conversion_name policy.

    Returns:
        list — nested according to socket's nesting_level convention.

    Raises:
        SvNoDataError: When no data exists and no default is provided.
    """

def sv_set(self, data: list) -> None:
    """
    Write data to this socket's cache slot.

    Called inside process() on OUTPUT sockets only.
    Data MUST conform to the socket's nesting_level:
      - SvStringsSocket (level 2): [[v1, v2, ...], [v1, v2, ...], ...]
      - SvVerticesSocket (level 3): [[(x,y,z), ...], [(x,y,z), ...], ...]
      - SvMatrixSocket (level 1): [Matrix, Matrix, ...]

    Args:
        data: Nested list matching socket nesting convention.
    """

def sv_forget(self) -> None:
    """
    Remove this socket's data from the global cache.

    ALWAYS call before replace_socket() to prevent stale data
    from persisting under a different socket type with the same cache key.
    """

def replace_socket(self, new_type: str, new_name: str = None) -> NodeSocket:
    """
    Replace this socket with a new socket of a different type.

    Preserves all existing links. The current socket object is destroyed.

    Args:
        new_type: bl_idname string of the new socket type
                  (e.g., 'SvCurveSocket', 'SvStringsSocket').
        new_name: New display name. If None, keeps current name.

    Returns:
        NodeSocket — the newly created replacement socket.
        The original socket reference is INVALID after this call.

    Implementation:
        def replace_socket(self, new_type, new_name=None):
            self.sv_forget()
            return replace_socket(self, new_type, new_name)
    """

def get_prop_name(self) -> str:
    """Return effective property name for UI display."""

def draw_property(self, layout, prop_origin=None, prop_name='default_property') -> None:
    """Draw the socket's embedded property in the node UI."""

def draw_quick_link(self, context, layout, node) -> None:
    """Draw the quick-link button (uses quick_link_to_node)."""

def draw(self, context, layout, node, text) -> None:
    """Main draw method — called by Blender for each socket in the node UI."""
```

### Computed Properties

```python
@property
def socket_id(self) -> str:
    """
    Unique string key for the socket data cache.

    Computed as: str(hash(node.node_id + socket.identifier + direction))
    where direction is 'o' for output, 'i' for input.

    Lazily computed and stored in self.s_id (SKIP_SAVE).
    Regenerated if s_id is empty (e.g., after undo/redo).

    Implementation:
        @property
        def socket_id(self):
            _id = self.s_id
            if not _id:
                self.s_id = str(hash(
                    self.node.node_id + self.identifier +
                    ('o' if self.is_output else 'i')
                ))
                _id = self.s_id
            return _id
    """

@property
def other(self) -> NodeSocket:
    """
    Returns the socket at the opposite end of the link.

    For INPUT sockets: returns the single connected output socket.
    For OUTPUT sockets: returns ONE arbitrarily chosen connected input socket.

    Returns None if the socket is not linked.

    Implementation:
        @property
        def other(self):
            return get_other_socket(self)

    NEVER rely on this for output sockets with multiple connections.
    """

@property
def index(self) -> int:
    """Zero-based index of this socket in node.inputs or node.outputs."""
```

---

## ConversionPolicies Enum

Source: `core/socket_conversions.py`

```python
from enum import Enum

class ConversionPolicies(Enum):
    DEFAULT = DefaultImplicitConversionPolicy
    FIELD   = FieldImplicitConversionPolicy
    LENIENT = LenientImplicitConversionPolicy
    SOLID   = SolidImplicitConversionPolicy

    @property
    def conversion(self):
        return self.value  # Returns the policy class

    @property
    def conversion_name(self):
        return self.value.__name__  # Returns the class name as string
```

---

## DefaultImplicitConversionPolicy

```python
class DefaultImplicitConversionPolicy(NoImplicitConversionPolicy):
    default_conversions = {
        ('SvVerticesSocket', 'SvMatrixSocket'): vectors_to_matrices,
        ('SvVerticesSocket', 'SvColorSocket'):  vector_to_color,
        ('SvMatrixSocket',   'SvVerticesSocket'): matrices_to_vectors,
        ('SvMatrixSocket',   'SvQuaternionSocket'): matrices_to_quaternions,
        ('SvQuaternionSocket', 'SvMatrixSocket'): quaternions_to_matrices,
        ('SvStringsSocket',  'SvVerticesSocket'): string_to_vector,
        ('SvStringsSocket',  'SvColorSocket'):  string_to_color,
    }

    lenient_socket_types = {
        'SvStringsSocket',
        'SvObjectSocket',
        'SvColorSocket',
        'SvVerticesSocket',
    }

    expected_data_types = {
        'SvScalarFieldSocket': SvScalarField,
        'SvVectorFieldSocket': SvVectorField,
        'SvSurfaceSocket':     SvSurface,
        'SvCurveSocket':       SvCurve,
    }

    @classmethod
    def convert(cls, to_sock, from_sock, source_data):
        convert_pattern = (from_sock.bl_idname, to_sock.bl_idname)
        if conversion := cls.default_conversions.get(convert_pattern):
            return conversion(source_data)
        elif to_sock.bl_idname in cls.lenient_socket_types \
                or cls.is_expected_type_from_string_socket(to_sock, from_sock, source_data):
            return source_data
        super().convert(to_sock, from_sock, source_data)  # raises
```

---

## FieldImplicitConversionPolicy

```python
class FieldImplicitConversionPolicy(DefaultImplicitConversionPolicy):
    default_conversions = {
        ('SvMatrixSocket',   'SvVectorFieldSocket'): matrices_to_vfield,
        ('SvVerticesSocket', 'SvVectorFieldSocket'): vertices_to_vfield,
        ('SvStringsSocket',  'SvScalarFieldSocket'): check_nesting_level(numbers_to_sfield),
    }

    @classmethod
    def convert(cls, to_sock, from_sock, source_data):
        types_pattern = (from_sock.bl_idname, to_sock.bl_idname)
        if conversion := cls.default_conversions.get(types_pattern):
            return conversion(source_data)
        return super().convert(to_sock, from_sock, source_data)
```

---

## LenientImplicitConversionPolicy

```python
class LenientImplicitConversionPolicy:
    """Passes any data type through unchanged. No type checking."""
    @classmethod
    def convert(cls, socket, other, source_data):
        return source_data
```

---

## SolidImplicitConversionPolicy

```python
class SolidImplicitConversionPolicy(NoImplicitConversionPolicy):
    @classmethod
    def convert(cls, socket, other, source_data):
        try:
            return to_solid_recursive(source_data)
        except TypeError as e:
            raise ImplicitConversionProhibited(
                socket,
                msg=f"Cannot perform implicit socket conversion for"
                    f" socket {socket.name}: {e}"
            )
```

---

## conversions Registry

```python
# core/socket_conversions.py
conversions = {
    'DefaultImplicitConversionPolicy': DefaultImplicitConversionPolicy,
    'FieldImplicitConversionPolicy':   FieldImplicitConversionPolicy,
    'LenientImplicitConversionPolicy': LenientImplicitConversionPolicy,
    'SolidImplicitConversionPolicy':   SolidImplicitConversionPolicy,
}
```

Used by `sv_get()` to look up the policy class by `socket.default_conversion_name`.

---

## Per-Socket Class Specifications

### SvStringsSocket
```python
bl_idname = 'SvStringsSocket'
color = (0.6, 1.0, 0.6, 1.0)       # Green
nesting_level = 2                   # [[val, ...], ...]
# No default_conversion_name set (accepts anything as lenient type)
```

### SvVerticesSocket
```python
bl_idname = 'SvVerticesSocket'
color = (0.9, 0.6, 0.2, 1.0)       # Orange
nesting_level = 3                   # [[(x,y,z), ...], ...]
default_conversion_name = ConversionPolicies.DEFAULT.conversion_name
quick_link_to_node = 'GenVectorsNode'
```

### SvMatrixSocket
```python
bl_idname = 'SvMatrixSocket'
color = (0.2, 0.8, 0.8, 1.0)       # Teal
nesting_level = 1                   # [Matrix, Matrix, ...]
quick_link_to_node = 'SvMatrixInNodeMK4'
```

### SvQuaternionSocket
```python
bl_idname = 'SvQuaternionSocket'
color = (0.9, 0.4, 0.7, 1.0)       # Pink
nesting_level = 2
```

### SvColorSocket
```python
bl_idname = 'SvColorSocket'
color = (0.9, 0.8, 0.0, 1.0)       # Yellow
nesting_level = 3                   # [[(r,g,b,a), ...], ...]
```

### SvCurveSocket
```python
bl_idname = 'SvCurveSocket'
color = (0.5, 0.6, 1.0, 1.0)       # Blue
nesting_level = 2                   # [[SvCurve, ...], ...]
# Has reparametrize: BoolProperty — triggers reparametrization on connect
```

### SvSurfaceSocket
```python
bl_idname = 'SvSurfaceSocket'
color = (0.4, 0.2, 1.0, 1.0)       # Deep purple
nesting_level = 2                   # [[SvSurface, ...], ...]
```

### SvScalarFieldSocket
```python
bl_idname = 'SvScalarFieldSocket'
color = (0.9, 0.4, 0.1, 1.0)       # Dark orange
nesting_level = 2
default_conversion_name = ConversionPolicies.FIELD.conversion_name
quick_link_to_node = 'SvNumberNode'
```

### SvVectorFieldSocket
```python
bl_idname = 'SvVectorFieldSocket'
color = (0.1, 0.1, 0.9, 1.0)       # Deep blue
nesting_level = 2
default_conversion_name = ConversionPolicies.FIELD.conversion_name
quick_link_to_node = 'GenVectorsNode'
```

### SvSolidSocket
```python
bl_idname = 'SvSolidSocket'
color = (0.0, 0.65, 0.3, 1.0)      # Green (FreeCAD)
nesting_level = 2
default_conversion_name = ConversionPolicies.SOLID.conversion_name
```

### SvDictionarySocket
```python
bl_idname = 'SvDictionarySocket'
color = (1.0, 1.0, 1.0, 1.0)       # White
nesting_level = 2
quick_link_to_node = 'SvDictionaryIn'
```

### SvObjectSocket
```python
bl_idname = 'SvObjectSocket'
color = (0.69, 0.74, 0.73, 1.0)    # Grey-green
nesting_level = 2
# Has object_kinds: StringProperty — filter by object type
```

### SvFilePathSocket
```python
bl_idname = 'SvFilePathSocket'
color = (0.9, 0.9, 0.3, 1.0)       # Yellow-green
nesting_level = 2
quick_link_to_node = 'SvFilePathNode'
```

### SvTextSocket
```python
bl_idname = 'SvTextSocket'
color = (0.68, 0.85, 0.90, 1.0)    # Light blue
nesting_level = 2
default_conversion_name = ConversionPolicies.LENIENT.conversion_name
```

### SvDummySocket
```python
bl_idname = 'SvDummySocket'
color = (0.8, 0.8, 0.8, 0.3)       # Grey, semi-transparent
# Placeholder only — not for data transport
```

### SvChameleonSocket
```python
bl_idname = 'SvChameleonSocket'
# color: dynamic, mirrors linked socket's color
# Adopts linked socket's type and nesting_level at connect time
```
