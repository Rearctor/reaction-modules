# Allocation Constraints

**Status: Concept / Research.** Open problem statement. No decision is recorded here.

Several modules may draw from the same 70% reaction allocation. This document sets out
the conflicts that creates and the options for resolving them. It deliberately stops
short of choosing.

## The shape of the problem

The reaction allocation is a **flow of unknown total size**, arriving in **two
denominations**, whose **rate and composition change at ignition**. Modules declare
claims against it before any of it exists.

Three properties a resolution scheme should have:

- **Predictable at purchase time.** A buyer during Charging should be able to read the
  manifest and know what claims exist.
- **Order-independent where possible.** A module's effective share should not depend
  on the arbitrary order in which peers were settled.
- **Non-starving.** A low-priority module should not be permanently unreachable if the
  declared set implies it should receive something.

These three are in tension. Strict priority ordering is predictable but starves; pro
rata is non-starving but makes any individual module's rate depend on the whole set.

## Candidate schemes

### A. Pro rata by basis points, must sum to 10000

Every module declares `allocation_bps`; the set must sum to exactly 10000. Each fee
event splits the allocation in those proportions.

- **Predictable:** yes - a module's share is a constant fraction.
- **Order-independent:** yes.
- **Non-starving:** yes.
- **Cost:** the creator cannot leave a remainder for themselves without declaring a
  `treasury-allocation` module for it. That may be acceptable, and arguably makes the
  creator's own share explicit rather than residual.

### B. Pro rata, may sum to less than 10000

Remainder falls through to the creator.

- Same properties as A, plus an implicit creator share.
- **Cost:** "unallocated" is silent. A manifest summing to 4000 gives the creator 60%
  without saying so anywhere. (Worked example. Illustrative only. Not protocol defaults or committed parameters.) Every property above is preserved except legibility, and
  legibility is the whole point of pre-declaration.

### C. Priority-ordered claims

Modules are ordered; each is filled to its target before the next receives anything.

- **Predictable:** only if you also know the flow size, which nobody does at
  declaration time.
- **Order-independent:** no, by construction.
- **Non-starving:** no. A module low in the order on a reaction with modest volume may
  never receive anything.
- **Possibly appropriate** for a narrow case: a vesting module that must be satisfied
  before discretionary spending.

### D. Hybrid - priority tiers, pro rata within tier

Tier 1 fills before tier 2; within a tier, pro rata.

- Expressive enough for "vest first, then split the rest".
- **Cost:** the manifest becomes materially harder to read, and starvation returns for
  lower tiers.

> **Open question.** Which of A-D, if any. A is the most legible and the working
> assumption for the draft schema; it is not a decision.

## Interaction with hard budget ceilings

`hard_budget_ceiling` is an absolute cap independent of the bps claim. Once a module
reaches its ceiling it stops spending - but its bps claim still exists.

Two behaviours are possible:

1. **Ceiling stops accrual.** The module's share redistributes to the remaining set.
   Every other module's effective rate then changes at an unpredictable moment.
2. **Ceiling stops spending only.** The module keeps accruing into a balance it will
   never spend. That value is stranded unless a fallback is declared.

Neither is obviously right. Option 1 breaks order-independence in a subtle way -
effective shares become a function of exhaustion order. Option 2 strands value.

> **Under research.** A third option: a declared `overflow_target` naming which module
> or address receives a ceilinged module's continued accrual. Adds a field and a
> validation rule (no cycles), and keeps both predictability and non-starvation.

## Denomination asymmetry

Pre-ignition the allocation is USDC-denominated. Post-ignition it arrives as both USDC
and reaction token. A module whose behaviour is meaningful in only one denomination -
a buyback spends USDC, holder rewards might distribute either - will accrue value it
cannot use if accrual is denomination-blind.

This interacts directly with the unresolved denomination handling in
[RFC 0001](../rfcs/0001-module-interface.md).

## Cross-boundary accounting

If a module accrues during Charging but activates at Expansion, its balance crosses
ignition. Questions:

- Is pre-activation accrual retained, or does the module start from zero at activation?
- Does a carried balance count against `hard_budget_ceiling`?
- If ignition never occurs, what becomes of accrued balances? A reaction that never
  reaches 5,042 USDC has a permanently unsettled allocation.

The last case deserves attention: **most reactions may never ignite.** A scheme that is
only coherent post-ignition leaves the common case undefined.

> **Open question.** Behaviour for reactions that never reach critical mass is
> unspecified across all of the above. This may be the most important gap in the
> current draft.

## Constraints a resolution must satisfy

Regardless of scheme:

1. Total distributed to modules never exceeds the reaction allocation credited.
2. No module may receive protocol-share revenue (the 30%).
3. A module's declared claim is readable before the reaction's first trade.
4. Settlement of one module cannot alter another module's accrued balance.
5. No scheme may make ignition conditional on module state.

## Related documents

- [Architecture](../docs/architecture.md)
- [Module lifecycle](../docs/module-lifecycle.md)
- [RFC 0001 - Module interface](../rfcs/0001-module-interface.md)
