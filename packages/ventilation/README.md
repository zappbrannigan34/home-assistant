# CO2 Ventilation Control — Drivent V2 (zap + eva)

Автоматическое управление проветриванием для двух комнат (`zap`, `eva`) через приводы Drivent V2 на основе уровня CO2.

---

## Почему не PID?

PID-регулятор дёргает привод на **каждое измерение**: CO2 постоянно меняется из-за внешних условий, и система никогда не приходит в равновесие. Результат — привод работает каждую минуту, изнашивается за год.

Этот пакет использует **гибридное зонное управление по длинному тренду**:
- Смотрим **30-минутное сглаженное среднее** CO2 и **30-минутное изменение**
- Двигаем привод **только когда длинное окно подтверждает необходимость**
- Двигаем привод **переменным шагом** в зависимости от величины ошибки CO2
- Обычная оценка и регулировка — **раз в 30 минут**

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

### CO2 Trend Calculator

Пакет использует room-specific статистические сенсоры:
- `sensor.ventilation_co2_mean_30m` — сглаженное среднее CO2 за 30 минут
- `sensor.ventilation_co2_change_30m` — изменение CO2 за 30 минут
- `sensor.ventilation_eva_co2_mean_30m` — сглаженное среднее CO2 для eva за 30 минут
- `sensor.ventilation_eva_co2_change_30m` — изменение CO2 для eva за 30 минут

Тренд определяется по `sensor.ventilation_co2_change_30m`:

| Разница | Тренд |
|---------|-------|
| > +30 ppm | `rising` |
| < -30 ppm | `falling` |
| в пределах ±30 | `stable` |

### Переменный шаг по ошибке CO2

Ошибка считается от **30-минутного сглаженного CO2**, а не от сырого мгновенного значения.

| Модуль ошибки | Шаг | Правило |
|---------------|-----|---------|
| `<= 40 ppm` | `0%` | hold / deadband |
| `40..80 ppm` | `5%` | только с подтверждением трендом на 2 окна |
| `80..140 ppm` | `10%` | сразу, если тренд не противоречит |
| `140..220 ppm` | `20%` | сразу, без подтверждения |
| `>220 ppm` | `30%` | сразу, без подтверждения |

Знак шага зависит от знака ошибки:
- CO2 выше цели → открыть окно
- CO2 ниже цели → закрыть окно

### Подтверждение сигнала

В малой зоне (`40..80 ppm`) пакет требует **2 одинаковых 30-минутных сигнала подряд**, прежде чем двигать привод. Это убирает дёрганость на нестабильных воздушных потоках.

В средней зоне (`80..140 ppm`) тренд остаётся фильтром, но подтверждение на 2 окна уже не требуется.

В большой и экстремальной зоне (`>140 ppm`) система двигается сразу крупным шагом, потому что ограничение должно быть по **частоте**, а не по величине движения.

### Rate Limit

- Оценка позиции: каждые 30 минут
- Минимальный интервал между реальными движениями: 30 минут
- Cooldown хранится отдельно по комнате:
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
| `sensor.co2_trend` | Тренд CO2 для zap |
| `sensor.ventilation_co2_mean_30m` | Сглаженный CO2 для zap за 30 минут |
| `sensor.ventilation_co2_change_30m` | Изменение CO2 для zap за 30 минут |
| `sensor.ventilation_recommended_position` | Рекомендуемая позиция окна для zap |
| `sensor.ventilation_co2_error` | Отклонение сглаженного CO2 для zap от цели |
| `sensor.eva_co2_trend` | Тренд CO2 для eva |
| `sensor.ventilation_eva_co2_mean_30m` | Сглаженный CO2 для eva за 30 минут |
| `sensor.ventilation_eva_co2_change_30m` | Изменение CO2 для eva за 30 минут |
| `sensor.ventilation_eva_recommended_position` | Рекомендуемая позиция окна для eva |
| `sensor.ventilation_eva_co2_error` | Отклонение сглаженного CO2 для eva от цели |

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
**Version:** 1.2.0
