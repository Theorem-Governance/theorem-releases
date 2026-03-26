# Packaging And Build Policy

This document defines the minimum packaging and build discipline for theoremOS release artifacts.

## Source Intake

- Release builds consume only committed workspace sources plus lockfile-pinned third-party dependencies.
- No release builder may fetch new crate versions during a release build. Cargo must run in frozen/offline mode for release composition.
- Auxiliary tooling required for release composition, such as `cargo-cyclonedx` and `cross`, must already exist on the Ring 1 build host. The release path must not install them opportunistically.

## Architecture Tiers

- `Primary`: release-blocking target. Missing artifacts or failed rehearsal block publication.
- `Legacy`: historical or compatibility target. Omission does not block theoremOS publication.
- `Planned`: intent only. No artifact or servicing promise until promoted into a maintained tier.

## Build Discipline

- Release builds use committed lockfiles and `cargo --frozen`.
- Builder provenance must record the exact build command, tag, and git material.
- Reproducibility checks compare independent release directories using `scripts/verify-release-reproducibility.sh`.

## Trust Boundaries

- The public release feed, manifest, integrity evidence, release notes, and revocation metadata are trusted only after signature verification against `releases/trust-root.json`.
- Hashes alone are insufficient for release admission.
