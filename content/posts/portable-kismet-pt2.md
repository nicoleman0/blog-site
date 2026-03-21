---
title: "Building a Portable Kismet Device: Part 2 - Software Setup and First Wardrive"
date: 2026-03-21
tags: ["kismet", "wardriving", "raspberry-pi", "gps", "wifi", "wigle"]
---

Part 1 covered the hardware assembly. The Pi 5 was in its case, the Alfa AWUS036ACM had arrived, and I had a VK-172 GPS dongle ready to go. This part covers getting the software stack operational and taking the whole thing out for its first real session.

## Software Stack

The OS is Raspberry Pi OS Lite (64-bit, headless). No desktop — there's no need for one, and it just adds overhead and attack surface. Kismet runs as a service and exposes a web UI on port 2501, which is enough.

Kismet is installed from the official repos:

```bash
wget -O - https://www.kismetwireless.net/repos/kismet-release.gpg.key | sudo apt-key add -
echo 'deb https://www.kismetwireless.net/repos/apt/release/bookworm bookworm main' | sudo tee /etc/apt/sources.list.d/kismet.list
sudo apt update
sudo apt install kismet
```

The Alfa AWUS036ACM uses the mt76x2u driver which is in the kernel mainline — no additional drivers needed on Pi OS.

For GPS, `gpsd` handles the VK-172:

```bash
sudo apt install gpsd gpsd-clients
```

Kismet is set to start on boot via systemd, which means it begins capturing the moment the Pi powers on. This is intentional — in the field I don't want to SSH in just to start a session.

## Wardrive Mode and WiGLE Export

With wardrive mode enabled in `/etc/kismet/kismet.conf`, Kismet reduces per-packet logging overhead and writes output optimised for wardriving rather than deep packet analysis:

```
wardrive=true
```

The tradeoff is that you lose some detail — but for the purpose of network mapping and WiGLE uploads, it's the right call. The web UI remains fully functional with wardrive mode on.

For WiGLE export, rather than relying on Kismet's native wiglecsv logger (which I had some config issues with initially), I extract from the SQLite `.kismet` file post-session using `kismetdb`:

```bash
pip install kismetdb
kismetdb_to_wiglecsv --in Kismet-<timestamp>.kismet --out session.wiglecsv
```

## The GPS Problem

This is the part that took most of the day, and it's worth documenting in full because the VK-172 has some genuine quirks that aren't well documented.

The hardware setup was correct — `gpsd` was running, `/dev/ttyACM0` was present, and the device was talking to the chip. But Kismet showed "Unknown Unknown" in the GPS panel for the entire first session. No fix.

The first issue was straightforward: `DEVICES=""` in `/etc/default/gpsd` meant gpsd was running but not actually pointed at anything. Fixing that was a one-liner:

```bash
sudo sed -i 's/DEVICES=""/DEVICES="\/dev\/ttyACM0"/' /etc/default/gpsd
sudo systemctl restart gpsd
```

After the fix, `gpspipe` confirmed gpsd was reading from the chip. But there was still no position fix.

Running `gpspipe -w` showed the problem: `mode:1` (no fix), `uSat:0`, all satellite signal strengths at 0, and a timestamp of `2019-04-07`. The chip had lost its almanac entirely — complete cold start with no time reference. The u-blox 7 in the VK-172 has a backup battery for retaining almanac data, but mine had clearly drained.

On a cold start with no almanac, the chip has to rebuild its satellite catalogue from scratch. This can take anywhere from a few minutes to — in bad conditions — much longer. Being on a bus for part of the session made it worse; the roof significantly attenuates GPS signal.

A second configuration issue compounded this. The `-n` flag tells gpsd to start polling the device immediately, without waiting for a client to connect. Without it, gpsd sits idle until something queries it:

```
GPSD_OPTIONS="-n"
```

With that in place and the dongle physically outside the bag with clear sky view, the chip started making real progress. After about 20 minutes outdoors I could see satellite signal strengths climbing in `gpspipe` output — `ss` values moving from 0 into the 20-30 range. Then:

```json
{"class":"TPV","device":"/dev/ttyACM0","mode":3,"lat":51.4898,"lon":-0.1597,...}
```

`mode:3` — full 3D fix. Kismet immediately picked it up and started tagging captures with coordinates.

The lesson here: the VK-172 needs a proper cold-start outside the bag the first time. After that, the almanac is written to the chip's flash and subsequent startups are much faster. The dongle also needs to be physically exposed to sky — inside a bag, even near a window, it simply can't get enough signal.

## The Session

King's Road, Chelsea. About an hour of walking, partly on a bus.

The data came back with 5,071 unique networks from 18,574 total observations. Some notes on the findings:

**No WPA3.** Zero. Across over five thousand networks in a relatively affluent part of central London, not a single WPA3 network was detected. WPA3 has been available since 2018 and mandatory on Wi-Fi certified hardware since 2020. What this tells you is that the vast majority of these networks are running ISP-issued hardware that was set up years ago and never touched. People don't replace their routers until they break.

**12.5% open networks.** 634 networks with no encryption at all. On King's Road this makes sense — the street is dense with cafes, restaurants, and retail, all running guest networks. But it's still a notable number. Open doesn't necessarily mean insecure in context (captive portals, deliberately public hotspots), but it's not nothing either.

**53% MAC randomisation.** Over half the captured MACs were locally administered — randomised by the client device. This is mostly phones and laptops with iOS/Android privacy features enabled. It's worth being aware of when interpreting wardrive data: a significant portion of what Kismet logs is ephemeral client traffic that will never be seen again, not fixed infrastructure. The 2,369 globally administered MACs are the meaningful contribution to the WiGLE dataset.

**ISP hardware.** Among real MACs, TP-Link and Technicolor lead, followed by Sagemcom — the OEM behind BT Home Hubs. The BT presence is consistent with London demographics. Sky and Arcadyan (Vodafone) round out the UK ISP hardware picture. It's a fairly expected distribution for residential London.

**Channel distribution.** Channel 1 dominates 2.4GHz, which is typical. The 5GHz channels (36-48 in particular) show healthy usage, which reflects the prevalence of dual-band routers in the area. Everything is running relatively modern hardware, even if the firmware and encryption standards haven't kept up.

## Extracting and Uploading

After the session, shut down properly first:

```bash
sudo shutdown now
```

Pulling power without a clean shutdown risks corrupting the log files.

With the SD card on my laptop (EndeavourOS, so the ext4 partition mounts natively):

```bash
source ~/kismet-tools/bin/activate
kismetdb_to_wiglecsv \
  --in /run/media/$USER/rootfs/home/nick/kismet-logs/Kismet-20260321-13-29-17-1.kismet \
  --out ~/21-03-2026.wiglecsv
```

The export throws warnings about malformed UTF-8 in some device records — obfuscated or corrupted SSIDs. These are skipped automatically and the output file is fine.

Upload to WiGLE under the Upload section. The file sits in a processing queue (trilateration, then geoindexing) before the networks appear on the map. With a 2.3MB file it took a couple of hours to clear the queue.

## What's Next

The rig works. The GPS issue is solved and will be faster on subsequent sessions now that the almanac is cached. The main hardware limitation is the VK-172 itself — it's a budget u-blox 7 with a weak antenna, and it needs proper sky view to maintain a fix. I'm looking at a u-blox M8N based dongle as a replacement, which adds multi-constellation support (GPS + GLONASS + Galileo simultaneously) and significantly better cold-start performance.

Part 3 will likely cover either passive PMKID harvesting with hcxdumptool running alongside Kismet, or adding a secondary interface for injection. Depends which way the build goes.

🐦‍⬛
