# RFC 0001 - Module Interface

- **Status:** Draft / Research. Not accepted, not scheduled, not implemented.
- **Scope:** How a common interface between a reaction and its declared modules might
  be shaped.
- **Supersedes:** nothing.

## Problem

A reaction needs to route part of the reaction allocation to a declared set of modules
without granting those modules reach into protocol mechanics. Two failure modes bound
the design:

- **Too permissive.** A general callback surface lets a module influence trading,
  graduation, or migration. This destroys the guarantee that makes a reaction
  inspectable.
- **Too rigid.** A hard-coded set of behaviours with no shared structure means each
  module type is a bespoke change to core code, and a participant has to learn each one
  separately.

The aim is a narrow, uniform interface over a closed set of module types.

## Proposed shape

### Settlement, not invocation

The working proposal inverts the usual extension pattern. Rather than the reaction
*calling* modules, modules **settle against accrued balances**.

```
fee event ──▶ 30/70 split ──▶ reaction allocation balance
                                        │
                                        ├── per-module accrual (by allocation_bps)
                                        │
                              ┌─────────┴─────────┐
                              │                   │
                     settle(module_id)     settle(module_id)
                     (external caller)     (external caller)
```

Consequences:

- A fee event does no module work. It updates accounting only. Trading cost stays
  independent of how many modules a reaction declared.
- A module cannot revert a trade, because it does not run during one.
- Settlement is a separate transaction that anyone may call, paying its own gas.
- A broken module is inert rather than dangerous: its balance accrues and is never
  claimed.

> **Open question.** Who pays settlement gas, and what incentivises anyone to call it?
> Options: the creator; a keeper compensated from the module's own budget; or settle
> lazily when the creator next interacts. A budget-funded keeper reintroduces a
> withdrawal path and needs care.

### Candidate interface surface

Conceptual, not a contract definition:

| Operation | Direction | Purpose |
| :--- | :--- | :--- |
| `accrue(amount, denomination)` | core → accounting | Credit the reaction allocation; distribute across modules by declared bps |
| `claimable(module_id)` | read | Current settleable balance for a module |
| `settle(module_id)` | external → module | Execute the module's declared behaviour against its claimable balance |
| `manifest(module_id)` | read | The declared, immutable configuration |
| `state(module_id)` | read | Accrued, spent, remaining budget, last settlement |

`accrue` is the only path core code takes. Everything else is read or externally
triggered.

### Denomination handling

Post-ignition revenue arrives in both USDC and the reaction token. A module declaring
a USDC budget will also accrue token-denominated revenue.

Options:
1. Accrue per denomination separately; a module settles each independently.
2. Normalise to USDC at settlement time using the migrated pool price.
3. Require modules to declare which denominations they accept; route the rest to the
   creator.

Option 2 introduces a price dependency inside settlement, which is an oracle problem
and a manipulation surface. Option 1 is the most conservative and is the working
assumption.

> **Under research.** Denomination handling is unresolved. Option 1 doubles the
> accounting surface; option 3 may leave revenue stranded.

## Invariants this interface must preserve

1. `accrue` must not be able to fail in a way that reverts the originating fee event.
2. Sum of all module accruals must never exceed the reaction allocation credited.
3. `settle` must not be able to spend more than `claimable(module_id)` at call time.
4. `settle` on one module must leave every other module's state unchanged.
5. No operation may reach initial supply, fee configuration, ignition threshold,
   migration, or migrated principal.

Invariant 4 in particular argues against shared mutable state between modules.

## Alternatives considered

**Direct invocation during fee events.** Rejected in this draft: puts module code in
the path of every trade and makes trading cost a function of the module set.

**Single aggregate module.** One behaviour per reaction, no set. Simpler, but a
creator wanting both buybacks and vesting is forced to pick.

**Off-chain execution with on-chain settlement proofs.** Moves complexity off-chain at
the cost of a new trust assumption. Not explored here; possibly worth a separate RFC.

## Unresolved

- Settlement gas economics and caller incentive.
- Denomination handling (above).
- Whether any module may execute inside the ignition transaction. See
  [module-lifecycle.md](../docs/module-lifecycle.md).
- Whether `allocation_bps` must sum to 10000. See
  [allocation-constraints.md](../research/allocation-constraints.md).
