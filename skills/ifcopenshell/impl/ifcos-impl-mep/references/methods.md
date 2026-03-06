# ifcos-impl-mep -- Methods Reference

Complete API signatures for the `ifcopenshell.api.system` module and related utility functions for MEP system modeling.

---

## ifcopenshell.api.system -- All 12 Functions

### System Management

```python
ifcopenshell.api.run("system.add_system", file,
    ifc_class: str = "IfcDistributionSystem"
) -> ifcopenshell.entity_instance
```
Creates a new distribution system. Use `ifc_class="IfcSystem"` for IFC2X3, `"IfcDistributionSystem"` for IFC4+, `"IfcBuildingSystem"` for facade systems. Returns the newly created system entity.

```python
ifcopenshell.api.run("system.edit_system", file,
    system: ifcopenshell.entity_instance,
    attributes: dict[str, Any]
) -> None
```
Modifies system attributes. Common attributes: `Name` (str), `Description` (str), `PredefinedType` (str, IFC4+ only), `ObjectType` (str, for USERDEFINED or IFC2X3 classification).

```python
ifcopenshell.api.run("system.remove_system", file,
    system: ifcopenshell.entity_instance
) -> None
```
Deletes a distribution system. Contained elements are NOT deleted -- they remain in the model as unassigned distribution elements.

```python
ifcopenshell.api.run("system.assign_system", file,
    products: list[ifcopenshell.entity_instance],
    system: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```
Assigns distribution elements to a system via `IfcRelAssignsToGroup`. Returns the relationship entity, or `None` if the products list is empty. Elements MUST be distribution elements (subclasses of `IfcDistributionElement`).

```python
ifcopenshell.api.run("system.unassign_system", file,
    products: list[ifcopenshell.entity_instance],
    system: ifcopenshell.entity_instance
) -> None
```
Removes distribution elements from system membership. Elements remain in the model.

### Port Management

```python
ifcopenshell.api.run("system.add_port", file,
    element: ifcopenshell.entity_instance | None = None
) -> ifcopenshell.entity_instance
```
Creates a new `IfcDistributionPort` and optionally nests it to the given element via `IfcRelNests`. If `element` is `None`, creates an orphaned port that must be later assigned with `assign_port`. Returns the new port entity.

```python
ifcopenshell.api.run("system.assign_port", file,
    element: ifcopenshell.entity_instance,
    port: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance
```
Assigns an orphaned port to a distribution element via `IfcRelNests`. Returns the nesting relationship entity. Use when ports are created separately from elements.

```python
ifcopenshell.api.run("system.unassign_port", file,
    element: ifcopenshell.entity_instance,
    port: ifcopenshell.entity_instance
) -> None
```
Detaches a port from its element, making it orphaned. The port entity is NOT deleted.

### Port Connections

```python
ifcopenshell.api.run("system.connect_port", file,
    port1: ifcopenshell.entity_instance,
    port2: ifcopenshell.entity_instance,
    direction: str = "NOTDEFINED",
    element: ifcopenshell.entity_instance | None = None
) -> None
```
Connects two ports together via `IfcRelConnectsPorts`, establishing a flow path between the elements that own these ports. The `direction` parameter describes the flow from `port1`'s perspective:
- `"SOURCE"` -- flow goes FROM port1 TO port2
- `"SINK"` -- flow goes FROM port2 TO port1
- `"SOURCEANDSINK"` -- bidirectional flow
- `"NOTDEFINED"` -- direction unspecified

The optional `element` parameter can specify a realizing element (e.g., a fitting that mediates the connection).

```python
ifcopenshell.api.run("system.disconnect_port", file,
    port: ifcopenshell.entity_instance
) -> None
```
Disconnects a port from its connected counterpart. Removes the `IfcRelConnectsPorts` relationship. Both ports remain assigned to their respective elements.

### Flow Control

```python
ifcopenshell.api.run("system.assign_flow_control", file,
    relating_flow_element: ifcopenshell.entity_instance,
    related_flow_control: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```
Assigns a control element (sensor, actuator, controller) to a flow element (duct, pipe, etc.) via `IfcRelFlowControlElements`. Returns the relationship entity, or `None` if the control is already assigned elsewhere. A flow control can only be assigned to ONE flow element at a time.

```python
ifcopenshell.api.run("system.unassign_flow_control", file,
    relating_flow_element: ifcopenshell.entity_instance,
    related_flow_control: ifcopenshell.entity_instance
) -> None
```
Removes a control element assignment from a flow element.

---

## ifcopenshell.util.system -- Query Functions

```python
ifcopenshell.util.system.get_system_elements(
    system: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]
```
Returns all distribution elements assigned to the given system.

```python
ifcopenshell.util.system.get_element_systems(
    element: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]
```
Returns all systems that the given element belongs to. An element can belong to multiple systems.

```python
ifcopenshell.util.system.get_ports(
    element: ifcopenshell.entity_instance,
    flow_direction: str | None = None
) -> list[ifcopenshell.entity_instance]
```
Returns ports nested on the given element. Optionally filter by `flow_direction` (`"SOURCE"`, `"SINK"`, `"SOURCEANDSINK"`, `"NOTDEFINED"`).

```python
ifcopenshell.util.system.get_connected_to(
    element: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]
```
Returns elements connected downstream (via SOURCE ports) from the given element.

```python
ifcopenshell.util.system.get_connected_from(
    element: ifcopenshell.entity_instance
) -> list[ifcopenshell.entity_instance]
```
Returns elements connected upstream (via SINK ports) to the given element.

```python
ifcopenshell.util.system.get_port_element(
    port: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```
Returns the element that owns the given port, or `None` if the port is orphaned.

```python
ifcopenshell.util.system.get_connected_port(
    port: ifcopenshell.entity_instance
) -> ifcopenshell.entity_instance | None
```
Returns the port connected to the given port, or `None` if the port is not connected.

```python
ifcopenshell.util.system.is_assignable(
    product: ifcopenshell.entity_instance,
    system: ifcopenshell.entity_instance
) -> bool
```
Checks whether a product can be assigned to a given system. Returns `True` if the product is a valid distribution element for the system type.

---

## Distribution Element Class Hierarchy

All MEP elements inherit from `IfcDistributionElement`:

```
IfcDistributionElement
├── IfcDistributionFlowElement
│   ├── IfcFlowSegment
│   │   ├── IfcDuctSegment
│   │   ├── IfcPipeSegment
│   │   ├── IfcCableSegment
│   │   └── IfcCableCarrierSegment
│   ├── IfcFlowFitting
│   │   ├── IfcDuctFitting
│   │   ├── IfcPipeFitting
│   │   ├── IfcCableFitting
│   │   ├── IfcCableCarrierFitting
│   │   └── IfcJunctionBox
│   ├── IfcFlowTerminal
│   │   ├── IfcAirTerminal
│   │   ├── IfcSanitaryTerminal
│   │   ├── IfcOutlet
│   │   ├── IfcLightFixture
│   │   ├── IfcFireSuppressionTerminal
│   │   ├── IfcStackTerminal
│   │   ├── IfcWasteTerminal
│   │   ├── IfcSpaceHeater
│   │   ├── IfcMedicalDevice
│   │   ├── IfcAudioVisualAppliance
│   │   └── IfcCommunicationsAppliance
│   ├── IfcFlowController
│   │   ├── IfcValve
│   │   ├── IfcDamper
│   │   ├── IfcSwitchingDevice
│   │   ├── IfcProtectiveDevice
│   │   ├── IfcFlowMeter
│   │   └── IfcElectricDistributionBoard
│   ├── IfcFlowMovingDevice
│   │   ├── IfcFan
│   │   ├── IfcPump
│   │   └── IfcCompressor
│   ├── IfcEnergyConversionDevice
│   │   ├── IfcBoiler
│   │   ├── IfcChiller
│   │   ├── IfcHeatExchanger
│   │   ├── IfcCoolingTower
│   │   ├── IfcCoil
│   │   ├── IfcEvaporativeCooler
│   │   ├── IfcAirToAirHeatRecovery
│   │   ├── IfcCondenser
│   │   ├── IfcEvaporator
│   │   ├── IfcBurner
│   │   ├── IfcEngine
│   │   ├── IfcSolarDevice
│   │   ├── IfcTransformer
│   │   ├── IfcElectricGenerator
│   │   ├── IfcElectricMotor
│   │   ├── IfcUnitaryEquipment
│   │   └── IfcHumidifier
│   ├── IfcFlowStorageDevice
│   │   ├── IfcTank
│   │   └── IfcElectricFlowStorageDevice
│   └── IfcFlowTreatmentDevice
│       ├── IfcFilter
│       ├── IfcInterceptor
│       └── IfcDuctSilencer
└── IfcDistributionControlElement
    ├── IfcSensor
    ├── IfcActuator
    ├── IfcAlarm
    ├── IfcController
    ├── IfcFlowInstrument
    └── IfcProtectiveDeviceTrippingUnit
```

---

## Port Count Guidelines

| Element Category | Typical Port Count | Description |
|-----------------|-------------------|-------------|
| Segments | 2 | Inlet + Outlet |
| Straight fittings | 2 | Inlet + Outlet (reducer, coupling) |
| Elbows | 2 | Inlet + Outlet (direction change) |
| Tees | 3 | Inlet + 2 Outlets (or 2 Inlets + 1 Outlet) |
| Crosses | 4 | Inlet + 3 Outlets (or variations) |
| Terminals | 1 | Inlet only (end of line) |
| Energy conversion | 2-4+ | Depends on medium connections |
| Pumps/Fans | 2 | Suction (SINK) + Discharge (SOURCE) |
| Controllers | 0 | Use assign_flow_control, not ports |
| Sensors | 0 | Use assign_flow_control, not ports |
