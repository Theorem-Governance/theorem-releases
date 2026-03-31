# Bridge Agent And Claude Bootstrap

This is the operator path for turning persistent state into a live working surface.

## What Exists

- `theorem-node` serves `/agent-bootstrap` and `/state*`.
- Claude sessions auto-load bootstrap context through [`.claude/settings.json`](/C:/Users/Eric/Theorem/.claude/settings.json).
- `theorem-govagent` can run a config-driven bridge worker that observes external systems and publishes governed corrections into `integration.*`.

## Required Environment

Set:

- `THEOREM_WITNESS_TOKEN`
- `THEOREM_CLAUDE_ENTITY_ID`

Optional:

- `THEOREM_NODE_URL`
- `THEOREM_CLAUDE_STATE_DIR`
- `THEOREM_CLAUDE_STATE_POLL_SECS`
- `THEOREM_BRIDGE_ENTITY_ID`
- `THEOREM_BRIDGE_AGENT_ID`

If `THEOREM_NODE_URL` is unset, Claude hooks and bridge bootstrap default to `http://127.0.0.1:3170`.
If `THEOREM_BRIDGE_AGENT_ID` is unset, the bridge generates a fresh agent session ID per run. Set it when you want a stable operator-visible bridge identity and a pre-admitted `bus.session.register` payload.

## Bridge Admin Flow

Initialize a bridge config:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-admin init --config .bridge/bridge.json
```

Add an HTTP status observer:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-admin add-http-status `
  --config .bridge/bridge.json `
  --id ring3-api-health `
  --source-system ring3-api `
  --url https://example.com/health `
  --domain integration.ring3-api `
  --key health_ok `
  --expected-status 200
```

Add an HTTP JSON observer:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-admin add-http-json `
  --config .bridge/bridge.json `
  --id release-feed-version `
  --source-system release-feed `
  --url https://example.com/releases/latest.json `
  --domain integration.release-feed `
  --key latest_version `
  --json-pointer /version `
  --value-kind string
```

Inspect the current config and runtime status:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-admin list --config .bridge/bridge.json
```

Pause or resume an observer:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-admin set-enabled `
  --config .bridge/bridge.json `
  --observer-id ring3-api-health `
  --enabled false
```

Explain one observer:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-admin explain `
  --config .bridge/bridge.json `
  --observer-id ring3-api-health
```

## Running The Bridge

Run one correction cycle:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-agent --config .bridge/bridge.json --once
```

Run continuously:

```powershell
cargo run -p theorem-govagent --bin theorem-govagent -- bridge-agent --config .bridge/bridge.json
```

The bridge does three things:

1. fetches the `bridge_agent` bootstrap bundle from `theorem-node`
2. subscribes to the recommended theorem state patterns
3. observes external sources and, when reality changed, revokes the stale attestation and publishes a fresh `Observe`

Runtime status is written beside the config as `<config filename>.status.json`.

## Claude Session Behavior

On session start, Claude receives the current bootstrap bundle plus recommended subscriptions.

During the session:

- a local poller refreshes bootstrap state
- state diffs are appended to the local session feed
- the `UserPromptSubmit` hook injects unread changes into Claude context

On session end, the session cleanup hook stops the poller.

## Expected Operator Outcome

When an external system changes:

1. the bridge notices
2. theorem revokes the stale bridged observation
3. theorem records the corrected observation
4. Claude sees the update on the next prompt without human narration
