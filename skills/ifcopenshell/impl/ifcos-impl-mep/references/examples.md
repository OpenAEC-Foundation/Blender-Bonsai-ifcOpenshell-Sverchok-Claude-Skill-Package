# ifcos-impl-mep -- Examples Reference

Working code examples for MEP system modeling with IfcOpenShell. All examples verified against the ifcopenshell.api.system module documentation.

---

## Example 1: Complete HVAC Supply Air System

Creates a supply air system with AHU, ductwork, damper, and air terminal.

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api
import numpy

# === Bootstrap ===
model = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProject", name="HVAC Demo")
ifcopenshell.api.run("unit.assign_unit", model)

model3d = ifcopenshell.api.run("context.add_context", model, context_type="Model")
body = ifcopenshell.api.run("context.add_context", model,
    context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=model3d)

site = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSite", name="Site")
building = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuilding", name="Office Building")
storey = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBuildingStorey", name="Level 1")
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[site], relating_object=project)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[building], relating_object=site)
ifcopenshell.api.run("aggregate.assign_object", model,
    products=[storey], relating_object=building)

# === Create HVAC System ===
hvac = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=hvac,
    attributes={"Name": "Supply Air - Zone A", "PredefinedType": "VENTILATION"})

# === Create Distribution Elements ===
ahu = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcUnitaryEquipment", name="AHU-01")
duct_main = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="SD-MAIN-01")
duct_branch = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="SD-BR-01")
tee = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctFitting", name="SD-TEE-01")
damper = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDamper", name="SD-DAM-01")
terminal1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAirTerminal", name="AT-01")
terminal2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAirTerminal", name="AT-02")

# === Place in Spatial Structure ===
all_elements = [ahu, duct_main, duct_branch, tee, damper, terminal1, terminal2]
ifcopenshell.api.run("spatial.assign_container", model,
    products=all_elements, relating_structure=storey)

# === Create Ports ===
# AHU: 1 supply outlet
ahu_out = ifcopenshell.api.run("system.add_port", model, element=ahu)

# Main duct: inlet + outlet
dm_in = ifcopenshell.api.run("system.add_port", model, element=duct_main)
dm_out = ifcopenshell.api.run("system.add_port", model, element=duct_main)

# Tee: 1 inlet + 2 outlets
tee_in = ifcopenshell.api.run("system.add_port", model, element=tee)
tee_out1 = ifcopenshell.api.run("system.add_port", model, element=tee)
tee_out2 = ifcopenshell.api.run("system.add_port", model, element=tee)

# Branch duct: inlet + outlet
db_in = ifcopenshell.api.run("system.add_port", model, element=duct_branch)
db_out = ifcopenshell.api.run("system.add_port", model, element=duct_branch)

# Terminals: 1 inlet each
t1_in = ifcopenshell.api.run("system.add_port", model, element=terminal1)
t2_in = ifcopenshell.api.run("system.add_port", model, element=terminal2)

# === Connect Ports ===
# AHU → Main Duct → Tee → Terminal1 (straight)
#                        → Branch Duct → Terminal2
ifcopenshell.api.run("system.connect_port", model,
    port1=ahu_out, port2=dm_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=dm_out, port2=tee_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=tee_out1, port2=t1_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=tee_out2, port2=db_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=db_out, port2=t2_in, direction="SOURCE")

# === Assign Flow Control ===
# Damper controls airflow in the main duct
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct_main, related_flow_control=damper)

# === Assign Elements to System ===
ifcopenshell.api.run("system.assign_system", model,
    products=all_elements, system=hvac)

# === Add Properties ===
pset_duct = ifcopenshell.api.run("pset.add_pset", model,
    product=duct_main, name="Pset_DuctSegmentTypeCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset_duct, properties={
    "NominalDiameter": 0.400,
    "Length": 6.0,
})

pset_term = ifcopenshell.api.run("pset.add_pset", model,
    product=terminal1, name="Pset_AirTerminalTypeCommon")
ifcopenshell.api.run("pset.edit_pset", model, pset=pset_term, properties={
    "AirFlowRateRange": "0.05 - 0.15",
})

# === Place Elements in 3D Space ===
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=ahu, matrix=numpy.eye(4))

matrix_duct = numpy.eye(4)
matrix_duct[0, 3] = 2.0  # Offset 2m in X
ifcopenshell.api.run("geometry.edit_object_placement", model,
    product=duct_main, matrix=matrix_duct)

# === Save ===
model.write("hvac_system.ifc")
```

---

## Example 2: Domestic Hot Water Piping

Creates a hot water supply system with pump, pipes, fittings, and sanitary terminals.

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
# ... assume bootstrap completed ...

# Create system
dhw = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=dhw,
    attributes={"Name": "Domestic Hot Water", "PredefinedType": "DOMESTICHOTWATER"})

# Create elements
boiler = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcBoiler", name="B-01")
pump = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPump", name="P-01")
pipe1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="HW-P01")
pipe2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="HW-P02")
pipe3 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="HW-P03")
valve = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcValve", name="V-01")
tee = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeFitting", name="T-01")
sink = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSanitaryTerminal", name="SINK-01")
shower = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSanitaryTerminal", name="SHOWER-01")

# Create ports
boiler_out = ifcopenshell.api.run("system.add_port", model, element=boiler)
pump_in = ifcopenshell.api.run("system.add_port", model, element=pump)
pump_out = ifcopenshell.api.run("system.add_port", model, element=pump)
p1_in = ifcopenshell.api.run("system.add_port", model, element=pipe1)
p1_out = ifcopenshell.api.run("system.add_port", model, element=pipe1)
p2_in = ifcopenshell.api.run("system.add_port", model, element=pipe2)
p2_out = ifcopenshell.api.run("system.add_port", model, element=pipe2)
p3_in = ifcopenshell.api.run("system.add_port", model, element=pipe3)
p3_out = ifcopenshell.api.run("system.add_port", model, element=pipe3)
tee_in = ifcopenshell.api.run("system.add_port", model, element=tee)
tee_out1 = ifcopenshell.api.run("system.add_port", model, element=tee)
tee_out2 = ifcopenshell.api.run("system.add_port", model, element=tee)
sink_in = ifcopenshell.api.run("system.add_port", model, element=sink)
shower_in = ifcopenshell.api.run("system.add_port", model, element=shower)

# Connect: Boiler → Pump → Pipe1 → Tee → Pipe2 → Sink
#                                       → Pipe3 → Shower
ifcopenshell.api.run("system.connect_port", model,
    port1=boiler_out, port2=pump_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=pump_out, port2=p1_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=p1_out, port2=tee_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=tee_out1, port2=p2_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=p2_out, port2=sink_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=tee_out2, port2=p3_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=p3_out, port2=shower_in, direction="SOURCE")

# Assign flow control (valve controls pipe1)
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=pipe1, related_flow_control=valve)

# Assign all to system
all_elems = [boiler, pump, pipe1, pipe2, pipe3, valve, tee, sink, shower]
ifcopenshell.api.run("system.assign_system", model,
    products=all_elems, system=dhw)

# Add pipe properties
for pipe, length in [(pipe1, 4.0), (pipe2, 2.5), (pipe3, 3.0)]:
    pset = ifcopenshell.api.run("pset.add_pset", model,
        product=pipe, name="Pset_PipeSegmentTypeCommon")
    ifcopenshell.api.run("pset.edit_pset", model, pset=pset, properties={
        "NominalDiameter": 0.025,
        "InnerDiameter": 0.021,
        "OuterDiameter": 0.025,
        "WorkingPressure": 600000.0,
    })
    qto = ifcopenshell.api.run("pset.add_qto", model,
        product=pipe, name="Qto_PipeSegmentBaseQuantities")
    ifcopenshell.api.run("pset.edit_qto", model, qto=qto, properties={
        "Length": length,
    })
```

---

## Example 3: Electrical Power Distribution

Creates an electrical distribution system with transformer, distribution board, cables, and outlets.

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
# ... assume bootstrap completed ...

# Create electrical system
elec = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=elec,
    attributes={"Name": "Power Distribution - Floor 1", "PredefinedType": "ELECTRICAL"})

# Create elements
transformer = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcTransformer", name="TR-01")
dist_board = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcElectricDistributionBoard", name="DB-01")
cable1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcCableSegment", name="C-01")
cable2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcCableSegment", name="C-02")
cable3 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcCableSegment", name="C-03")
breaker = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcProtectiveDevice", name="MCB-01")
outlet1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOutlet", name="OUT-01")
outlet2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcOutlet", name="OUT-02")
light = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcLightFixture", name="LF-01")

# Create ports and connections
tr_out = ifcopenshell.api.run("system.add_port", model, element=transformer)
c1_in = ifcopenshell.api.run("system.add_port", model, element=cable1)
c1_out = ifcopenshell.api.run("system.add_port", model, element=cable1)
db_in = ifcopenshell.api.run("system.add_port", model, element=dist_board)
db_out1 = ifcopenshell.api.run("system.add_port", model, element=dist_board)
db_out2 = ifcopenshell.api.run("system.add_port", model, element=dist_board)
c2_in = ifcopenshell.api.run("system.add_port", model, element=cable2)
c2_out = ifcopenshell.api.run("system.add_port", model, element=cable2)
c3_in = ifcopenshell.api.run("system.add_port", model, element=cable3)
c3_out = ifcopenshell.api.run("system.add_port", model, element=cable3)
out1_in = ifcopenshell.api.run("system.add_port", model, element=outlet1)
out2_in = ifcopenshell.api.run("system.add_port", model, element=outlet2)
light_in = ifcopenshell.api.run("system.add_port", model, element=light)

# Connect: Transformer → Cable1 → DistBoard → Cable2 → Outlet1
#                                             → Cable3 → Outlet2/Light
ifcopenshell.api.run("system.connect_port", model,
    port1=tr_out, port2=c1_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=c1_out, port2=db_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=db_out1, port2=c2_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=c2_out, port2=out1_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=db_out2, port2=c3_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=c3_out, port2=light_in, direction="SOURCE")

# Breaker protects distribution board
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=dist_board, related_flow_control=breaker)

# Assign all to system
all_elems = [transformer, dist_board, cable1, cable2, cable3,
             breaker, outlet1, outlet2, light]
ifcopenshell.api.run("system.assign_system", model,
    products=all_elems, system=elec)
```

---

## Example 4: System Traversal and Querying

Demonstrates how to query and traverse an existing MEP system.

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.util.system

model = ifcopenshell.open("building_mep.ifc")

# Find all distribution systems
systems = model.by_type("IfcDistributionSystem")
for sys in systems:
    print(f"System: {sys.Name} ({sys.PredefinedType})")

    # Get all elements in this system
    elements = ifcopenshell.util.system.get_system_elements(sys)
    print(f"  Elements: {len(elements)}")

    for elem in elements:
        print(f"    {elem.is_a()} - {elem.Name}")

        # Get ports for this element
        ports = ifcopenshell.util.system.get_ports(elem)
        for port in ports:
            connected = ifcopenshell.util.system.get_connected_port(port)
            if connected:
                connected_elem = ifcopenshell.util.system.get_port_element(connected)
                print(f"      Port → {connected_elem.Name}")

# Trace flow path from a specific element
start_element = model.by_type("IfcPump")[0]
print(f"\nTracing downstream from: {start_element.Name}")

visited = set()
queue = [start_element]
while queue:
    current = queue.pop(0)
    if current.id() in visited:
        continue
    visited.add(current.id())
    print(f"  → {current.is_a()}: {current.Name}")

    downstream = ifcopenshell.util.system.get_connected_to(current)
    queue.extend(downstream)
```

---

## Example 5: IFC2X3 Compatible System

Demonstrates creating MEP systems in IFC2X3 where IfcDistributionSystem is not available.

```python
# IFC2X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
# ... bootstrap for IFC2X3 ...

# IFC2X3: Use IfcSystem instead of IfcDistributionSystem
system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=system,
    attributes={"Name": "Heating System"})

# IFC2X3 has no PredefinedType on IfcSystem — use ObjectType
ifcopenshell.api.run("attribute.edit_attributes", model,
    product=system,
    attributes={"ObjectType": "HEATING"})

# Elements, ports, and connections work the same way
pipe = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="H-P01")
p_in = ifcopenshell.api.run("system.add_port", model, element=pipe)
p_out = ifcopenshell.api.run("system.add_port", model, element=pipe)

ifcopenshell.api.run("system.assign_system", model,
    products=[pipe], system=system)
```

---

## Example 6: Fire Protection System with Sensors

Creates a fire suppression system with sensors assigned as control elements.

```python
# IFC4 / IFC4X3
import ifcopenshell
import ifcopenshell.api

model = ifcopenshell.api.run("project.create_file", version="IFC4")
# ... assume bootstrap completed ...

# Create fire protection system
fire = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=fire,
    attributes={"Name": "Sprinkler System - Zone 1", "PredefinedType": "FIREPROTECTION"})

# Create elements
riser = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="FP-RISER-01")
branch = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="FP-BR-01")
sprinkler1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcFireSuppressionTerminal", name="SPR-01")
sprinkler2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcFireSuppressionTerminal", name="SPR-02")

# Smoke sensor (control element, not a flow element)
smoke_sensor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSensor", name="SM-01")

# Alarm
alarm = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcAlarm", name="AL-01")

# Create ports and connect flow path
r_in = ifcopenshell.api.run("system.add_port", model, element=riser)
r_out = ifcopenshell.api.run("system.add_port", model, element=riser)
b_in = ifcopenshell.api.run("system.add_port", model, element=branch)
b_out1 = ifcopenshell.api.run("system.add_port", model, element=branch)
b_out2 = ifcopenshell.api.run("system.add_port", model, element=branch)
spr1_in = ifcopenshell.api.run("system.add_port", model, element=sprinkler1)
spr2_in = ifcopenshell.api.run("system.add_port", model, element=sprinkler2)

ifcopenshell.api.run("system.connect_port", model,
    port1=r_out, port2=b_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=b_out1, port2=spr1_in, direction="SOURCE")
ifcopenshell.api.run("system.connect_port", model,
    port1=b_out2, port2=spr2_in, direction="SOURCE")

# Sensors and alarms are assigned as flow controls, NOT via ports
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=branch, related_flow_control=smoke_sensor)
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=branch, related_flow_control=alarm)

# Assign all to system (including control elements)
all_elems = [riser, branch, sprinkler1, sprinkler2, smoke_sensor, alarm]
ifcopenshell.api.run("system.assign_system", model,
    products=all_elems, system=fire)
```
