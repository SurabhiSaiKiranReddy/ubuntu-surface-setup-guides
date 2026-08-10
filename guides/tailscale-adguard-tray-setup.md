# Tailscale, OCI AdGuard DNS, and Tray Setup on Ubuntu

This guide recreates the working Tailscale setup on this Ubuntu computer. Tailscale starts as a system service during boot, routes DNS through an AdGuard Home server on the tailnet, and uses the unofficial Trayscale application for a GNOME top-panel icon.

---

## Technical Specifications

* **Target OS:** Ubuntu 26.04 LTS (Resolute), amd64
* **Desktop:** GNOME 50 on Wayland
* **Tailscale Package Track:** Official stable repository
* **Tailscale Service:** `tailscaled.service`
* **Local Tailnet Address:** `TAILSCALE-CLIENT-IP`
* **Tailnet DNS Resolver:** `TAILSCALE-DNS-IP`
* **Resolver Node:** `ADGUARD-NODE` on Oracle Cloud Infrastructure
* **Tray Application:** Trayscale from Flathub
* **Trayscale Application ID:** `dev.deedles.Trayscale`

> Tailnet addresses can change if a device is removed and enrolled again. Confirm the OCI resolver address before recreating the DNS configuration.

Replace uppercase placeholders such as `TAILSCALE-DNS-IP`, `ADGUARD-NODE`, `YOUR-TAILNET.ts.net`, and `YOUR-USERNAME` with values from the target tailnet.

---

## How the Components Fit Together

```text
tailscaled system service
├── starts automatically during system boot
├── creates the tailscale0 network interface
├── connects this computer to the tailnet
└── routes DNS to OCI AdGuard through Tailscale

Trayscale desktop application
├── starts after GNOME login
├── displays the top-panel tray icon
└── controls the existing tailscaled service
```

The tray application is optional. Tailscale networking and DNS continue to work when Trayscale is closed.

---

## Step 1: Install Tailscale

Use Tailscale's official Ubuntu installation script:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

The expected APT source for Ubuntu 26.04 is:

```text
deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/ubuntu resolute main
```

Authenticate and connect the computer:

```bash
sudo tailscale up
```

Open the displayed URL in a browser and sign in to the correct tailnet.

---

## Step 2: Enable Automatic Boot Startup

The package normally enables the daemon automatically. Enforce and verify that state:

```bash
sudo systemctl enable --now tailscaled.service
systemctl is-enabled tailscaled.service
systemctl is-active tailscaled.service
```

The final two commands should report `enabled` and `active`.

Check the connection:

```bash
tailscale status
tailscale ip
ip address show tailscale0
```

---

## Step 3: Allow the Desktop User to Control Tailscale

Trayscale needs the current user to be the Tailscale operator:

```bash
sudo tailscale set --operator="$USER"
```

Verify the setting:

```bash
tailscale debug prefs | grep OperatorUser
```

The expected operator is the local desktop account, shown below as `YOUR-USERNAME`.

---

## Step 4: Configure OCI AdGuard as Tailnet DNS

The OCI node must be connected to the same tailnet, and AdGuard Home must listen on TCP and UDP port 53 on its Tailscale address.

In the Tailscale admin console:

1. Open **DNS**.
2. Add a nameserver using the OCI node's Tailscale IP: `TAILSCALE-DNS-IP`.
3. Enable **Override local DNS** so tailnet devices use AdGuard for public DNS queries.
4. Keep **MagicDNS** enabled for tailnet hostnames.

On the Ubuntu client, enable Tailscale DNS acceptance:

```bash
tailscale set --accept-dns=true
```

Verify the resolver received from the coordination server:

```bash
tailscale dns status
```

The resolver list should contain:

```text
TAILSCALE-DNS-IP
```

Verify Ubuntu routes DNS through Tailscale:

```bash
resolvectl status tailscale0
```

Expected values include the local Tailscale DNS proxy and the catch-all routing domain:

```text
DNS Servers: 100.100.100.100
DNS Domain: YOUR-TAILNET.ts.net ~.
```

---

## Step 5: Test MagicDNS and AdGuard Filtering

Test tailnet hostname resolution:

```bash
resolvectl query ADGUARD-NODE
tailscale dns query ADGUARD-NODE A
```

Test a normal public lookup:

```bash
tailscale dns query example.com A
```

Test AdGuard blocking with a common advertising domain:

```bash
tailscale dns query pagead2.googlesyndication.com A
```

A blocked result should contain:

```text
0.0.0.0
```

Test the OCI resolver directly through the tailnet:

```bash
dig @TAILSCALE-DNS-IP example.com A
```

---

## Step 6: Install Trayscale for the Top-Panel Icon

The official Tailscale Linux package does not include a tray GUI. Install Flatpak and Trayscale:

```bash
sudo apt update
sudo apt install flatpak

flatpak remote-add --user --if-not-exists flathub \
  https://dl.flathub.org/repo/flathub.flatpakrepo

flatpak install --user flathub dev.deedles.Trayscale
```

Trayscale is an actively maintained but unofficial application. It communicates with the official `tailscaled` daemon and does not replace it.

Verify the installation:

```bash
flatpak info --user dev.deedles.Trayscale
```

Start it in tray-only mode:

```bash
flatpak run dev.deedles.Trayscale --hide-window
```

Ubuntu's `ubuntu-appindicators@ubuntu.com` GNOME extension displays its icon in the top panel.

---

## Step 7: Start Trayscale Automatically at Login

Create the autostart directory and file:

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/dev.deedles.Trayscale.desktop
```

Use this content:

```ini
[Desktop Entry]
Type=Application
Name=Trayscale
Comment=Show Tailscale status and controls in the system tray
Exec=/usr/bin/flatpak run dev.deedles.Trayscale --hide-window
Icon=dev.deedles.Trayscale
Terminal=false
StartupNotify=false
X-GNOME-Autostart-enabled=true
X-GNOME-Autostart-Delay=5
OnlyShowIn=GNOME;Unity;
```

Validate the file:

```bash
desktop-file-validate ~/.config/autostart/dev.deedles.Trayscale.desktop
```

Log out and back in once after the initial Flatpak installation. This refreshes GNOME's Flatpak application search path and tests the login startup entry.

---

## Complete Health Check

Run these commands after a restart:

```bash
printf 'Boot enabled: '; systemctl is-enabled tailscaled.service
printf 'Daemon active: '; systemctl is-active tailscaled.service
printf 'Online: '; tailscale status --json | jq -r '.Self.Online'
printf 'Resolver: '; tailscale dns status | \
  awk '/Resolvers \(in preference order\):/{getline; sub(/^  - /,""); print; exit}'
printf 'Ad blocking: '; tailscale dns query pagead2.googlesyndication.com A | \
  awk '$4=="TypeA"{print $5; exit}'
flatpak ps | grep dev.deedles.Trayscale
```

Expected results:

```text
Boot enabled: enabled
Daemon active: active
Online: true
Resolver: TAILSCALE-DNS-IP
Ad blocking: 0.0.0.0
```

---

## Troubleshooting

### No Tailscale Icon After Restart

Tailscale can still be connected without an icon. Check the daemon first:

```bash
systemctl is-active tailscaled.service
tailscale status
```

Then check Trayscale:

```bash
flatpak ps | grep dev.deedles.Trayscale
flatpak run dev.deedles.Trayscale --hide-window
```

Check Ubuntu tray support:

```bash
gnome-extensions info ubuntu-appindicators@ubuntu.com
```

Its state should be `ACTIVE`.

### GNOME VPN Options Are Disabled

This is expected. Tailscale creates an externally managed `tailscale0` interface instead of a NetworkManager VPN profile. Use Trayscale or the `tailscale` command for controls.

### Tailscale Is Not Connected After Boot

```bash
sudo systemctl enable --now tailscaled.service
sudo systemctl restart tailscaled.service
journalctl -u tailscaled.service -b --no-pager -n 100
tailscale status
```

If authentication expired, reconnect with:

```bash
sudo tailscale up --force-reauth
```

### DNS Does Not Use OCI AdGuard

```bash
tailscale debug prefs | grep CorpDNS
tailscale dns status
resolvectl status tailscale0
tailscale ping ADGUARD-NODE
dig @TAILSCALE-DNS-IP example.com A
```

Confirm the OCI node is online, AdGuard listens on its Tailscale interface, port 53 is allowed, and **Override local DNS** remains enabled in the Tailscale admin console.

### Check Authentication Expiry

```bash
tailscale status --json | jq -r '.Self.KeyExpiry'
```

For a personal laptop, keeping key expiry enabled is safer. Reauthenticate when required.

---

## Updates

Update Tailscale with normal Ubuntu updates:

```bash
sudo apt update
sudo apt upgrade
```

Update Trayscale and its Flatpak runtime:

```bash
flatpak update --user
```

---

## Safe Removal

Remove only the tray application while keeping Tailscale networking active:

```bash
rm ~/.config/autostart/dev.deedles.Trayscale.desktop
flatpak uninstall --user dev.deedles.Trayscale
```

To disconnect and remove Tailscale completely:

```bash
sudo tailscale down
sudo systemctl disable --now tailscaled.service
sudo apt remove tailscale
```

Removing Tailscale also removes this computer's automatic OCI AdGuard DNS path.