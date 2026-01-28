# Monitoring my ICS Honeypot with Grafana, Loki, and Promtail

## Introduction

As you know, I recently deployed a customized Conpot honeypot simulating a Siemens S7-1200 HVAC controller. The honeypot exposes multiple industrial protocols including Modbus, S7comm, BACnet, SNMP, HTTP, FTP, TFTP, and IPMI to attract and analyze attacks against building automation systems.

To monitor this honeypot in real-time and analyze attack patterns, I set up a logging and visualization stack using Grafana, Loki, and Promtail. This post documents the complete setup process, a critical production issue I encountered, and how I resolved it.

## Architecture Overview

The monitoring stack consists of three main components:

- **Conpot**: The honeypot container generating logs from attacker interactions
- **Promtail**: Log collection agent that scrapes Conpot logs
- **Loki**: Log aggregation backend that stores and indexes logs
- **Grafana**: Visualization dashboard for querying and displaying log data

All components run as Docker containers on a single DigitalOcean droplet in London.

## Initial Setup

### Directory Structure

```
/opt/monitoring/
├── docker-compose.yml
├── loki-config.yml
└── promtail-config.yml

/root/conpot-deployment/
├── docker-compose.yml
├── Dockerfile
├── conpot.cfg
├── logs/
│   └── conpot.log
└── templates/
```

### Loki Configuration

I configured Loki 2.9.0 with local filesystem storage and a 30-day retention period for analyzing attack trends over time:

```yaml
# /opt/monitoring/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    cache_location: /tmp/loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: filesystem
  filesystem:
    directory: /tmp/loki/chunks

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: filesystem

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  retention_period: 720h
```

### Promtail Configuration

Promtail monitors the Conpot log file and forwards entries to Loki:

```yaml
# /opt/monitoring/promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: conpot
    static_configs:
      - targets:
          - localhost
        labels:
          job: conpot
          __path__: /var/log/conpot/*.log
```

### Docker Compose Configuration

The monitoring stack runs as three interconnected containers:

```yaml
# /opt/monitoring/docker-compose.yml
version: "3"

services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=changeme
    restart: unless-stopped

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /root/conpot-deployment/logs:/var/log/conpot:ro
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped

volumes:
  loki-data:
  grafana-data:
```

### Conpot Configuration

The Conpot honeypot is configured to write logs to a file that Promtail can access:

```yaml
# /root/conpot-deployment/docker-compose.yml (excerpt)
services:
  conpot-hvac:
    image: conpot-hvac:latest
    container_name: conpot-hvac
    restart: unless-stopped
    command: ["/home/conpot/.local/bin/conpot", "--template", "custom", "--logfile", "/var/log/conpot/conpot.log", "-f", "--temp_dir", "/tmp"]
    volumes:
      - ./logs:/var/log/conpot
```

The key here is the volume mount `./logs:/var/log/conpot` which allows logs written inside the container to be accessible on the host filesystem at `/root/conpot-deployment/logs`.

## Initial Deployment

After configuring all components, I started the stack:

```bash
cd /opt/monitoring
docker-compose up -d

cd /root/conpot-deployment
docker-compose up -d
```

I configured Grafana with a Loki data source and created dashboards showing:
- Industrial protocol activity timeline
- Attack distribution by protocol (pie chart)
- Top interacting IP addresses
- User-agent strings from HTTP attacks
- Attack volume over time
- Activity logs with filtering

The setup worked perfectly, collecting data on reconnaissance activities from various actors including Censys scanners, Palo Alto Networks Cortex Xpanse, and various automated attack tools.

## The Production Issue

After running successfully for approximately 13 hours, I woke up to find my Grafana dashboard completely empty with "No data" displayed across all panels. This was concerning as it meant I was losing valuable attack data.

### Initial Diagnosis

My first step was to check if the containers were still running:

```bash
root@Conpot-LON1:~# docker ps
CONTAINER ID   IMAGE                    CREATED        STATUS        NAMES
9b9cf3cae5d3   conpot-hvac:latest       13 hours ago   Up 13 hours   conpot-hvac
e0a3c6d3dd78   grafana/grafana:latest   13 hours ago   Up 13 hours   monitoring-grafana-1
5daba8c41c5f   grafana/loki:2.9.0       13 hours ago   Up 13 hours   monitoring-loki-1
0a52996eaec7   grafana/promtail:2.9.0   13 hours ago   Up 13 hours   monitoring-promtail-1
```

All containers were running, so the issue wasn't a crashed service.

### Checking Container Logs

I examined the Conpot logs to verify the honeypot was still receiving traffic:

```bash
root@Conpot-LON1:~# docker logs --tail 50 conpot-hvac
2026-01-27 23:04:35,381 New http session from 85.11.183.6
2026-01-27 23:04:35,382 HTTP/1.1 GET request from ('85.11.183.6', 44048)
2026-01-27 23:12:59,076 New http session from 95.214.54.147
2026-01-27 23:32:54,489 New http session from 95.214.54.147
2026-01-27 23:36:01,400 New http session from 204.76.203.219
```

The honeypot was clearly still active and logging to stdout. However, Promtail monitors a log *file*, not Docker's stdout.

### Discovering the Volume Mount Issue

I compared the log timestamps in different locations:

```bash
# Logs inside the container (current)
root@Conpot-LON1:~# docker exec conpot-hvac tail -5 /var/log/conpot/conpot.log
2026-01-28 00:15:56,723 New http session from 124.198.132.2
2026-01-28 00:15:56,724 HTTP/1.1 response to ('124.198.132.2', 59371): 404
2026-01-28 00:15:56,926 HTTP/1.1 GET request from ('124.198.132.2', 59371)
2026-01-28 00:15:56,927 HTTP/1.1 response to ('124.198.132.2', 59371): 404
2026-01-28 00:16:28,943 Session timed out

# Logs on the host filesystem (stale)
root@Conpot-LON1:~# tail -5 /root/conpot-deployment/logs/conpot.log
2026-01-27 20:07:14,397 New http session from 204.76.203.219
2026-01-27 20:07:14,397 HTTP/1.1 GET request from ('204.76.203.219', 40548)
2026-01-27 20:07:14,398 HTTP/1.1 response to ('204.76.203.219', 40548): 302
2026-01-27 20:07:44,411 Session timed out
```

**The problem was clear**: The container's log file had entries up to 00:16, but the host's mounted volume only had logs up to 20:07 - nearly 4 hours earlier. The Docker volume mount had become desynchronized.

### Root Cause Analysis

This type of issue typically occurs when:

1. **File rotation**: If something rotated the log file, Conpot might still be writing to the old file descriptor while Promtail watches a new inode
2. **Container restart**: If the container had been restarted without being properly recreated, the volume mount might not have been re-established correctly
3. **Logging library issues**: Some Python logging configurations keep file handles open and don't properly sync to disk

In this case, the container had been running for 13 hours straight without restart, suggesting the volume mount had become stale during runtime.

## The Solution Journey

### Attempt 1: Docker Logs via Promtail (Failed)

My initial approach was to eliminate the file-based logging entirely and have Promtail scrape Docker's stdout logs directly:

```yaml
scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/conpot-hvac'
        action: keep
```

However, this immediately hit a version compatibility issue:

```
level=error ts=2026-01-28T09:52:15.403985449Z caller=refresh.go:80 
msg="Unable to refresh target groups" 
err="error while listing containers: Error response from daemon: 
client version 1.42 is too old. Minimum supported API version is 1.44"
```

Promtail 2.9.0 uses Docker API 1.42, but my Docker daemon required 1.44+.

### Attempt 2: Upgrading to Promtail 3.x (Failed)

I attempted to upgrade Promtail to 3.0.0 to support the newer Docker API, which introduced new compatibility issues with Loki 2.9.0:

```
level=error caller=client.go:430 msg="final error sending batch" status=400
error="server returned HTTP status 400 Bad Request (400): 
error at least one label pair is required per stream"
```

The label format changed between versions, requiring complex relabel configurations.

### Attempt 3: Upgrading Loki to 3.x (Failed)

I then tried upgrading Loki to match Promtail 3.0.0, but the Loki 3.x configuration format had breaking changes:

```
failed parsing config: /etc/loki/local-config.yaml: yaml: unmarshal errors:
  line 32: field shared_store not found in type compactor.Config
```

The `shared_store` configuration parameter was removed in Loki 3.x and replaced with a different structure. While this was fixable, I was now several layers deep in version incompatibilities.

### The Final Working Solution

Rather than continuing down the upgrade path, I reverted to the original file-based approach with two critical fixes:

**Fix 1: Restart Conpot to Re-establish Volume Mount**

```bash
cd /root/conpot-deployment
docker stop conpot-hvac
docker rm conpot-hvac
docker-compose up -d conpot-hvac
```

This created a fresh container with proper volume mounts. Immediately, logs started flowing to the host filesystem:

```bash
root@Conpot-LON1:~/conpot-deployment# tail -f logs/conpot.log
2026-01-28 10:04:15,486 Modbus server started on: ('0.0.0.0', 5020)
2026-01-28 10:04:15,487 S7Comm server started on: ('0.0.0.0', 102)
2026-01-28 10:04:15,488 HTTP server started on: ('0.0.0.0', 8800)
2026-01-28 10:04:15,786 SNMP server started on: ('0.0.0.0', 16100)
```

**Fix 2: Automated Watchdog Script**

To prevent this issue from recurring, I created a watchdog script that automatically restarts Conpot if logs become stale:

```bash
#!/bin/bash
# /root/conpot-watchdog.sh

LOG_FILE="/root/conpot-deployment/logs/conpot.log"

if [ -f "$LOG_FILE" ]; then
    LAST_MOD=$(stat -c %Y "$LOG_FILE")
    NOW=$(date +%s)
    AGE=$((NOW - LAST_MOD))
    
    # If log file hasn't been modified in 2 hours, restart Conpot
    if [ $AGE -gt 7200 ]; then
        echo "$(date): Conpot log stale (age: ${AGE}s), restarting container"
        cd /root/conpot-deployment && docker-compose restart conpot-hvac
    fi
fi
```

I made the script executable and added it to cron to run every hour:

```bash
chmod +x /root/conpot-watchdog.sh
(crontab -l 2>/dev/null; echo "0 * * * * /root/conpot-watchdog.sh >> /var/log/conpot-watchdog.log 2>&1") | crontab -
```

### Verification

After implementing these fixes, I verified the solution:

```bash
# Check Promtail discovered the log file
root@Conpot-LON1:/opt/monitoring# docker logs monitoring-promtail-1 --tail 50 | grep -E "added target|watching|Seeked"
level=info caller=filetargetmanager.go:361 msg="Adding target" key="/var/log/conpot/*.log:{job=\"conpot\"}"
level=info caller=filetarget.go:313 msg="watching new directory" directory=/var/log/conpot
level=info msg="Seeked /var/log/conpot/conpot.log - &{Offset:0 Whence:0}"
level=info caller=tailer.go:145 component=tailer msg="tail routine: started" path=/var/log/conpot/conpot.log

# Check for errors
root@Conpot-LON1:/opt/monitoring# docker logs monitoring-promtail-1 --tail 20 | grep -i error
# (No errors!)

# Verify Promtail is reading data
root@Conpot-LON1:/opt/monitoring# docker exec monitoring-promtail-1 cat /tmp/positions.yaml
positions:
  /var/log/conpot/conpot.log: "231167"

# Confirm watchdog is scheduled
root@Conpot-LON1:/opt/monitoring# crontab -l | grep conpot-watchdog
0 * * * * /root/conpot-watchdog.sh >> /var/log/conpot-watchdog.log 2>&1
```

The Grafana dashboard immediately populated with data again, showing:
- Real-time protocol activity logs
- Attack distribution across protocols (HTTP, Modbus, S7comm, etc.)
- 17,030+ requests captured
- Active scanning from various IP addresses

## Key Lessons Learned

### 1. Version Compatibility Matters

Mixing different major versions of the Grafana Loki stack (e.g., Promtail 3.x with Loki 2.x) leads to incompatibilities. When possible, keep all components on the same major version.

### 2. Docker Volume Mounts Can Become Stale

Long-running containers with volume mounts may experience synchronization issues. This is particularly problematic for logging applications where data consistency is critical.

### 3. Simple Solutions Are Often Better

While Docker log drivers and service discovery are elegant solutions, file-based logging with a monitoring script proved more reliable for this use case. The added complexity of Docker API compatibility wasn't worth the benefits.

### 4. Always Have Automated Recovery

The watchdog script transformed this from a recurring manual issue into a self-healing system. Even if the root cause recurs, the system will automatically recover within an hour.

### 5. Troubleshooting Methodology

The systematic approach that worked:
1. Verify containers are running
2. Check application logs for actual activity
3. Compare data sources (container vs. host filesystem)
4. Identify the disconnection point
5. Test the simplest fix first
6. Implement automated prevention

## Monitoring Results

With the monitoring system restored and protected by the watchdog, I'm now successfully collecting valuable threat intelligence:

- **Protocol Distribution**: HTTP attacks dominate (65%), followed by Modbus (18%) and SNMP (12%)
- **Attack Patterns**: Regular reconnaissance from security scanning services (Censys, Shodan, Palo Alto Cortex)
- **Malicious Activity**: `.env` file probing, WordPress vulnerability scans, and SQLi attempts against the HVAC web interface
- **Geographic Distribution**: Attacks originating from diverse locations globally

This data feeds directly into my MSc research on ICS security threat landscapes and attack methodologies against building automation systems.

## Conclusion

Setting up Grafana, Loki, and Promtail for honeypot monitoring provides powerful real-time visibility into attack patterns. However, production issues like volume mount desynchronization can disrupt data collection.

The combination of proper container lifecycle management and automated monitoring scripts creates a resilient logging infrastructure that can self-heal from common failure modes. This reliability is crucial for security research where losing even a few hours of attack data could mean missing significant threat intelligence.

For anyone deploying similar monitoring stacks, I recommend:
- Keeping Loki, Promtail, and Grafana on matching major versions
- Implementing health checks for critical data pipelines
- Using simple, proven solutions over complex architectures
- Always having automated recovery mechanisms

The complete configuration files and watchdog script are available in my research repository for anyone interested in replicating this setup.

## Technical Specifications

**Environment:**
- DigitalOcean Droplet (London region)
- Ubuntu 24.04 LTS
- Docker 24.0+
- Conpot 0.6.0
- Grafana Loki 2.9.0
- Grafana Promtail 2.9.0
- Grafana 10.x

**Resource Usage:**
- Loki: ~150MB RAM
- Promtail: ~30MB RAM
- Grafana: ~200MB RAM
- Conpot: ~100MB RAM

**Data Retention:**
- 30 days of logs (720 hours configured)
- Approximately 2-5MB of compressed logs per day
- BoltDB shipper for index management
