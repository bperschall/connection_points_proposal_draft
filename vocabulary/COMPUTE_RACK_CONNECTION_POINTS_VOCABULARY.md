# Connection Points Vocabulary: Compute Rack Exemplar

Copyright &copy; 2026, NVIDIA Corporation, version 0.1.0 (DRAFT), May 8, 2026

**Parent specification:** [Connection Points Vocabulary Spec v0.2.0](CONNECTION_POINTS_VOCABULARY_SPEC.md)

> **Status:** This exemplar is in draft mode as part of the Connection Points
> Vocabulary v0.2.0 draft. See the
> [parent specification](CONNECTION_POINTS_VOCABULARY_SPEC.md) for the current
> acceptance status and feedback instructions.

This exemplar applies the Connection Points vocabulary to a GB300 NVL72
liquid-cooled compute rack. It exercises the network domain heavily, with
thermal connections for the liquid cooling manifold, electrical for power
feeds, and airflow for residual air heat rejection. For vocabulary definitions,
design rationale, and structural rules, see the parent specification.

---

## What connections does a compute rack have?

A GB300 NVL72 rack is a liquid-cooled, high-density compute platform. Unlike a
CDU (which is thermal-dominant), the compute rack's connection profile is
network-dominant: the majority of its external connections are high-speed data
ports. It also has liquid cooling manifold connections and power feeds.

| Connection | What it does | Domain | Count |
|------------|-------------|--------|-------|
| OSFP high-speed ports | Compute fabric interconnect (800GbE) | network | 72 per tray, multiple trays |
| RJ45 management ports | Out-of-band management and BMC access | network | 2-4 per rack |
| TCS Supply | Chilled coolant IN from CDU | thermal | 1 (manifold) |
| TCS Return | Warm coolant OUT to CDU | thermal | 1 (manifold) |
| Power Feed A | Primary electrical power | electrical | 1 |
| Power Feed B | Redundant electrical power | electrical | 1 |
| Cold Aisle Intake | Room air drawn into the rack (cold aisle face) | airflow | 1 (face) |
| Hot Aisle Exhaust | Heated air expelled from the rack (hot aisle face) | airflow | 1 (face) |

The compute rack exemplar is deliberately more complex than the CDU: it has
many more connections, exercises the full depth of the network domain
vocabulary, and introduces the pattern for representing large port arrays
through USD instancing rather than unique prim definitions per port.

---

## Compute rack connection profile

The compute rack is network-dominant: dozens of high-speed OSFP ports make up
the bulk of its external interface. This makes it the ideal exemplar for
exercising the network domain vocabulary, complementing the CDU's thermal focus.

### Why the compute rack exercises network heavily

A GB300 NVL72 rack's primary purpose is compute fabric participation. Its OSFP
ports support multiple line rate configurations (1x800G, 2x400G, 4x200G),
various transceiver types (DR4, FR4, LR4), and hot-plug operation. This
demands the full network vocabulary: `supportedConfigurations`,
`allowedTransceivers`, `hotPlugCapable`, `fabricRole`, and `medium`.

The management ports (RJ45) are simpler but serve a different fabric role
(`mgmt`), demonstrating how the same network vocabulary properties express
fundamentally different connection purposes.

### Port arrays and instancing

A compute rack has far too many identical OSFP ports to define each as a unique
prim with unique properties. USD's native instancing mechanism handles this:
define one OSFP connection point with all vocabulary properties, then instance
it across tray positions. For v0.2.0, the vocabulary uses namespaced attributes
on each connection point and relies on USD composition for instancing. A future
v0.3.0 revision may formalize instance identity through `SemanticsLabelsAPI`
and PLM reference designators, building on Steve Ghee's PTC RJ45 PoC work.

The USDA snippets below define the connection point *template* for each
connection kind. In a production asset, these would be instanced across the
rack's physical topology.

---

## The compute rack fully dressed: representative connections

### OSFP High-Speed Port (compute fabric)

Primary compute fabric connection. Each GB300 NVL72 tray presents 72 OSFP
ports. The vocabulary captures the port's physical geometry, supported
configurations, and transceiver compatibility.

```usda
def Xform "osfp_port_01" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "network"
    token simready:connectionPoint:direction = "bidirectional"
    token simready:connectionPoint:system = "high_speed_data"
    token simready:connectionPoint:disconnectType = "OSFP"
    float simready:connectionPoint:serviceClearance = 0.05

    # Physical connection geometry (network domain)
    float simready:connectionPoint:network:portWidth = 0.02235
    float simready:connectionPoint:network:portHeight = 0.00906
    float simready:connectionPoint:network:matingDepth = 0.0127

    # Operating parameters (network domain)
    token simready:connectionPoint:network:portType = "OSFP"
    token simready:connectionPoint:network:protocol = "Ethernet"
    token simready:connectionPoint:network:dataRate = "800GbE"
    token simready:connectionPoint:network:medium = "fiber"
    token simready:connectionPoint:network:fabricRole = "compute"
    token[] simready:connectionPoint:network:supportedLineRates = ["800GbE", "400GbE", "200GbE"]
    token[] simready:connectionPoint:network:supportedConfigurations = ["1x800G", "2x400G", "4x200G"]
    token[] simready:connectionPoint:network:allowedTransceivers = ["DR4", "FR4", "LR4"]
    bool simready:connectionPoint:network:hotPlugCapable = true
}
```

This snippet demonstrates properties that do not appear on the CDU's minimal
BMS port: `supportedLineRates`, `supportedConfigurations`,
`allowedTransceivers`, `hotPlugCapable`, `fabricRole`, and `medium`. These are
essential for network topology planning tools that need to understand what a
port can do, not just what it is.

### RJ45 Management Port (out-of-band management)

Out-of-band management and BMC access. Physically identical to the CDU's BMS
port (RJ45), but serving a different system and fabric role. This demonstrates
that the same physical connector type can carry different semantic identity.

```usda
def Xform "mgmt_port_01" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "network"
    token simready:connectionPoint:direction = "bidirectional"
    token simready:connectionPoint:system = "mgmt"
    token simready:connectionPoint:disconnectType = "RJ45"
    float simready:connectionPoint:serviceClearance = 0.05

    # Physical connection geometry (network domain)
    float simready:connectionPoint:network:portWidth = 0.01118
    float simready:connectionPoint:network:portHeight = 0.0064
    float simready:connectionPoint:network:matingDepth = 0.011

    # Operating parameters (network domain)
    token simready:connectionPoint:network:portType = "RJ45"
    token simready:connectionPoint:network:protocol = "Ethernet"
    token simready:connectionPoint:network:dataRate = "1GbE"
    token simready:connectionPoint:network:medium = "copper"
    token simready:connectionPoint:network:fabricRole = "mgmt"
    token[] simready:connectionPoint:network:supportedLineRates = ["1GbE"]
    token[] simready:connectionPoint:network:supportedConfigurations = []
    token[] simready:connectionPoint:network:allowedTransceivers = []
    bool simready:connectionPoint:network:hotPlugCapable = true
}
```

### TCS Supply (chilled coolant IN from CDU)

The rack's liquid cooling manifold supply. Coolant arrives from the CDU via the
technology cooling loop. Compared to the CDU's TCS connections, the rack-side
manifold may have a different port diameter and operating pressure depending on
the manifold design.

```usda
def Xform "tcs_supply_manifold" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "thermal"
    token simready:connectionPoint:direction = "supply"
    token simready:connectionPoint:system = "TCS"
    token simready:connectionPoint:disconnectType = "blind_mate"
    float simready:connectionPoint:serviceClearance = 0.2

    # Physical connection geometry (thermal domain)
    float simready:connectionPoint:thermal:portDiameter = 0.0508
    float simready:connectionPoint:thermal:matingDepth = 0.035

    # Operating parameters (thermal domain)
    float simready:connectionPoint:thermal:designFlowRate = 4.5
    float simready:connectionPoint:thermal:maxFlowRate = 6.0
    float simready:connectionPoint:thermal:designTemperature = 12.0
    float simready:connectionPoint:thermal:maxTemperature = 45.0
    float simready:connectionPoint:thermal:operatingPressure = 550000
    float simready:connectionPoint:thermal:maxPressure = 800000
    token simready:connectionPoint:thermal:fluidType = "glycol_water_30"
    token simready:connectionPoint:thermal:flangeRating = "none"
    token simready:connectionPoint:thermal:flangeSize = "none"
}
```

### TCS Return (warm coolant OUT to CDU)

Warm coolant exits the rack back to the CDU. Same physical manifold connection
as supply, warmer fluid, lower pressure.

```usda
def Xform "tcs_return_manifold" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "thermal"
    token simready:connectionPoint:direction = "return"
    token simready:connectionPoint:system = "TCS"
    token simready:connectionPoint:disconnectType = "blind_mate"
    float simready:connectionPoint:serviceClearance = 0.2

    # Physical connection geometry (thermal domain)
    float simready:connectionPoint:thermal:portDiameter = 0.0508
    float simready:connectionPoint:thermal:matingDepth = 0.035

    # Operating parameters (thermal domain)
    float simready:connectionPoint:thermal:designFlowRate = 4.5
    float simready:connectionPoint:thermal:maxFlowRate = 6.0
    float simready:connectionPoint:thermal:designTemperature = 40.0
    float simready:connectionPoint:thermal:maxTemperature = 55.0
    float simready:connectionPoint:thermal:operatingPressure = 400000
    float simready:connectionPoint:thermal:maxPressure = 800000
    token simready:connectionPoint:thermal:fluidType = "glycol_water_30"
    token simready:connectionPoint:thermal:flangeRating = "none"
    token simready:connectionPoint:thermal:flangeSize = "none"
}
```

### Power Feed A (primary electrical)

Primary power feed for the rack. The GB300 NVL72 is a high-power-density
platform. Values below use a North American installation as the exemplar.

```usda
def Xform "power_feed_a" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "input"
    token simready:connectionPoint:system = "power"
    token simready:connectionPoint:disconnectType = "hardwired"
    float simready:connectionPoint:serviceClearance = 0.5

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 480.0
    float simready:connectionPoint:electrical:maxCurrent = 200.0
    int simready:connectionPoint:electrical:phases = 3
    float simready:connectionPoint:electrical:frequency = 60.0
    token simready:connectionPoint:electrical:connectorType = "hardwired"
    float simready:connectionPoint:electrical:ratedPower = 142000.0
    float simready:connectionPoint:electrical:breakerRating = 250.0
    float simready:connectionPoint:electrical:powerFactor = 0.98
}
```

### Power Feed B (redundant electrical)

Redundant power feed. Identical electrical properties to Feed A. The
relationship between Feed A and Feed B (redundancy group, failover behavior)
is not expressed in the vocabulary today -- this is deferred to the
`simready:connectionPoint:redundancyGroup` property pattern reserved in the parent
specification. When that pattern is defined (likely with the UPS equipment
class), both feeds would reference a shared redundancy group identifier.

```usda
def Xform "power_feed_b" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "electrical"
    token simready:connectionPoint:direction = "input"
    token simready:connectionPoint:system = "power"
    token simready:connectionPoint:disconnectType = "hardwired"
    float simready:connectionPoint:serviceClearance = 0.5

    # Physical connection geometry (electrical domain)
    float simready:connectionPoint:electrical:matingDepth = 0.0

    # Operating parameters (electrical domain)
    float simready:connectionPoint:electrical:nominalVoltage = 480.0
    float simready:connectionPoint:electrical:maxCurrent = 200.0
    int simready:connectionPoint:electrical:phases = 3
    float simready:connectionPoint:electrical:frequency = 60.0
    token simready:connectionPoint:electrical:connectorType = "hardwired"
    float simready:connectionPoint:electrical:ratedPower = 142000.0
    float simready:connectionPoint:electrical:breakerRating = 250.0
    float simready:connectionPoint:electrical:powerFactor = 0.98
}
```

### Cold Aisle Intake (room air IN)

Even on a liquid-cooled rack, residual heat from power supplies, network
switches, and management components is rejected to the surrounding air. The
cold aisle face is where room air enters the rack -- this connection point
represents the entire front face as a contiguous airflow interface, not
individual perforations in the door panel. The `freeAreaRatio` captures the
door's perforation percentage, which is the critical input for a CFD tool
computing effective flow area.

```usda
def Xform "cold_aisle_intake" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "airflow"
    token simready:connectionPoint:direction = "input"
    token simready:connectionPoint:system = "equipment_cooling"
    token simready:connectionPoint:disconnectType = "open_vent"
    float simready:connectionPoint:serviceClearance = 1.0

    # Physical connection geometry (airflow domain)
    float simready:connectionPoint:airflow:interfaceWidth = 0.60
    float simready:connectionPoint:airflow:interfaceHeight = 2.0
    float simready:connectionPoint:airflow:freeAreaRatio = 0.70

    # Operating parameters (airflow domain)
    float simready:connectionPoint:airflow:designAirflowRate = 0.25
    float simready:connectionPoint:airflow:maxAirflowRate = 0.40
    float simready:connectionPoint:airflow:designTemperature = 24.0
    float simready:connectionPoint:airflow:maxTemperature = 35.0
    float simready:connectionPoint:airflow:staticPressure = 15.0
    token simready:connectionPoint:airflow:filterType = "none"
}
```

### Hot Aisle Exhaust (heated air OUT)

The hot aisle face is where heated air exits the rack. The exhaust temperature
reflects the residual heat not captured by the liquid cooling system. On a
fully liquid-cooled NVL72, this temperature delta is smaller than on an
air-cooled rack, but the exhaust location, temperature, and effective open
area are still critical inputs for hot aisle containment design and
room-level CFD.

```usda
def Xform "hot_aisle_exhaust" (
    purpose = "guide"
)
{
    # Semantic identity (base namespace)
    token simready:connectionPoint:domain = "airflow"
    token simready:connectionPoint:direction = "output"
    token simready:connectionPoint:system = "equipment_cooling"
    token simready:connectionPoint:disconnectType = "open_vent"
    float simready:connectionPoint:serviceClearance = 0.5

    # Physical connection geometry (airflow domain)
    float simready:connectionPoint:airflow:interfaceWidth = 0.60
    float simready:connectionPoint:airflow:interfaceHeight = 2.0
    float simready:connectionPoint:airflow:freeAreaRatio = 0.70

    # Operating parameters (airflow domain)
    float simready:connectionPoint:airflow:designAirflowRate = 0.25
    float simready:connectionPoint:airflow:maxAirflowRate = 0.40
    float simready:connectionPoint:airflow:designTemperature = 32.0
    float simready:connectionPoint:airflow:maxTemperature = 45.0
    float simready:connectionPoint:airflow:staticPressure = 10.0
    token simready:connectionPoint:airflow:filterType = "none"
}
```

---

## Compute rack-specific notes

### Network port scale and instancing

A single GB300 NVL72 tray has 72 OSFP ports. A rack may contain multiple
trays. Defining each port as a unique prim with unique vocabulary properties
would be impractical and would not reflect the reality that these ports are
physically and functionally identical -- they differ only in position and
instance identity.

USD instancing solves this: define the OSFP connection point once (as shown
above), then instance it across tray positions. For v0.2.0, each instanced
connection point carries the same namespaced vocabulary properties -- the
vocabulary defines *what* a connection is, not *which instance* it is.

Formal instance identity (e.g., distinguishing port p1 on tray t3 from port
p1 on tray t4) is a composition-layer concern targeted for v0.3.0. Steve
Ghee's PTC RJ45 PoC has demonstrated a promising approach using USD's
`SemanticsLabelsAPI` with Windchill reference designators (refdes), where each
instance inherits a unique identifier chain from the composition hierarchy.
That work will inform the v0.3.0 design, but is not part of the v0.2.0
vocabulary.

### CDU vs compute rack: same vocabulary, different profiles

The CDU and compute rack demonstrate that the vocabulary generalizes across
fundamentally different equipment types:

| Dimension | CDU | Compute Rack |
|-----------|-----|--------------|
| Dominant domain | Thermal (4 of 8 connections) | Network (72+ per tray) |
| Network complexity | Single BMS port, 100Mbps | OSFP 800GbE with breakout configs |
| Thermal complexity | Two distinct fluid loops (FWS + TCS) | Single TCS manifold |
| Airflow profile | Internal electronics cooling (small vents) | Cold/hot aisle faces (full rack height) |
| Port count | 8 | 70+ (unique types), 100s (instances) |
| Instancing need | None (each connection unique) | Critical (port arrays) |
| Disconnect types | Flanged, quick-disconnect, hardwired, RJ45, open vent | Blind-mate, hardwired, OSFP, RJ45, open vent |

The vocabulary properties are the same; the equipment profile determines which
domains and which properties are populated.

### Blind-mate manifold connections

The rack's TCS connections use `disconnectType = "blind_mate"` rather than the
CDU's flanged or quick-disconnect fittings. Blind-mate connections allow the
rack to be rolled into position and mate with the facility plumbing without
manual coupling -- a common pattern for high-density liquid-cooled racks. The
vocabulary handles this without any special cases: `disconnectType` is a
semantic identity property in the base namespace, and the thermal domain
properties describe the operating parameters regardless of how the physical
mating works.

### Redundant power feeds

This exemplar includes both Feed A and Feed B to show the pattern, even though
the `redundancyGroup` property is deferred. The two feeds have identical
electrical properties today. When redundancy modeling is addressed, a
`simready:connectionPoint:redundancyGroup` property would link them and express the
failover relationship. This is explicitly noted as a future vocabulary
extension in the parent specification.

### Airflow on a liquid-cooled rack

The GB300 NVL72's primary cooling path is the TCS liquid loop, but residual
heat from power supplies, management switches, and other non-liquid-cooled
components is still rejected to the surrounding air. The airflow connections
represent the cold aisle intake and hot aisle exhaust faces of the rack.

On a liquid-cooled rack, the airflow volume and temperature delta are both
significantly smaller than on an air-cooled rack. A CRAH exemplar (when
available) will demonstrate the airflow domain at full scale, where forced-air
cooling is the primary heat rejection mechanism and airflow rates are an order
of magnitude higher.

### Compute fabric topology

The OSFP ports' `fabricRole = "compute"` property distinguishes them from
management ports (`fabricRole = "mgmt"`) at the vocabulary level. A network
topology planning tool can query for all connections where
`simready:connectionPoint:network:fabricRole == "compute"` to build a fabric map
without parsing prim names or relying on naming conventions. This is the same
pattern the CDU uses for system classification (`simready:connectionPoint:system`), but
applied within the network domain for finer-grained role distinction.
