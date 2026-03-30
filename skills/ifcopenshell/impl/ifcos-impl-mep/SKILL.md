---
name: ifcos-impl-mep
description: >
  Use when modeling MEP (Mechanical, Electrical, Plumbing) systems in IFC -- HVAC, piping,
  electrical circuits, distribution elements, ports, and connections. Prevents the common
  mistake of not creating ports for flow connections between elements. Covers IfcSystem,
  IfcDistributionElement, flow segments, fittings, and MEP property sets.
  Keywords: MEP, HVAC, piping, electrical, IfcSystem, IfcDistributionElement, port,
  connection, flow segment, fitting, mechanical, plumbing.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell Python library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MEP System Implementation Guide

## Quick Reference

### Critical Warnings

- **ALWAYS** create ports on distribution elements BEFORE attempting to connect them. Elements without ports cannot participate in flow networks.
- **ALWAYS** create at least two ports per segment element (one inlet, one outlet). Segments are flow-through elements.
- **NEVER** connect two ports with the same flow direction. A `SOURCE` port MUST connect to a `SINK` port.
- **NEVER** use `IfcDistributionSystem` in IFC2X3 models. Use `ifc_class="IfcSystem"` for IFC2X3 compatibility.
- **ALWAYS** use `ifcopenshell.api.run("system.add_system", ...)` to create systems. NEVER use `model.create_entity("IfcDistributionSystem")` directly — it skips required relationship setup.
- **ALWAYS** assign elements to a system AFTER creating port connections. The system is a logical grouping; ports define the physical topology.
- **NEVER** assume `system.remove_system` deletes contained elements. It ONLY removes the system grouping; elements and their ports remain in the model.
- **ALWAYS** use `ifcopenshell.api.run("root.create_entity", ...)` for MEP elements, not `model.create_entity()`. The API sets GlobalId, ownership, and validates predefined types.
- **ALWAYS** pass `products` as a **list** to `system.assign_system` (v0.8+ convention).

### Decision Tree: MEP Workflow

```
Creating an MEP system?
├── Step 1: What IFC schema version?
│   ├── IFC2X3 → Use ifc_class="IfcSystem" for add_system
│   └── IFC4 / IFC4X3 → Use ifc_class="IfcDistributionSystem" (default)
│
├── Step 2: Create the system
│   └── system.add_system → system.edit_system (set Name + PredefinedType)
│
├── Step 3: Create distribution elements
│   ├── Duct/Pipe/Cable? → IfcDuctSegment, IfcPipeSegment, IfcCableSegment
│   ├── Elbow/Tee/Junction? → IfcDuctFitting, IfcPipeFitting, IfcCableFitting
│   ├── Air terminal/Fixture? → IfcAirTerminal, IfcSanitaryTerminal
│   ├── Valve/Damper/Switch? → IfcValve, IfcDamper, IfcSwitchingDevice
│   ├── Pump/Fan? → IfcPump, IfcFan
│   ├── Boiler/Chiller? → IfcBoiler, IfcChiller
│   └── Tank/Vessel? → IfcTank
│
├── Step 4: Create ports on each element
│   └── system.add_port (per connection point)
│
├── Step 5: Connect ports between elements
│   └── system.connect_port (port1, port2, direction)
│
├── Step 6: Assign elements to system
│   └── system.assign_system (products=[...], system=system)
│
├── Step 7: Assign flow controls (optional)
│   └── system.assign_flow_control (damper→duct, valve→pipe)
│
└── Step 8: Add properties and placement
    ├── pset.add_pset / pset.edit_pset (MEP property sets)
    └── geometry (placement, representation)
```

### Decision Tree: Which Distribution Element Class

```
What type of MEP element?
├── Carries flow (segments)?
│   ├── Air/gas → IfcDuctSegment
│   ├── Liquid → IfcPipeSegment
│   ├── Electrical → IfcCableSegment
│   └── Cable carrier → IfcCableCarrierSegment
│
├── Changes direction/splits (fittings)?
│   ├── Air/gas → IfcDuctFitting
│   ├── Liquid → IfcPipeFitting
│   ├── Electrical → IfcCableFitting
│   └── Cable carrier → IfcCableCarrierFitting
│
├── End point (terminals)?
│   ├── Air supply/return → IfcAirTerminal
│   ├── Plumbing fixture → IfcSanitaryTerminal
│   ├── Electrical outlet → IfcOutlet
│   ├── Light fixture → IfcLightFixture
│   ├── Fire sprinkler → IfcFireSuppressionTerminal
│   └── Communication jack → IfcCommunicationsAppliance
│
├── Controls flow (controllers)?
│   ├── Airflow → IfcDamper
│   ├── Liquid flow → IfcValve
│   ├── Electrical → IfcSwitchingDevice / IfcProtectiveDevice
│   └── Flow meter → IfcFlowMeter
│
├── Moves medium (movers)?
│   ├── Air → IfcFan
│   ├── Liquid → IfcPump
│   └── Gas → IfcCompressor
│
├── Converts energy?
│   ├── Heating → IfcBoiler
│   ├── Cooling → IfcChiller
│   ├── Heat exchange → IfcHeatExchanger
│   ├── Cooling tower → IfcCoolingTower
│   ├── Air handling → IfcAirToAirHeatRecovery / IfcUnitaryEquipment
│   └── Electrical → IfcTransformer / IfcElectricGenerator
│
├── Stores medium?
│   ├── Liquid → IfcTank
│   └── Electrical → IfcElectricFlowStorageDevice
│
├── Treats medium?
│   ├── Air filter → IfcFilter
│   └── Grease/oil trap → IfcInterceptor
│
└── Senses/monitors (control elements)?
    ├── Temperature → IfcSensor (TEMPERATURESENSOR)
    ├── Pressure → IfcSensor (PRESSURESENSOR)
    ├── Flow rate → IfcSensor (FLOWSENSOR)
    ├── Smoke → IfcSensor (SMOKESENSOR)
    └── Actuator → IfcActuator
```

---

## Essential Patterns

### Pattern 1: Complete HVAC System (IFC4)

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")

# Bootstrap (project, units, contexts, spatial hierarchy)
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="MEP Project")
ifcopenshell.api.run("unit.assign_unit", model)
model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)
site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Building A")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Level 1")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# Step 1: Create distribution system
hvac = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=hvac,
    attributes={"Name": "Supply Air System", "PredefinedType": "VENTILATION"})

# Step 2: Create elements
ahu = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcUnitaryEquipment", name="AHU-01")
duct1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="SD-01")
duct2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="SD-02")
terminal = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAirTerminal", name="AT-01")

# Step 3: Place in spatial structure
ifcopenshell.api.run("spatial.assign_container", model,
    products=[ahu, duct1, duct2, terminal], relating_structure=storey)

# Step 4: Create ports (inlet + outlet per segment)
ahu_out = ifcopenshell.api.run("system.add_port", model, element=ahu)
d1_in = ifcopenshell.api.run("system.add_port", model, element=duct1)
d1_out = ifcopenshell.api.run("system.add_port", model, element=duct1)
d2_in = ifcopenshell.api.run("system.add_port", model, element=duct2)
d2_out = ifcopenshell.api.run("system.add_port", model, element=duct2)
term_in = ifcopenshell.api.run("system.add_port", model, element=terminal)

# Step 5: Connect ports (AHU → Duct1 → Duct2 → Terminal)
ifcopenshell.api.run("system.connect_port", model,
    port1=ahu_out, port2=d1_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=d1_out, port2=d2_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=d2_out, port2=term_in, direction="SOURCE")

# Step 6: Assign all elements to the system
ifcopenshell.api.run("system.assign_system", model,
    products=[ahu, duct1, duct2, terminal], system=hvac)
```

### Pattern 2: Piping Network with Fittings

```python
# IFC4 / IFC4X3: requires completed bootstrap
# Create plumbing system
plumbing = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=plumbing,
    attributes={"Name": "Hot Water Supply", "PredefinedType": "DOMESTICHOTWATER"})

# Create pipe elements
pipe1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="HWS-P01")
elbow = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeFitting", name="HWS-E01")
pipe2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="HWS-P02")
valve = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcValve", name="HWS-V01")

# Ports: pipes need 2, fittings need 2+, terminals need 1
p1_in = ifcopenshell.api.run("system.add_port", model, element=pipe1)
p1_out = ifcopenshell.api.run("system.add_port", model, element=pipe1)
elb_in = ifcopenshell.api.run("system.add_port", model, element=elbow)
elb_out = ifcopenshell.api.run("system.add_port", model, element=elbow)
p2_in = ifcopenshell.api.run("system.add_port", model, element=pipe2)
p2_out = ifcopenshell.api.run("system.add_port", model, element=pipe2)

# Connect: Pipe1 → Elbow → Pipe2
ifcopenshell.api.run("system.connect_port", model,
    port1=p1_out, port2=elb_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=elb_out, port2=p2_in, direction="SOURCE")

# Assign flow control (valve controls pipe)
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=pipe1, related_flow_control=valve)

# Assign to system
ifcopenshell.api.run("system.assign_system", model,
    products=[pipe1, elbow, pipe2, valve], system=plumbing)
```

### Pattern 3: Tee Fitting (3 Ports)

```python
# IFC4 / IFC4X3
tee = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeFitting", name="HWS-T01")

# Tee fittings have 3 ports: inlet, outlet-main, outlet-branch
tee_in = ifcopenshell.api.run("system.add_port", model, element=tee)
tee_out1 = ifcopenshell.api.run("system.add_port", model, element=tee)
tee_out2 = ifcopenshell.api.run("system.add_port", model, element=tee)

# Connect: upstream pipe → tee inlet
# tee outlet1 → downstream main
# tee outlet2 → downstream branch
ifcopenshell.api.run("system.connect_port", model,
    port1=upstream_pipe_out, port2=tee_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=tee_out1, port2=main_pipe_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=tee_out2, port2=branch_pipe_in, direction="SOURCE")
```

### Pattern 4: IFC2X3 System (Backward Compatible)

```python
# IFC2X3: use IfcSystem instead of IfcDistributionSystem
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
# ... bootstrap ...

system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcSystem")  # REQUIRED for IFC2X3
ifcopenshell.api.run("system.edit_system", model,
    system=system,
    attributes={"Name": "Heating System"})
# NOTE: PredefinedType is NOT available on IfcSystem in IFC2X3.
# Use ObjectType attribute for classification instead.
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=system,
    attributes={"ObjectType": "HEATING"})
```

---

## Distribution System Types (IFC4 / IFC4X3)

Set via `PredefinedType` attribute on `IfcDistributionSystem`:

| PredefinedType | Domain | Description |
|----------------|--------|-------------|
| `VENTILATION` | HVAC | Supply/return/exhaust air systems |
| `HEATING` | HVAC | Hot water heating, radiator circuits |
| `COOLING` | HVAC | Chilled water, refrigerant loops |
| `AIRCONDITIONING` | HVAC | Combined heating/cooling air systems |
| `EXHAUST` | HVAC | Kitchen/fume exhaust systems |
| `DOMESTICHOTWATER` | Plumbing | Hot water supply |
| `DOMESTICCOLDWATER` | Plumbing | Cold water supply |
| `DRAINAGE` | Plumbing | Gravity drainage, waste water |
| `WASTEWATER` | Plumbing | Waste water systems |
| `STORMWATER` | Plumbing | Rainwater drainage |
| `SEWAGE` | Plumbing | Sewer connections |
| `RAINWATER` | Plumbing | Rainwater collection |
| `ELECTRICAL` | Electrical | Power distribution |
| `LIGHTING` | Electrical | Lighting circuits |
| `POWERGENERATION` | Electrical | On-site power generation |
| `LIGHTNINGPROTECTION` | Electrical | Lightning protection systems |
| `EARTHING` | Electrical | Grounding/earthing |
| `FIREPROTECTION` | Fire | Sprinklers, fire suppression |
| `SECURITY` | Security | Access control, CCTV |
| `COMMUNICATION` | Comms | Data/telephone networks |
| `DATA` | Comms | Data networks |
| `TV` | Comms | Television distribution |
| `SIGNAL` | Controls | Signal/control systems |
| `CONTROL` | Controls | Building automation |
| `GAS` | Fuel | Natural gas distribution |
| `FUEL` | Fuel | Fuel supply systems |
| `COMPRESSEDAIR` | Pneumatic | Compressed air distribution |
| `VACUUM` | Pneumatic | Vacuum systems |
| `REFRIGERATION` | HVAC | Refrigeration systems |
| `CHILLEDWATER` | HVAC | Chilled water distribution |
| `CONDENSERWATER` | HVAC | Condenser water loops |
| `CHEMICAL` | Process | Chemical distribution |
| `DISPOSAL` | Waste | Waste disposal systems |
| `HAZARDOUS` | Safety | Hazardous material systems |
| `CONVEYING` | Transport | Material conveying |
| `USERDEFINED` | Custom | Set ObjectType attribute |
| `NOTDEFINED` | Default | Unspecified |

---

## Port Flow Directions

| Direction | Meaning | When to Use |
|-----------|---------|-------------|
| `SOURCE` | Flow exits through this port | Outlet ports, downstream direction |
| `SINK` | Flow enters through this port | Inlet ports, upstream direction |
| `SOURCEANDSINK` | Bidirectional flow | Reversible systems, heat exchangers |
| `NOTDEFINED` | Direction not specified | When flow direction is unknown |

**Rule:** When connecting two ports, `direction` on `connect_port` describes the relationship from `port1`'s perspective. If `port1` is a SOURCE, flow goes from port1 to port2.

---

## System Traversal (ifcopenshell.util.system)

```python
# IFC4 / IFC4X3
import ifcopenshell.util.system

# Get all elements in a system
elements = ifcopenshell.util.system.get_system_elements(hvac)

# Get all systems an element belongs to
systems = ifcopenshell.util.system.get_element_systems(duct1)

# Get ports on an element (optionally filter by direction)
all_ports = ifcopenshell.util.system.get_ports(duct1)
outlets = ifcopenshell.util.system.get_ports(duct1, flow_direction="SOURCE")
inlets = ifcopenshell.util.system.get_ports(duct1, flow_direction="SINK")

# Traverse connections: downstream / upstream
downstream = ifcopenshell.util.system.get_connected_to(duct1)
upstream = ifcopenshell.util.system.get_connected_from(duct1)

# Get element that owns a port
owner = ifcopenshell.util.system.get_port_element(d1_out)

# Get port connected to a given port
partner = ifcopenshell.util.system.get_connected_port(d1_out)
```

---

## MEP Property Sets

### Standard Property Sets for Distribution Elements

```python
# IFC4 / IFC4X3: Add MEP-specific properties
# Duct segment properties
pset = ifcopenshell.api.run("pset.add_pset", model,
    product=duct1, name="Pset_DuctSegmentTypeCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
    "NominalDiameter": 0.315,      # meters
    "InnerDiameter": 0.313,
    "OuterDiameter": 0.315,
    "Length": 3.0,
})

# Pipe segment properties
pset_pipe = ifcopenshell.api.run("pset.add_pset", model,
    product=pipe1, name="Pset_PipeSegmentTypeCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset_pipe, properties={
    "NominalDiameter": 0.025,
    "InnerDiameter": 0.021,
    "OuterDiameter": 0.025,
    "WorkingPressure": 600000.0,    # Pascals
    "FlowRateRange": "0.1 - 0.5",  # m3/s
})

# Quantity set for a duct
qto = ifcopenshell.api.run("pset.add_qto", model,
    product=duct1, name="Qto_DuctSegmentBaseQuantities")
ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
    "Length": 3.0,
    "GrossCrossSectionArea": 0.078,
    "OuterSurfaceArea": 2.97,
})
```

### Common MEP Pset Names

| Element Type | Property Set | Key Properties |
|-------------|-------------|----------------|
| Duct segments | `Pset_DuctSegmentTypeCommon` | NominalDiameter, Length |
| Pipe segments | `Pset_PipeSegmentTypeCommon` | NominalDiameter, WorkingPressure |
| Air terminals | `Pset_AirTerminalTypeCommon` | AirFlowRateRange, NominalAirFlowRate |
| Valves | `Pset_ValveTypeCommon` | ValvePattern, WorkingPressure |
| Pumps | `Pset_PumpTypeCommon` | FlowRateRange, NominalRotationSpeed |
| Fans | `Pset_FanTypeCommon` | NominalAirFlowRate, NominalTotalPressure |
| Boilers | `Pset_BoilerTypeCommon` | NominalEnergyConsumption, WaterTemperatureRange |

---

## Version Differences

| Feature | IFC2X3 | IFC4 | IFC4X3 |
|---------|--------|------|--------|
| System class | `IfcSystem` | `IfcDistributionSystem` | `IfcDistributionSystem` |
| PredefinedType on system | Not available | Full enum (37 values) | Full enum + additions |
| Building system | Not available | `IfcBuildingSystem` | `IfcBuildingSystem` |
| Port nesting | `IfcRelNests` | `IfcRelNests` | `IfcRelNests` |
| Port connection | `IfcRelConnectsPorts` | `IfcRelConnectsPorts` | `IfcRelConnectsPorts` |
| Flow direction enum | Same | Same | Same |

### Version-Safe Pattern

```python
# Detect schema and choose correct system class
schema = model.schema
if schema == "IFC2X3":
    system = ifcopenshell.api.run("system.add_system", model,
        ifc_class="IfcSystem")
else:
    system = ifcopenshell.api.run("system.add_system", model,
        ifc_class="IfcDistributionSystem")
```

---

## Dependency

This skill depends on:
- **ifcos-syntax-api** for `api.run()` invocation patterns, module table, and parameter conventions
- **ifcos-syntax-fileio** for file creation, writing, and transaction management

---

## Reference Links

- [API Method Signatures](references/methods.md) -- Complete signatures for all 12 system API functions
- [Working Code Examples](references/examples.md) -- End-to-end MEP system examples
- [Anti-Patterns](references/anti-patterns.md) -- Common MEP modeling mistakes and how to avoid them

### Official Sources

- https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/system/
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/schema/ifcsharedbldgserviceelements/
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/schema/ifchvacdomain/
