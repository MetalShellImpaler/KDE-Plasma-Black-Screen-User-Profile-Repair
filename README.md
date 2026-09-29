# KDE Plasma Black Screen & User Profile Repair

A troubleshooting guide for a KDE Plasma issue where the desktop fails to load, the screen remains black, and applications such as Dolphin may crash with segmentation faults.

## Symptoms

The issue may appear as:

- KDE Plasma fails to load
- The screen remains mostly black
- Panel / taskbar / desktop are missing
- `plasmashell` crashes with `SIGSEGV`
- Dolphin may also crash with:

```text
Segmentation fault (11)
```

In some cases, even the graphical login screen may initially fail to appear.

A terminal may still be available using:

```text
Ctrl + Alt + T
```

## Check Plasma

Check the Plasma service:

```bash
systemctl --user status plasma-plasmashell.service
```

A crash may appear as:

```text
code=dumped, signal=SEGV
```

Check the logs:

```bash
journalctl --user -b -u plasma-plasmashell.service
```

or:

```bash
journalctl -b -p err
```

## Check for Invalid GPU-related udev Rules

If the logs show `udev` errors, it is worth checking for custom rules related to the GPU or graphics stack.

Start by listing custom udev rules:

```bash
ls -l /etc/udev/rules.d/
```

You can also search for AMD/GPU-related entries:

```bash
grep -RniE 'amdgpu|kfd|video|render' /etc/udev/rules.d/
```

If a suspicious rule is found, inspect it:

```bash
cat /etc/udev/rules.d/<rule-file>
```

Pay attention to invalid operators or assignments.

In udev rules:

```text
==
```

is used for matching, while:

```text
=
```

is used for assigning values.

For example:

```udev
KERNEL=="kfd", GROUP="video", MODE="0660"
```

matches the `kfd` device and assigns it to the `video` group.

If a custom rule looks incorrect or is no longer needed, it is safer to disable it temporarily instead of deleting it:

```bash
sudo mv /etc/udev/rules.d/<rule-file> \
/etc/udev/rules.d/<rule-file>.disabled
```

Then reload the rules:

```bash
sudo udevadm control --reload-rules
```

and reboot:

```bash
sudo reboot
```

If disabling the rule does not change the Plasma behavior, the rule is probably not the main cause of the desktop crash and troubleshooting should continue elsewhere.

## Important Test: Try a New User

Before reinstalling KDE, replacing GPU drivers, or reinstalling the operating system, create a temporary user:

```bash
sudo adduser testuser
```

Log out and log in with the new account.

If Plasma works normally with the new user, the issue is most likely inside the original user's Plasma configuration or cached state.

The new account still uses the same:

- KDE installation
- Kernel
- GPU driver
- System libraries
- Hardware

Only the user profile is different.

## Fix That Worked

The following reset fixed the issue.

Replace `USERNAME` with the affected Linux username:

```bash
sudo mv /home/USERNAME/.config/plasma-workspace \
/home/USERNAME/.config/plasma-workspace.BAK 2>/dev/null

sudo mv /home/USERNAME/.config/plasmashellrc \
/home/USERNAME/.config/plasmashellrc.BAK 2>/dev/null

sudo mv /home/USERNAME/.cache \
/home/USERNAME/.cache.BAK
```

Then reboot:

```bash
sudo reboot
```

The original files are not deleted. They are renamed with the `.BAK` extension.

After login, KDE Plasma recreates fresh configuration and cache files.

## If the Issue Persists

If resetting the Plasma user configuration does not resolve the problem, continue with system-wide troubleshooting.

Possible areas to investigate include:

- GPU drivers and graphics stack
- Mesa / LLVM
- Broken or partially upgraded packages
- Kernel regressions
- Display manager issues
- Third-party Plasma widgets or extensions
- Custom udev rules
- Disk or filesystem errors

Useful commands include:

```bash
journalctl -b -p err
```

```bash
dmesg --level=err,warn
```

```bash
dpkg --audit
```

```bash
systemctl --failed
```

Avoid making multiple major system changes at once. Test one change at a time so the actual cause can be identified.

## After Recovery

Once Plasma is working again, some desktop-specific settings may need to be recreated manually, such as:

- Panels
- Widgets
- Wallpaper
- Task Manager layout
- Desktop layout

Keep the `.BAK` files until the system has been stable for a while.

If everything works normally, the backup files can be removed later.

## Notes

This procedure is intended as a non-destructive recovery method.

The original Plasma configuration is renamed rather than deleted, making it possible to inspect or restore individual settings if needed.

If a clean user profile also fails to load Plasma, the problem is more likely system-wide and should be investigated at the driver, package, kernel, or display-manager level.
