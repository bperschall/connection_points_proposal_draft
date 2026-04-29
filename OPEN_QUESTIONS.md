# Connection Points: Open Questions & Strawman Recommendations

Companion document to
[AIF_Connection_Points_Proposal_Concise.md](AIF_Connection_Points_Proposal_Concise.md)
for use in stakeholder alignment discussions.

Each question includes context, a concrete USD example, and a strawman
recommendation aligned with the proposal's direction.

---

## Q1: Base vocabulary scope

**What belongs in the base `connectionPoint:` namespace vs. domain-specific
prefixes?**

### Context

Some properties are universal (every connection has a type, a direction, a
size) and some are domain-specific (only thermal connections have flow rates,
only electrical connections have voltage). The line between base and
domain-specific affects how much CAD tools can populate without domain
expertise, and how cleanly simulation tools can filter to the properties they
care about.

### Example

```usda
def Xform "fws_supply_main" {
    # --- Base namespace (shared across ALL domains) ---
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
    token connectionPoint:system = "FWS"
    float connectionPoint:portDiameter = 0.1016
    float connectionPoint:matingDepth = 0.05
    float connectionPoint:serviceClearance = 0.3
    token connectionPoint:disconnectType = "flanged"

    # --- Domain-specific namespace (thermal only) ---
    float connectionPoint:thermal:designFlowRate = 6.3
    float connectionPoint:thermal:designTemperature = 7.2
    float connectionPoint:thermal:operatingPressure = 1034000
    token connectionPoint:thermal:fluidType = "water"
    token connectionPoint:thermal:flangeRating = "ANSI_150"
}
```

### Strawman recommendation

The base `connectionPoint:` namespace includes: `type`, `direction`, `system`,
`portDiameter`, `ventArea`, `matingDepth`, `serviceClearance`,
`disconnectType`, and the cross-cutting mechanical properties (insertion force,
keying, temperature rating, material). Everything else goes under a domain
prefix (`thermal:`, `electrical:`, `network:`, `mechanical:`). This keeps the
base small enough that CAD tools can populate it without domain expertise, while
the domain prefixes give simulation tools exactly the properties they need.

### Discussion question

> "Does this base set feel right? Is anything in the base that should be
> domain-specific, or anything in a domain prefix that every connection
> actually needs?"

---

## Q2: Multi-domain connection points [UPDATE - April 28, 2026]

**Should a single prim carry properties from multiple domain prefixes, or
should multi-domain interfaces be modeled as co-located single-domain prims?**

### Context

Some real-world connectors do multiple things at once. A robot tool changer
isn't just a mechanical mount -- it simultaneously passes compressed air
(pneumatic), electrical signals, and sometimes data. The modeling choice affects
how validation tools check compatibility and how CAD tools represent the
interface.

### Example

**Option A: Single prim, multiple domain prefixes**

```usda
def Xform "wrist_tool_changer" {
    token connectionPoint:type = "multi_domain"
    token connectionPoint:direction = "mate"
    token[] connectionPoint:domains = ["mechanical", "pneumatic", "electrical"]

    # Mechanical properties
    token connectionPoint:mechanical:interfaceStandard = "ATI_QC21"
    token connectionPoint:mechanical:boltPattern = "ISO_9409-1-50-4-M6"
    float connectionPoint:mechanical:payloadRating = 10.0

    # Pneumatic properties
    int connectionPoint:mechanical:pneumaticPorts = 6
    float connectionPoint:mechanical:pneumaticPressure = 620000

    # Electrical pass-through
    int connectionPoint:electrical:pinCount = 12
    token connectionPoint:electrical:signalType = "24VDC_digital"
}
```

**Option B: Co-located single-domain prims**

```usda
def Xform "wrist_tool_changer" {
    def Xform "mechanical" {
        token connectionPoint:type = "mechanical"
        token connectionPoint:mechanical:interfaceStandard = "ATI_QC21"
        float connectionPoint:mechanical:payloadRating = 10.0
    }
    def Xform "pneumatic" {
        token connectionPoint:type = "thermal"
        int connectionPoint:thermal:portCount = 6
    }
    def Xform "electrical" {
        token connectionPoint:type = "electrical"
        int connectionPoint:electrical:pinCount = 12
    }
}
```

### Decision: Option B (co-located single-domain prims)

Each physical domain (thermal, electrical, airflow, network) gets its own prim
under the `ConnectionPoints` scope, rather than combining multiple domain
prefixes on a single prim.

### Rationale

1. **Parallel authoring across disciplines.** Different engineering disciplines
   own different domains and update them independently. A thermal engineer
   should not need to touch the electrical prim to update cooling metadata, and
   vice versa. Option B enables parallel authoring without merge conflicts or
   coordination overhead.

2. **Clean alignment with the layer authoring model.** Option B maps directly
   to the confirmed layer authoring approach (Q4): each domain's connection
   interface data can live in its own USD file, composed via sublayers. This is
   consistent with Shaad Boochoon's payload variant switching approach.

3. **Reflects real-world engineering workflows.** Aaron Gilroy (who manages BOM
   creation for our equipment internally) made the case that real-world
   engineering workflows are organized by discipline, not by connection point.
   The data model should reflect how people actually work.

### Where Option A still applies

Option A (single prim with multiple domain prefixes) remains valid for simple
single-domain assets where only one domain is present (e.g., a purely
electrical junction box). It is not deprecated, just not the default starting
point for multi-domain interfaces.

### Validation via PoC

Steve Ghee at PTC is building an RJ45 connector PoC using Option B in
Creo/Windchill. Target: May 11 NVIDIA HQ visit. This will be the first
non-theoretical asset testing the co-located prim layout at scale (1 port to 4
on a tray, tray into rack, rack instanced).

### Meeting source

Decision reached April 28, 2026 in the Connection Points Proposal Part 2
session (Beau Perschall, Jason Batchkoff, Shaad Boochoon, Christian Akesson,
Aaron Gilroy). Dassault 3DEXPERIENCE team (morning call same day) also
gravitates toward this approach, aligning it with their "skeleton representation"
concept.

---

## Q3: Connection relationships

**Should the vocabulary support explicit relationships between mated connection
points, or should mating be expressed only through spatial proximity and
external tooling?**

### Context

When two pieces of equipment are connected (e.g., a CDU's supply pipe to a rack
manifold's supply inlet), that "connected to" relationship could be written
explicitly in the USD file or inferred by tools based on physical proximity and
compatible properties.

### Example

**Option A: Explicit USD relationship**

```usda
def Xform "CDU_01" {
    def Xform "ConnectionPoints" {
        def Xform "fws_supply_out" {
            token connectionPoint:type = "thermal"
            token connectionPoint:direction = "supply"
            rel connectionPoint:matedTo = </Facility/RackRow_01/Rack_01/ConnectionPoints/fws_supply_in>
        }
    }
}
```

**Option B: No relationship -- mating determined by spatial proximity and
external tooling**

```
# The CDU and rack connection points are positioned close together
# in the facility layout. A routing/validation tool discovers the
# pairing by proximity + compatible properties. No relationship
# authored in USD.
```

### Strawman recommendation

Defer explicit relationships to a later version. Start with Option B. Explicit
relationships are fragile during layout iteration -- every time you move a rack
or swap a CDU, relationship targets must be updated. Layout and routing tools
already work spatially. Adding relationships creates a maintenance burden the
current workflow doesn't need. However, the vocabulary should be designed so
relationships can be added later without breaking anything -- connection point
prim paths should be stable and predictable.

### Discussion question

> "Is there a consumption workflow today that absolutely requires an explicit
> 'connected to' relationship in the USD file, or can your tools discover
> connections spatially?"

---

## Q4: Layer authoring model

**The current `_ConnectionPoints.usd` layer pattern provides clean separation.
Should this be formalized?**

### Context

Today connection points live in a separate file (e.g.,
`Vertiv_CDU_ConnectionPoints.usd`), layered on top of the geometry file. This
means connection point metadata can be updated without touching geometry, and
CAD export pipelines can regenerate connection points independently.

### Example

```
# Current pattern (separate layer file):
Vertiv_CDU.usd                    # Main asset geometry
Vertiv_CDU_ConnectionPoints.usd   # Connection points in a separate file
Vertiv_CDU.usda                   # Root file that sublayers both together
```

If formalized, the vocabulary spec would say: "Connection point Xforms MUST be
authored in a dedicated `<AssetName>_ConnectionPoints.usd` layer." If a
recommendation, tools SHOULD follow the pattern but aren't required to.

### Strawman recommendation

Formalize it as a SHOULD (strong recommendation), not a MUST (hard
requirement). The separate layer pattern is good practice -- it supports
independent update of connection metadata and clean CAD pipeline regeneration.
But there will be edge cases (small simple assets, rapid prototyping) where
forcing a separate file is overhead without benefit. SimReady Foundation
validation can check for it and warn if missing, without failing the asset.

### Discussion question

> "Does anyone have a workflow where connection points MUST live in the same
> file as geometry? If not, we'll formalize the separate layer as the
> recommended pattern."

---

## Q5: `guide` purpose

**Connection point Xforms carry no geometry, but `guide` purpose still signals
metadata scaffolding. Should the vocabulary mandate it?**

### Context

In USD, `guide` purpose tells renderers "this is helper scaffolding, don't
render it." Today's connection point geometry prims use `guide` so they're
invisible in renders. The proposal replaces geometry prims with Xforms, which
have no visible geometry anyway. But `guide` has a secondary benefit: it tells
traversal tools "this prim is scaffolding, skip it if you're only looking for
real content."

### Example

```usda
def Xform "fws_supply_main" (
    purpose = "guide"
) {
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
    float connectionPoint:portDiameter = 0.1016
}
```

### Strawman recommendation

Mandate `guide` purpose. The cost is nearly zero (one extra line per prim) and
the benefit is real: it provides a second discovery/filtering mechanism beyond
the `connectionPoint:` namespace. A rendering tool that respects purpose will
skip these prims entirely. A scene traversal tool looking for "just the real
geometry" can filter on purpose. It also maintains continuity with the existing
workflow, making migration smoother.

### Discussion question

> "Any objection to requiring `guide` purpose on all connection point Xforms?
> It costs nothing and helps with filtering."

---

## Q6: Units and standards

**Should the vocabulary enforce SI units, support unit annotations, or defer to
USD's existing `UsdGeomLinearUnits`?**

### Context

When a property says `portDiameter = 0.1016`, the unit must be unambiguous. If
one author writes `4` (meaning 4 inches) and another writes `0.1016` (meaning
meters), you get a compatibility mismatch for the same physical dimension.

### Example

```usda
# Option A: Enforce SI units (meters, Pascals, kg, etc.)
float connectionPoint:portDiameter = 0.1016        # Always meters

# Option B: Unit annotations per property
float connectionPoint:portDiameter = 4.0
token connectionPoint:portDiameter:unit = "inch"    # Explicit unit tag

# Option C: Defer to USD's existing metersPerUnit
# (USD stages declare metersPerUnit at the stage level,
#  but this only covers linear dimensions -- not pressure, flow, temperature)
```

### Strawman recommendation

Enforce SI units in the vocabulary specification:

| Property | SI unit |
|---|---|
| `portDiameter`, `matingDepth`, `serviceClearance` | meters (m) |
| `ventArea` | square meters (m²) |
| `operatingPressure` | Pascals (Pa) |
| `designFlowRate` | cubic meters per second (m³/s) |
| `designTemperature` | degrees Celsius (°C) |
| `currentRating` | Amperes (A) |
| `voltageRating` | Volts (V) |
| `payloadRating` | kilograms (kg) |

USD's own convention for geometry is meters-based (`metersPerUnit`). Unit
annotations (Option B) sound flexible but create a combinatorial problem --
every consuming tool must check for a unit tag and convert. SI-only eliminates
this. Display tools can convert to imperial for human presentation, but stored
values are always SI.

### Discussion question

> "Are we comfortable with SI-only for stored values? The main risk is that
> teams working in imperial units need to convert on write -- but the benefit
> is that every consumer gets consistent units without conversion logic."

---

## Q7: Backward compatibility

**Existing GB300 and Vertiv assets use naming conventions. The solution should
define a migration path.**

### Context

GB300 racks and Vertiv cooling equipment already have connection points using
the current approach (geometry prims, naming conventions like
`vertiv_fws_supply_piping_connection_main`). These existing assets must
continue to function during and after the transition.

### Example

```usda
# BEFORE (current convention):
def Mesh "vertiv_fws_supply_piping_connection_main" (
    purpose = "guide"
) {
    # Position encoded in mesh transform
    # Type encoded in name: "fws" = facility water system
    # Direction encoded in name: "supply"
    # Port diameter inferred from mesh dimensions (disk radius)
}

# AFTER (proposed approach):
def Xform "fws_supply_main" (
    purpose = "guide"
) {
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
    token connectionPoint:system = "FWS"
    float connectionPoint:portDiameter = 0.1016
    float connectionPoint:thermal:designFlowRate = 6.3
}
```

### Strawman recommendation

Define a three-phase migration path:

1. **Phase 1 (now):** New assets are authored in the new format. Existing
   assets continue to work as-is. SimReady Foundation v0.1.0 requirements
   (CP.001-CP.006) validate the old format.
2. **Phase 2 (v0.2.0):** Migration tooling converts existing assets. The tool
   reads the old naming convention, parses out type/direction/system, creates
   the Xform with typed properties, and removes the mesh geometry. Both old and
   new formats are accepted during transition.
3. **Phase 3 (v0.3.0+):** Old format is deprecated. Validation warns on
   old-format connection points. Eventually, old format fails validation.

Existing GB300 and Vertiv assets are not broken at any phase. There is no
"flag day" where everything must convert simultaneously.

### Discussion question

> "Is a phased migration where both formats coexist temporarily acceptable? Or
> does anyone need a hard cutover?"

---

## Q8: CAD/PLM integration boundaries

**Which properties should CAD exporters populate vs. PLM? How should the
vocabulary accommodate partial population?**

### Context

A connection point's properties come from different sources. CAD models provide
the physical and geometric aspects (where, how big, what type of connector).
PLM systems provide the operational and engineering aspects (rated flow,
approved fluids, compliance data). The vocabulary needs to work when only part
of the data is available.

### Example

```usda
# Step 1: CAD exporter creates the connection point
# (knows geometry, doesn't know operating parameters)
def Xform "fws_supply_main" {
    token connectionPoint:type = "thermal"
    token connectionPoint:direction = "supply"
    float connectionPoint:portDiameter = 0.1016
    float connectionPoint:matingDepth = 0.05
    token connectionPoint:disconnectType = "flanged"
}

# Step 2: PLM integration adds operating parameters in a stronger layer
over "fws_supply_main" {
    token connectionPoint:system = "FWS"
    float connectionPoint:thermal:designFlowRate = 6.3
    float connectionPoint:thermal:operatingPressure = 1034000
    token connectionPoint:thermal:fluidType = "water"
    token connectionPoint:thermal:flangeRating = "ANSI_150"
}
```

After Step 1, the connection point is partially populated but useful -- layout
tools know where it is, how big it is, and what type it is. After Step 2, it is
fully populated and simulation tools can consume it.

### Strawman recommendation

The vocabulary specification should define three property tiers:

| Tier | Source | Properties |
|---|---|---|
| **CAD-authored** | Geometry export | position/orientation (Xform transform), `type`, `direction`, `portDiameter`/`ventArea`, `matingDepth`, `disconnectType`, `interfaceStandard` |
| **PLM-authored** | Engineering data | `system`, `designFlowRate`, `operatingPressure`, `fluidType`, `flangeRating`, `allowedTransceivers`, `compatibleAccessories` |
| **Either/both** | Varies | `serviceClearance` (sometimes in CAD as a design envelope, sometimes from facilities engineering) |

A connection point is valid at any population level. Validation rules should be
tier-aware -- a CAD-only connection point passes "CAD completeness" checks but
warns on missing PLM properties. A fully populated one passes all checks.

### Discussion question

> "Does this CAD vs. PLM property split match how your data actually flows? Are
> there properties in the wrong bucket?"

---

## Q9: Port naming and hierarchy

**How should physical port identity relate to logical network identity?**

### Context

On a GPU rack, a physical port has an identity like `OSFP1` (the first OSFP
connector on the tray). In the network design, that same port might be called
`C1` (compute network port 1). These are different naming systems serving
different audiences -- the physical technician plugging in a cable vs. the
network architect designing the topology.

### Example

```usda
def Xform "ConnectionPoints" {
    def Xform "OSFP1" {
        token connectionPoint:type = "network"
        token connectionPoint:portType = "OSFP"
        token connectionPoint:network:medium = "fiber"
        token connectionPoint:network:category = "high_speed_data"
        token connectionPoint:network:fabricRole = "compute"

        # Physical identity (what's printed on the hardware)
        token connectionPoint:physicalLabel = "OSFP1"

        # Logical identity (from the network design / SysML model)
        token connectionPoint:network:logicalName = "C1"
        token connectionPoint:network:switchPort = "Leaf01/Eth1/1"
    }

    def Xform "OSFP2" {
        token connectionPoint:type = "network"
        token connectionPoint:portType = "OSFP"
        token connectionPoint:network:medium = "fiber"
        token connectionPoint:network:fabricRole = "compute"
        token connectionPoint:physicalLabel = "OSFP2"
        token connectionPoint:network:logicalName = "C2"
        token connectionPoint:network:switchPort = "Leaf01/Eth1/2"
    }

    def Xform "ETH1" {
        token connectionPoint:type = "network"
        token connectionPoint:portType = "RJ45"
        token connectionPoint:network:medium = "copper"
        token connectionPoint:network:category = "management"
        token connectionPoint:network:fabricRole = "out_of_band"
        token connectionPoint:physicalLabel = "ETH1"
        token connectionPoint:network:logicalName = "BMC_MGMT"
    }
}
```

### Strawman recommendation

The prim name is the physical port identity (`OSFP1`, `ETH1`) because it comes
from the hardware/CAD model and is stable across deployments. Logical network
identity lives in properties because it varies by deployment -- the same
physical rack may be cabled differently in different datacenters. Specifically:

| Property | Source | Stability |
|---|---|---|
| Prim name | Hardware / CAD model | Stable across all installations of the same hardware |
| `connectionPoint:physicalLabel` | Hardware silk-screen | Stable (matches prim name, human-readable) |
| `connectionPoint:network:logicalName` | Network design tools | Varies per deployment |
| `connectionPoint:network:switchPort` | Network design / SysML | Varies per deployment (site-specific layer) |

This keeps the physical and logical worlds cleanly separated. Logical names are
site-specific overrides in a stronger composition layer, consistent with the
CAD/PLM layering model described in Q8.

### Discussion question

> "Does treating the prim name as physical identity and logical names as
> deployment-specific properties work for the networking team? Is there a case
> where the physical port identity isn't stable across installations?"
