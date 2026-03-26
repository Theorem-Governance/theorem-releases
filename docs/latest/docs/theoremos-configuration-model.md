# theoremOS Configuration Model

theoremOS now uses `/etc/theorem/machine-spec.toml` as the authoritative host
declaration. `/etc/theorem/theorem.conf`, `/etc/rc.conf`, `/boot/loader.conf`,
`/etc/pf.conf`, `/etc/theorem/hostname`, and `/etc/theorem/timezone` are
rendered artifacts derived from that machine spec.

## Design Rules

- The machine spec is the declared source of truth for theorem-init boot, store,
  logging, network, security, and rollback-retention policy.
- Rendered host files must be reproducible from the machine spec without manual
  edits to generated outputs.
- Release upgrades must preserve operator-owned configuration by preserving or
  importing the existing machine spec before rendering the target boot
  environment.
- Boot-environment retention and boot-acceptance policy are part of host
  configuration and must be visible through `theorem-init boot status`.

## Current Scope

- This is a structured declarative host and image-configuration surface for the
  current theoremOS distro line.
- A machine spec is expected to round-trip through install, upgrade, rollback,
  and rebuild of canonical FreeBSD host files.
- theorem-init provides `machine default-spec`, `machine import-legacy`, and
  `machine render` so the declarative model is executable rather than a
  documentation-only contract.
