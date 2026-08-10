# Windows D Drive Network Share

## Current setup

The Ubuntu application launcher is:

```text
~/.local/share/applications/windows-d-drive.desktop
```

It opens this Windows SMB share:

```text
smb://WINDOWS-IP/SHARE-NAME/
```

Replace `WINDOWS-IP` and `SHARE-NAME` throughout this guide with the target Windows computer's private LAN address and SMB share name.

Current launcher content:

```ini
[Desktop Entry]
Type=Application
Name=Windows D Drive
Comment=Open the shared Windows drive
Exec=nautilus --new-window smb://WINDOWS-IP/SHARE-NAME/
Icon=folder-remote
Terminal=false
Categories=Network;
StartupNotify=true
```

The important line is:

```ini
Exec=nautilus --new-window smb://WINDOWS-IP/SHARE-NAME/
```

Nautilus (GNOME Files) performs the SMB mount and can display a graphical credentials dialog. Do not replace this with `gio open`; `gio open` may reject the address when the share is not already mounted.

## When the Windows computer starts later

The Windows computer does not need to be running when Ubuntu starts.

If the Windows computer is off when the launcher is clicked, the share cannot mount. After Windows has started and reached the network:

1. Close any Files window that is still loading the unavailable share.
2. Click **Windows D Drive** again.
3. Nautilus will make a fresh mount attempt.

The share does not mount automatically merely because the Windows computer comes online. Clicking the launcher again triggers the connection.

## Verify connectivity

Check that the Windows computer responds:

```bash
ping -c 2 WINDOWS-IP
```

Check that Windows SMB port 445 is reachable:

```bash
timeout 5 nc -zvw3 WINDOWS-IP 445
```

Expected result includes:

```text
Connection to WINDOWS-IP 445 port [tcp/microsoft-ds] succeeded!
```

## Verify the mount

List GNOME mounts:

```bash
gio mount -l
```

When connected, the output should include:

```text
SHARE-NAME on WINDOWS-IP -> smb://WINDOWS-IP/SHARE-NAME/
```

Test that the share responds to file operations:

```bash
gio list smb://WINDOWS-IP/SHARE-NAME/
```

The local GVFS path is normally:

```text
/run/user/1000/gvfs/smb-share:server=WINDOWS-IP,share=SHARE-NAME
```

Applications should normally use the `smb://` address or open the share through Files instead of relying on that user-session-specific local path.

## Network thumbnails

Nautilus normally uses the `local-only` thumbnail policy, so generic icons on SMB shares are expected even when local files show previews.

Remote thumbnails are enabled on this computer:

```bash
gsettings set org.gnome.nautilus.preferences \
	show-image-thumbnails 'always'
```

The existing 50 MB image limit remains enabled. This prevents Nautilus from downloading and decoding unusually large remote images merely to display a preview:

```bash
gsettings get org.gnome.nautilus.preferences thumbnail-limit
```

Expected output:

```text
uint64 50
```

Verify the active thumbnail policy:

```bash
gsettings get org.gnome.nautilus.preferences show-image-thumbnails
```

Expected output:

```text
'always'
```

Open or revisit the share to trigger thumbnails for visible supported files:

```bash
nautilus --new-window smb://WINDOWS-IP/SHARE-NAME/
```

Check whether a specific remote file received a cached thumbnail:

```bash
gio info -a \
	'standard::content-type,standard::size,thumbnail::path,thumbnail::failed' \
	'smb://WINDOWS-IP/SHARE-NAME/FILE-NAME.jpg'
```

A successful result includes a `thumbnail::path` under `~/.cache/thumbnails/` and no `thumbnail::failed` value. The setting applies to all supported previewable formats, not only images. Installed thumbnailers currently cover common images, HEIF, JPEG XL, SVG, WebP, videos, audio, PDFs, and fonts.

The first visit to a media-heavy network folder can use extra network bandwidth and CPU while previews are generated. Later visits normally reuse the local thumbnail cache. Files over 50 MB, unsupported formats, inaccessible files, and folders not yet visited may continue to display generic icons.

Return to Nautilus's default local-only behavior at any time:

```bash
gsettings reset org.gnome.nautilus.preferences show-image-thumbnails
```

## Unmount and reconnect

If the Windows computer was restarted or the existing mount became stale, unmount it:

```bash
gio mount -u smb://WINDOWS-IP/SHARE-NAME/
```

Then click **Windows D Drive** again.

If it is already unmounted, `gio mount -u` may report that the location is not mounted; that is harmless.

## Validate the launcher

Check the desktop-entry syntax and refresh the application database:

```bash
desktop-file-validate ~/.local/share/applications/windows-d-drive.desktop
update-desktop-database ~/.local/share/applications
```

Launch it from a terminal for testing:

```bash
gtk-launch windows-d-drive
```

## Credentials

No Windows username or password is stored in the `.desktop` file. GNOME may save credentials in the user's login keyring if that option is selected in the graphical authentication dialog.

If authentication fails, verify on Windows that:

- the `SHARE-NAME` share still exists;
- the Windows account has permission to access it;
- Network Discovery and File and Printer Sharing are enabled;
- the Windows network is marked Private;
- the Windows computer still uses `WINDOWS-IP`, or update the launcher if its address changed.

## Change the Windows address or share name

Edit the URI in the launcher's `Exec` line, then refresh the application database:

```bash
update-desktop-database ~/.local/share/applications
```

Keep the URI in this form:

```text
smb://WINDOWS-IP/SHARE-NAME/
```

## Remove the launcher

```bash
rm ~/.local/share/applications/windows-d-drive.desktop
update-desktop-database ~/.local/share/applications
```

Removing the launcher does not delete or modify files on the Windows drive.
