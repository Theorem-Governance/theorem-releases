# theoremOS v0.6.1

Full-stack FreeBSD installer with theorem-native BIOS kernel. Ships both the sovereign governance OS and bare-metal kernel in one release.

## What's included

### FreeBSD Installer (289 MB xz)
- **Direct-write disk image** — dd to disk, boot into full theoremOS on FreeBSD 14.4
- **Rootfs tarball** — theorem-init, theorem-node, theorem-gateway, theorem-supervisor, theorem-govagent
- **Upgrade bundle** — in-place upgrade from v0.5.x
- **Standalone theorem-node** — single-binary deployment

### Theorem-native Kernel
- **BIOS boot kernel** (126 KiB xz) — bare-metal x86_64, no OS dependency
- Virtio-net + smoltcp TCP/IP stack
- Management server on port 2222 with token auth

## Changes since v0.5.0

### Theorem-native kernel (new)
- BIOS-native boot chain: pMBR → loader → 64-bit long mode kernel
- PCI enumeration with PCIe bridge scanning (cloud VPS support)
- Virtio-net driver (modern + legacy) with smoltcp TCP/IP
- DHCP with /32 cloud VPS workaround (Hetzner, AWS, GCP)
- PIT real-time clock, virtio-blk driver
- Token-based management auth from activation manifest
- Measurement chain with SHA-256 attestation

### FreeBSD stack
- Runtime integration: rc.d services, config UX, port env vars
- Native direct-write image builder on FreeBSD
- Release pipeline hardening and pre-publish validation
- Kernel-only release pipeline and image builder

### Verified on
- **Hetzner CPX11** — SeaBIOS, virtio-net behind PCIe bridges, /32 DHCP
- **QEMU** — Various network configurations
- **Local FreeBSD rig** — FreeBSD 14.4-RELEASE amd64

## Deployment

### Full install (FreeBSD)
```bash
xz -d theoremos-v0.6.1-amd64-direct-write.img.xz
dd if=theoremos-v0.6.1-amd64-direct-write.img of=/dev/sda bs=1M
```

### Upgrade from v0.5.x
```bash
tar xzf theoremos-v0.6.1-x86_64-unknown-freebsd-upgrade-bundle.tar.gz
cd theoremos-v0.6.1-x86_64-unknown-freebsd-upgrade-bundle
sh upgrade.sh
```

### Kernel-only (bare metal)
```bash
xz -d theorem-kernel-v0.6.1-amd64-direct-write.img.xz
dd if=theorem-kernel-v0.6.1-amd64-direct-write.img of=/dev/sda bs=1M
```
