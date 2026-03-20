# Deterministic AI Governance — State of the Field

**Date:** 2026-03-16

## Purpose

This document maps the current landscape of deterministic governance for AI agents. It describes what approaches exist, what they can and can't do, and where the gaps are. Use this to validate design decisions and identify where Theorem's architecture occupies uncontested ground.

---

## Approaches That Exist

The field has converged on several patterns. Most systems implement one or two. None implement all of them.

### 1. Runtime Policy Interception (Sidecar Pattern)

The dominant approach. A governance layer sits between agents and the outside world, intercepting actions before execution. Policy is evaluated at runtime — deterministic but not compile-time enforced.

**What it does well:**
- Works with any agent framework without modification
- Sub-millisecond evaluation latency is achievable
- Can be added to existing systems without rebuilding them
- Policy-as-code, version-controlled, testable

**What it can't do:**
- Can't govern processes from birth — only intercepts at the boundary
- Runtime enforcement means a code path that bypasses the interceptor is theoretically possible
- No mechanism to transition a system from "governed at the boundary" to "structurally governed"
- The governance layer and the governed system share failure domains (usually same infrastructure)

**Prevalence:** Multiple production-grade implementations exist. Open-source and commercial. Several have broad framework compatibility (10+ integrations). This is the table-stakes approach — well-understood, widely deployed.

### 2. Sovereignty Kernels (Enclave Pattern)

Emerging in academic research. A kernel launches and governs processes it controls. Agents are born inside the governance boundary. Formal invariants with structured proof sketches.

**What it does well:**
- Processes are governed from birth — no gap between launch and governance
- Capability-based isolation
- Merkle tree audit logs with compact inclusion proofs
- Sub-2ms latency, hundreds of actions per second
- Formally stated invariants (published, peer-reviewable)

**What it can't do:**
- Can't govern external systems — enclave-only, no inbound API
- No migration path for existing ungoverned systems
- Attestation proofs stored on the same system (not hardware-independent)
- No compile-time enforcement of write restrictions

**Prevalence:** Academic. At least one published Rust implementation with formal proof sketches (early 2026). Not yet productized.

### 3. Cryptographic Attestation (Observer Pattern)

Systems that create tamper-evident records of AI interactions. Hash requests and responses, batch into Merkle trees, anchor to blockchain or append-only logs. Independent verification without service cooperation.

**What it does well:**
- Creates independently verifiable receipts for every interaction
- Users can verify directly — no trust in the service required
- Blockchain anchoring provides decentralized tamper evidence
- Open-source implementations exist

**What it can't do:**
- No enforcement — observer only. Records what happened but can't prevent anything
- No governance logic — no concept of rules, rulings, or rejections
- No enclave or sidecar capability
- No formal verification of properties

**Prevalence:** Several open-source tools and at least one patent filing. Active area but narrow scope — attestation, not governance.

### 4. Governance-as-a-Service (Academic Pattern)

Frameworks that separate governance from agent cognition entirely. Declarative rules, trust scoring, graduated enforcement (coercive, normative, adaptive). Agents don't need to cooperate — governance layer regulates outputs independently.

**What it does well:**
- Clean separation of governance from agent internals
- Trust scoring mechanisms that adapt enforcement based on compliance history
- Model-agnostic — tested across multiple model families
- Graduated enforcement rather than binary allow/deny

**What it can't do:**
- Simulation-only — not deployed in production
- No cryptographic attestation
- No formal verification
- No independent witness
- No enclave mode

**Prevalence:** Academic papers (mid-2025 onward). Theoretical frameworks, not shipped systems.

---

## Gaps in the Field

These are things no existing approach does:

### No system offers both enclave and sidecar with a migration path between them
Sidecar systems can't birth-govern. Enclave systems can't accept external callers. Nobody offers a way to take a system from "governed at the boundary" to "born governed." The transition itself is undefined in the literature.

### No system uses compile-time enforcement of governance invariants
Every existing approach uses runtime evaluation — policy is checked when an action occurs. No system uses the type system to make ungoverned writes structurally impossible at compile time. The distinction: runtime enforcement means "there is a check that prevents it." Compile-time enforcement means "the code that would do it cannot be written."

### No system places the attestation witness on physically independent hardware
Merkle trees on the same system. Blockchain anchoring (shared infrastructure). Append-only logs on the same machine. Nobody puts the witness on a separate machine with separate credentials and separate storage, where full compromise of the governed system leaves the witness unaffected.

### No system defines governance as a pure function
Deterministic policy evaluation exists. But "deterministic" and "pure function" are different. A pure function has no I/O, no randomness, no side effects — same inputs always produce identical outputs. Existing systems have I/O in the evaluation path (reading policy files, checking databases, calling external services). A truly pure governance kernel is formally verifiable without modeling the external world.

### No system addresses "trust verification of outputs, not provenance of weights"
Existing governance evaluates what model was used, what permissions the agent has, what policy applies. Nobody evaluates the structural properties of the output itself, independent of where it came from. This matters because models change faster than certifications — governing outputs is future-proof in a way that governing model provenance is not.

---

## Regulatory Context

The EU AI Act mandates automatic logging for high-risk AI systems, enforceable August 2026. This creates a compliance floor that simple logging and attestation tools can satisfy.

The question is whether the market will demand only the floor (logging) or something above it (provable governance with formal properties). Sectors with real liability — finance, healthcare, defense, critical infrastructure — are more likely to need the latter. But the broader market may settle for the floor.

Mapping governance rules to specific regulatory articles, with formal proofs of compliance, would create a moat that runtime policy engines cannot match. No existing system has done this.

---

## Relevant Academic Work

- Sovereignty kernels for verifiable agent execution (arXiv, February 2026)
- Governance-first paradigms for principled agent engineering (arXiv, October 2025)
- Constant-size cryptographic evidence structures for regulated AI workflows (arXiv, November 2025)
- Verifiable AI safety benchmarks using trusted execution environments (arXiv, June 2025)
- Governance architecture for autonomous agent systems (arXiv, March 2026)
- Infrastructural sovereignty and diffused accountability in decentralized AI (arXiv, February 2026)
- Autonomous agents on blockchains: standards, execution models, trust boundaries (arXiv, January 2026)
- High-performance co-designed tamper-evident logging (arXiv, September 2025)
- Multi-agent compliance framework with trust factor scoring (arXiv, August 2025)
