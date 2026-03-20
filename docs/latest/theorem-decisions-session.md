# Trust Kernel — Session Decision Ledger

Decisions made during the implementation session on 2026-03-11. These need to be flushed to DECISIONS.md in the ref-impl repo when back in Claude Code.

---

## DEC-013: Substrate trust independence

**Decision:** Builder kernels declare their own substrate trust independently. No inheritance chain from platform kernel.

**Mechanism:** Builders reference the platform governance record (attestation log URL + instance ID), not the platform substrate declaration. Declaration says "runs on infrastructure governed by [instance ID]" not "runs on [specific hardware/provider]."

**Rationale:** Inheritance cascades proof invalidation on infrastructure change. Reference survives change. A builder leaving the platform updates their own substrate declaration — no cascade, no proof invalidation on the governance layer. Clean exit.

**Informed by:** PROP-SUBSTRATE-001, PROP-PROOF-001 cascade mechanism.

**Status:** Decided. Shapes builder kernel bootstrap.

---

## DEC-014: Attestation serialization format

**Decision:** Deferred to Rust phase.

**Constraint:** Irreversible once first attestation is published and a third party verifies it. Format must produce identical bytes for identical input, forever.

**Candidates:** Canonical JSON (RFC 8785), CBOR, protobuf.

**Timing:** Must be decided before first production bootstrap.

**Status:** Open.

---

## DEC-015: GENESIS hash algorithm

**Decision:** Deferred to first production bootstrap.

**Constraint:** Irreversible. Write-once on GENESIS records. KERNEL_PARAMETERS.genesis_hash_algorithm must match.

**Current candidate:** SHA-256.

**Risk:** Quantum resistance is a declared risk acceptance, not a solved problem. Spec allows rotating approved_algorithms for future operations, but GENESIS records carry the original algorithm forever.

**Informed by:** GENESIS constraints, PROP-INFRA-002 minimum strength floor.

**Status:** Open.

---

## DEC-016: Linux substrate image

**Decision:** Phase 3, after Rust engine.

**Shape:** Minimal image (NixOS or buildroot), engine as primary userspace process, measured boot chain (UEFI Secure Boot → kernel signature → engine binary hash), pinned crypto library, declared clock source.

**Rationale:** The kernel controlling its own substrate means PROP-SUBSTRATE-001 declaration and reality are the same thing. PA-008 becomes a hardware audit.

**Informed by:** Ring 1 architecture, PROP-SUBSTRATE-001.

**Status:** Roadmap. Architecture supports it.

---

## DEC-017: Instance provisioning model

**Decision:** Not yet decided.

**Constraint:** Sub-second bootstrap required for platform-speed builder onboarding. Bootstrap (GENESIS, KERNEL_PARAMETERS, system entity, initial PROOF_OBLIGATION records for 25 properties) is non-trivial.

**Shapes:** Builder kernel lifecycle, Ring 3 agent runtime integration.

**Timing:** Must be decided before Ring 3 integration.

**Status:** Open.

---

## DEC-018: Attestation witness network

**Decision:** Ring 2 infrastructure. Not yet designed.

**Shape:** Platform-hosted first attestation location + public witness network as second independent location. Mutual witnessing, not consensus. Instances publish attestations and verify each other's logs.

**Constraint:** P13 requires at least two independent locations. "Independent" means different DNS registrars, different hosting providers, or no shared administrative credentials.

**Timing:** Required before multi-tenant production.

**Informed by:** P13, INTEGRITY_MECHANISM.public_verification_artifact_locations.

**Status:** Open.

---

## Architecture — Ring Model

For reference, the concentric ring architecture discussed:

- **Ring 0:** Kernel engine (Rust binary). Spec's 15 propositions, enforcement rules, proof lifecycle.
- **Ring 1:** Substrate (Linux host). Measured boot, pinned crypto, declared clock.
- **Ring 2:** Attestation network. Peer instances cross-verifying attestation logs. Gossip, not consensus.
- **Ring 3:** Agent runtime. Platform execution layer. Agents as governed entities with authorizations.
- **Ring 4:** Application layer. Generated apps. Each tenant's operations flow through Ring 3 → Ring 0.
- **Ring 5:** Verification ecosystem. Independent auditors, PA-001 through PA-008. External by design.

## Scalability — Three-Layer Governance

- Layer 1: Platform kernel (one instance, moderate volume)
- Layer 2: Builder kernels (one per builder, scales horizontally)
- Layer 3: End-user kernels (optional, per builder's end users)

Key scaling concerns: instance provisioning speed, attestation storage at scale, cross-layer verification via attestation log references (not inheritance).

## Irreversible Decisions Identified

1. Attestation serialization format (DEC-014)
2. GENESIS hash algorithm (DEC-015)
3. Identity binding model (amendable by spec, effectively irreversible at scale)
4. Substrate dependency model between platform and tenant kernels (DEC-013 — decided: independence via reference)

---

## DEC-019: Ring 1 substrate hardware

**Decision:** First production substrate is a dedicated physical machine under Eric's direct control. i7-8700, 32GB RAM, GTX 2080. Located in Eric's home.

**Target OS:** NixOS. Declarative, reproducible, version-controlled OS config. Engine runs as systemd service.

**Boot chain:** UEFI Secure Boot with self-managed keys → NixOS kernel signature → Rust engine binary hash. Full measured boot for PROP-SUBSTRATE-001.

**Clock:** NTP authenticated.

**Storage:** Local SSD for attestation log. Replication to one external location (Ring 2, DEC-018).

**GPU:** Not used for kernel operations. Available for local inference if Ring 3 agent runtime runs on same hardware.

**Substrate trust declaration:** "Runs on hardware physically controlled by the operator, inspectable by any auditor granted physical access." The most honest PA-008 declaration possible.

**Timing:** Phase 3, after Rust engine (DEC-016). Machine is available now.

**Informed by:** PROP-SUBSTRATE-001, PA-008, Ring 1 architecture.

---

## DEC-020: NAS as second independent attestation location

**Decision:** Synology NAS serves as the second independent hosting location for public verification artifacts (P13) and as the attestation log archive.

**Independence claim:** Different hardware, different storage medium, different failure domain from the Ring 1 substrate machine. Same physical location but independent access-control domain (separate admin credentials, separate network interface).

**Functions:**
- Second location for INTEGRITY_MECHANISM.public_verification_artifact_locations
- Attestation log replication target (append-only, RAID-backed)
- Long-term archive for governance records beyond the active kernel's working set

**Limitation:** Same physical location as Ring 1 machine. Two independent failure domains (hardware, storage, credentials) but one geographic failure domain. Geographic redundancy requires a third location — Ring 2 witness network or a remote replication target. Declared as a known risk acceptance in the substrate trust declaration. Strong for governance integrity (physical control, real PA-008). Weak for durability (fire, flood, power loss takes both).

**Timing:** Available now. Configure when Ring 1 is operational.

**Informed by:** P13, INTEGRITY_MECHANISM.public_verification_artifact_locations, ATTESTATION_LOG technology requirements (3-year retention).

---

## DEC-021: Local inference calibration test — 8B as accidental adversary

**Decision:** Run a quantized 8B model (Qwen 2.5 7B or Llama 3.1 8B) on the gaming rig's GTX 2080 as the agent producing governance artifacts. The kernel's substance checks (E14, PROP-SUBSTANCE-001 through 006) must reject the 8B model's outputs as insufficiently deliberative. Then run a 30B model (requires GPU upgrade to 3090 or CPU offloading) and verify it passes the same checks.

**Purpose:** Calibration of substance verification against the real-world threat — not sophisticated actors gaming the system (R6 intentional), but sincere agents producing hollow artifacts because they lack the capacity for genuine deliberation (R6 accidental). The 8B model is an unintentional compliance theater generator.

**Product insight:** "Not every model is fit to govern. The kernel proves which ones are." The capability threshold where governed AI becomes real, not theater, is demonstrable. This is the demo.

**Hardware:** GTX 2080 (8GB VRAM) runs 8B models at Q4_K_M, 20-30 tokens/sec. Sufficient for POC. GPU upgrade to RTX 3090 (24GB VRAM, ~$800-900 used) required for 30B models at full speed.

**Timing:** After Rust engine, when Ring 3 agent runtime is operational. Can prototype earlier against the TypeScript reference impl with Ollama.

**Informed by:** R6 (Compliance Theater), E14, PROP-SUBSTANCE-001 through 006, PROP-SINCERITY-001.

