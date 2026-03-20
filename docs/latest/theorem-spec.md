# Trust Kernel — Specification

*Property-proven. Any instance must satisfy all propositions and prove all properties.*
*The spec defines properties. Implementations prove their choices satisfy those properties. The audit verifies proofs. The deterministic layer checks the math.*

*This specification is self-contained.*

*This specification describes requirements for conforming implementations. It does not warrant that any specific implementation is correct, secure, or fit for any purpose. Conformance is determined by the verification procedures defined herein, not by this document's claims. See the Trust Boundary section for declared scope limitations.*

*Specification hash: Computed externally. See theorem-spec.sha256.*

---

## Layer 0 — Definitions

**Governance record.** The set of kernel-maintained output primitives: AUTHORIZATION (lifecycle transitions), ATTRIBUTION, ATTESTATION, DRIFT_SIGNAL, RESOURCE_CONSUMPTION, SIGNAL, KERNEL_PARAMETERS, MEASUREMENT_METHOD, RESOURCE_MEASUREMENT_METHOD, INTEGRITY_MECHANISM, ATTESTATION_LOG, ATTESTATION_SCHEMA, SCOPE (amendments), ENTITY (status transitions), WORKFLOW (status transitions), GOVERNANCE_ACT (status transitions), AUDIT_EXECUTION, GENESIS, and PROOF_OBLIGATION. Every write to any record in this set is governed by P7.

**Kernel-authored primitives.** Records the kernel writes: ATTRIBUTION, ATTESTATION, DRIFT_SIGNAL, RESOURCE_CONSUMPTION, and system-entity-produced SIGNALs (escalation and acknowledgment types). These are written by the kernel in response to presented acts or observed governance state.

**Entity-authored primitives.** Records presented to the kernel by participating entities as inputs: ACT, INTENT, WORKFLOW, and participant-authored SIGNALs (feedback, challenge, consent, dissent, safety_assessment). The kernel validates these inputs but does not originate them. They are governance inputs, not governance record entries. Entity-authored primitives are subject to kernel validation (E1–E16) but the act of writing them is not itself a kernel_write operation. Kernel validation of entity-authored inputs includes acceptance or rejection — the kernel's validation response (e.g., ACT status set to `rejected` by E1) is intrinsic to the validation mechanism, not a separate write operation. System-entity SIGNALs (escalation, acknowledgment) are kernel-authored, not entity-authored.

**Governance output primitives.** Records produced as outputs of GOVERNANCE_ACTs: AUTHORIZATION (created via `output_authorization_ids`) and ENTITY (created via `established_via_act_id`). These are governance record entries created as authorized side effects of GOVERNANCE_ACTs, with instance-level recording in the governing act's ATTRIBUTION `side_effect_record_ids`. They are neither kernel-authored (the kernel does not originate them) nor entity-authored inputs (entities do not present them to the kernel for validation). They are the products of governed governance decisions.

**Governance infrastructure primitives.** Records that configure and govern the kernel itself: KERNEL_PARAMETERS, MEASUREMENT_METHOD, RESOURCE_MEASUREMENT_METHOD, INTEGRITY_MECHANISM, ATTESTATION_LOG, ATTESTATION_SCHEMA. These are established and amended through governance_write acts, which are entity-authored and kernel-validated. The resulting records are part of the governance record.

**Monitoring kernel_write.** A kernel_write ACT executed by the system entity with no parent_act_id, whose purpose is to record a governance state observation and produce escalation or acknowledgment signals as authorized side effects. Monitoring kernel_write acts are triggered by governance state conditions: entity non-participation (E8), orphan signal aging (E8), attestation provisional expiry (E11), audit arrears (E13), proof expiry (E15), and liveness failure (E16). They produce ATTRIBUTION and ATTESTATION for the monitoring observation. Their scope is the system entity's scope (root). Act-triggered escalation signals (drift threshold exceeded, resource limit exceeded) and acknowledgment signals are instead authorized side effects of the kernel_write recording the triggering act or resolving GOVERNANCE_ACT, not monitoring kernel_writes.

**Deployment authorization.** An AUTHORIZATION whose scope includes one or more domains declared as deployment domains in the SCOPE's `deployment_domains` field. Deployment domains are instance-defined. An AUTHORIZATION covering deployment domains requires `conditions.risk_classification`.

**Property.** A named, falsifiable claim about the governance system's behavior. A property describes what must be true, not how to make it true. Properties are the fundamental unit of conformance — every instance-defined choice must satisfy at least one property.

**Proof obligation.** A requirement that the implementation demonstrate — not merely declare — that a named property holds. Every instance-defined choice in the specification is a proof obligation target. The implementation chooses its mechanism; the proof obligation requires it to prove the mechanism works.

**Proof.** A structured argument that a property holds for a given implementation choice. Proofs are governance records — they are attested, immutable, and externally verifiable. A proof that cannot be independently verified is not a proof. Proofs have types: formal (machine-verifiable or logically rigorous), empirical (evidence from testing or measurement), and operational (evidence from demonstrated procedure).

**Verification procedure.** A defined method for checking whether a proof demonstrates what it claims. Verification procedures are part of the specification — the implementation does not choose how its proofs are checked. Verification procedures are either deterministic (hash comparison, structural check, arithmetic) or judgmental (requires analytical assessment). Deterministic procedures terminate the verification regress.

**Degraded mode.** A state in which one or more governance infrastructure components (attestation log, integrity mechanism, measurement method, external verification endpoint) are temporarily unavailable. The kernel continues to operate with reduced assurance, producing governance records that are flagged as `degraded` and subject to reconciliation when infrastructure is restored. Degraded mode is not a suspension of governance — it is governance with declared, flagged, and reconciled limitations.

---

## Layer 1 — Propositions

### P1 — Every act is attributed
No act occurs in the system without a complete, verifiable record of who authorized it, who executed it, and whose interests it was intended to serve. An act with incomplete attribution is indistinguishable from an unauthorized act and must be treated as one. The sole exception is kernel_write acts, whose existence in the governance record is their own attribution — established by the three-way atomic reference between the kernel_write ACT, the ATTRIBUTION record it writes, and the ATTESTATION record it produces. The three-way atomic operation is self-authorizing within its atomic boundary — the three records are co-created, none preceding the others in authorization order. Execution-act and governance-act ATTRIBUTIONs are both governed by this rule. No ATTRIBUTION is created outside the kernel_write mechanism. Every ATTRIBUTION is externally attestable. The only records exempt from attribution are GENESIS records — write-once, content-addressed, bounded to exactly two records per instance, referenced by the bootstrap ATTRIBUTION. GENESIS is the only named, bounded exception to P1.

### P2 — Every act of governance is itself governed
The audit is a governed act. **Regress termination:** The audit-governs-audit cycle is broken by the deterministic verifier (Layer 7a), which is a script — not a governed act — that checks the math independently. Every execution of Layer 4 audit criteria produces an AUDIT_EXECUTION record attributed to the auditing entity, authorized under a valid AUTHORIZATION, subject to the same kernel rules as any other governance act. An instance that runs unattributed audits is not conforming.

### P3 — Every entity is bidirectional
Every entity in the system both produces signals that influence governed acts and receives outputs that those acts produce. There are no pure senders and no pure receivers. An entity that only receives outputs and produces no signals within a governed time window is not participating in governance — it is being acted upon without standing. Observing entities are a limited participation type — they receive outputs and produce feedback and consent signals only. The system entity is a monitoring participant: it produces escalation and acknowledgment signals and receives inputs through its defined observation channel — the governance state stream consisting of DRIFT_SIGNAL records, RESOURCE_CONSUMPTION records, SIGNAL ages, and ATTESTATION statuses. Observation of this state stream is the system entity's structured input channel. It is not a beneficiary of governed execution acts in the standard sense, but it is bidirectional: its signals influence governance decisions, and the governance state it observes is the output it receives. System entity observation is verified by PROP-LIVENESS-004.

### P4 — Authorization is traceable to its source
Every authorization record traces to a governance act that sanctioned it. The chain does not terminate at an unchecked assertion. Non-system authorization chains terminate at the root AUTHORIZATION produced by the bootstrap GOVERNANCE_ACT. The system entity AUTHORIZATION is a distinct bootstrap-produced authorization — also terminal at bootstrap, not the root. Both bootstrap-produced authorizations are exempt from the prior-authorization requirement — this is the bootstrap authorization exemption. Mutual authorization at any depth is prohibited.

### P5 — Intent is declared before execution, and its content is governed
Every governed act is executed against a declared intent that existed before the act began. Intent and execution may not occur in the same transaction. The minimum interval is a governed instance parameter, strictly greater than zero. Intent content is not free-form — every INTENT must satisfy the kernel's minimum content schema and the substance verification requirements of the applicable property (PROP-SUBSTANCE-001). Intent that cannot satisfy the minimum schema is not intent — it is an assertion, and it does not authorize execution. INTENTs are entity-authored primitives presented to the kernel for validation before any referencing ACT may proceed.

### P6 — Drift is measurable at every act, and violations gate outputs
Every act produces a drift signal for every beneficiary entity — except kernel_write acts, which have no beneficiary entities and are explicitly exempt. Bootstrap acts are exempt. Partial drift coverage for non-exempt acts is a violation equivalent to no drift coverage. Drift is computed at execution time and may not be reconstructed afterward. A drift signal that exceeds the declared threshold produces an escalation signal from the system entity. The act's outputs may not be consumed by any downstream act until the escalation is resolved. The `outputs_released` transition from false to true is an authorized side effect of the GOVERNANCE_ACT that resolves the escalation — it is not an unattributed field write — and its instance-level recording is captured in the resolving GOVERNANCE_ACT's ATTRIBUTION via `released_drift_signal_ids`. An act whose drift escalation is unresolved is not complete. Every measurement method must satisfy the discriminative power property (PROP-SUBSTANCE-003).

### P7 — The governance record is immutable and self-attributing
Every write to the governance record is: executed via kernel_write ACT (for ATTRIBUTION and ATTESTATION), a type-authorized side effect of a governing ACT declared in that ACT's `authorized_side_effects` with instance-level recording in the governing act's ATTRIBUTION, a GENESIS record (the only explicitly bounded exception — write-once, content-addressed, exactly two per instance), or a kernel-validated entity-authored primitive (ACT, INTENT, WORKFLOW, and participant-authored SIGNALs — inputs presented to the kernel, validated but not kernel-written; kernel validation responses such as ACT rejection are intrinsic to this mechanism). Post-production writes to ATTESTATION fields are authorized by the kernel_write ACT's own `authorized_side_effects`. The kernel_write ACT is written within the three-way atomic boundary — its creation is self-authorizing within that boundary. The authorization regress terminates at the atomic boundary. GOVERNANCE_ACT status transitions to `completed` are authorized side effects of the kernel_write ACT that produces the GOVERNANCE_ACT's ATTRIBUTION — declared in that kernel_write's `authorized_side_effects` and recorded in the ATTRIBUTION. A governance system writable outside these mechanisms is not a governance system.

### P8 — Every entity's participation is scoped
An entity's authorization to act is bounded by its declared scope. Scope is a directed acyclic graph — extension requires a sanctioned governance act, cycles are prohibited. Kernel_write scope is inherited from the parent act. An act exceeding the executing entity's scope is a violation regardless of authorization.

### P9 — The governance loop closes
Every signal is either consumed, acknowledged, superseded, or ignored — never both consumed and acknowledged. Ignored and superseded are authorized status transitions, not unilateral disposals — they require type-level authorization in a GOVERNANCE_ACT. Orphan signals must be resolved before the instance-declared maximum age. The governance loop includes audit signals — an audit finding that is never consumed or acknowledged is an orphan signal subject to the same resolution rules.

### P10 — No entity governs itself
Self-authorization is invalid. Mutual authorization at any depth is invalid. The system entity may only hold authorizations produced by bootstrap.

### P11 — Downstream participation is first-class
Entities receiving outputs are full participants. Their intent is a valid drift input. Their active signals at execution time are recorded in the attribution. Paths not taken in decisions affecting them are recorded and attested. Paths not taken must satisfy the substance verification requirements of the applicable property (PROP-SUBSTANCE-002).

### P12 — Multi-agent workflows are governed as units
Every workflow is a governed object. Individual act attribution is necessary but insufficient. Every workflow requires an explicit GOVERNANCE_ACT to establish it — standing authorization alone is not sufficient. The establishing GOVERNANCE_ACT and the initiating execution ACT are always distinct records of different categories. The full workflow is auditable as a unit. A workflow step may not execute while any predecessor act has unreleased outputs.

### P13 — Attestations are externally verifiable
Every ATTRIBUTION is expressible as a signed attestation consumable by an independent auditor without instance access. The mechanism is cryptographic, the log is tamper-evident and publicly accessible, the schema is machine-readable — all declared as governed instance parameters. No specific technology is mandated. An independent auditor must be able to verify any attestation using only public artifacts. Signal summaries (type, escalation_type, grade for safety assessments) are included in attestations. Full signal content beyond these summaries is governed but not attested — complete signal content requires instance access. The infrastructure hosting verification artifacts must satisfy the infrastructure governance properties (PROP-INFRA-001 through PROP-INFRA-003).

### P14 — Every instance-defined choice is proven
No mechanism choice delegated to the implementation is accepted on declaration alone. Every instance-defined choice must satisfy at least one named property. The implementation provides a proof. The audit verifies the proof against the property's verification procedure. The deterministic verification layer checks all mechanically verifiable aspects. The proof is a governance record — attested, immutable, externally verifiable. An implementation with any required property in `invalidated` or `expired` status is non-conforming.

### P15 — Governance continues under infrastructure impairment
The governance system does not halt when infrastructure components are temporarily unavailable. Governance operations continue in degraded mode with reduced assurance. Records produced during degraded mode are flagged as `degraded` and subject to mandatory reconciliation when infrastructure is restored. Degraded mode is time-bounded — if infrastructure is not restored within the declared maximum degraded window, the system escalates to liveness failure (PROP-LIVENESS-001 / E16).

---

## Layer 2 — Schema Primitives

#### GENESIS
A write-once, content-addressed record of a pre-governance primitive. Exactly two GENESIS records per instance: one for the bootstrap entity, one for the root SCOPE. These are the only records exempt from P1 and P7. They exist before the first governed act.

Required fields:
- `id` — assigned by the instance at creation, before bootstrap.
- `primitive_type` — vocabulary: bootstrap_entity / root_scope.
- `content` — complete structured record of the primitive in its initial state.
- `hash_algorithm` — the algorithm used to compute `content_hash`. Self-declared in each GENESIS record. Does not depend on KERNEL_PARAMETERS. The instance declares the algorithm at GENESIS creation time. KERNEL_PARAMETERS `genesis_hash_algorithm` must match this field on both GENESIS records. Must meet the minimum algorithm strength floor (PROP-INFRA-002 verification procedure).
- `content_hash` — deterministic hash of `content` using the declared `hash_algorithm`. An independent party verifies by recomputing.
- `created_at`
- `instance_id`
- `specification_version_hash` — SHA-256 of the specification text this instance claims conformance to. Must match a known specification version in the specification provenance chain.

Constraints:
- Exactly two per instance.
- Write-once. No field may be updated after creation.
- `content_hash` recomputable from `content` using `hash_algorithm`. Mismatch — invalid.
- Both GENESIS records referenced in bootstrap ATTRIBUTION `genesis_record_ids`.
- `hash_algorithm` values on both GENESIS records must match KERNEL_PARAMETERS `genesis_hash_algorithm`.
- `specification_version_hash` must reference a valid, content-addressed specification version.

---

#### KERNEL_PARAMETERS
A single governed record per instance. Complete before the instance accepts any acts.

Required fields:
- `id`
- `minimum_intent_execution_interval_ms` — strictly greater than zero.
- `orphan_signal_max_age_ms` — floor: `audit_interval_ms × 2`. Ceiling: instance-declared.
- `signal_production_window_ms` — floor: `minimum_intent_execution_interval_ms × 3`. Ceiling: instance-declared.
- `drift_threshold_default` — declared at GENESIS, immutable without supermajority.
- `default_measurement_method_id` — FK to MEASUREMENT_METHOD.
- `default_resource_measurement_method_ids` — map of { resource_type → method_id }, FK to RESOURCE_MEASUREMENT_METHOD per type. Must cover every type in `resource_type_vocabulary`.
- `attestation_integrity_mechanism_id` — FK to INTEGRITY_MECHANISM.
- `attestation_log_id` — FK to ATTESTATION_LOG.
- `attestation_schema_id` — FK to ATTESTATION_SCHEMA.
- `attestation_provisional_window_ms` — floor: 30000 (30 seconds). Ceiling: instance-declared.
- `resource_type_vocabulary` — governed vocabulary of resource types. Non-empty.
- `cost_unit_vocabulary` — governed vocabulary of cost units.
- `genesis_hash_algorithm` — must match `hash_algorithm` on both GENESIS records.
- `hazard_category_vocabulary` — governed vocabulary of hazard categories used in SAFETY_ASSESSMENT_CONTENT. Must be declared before any safety_assessment signal is produced.
- `audit_interval_ms` — maximum interval between consecutive full audit runs. Strictly greater than zero.
- `established_via_act_id` — FK to bootstrap GOVERNANCE_ACT.
- `last_amended_via_act_id` — FK to most recent amending GOVERNANCE_ACT. Nullable until first amendment.
- `identity_binding_model` — structured declaration specifying: binding_type (cryptographic_key / institutional_attestation / biometric / composite), uniqueness_guarantee, verification_procedure. Set at bootstrap. Amendable only via supermajority governance_write.
- `system_entity_liveness_model` — structured declaration specifying: mechanism_type (heartbeat / watchdog / external_endpoint), detection_interval_ms (≤ audit_interval_ms / 4), dependency_declaration (must not include system entity). Set at bootstrap.
- `parameter_bounds` — structured record of { parameter_name → { floor, ceiling } } for every amendable time-based parameter. Floors must meet normative minimums declared in this specification.
- `specification_version_hash` — SHA-256 of the specification text this instance claims conformance to. Must match GENESIS `specification_version_hash`.
- `max_degraded_window_ms` — maximum duration the kernel may operate in degraded mode before escalating to liveness failure. Strictly greater than zero. Floor: `audit_interval_ms`.
- `proof_expiry_max_cycles` — maximum number of audit cycles before a proof must be renewed. Default: 2. Minimum: 1.
- `minimum_monitoring_observations_per_cycle` — minimum number of monitoring kernel_writes the system entity must produce per audit cycle, even when no escalation-worthy state exists. Default: 3. Floor: 1.
- `approved_algorithms` — governed vocabulary of cryptographic algorithms meeting the minimum security level (128-bit security, 256-bit hash output minimum).
- `substrate_trust_declaration` — structured declaration specifying: tee_present (boolean), tee_mechanism (if present), boot_integrity_chain (measured_boot / secure_boot / none), clock_synchronization (ntp_authenticated / ptp / gps / none), crypto_implementation_provenance (library, version, validation_status). Set at bootstrap.

Constraints:
- Exactly one per instance.
- `minimum_intent_execution_interval_ms` strictly greater than zero.
- All four attestation parameters non-null before instance accepts any acts.
- `resource_type_vocabulary` non-empty before instance accepts any acts.
- `default_resource_measurement_method_ids` must cover all declared resource types.
- `audit_interval_ms` non-null and strictly greater than zero.
- `genesis_hash_algorithm` must match GENESIS records' `hash_algorithm` field.
- `hazard_category_vocabulary` non-empty before any safety_assessment signal is produced.
- May only be amended through a GOVERNANCE_ACT. Single parameter per amendment (E14).
- All time-based parameters must be within their declared floor and ceiling at all times (E14).
- `identity_binding_model` amendable only via supermajority governance_write.
- `system_entity_liveness_model` dependency_declaration must not include system entity id.
- `specification_version_hash` must match GENESIS `specification_version_hash` until a governed specification amendment updates it.
- `approved_algorithms` must reference algorithms meeting the minimum security level.

---

#### INTEGRITY_MECHANISM
A governed declaration of the cryptographic signing mechanism.

Required fields:
- `id`
- `name`
- `description`
- `public_verification_artifact` — independently accessible without instance access.
- `public_verification_artifact_locations` — array of at least two independent hosting locations with independent access-control domains. "Independent" means different DNS registrars, different hosting providers, or no shared administrative credentials.
- `signing_key_ref` — internal reference. Not exposed in attestations.
- `established_via_act_id` — FK to GOVERNANCE_ACT.
- `status` — active / deprecated.
- `infrastructure_security` — structured declaration of hosting security properties: transport_security (TLS version minimum, certificate pinning policy), dns_security (DNSSEC required or compensating control), availability_target (uptime percentage, maximum acceptable downtime), access_control_model (who can modify hosted artifacts).
- `algorithm` — must be in KERNEL_PARAMETERS `approved_algorithms`.

Constraints:
- Must be cryptographic.
- `public_verification_artifact` independently accessible from all declared locations.
- Content at all declared locations must be identical (deterministic verification).
- `algorithm` must meet minimum security level (128-bit security).
- Established or deprecated only through a GOVERNANCE_ACT with `act_type = governance_write`.
- Infrastructure changes (DNS, TLS, hosting, access control) must be preceded by a governance_write act.

---

#### ATTESTATION_LOG
A governed declaration of the tamper-evident public log.

Required fields:
- `id`
- `name`
- `description`
- `log_endpoint` — publicly accessible.
- `verification_method`
- `established_via_act_id` — FK to GOVERNANCE_ACT.
- `status` — active / deprecated.
- `infrastructure_security` — structured declaration matching INTEGRITY_MECHANISM `infrastructure_security` schema.
- `technology_requirements` — append-only semantics enforced at storage layer, cryptographic hash chaining for independent verification, support for third-party verification, minimum retention of 3 years.

Constraints:
- Tamper-evident, append-only, publicly accessible.
- Established or deprecated only through a GOVERNANCE_ACT with `act_type = governance_write`.
- `log_endpoint` must be reachable and pass periodic verification (E16).
- Infrastructure changes must be preceded by a governance_write act.

---

#### ATTESTATION_SCHEMA
A governed declaration of the external schema for attestation content.

Required fields:
- `id`
- `name`
- `version`
- `description`
- `field_mappings` — covers: ACT, ENTITY, INTENT, AUTHORIZATION, ATTRIBUTION, DRIFT_SIGNAL, RESOURCE_CONSUMPTION, WORKFLOW, GENESIS, PATHS_NOT_TAKEN_CONTENT, AUDIT_EXECUTION, SIGNAL_SUMMARY, GOVERNANCE_ACT, PROOF_OBLIGATION. WORKFLOW, GENESIS, AUDIT_EXECUTION, GOVERNANCE_ACT, PROOF_OBLIGATION may be null in specific attestation instances per their conditions; the mapping must exist in the schema regardless. GOVERNANCE_ACT mapping must include: `governance_type`, `participating_entity_ids`, `output_authorization_ids`, `signal_ids_considered`, and `paths_not_taken`.
- `machine_readable` — boolean. Must be true.
- `external_artifact_type_vocabulary` — instance-defined governed vocabulary.
- `established_via_act_id` — FK to GOVERNANCE_ACT.
- `status` — active / deprecated.

Constraints:
- `machine_readable` must be true.
- `field_mappings` must cover all fourteen listed primitives.
- Schema version changes require a GOVERNANCE_ACT with `act_type = governance_write`.

---

#### SCOPE
A governed, structured description of participation boundaries. The hierarchy is a DAG.

Required fields:
- `id`
- `domains` — governed vocabulary of domain identifiers.
- `deployment_domains` — subset of `domains` whose presence in an AUTHORIZATION scope makes it a deployment authorization. May be empty.
- `contexts` — governed vocabulary.
- `entity_types_permitted`
- `child_scope_ids` — array of FK to SCOPE. DAG-enforced.
- `is_root` — boolean. Exactly one per instance.
- `established_via_act_id` — FK to GOVERNANCE_ACT. Nullable for root scope (recorded in GENESIS instead).

Constraints:
- Extending scope requires a GOVERNANCE_ACT with `act_type = governance_write`.
- `child_scope_ids` must form a DAG at all times. Cycle — reject.
- Root scope creation recorded in GENESIS, not `established_via_act_id`.

---

#### ENTITY
The fundamental unit of participation.

Required fields:
- `id` — unique, stable, non-reassignable.
- `scope_id` — FK to SCOPE. Non-null.
- `participation_type` — vocabulary: authorizing / executing / participating / observing.
- `is_system` — boolean. Exactly one per instance.
- `is_bootstrap` — boolean. Exactly one per instance.
- `status` — active / suspended / retired.
- `established_via_act_id` — FK to GOVERNANCE_ACT or ACT.
  - Bootstrap entity: nullable (creation recorded in GENESIS).
  - System entity: must equal the bootstrap ACT id. Non-null.
  - All other entities: FK to the GOVERNANCE_ACT that established them. Non-null.
- `identity_attestation_ref` — reference to an attestation binding the entity identifier to a principal per the `identity_binding_model`. Non-null for all entities except bootstrap and system entity. Must be independently verifiable. Has a declared validity period (≤ 365 days).

Constraints:
- Exactly one `is_system = true`. System entity: `participation_type = executing`, scope = root scope, `established_via_act_id = bootstrap_act_id`.
- Exactly one `is_bootstrap = true`. Bootstrap entity creation recorded in GENESIS.
- Observing entities: `feedback` and `consent` signals only. May not initiate or execute acts.
- System entity: `escalation` and `acknowledgment` signals only. Kernel-generated, not human-initiated.
- Status transitions are authorized side effects of a governing ACT.
- No two active entities may share the same principal binding (duplicate detection per `identity_binding_model`).
- `identity_attestation_ref` must be valid and unexpired. Expiry triggers automatic suspension (E16).
- Identity attestation expiry warning (escalation SIGNAL) produced 30 days before expiry.

---

#### SIGNAL
A governance-relevant output produced by an ENTITY.

Required fields:
- `id`
- `entity_id` — FK to ENTITY. Non-null. System-generated signals use system entity id.
- `signal_type` — vocabulary: feedback / challenge / consent / dissent / escalation / acknowledgment / safety_assessment / proof_challenge.
- `content` — structured per signal_type. `safety_assessment` must conform to SAFETY_ASSESSMENT_CONTENT. `escalation` must conform to ESCALATION_CONTENT. `proof_challenge` must conform to PROOF_CHALLENGE_CONTENT. All other types: instance-defined schema.
- `references_act_id` — FK to ACT. Nullable.
- `references_attribution_id` — FK to ATTRIBUTION. Nullable.
- `references_proof_id` — FK to PROOF_OBLIGATION. Nullable. Required for proof_challenge signals.
- `produced_at`
- `governance_input_id` — FK to GOVERNANCE_ACT. Nullable. Mutually exclusive with `acknowledgment_act_id`.
- `acknowledgment_act_id` — FK to GOVERNANCE_ACT. Nullable. Mutually exclusive with `governance_input_id`.
- `superseded_by_signal_id` — FK to SIGNAL. Nullable. Non-null when status = superseded.
- `status` — pending / consumed / acknowledged / superseded / ignored.

**SAFETY_ASSESSMENT_CONTENT schema:**
- `assessment_tool` — must reference a method registered in the instance's governance records.
- `assessment_tool_version` — nullable.
- `hazard_categories` — array drawn from KERNEL_PARAMETERS `hazard_category_vocabulary`. Must cover all entries in the vocabulary.
- `test_count` — must be ≥ count of `hazard_category_vocabulary` entries.
- `violation_count`
- `violation_rate` — `violation_count / test_count`.
- `grade` — pass / conditional_pass / fail. Assessment with `test_count = 1` and `violation_count = 0` is insufficient — minimum test count is the number of hazard categories.
- `assessed_at`
- `assessor_entity_id` — FK to ENTITY. Must differ from the proponent entity (independence requirement).

**ESCALATION_CONTENT schema:**
- `escalation_type` — vocabulary: drift_threshold_exceeded / resource_limit_exceeded / non_participating_entity / orphan_signal_aged / attestation_provisional_expired / audit_in_arrears / parameter_trajectory_anomaly / duplicate_principal_detected / identity_attestation_expiring / system_entity_liveness_failure / measurement_invariance_detected / governance_uniformity_anomaly / verification_artifact_inconsistency / attestation_log_endpoint_failure / ungoverned_infrastructure_change / proof_expired / proof_invalidated / proof_dependency_cascade / degraded_mode_entered / degraded_mode_timeout / duplicate_intent_content / insufficient_alternative_substance / insufficient_safety_coverage.
- `severity` — critical / high / medium / low.
- `target_record_id` — FK to the record that triggered the escalation. Nullable.
- `details` — structured per escalation_type.

**PROOF_CHALLENGE_CONTENT schema:**
- `target_proof_id` — FK to PROOF_OBLIGATION.
- `challenge_type` — counterexample / logical_flaw / assumption_violation / expiry.
- `evidence` — structured argument or counterexample demonstrating the proof's invalidity.

Constraints:
- System entity may produce only `escalation` and `acknowledgment` signals.
- Observing entities may produce only `feedback` and `consent` signals.
- `safety_assessment` grade requirements: `test_count ≥ hazard_category_count`, `assessor_entity_id ≠ proponent_entity_id`.
- `proof_challenge` signals must reference a valid, active PROOF_OBLIGATION.

---

#### PROOF_OBLIGATION
A governed record of a proof that a named property holds for this implementation.

Required fields:
- `id`
- `property_id` — reference to a property in the Property Catalog (Layer 6).
- `proof_type` — formal / empirical / operational.
- `proof_content` — the structured argument, test results, or operational evidence.
- `proof_hash` — SHA-256 of `proof_content`. Content-addressed for integrity verification.
- `specification_version_hash` — the specification version this proof was validated against. Must match KERNEL_PARAMETERS `specification_version_hash`.
- `status` — submitted / verified / active / challenged / reaffirmed / invalidated / expired.
- `submitted_at`
- `verified_at` — nullable until verified.
- `expires_at` — `submitted_at + (proof_expiry_max_cycles × audit_interval_ms)`.
- `submitted_via_act_id` — FK to GOVERNANCE_ACT.
- `verified_via_audit_id` — FK to AUDIT_EXECUTION. Nullable until verified.
- `depends_on_proof_ids` — array of FK to PROOF_OBLIGATION. The proof dependency graph must be acyclic.
- `challenge_signal_ids` — array of FK to SIGNAL of type proof_challenge. Nullable.

Constraints:
- Exactly one active proof per property_id per instance. Submitting a new proof for a property with an active proof requires the old proof to transition to `expired` or `invalidated` first.
- `depends_on_proof_ids` must form a DAG. Cycle — reject.
- When a proof transitions to `invalidated`, all proofs with this proof in their `depends_on_proof_ids` must also transition to `invalidated` within one governance cycle (cascade invalidation). The kernel produces escalation SIGNALs with `escalation_type = proof_dependency_cascade` for each cascaded invalidation.
- `specification_version_hash` must match current KERNEL_PARAMETERS `specification_version_hash`. When the specification version changes, all proofs with a non-matching hash enter `expired` status and must be re-validated.
- Status `active` may only be set by the audit process (E15).
- Status `challenged` is set when a proof_challenge SIGNAL is produced referencing this proof.
- Status `reaffirmed` is set when a challenge is resolved in favor of the proof (challenge signal consumed).
- Status `invalidated` is set when a challenge is upheld or when a dependency is invalidated.
- Status `expired` is set automatically when `expires_at` is reached or when specification version changes.
- An implementation with any required property lacking an active proof is non-conforming. Non-conformance produces an escalation SIGNAL and gates deployment authorizations.

---

#### MEASUREMENT_METHOD
A governed declaration of how drift is computed.

Required fields:
- `id`
- `name`
- `description`
- `input_schema`
- `output_schema` — includes scale definition and threshold semantics.
- `confidence_schema` — defines scale, range, and interpretation of confidence values. Must specify: scale type, high/low confidence definition, interpretation guidance. Non-null.
- `error_model` — documented error model with declared precision and recall bounds.
- `test_vectors` — array of { input, expected_result (compliant / non_compliant) }. Minimum: 2 compliant inputs, 2 non-compliant inputs. Used by PROP-SUBSTANCE-003 to verify discriminative power against known inputs.
- `established_via_act_id` — FK to GOVERNANCE_ACT.
- `status` — active / deprecated.

Constraints:
- Established or deprecated only through a GOVERNANCE_ACT with `act_type = governance_write`.
- Must satisfy PROP-SUBSTANCE-003 (discriminative power) before activation.
- Must produce non-zero variance across measurement history (coefficient of variation > 0.01 over rolling window of 20 measurements).
- Automatic conformance warnings when operating below declared precision/recall bounds.

---

#### RESOURCE_MEASUREMENT_METHOD
A governed declaration of how resource consumption is measured. One per resource type.

Required fields:
- `id`
- `name`
- `description`
- `resource_type` — from `resource_type_vocabulary`. One per type.
- `input_schema` — inputs required to produce the measurement.
- `output_schema` — unit, scale, precision.
- `measurement_procedure` — structured, instance-defined description of the measurement process. Its adequacy is the responsibility of the establishing GOVERNANCE_ACT.
- `established_via_act_id` — FK to GOVERNANCE_ACT.
- `status` — active / deprecated.

Constraints:
- One active per resource type in `resource_type_vocabulary`. All types must have an active method before the instance accepts any acts.
- Established or deprecated only through a GOVERNANCE_ACT with `act_type = governance_write`.
- KERNEL_PARAMETERS `default_resource_measurement_method_ids` must reference an active method for every declared type.

---

#### DRIFT_SIGNAL
One record per beneficiary entity per act. Kernel-authored. Never reconstructed.

Required fields:
- `id`
- `attribution_id` — FK to ATTRIBUTION.
- `act_id` — FK to ACT.
- `entity_id` — FK to ENTITY.
- `intent_id` — FK to INTENT.
- `measured_at` — must precede or equal ATTRIBUTION `written_at`.
- `distance` — numeric on scale defined by MEASUREMENT_METHOD `output_schema`.
- `confidence` — numeric on scale defined by MEASUREMENT_METHOD `confidence_schema`. Must be within declared range.
- `method_id` — FK to active MEASUREMENT_METHOD.
- `within_threshold` — boolean.
- `threshold_used`
- `active_signals_at_act` — array of { signal_id, signal_status } for all signals this entity had pending at measurement time.
- `escalation_signal_id` — FK to SIGNAL of type escalation. Required when `within_threshold = false`. Null when within threshold.
- `outputs_released` — boolean. True when `within_threshold = true`. False when `within_threshold = false` until the GOVERNANCE_ACT that resolves the escalation declares the transition as an authorized side effect and the resulting ATTRIBUTION records it in `released_drift_signal_ids`.
- `degraded_flag` — boolean. True when the measurement was produced during degraded mode. Subject to reconciliation.

Constraints:
- One per beneficiary entity per ATTRIBUTION.
- `method_id` active at `measured_at`.
- `measured_at` ≤ ATTRIBUTION `written_at`.
- `confidence` within MEASUREMENT_METHOD `confidence_schema` range.
- `within_threshold = false` → `escalation_signal_id` non-null, `outputs_released = false`.
- `within_threshold = true` → `escalation_signal_id` null, `outputs_released = true`.
- `outputs_released` transition authorized as per P6.

---

#### ACT
The fundamental unit of governance action.

Required fields:
- `id`
- `act_type` — vocabulary: execution_act / governance_act / kernel_write / bootstrap.
- `authorization_id` — FK to AUTHORIZATION. Nullable for bootstrap and kernel_write.
- `intent_ids` — array of FK to INTENT. Non-empty for execution_act and governance_act.
- `scope_id` — FK to SCOPE.
- `initiated_at`
- `executing_entity_id` — FK to ENTITY.
- `status` — accepted / rejected / completed / rolled_back / conflict_paused / superseded.
- `workflow_id` — FK to WORKFLOW. Nullable.
- `parent_act_id` — FK to ACT. Nullable.
- `resource_estimate` — required when AUTHORIZATION `conditions.resource_limits` is non-null.
- `target_attribution_id` — FK to ATTRIBUTION. Required for kernel_write.
- `target_attestation_id` — FK to ATTESTATION. Required for kernel_write.
- `authorized_side_effects` — structured declaration of all mutation classes this act authorizes. Required for kernel_write, governance_act.

Constraints:
- All E1-E16 enforcement rules apply per act_type.
- `act_type = bootstrap` — permitted exactly once.
- `act_type = kernel_write` — self-authorizing within three-way atomic boundary.

---

#### INTENT
A structured declaration of purpose, presented before execution.

Required fields:
- `id`
- `entity_id` — FK to ENTITY. The declaring entity.
- `declared_at`
- `objective` — must not be tautological (PROP-SUBSTANCE-001 verification). Must not be textually identical to any INTENT submitted by the same entity within `audit_interval_ms`.
- `success_criteria` — array of { criterion, measurable_indicator }. `measurable_indicator` must not match completion-status patterns (E14 substance checks).
- `constraints` — non-empty.
- `scope_rationale` — must not be identical to scope description (similarity threshold < 0.90).
- `paths_not_taken` — conforming to PATHS_NOT_TAKEN_CONTENT. Non-null.
- `beneficiary_intent_snapshots` — array of { entity_id, intent_summary }.

**INTENT_CONTENT_MINIMUM:** `objective` non-empty, `success_criteria` non-empty with at least one measurable_indicator, `constraints` non-empty, `scope_rationale` non-empty. All must pass substance verification (E14).

**PATHS_NOT_TAKEN_CONTENT schema:**
- `alternatives` — array, minimum two entries per decision point.
  - `description` — must reference at least one concrete governance state (entity_id, scope_id, parameter name, or signal). Must not be a negation or inversion of the objective (token overlap < 0.70).
  - `reason_rejected` — must not be a restatement of the objective (token overlap < 0.70). Must not match pattern "would not achieve the objective."
  - `signal_ids_that_influenced` — array of FK to SIGNAL. May be empty.
- `deliberation_summary`

Constraints:
- `declared_at` must precede referencing ACT `initiated_at` by at least `minimum_intent_execution_interval_ms`.
- Must satisfy INTENT_CONTENT_MINIMUM.
- Must pass substance verification per E14.
- `paths_not_taken` must satisfy PATHS_NOT_TAKEN_CONTENT with substance requirements.

---

#### AUTHORIZATION, ATTRIBUTION, ATTESTATION, RESOURCE_CONSUMPTION, WORKFLOW, GOVERNANCE_ACT, AUDIT_EXECUTION

**AUTHORIZATION:** Required fields: `id`, `principal_id` (FK to ENTITY), `scope_id` (FK to SCOPE), `action_vocabulary`, `granting_act_id` (FK to GOVERNANCE_ACT), `granted_by_id` (FK to ENTITY), `status` (active / revoked / expired / consumed), `conditions` — structured record including: `governance_type_restrictions` (nullable array), `resource_limits` (nullable), `requires_safety_assessment` (boolean), `risk_classification` (nullable, required for deployment authorizations), `workflow_only` (boolean).

**ATTRIBUTION:** Required fields: `id`, `kernel_write_act_id` (FK to ACT), `attestation_id` (FK to ATTESTATION), `act_id` (FK to ACT), `executing_entity_id` (FK to ENTITY), `authorizing_entity_id` (FK to ENTITY), `beneficiary_entity_ids` (array of FK to ENTITY), `beneficiary_intent_snapshots`, `written_at`, `scope_id` (FK to SCOPE), `genesis_record_ids` (array, non-null for bootstrap only), `side_effect_record_ids` — structured as { primitive_type → [record_id, ...] }, `released_drift_signal_ids` (array), `resource_consumption_id` (FK to RESOURCE_CONSUMPTION), `audit_execution_id` (FK to AUDIT_EXECUTION, nullable), `degraded_flag` — boolean, true when produced during degraded mode.

**ATTESTATION:** Required fields: `id`, `attribution_id` (FK to ATTRIBUTION), `kernel_write_act_id` (FK to ACT), `canonical_serialization`, `integrity_proof` (nullable until signed), `log_entry_ref` (nullable until submitted), `external_schema_representation` (covers all fourteen required primitives), `drift_summary`, `resource_consumption_summary`, `signal_summary` (non-null), `paths_not_taken_summary` (non-null except bootstrap), `workflow_summary` (nullable), `status` (provisional / verifiable), `proof_obligation_summary` — array of { proof_id, property_id, status } for all proofs (any status) at attestation time. Non-null for all ATTESTATIONs produced after bootstrap. An empty array (no proofs yet active) satisfies the non-null requirement — this is expected between bootstrap and first proof verification.

**RESOURCE_CONSUMPTION:** Required fields: `id`, `act_id` (FK to ACT), `consumption` — one entry per resource type in `resource_type_vocabulary`, each with `resource_type`, `quantity`, `unit`, `measurement_method_id` (FK to RESOURCE_MEASUREMENT_METHOD), `measured_at`, `within_limits` (boolean), `escalation_signal_id` (FK to SIGNAL, non-null when `within_limits = false`).

**WORKFLOW:** Required fields: `id`, `name`, `scope_id` (FK to SCOPE), `established_via_governance_act_id` (FK to GOVERNANCE_ACT, non-null), `initiating_act_id` (FK to ACT), `act_ids` (ordered array of FK to ACT), `status` (active / completed / failed).

**GOVERNANCE_ACT:** Required fields: `id`, `governance_type` — vocabulary: amendment / audit / override / establishment / deprecation, `is_bootstrap` (boolean, exactly one per instance), `participating_entity_ids` (array of FK to ENTITY), `output_authorization_ids` (array of FK to AUTHORIZATION), `signal_ids_considered` (array of FK to SIGNAL), `authorized_side_effects` — structured declaration of all mutation classes, `paths_not_taken` — conforming to PATHS_NOT_TAKEN_CONTENT with substance requirements, `status` (in_progress / completed / conflict_paused / superseded), `target_governance_act_id` (FK to GOVERNANCE_ACT, for overrides).

**AUDIT_EXECUTION:** Required fields: `id`, `governance_act_id` (FK to GOVERNANCE_ACT), `auditing_entity_id` (FK to ENTITY), `criteria_evaluated` (array of C1-C22 identifiers), `findings` (structured), `executed_at`, `is_first_audit` (boolean), `proof_verification_results` — array of { proof_id, property_id, verification_result (pass/fail), verification_procedure_used }. Non-null. `deterministic_verification_output` — structured output of the Layer 7a deterministic verifier run. Non-null.

---

## Layer 3 — Enforcement Rules

### Traceability Matrix

| Proposition | Enforcement Rules | Audit Criteria |
|------------|-------------------|----------------|
| P1 (attribution) | E1, E6, E9 | C1, C3, C6, C8, C12, C16 |
| P2 (audit governed) | E13, Layer 7c | C14, C21 |
| P3 (bidirectional) | E8, E16, PROP-LIVENESS-004 | C4, C5, C9, C17 |
| P4 (authorization traceable) | E2, E7, E9 | C1, C6, C16 |
| P5 (intent before execution) | E1, E4, E14 | C1, C8, C15, C18 |
| P6 (drift measurable) | E5, E14 | C2, C15, C18 |
| P7 (immutable record) | E6, E9 | C3, C6 |
| P8 (scope enforced) | E3 | C7 |
| P9 (loop closes) | E8 | C4, C5 |
| P10 (no self-governance) | E7 | C1 |
| P11 (downstream first-class) | E8, E12, E14 | C4, C13, C18 |
| P12 (workflows governed) | E10 | C10 |
| P13 (externally verifiable) | E11, E16 | C11, C19 |
| P14 (choices proven) | E14, E15 | C15, C18, C20, C21 |
| P15 (degraded mode) | E16 | C22 |

---

### E1–E13 — Structural Enforcement

E1 through E13 enforce the structural integrity of governance records. They validate that records exist, fields are populated, references resolve, schemas are satisfied, and authorization chains are sound.

- **E1 (Act completeness):** Every ACT presented for execution must carry valid AUTHORIZATION (except bootstrap and kernel_write), referenced INTENTs satisfying INTENT_CONTENT_MINIMUM with conforming `paths_not_taken`, a SCOPE within the executing entity's scope, and RESOURCE_CONSUMPTION coverage. INTENTs must additionally pass substance verification per E14. Missing or non-conforming — reject.
- **E2 (Authorization validity):** AUTHORIZATION active, unexpired, unrevoked, unconsumed. Granting entity held authority at grant time. Full authorization chain traversal to bootstrap at runtime — not just one-hop grant-time check. The chain must resolve to the root AUTHORIZATION (non-system) or bootstrap GOVERNANCE_ACT (system entity) at every act, not only at audit time. Act type within `action_vocabulary`. Safety assessments must meet minimum rigor requirements (test_count ≥ hazard_category_count, assessor ≠ proponent). Any failure — reject.
- **E3 (Scope enforcement):** ACT scope fully contained within executing entity's SCOPE and AUTHORIZATION SCOPE. kernel_write scope inherited from parent act. Any exceedance — reject.
- **E4 (Intent precedence):** Every referenced INTENT `declared_at` strictly earlier than `initiated_at` by at least `minimum_intent_execution_interval_ms`. Sub-interval — reject.
- **E5 (Drift signal completeness):** DRIFT_SIGNAL records computed for every beneficiary entity before ATTRIBUTION is written. `confidence` within range. Threshold violations produce escalation SIGNAL. Output gating enforced. MEASUREMENT_METHOD must have non-zero variance per PROP-SUBSTANCE-003; variance failure (coefficient of variation ≤ 0.01) produces escalation SIGNAL with `escalation_type = measurement_invariance_detected`. Partial coverage — reject.
- **E6 (Governance record self-attribution):** Every write via one of the four mechanisms in P7. Entity-authored primitives validated by kernel. Any mutation outside authorized mechanisms — reject.
- **E7 (Self-authorization and mutual authorization prevention):** Full bidirectional traversal at grant time. System entity holds only bootstrap-produced authorizations. System entity AUTHORIZATION is irrevocable. Any self-authorization or mutual authorization detected — reject.
- **E8 (Feedback channel integrity):** Every ENTITY receiving outputs must produce SIGNALs within `signal_production_window_ms`. Orphan signals older than `orphan_signal_max_age_ms` produce escalation. Signal type vocabulary enforced per participation_type.
- **E9 (Bootstrap integrity):** Bootstrap ACT exempt from E1-E8 and E14 (substance checks). Bootstrap acts are exempt from substance verification because minimal governance state exists at bootstrap time to reference in paths-not-taken or intent content. Must produce GOVERNANCE_ACT with `is_bootstrap = true`, exactly two output authorizations, system entity established. GENESIS `specification_version_hash` must be present. Initial PROOF_OBLIGATION records for genesis-time properties must be submitted as authorized side effects. Genesis-time properties are all root properties in the proof dependency graph (properties with no dependencies): PROP-PARAM-001, PROP-PARAM-003, PROP-IDENTITY-001, PROP-LIVENESS-001, PROP-SUBSTANCE-001 through PROP-SUBSTANCE-005, PROP-INFRA-001, PROP-INFRA-002, PROP-PROOF-001, PROP-PROOF-002, PROP-ENFORCEMENT-001, PROP-SUBSTRATE-001.
- **E10 (Workflow integrity):** Every workflow ACT references a WORKFLOW with `established_via_governance_act_id`. Establishing GOVERNANCE_ACT and initiating ACT are distinct records. No workflow step executes while predecessor has unreleased outputs.
- **E11 (Attestation completeness):** Every ATTRIBUTION has corresponding ATTESTATION written atomically. Post-production writes within `attestation_provisional_window_ms`. Failure — escalation.
- **E12 (Paths not taken completeness):** Every INTENT and GOVERNANCE_ACT has non-null, conforming `paths_not_taken`. Must meet substance requirements per E14. Missing or empty — reject.
- **E13 (Audit currency):** Full audit within `audit_interval_ms` of previous audit. Arrears — escalation.

---

### E14 — Parameter outcome enforcement and substance verification

**Parameter governance:**
- No single governance_write act may amend more than one KERNEL_PARAMETERS parameter.
- No parameter may be amended more than once within `audit_interval_ms`.
- Every amendment must result in a value within the declared floor and ceiling for that parameter. Outside bounds — reject.
- Cumulative amendments to any parameter exceeding 50% of its current value in the same direction within a rolling window of `audit_interval_ms × 4` produce an escalation SIGNAL with `escalation_type = parameter_trajectory_anomaly`.
- `drift_threshold_default` immutable without supermajority authorization.

**Intent substance verification:**
- INTENT `objective` must not match tautological patterns: "execute/perform/complete/carry out" followed by "this act/the act/this intent/described herein" (normalized, case-insensitive substring match).
- INTENT `success_criteria.measurable_indicator` must not match completion patterns: "status = completed", "act completes", "execution finishes" (normalized substring match).
- INTENT `constraints` must be non-empty.
- INTENT `scope_rationale` similarity to scope description must be < 0.90.
- No two INTENTs from the same entity within `audit_interval_ms` may share identical `objective` hash (SHA-256 of normalized text). Duplicate — reject with `escalation_type = duplicate_intent_content`.

**Paths-not-taken substance verification:**
- Minimum two alternatives per decision point.
- Alternative `reason_rejected` token overlap with parent objective must be < 0.70.
- Alternative `description` must not be a negation/inversion of the objective (substring match with 60% token overlap threshold).
- Alternative `description` must reference at least one concrete governance state (entity_id, scope_id, parameter name, or non-empty signal_ids_that_influenced).
- All alternatives in the same record must have distinct description hashes.
- Alternatives failing any of the above checks — reject the referencing ACT with `escalation_type = insufficient_alternative_substance`.

**Safety assessment substance verification:**
- `test_count` ≥ count of `hazard_category_vocabulary` entries.
- `hazard_categories` must cover all vocabulary entries.
- `test_count` must be > 1. Single-test-pass (`test_count = 1, violation_count = 0`) — reject with `escalation_type = insufficient_safety_coverage`.
- `assessor_entity_id` must differ from proponent entity. Self-assessment — reject.

Any substance verification failure — reject the referencing ACT.

**Cross-artifact uniformity monitoring:** Per PROP-SUBSTANCE-005, the kernel monitors cross-artifact uniformity across entities. When the uniformity score exceeds the declared threshold, the system entity produces an escalation SIGNAL with `escalation_type = governance_uniformity_anomaly`. This is a monitoring kernel_write, not an act rejection — the individual acts may be valid, but the pattern across entities warrants investigation.

---

### E15 — Proof obligation enforcement

- Every property in the Property Catalog (Layer 6) must have exactly one PROOF_OBLIGATION with status `active` for the instance to accept deployment authorizations.
- A proof in `submitted` status must be verified by the next audit cycle. Unverified after one cycle — transition to `expired`.
- A proof in `active` status must be renewed before `expires_at`. Expiry — system entity produces escalation SIGNAL with `escalation_type = proof_expired`.
- A proof in `invalidated` status triggers cascade invalidation of all dependent proofs. System entity produces escalation SIGNAL with `escalation_type = proof_dependency_cascade` for each cascade.
- The proof dependency graph (across all PROOF_OBLIGATION `depends_on_proof_ids`) must be acyclic at all times. Cycle — reject the proof submission.
- When KERNEL_PARAMETERS `specification_version_hash` changes, all proofs with non-matching `specification_version_hash` transition to `expired`. System entity produces escalation SIGNAL with `escalation_type = proof_expired` for each.
- A proof_challenge SIGNAL may be produced by any entity referencing an active proof. The proof transitions to `challenged` status. The challenge must be resolved (proof reaffirmed or invalidated) within one audit cycle. Unresolved — proof transitions to `invalidated`.
- Proof status `active` may only be set by the audit process — specifically, by the verification procedure defined in the Property Catalog for the relevant property. The auditing entity records the verification in the AUDIT_EXECUTION `proof_verification_results`.

---

### E16 — Liveness, identity, and infrastructure enforcement

**System entity liveness:**
- The liveness mechanism declared in `system_entity_liveness_model` must be active at all times.
- For heartbeat model: system entity must publish a signed proof-of-liveness to the attestation log at intervals ≤ `system_entity_liveness_model.detection_interval_ms`. Missed heartbeat — the watchdog or liveness oracle produces escalation SIGNAL with `escalation_type = system_entity_liveness_failure`.
- On liveness failure detection: all governance mutations suspended (no new governance_write acts accepted) without system entity involvement. The suspension mechanism must be independent of the system entity.
- Recovery requires: fresh liveness proof, reconciliation audit of the failure period, quorum authorization (≥ 2/3 of active entities).

**Entity identity:**
- At entity creation time, `identity_attestation_ref` must be present (except bootstrap and system entity).
- The principal binding hash must not collide with any existing active entity's binding hash. Collision — reject with `escalation_type = duplicate_principal_detected`.
- Identity attestation validity period must not exceed 365 days.
- 30 days before expiry: system entity produces escalation SIGNAL with `escalation_type = identity_attestation_expiring`.
- On expiry: entity status transitions to `suspended`. No governance_write acts accepted from suspended entities.

**Infrastructure verification:**
- The kernel must periodically verify ATTESTATION_LOG `log_endpoint` at intervals ≤ `audit_interval_ms`: endpoint reachability, TLS certificate validity, content integrity (most recent entry matches kernel's local record), append-only property (no previous entries modified or removed). Failure — escalation SIGNAL with `escalation_type = attestation_log_endpoint_failure`.
- The kernel must verify INTEGRITY_MECHANISM `public_verification_artifact` consistency across all declared locations at intervals ≤ `audit_interval_ms`. Inconsistency — escalation SIGNAL with `escalation_type = verification_artifact_inconsistency`.
- Infrastructure changes (DNS records, TLS certificates, hosting configuration, access control) not preceded by a governance_write act — escalation SIGNAL with `escalation_type = ungoverned_infrastructure_change`.

**Degraded mode:**
- When any infrastructure component (attestation log, integrity mechanism, measurement method, external verification endpoint) becomes unavailable, the kernel enters degraded mode.
- System entity produces escalation SIGNAL with `escalation_type = degraded_mode_entered` and `severity = high`.
- All ATTRIBUTION and DRIFT_SIGNAL records produced during degraded mode must have `degraded_flag = true`.
- The kernel must continue accepting and processing governance acts during degraded mode. Attestation production continues but `integrity_proof` and `log_entry_ref` may be deferred (ATTESTATION remains `provisional`).
- If degraded mode persists beyond `max_degraded_window_ms`, system entity produces escalation SIGNAL with `escalation_type = degraded_mode_timeout` and `severity = critical`. The instance transitions to the same governance mutation suspension used for liveness failure.
- On infrastructure restoration: all deferred attestation operations must complete within `attestation_provisional_window_ms`. A reconciliation audit must cover the degraded period, verifying that all `degraded_flag = true` records are structurally valid and that no governance constraints were violated during impairment.

---

## Layer 4 — Audit Criteria

### C1–C14 — Structural Audit Criteria

C1 through C14 verify structural compliance — that records exist, fields are populated, references resolve, and schemas are satisfied. Each criterion follows the pattern "Select any [record type]. Verify [structural properties]. If any failure — not satisfied." These criteria are defined inline:

- **C1:** Can every act be traced to an authorization? (Full chain traversal, mutual authorization detection, safety assessment presence, irrevocability verification.)
- **C2:** Can every entity's drift be measured? (DRIFT_SIGNAL presence, method validity, confidence range, output gating.)
- **C3:** Is the governance record self-attributing? (Three-way reference, authorized side effects, GENESIS integrity.)
- **C4:** Are downstream participants represented? (Beneficiary entity presence, intent snapshots, paths not taken summary.)
- **C5:** Does the feedback loop close? (Signal resolution, orphan signal detection.)
- **C6:** Is bootstrap traceable? (Bootstrap GOVERNANCE_ACT, two authorizations, KERNEL_PARAMETERS, GENESIS.)
- **C7:** Is scope enforced at every layer? (Scope containment, DAG enforcement.)
- **C8:** Does intent precede execution? (Timing, content minimum, paths not taken.)
- **C9:** Are non-participating entities detected? (Signal production window, escalation.)
- **C10:** Are workflows auditable as units? (Establishing act, chain integrity, output gating.)
- **C11:** Are attestations externally verifiable? (Log entry, integrity proof, schema coverage.)
- **C12:** Is resource consumption accounted for? (Coverage per type, pre-execution gate.)
- **C13:** Are paths not taken recorded? (INTENT and GOVERNANCE_ACT, attestation summary consistency.)
- **C14:** Is audit execution governed and current? (AUDIT_EXECUTION attribution, currency, finding signals.)

---

### C15 — Are parameter values within declared bounds?
Select any KERNEL_PARAMETERS record. Verify every time-based parameter is within its declared floor and ceiling per `parameter_bounds`. Verify no parameter has been amended more than once within `audit_interval_ms`. Verify no unresolved `parameter_trajectory_anomaly` escalation exists. Verify `drift_threshold_default` has not been relaxed below its GENESIS value without supermajority authorization. If any failure — not satisfied.

### C16 — Is every entity bound to a verified principal?
Select any active ENTITY (excluding bootstrap and system entity). Verify `identity_attestation_ref` is non-null. Verify the attestation is independently verifiable using only public artifacts. Verify the attestation is not expired. Verify no other active entity shares the same principal binding hash. If any failure — not satisfied.

### C17 — Is the system entity provably alive?
Retrieve `system_entity_liveness_model` from KERNEL_PARAMETERS. Verify the declared mechanism is active and its dependency graph does not include the system entity. Verify the most recent liveness proof is within the required interval. Verify no unresolved `system_entity_liveness_failure` escalation exists. If any failure — not satisfied.

### C18 — Is governance substance distinguishable from governance theater?
Select 10 INTENT records (or all, if fewer than 10) within `audit_interval_ms`. Verify no two from the same entity share identical `objective` hash. Verify no INTENT matches tautological patterns per E14. Verify PATHS_NOT_TAKEN alternatives reference concrete governance state per E14. Verify MEASUREMENT_METHOD history shows non-trivial variance (coefficient of variation > 0.01) per PROP-SUBSTANCE-003. Verify SAFETY_ASSESSMENT records meet minimum coverage per E14. Verify no unresolved `governance_uniformity_anomaly` escalation exists. Run the hollow instance test: submit the known-tautological templates to the substance checks and verify they are rejected. If any failure — not satisfied.

### C19 — Is external verification infrastructure governed and healthy?
Verify every active INTEGRITY_MECHANISM has non-null `infrastructure_security` and `public_verification_artifact_locations` with at least two independent entries. Verify artifacts at all declared locations are consistent (identical hash). Verify ATTESTATION_LOG endpoint passes all four verification checks per E16. Verify no unresolved `verification_artifact_inconsistency`, `attestation_log_endpoint_failure`, or `ungoverned_infrastructure_change` escalation exists. Verify all infrastructure changes within `audit_interval_ms` were preceded by governance_write acts. If any failure — not satisfied.

### C20 — Are all properties proven?
For each property in the Property Catalog (Layer 6) — currently 25 properties across 8 boundaries: verify a PROOF_OBLIGATION with status `active` exists. Verify `specification_version_hash` on each proof matches current KERNEL_PARAMETERS `specification_version_hash`. Verify no proof is expired. Verify the proof dependency graph is acyclic. Verify no unresolved proof_challenge SIGNALs exist. If any property lacks an active proof — not satisfied.

### C21 — Has the deterministic verifier passed?
Retrieve the most recent deterministic verification output (Layer 7a). Verify all mechanically verifiable checks passed. Verify the verification was run within the current audit cycle. If no verification exists or any check failed — not satisfied.

### C22 — Were degraded-mode records reconciled?
Retrieve all ATTRIBUTION records with `degraded_flag = true` produced since the last audit. For each, verify: a reconciliation audit covers the degraded period, all deferred attestation operations completed within `attestation_provisional_window_ms` of infrastructure restoration, structural validity confirmed. If any degraded records are unreconciled — not satisfied.

---

## Layer 5 — Attestation

#### Purpose
Every ATTRIBUTION produces a corresponding ATTESTATION in the same three-way atomic operation. An instance unable to produce conforming attestations is not a conforming kernel.

#### Design principle
The kernel mandates properties, not technologies. Any conforming technology satisfies the properties. Hard-coding a specific technology without declaring it as a governed replaceable parameter is non-conforming.

#### Required properties of the integrity mechanism
- Cryptographic. Algorithm in KERNEL_PARAMETERS `approved_algorithms`.
- Publicly verifiable without instance access via declared `public_verification_artifact` from at least two independent locations.
- Deterministic — same ATTRIBUTION always produces same canonical serialization.
- Replaceable via GOVERNANCE_ACT with `act_type = governance_write`.
- Infrastructure security properties declared and attestable.

#### Required properties of the attestation log
- Tamper-evident and append-only with cryptographic hash chaining.
- Publicly accessible without instance access.
- Support for third-party verification.
- Minimum retention of 3 years.
- Periodic endpoint verification per E16.
- Replaceable via GOVERNANCE_ACT with `act_type = governance_write`.

#### Required properties of the attestation schema
- Machine-readable.
- Covers all fourteen required kernel primitives (including PROOF_OBLIGATION).
- Versioned — changes require GOVERNANCE_ACT with `act_type = governance_write`.
- Replaceable via GOVERNANCE_ACT.

#### Attestation lifecycle

1. **Production** — ATTESTATION, ATTRIBUTION, kernel_write ACT written atomically. ATTESTATION `status = provisional`. `integrity_proof` and `log_entry_ref` null. In degraded mode, production proceeds but steps 2-3 are deferred.
2. **Signing** — `canonical_serialization` signed via INTEGRITY_MECHANISM. `integrity_proof` written as authorized side effect declared in kernel_write ACT `authorized_side_effects`.
3. **Log submission** — Attestation submitted to ATTESTATION_LOG. `log_entry_ref` written as authorized side effect. `status` transitions to `verifiable` — also declared in kernel_write ACT `authorized_side_effects`.
4. **Promotion window** — Steps 2 and 3 must complete within `attestation_provisional_window_ms`. Failure — system entity escalates with `escalation_type = attestation_provisional_expired`. In degraded mode, window begins when infrastructure is restored.
5. **External verification** — Auditor retrieves `integrity_proof`, `log_entry_ref`, INTEGRITY_MECHANISM `public_verification_artifact` from any declared location. Verifies proof against `canonical_serialization`. Verifies log entry. No instance access required.

#### What an independent auditor can verify
- Every act: timing, executor, authorization chain, conditions — via `external_schema_representation`
- Drift per entity: distance, confidence, threshold, within_threshold, outputs_released, escalation type — via `drift_summary`
- Resource consumption per type: quantity, unit, measurement method, within_limits, escalation type — via `resource_consumption_summary`
- Signal types and outcomes: signal type, escalation type, safety assessment grade, signal status — via `signal_summary`
- Paths not taken at every decision — via `paths_not_taken_summary`
- Workflow sequence and chain integrity — via `workflow_summary`
- Pre-instance precondition integrity — via GENESIS content hashes in bootstrap ATTESTATION
- Audit execution history and findings — via AUDIT_EXECUTION ATTESTATIONs
- External governance artifact associations — via `external_artifact_refs`
- Cryptographic integrity and tamper-evidence — via `integrity_proof` and `log_entry_ref`
- Proof obligation status for all properties — via `proof_obligation_summary`
- Degraded mode periods and reconciliation status — via `degraded_flag` and reconciliation AUDIT_EXECUTION
- Specification version conformance — via `specification_version_hash`

#### What requires instance access to verify
The following are verified internally via Layer 4 (C1-C14) and are available to a physical auditor (PA-001 through PA-007) with instance access. They are not unverified — they are verified at a different layer:
- Full content of SIGNAL records beyond type, escalation_type, grade, and status
- Full `paths_not_taken.alternatives` detail beyond the structured summary
- Full `input_schema` and `output_schema` of MEASUREMENT_METHOD and RESOURCE_MEASUREMENT_METHOD records
- Full `measurement_procedure` content of RESOURCE_MEASUREMENT_METHOD records
- Full proof_content of PROOF_OBLIGATION records — summaries are attested, full proofs require instance access

An instance may include full content as governed optional extensions to the attestation schema, making these externally verifiable. Physical audit exit PA-007 (substance sampling) explicitly accesses full governance artifact content to verify sincerity beyond the attested summaries.

---

## Adversary Model Catalog

The following adversary models define the threat scenarios each property is designed to resist. Each model specifies the adversary's goal, capabilities, and the governance boundary they exploit.

| ID | Name | Goal | Capabilities | Boundary Exploited |
|----|------|------|-------------|-------------------|
| R1 | Captured Council | Achieve governance capture through legitimate governance acts | Majority coalition of authorized entities; all acts structurally valid | Parameter outcomes, measurement methods — governance mechanisms used against governance |
| R2 | Authorization Escalation | Acquire privileges beyond those legitimately granted | Single entity with valid authorization; exploits delegation chains or mutual authorization gaps | Authorization traceability — chain traversal depth or cycle detection failures |
| R3 | Bootstrap Substitution | Replace the trust root with an attacker-controlled genesis | Access to pre-governance infrastructure; physical or administrative access to bootstrap environment | Bootstrap integrity — pre-governance window has no enforcement |
| R4 | Silent Collapse | Disable all automatic enforcement without detection | Ability to suppress the system entity's runtime process | System entity liveness — detection mechanism depends on the entity being detected |
| R5 | Verification Subversion | Make forged attestations appear valid to external verifiers | DNS or CDN compromise for verification artifact hosting | External infrastructure — verification depends on infrastructure the spec doesn't govern |
| R6 | Compliance Theater | Satisfy all structural checks with substantively empty governance | Any authorized entity; no special capabilities required | Governance substance — form governed, meaning ungoverned |
| R7 | Slow Bleed | Degrade governance parameters to ineffectiveness through individually valid amendments | Authorized entity with parameter amendment rights; patience across multiple governance cycles | Parameter outcomes — mechanism of change governed, outcome of change ungoverned |
| R8 | Sybil Capture | Achieve governance capture through fabricated entity consensus | Ability to create multiple entity identities bound to a single principal | Entity identity — behavior governed, authenticity ungoverned |

Every property in the Property Catalog references at least one adversary model by R# identifier. The model defines the attack the property defends against. An implementation's proof must demonstrate resistance to the declared adversary model's capabilities.

---

## Layer 6 — Property Catalog

*Properties are the fundamental unit of conformance. Every instance-defined choice must satisfy at least one property. Each property has: a falsifiable statement, an adversary model, acceptable proof types, a verification procedure, and declared dependencies on other properties.*

**Structural vs. instance-defined propositions.** Propositions P1, P3, P4, P7, P8, P9, P10, and P12 are structurally enforced by E1-E13 and verified by C1-C14. Their enforcement mechanisms are defined in the specification, not chosen by the implementation. These propositions do not require proof obligations because the implementation does not choose how to satisfy them — the specification prescribes the enforcement. Propositions P5, P6, P11, P13, P14, and P15 involve instance-defined choices (measurement methods, identity binding, liveness mechanisms, substance validation, infrastructure hosting, degraded mode behavior) and require proof obligations via the properties below. Additionally, PROP-ENFORCEMENT-001 requires implementations to prove their enforcement rule implementations are correct, closing the gap between structural specification and behavioral correctness.

---

**Implicit substrate dependency.** All properties in this catalog implicitly depend on faithful substrate execution. If the substrate is compromised, no property's proof is trustworthy. This implicit dependency is encoded as follows: every root property (property with no declared dependencies) carries `depends_on_substrate: true`. Invalidation of the substrate trust declaration (PA-008 failure or future PROP-SUBSTRATE-001 invalidation) cascades to all properties via PROP-PROOF-001's cascade mechanism. Until formal PROP-SUBSTRATE properties are added to the catalog, this dependency is verified by PA-008 (substrate attestation) rather than by a proof obligation.

---

#### Boundary 1: Parameter Outcomes

*The spec governs how parameters change. These properties govern what parameter states are acceptable.*

#### PROP-PARAM-001: Parameter Boundedness

**Statement:** Every amendable parameter in KERNEL_PARAMETERS has a declared floor and ceiling such that no governance_write can set the parameter outside those bounds.

**Verifies:** P14

**Adversary model:** Slow Bleed (R7) — cumulative legitimate amendments degrade governance to ineffectiveness.

**Proof type:** Formal.

**Verification procedure (deterministic):**
```
FOR each amendable parameter P in KERNEL_PARAMETERS:
  VERIFY P.floor is declared and non-null
  VERIFY P.ceiling is declared and non-null
  VERIFY P.floor ≤ P.current_value ≤ P.ceiling
  VERIFY P.floor ≥ normative_minimum(P)
  SUBMIT test governance_write setting P = P.floor       → MUST accept
  SUBMIT test governance_write setting P = P.floor - 1   → MUST reject
  SUBMIT test governance_write setting P = P.ceiling      → MUST accept
  SUBMIT test governance_write setting P = P.ceiling + 1  → MUST reject
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-PARAM-002: Parameter Trajectory Resistance

**Statement:** No sequence of individually valid parameter amendments can reduce any parameter to a value where its governed function is effectively disabled, within a time window shorter than two audit cycles.

**Verifies:** P14

**Adversary model:** Slow Bleed (R7) — multi-step degradation evading per-amendment review.

**Proof type:** Formal.

**Verification procedure (deterministic):**
```
FOR each amendable parameter P in KERNEL_PARAMETERS:
  RETRIEVE amendment history over rolling window (audit_interval_ms × 4)
  COMPUTE cumulative_delta = sum of all amendments in same direction
  COMPUTE max_degradation = P.current_value - P.floor
  VERIFY cumulative_delta < (max_degradation × 0.5)
  IF cumulative_delta ≥ (max_degradation × 0.5):
    VERIFY escalation SIGNAL with type=parameter_trajectory_anomaly exists and is resolved
PASS if all checks pass.
```

**Dependencies:** PROP-PARAM-001.

---

#### PROP-PARAM-003: Parameter Amendment Isolation

**Statement:** No single governance_write act can amend more than one parameter simultaneously. No parameter is amended more than once within `audit_interval_ms`.

**Verifies:** P14

**Adversary model:** Slow Bleed (R7) — compound parameter changes that are individually harmless but collectively disabling.

**Proof type:** Formal.

**Verification procedure (deterministic):**
```
RETRIEVE all governance_write acts targeting KERNEL_PARAMETERS within audit window
FOR each act:
  COUNT number of distinct parameters modified
  VERIFY count = 1
FOR each parameter:
  VERIFY no two amendments within audit_interval_ms
PASS if all checks pass.
```

**Dependencies:** None.

---

#### Boundary 2: Entity Identity

*The spec governs entity behavior. These properties govern entity authenticity.*

#### PROP-IDENTITY-001: Principal Uniqueness

**Statement:** No two active entities are controlled by the same principal. The identity binding mechanism prevents one principal from holding multiple entity identities.

**Verifies:** P1, P4, P10

**Adversary model:** Sybil attack (R8) — one principal creates multiple entities to capture governance.

**Proof type:** Formal or empirical.

**Verification procedure (deterministic + physical audit exit):**
```
RETRIEVE identity_binding_model from KERNEL_PARAMETERS
RETRIEVE all active ENTITYs
FOR each ENTITY:
  VERIFY identity_attestation_ref is non-null
  EXTRACT principal_binding_hash from attestation
FOR all pairs of active entities (E_i, E_j):
  VERIFY principal_binding_hash(E_i) ≠ principal_binding_hash(E_j)
PASS if all hashes unique.
```
**PHYSICAL AUDIT EXIT PA-001:** A human auditor with physical or administrative access to the identity binding infrastructure must verify: (a) the binding type is resistant to Sybil attack given the deployment's actual threat environment, (b) the uniqueness guarantee cannot be circumvented by a principal with access to the binding issuance process. The auditor must produce a signed attestation referencing this property, the binding mechanism inspected, and the finding (pass/fail with rationale). This attestation is a governance record. Failure to produce it within one audit cycle is a finding.

**Dependencies:** None.

---

#### PROP-IDENTITY-002: Identity Binding Verifiability

**Statement:** An independent party can verify the binding between an entity identifier and its principal without instance access.

**Verifies:** P1, P4, P13

**Adversary model:** Sybil Capture (R8) — impersonation via forged or stolen identity binding.

**Proof type:** Operational.

**Verification procedure (deterministic):**
```
SELECT random sample of 10 active ENTITYs (or all, if fewer)
FOR each:
  RETRIEVE identity_attestation_ref
  EXECUTE declared verification procedure using only public artifacts
  VERIFY result = yes
PASS if all verifications succeed.
```

**Dependencies:** PROP-IDENTITY-001.

---

#### PROP-IDENTITY-003: Identity Binding Revocability

**Statement:** When an identity binding is compromised, the implementation can revoke it within one governance cycle without disrupting governance for non-compromised entities.

**Verifies:** P4

**Adversary model:** Sybil Capture (R8) — key compromise enabling unauthorized entity control.

**Proof type:** Operational.

**Verification procedure (deterministic + physical audit exit):**
```
FOR each active ENTITY:
  VERIFY identity attestation expiry_date is declared (≤ 365 days from issuance)
  IF days_until_expiry ≤ 30:
    VERIFY escalation SIGNAL with type=identity_attestation_expiring exists
FOR each suspended ENTITY where reason = identity_expiry:
  VERIFY no governance_write acts accepted from this entity after suspension
PASS if all checks pass.
```
**PHYSICAL AUDIT EXIT PA-002:** A human auditor must execute a timed revocation test: revoke one identity binding and measure time to completion. The procedure must complete within one governance cycle. If no revocation has occurred in production, a tabletop exercise with timed walkthrough is required. The auditor must produce a signed attestation with the measured or estimated completion time. If completion time exceeds one governance cycle, this is a finding.

**Dependencies:** PROP-IDENTITY-001, PROP-IDENTITY-002.

---

#### Boundary 3: System Entity Liveness

*The spec governs what the system entity does. These properties govern that it's doing it.*

#### PROP-LIVENESS-001: Independent Failure Detection

**Statement:** System entity non-responsiveness is detectable by a mechanism that does not depend on the system entity's own operation. Detection occurs within `audit_interval_ms / 4`.

**Verifies:** P3

**Adversary model:** Silent collapse (R4) — total enforcement failure with zero internal detection.

**Proof type:** Formal + operational.

**Verification procedure (deterministic + physical audit exit):**
```
RETRIEVE system_entity_liveness_model from KERNEL_PARAMETERS
VERIFY model is declared and non-null

IF heartbeat:
  RETRIEVE most recent heartbeat from attestation log
  VERIFY timestamp ≤ now - detection_interval_ms
  VERIFY heartbeat signed by system entity integrity mechanism
  VERIFY signing key ≠ any non-system-entity key

IF watchdog:
  RETRIEVE watchdog ENTITY
  VERIFY watchdog ≠ system entity
  VERIFY watchdog has produced ≥ 1 SIGNAL within audit_interval_ms
  VERIFY watchdog AUTHORIZATION chain does not pass through system entity

IF external_endpoint:
  FETCH endpoint
  VERIFY response signed, timestamp within detection_interval_ms
  VERIFY endpoint reachable from outside instance network

RETRIEVE dependency declaration
VERIFY system entity not in dependency graph
PASS if all checks pass.
```
**PHYSICAL AUDIT EXIT PA-003:** A human auditor with physical or administrative access to the deployment infrastructure must verify: (a) the liveness mechanism runs on infrastructure physically independent of the system entity (different host, different network segment, different power domain — at minimum one physical separation), (b) no undeclared runtime dependency exists (shared process, shared database, shared orchestrator). The auditor must inspect the actual deployment topology, not just the declared dependency graph. Produce a signed attestation with the infrastructure topology inspected and the independence finding. If any shared physical dependency is found, this is a finding.

**Dependencies:** None.

---

#### PROP-LIVENESS-002: Liveness Failure Containment

**Statement:** When system entity non-responsiveness is detected, governance mutations are suspended within one governance cycle without requiring the system entity to perform the suspension.

**Verifies:** P3

**Adversary model:** Silent Collapse (R4) — continued unmonitored governance during outage.

**Proof type:** Operational.

**Verification procedure (deterministic + physical audit exit):**
```
VERIFY suspension mechanism's executor ≠ system entity
RETRIEVE all governance_write acts during any recorded liveness failure period
VERIFY count = 0 OR all were rejected/rolled back
PASS if all checks pass.
```
**PHYSICAL AUDIT EXIT PA-004:** If no liveness failure has occurred in production, the implementation must execute a supervised simulation: suppress the system entity process and measure (a) time to detection, (b) time to governance mutation suspension, (c) whether any governance_write was accepted during the failure window. The simulation must be witnessed by the auditor or recorded with tamper-evident logging. Produce a signed attestation with simulation results. If containment did not activate within one governance cycle during simulation, this is a finding. Simulation must be repeated at least once per audit cycle.

**Dependencies:** PROP-LIVENESS-001.

---

#### PROP-LIVENESS-003: Liveness Recovery Integrity

**Statement:** Recovery from system entity liveness failure cannot be exploited to install a compromised system entity or bypass governance constraints.

**Verifies:** P3

**Adversary model:** Silent Collapse (R4) — attack via recovery; adversary triggers failure to exploit recovery procedure.

**Proof type:** Formal.

**Verification procedure (deterministic + physical audit exit):**
```
FOR each recorded liveness recovery event:
  VERIFY fresh liveness proof with post-recovery timestamp
  VERIFY reconciliation AUDIT_EXECUTION covering failure period
  VERIFY quorum authorization (≥ 2/3) on recovery GOVERNANCE_ACT
  VERIFY C1-C14 structural checks pass on post-recovery state
PASS if all checks pass.
```
**PHYSICAL AUDIT EXIT PA-005:** A human auditor must conduct a red-team exercise against the recovery procedure: (a) simulate a liveness failure, (b) attempt to exploit the recovery process to install a modified system entity or relax a governance constraint, (c) verify the recovery output state satisfies all propositions P1-P15. The exercise must be documented with the attack paths attempted and their outcomes. If any exploitation path succeeds, this is a finding. The exercise must be conducted at least once per audit cycle. If recovery has occurred in production, the auditor must additionally verify the production recovery event was not exploited by reviewing the reconciliation audit covering the failure period.

**Dependencies:** PROP-LIVENESS-001, PROP-LIVENESS-002.

---

#### PROP-LIVENESS-004: System Entity Observation

**Statement:** The system entity demonstrably observes the governance state stream (DRIFT_SIGNAL records, RESOURCE_CONSUMPTION records, SIGNAL ages, ATTESTATION statuses) and its monitoring actions are causally linked to observed state, not produced on a fixed schedule.

**Verifies:** P3

**Adversary model:** R4 variant — a system entity that publishes heartbeats and produces scheduled escalations but never observes governance state, passing all liveness checks while being effectively blind.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
RETRIEVE all escalation SIGNALs produced by system entity within audit_interval_ms
FOR each escalation:
  VERIFY escalation.details references a specific governance state observation
    (a specific DRIFT_SIGNAL id, SIGNAL id, ATTESTATION id, or RESOURCE_CONSUMPTION id)
  VERIFY the referenced record exists and its state matches the escalation trigger condition
  VERIFY escalation.produced_at is AFTER the referenced record's timestamp
    (causal ordering — observation preceded escalation)

# Schedule detection
COMPUTE intervals between consecutive system entity monitoring kernel_writes
VERIFY intervals are NOT constant (coefficient of variation > 0.05)
  # A system entity on a fixed timer produces constant intervals
  # A system entity reacting to governance state produces variable intervals

# State-reactive test
RETRIEVE any governance state change that should trigger escalation
  (e.g., SIGNAL older than orphan_signal_max_age_ms)
VERIFY system entity produced corresponding escalation within detection_interval_ms
  of the trigger condition becoming true

# Quiescent system check (when no escalation-worthy state exists)
RETRIEVE all monitoring kernel_write ATTRIBUTIONs produced by system entity within audit_interval_ms
VERIFY count >= minimum_monitoring_observations_per_cycle (KERNEL_PARAMETERS, default: 3)
  # Even when no escalations are warranted, the system entity must produce
  # monitoring kernel_writes demonstrating it scanned governance state.
  # Each monitoring kernel_write produces ATTRIBUTION records.
FOR each monitoring ATTRIBUTION:
  VERIFY attribution.details references governance state scanned
    (list of record types checked, count of records evaluated)
  # A monitoring kernel_write that scans zero records is vacuous

PASS if all checks pass.
```

**Dependencies:** PROP-LIVENESS-001.

---

#### Boundary 4: Governance Substance

*The spec governs form. These properties govern that form carries meaning.*

#### PROP-SUBSTANCE-001: Intent Distinguishability

**Statement:** The kernel's intent validation rejects tautological intents that are satisfied by any act.

**Verifies:** P5

**Adversary model:** Compliance theater (R6) — tautological templates satisfy every structural check.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
RETRIEVE all INTENT records within audit_interval_ms
FOR each INTENT:
  COMPUTE hash_objective = SHA-256(normalize(objective))
  # Duplicate detection
  GROUP by entity, VERIFY < 10% duplication rate
  # Tautology detection
  VERIFY objective does not contain self-referential patterns
  VERIFY success_criteria do not match completion-status patterns
  VERIFY constraints non-empty
  VERIFY scope_rationale similarity to scope description < 0.90
  # Counterfactual template test
  FOR each known tautological template:
    VERIFY Levenshtein distance > 30% of template length
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-SUBSTANCE-002: Paths-Not-Taken Substantiveness

**Statement:** Recorded alternatives are not negations of the objective and reference concrete governance state.

**Verifies:** P5, P6, P11

**Adversary model:** Compliance Theater (R6) — fabricated alternatives that satisfy structural checks.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
FOR each PATHS_NOT_TAKEN record:
  FOR each alternative:
    VERIFY reason_rejected token overlap with objective < 0.70
    VERIFY description is not a negation/inversion of objective
    VERIFY description references ≥ 1 concrete governance state element
  VERIFY all alternatives in same record have distinct hashes
COMPUTE cross-record unique_rate
VERIFY unique_rate > 0.60
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-SUBSTANCE-003: Measurement Discriminative Power

**Statement:** Every active measurement method produces both compliant and non-compliant results.

**Verifies:** P6

**Adversary model:** Captured measurement (R1) — method always returns within_threshold = true.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
FOR each active MEASUREMENT_METHOD:
  RETRIEVE DRIFT_SIGNAL records using this method
  IF count < 5: FLAG as insufficient_history, skip
  COMPUTE coefficient of variation
  VERIFY cv > 0.01
  VERIFY at least one within_threshold = false result exists
  VERIFY count(distinct rounded distances) > 1

  # Known-misaligned input test
  RETRIEVE method's declared test_vectors (required field on MEASUREMENT_METHOD)
  FOR each test_vector with expected_result = non_compliant:
    EXECUTE method against test_vector input
    VERIFY result = non_compliant  # method detects known misalignment
  FOR each test_vector with expected_result = compliant:
    EXECUTE method against test_vector input
    VERIFY result = compliant  # method does not false-positive on known alignment

PASS if all checks pass for methods with sufficient history.
```

**Dependencies:** None.

---

#### PROP-SUBSTANCE-004: Safety Assessment Rigor

**Statement:** Safety assessments meet minimum coverage, are independently performed, and do not uniformly pass.

**Verifies:** P5, P11

**Adversary model:** Compliance Theater (R6) — single-test-pass gaming.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
FOR each safety_assessment SIGNAL:
  VERIFY test_count ≥ hazard_category_count
  VERIFY hazard_categories covers all vocabulary entries
  VERIFY NOT (test_count = 1 AND violation_count = 0)
  VERIFY assessor_entity_id ≠ proponent_entity_id
COMPUTE pass_rate across all assessments in audit window
VERIFY pass_rate < 0.99 (if count > 10)
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-SUBSTANCE-005: Cross-Artifact Uniformity Detection

**Statement:** Governance artifacts are not systematically uniform — template reuse does not dominate the governance record.

**Verifies:** P5, P6, P11

**Adversary model:** Compliance Theater (R6) — automated theater at scale.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
COMPUTE intent_unique_rate = distinct_hashes / total_intents
VERIFY intent_unique_rate > 0.40
COMPUTE pnt_unique_rate = distinct_hashes / total_pnt_records
VERIFY pnt_unique_rate > 0.40
COMPUTE sa_unique_rate = distinct_hashes / total_assessments
VERIFY sa_unique_rate > 0.40
PER entity with > 5 intents:
  VERIFY entity_unique_rate > 0.50
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-SUBSTANCE-006: Hollow Instance Rejection

**Statement:** The kernel's substance verification rejects known-tautological governance artifacts.

**Verifies:** P5, P11, P14

**Adversary model:** Compliance Theater (R6) — validation of the substance checks themselves.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
DEFINE hollow_intent = {
  objective: "Execute the act described in this intent",
  success_criteria: [{ criterion: "Act completes", measurable_indicator: "status = completed" }],
  constraints: [],
  scope_rationale: "Scope matches authorization scope"
}
DEFINE hollow_paths_not_taken = {
  alternatives: [
    { description: "Do not perform this act",
      reason_rejected: "Would not achieve the objective",
      signal_ids_that_influenced: [] },
    { description: "Delay this act",
      reason_rejected: "Would not achieve the objective in time",
      signal_ids_that_influenced: [] }
  ],
  deliberation_summary: "The alternative of inaction was considered and rejected"
}
NOTE: This template has two alternatives to avoid rejection by the structural
minimum-count check (minimum two per decision point). Both alternatives must be
rejected by the SUBSTANCE checks (negation detection, concreteness check), not
by the structural count. If either alternative passes substance checks, the
hollow instance test has a gap.
SUBMIT hollow_intent to E14 substance checks → MUST reject
SUBMIT hollow_paths_not_taken to E14 substance checks → MUST reject
PASS if both are rejected.
```

**Dependencies:** PROP-SUBSTANCE-001, PROP-SUBSTANCE-002.

---

#### Boundary 5: External Infrastructure

*The spec governs what's produced. These properties govern where it lives and whether it can be trusted.*

#### PROP-INFRA-001: Verification Artifact Integrity

**Statement:** The verification artifact cannot be substituted by compromising a single infrastructure component.

**Verifies:** P13

**Adversary model:** DNS/CDN compromise (R5).

**Proof type:** Formal or operational.

**Verification procedure (deterministic):**
```
FOR each active INTEGRITY_MECHANISM:
  VERIFY count(locations) ≥ 2
  FOR each location: FETCH artifact, COMPUTE hash
  VERIFY all hashes identical
  FOR all pairs: VERIFY ≥ 1 infrastructure dimension differs
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-INFRA-002: Attestation Log Tamper Evidence

**Statement:** Any modification to a previously published log entry is detectable by an independent party within one governance cycle.

**Verifies:** P13

**Adversary model:** Verification Subversion (R5) — log tampering.

**Proof type:** Formal.

**Verification procedure (deterministic):**
```
SELECT random sample of 20 log entries
FOR each: VERIFY integrity_proof against hash chain
VERIFY monotonic timestamps
VERIFY published root matches computed root
VERIFY no entries removed (count non-decreasing)
VERIFY content of previously retrieved entries unchanged
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-INFRA-003: Infrastructure Change Governance

**Statement:** Infrastructure changes to INTEGRITY_MECHANISM or ATTESTATION_LOG are preceded by governance_write acts.

**Verifies:** P13

**Adversary model:** Verification Subversion (R5) — ungoverned infrastructure modification.

**Proof type:** Operational.

**Verification procedure (deterministic):**
```
FOR each endpoint:
  VERIFY TLS fingerprint matches declared fingerprint
  VERIFY DNS records match declared dns_hash
  IF mismatch: VERIFY governance_write act exists for the change
  VERIFY endpoint responds within 30 seconds
  VERIFY TLS version ≥ declared minimum
PASS if all checks pass.
```

**Dependencies:** PROP-INFRA-001, PROP-INFRA-002.

---

#### Boundary 6: Enforcement Correctness

*The spec defines enforcement rules. This property governs that the implementation's enforcement is correct.*

#### PROP-ENFORCEMENT-001: Enforcement Rule Behavioral Correctness

**Statement:** The implementation's enforcement rule logic correctly accepts valid inputs and rejects invalid inputs for each of E1-E16.

**Verifies:** P1–P15

**Adversary model:** R2 (Authorization Escalation), R6 (Compliance Theater) — subtle enforcement bugs allow proposition violations while structural checks pass.

**Proof type:** Empirical.

**Verification procedure (deterministic):**
```
FOR each enforcement rule E_n (E1 through E16):
  RETRIEVE implementation's declared test vectors for E_n
  VERIFY test vectors include:
    - At least 2 valid inputs that should be ACCEPTED
    - At least 2 invalid inputs that should be REJECTED
    - At least 1 boundary case (input at the exact threshold)
  FOR each test vector:
    EXECUTE E_n against test vector input
    VERIFY result matches expected outcome (accept/reject)

# Specification-mandated tests (implementation CANNOT choose these — they are fixed):

# E2 full chain traversal:
  SUBMIT act with valid one-hop authorization but disconnected chain (no path to bootstrap)
  VERIFY rejected at runtime, not deferred to audit

# E3 scope enforcement:
  SUBMIT act with scope exceeding executing entity's SCOPE by one level
  VERIFY rejected
  SUBMIT act with scope exactly matching executing entity's SCOPE boundary
  VERIFY accepted

# E7 mutual authorization detection:
  SUBMIT authorization chain with depth-3 mutual authorization
  VERIFY rejected
  SUBMIT authorization chain with depth-1 self-authorization
  VERIFY rejected

# E8 feedback channel integrity:
  CREATE entity receiving outputs, do NOT produce SIGNALs within signal_production_window_ms
  VERIFY non_participating_entity escalation produced
  CREATE SIGNAL older than orphan_signal_max_age_ms
  VERIFY orphan_signal_aged escalation produced

# E11 attestation completeness:
  PRODUCE ATTRIBUTION without corresponding ATTESTATION
  VERIFY attestation_provisional_expired escalation after attestation_provisional_window_ms
  PRODUCE ATTRIBUTION with ATTESTATION within window
  VERIFY no escalation

# E14 substance checks:
  SUBMIT tautological intent template (from PROP-SUBSTANCE-006 hollow test)
  VERIFY rejected by substance check, not structural check

# E15 proof expiry:
  MODIFY specification_version_hash to a non-matching value
  VERIFY all PROOF_OBLIGATION records transition to expired status

PASS if all test vectors produce correct results.
```

**Dependencies:** None. `depends_on_substrate: true`.

---

#### Boundary 7: Proof and Specification Integrity

*Properties that govern the proof system itself.*

#### PROP-PROOF-001: Proof Dependency Acyclicity

**Statement:** The proof dependency graph is acyclic. Invalidation cascades to all dependents within one governance cycle.

**Verifies:** P14

**Adversary model:** Captured Council (R1) — circular proof dependencies that prevent cascade invalidation, allowing a coalition to protect its proofs from challenge.

**Proof type:** Formal.

**Verification procedure (deterministic):**
```
CONSTRUCT directed graph from all PROOF_OBLIGATION depends_on_proof_ids
RUN cycle detection (DFS)
VERIFY no cycles exist
FOR each invalidated proof:
  RETRIEVE all dependents (transitive closure)
  VERIFY all dependents also in invalidated or expired status
  VERIFY cascade completed within one audit_interval_ms
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-PROOF-002: Specification Version Integrity

**Statement:** Every proof references the specification version it was validated against. Specification changes trigger re-validation of all affected proofs.

**Verifies:** P14

**Adversary model:** Captured Council (R1) — proofs attesting to properties that no longer exist or have changed meaning, exploiting specification lag to maintain stale governance.

**Proof type:** Formal.

**Verification procedure (deterministic):**
```
RETRIEVE current specification_version_hash from KERNEL_PARAMETERS
FOR each active PROOF_OBLIGATION:
  VERIFY proof.specification_version_hash = current specification_version_hash
  IF mismatch: VERIFY proof status = expired (not active)
VERIFY GENESIS specification_version_hash matches KERNEL_PARAMETERS
PASS if all checks pass.
```

**Dependencies:** None.

---

#### PROP-AVAILABILITY-001: Degraded Mode Continuity

**Statement:** The governance system continues to operate under infrastructure impairment. Records produced during degraded mode are flagged and subject to reconciliation.

**Verifies:** P15

**Adversary model:** Silent Collapse (R4), Verification Subversion (R5) — denial of service via infrastructure disruption causing governance halt.

**Proof type:** Operational.

**Verification procedure (deterministic + physical audit exit):**
```
VERIFY max_degraded_window_ms is declared in KERNEL_PARAMETERS
FOR each recorded degraded mode period:
  VERIFY escalation SIGNAL with type=degraded_mode_entered was produced
  VERIFY all ATTRIBUTION records in period have degraded_flag = true
  VERIFY governance acts continued to be processed
  VERIFY degraded period did not exceed max_degraded_window_ms
    (or: degraded_mode_timeout escalation was produced)
  IF infrastructure restored:
    VERIFY reconciliation audit covers the period
    VERIFY all deferred attestation operations completed
PASS if all checks pass.
```
**PHYSICAL AUDIT EXIT PA-006:** If no degraded mode period has occurred in production, the implementation must execute a supervised degraded mode simulation: disable the attestation log endpoint (or equivalent infrastructure component) and verify (a) the kernel enters degraded mode and produces the correct escalation SIGNAL, (b) governance acts continue to be processed with `degraded_flag = true`, (c) if the simulated outage exceeds `max_degraded_window_ms`, the timeout escalation fires and mutations are suspended, (d) on restoration, deferred attestation operations complete and a reconciliation audit runs. The simulation must be witnessed by the auditor or recorded with tamper-evident logging. Produce a signed attestation with simulation results. Simulation must be repeated at least once per audit cycle.

**Dependencies:** PROP-LIVENESS-001.

---

#### Boundary 8: Sincerity and Substrate

*Properties governing the physical audit exits that were previously outside the property framework.*

#### PROP-SINCERITY-001: Governance Deliberation Sincerity

**Statement:** Governance artifacts that pass all deterministic substance checks represent genuine deliberation when evaluated by a human auditor against a defined rubric.

**Verifies:** P2, P5, P14

**Adversary model:** R6 (Compliance Theater) — sophisticated theater that passes automated checks by producing unique, varied, well-structured artifacts that are substantively hollow.

**Proof type:** Operational (physical audit).

**Verification procedure (physical audit exit PA-007):**
The full procedure is defined in the Verification Coverage section (PA-007). The rubric evaluates five dimensions: specificity, decision cost, constraint engagement, temporal awareness, and falsifiability. An artifact with 2 or more dimension failures is "suspected theater." More than 1 of 5 sampled artifacts rated "suspected theater" is a finding.

**Dependencies:** PROP-SUBSTANCE-001, PROP-SUBSTANCE-002. `depends_on_substrate: true`.

---

#### PROP-SUBSTRATE-001: Substrate Trust Declaration Accuracy

**Statement:** The implementation's declared substrate trust model matches the actual deployment, verified by physical inspection.

**Verifies:** P13

**Adversary model:** R3 variant (Bootstrap Substitution at the infrastructure level) — a declared TEE that does not exist, a declared measured boot that is not enforced, a declared clock source that is not connected.

**Proof type:** Operational (physical audit).

**Verification procedure (physical audit exit PA-008):**
The full procedure is defined in the Trust Boundary section (PA-008). The auditor physically inspects the deployment and verifies each element of the `substrate_trust_declaration` against reality.

**Dependencies:** None. This is the substrate root — all other properties implicitly depend on it via the `depends_on_substrate: true` declaration.

---

## Layer 7 — Verification Architecture

#### Layer 7a — Deterministic Verification (the gate)

A script. Not an LLM. Checks everything checkable without judgment.

The deterministic verifier runs after every audit cycle. It is triggered automatically. Its output is a structured pass/fail report that becomes part of the AUDIT_EXECUTION record.

**Inputs:**
- The audit output directory (epoch manifest, evidence files, findings)
- KERNEL_PARAMETERS (for parameter bounds, thresholds, intervals)
- All active PROOF_OBLIGATION records
- The Property Catalog (this specification's Layer 6)

**Checks:**
```
STRUCTURAL INTEGRITY
□ Epoch manifest exists and is valid JSON (if epoch protocol is in use)
□ Every methodology corpus file hash matches manifest
□ Every subject corpus file hash matches manifest
□ Every agent evidence record contains required attestation blocks
□ All expected output files exist
□ SARIF findings cross-reference correctly with domain outputs

PARAMETER OUTCOMES (PROP-PARAM-001, 002, 003)
□ All parameters within declared bounds
□ No multi-parameter amendments
□ No excessive amendment frequency
□ Trajectory monitoring values within thresholds

ENTITY IDENTITY (PROP-IDENTITY-001, 002, 003)
□ All active entities have non-null identity_attestation_ref
□ No principal binding hash collisions
□ No expired identity attestations on active entities
□ Identity verification succeeds for sampled entities

SYSTEM ENTITY LIVENESS (PROP-LIVENESS-001)
□ Liveness mechanism active
□ Most recent liveness proof within required interval
□ Dependency graph excludes system entity
□ No unresolved liveness failure escalations

GOVERNANCE SUBSTANCE (PROP-SUBSTANCE-001 through 006)
□ No tautological intent patterns
□ Intent duplicate rate < 10% per entity
□ Paths-not-taken pass negation and concreteness checks
□ Measurement methods show non-trivial variance
□ Safety assessments meet coverage and independence requirements
□ Cross-artifact uniformity below thresholds
□ Hollow instance test: known-bad templates rejected

EXTERNAL INFRASTRUCTURE (PROP-INFRA-001, 002, 003)
□ Verification artifacts consistent across locations
□ Attestation log integrity chain valid
□ Infrastructure changes preceded by governance_write acts
□ All endpoints reachable with valid TLS

PROOF AND SPECIFICATION INTEGRITY (PROP-PROOF-001, 002, PROP-AVAILABILITY-001)
□ Proof dependency graph acyclic
□ All active proofs reference current specification version
□ Cascade invalidation complete for any invalidated proofs
□ Degraded mode records flagged and reconciled

PROOF STATUS
□ Every required property has an active proof
□ No expired proofs on required properties
□ No unresolved proof challenges
```

**Output:** PASS or FAIL per check with details. Final line: AUDIT VALID or AUDIT INVALID. Exit code 0 for valid, 1 for invalid.

**If any check fails: the implementation is non-conforming. No judgment required. No LLM opinion needed. The math is the verdict.**

---

#### Layer 7b — Physical Audit (the field verification)

A human auditor with physical or administrative access to the deployment infrastructure. Runs only if 7a passes. Executes the eight physical audit exits defined in the Property Catalog and Trust Boundary section:

| Exit | Property | Verification |
|------|----------|-------------|
| PA-001 | PROP-IDENTITY-001 | Identity binding mechanism soundness — inspect binding infrastructure, verify Sybil resistance for deployment context |
| PA-002 | PROP-IDENTITY-003 | Revocation procedure timing — execute or simulate timed revocation, verify completion within one governance cycle |
| PA-003 | PROP-LIVENESS-001 | Infrastructure independence — physically inspect deployment topology, verify no undeclared shared dependencies |
| PA-004 | PROP-LIVENESS-002 | Containment simulation — suppress system entity, measure detection and suspension timing |
| PA-005 | PROP-LIVENESS-003 | Recovery red-team — attempt exploitation of recovery procedure, verify all propositions hold post-recovery |
| PA-006 | PROP-AVAILABILITY-001 | Degraded mode simulation — disable infrastructure component, verify degraded mode behavior and reconciliation |
| PA-007 | PROP-SINCERITY-001 | Governance artifact sampling — select 5 artifacts that passed deterministic checks, evaluate whether they represent genuine deliberation |
| PA-008 | PROP-SUBSTRATE-001 | Substrate attestation — verify declared trust model matches actual deployment, check TEE/boot/clock/crypto provenance |

Each physical audit exit produces a signed attestation that becomes a governance record. The attestation must reference the specific property, the infrastructure inspected or simulation executed, and the finding (pass/fail with rationale). Failure to produce any required physical audit attestation within one audit cycle is a finding.

These eight verifications plus the five auditor accountability requirements (rotation, independence, cross-verification, specificity, legal accountability) are the complete scope of the physical audit layer. Everything else has been verified deterministically by Layer 7a. The terminal boundary — hardware fidelity and human conspiracy — exits to the physical and legal systems respectively.

---

#### Layer 7c — Meta-Verification

A structured review of Layer 7a and 7b outputs. May be performed by the implementation operator, a second auditor, or the same physical auditor after a cooling-off period (minimum 7 days). Produces a META_VERIFICATION_REPORT as a governance record.

**Deterministic meta-checks (automated):**
```
RETRIEVE Layer 7a output
VERIFY every property in the Property Catalog has a corresponding check result
VERIFY every check result is PASS or FAIL (no missing, no "skipped")
VERIFY Layer 7a was run within the current audit cycle (timestamp check)
RETRIEVE all PA attestations (PA-001 through PA-008)
VERIFY count = 8 (no missing exits)
FOR each PA attestation:
  VERIFY attestation references a specific property or verification target
  VERIFY attestation contains procedure-specific details (not generic)
  VERIFY auditor independence declaration is present
  VERIFY auditor identity differs from any governance entity in the instance
FOR PA attestations in current and prior cycle:
  VERIFY auditor identity has rotated (no same auditor for > 2 consecutive cycles)
```

**Judgmental meta-checks (human review):**
1. For each PA attestation rated "pass": does the documented evidence support the finding? Read the attestation details and assess whether the procedure was actually performed or merely claimed.
2. For PA-005 (red-team): were the attack paths attempted substantive? Did the red-team exercise the adversary models declared on the properties, or did it test only trivial scenarios?
3. For PA-007 (substance sincerity): does the rubric scoring appear calibrated? Are the "genuine" ratings defensible given the artifact content?
4. Are there proofs the deterministic verifier accepted that appear to satisfy the letter but not the spirit of the verification procedure?

**Output format:** META_VERIFICATION_REPORT with: deterministic meta-check results (pass/fail per check), judgmental meta-check assessments (finding per question with rationale), overall verdict (pass / pass with observations / fail), auditor identity and independence declaration.

**Pass criteria:** All deterministic meta-checks pass AND no judgmental meta-check produces a finding rated "material concern." An observation (minor concern, documented for next cycle) does not block a pass.

**Regress termination:** Layer 7c is the final verification layer. Its scope is finite and enumerable (the checks listed above). Its output is a governance record subject to the same attestation and integrity requirements as all governance records. No Layer 7d exists. Third-party inspection of the META_VERIFICATION_REPORT is the external check — any party can read the report and assess whether 7c's findings are reasonable. This is the explicit regress termination point for the verification architecture. The regress terminates at third-party inspection of a finite, structured, published report.

---

## Layer 8 — Adoption

#### Brownfield Adoption Protocol

An existing governance system that was not built to this specification may adopt it through the following governed process. This is the only supported adoption path for systems with pre-existing governance records.

1. **Conformance declaration.** The adopting system executes a bootstrap-equivalent GOVERNANCE_ACT declaring intent to conform. This act must be authorized by supermajority (≥ 2/3) of all recognized entities. It establishes: GENESIS records (or equivalents mapped from the existing system), KERNEL_PARAMETERS with all required fields, and the `specification_version_hash` referencing this specification. Existing governance records produced before the conformance declaration are not retroactively governed by this specification — they are historical artifacts. The conformance boundary begins at the declaration.

2. **Field population.** The following must be established via governance_write acts within two audit cycles of the conformance declaration:
   - KERNEL_PARAMETERS: `identity_binding_model`, `system_entity_liveness_model`, `parameter_bounds`, `max_degraded_window_ms`, `proof_expiry_max_cycles`, `approved_algorithms`
   - All active ENTITY records: `identity_attestation_ref`
   - All active INTEGRITY_MECHANISM records: `infrastructure_security`, `public_verification_artifact_locations`, `algorithm`
   - ATTESTATION_LOG: `infrastructure_security`, `technology_requirements`

3. **Initial proof submission.** For each property in the Property Catalog, a PROOF_OBLIGATION must be submitted via governance_write act. Initial proofs carry an adoption exemption: they may be submitted in a batch governance_write (waiving E14's single-parameter-per-amendment rule for proofs only), and they enter `submitted` status with a two-cycle verification window (rather than the standard one-cycle).

4. **Adoption audit.** A full audit (C1-C22) must be completed within two audit cycles of the conformance declaration. This audit verifies all required fields are populated, runs the deterministic verifier, and validates initial proofs. Pre-declaration governance records are out of scope — the audit evaluates only the system's current state and post-declaration operations.

5. **Adoption completion.** When the adoption audit passes, the system is conforming. Adoption exemptions expire. All subsequent operations follow standard rules.

A system that does not complete adoption within three audit cycles of the conformance declaration reverts to non-conforming status. The conformance declaration itself remains in the governance record as a historical artifact.

#### Greenfield Instantiation

A new system built to this specification from inception follows the standard bootstrap process (E9). No adoption protocol is needed. GENESIS records, KERNEL_PARAMETERS, and initial PROOF_OBLIGATION records are established at bootstrap. The first audit cycle validates conformance.

---

## Specification Integrity and Self-Governance

This specification is content-addressed. The specification hash is computed as SHA-256 of the complete specification text (this document). Every GENESIS record, KERNEL_PARAMETERS record, and PROOF_OBLIGATION record references this hash.

#### Amendment Governance

P14 requires every instance-defined choice to be proven. P14 applies to implementations, not to the specification itself — the specification is not an entity and does not make "instance-defined choices." However, the specification governs its own evolution to prevent amendments from introducing the inconsistencies it guards against at the instance level.

**Amendment authority.** A specification amendment may only be initiated by: (a) an audit finding against the specification (the audit identifies a gap, contradiction, or incompleteness), or (b) a governance proposal from a conforming instance's governance process (the instance identifies a need the specification does not address). In both cases, the amendment source is recorded in the provenance chain.

**Amendment process:**
1. **Trigger.** An audit finding or governance proposal identifies the need for amendment. The trigger is recorded as the `amendment_source`.
2. **Draft.** The amendment is drafted with: (a) the specific text changes, (b) the rationale referencing the trigger, (c) an impact analysis identifying which properties, enforcement rules, and audit criteria are affected, (d) the executor (who drafted the amendment) and authorizer (who approved it).
3. **Review.** The amendment must be reviewed by at least one conforming instance's audit process. The review verifies that the amendment does not introduce contradictions, does not invalidate properties without replacement, and does not weaken enforcement.
4. **Acceptance.** Amendment acceptance requires a supermajority (≥ 2/3) of conforming instances to update their `specification_version_hash` within two audit cycles. An amendment not accepted by supermajority within this window is void.
5. **Provenance.** The provenance chain records: prior hash, new hash, amendment source (audit_id or governance_act_id), executor, authorizer, rationale summary, acceptance count.

**Self-consistency boundary.** The specification satisfies P13 (external verifiability) for its own text via content-addressing. It satisfies P7 (immutability) via content-addressing and provenance chain. It satisfies P1 (attribution) via the executor/authorizer fields in the provenance entry. It does not satisfy P14 (its own design choices are not proven in the formal sense) because the specification is a document, not an entity — P14 governs entities' mechanism choices, not the specification's design decisions. This boundary is explicit: the specification governs instances; instances' audit processes govern the specification's evolution; this is a designed feedback loop, not self-governance (P10), because the specification is not an entity and the audit process is performed by independent entities.

**Regress termination for specification self-governance.** This cycle — specification governs instances, instances audit the specification — is broken by the same mechanism that breaks the audit-governs-audit cycle (F-CIRC-001): the deterministic layer. An amendment's impact analysis can be mechanically verified (do the changed text sections maintain cross-reference integrity? do affected properties still have valid verification procedures?). The deterministic verifier, extended for specification analysis, provides the external reference point.

When the specification is amended:
1. A new specification hash is computed.
2. All instances claiming conformance must update `specification_version_hash` via governance_write.
3. All PROOF_OBLIGATION records with non-matching `specification_version_hash` transition to `expired`.
4. Expired proofs must be re-validated within two audit cycles.
5. The specification maintains a provenance chain.

**Specification provenance chain format:**
```
version_N-1 → [hash_N-1] → version_N → [hash_N, amendment_source, executor, authorizer, rationale, acceptance_count]
```

---

## Trust Boundary

This specification governs the governance layer and verifies down to the infrastructure layer. Two boundaries remain: the computational substrate below and the human auditor above. Both have verification procedures.

#### Below: The Computational Substrate

The specification's guarantees depend on faithful execution by the underlying hardware, firmware, operating system, runtime, cryptographic implementations, and clock. A compromised substrate can defeat every guarantee while every audit passes clean.

**PHYSICAL AUDIT EXIT PA-008: Substrate Attestation.** The implementation must declare its substrate trust model in KERNEL_PARAMETERS as `substrate_trust_declaration`, specifying: (a) whether the kernel executes within a Trusted Execution Environment (TEE) with hardware attestation, (b) the hardware attestation mechanism if present (e.g., Intel SGX, AMD SEV, ARM TrustZone), (c) the boot integrity chain (measured boot, secure boot, or none), (d) the clock synchronization mechanism (NTP with authenticated sources, PTP, GPS, or none), (e) the cryptographic implementation provenance (library name, version, FIPS 140 validation status or equivalent).

A human auditor with physical access must verify at least once per audit cycle: (a) the declared substrate trust model matches the actual deployment, (b) if a TEE is declared, remote attestation produces a valid quote binding the trust kernel binary to the hardware, (c) if measured boot is declared, the boot measurement log is consistent with the declared firmware and OS versions, (d) the clock source is operational and within declared accuracy bounds, (e) the cryptographic library version matches the declaration and has no known vulnerabilities at or above the declared security level.

If no TEE is present, the auditor must document this as an explicit trust assumption in the PA-008 attestation: "This implementation trusts an unattested substrate. The following components are trusted without hardware verification: [list]." This is not a finding — it is a declared and attested risk acceptance. An implementation with an unattested substrate is conforming if the risk acceptance is documented. An implementation that claims TEE attestation but fails the remote attestation check is non-conforming.

The specification does not mandate a TEE. It mandates that the substrate trust model is declared, verified against reality by a human, and attested as a governance record. The gap between "declared" and "actual" is closed by physical inspection. The gap between "actual" and "faithful" — whether the hardware itself is honest — is the terminal boundary. It is addressable only by hardware attestation protocols that are themselves subject to the same regress. The specification terminates this regress by requiring the implementation to declare its hardware trust anchor and the auditor to verify the declaration matches reality. Beyond that: the laws of physics.

#### Above: The Human Auditor

The specification's physical audit exits depend on a competent, honest, independent human producing truthful attestations. A captured, negligent, or colluding auditor can sign pass attestations for a compromised system.

**Auditor independence and accountability requirements:**

1. **Rotation.** No auditor may perform physical audit exits for the same implementation for more than two consecutive audit cycles. Rotation is enforced by recording auditor identity in each PA attestation. C20 verification must check that the auditing entity for PA attestations has rotated per this requirement.

2. **Independence.** The physical auditor must have no financial, employment, or governance relationship with the implementation operator beyond the audit engagement. The auditor's independence declaration must be recorded as part of each PA attestation. An auditor who is also a governance entity in the implementation they are auditing is non-conforming.

3. **Cross-verification.** For implementations with `risk_classification` of critical or high on any deployment authorization: PA-001 through PA-007 must be performed by at least two independent auditors. Both must produce attestations. Discrepancies between attestations are findings that must be resolved by a third auditor before the audit cycle closes.

4. **Attestation specificity.** PA attestations must reference the specific infrastructure inspected, the specific procedure executed, the specific measurements taken, and the specific finding. Generic attestations ("I verified the infrastructure is independent") are non-conforming. The attestation must contain enough detail that a subsequent auditor could reproduce the verification.

5. **Legal accountability.** The PA attestation must reference the auditor engagement agreement under which the auditor operates. The implementation operator is responsible for establishing auditor engagement terms that include appropriate accountability mechanisms under the governing jurisdiction. The attestation records that such an agreement exists — it does not itself constitute the agreement. This is not a specification enforcement mechanism — it is an exit to the legal system, which is the external accountability mechanism for human honesty.

The specification does not verify the human's sincerity. It creates conditions where insincerity has consequences: rotation prevents capture from compounding, independence prevents conflicts, cross-verification prevents unilateral deception, specificity prevents generic sign-offs, and legal accountability provides external enforcement. The gap between "the human signed it" and "the human did the work" is addressed by institutional mechanisms, not by the specification. The specification's role is to require those mechanisms exist and to produce the records that make institutional accountability possible.

#### The Terminal Boundary

The specification verifies every property within its scope. Physical audit exits extend verification to the infrastructure and human layers. PA-008 extends verification to the substrate declaration. Auditor controls extend accountability to the human.

Two things remain genuinely beyond verification:

1. **Whether the hardware is physically faithful.** A hardware backdoor at the transistor level defeats all software, firmware, and attestation mechanisms. This is addressable only by hardware supply chain integrity — a problem shared by every computational system, not specific to governance.

2. **Whether the human is ultimately sincere despite all institutional controls.** Rotation, independence, cross-verification, specificity, and legal liability reduce the probability of undetected auditor capture to the level of organized conspiracy. Organized conspiracy against a governance system is addressable only by law enforcement — again, a problem shared by every institutional system, not specific to this specification.

These two — physics and conspiracy — are the terminal boundaries of any governance system, digital or institutional. The specification does not solve them. It reduces every other boundary to a verification procedure and exits to the physical and legal systems for these two. That is the limit.

#### Substrate Extension Path

The trust boundary is designed to move. The current specification verifies down to the infrastructure layer via PA-008 (substrate declaration and inspection). The architecture supports extending verification to the BIOS, TPM, and firmware layers without restructuring anything above Layer 6.

**How to extend.** Add properties to the Property Catalog:

- **PROP-SUBSTRATE-001 (Boot Chain Verifiability):** The implementation's boot sequence produces a measurement chain from BIOS through OS to trust kernel binary, verifiable via TPM-anchored remote attestation. The deterministic verifier retrieves the TPM quote, validates PCR values against the declared boot chain, and confirms the trust kernel binary hash matches the measurement log.
- **PROP-SUBSTRATE-002 (Firmware Integrity):** BIOS/UEFI firmware version is declared, its hash is recorded in the attestation log at boot, and firmware changes trigger a governance_write act. Unattested firmware changes produce an escalation SIGNAL.
- **PROP-SUBSTRATE-003 (Clock Attestation):** The system clock is anchored to an external time source via authenticated protocol. Clock drift beyond a declared threshold produces an escalation SIGNAL. The deterministic verifier cross-references governance record timestamps against the external time source.

**Why the structure holds.** The Property Catalog is the extension point. P14 already requires every instance-defined choice to be proven — the substrate trust model is an instance-defined choice. Adding substrate properties exercises P14, it does not modify it. The proof lifecycle, verification procedures, PA exit pattern, deterministic verifier, and enforcement rules all operate on properties generically. Adding properties is adding tests to a test suite — the framework does not change, the coverage does.

**What changes at extension time.** The specification hash changes (new properties added). E15 triggers: all proofs with non-matching specification version hashes expire. Implementations re-prove. Existing proofs for PROP-PARAM-001 through PROP-AVAILABILITY-001 are re-validated mechanically (their content has not changed). Only the new substrate properties require new proofs. PA-008's inspection procedure deepens. The deterministic verifier gets TPM quote validation added to its check list. Everything above Layer 6 is untouched.

**Dependency graph placement.** When substrate properties are added, they must sit at the bottom of the proof dependency graph. Every existing property implicitly depends on faithful substrate execution. If PROP-SUBSTRATE-001 is invalidated (the boot chain is compromised), every proof that depends on faithful execution — which is all of them — must cascade-invalidate. The dependency graph should reflect this: substrate properties as roots, all other properties as transitive dependents. This means a substrate compromise invalidates the entire proof set, which is the correct behavior — if the substrate is compromised, nothing above it is trustworthy.

This is a design decision deferred to extension time, not a structural limitation. The cascade invalidation mechanism (PROP-PROOF-001, E15) already handles it. Adding substrate properties as dependencies of root properties is a governance_write act that modifies the proof dependency graph within the existing architecture.

**The terminal boundary moves but does not disappear.** Extending to TPM pushes the terminal boundary from "the spec exits to physics at the OS layer" to "the spec exits to physics at the silicon layer." The boundary still terminates at transistor-level fidelity and hardware supply chain integrity. No software specification can verify below that. But each extension reduces the attack surface between the specification's verification and the physical world.

---

## Verification Coverage

Within the trust boundary, every property is verified through one of three layers:

- **Deterministic (Layer 7a):** 17 of 25 properties are fully verified by hash comparison, arithmetic, pattern matching, and structural checks. No human required.
- **Deterministic + physical (Layer 7b hybrid):** 6 properties have deterministic cores with mandatory physical audit exits (PA-001 through PA-006) for infrastructure verification that cannot be checked by software alone.
- **Physical only (Layer 7b):** 2 properties are verified entirely by physical audit — PA-007 (substance sincerity via artifact sampling) and PA-008 (substrate integrity via hardware and deployment inspection). These have no deterministic component.
- All 8 physical audit exits produce signed attestations that are governance records.
- **Meta-verification (Layer 7c):** Verifies completeness of both layers above.

The trust boundary itself is addressed by PA-008 (substrate) and auditor accountability controls (human). The terminal boundary — transistor-level hardware fidelity and organized human conspiracy — exits to physics and law.

**Implementation correctness at the code level** is verified through the proof obligation model. Each implementation choice must satisfy a named property. The property's verification procedure tests the implementation's behavior (parameter rejection at boundaries, duplicate detection, heartbeat timing, tautology rejection). These are behavioral tests, not code reviews. A correct implementation passes the tests. An incorrect one fails. If the tests are insufficient, that is a finding against the property's verification procedure — not an unverifiable boundary.

**Adversary capabilities** are scoped by the adversary model declared on each property. An adversary operating outside all declared models is addressed by **PA-005 (recovery red-team)** — the physical audit exit that explicitly attempts exploitation paths beyond the standard models. If the red-team discovers an attack path not covered by any property's adversary model, the finding triggers a specification amendment to add a new property covering that path. The specification evolves; the boundary does not remain open.

**The specification's completeness** is addressed by **C20 (are all properties proven?)** and the proof challenge mechanism. Any entity may challenge any proof. Any entity may propose a new property via governance_write. The property catalog is extensible through governed acts — it is not a fixed list with acknowledged gaps. If a governance failure mode exists that no property covers, the failure itself is the evidence for a new property proposal. The specification does not declare itself complete — it declares a mechanism for becoming more complete.

**Sophisticated compliance theater** is addressed by the substance properties (PROP-SUBSTANCE-001 through 006) deterministically, and by the physical audit exits for sincerity:

**PHYSICAL AUDIT EXIT PA-007:** At least once per audit cycle, a human auditor must select 5 governance artifacts (INTENTs, PATHS_NOT_TAKEN, SAFETY_ASSESSMENTs) that passed all deterministic substance checks and evaluate them against the following rubric:

**Deliberation quality rubric (all five dimensions must be evaluated per artifact):**
1. **Specificity** — Does the artifact reference concrete governance state (specific entities, parameters, scopes) rather than generic categories? Score: specific (pass) / generic (fail).
2. **Decision cost** — Does the chosen path differ meaningfully from the rejected alternatives? Could a random selection have produced the same outcome? Score: differentiated (pass) / interchangeable (fail).
3. **Constraint engagement** — Do the declared constraints actually constrain the act? Would the act proceed identically without them? Score: constraining (pass) / vacuous (fail).
4. **Temporal awareness** — Does the artifact reference governance state that existed at the time of deliberation? Or does it use timeless abstractions that could apply to any act at any time? Score: temporally grounded (pass) / atemporal (fail).
5. **Falsifiability** — Could the success criteria produce a "not met" result? Is there a defined state of the world where the intent would be unsatisfied? Score: falsifiable (pass) / tautological (fail).

Each artifact is scored pass/fail on each dimension. An artifact with 2 or more fails is rated "suspected theater." An artifact with 0-1 fails is rated "genuine." The auditor must document: (a) the artifacts selected, (b) the score per dimension per artifact, (c) the overall rating, (d) the rationale for any fail scores. If more than 1 of 5 sampled artifacts is rated "suspected theater," this is a finding. The sample must include artifacts from at least 3 different entities. This attestation is a governance record.

Within the specification's scope, there is a verification procedure for every property. The trust boundary is not a gap — it is a defined exit to external verification systems (hardware attestation for the substrate, institutional accountability for the human). The terminal boundary — physics and conspiracy — is where all governance systems, digital and institutional, terminate.

---

---

## Summary

| Layer | Purpose | Key Contents |
|-------|---------|-------------|
| 0 — Definitions | Vocabulary | Governance record, property, proof obligation, proof, verification procedure, degraded mode |
| 1 — Propositions | Claims | P1-P15: attribution, governance, bidirectionality, authorization, intent, drift, immutability, scope, loop closure, self-authorization prevention, downstream participation, workflows, attestation, proof obligations, availability |
| 2 — Schema Primitives | Data model | GENESIS, KERNEL_PARAMETERS, ENTITY, SIGNAL, PROOF_OBLIGATION, INTENT, ACT, AUTHORIZATION, ATTRIBUTION, ATTESTATION, and supporting primitives |
| 3 — Enforcement Rules | Validation | E1-E13 (structural), E14 (parameter + substance), E15 (proof obligations), E16 (liveness + identity + infrastructure + degraded mode) |
| 4 — Audit Criteria | Verification | C1-C14 (structural), C15-C22 (parameter bounds, identity, liveness, substance, infrastructure, proofs, deterministic verifier, degraded reconciliation) |
| 5 — Attestation | External verifiability | Attestation lifecycle, external verification, proof obligation summary |
| 6 — Property Catalog | Proof obligations | 25 properties across 8 boundaries with deterministic verification procedures |
| 7 — Verification Architecture | Trust model | Deterministic gate (7a), physical audit (7b), meta-verification (7c) |
| 8 — Adoption | Onboarding | Brownfield adoption protocol, greenfield instantiation |

**Property count:** 25 across 8 boundaries (3 parameter + 3 identity + 4 liveness + 6 substance + 3 infrastructure + 1 enforcement + 3 proof/spec/availability + 2 sincerity/substrate).

**Deterministic coverage:** 17 of 25 properties are fully deterministic. 6 have a deterministic core with mandatory physical audit exits (PA-001 through PA-006). 2 are entirely physical: PA-007 (substance sincerity, PROP-SINCERITY-001) and PA-008 (substrate integrity, PROP-SUBSTRATE-001). Every property is verified. The terminal boundary exits to physics and law.

**Physical audit exits:** 8 total (PA-001 through PA-008). Each produces a signed attestation that is a governance record. Missing attestations are findings. The attestations verify that the human performed the procedure. Auditor rotation, independence, cross-verification, specificity requirements, and legal accountability address the human sincerity boundary.

**The tesseract closes:** Spec defines properties → Implementation proves them → Audit verifies proofs → Meta-audit verifies the audit → Deterministic layer terminates the digital regress at math → Physical audit exits terminate the infrastructure regress at human inspection → Auditor controls terminate the human regress at institutional accountability → Terminal boundary: physics and conspiracy, which exit to the physical and legal systems.
