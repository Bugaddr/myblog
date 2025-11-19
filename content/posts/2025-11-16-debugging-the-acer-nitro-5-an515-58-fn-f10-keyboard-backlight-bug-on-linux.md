---
draft: false
title: Debugging Acer Nitro 5 Fn+F10 Keyboard Backlight Bug on Linux
tags:
  - Linux
  - Kernel
  - Debugging
  - Acer
  - ArchLinux
  - FirmwareBug
  - KDE
  - DMI
  - ACPI
comments: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowCodeCopyButtons: true
ShowReadingTime: true
ShowRssButtonInSectionTermList: true
ShowWordCount: true
ShowToc: true
date: 2025-11-17T03:10:00.000+05:30
showtoc: true
tocopen: false
ShowShareButtons: true
---
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

### Step 5: Analyzing DMI Information

To understand the hardware better and create proper device matching rules, I extracted the DMI (Desktop Management Interface) information:

```bash
sudo dmidecode -t system
```

Output:

```
# dmidecode 3.5
Getting SMBIOS data from sysfs.
SMBIOS 3.3.0 present.

Handle 0x0001, DMI type 1, 27 bytes
System Information
        Manufacturer: Acer
        Product Name: Nitro AN515-58
        Version: V1.15
        Serial Number: NXXXXXXXXXXX
        UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
        Wake-up Type: Power Switch
        SKU Number: 0000000000000000
        Family: Nitro 5
```

I also checked the BIOS information:

```bash
sudo dmidecode -t bios
```

Output:

```
Handle 0x0000, DMI type 0, 26 bytes
BIOS Information
        Vendor: Insyde Corp.
        Version: V1.15
        Release Date: 06/20/2023
        Address: 0xE0000
        Runtime Size: 128 kB
        ROM Size: 16 MB
        Characteristics:
                PCI is supported
                BIOS is upgradeable
                BIOS shadowing is allowed
                Boot from CD is supported
                Selectable boot is supported
                EDD is supported
                ACPI is supported
                USB legacy is supported
                BIOS boot specification is supported
                Targeted content distribution is supported
                UEFI is supported
        BIOS Revision: 1.15
        Firmware Revision: 1.15
```

This DMI information is crucial because:

1. **Product Name matching:** The exact string "Nitro AN515-58" is used for DMI matching in hwdb rules
2. **BIOS vendor:** Insyde Corp. is a common BIOS vendor for Acer laptops
3. **BIOS version:** V1.15 - This helps identify if BIOS updates might fix the issue
4. **Family:** "Nitro 5" - Shows this is part of the broader Nitro 5 family

I also examined the baseboard (motherboard) information:

```bash
sudo dmidecode -t baseboard
```

Output:

```
Handle 0x0002, DMI type 2, 15 bytes
Base Board Information
        Manufacturer: Acer
        Product Name: Jarvis_RN
        Version: V1.15
        Serial Number: NXXXXXXXXXXX
        Asset Tag: Type2 - Board Asset Tag
        Features:
                Board is a hosting board
                Board is replaceable
        Location In Chassis: Type2 - Board Chassis Location
        Chassis Handle: 0x0003
        Type: Motherboard
        Contained Object Handles: 0
```

The baseboard product name "Jarvis_RN" is an internal codename for this model.

### Step 6: Verifying Input Device Information

To understand the complete input device topology:

```bash
cat /proc/bus/input/devices | grep -A 15 "AT Translated"
```

Output:

```
N: Name="AT Translated Set 2 keyboard"
P: Phys=isa0060/serio0/input0
S: Sysfs=/devices/platform/i8042/serio0/input/input4
U: Uniq=
H: Handlers=sysrq kbd leds event4 rfkill 
B: PROP=0
B: EV=120013
B: KEY=180000 10000 c020000000000 0 0 100700f02000007 ff802078f870f401 febfffdfffcfffff fffffffffffffffe
B: MSC=10
B: LED=7
```

Key findings:

* **Device path:** `/devices/platform/i8042/serio0/input/input4`
* **Event handler:** `event4`
* **Capabilities:** Standard AT keyboard with LED support (Num Lock, Caps Lock, Scroll Lock)

### Step 7: Testing the Workaround

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
iasl -d dsdt.dat
grep -i -A 5 -B 5 "brightness\|_Q.*F\|Method.*F10" dsdt.dsl | head -50
```

The DSDT revealed ACPI methods designed for display brightness control:

```asl
Method (_Q11, 0, NotSerialized)  // Brightness UP
{
    Debug = "=====QUERY_11====="
    ^^^WMID.FEBC [Zero] = One
    ^^^WMID.FEBC [One] = HTBN
    ^^^WMID.FEBC [One] = BRTS
    Notify (^^^PEG1.PEGP.LCD0, 0x87)  // Display brightness up
}

Method (_Q12, 0, NotSerialized)  // Brightness DOWN
{
    Debug = "=====QUERY_12====="
    ^^^WMID.FEBC [Zero] = One
    ^^^WMID.FEBC [One] = HTBN
    ^^^WMID.FEBC [One] = BRTS
    Notify (^^^PEG1.PEGP.LCD0, 0x86)  // Display brightness down
}
```

These ACPI methods reference:

* `BRTS`: Brightness control variable
* `HTBN`: Hotkey Button Number
* `LCD0`: LCD display device (PEG1.PEGP.LCD0)
* `WMID.FEBC`: Acer's proprietary WMI interface

The key insight here is that these methods notify the display device (`LCD0`) with ACPI notify codes `0x86` (brightness down) and `0x87` (brightness up). There are **no equivalent methods for keyboard backlight control**.

**The design flaw:** Acer designed this firmware for Windows, where their proprietary driver intercepts scancode `0xef` and handles it specially for keyboard backlight control. On Linux, this scancode is interpreted according to standard PS/2 keyboard mappings, causing the brightness bug.

### WMI Interface Issues

Additional evidence from kernel logs showed WMI problems:

```
wmi_bus wmi_bus-PNP0C14:01: [Firmware Bug]: WQ00 data block query control method not found
acer_wmi: Unknown function number - 4 - 0
```

The WMI interface implementation is incomplete, with missing or unsupported methods. When pressing the actual display brightness keys, `acer-wmi` reports "Unknown function number - 4 - 0", confirming the firmware's WMI implementation is broken or non-standard.

I checked which WMI GUIDs are available:

```bash
ls /sys/bus/wmi/devices/
```

Output:

```
05901221-D566-11D1-B2F0-00A0C9062910
676AA15E-6A47-4D9F-A2CC-1E6D18D14026
6AF4F258-B401-42FD-BE91-3D4AC2D7C0D3
95764E09-FB56-4E83-B31A-37761F60994A
ABBC0F6D-8EA1-11D1-00A0-C90629100000
```

The relevant Acer WMI GUIDs:

* `6AF4F258-B401-42FD-BE91-3D4AC2D7C0D3` - WMID_GUID1 (main WMI interface)
* `95764E09-FB56-4E83-B31A-37761F60994A` - WMID_GUID2 (device capabilities)
* `676AA15E-6A47-4D9F-A2CC-1E6D18D14026` - ACERWMID_EVENT_GUID (WMI events)

### No Kernel Interface for Keyboard Backlight

I verified whether any kernel interface existed for the keyboard backlight:

```bash
ls -la /sys/class/leds/ | grep -i kbd
# No results

find /sys/class/leds -name "*kbd*" -o -name "*backlight*" 2>/dev/null
# No results
```

**Conclusion:** The kernel has no interface to control the keyboard backlight on this model, confirming that the hardware/firmware doesn't expose this functionality properly through standard interfaces.

## The Solution

### Permanent Fix: udev hwdb Rule (I am using this)

The proper solution is to create a hardware database (hwdb) rule that remaps the problematic scancode at the input subsystem level.

The hwdb rule uses DMI information for precise device matching. Here's how the matching works:

```
evdev:atkbd:dmi:bvn*:bvr*:bd*:svnAcer*:pnNitro*AN*515-58:pvr*
```

Breaking down this match string:

* `evdev:atkbd:` - Matches AT keyboard devices through evdev
* `dmi:` - Uses DMI information for matching
* `bvn*` - BIOS vendor name (any)
* `bvr*` - BIOS version (any)
* `bd*` - BIOS date (any)
* `svnAcer*` - System vendor name = "Acer"
* `pnNitro*AN515-58*` - Product name contains "Nitro" and "AN515-58"

Create `/etc/udev/hwdb.d/90-acer-nitro5-an515-58.hwdb`:

```
evdev:atkbd:dmi:bvn*:bvr*:bd*:svnAcer*:pnNitro*AN*515-58:pvr*
 KEYBOARD_KEY_ef=kbdillumup                             # Fn+F10
 KEYBOARD_KEY_f0=kbdillumdown                           # Fn+F9
```

Apply the rule:

```bash
sudo systemd-hwdb update
sudo udevadm trigger --sysname-match="event*"
sudo reboot
```

### Verifying the hwdb Rule Application

After reboot, verify the rule was applied:

```bash
# Check if hwdb rule matches your device
udevadm info /sys/class/input/event4 | grep -i "keyboard_key"

# Should show: E: KEYBOARD_KEY_ef=reserved
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
ExecStart=/usr/bin/setkeycodes ef 0 && /usr/bin/setkeycodes f0 0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fix-acer-nitro5-fn10.service
```

## Impact and Affected Models

Based on community reports and DMI analysis, this issue affects multiple Acer Nitro 5 variants:

| Model    | Status    | DMI Product Name | Notes             |
| -------- | --------- | ---------------- | ----------------- |
| AN515-43 | Confirmed | Nitro AN515-43   | Community reports |
| AN515-54 | Likely    | Nitro AN515-54   | Similar firmware  |
| AN515-57 | Likely    | Nitro AN515-57   | Similar firmware  |
| AN515-58 | Confirmed | Nitro AN515-58   | This analysis     |

If you have a different Nitro 5 model with this issue, the same hwdb fix should work. You can check your exact product name with:

```bash
sudo dmidecode -s system-product-name
```

Then adjust the hwdb DMI matching pattern accordingly.

## Conclusion

What appeared to be a simple key mapping issue turned out to be a complex firmware bug requiring investigation across multiple system layers. The root cause is Acer's Windows-centric firmware design that sends incorrect scancodes on Linux.

While we cannot fix the firmware itself, the hwdb workaround provides a clean, permanent solution that integrates properly with the Linux input subsystem without requiring kernel patches or custom drivers.

## References

* [Arch Linux Forum Discussion](https://bbs.archlinux.org/viewtopic.php?id=304871)
* [Acer Community Report](https://community.acer.com/en/discussion/691263/backlight-brightness-f10-not-working)
* [Ubuntu StackExchange Discussion](https://askubuntu.com/questions/1308045/brightness-keys-fn-left-right-on-acer-nitro-5-not-working)
* [Linux Kernel Source: acer-wmi.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/platform/x86/acer-wmi.c)
* [systemd hwdb Documentation](https://www.freedesktop.org/software/systemd/man/hwdb.html)
* [DMI/SMBIOS Specification](https://www.dmtf.org/standards/smbios)
* [ACPI Specification](https://uefi.org/specifications)

- - -

*Have you encountered similar firmware bugs on your laptop? Share your experiences in the comments below or reach out for technical discussions.*
