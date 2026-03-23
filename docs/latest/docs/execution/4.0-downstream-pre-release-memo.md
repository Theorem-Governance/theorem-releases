# Theorem 4.0 Downstream Pre-Release Memo

Date: 2026-03-22

Audience: downstream platform operators, integration owners, and release managers

Status: public pre-release memo for the staged candidate; final verification audit closeout pending

Candidate tag: `v4.0.0`

## Purpose

This memo is the operator-facing pre-release position for the current `4.0`
candidate.

It is intended to be published before final declaration so downstream teams can
prepare against the explicit contract in advance.

It is meant to answer one practical question clearly:

What should a downstream team prepare for now, and what should they refuse to
assume until the final release gate closes?

## Current Release Position

As of 2026-03-22, the `4.0` candidate is positioned as an enterprise-grade
governed platform-operator release for multi-tenant fleets and governed
automation.

The candidate already has:

- a passed staged conformance report
- a generated staged demonstration path
- resolved downstream revision disposition
- completed initial staged audit
- completed remediation package for that audit

The remaining release gate is the final staged verification audit closeout.

Until that verification audit lands, `4.0` should be treated as a staged
candidate rather than a declared final release.

## What Downstream Teams Can Prepare Now

Prepare against the public contract that is already explicit in the candidate:

- release discovery through `releases/feed.json`
- release manifest consumption through `releases/<tag>/manifest.json`
- release integrity enforcement through `releases/<tag>/integrity.json`
- runtime discovery through `GET /`, `GET /openapi.json`, `GET /capabilities`,
  `GET /capability-negotiation`, and `GET /auth/discovery`
- conformance and rehearsal consumption through the published conformance and
  demo-path artifacts

Prepare operationally for these `4.0` contract surfaces:

- measured-boot recovery and authority-state equivalence
- release integrity, minimum-version floors, revocation, and mirror freshness
- tenant and delegated-admin boundary enforcement
- governed fleet economics, reservation, and capacity admission
- governed provisioning and orphan reconciliation
- enterprise operator identity and bounded break-glass
- downstream automation, migration, and rehearsal surfaces
- compatibility and third-party admission rules

## What Downstream Teams Should Not Assume Yet

Do not treat the candidate as finally declared until the verification audit
closes.

Specifically:

- do not treat staged-candidate evidence as final declaration evidence
- do not rely on any behavior not described in the published release contract,
  manifest, integrity evidence, or operator guides
- do not automate against the GitHub "latest" pointer
- do not bypass mirror freshness, minimum-version, or revocation checks
- do not assume undocumented shell procedure is part of the supported product
  contract

## Recommended Downstream Readiness Actions

Before promoting `4.0` into downstream staging or production workflows:

1. Consume explicit tags such as `v4.0.0`, not floating labels.
2. Gate promotion on `releases/feed.json` freshness and fail closed if stale.
3. Verify `releases/<tag>/manifest.json` and `releases/<tag>/integrity.json`
   before admitting the release.
4. Review the operator guides for Sessions `12-27`, especially runtime,
   storage/networking/local-admin, observability, automation, compatibility,
   and conformance.
5. Treat the release declaration route and release evidence bundle as required
   production inputs, not optional paperwork.

## What The Public Repo Ships

The public release repo is intended to ship:

- `README.md` and `quickstart.md` from the current quickstart source
- the release contract and machine-readable schema files
- the `4.0` runbook, steering sheet, and session approval checklist
- the operator guide set for the current `4.0` contract line
- the release manifest, integrity evidence, conformance report, and demo path
- this downstream pre-release memo

## Bottom Line

The current `4.0` candidate is ready for downstream preparation and contract
review.

It is not ready for a final declaration until the concluding verification audit
publishes its closeout artifacts and clears the staged candidate.
