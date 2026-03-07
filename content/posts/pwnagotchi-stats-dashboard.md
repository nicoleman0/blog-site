---
title: "Building a Pwnagotchi Stats Dashboard"
date: 2026-03-07
tags: ["pwnagotchi", "python", "wifi", "tools", "warwalking"]
description: "Parsing handshake pcaps into a visual HTML dashboard with tshark and Chart.js."
---

I've been warwalking around London with my Pwnagotchi (named Bootsy) and ended up with a directory full of `.pcap` files and no clean way to visualise what I'd actually captured. This post documents how I built a single-script tool to parse those captures and generate a self-contained HTML dashboard.

## The Problem

Pwnagotchi stores captured handshakes as individual `.pcap` files, named in a loosely consistent format:

```bash
SSID_aabbccddeeff.pcap
HASH_SSID_aabbccddeeff.pcap
HASH_aabbccddeeff.pcap
```

The BSSID is always the last underscore-separated segment — a 12-character hex string without colons. SSIDs appear before it, sometimes prefixed with what looks like a session hash. There's no single canonical format, which meant any parsing logic had to handle all three variants.

The goal was to turn this pile of pcaps into something that answered basic questions: how many unique networks have I captured, what vendors dominate, what's the handshake type breakdown, and how has capture rate changed over time?

## Approach

The script uses `tshark` for pcap parsing rather than `scapy`. Scapy is more portable but considerably slower when you're iterating over hundreds of files. tshark gets in, extracts what it needs, and gets out.

For each pcap, the script runs three tshark passes:

1. Extract frame timestamps to get first-seen time and frame count
2. Filter for EAPOL frames to detect 4-way handshakes
3. Filter for `wlan.rsn.ie.pmkid` to detect PMKID captures

Vendor lookups go through `api.macvendors.com` using the OUI prefix (first 6 hex chars of the BSSID), with an in-session cache to avoid hammering the API for duplicate vendors. The `--no-oui` flag skips this entirely if you're offline or want speed.

## Filename Parsing

The initial version used a regex looking for colon-separated MAC addresses in filenames, which was completely wrong for Pwnagotchi's actual naming scheme. The fix was simpler:

```python
def bssid_from_filename(filename: str) -> str | None:
    stem = Path(filename).stem
    last = stem.split("_")[-1].lower()
    if re.fullmatch(r"[0-9a-f]{12}", last):
        return ":".join(last[i:i+2] for i in range(0, 12, 2))
    return None

def ssid_from_filename(filename: str) -> str | None:
    stem = Path(filename).stem
    parts = stem.split("_")
    if len(parts) < 2:
        return None
    if re.fullmatch(r"[0-9a-f]{8}", parts[0].lower()) and len(parts) >= 3:
        ssid_parts = parts[1:-1]
    else:
        ssid_parts = parts[:-1]
    return "_".join(ssid_parts) if ssid_parts else None
```

The BSSID is always last. The SSID is everything in between — skipping the leading hash segment if present.

## Output

The script generates a single self-contained HTML file with Chart.js for visualisation. No dependencies to install on the viewer side, no server required.

Charts included:

- **Captures per month** — bar chart, useful for seeing how active sessions have been over time
- **Handshake type breakdown** — doughnut chart splitting EAPOL vs PMKID vs unknown
- **Top 10 vendors** — horizontal bar chart with per-vendor colour coding

There's also a full sortable, searchable network table covering BSSID, SSID, vendor, handshake type, frame count, and first-seen timestamp.

The dashboard is styled with Catppuccin Mocha throughout - mauve accent on the stat cards, scanline overlay for texture, Space Mono for the monospace elements and Syne for display headings.

## Usage

```bash
python3 pwnagotchi_stats.py ~/handshakes/ -o dashboard.html

# skip OUI lookups
python3 pwnagotchi_stats.py ~/handshakes/ -o dashboard.html --no-oui
```

## Legality

To be 100% frank, warwalking in the UK sits in a grey legal area. The relevant legislation is the Computer Misuse Act 1990, which criminalises unauthorised access to computer material. But passive capture of probe requests and handshakes during association is not the same as accessing a network. I'm not connecting to anything, not decrypting traffic in real time, and not using any captured material to gain access to networks I don't own. This is pretty important.

To keep things clean:

- Bootsy captures handshakes passively as part of normal monitor mode operation. I don't direct it at specific targets.
- The dashboard produced by this script contains no keys. Only metadata derived from the pcap files themselves (BSSIDs, SSIDs, timestamps, frame counts, vendor prefixes).
- I'm not acting on any of this data beyond the statistical analysis documented here.

The closest legal precedent in the UK is the long-running ambiguity around wardriving itself. Courts have generally not pursued passive capture without evidence of intent to access. That said, I'm not a lawyer, and if you're doing something similar you should read the CMA 1990 yourself and make your own judgement call.

- If a pcap contains no EAPOL frames and no PMKID field, it gets tagged as "Unknown" type. This happens with half-handshakes or corrupted captures — it's not an error, just incomplete data.
- OUI lookups require network access. The cache is in-session only, so repeated runs will re-query.
