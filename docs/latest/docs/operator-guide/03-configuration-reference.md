# Configuration Reference

This document covers all KernelParameters fields, the parameter amendment procedure, interdependencies, bounds, and trajectory limits.

## KernelParameters Fields

KernelParameters is a single governed record per instance, established at bootstrap and amendable only through governance acts (one parameter per amendment, enforced by E14).

### Time-Based Parameters

| Field | Type | Constraint | Description |
|-------|------|------------|-------------|
| `audit_interval_ms` | i64 | > 0; bounds: [60000, 86400000] | Maximum time between full audit cycles. Default 3600000 (1 hour). |
| `minimum_intent_execution_interval_ms` | i64 | > 0 | Minimum elapsed time between intent declaration and act execution. Prevents reflexive execution. |
| `orphan_signal_max_age_ms` | i64 | >= audit_interval_ms * 2 | Maximum age for a pending signal before it is flagged as orphaned. |
| `signal_production_window_ms` | i64 | >= minimum_intent_execution_interval_ms * 3 | Window within which an active entity must produce at least one signal. |
| `attestation_provisional_window_ms` | i64 | >= 30000 (30 seconds) | Maximum time an attestation may remain in Provisional status before escalation. |
| `max_degraded_window_ms` | i64 | > 0; >= audit_interval_ms | Maximum time the instance may operate in degraded mode before escalation. |

### Numeric Parameters

| Field | Type | Constraint | Description |
|-------|------|------------|-------------|
| `drift_threshold_default` | f64 | Immutable without supermajority (2/3 of active entities) | Default threshold for drift signal distance. Signals exceeding this threshold trigger output gating and escalation. |
| `proof_expiry_max_cycles` | u32 | >= 1; default 2 | Number of audit cycles before an active proof obligation expires. |
| `minimum_monitoring_observations_per_cycle` | u32 | >= 1; default 3 | Minimum number of monitoring observations required per audit cycle. |

### Vocabulary and Reference Fields

| Field | Type | Constraint | Description |
|-------|------|------------|-------------|
| `resource_type_vocabulary` | Vec\<String\> | Non-empty before instance accepts acts | Controlled vocabulary of resource types (e.g., "compute", "storage", "bandwidth"). |
| `cost_unit_vocabulary` | Vec\<String\> | -- | Controlled vocabulary of cost units. |
| `hazard_category_vocabulary` | Vec\<String\> | Non-empty before safety_assessment signals | Controlled vocabulary of hazard categories. Safety assessments must cover all entries. |
| `approved_algorithms` | Vec\<String\> | Non-empty | Cryptographic algorithms meeting 128-bit security, 256-bit hash output minimum. |
| `genesis_hash_algorithm` | String | Non-empty; must match both GENESIS records | Hash algorithm used for genesis record integrity. |
| `specification_version_hash` | String | Must match GENESIS specification_version_hash | SHA-256 of the governance specification document. |

### Infrastructure Reference Fields

| Field | Type | Description |
|-------|------|-------------|
| `default_measurement_method_id` | MeasurementMethodId | FK to the default drift measurement method. |
| `default_resource_measurement_method_ids` | HashMap\<String, ResourceMeasurementMethodId\> | Map of resource_type -> method_id. Must cover every resource_type_vocabulary entry. |
| `attestation_integrity_mechanism_id` | IntegrityMechanismId | FK to the integrity mechanism used for attestation signing. |
| `attestation_log_id` | AttestationLogId | FK to the attestation log configuration. |
| `attestation_schema_id` | AttestationSchemaId | FK to the attestation schema. |

### Model Fields

| Field | Type | Description |
|-------|------|-------------|
| `identity_binding_model` | IdentityBindingModel | How entities are bound to principals. Contains binding_type (cryptographic_key, institutional_attestation, biometric, composite), uniqueness_guarantee, verification_procedure. |
| `system_entity_liveness_model` | SystemEntityLivenessModel | Liveness mechanism configuration. Contains mechanism_type (heartbeat, watchdog, external_endpoint), detection_interval_ms (<= audit_interval_ms / 4), dependency_declaration (must not include system entity). |
| `substrate_trust_declaration` | SubstrateTrustDeclaration | Declares TEE presence, boot integrity chain, clock synchronization, crypto implementation provenance. |

### Governance Tracking Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | KernelParametersId | Primary key. |
| `established_via_act_id` | GovernanceActId | FK to the bootstrap governance act. |
| `last_amended_via_act_id` | Option\<GovernanceActId\> | FK to the most recent amending governance act. Null until first amendment. |
| `parameter_bounds` | ParameterBounds | Map of { parameter_name -> { floor, ceiling } } for every amendable time-based parameter. |

## Parameter Bounds

The `parameter_bounds` field contains declared floor and ceiling values for each amendable time-based parameter. The kernel validates `floor <= ceiling` for every entry. E14 rejects amendments that would set a value outside the declared bounds.

### Default Bounds

| Parameter | Floor | Ceiling |
|-----------|-------|---------|
| `audit_interval_ms` | 60000 (1 min) | 86400000 (24 hours) |
| `minimum_intent_execution_interval_ms` | 1 | 86400000 |
| `orphan_signal_max_age_ms` | 120000 (2 min) | 172800000 (48 hours) |
| `signal_production_window_ms` | 3 | 259200000 (72 hours) |
| `attestation_provisional_window_ms` | 30000 (30 sec) | 86400000 (24 hours) |
| `max_degraded_window_ms` | 60000 (1 min) | 604800000 (7 days) |

The bounds themselves are part of the KernelParameters record and are subject to the same amendment procedure. Amending bounds is a governance act.

## Parameter Interdependencies

Several parameters have relational constraints that must hold at all times:

1. **`orphan_signal_max_age_ms >= audit_interval_ms * 2`**
   An orphan signal must survive at least two full audit cycles before being flagged. This ensures the audit has had an opportunity to process it.

2. **`signal_production_window_ms >= minimum_intent_execution_interval_ms * 3`**
   The signal production window must be long enough for an entity to declare intent, wait the minimum interval, execute, and produce signals. The 3x multiplier provides margin.

3. **`max_degraded_window_ms >= audit_interval_ms`**
   The degraded mode timeout must be at least one full audit cycle. A shorter timeout would trigger before the audit could resolve the degraded condition.

4. **`system_entity_liveness_model.detection_interval_ms <= audit_interval_ms / 4`**
   Liveness checks must run frequently enough to detect failures well within an audit cycle.

5. **`attestation_provisional_window_ms >= 30000`**
   Attestations need at least 30 seconds to transition from Provisional to Verifiable (time for witness network publication).

When amending any parameter, the kernel validates all interdependencies. An amendment that would break an invariant is rejected by E14 or by `KernelParameters::validate()`.

## Parameter Amendment Procedure

### Via CLI

```bash
theorem-node --db theorem.db amend-parameter \
  --param audit_interval_ms \
  --value 7200000 \
  --entity <ENTITY_ID>
```

The `--entity` must be an entity that holds a governance_act authorization.

### Via HTTP API

```
POST /amend-parameter
Content-Type: application/json

{
  "param": "audit_interval_ms",
  "value": 7200000,
  "entity_id": "<ENTITY_ID>"
}
```

Returns `200 OK` with the amendment result on success, or `422 Unprocessable Entity` with violation messages on enforcement failure.

### Amendment Enforcement (E14)

The following checks apply to every parameter amendment:

1. **Single parameter per amendment** -- the governance act may amend at most one kernel parameter.
2. **Within declared bounds** -- the proposed value must be within the parameter's declared floor and ceiling in `parameter_bounds`.
3. **No double-amendment** -- the same parameter may not be amended more than once within `audit_interval_ms`.
4. **Trajectory anomaly detection** -- cumulative amendments in the same direction (up or down) within `audit_interval_ms * 4` must not exceed 50% of the current value. Exceeding this threshold produces a `parameter_trajectory_anomaly` escalation.
5. **Supermajority for drift_threshold_default** -- amending `drift_threshold_default` requires >= 2/3 of active entities in the governance act's `participating_entity_ids`.
6. **Interdependency validation** -- after amendment, all relational constraints must hold (see above).

### Amendment Trajectory Limits

The trajectory anomaly detection works as follows:

- Look back `audit_interval_ms * 4` from the current time
- Sum all upward amendments to the parameter in that window (including the proposed one)
- Sum all downward amendments to the parameter in that window (including the proposed one)
- If either cumulative sum >= 50% of the current parameter value, produce a `parameter_trajectory_anomaly` escalation signal

This prevents gradual circumvention of bounds by making many small amendments in the same direction. The 4x window ensures the check covers multiple audit cycles.

### Amendment Workflow

1. Entity with governance_act authorization declares intent for the amendment
2. Wait `minimum_intent_execution_interval_ms`
3. Submit the amendment via CLI or HTTP
4. Kernel creates a GovernanceAct (type=Amendment) with paths_not_taken
5. E14 validates: bounds, no double-amendment, trajectory, interdependencies
6. On success: KernelParameters updated, `last_amended_via_act_id` set to the new governance act
7. Attribution + Attestation produced for the amendment
8. Attestation published to witness network

See [Operations Runbook](../quick-reference/operations-runbook.md) for the step-by-step procedure.
