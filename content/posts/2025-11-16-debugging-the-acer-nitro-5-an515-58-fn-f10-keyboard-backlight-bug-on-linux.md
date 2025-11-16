---
title: Debugging the Acer Nitro 5 AN515-58 Fn+F10 Keyboard Backlight Bug on Linux
date: 2025-11-17T03:01:00.000+05:30
draft: true
ShowToc: true
ShowShareButtons: true
ShowReadingTime: true
ShowPostNavLinks: true
ShowBreadCrumbs: true
ShowCodeCopyButtons: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
---
# Debugging the Acer Nitro 5 AN515-58 Fn+F10 Keyboard Backlight Bug on Linux

## Introduction

While using my Acer Nitro 5 AN515-58 laptop running Arch Linux with KDE Plasma on Wayland, I encountered a frustrating bug: pressing **Fn+F10** (which should control the keyboard backlight) caused my display brightness to suddenly drop to zero instead. This post documents my investigation into the root cause and the solution I developed.

## The Problem

**Expected behavior:** Pressing Fn+F10 should increase the keyboard backlight brightness.

**Actual behavior:** The display brightness immediately drops to zero, leaving the screen completely dark.

This issue made the laptop practically unusable, as I had to manually increase the display brightness every time I accidentally pressed Fn+F10.

## Initial Investigation

### Step 1: Observing the Key Events

I started by monitoring input events using `libinput`:

```bash
libinput debug-events
```

When pressing Fn+F10, I observed:

```
event11  KEYBOARD_KEY  +0.000s  KEY_BRIGHTNESSDOWN (224) pressed
event11  KEYBOARD_KEY  +0.000s  KEY_BRIGHTNESSDOWN (224) released
```

This immediately revealed the problem: Fn+F10 was generating `KEY_BRIGHTNESSDOWN` events, which control display brightness rather than keyboard backlight.

### Step 2: Isolating the acer-wmi Driver

My first suspicion was the `acer-wmi` kernel driver, which handles WMI (Windows Management Instrumentation) events on Acer laptops. I tested whether it was the culprit:

```bash
sudo rmmod acer_wmi
# Pressed Fn+F10 again
```

**Result:** The issue persisted even with the driver unloaded.

**Conclusion:** This is not an `acer-wmi` driver bug. The problem exists at a lower level—either in firmware or the AT keyboard driver.

### Step 3: Examining Kernel Messages

I checked for keyboard-related errors in the kernel log:

```bash
sudo dmesg | grep -i "keyboard\|atkbd\|i8042"
```

Output revealed:

```
[    0.704426] i8042: PNP: PS/2 Controller [PNP0303:PS2K] at 0x60,0x64 irq 1
[    0.722664] input: AT Translated Set 2 keyboard as /devices/platform/i8042/serio0/input/input4
[   76.764378] atkbd serio0: Unknown key pressed (translated set 2, code 0xf0 on isa0060/serio0).
[   76.764409] atkbd serio0: Use 'setkeycodes e070 <keycode>' to make it known.
```

The `atkbd` driver reported receiving an unknown key with code `0xf0`. This suggested the keyboard's embedded controller (EC) was sending a scancode that Linux didn't recognize properly.

### Step 4: Identifying the Exact Scancode

To pinpoint the exact scancode, I used `evtest`:

```bash
sudo evtest /dev/input/event4
```

When pressing Fn+F10:

```
Event: time 1763323207.693168, type 4 (EV_MSC), code 4 (MSC_SCAN), value ef
Event: time 1763323207.693168, type 1 (EV_KEY), code 224 (KEY_BRIGHTNESSDOWN), value 2
```

**Critical discovery:** The scancode is `0xef`, not `0xf0` as initially suggested by dmesg. This scancode is being mapped to `KEY_BRIGHTNESSDOWN (224)` by the standard PS/2 keyboard translation table.

### Step 5: Testing the Workaround

Based on the scancode discovery, I tested an immediate fix:

```bash
sudo setkeycodes ef 0
```

This command remaps scancode `0xef` to keycode `0` (reserved/disabled).

**Result:** Success! Pressing Fn+F10 no longer dropped the display brightness to zero.

## Root Cause Analysis

### The Complete Event Chain

Here's what happens when Fn+F10 is pressed:

1. **User presses Fn+F10**
2. **Laptop firmware/EC sends scancode 0xef** (firmware bug)
3. **i8042 PS/2 controller receives scancode**
4. **atkbd driver translates it to KEY_BRIGHTNESSDOWN (224)**
5. **Input subsystem sends event to userspace**
6. **KDE Plasma receives brightness down event**
7. **Display brightness drops to minimum (0)**

### Why This Happens: Firmware Design for Windows

I analyzed the laptop's ACPI DSDT (Differentiated System Description Table) to understand the firmware's perspective:

```bash
sudo acpidump -o acpi.dat
acpixtract -a acpi.dat
iasl -d DSDT.dat
grep -i -A 5 "brightness" dsdt.dsl
```

The DSDT revealed ACPI methods designed for display brightness control:

```asl
Method (_Q11, 0, NotSerialized)  // Brightness UP
{
    Debug = "=====QUERY_11====="
    ^^^WMID.FEBC [One] = BRTS
    Notify (^^^PEG1.PEGP.LCD0, 0x87)  // Display brightness up
}

Method (_Q12, 0, NotSerialized)  // Brightness DOWN
{
    Debug = "=====QUERY_12====="
    ^^^WMID.FEBC [One] = BRTS
    Notify (^^^PEG1.PEGP.LCD0, 0x86)  // Display brightness down
}
```

These ACPI methods reference:
- `BRTS`: Brightness control
- `LCD0`: LCD display device
- They notify the display device, not keyboard backlight

**The design flaw:** Acer designed this firmware for Windows, where their proprietary driver intercepts scancode `0xef` and handles it specially for keyboard backlight control. On Linux, this scancode is interpreted according to standard PS/2 keyboard mappings, causing the brightness bug.

### WMI Interface Issues

Additional evidence from kernel logs showed WMI problems:

```
wmi_bus wmi_bus-PNP0C14:01: [Firmware Bug]: WQ00 data block query control method not found
acer_wmi: Unknown function number - 4 - 0
```

The WMI interface implementation is incomplete, with missing or unsupported methods. When pressing the actual display brightness keys, `acer-wmi` reports "Unknown function number - 4 - 0", confirming the firmware's WMI implementation is broken or non-standard.

### No Kernel Interface for Keyboard Backlight

I verified whether any kernel interface existed for the keyboard backlight:

```bash
ls -la /sys/class/leds/ | grep -i kbd
# No results

find /sys/class/leds -name "*kbd*" -o -name "*backlight*" 2>/dev/null
# No results
```

**Conclusion:** The kernel has no interface to control the keyboard backlight on this model, confirming that the hardware/firmware doesn't expose this functionality properly.

## The Solution

### Permanent Fix: udev hwdb Rule

The proper solution is to create a hardware database (hwdb) rule that remaps the problematic scancode at the input subsystem level:

Create `/etc/udev/hwdb.d/90-acer-nitro5-an515-58.hwdb`:

```
# Acer Nitro 5 AN515-58 - Fix Fn+F10 scancode 0xef
# Fn+F10 sends scancode 0xef which incorrectly maps to display brightness down
# causing display brightness to drop to zero instead of controlling keyboard backlight

evdev:atkbd:dmi:bvn*:bvr*:bd*:svnAcer*:pnNitro*AN515-58*
 KEYBOARD_KEY_ef=reserved
```

Apply the rule:

```bash
sudo systemd-hwdb update
sudo udevadm trigger --sysname-match="event*"
sudo reboot
```

### Alternative: systemd Service

For those who prefer a systemd service approach:

Create `/etc/systemd/system/fix-acer-nitro5-fn10.service`:

```ini
[Unit]
Description=Fix Acer Nitro 5 AN515-58 Fn+F10 scancode
After=local-fs.target
Before=display-manager.service

[Service]
Type=oneshot
ExecStart=/usr/bin/setkeycodes ef 0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fix-acer-nitro5-fn10.service
```

## Kernel Patch Submission

I've prepared a kernel patch to document this issue in the `acer-wmi` driver. While this won't fix the bug (it's a firmware issue, not a driver bug), it will help future users by logging helpful information at boot:

The patch adds:
- A quirk entry for the AN515-58 model
- Boot-time messages directing users to the hwdb workaround
- Reference links to community discussions

I'll be submitting this patch to the Linux kernel mailing list (platform-driver-x86@vger.kernel.org) for inclusion in future kernels.

## Lessons Learned

This debugging process taught me several valuable lessons:

1. **Don't assume driver bugs first:** The issue wasn't in `acer-wmi` but at the firmware level.

2. **Use the right tools:** `evtest` provided the exact scancode, which was crucial for the fix.

3. **Understand the hardware:** Analyzing the ACPI DSDT revealed the Windows-centric firmware design.

4. **Multiple abstraction layers:** The problem spanned firmware → i8042 controller → atkbd driver → input subsystem → userspace.

5. **Document for others:** Many Nitro 5 users face this issue. Proper documentation helps the community.

## Impact and Affected Models

Based on community reports, this issue affects multiple Acer Nitro 5 variants:
- AN515-43 (confirmed)
- AN515-54 (likely)
- AN515-57 (likely)
- AN515-58 (confirmed - this analysis)

If you have a different Nitro 5 model with this issue, the same hwdb fix should work—just adjust the DMI matching pattern.

## Conclusion

What appeared to be a simple key mapping issue turned out to be a complex firmware bug requiring investigation across multiple system layers. The root cause is Acer's Windows-centric firmware design that sends incorrect scancodes on Linux.

While we cannot fix the firmware itself, the hwdb workaround provides a clean, permanent solution that integrates properly with the Linux input subsystem.

If you're experiencing similar issues with Fn keys on laptops, the debugging methodology I've outlined here should help you identify and fix the problem.

## References

- [Arch Linux Forum Discussion](https://bbs.archlinux.org/viewtopic.php?id=304871)
- [Acer Community Report](https://community.acer.com/en/discussion/691263/backlight-brightness-f10-not-working)
- [Ubuntu StackExchange Discussion](https://askubuntu.com/questions/1308045/brightness-keys-fn-left-right-on-acer-nitro-5-not-working)
- [Linux Kernel Source: acer-wmi.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/platform/x86/acer-wmi.c)

## System Information

- **Laptop:** Acer Nitro 5 AN515-58
- **Kernel:** Linux 6.18-rc5
- **Distribution:** Arch Linux
- **Desktop Environment:** KDE Plasma (Wayland)
- **Date:** November 16, 2025

---

*Have you encountered similar firmware bugs on your laptop? Feel free to share your experiences in the comments below.*

*For technical discussions or questions, you can reach me at bugaddr@archlinux.org*
