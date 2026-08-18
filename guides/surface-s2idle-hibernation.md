# Reliable Hibernation for a Surface with Broken s2idle Resume

## Scope and tested environment

This guide replaces unreliable suspend with disk-backed hibernation on a Microsoft Surface Laptop 4 running Ubuntu 26.04.

The tested machine has:

- 32 GB of RAM;
- an ext4 root filesystem;
- a swap file at `/swap.img`;
- dracut-generated initramfs images;
- GRUB;
- systemd 259;
- only `s2idle` available for suspend; and
- Secure Boot disabled.

The final tested configuration uses a 36 GiB swap file. Manual hibernation restored the same boot and desktop session twice, including open application windows.

Do not copy the UUID or swap offset from another machine. Both values are storage-layout-specific and must be calculated locally.

The final configuration intentionally retains:

- `/swap.img` at 36 GiB;
- `/etc/default/grub.d/40-hibernate.cfg`;
- `/etc/dracut.conf.d/40-hibernate.conf`;
- `/etc/polkit-1/rules.d/10-enable-hibernate.rules`;
- `/etc/systemd/logind.conf.d/90-surface-hibernate.conf`;
- `/etc/chrony/conf.d/90-hibernate-clock.conf`;
- `/boot/grub/grub.cfg.pre-hibernate`; and
- `/boot/initrd.img-KERNEL-VERSION.pre-hibernate`.

An emergency user service named `prevent-lid-suspend.service` was used temporarily to keep the machine awake while hibernation was being prepared. It was disabled and removed after hibernation passed controlled tests. No temporary lid inhibitor remains in the final configuration.

## Why hibernation was used

The Surface exposed only `s2idle`:

```bash
cat /sys/power/mem_sleep
```

Observed output:

```text
[s2idle]
```

Several failed boots ended with the same final kernel record:

```text
PM: suspend entry (s2idle)
```

There was no subsequent resume, orderly shutdown, kernel panic, out-of-memory event, thermal shutdown, or storage error. The following boot recovered the ext4 journal, confirming an unclean power cycle. Battery history also ruled out depletion.

Hibernation avoids this failing resume path. It writes the active memory image to swap, powers the computer off, and restores the image during the next Ubuntu boot.

## Safety constraints

- Save important work before every setup or test step.
- Keep the normal stock kernel available in GRUB.
- Do not configure lid-triggered hibernation until a manual hibernation and restore succeeds.
- Do not test suspend and hibernation changes simultaneously.
- Do not boot another operating system between hibernating Ubuntu and resuming Ubuntu.
- A hibernation image contains data that was present in RAM. On an unencrypted root filesystem, the swap file is also unencrypted at rest.
- Hibernation can fail after recreating, moving, or defragmenting the swap file because its physical offset can change.
- This procedure is for an ext4 swap file and dracut. Btrfs, encrypted swap, LVM, zram-only systems, and initramfs-tools require different handling.

## 1. Inspect the current system

Confirm that the kernel supports hibernation but only offers `s2idle` for suspend:

```bash
cat /sys/power/state
cat /sys/power/mem_sleep
cat /sys/power/disk
```

The tested system reported `disk` in `/sys/power/state` and `platform` in `/sys/power/disk`.

Check memory, swap, free disk space, root filesystem type, and initramfs tooling:

```bash
free -h
swapon --show
df -hT /
findmnt -no SOURCE,FSTYPE,UUID /
dpkg-query -W -f='${Package}\t${Version}\n' dracut dracut-core systemd grub2-common util-linux
```

This guide assumes:

```text
root filesystem: ext4
swap path:       /swap.img
initramfs:       dracut
```

Check whether hibernation is initially available:

```bash
busctl call \
  org.freedesktop.login1 \
  /org/freedesktop/login1 \
  org.freedesktop.login1.Manager \
  CanHibernate
```

An undersized swap file or Ubuntu's default authorization policy can make this return `s "no"`.

## 2. Size the swap file

For the tested 32 GB machine, a 36 GiB swap file provides conservative space for a hibernation image. A compressed image may be smaller than installed RAM, but sizing swap at least as large as usable RAM avoids relying on compression.

The following recreates `/swap.img`. It changes the swap UUID, which is harmless here because `/etc/fstab` references the file path rather than the swap UUID.

First confirm `/etc/fstab` contains:

```text
/swap.img none swap sw 0 0
```

Then recreate the swap file:

```bash
sudo swapoff /swap.img && \
  sudo rm /swap.img && \
  sudo fallocate -l 36G /swap.img && \
  sudo chmod 0600 /swap.img && \
  sudo mkswap /swap.img && \
  sudo swapon /swap.img
```

Immediately verify it:

```bash
swapon --show
free -h
stat -c '%U:%G %a %s %n' /swap.img
```

Expected properties include approximately 36 GiB, owner `root:root`, and mode `600`.

If any creation command fails, do not reboot until a valid swap file is active again.

## 3. Calculate the resume UUID and offset

The `resume=` parameter identifies the block device containing the root filesystem. The `resume_offset=` parameter identifies the first physical page of the swap file on that device.

Calculate both values:

```bash
ROOT_UUID=$(findmnt -no UUID /)
FIRST_BLOCK=$(sudo filefrag -v /swap.img | \
  awk '
    $1 == "0:" {
      ranges = 0
      for (field = 2; field <= NF; field++) {
        if ($field ~ /^[0-9]+\.\./) {
          ranges++
          if (ranges == 2) {
            sub(/\.\..*$/, "", $field)
            print $field
            exit
          }
        }
      }
    }
  ')
FS_BLOCK_SIZE=$(stat -fc %s /swap.img)
PAGE_SIZE=$(getconf PAGESIZE)

if [[ -z "$ROOT_UUID" ]]; then
  echo "Could not determine the root filesystem UUID" >&2
  exit 1
fi

if [[ ! "$FIRST_BLOCK" =~ ^[0-9]+$ ]]; then
  echo "Could not extract the first swap-file block from filefrag" >&2
  exit 1
fi

if (( FIRST_BLOCK * FS_BLOCK_SIZE % PAGE_SIZE != 0 )); then
  echo "Swap offset does not align to a memory page" >&2
  exit 1
fi

SWAP_OFFSET=$((FIRST_BLOCK * FS_BLOCK_SIZE / PAGE_SIZE))

printf 'ROOT_UUID=%s\n' "$ROOT_UUID"
printf 'SWAP_OFFSET=%s\n' "$SWAP_OFFSET"
```

On the tested ext4 installation, both block sizes were 4096 bytes, so the first physical block number was also the resume page offset.

Record the values locally as:

```text
ROOT_UUID=ROOT-FILESYSTEM-UUID
SWAP_OFFSET=SWAPFILE-PHYSICAL-OFFSET
```

Never reuse example values from documentation.

## 4. Add resume support to dracut

Create a dracut drop-in:

```bash
sudo install -d -m 0755 /etc/dracut.conf.d
printf '%s\n' 'add_dracutmodules+=" resume "' | \
  sudo tee /etc/dracut.conf.d/40-hibernate.conf >/dev/null
```

Back up and rebuild the current initramfs:

```bash
INITRD_BACKUP="/boot/initrd.img-$(uname -r).pre-hibernate"

if ! sudo test -e "$INITRD_BACKUP"; then
  sudo cp -a "/boot/initrd.img-$(uname -r)" "$INITRD_BACKUP"
fi

sudo dracut --force "/boot/initrd.img-$(uname -r)" "$(uname -r)"
```

Verify that dracut included both systemd and resume support:

```bash
sudo lsinitrd "/boot/initrd.img-$(uname -r)" -m | \
  grep -E '^(systemd|resume)$'
```

Expected output:

```text
systemd
resume
```

## 5. Add the resume kernel parameters

Create a GRUB drop-in using the values calculated earlier. `ROOT_UUID` and `SWAP_OFFSET` are shell variables, so rerun the calculation block in Section 3 first if this is a new terminal session.

```bash
: "${ROOT_UUID:?Run the calculation block in Section 3 first}"
: "${SWAP_OFFSET:?Run the calculation block in Section 3 first}"

if [[ ! "$SWAP_OFFSET" =~ ^[0-9]+$ ]]; then
  echo "SWAP_OFFSET is not numeric" >&2
  exit 1
fi

sudo install -d -m 0755 /etc/default/grub.d

sudo tee /etc/default/grub.d/40-hibernate.cfg >/dev/null <<EOF
GRUB_CMDLINE_LINUX_DEFAULT="\${GRUB_CMDLINE_LINUX_DEFAULT} resume=UUID=${ROOT_UUID} resume_offset=${SWAP_OFFSET}"
EOF
```

Back up and rebuild GRUB:

```bash
if ! sudo test -e /boot/grub/grub.cfg.pre-hibernate; then
  sudo cp -a /boot/grub/grub.cfg /boot/grub/grub.cfg.pre-hibernate
fi

sudo update-grub
```

Do not continue if `dracut` or `update-grub` reports an error.

## 6. Allow active local administrators to hibernate

Ubuntu includes a vendor polkit rule that disables hibernation by default. The following local rule overrides it only for an active, local member of the `sudo` group. It does not grant permission to bypass application inhibitors.

```bash
sudo install -d -m 0755 /etc/polkit-1/rules.d

sudo tee /etc/polkit-1/rules.d/10-enable-hibernate.rules >/dev/null <<'EOF'
polkit.addRule(function(action, subject) {
    if ((action.id == "org.freedesktop.login1.hibernate" ||
         action.id == "org.freedesktop.login1.hibernate-multiple-sessions" ||
         action.id == "org.freedesktop.login1.handle-hibernate-key") &&
        subject.active == true && subject.local == true &&
        subject.isInGroup("sudo")) {
            return polkit.Result.YES;
    }
});
EOF

sudo chmod 0644 /etc/polkit-1/rules.d/10-enable-hibernate.rules
```

Confirm the intended user is in the `sudo` group:

```bash
id -nG
```

## 7. Reboot and validate readiness

Reboot before attempting hibernation:

```bash
systemctl reboot
```

After logging back in, confirm the boot parameters and kernel resume target:

```bash
tr ' ' '\n' </proc/cmdline | grep '^resume'
cat /sys/power/resume
cat /sys/power/resume_offset
swapon --show
```

Confirm the normal desktop session can hibernate:

```bash
busctl call \
  org.freedesktop.login1 \
  /org/freedesktop/login1 \
  org.freedesktop.login1.Manager \
  CanHibernate
```

Expected result:

```text
s "yes"
```

On the first normal boot, the journal may say that no hibernation image was found. That is expected because no image exists yet:

```text
Unable to resume ... continuing boot process
PM: Image not found
```

## 8. Perform a controlled manual test

Save all work. Keep the charger connected for the first test.

Record the current boot ID:

```bash
cat /proc/sys/kernel/random/boot_id | tee /tmp/pre-hibernate-boot-id
```

Trigger hibernation:

```bash
systemctl hibernate
```

The computer should write the image and power off completely. Wait about ten seconds, then press the power button once and boot Ubuntu normally.

After the desktop returns, compare the boot ID:

```bash
diff -u /tmp/pre-hibernate-boot-id /proc/sys/kernel/random/boot_id
```

No output means the same boot was restored. Also verify the kernel and systemd markers:

```bash
journalctl -b 0 --no-pager \
  --grep="PM: hibernation: hibernation (entry|exit)|System returned from sleep operation 'hibernate'"
```

A successful cycle contains all three events:

```text
PM: hibernation: hibernation entry
System returned from sleep operation 'hibernate'.
PM: hibernation: hibernation exit
```

Do not configure automatic lid actions unless this test succeeds and the expected applications remain open.

## 9. Replace suspend with hibernation

After a successful manual restore, configure logind to hibernate on ordinary lid closure:

```bash
sudo install -d -m 0755 /etc/systemd/logind.conf.d

sudo tee /etc/systemd/logind.conf.d/90-surface-hibernate.conf >/dev/null <<'EOF'
[Login]
HandleLidSwitch=hibernate
EOF

sudo systemctl reload systemd-logind.service
```

Verify the live action:

```bash
busctl get-property \
  org.freedesktop.login1 \
  /org/freedesktop/login1 \
  org.freedesktop.login1.Manager \
  HandleLidSwitch
```

Expected result:

```text
s "hibernate"
```

The systemd default for a docked laptop remains `ignore`. Check it with:

```bash
busctl get-property \
  org.freedesktop.login1 \
  /org/freedesktop/login1 \
  org.freedesktop.login1.Manager \
  HandleLidSwitchDocked
```

Align GNOME's visible power settings with the system policy. These settings are per-user; run them from each local desktop account that should use this policy.

```bash
gsettings set org.gnome.settings-daemon.plugins.power \
  sleep-inactive-ac-type 'nothing'

gsettings set org.gnome.settings-daemon.plugins.power \
  sleep-inactive-battery-type 'hibernate'

gsettings set org.gnome.settings-daemon.plugins.power \
  sleep-inactive-battery-timeout 1800

gsettings set org.gnome.settings-daemon.plugins.power \
  lid-close-ac-action 'hibernate'

gsettings set org.gnome.settings-daemon.plugins.power \
  lid-close-battery-action 'hibernate'
```

This final policy means:

- ordinary lid closure on AC or battery hibernates;
- 30 minutes of inactivity on battery hibernates;
- inactivity on AC does nothing; and
- docked lid closure is ignored by logind.

Using hibernation on AC lid closure also protects against closing the lid while plugged in and then unplugging the closed laptop before putting it in a bag.

## 10. Final verification

Check all final state in one pass:

```bash
busctl call \
  org.freedesktop.login1 \
  /org/freedesktop/login1 \
  org.freedesktop.login1.Manager \
  CanHibernate

busctl get-property \
  org.freedesktop.login1 \
  /org/freedesktop/login1 \
  org.freedesktop.login1.Manager \
  HandleLidSwitch

swapon --show
cat /sys/power/resume
cat /sys/power/resume_offset

for key in \
  sleep-inactive-ac-type \
  sleep-inactive-battery-type \
  sleep-inactive-battery-timeout \
  lid-close-ac-action \
  lid-close-battery-action
do
  printf '%s = ' "$key"
  gsettings get org.gnome.settings-daemon.plugins.power "$key"
done
```

The tested machine reported hibernation available, a 36 GiB active swap file, a nonzero resume device and offset, and `hibernate` for lid closure and battery inactivity.

## Maintenance after updates

The dracut and GRUB drop-ins persist across normal kernel updates. After a major kernel, dracut, GRUB, systemd, or firmware update:

```bash
tr ' ' '\n' </proc/cmdline | grep '^resume'
sudo lsinitrd "/boot/initrd.img-$(uname -r)" -m | grep -E '^(systemd|resume)$'
busctl call org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager CanHibernate
```

Then run one controlled manual hibernation test before trusting unattended lid closure.

If `/swap.img` is ever recreated, moved, restored from backup, or defragmented:

1. Recalculate `SWAP_OFFSET`.
2. Update `/etc/default/grub.d/40-hibernate.cfg`.
3. Run `sudo update-grub`.
4. Reboot and perform a controlled test.

Do not use an old offset after changing the swap file.

## Troubleshooting

### `CanHibernate` remains `no`

Compare administrator and normal-session results:

```bash
busctl call org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager CanHibernate

pkexec /usr/bin/busctl call org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager CanHibernate
```

If the administrator result is `yes` but the normal result is `no`, inspect the local polkit rule and confirm the user is active, local, and in the `sudo` group.

### The machine cold-boots instead of restoring

Check:

```bash
tr ' ' '\n' </proc/cmdline | grep '^resume'
cat /sys/power/resume
cat /sys/power/resume_offset
sudo filefrag -v /swap.img | sed -n '1,8p'
journalctl -b 0 --no-pager \
  --grep='hibernate|resume|PM: Image'
```

Recalculate the offset if the swap file changed. Also confirm the initramfs for the running kernel contains the `resume` module.

### Closing the lid still suspends

Check the live logind value rather than only the file:

```bash
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager HandleLidSwitch
```

If it still says `suspend`, reload logind:

```bash
sudo systemctl reload systemd-logind.service
```

### The clock remains wrong after hibernation

A brief clock correction immediately after resume can be normal. On the tested Surface, however, Chrony detected an offset equal to the time spent hibernating and attempted to correct it gradually. The default Ubuntu directive, `makestep 1 3`, permits an immediate step only during Chrony's first three clock updates after startup. Because Chrony remains running across hibernation, the restored clock could remain hours behind.

Confirm the condition before changing the policy:

```bash
timedatectl status
chronyc tracking
journalctl -b -u chrony.service --no-pager | tail -n 50
```

If NTP sources are healthy but `System time` reports a large offset after resume, allow Chrony to step offsets greater than one second throughout its service lifetime:

```bash
sudo tee /etc/chrony/conf.d/90-hibernate-clock.conf >/dev/null <<'EOF'
# Surface hibernation can restore stale wall-clock time.
# Allow chronyd to step offsets over one second after resume.
makestep 1 -1
EOF

sudo chmod 0644 /etc/chrony/conf.d/90-hibernate-clock.conf
sudo chronyd -p -f /etc/chrony/chrony.conf >/dev/null
sudo chronyc makestep
sudo systemctl restart chrony.service
chronyc waitsync 30 0.1 1000 1
```

Verify that the system clock, RTC, and NTP state agree:

```bash
timedatectl status
chronyc tracking
```

The tested result reported `System clock synchronized: yes`, `Leap status: Normal`, and a sub-millisecond system-time offset. The existing `rtcsync` directive subsequently updated the hardware RTC in UTC.

This override permits a clock step after any large offset, not only after hibernation. A time step can affect application timers, logs, and active jobs. Use it only with trusted time sources; the tested configuration preferred Canonical's NTS-authenticated Ubuntu pools.

## Rollback

### Stop automatic hibernation but keep manual hibernation

Disable GNOME's automatic actions:

```bash
gsettings set org.gnome.settings-daemon.plugins.power \
  sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power \
  sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power \
  lid-close-ac-action 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power \
  lid-close-battery-action 'nothing'
```

Remove the logind lid policy and reload it:

```bash
sudo rm /etc/systemd/logind.conf.d/90-surface-hibernate.conf
sudo systemctl reload systemd-logind.service
```

On an unmodified Ubuntu installation, removing this file usually restores lid-triggered suspend. If `s2idle` resume is known to fail, do not close the lid after this rollback until another safe lid policy is configured.

### Completely remove hibernation support

First disable automatic hibernation as above. Then remove the local policy and boot drop-ins:

```bash
sudo rm -f \
  /etc/polkit-1/rules.d/10-enable-hibernate.rules \
  /etc/dracut.conf.d/40-hibernate.conf \
  /etc/default/grub.d/40-hibernate.cfg \
  /etc/chrony/conf.d/90-hibernate-clock.conf

sudo dracut --force "/boot/initrd.img-$(uname -r)" "$(uname -r)"
sudo update-grub
sudo systemctl restart chrony.service
systemctl reboot
```

After reboot, confirm `/proc/cmdline` no longer contains `resume=` or `resume_offset=`.

The 36 GiB swap file can remain in place safely. To reclaim space, recreate it at a smaller size only after hibernation support has been removed and the machine has rebooted:

```bash
sudo swapoff /swap.img && \
  sudo rm /swap.img && \
  sudo fallocate -l 8G /swap.img && \
  sudo chmod 0600 /swap.img && \
  sudo mkswap /swap.img && \
  sudo swapon /swap.img
swapon --show
```

Keep the `/swap.img none swap sw 0 0` entry in `/etc/fstab` if normal swap should remain enabled.