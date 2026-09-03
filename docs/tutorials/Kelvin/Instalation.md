# Installing Raspberry Pi OS on the Kelvin Carrier Board for CM5

This guide describes how to install Raspberry Pi OS on a **Raspberry Pi Compute Module 5 (CM5)** mounted on the **Kelvin carrier board**, and how to enable the UART console for debugging and development.

[Kelvin Carrier Board documentation](https://docs.aerium.co.il/Product%20Manuals/Kelvin/Kelvin-Revision%20B)

---

## 1. Flashing Raspberry Pi OS to the CM5

### 1.1 Required connections

Connect the following to the Kelvin carrier board:

* USB cable from the PC to the **USB OTG** port
* Power supply to the carrier board
* DIY USB-to-UART adapter to the **PWM/UART0** connector
* Ethernet cable/adapter, if required

> **Note:** The USB OTG connection is used to expose the CM5's storage to the host PC while the module is in recovery mode.

---

## 2. Install `rpiboot`

The `rpiboot` utility is required to put the CM5 into USB mass-storage mode.

Install the required dependencies:

```bash
sudo apt install git libusb-1.0-0-dev pkg-config build-essential
```

Clone the Raspberry Pi `usbboot` repository:

```bash
git clone --recurse-submodules --shallow-submodules --depth=1 \
    https://github.com/raspberrypi/usbboot
```

Enter the directory:

```bash
cd usbboot
```

Build the project:

```bash
make
```

Install `rpiboot`:

```bash
sudo make install
```

You can now run:

```bash
sudo rpiboot
```

---

## 3. Boot the CM5 into Recovery Mode

To flash the CM5, it must first be placed into **USB recovery mode**.

1. Make sure the CM5 is installed on the Kelvin carrier board.
2. Connect the USB OTG cable between the carrier board and your PC.
3. Press and hold the **FRC button** for approximately **3 seconds**.
4. While holding the button, apply power to the carrier board.
5. Release the button after a few seconds.

The CM5 should now be in recovery mode.

---

## 4. Expose the CM5 Storage

On the PC, run:

```bash
sudo rpiboot
```

`rpiboot` will initialize the CM5's USB boot interface.

After it completes, the CM5's storage should appear on the PC as a regular disk.

You can verify this using:

```bash
lsblk
```

> **Important:** Make sure you identify the correct storage device before writing an operating system to it.

---

## 5. Flash Raspberry Pi OS

Once the CM5 appears as a storage device:

1. Open **Raspberry Pi Imager**.
2. Select the desired Raspberry Pi OS image.
3. Select the CM5 storage device as the target.
4. Configure the OS settings if required.
5. Start the flashing process.
6. Wait until the process has completed successfully.
7. Disconnect power from the Kelvin carrier board.

The CM5 now has Raspberry Pi OS installed.

---

# 6. Enabling the UART Console

The following steps configure the CM5 so that the Linux console is available through the carrier board's UART interface.

## 6.1 Enter Recovery Mode Again

To modify the files on the CM5:

1. Power off the Kelvin carrier board.
2. Press and hold the **recovery/FRC button located above the CAN port** for approximately **3 seconds**.
3. Apply power while holding the button.
4. Release the button.
5. Connect the USB OTG cable to your PC.

Then run:

```bash
sudo rpiboot
```

The CM5 storage should once again appear as a disk on your PC.

---

## 6.2 Configure the Linux Kernel Command Line

Open the `cmdline.txt` file located on the CM5's **bootfs** partition.

Remove:

```text
quiet
```

Then add:

```text
console=ttyAMA0,115200
```

Save the file.

> **Note:** `cmdline.txt` is normally a single-line file. Avoid adding unnecessary line breaks.

---

## 6.3 Enable the UART Interfaces

Open:

```text
config.txt
```

on the **bootfs** partition.

Add the following lines to the end of the file:

```text
dtoverlay=uart0-pi5
dtoverlay=uart2-pi5
dtoverlay=uart4-pi5
enable_rp1_uart=1
enable_uart=1
```

Save the file.

These settings enable the relevant UART interfaces on the Raspberry Pi 5/CM5 platform.

---
# Acess terminal
I used puttty on port /dev/USB0 at baudrate 115200
# 7. Set a Password for the `pi` User

If the `pi` user does not have a password configured, one can be generated manually.

Run:

```bash
mkpasswd --method=SHA-512 --stdin
```

Enter the desired password when prompted.

The command will return a SHA-512 password hash similar to:

```text
$6$...
```

---

## 7.1 Modify `/etc/shadow`

Open the `shadow` file located on the CM5's **rootfs** partition:

```text
/etc/shadow
```

You will need root privileges to modify this file.

Locate the `pi` user entry. It may look similar to:

```text
pi:!:20416:0:99999:7:::
```

Replace the `!` with the generated password hash.

For example:

```text
pi:$6$D15Mwu952a/5tdB9$0O4BtJD1MS0FCuVEpmxRN71WiY9Na1v9rA4XTjAuQqkeE8wDnfAa3mJPCsj8RZCCAYpUlvA/HsR87ZZXadzOU/:20416:0:99999:7:::
```

Save the file.

> **Security warning:** The hash shown above is only an example. Generate your own password hash rather than reusing an example password/hash.

---

# 8. Enable SSH

Once Raspberry Pi OS has booted normally, SSH can be enabled using:

```bash
sudo raspi-config
```

Navigate to the SSH settings and enable the SSH server.

You can then connect to the CM5 remotely using SSH:

```bash
ssh pi@<CM5-IP-address>
```

---

# 9. Important Note About Recovery Mode

When the CM5 is in **USB recovery mode**, it is possible to access a limited Linux environment as the `root` user without being prompted for a password.

However, this environment is **not the Raspberry Pi OS installation that was flashed to the CM5**.

Recovery mode provides a minimal environment used for booting and accessing the CM5's storage. It should therefore not be confused with the normal Raspberry Pi OS environment.

---

# 10. Troubleshooting

### CM5 does not appear as a disk

Check the following:

* The CM5 is actually in recovery mode.
* The FRC/recovery button is being held for long enough.
* The USB cable supports data transfer.
* The cable is connected to the correct **USB OTG** port.
* `rpiboot` is running with `sudo`.
* The carrier board has sufficient power.

You can also check whether the USB device is detected by Linux:

```bash
lsusb
```

### `rpiboot` cannot access the USB device

Make sure the required dependencies are installed:

```bash
sudo apt install libusb-1.0-0-dev pkg-config build-essential
```

Then rebuild and reinstall `rpiboot`:

```bash
cd usbboot
make
sudo make install
```

### UART console does not work

Check:

1. `cmdline.txt` contains:

   ```text
   console=ttyAMA0,115200
   ```
2. `quiet` has been removed.
3. The UART overlays have been added to `config.txt`.
4. The USB-UART adapter is connected to the correct Kelvin carrier board pins.
5. The terminal application is configured for:

   * **Baud rate:** `115200`
   * **Data bits:** `8`
   * **Parity:** None
   * **Stop bits:** `1`
   * **Flow control:** None

A typical Linux terminal command for a serial device is:

```bash
screen /dev/ttyUSB0 115200
```

The actual device name may be different, such as `/dev/ttyACM0`.

---

# 11. References

* **Kelvin Carrier Board**
  [Aerium documentation](https://docs.aerium.co.il/Product%20Manuals/Kelvin/Kelvin-Revision%20B)
* **Raspberry Pi `config.txt`** [documentation](https://www.raspberrypi.com/documentation/computers/config_txt.html)
* **Raspberry Pi `usbboot`**[repository](https://github.com/raspberrypi/usbboot)
* **Forum post** [CM5 no console using buildroot](https://forums.raspberrypi.com/viewtopic.php?t=391396)

Post has been edited, reformated and translated by AI from original rought sketch.