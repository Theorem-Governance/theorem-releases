# Theorem 4.0 Downstream Pre-Release Memo

Date: 2026-03-23

Audience: downstream platform operators, integration owners, and release managers

Status: public pre-release memo for the staged candidate; final verification audit closeout pending

Candidate tag: `v4.0.1`

## Position

`4.0` is the release where Theorem stops being only a governance layer above the
machine and starts presenting a downstream-consumable machine contract.

This candidate is aimed at downstream teams that need to run a governed
multi-tenant platform with explicit release evidence, operator authority,
recovery semantics, and automation surfaces.

It is being published before final declaration so downstream teams can prepare
against the real contract in advance rather than reverse-engineering the final
release after the fact.

The short version:

`4.0` turns several previously soft edges into shipped product law. Release
trust, recovery truth, tenant boundaries, capacity admission, operator
authority, and downstream automation are no longer supposed to live in private
runbooks or wrapper glue.

## What We Are Shipping

The public `4.0` line is built around six release-defining surfaces:

- governed release artifacts with manifest, integrity evidence, revocation
  paths, and stale-mirror failure
- measured-boot recovery that is judged by authority-state equivalence, not
  merely successful reboot
- explicit tenant and delegated-admin boundaries for shared-fleet operation
- governed fleet economics, reservation, and capacity-admission law
- bounded provisioning and runtime-control surfaces for platform operators
- downstream automation, compatibility, and operator-guide surfaces intended to
  stand on their own as product contract

If you want the release in one sentence:

Theorem `4.0` is the first line we are willing to hand to a downstream platform
operator and say, "use the published contract, not our private memory, to
decide what may run, what may change, how recovery works, and whether the
release itself is trustworthy."

At publication time, the public repo is expected to ship:

- `README.md` and `quickstart.md` for fast install
- `RELEASE-MEMO.md` for downstream release framing
- operator guides for the current `4.0` contract line
- release manifest and integrity evidence
- conformance and rehearsal artifacts
- the machine-readable release feed

## Feature Set

The current candidate already makes these feature families explicit and
downstream-visible:

- Release integrity:
  minimum-version floors, rollback authorization, revocation records, SBOM and
  provenance attachment requirements, and mirror freshness semantics
- Recovery:
  measured boot, trust-root custody, replay integrity, anti-replay binding, and
  authority-state equivalence
- Multi-tenant operation:
  tenant visibility boundaries, delegated-admin ceilings, shared-fleet
  isolation, and governed cross-tenant operator scope
- Capacity and economics:
  quotas, reservations, failure-domain-aware placement, GPU/capacity law, and
  budget-sensitive admission
- Provisioning:
  theorem-native admission over cloud and datacenter bring-up rather than
  provider console procedure as the authority path
- Operator authority:
  enterprise operator identity, bounded break-glass, and explicit operator
  discovery surfaces
- Runtime and automation:
  published runtime discovery, conformance runners, rehearsal paths, and
  downstream automation surfaces over canonical objects
- Compatibility:
  explicit third-party admission and platform-boundary rules instead of
  trial-and-error integration

From a downstream operator perspective, the usable feature set is:

- a release feed you can automate against
- a manifest that tells you what changed and how to negotiate it
- integrity evidence that tells you whether the release should be admitted
- recovery evidence that tells you whether a damaged machine came back lawfully
- operator guides that explain runtime, storage, networking, local-admin,
  automation, compatibility, and rehearsal surfaces as one contract family
- conformance and demo artifacts that prove the contract is not just prose

## What Moves The Needle

The important change is not that `4.0` adds more features. The change is that
several previously hand-waved areas become part of the shipped contract.

This release materially improves downstream readiness because it moves:

- release trust from maintainer narrative into manifest, integrity, and
  revocation artifacts
- recovery from "machine came back" into "lawful authority state was restored"
- tenancy from policy language into explicit operator and automation boundaries
- provisioning from substrate-specific ritual into governed admission and
  reconciliation
- automation from wrapper architecture into declared product surfaces
- supportability from internal knowledge into public operator guides and
  rehearsal evidence

What actually moves the needle for downstream teams is this:

- you can gate promotion on explicit release evidence instead of trust in the
  release team
- you can treat recovery as governed state restoration rather than disaster
  folklore
- you can reason about multi-tenant operation from published boundaries rather
  than policy slideware
- you can build automation on top of declared surfaces instead of wrapper code
- you can explain why a release, machine, or operator action is lawful from the
  shipped artifacts

If those surfaces hold under the final verification audit, downstream teams can
evaluate `4.0` as a real platform contract rather than a promising internal
system.

## Key Decisions Made

Several product and release decisions are already locked in this candidate:

- `4.0` is positioned as an enterprise-grade governed platform-operator release,
  not as a generic Linux distribution
- the downstream revision packet selected the extension path rather than the
  smaller "declare now" path
- release integrity is treated as shipped product evidence, not release-team
  paperwork
- mirror freshness, minimum-version floors, and revocation are fail-closed
  parts of the public contract
- recovery is judged by authority-state equivalence, not liveness
- provisioning adapters are execution helpers, not shadow control planes
- shell procedure is not treated as the supported product contract
- the external claim waits for public downstream surfaces, not internal progress
  alone

The most important decisions were:

- we chose "enterprise governed platform operator" over "generic distro"
- we chose the extension fork over the smaller declare-now fork
- we chose fail-closed release trust over convenience publication
- we chose authority-state recovery over reboot success as the recovery bar
- we chose public downstream contract clarity over internal-only milestone
  satisfaction
- we chose to force product truth into manifests, guides, conformance, and
  evidence instead of letting the release ride on narrative

Those decisions make the release narrower, but much more honest.

## What We Deferred

This candidate is stronger because some things were explicitly not smuggled into
the release as vague promises.

We are not claiming final release closure yet on anything that still depends on
the last staged verification audit.

We also continue to defer or bound:

- any claim that depends on undocumented shell knowledge or private operator
  runbooks
- any attempt to position `4.0` as a broad general-purpose distro rather than a
  governed enterprise machine platform
- any feature story that is only proven in local proof layers but not yet
  closed through staged compiled-candidate verification
- any downstream assumption that "latest" is a stable deployment contract
- any release promotion that bypasses manifest, integrity, freshness, or
  revocation checks

What we deliberately did not do:

- we did not pretend the final verification audit is already complete
- we did not broaden the claim into a general-purpose OS story
- we did not treat undocumented shell paths as supported operator product
  surfaces
- we did not call proof-layer closure the same thing as staged release truth
- we did not let downstreams assume that convenience beats explicit release
  controls

In practical terms, downstream teams should treat the current candidate as ready
for evaluation, integration planning, and contract review, but not as finally
declared release truth until the concluding verification audit closes.

## Current Release Position

As of 2026-03-22, the staged candidate already has:

- a passed staged conformance report
- a generated staged demonstration path
- resolved downstream revision disposition
- completed initial staged audit
- completed remediation package for that audit

The remaining release gate is the final staged verification audit closeout.

Until that verification audit lands, `4.0` should be treated as a staged
candidate rather than a declared final release.

## What Downstream Teams Can Do Now

Prepare against the contract that is already explicit:

- consume `releases/feed.json` rather than scraping release pages
- plan automation around `releases/<tag>/manifest.json` and
  `releases/<tag>/integrity.json`
- review operator guides for runtime, storage/networking/local-admin,
  observability, automation, compatibility, and conformance
- wire staging promotion to mirror freshness, minimum-version, and revocation
  checks
- treat the release declaration and evidence bundle as required production
  inputs

## Bottom Line

The current `4.0` candidate is ready for downstream preparation because the
shape of the contract is now explicit.

The final question is no longer "what is `4.0` supposed to be?" The remaining
question is whether the last staged verification audit clears the candidate to
make that contract final.
