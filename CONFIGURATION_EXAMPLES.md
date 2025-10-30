# Configuration Examples

## Quick Start Configurations

### Example 1: Single Relay Setup

```yaml
substitutions:
  name: 'bedroom_tx'
  friendly_name: "Bedroom-TX"
  relay_count: "1"
  device_ip: "192.168.1.101"
  ha_ip: "192.168.1.100"
```

### Example 2: Triple Relay Setup

```yaml
substitutions:
  name: 'kitchen_tx'
  friendly_name: "Kitchen-TX"
  relay_count: "3"
  device_ip: "192.168.1.102"
  ha_ip: "192.168.1.100"
```

### Example 3: Fast Response Configuration

For faster failover detection (useful for critical lights):

```yaml
substitutions:
  wifi_check_interval: "300s"     # Check WiFi every 5 min
  ha_check_interval: "120s"       # Check HA every 2 min
```

## Color Customization Examples

### Warm White Theme

```yaml
substitutions:
  button_color: "{80,70,20}"
  nightlight_color: "{90,60,10}"
  touch_color: "{80,70,20}"
```

### Cool Blue Theme

```yaml
substitutions:
  button_color: "{0,30,100}"
  nightlight_color: "{0,20,80}"
  touch_color: "{0,50,100}"
```

### RGB Rainbow Theme

```yaml
substitutions:
  button_color: "{100,0,0}"       # Red when on
  touch_color: "{0,100,100}"      # Cyan touch
  long_press_color: "{100,0,100}" # Magenta long press
  swipe_left_color: "{0,100,0}"   # Green swipe left
  swipe_right_color: "{0,0,100}"  # Blue swipe right
```

### Status Indicator Color Customization

Change the failover indicator colors:

```yaml
substitutions:
  # Orange for no WiFi instead of yellow
  no_wifi_color_r: "1.0"
  no_wifi_color_g: "0.5"
  no_wifi_color_b: "0.0"

  # Purple for HA down instead of red
  ha_down_color_r: "0.5"
  ha_down_color_g: "0.0"
  ha_down_color_b: "0.5"

  # Green for HA online instead of cyan
  ha_online_color_r: "0.0"
  ha_online_color_g: "1.0"
  ha_online_color_b: "0.0"
```

## Network Configuration Examples

### Static IP with VLAN

```yaml
substitutions:
  device_ip: "192.168.10.56"
  ha_ip: "192.168.10.100"
  ha_port: "8123"
```

### Custom Home Assistant Port

If you use a reverse proxy or custom port:

```yaml
substitutions:
  ha_ip: "192.168.1.100"
  ha_port: "443"
```

### Multiple Devices in Same Home

```yaml
# Device 1 - Living Room
substitutions:
  name: 'living_tx'
  friendly_name: "Living-TX"
  device_ip: "192.168.1.101"
  ap_ssid: "TX-Living-Fallback"

# Device 2 - Bedroom
substitutions:
  name: 'bedroom_tx'
  friendly_name: "Bedroom-TX"
  device_ip: "192.168.1.102"
  ap_ssid: "TX-Bedroom-Fallback"

# Device 3 - Kitchen
substitutions:
  name: 'kitchen_tx'
  friendly_name: "Kitchen-TX"
  device_ip: "192.168.1.103"
  ap_ssid: "TX-Kitchen-Fallback"
```

## Location Examples

### Dublin, Ireland

```yaml
substitutions:
  latitude: "53.3498°"
  longitude: "-6.2603°"
```

### New York, USA

```yaml
substitutions:
  latitude: "40.7128°"
  longitude: "-74.0060°"
```

### Sydney, Australia

```yaml
substitutions:
  latitude: "-33.8688°"
  longitude: "151.2093°"
```

### Tokyo, Japan

```yaml
substitutions:
  latitude: "35.6762°"
  longitude: "139.6503°"
```

## Advanced Configurations

### Energy-Saving Mode

Reduce check frequency to save power:

```yaml
substitutions:
  wifi_check_interval: "900s"        # 15 minutes
  ha_check_interval: "600s"          # 10 minutes
  nightlight_check_interval: "10"    # 10 minutes
```

### Critical System Mode

Maximum responsiveness for critical applications:

```yaml
substitutions:
  wifi_check_interval: "120s"        # 2 minutes
  ha_check_interval: "60s"           # 1 minute
  nightlight_check_interval: "1"     # 1 minute
```

### Disable Nightlight

```yaml
substitutions:
  nightlight: "off"
```

### Different Relay Restore Modes

By default, relays restore to OFF. To change this behavior, modify the switch section:

```yaml
switch:
  - platform: gpio
    id: relay_1
    name: "${friendly_name} L1"
    pin: ${relay_1_pin}
    restore_mode: RESTORE_DEFAULT_ON    # Change to ON
    # Other options: ALWAYS_ON, ALWAYS_OFF
```

## Troubleshooting Configurations

### Increase Logging

```yaml
logger:
  level: VERBOSE  # Change from DEBUG to VERBOSE
  logs:
    wifi: VERBOSE
    api: VERBOSE
```

### Disable WiFi Watchdog (for testing)

Set a very long interval:

```yaml
substitutions:
  wifi_check_interval: "86400s"  # 24 hours
```

### Test HA Connection Manually

Add a button to test HA connectivity:

```yaml
button:
  - platform: template
    name: "Test HA Connection"
    on_press:
      - lambda: |-
          WiFiClient client;
          if (client.connect("${ha_ip}", ${ha_port})) {
            ESP_LOGI("test", "Home Assistant is reachable!");
            client.stop();
          } else {
            ESP_LOGW("test", "Cannot reach Home Assistant!");
          }
```

## Complete Example Configuration

A complete example combining multiple features:

```yaml
substitutions:
  # Device
  name: 'main_hallway_tx'
  friendly_name: "Main Hallway TX"

  # Network
  ha_ip: "192.168.1.100"
  ha_port: "8123"
  device_ip: "192.168.1.105"
  ap_ssid: "TX-Hallway-AP"

  # Hardware
  relay_count: "3"

  # Timing
  wifi_check_interval: "300s"
  ha_check_interval: "180s"
  nightlight_check_interval: "5"

  # Colors - Warm theme
  button_color: "{80,60,20}"
  nightlight_color: "{90,50,10}"
  touch_color: "{80,60,20}"
  long_press_color: "{100,20,0}"

  # Location - London
  latitude: "51.5074°"
  longitude: "-0.1278°"
```
