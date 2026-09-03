# управление вентиляцией по CO₂ — Drivent V2

пакет управляет вентиляцией комнат `zap` и `eva`. В версии 2.1.0 новый cycle-identified PI controller и temperature protection применяются только к `zap`; EVA остаётся на существующем control law.

## архитектурные инварианты

- controller layer пересчитывает recommendation каждую минуту;
- горизонт прогноза controller — 10 минут;
- actuator layer двигает физический привод не чаще одного раза в 30 минут;
- actuator layer не содержит CO₂, slope, deadband или temperature logic;
- recommendation не использует current cover position либо cooldown как controller state;
- safety close остаётся отдельным приоритетным слоем;
- коэффициенты ZAP выводятся автоматически из принятых cycle samples.

30-минутный actuator gate защищает ресурс WindowMaster/Drivent. Для WindowMaster WMX 803 производитель указывает 10 000 opening/closing movements; проектный ориентир — не более 20–40 реальных движений в сутки.

## ZAP signal chain

- `sensor.sensor_zap_co2` — raw CO₂;
- `sensor.ventilation_co2_filtered` — low-pass CO₂;
- `sensor.ventilation_co2_slope` — 15-минутная time-weighted derivative;
- `sensor.ventilation_forecast_co2_30m` — совместимый legacy ID, фактический горизонт 10 минут;
- `sensor.ventilation_forecast_error_30m` — forecast error;
- `sensor.ventilation_zap_room_model` — cycle-synchronous room model;
- `sensor.ventilation_zap_thermal_cap` — temperature-based upper bound;
- `sensor.ventilation_recommended_position` — final continuous recommendation;
- `sensor.ventilation_co2_error` — текущая filtered CO₂ error.

## cycle-identified room model

`sensor.ventilation_zap_room_model` оценивает параметры только на границе завершённого физического цикла — на 29-й и 59-й минуте часа.

sample принимается, если:

- после последнего движения прошло минимум 15 минут;
- target не менялся между соседними model ticks;
- CO₂ slope доступен;
- overload sensors выключены;
- sample находится в заданных bounds и не является резким relative outlier.

модель хранит:

- `closed_load_estimate_ppm_min`;
- `ventilation_gain_estimate_ppm_min_per_fraction`;
- `tau_estimate_min`;
- отдельные sample counts;
- `model_confidence_pct`;
- last-known-good estimates.

оценка не обновляется на каждом sensor event и не использует actuator cooldown в recommendation.

## ZAP PI controller

`Ventilation Recommended Position` выполняется одним minute trigger.

control law:

`co2_demand = equilibrium_opening + Kp × error_outside_deadband + bounded_forecast_assist + integral`

- `equilibrium_opening` выводится из room load и ventilation gain;
- `Kp` и `Ti` автоматически выводятся из gain и tau;
- integral умножается на фактический `dt`;
- при первом переходе на generation 3 и при изменении target/min/max используется bumpless tracking;
- anti-windup удерживает integral на min/max и на thermal cap;
- forecast assist ограничен 5% room-local диапазона, а не прежними 20%;
- final recommendation ограничивается room-local min/max.

## temperature protection ZAP

основная room temperature:

`sensor.sensor_zap_temperature`

heating setpoint:

`state_attr('climate.radiator_left_zap', 'temperature')`

`climate.radiator_back_zap` не является обязательным входом: на момент реализации entity была `unavailable`.

thermal supervisor каждую минуту:

1. обновляет сглаженный room temperature slope с реальным `dt`;
2. вычисляет `predicted_room_temperature_10m`;
3. сравнивает прогноз с heating setpoint;
4. формирует continuous `temperature_cap`.

в диапазоне дефицита от 0 до 5°C quadratic cap плавно уменьшается от `max_position` к `min_position`: малый дефицит даёт небольшое призакрытие, а сильное охлаждение усиливает ограничение.

`final_recommendation = min(co2_demand, temperature_cap)`

`min_position` остаётся ventilation floor, поэтому thermal supervisor сам не закрывает окно полностью. При недоступной room temperature или setpoint thermal cap отключается; существующие outdoor temperature, overload и CO₂-unavailable safety automations продолжают действовать.

## actuator layer

`automation.ventilation_control_main`:

- срабатывает по `/30`;
- читает latest `sensor.ventilation_recommended_position`;
- при отличии от current position и выполненных safety/cooldown conditions копирует recommendation целиком;
- обновляет `input_datetime.ventilation_last_adjustment`.

fixed steps и дополнительные CO₂ conditions в actuator layer отсутствуют.

## safety ZAP

обычное управление блокируется, а safety automation закрывает окна при:

- наружной температуре ниже `input_number.ventilation_min_outdoor_temp`;
- overload любого ZAP actuator;
- недоступности `sensor.sensor_zap_co2` более двух минут.

## EVA

EVA сохраняет существующие entities и controller:

- `sensor.ventilation_eva_co2_filtered`;
- `sensor.ventilation_eva_co2_slope`;
- `sensor.ventilation_eva_forecast_co2_30m`;
- `sensor.ventilation_eva_recommended_position`;
- `automation.ventilation_eva_control_main`.

ZAP model и temperature entities не используются EVA. Для EVA нужна отдельная room temperature/setpoint binding и отдельное пользовательское решение.

## основные настройки

- `input_number.ventilation_target_co2`;
- `input_number.ventilation_min_position`;
- `input_number.ventilation_max_position`;
- `input_number.ventilation_deadband_ppm`;
- `input_number.ventilation_min_outdoor_temp`;
- `input_datetime.ventilation_last_adjustment`.

## ZAP diagnostics

- raw/filtered CO₂ и slope;
- current error и 10-minute forecast;
- recommendation и actual group position;
- room model confidence/load/gain/tau;
- equilibrium, automatic gains, integral и bounded forecast assist;
- room temperature, heating setpoint, predicted temperature и thermal cap;
- overload/safety state.

**Device:** Drivent V2, grouped ZAP covers + single EVA cover

**Version:** 2.1.0