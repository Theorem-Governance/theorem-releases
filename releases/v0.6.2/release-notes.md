# theoremOS v0.6.2 Release Notes

## Summary

v0.6.2 is a patch release on the 0.6.x preview line. It fixes the consumer direct-write theoremOS image so it boots on the validated Hetzner UEFI target, repairs first-boot configuration mutation and runtime secret generation, emits concrete activation digests in release artifacts, and hardens release publication so direct-write assets cannot publish without exact-artifact hardware proof.

This coordinated release session also ships the companion theorem-native BIOS machine-kernel artifacts for v0.6.2.

## Operator Highlights

- Consumer direct-write boot now uses the installed FreeBSD loader.efi on the ESP with startup.nsh as the UEFI shell fallback.
- First boot now preserves multi-line TOML arrays correctly, generates theorem-node.env automatically, and avoids auto-enabling services that require operator-supplied environment files.
- Installer and upgrade-bundle activation manifests now carry non-null generation and artifact digests.
- Release publication now fails closed unless the direct-write image hash matches a fresh Hetzner hardware-validation report.
- Companion BIOS theorem-kernel assets are published in the same release session for the separate theorem-native machine-kernel lane.

## Validated theoremOS Artifact

- theoremos-v0.6.2-amd64-direct-write.img.xz
- SHA-256: ad5bbd05e0db7e8e74ac4c9e5f6eca14368fefa18c69078b878e7e68c813dcd1
- Hardware validation target: Hetzner 125042903 / 5.161.121.1 on March 29, 2026

## Companion BIOS Kernel Artifacts

- theorem-kernel-v0.6.2-amd64-direct-write.img.xz
- SHA-256: 3c4e5db84e7d3700cd010f1a299f3445871984ddc7c48b0bdddb7269c59c47aa
- theorem-kernel-v0.6.2-x86_64-bios.bin
- SHA-256: a0a7386c328812234a83efab261b5f6a0a2f1bc8d51f2720adcacf6f894d2fdf
- Verified separately on Hetzner 125119129 / 178.156.240.42 for the BIOS machine-kernel management lane.

## Known Scope

- The consumer release-blocking lane is the FreeBSD direct-write theoremOS image.
- The theorem-native BIOS and theorem-native UEFI machine-kernel lanes remain separate validation surfaces and are not substituted for consumer theoremOS release proof.
