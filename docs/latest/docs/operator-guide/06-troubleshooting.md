# Troubleshooting

This document covers common failure modes, enforcement rejection diagnosis, audit criterion failure interpretation, witness sync issues, degraded mode, and authorization chain failures.

## Common Failure Modes

### "kernel: system entity not found -- is the instance bootstrapped?"

**Cause:** The kernel write path could not find the system entity in governance state. This happens when you attempt to submit acts before running bootstrap.

**Resolution:** Run bootstrap first:
```bash
theorem-node --db theorem.db bootstrap --spec-hash <HASH> --artifact-url <URL> --log-endpoint <URL>
```

### "kernel: kernel parameters not found -- is the instance bootstrapped?"

**Cause:** Same as above -- kernel parameters are created during bootstrap.

**Resolution:** Run bootstrap.

### "Command failed" with process exit code 1

**Cause:** A CLI command failed. The structured JSON log on stderr contains the error details.

**Resolution:** Check `journalctl -u theorem-node --since "5 minutes ago"` for the error event with `level: "ERROR"`.

### Server binds but rejects all requests

**Cause:** The server is running but the governance database has no bootstrap state. All endpoints except `/status` and `/metrics` require a bootstrapped instance.

**Resolution:** Check `/status` -- if `bootstrapped` is false, run the bootstrap procedure.

## Enforcement Rejection Diagnosis

When an act is rejected, the HTTP API returns `422 Unprocessable Entity` with a JSON body containing the violation messages:

```json
{
  "violations": [
    "E1: authorization AUTH-123 not found",
    "E4: intent INT-456 elapsed 500ms since declared_at, minimum 1000ms required"
  ]
}
```

The CLI prints violations to stdout and exits with code 1.

### Diagnosing by Rule

#### E1 Violations (Act Completeness)

| Message Pattern | Cause | Resolution |
|-----------------|-------|------------|
| `E1: authorization_id is null` | Non-bootstrap act submitted without authorization | Ensure the act JSON includes a valid `authorization_id` |
| `E1: authorization <ID> not found` | The referenced authorization does not exist | Verify the authorization ID exists in state via `/status` or `/export-state` |
| `E1: intent <ID> not found` | A referenced intent does not exist in state | Intents must be stored before the kernel write. Submit intents first. |
| `E1: intent <ID> objective is empty` | Intent fails INTENT_CONTENT_MINIMUM | All intent fields must be non-empty: objective, success_criteria, constraints, scope_rationale |
| `E1: intent <ID> paths_not_taken has N alternatives, minimum 2` | Insufficient alternatives | Provide at least 2 alternatives in paths_not_taken |
| `E1: scope <ID> not found` | Referenced scope does not exist | Use an existing scope ID |
| `E1: executing entity <ID> not found` | Entity does not exist | Establish the entity first via `/establish-entity` |

#### E2 Violations (Authorization Validity)

| Message Pattern | Cause | Resolution |
|-----------------|-------|------------|
| `E2: authorization <ID> status is Revoked` | Authorization has been revoked | Use a different active authorization |
| `E2: act type <TYPE> not in authorization action_vocabulary` | Authorization does not permit this act type | The authorization's `action_vocabulary` must include the act type |
| `E2: authorization chain from <ID> does not resolve to bootstrap` | Broken authorization chain | One or more links in the chain are missing or revoked. Trace the chain via export-state. |
| `E2: mutual authorization detected between entities <A> and <B>` | A granted authority to B and B granted authority to A | This is a governance structure error. Revoke one direction. |
| `E2: requires_safety_assessment but no valid safety assessment signal found` | Authorization requires safety assessment but none exists | Submit a safety_assessment signal before the act |

#### E3 Violations (Scope Enforcement)

| Message Pattern | Cause | Resolution |
|-----------------|-------|------------|
| `E3: act scope <S1> exceeds executing entity scope <S2>` | Entity operating outside its scope | The act scope must be contained within the entity's scope |
| `E3: act scope <S1> exceeds authorization scope <S2>` | Authorization does not cover this scope | Use an authorization that covers the intended scope |
| `E3: kernel_write scope does not match parent act scope` | kernel_write scope mismatch | This is an internal error -- kernel_write inherits scope from parent |

#### E4 Violations (Intent Precedence)

| Message Pattern | Cause | Resolution |
|-----------------|-------|------------|
| `E4: intent <ID> elapsed Nms since declared_at, minimum Mms required` | Intent was declared too recently | Wait at least `minimum_intent_execution_interval_ms` after declaring the intent |

#### E7 Violations (Self/Mutual Authorization)

| Message Pattern | Cause | Resolution |
|-----------------|-------|------------|
| `E7: self-authorization detected` | Entity is both principal and granter | An entity cannot authorize itself. Use a different granter. |
| `E7: mutual authorization detected` | Grant cycle between entities | Revoke one of the grants to break the cycle |
| `E7: system entity authorization not produced by bootstrap` | Attempt to grant non-bootstrap authorization to system entity | System entity can only hold bootstrap-produced authorizations |
| `E7: system entity authorization is irrevocable` | Attempt to revoke system entity authorization | System entity authorizations cannot be revoked |

#### E14 Violations (Substance and Parameters)

| Message Pattern | Cause | Resolution |
|-----------------|-------|------------|
| `E14: intent objective matches tautological pattern` | Objective is self-referential ("execute this act", etc.) | Write a substantive objective that describes the intended outcome |
| `E14: success_criteria measurable_indicator matches completion pattern` | Indicator is a trivial completion check | Write measurable indicators that describe observable outcomes |
| `E14: alternative description does not reference concrete governance state` | Path alternative is generic, not grounded in actual governance state | Reference specific entity IDs, scope IDs, or parameter names |
| `E14: proposed value outside bounds [floor, ceiling]` | Amendment exceeds declared parameter bounds | Choose a value within the declared bounds (see [Configuration Reference](03-configuration-reference.md)) |
| `E14: parameter already amended within audit_interval_ms` | Double-amendment guard triggered | Wait for the next audit interval before amending again |
| `E14: cumulative change >= 50% -- parameter_trajectory_anomaly` | Too many amendments in the same direction | Slow down amendments or adjust in the opposite direction first |

## Audit Criterion Failure Interpretation

When `/audit` reports a criterion as `satisfied: false`, the `findings` array explains what failed. Here is what each criterion failure means operationally and how to respond.

| Criterion | Failure Means | Operational Response |
|-----------|---------------|---------------------|
| C1 | Authorization chain integrity broken for acts in window | Investigate broken authorization chains. Trace from the flagged act to bootstrap via export-state. |
| C2 | Active entities lack drift signals for their attributions | Ensure drift measurements are being submitted with acts that have beneficiaries. |
| C3 | Three-way reference integrity broken (act/attribution/attestation) | This indicates a storage or kernel bug. Check database integrity. |
| C4 | Beneficiary entity IDs empty or attestation missing paths_not_taken_summary | Ensure acts include beneficiary entities and attestations include paths_not_taken_summary. |
| C5 | Orphan signals exist (pending too long) | Signals are not being consumed/acknowledged. Investigate which entity is not responding. |
| C6 | Bootstrap state incomplete or corrupt | Critical: genesis records, bootstrap entity, system entity, or kernel parameters are missing. May require database restore. |
| C7 | Scope DAG has cycle or act scope exceeds authorization scope | Check for circular scope containment. Verify act scope assignments. |
| C8 | Intent declared after act execution | Check clock synchronization. Verify intent timestamps. |
| C9 | Active entities not producing signals | Entities must produce signals within signal_production_window_ms. Non-participating entities need engagement or status change. |
| C10 | Workflow acts reference invalid or inactive workflows | Check workflow status and establishing governance act. |
| C11 | Attestations stuck in Provisional or missing external_schema_representation | Check witness network connectivity. Verify attestation finalization is running. |
| C12 | Acts with resource_limits authorization missing resource_consumption records | Include resource consumption entries when the authorization requires them. |
| C13 | Governance acts missing paths_not_taken or intents have insufficient alternatives | All non-bootstrap governance acts and intents need at least 2 alternatives. |
| C14 | Audit cycle overdue | Run an audit. If automated audits are configured, check why they stopped. |
| C15 | Parameter values outside declared bounds or trajectory anomaly unresolved | Check parameter_bounds in KernelParameters. Resolve any pending escalation signals. |
| C16 | Entity identity attestation missing, expired, or hash collision | Renew expired identity attestations. Investigate hash collisions. |
| C17 | System entity liveness failure or dependency includes system entity | Check system entity status. Verify liveness model configuration. |
| C18 | Duplicate intent objectives or governance uniformity anomaly | Entities are submitting identical objectives. Investigate whether governance is substantive. |
| C19 | Infrastructure verification artifacts missing or inconsistent | Check integrity mechanism artifact locations and attestation log endpoints. |
| C20 | Required properties lack active proofs | Submit or renew proofs for properties without active proof obligations. |
| C21 | Deterministic verifier not run in current audit cycle | Run the verifier: `theorem-node --db theorem.db verify` |
| C22 | Degraded attributions not reconciled (attestations still Provisional) | Finalize provisional attestations from degraded-mode operations. |

## Witness Sync Issues

### Symptoms

- `GET /witness/digest` shows different head_hash from peers
- `theorem_witness_entries_appended_total` metric not incrementing
- C11 (attestations externally verifiable) failing

### Diagnosis

1. Check witness digest:
   ```bash
   curl http://localhost:3170/witness/digest
   ```

2. Compare with peer digests. If log_length differs, publications may not be reaching the witness.

3. Check the witness database integrity:
   ```bash
   sqlite3 /var/lib/theorem/theorem-witness.db "PRAGMA integrity_check;"
   ```

4. Check for `WitnessError` entries in logs:
   ```bash
   journalctl -u theorem-node | grep -i witness
   ```

### Common Causes

- **Network connectivity:** Witness endpoints unreachable. Check firewall rules and DNS.
- **Duplicate attestation ID:** The witness rejects publications with attestation IDs already in the log. This is expected behavior (idempotency guard).
- **Integrity proof mismatch:** The `integrity_proof` in the publication does not match the `canonical_serialization` content hash. This indicates a bug in the serialization or signing path.
- **Hash chain break:** If entries are appended out of order or a backup was restored, the hash chain may be broken. Run `GET /witness/digest` to check `chain_valid`.

## Degraded Mode

### What Triggers Degraded Mode

The kernel write path runs health checks (E8, E13, E15, E16) after input enforcement passes. If any health check fails, the write proceeds but:

1. Escalation signals are produced (type: `system_health_violation`)
2. `degraded_flag = true` is set on the attribution
3. The attestation is produced normally but carries the degraded flag in its summaries

### What Degraded Mode Means

The governance instance is operating with known health issues. Acts are still being processed and enforced, but the system is not meeting all health requirements. The degraded attribution will need reconciliation before the next audit cycle (C22 checks this).

### Health Checks That Trigger Degraded Mode

| Check | Escalation Type | Meaning |
|-------|----------------|---------|
| E8: signal production window | NonParticipatingEntity | An entity has not produced signals within the required window |
| E8: orphan signal age | OrphanSignalAged | Pending signals older than orphan_signal_max_age_ms |
| E13: audit currency | AuditInArrears | No audit has run within audit_interval_ms |
| E15: proof expiry | ProofExpired | Active proofs past their expiry date |
| E15: proof cascade | ProofDependencyCascade | Proofs depending on invalidated proofs still active |
| E16: system liveness | SystemEntityLivenessFailure | System entity has no recent acknowledgment signal |
| E16: entity identity | IdentityAttestationExpiring | Entity identity attestation expired or near expiry |
| E16: infrastructure | various | Integrity mechanism or attestation log configuration issues |
| E16: degraded mode params | DegradedModeEntered | max_degraded_window_ms < audit_interval_ms |

### How to Recover from Degraded Mode

1. **Identify the cause:** Check the escalation signals produced during degraded writes. Use `/export-state` or query the database for signals with `signal_type = "escalation"` and `status = "pending"`.

2. **Resolve the root cause:**
   - For E13 (audit arrears): Run a full audit via `/audit` or CLI
   - For E15 (proof expiry): Submit new proofs via `/submit-proof` or CLI
   - For E16 (liveness): Verify system entity is producing acknowledgment signals
   - For E8 (signal production): Ensure entities are actively participating

3. **Reconcile degraded records:** Run an audit. C22 will verify that degraded attributions have had their attestations transitioned from Provisional to Verifiable.

4. **Acknowledge escalation signals:** Process pending escalation signals to clear them.

## "Intent Not Found" Error

**Message:** `E1: intent <ID> not found`

**Cause:** The kernel write path looks up intents from governance state. Intents must be stored in state before they can be referenced by an act. This is a deliberate design -- intent declaration is a separate governance event from act execution.

**Resolution:** When using the HTTP API, the `/submit-act` endpoint handles intent creation as part of the submission. When using lower-level APIs or the CLI with `--input` JSON files, ensure the intents are included in the submission payload.

## Authorization Chain Failures

### Broken Chain

**Message:** `E2: authorization chain from <ID> does not resolve to bootstrap`

The authorization chain is traversed from the act's authorization back to the bootstrap governance act. If any link in the chain is missing (authorization not found) or broken (parent authorization not found), the chain does not resolve.

**Diagnosis:**
1. Export state: `theorem-node --db theorem.db export-state --pretty > state.json`
2. Find the authorization referenced by the act
3. Follow `parent_authorization_id` links until you reach bootstrap or find the break
4. Check if any link has been revoked (status != Active)

**Resolution:** Establish a new authorization chain from an entity with a valid chain to bootstrap.

### Mutual Authorization Detection

**Message:** `E7: mutual authorization detected -- entity <A> and entity <B> form a grant cycle`

Entity A granted authority to Entity B, and Entity B granted authority to Entity A (directly or transitively through other entities). This creates a circular dependency that undermines the authorization hierarchy.

**Resolution:** Revoke one of the authorizations to break the cycle:

```bash
theorem-node --db theorem.db revoke-authorization \
  --auth-id <AUTHORIZATION_ID> \
  --entity <REVOKING_ENTITY_ID>
```

The revoking entity must be the granter of the authorization or the bootstrap entity.

### System Entity Authorization Issues

System entity authorizations are produced only at bootstrap and are irrevocable. If you see `E7: system entity authorization not produced by bootstrap`, something has attempted to grant a new authorization to the system entity outside of bootstrap. This should not happen in normal operation -- investigate the governance act that attempted it.
