#AIC8800M80 (AX900) Installation Guide for Linux 

This repository provides an automated fix for AIC8800M80 (AX900) Wi-Fi 6 / Bluetooth USB adapters (normally from aliexpress) that get stuck in Mass Storage mode on Linux.

When plugged in, these devices default to a read-only virtual CD-ROM identified as ID 1111:1111 Pandora International Ltd 88M80.

This fix bypasses the USB subsystem and injects a specific 16-byte SCSI payload directly into the hardware controller, forcing the device to reboot into its Wi-Fi configuration (usually ID a69c:8d80).
Prerequisites

Before applying the fix, ensure your system has the required SCSI tools and the correct DKMS driver installed.

1. Install required tools:
You need sg3_utils to send raw SCSI commands.

sudo pacman -S sg3_utils lsscsi

2. Install the driver:
Install the specific DKMS driver for the aic8800 chipset from the AUR.

paru -S aic8800d80-wifi-bt-git-dkms

Installation (Automated Fix)

To automate the mode switch every time the device is plugged in, we use a udev rule that triggers the sg_raw command.

1. Enable the Generic SCSI Module

The payload requires the sg (SCSI generic) kernel module to bypass the read-only filesystem lock. Force this module to load on boot:

echo "sg" | sudo tee /etc/modules-load.d/sg.conf

(To apply this immediately to your current session, run sudo modprobe sg)

2. Add the Udev Rule

Create a udev rule that listens for the 1111:1111 device and injects the initialization payload.

Run this command to create the file

echo 'ACTION=="add", SUBSYSTEM=="scsi_generic", ATTRS{idVendor}=="1111", ATTRS{idProduct}=="1111", RUN+="/usr/bin/sg_raw /dev/%k FD 00 00 00 00 00 00 00 00 00 00 00 00 00 00 F2"' | sudo tee /etc/udev/rules.d/99-aic8800-switch.rules

3. Reload Udev Rules

Apply the new rule without rebooting:

sudo udevadm control --reload-rules

Usage: Unplug the USB adapter and plug it back in. Within 5–10 seconds, the device will process the payload, and reappear as a Wi-Fi/BT adapter. The DKMS driver will automatically initialize the network interfaces.

Manual Testing

If you prefer to test the payload before creating the udev rule:

    Ensure the sg module is loaded: sudo modprobe sg

    Find the generic SCSI node for the adapter: lsscsi -g (Look for LGX WIFI6 and note the /dev/sgX path).

    Inject the payload manually:

    sudo sg_raw /dev/sgX FD 00 00 00 00 00 00 00 00 00 00 00 00 00 00 F2

    Verify the mode switch with lsusb.
