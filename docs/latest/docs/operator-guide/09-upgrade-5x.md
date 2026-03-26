# Upgrade: 0.5.x to 0.6.x

## Summary

The supported `0.5.x -> 0.6.x` update path is an in-place theoremOS upgrade on
an existing FreeBSD/ZFS host. It is **not** a bare-metal re-image. The release
must publish an `upgrade-bundle` artifact plus a machine-readable
`upgrade_contract.paths` entry declaring the path from `v0.5.x` to the target
`v0.6.x` release.

## Upgrade Classification

| Property | Value |
|----------|-------|
| Compatibility | declared in `manifest.json` |
| Upgrade type | in-place boot-environment upgrade |
| Machine executable | yes |
| Required artifact kind | `upgrade-bundle` |
| Reboot boundary | required |
| Rollback type | revert `bootfs` to prior boot environment |

## Preconditions

All of the following must be true before running the upgrader:

- the current machine is already running theoremOS `0.5.x`
- the root pool is ZFS and named `theorem`
- the current root dataset is under `theorem/ROOT/*`
- the release manifest contains an `upgrade_contract.paths` entry from
  `v0.5.x` to the target release using `artifact_kind=upgrade-bundle`
- the `upgrade-bundle` artifact hash matches `integrity.json`

## Artifact Selection

Resolve the artifact from `releases/<tag>/manifest.json`.

Valid selection rule:

- choose `artifact_kind=upgrade-bundle`
- verify `supported_upgrade_from` includes `v0.5.x`
- verify the bundle contains a `supported-upgrade-from` file matching the
  manifest declaration
- resolve the upgrader at `${archive_root}/${upgrade_entrypoint}`

Fail closed if any of those conditions are absent.

## Expected Upgrade Behavior

The `upgrade.sh` entrypoint is expected to perform the following steps:

1. Read the currently installed theoremOS version from
   `/etc/theorem/release.version` and verify it matches the supported source
   line declared by `supported-upgrade-from`.
2. Snapshot the current boot environment.
3. Snapshot `/var/lib/theorem` state or promote it to a preserved ZFS dataset
   if it is still rooted inside the active boot environment.
4. Create the target boot environment, for example `theorem/ROOT/0.6.0`.
5. Install the new theoremOS payload into that target boot environment.
6. Preserve `theorem/store`, `theorem/log`, and `/var/lib/theorem`.
7. Preserve or import the existing machine spec and render the target host
   configuration from it.
8. Set `zpool bootfs=theorem/ROOT/<target>`.
9. Emit pending-upgrade markers and require a reboot.

The new release is not considered active until the host reboots into the new
boot environment, theorem-init observes healthy service/runtime state, and the
boot is promoted to known-good.

## Rollback

Rollback is a boot-environment operation, not an ad hoc file restore.

Minimum rollback sequence:

1. Boot from rescue or an approved fallback environment if the upgraded system
   is not reachable.
2. Import pool `theorem`.
3. Point `bootfs` back to the prior boot environment snapshot or dataset.
4. If the upgrader migrated `/var/lib/theorem` into a preserved state dataset,
   keep that dataset mounted.
5. Reboot.

If a release applied forward-only database migrations after first boot, rollback
of the runtime binary is still allowed, but rollback of the live database may
require restoring the pre-upgrade backup.

## Verification

After reboot into `0.6.x`, all of the following must pass:

- `curl -f http://localhost:3170/health`
- `theorem-node --db /var/lib/theorem/theorem.db status`
- `sqlite3 /var/lib/theorem/theorem.db "PRAGMA integrity_check;"`
- `theorem-init boot status`
- `zpool get bootfs theorem`
- `zfs list -o name,mountpoint | grep 'theorem/ROOT/'`

`theorem-init boot status` should show:

- `release_version`
- `booted_dataset`
- `next_default_dataset`
- `rollback_candidates`
- `known_good_dataset`
- `accepted_release_version`
- `pending_upgrade_dataset`
- `pending_upgrade_version`
- upgrade provenance fields when the current boot environment came from a prior
  release

## What This Replaces

This upgrade path replaces the need to re-image bare metal for theoremOS to
theoremOS transitions. Re-image plus state restore remains the supported path
for cross-substrate migrations such as Linux/NixOS `4.x -> 0.5.x`.
