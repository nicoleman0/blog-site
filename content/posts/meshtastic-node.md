---
title: "Setting Up My Meshtastic Node"
date: 2026-02-21
draft: false
description: "How I setup my first Meshtastic node."
tags: ["meshtastic", "lora", "hardware", "radio", "esp32"]
categories: ["meshtastic", "lora"]
toc: false
---

This weekend I set up a permanent LoRa mesh network node on my windowsill overlooking Chelsea. Here's what I learned, what worked, and what didn't...

I decided to do this because of the increase in privacy concerns regarding messaging apps like Discord, etc. It seemed like as good a time as any to set this up, and I had been considering this for a long time.

## Meshtastic?

Meshtastic is an open-source project that runs on cheap LoRa radio hardware and lets devices form a decentralised mesh network. That means no internet, no cellular, and no central infrastructure. :p

Nodes relay messages between each other over radio, extending range with each hop.

The protocol uses LoRa (Long Range) radio, which trades data rate for distance. You're not streaming anything over this, so messages are short and transmission is slow by design. What you get in return is multi-kilometre range on low power hardware that costs under £30.

In the UK, Meshtastic operates on the 868MHz band, which is part of the unlicensed ISM spectrum. No licence required.

## The Hardware

I picked up an **M5Stack Unit C6L** — a compact ESP32-C6 based board with dual antennas (one for LoRa at 868-923MHz, one for Wi-Fi/Bluetooth), a small OLED display, and USB-C power. It ships pre-flashed with Meshtastic firmware, which was a pleasant surprise. I was expecting to have to flash the firmware myself.

The form factor is small enough to sit on a windowsill without being obtrusive, and since it's USB-C powered with no internal battery, it's designed for permanent deployment.

One thing to note immediately: **attach the antennas before powering on**. Transmitting without an antenna connected can damage the RF frontend. The C6L comes with two — red collar for LoRa, blue for Wi-Fi/Bluetooth.

## Configuration

Configuration was relatively simple and is done through the Meshtastic iOS app over Bluetooth. The device appears as "Meshtastic ff6c" (the last bytes of the MAC address) and pairs without a PIN.

Here's what I ended up with:

**Device role** — Left as Client rather than Router. The conventional wisdom is that Router mode makes sense for sparse meshes, but in urban environments it can cause congestion. Chelsea isn't sparse enough to justify it.

**Node name** — Changed from the default "Meshtastic ff6c" to something non-identifying. Short name is limited to 4 characters.

**Position config** — This is where things got complicated. The node has no GPS, so you either leave it with no position or set a fixed one. I disabled Smart Position (designed for mobile nodes) and set a 6-hour broadcast interval. Setting actual coordinates is another matter... more on that below.

## iOS App Limitation

Unfortunately, there was a genuine gap in the tooling: **the iOS Meshtastic app cannot manually enter fixed GPS coordinates**. This is a known missing feature, and the Android app has it but the iOS app doesn't. (Subtle W for Android)....

When you enable Fixed Position, the app just pushes your iPhone's current GPS coordinates to the node, overwriting whatever you might have set elsewhere.

The correct way to set coordinates is via the Python CLI:

```bash
meshtastic --port /dev/tty.usbserial-XXXX --setlat 51.4935 --setlon -0.1544 --setalt 15
```

In practice however, this didn't work either. The ESP32-C6 exposes a USB JTAG/CDC interface rather than a standard UART, which the Meshtastic Python CLI doesn't communicate with reliably. The device showed up in System Information as "USB JTAG/serial debug unit" (Espressif), but the CLI just hung indefinitely.

I could have gone the TCP route by enabling Wi-Fi on the node and connecting over the network, but at that point the privacy gain from an offset position is marginal. I'm already broadcasting from a known area of Chelsea, and the mesh density in London isn't high enough for precise triangulation to be a real concern.

## What LoRa Means in Practice

A few things became clear through this process that aren't obvious from the marketing:

1. **Range is real but line of sight matters.** LoRa at 868MHz can genuinely reach several kilometres in open space. Urban environments with buildings, glass, and interference are more variable. But an elevated windowsill with a view over an open square is about as good as you'll get for a static indoor deployment.

2. **The mesh is only as strong as its density.** With a sparse mesh, messages may not reach anyone. The London Meshtastic network exists but is patchy — the nearest node to Chelsea visible on meshmap.net at time of writing was in Wandsworth. Messages will transmit regardless, but without acknowledgement from another node, you can't confirm delivery.

3. **Duty cycle matters.** EU 868MHz band has a 1% duty cycle restriction — the node can only transmit for 1% of any given hour. This is why broadcast intervals for position and telemetry should be set to hours, not minutes. My configuration runs position broadcasts every 6 hours, which is sensible for a static node.

4. **The default channel is public.** Messages on the primary channel are encrypted with a well-known default key, which means anyone running Meshtastic can read them. This is intentional as it's the shared public channel for the mesh. Private channels with custom PSKs exist if you want actual confidentiality. I am thinking of setting one up for my family in the near future, once I have gotten them their own nodes.

## Current State

The node is running, connected to USB-C power, sitting on the windowsill overlooking busy London. It's transmitting but hasn't heard any other nodes yet, which is just the reality of a nascent network. Every node that goes up improves the situation for whoever deploys next though. So it feels good to contribute to this project.

If you're running a Meshtastic node in London and want to compare notes, feel free to reach out.
