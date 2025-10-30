# Quick Start Guide

This guide will get your TX Ultimate up and running in under 5 minutes.

## Prerequisites

- ESPHome installed (either standalone or as Home Assistant add-on)
- TX Ultimate touch panel
- Access to your Home Assistant instance

## Installation Steps

### 1. Clone the Repository

```bash
cd /config/esphome
git clone https://github.com/YOUR_USERNAME/tx-ultimate-ha-failover.git
cd tx-ultimate-ha-failover
```

### 2. Run the Installer

```bash
./install.sh
```

The installer will ask you for:

1. **WiFi Credentials**
   - Your WiFi network name (SSID)
   - WiFi password
   - Fallback access point password (used when WiFi is down)

2. **Device Information**
   - Device name (e.g., `living_tx`)
   - Friendly name (e.g., `Living-TX`)
   - Fixed IP address for the device (e.g., `192.168.1.101`)

3. **Home Assistant**
   - Your Home Assistant IP address (e.g., `192.168.1.100`)

4. **Hardware**
   - Number of relays (1, 2, or 3)

5. **Location**
   - Latitude and longitude for automatic nightlight
   - Find yours at: https://www.latlong.net/

### 3. Flash the Device

Connect your TX Ultimate via USB and run:

```bash
esphome run tx_ultimate_failover.yaml
```

Select the USB port when prompted, and wait for the flash to complete.

### 4. Test the Device

Once flashed:

1. The device will connect to your WiFi
2. Home Assistant should auto-discover it
3. Touch the panel - LEDs should light up
4. Check logs: `esphome logs tx_ultimate_failover.yaml`

## What You Get

After installation, your device will have:

### Status Indicators

- **Yellow pulsing LED**: No WiFi connection
- **Red pulsing LED**: Home Assistant offline
- **Cyan pulsing LED**: Home Assistant back online (confirmation)

### Automatic Failover

- If WiFi drops: Device restarts automatically
- If Home Assistant is unreachable for 10 minutes:
  - Buttons control relays directly (local mode)
  - Status indicator shows red pulse when touched
  - Automatically returns to normal mode when HA is back

### Touch Controls

- **Single touch**: Trigger automation in HA (or toggle relay in failover mode)
- **Long press**: Long press event
- **Swipe left/right**: Swipe events
- **Multi-touch**: Multi-touch event

### Entities in Home Assistant

You'll see these entities:

- `switch.<name>_l1` - Relay 1
- `switch.<name>_l2` - Relay 2 (if configured)
- `switch.<name>_l3` - Relay 3 (if configured)
- `binary_sensor.<name>_touchfield_1` - Touch sensor 1
- `binary_sensor.<name>_touchfield_2` - Touch sensor 2
- `binary_sensor.<name>_touchfield_3` - Touch sensor 3
- `binary_sensor.<name>_swipe_left` - Swipe left
- `binary_sensor.<name>_swipe_right` - Swipe right
- `binary_sensor.<name>_multi_touch` - Multi-touch
- `binary_sensor.<name>_long_press` - Long press
- `switch.<name>_nightlight` - Nightlight control
- `light.<name>_neopixel_light` - Full LED control

## Common Issues

### "Device not found"

If ESPHome can't find your device after flashing:

```bash
esphome logs tx_ultimate_failover.yaml
```

Check for error messages about WiFi or API connection.

### "Wrong IP address"

If the device gets a different IP than expected:

1. Check your router's DHCP settings
2. Reserve the IP for the device's MAC address
3. Update `device_ip` in the YAML file

### "Touch not responding"

Make sure the external component is available:

```bash
# If using Home Assistant
mkdir -p /config/esphome/components
# Copy tx_ultimate_touch component to this directory
```

## Next Steps

### Customize Colors

Edit `tx_ultimate_failover.yaml` to change LED colors:

```yaml
substitutions:
  button_color: "{100,0,0}"      # Red buttons
  touch_color: "{0,100,0}"       # Green touch feedback
```

See [CONFIGURATION_EXAMPLES.md](CONFIGURATION_EXAMPLES.md) for more options.

### Create Home Assistant Automations

Example automation to control lights:

```yaml
automation:
  - alias: "Living TX - Button 1"
    trigger:
      - platform: state
        entity_id: binary_sensor.living_tx_touchfield_1
        to: "on"
    action:
      - service: light.toggle
        target:
          entity_id: light.living_room
```

### Adjust Failover Timing

To make the failover more/less aggressive:

```yaml
substitutions:
  wifi_check_interval: "300s"     # Check WiFi every 5 min
  ha_check_interval: "120s"       # Check HA every 2 min
```

### Monitor Device Health

Add a template sensor in Home Assistant:

```yaml
template:
  - sensor:
      - name: "Living TX Status"
        state: >
          {% if is_state('binary_sensor.living_tx_status', 'on') %}
            Online
          {% else %}
            Offline
          {% endif %}
```

## Support

- **Issues**: https://github.com/YOUR_USERNAME/tx-ultimate-ha-failover/issues
- **Discussions**: https://github.com/YOUR_USERNAME/tx-ultimate-ha-failover/discussions
- **Documentation**: See [README.md](README.md) for full documentation

## Updates

To update to the latest version:

```bash
cd /config/esphome/tx-ultimate-ha-failover
git pull
```

Then re-flash your device:

```bash
esphome run tx_ultimate_failover.yaml
```

Your `secrets.yaml` and custom configurations will be preserved.
