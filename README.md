# TX Ultimate HA Failover

ESPHome configuration for TX Ultimate touch panel with **automatic Home Assistant failover support**.

When Home Assistant goes offline, your TX Ultimate automatically switches to **manual mode** - buttons work as physical switches until HA comes back online.

## 🚀 Features

- **Home Assistant Failover**: Automatic manual control when HA is unreachable
  - HTTP heartbeat monitoring every 60 seconds
  - Activates manual mode after 10 minutes offline
  - Auto-restores HA control when back online
  - Saves relay states and restores them on recovery
- **WiFi Watchdog**: Auto-restart if WiFi connection is lost
- **Visual Indicators**: Red nightlight when HA is offline
- **Optional Auto-On**: Configure lights to turn on in failover (living room) or stay off (bedrooms)
- **Touch Panel Control**: Full support for touch, swipe, long press, and multi-touch gestures
- **Nightlight Mode**: Automatic nightlight based on sunset/sunrise

## 📦 Installation (Remote Package - Recommended)

**No git clone needed!** Just create one YAML file and ESPHome loads everything from GitHub.

### Step 1: Create your device YAML

In your ESPHome folder, create a new file (e.g., `living-tx.yaml`):

```yaml
substitutions:
  ###### CHANGE ME START ######
  name: "living-tx"
  friendly_name: "Living Room TX"

  # Home Assistant
  ha_url: "https://192.168.1.100:8123"
  ha_api_token: !secret tx_status_api_token  # See Step 2

  # Hardware
  relay_count: "2"  # Number of relays: 1, 2, or 3

  # Failover behavior
  ha_failover_turn_on_lights: "true"  # true for living room, false for bedrooms

  # Location (for nightlight)
  latitude: !secret latitude
  longitude: !secret longitude

  # LED Colors (RGB 0-100)
  button_brightness: "0"
  button_color: "{0,0,90}"
  nightlight: "on"
  nightlight_brightness: "0.2"
  nightlight_color: "{80,70,0}"
  touch_brightness: "1"
  touch_color: "{0,100,100}"
  touch_effect: "Scan"
  long_press_brightness: "1"
  long_press_color: "{100,0,0}"
  long_press_effect: ""
  multi_touch_brightness: "1"
  multi_touch_color: "{0,0,0}"
  multi_touch_effect: "Rainbow"
  swipe_left_brightness: "1"
  swipe_left_color: "{0,100,0}"
  swipe_left_effect: ""
  swipe_right_brightness: "1"
  swipe_right_color: "{100,0,70}"
  swipe_right_effect: ""

  # Timings
  vibra_time: 400ms
  button_on_time: 500ms

  # Pins (TX Ultimate standard - don't change)
  relay_1_pin: GPIO18
  relay_2_pin: GPIO17
  relay_3_pin: GPIO27
  relay_4_pin: GPIO23
  vibra_motor_pin: GPIO21
  pa_power_pin: GPIO26
  led_pin: GPIO13
  status_led_pin: GPIO33
  uart_tx_pin: GPIO19
  uart_rx_pin: GPIO22
  audio_lrclk_pin: GPIO4
  audio_bclk_pin: GPIO2
  audio_sdata_pin: GPIO15
  touchpanel_power_pin: GPIO5
  ###### CHANGE ME END ######

##### DO NOT CHANGE BELOW! #####

packages:
  remote_package:
    url: https://github.com/cfpandrade/tx-ultimate-ha-failover
    ref: main  # or use 'v1.3.1' for specific version
    files: [tx_ultimate_base.yaml]
    refresh: 300s

##### My customization - Start #####

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_pass

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  use_address: 192.168.35.56  # Optional: your device IP
  ap:
    ssid: "Tx-Living"
    password: !secret ap_pass

##### My customization - End #####
```

### Step 2: Get Home Assistant Long-Lived Token

1. In Home Assistant, click your **profile** (bottom left)
2. Scroll to **"Long-Lived Access Tokens"**
3. Click **"CREATE TOKEN"**
4. Name it "TX Living Room"
5. **Copy the token** (starts with "eyJ...")
6. Add to `secrets.yaml`:
   ```yaml
   tx_status_api_token: "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
   ```
   **Important**: Must start with "Bearer "

### Step 3: Configure secrets.yaml

Create or update `secrets.yaml` in your ESPHome folder:

```yaml
# WiFi
wifi_ssid: "YourWiFiName"
wifi_password: "YourWiFiPassword"
ap_pass: "FallbackAP123"

# ESPHome
api_key: "generate-this-in-esphome-dashboard"  # Click the key icon
ota_pass: "your-ota-password"

# Location
latitude: "40.7128°"
longitude: "-74.0060°"

# HA Token (from Step 2)
tx_status_api_token: "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Step 4: Flash to device

```bash
esphome run living-tx.yaml
```

Or use the ESPHome dashboard **"Install"** button.

**Done!** ✅

---

## 📖 How It Works

### Normal Operation
- Buttons trigger Home Assistant automations
- HA controls the lights through automations
- Visual feedback on all gestures

### Failover Mode (HA Offline)
1. **Detection**: HTTP heartbeat fails for 10 minutes (10 consecutive checks)
2. **Activation**:
   - Saves current relay states
   - Enables manual button control
   - Optional: Turns on all lights (if `ha_failover_turn_on_lights: "true"`)
   - Nightlight turns **red** to indicate offline mode
3. **Recovery**:
   - When HA comes back online, restores saved states
   - Returns buttons to automation mode
   - Nightlight returns to normal color

### WiFi Watchdog
- Checks WiFi every 10 minutes
- Auto-restarts device if WiFi lost for >3 minutes

---

## 🎨 Customization

### Failover Behavior

**Living Room / Dining Room** (turn on lights):
```yaml
ha_failover_turn_on_lights: "true"
```

**Bedroom / Bathroom** (keep current state):
```yaml
ha_failover_turn_on_lights: "false"
```

### LED Colors

RGB format (0-100):
- Blue: `{0,0,90}`
- Red: `{100,0,0}`
- Green: `{0,100,0}`
- Cyan: `{0,100,100}`
- Warm White: `{80,70,0}`

### Relay Count

```yaml
relay_count: "1"  # Single button (middle)
relay_count: "2"  # Two buttons (left + right)
relay_count: "3"  # Three buttons (left + middle + right)
```

---

## 📁 Repository Structure

```
tx-ultimate-ha-failover/
├── README.md                    # This file
├── tx_ultimate_base.yaml        # Core configuration (loaded from GitHub)
├── tx_ultimate_example.yaml     # Full example with all options
├── secrets.yaml.example         # Template for credentials
└── .gitignore                   # Prevents secrets from being committed
```

---

## 🔧 Troubleshooting

### "HA Online" sensor shows offline
- Check `ha_url` is correct and accessible from device
- Verify `ha_api_token` in secrets.yaml starts with "Bearer "
- Test: `curl -H "Authorization: Bearer YOUR_TOKEN" https://192.168.1.100:8123/api/`

### Buttons don't work in normal mode
- This is **correct** - buttons should trigger HA automations
- Create automations in HA for each touchfield
- Example HA automation:
  ```yaml
  - alias: "Living TX - Button 1"
    trigger:
      - platform: state
        entity_id: binary_sensor.touchfield_1
        to: 'on'
    action:
      - service: light.toggle
        target:
          entity_id: light.living_room
  ```

### Buttons work as switches immediately
- You're in failover mode (HA is offline or not configured)
- Check logs: ESPHome → Device → Logs
- Look for "HA offline" or "activando MODO MANUAL"

### WiFi keeps restarting device
- Check WiFi credentials in secrets.yaml
- Ensure `use_address` IP is correct and available
- Disable WiFi watchdog temporarily (remove interval section)

---

## 🆚 Versions

### Use stable version (recommended):
```yaml
packages:
  remote_package:
    url: https://github.com/cfpandrade/tx-ultimate-ha-failover
    ref: v1.3.1  # Specific version
    files: [tx_ultimate_base.yaml]
```

### Use latest (auto-updates):
```yaml
packages:
  remote_package:
    url: https://github.com/cfpandrade/tx-ultimate-ha-failover
    ref: main  # Latest changes
    files: [tx_ultimate_base.yaml]
```

---

## 🤝 Contributing

Pull requests welcome! For major changes, please open an issue first.

## 📜 License

MIT License

## 💡 Credits

Based on the original TX Ultimate configuration by [SmartHomeYourself](https://github.com/SmartHome-yourself/sonoff-tx-ultimate-for-esphome)

---

## 📚 Advanced: Full Example

See [tx_ultimate_example.yaml](tx_ultimate_example.yaml) for a complete configuration with all available options and detailed comments.
