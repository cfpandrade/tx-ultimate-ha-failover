# Architecture & Technical Details

## System Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                      TX Ultimate Device                            │
│                                                                    │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     │
│  │   WiFi       │────▶│   ESPHome    │────▶│  Relays      │     │
│  │  Connection  │     │   Firmware   │     │  (1-3)       │     │
│  └──────────────┘     └──────────────┘     └──────────────┘     │
│         │                     │                                   │
│         │                     ▼                                   │
│         │            ┌──────────────┐                             │
│         │            │   Touch      │                             │
│         │            │   Panel      │                             │
│         │            └──────────────┘                             │
│         │                     │                                   │
│         │                     ▼                                   │
│         │            ┌──────────────┐                             │
│         │            │  BLE Tracker │◀─── BLE Devices            │
│         │            │  (Bluetooth) │     (sensors, trackers)    │
│         │            └──────────────┘                             │
│         │                     │                                   │
│         ▼                     ▼                                   │
│  ┌──────────────────────────────────┐                            │
│  │      Watchdog System             │                             │
│  │  - WiFi Monitor (10 min)         │                             │
│  │  - HA Ping Monitor (5 min)       │                             │
│  │  - Packet Loss Tracking          │                             │
│  └──────────────────────────────────┘                            │
│                      │                                             │
└──────────────────────┼─────────────────────────────────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Home Assistant  │
              │  (192.168.x.x)  │
              │  - API (native) │
              │  - BLE Proxy    │
              │  - Ping ICMP    │
              └─────────────────┘
```

## State Machine

The device operates in three states:

### State 0: Normal Operation

```
┌─────────────────────┐
│   NORMAL MODE       │
│                     │
│ • WiFi: Connected   │
│ • HA: Online        │
│ • Touch: Sends HA   │
│ • Relays: HA ctrl   │
└─────────────────────┘
         │
         │ WiFi Lost
         ▼
┌─────────────────────┐
│   STATE 1           │
│   NO WIFI           │
│                     │
│ • Device restarts   │
│ • Yellow LED pulse  │
└─────────────────────┘
```

### State 1: WiFi Lost

```
┌─────────────────────┐
│   WIFI LOST         │
│                     │
│ • Attempts reconnect│
│ • After timeout:    │
│   Device restarts   │
│ • Yellow LED shown  │
│   on touch          │
└─────────────────────┘
         │
         │ WiFi Back
         ▼
┌─────────────────────┐
│   NORMAL MODE       │
└─────────────────────┘
```

### State 2: Home Assistant Unreachable

```
┌─────────────────────┐
│   NORMAL MODE       │
│                     │
│ • WiFi: OK          │
│ • HA: Online        │
└─────────────────────┘
         │
         │ HA unreachable for 10 min
         ▼
┌─────────────────────┐
│   STATE 2           │
│   HA DOWN           │
│                     │
│ • WiFi: OK          │
│ • HA: Offline       │
│ • Touch: Toggles    │
│   relays locally    │
│ • Red LED pulse     │
└─────────────────────┘
         │
         │ HA back online
         ▼
┌─────────────────────┐
│   NORMAL MODE       │
│                     │
│ • Cyan LED pulse    │
│   (confirmation)    │
└─────────────────────┘
```

## Watchdog Logic

### WiFi Watchdog

**Interval**: Configurable (default: 10 minutes)

```python
def wifi_watchdog():
    if not WiFi.isConnected():
        log("WiFi lost. Restarting...")
        ESP.restart()
```

**Purpose**:
- Ensures device doesn't stay disconnected indefinitely
- Fresh restart often fixes WiFi issues
- Prevents memory leaks from repeated connection attempts

### Home Assistant Watchdog (Ping-based)

**Interval**: Configurable (default: 5 minutes)

```python
def ha_watchdog():
    # First, check WiFi
    if not WiFi.isConnected():
        connectivity_status = 1  # State: No WiFi
        enable_local_control()
        return

    # Check ping sensor results (runs every 5 minutes)
    # Sends 5 ICMP packets with 1s timeout each
    packet_loss = ha_packet_loss.state
    latency = ha_latency.state

    # If packet loss > 80% or NaN (no response)
    if isnan(packet_loss) or packet_loss > 80.0:
        ha_down_count++
        log_warning("HA unreachable", ha_down_count, packet_loss)

        # After 2 failed attempts (10 minutes)
        if ha_down_count >= 2:
            connectivity_status = 2  # State: HA down
            enable_local_control()
    else:
        # HA is responding (packet loss < 80%)
        if ha_down_count > 0:
            connectivity_status = 0  # State: Normal
            disable_local_control()
            show_ha_online_indicator()
            log_info("HA back online", packet_loss, latency)

        ha_down_count = 0
```

**Advantages over TCP connection**:
- **More reliable**: ICMP ping is lower-level and less likely to be blocked
- **Better diagnostics**: Provides packet loss % and latency metrics
- **Lighter weight**: No need to establish TCP connections
- **Firewall-friendly**: Ping usually allowed in most network configurations
- **Accurate detection**: Can detect network congestion (high latency) vs complete outage

**Purpose**:
- Detects when Home Assistant is unreachable using ICMP ping
- Requires 2 consecutive failures before switching to local mode (10 minutes total)
- Reduces false positives from temporary network issues
- Provides detailed metrics (loss %, latency) in logs
- Automatically restores normal operation when HA returns

## Touch Panel Integration

### Touch Events Flow

```
Touch Panel (UART)
       │
       ▼
┌──────────────────┐
│  tx_ultimate     │
│  touch component │
└──────────────────┘
       │
       ├──▶ on_press ──────▶ LED indicator
       │                     (shows connectivity status)
       │
       ├──▶ on_release ────▶ Binary sensor
       │                     (sent to HA)
       │                     │
       │                     ▼
       │              If local mode:
       │              Toggle relay directly
       │
       ├──▶ on_swipe_left ─▶ Swipe event
       │
       ├──▶ on_swipe_right ▶ Swipe event
       │
       ├──▶ on_full_touch ─▶ Multi-touch event
       │
       └──▶ on_long_touch ─▶ Long press event
```

### Local Control Logic

```yaml
# Global flags
toggle_relay_1_on_touch: false  # Normal mode
toggle_relay_2_on_touch: false
toggle_relay_3_on_touch: false

# When touch detected
on_press:
  - if connectivity_status == 1:  # No WiFi
      show yellow LED pulse
  - if connectivity_status == 2:  # HA down
      show red LED pulse
  - else:
      show normal touch LED

on_release:
  - publish binary_sensor event
  - if toggle_relay_X_on_touch:
      toggle relay directly
```

## LED Indicators

### LED Partitions

The device has 28 LEDs divided into sections:

```
LED Layout (28 LEDs total):
┌────────────────────────────────┐
│  Top Strip (20-26): 7 LEDs     │ ◄── Effects & Status
├────────────────────────────────┤
│  ┌─────┐  ┌─────┐  ┌─────┐   │
│  │  L  │  │  M  │  │  R  │   │ ◄── Button indicators
│  │6-9  │  │8-10 │  │9-12 │   │
│  └─────┘  └─────┘  └─────┘   │
├────────────────────────────────┤
│  Nightlight (0-5, 8, 10,       │ ◄── Ambient lighting
│              13-19, 27): 18 LEDs│
└────────────────────────────────┘

L = Left button
M = Middle button
R = Right button
```

### Status Colors

| Status | Color | When Shown | Duration |
|--------|-------|------------|----------|
| No WiFi | Yellow pulsing | Touch while disconnected | 3s |
| HA Down | Red pulsing | Touch while HA offline | 3s |
| HA Back | Cyan pulsing | HA returns online | 3s |
| Touch | Cyan (default) | Normal touch | 6s |
| Button On | Blue (default) | Relay active | Persistent |
| Nightlight | Warm (default) | Sunset-sunrise | Persistent |

## Configuration Variables

### Network Configuration

```yaml
substitutions:
  ha_ip: "192.168.50.168"      # Home Assistant IP
  ha_port: "8123"               # Home Assistant port
  device_ip: "192.168.35.56"    # Fixed IP for this device
```

### Timing Configuration

```yaml
substitutions:
  wifi_check_interval: "600s"   # 10 minutes
  ha_check_interval: "300s"     # 5 minutes

  # Failover triggers after:
  # ha_check_interval × 2 = 10 minutes
```

### Hardware Configuration

```yaml
substitutions:
  relay_count: "2"              # 1, 2, or 3 relays
  vibra_time: 400ms             # Haptic feedback duration
  button_on_time: 500ms         # Touch event duration
```

## Memory Usage

Approximate ESP32 resource usage:

- **Flash**: ~1.2 MB
- **RAM**: ~45 KB (heap)
- **CPU**: <5% average
- **Network**: Minimal (only HA health checks)

## Security Considerations

### API Encryption

```yaml
api:
  encryption:
    key: !secret api_key  # 32-character key
```

All API communication with Home Assistant is encrypted using this key.

### Fallback Access Point

```yaml
wifi:
  ap:
    ssid: ${ap_ssid}
    password: !secret ap_pass
```

When WiFi fails, creates a secure AP for recovery/configuration.

### No External Dependencies

- No cloud services required
- No external API calls
- All processing done locally
- HA communication over local network only

## Extensibility

### Adding Custom Sensors

```yaml
sensor:
  - platform: template
    name: "Connectivity Status"
    lambda: |-
      if (id(connectivity_status) == 0) return "Normal";
      if (id(connectivity_status) == 1) return "No WiFi";
      if (id(connectivity_status) == 2) return "HA Down";
      return "Unknown";
```

### Custom Automations

```yaml
on_boot:
  - lambda: |-
      // Your custom boot logic here
```

### Adding New Touch Actions

```yaml
tx_ultimate_touch:
  on_long_touch_release:
    - lambda: |-
        // Your custom long press action
```

## Performance Optimization

### Reduced Network Traffic

- Health checks only every 5-10 minutes
- No continuous polling
- Event-driven updates only

### Efficient LED Updates

- Uses NeoPixel effects (hardware-accelerated)
- LED updates batched
- Smooth transitions with minimal CPU

### Power Efficiency

- No busy-waiting loops
- Sleep between checks
- Minimal logging in production

## Debugging

### Enable Verbose Logging

```yaml
logger:
  level: VERBOSE
  logs:
    wifi: VERBOSE
    api: VERBOSE
    tx_ultimate_touch: VERBOSE
```

### Monitor WiFi Connection

```yaml
wifi:
  on_connect:
    - logger.log: "WiFi connected!"
  on_disconnect:
    - logger.log: "WiFi disconnected!"
```

### Monitor HA Connection

Check logs for:
```
[ha_watchdog] Home Assistant not reachable (attempt 1).
[ha_watchdog] Home Assistant down for 10+ minutes. Enabling local touch control.
[ha_watchdog] Home Assistant back online. Restoring normal touch behavior.
```

## Bluetooth Proxy

### Overview

The device includes a **Bluetooth Low Energy (BLE) proxy** that extends Home Assistant's Bluetooth range.

### Configuration

```yaml
esp32_ble_tracker:
  scan_parameters:
    interval: 1100ms    # How often to scan
    window: 1100ms      # How long to scan
    active: true        # Active scanning (request more data)

bluetooth_proxy:
  active: true
```

### How It Works

```
BLE Device (sensor, tracker)
         │
         │ Bluetooth 5.0
         ▼
    TX Ultimate
    (BLE Proxy)
         │
         │ WiFi
         ▼
  Home Assistant
```

### Use Cases

1. **Extended Range**:
   - Place TX Ultimate between HA and BLE devices
   - Each proxy extends BLE coverage by ~10-30 meters

2. **Supported Devices**:
   - Xiaomi/Mijia temperature sensors
   - Bluetooth trackers (keys, wallets)
   - Fitness bands
   - Smart locks
   - Any BLE device supported by HA

3. **Performance**:
   - **Scan interval**: 1.1 seconds
   - **Active scanning**: Requests additional device info
   - **No configuration needed**: Auto-discovered by HA

### Benefits

- **Zero configuration**: Works automatically once device is adopted
- **Low overhead**: Minimal impact on device performance
- **Reliable**: Uses proven ESPHome BLE stack
- **Secure**: All traffic encrypted through Home Assistant API

### Monitoring

Check BLE proxy status in Home Assistant:
- Navigate to **Settings** → **Devices & Services** → **ESPHome**
- Select your TX Ultimate device
- Check **Bluetooth Proxy** entity

## Future Enhancements

Potential additions:

- **MQTT Support**: Alternative to HA API
- **Web Interface**: Built-in configuration page
- **Sensor Integration**: Temperature, humidity, etc.
- **Voice Control**: I2S audio integration
- **Scene Controller**: Multi-device coordination
- **Energy Monitoring**: Power consumption tracking
- **BLE Beacon**: Broadcast iBeacon/Eddystone for presence detection
