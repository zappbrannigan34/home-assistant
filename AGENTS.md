# AGENTS.md — Home Assistant Automations

## Project Overview

YAML-пакеты автоматизаций для Home Assistant. Каждый пакет — изолированная автоматизация, подключаемая через `homeassistant: packages`.

**Repo:** `zappbrannigan34/home-assistant`
**Owner language:** Russian (comments in YAML, docs bilingual). Entity names — English only.

---

## Server Access

Credentials хранятся в **`.secrets.env`** (gitignored, никогда не коммитить).

```bash
# Загрузить переменные
source .secrets.env

# Проверить API
curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/ | jq

# Проверить entity
curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/states/sensor.adaptive_target_humidity | jq '{entity_id, state, last_changed}'
```

### Файловый доступ к серверу

HA config directory доступен через SMB (сетевой диск):
- **Windows:** диск `Z:\` (mapped) или `\\<SMB_HOST>` (creds в `.secrets.env`)
- **Путь к конфигу:** `Z:\` → `configuration.yaml`, `packages/`, `.storage/` и т.д. (Z: = `/config/` напрямую, без подпапки config)

Для деплоя YAML-пакетов — копировать файлы в `Z:\packages\`.

---

## Repository Structure

```
home-assistant/
├── AGENTS.md                        # ← этот файл
├── .secrets.env                     # Токены, пароли (GITIGNORED)
├── .gitignore
├── README.md
├── dashboards/
│   ├── humidity_dashboard.yaml      # Plotly-дашборд увлажнителя
│   └── ventilation_dashboard.yaml   # Plotly-дашборд вентиляции (CO2 + окно)
├── docs/
│   ├── NAMING_CONVENTIONS.md        # Правила именования сущностей (ОБЯЗАТЕЛЬНО читать)
│   ├── LESSONS_LEARNED.md           # Ошибки и как их избегать
│   ├── DEBUGGING_HA.md              # Команды для отладки через API
│   ├── TROUBLESHOOTING_TEMPLATE_SENSORS.md
│   └── HUMIDITY_CHANGELOG.md        # История изменений пакета humidity
└── packages/
    ├── humidity/
    │   ├── humidity_control.yaml    # Основной YAML-пакет (v3.1.0)
    │   └── README.md               # Документация пакета
    └── ventilation/
        ├── ventilation_control.yaml # Trend-based CO2 контроль окна (v1.0.0)
        └── README.md               # Документация пакета
```

---

## Zigbee & Device Infrastructure

### Zigbee2MQTT Bridge

Z2M v2.9.1, подключен через MQTT. Управляющие entity:

| Entity | Purpose |
|--------|---------|
| `binary_sensor.zigbee2mqtt_bridge_connection_state` | Статус подключения (on = ОК) |
| `sensor.zigbee2mqtt_bridge_version` | Версия Z2M |
| `switch.zigbee2mqtt_bridge_permit_join` | Разрешить присоединение новых устройств |
| `button.zigbee2mqtt_bridge_restart` | Рестарт Z2M |
| `select.zigbee2mqtt_bridge_log_level` | Уровень логирования (info) |
| `update.zigbee2mqtt_update` | Обновление Z2M |

### Rooms / Zones

В доме минимум 3 зоны — `zap`, `eva`, `masyanya`. Entity именуются по зоне:

| Зона | Мультисенсор | Drivent привод | Радиатор |
|------|-------------|----------------|----------|
| **zap** | `sensor_zap` (Wirenboard) | `drivent_zap_left` | `radiator_left_zap` |
| **eva** | `sensor_eva` (Wirenboard) | `drivent_eva` | ? |
| **masyanya** | ? | `drivent_masyanya` | ? |

### Wirenboard Wall-mounted Multi Sensor

Модель: **Wirenboard Wall-mounted multi sensor** (Zigbee). Две штуки: `sensor_zap` и `sensor_eva`.

| Zigbee ID | HA device name | Device ID (registry) |
|-----------|---------------|---------------------|
| `0x04cd15fffe9eaed2` | sensor-zap | `16f353bff4963968806804138ea07e6e` |
| `0x04cd15fffea0bc14` | sensor-eva | `c0ede7f9524fc9261e61577387af7a15` |

**Entity на каждый сенсор** (замени `{name}` на `sensor_zap` или `sensor_eva`):

| Entity | Type | Purpose |
|--------|------|---------|
| `sensor.{name}_temperature` | Sensor | Температура |
| `sensor.{name}_humidity` | Sensor | Влажность |
| `sensor.{name}_co2` | Sensor | CO2 (ppm) |
| `sensor.{name}_voc` | Sensor | VOC |
| `sensor.{name}_noise` | Sensor | Уровень шума |
| `sensor.{name}_illuminance` | Sensor | Освещённость |
| `sensor.{name}_occupancy_level` | Sensor | Уровень присутствия |
| `binary_sensor.{name}_occupancy` | Binary | Присутствие |
| `binary_sensor.{name}_noise_detected` | Binary | Обнаружен шум |
| `switch.{name}_l1` | Switch | Выход L1 |
| `switch.{name}_l2` | Switch | Выход L2 |
| `switch.{name}_l3` | Switch | Выход L3 |
| `number.{name}_noise_timeout` | Number | Таймаут шума |
| `number.{name}_occupancy_timeout` | Number | Таймаут присутствия |
| `number.{name}_temperature_offset` | Number | Коррекция температуры |
| `number.{name}_occupancy_sensitivity` | Number | Чувствительность присутствия |
| `number.{name}_noise_detect_level` | Number | Порог детекции шума |
| `select.{name}_co2_autocalibration` | Select | Автокалибровка CO2 |
| `select.{name}_co2_manual_calibration` | Select | Ручная калибровка CO2 |
| `select.{name}_th_heater` | Select | Обогреватель (антиконденсат?) |
| `update.{name}` | Update | Обновление прошивки |

### Drivent Covers (все 3 привода)

| Entity | Зона | Status |
|--------|------|--------|
| `cover.drivent_zap_left_driventokonnyi_privod` | zap | Используется ventilation package |
| `cover.drivent_eva_driventokonnyi_privod` | eva | Не автоматизирован |
| `cover.drivent_masyanya_driventokonnyi_privod` | masyanya | Не автоматизирован |

Дополнительные entity для `drivent_zap_left`:
- `binary_sensor.drivent_zap_left_datchik_peregruzki` — перегрузка
- `sensor.drivent_zap_left_datchik_dozhdia` — дождь (%)
- `switch.drivent_zap_left_raspisanie` — встроенное расписание (off)

### Радиатор

| Entity | Purpose |
|--------|---------|
| `switch.radiator_left_zap_open_window` | Режим открытого окна (вкл/выкл) |
| `number.radiator_left_zap_open_window_temperature` | Температура в режиме открытого окна (°C) |

---

## Server Configuration (не в репозитории)

`Z:\configuration.yaml` — **только на сервере**, не трекается в git.

### Ключевые секции

```yaml
homeassistant:
  packages: !include_dir_named packages    # ← packages/*.yaml

lovelace:
  mode: yaml                               # ← дашборды из YAML файлов
  resources:                               # ← HACS frontend компоненты
    - plotly-graph-card
    - layout-card
    - kiosk-mode
    - auto-reload-card
    - card-mod
  dashboards:
    humidity-control:
      filename: dashboards/humidity_dashboard.yaml
    ventilation-control:
      filename: dashboards/ventilation_dashboard.yaml

logger:
  default: warning
  logs:
    custom_components.localtuya: error
```

### Server Storage (прямой доступ к registry)

REST API `api/config/entity_registry` и `api/error_log` возвращают **404**. Данные читать напрямую из файлов:

| Файл | Содержимое |
|------|-----------|
| `Z:\.storage\core.entity_registry` | Все entity: platform, device_id, unique_id, disabled_by |
| `Z:\.storage\core.device_registry` | Все устройства: manufacturer, model, identifiers, connections |
| `Z:\.storage\core.config_entries` | Config entries (интеграции) |

Формат: JSON, данные в `data.entities[]` / `data.devices[]`.

```python
# Пример чтения entity registry
import json
with open(r'Z:\.storage\core.entity_registry', 'r') as f:
    data = json.load(f)
for e in data['data']['entities']:
    if e['entity_id'] == 'sensor.sensor_zap_co2':
        print(e['platform'], e['device_id'], e['unique_id'])
```

---

## Known Issues

### Wirenboard сенсоры офлайн (обнаружено 2026-03-25)

**Проблема:** Оба Wirenboard мультисенсора (`sensor_zap`, `sensor_eva`) — все entity unavailable с 2026-03-24T22:04:33.

**Диагностика:**
- Z2M bridge online, connection_state: on
- Drivent приводы (тоже Zigbee) работают нормально → Zigbee сеть жива
- Проблема изолирована на два одинаковых устройства Wirenboard
- Все 21 entity каждого сенсора отвалились одновременно → device-level issue

**Возможные причины:**
1. Прошивка зависла — нужен power cycle (снять питание на 10 сек)
2. Zigbee роутер, через который они работали, отвалился
3. OTA обновление сломало оба (одинаковая модель)

**Влияние:**
- Пакет `ventilation` не работает (нет CO2 данных)
- Пакет `humidity` не работает (нет temp/humidity с sensor_eva)
- Template sensors каскадно unavailable

**TODO:** Восстановить сенсоры (power cycle / re-pair). После восстановления проверить что automation пакеты подхватят данные автоматически.

---

## Existing Package: `humidity` (v3.1.0)

**Device:** Polaris PUH-9105 / PUH-2709 (MQTT via `polaris-mqtt` integration)

PD-регулятор с предиктивным демпфированием для управления увлажнителем. Включает:

- Адаптивную целевую влажность (учитывает T комнаты и T улицы)
- PD-контроллер с прогнозом ошибки на 7 мин
- Виртуальный датчик уровня воды с автокалибровкой per-level
- Автоматическое управление питанием, режимом, интенсивностью
- Уведомления об ошибках (notify.zapgroup)

### Key Entities

| Entity | Type | Purpose |
|--------|------|---------|
| `sensor.sensor_eva_humidity` | Source | Текущая влажность (датчик Eva) |
| `sensor.sensor_eva_temperature` | Source | Температура в комнате |
| `weather.forecast_home_assistant` | Source | Погода (attr: temperature) |
| `sensor.adaptive_target_humidity` | Template | Адаптивная целевая влажность |
| `sensor.humidity_error` | Template | Ошибка: target − current |
| `sensor.humidity_rate` | Template | Скорость изменения %/мин |
| `sensor.recommended_intensity` | Template | PD-регулятор → 0-7 |
| `sensor.humidifier_water_level` | Template | Виртуальный уровень воды % |
| `number.humidifier_puh_9105_puh_2709_intensity` | Device | Интенсивность 1-7 |
| `switch.humidifier_puh_9105_puh_2709_power` | Device | Питание вкл/выкл |
| `humidifier.humidifier_puh_9105_puh_2709_humidifier` | Device | Режим работы |

### Automations

| Automation | Purpose |
|------------|---------|
| `humidity_device_setup` | Инициализация при старте (power, mode, switches) |
| `humidity_mode_enforcer` | Гарантирует режим `home` каждые 2 мин |
| `humidity_control_main` | Основная логика: intensity + power |
| `humidifier_calibrate_rates` | Калибровка расхода при low_water |
| `humidity_device_error` | Уведомления об ошибках |

---

## Existing Package: `ventilation` (v1.0.0)

**Device:** Drivent V2 (WindowMaster chain actuator) — `cover.drivent_zap_left_driventokonnyi_privod`

Trend-based CO2 ventilation control. НЕ PID — зонный алгоритм, аналогичный Drivent Air. Привод двигается только когда тренд CO2 подтверждает необходимость. Оценка каждые 5 минут, шаг 2-5%, deadband ±50 ppm.

**Ресурс привода:** WindowMaster WMX 803 рейтинг ~10 000 циклов. С тренд-фильтром: ~20-40 движений/день → 1.5-3 года.

Включает:
- Кольцевой буфер CO2 (5 значений, замер/мин) → тренд rising/falling/stable
- Зонное управление (5 зон относительно target CO2)
- Антиосцилляцию (4-step sign change buffer)
- Аварийное закрытие (холод, перегрузка, потеря CO2 сенсора)
- Координацию с радиатором (open window mode)
- Уведомления (notify.zapgroup)

### Key Entities

| Entity | Type | Purpose |
|--------|------|---------|
| `sensor.sensor_zap_co2` | Source | CO2 в комнате (ppm) |
| `cover.drivent_zap_left_driventokonnyi_privod` | Device | Оконный привод (position 0-100%) |
| `binary_sensor.drivent_zap_left_datchik_peregruzki` | Device | Датчик перегрузки привода |
| `sensor.drivent_zap_left_datchik_dozhdia` | Device | Датчик дождя (%) |
| `weather.forecast_home_assistant` | Source | Погода (attr: temperature) |
| `switch.radiator_left_zap_open_window` | Device | Режим открытого окна на радиаторе |
| `sensor.co2_trend` | Template | Тренд CO2: rising/falling/stable |
| `sensor.ventilation_recommended_position` | Template | Рекомендуемая позиция окна (%) |
| `sensor.ventilation_co2_error` | Template | Ошибка: CO2 − target (ppm) |
| `input_number.ventilation_target_co2` | Input | Целевой CO2 (default 650 ppm) |
| `input_number.ventilation_min_position` | Input | Мин. позиция окна (default 0%) |
| `input_number.ventilation_max_position` | Input | Макс. позиция окна (default 80%) |
| `input_number.ventilation_min_outdoor_temp` | Input | Мин. температура улицы (default -10°C) |

### Automations

| Automation | Purpose |
|------------|---------|
| `ventilation_control_main` | Применяет рекомендуемую позицию к приводу (rate limit 4.5 мин) |
| `ventilation_safety_close` | Аварийное закрытие: холод, перегрузка, потеря CO2 |
| `ventilation_radiator_sync` | Синхронизация радиатора: open window mode при >5%, выкл при <3% |
| `ventilation_device_error` | Уведомление о перегрузке привода |

### Algorithm Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Trend threshold | ±15 ppm | Разница текущего CO2 vs среднее 3 старейших в буфере |
| Evaluation interval | 5 min | Частота пересчёта рекомендуемой позиции |
| Steps | +5%, +2%, -2%, -5% | Зависит от зоны CO2 и тренда |
| Deadband | ±50 ppm | Зона вокруг target где движения минимальны |
| Rate limit | 1 hour | Минимальный интервал между движениями привода |
| Anti-oscillation | 4-step buffer | Блокирует движение при ≥2 смен знака за последние 4 шага |

---

## Conventions (CRITICAL — read before any changes)

### Entity Naming

Полная документация: `docs/NAMING_CONVENTIONS.md`

**Ключевые правила:**

1. **`name` для template sensors — ТОЛЬКО на английском** (slugify генерирует entity_id)
2. **`alias` для automations — на английском**, `id` — на русском для UI
3. **`unique_id`** обязателен для всех template sensors
4. Modern `template:` формат (не legacy `sensor: - platform: template`)

```yaml
# ПРАВИЛЬНО:
template:
  - trigger:
      - platform: state
        entity_id: sensor.source
      - platform: time_pattern
        minutes: "/5"
    sensor:
      - name: "My Sensor Name"          # English! → sensor.my_sensor_name
        unique_id: my_sensor_name
        availability: >
          {{ has_value('sensor.source') }}
        state: >
          {{ states('sensor.source') | float(0) }}

automation:
  - id: "Описание для UI на русском"
    alias: my_automation_name             # English! → automation.my_automation_name
```

### Template Sensors — Trigger-Based ONLY

**В packages template sensors ОБЯЗАТЕЛЬНО используют trigger-based формат.** Без `trigger:` блока `states()` не создаёт entity tracking → сенсор застревает на `unavailable`.

**Обязательные элементы:**
- `trigger:` с `platform: state` для каждого source entity
- `platform: time_pattern` как fallback
- `availability:` с `has_value()` для предотвращения каскадных unavailable

### Reload vs Restart

| Изменение | Что делать |
|-----------|-----------|
| Логика template (формула) | `template.reload` или `reload_all` |
| Новый template sensor | **RESTART** (reload не создаст entity) |
| Структурные изменения YAML | **RESTART** |
| Условия/действия automation | `automation.reload` |

### Comments Language

- YAML comments: русский
- Entity names, aliases: English
- Documentation: bilingual (русский + English)

---

## Deploy Workflow

1. Отредактировать YAML в репозитории
2. Валидировать синтаксис (HA configuration check)
3. Скопировать на сервер: `Z:\packages\<package_name>\`
4. Для дашбордов: скопировать в `Z:\dashboards\`
5. **Регистрация нового дашборда:** добавить запись в `Z:\configuration.yaml` → `lovelace: dashboards:` (файл не в репозитории, только на сервере). Формат:
   ```yaml
   lovelace:
     dashboards:
       <slug>:
         mode: yaml
         title: <Title>
         icon: mdi:<icon>
         show_in_sidebar: true
         filename: dashboards/<filename>.yaml
   ```
6. Применить:
   - Изменения логики → `reload_all` через API
   - Новые entities / структурные изменения → **restart**
   - Новый дашборд / изменение lovelace config → **restart**

```bash
source .secrets.env

# Reload (для изменений логики)
curl -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" -d '{}' $HA_URL/api/services/homeassistant/reload_all

# Restart (для новых entities)
curl -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" -d '{}' $HA_URL/api/services/homeassistant/restart
```

---

## Validation & Debugging

### Перед коммитом

1. **Проверить все entity_id** — что referenced entities существуют на сервере:
   ```bash
   source .secrets.env
   curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/states/sensor.MY_ENTITY | jq '{entity_id, state}'
   ```

2. **Не коммитить без проверки** — особенно переименования entity_id

3. **Не менять entity_id source-сенсоров** (типа `sensor.sensor_eva_*`) без явного разрешения владельца

### При проблемах

Порядок документации:
1. `docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md` — если сенсор unavailable
2. `docs/DEBUGGING_HA.md` — команды для исследования через API
3. `docs/LESSONS_LEARNED.md` — типичные ошибки и их причины

---

## Ideas for New Packages

Все новые пакеты создаются в `packages/<name>/` с README и YAML-файлом(ами).

Возможные направления:

- **Климат-контроль**: управление кондиционером/обогревателем на основе температуры, расписания, присутствия
- **Освещение**: адаптивная яркость/цветовая температура по времени суток, освещённости, присутствию
- **Присутствие**: детекция присутствия/отсутствия (BLE, Wi-Fi, motion sensors) → общий binary_sensor для других пакетов
- **Энергомониторинг**: трекинг потребления устройств, дневные/месячные отчёты, алерты на аномалии
- **Уведомления**: централизованная система нотификаций (Telegram, push) с приоритетами, тротлингом, группировкой
- **Безопасность**: мониторинг протечек, дыма, открытия дверей/окон → каскадные действия и алерты
- **Медиа**: управление колонками/ТВ по сценариям (утро, вечер, гости)
- **Растения**: мониторинг влажности почвы, освещённости → автополив, алерты
- **Бытовая техника**: трекинг циклов стиральной/посудомоечной машины, уведомления о завершении
- **Сон**: ночной режим (свет, звук, температура, увлажнитель) по расписанию или ручному триггеру

### Шаблон нового пакета

```
packages/<name>/
├── <name>_control.yaml   # Основной YAML-пакет
└── README.md             # Документация: зачем, как работает, entities, установка
```

Обновить таблицу пакетов в корневом `README.md` после добавления.

---

## Authorization Protocol

**Деструктивные операции** (переименование entity_id, удаление entities, изменение source-сенсоров, bulk-замены) требуют явного разрешения владельца.

Перед выполнением:
1. Показать план с конкретными изменениями
2. Объяснить влияние (что сломается если что-то пойдёт не так)
3. **Дождаться явного "OK" / "да" / "делай"**
4. Не предполагать намерение — спрашивать

> Пример прошлой ошибки: замена `sensor.sensor_eva_*` → `sensor.sensor_zap_*` без проверки сломала все автоматизации. Это были РАЗНЫЕ физические датчики в разных комнатах. См. `docs/LESSONS_LEARNED.md § 2`.

---

## Research Protocol

При исследовании проблем — **сначала проверь гипотезу, потом ресёрч**.

### Порядок действий
1. **Воспроизвести проблему** — убедиться что она существует
2. **Проверить гипотезу** — тест на сервере через API
3. **Глубокий ресёрч** — минимум 10 источников:
   - Official HA docs (2-3)
   - GitHub issues open + closed (3-4)
   - Community forums — HA Community, Reddit (2-3)
   - Expert repositories — **kvazis** (Russian HA expert), HACS (2-3)
4. **Кросс-валидация** — результаты должны подтверждаться из нескольких источников

### Reference Repositories
- **kvazis/newHA** — gold standard для русскоязычных HA-паттернов: https://github.com/kvazis/newHA
- **HA REST API docs** — https://developers.home-assistant.io/docs/api/rest/
- **Template integration** — https://www.home-assistant.io/integrations/template/

---

## Debugging Quick Reference

```bash
source .secrets.env

# Проверить entity
curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/states/sensor.ENTITY | jq '{entity_id, state, last_changed}'

# Найти все entity по паттерну
curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/states | jq '.[] | select(.entity_id | test("humidity")) | {entity_id, state}'

# Найти unavailable entities
curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/states | jq '.[] | select(.state == "unavailable") | .entity_id'

# Entity registry (platform, created_at)
curl -s -H "Authorization: Bearer $HA_TOKEN" $HA_URL/api/config/entity_registry | jq '.[] | select(.entity_id == "sensor.ENTITY")'

# Reload all
curl -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" -d '{}' $HA_URL/api/services/homeassistant/reload_all

# Restart
curl -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" -d '{}' $HA_URL/api/services/homeassistant/restart
```

Подробнее: `docs/DEBUGGING_HA.md`

---

## Commit Message Format

Формат: `<type>: <short summary>` (max 72 chars)

**Types:** `fix`, `feat`, `refactor`, `docs`, `style`, `chore`, `revert`

```
feat: add bedroom climate control automation

Adds adaptive climate control based on temperature, time of day,
outdoor temperature, and manual override detection.

Files:
- packages/climate/bedroom_climate.yaml (NEW)

Verified:
- automation triggers on temperature changes
- manual override detection working
```

### Pre-Commit Checklist
- [ ] YAML syntax valid
- [ ] Entity references verified via API
- [ ] Full restart tested (if structural changes)
- [ ] CHANGELOG updated (if package modified)
- [ ] No sensitive data in commit
- [ ] Version number incremented (if applicable)

Подробнее: `.github/COMMIT_TEMPLATE.md`

---

## plotly-graph-card Reference

Карточка: `dbuezas/lovelace-plotly-graph-card` (HACS).

### $fn / $ex / vars

- `$fn` = explicit function: `$fn ({hass, vars, ys}) => expression`
- `$ex` = short form (auto-adds params): `$ex hass.states['sensor.x'].state`
- `vars` = объект для передачи данных между entity filters и layout. Устанавливается в entity filters, читается в layout.
- `hass` **ДОСТУПЕН** в layout properties (yaxis.range, shapes, title и т.д.)
- `xs`, `ys`, `states`, `statistics`, `meta` — **ТОЛЬКО** внутри entity context, НЕ в layout
- `getFromConfig("entities")` — доступ ко ВСЕМ entity данным (включая y values) из layout $fn
- **Порядок вычисления:** Entity Filters → Entity Properties → Layout → Shapes (top-to-bottom, shallow-to-deep)
- `$fn` в `yaxis.range` работает (подтверждено issues #60, #233)
- Async/promises **НЕ** поддерживаются в $fn
- Functions cannot return functions

### Axis Range Techniques

```yaml
# Фиксированный диапазон
range: [0, 100]

# Динамический $fn
range: |
  $fn ({getFromConfig}) => {
    const all = getFromConfig("entities").flatMap(({y}) => y);
    return [Math.min(...all), Math.max(...all)];
  }

# Частичный авто
range: [-5, null]    # fix min, auto max

# Bounded auto-range
autorangeoptions:
  minallowed: 5
  maxallowed: 20
```

**Симметричное выравнивание нулей на разных осях:** Все оси должны иметь **одинаковый ratio** между min и max. Пример: y1=[-5,20] (ratio -4), y2=[-0.1,0.4] (ratio -4) → нули визуально совпадают.

### ⚠️ autorange_after_scroll BUG

`autorange_after_scroll: true` **тихо перезаписывает** `$fn` в `yaxis.range` на **primary axis**. Secondary axes (yaxis2, yaxis3) НЕ затрагиваются.

**Фикс:** Установить ОБА:
```yaml
# На уровне карточки
autorange_after_scroll: false

# На уровне yaxis
layout:
  yaxis:
    autorange: false
    range: |
      $fn ({vars, hass}) => [...]
```

Без `autorange: false` + `autorange_after_scroll: false` — $fn range на primary yaxis игнорируется.

### Рабочий пример: target CO2 по центру оси

```yaml
yaxis:
  autorange: false
  range: |
    $fn ({vars, hass}) => {
      const target = parseFloat(hass.states['input_number.ventilation_target_co2']?.state) || 650;
      const cMin = vars.co2_min ?? target;
      const cMax = vars.co2_max ?? target;
      const maxDev = Math.max(Math.abs(cMax - target), Math.abs(cMin - target), 30);
      const padded = maxDev * 1.15;
      return [target - padded, target + padded];
    }
```

---

## CDP Screenshots (Chrome DevTools Protocol)

Используется для скриншотов HA дашбордов. **НЕ Playwright MCP** — прямое подключение к запущенному браузеру.

### Подключение

Chrome должен быть запущен с `--remote-debugging-port=9222`. Подключение через WebSocket.

```bash
# Получить список табов
curl -s http://localhost:9222/json | jq '.[].title'

# Найти HA таб
curl -s http://localhost:9222/json | jq '.[] | select(.url | test("hass")) | {id, title, webSocketDebuggerUrl}'
```

### Reliable Screenshot Sequence

1. **Navigate** → `Page.navigate` к нужному дашборду
2. **Wait 12s** — HA shadow DOM полностью строится долго
3. **Force layout** — `getComputedStyle()` на всех shadow hosts
4. **Viewport resize trick** — `Emulation.setDeviceMetricsOverride` toggle (разные размеры) → triggers ResizeObserver
5. **Find plotly card** — рекурсивный обход shadow DOM:
   ```javascript
   function findInShadow(root, selector) {
     let el = root.querySelector(selector);
     if (el) return el;
     for (const child of root.querySelectorAll('*')) {
       if (child.shadowRoot) {
         el = findInShadow(child.shadowRoot, selector);
         if (el) return el;
       }
     }
     return null;
   }
   ```
6. **Force render** — `card.plot({should_fetch: true})`
7. **Wait 5s** → screenshot через `Page.captureScreenshot`

### Shadow DOM Gotchas

- `document.querySelector()` НЕ проникает в shadow DOM — нужен рекурсивный traversal
- `getComputedStyle(shadowHost)` — специфично форсирует layout для shadow DOM elements
- `element.offsetHeight` / `element.offsetWidth` — форсирует reflow
- Puppeteer `>>>` selector pierces shadow DOM: `page.waitForSelector('>>> .my-class')`
- IntersectionObserver имеет known issues в headless Chrome (Chromium #384770154)

### plotly-graph-card Rendering Pipeline

- Карточка ставит `visibility: hidden` пока `Plotly.react()` не завершится
- Prerequisites: `this.config && this.hass && this.isConnected` — все 3 должны быть true
- Debounce через `requestAnimationFrame` — может не сработать в headless/CDP
- **ResizeObserver** на `ha-card` → `updateCardSize()` → `plot()`
- Прямой вызов: `card.plot({ should_fetch: true })` — самый надёжный способ

---

## Known Gotchas

### input_number `initial:` сбрасывает значение при рестарте

**Проблема:** `initial: 650` в YAML **принудительно ставит** значение при каждом рестарте HA, перезаписывая сохранённое в `.storage/core.restore_state`.

**Фикс:** Удалить ключ `initial:`. Без него HA использует последнее сохранённое значение.

```yaml
# НЕПРАВИЛЬНО — сбросится при рестарте:
input_number:
  ventilation_target_co2:
    initial: 650    # ← удалить!
    min: 400
    max: 1200

# ПРАВИЛЬНО — сохраняет значение:
input_number:
  ventilation_target_co2:
    min: 400
    max: 1200
```

### plotly-graph-card рендерит пустой/чёрный в CDP

**Проблема:** Plotly chart area чёрная/пустая при CDP скриншоте. Gauge и entity cards рендерятся нормально.

**Причина:** `requestAnimationFrame`-based debounce и IntersectionObserver могут не сработать в CDP. Shadow DOM tree строится 10-12s после navigate.

**Фикс:** Wait 12s → force layout → viewport resize → `card.plot({should_fetch: true})` → wait 5s → screenshot. См. секцию "CDP Screenshots".

### ventilation_co2_error — знак ошибки

**Формула:** `target - co2` (НЕ `co2 - target`).
- Положительное значение = CO2 ниже target (комната чистая)
- Отрицательное = CO2 выше target (нужна вентиляция)

Это **инверсия** стандартного PID error. Сделано чтобы график ошибки "разворачивался" относительно графика CO2 (не сливался).

---

## Hard Rules

1. **NEVER commit `.secrets.env`** или любые файлы с токенами/паролями
2. **NEVER rename source entity_id** (`sensor.sensor_eva_*`, device entities) без разрешения
3. **NEVER make bulk changes** без явного одобрения владельца
4. **ALWAYS use trigger-based templates** в packages
5. **ALWAYS use English** для name/alias, генерирующих entity_id
6. **ALWAYS verify entity existence** через API перед коммитом
7. **ALWAYS restart** (не reload) при добавлении новых template sensors
8. **ALWAYS verify hypothesis FIRST** — потом ресёрч, потом решение
9. **Spook warnings могут быть false positive** — всегда проверять через API
10. **kvazis patterns** — проверенный источник для HA-паттернов
11. **NEVER delete, disconnect, or modify system resources** (network drives, services, credentials, mapped drives) без явного разрешения владельца. Это включает `net use /delete`, отключение шар, удаление файлов за пределами репозитория. Инцидент: агент удалил маппинг Z: диска без спроса.
12. **NEVER accept `unavailable` / `unknown` as "expected"** — всегда диагностировать причину и сообщить владельцу. Если source entity unavailable → найти device, проверить интеграцию, Z2M bridge, дать actionable диагноз. Инцидент: агент подписал "unavailable expected ✅" вместо того чтобы выяснить что оба Wirenboard сенсора мертвы.
13. **NEVER use Playwright MCP** (`skill_mcp`) — запрещено. Для скриншотов и взаимодействия с браузером использовать **только CDP** (Chrome DevTools Protocol) через прямое подключение к запущенному браузеру пользователя (`localhost:9222`). См. секцию "CDP Screenshots".
14. **ALWAYS research BEFORE implementation** — при работе с незнакомыми библиотеками/API сначала изучить документацию, найти примеры, понять как работает. Не пытаться угадать API наугад. Инцидент: агент несколько раз подряд неправильно использовал plotly-graph-card $fn, теряя время на trial-and-error.
15. **ALWAYS use user's running browser** — не запускать headless Chrome, не создавать новые инстансы. Подключаться к уже открытому браузеру пользователя через CDP на `localhost:9222`. Таб HA уже открыт.
16. **ALWAYS write knowledge to AGENTS.md** — все найденные знания, паттерны, баги, готчи записывать в этот файл (`AGENTS.md`), а НЕ в OpenCode skill файлы (`~/.config/opencode/skills/`). Этот файл — единственный источник знаний для агентов проекта.
