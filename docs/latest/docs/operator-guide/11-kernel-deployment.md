# Kernel Deployment

This document covers production deployment of the theorem-kernel bare-metal
image: hardware requirements, deployment procedures, serial console access,
network configuration, health verification, and recovery.

For a fast first deploy, use `docs/kernel-quickstart.md` instead.

## Overview

The theorem kernel is a standalone x86_64 binary that boots directly from BIOS
without any host operating system. It provides PCI enumeration, virtio-net
networking, a TCP/IP stack, and a management server — all in a 126 KiB
compressed disk image.

This is architecturally distinct from the FreeBSD-based theoremOS deployment.
The kernel owns the entire machine from power-on. There is no userspace, no
filesystem layer, no process model. The kernel IS the runtime.

## Hardware Requirements

### Minimum

| Resource | Requirement |
|----------|-------------|
| Architecture | x86_64 |
| Firmware | BIOS or SeaBIOS (CSM mode) |
| RAM | 512 MiB |
| Disk | 64 MiB (raw image size) |
| Network | One virtio-net or compatible NIC |
| DHCP | Required — static IP not yet supported |

### Verified Platforms

| Platform | Firmware | NIC | Status |
|----------|----------|-----|--------|
| Hetzner CPX11 | SeaBIOS | virtio-net behind PCIe bridges | Verified |
| Hetzner CPX31 | SeaBIOS | virtio-net behind PCIe bridges | Verified |
| QEMU/KVM | SeaBIOS | virtio-net-pci | Verified |

### Known Incompatible

- UEFI-only machines (no CSM/legacy boot option)
- Hardware with non-virtio NICs (e1000, realtek, etc. — drivers not yet
  included)
- ARM/aarch64 machines

## Deployment Procedures

### Cloud VPS (Hetzner)

1. **Provision** a CPX11 or larger instance. Location does not matter.

2. **Enter rescue mode**: In the Hetzner Cloud console, go to Rescue →
   Enable rescue & power cycle. Select Linux 64-bit. Note the root password.

3. **SSH into rescue**:
   ```bash
   ssh root@<server-ip>
   ```

4. **Write the image**:
   ```bash
   VERSION="v0.6.1"
   curl -LO "https://github.com/Theorem-Governance/theorem-releases/download/${VERSION}/theorem-kernel-${VERSION}-amd64-direct-write.img.xz"
   curl -LO "https://github.com/Theorem-Governance/theorem-releases/download/${VERSION}/theorem-kernel-${VERSION}-amd64-direct-write.img.xz.sha256"
   sha256sum -c "theorem-kernel-${VERSION}-amd64-direct-write.img.xz.sha256"
   xz -d "theorem-kernel-${VERSION}-amd64-direct-write.img.xz"
   dd if="theorem-kernel-${VERSION}-amd64-direct-write.img" of=/dev/sda bs=1M
   ```

5. **Disable rescue mode** in the console and reboot. The machine will boot
   directly into the theorem kernel.

6. **Verify**: After 5-10 seconds, connect to the management server:
   ```bash
   nc <server-ip> 2222
   ```

### Bare Metal (Physical Hardware)

1. Write the image to a USB drive or target disk from any Linux/FreeBSD system:
   ```bash
   xz -d theorem-kernel-${VERSION}-amd64-direct-write.img.xz
   dd if=theorem-kernel-${VERSION}-amd64-direct-write.img of=/dev/sdX bs=1M
   ```

2. Ensure BIOS is configured to boot from the target disk.

3. Boot. The kernel prints diagnostics to both VGA (if connected) and COM1
   serial (115200 8N1).

### QEMU (Development / Testing)

```bash
qemu-system-x86_64 \
  -drive file=theorem-kernel-${VERSION}-amd64-direct-write.img,format=raw \
  -m 512M \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:2222 \
  -nographic
```

For bridged networking (machine gets a real IP):
```bash
qemu-system-x86_64 \
  -drive file=theorem-kernel-${VERSION}-amd64-direct-write.img,format=raw \
  -m 512M \
  -device virtio-net-pci,netdev=net0 \
  -netdev bridge,id=net0,br=br0 \
  -nographic
```

## Boot Sequence

The kernel boot chain is:

```
BIOS POST
  → pMBR (sector 0, 440 bytes)
    → loader.bin (GPT partition 1, BIOS boot)
      → THEOKERN.BIN loaded from FAT16 partition
        → 64-bit long mode entry
          → PCI bus enumeration
            → virtio-net driver init
              → DHCP lease acquisition
                → Management server on TCP :2222
```

Total boot time is typically 2-5 seconds from BIOS handoff to TCP listener
ready.

## Serial Console

The kernel outputs all diagnostics to COM1 at 115200 baud, 8N1. This is the
primary observability surface.

### Hetzner VNC Console

Hetzner Cloud provides a VNC console in the web UI. The kernel also outputs to
VGA text mode (80x25), so you can observe boot progress through the Hetzner
console tab.

### Physical Serial

Connect a null-modem cable to COM1. Use any terminal emulator:

```bash
# Linux
screen /dev/ttyS0 115200

# macOS
screen /dev/cu.usbserial-* 115200
```

### QEMU Serial

With `-nographic`, serial output goes to stdout. With `-serial mon:stdio`, you
get an interactive serial console.

## Network Configuration

The kernel acquires its IP address via DHCP. There is no static IP
configuration in this release.

### Cloud VPS Specifics

Cloud providers like Hetzner assign /32 addresses with a gateway that is not
on the same subnet. The kernel includes a workaround for this: it adds a /32
host route to the gateway, then uses that as the default route. This is
transparent — DHCP just works.

### Ports

| Port | Protocol | Service |
|------|----------|---------|
| 2222 | TCP | Management server |

The management server is the sole network surface. It accepts JSON-formatted
commands over raw TCP.

## Health Verification

### Connection Test

```bash
nc -z <machine-ip> 2222 && echo "UP" || echo "DOWN"
```

### Boot Attestation

The kernel computes SHA-256 measurements during boot:
- Kernel binary digest
- Activation manifest digest
- Init binary digest

These measurements are available through the management server for remote
attestation.

## Recovery

### Machine Won't Boot

1. Check that the firmware is set to BIOS (not UEFI)
2. Verify the disk image was written completely (`dd` exit status 0)
3. If using Hetzner, ensure rescue mode is disabled before rebooting

### Machine Boots But No Network

1. Check serial output for DHCP lease messages
2. Verify the VPS has a public IPv4 address assigned
3. Check that no firewall blocks outbound DHCP (ports 67/68)
4. If the machine is behind a bridge with no DHCP server, the kernel will
   retry indefinitely

### Re-imaging

To recover a machine, boot into rescue mode (or a USB live system) and re-run
the `dd` command. The image is self-contained — no external state to
restore.

## Disk Layout

The 64 MiB GPT image contains:

| Region | Offset | Size | Contents |
|--------|--------|------|----------|
| Sector 0 | 0 | 512 B | Protective MBR with pMBR bootstrap code |
| Sector 1 | 512 | 512 B | GPT header |
| Sectors 2-33 | 1 KiB | 16 KiB | GPT partition entries |
| Partition 1 | 34 sectors | 512 KiB | BIOS boot code (loader.bin) |
| Partition 2 | after p1 | ~63 MiB | FAT16: THEOKERN.BIN, INIT.ELF, ACTIVAT.JSN |

### FAT16 Partition Contents

| File | Purpose |
|------|---------|
| `THEOKERN.BIN` | Flat x86_64 kernel binary (~274 KiB) |
| `INIT.ELF` | Stub init binary (121 bytes) |
| `ACTIVAT.JSN` | Activation manifest (JSON) |

The loader reads all three files from the FAT16 partition. `THEOKERN.BIN` is
loaded at physical address 0x100000 and entered in 64-bit long mode.

## Upgrading

To upgrade a running machine:

1. Boot into rescue mode (Hetzner) or a USB live system
2. Download and write the new version's disk image
3. Reboot

There is no in-place upgrade mechanism for the kernel release. Each version
is a complete disk image replacement.

## Building from Source

The kernel can be built entirely on a Windows, Linux, or macOS workstation.
Prerequisites: `nasm`, `cargo` (with `x86_64-unknown-none` target),
`llvm-objcopy`, and Python 3.

```bash
# From the theorem-machine-kernel-bios repository
python3 scripts/build-kernel-image-local.py <tag> <output-dir>
```

This produces the disk image, compressed image, and checksums. See
`scripts/build-kernel-image-local.py` for the full build chain.
