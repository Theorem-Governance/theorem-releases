# Theorem 0.5 Release Memo

Date: 2026-03-24

Audience: downstream platform operators, integration owners, and release managers

Status: released

Tag: `v0.5.0`

## Position

`0.5` is the release where Theorem becomes a complete FreeBSD-native OS from
PID 1 to organizational authority. It extends the `0.4` governed platform
contract with a full substrate layer: ZFS-backed storage, jail-based compute,
PF networking, hardware device governance, encrypted secret management,
governed image pipelines, distributed fabric, economic settlement, branch
reality, and autonomous governance runtime.

## What Changed From 4.0

`0.4` shipped six surfaces: governed release artifacts, measured-boot recovery,
tenant boundaries, capacity economics, provisioning, and operator authority.

`0.5` keeps all of that and adds the substrate — the OS layer that these
governance surfaces now run on natively rather than wrapping an external host.

The 30-crate workspace now covers:

- **PID 1 supervisor** with governance runtime, quarantine escalation, and
  alert dispatch (theorem-init)
- **ZFS storage** with dataset lifecycle and content-addressed artifact store
  (theorem-zfs, theorem-store)
- **Jail compute** with workload governance (theorem-compute)
- **PF networking** with firewall rule governance (theorem-net)
- **Encrypted secrets** with Argon2id+AES-256-GCM sealing and Capsicum
  capability restriction (theorem-secrets, theorem-crypto)
- **Image pipelines** with staged rollout (theorem-image)
- **Distributed fabric** with fleet mesh (theorem-fabric)
- **Economic settlement** with crossing settlement, privacy budgets, and data
  retention (theorem-econ, theorem-settle)
- **Branch reality** with ZFS-backed counterfactual governance lanes
  (theorem-branch)
- **Autonomous governance** with intent compiler, self-challenge, drift
  detection, and circuit breaker (theorem-auto)

## Security

Verified by two-pass audit-remediation cycle:

- AUD-2026-03-24-0011: Advisory depth, all 7 domains, 49 findings
- REMED-2026-03-24-0011: 49 fixes, zero deferrals
- AUD-2026-03-24-0012: Static verification, 8 residuals
- REMED-2026-03-24-0012: 8 fixes, zero deferrals

Key security changes:

- Argon2id+AES-256-GCM replaces SipHash+XOR for secret sealing
- Ed25519 signing with ZeroizeOnDrop for all key material
- Governance quarantine enforcement on PID 1 supervisor restarts
- GovernanceAlertDispatcher for external violation monitoring
- Fail-closed privacy budget in settlement crossings
- Capsicum capability restriction for admin operations
- HHI-based governance capture detection

## Release Assets

- `theorem-node-v0.5.0-x86_64-unknown-linux-gnu.tar.gz` — binary
- `theorem-node-v0.5.0-x86_64-unknown-linux-gnu.tar.gz.sha256` — checksum
- `theorem-node-v0.5.0.cdx.xml` — CycloneDX SBOM

## What Downstream Teams Can Do Now

- Consume `releases/feed.json` for the v0.5.0 entry
- Download and verify the binary from the release URL
- Review the release contract for automation guidance
- Automate against `v0.5.0` tag, not "latest"
