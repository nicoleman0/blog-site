---
title: "Customizing Conpot for Realistic ICS Emulation"
date: 2026-01-26
description: "Building a believable HVAC controller honeypot"
tags: ["ICS", "Industrial", "Honeypot", "Security-Research", "SCADA"]
---

## Table of Contents

* [Default Templates](#the-problem-with-default-templates)
* [Research-Driven Customization](#research-driven-customization)
* [Template Architecture Deep Dive](#template-architecture-deep-dive)
  + [Understanding Conpot's Template System](#understanding-conpots-template-system)
  + [Key Configuration Files](#key-configuration-files)
* [Building a Realistic S7-1200 HVAC Controller](#building-a-realistic-s7-1200-hvac-controller)
  + [Device Identity Customization](#device-identity-customization)
  + [S7comm Protocol Responses](#s7comm-protocol-responses)
  + [Modbus Register Configuration](#modbus-register-configuration)
* [Docker Build Process](#docker-build-process)
  + [Template Path Discovery](#template-path-discovery)
  + [Custom Image Creation](#custom-image-creation)
  + [Deployment Challenges](#deployment-challenges)
* [Verification and Testing](#verification-and-testing)
* [Why These Details Matter](#why-these-details-matter)
* [Observations and Next Steps](#observations-and-next-steps)

Following my [initial Conpot deployment](https://nicoleman0.github.io/blog-site/posts/conpot-deployment-blogpost/), I faced a critical question: would sophisticated attackers recognize the default "Technodrome" template as an obvious honeypot? The generic nature of Conpot's out-of-the-box configuration - with placeholder values like "Mouser Factory" and "Venus" as the system location - seemed unlikely to fool anyone conducting targeted ICS reconnaissance.

This post documents the process of transforming a generic honeypot into a convincing simulation of a real-world industrial system: specifically, a Siemens S7-1200 PLC controlling HVAC equipment in a building automation deployment.

## The Problem with Default Templates

Conpot's default template emulates an S7-200 PLC with whimsical configuration values clearly designed for demonstration purposes rather than realism. Examining the default `template.xml` reveals:

```xml
<entity name="unit">S7-200</entity>
<key name="FacilityName">
    <value type="value">"Mouser Factory"</value>
</key>
<key name="SystemName">
    <value type="value">"Technodrome"</value>
</key>
<key name="sysLocation">
    <value type="value">"Venus"</value>
</key>
```

While functional for basic honeypot deployment, these values present several issues for research purposes:

1. **Obvious honeypot indicators**: "Technodrome" and "Venus" are clearly not legitimate facility names
2. **Outdated hardware**: S7-200 PLCs reached end-of-life in 2009, though they remain in use
3. **Generic configuration**: No context suggesting actual industrial application
4. **Random register values**: Modbus/S7comm registers contain randomized data rather than plausible sensor readings

For my research into attacker behavior against building automation systems, I needed something more convincing: a deployment that would pass initial scrutiny from experienced ICS security researchers or sophisticated threat actors.

## Research-Driven Customization

My OSINT research using Shodan revealed patterns in real exposed Siemens deployments:

- **S7-1200 series dominance**: The 1200 series represents current-generation PLCs commonly found in building automation
- **Realistic naming conventions**: System names often reference function (HVAC, chiller, boiler) and location
- **Facility context**: Building automation controllers typically identify mechanical spaces (basement, roof, equipment rooms)
- **Proper model numbers**: Real systems expose legitimate Siemens order codes and firmware versions

These observations informed my customization strategy: create a honeypot that mirrors actual building automation deployments while maintaining the logging and monitoring capabilities essential for research.

## Template Architecture Deep Dive

### Understanding Conpot's Template System

Conpot organizes templates as directory structures containing protocol-specific configuration files. Each template directory includes:

```
template_name/
├── template.xml          # Core system identity and databus keys
├── s7comm/
│   └── s7comm.xml       # S7 protocol responses
├── modbus/
│   └── modbus.xml       # Modbus register mappings
├── http/
│   └── index.html       # Web interface
├── snmp/
│   └── snmp.xml         # SNMP OID tree
└── [other protocols]
```

The `template.xml` file defines a "databus" - a key-value store that other protocol handlers reference. This architecture allows changing device identity in one place while having those changes propagate across all protocol implementations.

### Key Configuration Files

**template.xml**: Defines system-wide identity
**s7comm/s7comm.xml**: Controls responses to Siemens S7 protocol queries  
**modbus/modbus.xml**: Maps Modbus registers to databus values

Understanding this separation is crucial: you modify *device identity* in template.xml, but *protocol behavior* in protocol-specific files.

## Building a Realistic S7-1200 HVAC Controller

### Device Identity Customization

Based on Shodan reconnaissance of real systems, I configured the template to present as a Siemens S7-1200 CPU 1214C in a building automation role:

```xml
<template>
    <entity name="unit">S7-1200</entity>
    <entity name="vendor">Siemens</entity>
    <entity name="description">Siemens S7-1200 PLC - Building Automation System</entity>
    <entity name="protocols">HTTP, MODBUS, s7comm, SNMP</entity>
    <entity name="creator">Building Management System</entity>
</template>
```

Critical databus values were updated to reflect a realistic deployment:

```xml
<key name="FacilityName">
    <value type="value">"Central Heating Plant"</value>
</key>
<key name="SystemName">
    <value type="value">"HVAC-Controller-01"</value>
</key>
<key name="SystemDescription">
    <value type="value">"Siemens, SIMATIC, S7-1200, CPU 1214C DC/DC/DC"</value>
</key>
<key name="sysContact">
    <value type="value">"sys-help@blue-corvus.local"</value>
</key>
<key name="sysLocation">
    <value type="value">"Basement - Mechanical Room B2"</value>
</key>
```

The order code was set to a legitimate S7-1200 part number:

```xml
<key name="s7_id">
    <value type="value">"6ES7214-1AG40-0XB0"</value>
</key>
<key name="s7_module_type">
    <value type="value">"CPU 1214C DC/DC/DC"</value>
</key>
```

These values were chosen based on actual Siemens naming conventions and part numbering schemes, ensuring they would validate correctly against Siemens documentation if an attacker conducted verification.

### S7comm Protocol Responses

The S7comm protocol implementation required updates to match the S7-1200 hardware profile. The default template's `s7comm.xml` had incomplete module identification fields that I populated with realistic values:

```xml
<s7comm enabled="True" host="0.0.0.0" port="102">
    <system_status_lists>
        <ssl id="W#16#xy1C" name="Component Identification">
            <system_name id="W#16#0001">SystemName</system_name>
            <module_name id="W#16#0002">SystemDescription</module_name>
            <plant_ident id="W#16#0003">FacilityName</plant_ident>
            <copyright id="W#16#0004">Copyright</copyright>
            <serial id="W#16#0005">s7_id</serial>
            <module_type_name id="W#16#0007">s7_module_type</module_type_name>
            <location id="W#16#000B">sysLocation</location>
        </ssl>
        <ssl id="W#16#xy11" name="Module Identification">
            <module_identification id="W#16#0001">6ES7214-1AG40-0XB0</module_identification>
            <hardware_identification id="W#16#0006">V4.2</hardware_identification>
            <firmware_identification id="W#16#0007">V4.2.3</firmware_identification>
        </ssl>
    </system_status_lists>
</s7comm>
```

Note the firmware version V4.2.3 - this corresponds to a legitimate S7-1200 firmware release, making the system appear authentic to fingerprinting tools like `snap7` or `s7scan`.

The port was also corrected from 10201 to the standard S7comm port 102, ensuring the honeypot matches expected protocol behavior.

### Modbus Register Configuration

The default Modbus configuration used random binary values for registers, which would appear suspicious during reconnaissance. Real HVAC systems expose sensor readings, setpoints, and equipment status in specific value ranges.

I reconfigured the analog input registers to simulate realistic HVAC telemetry:

```xml
<key name="memoryModbusSlave2BlockC">
    <!-- ANALOG_INPUTS: HVAC sensor readings -->
    <value type="value">[220, 185, 650, 420, 850, 1050, 730, 0]</value>
</key>
<key name="memoryModbusSlave2BlockD">
    <!-- HOLDING_REGISTERS: HVAC setpoints -->
    <value type="value">[210, 200, 500, 400, 800, 1000, 750, 0]</value>
</key>
```

These values, when interpreted with appropriate scaling factors, represent:

- **220**: 22.0°C supply air temperature
- **185**: 18.5°C return air temperature
- **650**: 6.5 bar chilled water pressure
- **420**: 42% damper position
- **850**: 85% pump speed
- **1050**: 10.5 m³/min flow rate
- **730**: 73% humidity

Values remain static (unlike a real system with fluctuating sensors) but fall within plausible operational ranges for commercial HVAC equipment.

The Modbus device info was also updated:

```xml
<device_info>
    <VendorName>Siemens</VendorName>
    <ProductCode>SIMATIC S7-1200</ProductCode>
    <MajorMinorRevision>V4.2.3</MajorMinorRevision>
</device_info>
```

## Docker Build Process

### Template Path Discovery

Implementing these customizations required building a custom Docker image. The first challenge: determining where Conpot actually stores its templates inside the container.

The base image path wasn't documented clearly, requiring inspection:

```bash
docker run --rm honeynet/conpot:latest find / -name "template.xml" 2>/dev/null
```

This revealed templates reside at:
```
/home/conpot/.local/lib/python3.6/site-packages/conpot-0.6.0-py3.6.egg/conpot/templates/
```

A significantly different path than the `/usr/src/conpot/conpot/templates/` location suggested in some documentation.

### Custom Image Creation

With the correct path identified, I created a Dockerfile to overlay my customized template:

```dockerfile
FROM honeynet/conpot:latest

# Copy custom template to correct location
COPY templates/custom/default /home/conpot/.local/lib/python3.6/site-packages/conpot-0.6.0-py3.6.egg/conpot/templates/custom

# Expose standard ICS ports
EXPOSE 80 102 502 161/udp
```

The directory structure required careful attention - Conpot expects template files directly under the template name directory, not nested in a subdirectory.

Building the image:

```bash
docker build -t conpot-hvac:latest .
```

### Deployment Challenges

Initial deployment attempts failed with "Template not found: custom" errors. Troubleshooting revealed several issues:

**Problem 1: Incorrect ENTRYPOINT syntax**

The base image uses a specific entrypoint that couldn't be overridden with simplified commands. Inspection revealed:

```bash
docker inspect honeynet/conpot:latest --format='{{.Config.Cmd}}'
# Output: [/home/conpot/.local/bin/conpot --template default ...]
```

**Solution**: Use complete command path in docker-compose:

```yaml
command: ["/home/conpot/.local/bin/conpot", "--template", "custom", 
          "--logfile", "/var/log/conpot/conpot.log", "-f", "--temp_dir", "/tmp"]
```

**Problem 2: Port mapping discrepancies**

Conpot runs services on non-standard internal ports that must be mapped to standard external ports:

- HTTP runs internally on 8800 → external 80
- Modbus runs internally on 5020 → external 502
- S7comm runs internally on 102 → external 102 (correctly configured)
- SNMP runs internally on 16100 → external 161

Docker-compose configuration:

```yaml
ports:
  - "80:8800"         # HTTP
  - "102:102"         # S7comm
  - "502:5020"        # Modbus
  - "161:16100/udp"   # SNMP
```

## Verification and Testing

With the customized honeypot deployed, verification confirmed proper operation:

**HTTP Interface**:
```bash
curl http://localhost:80
# Returns: HVAC-Controller-01 status page
```

**Log Output**:
```
2026-01-26 22:10:47,057 Starting Conpot using template: .../templates/custom
2026-01-26 22:10:47,122 Conpot modbus initialized
2026-01-26 22:10:47,126 Conpot S7Comm initialized
2026-01-26 22:10:47,174 Modbus server started on: ('0.0.0.0', 5020)
2026-01-26 22:10:47,175 S7Comm server started on: ('0.0.0.0', 102)
2026-01-26 22:10:47,176 HTTP server started on: ('0.0.0.0', 8800)
```

All protocols initialized successfully with the custom configuration loaded.

**S7comm Fingerprinting** (when performed by attackers) will now return:
- **Model**: CPU 1214C DC/DC/DC
- **Serial**: 6ES7214-1AG40-0XB0
- **Firmware**: V4.2.3
- **Location**: Basement - Mechanical Room B2
- **System**: HVAC-Controller-01

These values match real Siemens deployments observed via Shodan, significantly improving the honeypot's plausibility.

## Why These Details Matter

The investment in creating a realistic template serves several research objectives:

**1. Attracting sophisticated actors**: Generic honeypots may be ignored by experienced attackers who quickly identify obvious deception. A convincing configuration is more likely to attract serious reconnaissance activity.

**2. Understanding attacker validation**: By providing legitimate-looking model numbers and firmware versions, I can observe whether attackers validate these details against vendor documentation - indicating more sophisticated tradecraft.

**3. Behavioral authenticity**: Realistic Modbus register values may encourage attackers to proceed with exploitation attempts, whereas random data might trigger suspicion.

**4. Comparative analysis**: With both the default "Technodrome" and this customized deployment, I can compare which attracts more attention and what types of interaction occur with each.

**5. Building automation targeting**: This specific configuration allows research into attacks targeting building management systems, an increasingly concerning attack surface as smart buildings proliferate.

## Observations and Next Steps

The customized honeypot has been operational for several hours. Initial observations:

**Immediate indexing**: Shodan has already indexed the new deployment, correctly identifying it as a Siemens S7-1200 rather than generic ICS equipment.

**Log capture**: All interactions are being logged, including the complete HTTP headers from initial reconnaissance:

```
HTTP/1.1 GET request from ('144.126.203.72', 54496): 
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0
```

**Next research phases**:

1. **Comparative analysis**: Monitor interaction patterns between default and customized deployments
2. **Log aggregation**: Implement Grafana + Loki stack for centralized log analysis (guide prepared separately)
3. **Behavioral documentation**: Track how attackers interact with realistic vs. generic configurations
4. **Temporal analysis**: Identify patterns in when attacks occur and from which geographic regions
5. **Protocol exploitation**: Document specific S7comm and Modbus exploit attempts

**Long-term deployment strategy**:

I plan to maintain both the original Technodrome deployment (for baseline comparison) and this customized S7-1200 HVAC controller. Running them on separate infrastructure will allow direct comparison of attack patterns against obviously fake versus convincingly realistic honeypots.

The customization process, while technically involved, fundamentally improves the research value of the deployment. A honeypot that passes initial scrutiny from sophisticated attackers will generate higher-quality data about real-world attack methodologies against industrial control systems.

---

**Technical Resources**:

- Custom template files: [Available on request for research collaboration]
- Conpot documentation: https://conpot.readthedocs.io/
- Siemens S7-1200 documentation: https://support.industry.siemens.com/
- Shodan ICS search: https://www.shodan.io/explore/category/industrial-control-systems

**Configuration files referenced in this post**:
- `template.xml`: Core device identity and databus configuration
- `s7comm/s7comm.xml`: S7 protocol response definitions
- `modbus/modbus.xml`: Modbus register mappings
- `Dockerfile`: Custom image build instructions
- `docker-compose.yml`: Deployment orchestration

*This research continues as part of my MSc in Information Security at Royal Holloway, University of London, investigating threat landscapes facing industrial control systems and building automation infrastructure.*
