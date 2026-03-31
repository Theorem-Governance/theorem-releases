# Deployment

This document covers the production deployment path, release artifact
selection, bare-metal installation rules, the legacy Linux/NixOS deployment
path, hardware requirements, network configuration, bootstrap procedure,
database layout, and environment variables. For a fast developer install, use
`docs/quickstart.md` instead.

## Artifact Selection Rules

For `0.5.x` and later FreeBSD bare-metal releases, automation must resolve
artifacts from `releases/<tag>/manifest.json` and `releases/<tag>/integrity.json`.
Do not infer deployability from filenames, GitHub release pages, asset order,
or assume the installer script is flat at the tarball root.

Deployment modes must consume artifacts by machine-readable contract:

| Deploy mode | Allowed artifact kinds | Forbidden artifact kinds |
|-------------|------------------------|--------------------------|
| Unattended bare-metal direct disk write | `direct-write-disk-image` | `install-media`, `installer-rootfs`, `single-binary` |
| Unattended bare-metal install flow | `installer-rootfs` | `install-media` |
| In-place theoremOS upgrade | `upgrade-bundle` | `install-media`, `direct-write-disk-image` |
| Human-operated boot/install media | `install-media` | n/a |
| Service-only binary extraction | `single-binary` | `install-media` |

Additional hard rules:

- Any artifact with `direct_disk_write=forbidden` must be rejected for raw disk
  writes.
- For `installer-rootfs`, resolve the installer at
  `${archive_root}/${installer_entrypoint}` after extraction.
- For `upgrade-bundle`, resolve the upgrader at
  `${archive_root}/${upgrade_entrypoint}` after extraction and use it only on
  an already-running theoremOS machine that matches
  `supported_upgrade_from`.
- For `direct-write-disk-image`, the image is self-contained and must boot
  without any required seed injection. The FAT seed partition named by
  `seed_partition_label` remains available as an optional override channel for
  `/first-boot.conf` before first boot when a deployment needs host-specific
  SSH or network settings. After writing the image, repair GPT to the full
  target device and boot it. theoremOS expands the final ZFS partition and root
  pool on first boot via `theorem_autogrow`; consumers must not pre-grow the
  root partition offline before the first theoremOS boot. The manifest must
  declare `self_contained_bootstrap=true`,
  `bootstrap_strategy=bundled-release-metadata`, and
  `first_boot_config_policy=optional-override`. The expected template path on
  that partition is `first_boot_config_template`.
- `theoremos-*.iso` and `theoremos-*-memstick.img` are install media only.
- Current `0.5.x` unattended bare-metal contract is the
  `direct-write-disk-image` artifact. `installer-rootfs` remains valid only for
  installer-driven automation on a prepared FreeBSD host.
- Current `0.5.x -> 0.6.x` update contract is an in-place ZFS boot-environment
  upgrade via `upgrade-bundle`, not a fresh bare-metal disk write.

Minimal selection flow:

```bash
python3 - <<'PY'
import json
manifest = json.load(open("manifest.json"))

deploy_mode = "direct-disk-write"
allowed = {
    "bare-metal-install": {"installer-rootfs"},
    "direct-disk-write": {"direct-write-disk-image"},
}

candidates = [
    artifact for artifact in manifest["artifacts"]
    if artifact.get("artifact_kind") in allowed[deploy_mode]
]

if not candidates:
    raise SystemExit(f"FAIL: no artifact for deploy mode {deploy_mode}")

artifact = candidates[0]
if artifact.get("direct_disk_write") == "forbidden" and deploy_mode == "direct-disk-write":
    raise SystemExit(f"FAIL: {artifact['name']} forbids direct disk writes")

print(artifact["name"])
PY
```

The remainder of this page includes legacy Linux/NixOS service deployment
guidance for the 4.x line. For `0.5.x` FreeBSD machine replacement and restore,
use the manifest-driven selection rules above together with
`docs/operator-guide/08-migration-4x-to-5x.md`.

## Release-Only Install Layout

For production hosts, prefer a versioned install directory with a stable
symlink rather than unpacking over an in-place binary.

Recommended layout:

```text
/var/lib/theorem/releases/theorem-node-v3.0.0-x86_64-unknown-linux-gnu/
/var/lib/theorem/current -> /var/lib/theorem/releases/theorem-node-v3.0.0-x86_64-unknown-linux-gnu/
```

Recommended install sequence:

```bash
VERSION="v3.0.0"
TARGET="x86_64-unknown-linux-gnu"
BASE="/var/lib/theorem"
RELEASE_DIR="${BASE}/releases/theorem-node-${VERSION}-${TARGET}"

install -d "${BASE}/releases"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/releases/download/${VERSION}/theorem-node-${VERSION}-${TARGET}.tar.gz"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/releases/download/${VERSION}/theorem-node-${VERSION}-${TARGET}.tar.gz.sha256"
shasum -a 256 -c "theorem-node-${VERSION}-${TARGET}.tar.gz.sha256"
mkdir -p "${RELEASE_DIR}"
tar xzf "theorem-node-${VERSION}-${TARGET}.tar.gz" -C "${BASE}/releases"
ln -sfn "${RELEASE_DIR}" "${BASE}/current"
```

This pattern supports safe upgrades and rollbacks. The old release directory
remains present, and switching the `current` symlink is atomic.

## NixOS Module

The Theorem governance node runs as a systemd service managed by a NixOS module at `nix/module.nix`. The service unit is `theorem-node.service`.

### Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `services.theorem.enable` | bool | false | Enable the governance node service |
| `services.theorem.port` | port | 3170 | TCP port for the HTTP API |
| `services.theorem.dataDir` | path | `/var/lib/theorem` | Directory for SQLite databases and state |
| `services.theorem.specHash` | string | (required) | SHA-256 hash of `theorem-spec.md` -- written into GENESIS, cannot be amended |
| `services.theorem.witnessEndpoints` | list of string | [] | Peer witness node URLs for attestation cross-verification |
| `services.theorem.logLevel` | enum | "info" | Log verbosity: error, warn, info, debug, trace |
| `services.theorem.artifactUrl` | string | (required) | URL for public verification artifact (PROP-INFRA-001) |
| `services.theorem.logEndpoint` | string | (required) | URL for attestation log endpoint (PROP-INFRA-002) |
| `services.theorem.package` | package | flake default | The theorem-node binary package |

### Deployment Assertions

The NixOS module includes assertions that prevent deployment with incomplete configuration:

- `specHash` must not be the placeholder value
- `witnessEndpoints` must not be empty (P13 requires >= 2 independent verification locations)
- `artifactUrl` must not be the default `theorem://bootstrap-local/artifact`
- `logEndpoint` must not be the default `theorem://bootstrap-local/log`

If any assertion fails, `nixos-rebuild` will refuse to apply the configuration with an explanatory error message.

### Systemd Service Configuration

The service runs with:

```
ExecStart = theorem-node --db /var/lib/theorem/theorem.db serve --port 3170
```

If you are not using the NixOS module and are deploying a release tarball
directly, use an explicit versioned path or the stable symlink:

```ini
[Unit]
Description=Theorem governance node
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/var/lib/theorem
Environment=THEOREM_COMMITMENT_SECRET=replace-with-real-secret
ExecStart=/var/lib/theorem/current/theorem-node --db /var/lib/theorem/theorem.db serve --port 3170
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Security hardening settings (verifiable via `systemctl show theorem-node`):

- **DynamicUser = true** -- transient user, no persistent account to compromise
- **StateDirectory = "theorem"** -- creates `/var/lib/theorem` with 0700 permissions
- **ProtectSystem = strict** -- read-only filesystem except state directory
- **ProtectHome = true** -- no access to /home, /root, /run/user
- **PrivateTmp = true** -- isolated /tmp
- **PrivateDevices = true** -- no /dev access
- **NoNewPrivileges = true** -- cannot escalate via setuid/setgid
- **MemoryDenyWriteExecute = true** -- W^X enforcement, no JIT
- **CapabilityBoundingSet = ""** -- all capabilities dropped
- **SystemCallFilter** -- restricted to @system-service minus privileged/resources/mount/clock/debug/module/raw-io/reboot/swap
- **RestrictAddressFamilies** -- AF_INET, AF_INET6, AF_UNIX only
- **LimitNOFILE = 4096, LimitNPROC = 64** -- resource caps
- **Restart = on-failure** with 5s initial delay, max 5min backoff, burst limit 5 in 10min

Logging goes to the systemd journal (`StandardOutput = journal`, `StandardError = journal`, `SyslogIdentifier = theorem-node`).

## Hardware Requirements (DEC-007)

The spec requires:

- **Boot chain:** UEFI Secure Boot with self-managed keys -> NixOS kernel signature -> Rust engine binary hash (full measured boot for PROP-SUBSTRATE-001)
- **Storage:** Local SSD for attestation log and governance databases (not network-attached storage)
- **Clock:** NTP authenticated via Chrony with NTS (Network Time Security, RFC 8915) -- see `nix/hardening.nix` for configuration
- **Network:** TCP connectivity to witness peer endpoints

The `nix/hardening.nix` module implements:

- Measured boot with automatic re-signing of Secure Boot binaries after NixOS rebuilds
- Kernel hardening (vsyscall=none, slab_nomerge, init_on_alloc/free, spectre mitigations, lockdown=confidentiality)
- Kernel module loading locked after boot
- Authenticated NTP with NTS from multiple independent providers (Cloudflare, Netnod, PTB)
- Firewall with only SSH (22) and theorem-node port open
- Linux audit subsystem monitoring `/var/lib/theorem/`, NixOS config, chrony config, and system clock changes
- SSH hardened (key-only auth, curve25519 key exchange, chacha20-poly1305/aes256-gcm ciphers)

## Network Configuration

### Authority Endpoint Classes

Production exposure should follow the authority contract:

- Stable public authority routes may be published through the reverse proxy for
  the clients that actually need them.
- Operator/admin routes such as `GET /export-state`,
  `GET /rebuild-authority-state`, and `POST /restore-authority-state` should be
  restricted to operator-controlled callers.
- Adapter-internal routes (`/commit-intent-hash`, `/refresh-commitment-token`,
  `/register-measurement-method`, and the witness commitment helpers) are not
  public integration surfaces and should stay behind the same operator network
  boundary as the Ring 3 stack.

### Port Binding

The HTTP server binds to `127.0.0.1:<port>` by default (port 3170). This means it only accepts connections from localhost. For remote access, place a TLS-terminating reverse proxy (nginx, caddy, etc.) in front.

The NixOS module opens the configured port in the firewall for peer witness communication:

```nix
networking.firewall.allowedTCPPorts = [ cfg.port ];
```

### TLS Termination

The theorem-node binary does not terminate TLS itself. For production deployments, configure a reverse proxy:

1. TLS termination at the reverse proxy
2. Proxy passes to `127.0.0.1:3170`
3. Witness endpoints in peer configuration use the external HTTPS URL

### Witness Network

P13 requires at least two independent locations for public verification artifacts. The local instance is one location; configure at least one peer witness endpoint. "Independent" means different DNS registrars, different hosting providers, or no shared administrative credentials.

```nix
services.theorem.witnessEndpoints = [
  "https://witness-1.example.com"
  "https://nas.local:3171"
];
```

Witness endpoints are passed to the engine as `THEOREM_WITNESS_ENDPOINTS` (comma-separated list).

## Bootstrap Behavior

Current theoremOS release artifacts are self-bootstrapping by default. A fresh
direct-write deployment or installer-rootfs install carries
`bootstrap-config.json`, and first boot applies that bundled release metadata
automatically through `theorem_runtime_prepare` and `theorem-auto-bootstrap`.
Operators should not run a separate manual bootstrap ceremony during the normal
deployment path.

The automatic bootstrap path still creates the same genesis state:

- 2 genesis records (BootstrapEntity + RootScope)
- 2 entities (system entity + bootstrap entity)
- 1 root scope
- 2 authorizations (system + bootstrap, each covering execution_act and governance_act)
- 25 proof obligations (one per required property, 15 genesis-time properties validated by E9)
- Kernel parameters (audit_interval_ms, minimum_intent_execution_interval_ms, etc.)
- Measurement methods, resource measurement methods, integrity mechanism, attestation log, attestation schema

Use the explicit CLI bootstrap path only for recovery, custom infrastructure,
or nonstandard installs where bundled release metadata is intentionally not the
bootstrap source. In those cases, verify `theorem-spec.sha256` against
`theorem-spec.md` first and then run `theorem-node ... bootstrap` yourself.

### Post-Boot Verification

```bash
theorem-node --db /var/lib/theorem/theorem.db status
theorem-node --db /var/lib/theorem/theorem.db audit
theorem-node --db /var/lib/theorem/theorem.db verify
```

Expected output includes `"bootstrapped": true` with entity_count >= 2,
scope_count >= 1, and proof-obligation counts. On a normal theoremOS
deployment, the server is already started by the supervised first-boot path,
so manual `serve` invocation is not part of the standard install flow.

### Canonical Liveness and Readiness Checks

Use these checks in production:

```bash
curl -fsS http://127.0.0.1:3170/health
theorem-node --db /var/lib/theorem/theorem.db status
```

- `/health` is the canonical liveness probe and should be used by service
  managers and external monitors that only need process health.
- `status` or `GET /status` is the canonical readiness and state-inspection
  path.

## Upgrade and Rollback Guidance

Upgrade:

1. Download the new tagged release into a fresh versioned directory.
2. Verify the checksum before unpacking.
3. Update `/var/lib/theorem/current` to point at the new release directory.
4. Restart the service.
5. Confirm `GET /health` returns `200` and `GET /status` reports the expected
   bootstrapped state.

Rollback:

1. Repoint `/var/lib/theorem/current` to the previous release directory.
2. Restart the service.
3. Re-run the same health and status checks.

Do not automate against the GitHub "latest" pointer for production upgrades.
Pin the exact release tag you intend to install.

## Database Files

Theorem uses two separate SQLite database files:

| File | Purpose | Location |
|------|---------|----------|
| `theorem.db` | Governance state (acts, entities, authorizations, attributions, attestations, etc.) | `--db` CLI argument or `THEOREM_DATA_DIR/theorem.db` |
| `theorem-witness.db` | Witness log (append-only attestation entries with hash chain) | Derived from governance DB path: `theorem-witness.db` sibling |

The witness DB path is computed by the `serve` command: it replaces `.db` with `-witness.db` in the governance DB path. For `theorem.db`, the witness DB is `theorem-witness.db` in the same directory.

Both files should be on the same local SSD (DEC-007). They must not be on network-attached storage. The systemd service's `StateDirectory = "theorem"` with `StateDirectoryMode = 0700` ensures correct ownership and permissions.

### File Permissions

- The governance and witness databases should be readable/writable only by the theorem-node process user
- The `DynamicUser = true` systemd setting creates a transient user with access only to the state directory
- Backup processes need read access to both files (see [Backup and Recovery](05-backup-and-recovery.md))

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RUST_LOG` | `info` | Controls log verbosity. Format: `tracing_subscriber::EnvFilter` syntax. Examples: `info`, `theorem_node=debug`, `theorem_kernel::enforcement=trace` |
| `THEOREM_DATA_DIR` | (set by NixOS module) | Data directory path, used by systemd environment |
| `THEOREM_LOG_LEVEL` | `info` | Log level set by NixOS module. Used as fallback when `RUST_LOG` is not set. Accepts same syntax as `RUST_LOG`. |
| `THEOREM_WITNESS_ENDPOINTS` | (empty) | Comma-separated list of peer witness URLs, set by NixOS module |
| `THEOREM_WITNESS_TOKEN` | (required; startup fails closed if missing or shorter than 32 characters) | Bearer token used by `GET /export-state`, `GET /rebuild-authority-state`, and `POST /restore-authority-state`, and by Ring 3 consumers that read token-protected authority snapshots |

The same bearer token currently protects the published `3.0` organization and
lifecycle surfaces that are intentionally operator-controlled:

- `GET /organizations/events`
- `GET /organizations/explain`
- `GET /organization-roles/explain`
- `GET /decision-paths/explain`
- `GET /execution-lanes/explain`
- `GET /budget-envelopes/explain`
- `GET /interventions/explain`
- `GET /treaties/explain`
- `GET /execution-capabilities/lifecycle`
- `POST /execution-capabilities/lifecycle`
- `POST /organizations/bootstrap`
- `POST /organizations/admin-batch`
- `POST /releases/declared`

The running server also publishes:

- `GET /openapi.json`
- `GET /openapi.yaml`
- `GET /capabilities`
- `GET /capability-negotiation`
- `GET /auth/discovery`

Use those runtime discovery surfaces when integrating against a live node
rather than inferring request shapes or auth from validation failures.

The CLI `--db` argument takes precedence over environment-derived paths. When running via systemd, the NixOS module constructs the full `ExecStart` command with `--db ${cfg.dataDir}/theorem.db`.
