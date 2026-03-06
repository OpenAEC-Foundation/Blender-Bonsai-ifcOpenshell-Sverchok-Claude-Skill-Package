# ifcos-impl-mep -- Anti-Patterns Reference

Common mistakes when modeling MEP systems with IfcOpenShell, with explanations and correct alternatives.

---

## AP-01: Connecting Elements Without Ports

**WRONG:**
```python
# Attempting to connect elements directly -- DOES NOT WORK
# There is no "connect_element" function in the API
duct1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Duct 1")
duct2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Duct 2")
# Missing: ports are never created
# connect_port will fail because there are no ports to connect
```

**CORRECT:**
```python
duct1 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Duct 1")
duct2 = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcDuctSegment", name="Duct 2")

# ALWAYS create ports FIRST
d1_in = ifcopenshell.api.run("system.add_port", model, element=duct1)
d1_out = ifcopenshell.api.run("system.add_port", model, element=duct1)
d2_in = ifcopenshell.api.run("system.add_port", model, element=duct2)
d2_out = ifcopenshell.api.run("system.add_port", model, element=duct2)

# THEN connect ports
ifcopenshell.api.run("system.connect_port", model,
    port1=d1_out, port2=d2_in, direction="SOURCE")
```

**Why:** IFC uses a port-based connection model. Distribution elements MUST have ports to participate in flow networks. Without ports, there is no way to represent physical connection points.

---

## AP-02: Connecting Ports with Same Flow Direction

**WRONG:**
```python
# Both ports are SOURCE -- invalid connection
ifcopenshell.api.run("system.connect_port", model,
    port1=duct1_outlet, port2=duct2_outlet, direction="SOURCE")
```

**CORRECT:**
```python
# SOURCE connects to SINK (outlet to inlet)
ifcopenshell.api.run("system.connect_port", model,
    port1=duct1_outlet, port2=duct2_inlet, direction="SOURCE")
```

**Why:** A SOURCE port (outlet) MUST connect to a SINK port (inlet) for the connection to represent a valid flow path. Connecting two outlets or two inlets is physically nonsensical and produces invalid IFC data.

---

## AP-03: Using IfcDistributionSystem in IFC2X3

**WRONG:**
```python
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
# IfcDistributionSystem does NOT exist in IFC2X3 -- will raise error
system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
```

**CORRECT:**
```python
model = ifcopenshell.api.run("project.create_file", version="IFC2X3")
# Use IfcSystem for IFC2X3
system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcSystem")
```

**Why:** `IfcDistributionSystem` was introduced in IFC4. In IFC2X3, only `IfcSystem` is available. The `PredefinedType` attribute is also not available on `IfcSystem`; use `ObjectType` instead for classification.

---

## AP-04: Creating Segments with Only One Port

**WRONG:**
```python
# Segment with only one port -- cannot pass flow through
pipe = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="Pipe")
port = ifcopenshell.api.run("system.add_port", model, element=pipe)
# Only 1 port -- this pipe is a dead end, cannot connect both upstream and downstream
```

**CORRECT:**
```python
pipe = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcPipeSegment", name="Pipe")
# Segments ALWAYS need at least 2 ports: inlet + outlet
pipe_in = ifcopenshell.api.run("system.add_port", model, element=pipe)
pipe_out = ifcopenshell.api.run("system.add_port", model, element=pipe)
```

**Why:** Segments are flow-through elements. They need an inlet port AND an outlet port to connect to both upstream and downstream elements. Only terminal elements (end-of-line) have a single port.

---

## AP-05: Creating System Entities Directly

**WRONG:**
```python
# Using model.create_entity bypasses API relationship management
system = model.create_entity("IfcDistributionSystem", Name="HVAC")
# Missing: no IfcRelAssignsToGroup, no ownership, no GlobalId management
```

**CORRECT:**
```python
system = ifcopenshell.api.run("system.add_system", model,
    ifc_class="IfcDistributionSystem")
ifcopenshell.api.run("system.edit_system", model,
    system=system, attributes={"Name": "HVAC"})
```

**Why:** `system.add_system` handles GlobalId generation, ownership history, and relationship setup. Using `model.create_entity()` directly skips all of this, producing an incomplete entity.

---

## AP-06: Assigning Elements to System Before Creating Connections

**WRONG (order):**
```python
# Assigning to system before port connections are established
ifcopenshell.api.run("system.assign_system", model,
    products=[duct1, duct2], system=hvac)

# Then creating ports and connections -- the system grouping
# exists but the flow topology is incomplete
d1_out = ifcopenshell.api.run("system.add_port", model, element=duct1)
d2_in = ifcopenshell.api.run("system.add_port", model, element=duct2)
ifcopenshell.api.run("system.connect_port", model,
    port1=d1_out, port2=d2_in, direction="SOURCE")
```

**CORRECT (order):**
```python
# 1. Create elements
# 2. Create ports
# 3. Connect ports
# 4. THEN assign to system (last step)
d1_out = ifcopenshell.api.run("system.add_port", model, element=duct1)
d2_in = ifcopenshell.api.run("system.add_port", model, element=duct2)
ifcopenshell.api.run("system.connect_port", model,
    port1=d1_out, port2=d2_in, direction="SOURCE")
ifcopenshell.api.run("system.assign_system", model,
    products=[duct1, duct2], system=hvac)
```

**Why:** While technically both orders produce valid IFC, establishing the physical topology (ports + connections) BEFORE the logical grouping (system assignment) is the recommended workflow. It ensures the flow network is complete when the system is finalized.

---

## AP-07: Assuming remove_system Deletes Elements

**WRONG:**
```python
# Expecting elements to be deleted with the system
ifcopenshell.api.run("system.remove_system", model, system=hvac)
# duct1, duct2, terminal still exist in the model!
# They are now orphaned (not in any system) but still present
```

**CORRECT:**
```python
# To remove everything: remove elements first, then the system
elements = ifcopenshell.util.system.get_system_elements(hvac)
for elem in elements:
    ifcopenshell.api.run("root.remove_product", model, product=elem)
ifcopenshell.api.run("system.remove_system", model, system=hvac)
```

**Why:** `remove_system` only removes the `IfcDistributionSystem` entity and its `IfcRelAssignsToGroup` relationships. The distribution elements, their ports, and port connections all remain in the model. This is by design -- elements may belong to multiple systems.

---

## AP-08: Passing Single Element Instead of List to assign_system

**WRONG (v0.8+):**
```python
# Single element without list wrapping -- KeyError in v0.8+
ifcopenshell.api.run("system.assign_system", model,
    product=duct1, system=hvac)
```

**CORRECT:**
```python
# ALWAYS use products (plural) with a list
ifcopenshell.api.run("system.assign_system", model,
    products=[duct1], system=hvac)
```

**Why:** Since IfcOpenShell v0.8, `assign_system` expects a `products` list parameter (plural), not a `product` singular parameter. This is consistent with other relationship functions like `spatial.assign_container` and `aggregate.assign_object`.

---

## AP-09: Using Ports for Control Elements (Sensors, Actuators)

**WRONG:**
```python
# Creating ports on a sensor -- sensors don't have flow ports
sensor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSensor", name="Temp Sensor")
sensor_port = ifcopenshell.api.run("system.add_port", model, element=sensor)
# Then trying to connect sensor port to a duct port -- incorrect approach
```

**CORRECT:**
```python
sensor = ifcopenshell.api.run("root.create_entity", model,
    ifc_class="IfcSensor", name="Temp Sensor")
# Use assign_flow_control to associate sensor with the element it monitors
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct1, related_flow_control=sensor)
```

**Why:** Control elements (sensors, actuators, alarms, controllers) do NOT participate in the flow network. They monitor or control flow elements. Use `assign_flow_control` to create the `IfcRelFlowControlElements` relationship instead of port connections.

---

## AP-10: Forgetting to Set PredefinedType on System

**WRONG:**
```python
system = ifcopenshell.api.run("system.add_system", model)
ifcopenshell.api.run("system.edit_system", model,
    system=system, attributes={"Name": "HVAC Supply"})
# PredefinedType defaults to None -- downstream tools cannot classify this system
```

**CORRECT:**
```python
system = ifcopenshell.api.run("system.add_system", model)
ifcopenshell.api.run("system.edit_system", model,
    system=system,
    attributes={"Name": "HVAC Supply", "PredefinedType": "VENTILATION"})
```

**Why:** The `PredefinedType` attribute on `IfcDistributionSystem` classifies the system for downstream MEP analysis tools, clash detection, and IFC viewers. Without it, the system appears as an unclassified group. ALWAYS set `PredefinedType` in IFC4+ models.

---

## AP-11: Assigning Flow Control to Multiple Elements

**WRONG:**
```python
# A flow control can only be assigned to ONE flow element
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct1, related_flow_control=damper)
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct2, related_flow_control=damper)
# Second call returns None -- damper is already assigned to duct1
```

**CORRECT:**
```python
# One control per flow element assignment
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct1, related_flow_control=damper1)
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct2, related_flow_control=damper2)

# Or unassign first if reassigning
ifcopenshell.api.run("system.unassign_flow_control", model,
    relating_flow_element=duct1, related_flow_control=damper)
ifcopenshell.api.run("system.assign_flow_control", model,
    relating_flow_element=duct2, related_flow_control=damper)
```

**Why:** `assign_flow_control` returns `None` if the control element is already assigned elsewhere. A flow control element has a one-to-one relationship with a flow element. To reassign, unassign first.

---

## AP-12: Mixing MEP Domains in One System

**WRONG:**
```python
# Creating one system for mixed domains
system = ifcopenshell.api.run("system.add_system", model)
ifcopenshell.api.run("system.edit_system", model,
    system=system, attributes={"Name": "Building MEP", "PredefinedType": "NOTDEFINED"})

# Assigning ducts AND pipes AND cables to the same system
ifcopenshell.api.run("system.assign_system", model,
    products=[duct, pipe, cable], system=system)
```

**CORRECT:**
```python
# Separate systems per domain
hvac = ifcopenshell.api.run("system.add_system", model)
ifcopenshell.api.run("system.edit_system", model,
    system=hvac, attributes={"Name": "HVAC Supply", "PredefinedType": "VENTILATION"})

plumbing = ifcopenshell.api.run("system.add_system", model)
ifcopenshell.api.run("system.edit_system", model,
    system=plumbing, attributes={"Name": "Hot Water", "PredefinedType": "DOMESTICHOTWATER"})

electrical = ifcopenshell.api.run("system.add_system", model)
ifcopenshell.api.run("system.edit_system", model,
    system=electrical, attributes={"Name": "Power", "PredefinedType": "ELECTRICAL"})

ifcopenshell.api.run("system.assign_system", model, products=[duct], system=hvac)
ifcopenshell.api.run("system.assign_system", model, products=[pipe], system=plumbing)
ifcopenshell.api.run("system.assign_system", model, products=[cable], system=electrical)
```

**Why:** Each `IfcDistributionSystem` should represent a single logical system with a specific `PredefinedType`. Mixing domains in one system defeats the purpose of system classification and makes MEP analysis, clash detection, and coordination impossible.
