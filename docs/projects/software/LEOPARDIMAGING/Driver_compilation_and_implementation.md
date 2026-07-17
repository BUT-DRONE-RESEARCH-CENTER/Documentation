# Driver Guide: LI-IMX900-MIPI-FR-EO Camera Driver for Jetson Orin NX

**Copyright (c) 2021, Leopard Imaging Inc. All Rights Reserved.**

---

## 1. Overview

This driver guide describes the setup and installation process for the **LI-IMX900-MIPI-FR-EO V1.0** camera kit on the **NVIDIA Jetson Orin NX Developer Kit**.

### Key Features
* **Supported Camera:** 1 × LI-IMX900-MIPI-FR-EO V1.0 camera
* **Resolution & Frame Rate:** 2064×1552 @ 30fps
* **Driver Base Linux Version:** L4T R36.4.3 (JetPack 6.2)
* **Default Port:** The camera connects to the **CAM1** interface (maps to `/dev/video0`).

---

## 2. System Components & Hardware Requirements

| Category | Component Description | Quantity |
| :--- | :--- | :--- |
| **Platform** | NVIDIA Jetson Orin NX Developer Kit | 1 |
| **Camera** | LI-IMX900-MIPI-FR-EO V1.0 | 1 |
| **Cable** | FAW-1233 | 2 |
| **Adapter / Carrier Board** | LI-FPC22-IPEX-PI V1.0 | 1 |
| **Adapter / Carrier Board** | LI-MIPI-IPEX-20-30-ADP V1.0 | 1 |
| **Power Supply** | 19V DC Power Supply | 1 |

---

## 3. Revision History & Known Issues

### Revision History
| Revision | Description | Release Date | Author | Tested By |
| :--- | :--- | :--- | :--- | :--- |
| **20251015** | SVN version updates / Documentation review | 10/15/2025 | Jueming Lin | — |
| **20241015** | First Release | 10/15/2025 | Jueming Lin | — |

### Known Bugs / Issues
1. **LI-FPC22-IPEX-PI** requires a hardware rework.

---

## 4. Setup & Installation Procedure

### Step 1: Driver Installation (Host PC Setup)
Perform these steps on an **Intel x64 Host PC** running **Ubuntu 20.04 64-bit OS**.

1. Download the R36.4.3 Driver Package (BSP), Sample Root Filesystem, and Driver Package Sources from the NVIDIA Developer website:
   * [NVIDIA Jetson Linux 36.4.3 Page](https://developer.nvidia.com/embedded/jetson-linux-r3643)

2. Extract the files and assemble the root filesystem (`rootfs`) using the following commands:


```bash
# Extract the Driver Package
sudo tar xpf Jetson_Linux_R36.4.3_aarch64.tbz2
   
# Navigate into the extracted directory structure
cd Linux_for_Tegra/rootfs/
   
# Extract the Sample Root Filesystem
sudo tar xpf ../../Tegra_Linux_Sample-Root-Filesystem_R36.4.3_aarch64.tbz2
   
# Return to the Linux_for_Tegra root directory and apply binaries
cd ..
sudo ./apply_binaries.sh
   
# Install flash prerequisites
sudo ./tools/l4t_flash_prerequisites.sh
```

3. Flash the target Orin NX device:
```bash
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 \\
  -c tools/kernel_flash/flash_l4t_t234_nvme.xml \\
  -p "-c bootloader/generic/cfg/flash_t234_qspi.xml" \\
  --showlogs --network usb0 jetson-orin-nano-devkit internal
```



### Step 2: Device Configuration (Orin NX Target Setup)

1. Boot or reboot the Jetson Orin NX system and verify which Device Tree Blob (`.dtb`) file is being used inside the `/boot/dtb/` directory:
* **For Orin NX 16GB:** `kernel_tegra234-p3768-0000+p3767-0000-nv.dtb`
* **For Orin NX 8GB:** `kernel_tegra234-p3768-0000+p3767-0001-nv.dtb`


2. Edit `/boot/extlinux/extlinux.conf` to add the `FDT` and `OVERLAYS` paths. For example, if using the 16GB variant, add the following lines:
```text
FDT /boot/dtb/kernel_tegra234-p3768-0000+p3767-0000-nv.dtb
OVERLAYS /boot/tegra234-p3768-0000+p3767-0000-dynamic.dtbo
```


3. Copy the dynamic device tree overlay file (`.dtbo`) from the Leopard software package to the Jetson's `/boot` directory and reboot:
```bash
sudo cp tegra234-p3768-0000+p3767-0000-dynamic.dtbo /boot/
sudo reboot
```


4. Once rebooted, copy the camera kernel module (`nv_imx900.ko`) to the device and insert it:
```bash
sudo insmod nv_imx900.ko
```


*Note: Upon loading, `CAM1` should map successfully to `/dev/video0`.*

---

## 5. Running and Testing the Camera

### Method 1: NVIDIA Argus Application

1. Download the Multimedia API package from NVIDIA:
* [Jetson Multimedia API r36.4.3](https://developer.nvidia.com/embedded/L4T/r36_release_v4.3/Release/Jetson_Multimedia_API_r36.4.3_aarch64.tbz2)


2. Install dependencies on the Jetson Orin NX:
```bash
sudo apt-get update
sudo apt-get install cmake build-essential pkg-config libx11-dev libgtk-3-dev libexpat1-dev libjpeg-dev libgstreamer1.0-dev
```


3. Extract the downloaded source files:
```bash
tar -xvf jetson_multimedia_api.tar
```


4. Build and install the Argus camera sample application:
```bash
cd jetson_multimedia_api/argus/cmake
cmake ..
make
sudo make install
```


5. Execute the application to preview the stream:
```bash
argus_camera --device=0
```


### Method 2: V4L2 Control Tool (`v4l2-ctrl`)

Verify raw frame captures directly using standard video4linux utilities:

```bash
sudo apt-get update
sudo apt install v4l-utils

# Capture a test stream stream sequence
v4l2-ctl -V -v width=2064,height=1552,pixelformat=BA10 \\
  --set-ctrl bypass_mode=0 --set-ctrl sensor_mode=0 \\
  --stream-mmap --stream-count=100 -d /dev/video0
```

### Method 3: GStreamer Pipeline

Launch a hardware-accelerated video stream rendering to the screen display:

```bash
sudo apt-get update
sudo apt-get install nvidia-l4t-gstreamer gstreamer1.0-tools gstreamer1.0-alsa \\
  gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad \\
  gstreamer1.0-plugins-ugly gstreamer1.0-libav libgstreamer1.0-dev \\
  libgstreamer-plugins-base1.0-dev libgstreamer-plugins-good1.0-dev \\
  libgstreamer-plugins-bad1.0-dev

# Run pipeline
gst-launch-1.0 nvarguscamerasrc sensor-id=0 ! \\
  'video/x-raw(memory:NVMM), width=(int)2064, height=(int)1552, framerate=30/1' ! \\
  nvvidconv flip-method=0 ! 'video/x-raw, format=(string)I420' ! xvimagesink -e
```

### Method 4: `nvgstcapture` Tool

Alternatively, run the native NVIDIA high-level capture utility:

```bash
nvgstcapture-1.0
```

---

## 6. Driver Re-compilation Guide (Advanced)

Follow these instructions to modify or re-compile the camera kernel module from source code.

### Required Source Code & Toolchain Downloads

* **Kernel Source Code:** [public_sources.tbz2](https://www.google.com/search?q=https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.3/sources/public_sources.tbz2)
* **GCC Cross-Compilation Toolchain:** [aarch64--glibc--stable-2022.08-1.tar.bz2](https://www.google.com/search?q=https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/toolchain/aarch64--glibc--stable-2022.08-1.tar.bz2)

### Build Environment Setup (Host PC)

Perform the compilation on your Ubuntu 20.04 64-bit host machine:

1. Create a dedicated directory for the cross-compiler toolchain:
```bash
sudo mkdir /opt/aarch64--glibc--stable-final
```


2. Extract the toolchain tool to the newly created folder:
```bash
# Copy your downloaded aarch64--glibc--stable-final.tar.gz file to /opt/aarch64--glibc--stable-final first
sudo tar xpf aarch64--glibc--stable-final.tar.gz
```


3. Extract the kernel source package and apply the camera patch inside your working directory:
```bash
tar xpf public_sources.tbz2
cd Linux_for_Tegra/source

tar -xjf kernel_src.tbz2
tar -xjf kernel_oot_modules_src.tbz2
tar -xjf nvidia_kernel_display_driver_source.tbz2
```

4. Apply the camera patch inside your working directory:
```bash
# Apply the Leopard Imaging IMX900 patches
cd ~/leopard_jetson_configured/Linux_for_Tegra/source
```

```bash
# Test the Leopard Imaging IMX900 kernel patche
patch -p1 --dry-run -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_kernel.patch

# Apply the Leopard Imaging IMX900 kernel patche
patch -p1 -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_kernel.patch
```

```bash
# Test the Leopard Imaging IMX900 dtbs patch
patch -p1 --dry-run -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_dtbs.patch

# Apply the Leopard Imaging IMX900 dtbs patch
patch -p1 -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_dtbs.patch
```

5. Customizing Device Tree Configuration (Optional)
```bash 
# If needed update the .dtsi file:
nano ~/leopard_jetson_configured/Linux_for_Tegra/source/hardware/nvidia/t23x/nv-public/overlay/tegra234-camera-rbpcv3-imx900.dtsi
```


### Kernel & Module Compilation

1. Export the environment variables pointing to your cross-compiler and kernel headers:
```bash
export CROSS_COMPILE=/opt/aarch64--glibc--stable-2022.08-1/bin/aarch64-buildroot-linux-gnu-
export KERNEL_HEADERS=${HOME}/<source_code_dir>/Linux_for_Tegra/source/kernel/kernel-jammy-src
```


2. Execute the compilation suite inside the root source directory:
```bash
cd ${HOME}/<source_code_dir>/

# Clean any previous builds
make -C kernel clean
make clean

# Compile kernel, modules, and device trees
make -C kernel
make modules
make dtbs
```

### Output Files

Upon successful compilation, the required binaries will be available at the following output targets:

* **Device Tree Overlay:** `kernel-devicetree/generic-dts/dtbs/tegra234-p3737-0000+p3701-0000-dynamic.dtbo`
* **Camera Driver Module:** `nvidia-oot/drivers/media/i2c/nv_imx900.ko`
"""

---

### Build Environment Setup (NVIDIA Jetson)

Perform the compilation on an **Nvidia Jetson**:

1. Install Native Compilation Prerequisites:
```bash
sudo apt update
sudo apt install -y build-essential bc bison flex libssl-dev nano
```

2. Extract the camera drivers: 
```bash 
tar xpf LI-IMX900-MIPI-FR-EO_20251015.tar.xz
```

3. Extract the kernel source package: 
```bash
# Unpack the main Linux for Tegra source
tar xpf public_sources.tbz2
cd Linux_for_Tegra/source 
```

```bash
# Unpack the kernel packages
tar -xjf kernel_src.tbz2
tar -xjf kernel_oot_modules_src.tbz2
tar -xjf nvidia_kernel_display_driver_source.tbz2
```

4. Apply the camera patch inside your working directory:
```bash
# Apply the Leopard Imaging IMX900 patches
cd ~/leopard_jetson_configured/Linux_for_Tegra/source
```

```bash
# Test the Leopard Imaging IMX900 kernel patche
patch -p1 --dry-run -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_kernel.patch

# Apply the Leopard Imaging IMX900 kernel patche
patch -p1 -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_kernel.patch
```

```bash
# Test the Leopard Imaging IMX900 dtbs patch
patch -p1 --dry-run -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_dtbs.patch

# Apply the Leopard Imaging IMX900 dtbs patch
patch -p1 -i ~/leopard_jetson_configured/LI-IMX900-MIPI-FR-EO_20251015/patch/imx900_R36.4.3_Orin_NX_20251017_dtbs.patch
```

5. Customizing Device Tree Configuration (Optional)
```bash 
# If needed update the .dtsi file:
nano ~/leopard_jetson_configured/Linux_for_Tegra/source/hardware/nvidia/t23x/nv-public/overlay/tegra234-camera-rbpcv3-imx900.dtsi
```

### Kernel & Module Compilation

1. Export the environment variables pointing to your cross-compiler and kernel headers:
```bash
export KERNEL_HEADERS=${HOME}/leopard_jetson_configured/Linux_for_Tegra/source/kernel/kernel-jammy-src
```

2. Execute the compilation suite inside the root source directory:
```bash
# cd ${HOME}/<source code dir>/
# In this case:
cd ${HOME}/leopard_jetson_configured/Linux_for_Tegra/source
```

```bash
# Clean any previous builds
make -C kernel clean
make clean
```

```bash
# Compile kernel, modules, and device trees
make -C kernel -j$(nproc)
make modules -j$(nproc)
make dtbs -j$(nproc)
```

### Output Files

Upon successful compilation, the required binaries will be generated at the following output targets:

* **Device Tree Overlay:** `kernel-devicetree/generic-dts/dtbs/tegra234-p3737-0000+p3701-0000-dynamic.dtbo`
* **Camera Driver Module:** `nvidia-oot/drivers/media/i2c/nv_imx900.ko`

1. Copy files to created directory
```bash
mkdir -p compiled_files_IMX900

cp ~/leopard_jetson_configured/Linux_for_Tegra/source/nvidia-oot/drivers/media/i2c/nv_imx900.ko ~/compiled_files_IMX900

cp ~/leopard_jetson_configured/Linux_for_Tegra/source/kernel-devicetree/generic-dts/dtbs/tegra234-p3737-0000+p3701-0000-dynamic.dtbo ~/compiled_files_IMX900
```

2. Copying dtbo file to /boot directory
```bash
sudo cp ~/compiled_files_IMX900/tegra234-p3737-0000+p3701-0000-dynamic.dtbo /boot/tegra234-p3768-0000+p3767-0000-dynamic.dtbo
```

### Driver recompilation after .dtsi change 
```bash
cd ~/leopard_jetson_configured/Linux_for_Tegra/source
rm -rf kernel-devicetree/generic-dts/dtbs
make dtbs

# Copy new dtbo file to /boot 
sudo cp kernel-devicetree/generic-dts/dtbs/tegra234-p3768-0000+p3767-0000-dynamic.dtbo /boot/tegra234-p3768-0000+p3767-0000-dynamic.dtbo

# Reboot jetson 
sudo reboot
```