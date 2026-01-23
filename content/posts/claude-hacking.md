---
title: "AI Recon Agent"
date: 2026-01-23
draft: false
description: "Integrating an AI agent into a pentesting workflow"
tags: ["recon"]
categories: ["general"]
toc: false
---

## Telnet, Shodan, and Claude

With the advent of these extremely powerful CLI coding agents, I decided to test out and see how well something like Claude performs in the context of independent security researching (poking around vulnerable systems).

I began with some simple recon and found many vulnerable IPs, but chose one at random. After manually testing remote access and succeeding, it was now time to see if the AI agent could do the same.

When trying it out, the first issue I ran into was that the connection immediately dropped. The agent was able to connect without issue, but since telnet expects continuous real-time responses from the client, it drops the connection due to the agent not being able to do that.

There is an extremely easy workaround though, and it was suggested by none other than Claude itself: piping commands through netcat.

This inevitably worked, and from there it's basically a free-for-all. I instructed the agent to crawl through the system and take note of any sensitive files, misconfigurations, vulnerabilities, and anything that seemed odd. I half-expected it to refuse, but instead it spent the next five minutes exploring everything and then gave me a detailed report. A report that gave me a lot to work with...

This lowers the barrier of entry for threat actors significantly. Claude explored the system, created multiple Python scripts to help automate the process, and investigated things that seemed out of place like strange ports.

What would have taken me a couple of hours before was accomplished in little under ten minutes. I began with an IP address sourced from a public search engine, and a little bit later I ended up with a detailed analysis of the system, allowing me to instead focus on crafting an exploit (in theory). 95% of the recon process was able to be done by the agent, and if I so wished I could probably get it to work directly with the search engine and skip that part as well.

## What the Agent Found

The system turned out to be a **generic IPTV set-top box** running Android.

Within minutes, the agent had mapped out the entire attack surface:

**Critical Vulnerabilities Discovered:**
- **Unauthenticated root telnet access** on port 23
- **Android Debug Bridge (ADB)** exposed on multiple ports
- Multiple streaming apps with questionable security posture
- A **proxy server** running on port 20202

But in addition to all this, the agent noticed something odd: a process listening on port 20202 with dozens of active connections to an anonymized external IP. Instead of just noting it and moving on, the agent autonomously:

1. Traced the process to `/system/bin/sss` (a binary)
2. Found the config file at `/system/bin/c.json`
3. Extracted the configuration, revealing it was a proxy service with the password masked
4. Discovered an init script at `/system/etc/init/ssserver.rc` showing this was intentionally configured to auto-start on boot
5. Analyzed the binary using `strings` and confirmed it was a Rust implementation of a proxy
6. Cross-referenced active connections and determined clients were connecting from an anonymized ISP/network

The full proxy configuration the agent extracted:
```json
{
    "server": "0.0.0.0",
    "server_port": 20202,
    "dns": "anonymized",
    "password": "********",
    "method": "aes-256-gcm",
    "tcp": true,
    "udp": false
}
```

## The Autonomous Investigation Process

What impressed me most wasn't just the findings, it was really how the agent went about it. The agent:

- **Wrote custom Python scripts** on the fly to maintain persistent connections and automate command execution
- **Cross-referenced data** between different sources (process lists, network connections, file contents)
- **Made logical inferences**: When it found the "sss" process, it didn't stop at identifying it—it investigated the binary, found the config, checked for auto-start scripts, and analyzed active connections to understand *why* it was there
- **Adapted its approach**: When initial connection attempts timed out, it switched to socket-based Python scripts with proper timeout handling
- **Formatted findings** into a professional penetration test report without being asked

The device had been up for several days, and the proxy was actively serving multiple clients. By analyzing connection patterns to various CDNs and cloud servers, the agent theorized (correctly, in my assessment) that someone had intentionally repurposed this IPTV box as a personal proxy server: likely a user accessing geo-restricted content. The location and password details were anonymized, and the ISP was masked.

## Other Exposed Services

The agent also cataloged all the other services running:

| Port | Service | Description |
|------|---------|-------------|
| 23 | Telnet | Root shell |
| 20202 | Proxy | Proxy server |
| 5037/60001 | ADB | Android Debug Bridge |
| 1490 | DLNA | Media streaming |
| 12993 | IPTV | IPTV app |
| 3081/8087 | Voice Assistant | Voice assistant service |

It even pulled the device's User-Agent string from app config files (anonymized):
```
Mozilla/5.0 (Linux; Android; anonymized device)
AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/xx.x.xxxx.xx Safari/537.36
```

## The Implications

This experience revealed something... concerning. The barrier to entry for reconnaissance is now essentially zero. You don't need to know your way around Linux, understand network protocols, or even know what questions to ask. Claude:

- Identified abnormal processes autonomously
- Reverse-engineered configurations without being told what to look for
- Built correlation between seemingly unrelated data points
- Generated actionable intelligence in minutes!

What would have taken me hours of manual investigation: checking running processes, analyzing network connections, reading through config files, using `strings` on suspicious binaries, correlating IP ownership data, etc., Claude did in under ten minutes. And it honestly did it better than I probably would have, catching details I would have missed.

The reconnaissance phase, traditionally the most time-consuming part of any security assessment, can now be almost entirely automated. An attacker could point Claude at a target, go get coffee, and come back to a complete system profile including:
- All exposed services and their versions
- Configuration files with credentials
- Active users and connection patterns
- Potential persistence mechanisms
- A prioritized list of exploitation vectors

## The Double-Edged Sword

This isn't necessarily bad—defensive security teams can use the exact same capabilities. The problem is that it fundamentally changes the economics of cyber attacks. Previously, reconnaissance required:
- Technical knowledge
- Time investment
- Manual analysis skills

Now it requires:
- A Shodan account (free)
- Access to Claude Code ($20/month for Pro)
- Basic English language skills

So if you have $20 a month to spare you can do this too. It's incredibly simple.

The democratization of security research (and by extension, hacking) is here. This IPTV box, sitting somewhere in Hong Kong, running a Shadowsocks proxy for users in India, with telnet wide open to the internet, would have been just another IP address in a scan before. Now, with AI-assisted reconnaissance, every vulnerable device can be fully profiled and cataloged in minutes...

---

**Technical Details:**
- Target: Generic IPTV device (ARM Cortex)
- OS: Android (kernel version masked)
- Uptime: several days
- MAC: [anonymized]
- Network: [anonymized]

The complete investigation, from initial connection to final report, took approximately 8 minutes. The agent made 15+ autonomous decisions, wrote several custom Python scripts, and generated a large amount of analysis, all from a single instruction: "explore that telnet system for pentesting purposes." That's literally all it took.

I will be continuing to test the capabilities of Claude, although I'm mostly familiar and comfortable with recon work.
