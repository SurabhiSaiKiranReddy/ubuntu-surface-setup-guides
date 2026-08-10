# Ubuntu Fractional Scaling Fix for Electron & Chromium Apps

When using fractional scaling (like **1.33**) on Ubuntu, Chromium-based browsers and Electron apps often appear blurry. This guide provides a clean, permanent fix by copying application launchers to your user directory and modifying them with Wayland and scaling flags so they stay sharp and won't get overwritten by system updates.

---

## 1. Google Chrome Fix

Run the following commands to copy the Chrome desktop launcher to your local directory and inject the Wayland scaling flags:

```bash
# Copy launcher to user directory
cp /usr/share/applications/google-chrome.desktop ~/.local/share/applications/

# Inject Wayland and scaling flags
sed -i 's|/usr/bin/google-chrome-stable|/usr/bin/google-chrome-stable --ozone-platform-hint=auto --force-device-scale-factor=1.33|g' ~/.local/share/applications/google-chrome.desktop
```

---

## 2. Visual Studio Code Fix

Run the following commands to apply the exact same crisp scaling behavior to VS Code:

```bash
# Copy launcher to user directory
cp /usr/share/applications/code.desktop ~/.local/share/applications/

# Inject Wayland and scaling flags
sed -i 's|/usr/share/code/code|/usr/share/code/code --ozone-platform-hint=auto --force-device-scale-factor=1.33|g' ~/.local/share/applications/code.desktop
```

---

## 3. Generic Instructions for Other Electron Apps

You can replicate this fix for almost any other Electron app (such as Discord, Slack, Spotify, or Obsidian) using this 3-step pattern.

### Step A: Identify the Executable Path & Desktop File
Look inside `/usr/share/applications/` to find the target `.desktop` file name and the executable string it uses inside the `Exec=` line.

### Step B: Run the Unified Template
Replace `APP_LAUNCHER.desktop` and `EXEC_PATH` with your specific application details:

```bash
# 1. Copy the desktop file locally
cp /usr/share/applications/APP_LAUNCHER.desktop ~/.local/share/applications/

# 2. Modify the executable path with scaling flags
sed -i 's|EXEC_PATH|EXEC_PATH --ozone-platform-hint=auto --force-device-scale-factor=1.33|g' ~/.local/share/applications/APP_LAUNCHER.desktop
```

### Core Flag Breakdown
* `--ozone-platform-hint=auto`: Forces the app to natively run on Wayland if available, eliminating display scaling blur.
* `--force-device-scale-factor=1.33`: Explicitly sets the layout rendering multiplier to match your Ubuntu system scaling exactly.
