# Module Lifecycle

**Status: Concept / Research.** Nothing described here is implemented, deployed, or
scheduled.

A module's behaviour is defined relative to the reaction lifecycle, not to wall-clock
time. This document maps what a module may do at each stage, and what changes at the
stage boundaries.

## Stage map

| Stage | Reaction state | Module behaviour under consideration |
| :--- | :--- | :--- |
| **Spark** | Deployed; fixed supply; fee config immutable | Module set is declared and its immutable configuration is fixed. No execution. |
| **Charging** | Trading on the USDC bonding curve; reserves accruing | Modules accrue claims against the reaction allocation. Some may execute; some may only accrue. |
| **Critical Mass** | Reserves approaching 5,042 USDC | No distinct module semantics proposed. Treated as late Charging. |
| **Ignition** | Threshold transaction: graduation + migration, atomic | Execution inside this transaction is the central open question. |
| **Expansion** | Migrated Uniswap V4 position generating fees | Modules draw from post-migration revenue under the same 30/70 split. |

## Stage transitions that matter

### Spark → Charging

The module set becomes final. After this point the declared set is the complete set:
no additions, no substitutions.

**Invariant candidate:** for a given reaction, the set of module identifiers observable
at the first trade equals the set observable at any later block.

### Charging → Ignition

This is the sharp edge. Ignition is a single transaction that graduates the reaction
and migrates liquidity together. Three questions arise:

1. **May a module execute inside the ignition transaction?**
   Executing there gives a module access to the exact migration moment. It also adds
   gas and failure surface to the most consequential transaction in the lifecycle.

2. **What happens to accrued-but-unexecuted claims at ignition?**
   Options: carry forward into Expansion; settle immediately before migration; or
   forfeit. Forfeiture is simplest and worst for the creator; carrying forward is
   friendlier and complicates accounting across the boundary.

3. **Can a module's presence change the migrated amount?**
   It must not. At least 20% of initial supply migrates; a module that could reduce
   that figure would be reaching into protocol mechanics.

> **Open question.** Whether any module type may execute inside the ignition
> transaction at all. A defensible default is *no module executes during ignition* -
> ignition does exactly what it does today, and modules resume in Expansion. This is
> the most conservative option and the current working assumption, not a decision.

### Ignition → Expansion

Revenue source changes from curve trades to pool fees. Consequences:

- **Denomination mix changes.** Curve trading is USDC-denominated. The migrated pool
  produces fees in both USDC and the reaction token.
- **Cadence changes.** Curve activity during a launch is typically bursty; pool fees
  accrue with ongoing trading.
- **Some module types become newly meaningful.** Liquidity reinvestment has no obvious
  meaning pre-ignition, when there is no pool to reinvest into.

This is why the draft schema carries an `activation_stage`: a module may be declared at
Spark but only become executable later.

## Activation semantics

`activation_stage` marks the earliest stage at which a module may execute. Candidate
values: `charging`, `ignition`, `expansion`.

A module with `activation_stage: "expansion"` is declared at Spark, visible throughout,
and inert until migration completes. It still occupies its `allocation_bps` claim from
declaration - otherwise the allocation picture would change at ignition, which
reintroduces exactly the uncertainty that pre-declaration exists to remove.

> **Under research.** Whether a module should accrue an allocation claim before its
> activation stage, or only from activation onward. Accruing from declaration is more
> predictable; accruing from activation wastes less.

## Termination

Modules are not proposed to have an end state. A module with a finite budget becomes
inert when exhausted; it is not removed, and its record remains part of the declared
set.

> **Open question.** Whether an exhausted module's residual `allocation_bps` should
> redistribute to remaining modules or fall through to the creator. Redistribution
> makes the effective share of every other module depend on the exhaustion order of
> its peers, which is hard to reason about at purchase time.

## Failure behaviour

A module execution that reverts must not:

- revert the trade that triggered it;
- prevent ignition;
- alter the state of any other module;
- consume budget it did not spend.

Isolation of failure is treated as a hard requirement rather than a quality goal. See
the parallel treatment in the Autopilot repository's safety invariants.

## Related documents

- [Architecture](architecture.md)
- [Allocation constraints](../research/allocation-constraints.md)
- [RFC 0001 - Module interface](../rfcs/0001-module-interface.md)
