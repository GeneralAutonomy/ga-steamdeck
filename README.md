# Steam Deck OLED: Ubuntu 24.04 Setup Guide

This guide provides step-by-step instructions for getting Ubuntu 24.04 LTS fully functional on the Steam Deck OLED without any Steam client dependencies.

## Hardware Requirements

- Steam Deck OLED (Galileo)

## Software Requirements

- Ubuntu 24.04 LTS
- OpenSD (Standalone Driver)

## Table of Contents

1. [Controller Setup (OpenSD)](#controller-setup-opensd)
2. [WiFi Connectivity (OLED Firmware Fix)](#wifi-connectivity-oled-firmware-fix)
3. [Desktop Optimization](#desktop-optimization)
4. [Default Browser Controls](#default-browser-controls)

---

## Controller Setup (OpenSD)

OpenSD allows you to use the joysticks and trackpads as a mouse without Steam.

### A. Installation

First, install the required dependencies:

```bash
sudo apt update && sudo apt install git cmake build-essential libevdev-dev libusb-1.0-0-dev libudev-dev -y
```

Clone the OpenSD repository and build it:

```bash
git clone https://codeberg.org/opensd/opensd.git
cd opensd && mkdir build && cd build
cmake .. && make
sudo make install
```

### B. OLED Hardware Authorization (Udev Rules)

OpenSD requires permission to access the OLED's unique hardware ID (1205). Create the udev rules file:

```bash
sudo bash -c 'cat <<EOF > /etc/udev/rules.d/60-opensd.rules
KERNEL=="hidraw*", ATTRS{idVendor}=="28de", ATTRS{idProduct}=="1102", MODE="0660", GROUP="opensd", TAG+="uaccess"
KERNEL=="hidraw*", ATTRS{idVendor}=="28de", ATTRS{idProduct}=="1201", MODE="0660", GROUP="opensd", TAG+="uaccess"
KERNEL=="hidraw*", ATTRS{idVendor}=="28de", ATTRS{idProduct}=="1205", MODE="0660", GROUP="opensd", TAG+="uaccess"
KERNEL=="uinput", SUBSYSTEM=="misc", OPTIONS+="static_node=uinput", TAG+="uaccess"
EOF'
```

Reload the udev rules to apply the changes:

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### C. Configuration and Service

Create the local configuration directory and copy the default configuration:

```bash
mkdir -p ~/.config/opensd
cp ~/opensd/data/config/config.ini ~/.config/opensd/
```

Enable and start the OpenSD background service:

```bash
systemctl --user daemon-reload
systemctl --user enable --now opensd
```

### D. Persist Across Reboots (Enable Linger)

By default, user services stop on logout and do not start at boot. Without lingering enabled, you would need to re-run `systemctl --user enable --now opensd` after every restart. Enable linger so the service auto-starts on boot:

```bash
loginctl enable-linger $USER
```

Verify linger is active and the service is enabled:

```bash
loginctl show-user $USER | grep Linger
systemctl --user is-enabled opensd
```

If the unit file is located in a tmpfs path (e.g. `/run/user/.../systemd/`), it will be lost on reboot. Move it to a persistent location:

```bash
mkdir -p ~/.config/systemd/user
# copy or symlink the opensd.service unit into ~/.config/systemd/user/
```

### E. Auto-Restart on First-Boot Crash

On some boots `opensdd` aborts with `terminate called without an active exception` (core-dump, status `6/ABRT`) the moment the user session starts — the service launches before the gamepad/uinput device is fully ready. The symptom is that you find yourself running `systemctl --user enable --now opensd` after every reboot to get the controller working.

The fix is a drop-in override that tells systemd to retry the service automatically until it succeeds:

```bash
mkdir -p ~/.config/systemd/user/opensd.service.d
cat > ~/.config/systemd/user/opensd.service.d/restart.conf <<'EOF'
[Service]
Restart=on-failure
RestartSec=3
StartLimitBurst=10
StartLimitIntervalSec=120
EOF
systemctl --user daemon-reload
```

Verify the override is applied:

```bash
systemctl --user show opensd.service -p Restart -p RestartUSec -p StartLimitBurst
# Expected: Restart=on-failure  RestartUSec=3s  StartLimitBurst=10
```

You can simulate the crash to confirm it auto-recovers without rebooting:

```bash
systemctl --user kill -s SIGABRT opensd.service
sleep 5
systemctl --user is-active opensd.service   # → active
```

---

## WiFi Connectivity (OLED Firmware Fix)

Ubuntu 24.04 does not ship with the OLED WiFi firmware. You must inject it manually.

### Step 1: Check if WiFi is Already Working

Before proceeding with firmware installation, verify if your WiFi is already functional:

```bash
ip link show | grep wlan
```

If you see a wireless interface listed and you can connect to WiFi networks, skip the firmware installation steps below and proceed to the Desktop Optimization section.

If no wireless interface is found or WiFi is not working, continue with the following steps.

### Step 2: Download the Firmware File

Download the `board-2.bin` file on another device. This file is required for the Steam Deck OLED's WiFi adapter to function properly.

### Step 3: Install the Firmware

Create the firmware directory and copy the file:

```bash
sudo mkdir -p /lib/firmware/ath11k/WCN6855/hw2.0/
sudo cp board-2.bin /lib/firmware/ath11k/WCN6855/hw2.0/
sudo chmod 644 /lib/firmware/ath11k/WCN6855/hw2.0/board-2.bin
```

### Step 4: Reload the WiFi Driver

Force the driver to reload with the new firmware:

```bash
sudo modprobe -r ath11k_pci && sudo modprobe ath11k_pci
```

After reloading the driver, your WiFi adapter should be functional.

---

## Desktop Optimization

### Display Orientation

1. Open Settings
2. Navigate to Displays
3. Set the orientation to Landscape

### Display Scaling

For better usability on the high-DPI display:

1. Open Settings
2. Navigate to Displays
3. Set the scaling to 150%

### On-Screen Keyboard

To enable the on-screen keyboard:

1. Open Settings
2. Navigate to Accessibility
3. Enable Screen Keyboard

### Conflict Fix: Cursor Stuttering

If you experience cursor stuttering, blacklist the default Steam driver:

```bash
echo "blacklist hid_steam" | sudo tee /etc/modprobe.d/blacklist-steam.conf
```

After creating this file, reboot your system for the change to take effect.

---

## Default Browser Controls (OpenSD)

The following input mappings are available by default when using OpenSD:

| Input | Action |
|-------|--------|
| Right Stick | Mouse Cursor |
| R2 | Left Click |
| L2 | Right Click |
| Left Stick | Vertical/Horizontal Scroll |
| Right Trackpad | Precision Mouse |
| Left Trackpad | High-speed Scroll |

---

## Troubleshooting

### OpenSD Service Not Starting

If the OpenSD service fails to start, check the service status:

```bash
systemctl --user status opensd
```

Review the logs for any error messages:

```bash
journalctl --user -u opensd -n 50
```

### OpenSD Crashes on Boot (status=6/ABRT)

If `journalctl --user -u opensd -b 0` shows `terminate called without an active exception` followed by `Main process exited, code=dumped, status=6/ABRT` immediately after login, the service is losing a startup race against the gamepad/uinput device. Apply the auto-restart override from [section E](#e-auto-restart-on-first-boot-crash) — systemd will retry until it succeeds.

### WiFi Not Working After Firmware Installation

If WiFi still does not work after installing the firmware:

1. Verify the firmware file is in the correct location:
   ```bash
   ls -l /lib/firmware/ath11k/WCN6855/hw2.0/board-2.bin
   ```

2. Check if the driver loaded correctly:
   ```bash
   dmesg | grep ath11k
   ```

3. Try rebooting the system to ensure all changes are applied.

### Controller Not Responding

If the controller does not respond:

1. Verify the udev rules are loaded:
   ```bash
   udevadm test /sys/class/misc/uinput
   ```

2. Check that you are in the `opensd` group:
   ```bash
   groups
   ```

3. If you are not in the group, add yourself:
   ```bash
   sudo usermod -aG opensd $USER
   ```
   Then log out and log back in for the group membership to take effect.

---

## Additional Resources

- OpenSD Project: https://codeberg.org/opensd/opensd
- Ubuntu 24.04 LTS Documentation: https://ubuntu.com/server/docs

---

## Notes

- This guide assumes you have already installed Ubuntu 24.04 LTS on your Steam Deck OLED
- Some steps require root privileges (sudo)
- After making system-level changes, a reboot may be required
- Keep your system updated with `sudo apt update && sudo apt upgrade`
