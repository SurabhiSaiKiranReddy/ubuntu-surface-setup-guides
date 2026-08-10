# GNOME Dock, Window Effects, and Top-Panel Customization

This guide documents the desktop appearance changes made on this Surface Laptop 4, starting with moving Ubuntu Dock to the bottom and ending with the translucent dock and custom semi-dark top panel.

## System and safety scope

- OS: Ubuntu 26.04
- Desktop: GNOME Shell 50.1 on Wayland
- Changes are limited to the current user's GNOME settings, extensions, Flatpaks, and theme files.
- No compositor was replaced.
- No GNOME compatibility validation was disabled.
- The translucent effects use static opacity, not blur, so they do not add a continuous blur-related GPU load.
- Newly installed extensions require a full logout and login on Wayland. Locking the screen is not sufficient.

## Verified final state

### Ubuntu Dock

The bottom dock is centered, compact, permanently visible, and translucent.

```text
dock-position          BOTTOM
extend-height          false
dock-fixed             true
intellihide            false
dash-max-icon-size     48
click-action           minimize-or-previews
transparency-mode      FIXED
background-opacity     0.72
```

The dock was initially configured with 40 px icons. After later GNOME logins, Ubuntu Dock reported its current value as 48 px, which is the value documented above.

### Installed desktop extensions

```text
Ubuntu Dock                         version 105
Compiz alike magic lamp effect     version 24
Just Perfection                    version 36
User Themes                        version 74 / package display 50.2
```

Extension UUIDs:

```text
ubuntu-dock@ubuntu.com
compiz-alike-magic-lamp-effect@hermes83.github.com
just-perfection-desktop@just-perfection
user-theme@gnome-shell-extensions.gcampax.github.com
```

All four extensions were active when this guide was written.

### Extension Manager

The graphical Extension Manager is installed as a per-user Flatpak:

```text
com.mattjakeman.ExtensionManager
```

Ubuntu's official `gnome-extensions-app` is also installed. Extension Manager provides the more complete browse, install, and preferences interface.

### Top-panel themes

Two custom GNOME Shell themes are available:

```text
~/.local/share/themes/Surface-Glass-Light/gnome-shell/gnome-shell.css
~/.local/share/themes/Surface-Glass-Semi-Dark/gnome-shell/gnome-shell.css
```

The selected theme is:

```text
Surface-Glass-Semi-Dark
```

Its panel uses:

- 30 px height
- charcoal `rgb(31, 33, 36)` background
- 94% opacity
- soft-white text and symbolic icons
- 16 px symbolic icons
- subtle hover and active states
- static CSS transparency without blur

The light theme remains available as a rollback or alternative.

## 1. Configure Ubuntu Dock

Move the dock to the bottom, stop it from spanning the full screen width, and use click-to-minimize or window previews:

```bash
gsettings set org.gnome.shell.extensions.dash-to-dock dock-position 'BOTTOM'
gsettings set org.gnome.shell.extensions.dash-to-dock extend-height false
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 48
gsettings set org.gnome.shell.extensions.dash-to-dock click-action 'minimize-or-previews'
```

The dock was briefly configured for intelligent auto-hide. The final decision was to keep it permanently visible:

```bash
gsettings set org.gnome.shell.extensions.dash-to-dock dock-fixed true
gsettings set org.gnome.shell.extensions.dash-to-dock intellihide false
```

Apply fixed translucency at 72% opacity:

```bash
gsettings set org.gnome.shell.extensions.dash-to-dock transparency-mode 'FIXED'
gsettings set org.gnome.shell.extensions.dash-to-dock background-opacity 0.72
```

Verify the dock:

```bash
schema='org.gnome.shell.extensions.dash-to-dock'
for key in \
  dock-position extend-height dock-fixed intellihide \
  dash-max-icon-size click-action transparency-mode background-opacity
do
  printf '%-24s ' "$key"
  gsettings get "$schema" "$key"
done

gnome-extensions info ubuntu-dock@ubuntu.com
```

## 2. Install Extension Manager

Flathub was already configured as a per-user Flatpak remote. Extension Manager was installed without system-wide changes:

```bash
flatpak install --user flathub com.mattjakeman.ExtensionManager
```

Verify it:

```bash
flatpak info --user com.mattjakeman.ExtensionManager
```

Open **Extension Manager** from the GNOME application grid to enable, disable, update, and configure extensions.

## 3. Install the Magic Lamp minimize effect

Compiz Alike Magic Lamp Effect version 24 was selected because that release explicitly supports GNOME 50.

```bash
extension_zip="$(mktemp --suffix=.shell-extension.zip)"

curl --fail --location --silent --show-error \
  'https://extensions.gnome.org/download-extension/compiz-alike-magic-lamp-effect@hermes83.github.com.shell-extension.zip?version_tag=69833' \
  --output "$extension_zip"

gnome-extensions install --force "$extension_zip"
rm -f "$extension_zip"
```

Log out and back in, then enable or verify it:

```bash
gnome-extensions enable \
  compiz-alike-magic-lamp-effect@hermes83.github.com

gnome-extensions info \
  compiz-alike-magic-lamp-effect@hermes83.github.com
```

The effect applies only while minimizing or restoring windows. Its preferences are available under **Extension Manager > Installed > Compiz alike magic lamp effect**.

## 4. Install Just Perfection

Just Perfection version 36 was selected because it explicitly supports GNOME 50.

```bash
extension_zip="$(mktemp --suffix=.shell-extension.zip)"

curl --fail --location --silent --show-error \
  'https://extensions.gnome.org/download-extension/just-perfection-desktop@just-perfection.shell-extension.zip?version_tag=68110' \
  --output "$extension_zip"

gnome-extensions install --force "$extension_zip"
rm -f "$extension_zip"
```

After a Wayland logout and login:

```bash
gnome-extensions enable just-perfection-desktop@just-perfection
gnome-extensions info just-perfection-desktop@just-perfection
```

During setup, a 30 px panel, tighter padding, and a right-positioned clock were tested through Just Perfection. After later relogins, these numeric settings returned to their defaults:

```text
panel-size                     0
panel-button-padding-size      0
panel-indicator-padding-size   0
panel-icon-size                0
clock-menu-position            0 (center)
clock-menu-position-offset     0
```

The current 30 px panel and 16 px icons are therefore supplied by the selected custom Shell theme, not by active Just Perfection overrides. Just Perfection remains installed for optional UI customization.

## 5. Install User Themes

User Themes version 74 was selected because it explicitly supports GNOME 50.

```bash
extension_zip="$(mktemp --suffix=.shell-extension.zip)"

curl --fail --location --silent --show-error \
  'https://extensions.gnome.org/download-extension/user-theme@gnome-shell-extensions.gcampax.github.com.shell-extension.zip?version_tag=71200' \
  --output "$extension_zip"

gnome-extensions install --force "$extension_zip"
rm -f "$extension_zip"
```

Log out and back in, then enable it:

```bash
gnome-extensions enable \
  user-theme@gnome-shell-extensions.gcampax.github.com
```

The extension loads custom Shell themes from the user's theme directories. It changes GNOME Shell elements such as the top panel; it does not change the GTK application theme.

## 6. Custom top-panel themes

The light theme imports Ubuntu's light Yaru Shell stylesheet:

```css
@import url("file:///usr/share/gnome-shell/theme/Yaru/gnome-shell.css");
```

The semi-dark theme must import Yaru Dark so Quick Settings, the calendar, notifications, and other panel popups inherit dark styling:

```css
@import url("file:///usr/share/gnome-shell/theme/Yaru-dark/gnome-shell.css");
```

Importing the light Yaru stylesheet in the semi-dark theme makes the top panel dark but leaves the battery, Wi-Fi, Bluetooth, and Quick Settings popup white.

The light variant uses a light background with dark text and symbolic icons. The semi-dark variant uses a charcoal background with soft-white text and icons.

The semi-dark panel's current background declarations are:

```css
background-color: rgba(31, 33, 36, 0.94) !important;
```

This declaration appears for the normal panel and its overview/login variants.

### Recreate the theme files

Create both theme directories:

```bash
mkdir -p \
  ~/.local/share/themes/Surface-Glass-Light/gnome-shell \
  ~/.local/share/themes/Surface-Glass-Semi-Dark/gnome-shell
```

The complete light theme file at `~/.local/share/themes/Surface-Glass-Light/gnome-shell/gnome-shell.css` is:

```css
@import url("file:///usr/share/gnome-shell/theme/Yaru/gnome-shell.css");

/* Light translucent top panel over the current Yaru shell theme. */
#panel {
  background-color: rgba(246, 247, 249, 0.90) !important;
  color: #202124 !important;
  height: 30px;
  box-shadow: inset 0 -1px 0 rgba(32, 33, 36, 0.14),
              0 1px 6px rgba(32, 33, 36, 0.10) !important;
}

#panel:overview,
#panel.unlock-screen,
#panel.login-screen {
  background-color: rgba(246, 247, 249, 0.90) !important;
}

#panel .panel-button,
#panel .panel-button.clock-display {
  color: #202124 !important;
  font-weight: 500 !important;
}

#panel .system-status-icon,
#panel .panel-button StIcon,
#panel .panel-button .messages-indicator {
  color: #202124 !important;
  icon-size: 16px;
}

#panel .panel-button#panelActivities .workspace-dot {
  background-color: #202124 !important;
}

#panel .panel-button:focus,
#panel .panel-button:hover,
#panel .panel-button.clock-display:focus .clock,
#panel .panel-button.clock-display:hover .clock {
  background-color: rgba(32, 33, 36, 0.10) !important;
  box-shadow: none !important;
}

#panel .panel-button:active,
#panel .panel-button:checked,
#panel .panel-button.clock-display:active .clock,
#panel .panel-button.clock-display:checked .clock {
  background-color: rgba(32, 33, 36, 0.17) !important;
  box-shadow: none !important;
}

#panel .panel-button.screen-recording-indicator,
#panel .panel-button.screen-sharing-indicator,
#panel .panel-button.screen-recording-indicator StIcon,
#panel .panel-button.screen-sharing-indicator StIcon {
  color: white !important;
}
```

The complete semi-dark theme file at `~/.local/share/themes/Surface-Glass-Semi-Dark/gnome-shell/gnome-shell.css` is:

```css
@import url("file:///usr/share/gnome-shell/theme/Yaru-dark/gnome-shell.css");

/* Semi-dark translucent top panel over the current Yaru shell theme. */
#panel {
  background-color: rgba(31, 33, 36, 0.94) !important;
  color: #f2f3f5 !important;
  height: 30px;
  box-shadow: inset 0 -1px 0 rgba(255, 255, 255, 0.10),
              0 1px 6px rgba(0, 0, 0, 0.22) !important;
}

#panel:overview,
#panel.unlock-screen,
#panel.login-screen {
  background-color: rgba(31, 33, 36, 0.94) !important;
}

#panel .panel-button,
#panel .panel-button.clock-display {
  color: #f2f3f5 !important;
  font-weight: 500 !important;
}

#panel .system-status-icon,
#panel .panel-button StIcon,
#panel .panel-button .messages-indicator {
  color: #f2f3f5 !important;
  icon-size: 16px;
}

#panel .panel-button#panelActivities .workspace-dot {
  background-color: #f2f3f5 !important;
}

#panel .panel-button:focus,
#panel .panel-button:hover,
#panel .panel-button.clock-display:focus .clock,
#panel .panel-button.clock-display:hover .clock {
  background-color: rgba(255, 255, 255, 0.12) !important;
  box-shadow: none !important;
}

#panel .panel-button:active,
#panel .panel-button:checked,
#panel .panel-button.clock-display:active .clock,
#panel .panel-button.clock-display:checked .clock {
  background-color: rgba(255, 255, 255, 0.20) !important;
  box-shadow: none !important;
}

#panel .panel-button.screen-recording-indicator,
#panel .panel-button.screen-sharing-indicator,
#panel .panel-button.screen-recording-indicator StIcon,
#panel .panel-button.screen-sharing-indicator StIcon {
  color: white !important;
}
```

### Switch between the themes

Set the User Themes schema path:

```bash
uuid='user-theme@gnome-shell-extensions.gcampax.github.com'
schema_dir="$HOME/.local/share/gnome-shell/extensions/$uuid/schemas"
schema='org.gnome.shell.extensions.user-theme'
```

Select the semi-dark theme:

```bash
gsettings --schemadir "$schema_dir" set "$schema" \
  name 'Surface-Glass-Semi-Dark'
```

Select the light theme:

```bash
gsettings --schemadir "$schema_dir" set "$schema" \
  name 'Surface-Glass-Light'
```

Verify the selection:

```bash
gsettings --schemadir "$schema_dir" get "$schema" name
```

### Reload an edited theme

If a selected theme file is edited, force a live reload by switching away and back:

```bash
gsettings --schemadir "$schema_dir" set "$schema" \
  name 'Surface-Glass-Light'

gsettings --schemadir "$schema_dir" set "$schema" \
  name 'Surface-Glass-Semi-Dark'
```

## Health checks

List enabled extensions:

```bash
gnome-extensions list --enabled
```

Inspect an extension:

```bash
gnome-extensions info just-perfection-desktop@just-perfection
gnome-extensions info compiz-alike-magic-lamp-effect@hermes83.github.com
gnome-extensions info user-theme@gnome-shell-extensions.gcampax.github.com
gnome-extensions info ubuntu-dock@ubuntu.com
```

Check recent Shell extension or theme errors:

```bash
journalctl --user -b --no-pager | \
  grep -Ei 'gnome-shell.*(extension|theme).*(error|exception)'
```

No output is the desired result.

Confirm GNOME still enforces extension version compatibility:

```bash
gsettings get org.gnome.shell disable-extension-version-validation
```

The expected value is `false`.

## Rollback

### Return Ubuntu Dock to its schema defaults

```bash
schema='org.gnome.shell.extensions.dash-to-dock'

for key in \
  dock-position extend-height dock-fixed intellihide \
  dash-max-icon-size click-action transparency-mode background-opacity
do
  gsettings reset "$schema" "$key"
done
```

This restores Ubuntu's packaged defaults, which may differ between Ubuntu releases.

### Disable individual extensions

```bash
gnome-extensions disable \
  compiz-alike-magic-lamp-effect@hermes83.github.com

gnome-extensions disable just-perfection-desktop@just-perfection

gnome-extensions disable \
  user-theme@gnome-shell-extensions.gcampax.github.com
```

Disabling User Themes returns GNOME Shell to its packaged Yaru theme. Ubuntu Dock remains independent.

### Remove the user extensions completely

Disable an extension before uninstalling it:

```bash
gnome-extensions uninstall \
  compiz-alike-magic-lamp-effect@hermes83.github.com

gnome-extensions uninstall just-perfection-desktop@just-perfection

gnome-extensions uninstall \
  user-theme@gnome-shell-extensions.gcampax.github.com
```

Log out and back in after removing Shell extensions on Wayland.

### Remove the custom themes

First disable User Themes or select another valid theme. Then remove only these two user-owned directories:

```bash
rm -rf \
  ~/.local/share/themes/Surface-Glass-Light \
  ~/.local/share/themes/Surface-Glass-Semi-Dark
```

### Remove Extension Manager

```bash
flatpak uninstall --user com.mattjakeman.ExtensionManager
```

The official GNOME Extensions application remains available as `gnome-extensions-app`.

## Important notes

- Do not enable `disable-extension-version-validation` to force an incompatible extension onto GNOME 50.
- Avoid Open Bar until a published release explicitly supports GNOME 50.
- Blur My Shell can add real-time GPU work. The current design intentionally uses opacity without blur.
- Dash to Panel would replace the current dock-plus-top-panel layout with a combined Windows-style panel and is not part of this setup.
- The built-in Wi-Fi, battery, sound, Bluetooth, and indicator symbols are GNOME symbolic icons. The custom theme recolors and resizes them but does not replace their underlying icon artwork.