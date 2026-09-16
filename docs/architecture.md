# Reaction Modules - Architecture

**Status: Concept / Research.** Nothing described here is implemented, deployed, or
scheduled. Field names, structures and constraints are drafts and are expected to
change.

## Purpose

A Rearctor reaction is deliberately rigid. Supply is fixed at 1,000,000,000 tokens,
the trading-fee configuration is immutable after deployment, the ignition threshold is
a constant 5,042 USDC, and migration is unconditional once that threshold is crossed.
That rigidity is the product: a participant can reason about the reaction's mechanics
without trusting the creator.

Reaction Modules ask a narrower question: **within the 70% reaction allocation, can a
creator declare in advance how that share will be used, in a way that is as inspectable
as the protocol parameters themselves?**

A module is a declared, bounded behaviour over the reaction allocation. It is not a
general extension point, and it is not a hook into protocol mechanics.

## Relationship to a reaction

```
Reaction (immutable core)
├── initial supply           1,000,000,000        fixed
├── trading fee config       1%-10%               immutable after deployment
├── ignition threshold       5,042 USDC           constant
├── migration                atomic, full-range   unconditional at threshold
└── fee split                30 / 70              protocol / reaction allocation
                                     │
                                     └── reaction allocation (70%)
                                             │
                                             └── module set  ← the only surface
                                                                modules touch
```

Modules sit strictly downstream of the split. They redirect and schedule the reaction
allocation. They do not participate in price discovery, graduation, or migration.

## Module declaration before launch

The design premise is that a module set is **declared at Spark and fixed there**, not
attached to a live reaction.

Rationale: a participant buying into a reaction during Charging is exposed to how the
reaction allocation will be used. If modules could be added afterward, that exposure
would be unbounded at the time of purchase. Declaring the set up front makes it part of
what a buyer can inspect before trading.

This implies the manifest must be:

- **resolvable before the first trade** - available at or before deployment;
- **content-addressed or on-chain** - so the declared set cannot be swapped later;
- **complete** - a module absent from the declared set can never be added.

> **Open question.** Where does the manifest live? Candidates: fully on-chain
> (expensive, maximally verifiable), a content hash on-chain with the document off-chain
> (cheap, verifiable, requires the document to remain retrievable), or event-log only
> (cheapest, weakest). Not decided.

## Immutable vs executable configuration

Not every module parameter needs to be frozen. The draft separates two classes.

| Class | Definition | Mutability | Examples |
| :--- | :--- | :--- | :--- |
| **Immutable configuration** | Determines *what* a module may ever do and *how much* it may ever touch | Fixed at deployment | module type, `allocation_bps`, activation stage, hard budget ceiling |
| **Executable configuration** | Determines *when* an already-permitted action fires | May vary within immutable bounds | trigger thresholds, cooldowns, pacing |

The boundary exists because a purely immutable module cannot respond to conditions, and
a fully mutable module offers a buyer no guarantee. Splitting the two lets the
*envelope* be fixed while leaving *timing* operable.

> **Under research.** Whether executable configuration should be changeable at all, or
> only selectable from a set enumerated at launch. The second is more restrictive and
> easier to reason about; it may be too rigid to be useful.

## Boundaries with core protocol parameters

A module **must not** be able to:

- alter initial supply or mint/burn outside declared module semantics;
- modify the trading-fee configuration, in either direction;
- change the ignition threshold or influence when ignition occurs;
- prevent, delay, or condition migration;
- withdraw principal from the migrated Uniswap V4 position;
- alter the 30/70 split;
- tax wallet-to-wallet transfers.

These are boundaries by construction, not by policy. A design in which a module *could*
do any of the above and is merely expected not to should be rejected. See
[safety framing in allocation constraints](../research/allocation-constraints.md).

## Interaction with the reaction allocation

The reaction allocation accrues continuously - from curve trading during Charging, and
from the migrated pool during Expansion. Modules draw from that same stream.

Two structural facts shape the design:

1. **The allocation is a flow, not a balance.** Modules are competing for future
   inflow whose size is unknown at declaration time.
2. **The flow changes character at ignition.** Pre-ignition it derives from curve
   trades; post-ignition from pool fees. Volume, denomination mix, and cadence all
   shift at that boundary.

A module that expresses its claim as a fixed absolute amount may never be satisfiable.
The draft therefore uses **basis points of the allocation** (`allocation_bps`) as the
primary unit, with absolute values treated as optional ceilings rather than targets.

> **Open question.** Whether `allocation_bps` across a module set must sum to exactly
> 10000, may sum to less (leaving an unallocated remainder to the creator), or may be
> expressed as priority-ordered claims. See
> [allocation-constraints.md](../research/allocation-constraints.md).

## Non-goals

- Arbitrary user-supplied code. The design assumes a closed set of module types with
  known semantics, not a virtual machine.
- Post-launch governance. There is no mechanism here for changing a reaction's
  economics after deployment.
- Cross-reaction behaviour. A module acts on one reaction.

## Related documents

- [Module lifecycle](module-lifecycle.md)
- [Draft manifest schema](../specs/module-manifest.schema.json)
- [RFC 0001 - Module interface](../rfcs/0001-module-interface.md)
- [Allocation constraints](../research/allocation-constraints.md)
