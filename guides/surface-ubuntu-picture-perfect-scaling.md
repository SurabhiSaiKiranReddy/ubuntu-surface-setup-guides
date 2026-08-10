# Ubuntu Picture-Perfect Scaling: Setup, Calibration, and Rollback

This guide documents the display and application scaling configuration finalized on the 15-inch Intel Surface Laptop 4. It also provides a reusable method for calibrating another Ubuntu computer without blindly copying values that may not suit a different panel.

## System and safety scope

- Hardware: Microsoft Surface Laptop 4, 15-inch Intel model
- Native display: 2496 x 1664, 3:2, approximately 200 PPI
- OS when documented: Ubuntu 26.04 LTS
- Desktop: GNOME Shell 50.1 on Wayland
- Display mode: native 2496 x 1664 at approximately 59.984 Hz with variable refresh mode
- GNOME display scale: 133 1/3% (`1.3333333730697632`)
- GNOME text scale: `1.0`
- Page zoom: 100%; page zoom was deliberately not used to compensate for application scaling
- Application changes use per-user desktop-file overrides. Package-owned files under `/usr/share/applications` remain untouched.

Scaling does not disable physical pixels or lower the panel resolution. It changes how many physical pixels represent one logical UI pixel, reducing logical workspace in exchange for larger controls and text.

## Final configuration

| Application or layer | Final setting | Mechanism |
|---|---:|---|
| GNOME display | 1.333333 / 133 1/3% | GNOME Displays / Mutter |
| GNOME text | 1.0 | `org.gnome.desktop.interface text-scaling-factor` |
| Google Chrome | 1.125 multiplier | `--force-device-scale-factor=1.125` |
| Visual Studio Code | 1.125 multiplier | `--force-device-scale-factor=1.125` |
| VS Code URL handler | 1.125 multiplier | Same Electron flag |
| Firefox | 1.5 absolute scale | `layout.css.devPixelsPerPx=1.5` |
| Jellyfin Media Player | 1.125 multiplier | `QT_SCALE_FACTOR=1.125` |
| Remmina | 1.125 text/control approximation | `GDK_DPI_SCALE=1.125` |

For Chrome, VS Code, Firefox, and Jellyfin, the intended final effective scale is:

```text
GNOME scale x application multiplier
1.333333 x 1.125 = 1.5
```

Firefox is different: its preference is an absolute final device-pixel ratio, so Firefox uses `1.5`, not `1.125`.

Remmina uses GTK3. GTK3 does not provide precise fractional scaling for every widget through `GDK_SCALE`; `GDK_DPI_SCALE=1.125` primarily enlarges text and controls whose dimensions follow their text. Icons and the remote framebuffer are not multiplied in exactly the same manner.

## Why 1.125 was selected

The following Chrome and VS Code multipliers were evaluated:

| Multiplier | Effective scale over GNOME 1.3333 | Approximate logical width | Result |
|---:|---:|---:|---|
| 1.0 | 1.3333 | 1872 | Maximum workspace, but Chrome appeared too small |
| 1.10 | 1.4667 | 1702 | Good balance and visually comfortable |
| 1.125 | 1.5 | 1664 | Preferred balance; clear, crisp, and comfortable |
| 1.25 | 1.6667 | 1498 | Comfortable controls but excessive workspace loss |

The selected value has several advantages:

- It increases application text and controls by 12.5% over native GNOME application sizing.
- It retains considerably more workspace than 1.25.
- The combined scale is exactly 1.5, giving a regular mapping of three physical pixels for every two logical pixels.
- Chromium/Electron applications render high-resolution buffers before composition, so the simple ratio can produce stable-looking edges and text.
- GNOME is configured for RGB subpixel antialiasing with slight hinting, which gives text stronger edge definition on this LCD.
- The resulting 1664 x 1109 theoretical application workspace is comfortable on a physically large 15-inch 3:2 display.

The simple 1.5 ratio contributes to the result but does not guarantee identical quality on every GPU, compositor, toolkit, or panel.

## Workspace and scaling mathematics

### Native panel to GNOME 133 1/3%

```text
2496 / 1.333333 = 1872 logical pixels wide
1664 / 1.333333 = 1248 logical pixels high
```

Compared with native 100% logical sizing:

- Width retained: 75%; width loss: 25%
- Height retained: 75%; height loss: 25%
- Logical area retained: 56.25%; area loss: 43.75%

### Additional 1.125 application multiplier

```text
1872 / 1.125 = 1664 logical pixels wide
1248 / 1.125 = approximately 1109.3 logical pixels high
```

Compared with an application at multiplier 1.0 on the same GNOME desktop:

- Additional width loss: 208 logical pixels, or 11.11%
- Additional height loss: approximately 139 logical pixels, or 11.11%
- Additional logical-area loss: approximately 20.99%

### Combined result relative to native logical sizing

```text
1.333333 x 1.125 = 1.5 effective scale
2496 / 1.5 = 1664
1664 / 1.5 = approximately 1109.3
```

Compared with native 100% logical sizing:

- Width retained: 66.67%; width loss: 33.33%
- Height retained: 66.67%; height loss: 33.33%
- Logical area retained: 44.44%; area loss: 55.56%

GNOME's top panel, application toolbars, tabs, status bars, and the dock consume additional workspace. Those elements are not included in the theoretical dimensions above.

## 1. Configure GNOME at native resolution

Use **Settings > Displays**:

1. Select the panel's native resolution. For this Surface, use `2496 x 1664`.
2. Select `133 1/3%` scaling.
3. Keep the normal orientation and preferred refresh mode.
4. Apply the change and confirm it before the timeout.

Verify the saved monitor configuration:

```bash
sed -n '/<logicalmonitor>/,/<\/logicalmonitor>/p' ~/.config/monitors.xml
```

Expected Surface values include:

```xml
<scale>1.3333333730697632</scale>
<width>2496</width>
<height>1664</height>
```

Verify the fractional-scaling features:

```bash
gsettings get org.gnome.mutter experimental-features
```

Expected result:

```text
['scale-monitor-framebuffer', 'xwayland-native-scaling']
```

Keep GNOME text scaling at 1.0:

```bash
gsettings set org.gnome.desktop.interface text-scaling-factor 1.0
```

The finalized font-rendering settings were:

```bash
gsettings set org.gnome.desktop.interface font-antialiasing 'rgba'
gsettings set org.gnome.desktop.interface font-hinting 'slight'
gsettings set org.gnome.desktop.interface font-rgba-order 'rgb'
```

Do not assume RGB is correct for every panel. If a panel has a different subpixel layout or is OLED, grayscale antialiasing may look better.

## 2. Google Chrome at 1.125

Create a per-user copy of Chrome's package launcher:

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/google-chrome.desktop \
  ~/.local/share/applications/google-chrome.desktop
```

Add the following flags after `/usr/bin/google-chrome-stable` in every `Exec=` line:

```text
--ozone-platform=wayland --force-device-scale-factor=1.125
```

The three finalized commands are:

```ini
Exec=/usr/bin/google-chrome-stable --ozone-platform=wayland --force-device-scale-factor=1.125 %U
Exec=/usr/bin/google-chrome-stable --ozone-platform=wayland --force-device-scale-factor=1.125
Exec=/usr/bin/google-chrome-stable --ozone-platform=wayland --force-device-scale-factor=1.125 --incognito
```

Validate and refresh the launcher database:

```bash
desktop-file-validate ~/.local/share/applications/google-chrome.desktop
update-desktop-database ~/.local/share/applications
```

Fully exit Chrome through **Menu > Exit** before reopening it. Closing only the visible window may leave Chrome running in the background with its old flags.

Verify the running command:

```bash
pgrep -a -f '^/opt/google/chrome/chrome '
```

Keep Chrome page zoom at 100% while evaluating application scale.

### Chrome rollback

```bash
rm ~/.local/share/applications/google-chrome.desktop
update-desktop-database ~/.local/share/applications
```

Then fully exit and reopen Chrome. The package launcher will become active again.

## 3. Visual Studio Code at 1.125

Copy both VS Code launchers so normal launches and `vscode://` links use the same scale:

```bash
cp /usr/share/applications/code.desktop \
  ~/.local/share/applications/code.desktop
cp /usr/share/applications/code-url-handler.desktop \
  ~/.local/share/applications/code-url-handler.desktop
```

Add these flags immediately after `/usr/share/code/code` in every relevant `Exec=` line:

```text
--ozone-platform-hint=auto --force-device-scale-factor=1.125
```

Final commands:

```ini
Exec=/usr/share/code/code --ozone-platform-hint=auto --force-device-scale-factor=1.125 %F
Exec=/usr/share/code/code --ozone-platform-hint=auto --force-device-scale-factor=1.125 --new-window %F
Exec=/usr/share/code/code --ozone-platform-hint=auto --force-device-scale-factor=1.125 --open-url %U
```

Validate, refresh, and bind the URL handler:

```bash
desktop-file-validate \
  ~/.local/share/applications/code.desktop \
  ~/.local/share/applications/code-url-handler.desktop
update-desktop-database ~/.local/share/applications
xdg-mime default code-url-handler.desktop x-scheme-handler/vscode
```

Fully close every VS Code window and reopen VS Code from the dock. Confirm the running command:

```bash
pgrep -a -f '^/usr/share/code/code ' | grep -v -- '--type='
```

### VS Code rollback

```bash
rm ~/.local/share/applications/code.desktop
rm ~/.local/share/applications/code-url-handler.desktop
update-desktop-database ~/.local/share/applications
xdg-mime default code-url-handler.desktop x-scheme-handler/vscode
```

The final `xdg-mime` command still names the same desktop ID; after deletion, it resolves to the package-owned launcher.

## 4. Firefox at absolute scale 1.5

Firefox's preference is an absolute device-pixel ratio, not a multiplier over GNOME. To match the final effective scale used by Chrome and VS Code:

```text
1.333333 x 1.125 = 1.5
```

Configure Firefox:

1. Open `about:config`.
2. Accept the warning.
3. Search for `layout.css.devPixelsPerPx`.
4. Set it to `1.5`.
5. Keep normal page zoom at 100%.

The preference changes both Firefox's interface and webpage content.

### Firefox rollback

Reset `layout.css.devPixelsPerPx` in `about:config`, or set it to:

```text
-1.0
```

The value `-1.0` restores automatic system scaling. Do not set Firefox to `1.125` when attempting to reproduce this configuration; that would be an absolute scale and would make Firefox smaller than intended.

## 5. Jellyfin Media Player at 1.125

Jellyfin Media Player 1.12 uses Qt5. Qt supports a precise application multiplier through environment variables.

Create a per-user launcher:

```bash
cp /usr/share/applications/com.github.iwalton3.jellyfin-media-player.desktop \
  ~/.local/share/applications/com.github.iwalton3.jellyfin-media-player.desktop
```

Prefix every Jellyfin `Exec=` command with:

```text
env QT_SCALE_FACTOR=1.125 QT_SCALE_FACTOR_ROUNDING_POLICY=PassThrough
```

Example:

```ini
Exec=env QT_SCALE_FACTOR=1.125 QT_SCALE_FACTOR_ROUNDING_POLICY=PassThrough jellyfinmediaplayer
```

Apply the same prefix to fullscreen, windowed, desktop, and TV actions. Validate and refresh:

```bash
desktop-file-validate \
  ~/.local/share/applications/com.github.iwalton3.jellyfin-media-player.desktop
update-desktop-database ~/.local/share/applications
```

The UI scale does not increase video resolution. Video continues to use the available player window and mpv rendering path.

### Jellyfin rollback

```bash
rm ~/.local/share/applications/com.github.iwalton3.jellyfin-media-player.desktop
update-desktop-database ~/.local/share/applications
```

## 6. Remmina GTK3 approximation

The active Remmina installation was the Ubuntu APT package. A second Snap installation also existed; avoid mixing the two launchers because the Snap desktop ID does not use these overrides.

Remmina uses GTK3. Precise fractional scaling of all widgets is not available through `GDK_SCALE`, which expects integer-oriented widget scaling. The closest safe per-app adjustment is:

```text
GDK_DPI_SCALE=1.125
```

This enlarges text and text-sized controls. It does not identically scale every icon or the remote framebuffer.

Copy both package launchers:

```bash
cp /usr/share/applications/org.remmina.Remmina.desktop \
  ~/.local/share/applications/org.remmina.Remmina.desktop
cp /usr/share/applications/org.remmina.Remmina-file.desktop \
  ~/.local/share/applications/org.remmina.Remmina-file.desktop
```

Prefix normal launch, profile, kiosk, tray, connect, and edit commands with:

```text
env GDK_DPI_SCALE=1.125
```

Examples:

```ini
Exec=env GDK_DPI_SCALE=1.125 remmina-file-wrapper %U
Exec=env GDK_DPI_SCALE=1.125 /usr/bin/remmina --new
Exec=env GDK_DPI_SCALE=1.125 remmina-file-wrapper -c %U
Exec=env GDK_DPI_SCALE=1.125 /usr/bin/remmina -e %U
```

The persistent login applet must carry the same environment variable. In `~/.config/autostart/remmina-applet.desktop`, use:

```ini
Exec=env GDK_DPI_SCALE=1.125 remmina -i
```

Validate and refresh:

```bash
desktop-file-validate \
  ~/.local/share/applications/org.remmina.Remmina.desktop \
  ~/.local/share/applications/org.remmina.Remmina-file.desktop \
  ~/.config/autostart/remmina-applet.desktop
update-desktop-database ~/.local/share/applications
```

Fully quit Remmina, including its tray process, before testing. Verify the active process environment:

```bash
pid=$(pgrep -o -f '^/usr/bin/remmina( |$)')
tr '\0' '\n' < "/proc/$pid/environ" | grep '^GDK_DPI_SCALE='
```

Expected result:

```text
GDK_DPI_SCALE=1.125
```

Remmina's RDP resolution, dynamic-resolution behavior, view mode, and `scale=` profile field are separate from GTK UI scaling. Do not change those merely to enlarge Remmina's local toolbar text.

### Remmina rollback

```bash
rm ~/.local/share/applications/org.remmina.Remmina.desktop
rm ~/.local/share/applications/org.remmina.Remmina-file.desktop
```

Change the autostart command back to:

```ini
Exec=remmina -i
```

Then run:

```bash
update-desktop-database ~/.local/share/applications
```

Fully quit and reopen Remmina.

## Applications that do not need an override

The following applications were left on native GNOME scaling:

- Nautilus and GNOME utilities
- Proton VPN
- Extension Manager
- Trayscale
- LibreOffice using its GTK3 integration

These applications follow GNOME's scale and theme directly. Adding global `GDK_SCALE`, `GDK_DPI_SCALE`, or `QT_SCALE_FACTOR` variables would affect unrelated programs and can produce inconsistent sizing.

Optional candidates should be changed only if they visibly appear too small:

- VLC uses a Qt5 interface and could use the same `QT_SCALE_FACTOR=1.125` launcher prefix.
- Thunderbird uses Mozilla's Gecko platform and can use `layout.css.devPixelsPerPx=1.5`, like Firefox.

Do not apply an override merely because an application supports one.

## Reusable calibration method for another computer

Do not copy `1.125` automatically to another display. Use this sequence.

### Step 1: Preserve native resolution

Select the panel's native resolution first. Lowering resolution generally reduces sharpness and may alter the aspect ratio. Use fractional scaling instead of a non-native resolution whenever the compositor and application support it correctly.

### Step 2: Choose a comfortable GNOME scale

Start with the scales offered by **Settings > Displays**. The available choices depend on the active display mode and hardware.

Record:

```text
Physical width  = W
Physical height = H
System scale    = S
```

System logical workspace is:

```text
W / S by H / S
```

### Step 3: Test native application scaling

Before adding flags:

- Maximize the application.
- Use 100% page or document zoom.
- Make browser bookmark-bar visibility identical.
- Compare the same webpage or document.
- Check text weight, toolbar comfort, horizontal overflow, and how much content fits.

### Step 4: Test small multipliers

Useful application multiplier candidates are:

```text
1.05
1.10
1.125
1.15
1.20
1.25
```

Calculate:

```text
Effective scale = system scale x application multiplier
Logical width   = physical width / effective scale
Logical height  = physical height / effective scale
```

For a desired effective scale `T`, calculate a starting multiplier:

```text
application multiplier = T / system scale
```

Example for target 1.5 over GNOME 1.333333:

```text
1.5 / 1.333333 = 1.125
```

### Step 5: Prefer comfort over a mathematically attractive number

A simple ratio such as 1.5 can help produce regular sampling, but visual quality also depends on:

- Physical PPI and screen size
- LCD or OLED subpixel layout
- Wayland versus XWayland
- GPU and compositor behavior
- Toolkit and application rendering engine
- Font hinting and antialiasing
- Viewing distance and eyesight

Select the smallest multiplier that makes controls and text genuinely comfortable.

### Step 6: Use the correct mechanism for each toolkit

| Toolkit/application family | Preferred mechanism | Meaning of value |
|---|---|---|
| Chromium/Electron | `--force-device-scale-factor=F` | Additional application factor in this tested setup |
| Firefox/Thunderbird | `layout.css.devPixelsPerPx=T` | Absolute final device-pixel ratio |
| Qt5/Qt6 | `QT_SCALE_FACTOR=F` | Application multiplier |
| GTK3 | `GDK_DPI_SCALE=F` | Primarily font/text-sized-control multiplier |
| Native GNOME/GTK4 | Usually no override | Follow GNOME display scale |

### Step 7: Restart completely and verify

Many GUI applications retain a background process. A changed desktop file affects only newly started processes.

Useful checks:

```bash
pgrep -a -f 'chrome|code|firefox|remmina|jellyfin'
```

For environment-variable overrides:

```bash
tr '\0' '\n' < /proc/PID/environ | grep -E 'QT_SCALE_FACTOR|GDK_DPI_SCALE'
```

For desktop files:

```bash
desktop-file-validate ~/.local/share/applications/APP.desktop
update-desktop-database ~/.local/share/applications
```

## Maintenance after application updates

A user desktop file overrides the package file with the same desktop ID. This is safe and reversible, but it means future package changes to launcher actions are not copied automatically.

After a major Chrome, VS Code, Jellyfin, or Remmina update:

1. Compare the user launcher with the package launcher.
2. Preserve new package actions or MIME types if needed.
3. Reapply only the scale arguments.
4. Run `desktop-file-validate` again.

Example comparison:

```bash
diff -u /usr/share/applications/code.desktop \
  ~/.local/share/applications/code.desktop
```

Do not put application-specific scale variables in `/etc/environment`, `~/.profile`, or a global environment file unless every application truly needs the same factor.

## Quick final-state audit

```bash
printf '%s\n' '--- application scale overrides ---'
find ~/.local/share/applications ~/.config/autostart \
  -maxdepth 1 -type f -name '*.desktop' -print0 2>/dev/null |
  xargs -0 grep -HnE \
  'force-device-scale-factor|QT_SCALE_FACTOR|GDK_(DPI_)?SCALE'

printf '%s\n' '--- GNOME scale ---'
sed -n '/<logicalmonitor>/,/<\/logicalmonitor>/p' ~/.config/monitors.xml

gsettings get org.gnome.desktop.interface text-scaling-factor
gsettings get org.gnome.desktop.interface font-antialiasing
gsettings get org.gnome.desktop.interface font-hinting
gsettings get org.gnome.desktop.interface font-rgba-order
```

## Final recommendation for this Surface

Keep the following combination unless the hardware, compositor, or eyesight requirements change:

```text
Native panel mode:          2496 x 1664
GNOME display scale:        133 1/3%
GNOME text scale:           1.0
Chrome multiplier:          1.125
VS Code multiplier:         1.125
Firefox absolute scale:     1.5
Jellyfin Qt multiplier:     1.125
Remmina GTK DPI multiplier: 1.125
Normal page zoom:           100%
```

This configuration preserves the panel's native sharpness, gives Chrome and VS Code an exact 1.5 effective scale, retains substantially more workspace than the tested 1.25 application multiplier, and provides consistent visual comfort across the applications used on this machine.
