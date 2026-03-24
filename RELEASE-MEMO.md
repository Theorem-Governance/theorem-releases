# Theorem 5.0 Release Memo

Date: 2026-03-24

Audience: downstream platform operators, integration owners, and release managers

Status: released

Tag: `v5.0.0`

## Position

`5.0` is the release where Theorem becomes a complete FreeBSD-native OS from
PID 1 to organizational authority. It extends the `4.0` governed platform
contract with a full substrate layer: ZFS-backed storage, jail-based compute,
PF networking, hardware device governance, encrypted secret management,
governed image pipelines, distributed fabric, economic settlement, branch
reality, and autonomous governance runtime.

## What We Are Shipping

The public `5.0` line adds these release-defining surfaces on top of the `4.0`
contract:

- **Substrate law** — theorem-init as PID 1 with async supervisor, governance
  runtime wired into process lifecycle, quarantine → shutdown escalation
- **Storage** — theorem-zfs for ZFS dataset lifecycle, theorem-store for
  content-addressed artifact admission with GC
- **Compute** — theorem-compute for FreeBSD jail workload governance
- **Networking** — theorem-net for PF firewall rule governance
- **Secrets** — theorem-secrets with Argon2id+AES-256-GCM envelope encryption,
  Capsicum capability restriction, Zeroizing key material
- **Images** — theorem-image for governed container image pipelines with staged
  rollout
- **Fabric** — theorem-fabric for distributed fleet mesh and node management
- **Economics** — theorem-econ for admission pricing, theorem-settle for
  crossing settlement with privacy budgets and data retention
- **Branch reality** — theorem-branch for ZFS-backed counterfactual governance
  lanes
- **Autonomy** — theorem-auto for intent compilation, self-challenge, drift
  detection, and circuit breaker

## Security Posture

This release was verified through a two-pass audit-remediation cycle:

- **AUD-2026-03-24-0011** — Advisory depth, all 7 domains, 49 findings
- **REMED-2026-03-24-0011** — 49 fixes, zero deferrals
- **AUD-2026-03-24-0012** — Static verification pass, 8 residual findings
- **REMED-2026-03-24-0012** — 8 fixes, zero deferrals

Key security improvements over `4.0`:

- Argon2id+AES-256-GCM replaces SipHash+XOR for secret sealing
- Ed25519 signing with ZeroizeOnDrop for all key material
- Governance quarantine enforcement on PID 1 supervisor restarts
- GovernanceAlertDispatcher for external violation monitoring
- Fail-closed privacy budget in settlement crossings
- Capsicum capability restriction for admin operations
- HHI-based governance capture detection in C7 audit criterion

## What Downstream Teams Can Do Now

- Consume `releases/feed.json` for the v5.0.0 entry
- Review the release contract at `docs/release-contract.md`
- Review operator guides for the expanded substrate surfaces
- Plan integration against the new economic settlement and fabric APIs
- Automate against `v5.0.0` tag, not "latest"

## Bottom Line

`5.0` is theoremOS — a governed FreeBSD-native OS with 30 crates covering
everything from PID 1 boot attestation to organizational economic settlement.
The full audit trail is published alongside the release.
