# Humidity Control Automation - Changelog

История изменений автоматизации управления увлажнителем.

**Repository:** https://github.com/zappbrannigan34/home-assistant.git
**File:** `packages/humidity/humidity_control.yaml`

---

## v8 - FIX INCREMENTAL RUNAWAY (2025-11-22)

**Commit:** [pending]
**Version:** 1.1.2

### Исправления:

**Incremental Logic Runaway Bug:**
- **Проблема:** Диапазон 0.5-2% использовал `current + 1` на КАЖДОМ цикле (5 мин)
- За 6 циклов достигал max=7 даже при error=1.5%!
- Intensity НЕ снижалась когда error падал с 2% к 0.5%
- **Root cause:** Incremental control применялся ВНЕ deadband (должен быть ТОЛЬКО внутри)

**Старая логика (баг):**
```yaml
error > 2   → 3 (zoned)
error > 0.5 → current + 1 (incremental) ← RUNAWAY!
-0.5 to 0.5 → maintain (deadband)
```

**Новая логика (fix):**
```yaml
error > 2   → 3 (zoned)
error > 1   → 2 (zoned, NEW)
error > 0.5 → 1 (zoned, NEW)
-0.5 to 0.5 → maintain (deadband)
error < -0.5 → 0 (turn off)
error < -2  → current - 1 (incremental decrement)
```

**Rationale:**
- Диапазон 0.5-2% - это ВСЁ ЕЩЁ средняя ошибка, НЕ "точная настройка"
- Incremental control нужен ТОЛЬКО для fine-tuning в deadband или при переувлажнении
- Zoned approach для 0.5-2% даёт предсказуемое поведение

### Production Testing (2025-11-22 12:49-12:58 UTC):

**Scenario 1: Small error (0.5-1%):**
- ✅ BEFORE: error=1.65% → recommended=7 (WRONG, runaway)
- ✅ AFTER: error=0.5% → recommended=1 (CORRECT)
- ✅ Intensity НЕ растёт, остаётся на 1

**Scenario 2: Approach to target:**
- ✅ Error: 0.5% → 0.2% → 0.1% → 0.0%
- ✅ Recommended: стабильно 1 (НЕТ runaway)
- ✅ Current: 39.84% → 40.25% (плавное приближение к target 40.3%)

**Scenario 3: Overshoot (переувлажнение):**
- ✅ Error: -0.1% → -0.4% → -0.5% → -0.6%
- ✅ Recommended: 1 → 0 (при error < -0.5)
- ✅ Power: ON → OFF (12:55:48)
- ✅ Power ОСТАЁТСЯ OFF (v7 fix сохранён, нет infinite loop)

### Files Changed:
- `packages/humidity/humidity_control.yaml` (lines 4, 137-158)

### Проблема v7:
- Incremental runaway при error 0.5-2% (не было обнаружено до production testing)
- Исправлено в v8

---

## v7 - ФИНАЛЬНОЕ РЕШЕНИЕ (2025-11-22)

**Commit:** 33fd799

### Исправления:

**1. Logic Deadlock в control_main:**
- **Проблема:** ПРОВЕРКА ГОТОВНОСТИ была ПЕРЕД УПРАВЛЕНИЕМ ПИТАНИЕМ
- Если power=off, automation останавливалась → УПРАВЛЕНИЕ ПИТАНИЕМ не выполнялось
- **Fix:** УПРАВЛЕНИЕ ПИТАНИЕМ перемещено ПЕРЕД ПРОВЕРКОЙ ГОТОВНОСТИ
- Теперь automation сначала решает включить/выключить power, затем проверяет готовность

**2. device_setup Infinite Loop:**
- **Проблема:** device_setup триггерился когда `power → off for 5 sec`
- Всегда включал power обратно, даже когда `recommended=0`
- Создавал цикл: control_main выключает → device_setup включает → control_main выключает...
- **Fix:** Добавлена condition `recommended > 0` в начало action
- Теперь device_setup останавливается если recommended=0, НЕ трогает power

### Тесты:
- ✅ При error=-1.9% power выключается И ОСТАЁТСЯ OFF
- ✅ device_setup не перезапускает power когда recommended=0
- ✅ Влажность снижается (error: -1.9% → -0.9%)

---

## v6 (corrected) - Fix elif order (2025-11-22)

**Commit:** 6959277

### Исправления:

**recommended_intensity elif order bug:**
- **Проблема:** `elif error < -0.5` было ПЕРЕД `elif error < -2`
- Condition `error < -2` была UNREACHABLE (dead code)
- **Fix:** Поменял порядок - сначала `error < -2`, потом `error < -0.5`

**Правильная логика:**
```yaml
elif error > 0.5:     # → increment
elif error < -2:      # → decrement (check larger threshold first!)
elif error < -0.5:    # → 0 (then smaller threshold)
else:                 # → maintain
```

### Тесты:
- ✅ При error=-1.9% recommended корректно становится 0

### Проблема v6:
- Обнаружен infinite loop: control_main выключает power → device_setup включает обратно
- Исправлено в v7

---

## v6 - Power Management (2025-11-22)

**Commit:** e42bb9a

### Новая функциональность:

**1. Power management в control_main:**
- Добавлена секция "УПРАВЛЕНИЕ ПИТАНИЕМ"
- Если `recommended=0` И `power=on` → выключить power
- Если `recommended>0` И `power=off` → включить power

**2. Conditional power в device_setup:**
- device_setup теперь проверяет recommended перед включением
- Если `recommended>0` → включить power
- Иначе → выключить power

**3. Исправлен порядок elif в recommended_intensity:**
- Логика: error<-2 → decrement, -2<error<-0.5 → 0, -0.5<error<0.5 → maintain

### Тесты:
- ✅ При error=-1.9% питание автоматически выключается

### Проблема v5:
- При error=-1.9% увлажнитель оставался включённым (питание ON, intensity 0)
- Исправлено в v6

---

## v5 - Detector Protection для control_main (2025-11-22)

**Commit:** dc269bb

### Исправления:

**control_main MQTT echo protection:**
- **Проблема:** control_main вызывает service calls → MQTT updates → detector triggers!
- **Fix:** control_main теперь временно выключает detector:
  1. `automation.turn_off` detector
  2. `number.set_value` (change intensity)
  3. `delay: 20 sec` (wait for MQTT)
  4. `automation.turn_on` detector

### Тесты:
- ✅ Helper остаётся ON 3+ минуты ДАЖЕ когда control_main срабатывает!

### Проблема v4:
- device_setup была защищена, но control_main НЕ была защищена
- Исправлено в v5

---

## v4 - Увеличен delay до 20 sec (2025-11-22)

**Commit:** 90c7208

### Исправления:

**device_setup delay increase:**
- **Проблема:** MQTT updates приходили до 14+ секунд
- Delay 5 секунд был недостаточен
- **Fix:** Увеличен delay с 5 до 20 секунд

### Проблема v3:
- Delay 5 sec был недостаточен для MQTT updates
- Частично работало, но control_main не была защищена

---

## v3 - device_setup detector protection (2025-11-22)

**Commit:** eb2c61e

### Исправления:

**device_setup MQTT echo protection:**
- **Проблема:** device_setup меняет intensity → MQTT echoes → detector triggers
- **Fix:** device_setup теперь временно выключает detector:
  1. `automation.turn_off` detector в начале
  2. All service calls
  3. `delay: 5 sec`
  4. `automation.turn_on` detector в конце

### Проблема v2:
- detector всё равно срабатывал на MQTT echoes
- Delay 5 sec был недостаточен

---

## v2 - Detector logic fixes (2025-11-22)

### Исправления:

**manual_control_detector:**
- Убран trigger на power
- Добавлен `for: 2 sec`
- Изменены conditions (проверка `current` вместо `parent_id`)

**reload_recovery:**
- Убрано condition `helper="on"`
- Добавлено включение helper в action

### Проблема v1:
- detector всё равно срабатывал на MQTT echoes

---

## v1 - KVAZIS Pattern (2025-11-22)

### Изменения:

**Automation naming:**
- Применён kvazis pattern для всех 5 automations:
  - `id`: Описательное имя для ЛЮДЕЙ (русский язык)
  - `alias`: Технический код для entity_id (английский, [a-z0-9_])

**Примеры:**
```yaml
- id: "Инициализация увлажнителя"
  alias: humidity_device_setup
  # → automation.humidity_device_setup

- id: "Управление интенсивностью увлажнителя"
  alias: humidity_control_main
  # → automation.humidity_control_main
```

---

## Initial Implementation (2025-01-22)

Создан файл `AUTOMATION_NAMING.md` после ЧЕТВЁРТОГО раза забывания правил именования automations 🤦

**Документированы:**
- entity_id generation rules (slugify)
- Доступные поля (id, alias, description)
- kvazis pattern best practices
- Проверка entity_id через API
