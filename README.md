# WD My Book Live Duo: The OpenWrt Migration Guide

This guide details the complete process of replacing the deprecated factory firmware on a Western Digital My Book Live Duo with OpenWrt. This breathes new life into the hardware, providing a secure, modern, and highly customizable network-attached storage solution.

Due to the device's specific hardware architecture and proprietary U-Boot bootloader, this is not a standard OpenWrt installation. Please read the architectural notes before proceeding.

## ⚠️ Architectural Quirks & Prerequisites

- Reverse SATA Mapping: When facing the front LED of the NAS, the Right Bay (Drive B) maps to SATA 0, and the Left Bay (Drive A) maps to SATA 1.
- Boot Constraints: The internal U-Boot loader strictly looks for the OS on SATA 0. Furthermore, it requires an MBR (DOS) partition table on this drive, limiting the primary OS drive to a maximum of 2TB of usable space.
- Secondary Drive Panics: U-Boot aggressively scans both SATA ports. If it detects an unrecognized partition table on SATA 1 (Drive A), the bootloader will hang indefinitely (Solid Orange LED). We bypass this using a "superfloppy" raw filesystem approach.

## ✅ Requirements

- A Linux host machine (bare metal or VM with physical disk passthrough).
- A SATA-to-USB dock with a dedicated 12V power adapter (standard 5V USB cables cannot spin up 3.5" desktop drives).
- Two hard drives.

## 🚀 Phase 1: Flashing the Primary OS (External Setup)

Because the NAS cannot overwrite its own active bootloader safely, the primary drive must be flashed externally.

1. Remove both drives from the NAS.
2. Connect the drive intended for the Right Bay (Drive B) to your Linux host via the powered SATA dock.
3. Identify the drive's block device using `lsblk` (e.g., `/dev/sdX`).
4. Execute the following script on your Linux host. Update the variables at the top before running.

```bash
#!/bin/bash
# ⚠️ DANGER: Verify your target disk. Using the wrong letter will overwrite your host OS!
TARGET_DISK="/dev/sdX" 

# OpenWrt Configuration Variables
VERSION="23.05.5"
TARGET_DIR="apm821xx/sata"
TARGET_FILE="apm821xx-sata"

IMG="openwrt-${VERSION}-${TARGET_FILE}-wd_mybooklive-squashfs-factory.img"
URL="https://downloads.openwrt.org/releases/${VERSION}/targets/${TARGET_DIR}/${IMG}.gz"

echo "Downloading OpenWrt Factory Image v${VERSION}..."
wget "$URL" -O "${IMG}.gz"

echo "Extracting Image..."
gunzip "${IMG}.gz"

echo "Writing image to ${TARGET_DISK}..."
sudo dd if="${IMG}" of="${TARGET_DISK}" bs=1M status=progress
sudo sync

echo "Flash complete. You may disconnect the drive."
```

## 🔐 Phase 2: Initial Boot & Security

1. Insert the newly flashed drive strictly into the Right Bay (Drive B). Leave the Left Bay empty.
2. Connect Ethernet and power on the NAS. Wait for the LED to turn green.
3. Check your router's DHCP lease table for the NAS's IP address (hostname is usually OpenWrt).
4. SSH into the NAS:

```bash
ssh root@<NAS_IP_ADDRESS>
```

5. Set the root password to secure the device and enable the LuCI web interface:

```bash
passwd
```

## 🛠️ Phase 3: Bypassing the U-Boot Hardware Hang

To prevent the NAS from hanging during future cold boots, we must initialize the secondary drive while bypassing U-Boot entirely.

1. While the NAS is fully booted and running, hot-plug the secondary drive into the Left Bay (Drive A). Wait 20 seconds for it to spin up.
2. Verify the OS sees the drive by running `dmesg | tail -n 20`. You should see `sdb` registered.
3. Apply the "Superfloppy" fix. We will wipe the partition table headers entirely and format the raw block device. U-Boot will ignore the raw data during startup.

```bash
# Obliterate GPT/MBR partition headers
dd if=/dev/zero of=/dev/sdb bs=1M count=10

# Format the raw block device as ext4
mkfs.ext4 -F -m 0 /dev/sdb
```

## 📦 Phase 4: Storage Expansion (The 3MB Gap Fix)

The factory flash leaves empty space on Drive B, but places a 3MB gap at the beginning of the drive. We must manually partition the remaining space to skip this gap.

1. Open the partition manager: `fdisk /dev/sda`
2. Type `p` to print the partition table. Look at the `End` column for the last OS partition (usually `/dev/sda2`). Note this number (e.g., `245759`).
3. Create the new storage partition:
   - Type `n` (new), then `p` (primary), then `3` (partition number).
   - First sector: Type a number exactly 1 higher than the `End` sector you noted above (e.g., `245760`). Do not accept the default `2048`.
   - Last sector: Press Enter to accept the maximum default (capped at the 2TB MBR limit).
   - Type `w` to write and exit.
4. Format the new partition:

```bash
mkfs.ext4 -F -m 0 /dev/sda3
```

## 🧩 Phase 5: Installation & Persistent Mounting

Install required NAS dependencies and permanently map the drives using their unique UUIDs so they survive reboots.

```bash
# Update and install dependencies
opkg update
opkg install block-mount kmod-fs-ext4 e2fsprogs fdisk samba4-server rsync luci-app-samba4

# Create mount points
mkdir -p /mnt/disk1 /mnt/disk2

# Mount drives manually to establish UUID state
mount /dev/sda3 /mnt/disk1
mount /dev/sdb /mnt/disk2

# Snapshot current successful mounts to configuration
block detect > /etc/config/fstab
sed -i 's/option enabled '\''0'\''/option enabled '\''1'\''/g' /etc/config/fstab
service fstab reload

# Apply open permissions to the primary network share
chmod -R 777 /mnt/disk1
```

## 🔄 Phase 6: Network Shares & Rsync Mirroring

Instead of relying on hardware RAID (which is unsupported and risky on this device), we utilize an automated Rsync script to mirror data from Drive B to Drive A nightly.

Execute the following block to provision the Samba share and setup the Cron job:

```bash
# 1. Configure Global Samba Parameters
uci set samba4.@samba[0].workgroup='WORKGROUP'

# 2. Provision Share for Primary Drive (Drive B)
uci add samba4 sambashare
uci set samba4.@sambashare[-1].name='NAS_Storage'
uci set samba4.@sambashare[-1].path='/mnt/disk1'
uci set samba4.@sambashare[-1].read_only='no'
uci set samba4.@sambashare[-1].guest_ok='yes'
uci set samba4.@sambashare[-1].create_mask='0666'
uci set samba4.@sambashare[-1].dir_mask='0777'

# 3. Commit and Restart Samba
uci commit samba4
service samba4 enable
service samba4 restart

# 4. Create Automated Rsync Mirror Script
cat << 'EOF' > /root/mirror.sh
#!/bin/sh
# Mirrors Primary Disk (/mnt/disk1) to Secondary Disk (/mnt/disk2)
rsync -av --delete /mnt/disk1/ /mnt/disk2/
EOF
chmod +x /root/mirror.sh

# 5. Schedule Backup (Default: Daily at 2:00 AM)
echo "0 2 * * * /root/mirror.sh" >> /etc/crontabs/root
service cron enable
service cron restart
```

## ✅ Verification

Reboot the NAS via the SSH terminal (`reboot`).

- The device should successfully cold boot with both drives inserted.
- Run `df -h` to verify `/mnt/disk1` and `/mnt/disk2` are mounted with their expected capacities.
- The `NAS_Storage` folder will be visible on your local network, ready for read/write operations without permission errors.
