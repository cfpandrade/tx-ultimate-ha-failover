# ⚠️ IMPORTANT: Use the Complete Configuration File

## The Problem
The README example is **simplified** and missing many required substitutions. This will cause warnings during compilation.

## The Solution
**Use the complete template:** [tx_ultimate_minimal.yaml](tx_ultimate_minimal.yaml)

This file includes ALL required substitutions with sensible defaults.

## What You Need to Change
Only these **7 values** in the "CHANGE ME" section:

```yaml
substitutions:
  ###### CHANGE ME START ######
  name: "living-tx"                   # ← Change this
  friendly_name: "Living Room TX"     # ← Change this
  ha_ip: "192.168.1.100"              # ← Change this
  device_ip: "192.168.1.101"          # ← Change this
  relay_count: "2"                    # ← Change this (1, 2, or 3)
  latitude: "40.7128°"                # ← Change this
  longitude: "-74.0060°"              # ← Change this
  ###### CHANGE ME END ######
```

Everything else has working defaults!

## Quick Start

1. Download: [tx_ultimate_minimal.yaml](https://raw.githubusercontent.com/cfpandrade/tx-ultimate-ha-failover/main/tx_ultimate_minimal.yaml)
2. Change only the 7 values above
3. Copy to ESPHome
4. Add secrets
5. Install

Done! ✅
