# Runtime Integration Guide

How to go from "binaries on disk" to "governance-mediated operations" on a
TheoremOS FreeBSD host.

## Prerequisites

- TheoremOS 0.6.0 installed (via `install.sh` or upgrade bundle)
- All binaries present: `/sbin/theorem-init`, `/usr/local/bin/theorem-node`,
  `/usr/local/bin/theorem-gateway`, `/usr/local/bin/theorem-govagent`
- FreeBSD rc.d scripts installed at `/usr/local/etc/rc.d/theorem_{init,node,gateway,govagent}`

## Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────┐
│  Application    │     │  theorem-gateway  │     │  theorem-node │
│  (workspace-web)│────▶│  :3180            │────▶│  :3170        │
│                 │Ring3│  Ring 3 sidecar    │ Bus │  Governance   │
│                 │ API │                    │     │  kernel       │
└─────────────────┘     └──────────────────┘     └───────────────┘
                                                        ▲
                                                        │ Events
                                                  ┌─────┴─────────┐
                                                  │theorem-govagent│
                                                  │  AI work agent │
                                                  └────────────────┘
```

**Your application talks to theorem-gateway, not theorem-node directly.**
The gateway provides the Ring 3 two-call API designed for external workloads.
theorem-node is the governance kernel — it enforces rules, stores state, and
runs audits. The gateway translates external HTTP requests into governance
ceremonies on the internal bus.

## Step 1: Bootstrap theorem-node

Initialize the governance database. This creates the genesis state, root scope,
and bootstrap entity.

```sh
# Pick a spec hash that identifies your governance specification version.
# This is written into GENESIS and cannot be amended.
SPEC_HASH=$(sha256 -q /path/to/your/governance-spec.md)

theorem-node --db /var/lib/theorem/theorem.db bootstrap \
  --spec-hash "$SPEC_HASH"
```

Output on success:

```
Bootstrap complete.
  Entities: 1  (bootstrap)
  Scopes: 1    (root)
  Genesis records: N
```

The bootstrap entity and root scope IDs are printed. **Save them** — you need
them for the next steps.

To verify:

```sh
theorem-node --db /var/lib/theorem/theorem.db status
```

## Step 2: Create the application entity

Every workload that submits governance acts needs an entity. Create one for
your application:

```sh
theorem-node --db /var/lib/theorem/theorem.db establish-entity \
  --name "workspace-web" \
  --type executing \
  --scope <ROOT_SCOPE_ID> \
  --authorized-by <BOOTSTRAP_ENTITY_ID>
```

Participation types:
- `authorizing` — can authorize other entities and amend parameters
- `executing` — can submit governance acts (this is what applications use)
- `participating` — can submit proofs and trust records
- `observing` — read-only access to governance state

The command prints the new entity ID. **Save it.**

## Step 3: Configure secrets

### theorem-node

Create `/etc/theorem/theorem-node.env`:

```sh
# Required for server operation
THEOREM_COMMITMENT_SECRET=<generate-a-random-secret>
THEOREM_WITNESS_TOKEN=<generate-a-random-token>

# Optional
THEOREM_ENFORCEMENT_MODE=advisory    # or "enforce" for production
# THEOREM_PEER_WITNESSES=https://witness1.example.com,https://witness2.example.com
```

Generate secrets: `openssl rand -hex 32`

### theorem-gateway

Create `/etc/theorem/theorem-gateway.env`:

```sh
# Required — from Step 2
THEOREM_GATEWAY_ENTITY_ID=<entity-id-from-step-2>
THEOREM_GATEWAY_SCOPE_ID=<root-scope-id-from-step-1>

# Auth tokens — comma-separated. Your application sends these as Bearer tokens.
THEOREM_GATEWAY_AUTH_TOKENS=<generate-a-token-for-workspace-web>

# Optional security settings (defaults shown)
# THEOREM_GATEWAY_RATE_LIMIT=10
# THEOREM_GATEWAY_REQUIRE_TLS=false
# THEOREM_GATEWAY_ALLOW_ANONYMOUS=false
```

### theorem-govagent

Create `/etc/theorem/theorem-govagent.env`:

```sh
# Required
GOVAGENT_ENTITY_ID=<entity-id-for-govagent>
GOVAGENT_MODEL_URL=http://localhost:11434        # Ollama
GOVAGENT_MODEL_NAME=qwen2.5:32b
GOVAGENT_TASK_PORT=3181
GOVAGENT_CORPUS_PATH=/var/lib/theorem/corpus.sqlite
GOVAGENT_GATEWAY_URL=http://localhost:3180
GOVAGENT_GATEWAY_TOKEN=<same-token-as-THEOREM_GATEWAY_AUTH_TOKENS>
GOVAGENT_SEED_PATH=/etc/theorem/seeds.json
GOVAGENT_WITNESS_TOKEN=<same-as-THEOREM_WITNESS_TOKEN>
GOVAGENT_TASK_TOKEN=<generate-a-token>
GOVAGENT_KERNEL_URL=http://localhost:3170
```

Run `theorem-govagent --help` for the full configuration reference including
all optional variables and the State Gardener subsystem.

Set permissions:

```sh
chmod 0600 /etc/theorem/theorem-node.env
chmod 0600 /etc/theorem/theorem-gateway.env
chmod 0600 /etc/theorem/theorem-govagent.env
```

## Step 4: Resolve port conflicts

theorem-node defaults to port 3170. If another process uses that port, set a
different port in `/etc/rc.conf`:

```sh
# Option A: Move theorem-node to a different port
theorem_node_port="3171"

# Option B: Set via env var in theorem-node.env
# THEOREM_NODE_PORT=3171

# Then update theorem-gateway to point to the new port:
# In theorem-gateway.env:
# THEOREM_KERNEL_URL=http://127.0.0.1:3171
```

Or move your application off port 3170 to let theorem-node use its default.

## Step 5: Start services

The rc.d dependency chain starts them in order:

```sh
# Start all at once (rcorder handles ordering)
service theorem_node start
service theorem_gateway start
service theorem_govagent start
```

Or enable them in `/etc/rc.conf` and reboot:

```sh
theorem_node_enable="YES"
theorem_gateway_enable="YES"
theorem_govagent_enable="YES"
```

The installer (`install.sh`) sets these automatically.

## Step 6: Verify health

### theorem-node

```sh
# Service status
service theorem_node status

# Health endpoint
fetch -q -o - http://127.0.0.1:3170/health
# {"status":"ok","healthy":true,"bootstrapped":true}

# Readiness (checks bootstrap + secrets configured)
fetch -q -o - http://127.0.0.1:3170/ready

# Full audit
theorem-node --db /var/lib/theorem/theorem.db audit

# 25-property verifier
theorem-node --db /var/lib/theorem/theorem.db verify
```

### theorem-gateway

```sh
service theorem_gateway status

# Health (no auth required)
fetch -q -o - http://127.0.0.1:3180/ring3/health
# {"status":"ok","in_flight_intents":0,"kernel_reachable":true}

# Metrics (requires auth)
fetch -q -o - -H "Authorization: Bearer <token>" http://127.0.0.1:3180/ring3/metrics
```

### theorem-govagent

```sh
service theorem_govagent status
# Check logs for startup confirmation:
tail /var/log/theorem/theorem-govagent.log
```

## Step 7: Integrate your application

### The Ring 3 Two-Call API

Your application submits governed operations through two HTTP calls to
theorem-gateway:

#### Call 1: Declare intent

```
POST /ring3/intent
Authorization: Bearer <your-auth-token>
Content-Type: application/json

{
  "action": "create-workspace",
  "scope": "<scope-id>"
}
```

Response (ruling: proceed):

```json
{
  "ruling": "proceed",
  "intent_id": "019..."
}
```

Response (ruling: rejected):

```json
{
  "ruling": "rejected",
  "violations": ["entity lacks authorization for action in scope"]
}
```

**If the ruling is "proceed", you receive an `intent_id`.** Execute your
operation, then report the outcome.

**If the ruling is "rejected", do not execute the operation.** The violations
array explains why.

#### Call 2: Report outcome

```
POST /ring3/outcome/<intent_id>
Authorization: Bearer <same-auth-token-as-intent>
Content-Type: application/json

{
  "result": "workspace created: ws-42",
  "drift": 0.01
}
```

Response:

```json
{
  "attestation_id": "att-...",
  "act_id": "act-..."
}
```

The `attestation_id` is the governance attestation proving this operation was
mediated by the enforcement substrate. The `act_id` is the governance act
record in the kernel.

**Important:** The outcome must be submitted with the same bearer token as the
intent. A different token returns 403 (auth fingerprint mismatch). Expired
intents (default TTL: 1 hour) return 410.

### Integration pattern

```python
import requests

GATEWAY = "http://127.0.0.1:3180"
TOKEN = "your-auth-token"
SCOPE = "root-scope-id"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

def governed_operation(action: str, execute_fn):
    """Submit a governed operation through the Theorem gateway."""

    # Step 1: Declare intent
    resp = requests.post(f"{GATEWAY}/ring3/intent", json={
        "action": action,
        "scope": SCOPE,
    }, headers=HEADERS)
    resp.raise_for_status()
    ruling = resp.json()

    if ruling["ruling"] != "proceed":
        raise PermissionError(f"Governance rejected: {ruling.get('violations', [])}")

    intent_id = ruling["intent_id"]

    # Step 2: Execute the operation
    result = execute_fn()

    # Step 3: Report outcome
    resp = requests.post(f"{GATEWAY}/ring3/outcome/{intent_id}", json={
        "result": str(result),
    }, headers=HEADERS)
    resp.raise_for_status()
    attestation = resp.json()

    return result, attestation["attestation_id"]
```

### What about your existing PostgreSQL tables?

theorem-node uses SQLite, not PostgreSQL. Your existing `mksc_*` tables are
application-level records. They do not replace or conflict with theorem-node's
governance state.

After integration, your application should:

1. Submit governed operations through the gateway (get attestation)
2. Store the `attestation_id` alongside your application record
3. Your `mksc_governance_acts` table becomes an application-level index that
   references kernel-attested governance acts

The governance truth lives in theorem-node's SQLite database. Your PostgreSQL
tables are your application's view of that truth.

## Smoke Test

After completing all steps, run this end-to-end test:

```sh
# 1. Verify theorem-node is bootstrapped and healthy
fetch -q -o - http://127.0.0.1:3170/health

# 2. Verify theorem-gateway can reach the kernel
fetch -q -o - http://127.0.0.1:3180/ring3/health

# 3. Submit a trivial governance act through the gateway
INTENT=$(fetch -q -o - \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  --body '{"action":"smoke-test","scope":"<root-scope-id>"}' \
  http://127.0.0.1:3180/ring3/intent)

echo "$INTENT"
# Should show: {"ruling":"proceed","intent_id":"..."}

INTENT_ID=$(echo "$INTENT" | python3 -c "import sys,json; print(json.load(sys.stdin)['intent_id'])")

# 4. Report outcome
OUTCOME=$(fetch -q -o - \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  --body "{\"result\":\"smoke-test-ok\"}" \
  "http://127.0.0.1:3180/ring3/outcome/${INTENT_ID}")

echo "$OUTCOME"
# Should show: {"attestation_id":"...","act_id":"..."}

# 5. Verify the act was recorded
theorem-node --db /var/lib/theorem/theorem.db status
# Entity count and act count should have increased

# 6. Run audit to confirm governance integrity
theorem-node --db /var/lib/theorem/theorem.db audit
```

## HTTP Status Reference

### POST /ring3/intent

| Status | Meaning |
|--------|---------|
| 200 | Ruling returned (check `ruling` field for proceed/rejected/timeout) |
| 400 | Empty action or scope |
| 401 | Missing or invalid bearer token |
| 403 | caller_id override attempted but not allowed |
| 429 | Rate limited (per-IP or per-entity) |
| 500 | Internal error (bus session or ceremony failure) |
| 503 | Intent table full or per-entity quota exceeded |

### POST /ring3/outcome/:intent_id

| Status | Meaning |
|--------|---------|
| 200 | Attestation returned |
| 400 | Empty result field |
| 401 | Missing or invalid bearer token |
| 403 | Bearer token doesn't match intent's auth fingerprint |
| 404 | intent_id not found |
| 410 | intent_id expired (older than TTL, default 1 hour) |
| 429 | Rate limited |
| 500 | Ceremony failure |

### GET /ring3/health

| Status | Meaning |
|--------|---------|
| 200 | Always — check `kernel_reachable` field |

No authentication required. Not rate-limited.

### GET /ring3/metrics

| Status | Meaning |
|--------|---------|
| 200 | Metrics returned |
| 401 | Missing or invalid bearer token |
| 429 | Rate limited |

Requires authentication.
