# Separation of Concerns for Connection Points in USD

Copyright &copy; 2026, NVIDIA Corporation, version 0.2 (DRAFT), May 8, 2026

Beau Perschall

> **See also:** A concise version of this document
> ([AIF_Connection_Points_Proposal_Concise.md](AIF_Connection_Points_Proposal_Concise.md))
> provides the core argument in a shorter format suitable for executive and
> partner review.

## Contents

- [Introduction](#introduction)
- [Motivation](#motivation)
  - [Connection points today](#connection-points-today)
  - [The expanding need for connection semantics](#the-expanding-need-for-connection-semantics)
- [Problem statement](#problem-statement)
  - [Five distinct domains for connection points](#five-distinct-domains-for-connection-points)
  - [Why this matters now](#why-this-matters-now)
- [Key questions](#key-questions)
  - [Geometry vs. semantics](#geometry-vs-semantics)
  - [Domain-specific schemas vs. a unified connection model](#domain-specific-schemas-vs-a-unified-connection-model)
  - [Connection compatibility and validation](#connection-compatibility-and-validation)
- [Existing mechanisms and practices](#existing-mechanisms-and-practices)
  - [AI Factory connection points workflow](#ai-factory-connection-points-workflow)
  - [USD composition and layering](#usd-composition-and-layering)
  - [SimReady metadata](#simready-metadata)
  - [Limitations of current approaches](#limitations-of-current-approaches)
- [Connection point domains](#connection-point-domains)
  - [Thermal cooling connections](#thermal-cooling-connections)
  - [Electrical connections](#electrical-connections)
  - [Airflow and ventilation connections](#airflow-and-ventilation-connections)
  - [Network and data connections](#network-and-data-connections)
  - [Mechanical connections](#mechanical-connections)
  - [Cross-cutting properties](#cross-cutting-properties)
- [Industry use cases](#industry-use-cases)
  - [AI Factories (AIF)](#ai-factories-aif)
  - [Visual Factory Intelligence (VFI)](#visual-factory-intelligence-vfi)
  - [Robotics and autonomous systems](#robotics-and-autonomous-systems)
  - [Industrial equipment and process plants](#industrial-equipment-and-process-plants)
- [Consumption workflows](#consumption-workflows)
- [Data sourcing: CAD and PLM](#data-sourcing-cad-and-plm)
  - [What comes from CAD](#what-comes-from-cad)
  - [What comes from PLM](#what-comes-from-plm)
  - [Parametric updates and revision continuity](#parametric-updates-and-revision-continuity)
- [Design considerations](#design-considerations)
  - [Principles](#principles)
  - [Alignment with Asset Structure Principles](#alignment-with-asset-structure-principles)
  - [Open questions for discussion](#open-questions-for-discussion)
- [Relationship to other proposals and frameworks](#relationship-to-other-proposals-and-frameworks)
- [Acknowledgments](#acknowledgments)
- [Appendix A: Current AIF connection point naming conventions](#appendix-a-current-aif-connection-point-naming-conventions)
- [Appendix B: Provenance and AI-assisted drafting](#appendix-b-provenance-and-ai-assisted-drafting)
- [Changelog](#changelog)

## Introduction

As OpenUSD adoption grows across industrial domains -- AI Factory digital
twins, manufacturing floors, robotic workcells, and process plants -- a
recurring challenge has emerged around how to represent **physical connection
points** on equipment assets. Connection points are the interfaces where
components physically interact -- not only where equipment connects to facility
infrastructure or to other equipment (pipes, ducts, electrical sockets), but
also where tooling attaches to machines, grippers mount to robotic arms, and
accessories dock to host systems. A CNC spindle that accepts specific tooling
bits, a robotic wrist flange that defines which grippers can attach, and a
coolant pipe fitting on a CDU are all connection points: physical interfaces
with position, orientation, identity, and compatibility constraints.

Today, connection points in USD are represented as geometry prims (planes and
disks) with naming conventions that encode connection type, vendor, and
directionality into the prim name itself. This conflates three fundamentally
different concerns: **spatial position and orientation** (where the connection
is and which direction it faces), **semantic identity** (what kind of
connection it is), and **domain-specific properties** (the engineering
parameters -- including spatial extents like diameter and area -- that govern
compatibility and behavior). These concerns are currently entangled in ad-hoc
naming conventions, geometry dimensions, and manual authoring workflows.

This proposal articulates the problem, surveys existing practices across
industries, and identifies the key questions the community must align on before
converging on a standardized connection point representation in USD. The goal
is to establish a shared conceptual framework that serves AI Factories,
visual factory intelligence, robotics, and any industrial domain where
equipment has physical connection interfaces.

## Motivation

### Connection points today

In the current AI Factory Digital Twin (AIFDT) workflow, connection points are
represented as simple geometry prims -- rectangular 2D planes or circular 2D
disks -- positioned at the physical locations where equipment connects to
facility infrastructure. These prims are:

- **Authored manually** in a separate USD layer file (e.g.,
  `XDU1350_ConnectionPoints.usd`) and composed into the main asset via
  sublayering.
- **Set to `guide` purpose** so they are not rendered visually in production.
- **Named using a convention** that encodes the vendor, connection domain, flow
  direction, and a descriptive suffix into the prim name (e.g.,
  `vertiv_fws_supply_piping_connection_main`).

This approach works for the immediate needs of datacenter simulation but has
significant limitations when extended to broader industrial use cases.

### The expanding need for connection semantics

Connection points are not unique to datacenters, nor are they limited to
infrastructure-to-equipment interfaces. Any physical interface where components
interact -- whether equipment connecting to facility piping, a tool bit
seating into a machine spindle, or a gripper attaching to a robotic arm -- is
a connection point:

- **Datacenter cooling equipment** (CDUs, CRAHs, rear-door heat exchangers)
  connects to facility water supply and return piping, with specific flow
  rates, temperatures, and pressures. Piping connections (liquid, refrigerant)
  are general industrial interfaces found across process plants,
  manufacturing, and datacenters alike, while airflow interfaces
  (intake/exhaust vent zones on equipment surfaces) are more specific to
  datacenter and HVAC environments.
- **Electrical distribution equipment** (PDUs, RPPs, UPS systems, busways)
  connects to facility power with specific voltages, amperages, phases, and
  connector types.
- **Robotic workcells** connect to pneumatic air lines, electrical power,
  Ethernet/fieldbus data networks, and mechanical mounting points with precise
  bolt patterns and load ratings.
- **Process plant equipment** (pumps, heat exchangers, reactors, separators)
  connects to piping systems with specific flange ratings, pipe diameters, and
  fluid compatibility requirements.
- **Manufacturing equipment** (CNC machines, injection molders, assembly
  stations) connects to coolant lines, compressed air, electrical power, and
  material conveyance systems. A CNC spindle is itself a connection point that
  defines which tooling bits are compatible by shank type, taper standard
  (CAT40, BT30, HSK-A63), and maximum RPM rating.
- **Robotic tooling interfaces** -- A robot's wrist flange defines which
  end-of-arm tools (grippers, welding torches, inspection cameras) can
  attach, governed by bolt pattern, payload capacity, pneumatic/electrical
  pass-through, and tool changer standard.
- **Modular industrial accessories** -- Vision systems mount to brackets with
  specific bolt patterns, sensors dock to standardized ports, and conveyor
  modules join through mechanical and electrical quick-connect interfaces.

In all these cases, a connection point carries information that spans three
distinct concerns -- and conflating them creates problems for simulation,
validation, and cross-system interoperability. Crucially, the connection point
concept is not limited to "connecting two separate pieces of equipment." It
encompasses any physical interface where interchangeability and compatibility
matter: what can attach here, what are the constraints, and what properties
govern the interaction.

## Problem statement

### Five distinct domains for connection points

A physical connection point is any interface on an asset where another
component can attach, dock, plug in, or flow through. This includes
infrastructure connections (piping, electrical feeds, ductwork), tooling
receptacles (CNC spindles, robotic wrist flanges), accessory mounts (sensor
brackets, vision system docks), and modular interfaces (conveyor segment
joints, panel-mount connectors). The common thread is that each connection
point carries information across three fundamental concerns:

| | **Spatial position and orientation** | **Semantic identity** | **Domain-specific properties** |
|---|---|---|---|
| **Purpose** | Define where the connection physically exists on the equipment and which direction it faces (including rotational keying) | Classify what kind of connection it is | Specify the engineering parameters that govern compatibility and behavior, including spatial extents (diameter, area, mating depth, clearance) |
| **Representation** | Xform transform: position (translation) + full orientation (rotation), with a standard axis convention for connection direction and keying | Connection type, flow direction, system membership, tooling standard | Flow rate, temperature, pressure, voltage, amperage, port diameter, flange rating, taper type, payload capacity, mating depth, service clearance, etc. |
| **Consumed by** | Spatial queries, routing algorithms, clearance checks, tool change simulation | Discovery, BOM generation, compatibility matching, connection validation, tooling selection | Simulation runtimes, engineering analysis, regulatory compliance, interchangeability verification |
| **Variability** | Changes per equipment instance and placement | Relatively stable across instances of the same equipment type | May vary by configuration, operating mode, or site-specific requirements |
| **Currently defined as** | Geometry prims (planes, disks) in a ConnectionPoints layer | Encoded in prim naming conventions | Scattered across `customData`, SimReady metadata, or absent entirely |

Today, there is no standardized representation in USD for connection points.
The first column (position and orientation) is captured indirectly through
geometry prims that also entangle visual representation with spatial data. The
second column (semantic identity) is encoded into prim names, losing structure
and queryability. The third column (domain-specific properties, including
spatial extents like diameter, area, and engagement depth) is either stored in
ad-hoc metadata or not captured at all.

Within the "domain-specific properties" column, connection points fall into
five primary domains, each with its own set of engineering properties:

1. **Thermal cooling** -- Liquid and refrigerant piping connections carrying
   flow rate, temperature, pressure, pipe diameter, fluid type, and allowed
   hoses and fluids.
2. **Electrical** -- Power distribution connections carrying voltage, amperage,
   phase, frequency, connector standard, and allowed power whips.
3. **Airflow and ventilation** -- Physical air intake and exhaust vent
   locations on equipment, carrying vent area, airflow volume, temperature
   delta, and static pressure. These vent locations are critical inputs for
   computational fluid dynamics (CFD) simulations that model heat dissipation,
   hot/cold aisle behavior, and environmental airflow patterns. Note: airflow
   interfaces are finite surfaces rather than discrete points -- the term
   "connection point" is used broadly to include any physical interface where
   interchangeability, compatibility, or simulation boundary conditions matter.
4. **Network and data** -- Physical ports for network connectivity and data
   transfer (OSFP, QSFP112, RJ45), carrying port type, supported line rates,
   supported configurations, pin count, and allowed transceivers. Physical
   and logical port mappings are not always one-to-one and must be captured.
5. **Mechanical** -- Physical mounting, structural, and tooling attachment
   interfaces carrying bolt pattern, load rating, interface standard, and
   compatible accessories.

These five domains are not mutually exclusive -- a single physical interface
may span multiple domains (e.g., a CNC spindle is both a mechanical tooling
connection and a coolant-through thermal connection; a network port has
mechanical retention, electrical signaling, and may require airflow for cooled
transceivers). Additionally, all domains share cross-cutting mechanical
properties such as insertion force, mated cycle count, retention mechanism,
keying, temperature rating, and shock and vibration limits. The domain-specific
properties in each category are what enable downstream simulation, validation,
and compatibility checking.

### Why this matters now

Without a clear separation of concerns for connection points:

1. **Manual authoring does not scale.** The current workflow requires a human
   to manually create geometry prims, name them according to convention, and
   position them precisely. This process cannot be automated from CAD source
   data without a structured schema that CAD export tools and PLM integrations
   can target. Design revisions require manual re-authoring rather than
   parametric propagation.

2. **Naming conventions are fragile.** Encoding connection semantics into prim
   names (e.g., `vertiv_fws_supply_piping_connection_main`) requires string
   parsing to extract meaning. Naming conventions evolve, vary across vendors,
   and provide no validation -- a typo in a prim name silently breaks
   downstream tools.

3. **Simulation runtimes lack structured input.** Thermal, electrical, and
   airflow simulations need engineering parameters (flow rates, temperatures,
   voltages) that are not captured in geometry or naming. Without structured
   properties, simulation tools must rely on external lookup tables or manual
   configuration, introducing error and friction.

4. **Cross-domain generalization is blocked.** Each industry vertical (AIF,
   VFI, robotics) is developing its own ad-hoc conventions for connection
   points. Without a shared abstraction, a connection point schema built for
   datacenter cooling cannot be reused for robotic workcell pneumatics or
   process plant piping -- even though the underlying concept is identical.

5. **Connection compatibility cannot be validated.** Determining whether two
   pieces of equipment can physically connect (matching pipe diameters,
   compatible voltages, aligned flange bolt patterns) requires structured
   metadata on both sides. The same applies to tooling interfaces: verifying
   that a specific gripper is compatible with a robot's wrist flange, or that
   a tooling bit matches a CNC spindle's taper standard and RPM limits,
   requires structured properties. Today, this validation is entirely manual.

## Key questions

Before proposing a specific representation, the community should align on
several foundational questions. Notably, the right answer may not require a
formal API schema at all -- there is a spectrum of approaches from lightweight
conventions to full schema infrastructure, and the practical constraints of
CAD export and PLM integration should guide the choice.

### Geometry vs. semantics

The current approach uses geometry prims (planes and disks) as the primary
carriers of connection point information, with semantics encoded in the prim
name. This raises a fundamental question: **does a connection point need
geometry at all?**

An Xform prim's transform already encodes the two spatial concerns that
matter for a connection point:

- **Position** -- where the connection is located on the equipment surface
  (translation).
- **Orientation** -- the full three-axis coordinate frame at the interface.
  This captures both the **connection direction** (e.g., the axis along which
  a pipe inserts or fluid flows) and the **rotational keying** (e.g., which
  way is "up" for a keyed connector like an RJ45 jack, an OSFP transceiver,
  or a polarized power connector that can only mate in one rotational
  orientation). A single direction vector is insufficient for keyed
  interfaces; the full orientation is required.

What an Xform's position *lacks* is **spatial extent** -- it is a
dimensionless point. A pipe nozzle has a diameter. A vent has a rectangular
area. A bolt flange has a pattern diameter. A connector has a width and
height. In the current geometry-based approach, these extents are inferred
from the mesh dimensions (the radius of a disk prim, the width of a plane
prim). But this conflates two distinct concerns:

- **Visual representation** -- what shape to draw in the viewport.
- **Interface dimensions** -- the engineering measurements that govern
  compatibility and spatial clearance.

These are different concerns consumed by different tools. A clearance checker
needs a port diameter value, not a mesh to measure. A routing algorithm needs
an engagement depth, not a polygon count. Expressing spatial extents as
**typed metadata properties** rather than inferring them from geometry is
itself a separation of concerns:

| Spatial extent | Property | Consumed by |
|---|---|---|
| Interface diameter or area | `connectionPoint:portDiameter`, `connectionPoint:ventArea` | Compatibility matching, clearance checks |
| Mating / engagement depth | `connectionPoint:matingDepth` | Insertion simulation, cable routing clearance. (Alternative naming "insertion depth" under community review.) |
| Service clearance envelope | `connectionPoint:serviceClearance` | Facility layout, maintenance access planning |
| Bolt pattern extent | `connectionPoint:boltPatternPCD`, `connectionPoint:boltCount` | Mounting compatibility, structural analysis |

This means the canonical connection point prim can be an **Xform** -- not a
mesh. The Xform's transform captures position and full orientation (including
keying), and namespaced properties capture the spatial extents, semantic
identity, and domain-specific engineering parameters. No geometry is required
from CAD export or PLM integration.

The community should adopt a **standard axis convention** for connection point
Xforms (analogous to how ISO 9409-1 defines a tool flange coordinate system
for robotic interfaces):

- **Local Z-axis** = connection direction (insertion axis, flow direction,
  mating direction -- pointing outward from the equipment surface).
- **Local Y-axis** = keying "up" direction (defines rotational orientation
  for connectors that are not rotationally symmetric).
- **Local X-axis** = derived (completes the right-handed frame).

This convention must be explicit in the vocabulary specification so that all
exporters and consumers agree on what the transform means.

**Question:** Should connection point semantics (type, direction, domain) be
expressed as an applied API schema on the Xform prim, as structured metadata
(e.g., a well-known `customData` dictionary), or as simple property namespaces
on the prim?

The answer should be guided by what CAD export tools can realistically produce.
A CAD tool that identifies a pipe nozzle during export can readily emit an
Xform at the nozzle location with properties like `connectionPoint:type` and
`connectionPoint:portDiameter`. Requiring that tool to fabricate geometry
(what shape? what tessellation?) or instantiate a registered applied API schema
is a significantly higher bar. The representation should be something that CAD
tools can emit simply, PLM integrations can augment afterward, and downstream
consumers can discover reliably.

### Domain-specific schemas vs. a unified connection model

Thermal cooling, electrical, airflow, network, and mechanical connections each
have distinct properties. A thermal connection needs flow rate and temperature;
an electrical connection needs voltage and phase; an airflow vent needs area
and airflow volume; a network port needs line rate and transceiver
compatibility; a mechanical connection needs bolt pattern and load rating. Yet
all share cross-cutting properties (insertion force, keying, temperature
rating) as documented in the
[SimReady Connection Points Examples](https://docs.google.com/document/d/1jZJGJpjW8kLDT6PubSTAD4WYMbWgPQM46_pkXdQli0I).

**Question:** Should the vocabulary use a flat `connectionPoint:` namespace
with a type discriminator and all properties at one level, or should it use
domain-specific property prefixes (`connectionPoint:thermal:flowRate`,
`connectionPoint:electrical:voltage`, `connectionPoint:airflow:ventArea`,
`connectionPoint:network:lineRate`, `connectionPoint:mechanical:boltPattern`)
under a shared base?

A flat namespace is simpler for CAD tools to emit but may become unwieldy as
domain properties accumulate. Domain-specific prefixes provide clearer
organization and make it easier for tools to process only the properties they
care about.

### Connection compatibility and validation

A key downstream use case is determining whether two connection points are
compatible -- whether equipment A's supply pipe can connect to equipment B's
return pipe (considering diameter, pressure rating, and fluid type), whether
a specific gripper can mount to a robot's wrist flange (considering bolt
pattern, payload, and pass-through channels), or whether a tooling bit fits
a CNC spindle (considering taper standard and RPM limits).

**Question:** Should the connection point schema include explicit
compatibility constraints (e.g., allowed mating types, required matching
properties), or should compatibility logic live entirely in external validation
tools that consume the structured metadata?

Embedding compatibility in the schema makes validation portable and
self-describing. Delegating to external tools provides more flexibility but
fragments the logic across implementations.

### Representation approaches: a spectrum

Rather than assuming that a formal applied API schema is the necessary
endpoint, it is worth examining the full spectrum of representation approaches.
Each occupies a different position on the tradeoff between simplicity of
authoring (what CAD tools can easily emit) and richness of downstream
consumption (what simulation and validation tools need).

In all approaches below, connection point prims are **Xform** prims, not
geometry prims. The Xform's transform captures position and full orientation
(including keying direction), and all spatial extents, semantic identity, and
engineering parameters are expressed as metadata or properties -- not inferred
from geometry dimensions.

**Approach 1: Convention-based naming + `customData` dictionaries**

This is the lightest-weight approach and the closest to what exists today. It
formalizes the current practice:

- Connection point prims are **Xforms** under a `ConnectionPoints` scope, set
  to `guide` purpose, in a dedicated layer.
- A **standardized vocabulary of `customData` keys** replaces ad-hoc naming
  as the carrier of semantic identity, spatial extents, and domain properties.
  For example:

```
def Xform "fws_supply_main" {
    customData = {
        string connectionPoint:type = "thermal"
        string connectionPoint:direction = "supply"
        string connectionPoint:system = "FWS"
        double connectionPoint:portDiameter = 0.1016
        double connectionPoint:matingDepth = 0.05
        double connectionPoint:designFlowRate = 6.3
        string connectionPoint:fluidType = "water"
    }
}
```

A CAD exporter writes the keys it knows (type, direction, port diameter from
the feature dimension). A PLM integration pass adds the keys it knows (flow
rate, fluid type, allowed hoses). The USD file stays simple -- no schema
registration, no plugin dependencies, no geometry fabrication. Downstream
tools discover connection points by looking for prims under
`ConnectionPoints` scope with `connectionPoint:type` in `customData`.

*Strengths:* Minimal barrier to CAD tool adoption. Any tool that can write
`customData` can participate. PLM decoration is additive -- just append more
keys. No schema infrastructure required. No geometry to fabricate.

*Weaknesses:* No compile-time typing or validation. Key names are conventions,
not enforced by USD. Typos in key names fail silently. Discoverability depends
on knowing the convention.

**Approach 2: Namespaced properties on the prim**

Instead of `customData`, connection point metadata is expressed as
**namespaced USD properties** directly on the Xform prim:

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

This is similar to how `primvars:` and `xformOp:` namespaces work in USD.
Properties participate in composition (they can be overridden in stronger
layers), they have typed values, and they are queryable through the standard
property API. Spatial extents like `portDiameter` and `matingDepth` are typed
floats rather than dimensions inferred from geometry.

*Strengths:* Typed values catch mismatches at authoring time. Full composition
semantics -- a facility layout layer can override a design flow rate for a
specific installation. No schema plugin required. CAD exporters emit an Xform
with properties, PLM adds more properties in a stronger layer. No geometry
to fabricate.

*Weaknesses:* No schema-level discoverability (you must know the namespace
convention). No built-in validation of which keys are expected for a given
connection type. Still convention-based, just with stronger typing than
`customData`.

**Approach 3: Semantic labels via SemanticsLabelsAPI**

USD's built-in `SemanticsLabelsAPI` provides a mechanism for classifying prims
with semantic labels without defining a custom schema. A connection point prim
could carry labels like:

```
def Xform "fws_supply_main" (
    prepend apiSchemas = ["SemanticsLabelsAPI:aif"]
) {
    uniform token[] semantics:labels:category = ["aif", "thermal", "connectionpoint"]

    # Engineering properties still use the connectionPoint: namespace
    float connectionPoint:portDiameter = 0.1016
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
}
```

Tools discover connection points by querying for prims with the
`SemanticsLabelsAPI` applied and filtering by label values, using built-in USD
infrastructure that every USD runtime already supports.

*Strengths:* Schema-aware discovery using a built-in API -- no custom schema
registration or distribution needed. Consuming applications get label-based
discovery for free when they read USD files. Labels can be layered onto
existing assets via composition (a sublayer overlay adds semantic labels
without modifying the base vocabulary layer). This enables retroactive
adoption: assets authored with Approach 2 properties today can gain Approach 3
discoverability later without any change to the base asset.

*Weaknesses:* Adds a parallel classification surface that must stay
synchronized with the `connectionPoint:` property namespace -- a prim's
labels and its properties must agree on what kind of connection point it is.
Authoring complexity increases because two taxonomies (label hierarchy and
property vocabulary) must be maintained in concert. The label taxonomy is
untyped (string tokens), providing no validation beyond string matching.
ISV vendors and integrators who are new to USD may find the simpler
property-namespace approach more approachable as a first step.

**Approach 4: Applied API schema**

A formal applied API schema (e.g., `ConnectionPointAPI`) defines the
properties, their types, and their defaults in a schema definition. Tools
discover connection points by querying whether the schema is applied to a
prim.

*Strengths:* Maximum discoverability and validation. Schema-aware tools can
enumerate all connection points on a stage. Codegen provides typed accessors
in C++ and Python. The schema definition itself documents the expected
properties.

*Weaknesses:* Requires schema definition, registration, and distribution as a
plugin. CAD exporters must either depend on the schema plugin or replicate its
property layout manually. Higher adoption barrier for tool developers.
Schema evolution (adding new properties) requires versioning and
redistribution.

**Approaches 2 and 3 are complementary, not competing.** Namespaced properties
(Approach 2) carry the engineering data -- type, flow rate, port diameter,
flange rating. Semantic labels (Approach 3) provide a built-in discovery
mechanism for finding connection point prims across large stages. Because
`SemanticsLabelsAPI` is part of core OpenUSD, consuming applications gain
label-based discovery without installing custom schemas. Critically, semantic
labels can be **added retroactively as a composition sublayer** -- an asset
authored with Approach 2 properties today can gain Approach 3 discoverability
later without any change to the base vocabulary layer. This layered adoption
path means the community does not need to choose between the two approaches
up front.

**Recommended path: start simple, formalize later**

The practical reality of CAD-to-USD pipelines argues for starting with
**Approach 2 (namespaced properties on Xform prims)** as the near-term
target, with Approach 3 (semantic labels) as a complementary layer that can
be adopted incrementally:

- CAD export tools emit an **Xform** at the location of each identified
  connection feature (pipe nozzle, flange, port cutout, vent opening), with
  the transform oriented per the standard axis convention. No geometry needs
  to be fabricated -- the tool writes `connectionPoint:type = "thermal"`,
  `connectionPoint:portDiameter = 0.1016`, and other properties it knows
  directly from the design feature. This is simpler than emitting a mesh prim
  and no harder than writing any other USD property.
- Additional properties (`connectionPoint:designFlowRate`,
  `connectionPoint:allowedFluids`, etc.) can be added through any valid
  path -- PLM integration, direct CAD annotation, or manual editing -- in a
  composition layer that overrides or augments the initial properties through
  normal USD composition.
- A **published vocabulary specification** (a document, not a schema plugin)
  defines the standardized property names, types, allowed values, and which
  properties are expected for each connection domain -- including the spatial
  extent properties (port diameter, mating depth, service clearance) that
  replace geometry-inferred dimensions. This specification is what CAD and
  PLM tools target.
- The Asset Validation Framework validates that connection point prims carry
  the expected properties with valid values, using the vocabulary
  specification as its rule set.
- Semantic labels (Approach 3) can be layered on at any point via
  composition to improve discoverability across large stages without
  modifying base assets.
- If and when adoption reaches a point where a formal schema adds clear value
  (codegen, schema-level queries across large stages, ecosystem-wide
  interoperability), the namespaced properties can be **promoted to a formal
  applied API schema** (Approach 4) whose property layout matches the
  already-established conventions. Existing assets remain valid because the
  property names and types are unchanged.

This approach deepens the separation of concerns: the Xform's transform
captures position and orientation (including keying), namespaced properties
capture spatial extents, semantic identity, and domain-specific engineering
parameters, and no visual geometry is required. Each concern is independently
authorable, queryable, and consumable.

## Existing mechanisms and practices

### AI Factory connection points workflow

The current AIFDT workflow, documented in the
[Connection Points Workflow Doc](https://docs.google.com/document/d/1EiWmKriSCvmX8GvwQk_w8ZAmmyKnOjWyXicnIr3y6sw),
defines connection points as follows:

- **Geometry representation.** Simple mesh prims (planes for rectangular
  openings, disks for circular openings) positioned at the physical interface
  location.
- **Naming convention.** Each prim is named with a structured prefix:
  `<vendor>_<system>_<direction>_<suffix>`. For example,
  `vertiv_fws_supply_piping_connection_main`.
- **Layer separation.** Connection point prims are authored in a dedicated
  layer file (`<AssetName>_ConnectionPoints.usd`) and composed into the main
  asset as a sublayer.
- **Purpose.** Connection point prims are set to `guide` purpose so they are
  excluded from visual rendering.
- **Scope.** All connection point prims reside under a `ConnectionPoints`
  scope prim within the asset hierarchy.

This workflow has been implemented for NVIDIA GB300 racks and Vertiv cooling
equipment, and is documented in the
[aif-samples repository](https://gitlab-master.nvidia.com/omniverse/samples/kit/scripts/aif-samples).

### USD composition and layering

USD's composition engine provides mechanisms that are relevant to connection
points:

- **Sublayering** allows connection point data to be authored and managed
  separately from geometry, which is the approach currently used.
- **Applied API schemas** (e.g., `PhysicsRigidBodyAPI`, `SemanticsAPI`) provide
  a mechanism for attaching typed metadata to any prim, with schema-defined
  properties that are discoverable and queryable.
- **Relationships** allow prims to reference other prims, which could express
  connections between equipment.
- **Collections** allow grouping prims for batch operations, which could
  organize connection points by type or system.

### SimReady metadata

The SimReady asset specification defines metadata categories for AI Factory
equipment including thermal cooling and electrical properties. These are
captured in metadata matrices (e.g., the
[AI Factory SimReady Thermal Cooling Metadata Matrix](https://docs.google.com/document/d/1K7l-PqzhmDC-lqBEuxRwrz81bAVfyclzmKanjig55oE)
and Electrical Metadata Matrix) but exist as flat properties on the equipment
prim, not as properties of individual connection points.

### Limitations of current approaches

The existing mechanisms, while functional for initial use cases, have
limitations:

- **Naming encodes semantics.** Connection type, direction, and system
  membership are encoded in prim names rather than structured metadata. This
  requires string parsing, is error-prone, and provides no schema validation.
- **Geometry encodes spatial extents.** Interface dimensions (port diameter,
  vent area) are encoded in mesh geometry dimensions rather than typed
  properties. Consuming tools must measure a disk's radius to determine a
  pipe diameter -- conflating visual representation with engineering data.
- **Properties are not connection-scoped.** Thermal and electrical properties
  exist on the equipment prim rather than on individual connection points. A
  CDU with different flow rates on its FWS and TCS loops cannot express this
  per-connection.
- **No compatibility model.** There is no mechanism to express that a supply
  connection mates with a return connection of matching diameter and pressure
  rating.
- **Manual authoring only.** Without a structured vocabulary, CAD export tools
  cannot automatically generate connection points from source design
  features (e.g., pipe flanges, electrical terminals). The requirement to
  fabricate mesh geometry is a higher barrier for CAD exporters than emitting
  a positioned Xform with typed properties.
- **AIF-specific.** The naming conventions and workflow are tailored to
  datacenter equipment. Extending to robotics, process plants, or general
  manufacturing requires rethinking the conventions.

## Connection point domains

The following sections describe the five primary domains of connection points
and the properties relevant to each. These are not exhaustive but illustrate
the breadth of structured metadata that connection point Xforms should carry
as namespaced properties.

### Thermal cooling connections

Thermal cooling connections represent interfaces for liquid or refrigerant
piping systems. In datacenter contexts, these include Facility Water System
(FWS) and Technology Cooling System (TCS) piping.

| Property | Description | Example values |
|---|---|---|
| Connection type | The physical medium being transported | Liquid, refrigerant, glycol |
| System | The cooling system this connection belongs to | FWS, TCS, chilled water, condenser water |
| Direction | Whether the connection is supply or return | Supply, return, bidirectional |
| Port size | The nominal pipe or fitting size at the interface | 2", 4", DN50, DN100 |
| Flange rating | The pressure class of the connection | ANSI 150, ANSI 300, PN16 |
| Design flow rate | The rated volumetric flow through the connection | 100 GPM, 6.3 L/s |
| Design temperature | The rated temperature range | 45-65 &deg;F supply, 55-75 &deg;F return |
| Operating pressure | The rated working pressure | 150 PSI, 10 bar |
| Pressure drop | The expected pressure loss across the connection | 5 PSI, 0.3 bar |
| Fluid type | Working fluid (token encodes concentration) | `water`, `glycol_water_30`, `refrigerant_R410A` |
| Allowed fluids | Compatible fluids for this connection | Water, PG 30%, EGW 25% |
| Allowed particulates | Particulate tolerance of the connection | &lt; 50 micron |
| Allowed hoses | Compatible hose or pipe assemblies | Vendor-specific part list |
| Disconnect type | The physical disconnect mechanism | Quick-disconnect, threaded, flanged |
| Hot swap capable | Whether the connection supports live disconnection | Yes, no |
| Material | Housing and seal materials | Stainless steel 316, EPDM seals |

### Electrical connections

Electrical connections represent interfaces for power distribution. These range
from high-voltage facility feeds to low-voltage equipment power cords.

| Property | Description | Example values |
|---|---|---|
| Connection type | The category of electrical interface | Power input, power output, grounding |
| Voltage rating | Maximum rated voltage | 480V, 208V, 400V |
| Operating voltage | Actual operating voltage in the installation | 480V, 208V |
| Current rating | Maximum rated current capacity | 30A, 60A, 200A |
| Operating current | Actual operating current in the installation | 24A, 48A |
| Phase | Electrical phase configuration | Single-phase, three-phase |
| Frequency | Line frequency | 50 Hz, 60 Hz |
| Connector type | Physical connector standard | IEC 60309, NEMA L6-30, busbar, hardwired |
| Number of pins / conductors | The pin or conductor count | 3-pin, 4-pin, 5-conductor |
| Contact resistance | Electrical resistance at the contact interface | &lt; 1 m&Omega; |
| Redundancy group | Whether the connection is part of a redundant feed | Primary (A-feed), secondary (B-feed) |
| Hot swap capable | Whether the connection supports live insertion/removal | Yes, no |
| Allowed power whips | Compatible power cable assemblies | Vendor-specific part list |
| Material | Housing and contact materials | Copper alloy, silver-plated |
| Simulation models | Associated electrical simulation model references | SPICE model path |

### Airflow and ventilation connections

Airflow and ventilation connections represent the physical locations of air
intake and exhaust vents on equipment. Unlike piping connections that carry
liquid through a sealed interface, airflow connection points define open-air
boundaries where equipment draws in or expels air. The precise location, area,
and orientation of these vents on the equipment surface are critical inputs
for computational fluid dynamics (CFD) simulations that model facility-level
airflow patterns, hot/cold aisle effectiveness, and equipment cooling
performance.

| Property | Description | Example values |
|---|---|---|
| Connection type | The category of airflow interface | Intake vent, exhaust vent, recirculation port |
| Direction | Whether air flows into or out of the equipment | Intake, exhaust, bidirectional |
| Vent area | The aggregate open area of the vent surface | 0.25 m&sup2;, 2.5 ft&sup2; |
| Vent geometry | The shape of the vent opening | Rectangular, circular, perforated panel |
| Airflow volume | The rated volumetric airflow through the vent | 500 CFM, 850 m&sup3;/h |
| Static pressure | The pressure differential across the vent | 0.1" WG, 25 Pa |
| Temperature delta | The expected temperature rise or drop across the vent | +15 &deg;C (exhaust), ambient (intake) |
| Equipment face | Which face or surface of the equipment the vent is located on | Front, rear, top, side, bottom |
| Obstruction factor | Whether the vent area is reduced by grilles, filters, or perforations | 60% open area, MERV-8 filter |

In the current AIF workflow, airflow vents are modeled as rectangular planes
or circular disks at the vent location, with the engineering properties above
not captured in the prim -- they must be inferred or looked up externally.
With the Xform-based approach, each vent is an Xform positioned at the vent
location and oriented with the local Z-axis pointing in the airflow direction.
The vent area, obstruction factor, and other engineering properties are typed
properties on the Xform, directly available to CFD tools without geometry
measurement or external lookup.

### Network and data connections

Network and data connections represent physical ports for network connectivity,
data transfer, and control signaling. These are increasingly critical in AI
Factory environments where GPU racks, switches, and storage systems
interconnect through high-density fiber and copper interfaces, and in
manufacturing and robotics where fieldbus and industrial Ethernet networks
connect controllers, sensors, and actuators.

A key consideration is that network ports have a dual identity: a **physical
identity** (the connector type and its location -- e.g., rack 4, tray 3,
second port from left) and a **logical identity** (the OS-level address or
network role assigned in orchestration). Different use cases need different
identities. A single physical OSFP port may support multiple logical
configurations (e.g., 2x400G or 1x800G). The vocabulary captures the physical
interface; logical port mapping is an orchestration-layer concern deferred to
a future revision.

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
| Port type | The physical connector standard | OSFP, QSFP112, QSFP-DD, RJ45, LC fiber, M12 D-coded |
| Direction | Whether the port is ingress, egress, or bidirectional | Ingress, egress, bidirectional |
| Pin count | Number of pins or conductors | 38-pin (OSFP), 8-pin (RJ45) |
| Supported line rates | Data rates the port can operate at | 400GbE, 800GbE, 1GbE, 100Mbps |
| Supported configurations | Logical configurations the physical port supports | 1x800G, 2x400G, 8x100G |
| Supported power | Power delivery capability of the port | 12W (PoE), 1.5W (SFP) |
| Hot plug capable | Whether the port supports live insertion/removal of transceivers | Yes, no |
| Allowed transceivers | Compatible pluggable optic or cable modules | DR4, FR4, LR4, AOC, DAC |
| Required airflow | For air-cooled transceivers, the airflow needed | 5 CFM minimum per port |
| Configurable sub-ports | Whether the port can be split into sub-ports | Yes (breakout), no |
| Material | Housing and contact materials | Copper alloy, gold-plated contacts |
| Simulation models | Associated network simulation model references | Signal integrity model path |

Port naming and hierarchy are also important considerations. In the current
GB300 model, ports are defined at the lowest reuse block (e.g., compute tray)
and names scale with hierarchy: `<instantiation_name>_<port_name>` (e.g.,
`CT1_OSFP1`). Human-readable naming (e.g., `C1`-`C4` for compute network,
`M1` for management, `S1` for storage) serves a different purpose than the
physical port type naming (`OSFP1`, `QSFP1`, `ETH1`), and the schema should
accommodate both.

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
attachment interfaces. These encompass not only equipment-to-infrastructure
mounts but also machine-to-tooling interfaces (spindle-to-bit, wrist-to-
gripper) and modular accessory docking points. They are relevant across all
industrial domains.

| Property | Description | Example values |
|---|---|---|
| Connection type | The category of mechanical interface | Bolt flange, quick-connect, rail mount, weld joint, tool receptacle, gripper mount |
| Interface standard | The industry standard governing the interface geometry | ISO 9409-1 (robot flanges), CAT40, BT30, HSK-A63 (CNC tapers), ATI QC (tool changers) |
| Mounting orientation | The expected installation orientation | Floor mount, wall mount, ceiling mount, rack mount, spindle axis, wrist axis |
| Bolt pattern | The fastener arrangement | 4-bolt square 100mm, 6-bolt circular 150mm PCD, ISO 9409-1-50-4-M6 |
| Load / payload rating | Maximum supported static and dynamic load or payload | 500 kg static, 200 kg dynamic, 10 kg payload at 1m reach |
| Speed / RPM limits | For rotary interfaces, the maximum operating speed | 8000 RPM, 24000 RPM, 15 rad/s |
| Vibration isolation | Whether the mount requires vibration damping | Required, optional, not applicable |
| Pneumatic / hydraulic | For connections carrying compressed air or hydraulic fluid | 6mm push-fit, 1/4" NPT, G1/2 BSP |
| Electrical pass-through | For interfaces that carry electrical signals alongside mechanical coupling | 12-pin, 24V signal, EtherCAT pass-through |
| Data / network | For physical data connectivity ports | RJ45, LC fiber, M12 D-coded |
| Compatible accessories | Enumeration or reference to compatible tooling or accessories | List of compatible gripper models, tool bit types, sensor modules |

### Cross-cutting properties

Regardless of domain, nearly all physical connection points share a set of
mechanical and physical properties that describe the interface itself. These
properties appear consistently across power, coolant, network, and mechanical
connections in the
[SimReady Connection Points Examples](https://docs.google.com/document/d/1jZJGJpjW8kLDT6PubSTAD4WYMbWgPQM46_pkXdQli0I)
analysis and should be factored into a base schema rather than duplicated
across domain-specific schemas.

| Property | Description | Applies to |
|---|---|---|
| Port diameter / interface area | The physical dimension of the interface opening; replaces inferring size from geometry | All domains |
| Mating depth | How far a connector, pipe, or tool inserts before fully seated. (Community naming discussion: "insertion depth" has been suggested as an alternative term -- feedback welcome.) | Coolant, network, mechanical |
| Service clearance | Minimum clear space required around the connection for access, cable bend radii, or tool change swing paths | All domains |
| Insertion force | The force required to mate the connection | Power, coolant, network, mechanical |
| Mated cycle count | Rated number of insertion/removal cycles | Power, coolant, network, mechanical |
| Retention mechanism | How the connection is secured once mated | Power, coolant, network, mechanical |
| Keying | Physical keying that prevents incorrect mating orientation (the Xform's rotational orientation captures the keying direction; this property describes the keying type) | Power, coolant, network, mechanical |
| Temperature rating | Operating temperature range of the connection | All domains |
| Shock and vibration | Maximum shock and vibration the connection can withstand | Power, coolant, network, mechanical |
| Material | Housing, contact, and seal materials | All domains |

Note that **location and orientation** are no longer listed as metadata
properties -- they are intrinsic to the Xform's transform. This is a clean
separation: the Xform prim captures *where* and *which direction*, while the
properties above capture *how big*, *how deep*, *how much clearance*, and the
physical characteristics of the interface. Spatial extents that were
previously inferred from geometry dimensions (measuring a disk radius to
determine a port diameter, or a plane's width to determine a vent area) are
now explicit, typed, queryable properties -- reinforcing the separation of
concerns between visual representation and engineering metadata.

These cross-cutting properties argue strongly for a base `connectionPoint:`
property namespace that captures the shared physical interface characteristics,
with domain-specific prefixes (e.g., `connectionPoint:thermal:`,
`connectionPoint:electrical:`) adding their own properties. If the community
later adopts a formal applied API schema, these cross-cutting properties map
naturally to a base `ConnectionPointAPI`.

## Industry use cases

The following examples illustrate the breadth of connection point requirements
across industries. They demonstrate that a generalized connection point model
serves a far wider set of stakeholders than the current AIF-specific approach.

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
  specifications. These drive electrical load balancing and capacity planning
  simulations.
- **Network port interfaces** -- OSFP, QSFP112, and RJ45 ports on compute
  trays and switches, with precise coordinates, orientations, supported line
  rates, and transceiver compatibility. These connection points drive fiber
  BOM generation, point-to-point cable routing, and cable length optimization.
  Logical network connectivity (defined in SysML or similar tools) binds to
  these physical port connection points.
- **Airflow vent locations** -- The physical positions of intake and exhaust
  vents on equipment cabinets. GPU racks draw cool air through their front
  face and expel heated air from the rear; CRAHs discharge conditioned air
  downward through raised-floor tiles; PDUs and UPS systems have their own
  ventilation patterns. Each vent connection point carries the location,
  area, airflow volume, temperature delta, and equipment face -- properties
  that CFD simulation tools consume directly to model hot/cold aisle
  effectiveness, recirculation risks, and facility-level thermal performance.

The current AIF workflow manually creates these as geometry prims with naming
conventions, but the target state is automated extraction from CAD source data
and structured metadata that simulation runtimes can consume directly.

### Visual Factory Intelligence (VFI)

VFI extends digital twin capabilities to general manufacturing and industrial
facilities. Equipment in these environments has diverse connection needs:

- **CNC machines and manufacturing cells** have connection points at multiple
  levels: facility-level connections for coolant supply, compressed air, and
  electrical power, but also **machine-level tooling interfaces** such as
  spindles that accept specific tool bit tapers (CAT40, BT30, HSK-A63). A
  spindle connection point defines which cutting tools are compatible,
  constrained by taper standard, maximum RPM, torque rating, and coolant-
  through capability. This is a connection point in the same sense as a pipe
  fitting -- it has geometry, identity, and compatibility constraints.
- **Conveyor systems** have mechanical mounting points with specific load
  ratings and bolt patterns, drive motor electrical connections, and sensor/
  network data connections for control system integration.
- **HVAC and environmental systems** connect to ductwork (intake and exhaust),
  chilled water piping, condensate drains, and electrical power. The physical
  locations of supply and return air grilles, exhaust hoods, and make-up air
  intakes are airflow connection points that drive environmental CFD
  simulation for worker comfort, fume extraction, and process temperature
  requirements.

VFI use cases emphasize the need for **mechanical** connection points (mounting
interfaces, structural supports) alongside thermal and electrical connections,
as factory layout must account for physical mounting constraints. Note that
manufacturing equipment and robotic systems share many of the same connection
point patterns -- a CNC spindle is itself a robotic tool interface, and a
robot wrist flange is a mechanical mount. The vocabulary treats these as a
single continuum of mechanical/tooling interfaces.

### Robotics and autonomous systems

Robot arms, humanoid robots, autonomous mobile robots (AMRs), and robotic
workcells all present rich connection point requirements that span every
domain.

**Industrial robot arms and workcells:**

- **End-of-arm tooling (EOAT) interfaces** are connection points on a robot's
  wrist flange that define which grippers, welding torches, inspection
  cameras, or other tools can physically attach and operate. The wrist flange
  connection point is governed by bolt pattern (ISO 9409-1), payload capacity,
  and pneumatic/electrical pass-through channels. Tool changers (e.g., ATI,
  Schunk) add another layer: the tool changer itself is a connection point
  with its own mechanical, pneumatic, and electrical interface specifications.
  A digital twin that models these attachment points enables simulation of
  tool changes, validation of gripper compatibility before procurement, and
  automated workcell configuration.
- **Robot base mounting** involves mechanical connections to floor plates,
  pedestals, or linear tracks with specific bolt patterns, load ratings, and
  vibration isolation requirements.
- **Peripheral equipment** (feeders, fixtures, conveyors, safety fencing)
  connects to the workcell through mechanical mounts, electrical power, and
  data/fieldbus networks (EtherCAT, PROFINET, EtherNet/IP).

**Humanoid robots:**

- **Modular limb and joint interfaces** -- Humanoid robots are increasingly
  designed with modular architectures where limb segments, hands, and sensor
  heads attach through standardized mechanical interfaces with electrical and
  data pass-through. Each joint or limb attachment point is a connection point
  that defines mechanical load capacity, actuator power requirements, and
  communication bus connectivity.
- **End-effector / hand interfaces** -- Humanoid hands and grippers attach at
  the wrist through connection points that must carry mechanical torque
  ratings, finger actuator power, tactile sensor data channels, and in some
  cases pneumatic lines for soft grippers. These connection points define
  which hands are compatible with which arm platforms.
- **Sensor and perception module mounts** -- Camera heads, LiDAR units, and
  microphone arrays mount to standardized brackets on the torso or head, with
  connection points carrying mechanical mounting constraints, power delivery,
  and high-bandwidth data interfaces.
- **Battery and power module docking** -- Swappable battery packs dock through
  connection points that combine electrical (high-current charging contacts),
  mechanical (latching and alignment), and data (battery management system
  communication) interfaces.

**Autonomous mobile robots (AMRs):**

- **Charging stations** represent connection points that combine electrical
  (charging contacts or inductive pads), mechanical (docking alignment), and
  data (fleet management communication) interfaces.
- **Payload module interfaces** -- AMRs often carry interchangeable payload
  modules (conveyors, lifts, collaborative robot arms) that attach through
  standardized top-plate connection points with mechanical, electrical, and
  data specifications.

Robotics use cases highlight the need for **multi-domain connection points** --
a single physical interface that carries mechanical, electrical, pneumatic, and
data connections simultaneously. They also demonstrate the importance of CAD
and PLM sourcing: robot manufacturers define their interface specifications in
CAD and product documentation, and downstream integrators need structured
connection point metadata to validate compatibility when configuring workcells,
selecting tooling, or swapping humanoid robot modules in simulation.

### Industrial equipment and process plants

Process industries (oil & gas, chemical, pharmaceutical, food & beverage)
have the most mature standards for connection interfaces:

- **Piping connections** follow well-defined standards (ASME B16.5 flanges, DIN
  EN 1092, ISO 7005) with specific ratings, materials, and gasket requirements.
  Process plant design tools (e.g., AVEVA E3D, Hexagon Smart 3D, Bentley
  OpenPlant) already model these connections in detail.
- **Instrumentation connections** (thermowell ports, pressure taps, analyzer
  sample points) combine mechanical interfaces (threaded fittings, flanged
  nozzles) with process parameters (temperature range, pressure class,
  wetted materials).
- **Vessel nozzles** on reactors, columns, and heat exchangers are connection
  points with complex properties: nozzle schedule, flange rating, projection
  length, reinforcement pad dimensions, and process conditions (pressure,
  temperature, fluid type, corrosion allowance).

Process industry use cases demonstrate the need for **standards-based
properties** on connection points -- properties that reference industry
standards (ASME, ISO, DIN) rather than arbitrary numeric values.

## Consumption workflows

The industry use cases above describe *what* tools need from connection
points. This section traces *how* representative tools query and consume the
data, because the consumption pattern drives asset structuring decisions. The
right answer for where connection point data lives in the asset hierarchy,
whether it should be accessible before loading payloads, and how tools
discover it all depend on understanding the actual access patterns.

### CFD thermal simulation (e.g., Cadence Reality DT)

A CFD tool modeling facility-level thermal performance needs the location,
area, airflow volume, and temperature delta of every vent zone on every piece
of equipment -- plus pipe diameters, flow rates, and temperatures for every
liquid cooling connection. Each airflow connection point maps to one boundary
condition in the simulation mesh (an aggregate vent zone, not individual
perforations). Each thermal piping connection point provides the inlet/outlet
conditions for a fluid circuit.

The tool traverses the asset hierarchy, queries connection point Xforms by
`connectionPoint:type` (thermal, airflow), reads the typed properties, and
feeds them into its boundary condition setup. It does not need the equipment's
visual geometry at this stage -- it needs engineering parameters for boundary
condition definition before any mesh is relevant.

**Key requirement:** Connection point metadata must be accessible **without
loading heavy geometry** -- ideally before loading payloads. This argues for
placing connection point data in a lightweight interface layer per the
reference-payload pattern, which the existing `_ConnectionPoints.usd` sublayer
already achieves. The tool needs typed properties (`connectionPoint:ventArea`,
`connectionPoint:designFlowRate`), not mesh dimensions to measure.

### BOM generation and cable routing

BOM generation and cable routing for AI Factory network fabrics require
discovering and classifying network port connection points across an entire
facility scene. Different cable schedules target different port populations,
and the tool must efficiently filter the full population without traversing
every prim in the scene graph:

- **Fiber BOM** (compute/storage fabric): Filters by
  `connectionPoint:type == "network"` and
  `connectionPoint:network:medium == "fiber"`, reads `portType` and
  `allowedTransceivers`, and generates point-to-point cable schedules with
  length optimization based on physical coordinates.
- **Copper patch schedule** (management network): Filters by
  `connectionPoint:network:category == "management"` and
  `connectionPoint:network:medium == "copper"`, generates copper cable runs
  with category requirements (Cat6a, Cat6).
- **DAC cable schedule** (short-reach fabric): Filters by
  `connectionPoint:network:medium == "DAC"`, generates direct-attach copper
  cable lists with length constraints.

In all cases, the tool expects connection points at a specific depth in the
`kind` hierarchy (below `component`) and uses the physical coordinates for
cable length calculations.

**Key requirement:** Connection points must be discoverable by broad type
(`connectionPoint:type == "network"`) and further filterable by medium,
category, and fabric role without full scene graph traversal. The broad type
query enables scene-wide network port enumeration; classification properties
enable targeted BOM generation for specific cable schedules. This argues for
both a well-known scope prim (`ConnectionPoints`) and property namespace
queries as complementary discovery mechanisms.

### Connection compatibility validation

Given two equipment assets placed adjacent in a facility layout, a validation
tool checks whether their facing connection points are compatible: matching
pipe diameters, compatible pressure ratings, aligned bolt patterns, or
matching electrical phases and voltages. The tool queries `connectionPoint:`
properties on both sides of a proposed connection and applies compatibility
rules from the vocabulary specification.

This workflow extends to tooling interfaces: verifying that a specific gripper
is compatible with a robot's wrist flange (matching bolt pattern, sufficient
payload capacity, required pass-through channels), or that a cutting tool
matches a CNC spindle's taper standard and RPM limits.

**Key requirement:** Compatibility checking requires structured, typed
properties on both sides of the connection. This is impossible with naming
conventions alone and is the strongest argument for the `connectionPoint:`
property namespace. The vocabulary specification defines which properties must
match and which must be within compatible ranges.

### Cross-domain validation (SimReady Foundation)

The AIF-to-SimReady Foundation migration defines cross-domain validators
([TC.002](https://gitlab-master.nvidia.com/bperschall/aif_simready_migration_plan),
[EL.004](https://gitlab-master.nvidia.com/bperschall/aif_simready_migration_plan))
that check whether an asset's metadata declarations are consistent with its
connection point prims. For example, a CDU declaring FWS/TCS piping metadata
must have corresponding FWS/TCS connection point prims. Today these validators
regex-match prim names; with `connectionPoint:type` and
`connectionPoint:system` as typed properties, they become property queries --
more robust, extensible, and not dependent on naming convention stability.

**Key requirement:** Validators must be able to correlate equipment-level
metadata (e.g., "this CDU supports FWS cooling") with the presence and
properties of specific connection point prims. Property-based queries are
more reliable than name-pattern matching and generalize across vendors and
naming conventions.

### CAD export pipeline

A CAD exporter (e.g., PTC Creo, Dassault SOLIDWORKS, Siemens NX) identifies
connection features in the design model and emits an Xform at each one with
typed properties from the design dimensions. No geometry needs to be
fabricated, no naming conventions need to be reverse-engineered. The exporter
writes what it knows (position, orientation, port diameter, interface standard)
and downstream PLM integration adds what it knows (operating parameters,
allowed mating components) in a composition layer.

**Key requirement:** The barrier for CAD tools must be low -- an Xform with
typed properties, no schema plugin dependencies. Partially populated
connection points are valid and useful; the CAD exporter should not be
blocked by properties it does not have access to.

## Data sourcing: CAD and PLM

A standardized connection point schema is only as useful as the data that
populates it. A key objective of this proposal is to enable **automated
population** of connection point properties from the systems where that data
originates, rather than requiring manual authoring in USD.

### What comes from CAD

CAD design software is the authoritative source for the physical and geometric
aspects of connection points. Much of the connection point data can and should
be extracted directly from the CAD model during USD export. With the
Xform-based representation, the CAD exporter's task is straightforward: emit
an Xform at each identified connection feature with the transform oriented per
the standard axis convention, and write the properties it knows directly from
the design feature dimensions. No geometry needs to be fabricated.

- **Location and orientation** -- The position, surface normal, and insertion
  direction of pipe nozzles, electrical terminals, vent openings, network
  port cutouts, and mounting holes are defined in the CAD geometry. The
  exporter maps these to the Xform's translation and rotation, with the
  local Z-axis aligned to the connection direction and the local Y-axis
  aligned to the keying orientation where applicable.
- **Spatial extents as properties** -- Pipe diameters, connector widths, vent
  opening areas, and flange bolt circle diameters are dimensioned in the
  design. The exporter writes these as typed properties
  (`connectionPoint:portDiameter`, `connectionPoint:ventArea`,
  `connectionPoint:boltPatternPCD`) rather than encoding them in geometry
  dimensions that must be measured after the fact.
- **Interface standards** -- When a designer specifies a CAT40 spindle taper,
  an ISO 9409-1 robot flange, or an OSFP port cutout, the standard governs
  the geometry and implicitly defines compatibility constraints.
- **Mechanical properties** -- Bolt patterns, mounting orientations, and
  structural load paths are inherent in the design geometry and exported as
  properties.

This model lowers the barrier for CAD export tools: emitting an Xform with a
handful of typed properties is simpler than constructing a correctly
tessellated mesh prim. It also supports **parametric updates**: when a
designer revises a model -- changing a pipe diameter, relocating a vent,
adding a port -- the connection point Xform and its properties regenerate
automatically through the USD export pipeline, maintaining consistency between
the design and its digital twin.

### What comes from PLM (or other engineering data sources)

Operating parameters, material specifications, and compatibility data may come
from PLM systems, but PLM is not the only valid source. Direct CAD annotation,
manual editing of the USD layer, or custom scripts are equally valid paths.
The vocabulary is agnostic to data provenance -- what matters is that the
properties are present and correctly typed.

Product Lifecycle Management (PLM) systems, where available, carry information
that is not represented in the CAD geometry but is essential for connection
point completeness:

- **Operating parameters** -- Design flow rates, operating pressures and
  temperatures, voltage and current ratings are specified in engineering
  documentation managed by PLM, not in the geometry.
- **Material specifications** -- Housing materials, contact plating, seal
  compounds, and fluid compatibility data live in material databases linked
  through PLM.
- **Allowed mating components** -- Lists of compatible hoses, power whips,
  transceivers, tooling bits, and grippers are maintained as approved vendor
  lists (AVLs) or part interchangeability records in PLM.
- **Simulation model references** -- SPICE models for electrical connections,
  signal integrity models for network ports, and thermal resistance models
  for coolant connections are managed as PLM artifacts.
- **Certification and compliance** -- Pressure ratings (ASME, PED),
  electrical certifications (UL, CE), and environmental ratings (IP/NEMA) are
  PLM-managed metadata.
- **Revision and lifecycle state** -- Part revision identifiers, approval
  status, and obsolescence information come from PLM and affect which
  connection point specifications are current.

### Parametric updates and revision continuity

When CAD and PLM are the sources of truth for connection point data, the USD
representation becomes a **downstream artifact** that should regenerate
cleanly when upstream data changes. This has important implications:

- **Design revisions** that change a pipe diameter, relocate a vent, or add a
  network port should propagate through the CAD-to-USD pipeline and
  automatically update the affected connection point prims and their
  properties.
- **PLM updates** that change an operating parameter, add a new allowed
  transceiver, or revise a material specification should flow into the USD
  metadata without requiring manual re-authoring.
- **Version continuity** requires that connection point identity survive
  revisions. If a CDU's FWS supply connection moves from one location to
  another in a design revision, downstream layouts and routing that reference
  that connection point should be able to track the change.

This sourcing model reinforces the need for a schema-based approach: CAD
exporters and PLM integrations need a well-defined target -- whether a
vocabulary specification or a formal schema -- not an ad-hoc naming convention
to reverse-engineer.

## Design considerations

This section does not propose a solution but outlines principles and open
questions to guide the community toward one.

### Principles

1. **Separation of concerns.** Connection point position and orientation
   (where and which direction), semantic identity (what), and domain
   properties including spatial extents (how big, how deep, how much
   clearance) serve different purposes and are consumed by different tools.
   The Xform's transform captures the first concern; namespaced properties
   capture the second and third. Spatial extents are explicit metadata, not
   inferred from geometry -- because the engineering dimension (a port
   diameter) and the visual representation (a disk mesh) are different
   concerns.

2. **Domain agnosticism with domain extensibility.** The base connection point
   model should not be specific to datacenter cooling or any single industry.
   It should provide a minimal, shared abstraction (type, direction, system)
   that domain-specific property sets extend with their own properties.

3. **Discoverability without heavy infrastructure.** Connection points should
   be discoverable through standardized conventions -- a well-known property
   namespace (e.g., `connectionPoint:`), a well-known scope prim
   (`ConnectionPoints`), and a well-known purpose (`guide`). This
   discoverability should not require a schema plugin to be installed. A
   formal applied API schema may be a future step, but discoverability should
   work from day one with conventions alone.

4. **Composability.** Connection point metadata should participate in USD's
   composition model. When an equipment asset is referenced into a facility
   layout, its connection points and their properties should compose correctly.
   Overrides on connection properties in the facility layer should be
   supported (e.g., adjusting a design flow rate for a specific installation).
   This argues for namespaced properties over `customData`, since properties
   participate in composition while `customData` does not compose
   element-wise across layers.

5. **Low barrier for CAD tools.** The representation must be something that
   CAD export tools can emit simply -- an Xform at the feature location with
   typed properties -- without requiring schema plugin dependencies, complex
   registration, or geometry fabrication. A CAD tool that identifies a pipe
   flange should be able to emit an Xform with
   `connectionPoint:type = "thermal"` and
   `connectionPoint:portDiameter = 0.1016` as easily as it writes any other
   USD property. Emitting an Xform with properties is simpler than
   constructing a correctly tessellated mesh prim.

6. **Additive decoration from any valid source.** Additional properties
   (operating parameters, allowed mating components, simulation models) should
   layer onto the initial connection point through normal USD composition -- a
   stronger layer or sublayer that adds or overrides properties. PLM
   integration is one valid source for these properties, but direct CAD
   annotation and manual editing of the USD layer are equally valid paths.
   The vocabulary is agnostic to data provenance -- what matters is that the
   properties are present and correctly typed. The connection point should be
   usable in a partially populated state (Xform + type + spatial extents
   initially, operating parameters added later). Parametric updates from
   design revisions should propagate cleanly.

7. **Validation-ready.** The property vocabulary should carry enough structured
   information that connection compatibility can be checked programmatically
   through the Asset Validation Framework -- matching pipe diameters,
   compatible voltages, aligned bolt patterns. Validation rules reference the
   vocabulary specification, not a compiled schema.

8. **Minimal disruption.** The solution should build on USD's existing
   strengths -- composition, namespaced properties, purpose-based visibility
   control. It should not require changes to the composition engine, new prim
   types, or mandatory schema plugins. A future applied API schema should be
   an optional upgrade, not a prerequisite.

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

**Discoverability** -- flagged as the most important aspect by stakeholders --
is served by complementary mechanisms:

1. **Scope prim discovery:** A tool can find all connection points on an asset
   by looking for the well-known `ConnectionPoints` child scope under the
   `defaultPrim`. This works for hierarchy-aware traversal.
2. **Property namespace discovery:** A tool can query for any prim carrying
   `connectionPoint:` properties, regardless of where it sits in the hierarchy.
   This works for scene-wide queries across multiple assets.
3. **Semantic label discovery (future/complementary):** USD's built-in
   `SemanticsLabelsAPI` can classify prims with labels like
   `["aif", "thermal", "connectionpoint"]`, enabling schema-aware discovery
   across large stages without custom schemas. This can be added as a
   composition sublayer overlay without changing existing property-based
   assets (see Approach 3 in the representation spectrum above).

Mechanisms 1 and 2 are recommended from day one. Mechanism 3 is a
complementary option that can be adopted incrementally as tooling matures.
Connection point metadata should be accessible before loading payloads,
ensuring that lightweight discovery and planning tools do not require
geometry loading.

### Open questions for discussion

1. **Base vocabulary scope.** What belongs in the base `connectionPoint:`
   namespace vs. domain-specific property prefixes (e.g.,
   `connectionPoint:thermal:`, `connectionPoint:electrical:`)? Candidates for
   the base: connection type, direction, system identifier, the spatial extent
   properties (port diameter, mating depth, service clearance), and the
   cross-cutting mechanical properties (insertion force, mated cycle count,
   retention mechanism, keying type, temperature rating, shock and vibration).
   If the community later adopts an applied API schema, the base namespace
   maps directly to a `ConnectionPointAPI`.

2. **Multi-domain connection points.** A robot tool changer is simultaneously a
   mechanical, pneumatic, and electrical connection. A CNC spindle is both a
   mechanical tooling interface and a coolant-through fluid connection. Should
   a single prim carry properties from multiple domain prefixes, or should
   multi-domain interfaces be modeled as a group of co-located single-domain
   connection prims?

3. **Connection relationships.** Should the vocabulary support explicit
   relationships between mated connection points (e.g., a relationship from a
   CDU's FWS supply to the piping segment it connects to)? Or should mating
   be expressed only through spatial proximity and external tooling?

4. **Layer authoring model.** The current practice of authoring connection
   points in a separate layer file (`_ConnectionPoints.usd`) provides clean
   separation of connection data from equipment geometry. Should this layer
   convention be formalized? This aligns well with the CAD/PLM sourcing model:
   the CAD exporter produces the connection points layer with Xforms and basic
   properties, and a PLM integration layer adds operating parameters in a
   composition arc.

5. **Relationship to `guide` purpose.** Connection point Xforms are currently
   set to `guide` purpose (inherited from the geometry-based approach) to
   exclude them from rendering. With Xform prims that carry no geometry, the
   `guide` purpose still serves as a signal that these prims are metadata
   scaffolding rather than visible scene elements. Should this remain the
   convention, and should the vocabulary specification mandate it?

6. **Units and standards.** Engineering properties (flow rate, pressure,
   temperature) require units. Should the vocabulary enforce SI units, support
   arbitrary unit annotations, or defer to USD's existing
   `UsdGeomLinearUnits` patterns?

7. **Backward compatibility with existing AIF assets.** Existing GB300 and
   Vertiv assets use the current naming convention. The solution should define
   a migration path that preserves compatibility with these assets.

8. **CAD and PLM integration boundaries.** Which connection point properties
   should CAD exporters be responsible for populating (position, orientation,
   spatial extents, interface standards), and which should come from PLM
   integration (operating parameters, allowed mating components, simulation
   models)? How should the vocabulary accommodate partial population --
   connection points that have CAD-sourced Xforms and spatial extents but
   are awaiting PLM-sourced operating parameters?

9. **Port naming and hierarchy.** For network and data ports, how should the
   schema relate physical port identity (e.g., `OSFP1`) to logical network
   identity (e.g., `C1` for compute network)? The vocabulary captures the
   physical interface; logical port mapping is an orchestration-layer concern.
   This question is deferred to a future revision, with the expectation that
   logical identity will be managed in a separate system model layer composed
   over the physical port connection points.

## Relationship to other proposals and frameworks

This proposal is related to several other efforts in the OpenUSD ecosystem.
The relationship to the SimReady Foundation, the Identifier Separation of
Concerns proposal, and the Asset Structure Principles are particularly
significant.

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

The [Separation of Concerns for Identifiers in USD](https://github.com/PixarAnimationStudios/OpenUSD-proposals/pull/105)
proposal (Luk, 2026) and this proposal are **instances of the same fundamental
problem**: USD prim names are being overloaded to carry information that should
live in structured metadata.

| | **Identifiers proposal** | **Connection points proposal** |
|---|---|---|
| **What is being encoded in prim names** | External system identifiers (IFC GUIDs, PLM part numbers, classification codes) | Connection semantics (type, direction, vendor, system) |
| **Why it doesn't fit** | External identifiers contain characters invalid in USD grammar, and conflate namespace identity with source identity | Connection semantics require string parsing to extract, are error-prone, and cannot carry engineering properties |
| **What is lost** | Round-trip fidelity to source systems, discoverability, cross-system linking | Structured queryability, compatibility validation, CAD/PLM automation |
| **Current workarounds** | `customData`, `displayName`, `assetInfo` -- all inadequate | Naming conventions, `customData`, SimReady metadata on equipment prim -- all inadequate |

The proposals share several core principles:

1. **Prim names should serve composition, not carry payload.** Both proposals
   argue that the USD namespace path should address a prim in the composed
   stage, and that any additional identity or metadata should live in a
   separate, structured mechanism -- not be shoe-horned into the name string.

2. **The same existing mechanisms are insufficient for both problems.**
   `customData` lacks standardization and discoverability. `assetInfo` is
   scoped to model roots. `displayName` conflates display with identity.
   Both proposals need a mechanism that is standardized, discoverable, and
   applicable to any prim.

3. **Composability is essential.** Both proposals require that the metadata
   compose correctly through USD's composition engine -- surviving references,
   payloads, and layer overrides. This shared requirement argues for
   properties (which compose) over `customData` (which does not compose
   element-wise).

4. **Industry agnosticism.** Both proposals explicitly aim for cross-industry
   applicability rather than being tied to a single vertical.

**Where the proposals are complementary:**

A connection point prim carries two distinct categories of "non-namespace"
information that map to the two proposals:

- **Source identity** (Luk's concern) -- The connection point's identity in
  the originating CAD system (a feature ID, a port name) and in PLM (a part
  number, a specification reference). This is the external identifier that
  must survive round-trips and enable cross-system linking. Aaron's proposal
  provides the mechanism for this.

- **Domain-specific properties** (this proposal's concern) -- The connection
  point's type, direction, engineering parameters, and compatibility
  constraints. This is the structured metadata that enables simulation,
  validation, and automated configuration.

These are not competing -- they are **different layers of metadata on the same
prim**. A CDU's FWS supply pipe connection point needs both:
- A source identifier linking it back to the CAD feature and PLM part
  (Aaron's mechanism)
- A set of thermal cooling properties (type, direction, flow rate, pipe
  diameter) for simulation and validation (this proposal's vocabulary)

**Potential for a unified approach:**

If both proposals converge on namespaced properties as the representation
mechanism, they could share a common pattern. Both namespaces coexist on the
same Xform prim, compose independently through USD's layer stack, and are
discoverable through namespace-based queries:

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

- **SimReady Specification** -- Defines metadata categories for
  simulation-ready assets including thermal and electrical properties. The
  connection point vocabulary should integrate with (not duplicate) SimReady
  metadata, moving per-connection properties from the equipment prim to
  individual connection point prims.

- **Asset Validation Framework** -- Enables validation rules for connection
  point completeness, property ranges, and compatibility without a formal
  schema plugin. The AIF-to-SimReady migration already defines cross-domain
  validators (TC.002, EL.004) that check metadata-to-connection-point
  consistency; the `connectionPoint:` property namespace would make these
  checks more robust by replacing name-pattern matching with property queries.

- **USD Physics Schema** -- Demonstrates the pattern of applied API schemas
  (`PhysicsRigidBodyAPI`, `PhysicsCollisionAPI`, `PhysicsJoint`) for attaching
  simulation-relevant metadata to geometry prims. If the connection point
  vocabulary matures to the point where a formal schema is warranted, the
  Physics schema pattern provides a proven model to follow.

## Acknowledgments

The author thanks the following individuals whose technical feedback and domain
expertise materially shaped this proposal:

- **Steve Ghee** (PTC) -- Environmental interface framing, SemanticsLabelsAPI
  discovery approach, industrial equipment generalization, PLM optionality,
  bolt pattern and tolerance considerations, and terminology refinements
  including the "insertion depth" naming suggestion.
- **Steve Blackwell** (Vertiv) -- Fluid type simplification, use-case-first
  design principle, IFC mapping potential, directionality feedback, and
  extensive datacenter equipment domain expertise across thermal, electrical,
  and airflow domains.
- **Aaron Gilroy** (NVIDIA) -- Logical schematic concept for network port
  identity, BOM generation workflow insights, and identifier separation of
  concerns alignment.
- **Asmita Wankhede** (NVIDIA) -- Venue guidance (SRF positioning), naming
  concerns in 3D context, Newton schema alignment direction, and SME
  coordination.
- **Jeremie Pigny** (Dassault) -- Skeleton representation alignment,
  3DEXPERIENCE export path considerations, and SI unit feedback.
- **Aaron Luk** (NVIDIA) -- Identifier separation of concerns proposal
  alignment and PCP coaching framework.

---

## Appendix A: Current AIF connection point naming conventions

The following table documents the naming conventions currently used for
connection points in the AI Factory Digital Twin workflow. This serves as both
a reference for the existing approach and a requirements baseline for any
schema-based replacement.

| Connection point type | USD prim naming convention | Example |
|---|---|---|
| Generic liquid intake pipe | `<vendor>_liq_supply_` | `vertiv_liq_supply_primary_01` |
| Generic liquid outflow pipe | `<vendor>_liq_return_` | `trane_liq_return_secondary` |
| FWS supply pipe | `<vendor>_fws_supply_piping_connection_` | `vertiv_fws_supply_piping_connection_main` |
| FWS return pipe | `<vendor>_fws_return_piping_connection_` | `vertiv_fws_return_piping_connection_main` |
| TCS supply pipe | `<vendor>_tcs_supply_piping_connection_` | `vertiv_tcs_supply_piping_connection_main` |
| TCS return pipe | `<vendor>_tcs_return_piping_connection_` | `vertiv_tcs_return_piping_connection_main` |
| Electrical power socket | `<vendor>_electrical_nominal_voltage_` | `trane_electrical_nominal_voltage_main` |
| Airflow intake vent | `<vendor>_airvent_intake_` | `trane_airvent_intake_frontplate` |
| Airflow outflow vent | `<vendor>_airvent_outflow_` | `trane_airvent_outflow_cabinet_top` |

All connection point prims:

- Reside under a `ConnectionPoints` scope prim.
- Are set to `guide` purpose (not rendered visually).
- Use simple mesh geometry (planes for rectangular, disks for circular
  openings) sized and positioned to match the physical interface.
- Are authored in a separate `<AssetName>_ConnectionPoints.usd` layer file
  composed into the main asset via sublayering.

## Appendix B: Provenance and AI-assisted drafting

This proposal was drafted with the assistance of an AI language model (Claude,
Anthropic) operating within Cursor IDE, under the direction of Beau Perschall.
All conceptual framing, editorial decisions, and technical judgment are the
responsibility of the human author. The AI was used as a drafting tool to
accelerate the writing process based on context and direction provided by the
author.

### Context provided to the AI

The following materials were provided as input context for drafting:

1. **[Separation of Concerns for Identifiers in USD](https://github.com/PixarAnimationStudios/OpenUSD-proposals/pull/105)**
   (Luk, 2026) -- Used as a style and structure reference for this proposal.

2. **[OMPE-67498: AIFDT-UID-16: Physical Connection Points](https://jirasw.nvidia.com/browse/OMPE-67498)**
   -- The JIRA user story and associated comments documenting the requirement
   for physical connection points on AIFDT assets, the current manual workflow,
   and links to existing implementation references.

3. **[Connection Points Workflow Doc](https://docs.google.com/document/d/1EiWmKriSCvmX8GvwQk_w8ZAmmyKnOjWyXicnIr3y6sw)**
   (Perschall, 2025) -- Detailed workflow documentation for manually creating
   connection point geometry prims in AI Factory SimReady assets, including
   naming conventions, geometry considerations, and layer organization.

4. **[AIF Samples Repository](https://gitlab-master.nvidia.com/omniverse/samples/kit/scripts/aif-samples)**
   -- Reference implementation and documentation for the CAD-to-USD workflow
   including connection point definitions.

5. **[SimReady Connection Points Examples](https://docs.google.com/document/d/1jZJGJpjW8kLDT6PubSTAD4WYMbWgPQM46_pkXdQli0I)**
   (Perschall, 2025) -- Detailed analysis of connection point parameters
   across power, coolant, and network domains, including cross-cutting
   mechanical properties, example workflows (fiber BOM, electrical simulation,
   thermal simulation, cable/hose checking), and GB300 port naming and
   hierarchy considerations.

6. **Direction from the author** -- Beau Perschall provided direction to:
   separate connection points into thermal cooling, electrical, airflow,
   network, and mechanical domains; generalize across VFI, AIF, and Robotics
   industries (including humanoid robotics); frame the proposal around any
   industrial equipment with connection interfaces; emphasize that connection
   points include tooling attachment points and gripper mounts (not just
   equipment-to-infrastructure connections); include airflow vent locations
   as a distinct domain for CFD simulation; position CAD design software
   and PLM as the upstream sources of truth for connection point data,
   enabling parametric updates when designs are revised; and recommend
   Xform prims (not geometry prims) as the canonical connection point
   representation, with spatial extents expressed as typed metadata
   properties rather than inferred from geometry dimensions, reinforcing
   the separation of concerns between position/orientation, engineering
   dimensions, and visual representation.

### Prompts provided to the AI

1. *Read the Separation of Concerns for Identifiers in USD proposal from
   OpenUSD-proposals PR #105 for style reference.*

2. *Read the JIRA task OMPE-67498 for context on the Physical Connection
   Points requirement.*

3. *Draft a new separation of concerns proposal for connection points,
   separating thermal cooling, electrical, and mechanical connections, and
   generalizing across VFI, AIF, and Robotics industries. Any industrial
   equipment with a connection interface/connection point can benefit.*

4. *Connection points also include tooling attachment points (CNC spindle
   to tool bit, robotic wrist flange to gripper), not just equipment-to-
   infrastructure connections.*

5. *Include airflow and ventilation as a distinct domain -- the physical
   locations of air vents on equipment contribute to CFD simulations.*

6. *Include humanoid robotics alongside robot arms and workcells. Ultimately
   much of this data should spawn from CAD design software (where it makes
   sense) and PLM for information not conveyed in the design file, enabling
   parametric updates when revising designs.*

7. *Incorporate the SimReady Connection Points Examples document for cross-
   industry connection point parameters.*

8. *Do connection points need geometry at all, or can Xforms with
   directionality and keying orientation serve as the representation?
   What spatial extents does the Xform position lack?*

9. *Keep it simple -- no child mesh prims. Express spatial extents as
   metadata properties, reinforcing the separation of concerns.*

---

## Changelog

### v0.2 (May 8, 2026)

- **SemanticsLabelsAPI as Approach 3.** Added USD's built-in
  `SemanticsLabelsAPI` as a complementary discovery mechanism between
  namespaced properties (Approach 2) and formal applied API schemas
  (renumbered to Approach 4). Includes USDA code example, balanced pros/cons
  analysis, and the key insight that semantic labels can be layered
  retroactively via composition without changing base assets.
- **Environmental interface acknowledgment.** Airflow domain now explicitly
  notes that these interfaces are finite surfaces rather than discrete points
  -- the term "connection point" is used broadly for any physical interface
  where compatibility or simulation boundary conditions matter.
- **Industrial vs. datacenter-specific framing.** Datacenter cooling section
  distinguishes piping connections (general industrial, found across process
  plants and manufacturing) from airflow interfaces (datacenter/HVAC-specific).
- **Manufacturing and robotics overlap.** Added cross-reference note between
  VFI and Robotics sections acknowledging shared connection point patterns
  (CNC spindles as robotic tool interfaces, robot flanges as mechanical
  mounts).
- **Network logical/physical identity.** Network domain rewritten to
  explicitly frame the dual identity of ports (physical connector location
  vs. OS-level/orchestration address) and defer logical mapping to a future
  revision.
- **PLM optionality.** Data sourcing language softened throughout -- PLM is
  now one of several valid provenance paths alongside direct CAD annotation
  and manual editing. Principle 6 renamed to "Additive decoration from any
  valid source." Vocabulary is explicitly provenance-agnostic.
- **Fluid type simplification.** Replaced separate `fluidType` +
  `glycolConcentration` properties with a single descriptive token encoding
  concentration (e.g., `glycol_water_30`). Applied across vocabulary spec
  and all exemplars.
- **Use-case justification.** Added "Use cases driving property selection"
  section to the vocabulary spec tracing all properties to three use cases:
  thermal/performance simulation, robotic datacenter assembly, assembly
  calculations/BOM.
- **Mating depth naming.** Added community naming discussion note --
  "insertion depth" suggested as alternative term, feedback welcome.
- **"Currently defined as" wording.** Updated three-concerns table row label
  for clarity (was "Current location").
- **Discoverability expanded.** Added semantic label discovery as a third
  complementary mechanism alongside scope prim and property namespace
  discovery.
- **Acknowledgments section.** Added contributor credits for stakeholders
  whose feedback materially shaped the proposal.
- **Next steps removed.** Moved to internal roadmap; not required for
  external stakeholder review.

### v0.1 (April 2026)

- Initial draft circulated for internal and partner review.
