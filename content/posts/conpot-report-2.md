---
title: "Conpot Honeypot: Automated Reconnaissance and ICS Probing Analysis"
date: 2026-01-28
author: Nicholas
tags: ["Security", "ICS", "Honeypot", "Threat-Intelligence"]
description: "Analysis of multi-vector attack traffic against a Conpot deployment, featuring sensitive file fuzzing, IoT botnet recruitment attempts, and BACnet protocol fingerprinting."
---

# Conpot Honeypot: Automated Reconnaissance and ICS Probing Analysis

Analysis of multi-vector attack traffic against a Conpot deployment, featuring sensitive file fuzzing, IoT botnet recruitment attempts, and BACnet protocol fingerprinting.

**January 28, 2026** · **8 min** · **Nicholas**

## Attack Overview

Following the initial deployment phase, the honeypot has seen a pivot from general noise to more targeted scanning. In the latest session, the system attracted a series of coordinated bursts aimed at both the web management interface and industrial automation protocols.

**Session Breakdown:**
* **Industrial Protocol Probing:** 1 session (BACnet)
* **Web Environment Fuzzing:** 20+ requests (Laravel/Symfony focus)
* **IoT Exploitation Probing:** 1 session (Hikvision)

---

## Web Application Attack Patterns

### Laravel & Symfony Credential Harvesting
**Time:** 14:40:00 - 14:40:06 UTC  
**Source:** `78.153.140.151` (Hostglobal.plus Ltd, UK)  
**Severity:** HIGH  

A high-frequency automated scanner targeted the honeypot's web root, specifically hunting for environment configuration files. 

| Timestamp | Request Path |
| :--- | :--- |
| `14:40:00` | `GET /.env` |
| `14:40:01` | `GET /.env.example` |
| `14:40:02` | `GET /?url=.env` |
| `14:40:03` | `GET /phpinfo.php` |
| `14:40:05` | `GET /.env.production` |

**Technical Analysis:**
The attacker utilized a multi-threaded fuzzer to check for exposed `.env` files across various subdirectories (`/vendor/`, `/app/`, `/api/`, `/staging/`). These files are high-value targets as they typically contain database credentials, App Secrets, and API keys.

**Threat Intelligence Note:**
While some intelligence feeds (CrowdSec) have associated this IP with "Apple AI Crawlers," the behavior—specifically the rapid probing of `/vendor/.env` and `/phpinfo.php`—is strictly indicative of a **vulnerability scanner**. The attacker rotated through several spoofed User-Agents (including legacy mobile browsers like SymbianOS and IE Mobile) to evade basic fingerprinting.

### Hikvision SDK Exploitation Attempt
**Time:** 15:18:54 UTC  
**Source:** `5.61.209.92` (Sperantus, DE)  
**Severity:** HIGH  

**Log Signature:** `GET /SDK/webLanguage HTTP/1.1`

**Technical Analysis:**
This is a targeted probe for **Hikvision** IP cameras and DVRs. The attacker is checking for the existence of the Hikvision web SDK, likely as a precursor to exploiting known vulnerabilities such as **CVE-2021-36260**. Successful identification typically leads to the deployment of Mirai or Gafgyt variants to recruit the device into a botnet.

---

## Industrial Protocol Attacks

### BACnet Protocol Fingerprinting
**Time:** 15:26:48 UTC  
**Source:** `199.45.154.179` (Censys, HK)  
**Severity:** MEDIUM (Research)  

The honeypot logged a BACnet/IP connection that triggered a PDU decoding error in the protocol stack:

> **Log:** `DecodingError - PDU: <0a.00.11.01.04.00.05.01.0c.0c.02.3f.ff.ff.19.4b>`

**Technical Analysis:**
The PDU hex breakdown reveals a **BACnet "Who-Is"** broadcast (BVLC Function 0x00). The scanner is attempting to discover any BACnet-compliant devices on the network.
* **Function:** Write-Broadcast
* **Object ID:** Device (0x0C)
* **Instance:** Wildcard (0x3FFFFF)

**Significance:**
This activity originated from **Censys**, a legitimate security research platform. While not malicious, it confirms the honeypot's visibility in global ICS mapping. If the honeypot responds correctly, it will be indexed as an active Building Automation controller, which inevitably leads to follow-up probes from malicious actors.

---

## Threat Intelligence Indicators

### High-Priority Indicators of Compromise (IOCs)

| IP Address | Activity | Threat Type | Severity |
| :--- | :--- | :--- | :--- |
| **78.153.140.151** | `.env` / Secret Fuzzing | Credential Hunter | **HIGH** |
| **5.61.209.92** | Hikvision SDK Probe | IoT Botnet | **HIGH** |
| **204.76.203.18** | `/bins/` Directory Scan | Malware Staging | **MEDIUM** |

**VirusTotal Findings:**
* `78.153.140.151`: Heavily flagged (20+ engines) for malicious scanning and SSH brute-force.
* `5.61.209.92`: Identified by multiple vendors as a C2 or botnet member.

---

## Key Findings

1.  **Secret Hunting is Immediate:** Automated scanners began searching for `.env` files across a dozen different path variations within hours of the honeypot being reachable.
2.  **Protocol Diversity:** The transition from HTTP-based attacks to BACnet-specific probing shows that actors are actively scanning for non-IT assets.
3.  **Masquerading:** Malicious scanners are increasingly using hosting provider IPs while spoofing headers to appear as legacy mobile devices or search crawlers.

## Next Steps

* **Apply HTTP Error Fix:** Address the `DecodingError` in the protocol stack to prevent fingerprinting of the honeypot's backend.
* **Expand Environment Templates:** Add dummy `.env` files with "canary" credentials to track post-extraction behavior.
* **BACnet Simulation:** Improve the BACnet template to provide a valid "I-Am" response to increase visibility in research indexes.