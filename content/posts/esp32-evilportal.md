---
title: "Building Custom Evil Portal ESP32 Firmware"
date: 2026-02-09
author: Nicholas
tags: ["security", "esp32", "evil-portal"]
description: "Social engineering/pentest tool"
---

Last night I developed some custom firmware for the ESP32 WROOM-32 that demonstrates captive portal attacks. This serves as both an educational tool and a practical demonstration of why users should be cautious when connecting to public WiFi networks.

**Disclaimer:** This tool is designed exclusively for authorized security testing and educational purposes. All testing was conducted on equipment I own in controlled environments.

## What Is a Captive Portal Attack?

Captive portals are the login pages you see when connecting to public WiFi at coffee shops, hotels, or airports. An evil portal attack exploits this familiar user experience by:

1. Creating a fake WiFi access point
2. Redirecting all DNS requests to a local web server
3. Presenting a convincing login page
4. Capturing credentials when users "authenticate"

While this technique has been around for years, I wanted to create a clean, modern implementation that could serve as an educational reference for security researchers.

## Why ESP32?

The ESP32 WROOM-32 is an ideal platform for this type of research:

- **Built-in WiFi**: Dual-mode WiFi with AP (Access Point) capability
- **Affordable**: ~£5-10 per board
- **Low power**: Can run on battery for portable demonstrations
- **Arduino compatible**: Easy to program and modify
- **Compact**: Small enough for discrete deployment in controlled tests

## Technical Architecture

### Core Components

The firmware implements three key systems:

**1. WiFi Access Point**
```cpp
WiFi.mode(WIFI_AP);
WiFi.softAPConfig(apIP, apIP, IPAddress(255, 255, 255, 0));
WiFi.softAP("Roasted Bean Cafe WiFi", "");
```

Creates an open WiFi network that devices can connect to automatically.

**2. DNS Server**
```cpp
dnsServer.start(DNS_PORT, "*", apIP);
```

Redirects all DNS queries to the ESP32's IP address (192.168.4.1), forcing the captive portal to appear regardless of what website the user tries to visit.

**3. Web Server**
```cpp
webServer.on("/", handleRoot);
webServer.on("/login", HTTP_POST, handleLogin);
webServer.on("/admin", handleAdmin);
```

Serves the captive portal interface and processes form submissions.

### Design Decisions

**Why a Coffee Shop Theme?**

I deliberately chose a generic cafe theme rather than impersonating a real brand like Google or Microsoft. This decision was both ethical and practical:

- Avoids trademark infringement
- Reduces potential for misuse
- Still demonstrates the attack vector effectively
- Clearly marks this as a research/educational tool

**Admin Panel for Monitoring**

A built-in admin panel accessible at `/admin` provides real-time monitoring:
- Live credential capture feed
- Client IP addresses
- Timestamps
- Auto-refresh every 10 seconds

Authentication is required (default: admin/admin81, but easily changed in source).

## Development Challenges

### Memory Constraints

The ESP32 has limited RAM, so I implemented:
- Circular buffer for credential storage (max 50 entries)
- PROGMEM storage for HTML/CSS to keep strings in flash
- Efficient string handling to avoid fragmentation

### HTML Minification

Initially, I used a beautifully formatted HTML template. But with limited flash space, I needed to minify:

```cpp
// Before: ~15KB readable HTML
// After: ~8KB minified (all whitespace removed)
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html><!doctypehtml><meta charset=utf-8>...
```

This was a trade-off between code readability and storage efficiency.

### Cross-Device Compatibility

Different devices handle captive portals differently:
- **Android**: Usually auto-opens portal
- **iOS**: Shows "Sign in to WiFi" notification
- **Windows**: May require manual browser navigation
- **Linux**: Varies by distribution/network manager

I tested across multiple devices to ensure broad compatibility. The captive portal works best on mobile devices, as it automatically opens. On other devices, you have to try and search something first.

## Building and Flashing

### PlatformIO Configuration

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
upload_speed = 460800
```

### Binary Distribution

For easier sharing, I created a complete firmware package including:
- Pre-compiled binaries (bootloader, partitions, firmware)
- Flash script with auto-detection
- Comprehensive documentation
- Legal disclaimers

Users can flash with a single command:
```bash
./tools/flash.sh
```

Or manually with esptool:
```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 \
  write_flash 0x1000 bootloader.bin 0x8000 partitions.bin \
  0xe000 boot_app0.bin 0x10000 firmware.bin
```

## Security Researcher Use Cases

This tool is valuable for:

**1. Penetration Testing**
- Testing organizational WiFi security policies
- Evaluating employee security awareness
- Demonstrating social engineering risks

**2. Security Training**
- Teaching WiFi attack vectors
- Demonstrating why VPNs matter on public networks
- Training incident response teams

**3. Red Team Operations**
- Physical security assessments
- Social engineering exercises
- Credential harvesting simulations (with authorization)

**4. Research and Development**
- Studying captive portal implementations
- Testing detection mechanisms
- Developing countermeasures

## Detection and Countermeasures

### How to Detect Evil Portals

Organizations can identify these attacks through:

1. **WiFi Monitoring**: Tools like Kismet or Wireshark can detect unusual SSIDs
2. **Certificate Analysis**: No HTTPS means no valid certificates
3. **Network Behavior**: Abnormal DNS response patterns
4. **SSID Verification**: Confirm network names with staff
5. **MAC Address Tracking**: Identify rogue access points

### User Protection

End users should:
- Verify network authenticity with staff
- Use VPNs on public WiFi
- Look for HTTPS on sensitive sites
- Be suspicious of unexpected login requests
- Use password managers (they won't auto-fill on fake sites)

## Lessons Learned

### Technical Insights

1. **ESP32 is powerful**: More than capable for complex WiFi attacks
2. **Memory management matters**: Every byte counts on embedded systems
3. **Cross-platform testing is essential**: Different OS/device behaviors vary significantly
4. **Documentation is crucial**: Good docs make tools more useful and safer

### Ethical Considerations

Building security tools requires responsibility:
- Always include prominent legal disclaimers
- Design with education in mind, not malicious use
- Make detection information available
- Don't impersonate real brands
- Share responsibly with proper context

## Project Statistics

- **Development Time**: ~8 hours (including testing and documentation)
- **Lines of Code**: ~450 (Arduino C++)
- **Firmware Size**: ~280 KB
- **RAM Usage**: ~45 KB
- **Supported Clients**: Up to 4 simultaneous connections
- **Credential Storage**: 50 entries max

## Future Improvements

Potential enhancements I'm considering:

1. **HTTPS Support**: Self-signed certificates to appear more legitimate
2. **Multi-Page Flow**: Simulate multi-step authentication
3. **Template System**: Easy customization for different scenarios
4. **SD Card Logging**: Extended storage for long-term deployments
5. **Bluetooth Coexistence**: Beacon spoofing alongside WiFi
6. **OLED Display**: Show stats without needing admin panel
7. **Battery Optimization**: Extended runtime for portable use

---

This project demonstrates that sophisticated WiFi attacks can be implemented on inexpensive, readily-available hardware. While the technical barrier to entry is low, the ethical and legal barriers must remain high.

The complete firmware package, including source code, binaries, and documentation, will soon be available for authorized security researchers and educators.

## Resources

- **ESP32 Documentation**: [Espressif Systems](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/)
- **PlatformIO**: [platformio.org](https://platformio.org)
- **OWASP Testing Guide**: [owasp.org](https://owasp.org/www-project-web-security-testing-guide/)

## Legal Notice

This project is for educational and authorized security testing only. Unauthorized interception of credentials is illegal under the Computer Fraud and Abuse Act (US), Computer Misuse Act (UK), GDPR (EU), and similar laws worldwide. Always obtain explicit written permission before conducting security testing.

---

*Questions or comments? Feel free to reach out. For security vulnerabilities in the firmware itself, please practice responsible disclosure.*

---

*Last Updated: 09/02/2026*
*Firmware Version: 1.0.0*