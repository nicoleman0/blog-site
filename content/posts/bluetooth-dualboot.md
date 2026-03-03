---
title: "Fixing Bluetooth Dual Boot on EndeavourOS and Windows"
date: 2026-03-02
draft: false
description: "Fixing a really annoying issue with Bluetoooth on Windows/EndeavourOS."
tags: ["bluetooth", "linux"]
categories: ["general"]
toc: false
---

If you dual boot Linux and Windows and share Bluetooth devices between them, you've probably hit this annoying issue: the speakers connect fine on one OS, and then after switching, they refuse to pair on the other.

When you pair a Bluetooth device, a link key is generated and stored on both the host and the device. The problem is that both Windows and Linux see the same physical Bluetooth adapter — same MAC address — but they don't share their key stores. So when you pair on Linux, it writes a new key to the device. Boot into Windows, and Windows has a stale key that no longer matches. Pair on Windows, and now Linux is the one with the stale key.

The fix is simple in principle: get both OSes using the same key. The easiest way to do that is to pair on Windows, extract the key Windows stored, and copy it into Linux's Bluetooth config.

## Why This Is Super Annoying

The Bluetooth link keys in Windows are stored in the registry under:

```cmd
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys
```

But they're locked to the `SYSTEM` account. Even opening regedit as Administrator won't show them — you get an empty folder. This catches a lot of people out.

## How To Fix

### Step 1: Pair on Windows

Make sure your device is paired and connected in Windows first. Go to Settings → Bluetooth & devices and confirm it's there.

### Step 2: Get PsExec

Download PsTools from Microsoft Sysinternals and extract `PsExec.exe` into `C:\Windows\System32`.

### Step 3: Open Regedit as SYSTEM

Open Command Prompt as Administrator and run:

```cmd
psexec -s -i regedit.exe
```

This launches regedit under the SYSTEM account, which is the only account that can actually read the Bluetooth key store.

### Step 4: Find the Key

Navigate to:

```cmd
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys
```

Expand the folder — you should now see a subfolder named after your Bluetooth adapter's MAC address. Inside that, either a value named after your device's MAC (for classic BR/EDR devices) or a subfolder for BLE devices.

In my case, the device (a Bose Mini II SoundLink) was a classic BR/EDR device. The link key was stored as a REG_BINARY value named with the device MAC, directly under the adapter MAC folder:

```cmd
Keys\<adapter_mac>\<device_mac> = xx xx xx xx xx xx xx xx xx xx xx xx xx xx xx xx
```

Note down the full hex value.

### Step 5: Update Linux

Boot into Linux. Find the info file for the device:

```bash
sudo ls /var/lib/bluetooth/
```

This shows your adapter MAC. Then:

```bash
sudo ls /var/lib/bluetooth/<ADAPTER_MAC>/
```

This shows paired device directories. Find the one for your device (the MAC may differ slightly from Windows — Linux uses uppercase with colons).

Open the info file:

```bash
sudo nano /var/lib/bluetooth/<ADAPTER_MAC>/<DEVICE_MAC>/info
```

Find the `[LinkKey]` section and replace the `Key=` value with the Windows key, formatted as uppercase hex with no spaces or separators:

```bash
[LinkKey]
Key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Type=4
PINLength=0
```

Save and restart Bluetooth:

```bash
sudo systemctl restart bluetooth
```

Your device should now connect without re-pairing.

## Notes

- If you re-pair the device on either OS at any point, the key changes and you'll need to redo this.
- BLE devices store keys differently — look for a `[LongTermKey]` section instead of `[LinkKey]`, with `LTK`, `ERand`, and `EDIV` values.
- The device MAC in Linux may not match the one shown in Windows exactly. Check `sudo ls /var/lib/bluetooth/<adapter>/` to find the right directory.
