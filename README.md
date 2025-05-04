# Raspberry Pi Zero USB Ethernet Setup

*KEH - My AI buddy (Grok) created the bulk of this guide. With Raspberry Pi OS Bookworm, older simpler configurations didn't work.*  

This guide provides instructions to configure a Raspberry Pi Zero (or Zero W) as a USB Ethernet gadget, allowing it to appear as a virtual Ethernet device on a host computer (e.g., Windows) for SSH access over USB. The setup assigns a static IP (`192.168.7.2`) to the `usb0` interface and enables SSH on first boot.

## Prerequisites
- Raspberry Pi Zero.
- MicroSD card with a fresh Raspberry Pi OS installation (Lite recommended) using Raspberry Pi Imager. __Set your login credentials__ with the Imager. (They will appear encripted in the `firstrun.sh` file.)
- USB cable to connect the Pi Zero to the host.

## Configuration Steps

1. **Enable USB Gadget Mode**:
   - On the `/boot` partition of the MicroSD card (accessible from Windows), edit `config.txt`:
     ```bash
     # Add at the end
     dtoverlay=dwc2
     ```
   - Edit `cmdline.txt` in the same partition. After `rootwait`, add (space-separated):
     ```bash
     modules-load=dwc2,g_ether 
     ```

2. **Edit the First-Run Script**:
   - Edit the file named `firstrun.sh` in the `/boot` partition and add this code right before the `rm -f /boot/firstrun.sh`:
     ```bash
     raspi-config nonint do_hostname "raspberrypi"
     systemctl enable ssh
     systemctl start ssh
     # Configure usb0 static IP
     cat <<EOF >/etc/systemd/system/usb0-network.service
     [Unit]
     Description=Configure usb0 network interface
     After=network.target

     [Service]
     Type=oneshot
     ExecStart=/sbin/ip addr add 192.168.7.2/24 dev usb0
     ExecStart=/sbin/ip link set usb0 up
     RemainAfterExit=yes

     [Install]
     WantedBy=multi-user.target
     EOF
     systemctl enable usb0-network.service
     systemctl start usb0-network.service
     
     ```
   - On Windows, ensure your editor preserves UNIX line endings:
  

3. **Boot and Connect**:
   - Insert the SD card into the Pi Zero, connect it to the host computer via USB, and power it on. __Use the USB port, not the PWR port.__
   - Wait ~7 minutes for the boot and reboot and networking to complete.
   - On the host, the Pi should eventually appear as a network device `USB Ethernet/RNDIS Gadget` in Windows Device Manager.
   - Configure the virtual Ethernet connection with Network Connections to be on the same subnet (i.e. 192.168.7.1).

4. **Access the Pi**:
   - SSH to the Pi using:
     ```bash
     ssh <username>@raspberrypi.local
     ```
   - Alternatively, use the static IP:
     ```bash
     ssh <username>@192.168.7.2
     ```
   

## Notes
- If the Pi appears as a COM port (e.g., COM11) on Windows, update the driver in Device Manager:
  - Right-click the device, select `Update Driver`. The correct driver is `USB Ethernet/RNDIS Gadget` (Windows may list it as an Acer driver).
- The `firstrun.sh` script self-deletes after running to keep the system clean.
- The systemd service (`usb0-network.service`) ensures the `usb0` interface is configured with a static IP on every boot.
- If SSH fails, verify the USB connection and check that the `usb0` interface is up (`ip addr show usb0` on the Pi).
- It should be possible to use Internet Connection Sharing on Windows such that the RPi can access it directly. [TBD]

## Troubleshooting
- **No network device on host**: Ensure the USB cable is connected to the data port (not power-only), and use Windows Update to check for optional drivers, etc.
- **Cannot resolve `raspberrypi.local`**: Use the IP address (`192.168.7.2`) or install Bonjour.
