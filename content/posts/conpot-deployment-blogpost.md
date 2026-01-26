---
title: "Deploying an ICS Honeypot"
date: 2026-01-26
draft: false
description: "Deploying an ICS Honeypot: A Practical Guide to Conpot on Cloud"
tags: ["ics", "industrial", "honeypot", "security-research"]
categories: ["general"]
toc: false
---

As part of my MSc research in Information Security at Royal Holloway, University of London, I've been investigating the threat landscape facing industrial control systems (ICS) and SCADA infrastructure. One of the most effective ways to understand attacker behavior in this space is through honeypot deployment; specifically, using Conpot to emulate vulnerable industrial systems.

This post documents my process of deploying a production ICS honeypot on DigitalOcean, the technical considerations involved, and some initial observations from the deployment.

## What is Conpot?

Conpot is a low-interaction ICS/SCADA honeypot developed by the Honeynet Project. It emulates multiple industrial protocols including Modbus, S7Comm (Siemens), BACnet, SNMP, and others. Unlike high-interaction honeypots that require full system simulation, Conpot provides protocol-level emulation sufficient to attract and log attacker reconnaissance and exploitation attempts.

The project supports several templates simulating different industrial systems:
- Default (Technodrome) - multi-protocol SCADA system
- Guardian AST - tank monitoring system
- Kamstrup 382 - smart meter infrastructure
- IPMI - baseboard management controllers
- IEC 60870-5-104 - power system automation

## Threat Context: Why ICS Honeypots Matter

Recent research using Shodan has revealed thousands of internet-exposed industrial control systems worldwide. Many of these systems communicate over plaintext protocols like Modbus and DNP3, originally designed for isolated networks but now inadvertently exposed to the internet.

Particularly concerning are deployments in critical infrastructure sectors where:
- Legacy equipment remains in production for 20-40 year lifecycles
- Security patches may be unavailable or difficult to apply without operational downtime
- Default credentials remain unchanged
- Network segmentation is insufficient or non-existent

Understanding how attackers discover, fingerprint, and attempt to compromise these systems is essential for developing effective defensive strategies.

## Deployment Architecture

### Infrastructure Selection

For the initial deployment, I selected a DigitalOcean droplet with the following specifications:

- **Operating System:** Ubuntu 22.04 LTS
- **Resources:** 1GB RAM, 1 vCPU, 25GB SSD
- **Location:** London datacenter
- **Cost:** $6/month

The minimal resource allocation is intentional for now, but I will most likely end up upgrading memory sometime in the future.

### Containerization Strategy

Rather than installing Conpot directly on the host system, I opted for Docker containerization for several reasons:

1. **Isolation:** The honeypot runs in a contained environment separate from host system processes
2. **Reproducibility:** The exact deployment can be replicated or scaled across multiple instances
3. **Simplified management:** Container lifecycle operations (start, stop, restart) are straightforward
4. **Log persistence:** Volume mounts allow log data to persist outside the container

### Network Configuration

The deployment exposes nine distinct services across different protocols:

```
HTTP (8800/tcp)       - Web-based control interface
S7Comm (102/tcp)      - Siemens PLC protocol
Modbus (502/tcp)      - Widely-used industrial protocol
SNMP (161/udp)        - Network device monitoring
BACnet (47808/udp)    - Building automation
IPMI (623/udp)        - Server management interface
FTP (21/tcp)          - File transfer
TFTP (69/udp)         - Trivial file transfer
EtherNet/IP (44818/tcp) - Industrial Ethernet protocol
```

Firewall rules (UFW) were configured to allow these specific ports while blocking all other inbound traffic except SSH on port 22 (for access to the VM).

## Implementation Process

### Initial Server Hardening

Before deploying the honeypot, basic server hardening was implemented:

```bash
# System updates
apt update && apt upgrade -y

# Firewall configuration
ufw allow OpenSSH
ufw allow 8800/tcp
ufw allow 102/tcp
ufw allow 502/tcp
# ... additional honeypot ports
ufw enable
```

While this is a honeypot designed to be discovered, the SSH service and host system *very importantly* still require protection from unauthorized access.

### Docker Installation and Configuration

Docker was installed using the official installation script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

Then the official Conpot image was pulled from Docker Hub:

```bash
docker pull honeynet/conpot:latest
```

### Addressing Permissions Challenges

An initial deployment attempt revealed a common containerization issue - the Conpot process inside the container lacked write permissions to the mounted log directory. This manifested as repeated crashes with `PermissionError: [Errno 13] Permission denied: '/var/log/conpot/conpot.log'`.

The solution involved setting appropriate permissions on the host directory before mounting:

```bash
chmod 777 ~/conpot-deployment/logs/
```

While this is permissive, it's acceptable for a honeypot log directory where the data itself is not operationally critical to the host system.

### Production Deployment

The final deployment command launches Conpot as a detached container with automatic restart capabilities:

```bash
docker run -d \
  --name conpot \
  --restart unless-stopped \
  -p 8800:8800 \
  -p 102:10201 \
  -p 502:5020 \
  -p 161:16100/udp \
  -p 47808:47808/udp \
  -p 623:6230/udp \
  -p 21:2121 \
  -p 69:6969/udp \
  -p 44818:44818 \
  -v ~/conpot-deployment/logs:/var/log/conpot \
  honeynet/conpot:latest
```

Note the port mappings: Conpot runs services on non-standard internal ports, which are then mapped to their standard protocol ports on the host. This allows the container to run unprivileged while still exposing services on ports below 1024.

## Operational Considerations

### Log Management

With only 25GB of disk space (for now), log management is critical. A daily backup script with automatic retention policies was implemented:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
tar -czf ~/backups/conpot-logs-$DATE.tar.gz ~/conpot-deployment/logs/
find ~/backups/ -name "conpot-logs-*.tar.gz" -mtime +30 -delete
```

This script, executed via cron at 02:00 daily, ensures logs are backed up while automatically purging backups older than 30 days.

### Resource Monitoring

Given the 1GB RAM constraint, monitoring is essential:

```bash
# Container resource usage
docker stats conpot

# Disk utilization
du -sh ~/conpot-deployment/logs/
df -h
```

If/when sustained high activity is observed, upgrading to the 2GB RAM tier ($12/month) will most likely be necessary to maintain stability.

### Indexing and Discovery

Internet-wide scanning services like Shodan and Censys continuously probe IPv4 address space for exposed services. For the most part, a newly deployed honeypot will be indexed within 24-48 hours of going live. First attacker interactions usually occur within 24-72 hours.

Verification can be performed via Shodan's host lookup:
```
https://www.shodan.io/host/[HONEYPOT_IP]
```

## Why S7-200 Emulation Remains Relevant

A valid question came to me during deployment: isn't emulating a discontinued PLC (S7-200, end-of-life 2009) too obvious an indicator that this is a honeypot?

In reality however:

**Industrial systems have exceptionally long operational lifespans.** Control systems deployed in the mid-2000s remain in production across manufacturing, utilities, and building automation sectors. Equipment replacement cycles in these environments span 20-40 years, not the 3-5 year cycles common in IT infrastructure.

**Legacy systems persist in developing regions.** OSINT research using Shodan reveals significant numbers of aging Siemens PLCs exposed on the internet, particularly in Russia, Eastern Europe, and other regions where Western sanctions, budget constraints, or maintenance practices have left outdated infrastructure operational.

**Attackers target old systems preferentially.** From an attacker's perspective, an S7-200 represents a more attractive target than newer equipment - it's more likely to have known vulnerabilities, default credentials, and inadequate security monitoring.

The prevalence of real S7-200 systems on the internet validates the honeypot's plausibility.

## Research Objectives

This deployment serves several research objectives for my MSc thesis:

1. **Attack pattern analysis:** Document the techniques, tactics, and procedures used against ICS infrastructure
2. **Protocol exploitation:** Identify which industrial protocols attract the most attention and what exploits are attempted
3. **Geographic distribution:** Analyze the source countries and networks conducting ICS reconnaissance
4. **Temporal patterns:** Understand when attacks occur and whether they're automated or human-driven
5. **Threat actor profiling:** Distinguish between opportunistic scanning, research activities, and targeted attacks

## Initial Observations

The deployment went live on January 27, 2025. Early verification confirmed all services are responding correctly:

- HTTP interface accessible on port 8800
- Nmap scans successfully identify open ports
- Container logs show services initialized without errors
- Log files are being written to persistent storage

The next phase involves monitoring for first contact from external scanners and documenting the progression from discovery to interaction attempts.

## Security and Ethical Considerations

Several important considerations apply to honeypot research:

**Isolation:** The honeypot runs on dedicated infrastructure with no access to production systems or sensitive data.

**Legal compliance:** Deployment adheres to institutional policies and does not violate Digital Ocean's TOS.

**Data handling:** All collected data (attacker IPs, payloads, session logs) is handled in accordance with research ethics guidelines and applicable data protection regulations.

**No offensive capabilities:** This is a passive monitoring system that logs attacker behavior but *does not* engage in active response or counterattacks.

## Takeaways

I learned several things during the deployment:

1. **Docker volume permissions matter:** Container user IDs may not match host user IDs, requiring explicit permission configuration for mounted volumes.

2. **Port mapping nuances:** Understanding the difference between container-internal ports and host-exposed ports is essential for proper service exposure.

3. **Template selection is contextual:** While specific vendor templates exist, the default multi-protocol template may better serve broad research objectives.

4. **Resource constraints drive discipline:** Limited disk space forces implementation of proper log rotation and backup strategies from the start.

5. **Patience required:** Unlike web application honeypots that see immediate traffic, ICS honeypots may take days to be discovered and weeks to see significant interaction.

## Next Steps

With the honeypot operational, my research enters its data collection phase:

- **Week 1:** Monitor for Shodan/Censys indexing and first external contacts
- **Week 2-4:** Analyze early attack patterns and refine logging configurations
- **Ongoing:** Develop automated analysis scripts for log parsing and visualization
- **Long-term:** Consider deploying additional instances with different templates or geographic locations for comparative analysis

## Ending Notes

Running an ICS honeypot provides valuable insight into the real-world threat landscape facing industrial control systems. While the technical deployment is for the most part straightforward - Docker container, open ports, wait for attackers - the research value lies in careful analysis of the collected data.

As industrial systems continue their slow migration toward internet connectivity, often without corresponding security improvements, understanding attacker behavior becomes increasingly critical. Honeypots like Conpot offer a low-risk, high-value method for gaining this understanding.

---

**Resources:**
- Conpot GitHub: https://github.com/mushorg/conpot
- Conpot Docker Hub: https://hub.docker.com/r/honeynet/conpot
- Honeynet Project: https://www.honeynet.org
- Shodan: https://www.shodan.io

*This research is being conducted as part of an MSc in Information Security at Royal Holloway, University of London.*
