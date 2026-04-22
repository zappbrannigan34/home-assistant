# Home Assistant Debugging Guide

**Purpose:** Practical commands and techniques for investigating HA issues
**Last Updated:** 2025-11-25
**Prerequisites:** Long-lived access token, curl, jq

---

## 🔧 Setup

### Get API Credentials

```bash
# Your credentials
TOKEN="your_long_lived_access_token"
URL="https://hass.example.org"

# Test API access
curl -s -H "Authorization: Bearer $TOKEN" $URL/api/ | jq
# Expected: {"message": "API running."}
```

### Create Token
1. HA UI → Profile (bottom left) → Long-Lived Access Tokens
2. Create Token → Copy → Save securely

---

## 📊 Entity State Inspection

### Check Single Entity

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.adaptive_target_humidity | jq
```

### Check Entity with Key Fields Only

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.adaptive_target_humidity | \
  jq '{entity_id, state, last_changed, last_updated}'
```

### Check Multiple Entities (Pattern)

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states | \
  jq '.[] | select(.entity_id | test("adaptive_target|humidity_error|recommended_intensity")) | {entity_id, state}'
```

### Find Unavailable Entities

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states | \
  jq '.[] | select(.state == "unavailable") | .entity_id'
```

---

## 🗂️ Entity Registry

### Check Entity Creation Time

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/config/entity_registry | \
  jq '.[] | select(.entity_id == "sensor.adaptive_target_humidity") | {entity_id, created_at, modified_at, platform, unique_id}'
```

### List All Template Entities

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/config/entity_registry | \
  jq '.[] | select(.platform == "template") | {entity_id, created_at}'
```

### Find Orphaned Entities (deleted but registered)

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/config/entity_registry | \
  jq '.[] | select(.disabled_by != null or .hidden_by != null) | {entity_id, disabled_by, hidden_by}'
```

---

## 🔍 Cross-Reference Validation

### Extract Entity References from Config

```bash
cd /config
grep -roh "sensor\.[a-z0-9_]*" packages/ | sort -u > /tmp/referenced_sensors.txt
```

### Verify Each Referenced Entity Exists

```bash
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

## 🚨 Repairs Registry

### Check Active Repair Issues

```bash
# Note: Requires local file access
cat /config/.storage/repairs.issue_registry | jq '.data.issues | length'
```

### List Spook Integration Warnings

```bash
cat /config/.storage/repairs.issue_registry | \
  jq '.data.issues[] | select(.domain == "spook") | {issue_id, translation_key}'
```

---

## 🔄 Service Calls

### Reload Template Integration

```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  $URL/api/services/homeassistant/reload_all
```

### Restart Home Assistant

```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  $URL/api/services/homeassistant/restart

# Wait for restart
sleep 60
curl -s -H "Authorization: Bearer $TOKEN" $URL/api/ | jq -r '.message'
```

---

## 📈 Monitoring Entity Updates

### Watch Entity State Changes (Real-time)

```bash
# Initial state
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.adaptive_target_humidity | \
  jq -r '"\(.entity_id): \(.state) at \(.last_changed)"'

# Wait 5 minutes
sleep 300

# Check again
curl -s -H "Authorization: Bearer $TOKEN" \
  $URL/api/states/sensor.adaptive_target_humidity | \
  jq -r '"\(.entity_id): \(.state) at \(.last_changed)"'

# Compare timestamps - should be different if updating
```

### Workflow: Analyze Dashboard Screenshot History Correctly

Если пользователь прислал скриншот графика и нужно понять, что происходило на графике:

1. **Сначала определить entity по YAML дашборда**
   - Не гадать по цветам на память.
   - Открыть соответствующий `dashboards/*.yaml` и сопоставить линии с entity.

2. **Не угадывать временное окно по скриншоту**
   - Ось X на картинке не является надёжным источником времени сама по себе.
   - Сначала зафиксировать предполагаемый интервал, потом подтвердить его через API-историю.

3. **Читать историю через HA API, а не через recorder DB, если просят API**

```bash
# Пример: история zap-графика вентиляции
curl -s -H "Authorization: Bearer $TOKEN" \
  "$URL/api/history/period/2026-04-21T22:20:00Z?end_time=2026-04-22T00:35:00Z&filter_entity_id=sensor.sensor_zap_co2,sensor.ventilation_co2_error,cover.ventilation_zap_group,sensor.ventilation_recommended_position" | jq
```

4. **Разбирать actuator отдельно от controller**
   - На вентиляционном графике `cover.*.current_position` — это physical actuator layer.
   - `sensor.ventilation*_recommended_position` нужно читать отдельно, если надо понять controller behavior.

5. **Только после этого делать вывод**
   - Что из линий — CO2
   - Что — error
   - Что — actual window position
   - Что происходило с recommendation в тот же интервал

**Нельзя делать так:**
- сначала угадывать дату/время по скриншоту;
- потом подменять API-анализ чтением `home-assistant_v2.db`;
- потом делать вывод о контроллере, не отделив его от actuator line.

---

## 🛠️ Common Debugging Workflows

### Workflow 1: Template Sensor Not Updating

1. **Check sensor exists:**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
     $URL/api/states/sensor.my_sensor | jq '.entity_id'
   ```

2. **Check current state:**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
     $URL/api/states/sensor.my_sensor | \
     jq '{state, last_changed, last_updated}'
   ```

3. **Check source entities working:**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
     $URL/api/states/sensor.source_sensor | \
     jq '{state, last_updated}'
   ```

4. **Check entity registry:**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
     $URL/api/config/entity_registry | \
     jq '.[] | select(.entity_id == "sensor.my_sensor") | {platform, created_at}'
   ```

5. **Diagnose:** If `last_changed` is old + source updating → entity tracking issue

### Workflow 2: Entity Not Found (404)

1. **Search entity registry:**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
     $URL/api/config/entity_registry | \
     jq '.[] | select(.entity_id | contains("my_sensor"))'
   ```

2. **Check similar entity_id (typo?):**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
     $URL/api/states | \
     jq '.[] | .entity_id' | grep -i "my_sensor"
   ```

3. **Diagnose:** If not in registry → entity not created (YAML error, needs restart)

---

## 📚 jq Cheatsheet

### Filter by Entity ID Pattern
```bash
jq '.[] | select(.entity_id | test("pattern"))'
```

### Filter by State Value
```bash
jq '.[] | select(.state == "unavailable")'
```

### Extract Specific Fields
```bash
jq '{entity_id, state, last_changed}'
```

### Count Results
```bash
jq 'length'
```

### Sort by Field
```bash
jq 'sort_by(.last_changed)'
```

---

## 🔗 References

- **HA REST API:** https://developers.home-assistant.io/docs/api/rest/
- **Entity Registry:** https://www.home-assistant.io/integrations/entity_registry/
- **jq Manual:** https://stedolan.github.io/jq/manual/
- **Related:** [TROUBLESHOOTING_TEMPLATE_SENSORS.md](TROUBLESHOOTING_TEMPLATE_SENSORS.md)

---

**Last Updated:** 2025-11-25
**Maintained by:** zappbrannigan34
