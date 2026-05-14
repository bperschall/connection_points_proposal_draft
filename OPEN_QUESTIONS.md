# Connection Points: Open Questions & Strawman Recommendations

Companion document to
[AIF_Connection_Points_Proposal_Concise.md](AIF_Connection_Points_Proposal_Concise.md)
for use in stakeholder alignment discussions.

Each question includes context, a concrete USD example, and a strawman
recommendation aligned with the proposal's direction.

---

## Q1: Base vocabulary scope

**What belongs in the base `simready:connectionPoint:` namespace vs. domain-specific
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
    token simready:connectionPoint:type = "thermal"
    token simready:connectionPoint:direction = "supply"
    token simready:connectionPoint:system = "FWS"
    float simready:connectionPoint:portDiameter = 0.1016
    float simready:connectionPoint:matingDepth = 0.05
    float simready:connectionPoint:serviceClearance = 0.3
    token simready:connectionPoint:disconnectType = "flanged"

    # --- Domain-specific namespace (thermal only) ---
    float simready:connectionPoint:thermal:designFlowRate = 6.3
    float simready:connectionPoint:thermal:designTemperature = 7.2
    float simready:connectionPoint:thermal:operatingPressure = 1034000
    token simready:connectionPoint:thermal:fluidType = "water"
    token simready:connectionPoint:thermal:flangeRating = "ANSI_150"
}
```

### Strawman recommendation

The base `simready:connectionPoint:` namespace includes: `type`, `direction`, `system`,
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

## Q2: Multi-domain connection points [UPDATE - May 7, 2026]

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
    token simready:connectionPoint:type = "multi_domain"
    token simready:connectionPoint:direction = "mate"
    token[] simready:connectionPoint:domains = ["mechanical", "pneumatic", "electrical"]

    # Mechanical properties
    token simready:connectionPoint:mechanical:interfaceStandard = "ATI_QC21"
    token simready:connectionPoint:mechanical:boltPattern = "ISO_9409-1-50-4-M6"
    float simready:connectionPoint:mechanical:payloadRating = 10.0

    # Pneumatic properties
    int simready:connectionPoint:mechanical:pneumaticPorts = 6
    float simready:connectionPoint:mechanical:pneumaticPressure = 620000

    # Electrical pass-through
    int simready:connectionPoint:electrical:pinCount = 12
    token simready:connectionPoint:electrical:signalType = "24VDC_digital"
}
```

**Option B: Co-located single-domain prims**

```usda
def Xform "wrist_tool_changer" {
    def Xform "mechanical" {
        token simready:connectionPoint:type = "mechanical"
        token simready:connectionPoint:mechanical:interfaceStandard = "ATI_QC21"
        float simready:connectionPoint:mechanical:payloadRating = 10.0
    }
    def Xform "pneumatic" {
        token simready:connectionPoint:type = "thermal"
        int simready:connectionPoint:thermal:portCount = 6
    }
    def Xform "electrical" {
        token simready:connectionPoint:type = "electrical"
        int simready:connectionPoint:electrical:pinCount = 12
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

### Option C: Semantic annotations on existing structure (DEFERRED)

Steve Ghee (PTC) proposed a third approach that goes beyond the Option A vs.
Option B framing. Rather than organizing connection points into a dedicated
top-level scope (whether as a single multi-domain prim or co-located
single-domain prims), Option C treats connection points as semantic annotations
on the parts themselves, discoverable via USDSemantics. The connection points
inherit the actual physical structure of the model rather than imposing a new
organizational hierarchy.

**Example (conceptual):**

```usda
# Instead of a top-level ConnectionPoints scope, annotations live
# on the physical structure itself:
def Xform "robot_arm" {
    def Xform "wrist_mount" {
        # Mechanical connection semantics annotated directly
        token simready:connectionPoint:type = "mechanical"
        token simready:connectionPoint:mechanical:interfaceStandard = "ATI_QC21"
    }
    def Xform "pneumatic_hose_fitting" {
        # Pneumatic connection semantics annotated directly
        token simready:connectionPoint:type = "pneumatic"
        int simready:connectionPoint:pneumatic:portCount = 6
    }
    def Xform "electrical_harness_plug" {
        # Electrical connection semantics annotated directly
        token simready:connectionPoint:type = "electrical"
        int simready:connectionPoint:electrical:pinCount = 12
    }
}
```

**Steve's rationale:**

1. **Different domains are owned by different users.** As Aaron Gilroy also pointed out, multiple domains are managed by different disciplines. Annotating existing structure respects that ownership naturally.

2. **Physical connections are physically distinct points.** The mechanical joint, pneumatic hose, and electrical cable on a robot arm are literally different locations on the part. The data model should reflect that physical reality rather than grouping them into an artificial scope.

3. **Flexibility and scalability.** Annotating the structure that already exists is more flexible than creating new organizational hierarchy. It avoids requiring ISVs to adopt a new schema before content can flow into their tools.

**Why Option C is deferred for v0.2.0:**

1. **Adoption barrier.** A schema-driven semantic annotation approach requires ISVs consuming connection point data to adopt that schema before any content flows into their tools. For this first iteration, the priority is keeping the barrier to entry as low as possible so connection point data can move between tools quickly.

2. **Taxonomy prerequisite.** Both Beau Perschall and Steve Ghee agree that semantic annotations without a constraining taxonomy risk becoming unmanageable noise that is impossible to validate. The taxonomy must be in place first to make annotations meaningful and verifiable. That work is underway but not yet complete.

3. **Discoverability tradeoff.** Annotations distributed across the model hierarchy require consumers to traverse the full structure to find all connection points. The co-located prim approach (Option B) provides immediate discoverability from a known location. A future version could combine both: a lightweight manifest or relationship referencing semantically annotated prims deeper in the hierarchy, giving both discoverability and structural fidelity.

**Path forward:** Once the connection point taxonomy is established and validated through the Option B implementation, Option C becomes a strong candidate for a subsequent version. The taxonomy work will provide the vocabulary needed to make semantic annotations meaningful, and real-world experience with Option B will clarify which discoverability patterns consumers actually need.

### Meeting source

Decision reached April 28, 2026 in the Connection Points Proposal Part 2
session (Beau Perschall, Jason Batchkoff, Shaad Boochoon, Christian Akesson,
Aaron Gilroy). Dassault 3DEXPERIENCE team (morning call same day) also
gravitates toward this approach, aligning it with their "skeleton representation"
concept. Option C added May 7, 2026 based on feedback from Steve Ghee (PTC)
referencing his response to the initial proposal draft.

---

## Q3: Connection relationships [DEFERRED - April 28, 2026]

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
            token simready:connectionPoint:type = "thermal"
            token simready:connectionPoint:direction = "supply"
            rel simready:connectionPoint:matedTo = </Facility/RackRow_01/Rack_01/ConnectionPoints/fws_supply_in>
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

### Decision: Deferred

Connection relationships are out of scope for v0.2.0.

### Rationale

1. **Orchestration-layer problem, not a single-asset concern.** The group
   reached consensus that explicit connection relationships (e.g.,
   `rel simready:connectionPoint:matedTo`) belong to the facility layout or routing
   tool, not the single-asset specification. A single asset should describe its
   own interfaces; how those interfaces connect to other assets is a scene
   composition problem.

2. **Strawman recommendation validated.** No one in the room identified a
   consumption workflow that requires explicit "connected to" relationships in
   the USD file today. Relationships are fragile during layout iteration, and
   spatial proximity plus compatible properties is sufficient for current tools.

3. **Door remains open.** Deferring does not close the door. The vocabulary is
   being designed so that relationship properties can be added in a future
   version without breaking existing assets or tools.

### Meeting source

Deferred by consensus on April 28, 2026 in the Connection Points Proposal
Part 2 session (Beau Perschall, Jason Batchkoff, Shaad Boochoon, Christian
Akesson, Aaron Gilroy).

---

## Q4: Layer authoring model [CONFIRMED - April 28, 2026]

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

### Decision: Confirmed

Connection interface data is authored in its own dedicated USD file, separate
from geometry. The existing `<AssetName>_ConnectionPoints.usd` pattern is
formalized.

### Rationale

1. **Stakeholder alignment across organizations.** Dassault 3DEXPERIENCE team
   (April 28 morning call) gravitates to this approach, seeing alignment with
   their internal "skeleton representation" concept. Shaad Boochoon's payload
   variant switching provides the implementation model. Christian Akesson's
   existing pipeline already follows this pattern.

2. **Enables parallel authoring.** Separate layer authoring enables independent
   update of connection metadata without touching geometry files, which is
   critical for the parallel authoring model confirmed in Q2 (Option B).

3. **Strength of recommendation: SHOULD, not MUST.** The strength stays as a
   formalized strong recommendation rather than a hard requirement. Edge cases
   (small simple assets, rapid prototyping) where a separate file adds overhead
   without benefit are acknowledged. SimReady Foundation validation checks for
   the separate layer and warns if missing, without failing the asset.

### Meeting source

Confirmed on April 28, 2026 in the Connection Points Proposal Part 2 session
(Beau Perschall, Jason Batchkoff, Shaad Boochoon, Christian Akesson, Aaron
Gilroy) with supporting alignment from the Dassault 3DEXPERIENCE morning call
(Beau Perschall, Max Bickley, Jeremie, Dassault team).

---

## Q5: `guide` purpose [INVESTIGATED - April 30, 2026]

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
def Xform "fws_supply_main"
{
    uniform token purpose = "guide"
    token simready:connectionPoint:type = "thermal"
    token simready:connectionPoint:direction = "supply"
    float simready:connectionPoint:portDiameter = 0.1016
}
```

### Strawman recommendation

Recommend `guide` purpose (SHOULD) on connection point Xforms, but do not
mandate it (MUST). The benefit is real: it provides a second discovery and
filtering mechanism beyond the `simready:connectionPoint:` namespace, and maintains
continuity with the existing workflow. However, `guide` purpose cascades to
descendant prims via USD's computed purpose inheritance (see investigation
below), which means any renderable geometry placed beneath a `guide` connection
point Xform becomes invisible unless it explicitly overrides purpose. This
authoring constraint is acceptable for v0.2.0 (connection point Xforms are
metadata carriers, not geometry containers), but mandating it would require a
complementary validation rule to catch unintended geometry hiding. A SHOULD
recommendation paired with a validator warning is the right balance for this
stage of maturity.

### Discussion question

> "Are there use cases where connection point Xforms would intentionally contain
> renderable child geometry? If not, SHOULD with a validator warning is
> sufficient. If so, we need to define the override pattern before escalating
> to MUST."

### [INVESTIGATION UPDATE - April 30, 2026]

**Finding:** `guide` purpose cascades to descendant prims via USD's computed
purpose inheritance. A child prim with no explicit purpose authored beneath a
`guide` Xform will also be treated as `guide` by renderers -- meaning it
becomes invisible in the viewport.

**Test:** Created a stage with an Xform (`purpose = "guide"`) containing a
child Mesh with no purpose set. The child geometry was invisible in Omniverse,
confirming that computed purpose inheritance applies. The child prim must have
an explicit `purpose = "render"` (or `"default"`) to override the parent's
`guide` and remain visible.

**Implication for the vocabulary:**

Mandating `guide` on connection point Xforms is still viable, but it introduces
an authoring constraint: any renderable geometry placed beneath a connection
point Xform (e.g., a visual indicator, debug geometry, or a nested component)
would be invisible unless it explicitly overrides purpose. This is the concern
Jason Batchkoff raised in the April 28 meeting.

For v0.2.0 connection points, this constraint is acceptable because:

1. Connection point Xforms are metadata carriers, not geometry containers. They
   hold `simready:connectionPoint:` properties and a transform -- no child geometry is
   expected.
2. The `ConnectionPoints` scope already separates connection point prims from
   the asset's renderable hierarchy.
3. If a future use case requires visible child geometry (e.g., debug
   visualization), the author can explicitly set `purpose = "render"` on those
   children.

However, mandating `guide` would require a complementary validation rule: **no
renderable geometry prims should exist as descendants of a `guide`-purpose
connection point Xform without an explicit purpose override.** Without this
validation, an author could unknowingly place geometry under a connection point
and wonder why it disappeared.

**Updated recommendation:** Recommend `guide` purpose (SHOULD) rather than
mandate it (MUST) for v0.2.0. Document the cascade behavior and the authoring
constraint. Add a validator warning (not error) if renderable geometry is found
under a `guide` connection point Xform without an explicit purpose override.
Revisit for MUST in a future version once tooling and authoring patterns are
established.

**Source:** April 28 CP Part 2 meeting (Jason Batchkoff raised cascade concern),
April 30 testing in Omniverse (Beau Perschall), Max Bickley confirmation of
runtime filtering risk.

---

## Q6: Units and standards [CONFIRMED - April 28, 2026]

**Should the vocabulary enforce SI units, support unit annotations, or defer to
USD's existing `UsdGeomLinearUnits`?**

### Context

When a property says `portDiameter = 0.1016`, the unit must be unambiguous. If
one author writes `4` (meaning 4 inches) and another writes `0.1016` (meaning
meters), you get a compatibility mismatch for the same physical dimension.

### Example

```usda
# Option A: Enforce SI units (meters, Pascals, kg, etc.)
float simready:connectionPoint:portDiameter = 0.1016        # Always meters

# Option B: Unit annotations per property
float simready:connectionPoint:portDiameter = 4.0
token simready:connectionPoint:portDiameter:unit = "inch"    # Explicit unit tag

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

### Decision: Confirmed

SI units are enforced at the SimReady neutral level. All stored values use SI
(meters, Pascals, kg, Amperes, Volts, Celsius, etc.) as specified in the
strawman table above.

### Rationale

1. **SimReady defines the neutral interchange format.** Dassault pushed back
   during the April 28 morning call, citing mixed incoming units from their
   data sources. Beau held firm: SimReady defines the neutral interchange
   format, and SI is the standard at that level. Runtime tools and display
   layers are free to convert to imperial or any other unit system for
   presentation, but the stored values in the USD file are always SI.

2. **Consistent with USD conventions, eliminates ambiguity.** This aligns with
   USD's own convention (`metersPerUnit`) and eliminates the combinatorial
   problem of every consuming tool needing to check for unit tags and convert.
   One unit system, zero ambiguity.

3. **Write-time cost accepted.** The group acknowledged that teams working in
   imperial units bear a conversion cost on write, but the benefit to every
   downstream consumer (consistent units, no conversion logic) outweighs that
   cost.

### Meeting source

Confirmed on April 28, 2026 in the Connection Points Proposal Part 2 session
(Beau Perschall, Jason Batchkoff, Shaad Boochoon, Christian Akesson, Aaron
Gilroy). Dassault 3DEXPERIENCE team acknowledged the decision in the morning
call despite their initial pushback.

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
    token simready:connectionPoint:type = "thermal"
    token simready:connectionPoint:direction = "supply"
    token simready:connectionPoint:system = "FWS"
    float simready:connectionPoint:portDiameter = 0.1016
    float simready:connectionPoint:thermal:designFlowRate = 6.3
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
    token simready:connectionPoint:type = "thermal"
    token simready:connectionPoint:direction = "supply"
    float simready:connectionPoint:portDiameter = 0.1016
    float simready:connectionPoint:matingDepth = 0.05
    token simready:connectionPoint:disconnectType = "flanged"
}

# Step 2: PLM integration adds operating parameters in a stronger layer
over "fws_supply_main" {
    token simready:connectionPoint:system = "FWS"
    float simready:connectionPoint:thermal:designFlowRate = 6.3
    float simready:connectionPoint:thermal:operatingPressure = 1034000
    token simready:connectionPoint:thermal:fluidType = "water"
    token simready:connectionPoint:thermal:flangeRating = "ANSI_150"
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
        token simready:connectionPoint:type = "network"
        token simready:connectionPoint:portType = "OSFP"
        token simready:connectionPoint:network:medium = "fiber"
        token simready:connectionPoint:network:category = "high_speed_data"
        token simready:connectionPoint:network:fabricRole = "compute"

        # Physical identity (what's printed on the hardware)
        token simready:connectionPoint:physicalLabel = "OSFP1"

        # Logical identity (from the network design / SysML model)
        token simready:connectionPoint:network:logicalName = "C1"
        token simready:connectionPoint:network:switchPort = "Leaf01/Eth1/1"
    }

    def Xform "OSFP2" {
        token simready:connectionPoint:type = "network"
        token simready:connectionPoint:portType = "OSFP"
        token simready:connectionPoint:network:medium = "fiber"
        token simready:connectionPoint:network:fabricRole = "compute"
        token simready:connectionPoint:physicalLabel = "OSFP2"
        token simready:connectionPoint:network:logicalName = "C2"
        token simready:connectionPoint:network:switchPort = "Leaf01/Eth1/2"
    }

    def Xform "ETH1" {
        token simready:connectionPoint:type = "network"
        token simready:connectionPoint:portType = "RJ45"
        token simready:connectionPoint:network:medium = "copper"
        token simready:connectionPoint:network:category = "management"
        token simready:connectionPoint:network:fabricRole = "out_of_band"
        token simready:connectionPoint:physicalLabel = "ETH1"
        token simready:connectionPoint:network:logicalName = "BMC_MGMT"
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
| `simready:connectionPoint:physicalLabel` | Hardware silk-screen | Stable (matches prim name, human-readable) |
| `simready:connectionPoint:network:logicalName` | Network design tools | Varies per deployment |
| `simready:connectionPoint:network:switchPort` | Network design / SysML | Varies per deployment (site-specific layer) |

This keeps the physical and logical worlds cleanly separated. Logical names are
site-specific overrides in a stronger composition layer, consistent with the
CAD/PLM layering model described in Q8.

### Discussion question

> "Does treating the prim name as physical identity and logical names as
> deployment-specific properties work for the networking team? Is there a case
> where the physical port identity isn't stable across installations?"
