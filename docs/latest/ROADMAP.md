# Theorem — Roadmap

**Last updated:** 2026-03-15
**Status:** Active — Steps 1-6 complete. Step 5 remediated and deployed to prod (21/22 audit, 23/25 verify). Step 7 (auditor) and Step 8 (DEC-010 bootstrap) next.
**Goal:** Conforming production instance operated by a single entity.

---

## Dependency Graph

```
Step 8: Independent Auditor (PA-001 through PA-008)
   │
   └── Step 7: DEC-010 Governance Inference Engine ←── NEXT
          │
          ├── 7a: D+E Bootstrap (model behind gateway, E14 boundary) ←── NOW
          │
          └── 7b: F-phase (distillation) ←── 7a operational maturity
          │
          ├── Step 5: Ring 3 Enclave Runtime (complete, remediated, deployed) ✓
          │      ├── 5a: Bus Foundation ✓
          │      ├── 5b: Enclave Mode (Supervisor) ✓
          │      ├── 5c: Sidecar Mode (Gateway) ✓
          │      ├── 5d: Verified Drift Measurement ✓
          │      └── 5e: Client SDK (optional) ✓
          │
          ├── Step 6: DEC-009 Calibration (complete, updated with 120B) ✓
          │
          └── Step 4: Proof Submissions (complete) ✓
                 ├── Step 3: Peer Witness on NAS (complete) ✓
                 │      └── Step 2: Deploy and Bootstrap (complete) ✓
                 │             └── Step 1: NixOS on DEC-007 Hardware (complete) ✓
                 └── Step 2 (also required) ✓
```

Steps 1-6 complete. Step 7a (D+E bootstrap — model behind gateway with E14 boundary) is next. The running model generates continuous governance activity (fixes C9), accumulates DPO corpus (enables 7b), and gives the auditor (Step 8) a live system to inspect.

---

## Step 1: NixOS on DEC-007 Hardware

**Status:** Complete
**Who:** Eric (manual)
**Depends on:** Nothing
**Produces:** Bootable NixOS machine with measured boot chain

The i7-8700 / 32GB / GTX 2080 machine (hostname: theorem-ring1, 192.168.50.176) running NixOS.

### Checklist

- [x] NixOS installed, SSH enabled, flakes configured
- [x] theorem-node binary built via `nix build`
- [x] Hardening applied (kernel hardening, firewall)
- [x] `nixos-rebuild switch` — everything applied

### Verification

```bash
uname -a                    # NixOS kernel
sbctl verify                # Secure Boot OK
mokutil --sb-state          # SecureBoot enabled
chronyc sources -v          # NTS sources active
systemctl status theorem-node  # Service unit exists (not yet started)
```

---

## Step 2: Deploy and Bootstrap

**Status:** Complete
**Who:** Eric + Claude Code (on the NixOS machine)
**Depends on:** Step 1
**Produces:** Running theorem-node with `bootstrapped: true`

Kernel running as systemd service on port 3170, bootstrapped with 25 proof obligations.

---

## Step 3: Peer Witness on NAS

**Status:** Complete
**Who:** Eric (manual + config)
**Depends on:** Step 2
**Produces:** P13 compliance — two independent attestation locations

Synology NAS (DS220+, eXodus, 192.168.50.50) runs theorem-node in witness-only mode on port 3171.

- Kernel configured: `THEOREM_PEER_WITNESSES=http://192.168.50.50:3171`
- Same geographic location — declared risk acceptance per DEC-008

---

## Step 4: Proof Submissions

**Status:** Complete
**Who:** Eric (operator)
**Depends on:** Steps 2 and 3
**Produces:** All 25 properties with Active proofs; `/verify` returns valid

- 23 properties: Pass (deterministic verification)
- 2 properties: RequiresPhysicalAudit (PROP-SINCERITY-001, PROP-SUBSTRATE-001 — deferred to Step 7)

---

## Step 5: Ring 3 Enclave Runtime

**Status:** Complete — remediated, deployed to prod
**Who:** Eric + Claude Code
**Depends on:** Step 2
**Produces:** Governance enclave — programmatic interface for governed entities to submit acts via bus protocol

Two deployment models (DEC-013):

- **Enclave mode** — Supervisor launches processes inside the governance boundary. Born governed.
- **Sidecar mode** — Gateway exposes HTTP/gRPC. External systems call in. Also the deployment model for DEC-010 governance inference engine.

### Remediation summary

Full audit (AUD-2026-03-15-0004) found 63 findings across 8 domains. All 63 remediated with fail-closed defaults:

- **Auth:** Services refuse to start without auth tokens or explicit `--allow-anonymous`
- **Shell injection:** `CommandMode::Direct` is the default (shell requires opt-in)
- **Rate limiting:** Token-bucket per-entity on gateway and bus
- **Metrics:** Prometheus endpoint with 12 instrumented metrics
- **869 tests pass**, 0 failures

Evidence: `docs/remediation/2026-03-15-AUD-0004/` (38 evidence records, full traceability)

### Production state (NixOS)

- **Binary:** `/nix/store/86165pysfvcm4y10m7cr3hdqmhxks6wj-theorem-node-0.1.0/bin/theorem-node`
- **Audit:** 21/22 criteria pass (C9 signal production cadence needs continuous agent activity)
- **Verify:** 23/25 properties pass (0 fail, 2 require physical audit)

### Step 5a: Bus Foundation

**Spec:** `docs/superpowers/specs/2026-03-14-ring3-agent-runtime-design.md`
**Plan:** `docs/superpowers/plans/2026-03-14-ring3-bus-foundation.md`

### Step 5b: Enclave Mode (Supervisor)

### Step 5c: Sidecar Mode (Gateway)

- Two-call governance: `POST /ring3/intent` → do work → `POST /ring3/outcome`

### Step 5d: Verified Drift Measurement

**Spec:** `docs/superpowers/specs/2026-03-15-verified-drift-measurement.md`
**Plan:** `docs/superpowers/plans/2026-03-15-verified-drift-measurement.md`

### Step 5e: Client SDK (Optional)

### Build order

5a → 5b and 5c (parallel) → 5d (extends bus) → 5e (optional)

---

## Step 6: DEC-009 Calibration

**Status:** Complete (2026-03-15, updated with 120B external inference)
**Who:** Eric + Claude Code
**Depends on:** Step 5
**Produces:** Evidence that substance checks discriminate between capable and incapable models

### Results

| Model | Params | Standard | Adversarial | Overall |
|-------|--------|----------|-------------|---------|
| Qwen 2.5 7B Q4_K_M | 7.6B | — | — | 1/3 (33%) |
| Qwen 2.5 14B Q4_K_M | 14B | — | — | 0/18 (0%) |
| Qwen 2.5 32B Q2_K | 32B | — | — | 5/18 (28%) |
| gpt-oss-120B | 120B | 28/30 (93%) | 0/30 (0%) | 28/60 (47%) |

120B tested via Together.ai. Local E14 substance checks, no kernel act submission.

### Decision gate outcomes

- Scale does not fix adversarial failures — concrete state grounding must be trained in
- DEC-010 thesis validated, deployment model and training pipeline designed
- Synthetic DPO corpus attempted and rejected — corrections are ID-stapling, not deliberation

---

## Step 7: DEC-010 Governance Inference Engine

**Status:** Next — D+E bootstrap
**Who:** Eric + Claude Code (system), model (autonomous after bootstrap)
**Depends on:** Step 5 (remediated), Step 6
**Produces:** Fine-tuned governance model grown from real operational data; continuous governance activity that fixes C9 and gives the auditor a live system

The governance inference engine is grown, not built. See DEC-010 in DECISIONS.md for full deployment model and trust architecture.

### Step 7a: D+E Bootstrap

**Depends on:** Step 5 remediation complete ✓

- General-purpose model deployed behind sidecar gateway (Step 5c) as untrusted infrastructure
- E14 evaluates every output (structural). PA-007 evaluates sincerity (substantive). Both required.
- Model weights on local hardware declared as substrate — PA-008 applies (C, always)
- Bus generates templates for adversarial failures — labeled honestly as bus-generated
- Three-artifact logging per adversarial failure: original, regeneration, bus template
- DPO pairs: (original, regeneration) only when regeneration passes E14

### Step 7b: F-phase (Distillation)

**Depends on:** Step 7a accumulated enough authentic correction pairs

- Trigger: measurable corpus size and per-component pass-rate trends
- DPO training on (failed-original, passed-regeneration) pairs from 7a
- Train on cloud GPU, deploy new weights behind gateway, E14 evaluates immediately
- Iterate: improved model generates new failures, new training signal

---

## Step 8: Independent Auditor

**Status:** Blocked on Step 7a operational maturity
**Who:** Eric (procurement) + external auditor
**Depends on:** Step 7a running (auditor needs a live system with governed entities to inspect)
**Produces:** PA-001 through PA-008 attestations; first complete audit cycle

### 8 Physical Audit Exits

| PA | Property | What |
|----|----------|------|
| PA-001 | PROP-IDENTITY-001 | Identity binding mechanism inspection |
| PA-002 | PROP-IDENTITY-003 | Timed revocation test |
| PA-003 | PROP-LIVENESS-001 | Deployment topology inspection |
| PA-004 | PROP-LIVENESS-002 | System entity suppression test |
| PA-005 | PROP-LIVENESS-003 | Recovery procedure red-team |
| PA-006 | PROP-AVAILABILITY-001 | Degraded mode behavior test |
| PA-007 | PROP-SINCERITY-001 | Governance artifact substance rubric |
| PA-008 | PROP-SUBSTRATE-001 | Substrate trust declaration verification (includes model weight provenance per DEC-010) |

After all 8 attestations: meta-verification, then the instance is conforming.

---

## Current production state

- [x] NixOS machine running theorem-node with Secure Boot and measured boot
- [x] Bootstrapped with all 25 properties proven (23 pass, 2 physical audit)
- [x] Witness log replicated to NAS
- [x] Enclave runtime operational: bus, supervisor, gateway (remediated, fail-closed)
- [x] Substance checks calibrated (8B through 120B)
- [x] Audit: 21/22 criteria pass (C9 needs continuous agent activity)
- [x] Verify: 23/25 properties pass, `valid: true`
- [ ] Governance inference engine in D+E bootstrap, logging three-artifact protocol
- [ ] Independent auditor has attested all 8 physical audit exits
- [ ] First audit cycle complete: `/audit` returns `all_satisfied: true`

---

## Not on this roadmap

These are real but not sequenced yet:

- **ARCH-002: Multi-layer governance** — builder kernels, end-user kernels
- **Geographic redundancy** — third witness location outside the home
- **Auditor rotation** — second independent auditor after two cycles
- **DEC-010 F-phase completion** — when the fine-tuned model consistently passes adversarial E14 checks
