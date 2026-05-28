# Connection Points Vocabulary: UPS Exemplar

Copyright &copy; 2026, NVIDIA Corporation, version 0.1.0 (DRAFT), May 8, 2026

**Parent specification:** [Connection Points Vocabulary Spec v0.2.0](CONNECTION_POINTS_VOCABULARY_SPEC.md)

> **Status:** This exemplar is in draft mode as part of the Connection Points
> Vocabulary v0.2.0 draft. See the
> [parent specification](CONNECTION_POINTS_VOCABULARY_SPEC.md) for the current
> acceptance status and feedback instructions.

This exemplar applies the Connection Points vocabulary to a 250 kW three-phase
Uninterruptible Power Supply. It exercises the electrical domain heavily; the
network and airflow connections are simpler but carry the full Production
property set. For vocabulary definitions, design rationale, and structural
rules, see the parent specification.

---

## What connections does a UPS have?

A UPS sits between the utility power feed and the critical load. It conditions
incoming power, provides battery-backed protection during outages, and
distributes clean power to downstream PDUs or switchgear. It also typically
exposes a monitoring network port and requires substantial airflow for
internal heat dissipation.

| Connection | What it does | Domain |
|------------|-------------|--------|
| Utility Input | Main AC power feed from building switchgear | electrical |
| Bypass Input | Alternate AC feed for maintenance bypass or static transfer | electrical |
| Output Feed 1 | Protected AC power to downstream PDU/switchgear (Feed A) | electrical |
| Output Feed 2 | Protected AC power to downstream PDU/switchgear (Feed B) | electrical |
| Battery Bus | DC connection to external battery cabinet | electrical |
| Monitoring Port | SNMP/BACnet/Modbus management and monitoring | network |
| Air Intake | Room air drawn in for internal cooling | airflow |
| Air Exhaust | Heated air expelled from the unit | airflow |

Larger UPS installations may have additional output feeds, parallel UPS
connections for N+1 redundancy, or generator tie connections. Those exist but
are out of scope for this exemplar. The `redundancyGroup` property pattern
reserved in the parent specification will address parallel UPS and N+1
configurations when that use case is formalized.

---

## UPS connection profile

The UPS is electrical-dominant: five of its eight connections are electrical,
spanning three distinct roles (input, output, battery). This makes it the
ideal exemplar for exercising the electrical domain vocabulary, complementing
the CDU's thermal focus and the Compute Rack's network focus.

### Why the UPS exercises electrical heavily

A UPS's core function is power protection and conditioning. Its electrical
connections differ from each other in ways that the vocabulary must capture:

- **Input connections** have high current draws and site-specific voltage/frequency
- **Output connections** have rated power capacities and tighter voltage regulation
- **Battery bus** operates at DC, not AC -- different voltage, zero frequency,
  fundamentally different electrical characteristics from the AC connections

This contrast demonstrates how the same electrical vocabulary properties carry
different values depending on the connection's role, and why per-connection
properties (rather than per-equipment) are essential. A single
`aif:spec:electricalInputVoltage` on the equipment prim cannot express that
the input is 480V AC while the battery bus is 540V DC.

### Equipment-level vs connection-level properties

Many UPS datasheet parameters are equipment-level characteristics, not
connection-point properties. The vocabulary draws this line clearly:

| Equipment-level (stays on equipment prim) | Connection-level (on the Xform) |
|------------------------------------------|--------------------------------|
| `aif:spec:upsRating` (overall kW rating) | `simready:connectionPoint:electrical:ratedPower` (per-feed capacity) |
| `aif:spec:inputCurrentDistortion` (THDi) | `simready:connectionPoint:electrical:nominalVoltage` (per-connection) |
| `aif:spec:batteryType` (VRLA, Li-ion) | `simready:connectionPoint:electrical:maxCurrent` (per-connection) |
| `aif:spec:transientRecoveryLoad` (transfer behavior) | `simready:connectionPoint:electrical:connectorType` (per-connection) |
| `aif:spec:overloadAtNominalVoltage` (overload tolerance) | `simready:connectionPoint:electrical:breakerRating` (per-connection) |

The vocabulary captures what a simulation tool needs to know about each
physical connection point. Equipment behavior characteristics (THD, transient
response, overload curves, battery chemistry) remain on the equipment prim
where they describe the unit as a whole.

---

## The UPS fully dressed: all 8 connections

### Utility Input (main AC power IN)

Primary AC power from building switchgear. This is the UPS's main power
source during normal operation. The values below use a North American
250 kW installation as the exemplar.

```usda
def Xform "utility_input_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "input"
    token simready:connectionPoint:system = "power"
    token simready:connectionPoint:disconnectType = "hardwired"
    float simready:connectionPoint:serviceClearance = 0.6

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 480.0
    float simready:connectionPoint:electrical:maxCurrent = 350.0
    int simready:connectionPoint:electrical:phases = 3
    float simready:connectionPoint:electrical:frequency = 60.0
    token simready:connectionPoint:electrical:connectorType = "hardwired"
    float simready:connectionPoint:electrical:ratedPower = 250000.0
    float simready:connectionPoint:electrical:breakerRating = 400.0
    float simready:connectionPoint:electrical:powerFactor = 0.99
}
```

### Bypass Input (alternate AC power IN)

Maintenance bypass or static transfer switch input. Allows the UPS to be
serviced while maintaining power to the load through an alternate path. This
connection typically has the same electrical characteristics as the utility
input, but may be fed from a different source (e.g., a separate utility
panel or generator connection point).

```usda
def Xform "bypass_input" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "input"
    token simready:connectionPoint:system = "power"
    token simready:connectionPoint:disconnectType = "hardwired"
    float simready:connectionPoint:serviceClearance = 0.6

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 480.0
    float simready:connectionPoint:electrical:maxCurrent = 350.0
    int simready:connectionPoint:electrical:phases = 3
    float simready:connectionPoint:electrical:frequency = 60.0
    token simready:connectionPoint:electrical:connectorType = "hardwired"
    float simready:connectionPoint:electrical:ratedPower = 250000.0
    float simready:connectionPoint:electrical:breakerRating = 400.0
    float simready:connectionPoint:electrical:powerFactor = 0.99
}
```

### Output Feed 1 (protected AC power OUT)

Protected power output to downstream distribution (PDU or switchgear). The
output voltage and frequency match the input under normal operation. The
`ratedPower` on each output feed represents the capacity of that individual
feed, not the total UPS rating -- a UPS may split its capacity across
multiple output connections.

```usda
def Xform "output_feed_1" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "output"
    token simready:connectionPoint:system = "power"
    token simready:connectionPoint:disconnectType = "hardwired"
    float simready:connectionPoint:serviceClearance = 0.6

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 480.0
    float simready:connectionPoint:electrical:maxCurrent = 175.0
    int simready:connectionPoint:electrical:phases = 3
    float simready:connectionPoint:electrical:frequency = 60.0
    token simready:connectionPoint:electrical:connectorType = "hardwired"
    float simready:connectionPoint:electrical:ratedPower = 125000.0
    float simready:connectionPoint:electrical:breakerRating = 200.0
    float simready:connectionPoint:electrical:powerFactor = 1.0
}
```

### Output Feed 2 (protected AC power OUT)

Second output feed. Identical electrical characteristics to Feed 1. Together,
the two output feeds represent the UPS's full 250 kW output capacity split
across two distribution paths. The relationship between these feeds (whether
they are independent, load-sharing, or redundant) is not expressed in the
vocabulary today -- that is deferred to the `simready:connectionPoint:redundancyGroup`
property pattern reserved in the parent specification.

```usda
def Xform "output_feed_2" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "output"
    token simready:connectionPoint:system = "power"
    token simready:connectionPoint:disconnectType = "hardwired"
    float simready:connectionPoint:serviceClearance = 0.6

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 480.0
    float simready:connectionPoint:electrical:maxCurrent = 175.0
    int simready:connectionPoint:electrical:phases = 3
    float simready:connectionPoint:electrical:frequency = 60.0
    token simready:connectionPoint:electrical:connectorType = "hardwired"
    float simready:connectionPoint:electrical:ratedPower = 125000.0
    float simready:connectionPoint:electrical:breakerRating = 200.0
    float simready:connectionPoint:electrical:powerFactor = 1.0
}
```

### Battery Bus (DC connection to battery cabinet)

DC connection to the external battery system. This connection has
fundamentally different electrical characteristics from the AC connections:
DC voltage, zero frequency, and a `phases` value of 1 (single DC bus). This
demonstrates that the electrical domain vocabulary handles both AC and DC
connections without requiring separate namespaces.

```usda
def Xform "battery_bus" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "bidirectional"
    token simready:connectionPoint:system = "battery"
    token simready:connectionPoint:disconnectType = "bus_bar"
    float simready:connectionPoint:serviceClearance = 0.5

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 540.0
    float simready:connectionPoint:electrical:maxCurrent = 500.0
    int simready:connectionPoint:electrical:phases = 1
    float simready:connectionPoint:electrical:frequency = 0.0
    token simready:connectionPoint:electrical:connectorType = "bus_bar"
    float simready:connectionPoint:electrical:ratedPower = 250000.0
    float simready:connectionPoint:electrical:breakerRating = 600.0
    float simready:connectionPoint:electrical:powerFactor = 0.0
}
```

### Monitoring Port (SNMP/BACnet/Modbus management)

Network port for UPS monitoring and management. Similar to the CDU's BMS port,
but typically supporting multiple protocols through swappable communication
cards. The `protocolsAvailable` and `cardCompatibility` properties from the
UPS datasheet are equipment-level characteristics that stay on the equipment
prim; the connection point captures the physical port and its active protocol.

```usda
def Xform "monitoring_port" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "network"
    token simready:connectionPoint:direction = "bidirectional"
    token simready:connectionPoint:system = "mgmt"
    token simready:connectionPoint:disconnectType = "RJ45"
    float simready:connectionPoint:serviceClearance = 0.1

    # Physical connection geometry (network domain)
    float simready:connectionPoint:network:portWidth = 0.01118
    float simready:connectionPoint:network:portHeight = 0.0064
    float simready:connectionPoint:network:matingDepth = 0.011

    # Operating parameters (network domain)
    token simready:connectionPoint:network:portType = "RJ45"
    token simready:connectionPoint:network:protocol = "SNMP"
    token simready:connectionPoint:network:dataRate = "100Mbps"
    token simready:connectionPoint:network:medium = "copper"
    token simready:connectionPoint:network:fabricRole = "mgmt"
    token[] simready:connectionPoint:network:supportedLineRates = ["100Mbps"]
    token[] simready:connectionPoint:network:supportedConfigurations = []
    token[] simready:connectionPoint:network:allowedTransceivers = []
    bool simready:connectionPoint:network:hotPlugCapable = true
}
```

### Air Intake (room air IN for internal cooling)

UPS units generate substantial heat from rectifier, inverter, and battery
charging circuits. The air intake interface is typically a front or side
panel with a filtered opening. As with all airflow connection points, this
represents the contiguous spatial region where air enters, not individual
perforations.

**Note:** The exemplar shows one intake and one exhaust as representative
examples. A real UPS may have multiple distinct airflow interfaces depending
on the cabinet design and internal thermal architecture. Each physically
distinct interface should be its own connection point Xform.

```usda
def Xform "air_intake_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "airflow"
    token simready:connectionPoint:direction = "input"
    token simready:connectionPoint:system = "equipment_cooling"
    token simready:connectionPoint:disconnectType = "open_vent"
    float simready:connectionPoint:serviceClearance = 0.5

    # Physical connection geometry (airflow domain)
    float simready:connectionPoint:airflow:interfaceWidth = 0.60
    float simready:connectionPoint:airflow:interfaceHeight = 1.20
    float simready:connectionPoint:airflow:freeAreaRatio = 0.55

    # Operating parameters (airflow domain)
    float simready:connectionPoint:airflow:designAirflowRate = 0.80
    float simready:connectionPoint:airflow:maxAirflowRate = 1.20
    float simready:connectionPoint:airflow:designTemperature = 24.0
    float simready:connectionPoint:airflow:maxTemperature = 40.0
    float simready:connectionPoint:airflow:staticPressure = 30.0
    token simready:connectionPoint:airflow:filterType = "MERV_8"
}
```

### Air Exhaust (heated air OUT)

Heated air exits the UPS after passing through the internal power electronics.
The exhaust temperature and volume are significant -- a 250 kW UPS at typical
efficiency (95-97%) dissipates 7.5-12.5 kW of heat to the room. This is a
larger airflow contribution than a CDU's electronics cooling, making the UPS
airflow connections more consequential for room-level CFD modeling.

```usda
def Xform "air_exhaust_main" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "airflow"
    token simready:connectionPoint:direction = "output"
    token simready:connectionPoint:system = "equipment_cooling"
    token simready:connectionPoint:disconnectType = "open_vent"
    float simready:connectionPoint:serviceClearance = 0.3

    # Physical connection geometry (airflow domain)
    float simready:connectionPoint:airflow:interfaceWidth = 0.55
    float simready:connectionPoint:airflow:interfaceHeight = 1.00
    float simready:connectionPoint:airflow:freeAreaRatio = 0.65

    # Operating parameters (airflow domain)
    float simready:connectionPoint:airflow:designAirflowRate = 0.80
    float simready:connectionPoint:airflow:maxAirflowRate = 1.20
    float simready:connectionPoint:airflow:designTemperature = 40.0
    float simready:connectionPoint:airflow:maxTemperature = 55.0
    float simready:connectionPoint:airflow:staticPressure = 25.0
    token simready:connectionPoint:airflow:filterType = "none"
}
```

---

## UPS-specific notes

### Input vs output: same domain, different roles

The UPS demonstrates that `simready:connectionPoint:direction` is essential for
distinguishing connections within the same domain. All five electrical
connections are `simready:connectionPoint:domain = "electrical"`, but they serve three
distinct roles:

| Connection | direction | system | Distinguishing characteristic |
|------------|-----------|--------|-------------------------------|
| Utility Input | `input` | `power` | Main AC feed, high current draw |
| Bypass Input | `input` | `power` | Alternate path for maintenance |
| Output Feed 1 | `output` | `power` | Protected AC to downstream loads |
| Output Feed 2 | `output` | `power` | Protected AC to downstream loads |
| Battery Bus | `bidirectional` | `battery` | DC, charges and discharges |

A simulation tool querying for all `simready:connectionPoint:direction == "output"`
connections on a UPS gets the two output feeds -- the connections whose
`ratedPower` determines what load the UPS can serve. This is a query that
was impossible under v0.1.0 where a single `aif:spec:outputVoltage` on the
equipment prim described the unit as a whole.

### AC vs DC on the same equipment

The battery bus connection (540V DC, `phases = 1`, `frequency = 0.0`,
`powerFactor = 0.0`) demonstrates that the electrical domain vocabulary
handles DC connections alongside AC connections without requiring a separate
domain namespace. The same properties apply; the values reflect the
fundamentally different nature of the connection. A `frequency` of 0.0 is
the literal truth for DC, and `powerFactor` of 0.0 correctly indicates that
the AC power factor concept does not apply (per the property completeness
principle in the parent specification).

### Output feed capacity split

The exemplar shows two output feeds, each rated at 125 kW (`ratedPower =
125000.0 W`), which together equal the UPS's 250 kW total rating. This is
a simplification -- real installations may have asymmetric feed capacities
or more than two output connections. The vocabulary supports any number of
output connections with any capacity split. The total does not need to
equal the equipment's `aif:spec:upsRating`; derating, efficiency losses,
and redundancy margins mean the sum of output feed capacities may be less
than the nameplate rating.

### Bypass input: same parameters, different purpose

The bypass input has identical electrical parameters to the utility input in
this exemplar. In practice, the bypass may be fed from a different source
(generator, separate utility panel) with different voltage or phase
characteristics. The vocabulary captures this naturally -- each connection
carries its own values regardless of what the other connections look like.

### Redundancy modeling: deferred

The relationship between the two output feeds (independent, load-sharing, or
redundant) and the UPS's role in a larger N+1 or 2N redundancy scheme are not
expressed in the v0.2.0 vocabulary. The `simready:connectionPoint:redundancyGroup`
property is reserved in the parent specification for when this use case is
addressed. The UPS equipment class is the natural vehicle for developing that
pattern, as redundancy is the UPS's primary design concern.

### Heat dissipation and airflow significance

Unlike the CDU (where most heat goes into the facility water loop) or the
liquid-cooled compute rack (where most heat goes into the TCS manifold), the
UPS rejects nearly all of its losses to the surrounding air. At 95-97%
efficiency, a 250 kW UPS dissipates 7.5-12.5 kW as heat through its airflow
interfaces. This makes the UPS's airflow connections more consequential for
room-level CFD than either the CDU's or compute rack's airflow connections,
and previews the kind of airflow significance the CRAH exemplar will
demonstrate at a much larger scale.
