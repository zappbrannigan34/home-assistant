# CO2 Ventilation Control — Drivent V2 (zap + eva)

Автоматическое управление проветриванием для двух комнат (`zap`, `eva`) через приводы Drivent V2 на основе уровня CO2.

---

## Почему не PID?

PID-регулятор дёргает привод на **каждое измерение**: CO2 постоянно меняется из-за внешних условий, и система никогда не приходит в равновесие. Результат — привод работает каждую минуту, изнашивается за год.

Этот пакет использует **двухслойное управление**:
- **controller layer** непрерывно пересчитывает `recommended_opening`
- **actuation layer** двигает привод не чаще чем **раз в 30 минут**
- controller использует **EMA / low-pass**, производную по сглаженному сигналу и **30-минутный прогноз**
- dead zone работает по `forecast_error_30m` с hysteresis `enter=50 ppm`, `exit=65 ppm` и trend-gating

`zap` управляется как **одна логическая группа окон** поверх двух физических приводов (`left` + `back`).

Реальных движений привода: **20-40 в день** вместо 1440 при PID.

---

## Алгоритм

### Комнаты и логические приводы

- `zap`:
  - CO2: `sensor.sensor_zap_co2`
  - логический привод: `cover.ventilation_zap_group`
  - физические приводы: `cover.drivent_zap_left_driventokonnyi_privod`, `cover.drivent_zap_back_driventokonnyi_privod`
- `eva`:
  - CO2: `sensor.sensor_eva_co2`
  - привод: `cover.drivent_eva_driventokonnyi_privod`

### Signal chain

Для каждой комнаты используется сигналовая цепочка:
- `raw CO2`
- `filtered CO2` через **EMA / low-pass**
- `slope` по сглаженному сигналу
- `forecast CO2 30m`
- `forecast error 30m`
- `continuous recommended_opening`

Для `zap`:
- `sensor.ventilation_co2_filtered`
- `sensor.ventilation_co2_slope`
- `sensor.ventilation_forecast_co2_30m`
- `sensor.ventilation_forecast_error_30m`

Для `eva`:
- `sensor.ventilation_eva_co2_filtered`
- `sensor.ventilation_eva_co2_slope`
- `sensor.ventilation_eva_forecast_co2_30m`
- `sensor.ventilation_eva_forecast_error_30m`

### CO2 Trend Calculator

Качественный тренд определяется по **slope сглаженного CO2**:

| Разница | Тренд |
|---------|-------|
| > +0.5 ppm/min | `rising` |
| < -0.5 ppm/min | `falling` |
| в пределах ±0.5 ppm/min | `stable` |

### Controller logic

Ошибка считается не от сырого CO2 и не от текущего cover position, а от:

- `forecast_error_30m = forecast_co2_30m - target`

То есть recommendation отвечает на вопрос:

> какое открытие окна нужно сейчас, чтобы через 30 минут CO2 оказался ближе к цели

### Dead zone + hysteresis

- вход в dead zone: `abs(forecast_error_30m) <= 50 ppm`
- удержание внутри зоны по hysteresis: пока `abs(forecast_error_30m) <= 65 ppm`
- внутри dead zone: `hold`
- но если slope резко уходит от цели, разрешается небольшое bias-смещение recommendation даже внутри зоны
- hysteresis используется только для dead-zone state и не делает recommendation ступенчатой

### Dynamic controller delta

Вместо старых фиксированных step bands теперь recommendation вычисляется **напрямую** как continuous controller output:

- magnitude зависит от:
  - `abs(forecast_error_30m)` за пределами dead zone
  - `abs(slope)` и того, поддерживает ли тренд уход от цели
- sign зависит от знака forecast error
- recommendation не копирует actuator state, не накапливает previous recommendation и не ждёт cooldown

### Actuation layer

- каждые 30 минут automation читает текущее `recommended_opening`
- если recommendation отличается от текущего положения окна и safety/cooldown позволяют — применяет его
- слой движения физически отделён от непрерывного расчёта recommendation

### Rate Limit

- Recommendation: continuous / minute-level recompute
- Physical movement: not more often than every 30 minutes
- Cooldown stored separately by room:
  - `input_datetime.ventilation_last_adjustment` — zap
  - `input_datetime.ventilation_eva_last_adjustment` — eva

---

## Безопасность

| Событие | Действие |
|---------|---------|
| T улицы < min порога | Закрыть окно + уведомление |
| Перегрузка привода | Закрыть окно + уведомление |
| CO2 сенсор unavailable (>2 мин) | Закрыть окно + уведомление |

## Состав пакета

### Template Sensors

| Sensor | Описание |
|--------|----------|
| `sensor.co2_trend` | Качественный тренд CO2 для zap |
| `sensor.ventilation_co2_mean_30m` | Сглаженный CO2 для zap за 30 минут |
| `sensor.ventilation_co2_change_30m` | Изменение CO2 для zap за 30 минут |
| `sensor.ventilation_co2_filtered` | Непрерывно сглаженный CO2 для zap |
| `sensor.ventilation_co2_slope` | Производная по сглаженному CO2 для zap |
| `sensor.ventilation_forecast_co2_30m` | Прогноз CO2 для zap на 30 минут |
| `sensor.ventilation_forecast_error_30m` | Прогнозная ошибка CO2 для zap |
| `sensor.ventilation_recommended_position` | Continuous recommended opening для zap |
| `sensor.ventilation_co2_error` | Forecast error для zap |
| `sensor.eva_co2_trend` | Качественный тренд CO2 для eva |
| `sensor.ventilation_eva_co2_mean_30m` | Сглаженный CO2 для eva за 30 минут |
| `sensor.ventilation_eva_co2_change_30m` | Изменение CO2 для eva за 30 минут |
| `sensor.ventilation_eva_co2_filtered` | Непрерывно сглаженный CO2 для eva |
| `sensor.ventilation_eva_co2_slope` | Производная по сглаженному CO2 для eva |
| `sensor.ventilation_eva_forecast_co2_30m` | Прогноз CO2 для eva на 30 минут |
| `sensor.ventilation_eva_forecast_error_30m` | Прогнозная ошибка CO2 для eva |
| `sensor.ventilation_eva_recommended_position` | Continuous recommended opening для eva |
| `sensor.ventilation_eva_co2_error` | Forecast error для eva |

У recommendation sensors есть диагностические attributes:

- `controller_family`
- `controller_mode`
- `dead_zone_state`
- `trend_gate`
- `trend_gate_factor`
- `error_outside_dead_zone_ppm`
- `normalized_error_factor`
- `slope_factor`
- `neutral_baseline`
- `raw_controller_output`

### Input Helpers

| Entity | Описание | Default |
|--------|----------|---------|
| `input_number.ventilation_target_co2` | Общий целевой CO2 для обеих комнат | 650 ppm |
| `input_number.ventilation_min_position` | Общая минимальная позиция | 0% |
| `input_number.ventilation_max_position` | Общая максимальная позиция | 80% |
| `input_number.ventilation_min_outdoor_temp` | Общий минимум уличной температуры | -10°C |
| `input_datetime.ventilation_last_adjustment` | Последняя реальная регулировка zap | 2000-01-01 00:00:00 |
| `input_datetime.ventilation_eva_last_adjustment` | Последняя реальная регулировка eva | 2000-01-01 00:00:00 |

### Automations

| Automation | Описание |
|------------|----------|
| `ventilation_control_main` | Основное управление zap |
| `ventilation_safety_close` | Аварийное закрытие zap |
| `ventilation_device_error` | Уведомления об ошибках приводов zap |
| `ventilation_eva_control_main` | Основное управление eva |
| `ventilation_eva_safety_close` | Аварийное закрытие eva |
| `ventilation_eva_device_error` | Уведомления об ошибках привода eva |

### Используемые внешние entity

| Entity | Тип | Назначение |
|--------|-----|-----------|
| `sensor.sensor_zap_co2` | Source | CO2 в комнате zap (ppm) |
| `sensor.sensor_eva_co2` | Source | CO2 в комнате eva (ppm) |
| `cover.ventilation_zap_group` | Device | Логическая группа двух окон zap |
| `cover.drivent_zap_left_driventokonnyi_privod` | Device | Физический привод zap left |
| `cover.drivent_zap_back_driventokonnyi_privod` | Device | Физический привод zap back |
| `cover.drivent_eva_driventokonnyi_privod` | Device | Привод eva |
| `binary_sensor.drivent_zap_left_datchik_peregruzki` | Device | Перегрузка zap left |
| `binary_sensor.drivent_zap_back_datchik_peregruzki` | Device | Перегрузка zap back |
| `binary_sensor.drivent_eva_datchik_peregruzki` | Device | Перегрузка eva |
| `weather.forecast_home_assistant` | Source | Уличная температура |

---

## Установка

1. Скопируйте `packages/ventilation/` в `/config/packages/`.
2. **Перезагрузите Home Assistant** (полный рестарт — reload не создаст input_number и trigger-based sensors).
3. Проверьте, что доступны:
   - `sensor.sensor_zap_co2`
   - `sensor.sensor_eva_co2`
   - `cover.ventilation_zap_group`
   - `cover.drivent_eva_driventokonnyi_privod`
4. Настройте параметры через UI:
   - Целевой CO2 (по умолчанию 650 ppm)
   - Минимальная уличная температура (по умолчанию -10°C)
   - Максимальная позиция окна (по умолчанию 80%)

---

## Ресурс привода

| Параметр | Значение |
|----------|---------|
| WindowMaster WMX 803 (аналог) | 10 000 циклов при номинальной нагрузке |
| Drivent V2 (разработчик) | 4+ года на PID раз в минуту |
| Этот пакет (оценка) | ~20-40 движений/день → **1.5-3 года** при 10k циклов |

---

**Device:** Drivent V2 (Wi-Fi, MQTT)
**Sensor:** sensor-zap (CO2, Senseair S8 или аналог)
**Version:** 1.4.0
