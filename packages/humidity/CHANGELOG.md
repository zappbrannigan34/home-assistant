# Changelog: Humidity Control Package

**Package:** `puh_9105_humidity`
**Device:** Polaris PUH 9105/2709 Humidifier (Zigbee)
**Repository:** https://github.com/zappbrannigan34/home-assistant

---

## [2.0.4] - 2025-11-25

### 🐛 Fixed
- **CRITICAL:** Template sensors stuck as "unavailable" despite source entities working
- Root cause: `states()` function doesn't create entity tracking

### 🔄 Changed
- **BREAKING:** Converted to trigger-based templates
  - Added `trigger:` block with explicit `platform: state` triggers
  - Added `platform: time_pattern` fallback (every 5 minutes)
  - Added `availability:` templates with `has_value()`
  - Kept `states()` function for safety

- Removed automation `description:` fields (kvazis pattern)

### ✅ Verified
```
sensor.adaptive_target_humidity: 40.0%
sensor.humidity_error: 16.1%
sensor.recommended_intensity: 7
```

### ⚠️ Migration
- **Requires FULL RESTART** (not reload_all)
- See: [TROUBLESHOOTING_TEMPLATE_SENSORS.md](../../docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md)

---

## [2.0.3] - 2025-11-24

### 🐛 Fixed
- Reverted incorrect `sensor_eva_*` → `sensor_zap_*` changes from v2.0.2
- Both sensor sets exist and are valid

### ❌ Notes
- Version 2.0.2 changes were unauthorized and broke automations
- Spook warnings were false positives

---

## [2.0.2] - 2025-11-24 [REVERTED]

### ❌ DO NOT USE
- Unauthorized sensor renaming
- Broke all humidity automations
- Fully reverted in v2.0.3

---

## [2.0.1] - 2025-11-24

### ✅ Features
- Adaptive target humidity (WHO + ASHRAE guidelines: 40-60%)
- Hybrid intensity control (0-7 scale)
- Manual override detection
- Device initialization
- Error notifications (Russian)

### 🚨 Known Issues
- Template sensors using `states()` may not update
- Fixed in v2.0.4

---

## Version Comparison

| Version | Type | Tracking | Restart | Status |
|---------|------|----------|---------|--------|
| 2.0.4 | Trigger | ✅ | Yes* | ✅ **Recommended** |
| 2.0.3 | State | ❌ | No | ⚠️ Sensors unreliable |
| 2.0.2 | State | ❌ | No | ❌ **Broken** |
| 2.0.1 | State | ❌ | No | ⚠️ Sensors unreliable |

\* Only on upgrade from state-based to trigger-based

---

## Quick Migration: 2.0.3 → 2.0.4

1. Update `humidity_control.yaml`
2. Config check
3. **Restart Home Assistant** (not reload!)
4. Wait 60s
5. Verify sensors show real values

---

**See also:**
- [Full Changelog](../../docs/LESSONS_LEARNED.md#version-history)
- [Troubleshooting Guide](../../docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md)
- [kvazis Repository](https://github.com/kvazis/newHA)
