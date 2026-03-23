# Glossary

Terms used in the Theorem governance kernel, ordered alphabetically.

---

**Act**
A record representing a single governance operation. Four types: `execution_act` (an entity executing work), `governance_act` (a governance decision), `kernel_write` (the system entity's internal write for attribution/attestation), `bootstrap` (the genesis operation). Defined in `theorem-kernel/src/primitives/act.rs`.

**AttestationLog**
A governed infrastructure record declaring the configuration of the append-only attestation log. Contains `log_endpoint`, `verification_method`, and `technology_requirements` (append_only, cryptographic_hash_chaining, third_party_verification, minimum_retention_years). The actual log implementation is the witness network.

**AttestationSchema**
A governed record defining the expected structure of attestation `external_schema_representation` via `field_mappings`. E11 validates that attestations conform to the schema.

**Attestation**
A record produced atomically with an Attribution during kernel write. Contains: `canonical_serialization` (deterministic JSON per RFC 8785), `integrity_proof`, `external_schema_representation`, summaries (drift, resource consumption, signals, paths_not_taken, workflow, proof obligations), and status (Provisional or Verifiable). Attestations begin as Provisional and transition to Verifiable once published to the witness network and confirmed.

**Attribution**
A record linking an act to its executing entity, authorizing entity, beneficiary entities, and all side-effect records (drift signals, resource consumption, escalation signals). Produced atomically with an Attestation during kernel write. Contains `degraded_flag` indicating whether health violations were present during the write.

**AuditCriterion**
One of 22 criteria (C1-C22) evaluated during a full audit. Each is a pure function over governance state and an audit window. All 22 criteria run even if earlier ones fail (no short-circuit).

**AuditExecution**
A record documenting that a full audit was performed, including the governance_act_id that authorized it, proof verification results, and criterion results.

**Authorization**
A record granting an entity the right to perform specific act types within a scope. Contains `action_vocabulary` (which act types are permitted), `conditions` (resource limits, safety assessment requirement, risk classification, workflow_only), `status` (Active, Revoked, Expired, Consumed). Authorization chains trace back to bootstrap through `parent_authorization_id`.

**Bootstrap**
The one-time initialization of a governance instance. Creates genesis records, entities (system + bootstrap), root scope, authorizations, kernel parameters, and initial proof obligations. Governed by E9 (bootstrap integrity). Cannot be re-run on an initialized instance.

**branded_id**
A macro that wraps UUID values in strongly-typed newtypes (e.g., `EntityId`, `ActId`, `ScopeId`). Prevents accidental mixing of ID types at compile time. All IDs use UUID v7 (time-ordered).

**CriterionResult**
The result of evaluating a single audit criterion. Contains `criterion` (C1-C22), `satisfied` (bool), and `findings` (Vec<String> of human-readable issues).

**DegradedMode**
A state where the governance instance is operating with known health violations (E8, E13, E15, E16 failures). Acts are still processed and enforced, but attributions carry `degraded_flag = true` and escalation signals are produced. C22 requires degraded records to be reconciled by the next audit.

**DriftSignal**
A measurement record produced during kernel write for each beneficiary entity. Contains `distance` (float, how far the outcome drifted from intent), `confidence` [0.0, 1.0], `within_threshold` (compared to `drift_threshold_default`), `outputs_released` (gating flag -- false when threshold exceeded), and `method_id` (FK to MeasurementMethod).

**DriftThreshold**
The `drift_threshold_default` kernel parameter. When a drift signal's distance exceeds this threshold, outputs are gated (`outputs_released = false`) and an escalation signal is produced. Amending this parameter requires supermajority (2/3 of active entities).

**EnforcementResult**
The return type of all enforcement rules (E1-E16). Contains `accepted` (bool) and `violations` (Vec<String>). When `accepted` is false, the violations explain why.

**Entity**
A participant in the governance system. Has `participation_type` (authorizing, executing, participating, observing), `status` (active, suspended, retired), `scope_id`, and `identity_attestation_ref`. Special entities: system entity (is_system=true, produces kernel_write acts) and bootstrap entity (is_bootstrap=true, produced during bootstrap).

**EscalationType**
A vocabulary of escalation signal types: `drift_threshold_exceeded`, `resource_limit_exceeded`, `non_participating_entity`, `orphan_signal_aged`, `attestation_provisional_expired`, `audit_in_arrears`, `parameter_trajectory_anomaly`, `duplicate_principal_detected`, `identity_attestation_expiring`, `system_entity_liveness_failure`, `measurement_invariance_detected`, `governance_uniformity_anomaly`, `verification_artifact_inconsistency`, `attestation_log_endpoint_failure`, `ungoverned_infrastructure_change`, `proof_expired`, `proof_invalidated`, `proof_dependency_cascade`, `degraded_mode_entered`, `degraded_mode_timeout`, `duplicate_intent_content`, `insufficient_alternative_substance`, `insufficient_safety_coverage`, `system_health_violation`.

**Genesis**
A write-once record created during bootstrap. Exactly two per instance: one with `primitive_type = bootstrap_entity` and one with `primitive_type = root_scope`. Contains `specification_version_hash` (SHA-256 of the governance spec) and `hash_algorithm`. Cannot be amended or deleted.

**GovernanceAct**
A governance decision record. Contains `governance_type` (amendment, audit, override, establishment, deprecation), `is_bootstrap`, `output_authorization_ids`, `participating_entity_ids`, `authorized_side_effects`, and `paths_not_taken`. The bootstrap governance act is the root of all authorization chains.

**GovernanceSideEffects**
Declared mutations accompanying a kernel write. Contains entities, authorizations, governance_acts, and proof_obligations that the caller intends to create. The kernel validates all side effects before sealing -- no post-enforcement mutation is possible.

**IdentityAttestationRef**
A sub-structure on Entity containing `principal_binding_hash` (must be unique among active entities), `issued_at`, `expiry` (max 365 days from issued_at), and verification metadata. E16 enforces identity attestation requirements.

**IntegrityMechanism**
A governed infrastructure record declaring the cryptographic signing mechanism for attestations. Contains `algorithm` (must be in `approved_algorithms`), `public_verification_artifact`, and `public_verification_artifact_locations` (minimum 2 independent locations per P13).

**Intent**
A declaration of what an entity intends to do, submitted before execution. Contains `objective`, `success_criteria` (with measurable indicators), `constraints`, `scope_rationale`, `paths_not_taken` (minimum 2 alternatives), and `beneficiary_intent_snapshots`. E14 verifies substance (tautological detection, completion-pattern rejection, duplicate detection).

**KernelParameters**
The single governed configuration record per instance. Contains all time-based parameters (audit_interval_ms, etc.), vocabularies, infrastructure references, and model declarations. Amendable only through governance acts, one parameter at a time. See [Configuration Reference](03-configuration-reference.md).

**KernelWrite**
An act of type `kernel_write`, produced by the system entity during every kernel write operation. Links to the parent act, the produced attribution, and the produced attestation. Forms the three-way atomic reference validated by E6.

**MeasurementMethod**
A governed record declaring how drift is measured. Contains `test_vectors` for discriminative power verification and `status` (active/deprecated). E5 validates that measurement methods are active and produce non-invariant output (variance check on last 20 measurements, CV > 0.01).

**ParameterBounds**
A map of `{ parameter_name -> { floor: i64, ceiling: i64 } }` in KernelParameters. E14 rejects amendments that would set a value outside the declared bounds. The bounds themselves are amendable.

**PathsNotTaken**
A structured record of alternatives considered and rejected at a governance decision point. Required on every Intent and non-bootstrap GovernanceAct. Must contain at least 2 alternatives, each with a description (referencing concrete governance state) and reason_rejected. E12 validates completeness; E14 validates substance (token overlap, negation detection, distinct descriptions).

**PathAlternative**
A single alternative in PathsNotTaken. Contains `description` (must reference concrete governance state -- entity IDs, scope IDs, or parameter names), `reason_rejected` (must not restate the objective), and `signal_ids_that_influenced`.

**ProofObligation**
A record declaring that a specific property must be proven. Contains `property_name` (e.g., "PROP-PARAM-001"), `status` (submitted, verified, active, challenged, reaffirmed, invalidated, expired), `depends_on_proof_ids` (dependency graph), `specification_version_hash`, `expires_at`, and `challenge_signal_ids`. E15 enforces acyclicity, expiry, cascade invalidation, and deployment gating.

**Provisional**
The initial status of an Attestation after creation. Transitions to Verifiable once published to the witness network and confirmed. E11 enforces the provisional window timing (attestation_provisional_window_ms). C22 requires degraded-mode Provisional attestations to be reconciled.

**RecordBatch**
A bag-of-optionals containing all primitive types. Both bootstrap and kernel_write produce RecordBatch instances. The caller applies the entire batch atomically. Never directly exposed to storage -- wrapped in SealedBatch first.

**ResourceConsumption**
A record documenting resources consumed by an act. Contains entries per resource type: resource_type, quantity, unit, measurement_method_id, within_limits, escalation_signal_id (when limits exceeded).

**Scope**
A governance boundary. Contains `domains`, `contexts`, `child_scope_ids` (forming a DAG), and `is_root`. E3 enforces scope containment: act scope must be within both entity scope and authorization scope. C7 validates the scope DAG is acyclic.

**SealedBatch**
A RecordBatch that has passed through kernel enforcement (E1-E16). Has no public constructor, no public mutable fields, and no mutation methods. The only way to obtain one is through `execute_kernel_write`, `execute_parameter_amendment`, or `execute_bootstrap`. StorageAdapter accepts only SealedBatch, providing a compile-time guarantee against ungoverned writes.

**Signal**
A communication record from an entity. Types: `feedback`, `challenge`, `consent`, `dissent`, `escalation`, `acknowledgment`, `safety_assessment`, `proof_challenge`. Content is discriminated by type. E8 enforces signal type vocabulary per participation_type (observing entities can only produce feedback and proof_challenge).

**SubstrateTrustDeclaration**
A KernelParameters sub-structure declaring the trust properties of the physical substrate: TEE presence, boot integrity chain (measured_boot, secure_boot, none), clock synchronization (ntp_authenticated, ptp, gps, none), and crypto implementation provenance. Verified by physical audit (PA-008).

**Verifiable**
The final status of an Attestation after publication to the witness network. Indicates the attestation's canonical serialization and integrity proof have been independently recorded by at least one witness peer.

**Workflow**
A sequence of related acts governed as a unit. Contains `established_via_governance_act_id`, `initiating_act_id`, `act_ids`, and `status` (active, completed, failed). E10 enforces that establishing and initiating are distinct record categories, and that no step executes while predecessor outputs are unreleased.

**WitnessEntry**
A single entry in the append-only witness log. Contains `attestation_id`, `canonical_serialization`, `integrity_proof`, `content_hash` (SHA-256 of canonical_serialization), `prev_hash` (hash chain link), `sequence_number`, and `received_at`.

**WitnessNode**
The orchestrator for receiving attestation publications, appending to the witness log, producing digests for peer verification, and verifying peer entries. Design follows "mutual witnessing, not consensus" -- witnesses verify each other's logs but do not agree on ordering.
