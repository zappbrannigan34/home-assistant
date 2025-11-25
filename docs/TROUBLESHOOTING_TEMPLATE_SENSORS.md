# Troubleshooting: Template Sensors Stuck as "unavailable"

**Problem:** Template sensors show "unavailable" even though source entities are working
**Last Updated:** 2025-11-25
**Applies to:** Home Assistant 2024.x - 2025.x

---

## 🚨 Symptoms

- ✅ Configuration check passes (no YAML errors)
- ✅ Home Assistant logs show no errors
- ✅ Source entities (sensors, weather) are working and updating
- ❌ Template sensors stuck as "unavailable"
- ❌ `last_changed` timestamp never updates
- ❌ Template logic never executes

**Example:**
```yaml
template:
  - sensor:
      - name: "Adaptive Target Humidity"
        state: >
          {% set T_room = states('sensor.sensor_eva_temperature') | float(22) %}
          {{ T_room * 2 }}  # Never evaluates!
```

**Result:** `sensor.adaptive_target_humidity` = `unavailable` forever

---

## 🔍 Diagnosis

### Step 1: Verify Source Entities Work

```bash
# Replace TOKEN and URL with your values
TOKEN="your_long_lived_access_token"
URL="https://hass.example.org"

# Check source sensor
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.sensor_eva_temperature | jq '{entity_id, state, last_updated}'
```

**Expected:** Should show current value and recent `last_updated` timestamp

**If unavailable:** Problem is with SOURCE sensor, not template

### Step 2: Verify Template Entity Exists

```bash
# Check template sensor
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.adaptive_target_humidity | jq '{entity_id, state, last_changed}'
```

**Check:**
- Does entity exist? (not 404 error)
- What is `last_changed` timestamp? (old = never updated)
- Is `state` = "unavailable"?

### Step 3: Check Entity Registry

```bash
# Check when entity was created
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/config/entity_registry | \
  jq '.[] | select(.entity_id == "sensor.adaptive_target_humidity") | {entity_id, created_at, platform}'
```

**Key info:**
- `created_at`: When entity was registered
- `platform`: Should be "template"

### Step 4: Check Home Assistant Logs

```bash
# Look for template errors
grep -i "template" /config/home-assistant.log | grep -i "error\|warning"
```

**Common errors to look for:**
- Template rendering errors
- Undefined entities
- Circular dependencies

---

## 🐛 Root Cause: `states()` Function Doesn't Create Entity Tracking

### The Problem

The `states('entity.id')` function is **safe** (returns default on unavailable) but **doesn't create automatic entity tracking**.

**What this means:**
```yaml
template:
  - sensor:
      - name: "My Sensor"
        state: >
          {% set value = states('sensor.source') | float(0) %}
          {{ value * 2 }}
```

Home Assistant:
1. ✅ Creates entity `sensor.my_sensor`
2. ✅ Evaluates template ONCE at startup
3. ❌ **Never re-evaluates** when `sensor.source` changes
4. ❌ No automatic subscription to `sensor.source`

**Why?** `states()` function is designed to be "safe" (doesn't fail if entity unavailable), so HA doesn't know you want to track it.

### Alternative That DOES Work

Using direct state reference (NOT recommended for packages):
```yaml
state: >
  {{ states.sensor.source.state | float(0) * 2 }}
```

This creates tracking BUT fails loudly if entity unavailable at startup.

---

## ✅ Solution: Trigger-Based Templates

### The Fix

Add explicit `trigger:` block to create entity tracking:

```yaml
template:
  - trigger:
      # Explicit triggers for entity tracking
      - platform: state
        entity_id: sensor.source_sensor
      - platform: state
        entity_id: weather.forecast
        attribute: temperature
      # Fallback: periodic update
      - platform: time_pattern
        minutes: "/5"
    sensor:
      - name: "My Sensor"
        unique_id: my_sensor
        # Availability check prevents cascading unavailable states
        availability: >
          {{ has_value('sensor.source_sensor') }}
        state: >
          # Still use states() for safety!
          {% set value = states('sensor.source_sensor') | float(0) %}
          {{ value * 2 }}
```

### Key Components

1. **`trigger:` block** - Explicitly tells HA when to re-evaluate
   - `platform: state` - Re-evaluate when entity changes
   - `platform: time_pattern` - Fallback periodic update
   - `attribute:` - Trigger on attribute changes (e.g., weather temperature)

2. **`availability:` template** - Prevents unavailable cascade
   - Uses `has_value()` to check entity exists and has value
   - Template shows "unavailable" only if sources unavailable
   - Prevents startup issues

3. **Keep `states()` function** - Still safe for missing entities
   - `states('entity') | float(0)` - Returns 0 if unavailable
   - Doesn't cause errors if entity temporarily missing

---

## 🔧 Implementation Steps

### Step 1: Update Template Syntax

**Before (broken):**
```yaml
template:
  - sensor:
      - name: "My Sensor"
        state: >
          {{ states('sensor.source') | float(0) }}
```

**After (working):**
```yaml
template:
  - trigger:
      - platform: state
        entity_id: sensor.source
      - platform: time_pattern
        minutes: "/5"
    sensor:
      - name: "My Sensor"
        availability: >
          {{ has_value('sensor.source') }}
        state: >
          {{ states('sensor.source') | float(0) }}
```

### Step 2: Apply Changes

**IMPORTANT:** Changing from `sensor:` to `trigger:` format requires **FULL RESTART**, not reload!

```bash
# WRONG: This won't work for structural changes
curl -X POST -H "Authorization: Bearer $TOKEN" \
  $URL/api/services/homeassistant/reload_all

# CORRECT: Full restart required
curl -X POST -H "Authorization: Bearer $TOKEN" \
  $URL/api/services/homeassistant/restart
```

**Wait 60+ seconds** for Home Assistant to restart.

### Step 3: Verify Fix

```bash
# Check template sensor now has real value
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.my_sensor | jq '{entity_id, state, last_changed}'
```

**Expected:**
- `state`: Real calculated value (not "unavailable")
- `last_changed`: Recent timestamp (within last few minutes)

### Step 4: Monitor Updates

```bash
# Check again after 5 minutes to verify it's updating
sleep 300
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.my_sensor | jq '{entity_id, state, last_changed}'
```

**Expected:** `last_changed` should have updated if source changed.

---

## 📋 Verification Checklist

After implementing trigger-based templates:

- [ ] Configuration check passes
- [ ] Full restart completed (not just reload)
- [ ] Template sensor shows real value (not "unavailable")
- [ ] `last_changed` timestamp is recent
- [ ] Sensor updates when source entity changes
- [ ] Sensor updates at least every 5 minutes (time_pattern fallback)
- [ ] Sensor shows "unavailable" only when sources unavailable
- [ ] No errors in Home Assistant logs

---

## 🔗 Cross-Reference Validation

After fixing template sensors, verify they're referenced correctly:

```bash
# Extract all entity_id references from config
cd /config
grep -roh "sensor\.[a-z0-9_]*" packages/ | sort -u > /tmp/referenced_sensors.txt

# Check each one exists in HA
while read entity; do
  result=$(curl -s -H "Authorization: Bearer $TOKEN" \
    $URL/api/states/$entity | jq -r '.entity_id // "NOT_FOUND"')
  if [ "$result" = "NOT_FOUND" ]; then
    echo "❌ Missing: $entity"
  else
    echo "✅ Found: $entity"
  fi
done < /tmp/referenced_sensors.txt
```

---

## 📚 References

**Home Assistant Documentation:**
- Template Integration: https://www.home-assistant.io/integrations/template/
- Trigger-based Templates: https://www.home-assistant.io/integrations/template/#trigger-based-template-binary-sensors-buttons-images-numbers-selects-and-sensors
- Template Functions: https://www.home-assistant.io/docs/configuration/templating/

**GitHub Issues:**
- Entity Tracking: https://github.com/home-assistant/core/issues/145567
- Package Support: https://github.com/home-assistant/core/issues/50157

**Community Patterns:**
- kvazis Repository (Russian HA Expert): https://github.com/kvazis/newHA

---

## 💡 Prevention Tips

**For New Template Sensors:**

1. **Always use trigger-based format in packages:**
   ```yaml
   template:
     - trigger:  # ← Start with this
         - platform: state
           entity_id: sensor.source
   ```

2. **Always add availability checks:**
   ```yaml
   availability: >
     {{ has_value('sensor.source') }}
   ```

3. **Always add time_pattern fallback:**
   ```yaml
   - platform: time_pattern
     minutes: "/5"  # Ensures updates even if triggers fail
   ```

4. **Test with full restart:**
   - Make changes → **Restart** (not reload) → Verify

5. **Use NAMING_CONVENTIONS.md:**
   - English names for entity_id generation
   - Unique_id for persistence
   - See: `/config/docs/NAMING_CONVENTIONS.md`

---

**Last Updated:** 2025-11-25
**Tested with:** Home Assistant 2025.11.3
**Related:** [NAMING_CONVENTIONS.md](NAMING_CONVENTIONS.md), [DEBUGGING_HA.md](DEBUGGING_HA.md)
