# Ubuntu Surface Setup Guides

Practical, tested setup and rollback notes for running Ubuntu on a Microsoft Surface Laptop and for reproducing the same desktop experience on comparable Linux hardware.

The guides were written from a working installation rather than as generic command collections. Each document records the final configuration, verification steps, tradeoffs, and rollback path.

## Tested environment

- Microsoft Surface Laptop 4, 15-inch Intel model
- Ubuntu 26.04 LTS
- GNOME 50 on Wayland
- Native 2496 x 1664, 3:2 display
- GNOME fractional scaling enabled

Hardware paths, package versions, and GNOME extension compatibility can differ on another release or device. Review commands before running them, especially commands using `sudo`, PAM, systemd, networking, or boot configuration.

## Guides

### Display and desktop

- [Picture-perfect application scaling](guides/surface-ubuntu-picture-perfect-scaling.md) - Calibrate GNOME, Chrome, VS Code, Firefox, Qt, and GTK applications while preserving native panel resolution.
- [Developer typography in Ghostty and VS Code](guides/developer-typography-ghostty-vscode.md) - Install Fira Code Retina for one user and configure Ghostty, the VS Code editor, integrated terminal, and Copilot Chat code blocks with native settings and rollback steps.
- [Electron and Chromium scaling](guides/electron-scaling-fix.md) - Focused launcher-based fixes for Electron and Chromium applications.
- [GNOME dock, panel, and window effects](guides/gnome-dock-top-panel-customization.md) - Bottom dock, translucent styling, extensions, custom panel themes, verification, and rollback.

### Surface hardware and authentication

- [Disable a broken internal Surface keyboard](guides/disable-surface-keyboard.md) - Reversible input-device disabling and boot automation.
- [Reliable hibernation for broken s2idle resume](guides/surface-s2idle-hibernation.md) - Diagnose failed Surface suspend, configure swap-file resume with dracut and GRUB, replace lid suspend, verify restoration, and roll back safely.
- [Surface IR face authentication pilot](guides/howdy-surface-face-auth.md) - Sudo-only Howdy pilot with password fallback and emergency rollback.

### Media and file support

- [HEIC, HEIF, and DNG support](guides/heic-dng-format-support.md) - Image codecs, thumbnailers, MIME integration, and validation.
- [Windows SMB network share](guides/windows-smb-network-share.md) - Nautilus launcher, connectivity checks, remote thumbnails, credentials, and recovery.

### Networking

- [Tailscale, AdGuard DNS, and tray integration](guides/tailscale-adguard-tray-setup.md) - Boot service, Tailnet DNS, Trayscale, health checks, and troubleshooting.

## Scaling configuration selected on the Surface

The final display setup keeps the native 2496 x 1664 panel mode and uses GNOME at 133 1/3%. Selected applications then use a 1.125 multiplier, producing an effective scale of exactly 1.5:

```text
1.333333 x 1.125 = 1.5
```

This produced a comfortable balance of text clarity, control size, and logical workspace on the 15-inch 3:2 panel. It is a calibration result, not a universal recommendation. The scaling guide explains how to derive and test values for another panel.

## Privacy and placeholders

Machine-specific private addresses, Tailnet identifiers, usernames, SMB share names, and biometric model labels are represented by uppercase placeholders such as:

```text
WINDOWS-IP
SHARE-NAME
TAILSCALE-DNS-IP
ADGUARD-NODE
YOUR-USERNAME
FACE-MODEL-NAME
```

Replace them locally. Do not commit credentials, private keys, real biometric descriptors, private network topology, or exported authentication data.

## Repository layout

```text
.
├── README.md
├── CONTRIBUTING.md
├── LICENSE
└── guides/
    └── *.md
```

## License

Released under the [MIT License](LICENSE).
