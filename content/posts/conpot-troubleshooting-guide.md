---
title: "Conpot Honeypot Troubleshooting Guide"
date: 2026-01-28
draft: false
description: "Troubleshooting Conpot traffic issues"
tags: ["ics", "industrial"]
categories: ["general"]
toc: false
---

## Issue: No Traffic Reaching Honeypot After Deployment

### Symptoms
- Honeypot container running for 30+ minutes with no log activity
- All protocols initialized successfully in logs
- No connection attempts visible in Grafana/Loki dashboards
- `docker logs conpot-hvac` shows only startup messages

---

## Root Cause Analysis

The issue typically stems from a **mismatch between internal container ports, external exposed ports, and firewall rules**.

### How Conpot Port Mapping Works

```
Internet Traffic → Firewall (UFW) → Docker Port Mapping → Conpot Container
                   [External Port]    [External→Internal]    [Internal Port]
```

**Example:**
- Conpot listens internally on port **5020** (Modbus)
- Docker maps **5020 → 502** (internal → external)
- Firewall must allow **502** (the external port)
- Internet sees your honeypot on **port 502**

---

## Step-by-Step Troubleshooting

### Step 1: Verify Container is Running

```bash
docker ps | grep conpot
```

**Expected:** Container should be `Up` and show port mappings

---

### Step 2: Check ALL Port Mappings

```bash
docker port conpot-hvac
```

**Expected Output (all 9 protocols):**
```
102/tcp -> 0.0.0.0:102
2121/tcp -> 0.0.0.0:21
5020/tcp -> 0.0.0.0:502
6230/tcp -> 0.0.0.0:623
6969/udp -> 0.0.0.0:69
8800/tcp -> 0.0.0.0:80
16100/udp -> 0.0.0.0:161
44818/tcp -> 0.0.0.0:44818
47808/udp -> 0.0.0.0:47808
```

**Problem:** If you only see 4 protocols, you didn't expose all ports when creating the container.

**Solution:** Recreate container with all port mappings (see Step 6)

---

### Step 3: Verify Docker is Listening on External Ports

```bash
ss -tulpn | grep docker-proxy
```

**Expected:** You should see `docker-proxy` listening on the **external** ports (502, 80, 161, etc.)

**Common Mistake:**
```
# WRONG - looking for internal ports
tcp   LISTEN   0.0.0.0:5020    # This won't appear - it's internal only!

# CORRECT - looking for external ports  
tcp   LISTEN   0.0.0.0:502     # This should appear
```

---

### Step 4: Check Firewall Rules Match EXTERNAL Ports

```bash
ufw status numbered
```

**Critical Rule:** Firewall must allow the **external** Docker-exposed ports, NOT the internal container ports.

**Common Mistakes:**

❌ **WRONG:**
```
ufw allow 5020/tcp    # Internal Modbus port
ufw allow 8800/tcp    # Internal HTTP port
ufw allow 16100/tcp   # Internal SNMP port
```

✅ **CORRECT:**
```
ufw allow 502/tcp     # External Modbus port
ufw allow 80/tcp      # External HTTP port
ufw allow 161/udp     # External SNMP port
```

---

### Step 5: Compare All Three Layers

Create a comparison table:

| Protocol | Internal Port | External Port | UFW Rule | Status |
|----------|---------------|---------------|----------|--------|
| Modbus   | 5020          | 502           | 502/tcp  | ✅     |
| S7Comm   | 102           | 102           | 102/tcp  | ✅     |
| HTTP     | 8800          | 80            | 80/tcp   | ✅     |
| SNMP     | 16100         | 161           | 161/udp  | ✅     |
| BACnet   | 47808         | 47808         | 47808/udp| ✅     |
| IPMI     | 6230          | 623           | 623/tcp  | ✅     |
| EtherNet/IP | 44818      | 44818         | 44818/tcp| ✅     |
| FTP      | 2121          | 21            | 21/tcp   | ✅     |
| TFTP     | 6969          | 69            | 69/udp   | ✅     |

**How to get this info:**
```bash
# Internal ports
docker logs conpot-hvac | grep "server started"

# External ports  
docker port conpot-hvac

# Firewall rules
ufw status numbered
```

---

### Step 6: Fix - Recreate Container with All Ports

If ports are missing, stop and recreate the container:

```bash
# Stop and remove old container
docker stop conpot-hvac
docker rm conpot-hvac

# Recreate with ALL port mappings
docker run -d \
  --name conpot-hvac \
  --restart unless-stopped \
  -p 102:102 \
  -p 502:5020 \
  -p 80:8800 \
  -p 161:16100/udp \
  -p 47808:47808/udp \
  -p 623:6230 \
  -p 44818:44818 \
  -p 21:2121 \
  -p 69:6969/udp \
  conpot-hvac \
  /home/conpot/.local/bin/conpot --template custom --force

# Verify all ports are mapped
docker port conpot-hvac
```

---

### Step 7: Fix - Update Firewall Rules

Remove incorrect internal port rules:

```bash
# Remove wrong rules (internal ports)
ufw delete allow 5020/tcp
ufw delete allow 8800/tcp
ufw delete allow 16100/tcp
ufw delete allow 6230/tcp
ufw delete allow 2121/tcp
ufw delete allow 6969/udp
```

Add correct external port rules:

```bash
# Add correct rules (external ports)
ufw allow 502/tcp comment 'Modbus'
ufw allow 80/tcp comment 'HTTP'
ufw allow 161/udp comment 'SNMP'
ufw allow 102/tcp comment 'S7Comm'
ufw allow 47808/udp comment 'BACnet'
ufw allow 623/tcp comment 'IPMI'
ufw allow 44818/tcp comment 'EtherNet/IP'
ufw allow 21/tcp comment 'FTP'
ufw allow 69/udp comment 'TFTP'

# Reload firewall
ufw reload

# Verify
ufw status numbered
```

---

### Step 8: Test External Connectivity

From your **local machine** (NOT the honeypot server):

```bash
# Test a few key ports are reachable
nmap -Pn -p 80,102,502 <HONEYPOT_IP>
```

**Expected:** Ports should show as `open`

**If ports show as filtered/closed:** Firewall issue or cloud provider network ACL

---

### Step 9: Monitor for Traffic

```bash
# Watch logs in real-time
docker logs -f conpot-hvac

# Check last 100 lines
docker logs --tail 100 conpot-hvac
```

**Expected Timeline:**
- **0-15 minutes:** Potentially quiet (normal)
- **15-30 minutes:** Should see first reconnaissance scans
- **30-60 minutes:** Regular scanning activity from automated tools

**Most commonly scanned ports first:**
1. Port 502 (Modbus) - heavily scanned by ICS reconnaissance tools
2. Port 102 (S7Comm) - Siemens PLC scanners
3. Port 80 (HTTP) - general web scanners

---

## Quick Reference Commands

### Container Management
```bash
# Check container status
docker ps -a | grep conpot

# View logs
docker logs conpot-hvac --tail 50

# Follow logs in real-time
docker logs -f conpot-hvac

# Restart container
docker restart conpot-hvac
```

### Port Verification
```bash
# See all container port mappings
docker port conpot-hvac

# See what's listening externally
ss -tulpn | grep docker-proxy

# Check specific port
netstat -tulpn | grep :502
```

### Firewall Management
```bash
# View firewall rules
ufw status numbered

# Add rule
ufw allow 502/tcp comment 'Modbus'

# Delete rule by number
ufw delete 7

# Reload firewall
ufw reload
```

### Network Testing
```bash
# Test from external machine
nmap -Pn -p 80,102,502,161 <HONEYPOT_IP>

# Check if port is open locally
nc -zv localhost 502

# See active connections
docker exec conpot-hvac netstat -an | grep ESTABLISHED
```

---

## Common Pitfalls

### 1. Using Internal Ports in Firewall Rules
**Symptom:** `docker port` shows mappings, but `ufw status` has internal ports

**Fix:** Always use external ports in firewall rules

### 2. Missing UDP Protocol Specifications
**Symptom:** TCP ports work but UDP ports don't

**Fix:** Always specify `/udp` for UDP ports:
```bash
ufw allow 161/udp   # CORRECT
ufw allow 161       # WRONG - defaults to TCP only
```

### 3. Incomplete Port Mappings
**Symptom:** Only 4 protocols mapped instead of 9

**Fix:** Must specify all 9 ports when creating container (see Step 6)

### 4. Docker Network Mode Issues
**Symptom:** Ports exposed but not accessible

**Fix:** Don't use `--network host` - use default bridge mode with explicit port mappings

### 5. Cloud Provider Firewall/Security Groups
**Symptom:** UFW allows ports but still no traffic

**Fix:** Check DigitalOcean firewall/security group settings in cloud console

---

## Verification Checklist

Before assuming "no traffic", verify:

- [ ] Container is running (`docker ps`)
- [ ] All 9 protocols show in `docker port conpot-hvac`
- [ ] `ss -tulpn | grep docker-proxy` shows external ports
- [ ] `ufw status` allows **external** ports, not internal
- [ ] External `nmap` scan shows ports open
- [ ] Waited at least 30 minutes for scanner discovery
- [ ] Cloud provider firewall allows traffic (if applicable)
- [ ] Grafana/Loki stack is running and ingesting logs

---

## Expected Normal Behavior

### Startup
```
2026-01-28 10:57:51,899 Modbus server started on: ('0.0.0.0', 5020)
2026-01-28 10:57:51,900 S7Comm server started on: ('0.0.0.0', 102)
2026-01-28 10:57:51,901 HTTP server started on: ('0.0.0.0', 8800)
...
[All 9 protocols should initialize]
```

### First Traffic (15-60 minutes after deployment)
```
2026-01-28 11:23:45,123 New connection from 198.51.100.45:54321 on port 502
2026-01-28 11:23:45,456 Modbus request: function_code=0x01 ...
```

### No Traffic for 40+ Minutes
**This is NORMAL** if:
- Fresh deployment (not indexed by Shodan/Censys yet)
- Non-standard ports (though you're using standard ones)
- Quiet period between scanner waves

**This is ABNORMAL** if:
- Firewall blocking (most common issue)
- Port mappings incomplete
- Cloud provider network ACL blocking

---

## Contact/Support

When reporting issues, include:
```bash
# Gather diagnostic info
echo "=== Container Status ===" && \
docker ps -a | grep conpot && \
echo -e "\n=== Port Mappings ===" && \
docker port conpot-hvac && \
echo -e "\n=== Listening Ports ===" && \
ss -tulpn | grep docker-proxy && \
echo -e "\n=== Firewall Rules ===" && \
ufw status numbered && \
echo -e "\n=== Recent Logs ===" && \
docker logs --tail 30 conpot-hvac
```

---

## Document Version
**Version:** 1.0  
**Date:** 2026-01-28  
**Author:** Nick (MSc ICS Security Research)  
**Environment:** DigitalOcean Ubuntu 24.04, Conpot custom HVAC template
