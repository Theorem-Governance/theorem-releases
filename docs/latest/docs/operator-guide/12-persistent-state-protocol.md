# Persistent State Protocol

## Purpose

The `0.6.x` persistent-state surfaces provide a governed way to record,
supersede, revoke, and subscribe to runtime facts that must survive across
sessions and restarts.

This protocol is the first durable memory substrate for the `0.7` and `0.8`
lines. It is not entity-native intelligence yet, but it is the substrate that
later entity memory can build on.

## HTTP Surfaces

All persistent-state routes are protected by the shared witness bearer token.

- `GET /state`
- `POST /state/observe`
- `POST /state/revoke`
- `POST /state/subscribe`
- `GET /state/subscriptions`
- `POST /state/session-cleanup`

## Observation Record

Each observation stores:

- `domain`
- `key`
- `value`
- `observed_by_entity_id`
- `scope_id`
- `authorization_id`
- `act_id`
- `attestation_id`
- `observed_at`
- `recorded_at`
- `confidence`
- optional `ttl_secs`
- optional `evidence`
- `observation_class`
- revocation fields when revoked

The protocol is append-oriented.
Recording a new observation does not overwrite prior records in place.

## Current Value Semantics

For a given `(domain, key)` pair:

- the current observation is the newest non-revoked, non-stale record by
  `observed_at`
- an older record becomes `stale` if it is superseded by a newer observation
  for the same `(domain, key)`
- a record also becomes `stale` if its TTL has expired
- a revoked record is never current

`GET /state?domain=<domain>&key=<key>` returns:

- `current`
- `conflicts`

`conflicts` are non-revoked records for the same `(domain, key)` that disagree
with the current value and are not selected as current.
They remain visible even when they are stale, because stale disagreement is
still part of the governed state history for that key.

## Domain And Pattern Queries

`GET /state` supports exactly one of:

- `domain + key`
- `domain`
- `pattern`

Semantics:

- `domain + key` returns the full snapshot for that exact pair
- `domain` returns current records for all keys in that domain
- `pattern` returns current records for domains matching the wildcard pattern

Pattern queries are current-state views, not history dumps.

## Revocation Semantics

`POST /state/revoke` revokes a specific observation by:

- `domain`
- `key`
- `attestation_id`

The revocation itself must carry:

- `revocation_attestation_id`
- `reason`

Revocation is explicit and durable.
If the revoked observation had been current, the protocol falls back to the next
newest non-revoked, non-stale record for that `(domain, key)`.

## Subscription Semantics

Subscriptions bind:

- `agent_id`
- `entity_id`
- `scope_id`
- `authorization_id`
- `domain_pattern`
- `mode`
- `persistent`

Two classes exist:

- persistent subscriptions
- session-scoped subscriptions

`POST /state/session-cleanup` deactivates only session-scoped subscriptions for
the given `agent_id`.
Persistent subscriptions remain active.

## Operational Rules

- Use `attestation_id` as the canonical handle for later revocation.
- Use `observed_at` for chronology, not `recorded_at`.
- Use TTL only for facts that genuinely expire on their own.
- Use revocation when the fact was wrong, unsafe, or no longer admissible.
- Do not treat pattern queries as an audit trail. They are current-state views.

## What This Protocol Is Not

This protocol is not yet:

- entity belief memory
- causal-model persistence
- strategy memory
- adapter history

Those belong to `0.7` and `0.8`.
This layer is the substrate memory contract they will depend on.
