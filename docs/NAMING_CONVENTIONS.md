# Home Assistant - Naming Conventions

**Repository:** https://github.com/zappbrannigan34/home-assistant

Единый справочник именования ВСЕХ типов сущностей в Home Assistant.

**Source:** kvazis pattern (https://github.com/kvazis/homeassistant)

---

## 📋 ОГЛАВЛЕНИЕ

1. [Automations](#automations)
2. [Template Sensors](#template-sensors)
3. [Binary Sensors](#binary-sensors)
4. [Input Helpers](#input-helpers)
5. [Scripts](#scripts)
6. [Scenes](#scenes)

---

## 1. AUTOMATIONS

### Kvazis Pattern

```yaml
automation:
  - id: "Описание на РУССКОМ для людей"
    alias: technical_entity_id_name
    description: "Опциональное описание"
```

### Entity ID Generation

```
id → IGNORED (только для UI)
alias → entity_id через slugify
```

**Slugify rules:**
- Lowercase
- Latin letters only [a-z0-9_]
- Spaces → underscores
- Special chars → removed

### Примеры

```yaml
# ✅ ПРАВИЛЬНО:
- id: "Инициализация увлажнителя"
  alias: humidity_device_setup
  # → automation.humidity_device_setup

- id: "Управление интенсивностью увлажнителя"
  alias: humidity_control_main
  # → automation.humidity_control_main

- id: "Детектор ручного управления"
  alias: humidity_manual_control_detector
  # → automation.humidity_manual_control_detector

# ❌ НЕПРАВИЛЬНО:
- id: humidity_device_setup  # ← НЕ для людей
  alias: "Инициализация увлажнителя"  # ← НЕ для entity_id
```

---

## 2. TEMPLATE SENSORS

### Modern Format (для configuration.yaml)

```yaml
template:
  - sensor:
      - name: "Display Name for UI"
        unique_id: persistent_identifier
        unit_of_measurement: "%"
        state_class: measurement
        icon: mdi:icon-name
        state: >
          {{ template }}
```

### Legacy Format (для packages - ОБЯЗАТЕЛЬНО!)

**⚠️ ВАЖНО:** Modern format `template:` НЕ РАБОТАЕТ в packages!

**Используй ТОЛЬКО legacy format в packages:**

```yaml
sensor:
  - platform: template
    sensors:
      sensor_name:
        friendly_name: "Display Name"
        unique_id: sensor_name
        unit_of_measurement: "%"
        value_template: >
          {{ template }}
```

### Entity ID Generation

```
name (modern) OR sensor_name (legacy) → entity_id через slugify
```

**Slugify rules:**
- Lowercase
- Replace spaces with underscores
- Keep only [a-z0-9_]
- Результат: sensor.{slugified_name}

### Примеры

**Modern format (configuration.yaml):**
```yaml
template:
  - sensor:
      - name: "Adaptive Target Humidity"
        unique_id: adaptive_target_humidity
        unit_of_measurement: "%"
        state_class: measurement
        # → sensor.adaptive_target_humidity

      - name: "Humidity Error"
        unique_id: humidity_error
        # → sensor.humidity_error

      - name: "Recommended Intensity"
        unique_id: recommended_intensity
        # → sensor.recommended_intensity
```

**Legacy format (packages):**
```yaml
sensor:
  - platform: template
    sensors:
      adaptive_target_humidity:
        friendly_name: "Adaptive Target Humidity"
        unique_id: adaptive_target_humidity
        # → sensor.adaptive_target_humidity

      humidity_error:
        friendly_name: "Humidity Error"
        # → sensor.humidity_error
```

### ⚠️ CRITICAL: Name Language for entity_id

**entity_id генерируется из `name` через slugify.**

**ПРАВИЛО:** Используй АНГЛИЙСКИЕ имена в `name` для предсказуемого entity_id.

```yaml
# ✅ ПРАВИЛЬНО:
- name: "Adaptive Target Humidity"
  unique_id: adaptive_target_humidity
  # → sensor.adaptive_target_humidity (predictable!)

# ❌ НЕПРАВИЛЬНО:
- name: "Адаптивная целевая влажность"
  unique_id: adaptive_target_humidity
  # → sensor.adaptivnaia_tselevaia_vlazhnost (unpredictable!)
  # Automations will reference sensor.adaptive_target_humidity
  # → ENTITY NOT FOUND ERROR!
```

**Почему:**
1. Automations ссылаются на entity_id по `unique_id` в коде
2. entity_id генерируется из `name` через slugify
3. Русский `name` → транслитерация → unpredictable entity_id
4. Automation ищет sensor.adaptive_target_humidity
5. Sensor создан как sensor.adaptivnaia_tselevaia_vlazhnost
6. **→ Spook ghost warning!**

---

## 3. BINARY SENSORS

### Kvazis Pattern

```yaml
binary_sensor:
  - platform: template
    sensors:
      notification_time:
        unique_id: notification_time
        friendly_name: "Уведомления"
        value_template: >
          {{ template }}
```

### Entity ID Generation

```
sensor_name → entity_id через slugify
```

### Примеры

```yaml
# ✅ ПРАВИЛЬНО:
binary_sensor:
  - platform: template
    sensors:
      notification_time:
        unique_id: notification_time
        friendly_name: "Уведомления"
        # → binary_sensor.notification_time

# МОЖНО использовать русский friendly_name (это НЕ для entity_id):
binary_sensor:
  - platform: template
    sensors:
      water_tank_removed:
        unique_id: water_tank_removed
        friendly_name: "Бак воды снят"
        # → binary_sensor.water_tank_removed (entity_id from sensor_name)
```

---

## 4. INPUT HELPERS

### Input Boolean

```yaml
input_boolean:
  helper_name:
    name: "Display Name"
    initial: true|false
    icon: mdi:icon-name
```

### Input Number

```yaml
input_number:
  helper_name:
    name: "Display Name"
    min: 0
    max: 100
    step: 1
    unit_of_measurement: "%"
    icon: mdi:icon-name
```

### Entity ID Generation

```
helper_name → input_boolean.helper_name (exact)
helper_name → input_number.helper_name (exact)
```

**NO slugify applied!**

### Примеры

```yaml
# ✅ ПРАВИЛЬНО:
input_boolean:
  humidity_auto_enabled:
    name: "Автоматизация влажности"
    # → input_boolean.humidity_auto_enabled

input_number:
  target_humidity:
    name: "Целевая влажность"
    min: 30
    max: 60
    # → input_number.target_humidity

# ❌ НЕПРАВИЛЬНО:
input_boolean:
  humidity auto enabled:  # ← spaces not allowed
  AutoEnabled:  # ← camelCase not recommended
```

---

## 5. SCRIPTS

### Format

```yaml
script:
  script_name:
    alias: "Display Name"
    description: "Optional description"
    sequence:
      - service: ...
```

### Entity ID Generation

```
script_name → script.script_name (exact)
```

### Примеры

```yaml
# ✅ ПРАВИЛЬНО:
script:
  notify_telegram:
    alias: "Уведомление в Telegram"
    # → script.notify_telegram

  turn_on_heating:
    alias: "Включить отопление"
    # → script.turn_on_heating
```

---

## 6. SCENES

### Format

```yaml
scene:
  - name: scene_name
    entities:
      light.living_room: 'on'
```

### Entity ID Generation

```
name → scene.{slugified_name}
```

---

## 📝 SUMMARY TABLE

| Entity Type | entity_id Source | Slugify? | Language Rule |
|-------------|------------------|----------|---------------|
| Automation | `alias` | YES | English |
| Template Sensor (modern) | `name` | YES | **English** |
| Template Sensor (legacy) | `sensor_name` | YES | **English** |
| Binary Sensor | `sensor_name` | YES | English |
| Input Boolean | `helper_name` | NO | English |
| Input Number | `helper_name` | NO | English |
| Script | `script_name` | NO | English |
| Scene | `name` | YES | English |

---

## 🚨 CRITICAL RULES

### 1. Template Sensors in Packages

**ТОЛЬКО LEGACY FORMAT:**

```yaml
# ✅ packages/humidity/sensors.yaml
sensor:
  - platform: template
    sensors:
      adaptive_target_humidity:
        friendly_name: "Adaptive Target Humidity"
        value_template: >
          {{ states('sensor.temperature') }}

# ❌ НЕ РАБОТАЕТ В PACKAGES:
template:
  - sensor:
      - name: "Adaptive Target Humidity"
```

**Причина:** Modern format `template:` cannot be merged in packages (GitHub issue #50157).

### 2. entity_id = English Names

**Template sensors:**
- `name` (modern) или `sensor_name` (legacy) → entity_id
- **ВСЕГДА используй английские имена!**

```yaml
# ✅ ПРАВИЛЬНО:
- name: "Recommended Intensity"
  # → sensor.recommended_intensity

# ❌ НЕПРАВИЛЬНО:
- name: "Рекомендуемая интенсивность"
  # → sensor.rekomenduemaia_intensivnost (UNPREDICTABLE!)
```

### 3. unique_id vs entity_id

- `unique_id` - для persistence в entity registry
- `name` / `sensor_name` / `alias` - для генерации entity_id
- **НЕ ПУТАТЬ!**

```yaml
# ✅ ПРАВИЛЬНО:
- name: "Adaptive Target Humidity"  # → entity_id
  unique_id: adaptive_target_humidity  # → persistence

# ❌ НЕПРАВИЛЬНО (но работает):
- name: adaptive_target_humidity  # ← not user-friendly in UI
  unique_id: adaptive_target_humidity
```

### 4. Automations: id vs alias

- `id` - для UI (русский OK)
- `alias` - для entity_id (английский ОБЯЗАТЕЛЬНО)

```yaml
# ✅ ПРАВИЛЬНО:
- id: "Инициализация увлажнителя"
  alias: humidity_device_setup

# ❌ НЕПРАВИЛЬНО:
- id: humidity_device_setup  # ← not user-friendly
  alias: "Инициализация увлажнителя"  # ← will slugify to Cyrillic!
```

---

## 🔍 DEBUGGING entity_id

### Check actual entity_id via API:

```bash
curl -s -H "Authorization: Bearer TOKEN" \
  "https://hass.zapbrannigan.org/api/states/sensor.recommended_intensity" | \
  jq '{entity_id, state, friendly_name: .attributes.friendly_name}'
```

### If entity not found:

1. **List all sensors:**
```bash
curl -s -H "Authorization: Bearer TOKEN" \
  "https://hass.zapbrannigan.org/api/states" | \
  jq '[.[] | select(.entity_id | startswith("sensor."))] | .[].entity_id' | \
  grep -i humidity
```

2. **Check for Cyrillic slugify:**
```bash
# If name was Russian, entity_id will be transliterated:
sensor.rekomenduemaia_intensivnost  # ← from "Рекомендуемая интенсивность"
```

3. **Fix:** Change `name` to English, reload YAML.

---

## ✅ VERIFIED WORKING EXAMPLES

From production (`packages/humidity/humidity_control.yaml`):

```yaml
# Input Helper
input_boolean:
  humidity_auto_enabled:
    name: "Автоматизация влажности"
    # → input_boolean.humidity_auto_enabled ✅

# Template Sensors (legacy format for packages)
template:
  - sensor:
      - name: "Adaptive Target Humidity"
        unique_id: adaptive_target_humidity
        # → sensor.adaptive_target_humidity ✅

      - name: "Humidity Error"
        unique_id: humidity_error
        # → sensor.humidity_error ✅

      - name: "Recommended Intensity"
        unique_id: recommended_intensity
        # → sensor.recommended_intensity ✅

# Automations
automation:
  - id: "Инициализация увлажнителя"
    alias: humidity_device_setup
    # → automation.humidity_device_setup ✅

  - id: "Управление интенсивностью увлажнителя"
    alias: humidity_control_main
    # → automation.humidity_control_main ✅
```

---

**Last Updated:** 2025-11-22
**Version:** 1.0
**Tested with:** Home Assistant 2025.11.3
