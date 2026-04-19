# WD My Book Live Duo → OpenWrt Conversion Guide

This guide documents a full replacement of the deprecated factory firmware on a Western Digital My Book Live Duo with OpenWrt to create a secure and modern NAS base.

## Read First: Architectural Notes

This device is **not** a standard OpenWrt target flow.

- The board uses a legacy SoC/boot chain and a **vendor U-Boot layout**.
- Boot, kernel, and rootfs placement must match the device's expected NAND/SATA boot sequence.
- Factory rescue assumptions used by many consumer routers do not apply directly here.
- You must be prepared to recover with serial console + U-Boot commands if boot fails.

Because of this, treat the process as a controlled migration, not a generic `sysupgrade` install.

---

## 1) Scope and Goal

The goal is to:

1. Back up everything needed for rollback.
2. Replace stock firmware boot flow with an OpenWrt-compatible boot flow.
3. Boot OpenWrt reliably from the device storage.
4. Reconfigure the box as a NAS platform (shares, users, services, updates).

---

## 2) Prerequisites

Prepare before touching firmware:

- **Hardware**
  - WD My Book Live Duo
  - USB-TTL serial adapter (3.3V)
  - Ethernet connection to your LAN
  - Optional: UPS during flashing
- **Host tools**
  - `ssh`, `scp`, `rsync`
  - `screen` or `minicom`
  - `tftp` server (or other transfer method available in U-Boot)
  - checksum tools (`sha256sum`)
- **Files**
  - Correct OpenWrt artifacts for this hardware/boot approach (kernel + rootfs format expected by your boot chain)
  - U-Boot environment script/commands validated for this model

> Do **not** flash images intended for unrelated targets even if CPU family appears similar.

---

## 3) Back Up Before Migration (Mandatory)

From stock firmware shell/SSH:

1. Export current config and user data metadata.
2. Back up bootloader environment and partition information.
3. Save full copies of critical flash areas if accessible.
4. Record MAC addresses, hostname, users, and share definitions.

Example backup checklist:

- `/etc` and firmware config files
- Partition table / flash map output
- U-Boot env dump
- Data volume UUIDs and mount layout

Keep backups off-device.

---

## 4) Gain Reliable Recovery Access

1. Connect serial console.
2. Verify you can interrupt U-Boot autoboot.
3. Capture current environment (`printenv`) to a file.
4. Confirm you can manually boot a test kernel/initramfs before overwriting persistent boot entries.

If you cannot do these steps, stop here.

---

## 5) Stage OpenWrt Images

Transfer validated image files to a temporary location reachable from U-Boot/Linux:

- via network (`tftpboot`/`dhcp` flow), or
- via attached storage.

Validate checksums on both host and device.

---

## 6) Update Boot Flow for OpenWrt

Using serial/U-Boot:

1. Define/adjust boot arguments for OpenWrt rootfs.
2. Set correct load addresses and boot command sequence.
3. Save environment.
4. Test boot once without destructive writes when possible.

Typical variables to verify:

- `bootcmd`
- `bootargs`
- kernel/rootfs source (NAND/SATA/network)
- console settings for serial recovery

---

## 7) Write OpenWrt Kernel/Rootfs

Apply your model-specific write procedure for kernel and rootfs locations expected by your U-Boot setup.

After writing:

1. Re-check partition/device mapping.
2. Ensure rootfs format matches kernel expectations.
3. Reboot into OpenWrt from serial-attached session.

---

## 8) First Boot Tasks in OpenWrt

Immediately after first successful boot:

1. Set root password and disable insecure defaults.
2. Configure LAN/static address as needed.
3. Update package lists and install NAS services (`block-mount`, `samba4-server`, etc. as required).
4. Mount data volumes by UUID.
5. Recreate shares and access controls.
6. Configure scheduled backups/snapshots.

---

## 9) Validation Checklist

Confirm all of the following:

- Device survives reboot loops cleanly
- Data volumes auto-mount correctly
- SMB/NFS shares accessible with expected permissions
- Time sync, DNS, and package updates work
- No fallback to broken/deprecated stock firmware paths

---

## 10) Rollback Plan

Keep a tested rollback path:

1. Serial access available
2. Original env + image backups stored safely
3. Documented commands to restore boot variables and original firmware layout

If production data is involved, validate rollback once in a maintenance window.
