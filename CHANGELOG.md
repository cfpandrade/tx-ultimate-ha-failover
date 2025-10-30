# Changelog

All notable changes to the TX Ultimate HA Failover project.

## [1.1.0] - 2024-10-30

### 🐛 Critical Bug Fix

#### Boot Loop Prevention
- **Fixed infinite boot loop issue** caused by watchdog checking WiFi before connection established
- Added `system_ready` global flag to delay watchdog activation
- Watchdogs now activate **2 minutes after boot** to allow proper WiFi connection
- Added protection checks in both WiFi and HA watchdogs

**What was happening:**
- WiFi watchdog checked connection immediately after boot
- Device restarted before WiFi could connect
- Created infinite restart loop

**Solution:**
```yaml
globals:
  - id: system_ready
    type: bool
    initial_value: "false"

interval:
  # System becomes ready after 2 minutes
  - interval: 2min
    then:
      - lambda: |-
          id(system_ready) = true;
          ESP_LOGI("system", "System ready. Watchdogs now active.");

  # Watchdogs only run if system is ready
  - interval: ${wifi_check_interval}
    then:
      - lambda: |-
          if (!id(system_ready)) return;  // Skip if not ready yet
          // ... rest of watchdog logic
```

### 📝 Changes

- Added `system_ready` flag to globals
- Modified WiFi watchdog to check `system_ready` before acting
- Modified HA watchdog to check `system_ready` before acting
- Added "System ready" log message at 2-minute mark
- Updated both `esphome_tx_sonoff.yaml` and `tx_ultimate_failover.yaml`

### 🔍 Technical Details

**Boot Timeline:**
```
0s    → Device boots
0-120s → Watchdogs INACTIVE (allows WiFi connection)
120s   → system_ready = true, "System ready" log message
120s+  → Watchdogs ACTIVE (normal monitoring)
```

### ⚡ Impact

- ✅ **Eliminates boot loops** completely
- ✅ **Safe first boot** - Device has time to connect
- ✅ **No configuration changes** needed - works automatically
- ✅ **Better logging** - Clear "System ready" indicator

### 🔄 Migration

**No action required!** This is a bug fix that applies automatically.

Just reflash your device with:
```bash
esphome run tx_ultimate_failover.yaml
```

---

## [1.0.0] - 2024-10-30 - Initial Release

### 🚀 Major Features

#### Ping-Based Monitoring
- **Replaced TCP connection checks with ICMP ping** for Home Assistant monitoring
- More reliable detection of HA availability
- Added packet loss percentage tracking
- Added network latency monitoring in milliseconds
- Better diagnostics in logs with detailed metrics

**Benefits:**
- Lower-level protocol (more reliable)
- Firewall-friendly (ping usually allowed)
- Can detect network congestion vs complete outages
- Lighter weight than TCP connections
- More accurate failure detection

#### Bluetooth Proxy
- **Added built-in Bluetooth Low Energy proxy**
- Extends Home Assistant's Bluetooth range by ~10-30 meters per device
- Zero configuration required - auto-discovered by HA
- Active scanning for better device detection
- Works with all HA-supported BLE devices:
  - Temperature/humidity sensors
  - Bluetooth trackers
  - Fitness bands
  - Smart locks
  - And more

**Configuration:**
```yaml
esp32_ble_tracker:
  scan_parameters:
    interval: 1100ms
    window: 1100ms
    active: true

bluetooth_proxy:
  active: true
```

### 🔧 Technical Changes

#### Dependencies
- Added `ESP32Ping` library to platformio_options
- Added `esphome-component-ping` external component from GitHub

#### Sensors
- New: `ha_packet_loss` - Tracks packet loss percentage to HA
- New: `ha_latency` - Tracks ping latency to HA in milliseconds
- Both sensors are internal (not exposed to HA interface)

#### Watchdog Improvements
- Watchdog now uses ping sensor data instead of WiFiClient
- Improved logging with packet loss and latency metrics
- Threshold changed from connection failure to >80% packet loss
- More granular detection of network issues

#### Code Quality
- Replaced `WiFiClient.connect()` with ping sensor checks
- Cleaner failure detection logic
- Better error messages with diagnostic data
- Removed dependency on TCP port configuration for monitoring

### 📚 Documentation

- Updated README.md with ping monitoring details
- Updated ARCHITECTURE.md with ping logic explanation
- Added Bluetooth Proxy section to ARCHITECTURE.md
- Updated PROJECT_SUMMARY.md with new features
- Added system diagram showing BLE tracker integration

### 🎯 Configuration

No breaking changes - all existing configurations remain compatible!

New optional features work out of the box with default settings.

### Additional Features

- WiFi watchdog with auto-restart
- Home Assistant failover detection
- Local relay control when HA unreachable
- Visual LED status indicators
- Fully configurable via substitutions
- Automated installation script
- Comprehensive documentation

### Components

- Touch panel integration
- 1-3 relay support
- NeoPixel LED control
- Nightlight with sunset/sunrise automation
- Media player support
- Complete automation examples

---

## Migration Guide

### From 1.0.0 to 2.0.0

**No migration needed!** Version 2.0.0 is fully backward compatible.

The new ping monitoring and Bluetooth proxy features are automatically enabled and require no configuration changes.

**What happens automatically:**
1. Ping sensor starts monitoring HA on next reboot
2. Bluetooth proxy activates and becomes discoverable in HA
3. Watchdog logic seamlessly switches to ping-based detection
4. All existing configurations continue to work

**Optional: View new sensors**

If you want to see packet loss and latency in HA, change `internal: true` to `internal: false`:

```yaml
sensor:
  - platform: ping
    ip_address: ${ha_ip}
    # ... other settings ...
    loss:
      name: "${friendly_name} HA Packet Loss"
      id: ha_packet_loss
      internal: false  # Changed from true
    latency:
      name: "${friendly_name} HA Latency"
      id: ha_latency
      internal: false  # Changed from true
```

---

## Roadmap

### Planned Features

- [ ] MQTT support as alternative to ESPHome API
- [ ] Built-in web configuration interface
- [ ] Temperature/humidity sensor integration
- [ ] Voice control via I2S audio
- [ ] Multi-device scene coordination
- [ ] Energy monitoring integration
- [ ] BLE beacon broadcasting for presence detection
- [ ] Custom watchdog thresholds via HA service calls

### Under Consideration

- [ ] Multiple HA instances failover
- [ ] Mesh network support for multi-device coordination
- [ ] Cloud backup endpoint
- [ ] Advanced LED patterns library
- [ ] Touch gesture customization via HA UI

---

## Version History

| Version | Date | Highlights |
|---------|------|------------|
| 1.1.0 | 2024-10-30 | **Critical fix: Boot loop prevention** |
| 1.0.0 | 2024-10-30 | Initial release with ping monitoring and Bluetooth Proxy |

---

## Contributing

See [README.md](README.md) for contribution guidelines.

## License

MIT License - See LICENSE file for details
