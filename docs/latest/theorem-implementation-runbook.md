# Trust Kernel — Implementation Runbook

---

| Step | Who | Application / Configuration | Prompt / Command |
|------|-----|---------------------------|-----------------|
| **1. Create directory** | Eric | PowerShell | `mkdir C:\Users\Eric\Theorem` |
| **2. Save spec** | Eric | PowerShell | `copy C:\Users\Eric\Downloads\theorem-spec.md C:\Users\Eric\Theorem\theorem-spec.md` |
| **3. Compute spec hash** | Eric | PowerShell | `Get-FileHash C:\Users\Eric\Theorem\theorem-spec.md -Algorithm SHA256` |
| **4. Record spec hash externally** | Eric | PowerShell | `echo <HASH> > C:\Users\Eric\Theorem\theorem-spec.sha256` — the spec references this file, not its own hash. |
| **5. Update spec hash reference** | Eric | Text editor | Open `theorem-spec.md`. Replace hash placeholder with `Computed externally. See theorem-spec.sha256.` Save. Re-run step 3. Overwrite sha256 file with new hash via step 4. |
| **6. Verify no project memory** | Eric | PowerShell | `Get-ChildItem "C:\Users\Eric\.claude\projects" -Name \| Select-String "Theorem"` — should return nothing. If a directory exists, inspect its memory files before proceeding. |
| **7. Close Claude Code terminals** | Eric | Manual | Close all Claude Code terminal windows manually. Do NOT run `Stop-Process` against claude processes — it kills the web chat too. |
| **8. Open fresh PowerShell** | Eric | New PowerShell window (not VS Code) | |
| **9. Navigate** | Eric | PowerShell | `cd C:\Users\Eric\Theorem` |
| **10. Launch Claude Code** | Eric | PowerShell | `claude --dangerously-skip-permissions` |
| **11. Trust the folder** | Eric | Claude Code | Select `1. Yes, I trust this folder` |
| **12. Re-compute spec hash** | Eric | PowerShell (new window, not inside Claude Code) | `Get-FileHash C:\Users\Eric\Theorem\theorem-spec.md -Algorithm SHA256` then `echo <NEW_HASH> > C:\Users\Eric\Theorem\theorem-spec.sha256` — must happen every pass, not just first time. |
| **13. Run pre-flight consistency check** | Claude Code | Claude Code, dangerously-skip | Paste Prompt C below. Fast pass — catches unsatisfied propositions and dangling references before burning a full audit. |
| **14. Review pre-flight output** | Eric | PowerShell (new window) | `Get-Content "C:\Users\Eric\Desktop\theorem-preflight.md"` — if any FAIL lines, remediate in web chat before proceeding. Re-run steps 7-14 after fixes. |
| **15. Run spec audit** | Auditor (Claude Code) | Claude Code, dangerously-skip | Paste Prompt A below. |
| **16. Wait** | — | — | Do not interrupt. Do not open other terminals. Do not modify files in Theorem. |
| **17. Read audit summary** | Eric | PowerShell (new window, not inside Claude Code) | `Get-Content "C:\Users\Eric\Desktop\theorem-audit-summary.md"` |
| **18. Review findings** | Eric + Claude (web chat) | This conversation | Upload audit output files here. Review each finding. |
| **19. Remediate all findings** | Eric + Claude (web chat) | This conversation | Draft fixes here. Apply to `theorem-spec.md`. No findings accepted — all remediated. |
| **20. Repeat steps 7-19** | Eric + Auditor | Same as above | Repeat until audit passes with zero findings. |
| **21. Build deterministic verifier** | Claude Code (fresh terminal) | Claude Code, dangerously-skip, in Theorem directory | Paste Prompt B below. |
| **22. Calibrate verifier — fail test** | Eric | PowerShell | `python C:\Users\Eric\Theorem\scripts\verify.py "C:\Users\Eric\Desktop\2026-03-10-epoch-collision-exhibit"` — must return AUDIT INVALID. |
| **23. Calibrate verifier — pass test** | Eric + Claude Code | Claude Code | Generate a synthetic conforming dataset. Run verifier. Must return AUDIT VALID. |
| **24. Architect the implementation** | Eric + Claude (web chat) | This conversation | Define: target platform (hosted or standalone), tech stack, deployment model, schema storage, enforcement rule execution model, attestation log technology, identity binding mechanism, liveness mechanism, cryptographic suite, substrate trust model. |
| **25. Build the implementation** | Claude Code (fresh terminal per module) | Claude Code, dangerously-skip, in implementation repo | One module at a time. Schema primitives first, enforcement rules second, audit criteria third, attestation lifecycle fourth, proof obligation mechanism fifth, deterministic verifier integration sixth. |
| **26. Bootstrap** | Implementation | Running instance | Execute bootstrap GOVERNANCE_ACT. Establish GENESIS (with specification_version_hash), KERNEL_PARAMETERS (all required fields including identity_binding_model, system_entity_liveness_model, parameter_bounds, substrate_trust_declaration, approved_algorithms, max_degraded_window_ms, proof_expiry_max_cycles), system entity, initial PROOF_OBLIGATION records for all 25 properties. |
| **27. Submit initial proofs** | Implementation operator (Eric) | Running instance | For each of the 25 properties: submit proof via governance_write. Batch submission allowed under adoption exemption. Genesis-time properties (root properties with no dependencies: PROP-PARAM-001, PROP-PARAM-003, PROP-IDENTITY-001, PROP-LIVENESS-001, PROP-SUBSTANCE-001 through 005, PROP-INFRA-001, PROP-INFRA-002, PROP-PROOF-001, PROP-PROOF-002, PROP-ENFORCEMENT-001, PROP-SINCERITY-001, PROP-SUBSTRATE-001) submitted as bootstrap authorized side effects. |
| **28. Run first deterministic verification** | Deterministic verifier (automated) | Script against running instance | `python verify.py <audit_output_directory>` — Layer 7a. All 25 properties, 8 boundaries, C15-C22, E14-E16. Pass or fail. |
| **29. Engage independent physical auditor** | Eric (procurement) | External — not Claude, not Eric | Independent auditor with: no relationship to the implementation, physical access capability, professional liability coverage. Cannot be the spec author or implementation builder. For critical/high risk: engage two independent auditors. |
| **30. Physical audit — PA-001** | Independent auditor | Physical access to identity binding infrastructure | Inspect binding mechanism. Verify Sybil resistance for deployment context. Produce signed attestation per PROP-IDENTITY-001. |
| **31. Physical audit — PA-002** | Independent auditor | Physical access or simulation environment | Execute timed revocation test. Measure completion time. Produce signed attestation per PROP-IDENTITY-003. |
| **32. Physical audit — PA-003** | Independent auditor | Physical access to deployment infrastructure | Inspect deployment topology. Verify liveness mechanism independence. Produce signed attestation per PROP-LIVENESS-001. |
| **33. Physical audit — PA-004** | Independent auditor | Supervised simulation environment | Suppress system entity. Measure detection and suspension timing. Produce signed attestation per PROP-LIVENESS-002. |
| **34. Physical audit — PA-005** | Independent auditor | Red-team exercise | Attempt exploitation of recovery procedure against adversary models R1-R8. Verify all propositions hold post-recovery. Produce signed attestation per PROP-LIVENESS-003. |
| **35. Physical audit — PA-006** | Independent auditor | Supervised simulation environment | Disable infrastructure component. Verify degraded mode behavior and reconciliation. Produce signed attestation per PROP-AVAILABILITY-001. |
| **36. Physical audit — PA-007** | Independent auditor | Instance access | Select 5 governance artifacts that passed deterministic checks from ≥ 3 different entities. Evaluate each against 5-dimension deliberation rubric: specificity, decision cost, constraint engagement, temporal awareness, falsifiability. Score pass/fail per dimension. ≥ 2 fails = "suspected theater." > 1 of 5 suspected = finding. Produce signed attestation per PROP-SINCERITY-001. |
| **37. Physical audit — PA-008** | Independent auditor | Physical access to hardware/deployment | Verify substrate trust declaration matches reality. Check TEE attestation (if declared), boot measurement log, clock source, cryptographic library version and validation status. Document unattested components as explicit risk acceptance. Produce signed attestation per PROP-SUBSTRATE-001. |
| **38. Meta-verification (Layer 7c)** | Eric or second auditor | Review of steps 28 and 30-37 | Run deterministic meta-checks: all 25 properties checked by 7a, all 8 PA attestations present with specific details, auditor independence declarations present, auditor rotation met. Perform judgmental meta-checks: PA evidence quality, red-team substantiveness, rubric calibration, spirit-of-verification assessment. Produce META_VERIFICATION_REPORT as governance record. Pass criteria: all deterministic meta-checks pass, no judgmental check rated "material concern." |
| **39. First audit cycle complete** | — | — | If step 28 passed, all 8 PA attestations pass, and meta-verification passed: the implementation is conforming. If any failed: remediate and repeat from the failing step. |
| **40. Ongoing** | All parties per role | Per `audit_interval_ms` | Deterministic verifier runs every cycle. All 8 PA exits repeat every cycle. Auditor rotates after two consecutive cycles. Proofs renewed before expiry. Enforcement rule test vectors re-run per PROP-ENFORCEMENT-001. System entity observation verified per PROP-LIVENESS-004. |

---

## Prompt A — Spec Audit

```
You are auditing the Trust Kernel specification at C:\Users\Eric\Theorem\theorem-spec.md. This is a specification audit — the subject is a formal specification document, not a running implementation.

Read the full specification before beginning any analysis.

Evaluate:
1. Internal consistency — do propositions, enforcement rules, audit criteria, properties, and PA exits form a coherent system with no contradictions?
2. Dangling references — does every enforcement rule reference valid propositions? Does every audit criterion reference valid enforcement rules? Does every property reference a valid adversary model from the Adversary Model Catalog (R1-R8)? Does every PA exit reference a valid PROP-* identifier?
3. Completeness — are there enforcement rules without corresponding audit criteria? Properties without verification procedures? PA exits without attestation requirements? Adversary models without corresponding properties? Genesis-time properties listed in E9 that don't exist in the catalog?
4. Substance gap — are there claims in the propositions that no enforcement rule, property, or PA exit actually verifies? Does PROP-LIVENESS-004 close the P3 observation gap? Does PROP-ENFORCEMENT-001 close the enforcement correctness gap?
5. Circular dependencies — does the proof dependency graph contain cycles? Does the enforcement rule set contain circular references? Are designed cycles (audit-governs-audit, spec self-reference) properly broken with explicit regress termination statements in normative text?
6. Specification self-consistency — does the specification satisfy its own requirements per the Specification Self-Governance section? Does the amendment governance section address P14 applicability? Is the P10 self-governance boundary explicitly stated?

The specification has 15 propositions (P1-P15), 16 enforcement rules (E1-E16), 22 audit criteria (C1-C22), 25 properties across 8 boundaries, 8 physical audit exits (PA-001 through PA-008), an Adversary Model Catalog (R1-R8), and a Specification Self-Governance section.

Follow the auditor skill at plugins\auditor\skills\auditor\SKILL.md if available. If not available, proceed with the evaluation above using your own methodology.

Write all output to C:\Users\Eric\Theorem\audit\
Write a summary to C:\Users\Eric\Desktop\theorem-audit-summary.md

This is autonomous — no human is available. If you encounter an issue that blocks the audit, document it in the summary and stop. Do not skip checks. Do not proceed past contradictions without documenting them.
```

---

## Prompt B — Deterministic Verifier

```
Build C:\Users\Eric\Theorem\scripts\verify.py — a standalone Python script (stdlib only, no external dependencies) that validates Trust Kernel audit structural integrity.

Read the Trust Kernel specification at C:\Users\Eric\Theorem\theorem-spec.md first. The verifier must implement Layer 7a as defined in the specification.

The script takes one argument: path to an audit output directory.

It must check:

STRUCTURAL INTEGRITY
- Epoch manifest exists and is valid JSON
- All corpus hashes match
- All evidence records present
- SARIF cross-references valid

PARAMETER OUTCOMES (PROP-PARAM-001, 002, 003)
- All parameters within declared bounds
- No multi-parameter amendments
- No excessive amendment frequency
- Trajectory monitoring within thresholds

ENTITY IDENTITY (PROP-IDENTITY-001, 002, 003)
- All active entities have identity_attestation_ref
- No principal binding hash collisions
- No expired identity attestations on active entities

SYSTEM ENTITY LIVENESS (PROP-LIVENESS-001, 002, 003, 004)
- Liveness mechanism active
- Most recent liveness proof within interval
- Dependency graph excludes system entity
- System entity escalations reference specific governance state observations (not fixed schedule)
- Escalation interval coefficient of variation > 0.05
- State-reactive test: trigger condition produces escalation within detection_interval_ms

GOVERNANCE SUBSTANCE (PROP-SUBSTANCE-001 through 006)
- No tautological intent patterns
- Intent duplicate rate < 10% per entity
- Paths-not-taken pass negation and concreteness checks
- Alternatives failing substance checks produce insufficient_alternative_substance escalation
- Measurement methods show non-trivial variance AND pass test_vector checks (known-misaligned inputs)
- Safety assessments meet coverage and independence
- Cross-artifact uniformity below thresholds
- Hollow instance test: two-alternative template rejected by substance checks not structural minimum count

EXTERNAL INFRASTRUCTURE (PROP-INFRA-001, 002, 003)
- Verification artifacts consistent across locations
- Attestation log integrity chain valid
- Infrastructure changes preceded by governance_write acts

ENFORCEMENT CORRECTNESS (PROP-ENFORCEMENT-001)
- Test vectors present for each enforcement rule E1-E16
- At least 2 valid, 2 invalid, 1 boundary case per rule
- All test vectors produce correct accept/reject results
- E7 depth-3 mutual authorization rejected
- E14 tautological template rejected by substance not structure
- E2 disconnected chain (valid one-hop, no path to bootstrap) rejected at runtime

PROOF AND SPECIFICATION INTEGRITY (PROP-PROOF-001, 002, PROP-AVAILABILITY-001)
- Proof dependency graph acyclic
- All active proofs reference current specification version
- Cascade invalidation complete
- Degraded mode records flagged and reconciled
- All root properties carry depends_on_substrate: true
- Substrate invalidation cascades to all properties

SINCERITY AND SUBSTRATE (PROP-SINCERITY-001, PROP-SUBSTRATE-001)
- PA-007 attestation present with 5-dimension rubric scores per artifact
- PA-008 attestation present with substrate trust declaration verification

PHYSICAL AUDIT ATTESTATIONS (PA-001 through PA-008)
- All 8 attestations present
- Each references specific PROP-* identifier
- Each contains procedure-specific details (not generic sign-offs)
- Auditor independence declaration present on each
- Auditor rotation verified (no same auditor > 2 consecutive cycles)

ADVERSARY MODEL COVERAGE
- Every property references at least one R# from the Adversary Model Catalog
- Every R1-R8 is referenced by at least one property

PROOF STATUS
- Every required property (25 total) has an active proof
- No expired proofs
- No unresolved proof challenges

Output: PASS or FAIL per check with details. Final line: AUDIT VALID or AUDIT INVALID. Exit code 0 for valid, 1 for invalid.

No external dependencies. Python stdlib only (hashlib, json, os, sys, pathlib, datetime).

This is a script, not an LLM. It checks math, not meaning.
```

---

## Prompt C — Pre-Flight Consistency Check

```
You are running a fast structural consistency check on the Trust Kernel specification at C:\Users\Eric\Theorem\theorem-spec.md. This is NOT an audit. This is a pre-flight that catches mechanical errors before the audit runs.

Read the full specification. Then execute every check below. For each check, output PASS or FAIL with the specific identifiers that failed.

PROPOSITION TRACEABILITY
- For each proposition P1-P15: identify the enforcement rule(s) and audit criterion/criteria that mechanically satisfy it. If a proposition has no enforcement rule or no audit criterion that references it, output FAIL with the proposition ID.

ENFORCEMENT RULE COMPLETENESS
- For each enforcement rule E1-E16: verify it references at least one proposition. Verify at least one audit criterion references this rule. If orphaned in either direction, FAIL.

AUDIT CRITERIA COVERAGE
- For each audit criterion C1-C22: verify it references at least one enforcement rule. If it references a rule that doesn't exist, FAIL.

PROPERTY-TO-PROPOSITION MAPPING
- For each property PROP-*: verify the proposition it claims to verify exists. If the proposition ID is invalid or missing, FAIL.

PHYSICAL AUDIT EXIT MAPPING
- For each PA-001 through PA-008: verify the PROP-* it references exists in the property catalog. If missing, FAIL.

ADVERSARY MODEL COVERAGE
- For each R1-R8: verify at least one property references it. For each property: verify it references at least one R#. Orphans in either direction are FAIL.

SCHEMA PRIMITIVE FIELD COMPLETENESS
- For each schema primitive defined in the spec: verify every field referenced by an enforcement rule actually exists on the primitive. If an enforcement rule references a field that isn't defined on the primitive, FAIL.

GENESIS PROPERTY CONSISTENCY
- Verify every property listed as genesis-time in E9 exists in the property catalog. Verify every property listed as genesis-time has depends_on: [] (empty) or depends_on_substrate: true only. If a genesis property has non-empty depends_on, FAIL.

PROOF DEPENDENCY ACYCLICITY
- Trace the depends_on graph across all 25 properties. If there is a cycle, FAIL with the cycle path.

Write output to C:\Users\Eric\Desktop\theorem-preflight.md

Format: one line per check. PASS or FAIL. If FAIL, list every failing identifier on the same line. Final line: PREFLIGHT CLEAN or PREFLIGHT DIRTY with total fail count.

Do not remediate. Do not suggest fixes. Do not editorialize. Report only.
```


