# TX Ultimate HA Failover

ESPHome configuration for TX Ultimate touch panel with Home Assistant failover support.

## Features

- **WiFi Watchdog**: Automatic restart if WiFi connection is lost
- **Home Assistant Failover**: Enables local relay control when HA is unreachable
  - **Smart Ping Detection**: Uses ICMP ping instead of TCP connections for more reliable HA monitoring
  - Tracks packet loss and latency for better diagnostics
- **Visual Status Indicators**: LED feedback for connectivity status
  - Yellow pulsing: No WiFi
  - Red pulsing: Home Assistant offline
  - Cyan pulsing: Home Assistant back online
- **Bluetooth Proxy**: Built-in Bluetooth Low Energy proxy for Home Assistant
- **Customizable Colors**: All LED colors configurable via substitutions
- **Touch Panel Control**: Full support for touch, swipe, long press, and multi-touch gestures
- **Nightlight Mode**: Automatic nightlight based on sunset/sunrise

## Documentation

- **[Quick Start Guide](QUICK_START.md)** - Get up and running in 5 minutes
- **[Configuration Examples](CONFIGURATION_EXAMPLES.md)** - Ready-to-use configuration templates
- **[Architecture](ARCHITECTURE.md)** - Technical details and system design

## Installation

You can install this configuration either using the automated script or manually.

### Option A: Automated Installation (Recommended)

```bash
cd /config/esphome
git clone https://github.com/YOUR_USERNAME/tx-ultimate-ha-failover.git
cd tx-ultimate-ha-failover
./install.sh
```

The installer will guide you through:
- Setting up WiFi credentials
- Configuring device name and IP
- Setting Home Assistant IP
- Configuring location for nightlight
- Generating encryption keys

### Option B: Manual Installation

#### 1. Clone this repository

```bash
cd /config/esphome
git clone https://github.com/YOUR_USERNAME/tx-ultimate-ha-failover.git
```

#### 2. Copy and configure secrets file

```bash
cd tx-ultimate-ha-failover
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml` with your credentials:

```yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"
ap_pass: "FallbackAPPassword"
api_key: "YourAPIEncryptionKey"
```

#### 3. Customize your device

Edit [tx_ultimate_failover.yaml](tx_ultimate_failover.yaml) and modify the substitutions section:

```yaml
substitutions:
  # Device identification
  name: 'living_tx'
  friendly_name: "Living-TX"

  # Network settings
  ha_ip: "192.168.50.168"          # Your Home Assistant IP
  ha_port: "8123"                   # HA port
  device_ip: "192.168.35.56"        # Fixed IP for this device

  # Monitoring intervals
  wifi_check_interval: "600s"       # WiFi watchdog check
  ha_check_interval: "300s"         # HA availability check

  # LED Colors (RGB format 0-100)
  button_color: "{0,0,90}"          # Button active color (blue)
  nightlight_color: "{80,70,0}"     # Nightlight color (warm)
  touch_color: "{0,100,100}"        # Touch feedback color (cyan)
  # ... more colors ...
```

#### 4. Flash to your device

```bash
esphome run tx_ultimate_failover.yaml
```

For more configuration examples, see [CONFIGURATION_EXAMPLES.md](CONFIGURATION_EXAMPLES.md)

## Configuration Options

### Substitutions Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `ha_ip` | `192.168.50.168` | Home Assistant IP address |
| `ha_port` | `8123` | Home Assistant port |
| `device_ip` | `192.168.35.56` | Fixed IP for this device |
| `wifi_check_interval` | `600s` | How often to check WiFi (10 min) |
| `ha_check_interval` | `300s` | How often to check HA (5 min) |
| `relay_count` | `2` | Number of relays (1-3) |

### Color Customization

All colors use RGB format with values 0-100:
- `{0,0,90}` = Blue
- `{80,70,0}` = Warm white
- `{0,100,100}` = Cyan
- `{100,0,0}` = Red
- `{0,100,0}` = Green

### Location Settings

For proper nightlight automation, set your location:

```yaml
substitutions:
  latitude: "53.47450213193437°"
  longitude: "-6.246157438775786°"
```

## How It Works

### Failover Logic

1. **WiFi Check** (every 10 minutes):
   - If WiFi lost: Device restarts automatically
   - Status indicator: Yellow pulsing LED

2. **Home Assistant Check** (every 5 minutes using ICMP ping):
   - Sends 5 ping packets to Home Assistant
   - Monitors packet loss percentage (>80% = HA down)
   - Tracks latency for diagnostics
   - After 2 failed attempts (10 minutes): Enables local control
   - Status indicator: Red pulsing LED
   - When HA returns: Cyan pulsing confirmation

3. **Local Control Mode**:
   - Touching buttons directly toggles relays
   - No Home Assistant automation required
   - Automatic return to normal mode when HA is back

4. **Bluetooth Proxy**:
   - Extends Home Assistant's Bluetooth range
   - Allows HA to communicate with BLE devices through this device
   - No additional configuration needed

## Project Structure

```
tx-ultimate-ha-failover/
├── README.md                        # This file
├── CONFIGURATION_EXAMPLES.md        # Configuration examples and templates
├── install.sh                       # Automated installation script
├── tx_ultimate_failover.yaml        # Main ESPHome config
├── esphome_tx_sonoff.yaml          # Original config (reference)
├── secrets.yaml.example             # Template for credentials
└── .gitignore                       # Prevents secrets from being committed
```

## Troubleshooting

### Device not connecting to WiFi
- Check `secrets.yaml` has correct WiFi credentials
- Verify `device_ip` is in your network range
- Check your router allows the fixed IP

### Home Assistant not detected
- Verify `ha_ip` is correct in substitutions
- Test connectivity: `ping 192.168.50.168` from device network
- Check HA port (default 8123)

### Touch panel not responding
- Verify external component is correctly installed at `/config/esphome/components`
- Check UART logs for touch events

## Contributing

Pull requests are welcome! For major changes, please open an issue first.

## License

[MIT](LICENSE)

## Credits

Based on the original TX Ultimate configuration by SmartHomeYourself
