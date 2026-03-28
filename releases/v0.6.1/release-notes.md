# theoremOS v0.6.1

Full-stack FreeBSD installer with hybrid BIOS+UEFI boot and theorem-native BIOS kernel. Ships both the sovereign governance OS and bare-metal kernel in one release.

## What's included

### FreeBSD Installer (287 MB xz)
- **Direct-write disk image** — dd to disk, boots on both BIOS and UEFI hardware
- **Hybrid boot**: BIOS via gptzfsboot, UEFI via theorem-uefi-loader (EFI/BOOT/BOOTX64.EFI)
- **Rootfs tarball** — theorem-init, theorem-node, theorem-gateway, theorem-supervisor, theorem-govagent
- **Upgrade bundle** — in-place upgrade from v0.5.x or v0.6.0
- **Standalone theorem-node** — single-binary deployment

### UEFI Boot Chain
- **theorem-uefi-loader** (BOOTX64.EFI) — UEFI fallback boot path, loads theorem-machine-kernel
- **theorem-machine-kernel** (THEOKERN.EFI) — UEFI kernel with boot catalog and activation manifest
- Populated ESP with `EFI/BOOT/BOOTX64.EFI`, `EFI/THEOREM/THEOKERN.EFI`, boot catalog, activation manifest

### Theorem-native Kernel
- **BIOS boot kernel** (126 KiB xz) — bare-metal x86_64, no OS dependency
- Virtio-net + smoltcp TCP/IP stack
- Management server on port 2222 with token auth

## Changes since v0.5.0

### UEFI boot support (new in this build)
- Populated EFI System Partition with theorem-uefi-loader and theorem-machine-kernel
- UEFI fallback boot path (EFI/BOOT/BOOTX64.EFI) for firmware auto-discovery
- Boot catalog and activation manifest on ESP for governed UEFI boot chain
- Build pipeline now requires --efi-dir and fails if ESP would be empty

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
- **Hetzner CCX13** (UEFI) — target hardware class for this fix
- **Hetzner CPX11** — SeaBIOS, virtio-net behind PCIe bridges, /32 DHCP
- **QEMU** — Various network configurations
- **Local FreeBSD rig** — FreeBSD 14.4-RELEASE amd64

## Deployment

### Full install (UEFI or BIOS)
```bash
xz -d theoremos-v0.6.1-amd64-direct-write.img.xz
dd if=theoremos-v0.6.1-amd64-direct-write.img of=/dev/sda bs=1M
```

### Upgrade from v0.5.x or v0.6.0
```bash
tar xzf theoremos-v0.6.1-x86_64-unknown-freebsd-upgrade-bundle.tar.gz
cd theoremos-v0.6.1-x86_64-unknown-freebsd-upgrade-bundle
sh upgrade.sh
```

### Kernel-only (bare metal BIOS)
```bash
xz -d theorem-kernel-v0.6.1-amd64-direct-write.img.xz
dd if=theorem-kernel-v0.6.1-amd64-direct-write.img of=/dev/sda bs=1M
```
