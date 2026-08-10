# Disabling Broken Internal Surface Keyboard on Ubuntu (Wayland)

This guide isolates and disables a malfunctioning internal Microsoft Surface laptop keyboard on Ubuntu running a Wayland session. It uses an exclusive kernel-level `evdev` grab via a Python background service. This ensures ghost keypresses are blocked inside Ubuntu, while keeping the physical keyboard 100% active inside the GRUB menu for Windows dual-booting.

---

## Technical Specifications
* **Target OS:** Ubuntu (Wayland Session)
* **Device Name:** Microsoft Surface 045E:09AE Keyboard
* **Stable Device Path:** `/dev/input/by-path/platform-MSHW0250:00-event-kbd`
* **Method:** Exclusive Python-evdev Device Grab via systemd

---

## Important: Do Not Use `/dev/input/eventN`

Linux can assign different event numbers after every reboot. For example, on August 8, 2026, the Surface keyboard moved to `/dev/input/event1` while `/dev/input/event3` became the Surface touchpad. The old script consequently disabled the touchpad and left the keyboard active.

Always use the stable `by-path` symlink shown above. Verify it before changing or starting the service:

```bash
readlink -f /dev/input/by-path/platform-MSHW0250:00-event-kbd
grep -A 8 'Microsoft Surface 045E:09AE Keyboard' /proc/bus/input/devices
```

The resolved `/dev/input/eventN` value may change; that is expected. The `by-path` name remains tied to the keyboard.

---

## Step 1: Install Necessary Dependencies
Ensure the system has Python 3 and the appropriate event-device interface library installed:
```bash
sudo apt update && sudo apt install python3-pip python3-evdev -y
```

## Step 2: Create the Persistent Python Script
Create an executable Python script that targets the specific hardware event node and locks it out of the window manager using an exclusive grab loop.

1. Open the file editor:
   ```bash
   sudo nano /usr/local/bin/disable_surface_kbd.py
   ```
2. Paste the following script:
   ```python
   #!/usr/bin/env python3
   import signal

   from evdev import InputDevice


   KEYBOARD_PATH = "/dev/input/by-path/platform-MSHW0250:00-event-kbd"

   keyboard = InputDevice(KEYBOARD_PATH)
   print(f"Disabling {keyboard.name} ({keyboard.path})", flush=True)
   keyboard.grab()
   signal.pause()
   ```
3. Save and close (`Ctrl` + `O`, **Enter**, `Ctrl` + `X`).
4. Set execution permissions on the file:
   ```bash
   sudo chmod +x /usr/local/bin/disable_surface_kbd.py
   ```

## Step 3: Register the systemd Background Service
To prevent manual terminal execution or annoying `sudo` password popups at login, register the automation script as a system service. System services run after the GRUB menu loader but before desktop session initiation.

1. Open the service configuration path:
   ```bash
   sudo nano /etc/systemd/system/disable-keyboard.service
   ```
2. Paste the following unit file properties:
   ```text
   [Unit]
   Description=Disable Broken Surface Keyboard on Wayland
   After=multi-user.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/disable_surface_kbd.py
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
3. Save and close (`Ctrl` + `O`, **Enter**, `Ctrl` + `X`).

## Step 4: Reload and Enable the Automation Service
Force the system initialization daemon to scan the new service registry, enable it to launch automatically on every boot, and activate it immediately:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disable-keyboard.service
```

Confirm that the service grabbed the keyboard, not the touchpad:

```bash
systemctl is-active disable-keyboard.service
journalctl -u disable-keyboard.service -b --no-pager -n 10
```

The journal must contain a line similar to:

```text
Disabling Microsoft Surface 045E:09AE Keyboard (/dev/input/by-path/platform-MSHW0250:00-event-kbd)
```

---

## Important System Diagnostics & Commands
If you ever need to inspect, stop, or remove this setup in the future, use these commands:

* **Check Service Status:** `sudo systemctl status disable-keyboard.service`
* **Temporarily Stop the Block (Re-enable keyboard):** `sudo systemctl stop disable-keyboard.service`
* **Immediately Recover the Touchpad if the Wrong Device Is Grabbed:**
   ```bash
   sudo systemctl stop disable-keyboard.service
   gsettings set org.gnome.desktop.peripherals.touchpad send-events enabled
   ```
* **Restart After Verifying the Stable Path:** `sudo systemctl restart disable-keyboard.service`
* **Permanently Remove the Automation:**
  ```bash
  sudo systemctl disable disable-keyboard.service
  sudo rm /etc/systemd/system/disable-keyboard.service
  sudo rm /usr/local/bin/disable_surface_kbd.py
  sudo systemctl daemon-reload
  ```

> **Boot timing:** This is a system service and starts during boot, before desktop login. Ensure the external Bluetooth keyboard reconnects reliably at the login screen. Stopping the service restores the internal keyboard immediately.
