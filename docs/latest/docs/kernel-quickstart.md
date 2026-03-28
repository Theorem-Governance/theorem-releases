# Theorem Kernel — Quick Start

Deploy a bare-metal theorem machine in under 5 minutes. No operating system
required.

## What This Gets You

A dedicated x86_64 machine running the theorem governance kernel directly on
hardware. The kernel boots via BIOS, configures networking over DHCP, and
exposes a management server on TCP port 2222.

This is not a VM appliance or a container. It is a sovereign theorem machine
that owns the entire hardware surface from power-on.

## Requirements

- An x86_64 machine or VPS with **BIOS boot** (SeaBIOS, legacy BIOS, CSM)
- A disk you can overwrite (the image replaces everything on the target disk)
- Network connectivity (DHCP)
- A way to write disk images: `dd`, Hetzner rescue mode, or equivalent

UEFI-only machines are not yet supported by the kernel release. Use the
FreeBSD-based theoremOS image for UEFI hardware.

## Download

Grab the latest kernel release from
[GitHub Releases](https://github.com/Theorem-Governance/theorem-releases/releases):

```bash
VERSION="v0.6.1"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/download/${VERSION}/theorem-kernel-${VERSION}-amd64-direct-write.img.xz"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/download/${VERSION}/theorem-kernel-${VERSION}-amd64-direct-write.img.xz.sha256"
```

### Verify integrity

```bash
sha256sum -c "theorem-kernel-${VERSION}-amd64-direct-write.img.xz.sha256"
```

Expected output: `theorem-kernel-v0.6.1-amd64-direct-write.img.xz: OK`

## Deploy

### Option A: Local disk (Linux rescue environment)

```bash
# Decompress
xz -d "theorem-kernel-${VERSION}-amd64-direct-write.img.xz"

# Write to disk — this DESTROYS all data on /dev/sda
dd if="theorem-kernel-${VERSION}-amd64-direct-write.img" of=/dev/sda bs=1M status=progress

# Reboot into theorem
reboot
```

### Option B: Hetzner Cloud VPS

1. Create a CPX11 (or larger) instance
2. Boot into **rescue mode** (Linux 64-bit)
3. SSH into the rescue system
4. Run:

```bash
VERSION="v0.6.1"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/download/${VERSION}/theorem-kernel-${VERSION}-amd64-direct-write.img.xz"
xz -d "theorem-kernel-${VERSION}-amd64-direct-write.img.xz"
dd if="theorem-kernel-${VERSION}-amd64-direct-write.img" of=/dev/sda bs=1M
reboot
```

5. After reboot, the machine boots directly into the theorem kernel

### Option C: QEMU (local testing)

```bash
xz -d "theorem-kernel-${VERSION}-amd64-direct-write.img.xz"
qemu-system-x86_64 \
  -drive file="theorem-kernel-${VERSION}-amd64-direct-write.img",format=raw \
  -m 512M \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:2222 \
  -nographic
```

The kernel will print boot progress on the serial console (`-nographic`).

## Verify It's Running

Once the kernel boots (typically 2-3 seconds), connect to the management
server:

```bash
# For Hetzner/bare-metal (replace with your machine's IP):
nc <machine-ip> 2222

# For QEMU with port forwarding:
nc localhost 2222
```

The management server accepts JSON commands over TCP. A successful connection
confirms the full boot chain completed: BIOS → pMBR → loader → kernel →
PCI scan → virtio-net → DHCP → TCP listener.

## What's in the Image

The disk image is a 64 MiB GPT layout:

| Partition | Type | Contents |
|-----------|------|----------|
| p1 | BIOS boot (512K) | Second-stage loader (`loader.bin`) |
| p2 | FAT16 data | `THEOKERN.BIN` (kernel), `INIT.ELF`, `ACTIVAT.JSN` (activation manifest) |

Sector 0 contains the protective MBR with the first-stage bootstrap (`pmbr.bin`).

The compressed image is ~126 KiB. The raw image is 64 MiB.

## Release Assets

Each kernel release publishes four files:

| Asset | Purpose |
|-------|---------|
| `theorem-kernel-<tag>-amd64-direct-write.img.xz` | Bootable disk image (xz compressed) |
| `theorem-kernel-<tag>-amd64-direct-write.img.xz.sha256` | SHA-256 checksum |
| `theorem-kernel-<tag>-x86_64-bios.bin` | Raw kernel binary for custom image builds |
| `theorem-kernel-<tag>-x86_64-bios.bin.sha256` | SHA-256 checksum |

## Capabilities

The theorem kernel provides:

- **BIOS boot chain**: Custom pMBR → loader → 64-bit long mode kernel
- **PCI enumeration**: Full bus scan with PCIe bridge traversal (required for
  cloud VPS where NICs sit behind bridges)
- **Virtio-net**: Modern (MMIO) and legacy (port I/O) virtio network driver
- **TCP/IP**: smoltcp-based network stack with DHCP
- **Cloud VPS support**: /32 subnet workaround for Hetzner, AWS, GCP
- **Management server**: TCP port 2222 with token authentication
- **Serial console**: COM1 at 115200 baud
- **VGA text output**: Standard 80x25 text mode

## Limitations

This is a bare-metal kernel, not a full operating system:

- No filesystem persistence beyond the boot partition
- No SSH — management is via raw TCP on port 2222
- No user accounts or login shell
- BIOS only — no UEFI support in this release
- x86_64 only

## Next Steps

- See `docs/operator-guide/11-kernel-deployment.md` for production deployment,
  serial console access, and monitoring
- See `docs/first-boot-networking.md` for the DHCP network configuration
  contract
- See `docs/operator-guide/01-architecture-overview.md` for how the kernel
  fits into the theorem machine architecture
