# Monitoring and Alerting

This document covers the Prometheus metrics endpoint, all available metrics, recommended alert rules, status and watchdog endpoint interpretation, and structured logging.

## Health Endpoint

**Endpoint:** `GET /health`

This is the canonical liveness probe for service managers, reverse proxies, and
external monitors. It returns HTTP `200` with a minimal JSON status body when
the process is alive.

### Response Format

```json
{
  "status": "ok",
  "healthy": true,
  "bootstrapped": true
}
```

`bootstrapped` indicates whether the node has completed bootstrap. For richer
governance-state inspection, use `GET /status`.

## Readiness Endpoint

**Endpoint:** `GET /ready`

This is the canonical readiness probe for orchestration and automated traffic
cutover. It returns:

- HTTP `200` with `{"status":"ready","ready":true,...}` when the instance is
  bootstrapped and both `THEOREM_COMMITMENT_SECRET` and
  `THEOREM_WITNESS_TOKEN` are configured
- HTTP `503` with `{"status":"not_ready","ready":false,...}` otherwise

### Response Format

```json
{
  "status": "ready",
  "ready": true,
  "bootstrapped": true,
  "witness_token_configured": true,
  "commitment_secret_configured": true
}
```

Use `/health` for liveness and `/ready` for traffic gating.

## Prometheus Metrics Endpoint

**Endpoint:** `GET /metrics`

Returns metrics in Prometheus text exposition format (Content-Type: text/plain). Scrape this endpoint from your Prometheus instance.

All counters start at 0 when the server process starts. They are in-memory atomic counters that reset on restart. For persistent metrics, use the audit and status endpoints.

## Available Metrics

### Scalar Counters

| Metric | Type | Description |
|--------|------|-------------|
| `theorem_acts_submitted_total` | counter | Acts processed via `execute_kernel_write` (both accepted and rejected) |
| `theorem_acts_rejected_total` | counter | Acts rejected by enforcement (E1-E16 violations) |
| `theorem_audits_run_total` | counter | Full audit cycles (C1-C22) completed |
| `theorem_attestations_finalized_total` | counter | Attestations transitioned from Provisional to Verifiable |
| `theorem_witness_entries_appended_total` | counter | Witness log entries appended |
| `theorem_bootstrap_total` | counter | Bootstrap operations completed (should be 1 for a correctly operating instance) |
| `theorem_parameter_amendments_total` | counter | Parameter amendments applied via `execute_parameter_amendment` |

### Labeled Counters

| Metric | Label | Description |
|--------|-------|-------------|
| `theorem_audit_criteria_failed_total` | `criterion` | Per-criterion failure count. Labels: C1, C2, ..., C22 |
| `theorem_enforcement_violations_total` | `rule` | Per-rule violation count. Labels: E1, E2, ..., E16 |
| `theorem_proof_transitions_total` | `type` | Proof status transition count. Labels vary by transition type |

### Example Output

```
# HELP theorem_acts_submitted_total Acts processed via execute_kernel_write
# TYPE theorem_acts_submitted_total counter
theorem_acts_submitted_total 47

# HELP theorem_acts_rejected_total Acts rejected by enforcement
# TYPE theorem_acts_rejected_total counter
theorem_acts_rejected_total 3

# HELP theorem_enforcement_violations_total Enforcement rule violations
# TYPE theorem_enforcement_violations_total counter
theorem_enforcement_violations_total{rule="E1"} 1
theorem_enforcement_violations_total{rule="E14"} 2
```

## Recommended Prometheus Alert Rules

```yaml
groups:
  - name: theorem
    rules:
      # Enforcement rejections indicate configuration or integration errors.
      - alert: TheoremHighRejectionRate
        expr: rate(theorem_acts_rejected_total[5m]) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High act rejection rate"
          description: "More than 0.1 rejections/sec for 10 minutes. Check enforcement violation labels for the specific rule."

      # Audit criteria failures indicate governance health issues.
      - alert: TheoremAuditCriterionFailing
        expr: increase(theorem_audit_criteria_failed_total[1h]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Audit criterion {{ $labels.criterion }} failing"
          description: "Criterion {{ $labels.criterion }} has failed in the last hour. See Troubleshooting guide for interpretation."

      # No audits running means the audit cycle may have stalled.
      - alert: TheoremAuditStalled
        expr: increase(theorem_audits_run_total[2h]) == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "No audits completed in 2 hours"
          description: "The audit cycle may have stalled. Check E13 (audit currency) and the /audit endpoint."

      # Attestation finalization backlog.
      - alert: TheoremAttestationBacklog
        expr: theorem_acts_submitted_total - theorem_attestations_finalized_total > 10
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Attestation finalization backlog"
          description: "Provisional attestations are not being finalized. Check witness network connectivity."

      # Witness log not growing means publications are not reaching the witness.
      - alert: TheoremWitnessStalled
        expr: increase(theorem_witness_entries_appended_total[1h]) == 0 and theorem_acts_submitted_total > 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Witness log not growing"
          description: "No witness entries appended in the last hour despite act submissions."

      # Bootstrap should happen exactly once.
      - alert: TheoremMultipleBootstraps
        expr: theorem_bootstrap_total > 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Multiple bootstrap operations detected"
          description: "Bootstrap count is {{ $value }}. This should be exactly 1."

      # Specific high-value enforcement rules to monitor individually.
      - alert: TheoremAuthorizationViolations
        expr: increase(theorem_enforcement_violations_total{rule="E2"}[1h]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Authorization violations (E2)"
          description: "Authorization chain traversal or validity failures detected."

      - alert: TheoremMutualAuthDetected
        expr: increase(theorem_enforcement_violations_total{rule="E7"}[1h]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Mutual authorization or self-authorization detected (E7)"
          description: "Grant cycle or self-authorization attempt detected. Investigate immediately."
```

## Status Endpoint

**Endpoint:** `GET /status`

Returns a JSON object summarizing the current governance state.

### Pre-Bootstrap Response

```json
{
  "bootstrapped": false
}
```

### Post-Bootstrap Response

```json
{
  "bootstrapped": true,
  "active_entities": 5,
  "system_entity": true,
  "bootstrap_entity": true,
  "scopes": 2,
  "proof_obligations": 25,
  "proof_status_counts": {
    "Active": 25
  },
  "last_audit": {
    "id": "aud-...",
    "executed_at": "2025-01-15T10:30:00Z"
  }
}
```

**Key fields to monitor:**

- `bootstrapped` -- must be `true` for a functioning instance
- `proof_status_counts` -- all values should be non-zero; `Active` count should match `proof_obligations`
- `active_entities` -- track growth over time
- `last_audit.executed_at` -- verify audits are running on schedule

## Watchdog Endpoint

**Endpoint:** `GET /watchdog`

Runs infrastructure health checks against governance state. Does not require network access -- these are structural checks against the local database.

### Checks Performed

1. **Attestation log verification** -- endpoint non-empty, technology requirements (append_only, cryptographic_hash_chaining, third_party_verification), verification_method non-empty
2. **Artifact consistency** -- integrity mechanism has >= 2 artifact locations, algorithm in approved_algorithms, public_verification_artifact non-empty
3. **Ungoverned changes detection** -- all infrastructure IDs referenced by KernelParameters exist in state

### Response Format

**Healthy (no issues found):**

```json
{
  "healthy": true,
  "issues": []
}
```

**Unhealthy (issues detected):**

```json
{
  "healthy": false,
  "issues": [
    {
      "check": "attestation_log",
      "target_id": "atl-...",
      "issues": ["endpoint is empty"]
    },
    {
      "check": "integrity_mechanism",
      "target_id": "im-...",
      "issues": ["fewer than 2 artifact locations"]
    }
  ]
}
```

Each entry in `issues` has a `check` field (the type of check performed), an optional `target_id` (the infrastructure record being checked), and a nested `issues` array containing the specific problems found. Only checks that find problems appear in the top-level `issues` array — when all checks pass, `"issues"` is an empty array. See [Troubleshooting](06-troubleshooting.md) for resolution procedures.

## Audit Endpoint

**Endpoint:** `GET /audit`

Runs the full C1-C22 audit against current state. Returns all criterion results regardless of pass/fail.

```json
{
  "all_satisfied": true,
  "pass_count": 22,
  "total_count": 22,
  "criteria": [
    { "criterion": "C1", "satisfied": true, "findings": [] },
    { "criterion": "C2", "satisfied": true, "findings": [] },
    ...
    { "criterion": "C22", "satisfied": true, "findings": [] }
  ]
}
```

Every criterion is evaluated even if earlier ones fail (no short-circuit, DEC-016). The `findings` array contains human-readable strings describing specific issues found.

## Verify Endpoint

**Endpoint:** `GET /verify`

Runs the 25-property deterministic verifier. Returns AUDIT VALID or AUDIT INVALID.

```json
{
  "valid": true,
  "checks": [
    { "property_id": "PROP-PARAM-001", "status": "pass", "details": [] },
    { "property_id": "PROP-SINCERITY-001", "status": "requires_physical_audit", "details": [...] },
    ...
  ]
}
```

Property check statuses: `pass`, `fail`, `skip`, `requires_physical_audit`. A result is AUDIT VALID if all deterministic checks pass; `requires_physical_audit` does not block validity.

## Proof Lifecycle Endpoint

**Endpoint:** `GET /proof-lifecycle`

Checks proof obligation expiry and cascade invalidation.

## Log Format

Theorem-node emits structured JSON logs to stderr via `tracing_subscriber::fmt().json()`.

### Environment Filter

Set `RUST_LOG` to control verbosity:

```bash
# Default: info level for everything
RUST_LOG=info

# Debug level for the node, info for the kernel
RUST_LOG=theorem_node=debug,theorem_kernel=info

# Trace enforcement rule evaluation
RUST_LOG=theorem_kernel::enforcement=trace

# Debug everything
RUST_LOG=debug
```

### Log Entry Format

```json
{
  "timestamp": "2026-03-13T10:00:00.000000Z",
  "level": "WARN",
  "target": "theorem_kernel::enforcement::e2",
  "fields": {
    "message": "Enforcement violation",
    "rule": "E2",
    "violation": "E2: authorization AUTH-001 status is Revoked, must be active"
  },
  "spans": [
    { "name": "enforce_e2" },
    { "name": "execute_kernel_write", "entity_id": "ENT-001" }
  ]
}
```

### Tracing Spans

Key spans and their fields:

| Span | Fields | Description |
|------|--------|-------------|
| `execute_kernel_write` | `entity_id` | Wraps the full kernel write path |
| `enforce_e1` through `enforce_e16` | -- | Individual enforcement rule evaluation |
| `run_full_audit` | -- | Full C1-C22 audit evaluation |
| `handle_submit_act` | -- | HTTP handler for act submission |
| `handle_establish_entity` | -- | HTTP handler for entity establishment |

### Events to Watch

| Event | Level | Meaning |
|-------|-------|---------|
| `Enforcement violation` | WARN | An enforcement rule detected a violation. Field `rule` identifies which rule, `violation` has the message. |
| `Command failed` | ERROR | A CLI or server command failed. The `error` field has details. |
| `Theorem node listening` | INFO | Server started. Fields: `port`, `bind`. |

### Forwarding to External Systems

Since logs go to stderr and the systemd service sends stderr to the journal, you can:

1. Use `journalctl -u theorem-node -f --output=json` for real-time JSON streaming
2. Configure a log forwarder (Vector, Fluent Bit, Promtail) to read from the journal
3. Forward to Loki, Elasticsearch, or your preferred log aggregation system

The JSON structure is stable and machine-parseable. Filter on `target` to isolate enforcement, audit, or verifier logs.
