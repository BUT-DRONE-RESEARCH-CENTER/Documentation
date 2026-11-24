# Backup and Restore NVIDIA Jetson device

## 1. Introduction

This document describes how to create, archive, and restore system backups of NVIDIA Jetson devices using the official NVIDIA Backup and Restore tool.

The procedure is primarily intended for Jetson Orin NX and Orin Nano devices running JetPack 6.x with NVMe storage.

### Backup vs. clone

NVIDIA distinguishes between a backup and a clone:

- **Backup:** Copies the relevant storage partitions, including QSPI bootloader data and selected external storage devices.
- **Clone:** Copies only the APP filesystem partition.
- **Raw disk image:** A sector-by-sector copy of a storage device, typically created using `dd`.

For disaster recovery and duplication of complete Jetson systems, the Backup and Restore tool is preferred.

> [!CAUTION]
> ## WARNING – RISK OF PERMANENT DATA LOSS
>
> The backup and restore operations use the same script and differ primarily in a single command-line argument:
>
> - `-b` = BACKUP
> - `-r` = RESTORE
>
> Accidentally replacing `-b` with `-r`  can initiate a restore operation and overwrite the operating system, bootloader, applications, configuration, and user data on the connected Jetson.
>
> **DO NOT assume that the script will ask for confirmation before overwriting the device.**
>
> Before executing any command:
>
> - Verify whether the command contains `-b` or `-r`.
> - Verify which physical Jetson is connected.
> - Verify the selected board configuration.
> - Verify the selected storage device.
> - Verify the contents of the `/images` directory when doing RESTORE.
> - Verify the `/images` is empty or does not exist when doing BACKUP.
>
> **NEVER execute a restore operation on a device containing important data without creating and verifying a backup first.**
> Connect only ONE Jetson in Force Recovery Mode at a time.

## 2. Before you start

### 2.1 Hardware requirements

- NVIDIA Jetson device.
- Compatible carrier board.
- Linux host computer, preferably running a supported native Ubuntu installation (Ubuntu 22.04 for JetPack 6.x).
- USB data cable compatible with the carrier board's recovery USB port (for Waveshare boards it is USB-C).
- Stable power supply for the Jetson.
- Sufficient free storage space on the host computer.
- Access to the Force Recovery button or recovery pins (F-F jumper wire for Waveshare board).
- Optional external SSD for long-term backup storage.

Avoid USB hubs, virtual machines, and unreliable USB cables
if possible.

The exact Force Recovery procedure depends on the carrier board.

### 2.2 Identify the Jetson hardware

Run the following commands directly on the Jetson while the operating system is still running.

Check the device tree compatibility:

```bash
cat /proc/device-tree/compatible | tr '\0' '\n'
```

Check the board model:

```bash
cat /proc/device-tree/model
echo
```

Check the bootloader configuration:

```bash
cat /etc/nv_boot_control.conf
```

Pay particular attention to the following entries:

- `TNSPEC`
- `COMPATIBLE_SPEC`
- `TEGRA_BOOT_STORAGE`
- `TEGRA_OTA_BOOT_DEVICE`

Example:

```text
COMPATIBLE_SPEC 3767-000-0000--1--jetson-orin-nano-devkit-super-
TEGRA_BOOT_STORAGE nvme0n1
TEGRA_OTA_BOOT_DEVICE /dev/mtdblock0
```

This indicates that the device uses the Super configuration,
NVMe storage, and QSPI bootloader storage.

### 2.3 Identify the board name

The board name specifies the hardware and boot configuration used by the NVIDIA flashing tools.

Common board names:

| Board name | Configuration |
|---|---|
| `jetson-orin-nano-devkit` | Orin NX / Orin Nano standard |
| `jetson-orin-nano-devkit-super` | Orin NX / Orin Nano Super |
| `jetson-agx-orin-devkit` | AGX Orin |
| `jetson-agx-orin-devkit-industrial` | AGX Orin Industrial |

The board name is NOT the hostname of the device.

The available board configurations can also be listed from the `Linux_for_Tegra` directory:

```bash
ls -1 *.conf
```

Not every `.conf` file represents a standalone board target.

Third-party carrier boards may require manufacturer-specific BSP packages and board configuration files.

> [!IMPORTANT]
> Always use the board configuration appropriate for the original system and carrier board. Do not switch between standard and Super configurations during a backup simply to obtain higher performance.

### 2.4 Identify the JetPack and L4T version

Run on the Jetson:

```bash
cat /etc/nv_tegra_release
```

Example output:

```text
# R36 (release), REVISION: 4.4, BOARD: generic, EABI: aarch64
```

This corresponds to Jetson Linux (L4T) R36.4.4.

Alternatively:

- Check the installed L4T package:

```bash
dpkg-query -W nvidia-l4t-core
```

- Check the Jetpack package with `jtop`, if installed:

```bash
sudo jtop
```

Common JetPack versions:

| JetPack | Jetson Linux (L4T) |
|---|---|
| 6.0 | R36.3 |
| 6.1 | R36.4 |
| 6.2 | R36.4.3 |
| 6.2.1 | R36.4.4 |
| 6.2.1 with security updates | R36.4.7 |
| 6.2.2 | R36.5.0 |


### 2.5 Identify the storage device

Run on the Jetson:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Identify which device contains the root filesystem:

```bash
findmnt -no SOURCE /
```

Common storage device names:

| Device | Description |
|---|---|
| `nvme0n1` | First NVMe SSD |
| `nvme1n1` | Second NVMe SSD |
| `mmcblk0` | eMMC or SD card, depending on hardware |
| `sda` | SATA or USB storage device |
| `mtdblock0` | Memory technology device, typically QSPI |

For an Orin NX using an NVMe SSD, the usual storage
device argument is:

```text
nvme0n1
```

Use the entire storage device name, not an individual partition.

**Correct**:

```text
nvme0n1
```

**Incorrect** for a complete NVMe backup:

```text
nvme0n1p1
```

## 3. Prepare the host computer

All commands in the following sections must be executed on the Linux host computer, unless stated otherwise.

### 3.1 Download the correct Jetson Linux BSP

Preferably use the same L4T version and BSP configuration that were used to configure the original Jetson.

Official Jetson Linux archive:

https://developer.nvidia.com/embedded/jetson-linux-archive

Official JetPack archive:

https://developer.nvidia.com/embedded/jetpack-archive

There are two common methods of obtaining the BSP:

1. Manual download using `wget`.
2. NVIDIA SDK Manager.

### 3.2 Method A – Download using wget

The following example uses JetPack 6.2.1 / L4T R36.4.4.

Create a dedicated BSP directory:

```bash
mkdir -p ~/Jetson/BSP/R36.4.4
cd ~/Jetson/BSP/R36.4.4
```

Download the NVIDIA driver package:

```bash
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.4/release/Jetson_Linux_r36.4.4_aarch64.tbz2
```

Download the sample root filesystem:

```bash
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.4/release/Tegra_Linux_Sample-Root-Filesystem_r36.4.4_aarch64.tbz2
```

Extract the driver package:

```bash
tar -xjf Jetson_Linux_r36.4.4_aarch64.tbz2
```

Extract the sample root filesystem:

```bash
sudo tar -xjpf \
    Tegra_Linux_Sample-Root-Filesystem_r36.4.4_aarch64.tbz2 \
    -C Linux_for_Tegra/rootfs/
```

Enter the BSP directory:

```bash
cd Linux_for_Tegra
```

Install flashing prerequisites:

```bash
sudo ./tools/l4t_flash_prerequisites.sh
```

Apply NVIDIA binaries to the host-side sample root filesystem:

```bash
sudo ./apply_binaries.sh
```

These preparation commands do not flash the connected Jetson.

The backup and restore tools should now be available at:

```text
Linux_for_Tegra/tools/backup_restore/
```

> [!NOTE]
> Download URLs differ between L4T releases. Obtain the exact download links from the official NVIDIA release page instead of blindly substituting version numbers in existing URLs.

### 3.3 Special case – L4T R36.4.7

L4T R36.4.7 was released as a security update following R36.4.4.

A standalone public BSP package for R36.4.7 is not listed in the official Jetson Linux archive. Therefore simply replacing `36.4.4` with `36.4.7` in the download URLs will not work.

Please use `36.4.4` version for R36.4.7.

> [!WARNING]
> Cross-version backup and restore compatibility is not guaranteed. A successful backup does not automatically guarantee a successful restore.

Do not downgrade or upgrade the original Jetson solely to make its version match the available BSP before securing its existing data.

### 3.4 Method B – NVIDIA SDK Manager

> [!NOTE]
> NVIDIA SDK Manager has limited OS compatibility based on target Jetpack version.
> Please check [supported SDKs](https://developer.nvidia.com/w/sdkmanager/resources/supported_sdks/index.html) before downloading SDK manager.

> [!WARNING]
> NVIDIA SDK Manager requires the X11 display protocol. If the application window does not appear after launching, you may be running a Wayland session.
> See [how to switch from Wayland to X11 on Ubuntu](https://itsfoss.community/t/switching-from-wayland-to-x11-in-ubuntu-22-04/10723) for troubleshooting instructions.

Download NVIDIA SDK Manager:

https://developer.nvidia.com/sdk-manager

Install the downloaded Debian package:

```bash
sudo apt install ./sdkmanager_*.deb
```

Launch SDK Manager:

```bash
sdkmanager
```

For archived SDK versions, SDK Manager can also be started with:

```bash
sdkmanager --archived-versions
```

In SDK Manager:

1. Log in using your NVIDIA Developer account.
2. Select Jetson as the product category.
3. Select the appropriate target hardware.
4. Select the required JetPack version.
5. Select the Jetson Linux components.
6. Enable "Download now. Install later."
7. Download the required packages.
8. Extract and prepare the downloaded BSP as needed.

Download-only mode downloads the packages but may not create
a fully prepared `Linux_for_Tegra` directory.

Depending on the SDK Manager installation settings, an already
prepared BSP may be available under:

```text
~/nvidia/nvidia_sdk/JetPack_<version>_Linux_<target>/Linux_for_Tegra/
```

Downloaded packages are typically stored under:

```text
~/Downloads/nvidia/sdkm_downloads/
```

Locate an existing BSP installation:

```bash
find ~/nvidia -type d -name Linux_for_Tegra 2>/dev/null
```

> [!CAUTION]
> The purpose of this step is solely to download the BSP and flashing tools.
>
> **DO NOT allow SDK Manager to flash the original Jetson.** To prevent accidental flashing, keep the USB cable between the Jetson and the host computer disconnected throughout this step.
>
> Flashing may overwrite the existing system, resulting in irreversible data loss before a backup can be created.


### 3.5 Install required dependencies

Enter the `Linux_for_Tegra` directory:

```bash
cd ~/Jetson/BSP/R36.4.4/Linux_for_Tegra
```

Install the NVIDIA flashing prerequisites:

```bash
sudo ./tools/l4t_flash_prerequisites.sh
```

Install the NFS server if necessary:

```bash
sudo apt install nfs-kernel-server
```

Start the NFS server:

```bash
sudo systemctl start nfs-kernel-server
```

Verify that the service is running:

```bash
systemctl status nfs-kernel-server
```


### 3.6 Disable automatic mounting

Temporarily disable automatic mounting of external storage:

```bash
sudo systemctl stop udisks2.service
```

Automatic mounting may interfere with NVIDIA's backup
and restore operations.

After completing all backup or restore operations,
the service can be enabled again:

```bash
sudo systemctl start udisks2.service
```


### 3.7 Disable USB autosuspend

USB autosuspend can cause communication failures during
RCM boot and flashing.

Record the original value:

```bash
cat /sys/module/usbcore/parameters/autosuspend
```

Disable USB autosuspend:

```bash
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend
```

Verify:

```bash
cat /sys/module/usbcore/parameters/autosuspend
```

Expected output:

```text
-1
```

This change is temporary and normally resets after
rebooting the host computer.


### 3.8 Put the Jetson into Force Recovery Mode

The exact procedure depends on the carrier board.

For a Waveshare board, a typical procedure is:

1. Shut down the Jetson.
2. Connect the `FC REC` (3rd from the right) to `GND` (4th from the right)  
3. Connect the recovery USB port to the host PC.
4. Power on or reset the Jetson.

Verify that the Jetson is detected:

```bash
lsusb -d 0955:
```

Example for Jetson Orin NX 16 GB:

```text
Bus 001 Device 005: ID 0955:7323 NVIDIA Corp. APX
```

The APX device indicates that the Jetson is in recovery mode.

> [!IMPORTANT]
> Force Recovery Mode is not the same as a normally running
> Jetson with a USB network connection.


## 4. Backup

### 4.1 Prepare the backup directory

NVIDIA stores backup files in:

```text
Linux_for_Tegra/tools/backup_restore/images/
```

Inspect the directory before starting:

```bash
sudo ls -lah tools/backup_restore/images/
```

If the directory contains a previous backup, archive it before continuing (see Section 5).

Never mix files from multiple backups.


### 4.2 Backup command syntax

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e <storage-device> \
    -b <board-name>
```

Arguments:

| Argument | Description |
|---|---|
| `-b` | Create a backup |
| `-e` | Specify the storage device |
| `<board-name>` | Jetson board configuration |

> [!CAUTION]
> VERIFY THAT THE COMMAND CONTAINS `-b`.
>
> Do not accidentally use `-r`, as this initiates a restore operation instead of creating a backup.


### 4.3 Example – Orin NX standard configuration

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -b jetson-orin-nano-devkit
```


### 4.4 Example – Orin NX Super configuration

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -b jetson-orin-nano-devkit-super
```

### 4.5 Backup multiple storage devices

If supported by the installed backup tool, multiple storage devices can be specified using a colon separator:

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e mmcblk0:nvme0n1 \
    -b <board-name>
```

Only specify storage devices that actually exist on the target hardware. Massflash workflows may impose additional limitations on multiple external storage devices.

### 4.6 Save the backup log

It is recommended to save the complete terminal output.

For example, using Bash:

```bash
set -o pipefail

sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -b jetson-orin-nano-devkit-super \
    2>&1 | tee backup.log
```

Check the command exit status immediately afterward:

```bash
echo $?
```

An exit status of `0` indicates successful script completion.

The `pipefail` option ensures that a failure of the backup script is not hidden by a successful execution of `tee`.


### 4.7 What happens during the backup?

The host computer:

1. Communicates with the Jetson in Force Recovery Mode.
2. Uploads a temporary recovery environment.
3. Boots the recovery environment on the Jetson.
4. Establishes a USB network connection.
5. Accesses the selected storage devices.
6. Reads the relevant storage partitions and QSPI data.
7. Transfers the backup files to the host computer.

The resulting backup is stored in:

```text
tools/backup_restore/images/
```

Do not disconnect the Jetson or interrupt its power supply during the operation. Verify that the operation did not exit with an error.


### 4.8 Verify the backup

Inspect the backup directory:

```bash
sudo ls -lh tools/backup_restore/images/
```

Check the total size:

```bash
sudo du -sh tools/backup_restore/images/
```

A typical NVMe backup may contain files similar to:

```text
images/
├── nvpartitionmap.txt
├── QSPI0.img
├── gptmbr.img
├── gptbackup.img
├── nvme0n1p1.tar.gz
├── nvme0n1p2_bak.img
├── nvme0n1p3_bak.img
└── ...
```

The exact names and number of files depend on the L4T version,
partition layout, and backup tool implementation.

Inspect the partition mapping file if present:

```bash
sudo cat tools/backup_restore/images/nvpartitionmap.txt
```

Verify that:

- The backup script completed successfully.
- The partition mapping file exists.
- The referenced partition images exist.
- The expected NVMe filesystem backup exists.
- The expected QSPI backup exists.
- No transfer or storage errors were reported.

The absence of a normal Linux directory hierarchy is expected.

Directories such as `/home`, `/etc`, and `/usr` are stored
inside the filesystem backup rather than directly in `images/`.


## 5. Archiving backups

### 5.1 General principles

The NVIDIA backup tool uses a fixed working directory:

```text
tools/backup_restore/images/
```

It does not provide a complete backup version management system.

**Treat this directory as a temporary workspace containing only one backup at a time.**

> [!WARNING]
> Always archive the entire `images/` directory before starting another backup.
>
> Never mix partition images from different Jetsons or different backup dates.
>
> Never rename individual partition images, since their
> filenames may be referenced by the partition mapping file.


### 5.2 Recommended directory structure

```text
Jetson/
├── BSP/
│   ├── R36.4.4/
│   │   └── Linux_for_Tegra/
│   └── R36.5.0/
│       └── Linux_for_Tegra/
│
└── Backups/
    ├── Waveshare_IO_B/
    │   ├── 2026-11-24_R36.4.4_super_stable/
    │   │   ├── images/
    │   │   ├── metadata.txt
    │   │   ├── backup.log
    │   │   └── SHA256SUMS
    │   │
    │   └── 2026-12-01_R36.4.7_camera_update/
    │       ├── images/
    │       └── metadata.txt
    │
    └── Aerium_Helios/
        └── 2026-11-24_R36.4.7_standard/
            ├── images/
            ├── metadata.txt
            ├── backup.log
            └── SHA256SUMS
```

### 5.3 Archive the current backup

Run from `Linux_for_Tegra`.

Define the backup destination:

```bash
DEVICE_NAME="Waveshare_IO_B"
L4T_VERSION="R36.4.4"
DESCRIPTION="super_stable"

BACKUP_DIR="$HOME/Jetson/Backups/$DEVICE_NAME/$(date +%F_%H%M%S)_${L4T_VERSION}_${DESCRIPTION}"
```

Create a new destination:

```bash
mkdir -p "$(dirname "$BACKUP_DIR")"
mkdir "$BACKUP_DIR"
```

Move the complete backup directory:

```bash
sudo mv tools/backup_restore/images "$BACKUP_DIR/"
```


Verify that the files were moved successfully:

```bash
sudo ls -lh "$BACKUP_DIR/images/"
sudo du -sh "$BACKUP_DIR/images/"
```

Recreate the empty working directory:

```bash
sudo mkdir -p tools/backup_restore/images
```

The NVIDIA backup tool is now ready to create another backup.


## 6. Restore

> [!CAUTION]
> ## DESTRUCTIVE OPERATION
>
> THE RESTORE COMMAND USES `-r`.
>
> Before continuing:
>
> - Physically identify the target Jetson.
> - Disconnect every other Jetson.
> - Verify that the target device contains no unprotected data.
> - Create a backup of the target device if necessary.
> - Verify the selected backup directory.
> - Verify the board configuration and BSP.
> - Verify the target storage capacity.
>
> Do not proceed unless the target device is intended to
> be overwritten.


### 6.1 Check hardware compatibility

The target Jetson should preferably have:

- The same Jetson module.
- The same carrier board.
- A compatible board configuration.
- The same storage device type.
- An SSD with equal or greater usable capacity.
- Compatible security and bootloader configurations.

Restoration between devices with different Secure Boot or fuse configurations may require additional procedures.


### 6.2 Select the backup

Enter the corresponding `Linux_for_Tegra` directory.

Select the archived backup:

```bash
BACKUP_DIR="$HOME/Jetson/Backups/Jetson_A/<backup-name>"
```

Replace `<backup-name>` with the actual archived directory name.

Inspect the backup metadata:

```bash
cat "$BACKUP_DIR/metadata.txt"
```

Inspect the images:

```bash
sudo ls -lh "$BACKUP_DIR/images/"
```

Verify the backup checksums if available:

```bash
(
    cd "$BACKUP_DIR" || exit 1
    sudo sha256sum -c SHA256SUMS
)
```


### 6.3 Prepare the restore directory

Ensure that the NVIDIA working directory is empty:

```bash
sudo ls -lah tools/backup_restore/images/
```

If files are present, archive them before continuing.

Copy the selected backup into the working directory:

```bash
sudo mkdir -p tools/backup_restore/images
```

```bash
sudo rsync -a \
    "$BACKUP_DIR/images/" \
    tools/backup_restore/images/
```

Verify the destination:

```bash
sudo ls -lh tools/backup_restore/images/
```

Do not proceed if the working directory contains files
from another backup.

Copy the backup rather than moving it. This preserves
the original archived backup if the restoration fails.


### 6.4 Connect the target Jetson

1. Shut down the target Jetson.
2. Disconnect any other Jetson devices.
3. Connect the target Jetson to the host.
4. Enter Force Recovery Mode.
5. Verify the USB connection:

```bash
lsusb -d 0955:
```

Ensure that only one Jetson in recovery mode is detected.


### 6.5 Restore command syntax

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e <storage-device> \
    -r <board-name>
```


### 6.6 Example – Restore standard configuration

> [!CAUTION]
> The following command overwrites the target Jetson.

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -r jetson-orin-nano-devkit
```


### 6.7 Example – Restore Super configuration

> [!CAUTION]
> The following command overwrites the target Jetson.

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -r jetson-orin-nano-devkit-super
```

After the restore operation completes:

1. Exit Force Recovery Mode.
2. Reboot or power-cycle the Jetson as required.
3. Verify that the operating system boots.
4. Log in to the restored system.
5. Verify the hardware and software configuration.

For Jetsons equipped with CSI or USB cameras:

```bash
v4l2-ctl --list-devices
```

Additionally, verify:

- Camera drivers.
- Device tree overlays.
- Network interfaces.
- Wi-Fi connectivity.
- CUDA and TensorRT applications.
- Custom applications.
- Docker containers.
- Startup scripts.
- System services.
- Peripheral communication.

If the restored device will operate alongside the original,
check for duplicated machine-specific settings:

- Hostname.
- Static IP addresses.
- Machine ID.
- SSH host keys.
- Device identifiers used by applications.
- Network credentials and certificates.

Do not blindly regenerate application-specific identifiers
without checking their dependencies.


## 7. Common problems and troubleshooting

### 7.1 USB write timeout during RCM boot

Typical error:

```text
Entering RCM boot
Sending bct_br
ERROR: might be timeout in USB write.
Error: Return value 3
```

This occurs during the early recovery boot process,
before the backup environment accesses the SSD.

Possible causes:

- USB autosuspend.
- Unreliable USB cable.
- USB hub.
- Host USB controller.
- Insufficient or unstable power supply.
- Carrier board USB implementation.
- Incorrect recovery configuration.

First, disable USB autosuspend on the host:

```bash
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend
```

Disconnect the USB cable, power-cycle the Jetson,
and enter Force Recovery Mode again.

Reconnect the USB cable and retry.

This solution has been successfully tested on one of
our Jetson Orin NX devices.

If the problem persists:

- Try a different USB cable.
- Try another USB port.
- Remove any USB hubs.
- Try another Linux host.
- Verify the BSP and carrier board configuration.


### 7.2 USBFS memory limit

For USB transfer failures, particularly during larger
transfers, increasing the USBFS memory limit may help.

Check the current value:

```bash
cat /sys/module/usbcore/parameters/usbfs_memory_mb
```

Temporarily increase the limit:

```bash
echo 2048 | sudo tee /sys/module/usbcore/parameters/usbfs_memory_mb
```

This sets the limit to 2048 MB.

This is a troubleshooting workaround rather than a
mandatory configuration step.


### 7.3 Jetson not detected in recovery mode

Check USB detection:

```bash
lsusb -d 0955:
```

Check the USB topology:

```bash
lsusb -t
```

Monitor kernel messages:

```bash
sudo dmesg -wH
```

Possible causes:

- Jetson is not in Force Recovery Mode.
- Incorrect recovery USB port.
- Charge-only USB cable.
- Insufficient power supply.
- Incorrect recovery button sequence.
- USB connection failure.

Consult the carrier board documentation.


### 7.4 NFS server is not running

Check the NFS server:

```bash
systemctl status nfs-kernel-server
```

Start it:

```bash
sudo systemctl start nfs-kernel-server
```

If necessary, inspect exported directories:

```bash
sudo exportfs -v
```

The backup tool uses NFS to transfer files between
the temporary recovery environment and the host.

## 8. Additional workflows

### 8.1 Massflash

The NVIDIA backup tool also supports preparing a backup
for deployment to multiple Jetson devices.

This is useful for:

- Fleet deployment.
- Manufacturing.
- Reproducing a validated system configuration.
- Flashing several identical devices concurrently.

The general procedure is:

1. Create a backup and convert it into an initrd flash image.
2. Generate a massflash package using the backup image.
3. Connect the target devices in Force Recovery Mode.
4. Flash the target devices using the generated package.

Consult the backup restore README and initrd flash README
included with the specific BSP version.

For a small number of Jetsons, standard backup and restore
operations are usually simpler.


### 8.2 Raw disk images

A raw storage image can also be created using `dd`.

For example:

```bash
sudo dd if=/dev/nvme0n1 \
    of=jetson_nvme_backup.img \
    bs=4M \
    status=progress \
    conv=fsync
```

This should be performed from an offline or otherwise quiescent storage device to avoid filesystem inconsistencies.

A raw NVMe image contains the entire NVMe device, including its partition table and unused space.

However, it does not automatically contain the Jetson's separate QSPI memory.

NVIDIA also provides a raw-image restore workflow for supported eMMC configurations.

Do not assume that the eMMC raw-image restore procedure supports NVMe storage without verifying the documentation.

> [!CAUTION]
> Reversing the `if` and `of` arguments of `dd` can
> permanently overwrite the original storage device.


## 9. Restore normal host configuration

After completing all operations, restart the automatic mounting service:

```bash
sudo systemctl start udisks2.service
```

Restore the original USB autosuspend and USBFS memory settings if they were modified.

Alternatively, rebooting the host normally resets these temporary kernel parameter changes.

If the NFS server was started exclusively for this operation, and no other users or services require it, it can be stopped:

```bash
sudo systemctl stop nfs-kernel-server
```


## 10. Quick reference

### Identify the Jetson

Run on the Jetson:

```bash
cat /proc/device-tree/compatible | tr '\0' '\n'
cat /etc/nv_boot_control.conf
cat /etc/nv_tegra_release
uname -r
lsblk
```


### Prepare the host

Run on the host:

```bash
sudo systemctl stop udisks2.service
sudo systemctl start nfs-kernel-server

echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend
```


### Verify recovery mode

```bash
lsusb -d 0955:
```


### Backup

NON-DESTRUCTIVE BACKUP OPERATION:

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -b jetson-orin-nano-devkit-super
```

### Archive

```bash
BACKUP_DIR="$HOME/Jetson/Backups/$(date +%F_%H%M%S)"

mkdir -p "$BACKUP_DIR"

sudo mv tools/backup_restore/images "$BACKUP_DIR/"
sudo mkdir -p tools/backup_restore/images
```

### Restore

> [!CAUTION]
> DESTRUCTIVE OPERATION – VERIFY THE TARGET DEVICE FIRST.
>
> Ensure that the desired backup is present in the working `images/` directory before executing this command.

```bash
sudo ./tools/backup_restore/l4t_backup_restore.sh \
    -e nvme0n1 \
    -r jetson-orin-nano-devkit-super
```

## 12. References

### NVIDIA official documentation

- Jetson Linux Archive:
  https://developer.nvidia.com/embedded/jetson-linux-archive

- JetPack Archive:
  https://developer.nvidia.com/embedded/jetpack-archive

- Jetson Linux R36.4.4:
  https://developer.nvidia.com/embedded/jetson-linux-r3644

- Jetson Linux Flashing Support:
  https://docs.nvidia.com/jetson/archives/r36.4.4/DeveloperGuide/SD/FlashingSupport.html

- Jetson Linux Quick Start:
  https://docs.nvidia.com/jetson/archives/r36.4.4/DeveloperGuide/IN/QuickStart.html

- NVIDIA SDK Manager:
  https://developer.nvidia.com/sdk-manager

- NVIDIA SDK Manager Documentation:
  https://docs.nvidia.com/sdk-manager/

### Local BSP documentation

- `Linux_for_Tegra/tools/backup_restore/README_backup_restore.txt`
- `Linux_for_Tegra/tools/kernel_flash/README_initrd_flash.txt`

### Troubleshooting references

- NVIDIA Jetson AGX Orin FAQ:
  https://forums.developer.nvidia.com/t/jetson-agx-orin-faq/237459

- R36.4.7 security update announcement:
  https://forums.developer.nvidia.com/t/jetpack-releases-with-security-fixes-cve-2025-33182-cve-2025-33177/347680

- NVIDIA Developer Forums:
  https://forums.developer.nvidia.com/