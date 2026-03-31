# theoremOS v0.6.3 Release Notes

## Summary

v0.6.3 is a patch release on the 0.6.x preview line. It fixes the consumer direct-write theoremOS image so it becomes self-contained by default, repairs first-boot configuration mutation and runtime secret generation, emits concrete activation digests in release artifacts, and hardens release publication so direct-write assets cannot publish without exact-artifact hardware proof.

This coordinated release session also ships the companion theorem-native UEFI machine-kernel artifacts for v0.6.3, including the proven two-boot acceptance path.

## Operator Highlights

- Consumer direct-write boot now uses the installed FreeBSD loader.efi on the ESP with startup.nsh as the UEFI shell fallback.
- First boot now preserves multi-line TOML arrays correctly, generates theorem-node.env automatically, and avoids auto-enabling services that require operator-supplied environment files.
- Installer and upgrade-bundle activation manifests now carry non-null generation and artifact digests.
- Release publication now fails closed unless the direct-write image hash matches a fresh Hetzner hardware-validation report.
- Companion theorem-native UEFI artifacts are published in the same release session for the separate theorem-native machine-kernel lane.

## Release-Blocking theoremOS Artifact

- theoremos-v0.6.3-amd64-direct-write.img.xz
- SHA-256: 3e5d86e6f09f521063bc382e26b112a1d5ad501e6458f29232fcac80099e7c1e
- Release note: this exact image is the release-blocking artifact for Hetzner UEFI hardware validation and on-box governed proof.

## Companion theorem-native UEFI Artifacts

- theorem-native-v0.6.3-esp.img
- SHA-256: d21dd9c1a975631d7b4ff1bea67c5134449e44eaeb6d844f022b89658b09a7a3
- theorem-machine-kernel-v0.6.3-x86_64-unknown-uefi.efi
- theorem-uefi-loader-v0.6.3-x86_64-unknown-uefi.efi
- Verified through the automated two-boot QEMU acceptance smoke for the theorem-native UEFI lane.

## Known Scope

- The consumer release-blocking lane is the FreeBSD direct-write theoremOS image.
- The theorem-native UEFI machine-kernel lane remains a separate validation surface and is not substituted for consumer theoremOS release proof.
