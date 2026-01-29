# Humidity Control Package — Session Context

## Окружение

- **Home Assistant**: `https://hass.zapbrannigan.org/`
- **API Token**: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJhNGVjYjNiMDM0M2I0NjU2YTBlNzNmOTViNDMwZGYyYyIsImlhdCI6MTc2OTYyNzExNSwiZXhwIjoyMDg0OTg3MTE1fQ.OEIO4o7H8kQfZlCJJlj86KwLYVAVIE7YjylvXxBx65Y`
- **HA Version**: 2026.1.3
- **Конфиг**: `Z:\packages\humidity\humidity_control.yaml`
- **Дашборд**: `Z:\dashboards\humidity_dashboard.yaml`
- **Git-репо (локально)**: `C:\Users\zap\projects\home-assistant`
- **Git-репо (remote)**: `https://github.com/zappbrannigan34/home-assistant.git` (branch: `main`)
- **Сетевая шара**: `\\192.168.88.42` → `Z:\`
- **Связь шара ↔ репо**: конфиг и дашборд лежат на шаре `Z:\`, репо — отдельная копия в `C:\Users\zap\projects\home-assistant`. Файлы нужно копировать вручную или синхронизировать. Дашборд добавлен в репу в `dashboards/`.

## Устройство

- **Модель**: Polaris PUH 9105 IQ Home
- **Бак**: 5 литров (5000 мл) — фиксированная ёмкость
- **Макс расход**: ~500 мл/час (на уровне 7, калибруется)
- **Интенсивность**: 0–7 (7 уровней)
- **Режим**: home (принудительно через mode_enforcer)

## Ключевые сущности

| Entity | Описание |
|--------|----------|
| `sensor.sensor_eva_humidity` | Физический датчик влажности |
| `sensor.sensor_eva_temperature` | Физический датчик температуры |
| `sensor.recommended_intensity` | PD-регулятор: рекомендуемая интенсивность |
| `sensor.humidity_error` | Ошибка регулирования (target - current) |
| `sensor.humidity_rate` | Скорость изменения влажности (%/мин) |
| `sensor.adaptive_target_humidity` | Адаптивная целевая влажность |
| `sensor.humidifier_water_consumed` | Виртуальный: накопленный расход воды (мл) |
| `sensor.humidifier_water_level` | Виртуальный: уровень воды в баке (%) |
| `sensor.humidifier_level_N_hours` | History Stats: часы работы на уровне N (1–7) за текущий цикл |
| `number.humidifier_puh_9105_puh_2709_intensity` | Текущая интенсивность устройства |
| `switch.humidifier_puh_9105_puh_2709_power` | Питание устройства |
| `humidifier.humidifier_puh_9105_puh_2709_humidifier` | Humidifier entity (mode, humidity) |
| `sensor.humidifier_puh_9105_puh_2709_error` | Ошибка устройства (no_error, low_water, etc) |
| `binary_sensor.humidifier_puh_9105_puh_2709_water_tank` | Бак снят (on = снят) |
| `input_number.humidifier_tank_capacity_ml` | Ёмкость бака (константа, 5000 мл) |
| `input_number.humidifier_calibration_cycles` | Кол-во циклов калибровки расхода |
| `input_text.humidifier_consumption_rates` | JSON: калиброванные ставки мл/ч для уровней 1–7 |
| `input_text.humidifier_last_calibration` | JSON: результат последней калибровки |
| `input_datetime.humidifier_cycle_start` | Начало текущего цикла (для history_stats) |

## Автоматизации

| ID | Назначение |
|----|-----------|
| `humidity_device_setup` | Инициализация при старте/включении |
| `humidity_mode_enforcer` | Гарантирует режим home |
| `humidity_control_main` | Основная логика управления интенсивностью |
| `humidifier_calibrate_rates` | Автокалибровка расхода per-level при опустошении (history_stats) |
| `humidity_device_error` | Уведомление об ошибке (только переход из нормы в ошибку) |

## Архитектура регулятора

### Блок 1: Humidity Rate
- Кольцевой буфер 5 шагов × 1 мин = 5 мин окно
- EMA сглаживание (0.4 new + 0.6 old)
- Триггер: `time_pattern /1`

### Блок 2: Recommended Intensity (PD-регулятор)
- `predicted_error = error - (rate × 7)` — прогноз на 7 мин
- **Недоувлажнение** (`predicted_error > DEADBAND 2.0`): **сразу максимум (7)**
- **Зона стабильности** (`-DEADBAND..+DEADBAND`): поддержание текущей или плавное снижение
- **Переувлажнение**: ступенчатое снижение до 0
- Rate limit: +3 вверх, -2 вниз
- Демпфирование при приближении к цели
- Антиосцилляция: кольцевой буфер i1–i5, при flapping ≥3 — держим текущую
- Антидребезг: 30 секунд между изменениями

### Блок 3: Water Level (автокалибровка per-level)
- Расход: калиброванная ставка `rates[intensity]` мл/ч (хранится в `input_text.humidifier_consumption_rates`)
- Начальные ставки: линейная модель `intensity × 500/7` мл/ч
- **Трекинг через history_stats** (нативный HA): 7 сенсоров `sensor.humidifier_level_N_hours` считают время на каждом уровне от `input_datetime.humidifier_cycle_start`
- Сброс consumed при `low_water → no_error` (заправка)
- Калибровка при `→ low_water` и consumed > 1000 мл:
  - `correction = capacity / model_consumed`
  - Уровни с >0.1ч работы — корректируем (EMA)
  - Уровни с <0.1ч — оставляем (недостаточно данных)
  - Сброс `cycle_start` на now() (history_stats начинают новый цикл)
- **ВАЖНО**: `trigger is not defined` guard обязателен — без него sensor не создаётся при старте HA

### Триггер основной автоматизации
- `sensor.recommended_intensity` (state change) — НЕ физические сенсоры
- Это устраняет race condition

### Manual Override Detector
- **Полностью удалён**. Все ссылки удалены. Delay сокращён до 3s.

## Как работать с HA API

### REST API — состояния
```bash
curl -s -H "Authorization: Bearer TOKEN" "https://hass.zapbrannigan.org/api/states/ENTITY_ID" | python -m json.tool
```

### REST API — история
```bash
curl -s -H "Authorization: Bearer TOKEN" "https://hass.zapbrannigan.org/api/history/period/2026-01-28T18:00:00Z?end_time=2026-01-29T06:00:00Z&filter_entity_id=ENTITY1,ENTITY2&minimal_response&significant_changes_only" | python -m json.tool
```

### REST API — проверка конфигурации
```bash
curl -s -H "Authorization: Bearer TOKEN" "https://hass.zapbrannigan.org/api/config/core/check_config" -X POST | python -m json.tool
```

### REST API — рендер шаблона
```bash
curl -s -H "Authorization: Bearer TOKEN" -X POST -H "Content-Type: application/json" -d '{"template": "{{ states(\"sensor.recommended_intensity\") }}"}' "https://hass.zapbrannigan.org/api/template"
```

### WebSocket — System Log (ОШИБКИ HA)
```python
python3 -c "
import json, asyncio, websockets
async def main():
    async with websockets.connect('wss://hass.zapbrannigan.org/api/websocket') as ws:
        await ws.recv()
        await ws.send(json.dumps({'type':'auth','access_token':'TOKEN'}))
        await ws.recv()
        await ws.send(json.dumps({'id':1,'type':'system_log/list'}))
        msg = json.loads(await ws.recv())
        for e in msg.get('result',[]):
            print(e['level'], '|', e['name'], '|', str(e.get('message',''))[:200])
asyncio.run(main())
"
```

### WebSocket — Spook Repairs
```python
python3 -c "
import json, asyncio, websockets
async def main():
    async with websockets.connect('wss://hass.zapbrannigan.org/api/websocket') as ws:
        await ws.recv()
        await ws.send(json.dumps({'type':'auth','access_token':'TOKEN'}))
        await ws.recv()
        await ws.send(json.dumps({'id':1,'type':'repairs/list_issues'}))
        msg = json.loads(await ws.recv())
        for issue in msg.get('result',{}).get('issues',[]):
            print(json.dumps(issue, indent=2, ensure_ascii=False))
asyncio.run(main())
"
```

### ВАЖНО: REST API `/api/error_log` НЕ РАБОТАЕТ (404). Логи только через WebSocket `system_log/list`.

## Уроки и грабли

1. **Дубликат ключа `action:` в YAML** — закомментированный блок с незакомментированным `action:` перезаписывает action предыдущей автоматизации.

2. **`device_class: water`** требует единиц `L`, `m³`, `gal`. Единица `ml` не поддерживается.

3. **`seconds: "/60"` невалидно** — допустимый диапазон 0–59. Использовать `minutes: "/1"`.

4. **`trigger` не существует при старте HA** для trigger-based template sensors. Обязательно: `{% if trigger is not defined %}{{ this.state | float(0) }}{% else %}...{% endif %}`.

5. **Reload YAML НЕ создаёт новые** `input_number`, `input_text`, `input_datetime` и trigger-based template sensors. Нужен полный рестарт HA.

6. **Spook warnings** — доступны ТОЛЬКО через WebSocket `repairs/list_issues`.

7. **Race condition** — если template sensor и automation триггерятся на одно событие, automation может прочитать старое значение. Решение: триггерить automation на output sensor.

8. **`mode: single` + delay** = потерянные триггеры. Минимизировать delay.

9. **`input_text` / `input_number` / `input_datetime` персистятся** в `.storage` автоматически. Поле `initial` — только для первого создания. Калиброванные данные сохраняются при рестарте.

10. **`dict()` в Jinja2** — для обновления JSON в шаблонах использовать `dict(ns.result, **{k: v})` с namespace.

11. **Калибровка ёмкости бака — бессмысленна**. Бак не резиновый, ёмкость фиксированная. Калибровать нужно расход per-level, а не ёмкость.

12. **`from_json` / `to_json` в Jinja2** — парсят/сериализуют JSON из `input_text`. Обязательно проверять `not in ['unknown','unavailable','']` перед `from_json`.

13. **Файлы в репозитории**:
    - `README.md` — общая информация
    - `packages/humidity/README.md` — документация пакета (v3.1.0)
    - `packages/humidity/humidity_control.yaml` — YAML-пакет
    - `packages/humidity/SESSION_CONTEXT.md` — контекст сессии (этот файл)
    - `dashboards/humidity_dashboard.yaml` — дашборд Plotly + gauges
    - `docs/HUMIDITY_CHANGELOG.md` — история изменений
    - `docs/NAMING_CONVENTIONS.md` — правила нейминга
    - `docs/DEBUGGING_HA.md`, `docs/LESSONS_LEARNED.md`, `docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md` — справочные доки

14. **Коммит-стиль репо**: семантический, английский. Примеры: `Fix: ...`, `Docs: ...`, `Add humidity control dashboard with ...`.

15. **Уведомления** отправляются через `notify.zapgroup`.

16. **Коды ошибок устройства**: `00`/`no_error` = норма, `01`/`low_water` = мало воды, `02`/`child_lock` = блокировка (не ошибка), `03`/`replace_filter`, `04`/`maximum_schedules`, `05`/`clean_tank`.

17. **Dashboard layout**: `custom:grid-layout` (2fr + 1fr), `custom:plotly-graph` для графика, `type: gauge` для шкал, `type: entities` для диагностики. Plotly поддерживает `fn: $fn(...)` для кастомных линий.

18. **Дашборд attribute row**: `type: attribute` с `entity`, `attribute`, `name`, `icon`, `suffix`.

19. **Jinja `>` block scalar + `{% %}` tags = whitespace в выводе**. Всегда использовать `{%- -%}` trim-маркеры в шаблонах которые генерируют значения для `set_value` / `set_datetime`. Без trim шаблон может молча фейлить из-за лишних пробелов.

20. **`history_stats` для трекинга per-level** — нативный HA, считает время в конкретном state. Работает с любым entity (не только binary_sensor). `start` может быть шаблоном с `input_datetime`. Заменил сломанный ручной трекинг через `input_text` + automation.

21. **`input_text.humidifier_level_minutes` + `automation.humidifier_track_level_minutes` — УДАЛЕНЫ**. Ручной трекинг минут не работал (automation.trigger срабатывал, но set_value молча не записывал). Заменено на history_stats.

## Потенциальные следующие шаги

- Мониторинг автокалибровки per-level (несколько циклов опустошения)
- Наблюдение за агрессивным режимом (всегда 7) — может потребоваться вернуть градуированные пороги
- Гистерезис осцилляций если обнаружится flapping в зоне стабильности
- График потребления воды за время
- Красивая карточка калибровки на дашборде
- **Дашборд: починить график** — axis alignment (humidity absolute vs error zero-line)
