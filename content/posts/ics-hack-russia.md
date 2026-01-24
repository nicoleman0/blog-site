---
title: "A Look at Internet-Exposed ICS in Russia"
date: 2026-01-24
draft: false
description: "Why ICS remain a critical security concern"
tags: ["recon", "ai-hacking", "claude", "russia", "ics", "industrial"]
categories: ["general"]
toc: false
---

## A Look at Internet-Exposed ICS in Russia

During recent OSINT research using Shodan, I discovered a Siemens Desigo building automation controller within Russia sitting directly on the public internet. What started as academic reconnaissance quickly revealed a textbook example of how industrial control systems should never be deployed...

## Discovery

The controller was identified through Shodan queries targeting building automation systems. The device in question was a Siemens Desigo PXC50-E.D controller, part of a widely-deployed building management platform used for HVAC control, scheduling, and facility monitoring. Based on network analysis, this appeared to be part of a large industrial or agricultural deployment in the suburbs of Moscow, with approximately 11 similar devices exposed on the same network segment.

What made this device interesting from a research perspective was not just its exposure, but the combination of vulnerabilities and misconfigurations that created multiple attack paths.

## The Attack Surface

### Web Interfaces Everywhere

The controller exposed two distinct web interfaces:

**Port 80/443:** The main Desigo management interface, featuring a standard login form with site selection dropdown. The interface revealed the site name "GoldTown" and accepted authentication via POST request to `/login.fn`.

**Port 81:** A touch panel HMI providing direct access to building automation controls including plant visualization, alarm management, HVAC scheduling, trend logging, and critically, a settings panel.

### The Hidden Features Problem

Examining the HTML source of the settings panel revealed dormant remote access capabilities:

```html
<a href="#" class="operation-button settings-button vnc hidden">Enable VNC</a>
<a href="#" class="operation-button settings-button ssh hidden">Enable SSH</a>
```

These hidden buttons represent a significant security concern. An authenticated attacker could enable VNC or SSH for persistent remote access to the controller, bypassing the need for repeated web authentication.

## Authentication Failures

The most significant vulnerability was the complete absence of brute force protection. Testing with Hydra demonstrated:

- 680+ login attempts without any blocking mechanism
- Sustained rate of approximately 34 attempts per minute
- No CAPTCHA, rate limiting, or account lockout
- No apparent monitoring or alerting on failed authentication

This makes credential attacks trivial. Given enough time and a comprehensive wordlist, discovering valid credentials becomes a matter of when, not if.

The system also exhibited other authentication weaknesses including sequential session IDs (observed: sId=26693), potential for session prediction, and no visible CSRF protection on the login form.

## Information Disclosure

The system leaked several pieces of reconnaissance-valuable information:

- **Firmware version:** 01.15.35.266
- **Internal IP address:** 192.168.1.90 (exposed in HTTP headers)
- **Network topology:** Through WHOIS and neighboring host scans
- **Building architecture:** Via the plant view interface

This information disclosure reduces the effort required for targeted attacks and provides intelligence that could support physical intrusion planning.

## What They Got Right

Not everything was misconfigured. The industrial control protocols were properly segmented from internet access:

- BACnet/IP (UDP 47808): Firewalled
- Modbus (UDP 502): Firewalled
- SNMP (UDP 161/162): Firewalled
- OPC-UA (UDP 4840): Firewalled

This demonstrates that someone understood the need to protect industrial protocols from direct internet exposure. The web interface exposure appears to be either an oversight or a misguided attempt at remote access without proper VPN infrastructure.

## Impact Assessment

The practical impact of compromising this system extends beyond simple unauthorized access:

**Operational disruption:** DoS attacks against the web interface could prevent monitoring and control of building systems. Slowloris testing confirmed susceptibility to connection exhaustion attacks.

**Climate manipulation:** Authenticated access would allow modification of HVAC schedules, setpoint adjustments, and potentially causing equipment damage through improper operation sequences, or worse.

**Intelligence gathering:** The exposed interfaces provide detailed information about building operations, schedules, and system architecture that could facilitate physical intrusion or sabotage.

**Persistent access:** The hidden VNC/SSH capabilities would allow an attacker to maintain long-term access beyond initial compromise.

## Security Lessons

This case study illustrates several fundamental security principles:

**Network segmentation** - Building automation systems should never be directly internet-accessible. VPN access or jump hosts provide the necessary remote access capability without exposing ICS devices.

**Defense in depth** - Even if network segmentation fails, rate limiting and account lockout would significantly raise the bar for attackers. The absence of basic authentication controls turned potential attacks into trivial exercises.

**Visibility is crucial** - There was no evidence of logging or monitoring on failed authentication attempts. Without visibility, defenders cannot detect active attacks or respond appropriately.

**Default configurations are dangerous** - The presence of hidden remote access features suggests default or factory configurations. Organizations must review and disable unnecessary functionality.

**Firmware updates** - Several known CVEs affect older Siemens Desigo firmware. A patch would fix that (not really applicable to the Russians though).

## Disclosure and Ethics

This research was conducted as part of academic OSINT work on internet-exposed industrial systems in Russia.

No credentials were compromised, no systems were accessed beyond publicly-available interfaces, and no disruption was caused. The findings have been documented for educational purposes only.

Organizations discovering similar exposures in their own environments should:

1. Immediately remove ICS devices from public internet access
2. Implement VPN or jump host access for legitimate remote needs
3. Enable rate limiting and account lockout on all authentication interfaces
4. Update firmware to patch known vulnerabilities
5. Review audit logs for signs of previous compromise
6. Consider engaging third-party security assessments

## Conclusion

In 2026, we somehow continue to find industrial control systems exposed to the internet with minimal security controls. This particular Siemens Desigo deployment represents a near-perfect storm of misconfigurations: internet exposure, no authentication protections, information disclosure, and hidden remote access capabilities.

### The Russia Factor

The case makes a lot more sense however when taking into consideration that this system is located in Russia.

Since 2022, Siemens has officially wound down its industrial operations in Russia, with its Russian entity (formerly Siemens LLC, later Systems LLC) entering voluntary liquidation in 2024 due to the inability to legally import equipment. This creates a unique security paradox for Russian organizations.

The equipment is aging, and very hard for them to replace. While Russia has successfully circumvented sanctions for military-critical items through complex third-party networks (particularly via China, Kazakhstan, and Hong Kong), building automation systems are far down the priority list. The defense industrial base gets the smuggled Siemens CNC controllers and semiconductors. Like always, civilian infrastructure gets what's already installed.

Additionally, domestic alternatives are basically non-existent. Russian building automation capabilities lag significantly behind Western manufacturers like Siemens, Honeywell, and Schneider Electric. Import substitution programs that have shown some success in defense sectors (UAVs, artillery munitions) have largely failed in industrial automation. Without access to genuine replacement parts, firmware updates, or technical support from Siemens, these systems will continue degrading with no clear upgrade path.

Organizations running these controllers cannot realistically update firmware to address known vulnerabilities. They cannot obtain official Siemens support for security configurations. They cannot replace aging hardware before it fails. The rational response might seem to be removing internet access entirely—but the operational need for remote monitoring remains, and VPN infrastructure requires expertise and investment that may not be available for smaller to medium sized companies.

This creates a unique situation where critical infrastructure becomes progressively more vulnerable over time, trapped between operational necessity and security reality. The exposed Desigo controller we examined may represent not just poor initial deployment decisions, but the practical constraints of maintaining industrial systems under heavy sanctions.

The broader implications are quite significant... Western sanctions aimed at degrading Russian military capabilities have the unintended consequence of freezing civilian infrastructure in an increasingly vulnerable state. As legitimate support channels disappear and equipment ages without replacement paths, we can expect to see more of these systems remaining exposed out of necessity rather than choice.

For security researchers, internet-exposed ICS remains a target-rich environment, particularly in jurisdictions facing technology isolation like Russia or Iran.

For the Russians, it's a reminder that geopolitics and supply chain disruptions create security challenges that go far beyond technical controls.

---

**Technical Details**

For those interested in the specific methodology:

- **Reconnaissance:** Shodan queries targeting Siemens Desigo systems
- **Scanning:** Nmap service and vulnerability scans
- **Analysis:** Manual examination of web interfaces and HTML source
- **Testing:** Hydra brute force testing (no valid credentials discovered)
- **Protocol analysis:** BACnet discovery attempts (all firewalled)

**References**

- CISA Advisory ICSA-25-135-04: Siemens Desigo
- CISA Advisory ICSA-16-355-01: Siemens Desigo PX Web Module  
- Nozomi Networks: Siemens Desigo Vulnerabilities
- CVE-2007-6750: Slowloris DoS
- CVE-2019-13927: HTTP DoS (firmware < V6.00.320)

---

*This research was conducted for educational purposes as part of graduate-level cybersecurity coursework. All testing remained within ethical boundaries and caused no disruption to operational systems.*
