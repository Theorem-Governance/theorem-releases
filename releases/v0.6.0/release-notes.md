# Theorem v0.6.0 Release Notes -- The Consequence Substrate

**Release date:** 2026-03-26
**Lifecycle phase:** Preview
**Compatibility:** breaking-major (from 0.5.x)
**Contract line:** 0.6.x (16 substrate capabilities + 1 changed release capability)

## Summary

Theorem 0.6.0 is the release where the substrate becomes aware of its own consequences, proves its own correctness, and grows governed structure in response to consequence patterns. The consequence substrate adds 15 modules and 18,446 lines of code to `theorem-consequence`, backed by 12 computational engines in `theorem-analytics`. This release also introduces theorem-native computing -- a UEFI-native boot chain that eliminates the last third-party bootloader dependency. All 0.5.x substrate capabilities carry forward under the 0.6.x contract line.

## What's New

### Consequence Substrate

**theorem-consequence** -- 15 modules, 18,446 LOC, 541 tests.

The consequence substrate extends the governed platform with three new substrate axes: consequence awareness, self-proving correctness, and morphogenetic growth. It integrates `theorem-analytics`, `theorem-kernel`, `theorem-act`, `theorem-branch`, `theorem-auto`, `theorem-econ`, and `theorem-settle` into a single consequence-metabolizing loop.

**Seven resource axes.** The substrate manages resources across seven first-class axes: Time, Truth, Trust, Capital, Attention, Commitment, and Discovery. Each axis has typed handles, values, and adaptive phase tracking (`resource.rs`).

**Consequence loop.** The core execution cycle is: Ingest (consequence records and signals) -> Forecast (engine-backed prediction with confidence envelopes) -> Policy Inference (regime-shielded policy extraction) -> Hypothesize (intervention candidates with quantilized selection) -> Anti-Corruption (five corruption monitors) -> Simplex (barrier-function-guarded self-modification). The loop runs continuously in `loop_engine.rs` with circuit breaker integration from `theorem-auto`.

**Self-modification with barrier function safety.** `SimplexSupervisor` governs all parameter changes through control barrier function (CBF) evaluation. The L1ParameterTuner applies proportional corrections with drift-activation thresholds and per-parameter clamping. Every self-modification step produces a TuningRecord with before/after trajectory comparison. CBFs cover budget, trust, and drift dimensions (`simplex.rs`).

**Anti-corruption detection.** Five monitors run on every loop step (`anticorruption.rs`):
- GoodhartDetector -- correlation degradation via `theorem-analytics` statistical engines
- EntropyMonitor -- Shannon entropy, KL divergence, and transfer entropy via info-theory engine
- ImpactMonitor -- consequence impact divergence from predicted trajectories
- FeedbackMonitor -- feedback loop detection across consequence signals
- OverrideMonitor -- operator override classification (PolicyChange, OperatorPreference, BadModel) with override-aware corpus updates

**Bilateral negotiation protocol.** Zeuthen-Rubinstein convergence with monotonic concession guarantees. Full lifecycle: open/propose/counter/accept/reject. Accepted negotiations produce `CovenantContract` registrations. Stale sessions expire automatically (`negotiate.rs`).

**Cross-axis resource scheduler.** LP-backed claim resolution using the constraint optimization engine (revised simplex). Phase-aware allocation strategy with priority ordering across all seven resource axes (`scheduler.rs`).

**Commitment lifecycle.** `ObligationState` machine with governed transitions. `ObligationMonitor` ticks on every loop step, detects breaches, and computes aggregate `breach_probability` across active obligations (`commitment.rs`).

**Dempster-Shafer belief composition.** `BeliefState` composition across regimes using Dempster's rule of combination with pignistic transform for decision-making. Supports four disagreement policies: Conservative, Optimistic, CausalPriority, and OpenCounterfactual (`composition.rs`).

**Causal reasoning.** `CausalDAG` with Pearl do-calculus: intervene, identify, estimate. Wired to graph engine (PageRank, betweenness centrality, community detection) and Bayesian engine for causal effect estimation. `CausalityChain` integration from `theorem-act` records causal links on every consequence record (`causal.rs`).

**Morphogenetic growth.** `GoverningOrgan` and `BoundaryOrgan` types with `OperationalMorphology` registry. Governed organ lifecycle: growth from consequence field gradients (E23 provenance), verified integration preserving system invariants (E24), atrophy without consequence field support (E25), governed morphology transitions (E26) (`morphogenetic.rs`).

**Consequence field.** Signal gradients, topology, and attractors direct analytical capacity allocation. Phase-aware governance detects adaptive cycle phases (Exploitation, Conservation, Release, Reorganization) via dynamical systems engine and adjusts enforcement posture accordingly (`field.rs`).

**Continuous proof generation.** Proof certificate generation for enforcement decisions with Lob-safe verification hierarchy (no circular self-proof). Anti-corruption proofs verify detector correctness independently of substrate code (`proof.rs`).

**Cross-substrate federation.** Proof bundles for portable certificate packages. Federation registry with mutual verification. Cross-substrate act protocol carries proof bundles for verified inter-substrate acts (`federation.rs`).

**theorem-analytics** -- 12 computational engines, 454 tests.

| Engine | ID | Function |
|--------|----|----------|
| Statistics | A | Regression, correlation, confidence envelopes |
| Survival | B | Cox PH, Kaplan-Meier for time-axis predictions |
| Bayesian | C | Posterior updating, Bayesian network causal inference |
| State-Space/Kalman | D | Kalman filter for drift detection in trajectory models |
| Dynamical Systems | E | Coupled resource dynamics, RK4, Lyapunov exponents, CBF QP solver |
| Game Theory | F | Normal-form games, adversarial equilibrium, covenant analysis |
| Constraint Optimization | G | LP (revised simplex), QP (active-set), multi-objective Pareto |
| Decision Theory | H | VOI computation, MCTS for intervention search |
| Graph | I | PageRank, betweenness centrality, community detection |
| Monte Carlo | J | Particle filter, scenario generation |
| Information Theory | K | Shannon entropy, KL divergence, transfer entropy |
| Formal Methods | L | Proof certificate generation and verification |

### Theorem-Native Boot

UEFI-native boot chain with no third-party bootloader dependency.

- **theorem-uefi-loader** loads **theorem-machine-kernel** directly from the ESP partition. No GRUB, no intermediate bootloader.
- **Boot catalog** with generation selection -- multiple boot generations coexist on the ESP for atomic upgrades and rollback.
- **Kernel digest verification** -- cryptographic binding between loader and kernel via SHA-256 digest in the boot contract JSON.
- **Runtime handoff protocol** with activation manifests -- the loader passes a structured activation manifest to the kernel at handoff.
- **theorem-machine-kernel-efi** provides the EFI-specific kernel entry point for UEFI environments.

### Release Engineering

- **Dual-surface release pipeline.** FreeBSD carrier and theorem-native boot ship through a single governed release pipeline. Both surfaces share the same trust root, signing keys, and validation gate.
- **Contract line 0.6.x** with 16 substrate capabilities (12 carried from 0.5.x, 4 new: consequence-awareness, self-proving, morphogenetic-growth, theorem-native-boot) and 1 changed capability (governed-publication).
- **Signed metadata (Ed25519)** with trust root at `releases/trust-root.json`. All release metadata (manifest, integrity, release-notes, feed, revocation ledger, revocation records) is signed.
- **Machine-enforced release validation gate.** `validate-release.sh` must pass before any release publishes. Conformance evidence, demo-path evidence, release notes, and signed metadata are all gated.
- **Downgrade prevention.** Once a consumer admits the 0.6.x feed, minimum version floor is v0.6.0. Rollback requires explicit governed approval from root-governance-authority.

## Capabilities Carried Forward From 0.5.x

All 0.5.x substrate capabilities remain stable under the 0.6.x contract line:

| Capability ID | Surface |
|---------------|---------|
| substrate.zfs-storage | theorem-zfs, theorem-store |
| substrate.jail-compute | theorem-compute |
| substrate.pf-networking | theorem-net |
| substrate.encrypted-secrets | theorem-secrets, theorem-crypto |
| substrate.veriexec-integrity | theorem-init, veriexec.manifest |
| substrate.pid1-supervisor | theorem-init |
| substrate.branch-reality | theorem-branch |
| substrate.autonomous-governance | theorem-auto |
| substrate.economic-settlement | theorem-econ, theorem-settle |
| substrate.distributed-fabric | theorem-fabric |
| substrate.governed-images | theorem-image |
| substrate.device-governance | theorem-devices |

## Platform Support

| Target | Tier | Release Blocking |
|--------|------|-----------------|
| x86_64-unknown-freebsd | Primary | Yes |
| x86_64-unknown-uefi | Planned | No |
| x86_64-unknown-linux-gnu | Legacy | No |
| aarch64-unknown-freebsd | Planned | No |

## Upgrade Path

### From 0.5.x (FreeBSD)
- **Mode**: In-place boot environment upgrade
- **Artifact**: `upgrade-bundle`
- **Procedure**: Run `upgrade.sh` from the upgrade-bundle archive on a running 0.5.x system
- **State preservation**: theorem/store, theorem/log, and /var/lib/theorem carry across the boot-environment switch
- **Rollback**: Activate previous boot environment via beadm/bectl and reboot

### From 0.5.x (theorem-native)
- **Mode**: UEFI ESP upgrade
- **Artifact**: `uefi-esp-image`
- **Procedure**: Write ESP image to target partition, update UEFI NVRAM boot entry
- **Rollback**: Restore previous ESP generation from backup partition

## Required Operator Actions

1. Consume `releases/feed.json`, `releases/v0.6.0/manifest.json`, and `releases/v0.6.0/integrity.json` before mirroring or admitting a 0.6.x release.
2. Deploy on FreeBSD 14.4-RELEASE or later with ZFS root pool named `theorem`.
3. Use `theoremos-v0.6.0-amd64-direct-write.img.xz` for unattended bare-metal direct disk writes. Use `theoremos-v0.6.0-x86_64-unknown-freebsd.tar.gz` for installer-driven bare-metal flows. Treat ISO and memstick assets as install media only.
4. Install `theorem-init` to `/sbin/theorem-init` and remaining binaries to `/usr/local/bin/`.
5. Load PF kernel module and configure `/etc/pf.conf` with theorem anchor.
6. Review `/etc/veriexec.d/theorem.manifest` and optionally load `mac_veriexec` for verified execution.
7. Verify signed release metadata against `releases/trust-root.json`.
8. For theorem-native targets: write the ESP image to a GPT ESP partition and configure UEFI NVRAM boot entry.
9. Verify the boot contract JSON references the correct kernel SHA-256 before deploying theorem-native boot.

## Capability Negotiation

- **Discovery at release time**: `releases/feed.json`, `releases/v0.6.0/manifest.json`
- **Discovery at runtime**: `GET /capabilities`, `GET /capability-negotiation`
- **Stability rule**: Capability IDs published in a 0.6.x manifest are stable within the 0.6.x line once released.
- Gate on `compatibility_classification` before enabling a new capability ID.
- Treat manifest capability IDs and `release_integrity` metadata as authoritative rather than inferring behavior from prose notes or semver alone.

## Security

- Same trust root and Ed25519 signing keys as 0.5.0.
- All artifacts (FreeBSD and theorem-native surfaces) signed through the same `sign-release-file.ps1` pipeline.
- Downgrade prevention enforced: minimum version floor is v0.6.0 once admitted.
- Anti-corruption monitors (Goodhart, entropy, impact, feedback, override) run continuously on every consequence loop step.
- Barrier function safety bounds all self-modification through CBF evaluation.
- Lob-safe proof verification hierarchy prevents circular self-proof.

## Release Integrity

- **Evidence**: `releases/v0.6.0/integrity.json`
- **SBOM**: CycloneDX BOM for each crate (30 `.cdx.xml` assets)
- **Provenance**: `theorem-node-v0.6.0.provenance.json`
- **Revocation**: `releases/revocations.json` (ledger), `releases/revocations/v0.6.0.json` (record)
- **Mirror TTL**: 24 hours, fail-closed on stale
