# Surface IR Face Authentication Pilot

## Scope and safety

This is an optional sudo-only Howdy pilot on Ubuntu 26.04. It is disabled by default.

- Face authentication can be enabled only for `sudo`.
- The normal resting state uses Ubuntu's original password-only sudo configuration.
- GDM login and `/etc/pam.d/common-auth` are unchanged.
- Password authentication remains the fallback.
- `pkexec` remains an independent rollback path.
- Howdy is beta software and is not a replacement for a password.
- Failed and successful image snapshots are disabled.

## Current paths

- Root-owned runtime: `/opt/howdy-pilot`
- PAM module: `/opt/howdy-pilot/pam/pam_howdy.so`
- Configuration: `/opt/howdy-pilot/etc/config.ini`
- Face descriptor: `/opt/howdy-pilot/models/FACE-MODEL-NAME.dat`
- Source and user staging area: `~/.local/share/howdy-pilot`
- Source archive tree: `~/.local/src/howdy`
- PAM backups: `/var/backups/howdy-pilot-20260808`
- Control command: `/usr/local/bin/howdy-sudo`
- Root-owned enable template: `/opt/howdy-pilot/pam-config/sudo.enabled`

The face descriptor is root-owned with mode `0600`. It contains a 128-value face encoding, not a saved photograph.

## Current configuration

The stable Surface IR camera path is:

```text
/dev/v4l/by-id/usb-Surface_Surface_Camera_Front_200901010001-video-index0
```

Important settings:

```ini
[core]
detection_notice = true
disabled = false
use_cnn = false
workaround = off

[video]
certainty = 3.5
timeout = 8
recording_plugin = opencv

[snapshots]
save_failed = false
save_successful = false
```

`workaround = off` avoids Howdy's concurrent password-input and virtual Enter-key paths. If face recognition fails, the sudo PAM stack continues to normal password authentication.

The `disabled = false` setting means the Howdy runtime is ready when requested. Howdy is still inactive while its PAM line is absent from `/etc/pam.d/sudo`.

## Enable and disable commands

Check the current state:

```bash
howdy-sudo status
```

Temporarily opt in to facial sudo authentication:

```bash
howdy-sudo enable
```

Return to normal password-only sudo authentication:

```bash
howdy-sudo disable
```

Both state changes use `pkexec` and a root-owned PAM template. Enabling persists until `howdy-sudo disable` is run, so disable it after the commands for which it was intentionally enabled.

## Health checks

Confirm the IR camera link exists:

```bash
readlink -f /dev/v4l/by-id/usb-Surface_Surface_Camera_Front_200901010001-video-index0
```

Confirm the runtime is root-owned and the descriptor is private:

```bash
stat -c '%U:%G %a %n' \
  /opt/howdy-pilot \
  /opt/howdy-pilot/pam/pam_howdy.so \
  /opt/howdy-pilot/etc/config.ini \
  /opt/howdy-pilot/models/FACE-MODEL-NAME.dat

find /opt/howdy-pilot -writable -not -user root -print
```

The `find` command should print nothing.

Test recognition directly without PAM:

```bash
pkexec sh -c 'cd /opt/howdy-pilot/python/howdy && PYTHONPATH=/opt/howdy-pilot/python/howdy /opt/howdy-pilot/venv/bin/python compare.py FACE-MODEL-NAME'
echo $?
```

Exit status `0` means recognition succeeded. This test does not alter authentication.

Enable and test sudo face authentication:

```bash
howdy-sudo enable
sudo -k
sudo true
```

Look at the camera after `Attempting facial authentication` appears. If recognition fails or times out, enter the normal account password.

## Emergency rollback

This command restores the original sudo PAM file without using sudo:

```bash
pkexec install -o root -g root -m 0644 \
  /var/backups/howdy-pilot-20260808/sudo \
  /etc/pam.d/sudo
```

Verify the restored checksum:

```bash
pkexec sha256sum /etc/pam.d/sudo
```

Expected original checksum:

```text
237bd06ddff88d0ab7735346ef15d412fdd121f5c07ab3c4f04a4a620acec158
```

After rollback, open a fresh terminal and confirm password-based sudo:

```bash
sudo -k
sudo true
```

## Re-enable the sudo-only pilot

Only do this after direct recognition succeeds:

```bash
howdy-sudo enable
```

Then test both face recognition and password fallback from a fresh terminal.

## Disable the sudo-only pilot

Restore Ubuntu's original password-only sudo PAM file:

```bash
howdy-sudo disable
```

Confirm the state:

```bash
howdy-sudo status
```

## Complete removal

First run `howdy-sudo disable` (or use the emergency rollback command) and verify password-based sudo in a new terminal. Only after that succeeds, remove the runtime:

```bash
pkexec rm -rf /opt/howdy-pilot
```

Optional user-owned cleanup:

```bash
rm -rf ~/.local/share/howdy-pilot ~/.local/src/howdy
```

Do not remove `/opt/howdy-pilot` while `/etc/pam.d/sudo` still references its PAM module.

## Files intentionally not changed

These checksums should remain:

```text
/etc/pam.d/gdm-password
08e1f43267a73fb6f57e58c13332d4d5c9057b3129a528479cc1287712c74ec0

/etc/pam.d/common-auth
117dab1c9b15121175650cfd8e705aac5f7ca3f1b1d30dd6d6422c250117df5e
```

No face authentication was enabled for graphical login, lock screen, `su`, or global PAM services.
