# Separation of Concerns for Connection Points in USD

Copyright &copy; 2026, NVIDIA Corporation, version 0.1 (DRAFT)

Beau Perschall

> **See also:** The full-length version of this document
> ([AIF_Connection_Points_Proposal.md](AIF_Connection_Points_Proposal.md))
> contains additional narrative detail and extended design rationale.

## Executive summary

As OpenUSD adoption grows across industrial domains, a recurring challenge has
emerged: how to represent **physical connection points** on equipment assets.
Connection points are the interfaces where components physically interact --
pipe fittings, electrical sockets, airflow vents, network ports, tooling
receptacles, and mechanical mounts.

Today, connection points in USD are geometry prims with naming conventions that
conflate three fundamentally different concerns:

| Concern | What it captures | Current location |
|---|---|---|
| **Spatial position & orientation** | Where the connection is, which direction it faces | Geometry prims (planes, disks) |
| **Semantic identity** | What kind of connection it is | Encoded in prim names |
| **Domain-specific properties** | Engineering parameters governing compatibility | Scattered in `customData`, SimReady metadata, or absent |

**This proposal argues that:**

1. Connection points should be **Xform prims, not geometry prims** -- the
   transform captures position and full orientation (including keying), while
   typed properties capture everything else.
2. A standardized **`connectionPoint:` property namespace** should replace
   ad-hoc naming conventions as the carrier of semantic identity and
   engineering parameters.
3. The approach should start with **namespaced properties** (not a formal
   applied API schema) to keep the barrier low for CAD export tools and PLM
   integrations.
4. The model should be **domain-agnostic** -- serving AI Factories,
   manufacturing, robotics, and process plants alike.

## Contents

- [Executive summary](#executive-summary)
- [Motivation](#motivation)
- [Problem statement](#problem-statement)
- [Existing mechanisms and their limitations](#existing-mechanisms-and-their-limitations)
- [Key questions and recommended approach](#key-questions-and-recommended-approach)
- [Connection point domains](#connection-point-domains)
- [Industry use cases](#industry-use-cases)
- [Consumption workflows](#consumption-workflows)
- [Data sourcing: CAD and PLM](#data-sourcing-cad-and-plm)
- [Design principles and open questions](#design-principles-and-open-questions)
- [Relationship to other proposals and frameworks](#relationship-to-other-proposals-and-frameworks)
- [Next steps](#next-steps)
- [Appendix A: Current AIF naming conventions](#appendix-a-current-aif-naming-conventions)

## Motivation

In the current AI Factory Digital Twin (AIFDT) workflow, connection points are
simple geometry prims (planes and disks) authored manually in a separate USD
layer, set to `guide` purpose, and named using a convention that encodes
vendor, domain, flow direction, and a descriptive suffix (e.g.,
`vertiv_fws_supply_piping_connection_main`).

This works for immediate datacenter needs but does not generalize. Connection
points appear wherever components physically interact:

- **Datacenter cooling** -- CDUs, CRAHs, rear-door heat exchangers connecting
  to facility water piping with specific flow rates, temperatures, pressures.
- **Electrical distribution** -- PDUs, RPPs, UPS connecting to facility power
  with specific voltages, amperages, phases, connector types.
- **Robotic workcells** -- Wrist flanges defining which grippers attach (bolt
  pattern, payload, pass-through channels); tool changers with mechanical,
  pneumatic, and electrical interfaces.
- **Manufacturing equipment** -- CNC spindles accepting specific tooling
  (CAT40, BT30, HSK-A63 tapers); machines connecting to coolant, compressed
  air, and power.
- **Process plants** -- Vessels, pumps, and heat exchangers connecting to
  piping with specific flange ratings, materials, and process conditions.

The common thread: every connection point carries position/orientation,
semantic identity, and domain-specific engineering properties -- and conflating
these creates problems for simulation, validation, and interoperability.
The connection point concept is not limited to connecting two separate pieces
of equipment -- it encompasses any physical interface where interchangeability
and compatibility matter.

## Problem statement

### Three concerns, currently entangled

| | **Spatial position & orientation** | **Semantic identity** | **Domain-specific properties** |
|---|---|---|---|
| **Purpose** | Where the connection physically exists and which direction it faces | Classify what kind of connection it is | Engineering parameters governing compatibility, including spatial extents (diameter, area, mating depth, clearance) |
| **Representation** | Xform transform: position + full orientation, with a standard axis convention | Connection type, flow direction, system membership, tooling standard | Flow rate, temperature, pressure, voltage, port diameter, flange rating, taper type, payload capacity, etc. |
| **Consumed by** | Spatial queries, routing, clearance checks | Discovery, BOM generation, compatibility matching | Simulation runtimes, engineering analysis, compliance |
| **Variability** | Changes per equipment instance and placement | Relatively stable across instances of the same equipment type | May vary by configuration or site-specific requirements |
| **Current location** | Geometry prims in a ConnectionPoints layer | Encoded in prim naming conventions | `customData`, SimReady metadata, or absent |

### Five connection domains

1. **Thermal cooling** -- Flow rate, temperature, pressure, pipe diameter,
   fluid type
2. **Electrical** -- Voltage, amperage, phase, frequency, connector standard
3. **Airflow / ventilation** -- Vent area, airflow volume, temperature delta,
   static pressure (critical for CFD simulation)
4. **Network / data** -- Port type (OSFP, QSFP112, RJ45), line rates,
   transceiver compatibility, physical-to-logical port mapping
5. **Mechanical** -- Bolt pattern, load rating, interface standard (ISO 9409-1,
   CAT40, ATI QC), compatible accessories

These domains are not mutually exclusive -- a CNC spindle is both a mechanical
tooling connection and a coolant-through thermal connection; a robot tool
changer carries mechanical, pneumatic, and electrical interfaces simultaneously.
All domains share cross-cutting mechanical properties (insertion force, keying,
temperature rating, shock and vibration limits).

### Why this matters now

1. **Manual authoring does not scale.** The current workflow requires manually
   creating geometry prims, naming them by convention, and positioning them.
   This cannot be automated from CAD source data without a structured schema.
   Design revisions require manual re-authoring rather than parametric
   propagation.
2. **Naming conventions are fragile.** Encoding semantics into prim names
   requires string parsing. Conventions evolve, vary across vendors, and a
   typo silently breaks downstream tools.
3. **Simulation runtimes lack structured input.** Engineering parameters (flow
   rates, temperatures, voltages) are not captured in geometry or naming.
   Simulation tools must rely on external lookup tables or manual
   configuration.
4. **Cross-domain generalization is blocked.** Each industry vertical develops
   its own ad-hoc conventions. A schema built for datacenter cooling cannot
   be reused for robotic workcell pneumatics or process plant piping -- even
   though the underlying concept is identical.
5. **Connection compatibility cannot be validated.** Matching pipe diameters,
   compatible voltages, aligned bolt patterns, or verifying that a gripper
   fits a robot's wrist flange requires structured metadata on both sides.
   Today, this validation is entirely manual.

## Existing mechanisms and their limitations

### Current AIFDT workflow

The current workflow, documented in the
[Connection Points Workflow Doc](https://docs.google.com/document/d/1EiWmKriSCvmX8GvwQk_w8ZAmmyKnOjWyXicnIr3y6sw),
defines connection points as:

- **Geometry representation** -- Simple mesh prims (planes for rectangular,
  disks for circular openings) at the physical interface location.
- **Naming convention** -- Each prim named as
  `<vendor>_<system>_<direction>_<suffix>`.
- **Layer separation** -- Authored in a dedicated
  `<AssetName>_ConnectionPoints.usd` layer, composed via sublayering.
- **Purpose** -- Set to `guide` (excluded from visual rendering).
- **Scope** -- All prims under a `ConnectionPoints` scope prim.

This has been implemented for NVIDIA GB300 racks and Vertiv cooling equipment
in the
[aif-samples repository](https://gitlab-master.nvidia.com/omniverse/samples/kit/scripts/aif-samples).

### Relevant USD mechanisms

- **Sublayering** allows connection point data to be managed separately from
  geometry (the approach currently used).
- **Applied API schemas** (e.g., `PhysicsRigidBodyAPI`, `SemanticsAPI`) attach
  typed, discoverable metadata to any prim.
- **Relationships** can express connections between equipment prims.
- **Collections** can group connection points by type or system.

### SimReady metadata

The SimReady specification defines thermal and electrical metadata for AI
Factory equipment (e.g., the
[Thermal Cooling Metadata Matrix](https://docs.google.com/document/d/1K7l-PqzhmDC-lqBEuxRwrz81bAVfyclzmKanjig55oE)),
but these exist as flat properties on the equipment prim, not on individual
connection points.

### Key limitations

- **Naming encodes semantics** -- Requires string parsing, error-prone, no
  schema validation.
- **Geometry encodes spatial extents** -- Tools must measure a disk's radius
  to determine a pipe diameter, conflating visual representation with
  engineering data.
- **Properties are not connection-scoped** -- A CDU with different flow rates
  on its FWS and TCS loops cannot express this per-connection.
- **No compatibility model** -- No mechanism to express that a supply
  connection mates with a return of matching diameter and pressure rating.
- **Manual authoring only** -- CAD tools cannot auto-generate connection
  points without a structured vocabulary. Fabricating mesh geometry is a
  higher barrier than emitting an Xform with typed properties.
- **AIF-specific** -- Naming conventions are tailored to datacenter equipment
  and do not extend to robotics, process plants, or general manufacturing.

## Key questions and recommended approach

### Does a connection point need geometry?

**No.** An Xform's transform already captures position and full orientation
(connection direction via local Z-axis, keying via local Y-axis). Spatial
extents (diameter, area, mating depth) should be **typed properties**, not
inferred from mesh dimensions -- because the engineering dimension (a port
diameter) and the visual representation (a disk mesh) are different concerns.

| Spatial extent | Property | Consumed by |
|---|---|---|
| Interface diameter / area | `connectionPoint:portDiameter`, `connectionPoint:ventArea` | Compatibility matching, clearance checks |
| Mating depth | `connectionPoint:matingDepth` | Insertion simulation, routing clearance |
| Service clearance | `connectionPoint:serviceClearance` | Facility layout, maintenance access |
| Bolt pattern extent | `connectionPoint:boltPatternPCD` | Mounting compatibility, structural analysis |

**Standard axis convention** (analogous to ISO 9409-1 tool flange coordinate
systems):

- **Local Z-axis** = connection direction (insertion/flow/mating direction,
  pointing outward from equipment surface)
- **Local Y-axis** = keying "up" direction (rotational orientation for
  non-symmetric connectors)
- **Local X-axis** = derived (right-hand rule)

This convention must be explicit in the vocabulary specification so that all
exporters and consumers agree on what the transform means.

### How should properties be represented?

Three approaches exist on a spectrum from lightweight to formal:

| Approach | Mechanism | Strengths | Weaknesses |
|---|---|---|---|
| **1. `customData` dictionaries** | Standardized `customData` keys | Minimal barrier, any tool can write | No typing, no validation, no composition |
| **2. Namespaced properties** | `connectionPoint:` properties on Xform | Typed values, full composition semantics, no schema plugin | Convention-based discoverability |
| **3. Applied API schema** | Formal `ConnectionPointAPI` | Maximum discoverability, codegen, validation | Requires schema registration, higher adoption barrier |

**Recommended path: start with Approach 2, formalize later.**

- CAD export tools emit an **Xform** at each identified connection feature
  with properties it knows from the design (type, port diameter, interface
  standard). No geometry needs to be fabricated.
- PLM integration adds more properties in a composition layer (design flow
  rate, allowed fluids, simulation models). These override or augment
  CAD-authored properties through normal USD composition.
- A **published vocabulary specification** (a document, not a schema plugin)
  defines the standardized property names, types, allowed values, and which
  properties are expected per domain.
- The Asset Validation Framework validates completeness and compatibility
  using the vocabulary specification as its rule set.
- If adoption warrants it, the namespaced properties can be **promoted to an
  applied API schema** with no change to property names -- existing assets
  remain valid.

Example:

```
def Xform "fws_supply_main" {
    float connectionPoint:portDiameter = 0.1016
    float connectionPoint:matingDepth = 0.05
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
    token connectionPoint:system = "FWS"
    float connectionPoint:designFlowRate = 6.3
}
```

### Flat namespace vs. domain-specific prefixes?

**Question:** Should it be `connectionPoint:flowRate` (flat) or
`connectionPoint:thermal:flowRate` (domain-prefixed)?

Flat is simpler for CAD tools to emit but may become unwieldy as domain
properties accumulate. Domain-specific prefixes provide clearer organization
and make it easier for tools to process only the properties they care about.

### Where should compatibility logic live?

**Question:** Should the schema include explicit compatibility constraints
(e.g., allowed mating types, required matching properties), or should
compatibility logic live entirely in external validation tools?

Embedding constraints makes validation portable and self-describing.
Delegating to external tools provides flexibility but fragments logic across
implementations.

## Connection point domains

The following sections describe the five primary domains and their
representative properties. These illustrate the breadth of structured metadata
that connection point Xforms should carry as namespaced properties.

### Thermal cooling connections

Thermal cooling connections represent interfaces for liquid or refrigerant
piping systems. In datacenter contexts, these include Facility Water System
(FWS) and Technology Cooling System (TCS) piping.

| Property | Description | Example values |
|---|---|---|
| Connection type | Physical medium being transported | Liquid, refrigerant, glycol |
| System | Cooling system this connection belongs to | FWS, TCS, chilled water |
| Direction | Supply or return | Supply, return, bidirectional |
| Port size | Nominal pipe or fitting size | 2", 4", DN50, DN100 |
| Flange rating | Pressure class of the connection | ANSI 150, ANSI 300, PN16 |
| Design flow rate | Rated volumetric flow | 100 GPM, 6.3 L/s |
| Design temperature | Rated temperature range | 45-65 &deg;F supply, 55-75 &deg;F return |
| Operating pressure | Rated working pressure | 150 PSI, 10 bar |
| Fluid type | Working fluid | Water, propylene glycol 30%, R-410A |
| Allowed hoses | Compatible hose or pipe assemblies | Vendor-specific part list |
| Disconnect type | Physical disconnect mechanism | Quick-disconnect, threaded, flanged |

### Electrical connections

Electrical connections represent interfaces for power distribution, from
high-voltage facility feeds to low-voltage equipment power cords.

| Property | Description | Example values |
|---|---|---|
| Connection type | Category of electrical interface | Power input, power output, grounding |
| Voltage rating | Maximum rated voltage | 480V, 208V, 400V |
| Current rating | Maximum rated current capacity | 30A, 60A, 200A |
| Phase | Electrical phase configuration | Single-phase, three-phase |
| Frequency | Line frequency | 50 Hz, 60 Hz |
| Connector type | Physical connector standard | IEC 60309, NEMA L6-30, busbar |
| Redundancy group | Whether part of a redundant feed | Primary (A-feed), secondary (B-feed) |
| Allowed power whips | Compatible power cable assemblies | Vendor-specific part list |

### Airflow and ventilation connections

Airflow connection points define open-air boundaries where equipment draws in
or expels air. Unlike piping connections that carry liquid through sealed
interfaces, these define the physical locations and characteristics of vent
**zones** on equipment surfaces. The precise location, area, and orientation
are critical inputs for CFD simulations modeling facility-level airflow
patterns, hot/cold aisle effectiveness, and equipment cooling performance.

A vent connection point represents an **aggregate vent zone** -- for example,
the entire front intake grille of a CRAH, or the full rear exhaust panel of a
compute rack -- not individual perforations or holes. A perforated panel with
hundreds of small openings is a single connection point whose properties
describe the zone's aggregate characteristics: total open area, obstruction
factor, and effective airflow. This matches how CFD tools consume the data:
each vent connection point maps to one boundary condition in the simulation
mesh, not one boundary per perforation.

| Property | Description | Example values |
|---|---|---|
| Direction | Air flows into or out of equipment | Intake, exhaust, bidirectional |
| Vent area | Aggregate open area of the entire vent zone | 0.25 m&sup2;, 2.5 ft&sup2; |
| Vent geometry | Overall shape of the vent zone | Rectangular, circular, perforated panel |
| Airflow volume | Rated volumetric airflow through the zone | 500 CFM, 850 m&sup3;/h |
| Static pressure | Pressure differential across the vent zone | 0.1" WG, 25 Pa |
| Temperature delta | Expected temperature rise or drop | +15 &deg;C (exhaust), ambient (intake) |
| Equipment face | Which surface the vent zone is located on | Front, rear, top, side, bottom |
| Obstruction factor | Effective open area ratio accounting for grilles, filters, and perforations | 60% open area, MERV-8 filter |

With the Xform-based approach, each vent zone is an Xform positioned at the
centroid of the vent area, oriented with the local Z-axis pointing in the
airflow direction. The aggregate vent area, obstruction factor, and other
engineering properties are typed properties directly available to CFD tools
as boundary condition inputs without geometry measurement or external lookup.

### Network and data connections

Network and data connections represent physical ports for connectivity, data
transfer, and control signaling. These are increasingly critical in AI Factory
environments where GPU racks, switches, and storage systems interconnect
through high-density fiber and copper interfaces, and in manufacturing and
robotics where fieldbus and industrial Ethernet connect controllers, sensors,
and actuators.

A key consideration is that physical and logical port mappings are not always
one-to-one. A single physical OSFP port may support multiple logical
configurations (e.g., 2x400G or 1x800G), and port naming must accommodate
both the physical connector identity and the logical network identity.

Beyond the physical connector standard, network connections benefit from
classification by transport medium and functional role. A fiber BOM tool should
not need to sift through copper management ports and industrial fieldbus
connections to find compute fabric OSFP ports. Classification properties enable
targeted discovery without requiring every tool to interpret every connector
standard.

| Property | Description | Example values |
|---|---|---|
| Medium | Physical transport medium | Fiber, copper, DAC, wireless |
| Category | Functional classification | High-speed data, management, control/fieldbus, sensor, console |
| Fabric role | Network fabric or subsystem served | Compute, storage, management, out-of-band |
| Port type | Physical connector standard | OSFP, QSFP112, QSFP-DD, RJ45, LC fiber, M12 D-coded |
| Pin count | Number of pins or conductors | 38-pin (OSFP), 8-pin (RJ45) |
| Supported line rates | Data rates the port can operate at | 400GbE, 800GbE, 1GbE |
| Supported configurations | Logical configurations | 1x800G, 2x400G, 8x100G |
| Hot plug capable | Supports live transceiver insertion | Yes, no |
| Allowed transceivers | Compatible pluggable modules | DR4, FR4, LR4, AOC, DAC |
| Required airflow | For air-cooled transceivers | 5 CFM minimum per port |

In the current GB300 model, ports are defined at the lowest reuse block (e.g.,
compute tray) and names scale with hierarchy: `<instantiation_name>_<port_name>`
(e.g., `CT1_OSFP1`). Human-readable naming (e.g., `C1`-`C4` for compute
network) serves a different purpose than physical port type naming (`OSFP1`,
`ETH1`), and the schema should accommodate both.

Example network connection point with classification properties:

```
def Xform "OSFP1" {
    token connectionPoint:type = "network"
    token connectionPoint:network:medium = "fiber"
    token connectionPoint:network:category = "high_speed_data"
    token connectionPoint:network:fabricRole = "compute"
    token connectionPoint:portType = "OSFP"
    token[] connectionPoint:supportedLineRates = ["400GbE", "800GbE"]
    token[] connectionPoint:supportedConfigurations = ["1x800G", "2x400G"]
    token[] connectionPoint:allowedTransceivers = ["DR4", "FR4", "LR4"]
    bool connectionPoint:hotPlugCapable = true
}
```

### Mechanical connections

Mechanical connections represent physical mounting, structural, and tooling
attachment interfaces -- equipment-to-infrastructure mounts, machine-to-tooling
interfaces (spindle-to-bit, wrist-to-gripper), and modular accessory docking
points. They are relevant across all industrial domains.

| Property | Description | Example values |
|---|---|---|
| Connection type | Category of mechanical interface | Bolt flange, quick-connect, tool receptacle, gripper mount |
| Interface standard | Industry standard governing the interface | ISO 9409-1, CAT40, BT30, HSK-A63, ATI QC |
| Bolt pattern | Fastener arrangement | 4-bolt square 100mm, ISO 9409-1-50-4-M6 |
| Load / payload rating | Maximum supported load or payload | 500 kg static, 10 kg payload at 1m reach |
| Speed / RPM limits | Maximum operating speed for rotary interfaces | 8000 RPM, 24000 RPM |
| Pneumatic / hydraulic | Compressed air or hydraulic fluid connections | 6mm push-fit, 1/4" NPT |
| Electrical pass-through | Electrical signals alongside mechanical coupling | 12-pin, EtherCAT pass-through |
| Compatible accessories | Compatible tooling or accessories | Gripper models, tool bit types, sensor modules |

### Cross-cutting properties (shared across all domains)

Regardless of domain, nearly all physical connection points share a set of
mechanical and physical properties that describe the interface itself. These
appear consistently across power, coolant, network, and mechanical connections
in the
[SimReady Connection Points Examples](https://docs.google.com/document/d/1jZJGJpjW8kLDT6PubSTAD4WYMbWgPQM46_pkXdQli0I)
analysis and should be factored into a base schema rather than duplicated
across domain-specific schemas.

| Property | Description | Applies to |
|---|---|---|
| Port diameter / interface area | Physical dimension of the interface opening; replaces inferring size from geometry | All domains |
| Mating depth | How far a connector, pipe, or tool inserts before fully seated | Coolant, network, mechanical |
| Service clearance | Minimum clear space for access, cable bend radii, or tool change swing paths | All domains |
| Insertion force | Force required to mate the connection | Power, coolant, network, mechanical |
| Mated cycle count | Rated number of insertion/removal cycles | Power, coolant, network, mechanical |
| Retention mechanism | How the connection is secured once mated | Power, coolant, network, mechanical |
| Keying | Physical keying preventing incorrect mating orientation | Power, coolant, network, mechanical |
| Temperature rating | Operating temperature range | All domains |
| Shock and vibration | Maximum shock and vibration limits | Power, coolant, network, mechanical |
| Material | Housing, contact, and seal materials | All domains |

Note that **location and orientation** are not metadata properties -- they are
intrinsic to the Xform's transform. The Xform captures *where* and *which
direction*, while properties capture *how big*, *how deep*, *how much
clearance*, and the physical characteristics of the interface.

These cross-cutting properties argue for a base `connectionPoint:` namespace
with domain-specific prefixes (`connectionPoint:thermal:`,
`connectionPoint:electrical:`, etc.) adding their own properties. If the
community later adopts a formal applied API schema, these map naturally to a
base `ConnectionPointAPI`.

## Industry use cases

The following examples illustrate the breadth of connection point requirements
across industries and demonstrate that a generalized model serves a far wider
set of stakeholders than the current AIF-specific approach.

### AI Factories (AIF)

AI Factory digital twins are the origin of the current connection points
workflow. Datacenter equipment -- GPU racks, CDUs, CRAHs, PDUs, RPPs, UPS
systems -- requires connection points for:

- **Liquid cooling piping** -- FWS and TCS supply/return connections on CDUs,
  rear-door heat exchangers, and liquid-cooled rack manifolds. Each connection
  has specific pipe diameters, flow rates, temperatures, and pressures that
  downstream thermal simulation runtimes (e.g., Cadence Reality DT) consume.
- **Electrical power distribution** -- Power feeds from PDUs and RPPs to
  individual racks, with voltage, amperage, phase, and redundancy
  specifications that drive electrical load balancing and capacity planning.
- **Network port interfaces** -- OSFP, QSFP112, and RJ45 ports on compute
  trays and switches, with precise coordinates, orientations, supported line
  rates, and transceiver compatibility. These drive fiber BOM generation,
  point-to-point cable routing, and cable length optimization. Logical
  network connectivity (defined in SysML or similar tools) binds to these
  physical port connection points.
- **Airflow vent locations** -- Physical positions of intake and exhaust vents
  on equipment cabinets, carrying area, airflow volume, temperature delta, and
  equipment face. CFD simulation tools consume these directly to model
  hot/cold aisle effectiveness, recirculation risks, and facility-level
  thermal performance.

The target state is automated extraction from CAD source data and structured
metadata that simulation runtimes can consume directly.

### Visual Factory Intelligence (VFI)

Manufacturing facilities add **mechanical** connection points (CNC spindle
tooling interfaces constrained by taper standard and RPM, conveyor mounting
points with bolt patterns and load ratings) alongside thermal and electrical
connections. HVAC air grilles and exhaust hoods are airflow connection points
driving environmental CFD simulation.

### Robotics and autonomous systems

Robotics highlights **multi-domain connection points** -- a robot wrist flange
(ISO 9409-1) carries mechanical, pneumatic, and electrical interfaces
simultaneously; tool changers (ATI, Schunk) add their own multi-domain specs.
Humanoid robots attach modular limbs, hands, and battery packs through
standardized connection points. AMRs use charging stations and payload module
interfaces combining electrical, mechanical, and data connections. CAD and PLM
sourcing is critical: manufacturers define interface specs in design files, and
integrators need structured metadata to validate compatibility.

### Industrial equipment and process plants

Process industries have the most mature connection standards (ASME B16.5, DIN
EN 1092, ISO 7005). Piping connections, instrumentation ports, and vessel
nozzles carry complex properties (flange rating, projection length, process
conditions) that demonstrate the need for **standards-based properties**
referencing industry standards rather than arbitrary numeric values.

## Consumption workflows

The industry use cases above describe *what* tools need from connection points.
This section traces *how* representative tools query and consume the data,
because the consumption pattern drives asset structuring decisions.

| Workflow | How it queries | Key structuring requirement |
|---|---|---|
| **CFD thermal simulation** (e.g., Cadence Reality DT) | Traverses assets, filters by `connectionPoint:type` (thermal, airflow), reads engineering properties for boundary conditions | Metadata accessible **before loading payloads** -- argues for interface-layer placement per the reference-payload pattern |
| **BOM generation / cable routing** | Filters by `connectionPoint:type == "network"`, then by `network:medium`, `network:category`, `network:fabricRole` for targeted cable schedules (fiber, copper, DAC) | Discoverable by broad type + filterable by classification without full scene traversal -- argues for scope prim + property namespace as complementary mechanisms |
| **Connection compatibility validation** | Queries `connectionPoint:` properties on both sides of a proposed connection; applies compatibility rules (matching diameters, pressures, bolt patterns) | Structured, typed properties on both endpoints -- impossible with naming conventions alone |
| **Cross-domain validation** ([TC.002](https://gitlab-master.nvidia.com/bperschall/aif_simready_migration_plan), [EL.004](https://gitlab-master.nvidia.com/bperschall/aif_simready_migration_plan)) | Correlates equipment-level metadata (e.g., "supports FWS cooling") with `connectionPoint:type` and `connectionPoint:system` on child prims | Property queries replace name-regex matching -- more robust, extensible, convention-independent |
| **CAD export pipeline** | Exporter emits Xform at each connection feature with typed properties from design dimensions; PLM adds operating parameters in a composition layer | Low barrier -- no schema plugin, no geometry fabrication; partially populated connection points are valid |

## Data sourcing: CAD and PLM

A standardized connection point schema is only as useful as the data that
populates it. A key objective is enabling **automated population** from the
systems where that data originates.

### What comes from CAD

CAD design software is the authoritative source for the physical and geometric
aspects of connection points. With the Xform-based representation, the CAD
exporter's task is straightforward: emit an Xform at each identified connection
feature with typed properties from the design dimensions. No geometry needs to
be fabricated.

- **Location and orientation** -- Position, surface normal, and insertion
  direction of pipe nozzles, electrical terminals, vent openings, network port
  cutouts, and mounting holes. The exporter maps these to the Xform's
  translation and rotation per the standard axis convention.
- **Spatial extents as properties** -- Pipe diameters, connector widths, vent
  areas, and bolt circle diameters are written as typed properties
  (`connectionPoint:portDiameter`, `connectionPoint:ventArea`) rather than
  encoded in geometry dimensions.
- **Interface standards** -- CAT40 spindle tapers, ISO 9409-1 robot flanges,
  OSFP port cutouts -- the standard governs the geometry and implicitly
  defines compatibility constraints.
- **Mechanical properties** -- Bolt patterns, mounting orientations, and
  structural load paths inherent in the design geometry.

This model supports **parametric updates**: when a designer revises a model,
the connection point Xforms and properties regenerate automatically through
the export pipeline, maintaining consistency between design and digital twin.

### What comes from PLM

PLM systems carry information not represented in CAD geometry but essential for
connection point completeness:

- **Operating parameters** -- Flow rates, pressures, temperatures, voltage and
  current ratings from engineering documentation.
- **Material specifications** -- Housing materials, contact plating, seal
  compounds, and fluid compatibility data.
- **Allowed mating components** -- Compatible hoses, power whips,
  transceivers, tooling bits, and grippers maintained as approved vendor lists
  (AVLs) in PLM.
- **Simulation model references** -- SPICE models, signal integrity models,
  thermal resistance models managed as PLM artifacts.
- **Certification and compliance** -- Pressure ratings (ASME, PED), electrical
  certifications (UL, CE), environmental ratings (IP/NEMA).
- **Revision and lifecycle state** -- Part revisions, approval status, and
  obsolescence affecting which specifications are current.

### Parametric updates and revision continuity

When CAD and PLM are sources of truth, the USD representation becomes a
downstream artifact that should regenerate cleanly when upstream data changes:

- **Design revisions** that change a pipe diameter, relocate a vent, or add a
  port should propagate automatically through the CAD-to-USD pipeline.
- **PLM updates** to operating parameters, allowed transceivers, or material
  specs should flow into USD metadata without manual re-authoring.
- **Version continuity** requires that connection point identity survive
  revisions so downstream layouts and routing can track changes.

This sourcing model reinforces the need for a well-defined target -- whether a
vocabulary specification or a formal schema -- not an ad-hoc naming convention
for CAD exporters and PLM integrations to reverse-engineer.

## Design principles and open questions

### Principles

1. **Separation of concerns.** Position/orientation (Xform transform),
   semantic identity (properties), and engineering parameters including
   spatial extents (properties) serve different tools and should be
   independently authorable. Spatial extents are explicit metadata, not
   inferred from geometry.

2. **Domain agnosticism with extensibility.** The base model should not be
   specific to any single industry. A minimal shared abstraction (type,
   direction, system) is extended by domain-specific property sets.

3. **Discoverability without heavy infrastructure.** Connection points should
   be discoverable through a well-known property namespace
   (`connectionPoint:`), scope prim (`ConnectionPoints`), and purpose
   (`guide`) -- no schema plugin required from day one.

4. **Composability.** Properties participate in USD composition and can be
   overridden in stronger layers (e.g., adjusting a design flow rate for a
   specific installation). This argues for namespaced properties over
   `customData`, which does not compose element-wise.

5. **Low barrier for CAD tools.** Emit an Xform at the feature location with
   typed properties -- no schema plugin dependencies, no complex
   registration, no geometry fabrication.

6. **Additive PLM decoration.** PLM-sourced properties layer onto CAD-authored
   connection points through normal USD composition. The connection point
   should be usable in a partially populated state (Xform + type + spatial
   extents from CAD, operating parameters from PLM to follow).

7. **Validation-ready.** Structured properties carry enough information for
   programmatic compatibility checking via the Asset Validation Framework --
   matching pipe diameters, compatible voltages, aligned bolt patterns.

8. **Minimal disruption.** The solution builds on existing USD strengths --
   composition, namespaced properties, purpose-based visibility. A formal
   applied API schema is an optional future upgrade, not a prerequisite.

### Alignment with Asset Structure Principles

The [Principles of Scalable Asset Structure in OpenUSD](https://docs.omniverse.nvidia.com/usd/latest/learn-openusd/independent/asset-structure-principles.html)
establishes patterns that connection points should build on. The consumption
workflows above inform how each principle applies:

| Principle | Connection Points Position |
|---|---|
| **Reference-Payload pattern** | Connection point metadata belongs in the lightweight interface layer, accessible before loading payloads. CFD tools and BOM generators need engineering parameters before any geometry is relevant. The existing `_ConnectionPoints.usd` sublayer already achieves this separation -- the vocabulary formalizes it. |
| **`defaultPrim` and entrypoints** | The `ConnectionPoints` scope prim lives under the asset's `defaultPrim`, not alongside it. Tools discover it via a well-known child scope name under the asset root. |
| **Model hierarchy and `kind`** | Connection points are sub-component detail, below the `component` kind boundary. They do not participate in `kind`-based traversal; they are found by descending into a known scope within a component. |
| **Naming conventions** | The property namespace (`connectionPoint:thermal:flowRate`) uses only valid USD property name characters. Prim names for individual connection points follow the restricted character set from the principles document. The migration from current vendor-prefixed prim names (`vertiv_fws_supply_piping_connection_main`) to cleaner identifiers should reference these conventions. |

**Discoverability** -- flagged by Steve Ghee at PTC as the most important
aspect -- is served by two complementary mechanisms:

1. **Scope prim discovery:** A tool can find all connection points on an asset
   by looking for the well-known `ConnectionPoints` child scope under the
   `defaultPrim`. This works for hierarchy-aware traversal.
2. **Property namespace discovery:** A tool can query for any prim carrying
   `connectionPoint:` properties, regardless of where it sits in the hierarchy.
   This works for scene-wide queries across multiple assets.

Both mechanisms should be available. The scope prim provides structural
predictability; the property namespace provides flexibility for tools that
need to search across entire facility scenes. Connection point metadata should
be accessible before loading payloads, ensuring that lightweight discovery
and planning tools do not require geometry loading.

### Open questions for discussion

1. **Base vocabulary scope.** What belongs in the base `connectionPoint:`
   namespace vs. domain-specific prefixes? Candidates for the base: type,
   direction, system identifier, spatial extents, and cross-cutting mechanical
   properties.

2. **Multi-domain connection points.** A robot tool changer is simultaneously
   mechanical, pneumatic, and electrical. Should a single prim carry
   properties from multiple domain prefixes, or should multi-domain interfaces
   be modeled as co-located single-domain prims?

3. **Connection relationships.** Should the vocabulary support explicit
   relationships between mated connection points, or should mating be
   expressed only through spatial proximity and external tooling?

4. **Layer authoring model.** The current `_ConnectionPoints.usd` layer
   pattern provides clean separation. Should this be formalized?

5. **`guide` purpose.** Connection point Xforms carry no geometry, but
   `guide` purpose still signals metadata scaffolding. Should the vocabulary
   mandate it?

6. **Units and standards.** Should the vocabulary enforce SI units, support
   unit annotations, or defer to USD's existing `UsdGeomLinearUnits`?

7. **Backward compatibility.** Existing GB300 and Vertiv assets use naming
   conventions. The solution should define a migration path.

8. **CAD/PLM integration boundaries.** Which properties should CAD exporters
   populate vs. PLM? How should the vocabulary accommodate partial population?

9. **Port naming and hierarchy.** How should physical port identity (e.g.,
   `OSFP1`) relate to logical network identity (e.g., `C1` for compute
   network)? Classification properties (`network:medium`, `network:category`,
   `network:fabricRole`) partially bridge this gap by capturing functional
   role, but the binding between a physical port and its logical network
   identity may still belong in a separate system model layer.

## Relationship to other proposals and frameworks

### Alignment with SimReady Foundation

The [SimReady Foundation](https://github.com/NVIDIA/simready-foundation) defines
a layered hierarchy for simulation content specifications:

```
Requirement -> Capability -> Feature -> Profile
```

Connection point requirements map into this model as follows:

| Foundation Layer | Connection Points Mapping |
|---|---|
| **Capability** | "AIF Connection Points" (`CP` code prefix) -- a new capability under `capabilities/aif/connection_points/` grouping all connection-point-specific requirements |
| **Features** | Domain-based features consuming CP requirements: `FET201_AIF_NEUTRAL` (Connection Points), with downstream features for Thermal Cooling (`FET202`) and Electrical (`FET203`) that include cross-domain connection point checks |
| **Profiles** | Equipment-class profiles (AIF-CDU-Neutral, AIF-CRAH-Neutral, AIF-UPS-Neutral, AIF-Rack-Neutral) select the domain features relevant to each equipment type |

The current AIF naming conventions represent **tier 1** (vendor-scoped) in the
SimReady positioning narrative. This proposal advances connection points to
**structured tier 1** -- namespaced, typed, validatable -- with a clear upgrade
path to **tier 3** (AOUSD-ratified schema) if adoption warrants formalization
as an applied API schema.

The [AIF-to-SimReady Foundation migration](https://gitlab-master.nvidia.com/bperschall/aif_simready_migration_plan)
formalizes current connection point conventions as v0.1.0 requirements
(CP.001-CP.006) within the Foundation model. This proposal defines the v0.2.0
evolution of those requirements -- replacing geometry prims with Xforms and
naming conventions with typed properties, while preserving the Foundation
scaffolding (FET201, profiles, validation infrastructure) that the migration
establishes.

> **Note:** SimReady FAQ Q8 Path 2 step 5 currently says: "Define facility
> connection interfaces using named geometry prims." This proposal argues for
> Xform prims with typed properties instead. The FAQ will need to be updated
> when this work lands.

### Alignment with "Separation of Concerns for Identifiers in USD"

The [Identifiers proposal](https://github.com/PixarAnimationStudios/OpenUSD-proposals/pull/105)
(Luk, 2026) and this proposal are **instances of the same problem**: USD prim
names being overloaded to carry information that should live in structured
metadata.

| | **Identifiers proposal** | **Connection points proposal** |
|---|---|---|
| **Overloaded in prim names** | External system IDs (IFC GUIDs, PLM part numbers) | Connection semantics (type, direction, vendor, system) |
| **Why it doesn't fit** | Invalid characters, conflates namespace with source identity | Requires string parsing, can't carry engineering properties |
| **What is lost** | Round-trip fidelity, cross-system linking | Structured queryability, compatibility validation, CAD/PLM automation |

The proposals are complementary, not competing. A connection point prim needs
both **source identity** (Luk's concern -- CAD feature ID, PLM part number)
and **domain properties** (this proposal's concern -- type, direction,
engineering parameters). Both namespaces coexist on the same Xform prim,
compose independently through USD's layer stack, and are discoverable through
namespace-based queries:

```
def Xform "fws_supply_main" {
    # Source identity (per identifier separation of concerns proposal)
    string sourceId:cadFeatureId = "FEAT-00472"
    string sourceId:plmPartNumber = "VRT-CDU-FWS-SUP-4IN"
    string sourceId:cadSystem = "SolidWorks"

    # Connection point domain properties (this proposal)
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
    token connectionPoint:system = "FWS"
    float connectionPoint:portDiameter = 0.1016
    float connectionPoint:matingDepth = 0.05
    float connectionPoint:designFlowRate = 6.3
    token connectionPoint:fluidType = "water"
    token connectionPoint:disconnectType = "flanged"
    token connectionPoint:flangeRating = "ANSI_150"
}
```

In this model, a CAD exporter authors `sourceId:` properties (tracing back to
the design feature) and spatial properties (`connectionPoint:portDiameter`
from the design geometry). A PLM integration adds operating parameters
(`connectionPoint:designFlowRate`, `connectionPoint:flangeRating`) in a
stronger composition layer. Both namespaces compose independently and are
queryable by any tool that understands the namespace convention.

A joint vocabulary specification covering both `sourceId:` and
`connectionPoint:` namespaces would demonstrate that this pattern generalizes
to any domain-specific metadata that needs to attach to USD prims.

### Other related efforts

- **SimReady Specification** -- Connection point properties should move from
  the equipment prim to individual connection point prims, integrating with
  (not duplicating) SimReady metadata.
- **Asset Validation Framework** -- Enables validation rules for connection
  point completeness, property ranges, and compatibility without a formal
  schema plugin. The AIF-to-SimReady migration already defines cross-domain
  validators (TC.002, EL.004) that check metadata-to-connection-point
  consistency; the `connectionPoint:` property namespace would make these
  checks more robust by replacing name-pattern matching with property queries.
- **USD Physics Schema** -- Demonstrates the pattern of applied API schemas
  (`PhysicsRigidBodyAPI`, `PhysicsCollisionAPI`) for simulation-relevant
  metadata. If the connection point vocabulary matures to warrant
  formalization, this pattern provides a proven model.

## Next steps

1. **Land the v0.1.0 Foundation baseline.** The
   [AIF-to-SimReady Foundation migration](https://gitlab-master.nvidia.com/bperschall/aif_simready_migration_plan)
   formalizes current connection point conventions (CP.001-CP.006) into the
   Foundation's Requirements/Capabilities/Features/Profiles model. This
   establishes the scaffolding (FET201, profiles, validation infrastructure)
   that the vocabulary specification will evolve.
2. **Coordinate with the Identifiers proposal.** Explore a joint vocabulary
   specification for namespaced property conventions with Aaron Luk and TAC
   stakeholders.
3. **Draft the vocabulary specification.** Define the `connectionPoint:`
   namespace: property names, types, allowed values, and domain-specific
   prefixes. Start with thermal and electrical domains where real AIF assets
   and active partner engagement (Cadence Reality DT, PTC) provide immediate
   validation.
4. **Prototype with partner tools.** Two proof-of-concept tracks:
   - **Cadence Reality DT consumption:** Author existing Vertiv/CRAH connection
     point data as Xform + `connectionPoint:` properties; demonstrate direct
     consumption by CFD simulation without external lookup tables.
   - **PTC CAD export:** Demonstrate PTC's SPT tools emitting Xform-based
     connection points with typed properties from CAD design features (starting
     with an RJ45 network port connection point).
5. **Publish the vocabulary specification.** Move to a public venue (GitHub
   Issue/PR on [NVIDIA/simready-foundation](https://github.com/NVIDIA/simready-foundation)
   or [OpenUSD-proposals](https://github.com/PixarAnimationStudios/OpenUSD-proposals))
   for external review and partner engagement.
6. **Evolve FET201 to v0.2.0.** Update connection point requirements from
   naming-convention-based (v0.1.0) to property-based (v0.2.0). New validators
   query `connectionPoint:` properties. Migration tooling converts existing
   assets. v0.1.0 requirements remain valid during transition.
7. **Evaluate schema promotion criteria.** Define conditions under which the
   vocabulary should be promoted to a formal applied API schema.

---

## Appendix A: Current AIF naming conventions

Representative examples of the naming convention currently in use (see the
[full proposal](AIF_Connection_Points_Proposal.md#appendix-a-current-aif-connection-point-naming-conventions)
for the complete table):

| Connection point type | USD prim naming convention | Example |
|---|---|---|
| FWS supply pipe | `<vendor>_fws_supply_piping_connection_` | `vertiv_fws_supply_piping_connection_main` |
| Electrical power socket | `<vendor>_electrical_nominal_voltage_` | `trane_electrical_nominal_voltage_main` |
| Airflow intake vent | `<vendor>_airvent_intake_` | `trane_airvent_intake_frontplate` |

All connection point prims reside under a `ConnectionPoints` scope prim, are
set to `guide` purpose, use simple mesh geometry, and are authored in a
separate `<AssetName>_ConnectionPoints.usd` layer file composed via
sublayering.
