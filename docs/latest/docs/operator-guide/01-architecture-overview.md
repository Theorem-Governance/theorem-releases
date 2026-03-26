# Architecture Overview

This document describes the architecture of the full Theorem runtime: the
governance model, trust layers, workspace crate structure, runtime boundaries,
enforcement rules, audit criteria, and the SealedBatch compile-time safety
pattern.

## Current Strategic Boundary

Theorem is no longer treating the host operating system as the durable product
substrate. The active architecture direction is the sovereign theorem machine
stack defined in [ADR-001-theorem-sovereign-machine-stack.md](/C:/Users/Eric/Theorem/docs/platform/decisions/ADR-001-theorem-sovereign-machine-stack.md)
and [2026-03-25-theorem-machine-foundation.md](/C:/Users/Eric/Theorem/docs/superpowers/specs/2026-03-25-theorem-machine-foundation.md).

In practical terms:

- current FreeBSD install/upgrade/boot-environment work is bootstrap carrier
  scaffolding
- theorem-native boot, kernel, activation, and recovery semantics are the
  durable architecture
- new product law should be expressed in theorem-native machine terms, not in
  host-OS configuration conventions

## Governance Model

Every operation in Theorem follows a four-phase pipeline:

1. **Acts** -- An entity declares intent, then submits an act (execution or governance) with authorization, scope, and resource consumption declarations.
2. **Enforcement** -- The kernel validates the act against rules E1-E16. Violations on input rules (E1-E4, E7, E10, E12, E14) cause hard rejection. Violations on health rules (E8, E13, E15, E16) produce escalation signals and set a degraded flag.
3. **Attribution** -- On acceptance, the kernel produces an Attribution record linking the act to its authorizing entity, executing entity, beneficiary entities, drift signals, and resource consumption.
4. **Attestation** -- An Attestation record is produced atomically with the Attribution, containing canonical serialization, integrity proof, and summaries. Attestations begin as Provisional and transition to Verifiable once published to the witness network.

Bootstrap is the sole exception: it produces genesis records, entities, scopes, authorizations, kernel parameters, and proof obligations in a single batch exempt from E1-E8 and E14.

## Trust Kernel Architecture

The kernel is organized into numbered layers. Higher layers depend on lower layers; lower layers never reference higher ones.

### Layer 2: Primitives

All governance data types. Defined in `theorem-kernel/src/primitives/`. Each primitive has a dedicated module: Act, Attribution, Attestation, Authorization, DriftSignal, Entity, Genesis, GovernanceAct, Intent, KernelParameters, MeasurementMethod, ProofObligation, ResourceConsumption, Scope, Signal, Workflow, and supporting types (shared sub-schemas, vocabularies, branded IDs).

Primitives are pure data structures with serde Serialize/Deserialize. They carry no logic beyond field-level validation (`KernelParameters::validate()`).

### Layer 3: Enforcement (E1-E16)

Pure functions in `theorem-kernel/src/enforcement/`. Each rule takes a candidate record and a read-only `&dyn GovernanceState` reference, returning `EnforcementResult { accepted: bool, violations: Vec<String> }`. Enforcement rules never call `Utc::now()` -- they receive a `reference_time` parameter for determinism.

### Layer 4: Audit (C1-C22)

Criteria in `theorem-kernel/src/audit/`. Each criterion is a pure function over `&dyn AuditableGovernanceState` (which extends GovernanceState with windowed collection queries) and an `AuditContext` (window_start, window_end, previous_audit_at, is_first_audit). The audit runner evaluates all 22 criteria sequentially with no short-circuit -- every criterion runs even if earlier ones fail.

### Layer 7a: Deterministic Verifier

25 property checks across 8 boundaries in `theorem-kernel/src/verifier/`. Returns `VerifierResult { checks: Vec<PropertyCheckResult>, valid: bool }`. A result is AUDIT VALID if every deterministically verifiable property passes. Properties that require physical audit return `RequiresPhysicalAudit` status, which does not block validity.

**Property boundaries:**
- Parameter outcomes: PROP-PARAM-001, PROP-PARAM-002, PROP-PARAM-003
- Entity identity: PROP-IDENTITY-001, PROP-IDENTITY-002, PROP-IDENTITY-003
- System entity liveness: PROP-LIVENESS-001, PROP-LIVENESS-002, PROP-LIVENESS-003, PROP-LIVENESS-004
- Governance substance: PROP-SUBSTANCE-001 through PROP-SUBSTANCE-006
- External infrastructure: PROP-INFRA-001, PROP-INFRA-002, PROP-INFRA-003
- Enforcement correctness: PROP-ENFORCEMENT-001
- Proof and specification integrity: PROP-PROOF-001, PROP-PROOF-002, PROP-AVAILABILITY-001
- Sincerity and substrate: PROP-SINCERITY-001, PROP-SUBSTRATE-001

## Crate Structure

```
theorem-kernel         Pure governance logic. No I/O. No network. No filesystem.
                       Enforcement (E1-E16), audit (C1-C22), verifier
                       (25 properties), kernel write path, bootstrap,
                       amendment, proof lifecycle.

theorem-storage-sqlite SQLite persistence implementing GovernanceState,
                       AuditableGovernanceState, and StorageAdapter traits.
                       Applies SealedBatch atomically via a single transaction.

theorem-node           Canonical runtime surface. CLI (clap) + HTTP server
                       (axum), attestation finalization, witness publication,
                       authority-state export and rebuild endpoints.

theorem-test-support   InMemoryState implementing GovernanceState and
                       AuditableGovernanceState for unit and integration tests.

theorem-witness        Append-only witness log with hash chain integrity.
                       WitnessLog trait, WitnessNode orchestrator, peer
                       verification, integrity reports.

theorem-witness-storage SQLite-backed WitnessLog implementation.

theorem-bus            Ring 3 governance bus. Turns a 4-message agent protocol
                       into theorem-node governance ceremonies.

theorem-bus-client     Typed Rust client for the bus WebSocket protocol.

theorem-gateway        HTTP sidecar over the bus for external systems that do
                       not speak the native bus protocol.

theorem-supervisor     Governed process supervisor. Provisions entities,
                       registers bus sessions, spawns and monitors agents.

theorem-govagent       Governed autonomous worker that polls events, calls
                       models, and executes work through the runtime.

theorem-dashboard      Monitoring dashboard proxy and observability surface.
```

**Dependency direction:** `theorem-kernel` depends on nothing in the workspace.
`theorem-storage-sqlite` and `theorem-witness-storage` depend on
`theorem-kernel` and `theorem-witness` respectively. `theorem-node` is the
canonical runtime. Ring 3 crates (`theorem-bus`, `theorem-gateway`,
`theorem-supervisor`, `theorem-govagent`) sit above it and consume its
authority surface.

## Runtime Boundaries

Theorem has four practical runtime layers:

1. **Ring 0: Kernel**
   `theorem-kernel` decides whether a governance mutation is accepted and
   emits a sealed batch.
2. **Persistence and witnessing**
   `theorem-storage-sqlite`, `theorem-witness`, and
   `theorem-witness-storage` make accepted governance durable and externally
   verifiable.
3. **Canonical runtime surface**
   `theorem-node` is the supported authority surface for bootstrap, audit,
   verification, admission, capability issuance, export, and rebuild.
4. **Ring 3 governed execution**
   `theorem-bus`, `theorem-gateway`, `theorem-supervisor`, and
   `theorem-govagent` are adapters and governed consumers of theorem-issued
   authority, not alternate sources of truth.

The current OpenAPI file documents the stable public subset of the node HTTP
API. The remaining authority routes stay on the node, but they are explicitly
classified so external integrations do not mistake recovery or adapter plumbing
for stable public contract.

## Authority Surface Contract

`theorem-node` is the canonical authority surface. Ring 3 crates consume this
surface; they do not define competing authority semantics.

### Stable Public Authority Routes

These are the supported theorem-native authority routes for external
integrations and Ring 3 consumers. This is the contract the root `openapi.yaml`
is allowed to promise.

| Area | Routes | Notes |
|------|--------|-------|
| Intent and execution | `POST /declare-intent`, `POST /submit-act` | Canonical two-step execution path |
| Governance mutation | `POST /establish-entity`, `POST /create-scope`, `POST /amend-parameter`, `POST /revoke-authorization`, `POST /submit-proof` | Governance writes still flow through theorem-native enforcement |
| Authority queries | `GET /authorizations`, `GET /scope-contains`, `GET /governance-events` | Read-only authority inspection |
| Release and admission | `POST /admit-artifact`, `POST /revoke-artifact`, `POST /admit-action`, `POST /revoke-action-admission`, `POST /verify-release` | Canonical admission and release-verification path |
| Execution capability lifecycle | `POST /issue-execution-capability`, `POST /validate-execution-capability`, `POST /revoke-execution-capability`, `POST /redeem-execution-capability` | Ring 3 runtimes must consume theorem-issued capability state here |
| Authority records | `POST /record-trust`, `POST /revoke-trust`, `POST /record-fork` | Trust and fork records are theorem-native governance records and can be revoked explicitly |

### Operator/Admin Routes

These routes are supported for operators and recovery workflows, but they are
not part of the stable public integration contract.

| Route | Why it is operator/admin |
|-------|--------------------------|
| `GET /export-state` | Full authority snapshot for diagnostics, bootstrap helpers, and recovery tooling |
| `GET /rebuild-authority-state` | Projection rebuild from governance events |
| `POST /restore-authority-state` | Authority-table restore from governance events |
| `GET /authority-lineage/execution-capability` | Operator explanation surface for theorem-issued execution capability causality |
| `GET /governance-context` | Minimal authority read model for Ring 3 path generation |
| `GET /trust-state` | Canonical trust-state and trust-history query for an entity in scope |
| `GET /runtime-eligibility/artifact` | Canonical trust-aware runtime eligibility read model for artifacts |
| `GET /govagent/runtime-context` | Typed deterministic govagent runtime context |
| `GET /govagent/prompt-context` | Bounded govagent prompt context without full authority export |
| `POST /retire-entity` | Lifecycle control for operator-run entity retirement |

HTTP access to these routes is bearer-token protected where applicable. Their
existence does not make them stable public APIs.

### Adapter-Internal Routes

These routes exist to support the current Ring 3 bus and verified-drift
pipeline. They are not stable public authority APIs and should not be treated
as long-term external integration points.

| Route | Internal consumer |
|-------|-------------------|
| `POST /commit-intent-hash` | theorem-bus intent-commitment flow |
| `POST /refresh-commitment-token` | theorem-bus commitment refresh |
| `POST /register-measurement-method` | drift-measurement bootstrap/plumbing |
| `POST /witness/commit-intent` | witness-backed intent commitment |
| `POST /witness/commit-outcome` | witness-backed outcome commitment |
| `POST /witness/publish-drift-attestation` | verified-drift witness publication |

The witness replication endpoints `POST /witness/publish`, `GET /witness/entries`,
and `GET /witness/digest` are part of the witness subsystem, not the authority
contract.

## Enforcement Rules (E1-E16)

| Rule | Protects | Phase |
|------|----------|-------|
| E1 | **Act completeness** -- authorization present, intents satisfy INTENT_CONTENT_MINIMUM (objective, success_criteria, constraints, scope_rationale non-empty; paths_not_taken >= 2 alternatives), scope resolves, executing entity exists, resource consumption coverage | Input (hard reject) |
| E2 | **Authorization validity** -- active/unexpired/unrevoked status, full chain traversal to bootstrap, act type within action_vocabulary, mutual authorization detection at any depth, safety assessment rigor when required | Input (hard reject) |
| E3 | **Scope enforcement** -- act scope contained within both executing entity scope and authorization scope; kernel_write scope inherited from parent act | Input (hard reject) |
| E4 | **Intent precedence** -- each intent declared_at + minimum_intent_execution_interval_ms <= act initiated_at | Input (hard reject) |
| E5 | **Drift signal completeness** -- one drift signal per beneficiary entity, confidence in [0.0, 1.0], threshold violations have escalation signal, output gating enforced, measurement method active with non-zero variance | Output |
| E6 | **Governance record self-attribution** -- kernel_write has three-way atomic references (act -> attribution -> attestation), governance_act has authorized_side_effects | Output |
| E7 | **Self-authorization and mutual authorization prevention** -- principal != granted_by, full bidirectional BFS for grant cycles, system entity holds only bootstrap-produced authorizations, system entity authorization irrevocable | Input (hard reject) |
| E8 | **Feedback channel integrity** -- signal type vocabulary per participation_type, signal production window compliance, orphan signal age escalation | Health (degraded) |
| E9 | **Bootstrap integrity** -- bootstrap batch structure: is_bootstrap=true, exactly 2 output authorizations, 2 genesis records (bootstrap_entity + root_scope), system entity, bootstrap entity, root scope, kernel parameters, initial proof obligations for 15 genesis-time properties | Bootstrap only |
| E10 | **Workflow integrity** -- workflow has establishing governance act, initiating act is distinct record category, no step executes while predecessor has unreleased outputs, workflow is active | Input (hard reject) |
| E11 | **Attestation completeness** -- attribution/attestation cross-reference, same kernel_write_act_id, canonical_serialization non-empty, attestation_schema field_mappings coverage, provisional window timing | Output |
| E12 | **Paths not taken completeness** -- minimum 2 alternatives with non-empty description and reason_rejected, non-empty deliberation_summary; delegates substance to E14 | Input (hard reject) |
| E13 | **Audit currency** -- full audit within audit_interval_ms of previous audit; arrears produces escalation | Health (degraded) |
| E14 | **Parameter outcome enforcement and substance verification** -- single parameter per amendment, within declared bounds, no double-amendment within audit_interval_ms, trajectory anomaly detection (50% cumulative in 4x audit_interval), drift_threshold_default supermajority guard; intent substance (tautological objective, completion-pattern indicators, scope rationale similarity, duplicate intent detection); paths-not-taken substance (token overlap, concrete governance state references, distinct descriptions) | Input (hard reject) |
| E15 | **Proof obligation enforcement** -- acyclicity via Kahn's algorithm, expiry check, spec version hash match, submitted proofs verified within one audit cycle, invalidation cascade (full transitive BFS), all 15 required properties have active proof, challenge resolution within one audit cycle | Health (degraded) |
| E16 | **Liveness, identity, infrastructure, degraded mode** -- system entity heartbeat within detection_interval_ms, entity identity attestation present/unexpired/unique binding hash/max 365-day validity, integrity mechanism >= 2 artifact locations with approved algorithm, attestation log append-only/hash-chaining/third-party-verification/3-year retention, max_degraded_window_ms >= audit_interval_ms | Health (degraded) |

**Input vs Health distinction:** Input rule violations (E1-E4, E7, E10, E12, E14) cause the kernel write to return `Err(violations)` -- the act is rejected. Health rule violations (E8, E13, E15, E16) produce escalation signals and set `degraded_flag = true` on the attribution, but the write proceeds. This prevents deadlock: you cannot run an audit to fix E13 if E13 blocks all writes.

## Audit Criteria (C1-C22)

| Criterion | Verifies |
|-----------|----------|
| C1 | **Authorization traceable** -- every non-bootstrap/kernel_write act has authorization that chains to bootstrap with no mutual authorization; safety assessment present when required |
| C2 | **Drift measurable** -- active non-system entities with attributions in window have drift signals with active measurement methods |
| C3 | **Governance record self-attributing** -- three-way reference integrity (act -> attribution -> attestation), kernel_write_act_id consistency, genesis record references resolve |
| C4 | **Downstream participants represented** -- beneficiary_entity_ids non-empty for execution/governance acts, attestation paths_not_taken_summary present |
| C5 | **Feedback loop closes** -- no orphan signals (status=pending, age > orphan_signal_max_age_ms) |
| C6 | **Bootstrap traceable** -- bootstrap governance_act exists with is_bootstrap=true, 2 genesis records, bootstrap entity, system entity, kernel parameters, root scope, >= 2 bootstrap authorizations |
| C7 | **Scope enforced at every layer** -- scope DAG is acyclic, act scope contained within authorization scope |
| C8 | **Intent precedes execution** -- execution/governance acts have non-empty intent_ids, each intent exists, declared_at precedes initiated_at |
| C9 | **Non-participating entities detected** -- active entities have produced signals within signal_production_window_ms |
| C10 | **Workflows auditable as units** -- acts with workflow_id reference existing workflows with valid status |
| C11 | **Attestations externally verifiable** -- attestations not perpetually provisional (within attestation_provisional_window_ms), external_schema_representation non-empty |
| C12 | **Resource consumption accounted** -- acts with resource_limits authorization have resource_consumption records covering resource_type_vocabulary |
| C13 | **Paths not taken recorded** -- non-bootstrap governance_acts have paths_not_taken with >= 2 alternatives; intents have conforming paths_not_taken |
| C14 | **Audit execution governed and current** -- audit_execution exists and most recent is within audit_interval_ms |
| C15 | **Parameter values within declared bounds** -- time-based kernel parameters within parameter_bounds, no unresolved parameter_trajectory_anomaly escalation |
| C16 | **Every entity bound to a verified principal** -- active non-bootstrap/system entities have identity_attestation_ref, not expired, no duplicate principal_binding_hash |
| C17 | **System entity provably alive** -- system entity exists and is active, liveness model dependency_declaration excludes system entity, no unresolved system_entity_liveness_failure escalation |
| C18 | **Governance substance distinguishable from governance theater** -- no duplicate intent objective hashes from same entity in window, no unresolved governance_uniformity_anomaly escalation |
| C19 | **External verification infrastructure governed and healthy** -- integrity mechanisms have >= 2 artifact locations, no unresolved verification_artifact_inconsistency/attestation_log_endpoint_failure/ungoverned_infrastructure_change escalations |
| C20 | **All properties proven** -- every required property has active proof (Active/Verified/Reaffirmed status), spec version hash matches, dependency graph acyclic, no unresolved proof_challenge signals |
| C21 | **Deterministic verifier passed** -- most recent audit_execution exists and was run within current audit cycle |
| C22 | **Degraded-mode records reconciled** -- degraded attributions since previous audit have attestations transitioned from Provisional to Verifiable |

## SealedBatch Pattern

`SealedBatch` is a compile-time guarantee against ungoverned writes to the database.

```
SealedBatch {
    pub(super) inner: RecordBatch,   // Only accessible within kernel module
}
```

- No public constructor -- the only way to obtain a `SealedBatch` is through `execute_kernel_write`, `execute_parameter_amendment`, or `execute_bootstrap`.
- No public mutable fields -- only read-only accessor methods.
- `StorageAdapter::apply_batch` accepts only `SealedBatch`, not `RecordBatch`.
- `SealedBatch::into_inner()` consumes the batch, returning the `RecordBatch` for storage implementations.

This means storage code physically cannot write records that have not passed through kernel enforcement. A `#[cfg(any(test, feature = "test-support"))]` escape hatch (`SealedBatch::for_testing`) exists for integration tests only.

The `RecordBatch` itself is a bag-of-optionals containing all primitive types (genesis records, kernel parameters, scopes, entities, signals, proof obligations, intents, acts, authorizations, attributions, attestations, drift signals, resource consumptions, workflows, governance acts, audit executions, measurement methods, integrity mechanisms, attestation logs, attestation schemas).

## v0.1 Bridge (Transitional)

The v0.1 bridge enables migration from a prior governance system version. When active, authorization chain verification (E2) delegates chain-of-custody checks to the v0.1 authorization kernel while still enforcing all governance-specific checks: mutual authorization detection, safety assessment rigor, and granting authority verification.

**Operator impact:**

- The bridge is transparent — no special operator configuration required
- `/submit-act` accepts the same request format regardless of bridge status
- Bridge authorization failures surface as E2 violations in audit findings
- The bridge is intended to be phased out; operators should plan migration to native v1.0 authorization chains

**Database:** Bridge state is stored in V002 schema tables (7 additional tables in `theorem.db`).

## ID Generation

All IDs are UUID v7 (time-ordered, globally unique) wrapped in branded newtypes via the `branded_id!` macro. The `IdGenerator` trait abstracts ID creation; the canonical implementation is `UuidIdGenerator`. This enables deterministic ID generation in tests while using real UUIDs in production.
