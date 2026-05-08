# Connection Points Vocabulary: CDU Exemplar

Copyright &copy; 2026, NVIDIA Corporation, version 0.1.0 (DRAFT), May 8, 2026

**Parent specification:** [Connection Points Vocabulary Spec v0.2.0](CONNECTION_POINTS_VOCABULARY_SPEC.md)

This exemplar applies the Connection Points vocabulary to an XDU1350-class
Coolant Distribution Unit. It exercises the thermal domain heavily; the
electrical, network, and airflow connections are simpler but carry the full
Production property set. For vocabulary definitions, design rationale, and
structural rules, see the parent specification.

---

## What connections does a CDU have?

A CDU sits between the facility water loop and the technology cooling loop. It
takes in cold facility water, exchanges heat with the warm technology coolant
returning from racks, and sends chilled coolant back out to the racks. It also
needs power and typically exposes a BMS/monitoring network port.

| Connection | What it does | Domain |
|------------|-------------|--------|
| FWS Supply | Cold facility water comes IN from building | thermal |
| FWS Return | Warm facility water goes OUT to building | thermal |
| TCS Supply | Chilled coolant goes OUT to compute racks | thermal |
| TCS Return | Warm coolant comes back IN from compute racks | thermal |
| Power Input | Electrical power feed for pumps and controls | electrical |
| BMS Network | Monitoring/control network port for building management | network (optional) |
| Air Intake | Room air drawn in to cool internal electronics | airflow |
| Air Exhaust | Heated air expelled back into the room | airflow |

Some CDUs also have condensate drains or redundant power feeds. Those exist but
are out of scope for this first vocabulary pass. Redundant power feeds are
deferred to keep the exemplar narrowly scoped; the `redundancyGroup` property
pattern is reserved for when that use case is addressed (likely with the UPS
equipment class).

---

## CDU connection profile

The CDU is thermal-dominant: four of its eight connections are thermal, with
one electrical power feed, one optional network port for BMS integration, and
two airflow connections for internal electronics cooling. This makes it an
ideal first exemplar for exercising the thermal domain vocabulary.

### Why the CDU exercises thermal heavily

A CDU's core function is heat exchange between two fluid loops with different
characteristics:

- **FWS (Facility Water System)** connections use large flanged pipes
  (0.1016 m / 4-inch NPS), plain water, and higher operating pressures
- **TCS (Technology Cooling System)** connections use smaller pipes
  (0.0762 m / 3-inch NPS), quick-disconnect fittings, and glycol/water mix
  for freeze protection

This contrast demonstrates how the same thermal vocabulary properties carry
different values depending on the system, and why per-connection properties
(rather than per-equipment) are necessary.

---

## The CDU fully dressed: all 8 connections

### FWS Supply (facility water IN)

Cold water from the building enters the CDU here. This is a large flanged pipe
connection, 0.1016 m port diameter on a unit this size.

```usda
def Xform "fws_supply_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "thermal"
    token connectionPoint:direction = "supply"
    token connectionPoint:system = "FWS"
    token connectionPoint:disconnectType = "flanged"
    float connectionPoint:serviceClearance = 0.3

    # Physical connection geometry (thermal domain)
    float connectionPoint:thermal:portDiameter = 0.1016
    float connectionPoint:thermal:matingDepth = 0.05

    # Operating parameters (thermal domain)
    float connectionPoint:thermal:designFlowRate = 6.3
    float connectionPoint:thermal:maxFlowRate = 8.0
    float connectionPoint:thermal:designTemperature = 7.2
    float connectionPoint:thermal:maxTemperature = 45.0
    float connectionPoint:thermal:operatingPressure = 1034000
    float connectionPoint:thermal:maxPressure = 1500000
    token connectionPoint:thermal:fluidType = "water"
    token connectionPoint:thermal:flangeRating = "ANSI_150"
    token connectionPoint:thermal:flangeSize = "NPS4"
}
```

### FWS Return (facility water OUT)

Warm water exits the CDU back to the building cooling system. Same physical
connection as supply, but the fluid is warmer and pressure is lower (return
side of the loop).

```usda
def Xform "fws_return_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "thermal"
    token connectionPoint:direction = "return"
    token connectionPoint:system = "FWS"
    token connectionPoint:disconnectType = "flanged"
    float connectionPoint:serviceClearance = 0.3

    # Physical connection geometry (thermal domain)
    float connectionPoint:thermal:portDiameter = 0.1016
    float connectionPoint:thermal:matingDepth = 0.05

    # Operating parameters (thermal domain)
    float connectionPoint:thermal:designFlowRate = 6.3
    float connectionPoint:thermal:maxFlowRate = 8.0
    float connectionPoint:thermal:designTemperature = 17.2
    float connectionPoint:thermal:maxTemperature = 55.0
    float connectionPoint:thermal:operatingPressure = 896000
    float connectionPoint:thermal:maxPressure = 1500000
    token connectionPoint:thermal:fluidType = "water"
    token connectionPoint:thermal:flangeRating = "ANSI_150"
    token connectionPoint:thermal:flangeSize = "NPS4"
}
```

### TCS Supply (chilled coolant OUT to racks)

The CDU sends chilled technology coolant out to the compute racks. This side
uses a smaller pipe (0.0762 m port diameter), quick-disconnect fittings, and a
glycol/water mix for freeze protection.

```usda
def Xform "tcs_supply_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "thermal"
    token connectionPoint:direction = "supply"
    token connectionPoint:system = "TCS"
    token connectionPoint:disconnectType = "quick_disconnect"
    float connectionPoint:serviceClearance = 0.25

    # Physical connection geometry (thermal domain)
    float connectionPoint:thermal:portDiameter = 0.0762
    float connectionPoint:thermal:matingDepth = 0.04

    # Operating parameters (thermal domain)
    float connectionPoint:thermal:designFlowRate = 5.8
    float connectionPoint:thermal:maxFlowRate = 7.5
    float connectionPoint:thermal:designTemperature = 12.0
    float connectionPoint:thermal:maxTemperature = 50.0
    float connectionPoint:thermal:operatingPressure = 689000
    float connectionPoint:thermal:maxPressure = 1000000
    token connectionPoint:thermal:fluidType = "glycol_water_30"
    token connectionPoint:thermal:flangeRating = "none"
    token connectionPoint:thermal:flangeSize = "none"
}
```

### TCS Return (warm coolant IN from racks)

Warm coolant returns from the compute racks to the CDU for re-chilling. Same
physical pipe as TCS Supply but the fluid is warmer and pressure is lower
(return side).

```usda
def Xform "tcs_return_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "thermal"
    token connectionPoint:direction = "return"
    token connectionPoint:system = "TCS"
    token connectionPoint:disconnectType = "quick_disconnect"
    float connectionPoint:serviceClearance = 0.25

    # Physical connection geometry (thermal domain)
    float connectionPoint:thermal:portDiameter = 0.0762
    float connectionPoint:thermal:matingDepth = 0.04

    # Operating parameters (thermal domain)
    float connectionPoint:thermal:designFlowRate = 5.8
    float connectionPoint:thermal:maxFlowRate = 7.5
    float connectionPoint:thermal:designTemperature = 32.0
    float connectionPoint:thermal:maxTemperature = 60.0
    float connectionPoint:thermal:operatingPressure = 552000
    float connectionPoint:thermal:maxPressure = 1000000
    token connectionPoint:thermal:fluidType = "glycol_water_30"
    token connectionPoint:thermal:flangeRating = "none"
    token connectionPoint:thermal:flangeSize = "none"
}
```

### Power Input (electrical feed)

Main electrical power for the CDU's pumps and control system. The values below
use a North American installation as the exemplar. A European deployment of the
same CDU model would use `nominalVoltage = 400.0` and `frequency = 50.0`.

Note: `matingDepth = 0.0` because this is a hardwired connection with no
plug-style mating engagement. Per the property completeness principle, the
property is present with an explicit zero rather than omitted.

```usda
def Xform "power_input_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "electrical"
    token connectionPoint:direction = "input"
    token connectionPoint:system = "power"
    token connectionPoint:disconnectType = "hardwired"
    float connectionPoint:serviceClearance = 0.4

    # Physical connection geometry (electrical domain)
    float connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float connectionPoint:electrical:nominalVoltage = 480.0
    float connectionPoint:electrical:maxCurrent = 60.0
    int connectionPoint:electrical:phases = 3
    float connectionPoint:electrical:frequency = 60.0
    token connectionPoint:electrical:connectorType = "hardwired"
    float connectionPoint:electrical:ratedPower = 28800.0
    float connectionPoint:electrical:breakerRating = 80.0
    float connectionPoint:electrical:powerFactor = 0.95
}
```

### BMS Network (monitoring/control port, optional)

Network port for building management system integration. The
[Compute Rack exemplar](COMPUTE_RACK_CONNECTION_POINTS_VOCABULARY.md)
exercises the network domain vocabulary more fully with OSFP high-speed ports
and breakout configurations. This BMS port is simpler but carries the full
Production property set; properties that do not apply to a fixed-function
100Mbps BMS port use explicit empty arrays or `"none"` values.

```usda
def Xform "bms_network_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "network"
    token connectionPoint:direction = "bidirectional"
    token connectionPoint:system = "BMS"
    token connectionPoint:disconnectType = "RJ45"
    float connectionPoint:serviceClearance = 0.15

    # Physical connection geometry (network domain)
    float connectionPoint:network:portWidth = 0.01118
    float connectionPoint:network:portHeight = 0.0064
    float connectionPoint:network:matingDepth = 0.011

    # Operating parameters (network domain)
    token connectionPoint:network:portType = "RJ45"
    token connectionPoint:network:protocol = "BACnet_IP"
    token connectionPoint:network:dataRate = "100Mbps"
    token connectionPoint:network:medium = "copper"
    token connectionPoint:network:fabricRole = "bms"
    token[] connectionPoint:network:supportedLineRates = ["100Mbps"]
    token[] connectionPoint:network:supportedConfigurations = []
    token[] connectionPoint:network:allowedTransceivers = []
    bool connectionPoint:network:hotPlugCapable = true
}
```

### Air Intake (room air IN for electronics cooling)

The CDU draws room air across its internal electronics (control boards, VFDs,
power supplies) for cooling. The intake interface is typically a louvered or
perforated panel on the front or side of the unit. This connection point
represents a single contiguous spatial region where air enters -- not
individual louvers or perforations. A CFD tool needs the overall interface
dimensions and open area fraction to set up the boundary condition.

**Note:** The exemplar shows one intake and one exhaust as representative
examples. A real CDU may have multiple distinct airflow interfaces (e.g.,
separate intake panels on the front and sides, multiple exhaust zones on the
rear). Each physically distinct interface should be its own connection point
Xform with its own airflow domain properties. The vocabulary places no limit
on the number of connection points of any domain per asset.

```usda
def Xform "air_intake_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "airflow"
    token connectionPoint:direction = "input"
    token connectionPoint:system = "equipment_cooling"
    token connectionPoint:disconnectType = "open_vent"
    float connectionPoint:serviceClearance = 0.3

    # Physical connection geometry (airflow domain)
    float connectionPoint:airflow:interfaceWidth = 0.45
    float connectionPoint:airflow:interfaceHeight = 0.30
    float connectionPoint:airflow:freeAreaRatio = 0.5

    # Operating parameters (airflow domain)
    float connectionPoint:airflow:designAirflowRate = 0.35
    float connectionPoint:airflow:maxAirflowRate = 0.50
    float connectionPoint:airflow:designTemperature = 24.0
    float connectionPoint:airflow:maxTemperature = 35.0
    float connectionPoint:airflow:staticPressure = 25.0
    token connectionPoint:airflow:filterType = "MERV_8"
}
```

### Air Exhaust (heated air OUT)

Heated air exits the CDU after passing over internal electronics. The exhaust
temperature is higher than intake, reflecting the heat absorbed from the
electronics. This is typically a smaller fraction of the CDU's total heat
rejection (most heat is transferred to the facility water loop), but it still
contributes to the room's thermal load. The exhaust interface may have a
different open area fraction than the intake (e.g., a rear grille vs a front
louvered panel).

```usda
def Xform "air_exhaust_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token connectionPoint:domain = "airflow"
    token connectionPoint:direction = "output"
    token connectionPoint:system = "equipment_cooling"
    token connectionPoint:disconnectType = "open_vent"
    float connectionPoint:serviceClearance = 0.15

    # Physical connection geometry (airflow domain)
    float connectionPoint:airflow:interfaceWidth = 0.40
    float connectionPoint:airflow:interfaceHeight = 0.25
    float connectionPoint:airflow:freeAreaRatio = 0.6

    # Operating parameters (airflow domain)
    float connectionPoint:airflow:designAirflowRate = 0.35
    float connectionPoint:airflow:maxAirflowRate = 0.50
    float connectionPoint:airflow:designTemperature = 35.0
    float connectionPoint:airflow:maxTemperature = 45.0
    float connectionPoint:airflow:staticPressure = 20.0
    token connectionPoint:airflow:filterType = "none"
}
```

---

## CDU-specific notes

### FWS vs TCS contrast

The two fluid loops demonstrate why per-connection properties matter. Key
differences between FWS and TCS connections on the same CDU:

| Property | FWS | TCS |
|----------|-----|-----|
| Port diameter | 0.1016 m (4-inch NPS) | 0.0762 m (3-inch NPS) |
| Disconnect type | Flanged | Quick-disconnect |
| Fluid type | `water` | `glycol_water_30` |
| Operating pressure | 1,034,000 Pa (supply) | 689,000 Pa (supply) |
| Design temperature (supply) | 7.2 C | 12.0 C |

Under v0.1.0, a single `aif:spec:nominalFlow` on the equipment prim could not
express these per-connection differences. The vocabulary makes each connection
self-describing.

### Fluid type tokens encode concentration

The `fluidType` property uses a single descriptive token rather than separate
fluid-type and concentration properties (see parent specification for the
full rationale and token table). The FWS connections use `water` (plain water),
while TCS connections use `glycol_water_30` (30% propylene glycol mix). This
simplification was adopted based on stakeholder feedback -- emerging refrigerant
cocktails make concentration a poor standalone
property; the fluid name itself is the meaningful identifier for simulation
tool lookup.

### Regional electrical variation

The power input uses North American values (480V, 60 Hz). The same CDU model
deployed in Europe would have different electrical property values but identical
thermal and network connections. This is why electrical properties are
site-specific, not equipment-inherent -- a point discussed in the parent
specification's electrical domain section.
