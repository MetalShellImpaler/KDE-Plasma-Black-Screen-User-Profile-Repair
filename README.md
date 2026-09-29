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

## Other Troubleshooting Performed

Before isolating the problem to the user profile, several other possible causes were tested:

- AMDGPU / graphics-related errors
- Mesa and LLVM
- Plasma package corruption
- Software rendering
- Different kernel version
- Invalid custom udev rule

None of these resolved the Plasma crash.

The key diagnostic result was:

```text
Affected user  -> Plasma crashes
New user       -> Plasma works
Reset profile  -> Plasma works
```

## Result

After resetting the Plasma configuration and cache:

- Plasma loaded normally
- The black screen disappeared
- The desktop and panel worked again
- Dolphin stopped crashing as part of the broken session

Some Plasma-specific settings may need to be configured again, including:

- Panels
- Widgets
- Wallpaper
- Task Manager layout
- Desktop layout

## Root Cause

The issue was isolated to the KDE Plasma user configuration or cached state.

The exact corrupted file or value was not identified.

The invalid AMDGPU udev rule was a separate issue discovered during troubleshooting.

## Recommendation

If KDE Plasma suddenly stops loading:

1. Check `plasmashell` status and logs
2. Check for obvious system errors such as invalid udev rules
3. Test with a newly created user
4. If the new user works, reset the affected user's Plasma configuration before attempting more invasive repairs

This can avoid unnecessary:

- KDE reinstallation
- GPU driver replacement
- Kernel downgrade
- Full operating system reinstall
