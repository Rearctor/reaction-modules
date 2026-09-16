# Reaction Modules

**Status: Concept / Research**

This repository documents a design direction. Nothing described here is implemented,
deployed, or scheduled.

## Overview

Reaction Modules would allow a creator to attach predefined economic or execution
modules to a reaction before launch. A module is a bounded behaviour - a rule for how
part of the reaction allocation is applied - selected and configured at launch rather
than governed afterward.

Rearctor reactions already carry a fixed initial supply, an immutable trading-fee
configuration, and a 30/70 split between the Rearctor protocol and the reaction
allocation. Modules would operate inside that structure. They would shape how the
reaction allocation is used without altering the parameters that are fixed at
deployment.

## Design principles

- **Pre-launch selection.** Modules are chosen and configured before the reaction is
  deployed, not attached to a live reaction afterward.
- **Immutability after deployment.** Critical module parameters become immutable once
  the reaction is live.
- **No reach into fixed parameters.** A module cannot alter initial supply, the
  trading-fee configuration, the ignition threshold, or the permanence of migrated
  liquidity.
- **Declared and observable.** A reaction's module set is declared at launch, so it can
  be inspected before participating.

## Potential modules

| Module | Concept |
| :--- | :--- |
| **Automated buybacks** | Route a portion of the reaction allocation into buying the reaction's own token. |
| **Liquidity reinvestment** | Direct a portion of the reaction allocation back into liquidity. |
| **Creator vesting** | Release a creator's allocation over time rather than in full at launch. |
| **Holder rewards** | Distribute a portion of the reaction allocation to holders. |
| **Referral allocations** | Assign a share of the reaction allocation to referring addresses. |
| **Treasury allocations** | Reserve a share of the reaction allocation for a reaction-controlled treasury. |

This list is illustrative. It is not a committed module set.

## Lifecycle integration

| Stage | Module behaviour under consideration |
| :--- | :--- |
| **Spark** | The module set is selected and configured. Critical parameters are fixed at deployment. |
| **Charging** | Modules drawing on trading fees begin accruing as the bonding curve trades. |
| **Ignition** | Module behaviour must remain compatible with graduation and liquidity migration executing in a single transaction. |
| **Expansion** | Modules that depend on ongoing fee generation continue to draw from the migrated pool's revenue. |

## Open research questions

- How should module parameters be bounded so that a configuration cannot produce a
  reaction that is structurally unable to reach ignition?
- Which module behaviours are safe to execute inside the ignition transaction, and
  which must be deferred until after migration?
- How should multiple modules drawing on the same reaction allocation be prioritised
  when the allocation is insufficient for all of them?
- Where is the correct boundary between a module and ordinary Control Room
  administration?
- How should a module set be represented so that it is verifiable by a participant
  before they trade?

## Research workspace

Current design work is organized across architecture notes, draft specifications and
RFCs. Everything below is concept and research: no implementation exists.

**Architecture**
- [Architecture](docs/architecture.md) - purpose, declaration before launch, boundaries with core protocol parameters
- [Module lifecycle](docs/module-lifecycle.md) - module behaviour across Spark, Charging, Critical Mass, Ignition and Expansion

**Draft specifications**
- [Module manifest schema](specs/module-manifest.schema.json) - draft JSON Schema for describing a module
- [Example: buyback module](specs/examples/buyback-module.json) - a manifest conforming to the draft schema

**RFCs**
- [RFC 0001 - Module interface](rfcs/0001-module-interface.md) - how a common module interface might work

**Open research**
- [Allocation constraints](research/allocation-constraints.md) - conflicts between modules drawing on the same reaction allocation

## v0 design work

Early design work is tracked through the v0 specification, implementation plan and open
GitHub Issues. These are draft design targets. Nothing is implemented, deployed or
scheduled.

- [v0 design specification](docs/v0/design-spec.md)
- [Implementation plan](docs/v0/implementation-plan.md)
- [Open issues](../../issues)

## Links

[Website](https://rearctor.io) · [Docs](https://rearctor.io/docs) · [GitHub](https://github.com/Rearctor) · [X](https://x.com/JoinRearctor) · [Telegram](https://t.me/rearctor)
