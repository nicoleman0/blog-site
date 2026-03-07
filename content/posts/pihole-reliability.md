---
title: "Diagnosing and Hardening a Flaky Pi-hole on a Pi Zero 2W"
date: 2026-03-07
description: "Fixing a slight issue with my Pi-hole"
tags: ["homelab", "pi-hole", "raspberry-pi", "networking", "linux"]
---

My Pi-hole had developed an annoying habit as of the last couple of weeks. The web UI would go unreachable, FTL would silently die, and DHCP would stop assigning addresses. The only fix I had was unplugging and replugging the device. Not ideal for something that sits in the middle of your network, especially when you live with other people.

This post covers how I diagnosed the issue and hardened the setup so that transient failures recover automatically rather than taking down the network.

## Symptoms

- Web UI intermittently unreachable
- When down for extended periods, devices on the network stopped receiving IP addresses
- Hard power cycle was the only fix

The IP assignment failures confirmed to me that this wasn't just a cosmetic web UI issue.

`pihole-FTL`, the process that handles both DNS and DHCP, was crashing or stalling entirely.

## Ruling Out Hardware

Before blaming software, I ruled out hardware.

The Pi Zero 2W is underpowered by modern standards (512MB RAM, limited I/O throughput), and SD card corruption or power instability are common failure modes.

This is also a good general principle for IT support and troubleshooting. Rule out the simplest explanations before reaching for complex ones. Hardware failures are often overlooked because they feel less interesting to investigate than software bugs, but a dodgy power supply or a dying SD card will produce exactly the kind of intermittent, hard-to-reproduce symptoms that send you down the wrong rabbit hole. Eliminate the boring stuff first.

```bash
vcgencmd get_throttled
```

Result: `throttled=0x0` — no throttling, no undervolt events. Clean.

```bash
free -h
df -h
dmesg | grep -i "error\|warn\|undervolt" | tail -20
```

Memory looked fine (264MB available, swap untouched). Disk was fine. `dmesg` showed nothing relevant — just some harmless staging module warnings from the BCM2835 drivers.

So hardware wasn't the culprit, at least not in any obvious way. The power supply however (a standard micro USB wall adapter) is a common failure point on Zeros, and swapping it is worth doing as a precaution regardless. So I am planning on doing this eventually.

## Setting FTL to Auto-Restart

With hardware mostly ruled out, the most pragmatic first step was ensuring FTL recovers automatically when it crashes rather than staying down indefinitely.

```bash
sudo systemctl edit pihole-FTL
```

This opens a drop-in override file at `/etc/systemd/system/pihole-FTL.service.d/override.conf`. I added:

```ini
[Service]
Restart=on-failure
RestartSec=10
```

This goes in the editable zone between the two comment blocks at the top of the file. Anything below `### Edits below this comment will be discarded` is ignored.

After saving:

```bash
sudo systemctl daemon-reload
```

Verify it took:

```bash
sudo systemctl cat pihole-FTL | grep -A3 "Restart"
```

You should see your override alongside the original service definition. The override takes precedence.

## Setting Up the Hardware Watchdog

The FTL restart handles process-level crashes, but if the Pi locks up entirely (kernel panic, memory exhaustion, etc.) the only recovery is a manual power cycle. The Pi Zero 2W has a hardware watchdog that can automate this.

On Raspberry Pi OS, systemd holds the watchdog device by default (you can confirm with `sudo fuser /dev/watchdog0` — PID 1 is systemd). Rather than fight over `/dev/watchdog` with a userspace daemon, the cleaner solution is to configure systemd's built-in watchdog support directly. I figured that out the hard way.

Edit `/etc/systemd/system.conf` and uncomment:

```bash
RuntimeWatchdogSec=15
RebootWatchdogSec=2min
```

Then reload:

```bash
sudo systemctl daemon-reload
```

Verify:

```bash
sudo systemctl show | grep -i watchdog
```

You should see `WatchdogDevice=/dev/watchdog0` and `WatchdogLastPingTimestamp` updating. If systemd stops petting the watchdog for 15 seconds, meaning it's hung, the hardware will force a reboot.

Note: even with these lines commented out, systemd on Pi OS may already be using a default 1-minute timeout. Check `RuntimeWatchdogUSec` in the output to see what's actually active.

## Updating Pi-hole

Worth doing while you're in there:

```bash
pihole -up
```

A bug in an older version could be causing the crashes, and there's no reason to debug against stale software.

## Health Check

Quick one-liner to confirm everything is working:

```bash
pihole status && systemctl is-active pihole-FTL
```

Expected output:

```bash
[✓] FTL is listening on port 53
    [✓] UDP (IPv4)
    [✓] TCP (IPv4)
    [✓] UDP (IPv6)
    [✓] TCP (IPv6)

[✓] Pi-hole blocking is enabled
active
```

If you get a permission error on `/etc/pihole/versions` (like I did), fix it with:

```bash
sudo chmod 644 /etc/pihole/versions
```

## Root Cause

The issue unfortunately resurfaced the next morning. The web UI was totally unreachable again, but this time SSH was still working. FTL's own log told the story:

```bash
INFO: Thread webserver (7) is idle, terminating it.
INFO: All threads joined
INFO: ########## FTL terminated after 15h 50m 45s (code 0)! ##########
```

Code 0 — a clean, intentional shutdown. FTL's internal webserver thread went idle, after which the entire FTL process terminated. That takes down DNS, DHCP, and the web UI together, which explains the full network impact.

The fix is a cron job that checks every 15 minutes and restarts FTL if the web UI stops responding:

```bash
sudo crontab -e
```

Add:

```bash
*/15 * * * * curl -sk http://localhost/ | grep -q "Pi-hole" || systemctl restart pihole-FTL
```

This checks for actual HTML content rather than just a successful connection, which is a more reliable signal that FTL is actually healthy.

## Reporting the Bug

After confirming the root cause, I checked the Pi-hole GitHub to see if this was already a known issue. There are several open issues touching on FTL instability and webserver thread behaviour, but none that matched this specific failure mode exactly: webserver thread going idle triggering a clean FTL shutdown after extended uptime on a Pi Zero 2W.

So, as every good citizen should, I filed an issue on the `pi-hole/FTL` repository with the relevant logs, hardware details, debug token, and the cron workaround. If you're hitting the same thing, it's worth adding your details there too. Here's the [link to the issue.](github.com/issues/created?issue=pi-hole|pi-hole|6559)

## Summary

The setup is now meaningfully more resilient:

- FTL restarts automatically within 10 seconds if it crashes
- A cron job catches the webserver thread idle bug and recovers within 15 minutes
- If the whole system locks up, the hardware watchdog forces a reboot within ~1 minute
- The network recovers without manual intervention

Not bad for an afternoon (and a bit of next morning) of debugging.

Hopefully I don't run into anymore issues.

🐦‍⬛
