---
title: "Telnet, Shodan, and Claude"
date: 2026-01-23
draft: false
description: "Claude is too good at reconnaissance"
tags: ["recon", "ai-hacking", "claude", "telnet"]
categories: ["general"]
toc: false
---

## Telnet, Shodan, and Claude

With the advent of these extremely powerful CLI coding agents, I decided to test out and see how well something like Claude performs in the context of independent security researching (poking around vulnerable systems).

I began with some simple recon. I went over to [shodan](https://shodan.io), and looked around for some vulnerable systems running Telnet with root already logged in (not going to post about how to do this, but it's not hard).

I found many vulnerable IPs, but chose one in Hong Kong at random. After manually testing whether or not I could remotely access the system and succeeding, it was now time to see if Claude could do the same.

When trying it out, the first issue I ran into was that the connection immediately dropped. Claude was able to connect without issue, but since telnet expects continuous real-time reponses from the client, it drops the connection due to Claude not being able to do that.

There is an extremely easy workaround though, and it was suggested by none other than Claude itself: piping commands through netcat.

This inevitably worked, and from there it's basically a free-for-all. I instructed Claude to crawl through the system and take note of any sensitive files, misconfigurations, vulnerabilities, and anything that seemed odd. I half-expected Claude to tell me that it wasn't allowed to do so, but instead it spent the next five minutes exploring everything and then gave me a detailed report. A report that gave me a lot to work with...

This lowers the barrier of entry for threat actors significantly. Claude explored the system, created multiple Python scripts to help automate the process, and investigated things that seemed out of place like strange ports.

What would have taken me a couple of hours before was accomplished in little under ten minutes. I began with an IP address sourced from shodan, and a little bit later I ended up with a detailed analysis of the system, and if I was a legitimate threat actor - this would give me a lot more time to work on crafting an exploit. 

95% of the recon process was able to be done by Claude, and if I so wished I could probably get Claude to work directly with shodan and skip the manual search part as well.

## What Claude Found

The system turned out to be a **Skyworth E900V22C IPTV set-top box** running Android 9, with a generic hostname located at IP XXX[.]XXX[.]XXX[.]XXX. Within minutes, Claude had mapped out the entire attack surface:

**Critical Vulnerabilities Discovered:**
- **Unauthenticated root telnet access** on port 23 (the obvious one)
- **Android Debug Bridge (ADB)** exposed on ports 5037 and 60001
- Multiple IPTV streaming apps with questionable security posture
- A **Shadowsocks proxy server** running on port 20202 (this one was interesting)

Everything so far has been pretty cut and dry. However, here's where it got interesting.

Claude noticed something odd: a process called "sss" listening on port 20202 with dozens of active connections to an IP in India. Instead of just noting it and moving on, Claude autonomously:

1. Traced the process to `/system/bin/sss` (a 7.2MB binary)
2. Found the config file at `/system/bin/c.json`
3. Extracted the configuration, revealing it was **shadowsocks-rust** with a weak password
4. Discovered an init script at `/system/etc/init/ssserver.rc` showing this was intentionally configured to auto-start on boot
5. Analyzed the binary using `strings` and confirmed it was the Rust implementation of Shadowsocks
6. Cross-referenced active connections and determined clients were connecting from an Indian ISP network

The full Shadowsocks configuration Claude extracted:
```json
{
    "server": "0.0.0.0",
    "server_port": 20202,
    "dns": "cloudflare",
    "password": "[REDACTED]",
    "method": "aes-256-gcm",
    "tcp": true,
    "udp": false
}
```

## The Autonomous Investigation Process

What impressed me most wasn't just the findings (which are relatively simple to get from a Telnet system that's been logged in as root), it was really how Claude went about it. The agent:

- **Wrote custom Python scripts** on the fly to maintain persistent connections and automate command execution
- **Cross-referenced data** between different sources (process lists, network connections, file contents)
- **Made logical inferences**: When it found the "sss" process, it didn't stop at identifying it—it investigated the binary, found the config, checked for auto-start scripts, and analyzed active connections to understand *why* it was there
- **Adapted its approach**: When initial connection attempts timed out, it switched to socket-based Python scripts with proper timeout handling
- **Formatted findings** into a professional penetration test report without being asked

The device had been up for 3 days, and the proxy was actively serving multiple clients. By analyzing connection patterns to Cloudflare CDN and Tencent Cloud servers, Claude theorized (correctly, in my assessment) that someone had intentionally repurposed this cheap IPTV box as a personal proxy server: likely someone in India accessing Chinese content that is geo-restricted to China. The Hong Kong IP location gives them an IP that allows them to view the restricted content.

I was confused at first, since I didn't know India had its own content restrictions that an HK proxy would help bypass (Chinese streaming platforms, etc.)

So more or less, an Indian user is watching Chinese content that is geo-restricted, by using a proxy server in Hong Kong to get around it.

## Other Exposed Services

Claude also cataloged all the other services running:

| Port | Service | Description |
|------|---------|-------------|
| 23 | Telnet | Root shell (our entry point) |
| 20202 | Shadowsocks | Proxy server |
| 5037/60001 | ADB | Android Debug Bridge |
| 1490 | DLNA | Media streaming |
| 12993 | IPTV | com.xiaojie.tv |
| 3081/8087 | Voice Assistant | iFlytek Xiri |

It even pulled the device's User-Agent string from app config files:
```
Mozilla/5.0 (Linux; Android 9; E900V22C Build/PPR1.180610.011; wv)
AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/83.0.4103.120 Safari/537.36
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
- Target: Skyworth E900V22C (Amlogic 905L ARM Cortex-A53)
- OS: Android 9 (kernel 4.9.113, built Sept 2021)
- Uptime: 3+ days
- MAC: [REDACTED]
- Network: [REDACTED]

The complete investigation, from initial connection to final report, took approximately 8 minutes. Claude made 15+ autonomous decisions, wrote 6 custom Python scripts, and generated over 40,000 tokens of analysis, all from a single instruction: "explore that telnet system for pentesting purposes." That's literally all it took.

Clearly in this case the entire process was made a lot simpler by the fact that the target system had Telnet enabled. However, this is just a simple example to demonstrate the power of Claude to speed up the entire hacking process. In my opinion, the ability to speed up the recon phase this rapidly is revolutionary itself.

I will be continuing to test the capabilities of Claude, although I'm mostly familiar and comfortable with recon work.
