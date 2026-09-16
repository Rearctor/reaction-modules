# Reaction Modules - v0 Implementation Plan

## Status

**Concept / Research.** This plan describes work that has not started. No phase is
complete, in progress, or scheduled.

There are **no dates** in this document, and none will be added until the specification
dependencies below are resolved. Phase ordering is a dependency order, not a timeline.

No implementation exists. No contracts exist. No tests exist. No benchmarks exist.

## How to read this

Each phase states an objective, its inputs, deliverables, blockers, and exit criteria.
A phase is startable only when its blockers are cleared. Exit criteria are written so
that a reviewer can disagree with a claim that a phase is finished.

Companion document: [v0 design specification](design-spec.md).

## Phase A - Specification stabilization

**Objective.** Reach a v0 specification that a reviewer can argue with in detail:
unambiguous entities, an agreed allocation scheme, and settled failure semantics.

**Inputs.**
- [docs/v0/design-spec.md](design-spec.md)
- [research/allocation-constraints.md](../../research/allocation-constraints.md)
- [rfcs/0001-module-interface.md](../../rfcs/0001-module-interface.md)
- [specs/module-manifest.schema.json](../../specs/module-manifest.schema.json)
- Issues #1 and #2

**Deliverables.**
- A selected allocation conflict resolution scheme, with the rejected alternatives and
  the reasoning recorded.
- A decision on module execution inside the ignition transaction.
- Defined behaviour for reactions that never reach critical mass.
- Manifest schema revised to reflect both decisions.

**Blockers.**
- Issue #1 is open. No allocation scheme is selected.
- Issue #2 is open. Ignition-boundary behaviour is undecided.

**Exit criteria.**
- Every item under *Unresolved decisions* in the design spec is either resolved with
  recorded reasoning, or explicitly deferred out of v0 with a stated consequence.
- The manifest schema validates its examples and expresses the selected scheme.
- No section of the design spec still reads "no scheme is selected here".

## Phase B - Reference data structures

**Objective.** A language-agnostic reference model of module state and accrual
accounting, sufficient to reason about the invariants without committing to a runtime.

**Inputs.** Phase A output; invariants M-1 through M-9.

**Deliverables.**
- Reference structures for module manifest, module state, and per-denomination accrual.
- A worked model of accrual distribution under the selected scheme.
- Documented state transitions for accrual and settlement.

**Blockers.**
- Phase A incomplete. Accrual distribution cannot be modelled before the allocation
  scheme is chosen.
- Denomination handling unresolved.

**Exit criteria.**
- Every field in the design spec's core entities table has a defined representation.
- Accrual distribution is deterministic and order-independent, or the dependence is
  documented and justified.
- Structures express all nine invariants without relying on runtime checks for those
  marked "by construction".

## Phase C - Execution model prototype

**Objective.** A non-production prototype of the settlement path, used to find where the
design is underspecified.

**Inputs.** Phase B structures; settlement model from the design spec.

**Deliverables.**
- A prototype settlement path: accrue, claimable, settle.
- At least one module type implemented against it for exercise purposes.
- A written record of every ambiguity the prototype exposed.

**Blockers.**
- Phase B incomplete.
- Settlement gas economics and caller incentive unresolved (Issue #2).

**Exit criteria.**
- The prototype exercises settlement for a multi-module set without shared mutable state.
- Failure of one module's settlement demonstrably leaves other modules unchanged.
- Ambiguities found are recorded as new Issues rather than silently resolved.

**Explicitly not an exit criterion:** production readiness, gas efficiency, or audit
suitability. This phase produces a prototype for learning, not a candidate for
deployment.

## Phase D - Invariant testing

**Objective.** Convert invariants M-1 through M-9 from assertions into executable checks
against the Phase C prototype.

**Inputs.** Phase C prototype; the invariants table.

**Deliverables.**
- An executable check per invariant, each able to fail.
- Adversarial cases: a module attempting to reach protocol parameters; concurrent
  settlement; settlement at the ignition boundary; ceiling exhaustion mid-sequence.
- A record of which invariants the prototype violates.

**Blockers.**
- Phase C incomplete.
- Invariants whose enforcement is "by construction" cannot be meaningfully tested until
  the construction exists.

**Exit criteria.**
- Each invariant has a check that fails when the invariant is deliberately broken - a
  test that cannot fail proves nothing.
- Violations are recorded, not suppressed.

## Phase E - Integration research

**Objective.** Understand how a module set would interact with the rest of the protocol,
and what the Control Room and Autopilot surfaces would need.

**Inputs.** Phases A-D output; the Autopilot repository's execution model and safety
invariants.

**Deliverables.**
- An analysis of accrual behaviour across the ignition boundary in both denominations.
- An assessment of overlap between module settlement and Autopilot strategy execution,
  both of which draw on the reaction allocation.
- Identification of any protocol change a module set would require - with the
  expectation that the answer should be none.

**Blockers.** Phases A-D incomplete.

**Exit criteria.**
- Interaction between modules and automated strategies over one allocation is described,
  including precedence.
- Any required protocol change is either eliminated or escalated as a blocking finding.

## Status of every phase

| Phase | Status |
| :--- | :--- |
| A - Specification stabilization | Not started. Blocked on Issues #1 and #2. |
| B - Reference data structures | Not started. Blocked on Phase A. |
| C - Execution model prototype | Not started. Blocked on Phase B. |
| D - Invariant testing | Not started. Blocked on Phase C. |
| E - Integration research | Not started. Blocked on Phase D. |

No phase has begun. No deliverable in this document exists.

## Open design issues

- [#1 - Define allocation conflict resolution](https://github.com/Rearctor/reaction-modules/issues/1)
- [#2 - Specify module settlement and invocation semantics](https://github.com/Rearctor/reaction-modules/issues/2)
