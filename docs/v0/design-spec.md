# Reaction Modules - v0 Design Specification

## Status

**Concept / Research.** This is a draft design target, not a production specification.

Nothing described here is implemented, deployed, scheduled, or audited. No contracts
exist. No APIs exist. There is no commitment that this design will ship, in this form
or at all.

This document builds on, and does not replace:

- [docs/architecture.md](../architecture.md)
- [docs/module-lifecycle.md](../module-lifecycle.md)
- [rfcs/0001-module-interface.md](../../rfcs/0001-module-interface.md)
- [research/allocation-constraints.md](../../research/allocation-constraints.md)
- [specs/module-manifest.schema.json](../../specs/module-manifest.schema.json)

Every numeric figure in this document that is not a Rearctor protocol constant is
illustrative only. Not protocol defaults or committed parameters.

## Scope

v0 is a **specification target**: the smallest coherent description of Reaction Modules
that could be reviewed, argued with, and eventually implemented against.

### Proposed for v0

- A declared module set, fixed at Spark, readable before the reaction's first trade.
- A closed set of module types with known semantics.
- Accrual of the reaction allocation to modules by declared share.
- Settlement-style execution: modules settle against accrued balances rather than being
  invoked during fee events.
- Per-module isolation of state and failure.
- A manifest format sufficient to express the above.

### Under research

- Allocation conflict resolution when several modules draw on one allocation.
  **Unresolved.** See Issue #1.
- Whether any module may execute inside the ignition transaction. **Unresolved.**
  See Issue #2.
- Denomination handling across the ignition boundary.
- Settlement gas economics and caller incentive.
- Behaviour for reactions that never reach critical mass.

### Explicitly out of scope

- Arbitrary user-supplied module code. The design assumes a closed type set, not a
  virtual machine.
- Post-launch governance of reaction economics.
- Cross-reaction module behaviour.
- Any module capability touching initial supply, the trading-fee configuration, the
  ignition threshold, migration, or migrated liquidity principal.

## Core entities

| Entity | Definition | Mutability |
| :--- | :--- | :--- |
| **Reaction** | The immutable core: fixed supply, immutable fee configuration, constant ignition threshold | Fixed at Spark |
| **Reaction allocation** | The 70% share of collected fees | A flow, not a balance |
| **Module set** | The complete collection of modules declared for a reaction | Fixed at Spark |
| **Module** | One declared, bounded behaviour over the reaction allocation | Immutable config fixed at Spark |
| **Accrual** | A module's credited but unspent balance | Increases by accrual, decreases only by settlement |
| **Settlement** | An externally initiated execution of a module against its accrual | Discrete, isolated |

The module set is **complete at declaration**. A module absent from the declared set can
never be added.

## Module manifest

The draft manifest format is
[specs/module-manifest.schema.json](../../specs/module-manifest.schema.json). v0 does not
change it.

| Field | Role in v0 |
| :--- | :--- |
| `module_id` | Identity within the reaction's set |
| `version` | Draft revision of the manifest format |
| `type` | Closed enum; determines semantics |
| `allocation_bps` | Declared claim on the reaction allocation |
| `activation_stage` | Earliest stage at which settlement may occur |
| `execution_mode` | How settlement is initiated |
| `hard_budget_ceiling` | Optional absolute cap, independent of the bps claim |
| `parameters` | Type-specific configuration |

**Unresolved:** where the manifest lives. v0 requires only that it is resolvable before
the first trade and cannot be substituted afterward.

## Lifecycle

| Stage | v0 behaviour |
| :--- | :--- |
| **Spark** | Module set declared; immutable configuration fixed. No settlement. |
| **Charging** | Modules accrue. Those whose activation stage has been reached may settle. |
| **Critical Mass** | No distinct semantics. Treated as late Charging. |
| **Ignition** | Working assumption: no module settles inside the ignition transaction. An assumption, not a decision - see Issue #2. |
| **Expansion** | Modules accrue from migrated-pool revenue and settle as configured. |

## Allocation model

**This section deliberately does not resolve the allocation conflict.**

What v0 fixes:

- A module's claim is expressed in **basis points of the reaction allocation**, not as an
  absolute amount. The allocation is a flow of unknown total size; absolute claims may
  never be satisfiable.
- Accrual is credited per module, per denomination.
- The sum of all module accruals never exceeds the reaction allocation credited.

What v0 does **not** fix:

- Whether declared shares across a set must sum to exactly 10000 bps, may sum to less,
  or express priority-ordered claims.
- What happens to a module's continued share once its hard budget ceiling is reached.
- Whether a module accrues before its activation stage.
- Behaviour for reactions that never ignite.

All four are dependencies of Issue #1. Four candidate schemes are set out with
trade-offs in
[research/allocation-constraints.md](../../research/allocation-constraints.md).
**No scheme is selected here.** A v0 implementation cannot be specified until one is.

## Settlement model

v0 adopts the settlement-style interface proposed in
[RFC 0001](../../rfcs/0001-module-interface.md):

```
  fee event ---> 30/70 split ---> reaction allocation
                                         |
                                         +-- per-module accrual (by declared share)
                                         |
                       settle(module_id)  <-- external caller, separate transaction
```

Properties this buys:

- A fee event performs accounting only. Trading cost is independent of the module set.
- A module cannot revert a trade, because it does not run inside one.
- A broken module is inert rather than dangerous.

**Unresolved:** who pays settlement gas and what incentivises them. A budget-funded
keeper reintroduces an outflow path that needs care. See Issue #2.

## Failure semantics

A settlement that reverts must not:

- revert any user transaction;
- prevent or delay ignition;
- alter the state of any other module;
- consume budget it did not spend.

Isolation of failure is a **hard requirement**, not a quality goal. This argues against
shared mutable state between modules and is the reason settlement is separated from fee
events.

**Unresolved:** whether a failed settlement records a failure marker, and whether
repeated failure should rate-limit further attempts.

## Invariants

Candidate invariants for a v0 implementation. Each is an assertion to be tested against,
not a verified property of anything.

| ID | Invariant | Enforcement |
| :--- | :--- | :--- |
| M-1 | Module configuration cannot alter initial supply, fee configuration, or the ignition threshold | By construction |
| M-2 | No module can prevent, delay, force or condition migration | By construction |
| M-3 | No module can withdraw principal from the migrated position | By construction |
| M-4 | No module receives any part of the 30% protocol share | By construction |
| M-5 | Sum of module accruals never exceeds the reaction allocation credited | Runtime |
| M-6 | Settlement spends at most the module's claimable balance at call time | Runtime |
| M-7 | Settlement of one module leaves every other module's state unchanged | By construction |
| M-8 | The module set observable at the first trade equals the set observable at any later block | By construction |
| M-9 | Wallet-to-wallet transfers remain untaxed regardless of module configuration | By construction |

**Not asserted:** no liveness guarantee. Nothing here promises a module ever settles.

## Interfaces

Conceptual surface only. Not a contract definition, not an API.

| Operation | Direction | Purpose |
| :--- | :--- | :--- |
| `accrue(amount, denomination)` | core to accounting | Credit the allocation; distribute by declared share |
| `claimable(module_id)` | read | Current settleable balance |
| `settle(module_id)` | external to module | Execute against the claimable balance |
| `manifest(module_id)` | read | Declared, immutable configuration |
| `state(module_id)` | read | Accrued, spent, remaining, last settlement |

## Unresolved decisions

1. Allocation conflict resolution scheme. **Blocks v0 implementation.**
2. Ceiling behaviour: redistribute, strand, or overflow?
3. Pre-activation accrual: from declaration, or from activation?
4. Module execution inside the ignition transaction: permitted or not?
5. Denomination handling post-ignition.
6. Settlement gas economics and caller incentive.
7. Manifest storage location and verifiability.
8. Behaviour for reactions that never reach critical mass.
9. Failure recording and retry rate-limiting.

## Dependencies on open Issues

| Dependency | Issue | Blocks |
| :--- | :--- | :--- |
| Allocation conflict resolution | #1 | Allocation model; Phase B onward of the implementation plan |
| Settlement and invocation semantics | #2 | Settlement model; failure semantics; Phase C onward |

## Open design issues

- [#1 - Define allocation conflict resolution](https://github.com/Rearctor/reaction-modules/issues/1)
- [#2 - Specify module settlement and invocation semantics](https://github.com/Rearctor/reaction-modules/issues/2)

## Related documents

- [Architecture](../architecture.md)
- [Module lifecycle](../module-lifecycle.md)
- [RFC 0001 - Module interface](../../rfcs/0001-module-interface.md)
- [Allocation constraints](../../research/allocation-constraints.md)
- [Implementation plan](implementation-plan.md)
