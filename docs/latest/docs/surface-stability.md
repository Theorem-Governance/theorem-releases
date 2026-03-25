# Public Contract Surface Stability Declaration

**v0.5.0** — which surfaces downstream operators can depend on today

## Stability Tiers

### Stable (0.5.x)

Published in manifest. Will not break within the 0.5.x line. Capability IDs are stable once released.

### Operator/Admin

Available for operator use. May change between minor versions. Not for downstream integration.

### Internal/Adapter

Plumbing between theorem components. No stability guarantees.

## Stable Public Authority Routes

| Route | Method | Purpose |
|-------|--------|---------|
| `/declare-intent` | POST | Declare intent before acting |
| `/submit-act` | POST | Submit a governed act |
| `/establish-entity` | POST | Register a new entity |
| `/create-scope` | POST | Create a governance scope |
| `/amend-parameter` | POST | Amend kernel parameters |
| `/revoke-authorization` | POST | Revoke an authorization |
| `/submit-proof` | POST | Submit a proof obligation |
| `/authorizations` | GET | Query authorizations by entity |
| `/scope-contains` | GET | Check scope membership |
| `/governance-events` | GET | Query governance event stream |
| `/admit-artifact` | POST | Admit artifact hash to registry |
| `/revoke-artifact` | POST | Revoke artifact admission |
| `/verify-release` | POST | Verify release candidate |
| `/issue-execution-capability` | POST | Issue execution capability token |
| `/validate-execution-capability` | POST | Validate capability token |
| `/revoke-execution-capability` | POST | Revoke capability token |
| `/redeem-execution-capability` | POST | Redeem capability token |
| `/record-trust` | POST | Record trust relationship |
| `/revoke-trust` | POST | Revoke trust |
| `/record-fork` | POST | Record governance fork |

## Stable Discovery Routes

| Route | Method | Purpose |
|-------|--------|---------|
| `/` | GET | Runtime discovery root |
| `/openapi.json` | GET | OpenAPI spec (JSON) |
| `/openapi.yaml` | GET | OpenAPI spec (YAML) |
| `/capabilities` | GET | Runtime capability list |
| `/capability-negotiation` | GET | Capability negotiation surface |
| `/auth/discovery` | GET | Auth contract discovery |
| `/health` | GET | Liveness probe |
| `/ready` | GET | Readiness probe |
| `/status` | GET | Full status and state inspection |

## Stable Release Feed

| Surface | Path | Purpose |
|---------|------|---------|
| Release feed | `releases/feed.json` | Machine-consumable release discovery |
| Release manifest | `releases/{tag}/manifest.json` | Per-release capability + compatibility |
| Integrity evidence | `releases/{tag}/integrity.json` | SBOM, provenance, revocation |
| Conformance report | `releases/{tag}/conformance.json` | Product conformance proof |

## Stable CLI Commands

| Command | Purpose |
|---------|---------|
| `theorem-node serve` | Start HTTP API |
| `theorem-node bootstrap` | Bootstrap governance kernel |
| `theorem-node status` | Show governance state |
| `theorem-node audit` | Run governance audit |
| `theorem-node verify` | Verify governance integrity |
| `theorem-node export-state` | Export full state JSON |
| `theorem-node establish-entity` | Register entity |
| `theorem-node submit-act` | Submit governed act |
| `theorem-node admit-artifact` | Admit artifact |
| `theorem-node verify-release` | Verify release candidate |
| `theorem-init run` | Run as PID 1 |
| `theorem-init check` | Validate configuration |
| `theorem-init default-config` | Print default config |

## Stable Error Taxonomy

All 0.5.x routes return errors as JSON with `error_code` + `message` fields.

### HTTP 400

- `not_bootstrapped`
- `entity_not_found`
- `scope_not_found`
- `request_hash_failed`
- `serialization_failed`

### HTTP 409

- `idempotency_conflict`

### HTTP 422

- `action_admission_inactive`
- `governance_authority_required`
- `principal_not_trusted`

## Operator/Admin Routes (NOT stable public contract)

These routes are available for operator use but may change between minor versions. Do not build downstream integrations against them.

- `GET /export-state` — export full governance state
- `GET /rebuild-authority-state` — rebuild authority state from event log
- `POST /restore-authority-state` — restore authority state from backup
- `GET /governance-context` — read model
- `GET /trust-state` — trust query
- `GET /runtime-eligibility/artifact` — runtime control
- Policy routes, organization routes, authority bundle routes, delegation routes

## Internal/Adapter Routes (NO stability guarantees)

These are plumbing between theorem components. They may change or disappear without notice.

- `POST /witness/publish` — publish witness entry
- `GET /witness/entries` — query witness entries
- `GET /witness/digest` — witness digest
- `POST /commit-intent-hash` — commit intent hash
- `POST /refresh-commitment-token` — refresh commitment token
- `POST /register-measurement-method` — register measurement method

## Not Yet Published

- **SDK / client library**: not available.
- **Migration contract CLI**: manual process documented in operator guide.
- **Support bundle / observability export**: structured JSON logging exists but no bundle tool.
- **Programmatic first-boot API**: use `first-boot.conf` file.

## Automation Guidance

- Consume `releases/feed.json`, not GitHub "latest".
- Verify checksums against the integrity evidence.
- Fail closed if the feed is stale.
- Gate on `capability_negotiation`, not prose or semver.
