# Migration Contract: 4.x (Linux/NixOS) to 0.5.x (FreeBSD)

## Summary

The 0.5.x release is a substrate-level break from the 4.x line: the operating
system changes from Linux (NixOS) to FreeBSD 14.4, the init system changes from
systemd to theorem-init (PID 1), and storage moves to ZFS with mandatory
integrity enforcement via MAC veriexec and securelevel=1. In-place upgrade is
not supported because the host OS itself is replaced. Migration is performed as
a re-image of the target machine followed by data restore. The governance
database and witness database are fully portable between the two lines -- they
are SQLite files with identical schemas and forward-only migration chains.

## Migration Classification

| Property                   | Value                                                                                     |
|----------------------------|-------------------------------------------------------------------------------------------|
| Compatibility              | `breaking-major`                                                                          |
| Migration type             | Re-image only                                                                             |
| In-place upgrade           | Not supported (different OS substrates)                                                   |
| Governance DB              | Portable (SQLite, forward-only migrations V001--V020 with SHA-256 integrity)              |
| Witness DB                 | Portable (SQLite, append-only hash-chain)                                                 |
| Secrets                    | Environment variables -- restore manually                                                 |

## What Changes Between 4.x and 0.5.x

| Component          | 4.x (Linux/NixOS)     | 0.5.x (FreeBSD)                        |
|--------------------|------------------------|-----------------------------------------|
| Operating system   | Linux / NixOS          | FreeBSD 14.4-RELEASE                    |
| Init system        | systemd                | theorem-init (PID 1)                    |
| Storage            | Filesystem (ext4/xfs)  | ZFS (pool `theorem`)                    |
| Process isolation  | None / cgroups         | FreeBSD jails                           |
| Binary integrity   | None                   | MAC veriexec + `securelevel=1`          |
| Networking         | iptables               | PF firewall                             |
| Secrets            | Environment variables  | Environment variables (unchanged)       |

## What Is Portable

The following artifacts carry forward from 4.x to 0.5.x without modification:

- **theorem.db** (governance state) -- same SQLite schema, same migration chain
  (V001--V020). The 0.5.x node will apply any new migrations on first startup.
- **theorem-witness.db** (witness log) -- same append-only hash-chain format.
  The chain is verified on restore; no entries are lost or rewritten.
- **Environment variables** -- restore these on the new host:
  - `THEOREM_COMMITMENT_SECRET`
  - `THEOREM_WITNESS_TOKEN`
  - `THEOREM_WITNESS_ENDPOINTS`
  - `THEOREM_ENFORCEMENT_MODE`
  - `THEOREM_DRIFT_THRESHOLD`
  - `THEOREM_COMMITMENT_TOKEN_TTL_HOURS`

## Pre-Migration Checklist

Complete every step before powering down the 4.x host.

1. **Record current schema version.**

   ```sh
   sqlite3 theorem.db "SELECT MAX(version) FROM schema_version;"
   ```

   Save the output -- you will compare it after migration.

2. **Verify governance DB integrity.**

   ```sh
   sqlite3 theorem.db "PRAGMA integrity_check;"
   ```

   The only acceptable output is `ok`. Any other result means the database is
   damaged and must be repaired before migration.

3. **Back up the governance DB.**

   ```sh
   sqlite3 theorem.db ".backup /backup/theorem-$(date +%Y%m%d).db"
   ```

4. **Back up the witness DB.**

   ```sh
   sqlite3 theorem-witness.db ".backup /backup/theorem-witness-$(date +%Y%m%d).db"
   ```

5. **Export governance state as JSON.**

   ```sh
   theorem-node --db theorem.db export-state --pretty > /backup/state-export.json
   ```

   This produces a human-readable snapshot for offline verification. It is not
   required for restore but is strongly recommended for audit trails.

6. **Record the witness head digest.**

   ```sh
   curl http://localhost:3170/witness/digest > /backup/witness-digest.json
   ```

   This captures the tip of the witness hash-chain so you can verify continuity
   after migration.

7. **Record environment variables.**

   Copy every `THEOREM_*` variable from the running environment or from
   `/etc/theorem/env`, `/etc/systemd/system/theorem-node.service.d/`, or
   wherever your deployment stores them. Store the values in a secure location
   (encrypted file, vault, etc.).

8. **Stop the 4.x service.**

   ```sh
   systemctl stop theorem-node
   ```

   Confirm it is fully stopped before proceeding:

   ```sh
   systemctl is-active theorem-node
   ```

   Expected output: `inactive`.

## Migration Procedure

1. **Provision a FreeBSD 14.4-RELEASE host with ZFS.**

   The target machine must have a ZFS pool named `theorem` (or one will be
   created by the installer). Minimum hardware requirements are unchanged from
   4.x.

2. **Download and verify the 0.5.x rootfs tarball.**

   ```sh
   fetch https://releases.theoremos.org/v0.5.0/theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz
   fetch https://releases.theoremos.org/v0.5.0/theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256
   sha256 -c theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256
   ```

   Do not proceed unless the checksum matches.

3. **Extract and run the installer.**

   ```sh
   tar xzf theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz
   cd theoremos-v0.5.0-x86_64-unknown-freebsd
   sh install.sh
   ```

   The installer creates the ZFS datasets, installs theorem-init, deploys the
   theorem-node binary with veriexec manifests, and configures PF.

4. **(Optional) Apply first-boot configuration before install.**

   If your deployment requires non-interactive network provisioning, place a
   `first-boot.conf` in the extracted directory before running `install.sh`:

   ```sh
   cp /backup/first-boot.conf theoremos-v0.5.0-x86_64-unknown-freebsd/first-boot.conf
   sh install.sh
   ```

   The installer detects `first-boot.conf` and applies network, DNS, and NTP
   settings on first boot.

5. **Restore the governance DB.**

   ```sh
   cp /backup/theorem-YYYYMMDD.db /var/lib/theorem/theorem.db
   chown theorem:theorem /var/lib/theorem/theorem.db
   chmod 0640 /var/lib/theorem/theorem.db
   ```

6. **Restore the witness DB.**

   ```sh
   cp /backup/theorem-witness-YYYYMMDD.db /var/lib/theorem/theorem-witness.db
   chown theorem:theorem /var/lib/theorem/theorem-witness.db
   chmod 0640 /var/lib/theorem/theorem-witness.db
   ```

7. **Restore environment variables.**

   Place the `THEOREM_*` variables in `/usr/local/etc/theorem/env`:

   ```sh
   cat > /usr/local/etc/theorem/env <<'ENVEOF'
   THEOREM_COMMITMENT_SECRET=<your-value>
   THEOREM_WITNESS_TOKEN=<your-value>
   THEOREM_WITNESS_ENDPOINTS=<your-value>
   THEOREM_ENFORCEMENT_MODE=<your-value>
   THEOREM_DRIFT_THRESHOLD=<your-value>
   THEOREM_COMMITMENT_TOKEN_TTL_HOURS=<your-value>
   ENVEOF
   chmod 0600 /usr/local/etc/theorem/env
   ```

8. **Start the service.**

   ```sh
   service theorem-node start
   ```

   Or reboot the host -- theorem-init will start theorem-node automatically as
   part of the boot sequence.

9. **Proceed to post-migration verification.**

## Post-Migration Verification

Run each check in order. All must pass before the migration is considered
complete.

1. **Health endpoint.**

   ```sh
   curl http://localhost:3170/health
   ```

   Expected: HTTP 200 with body `{"status":"ok"}`.

2. **Status endpoint.**

   ```sh
   curl http://localhost:3170/status
   ```

   Verify the governance state shows the expected entities, scopes, and
   enforcement mode.

3. **Governance audit.**

   ```sh
   theorem-node --db /var/lib/theorem/theorem.db audit
   ```

   Expected: all rules pass, zero violations.

4. **Schema version check.**

   ```sh
   sqlite3 /var/lib/theorem/theorem.db "SELECT MAX(version) FROM schema_version;"
   ```

   The version should match or exceed the value recorded in pre-migration step
   1. A higher version means 0.5.x applied new forward-only migrations on first
   startup -- this is expected and correct.

5. **Witness chain continuity.**

   ```sh
   curl http://localhost:3170/witness/digest
   ```

   Compare the output against `/backup/witness-digest.json`. The chain must
   extend from the same root. If the head digest differs, verify that the
   difference is due to new entries appended after migration (expected) rather
   than a broken chain (failure).

## Rollback

- **Rollback to 4.x:** Re-image the host back to Linux and restore from the
  pre-migration backups taken in the pre-migration checklist. The governance DB
  and witness DB from the backup are 4.x-compatible.

- **Forward-only schema migrations.** If the 0.5.x node applied new schema
  migrations after startup, that database cannot be used on a 4.x node. The
  migration chain is forward-only by design. Rollback in this case requires
  using the pre-migration backup, not the live 0.5.x database.

- **Post-migration governance state.** Any governance acts, attestations, or
  proof obligations created on the 0.5.x node after migration cannot be moved
  back to 4.x. They exist only in the 0.5.x database.

- **Manifest minimum version enforcement.** The 0.5.x manifest enforces
  `minimum_version: v0.5.0`. Downgrading to 4.x requires explicit governed
  approval: two approvals from `root-governance-authority` entities to override
  the version floor.

## FAQ

**Q: Can I run 4.x and 0.5.x side-by-side?**

A: Yes. Deploy them on separate hosts (or separate jails/VMs) pointing at
separate database files. They share the same wire protocol for witness peering,
so a 4.x node and a 0.5.x node can peer with each other and replicate witness
entries bidirectionally.

**Q: Will my 4.x proofs still be valid?**

A: Yes. Proof obligations are schema-compatible across versions. All proofs
created under 4.x carry forward through migration and remain verifiable on the
0.5.x node.

**Q: What about custom measurement methods?**

A: Custom measurement methods are stored in the governance database. They carry
forward with the database restore and require no additional steps.

**Q: Do I need to re-bootstrap on 0.5.x?**

A: No. The restored governance DB already contains the genesis records, entity
definitions, scopes, and authorizations from your 4.x deployment. The 0.5.x
node picks up where 4.x left off.

**Q: What if my witness DB is large?**

A: The witness DB is append-only and can grow substantially on long-running
deployments. It transfers as a single SQLite file. For very large databases
(>10 GB), consider using `rsync` or `zfs send` for the transfer rather than a
simple file copy, to take advantage of compression and incremental transfer.
