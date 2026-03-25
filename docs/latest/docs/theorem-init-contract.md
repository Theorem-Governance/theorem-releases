# theorem-init Service Contract

## 1. Overview

`theorem-init` is the PID 1 process for theoremOS. It owns the full machine lifecycle: bootstrapping the kernel environment, mounting ZFS datasets, creating system users, hardening the kernel, starting supervised services in dependency order, and enforcing continuous governance compliance. It is the only process that runs as init; every other daemon is a supervised child.

## 2. Configuration Format

Configuration is TOML. The default path is `/etc/theorem/theorem.conf`.

Generate a default configuration file:

```sh
theorem-init default-config > /etc/theorem/theorem.conf
```

Validate a configuration file without starting services:

```sh
theorem-init check --config /etc/theorem/theorem.conf
```

## 3. Service Registration Schema

Each managed service is declared as a `[[services]]` entry in the TOML configuration. The schema maps directly to the `ServiceEntry` struct:

| Field                        | Type             | Default | Description                                      |
|------------------------------|------------------|---------|--------------------------------------------------|
| `name`                       | String           | —       | Unique service identifier                        |
| `binary`                     | String           | —       | Absolute path to the service binary              |
| `args`                       | Array of String  | `[]`    | Command-line arguments passed to the binary      |
| `depends_on`                 | Array of String  | `[]`    | Names of services that must start first          |
| `restart_on_failure`         | Boolean          | `true`  | Restart the service automatically on exit        |
| `health_check_interval_secs` | Integer (u64)   | `30`    | Seconds between health check probes              |
| `store_hash`                 | String (optional)| `null`  | Content-addressed store hash for binary integrity |

### Example

```toml
[[services]]
name = "theorem-bus"
binary = "/usr/local/bin/theorem-bus"
args = ["--socket", "/var/run/theorem/bus.sock"]
depends_on = []
restart_on_failure = true
health_check_interval_secs = 10
store_hash = "sha256:ab12cd34..."

[[services]]
name = "theorem-node"
binary = "/usr/local/bin/theorem-node"
args = ["--bus", "/var/run/theorem/bus.sock"]
depends_on = ["theorem-bus"]
restart_on_failure = true
health_check_interval_secs = 30
store_hash = "sha256:ef56gh78..."
```

In this example, `theorem-node` will not start until `theorem-bus` has started and passed its first health check.

## 4. Configuration Reference

All fields from `TheoremConfig`, with their types and defaults:

| Field                      | Type             | Default                                    | Description                                              |
|----------------------------|------------------|--------------------------------------------|----------------------------------------------------------|
| `store_root`               | String           | `/theorem-store`                           | Root path for the content-addressed artifact store       |
| `boot_pool`                | String           | `theorem`                                  | ZFS pool name                                            |
| `boot_dataset`             | String           | `theorem/ROOT/default`                     | ZFS boot dataset                                         |
| `key_store_path`           | String           | `/etc/theorem/keys`                        | Path to cryptographic key material                       |
| `log_dir`                  | String           | `/var/log/theorem`                         | Directory for theorem log files                          |
| `max_log_size`             | Integer          | `10485760` (10 MB)                         | Maximum size of a single log file in bytes               |
| `max_log_files`            | Integer          | `5`                                        | Maximum number of rotated log files retained             |
| `pf_anchor`                | String           | `theorem`                                  | PF firewall anchor name                                  |
| `bus_socket`               | String           | `/var/run/theorem/bus.sock`                | Unix domain socket for the theorem message bus           |
| `hostname`                 | String           | `theoremos`                                | System hostname                                          |
| `timezone`                 | String           | `UTC`                                      | System timezone (IANA format)                            |
| `locale`                   | String           | `en_US.UTF-8`                              | System locale                                            |
| `serial_console_speed`     | Integer          | `115200`                                   | Serial console baud rate                                 |
| `swap_size`                | String           | `auto`                                     | Swap size: `auto` (2x RAM), `none`, or explicit (e.g. `4G`) |
| `zfs_arc_max_percent`      | Integer          | `50`                                       | Maximum percentage of RAM for ZFS ARC cache              |
| `nts_servers`              | Array of String  | `[time.cloudflare.com, nts.netnod.se]`     | NTS-capable time servers for chronyd                     |
| `dns_upstream`             | Array of String  | `[1.1.1.1@853, 9.9.9.9@853]`              | Upstream DNS resolvers (DNS-over-TLS, port 853)          |
| `base_packages`            | Array of String  | `[chrony, unbound]`                        | Packages required in the base image                      |
| `scrub_interval_days`      | Integer          | `7`                                        | Days between ZFS scrub operations                        |
| `gc_interval_hours`        | Integer          | `24`                                       | Hours between store garbage collection runs              |
| `log_rotate_interval_hours`| Integer          | `24`                                       | Hours between log rotation                               |
| `syslog_enabled`           | Boolean          | `true`                                     | Enable syslogd during bootstrap                          |
| `kern_msgbuf_size`         | Integer          | `65536`                                    | Kernel message buffer size in bytes                      |
| `coredump_enabled`         | Boolean          | `false`                                    | Enable core dumps (disabled for security by default)     |
| `system_users`             | Array            | `[]`                                       | Additional system users (see default users below)        |

### Default System Users

These users are always created during bootstrap, regardless of the `system_users` list:

| Username         | UID | Purpose                        |
|------------------|-----|--------------------------------|
| `_theorem`       | 900 | Primary theorem service account|
| `_theorem_node`  | 901 | Node daemon service account    |
| `_theorem_bus`   | 902 | Message bus service account    |

## 5. Bootstrap Lifecycle

When `theorem-init` starts as PID 1, it executes these 13 steps sequentially. Every step must succeed before the next begins. A failure in any step halts the boot.

| Step | Name               | Description                                                                 |
|------|--------------------|-----------------------------------------------------------------------------|
| 1    | `mount-devfs`      | Mount `/dev`, `/dev/fd`, and `/tmp`                                         |
| 2    | `mount-zfs`        | Import all ZFS pools (`zpool import -a`) and mount all datasets (`zfs mount -a`) |
| 3    | `set-hostname`     | Read hostname from `/etc/theorem/hostname`; fall back to `theoremos`        |
| 4    | `create-users`     | Create system users `_theorem:900`, `_theorem_node:901`, `_theorem_bus:902` |
| 5    | `configure-syslog` | Start `syslogd`                                                            |
| 6    | `configure-sysctl` | Apply kernel hardening: `kern.coredump=0`, `kern.msgbuf_size=65536`, `vm.panic_on_oom=0`, `kern.sugid_coredump=0` |
| 7    | `configure-network`| Bring up loopback interface; write `/etc/resolv.conf` pointing to `127.0.0.1` |
| 8    | `configure-clock`  | Start `chronyd` with NTS-authenticated time sources                         |
| 9    | `configure-dns`    | Start `unbound` with DNS-over-TLS to upstream resolvers                     |
| 10   | `set-timezone`     | Read timezone from `/etc/theorem/timezone`; symlink `/etc/localtime`        |
| 11   | `load-veriexec`    | Load `mac_veriexec` kernel module from `/etc/veriexec.d/theorem.manifest`   |
| 12   | `raise-securelevel`| Set `kern.securelevel=1` (no further kernel module loads, no raw disk writes) |
| 13   | `register-signals` | Register signal handlers for `SIGCHLD`, `SIGTERM`, `SIGHUP`                |

After step 13 completes, control passes to the async supervisor.

## 6. Async Supervisor Lifecycle

The async supervisor manages all declared services:

1. **Load configuration** -- Parse the TOML config and extract the `[[services]]` entries.
2. **Build dependency graph** -- Construct a directed acyclic graph from `depends_on` declarations. Cycles are a fatal configuration error.
3. **Topological sort** -- Sort services using Kahn's algorithm to determine start order.
4. **Start services** -- Launch each service in topological order. A service is not started until all of its dependencies are running and healthy.
5. **Run governance runtime** -- Begin continuous drift detection and compliance monitoring (see section 12).

## 7. Signal Handling

`theorem-init` registers handlers for three signals during bootstrap step 13:

| Signal    | Behavior                                                                                      |
|-----------|-----------------------------------------------------------------------------------------------|
| `SIGCHLD` | Reap terminated child processes. If `restart_on_failure` is `true` for the service, restart it after a 1-second delay. |
| `SIGHUP`  | Reload configuration from disk. The service graph is rebuilt and any changes are applied.      |
| `SIGTERM` | Initiate graceful shutdown. Services are stopped in reverse topological order (dependents first, then their dependencies). |

## 8. CLI Reference

```
theorem-init [OPTIONS] [COMMAND]
```

### Global Options

| Option                     | Default                        | Description                 |
|----------------------------|--------------------------------|-----------------------------|
| `-c, --config <CONFIG>`    | `/etc/theorem/theorem.conf`    | Path to configuration file  |
| `-l, --log-level <LOG_LEVEL>` | `info`                      | Log verbosity level         |

### Commands

| Command            | Description                                           |
|--------------------|-------------------------------------------------------|
| `run`              | Run as PID 1 (default when no command is specified)   |
| `check`            | Validate configuration and exit                       |
| `default-config`   | Print the default configuration to stdout             |
| `version`          | Show version and build information                    |
| `image build`      | Build a theoremOS image                               |
| `image rollback`   | Roll back to the previous boot environment            |
| `store admit`      | Admit an artifact to the content-addressed store      |
| `store verify`     | Verify store integrity                                |
| `store gc`         | Garbage collect unreferenced artifacts                |
| `boot`             | Show boot environment status                          |
| `governance`       | Show governance compliance report                     |
| `workload list`    | List running workloads                                |

### Environment Variables

| Variable       | Description                                                    |
|----------------|----------------------------------------------------------------|
| `THEOREM_LOG`  | Overrides the log level set by `--log-level`                   |

## 9. rc.d Integration

`theorem-init` ships an rc.d script at `/usr/local/etc/rc.d/theorem_init`:

```sh
# PROVIDE: theorem_init
# REQUIRE: FILESYSTEMS NETWORKING
# BEFORE:  LOGIN
# KEYWORD: shutdown
```

| Property    | Value                                                          |
|-------------|----------------------------------------------------------------|
| `command`   | `/sbin/theorem-init run --config /etc/theorem/theorem.conf` |
| `pidfile`   | `/var/run/theorem/theorem-init.pid`                            |

The script declares `REQUIRE: FILESYSTEMS NETWORKING` to ensure the kernel has mounted base filesystems and brought up network interfaces before `theorem-init` takes over. It declares `BEFORE: LOGIN` so that all theorem services are running before any user login is permitted. The `shutdown` keyword ensures the script's stop logic is invoked during system shutdown.

## 10. First Boot Expectations

The following must exist on disk before `theorem-init run` can succeed:

| Path                                | Contents                                                |
|-------------------------------------|---------------------------------------------------------|
| `/etc/theorem/theorem.conf`         | TOML configuration file                                 |
| `/etc/theorem/keys/`                | Cryptographic key material (signing keys, TLS certs)    |
| `/etc/theorem/hostname`             | Single-line hostname (optional; defaults to `theoremos`)|
| `/etc/theorem/timezone`             | Single-line IANA timezone (optional; defaults to `UTC`) |
| `/etc/veriexec.d/theorem.manifest`  | veriexec manifest for binary integrity verification     |
| `/var/run/theorem/`                 | Runtime directory for PID file and bus socket            |
| `/var/log/theorem/`                 | Log directory                                           |
| `/theorem-store/`                   | Content-addressed artifact store root                   |
| Service binaries referenced in config | All `binary` paths declared in `[[services]]`         |

## 11. Health Checking

Each service has an independent health check loop running at the interval specified by `health_check_interval_secs` (default: 30 seconds).

- The supervisor probes the service process at each interval.
- A service that has exited is marked unhealthy immediately via `SIGCHLD` reaping.
- If `restart_on_failure` is `true`, the supervisor restarts the service after a 1-second delay.
- Health status feeds into the governance runtime's drift detection.

## 12. Governance Runtime

The governance runtime runs continuously after service startup and enforces compliance:

- **Drift detection** -- The runtime compares the current system state against the declared configuration. A drift score exceeding the threshold of `0.5` triggers a governance violation.
- **Quarantine** -- On a governance violation, the offending service or component is quarantined (isolated from the rest of the system).
- **Shutdown threshold** -- After 3 consecutive governance failures, `theorem-init` initiates a full system shutdown. This prevents a compromised or misconfigured system from continuing to operate.

The governance compliance report is available at any time via:

```sh
theorem-init governance
```
