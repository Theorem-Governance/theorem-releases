# Theorem — Product Roadmap

**Last updated:** 2026-03-20
**Owner:** Product / CPO
**Status:** Active

## Current Control Plane

This roadmap captures the earlier `v0.3.0 -> 2.x -> 3.0` product framing.

It is not the active execution control plane for the current repo state.
The active line is:

- `0.6.3` substrate closure
- `0.7` entity birth
- `0.8` entity learning
- `0.9` entity interoperation
- `1.0` autonomous economy

Use [`docs/execution/1.0-runbook.md`](docs/execution/1.0-runbook.md) as the
current release train and gate source of truth.

## Product Thesis

Theorem is not trying to become a better helper process around an existing
system. Theorem is trying to become the trusted state machine for the larger
system.

The governing roadmap question is therefore:

"What important thing still has to be trusted outside Theorem?"

If a planned item reduces that answer, it is core roadmap work. If it does not,
it is secondary.

## Product Standard

Theorem is only succeeding when all of the following become true:

- canonical authority state lives in Theorem
- irreversible transitions originate in theorem-governed surfaces
- artifact and release authority are theorem-native
- external execution is capability-based and fail-closed
- operators have a production-grade contract
- authoritative truth can be rebuilt from theorem-native records and events

This means the roadmap should optimize for authority, revocation,
rebuildability, and fail-closed control before it optimizes for breadth,
ergonomics, or ecosystem polish.

## What v0.3.0 Changed

`v0.3.0` was not just "more governance features." It was the first threshold
where the product changed who the system trusts.

At that point:

- an operator can prove why a release is eligible before rollout
- a governed runtime shell cannot self-authorize work
- artifact and capability revocation have real stopping power
- theorem-native truth survives projection, cache, and adapter loss
- downstream systems can consume theorem authority without guessing which
  surfaces are canonical

## Current Product Position

Today Theorem is between two product states:

- it is already a deployable runtime with a deterministic kernel, audit
  surface, witness model, real host deployment path, and ring3 execution stack
- it has crossed the fail-closed runtime-control threshold in `v0.3.0`
- it is not yet the complete trusted state machine for lineage, trust,
  explanation, rebuild, delegation, and networked authority

In product terms: the runtime exists, runtime control is now real, but the
trusted state machine and trust control plane are not yet complete.

## Strategic Priorities

The roadmap priorities, in order, are:

1. canonical authority state
2. governed irreversible transitions
3. runtime and release authority
4. capability-based external execution
5. rebuildable truth
6. production operational contract
7. model-driven governance quality and auditability

## Product Programs

These are the major product enhancement tracks. Releases package them into
thresholds, but the tracks themselves describe the durable scope of the
product.

### 1. Authority Ownership

Move authority-bearing facts into theorem-native state:

- intent, admission, commitment, capability, trust, fork, and lineage records
- theorem-owned transitions for admission, verification, revocation, and
  governed mutation
- explicit distinction between canonical theorem truth and adapter-local
  convenience state

### 2. Runtime Gatekeeping

Make theorem-native authority decide what may run:

- capability issuance, validation, expiry, redemption, and revocation
- runtime rechecks at the supervisor, bus, gateway, and govagent layers
- removal of self-authorization and implicit local bypass paths
- artifact and release eligibility enforced at runtime boundaries

### 3. Recovery And Rebuild

Make theorem-native truth survivable:

- stable event and export surfaces
- authoritative rebuild and restore flows
- lineage and fork reasoning that survive projection destruction
- trust downgrade and revocation semantics tied to the same history

### 4. Operator Contract

Make the product deployable and supportable on real hosts:

- stable release assets, checksums, and SBOMs
- source-tarball and flake verification
- install, observe, upgrade, rollback, and recovery guidance
- stable distinction between public, operator/admin, and adapter-internal
  surfaces

### 5. Consumption Integrity

Make every consumer of theorem authority behave coherently:

- one canonical contract across CLI, HTTP, Ring 3, and rebuild flows
- stable public surfaces that are safe to integrate against
- operator/admin surfaces that are explicit rather than leaked through docs or
  code discovery
- downstream consumers that transport theorem authority without redefining it

### 6. Authority Legibility

Make theorem-native authority explainable rather than only enforceable:

- explicit lineage and decision traversal
- trust-state history and downgrade visibility
- branch / fork inspection
- operator and auditor queries that explain why authority exists

### 7. Programmable Governance

Make theorem-native control programmable rather than fixed to today’s runtime
paths:

- rollout policies
- staged admission lanes
- delegated authority models
- simulation and counterfactual evaluation
- policy-aware revocation and approval semantics

### 8. Networked Trust

Make theorem-native trust portable across systems:

- portable authority bundles
- cross-instance witnessing
- delegated scopes across theorem instances
- federated verification
- dispute and divergence handling

### 9. Autonomous Organization Surface

Expose theorem-native authority as the operating substrate for AI-native
organizations:

- visible execution lanes
- governed delegation between agents and roles
- auditable interventions and overrides
- organization-level trust and lineage
- unified control over release, runtime, trust, and delegation

## Release Train To v0.3.0

This roadmap is intentionally release-shaped rather than phase-shaped. The goal
is not to ship incremental patches that feel busy. The goal is to make each
release cross a real product threshold.

```text
v0.1.1 -> production contract and host reliability
v0.2.0 -> canonical authority layer
v0.2.1 -> capability lifecycle and runtime bridge
v0.2.2 -> lineage, trust, and rebuildable authority
v0.3.0 -> fail-closed runtime control
```

Each release below should retire one whole class of misplaced trust.

---

## v0.1.1 — Production Contract

**Release outcome:** Operators can install, verify, observe, upgrade, and roll
back a real Theorem host without reading the code.

### Required scope

- Linux-first release publication independent of Darwin
- explicit liveness and readiness semantics
- stable release asset, checksum, and SBOM contract
- production deployment docs
- supported Nix story
- explicit authority-surface classification

### Exit gate

- release path is verified on the real host path
- root flake and source-tarball build are verified
- operators have a stable release/install/rollback contract
- public authority contract is separated from operator/admin and
  adapter-internal surfaces

### Why it matters

We should not build authority on top of a runtime whose operational contract is
still ambiguous.

---

## v0.2.0 — Canonical Authority Layer

**Release outcome:** Theorem stops being only the place where governance is
checked and becomes the place where authority lives.

### Product objective

Move the minimum canonical authority objects and transitions into theorem-native
state so later runtime control work has something real to enforce.

This is the release where theorem-native authority stops being implied and
starts being structurally unavoidable.

### Required scope

Canonical object ownership:

- intent
- admission
- commitment
- capability
- trust record
- fork record
- authority history sufficient for export and rebuild

Stable theorem-owned authority surfaces:

- declare intent
- submit act
- establish entity
- create scope
- amend parameter
- revoke authorization
- submit proof
- admit artifact
- revoke artifact
- admit action
- revoke action admission
- verify release
- issue execution capability
- validate execution capability
- revoke execution capability
- redeem execution capability
- record trust
- record fork

Authoritative operator/admin surfaces:

- export state
- rebuild authority state
- restore authority state

### Enhancement themes

- stop mirroring external release decisions and start recording theorem-native
  release authority
- stop treating Ring 3 runtime state as implicit authority and start treating
  it as a consumer of theorem-native capability state
- make export and rebuild a first-class authority story rather than a
  diagnostic convenience

### Exit gate

- theorem-native authority objects are explicit and durable
- public and operator/admin contract boundaries are documented and enforced
- export and rebuild treat authority objects as canonical
- no important authority primitive still exists only as adapter-local state

### What does not count

- adding more helper APIs that bypass theorem-native authority objects
- documenting adapter behavior as if it were canonical product truth

---

## v0.2.1 — Capability Lifecycle And Runtime Bridge

**Release outcome:** Theorem-issued capability state becomes the canonical
runtime bridge into external execution.

### Product objective

Close the gap between "capabilities exist in theorem state" and "Ring 3
actually runs through those capabilities."

This is the release where capability becomes the control membrane between
theorem authority and external execution.

### Required scope

Capability lifecycle completion:

- issuance
- validation
- redemption
- revocation
- expiry
- replay protection
- payload binding
- scope binding
- outcome binding

Ring 3 bridge work:

- theorem-bus consumes theorem-issued capability state for governed execution
- theorem-gateway cannot tunnel around capability requirements
- theorem-supervisor runtime admission and rechecks are theorem-native
- theorem-govagent paths that perform governed work operate through theorem
  capability semantics, not local convention

### Enhancement themes

- remove or deprecate legacy execution paths that can bypass theorem-native
  capability state
- make runtime denial observable and auditable, not merely implied
- make capability outcome return part of the authority story, not an afterthought

### Exit gate

- every governed external execution path can be expressed through theorem
  capability state
- denial, expiry, and revocation have one canonical meaning across bus,
  gateway, supervisor, and govagent
- no Ring 3 component can honestly claim theorem-native authority while still
  bypassing theorem-native capability checks

### What does not count

- issuing capabilities while continuing to let runtime shells self-authorize
- adding more compatibility shims without removing the bypass paths

---

## v0.2.2 — Lineage, Trust, And Rebuildable Authority

**Release outcome:** Theorem-native authority becomes survivable, explainable,
and branch-aware.

### Product objective

Complete the authority layer so operators and downstream systems can answer not
just "what is true?" but "why is it true, what branch did it come from, and can
it be rebuilt after loss?"

This is the release where theorem-native truth becomes legible enough to trust
under damage, rollback, and disagreement.

### Required scope

Lineage and trust:

- explicit decision lineage or equivalent authority history structure
- trust downgrade and revocation semantics tied to artifact, release, witness,
  and capability flows
- fork reasoning that is theorem-native rather than reconstructed from logs

Rebuildability:

- stable governance event feed shape
- projection destruction and rebuild drills
- restore path that proves theorem-native history is sufficient to recover
  authority state

Operational truth:

- event consumption contract for operators and Ring 3 consumers
- rebuild and recovery guidance that uses theorem-native history as the source
  of truth

### Enhancement themes

- move from "authority records exist" to "authority records explain the system"
- move from "export can dump data" to "history can reconstruct truth"
- move from "trust is scattered across side effects" to "trust state is
  explicit and governable"

### Exit gate

- destroyed projections can be rebuilt from theorem-native history
- trust-critical state is explicit rather than spread across unrelated records
- lineage and fork history are theorem-native enough to support rollback,
  rebuild, and branch reasoning

### What does not count

- keeping event history as a debugging artifact rather than a rebuild substrate
- claiming trust is explicit when it is still inferred from operator convention

---

## v0.3.0 — Fail-Closed Runtime Control

**Release outcome:** Theorem becomes the real gatekeeper for what may run, what
may continue running, and what must stop.

This is the first release that should feel like a threshold crossing rather
than a feature accumulation release.

### Product objective

Make theorem-native authority actually control runtime behavior. If Theorem is
absent, revoked, expired, mismatched, or denied, governed external execution
must fail closed.

### Required scope

Artifact and release control:

- unadmitted artifacts are denied at runtime boundaries
- revoked artifacts are denied at runtime boundaries
- release verification is a real eligibility gate
- rollout eligibility is theorem-governed rather than external convention

Capability-based runtime control:

- governed execution requires theorem-issued capability
- capability expiry blocks execution
- capability revocation blocks execution
- payload mismatch blocks execution
- scope mismatch blocks execution
- unavailability of theorem authority blocks governed execution rather than
  silently degrading into self-authorization

Runtime control across the stack:

- theorem-supervisor enforces runtime admission and revalidation fail-closed
- theorem-bus cannot admit governed work without theorem-native authority
- theorem-gateway cannot present a convenience surface that weakens theorem
  control
- theorem-govagent behaves correctly under denial, expiry, revocation, and
  rebuild

Operational survivability:

- governance event surface is stable enough for subscription and recovery
- destroy/rebuild drills prove loss of projections/adapters does not destroy
  authoritative truth

### Enhancement themes

- eliminate the last meaningful self-authorization paths
- turn theorem-native revocation into real runtime stopping power
- make runtime control and recovery one coherent product, not separate stories

### v0.3.0 release test

Ask these questions before calling the release done:

- can an external runtime still self-authorize governed work?
- can an unadmitted or revoked artifact still run?
- can a governed action continue when theorem capability state is absent,
  expired, revoked, or mismatched?
- can authoritative state be restored after adapter or projection destruction?

If any answer is yes, `v0.3.0` is not complete.

### What does not count

- advisory denial paths
- warnings without blocking behavior
- runtime checks that can be bypassed by alternate Ring 3 paths
- rebuild stories that still depend on adapter-local truth

---

## What We Are Explicitly Not Prioritizing Ahead Of The Trust-Control-Plane Path

These may still ship, but they should not lead the roadmap:

- broader UI polish
- non-critical SDK ergonomics
- integrations that widen surface area without increasing authority ownership
- analytics-first work that assumes external truth
- model-quality work presented as a substitute for authority control

## Parallel Track — DEC-010 Governance Inference Engine

DEC-010 remains strategically important, but it is not the definition of
`v0.3.0` and it is not the definition of 2.0.

### What it should do

- increase governance throughput
- increase corpus quality
- exercise theorem-native transitions under realistic load
- improve decision quality and auditability

### What it must not do

- become the reason we declare authority ownership complete
- substitute model quality for theorem-native control

## 2.0 Gate Beyond v0.3.0

`v0.3.0` is not 2.0. It is the release where runtime control becomes real.

Theorem should only be declared 2.0 when all of the following are true:

- authority objects are theorem-native
- irreversible transitions originate in Theorem
- artifacts, releases, and governed execution are theorem-controlled
- external execution is capability-based and fail-closed
- operators have a stable install / verify / observe / upgrade / rollback /
  subscribe contract
- authoritative state can be rebuilt from theorem-native commitments and event
  history

If any important irreversible truth still originates outside Theorem, we are
not at 2.0.

## Long-Range Horizon Beyond v0.3.0

`v0.3.0` is the threshold where runtime control becomes real. It is not the end
of the roadmap.

The long-range product path from here is:

```text
v0.3.0 -> fail-closed runtime control
2.0    -> trusted state machine declaration
2.1    -> network trust fabric
2.2    -> programmable local governance
2.3    -> authority graph and explanation product
3.0    -> autonomous organization OS
```

### 2.0 Declaration Gate

Theorem should only be declared `2.0` when:

- authority-bearing truth is theorem-native
- irreversible transitions originate in theorem-governed surfaces
- trust-critical state is explicit rather than inferred from adapters
- lineage and rebuild are product-grade rather than partial
- governed execution cannot self-authorize outside theorem-native control

This is the completion of the current thesis.

### 2.1: Network Trust Fabric

After `2.0`, Theorem should first make theorem-native trust portable across
theorem boundaries.

This threshold should deliver:

- cross-instance trust exchange
- portable authority packages
- delegated scopes across theorem instances
- federated release and artifact verification
- divergence and dispute handling

This is where Theorem starts to govern ecosystems instead of single stacks.

### 2.2: Programmable Local Governance

After network trust exists, Theorem should make local trust control
programmable:

- rollout policies
- staged admissions
- delegated authority models
- conditional revocation
- simulation and counterfactual evaluation

At this point Theorem stops feeling like a fixed set of governance endpoints and
starts feeling like a real control plane.

### 2.3: Authority Graph And Explanation Product

After trust is portable and programmable, Theorem should make authority,
delegation, and trust state legible as a stable product surface.

This threshold should deliver:

- authority timelines
- decision-edge traversal
- branch / fork inspection
- trust-state history
- runtime-eligibility explanation
- event subscriptions safe for downstream consumption

### 3.0: Autonomous Organization OS

At `3.0`, Theorem should become the operating system for autonomous
organizations.

That includes:

- theorem-native roles and charters
- theorem-native decision paths
- governed execution lanes
- budget and blast-radius boundaries
- auditable interventions and overrides
- cross-organization treaties over theorem-native trust
- visible organizational operating surface
- unified control over release, runtime, trust, delegation, and intervention

This is not a side market. It is the natural product consequence of making
Theorem the system that decides who may act, why, and under what revocable
authority.

## Strategic Order From Here

Recommended order from here:

1. complete the remaining `2.0` declaration-gate work: lineage, trust
   explicitation, and rebuildability with no adapter-local truth dependency
2. make theorem-native trust portable across theorem boundaries
3. make theorem-native trust control programmable rather than fixed to today’s
   runtime flows
4. make authority, delegation, and trust state legible as stable product
   surfaces
5. make the organization itself theorem-native and expose it as an operating
   system

This is the strategic path. Anything that does not reduce trust outside Theorem
or increase theorem-native control should sequence behind it.

## Summary

Theorem should now be managed as a product becoming the trust control plane for
autonomous systems, not as a runtime accumulating features.

The roadmap to `v0.3.0` should therefore feel like a control-surface takeover:

- `v0.1.1` makes the runtime operable
- `v0.2.0` makes authority explicit
- `v0.2.1` makes capability the runtime bridge
- `v0.2.2` makes authority rebuildable and explainable
- `v0.3.0` makes runtime control real

The roadmap after `v0.3.0` should feel like category formation:

- `2.0` completes the trusted state machine
- `2.1` makes theorem-native trust portable across systems
- `2.2` makes trust control programmable
- `2.3` makes authority and trust state legible as product surfaces
- `3.0` makes Theorem the OS for autonomous organizations
