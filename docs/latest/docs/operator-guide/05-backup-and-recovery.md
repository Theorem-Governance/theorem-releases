# Backup and Recovery

This document covers SQLite database backup, JSON state export, disaster recovery, the forward-only migration design, and database integrity.

## Database Files to Back Up

Two SQLite database files must be backed up:

| File | Contents | Criticality |
|------|----------|-------------|
| `theorem.db` | Governance state: acts, entities, authorizations, attributions, attestations, drift signals, kernel parameters, proof obligations, scopes, genesis records, workflows, governance acts, audit executions, measurement methods, integrity mechanisms, attestation logs, attestation schemas | Critical -- loss means loss of governance state |
| `theorem-witness.db` | Witness log: append-only sequence of attestation publications with hash chain integrity | High -- can be reconstructed from peer witnesses, but local integrity history is lost |

Both files are in the data directory (default `/var/lib/theorem/`, configurable via `services.theorem.dataDir` or `--db` CLI argument).

## SQLite Backup Procedure

### Online Backup (Recommended)

SQLite supports online backup while the database is in use. Use the SQLite `.backup` command or the `sqlite3` CLI:

```bash
# Stop writes briefly for consistency (optional but recommended)
sqlite3 /var/lib/theorem/theorem.db ".backup /backup/theorem-$(date +%Y%m%d-%H%M%S).db"
sqlite3 /var/lib/theorem/theorem-witness.db ".backup /backup/theorem-witness-$(date +%Y%m%d-%H%M%S).db"
```

The `.backup` command creates a consistent snapshot even if the database is being written to. However, for maximum consistency between the governance and witness databases, briefly pause the server or perform both backups in quick succession.

### Offline Backup

If you can tolerate downtime:

```bash
systemctl stop theorem-node
cp /var/lib/theorem/theorem.db /backup/theorem-$(date +%Y%m%d-%H%M%S).db
cp /var/lib/theorem/theorem-witness.db /backup/theorem-witness-$(date +%Y%m%d-%H%M%S).db
systemctl start theorem-node
```

### Backup Schedule

Recommended backup frequency depends on your `audit_interval_ms`:

- **Minimum:** Back up at least every `audit_interval_ms` (default: every hour)
- **Recommended:** Back up every 15 minutes, retain 7 days of hourly snapshots and 90 days of daily snapshots
- Back up both files in the same backup job to maintain consistency

### Backup Verification

After each backup, verify the backup file is a valid SQLite database:

```bash
sqlite3 /backup/theorem-latest.db "PRAGMA integrity_check;"
```

Expected output: `ok`

## JSON State Export

**Endpoint:** `GET /export-state`

**CLI:**
```bash
theorem-node --db theorem.db export-state
theorem-node --db theorem.db export-state --pretty   # pretty-print (CLI only)
```

The `--pretty` flag is only available via the CLI; the HTTP endpoint always returns compact JSON.

Returns the full governance state as a JSON document. This is a logical export -- it contains all records but not SQLite-specific metadata (WAL files, indexes).

### Response Format

```json
{
  "bootstrapped": true,
  "summary": {
    "entity_count": 5,
    "scope_count": 2,
    "authorization_count": 8,
    "proof_obligation_count": 25,
    "genesis_record_count": 2,
    "proof_status_counts": { "Active": 25 }
  },
  "entities": [
    { "id": "ent-...", "scope_id": "scp-...", "participation_type": "Authorizing", "is_system": false, "is_bootstrap": true, "status": "Active" }
  ],
  "authorizations": [
    { "id": "auth-...", "principal_id": "ent-...", "scope_id": "scp-...", "action_vocabulary": ["execution_act"], "granted_by_id": "ent-...", "status": "Active" }
  ],
  "scopes": [
    { "id": "scp-...", "is_root": true, "domains": ["governance"] }
  ],
  "proof_obligations": [
    { "id": "po-...", "property_name": "PROP-PARAM-001", "proof_type": "Formal", "status": "Active", "submitted_at": "...", "expires_at": "...", "proof_hash": "..." }
  ],
  "last_audit": null
}
```

### Use Cases

- **Audit trail:** Capture a point-in-time snapshot for external audit
- **Comparison:** Diff two state exports to identify changes
- **Migration:** Transfer state to a new instance (requires re-bootstrapping with the same spec hash)
- **Debugging:** Inspect state without direct SQLite access

### Limitations

- The JSON export does not include the witness log (use the witness database backup for that)
- Re-importing a JSON export into a running instance is not supported -- there is no import endpoint
- Large instances may produce multi-megabyte JSON files; use `--pretty` (CLI only) for human inspection

## Disaster Recovery Procedure

### Scenario: Database Corruption or Loss

1. **Stop the service:**
   ```bash
   systemctl stop theorem-node
   ```

2. **Restore from backup:**
   ```bash
   cp /backup/theorem-latest.db /var/lib/theorem/theorem.db
   cp /backup/theorem-witness-latest.db /var/lib/theorem/theorem-witness.db
   ```

3. **Verify database integrity:**
   ```bash
   sqlite3 /var/lib/theorem/theorem.db "PRAGMA integrity_check;"
   sqlite3 /var/lib/theorem/theorem-witness.db "PRAGMA integrity_check;"
   ```

4. **Start the service:**
   ```bash
   systemctl start theorem-node
   ```

5. **Verify governance state:**
   ```bash
   curl http://localhost:3170/status
   curl http://localhost:3170/audit
   curl http://localhost:3170/verify
   ```

6. **Verify witness integrity:**
   ```bash
   curl http://localhost:3170/witness/digest
   ```

### Scenario: Hardware Failure (Full Server Loss)

1. Provision new hardware meeting DEC-007 requirements (see [Deployment](02-deployment.md))
2. Deploy NixOS configuration from version control
3. Restore both database files from off-site backup
4. Start the service
5. Run full audit and verifier to confirm state consistency
6. Verify witness log integrity and re-sync with peer witnesses if needed

### Scenario: Witness Log Divergence

If the local witness log diverges from peers:

1. Use `GET /witness/digest` to compare your log's head hash and length against peer digests
2. If peers agree and your log diverges, your local log may be corrupted
3. Restore the witness database from the most recent backup that pre-dates the divergence
4. Re-publish attestations created since the backup by re-running the attestation finalization process

See [Troubleshooting](06-troubleshooting.md) for witness divergence investigation procedures.

## Forward-Only Migration Design

Theorem's governance model is forward-only by design:

- **No rollbacks:** There is no mechanism to undo a governance act. The act, its attribution, and its attestation are permanent records.
- **No schema downgrades:** Database schema changes are forward-only migrations applied on startup.
- **Amendment, not deletion:** Parameters are amended via governance acts, not overwritten. The amendment history is preserved in the governance act chain.
- **Proof invalidation, not removal:** Invalid proofs are marked as Invalidated (with cascade through the dependency graph), not deleted.

### Implications for Operators

- Restoring from backup means losing governance records created after the backup timestamp. Those records cannot be reconstructed from the kernel alone -- they must be re-submitted.
- If a parameter was amended to an incorrect value, the correction is another amendment (not a rollback). Both amendments appear in the governance record.
- If a proof was incorrectly invalidated, the correction is a new proof submission. The invalidated proof remains in the record with its cascade effects.

## Database File Permissions and Integrity

### File Permissions

The systemd service runs with `DynamicUser = true` and `StateDirectory = "theorem"` with mode `0700`. This means:

- The database files are owned by the dynamic user created for the service
- Only the service process can read or write the files
- Backup scripts need `sudo` or a dedicated user with read access to `/var/lib/theorem/`

### WAL Mode

SQLite databases in Theorem use WAL (Write-Ahead Logging) mode for concurrent read/write access. When backing up, ensure you capture both the main database file and its WAL file (`theorem.db-wal`) if it exists. The `.backup` command handles this automatically.

### Integrity Monitoring

The NixOS `hardening.nix` module configures Linux audit rules to monitor the theorem data directory:

```
-w /var/lib/theorem/ -p rwxa -k theorem-data
```

This logs all read, write, execute, and attribute-change operations on files in the data directory. Review these audit logs periodically for unexpected access patterns.

### Nix Store Integrity

The theorem-node binary itself is integrity-protected by the Nix store's content-addressed design. Verify binary integrity with:

```bash
nix-store --verify --check-contents
```

Any modification to the binary will be detected as a hash mismatch.
