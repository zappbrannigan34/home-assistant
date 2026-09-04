# управление вентиляцией по CO₂ — Drivent V2

пакет управляет вентиляцией комнат `zap` и `eva`. В версии 2.6.0 computed TRV-to-Min-Indoor range, signed room forecast, normalized outdoor deviation, low-CO₂ ceiling и persistent UI helpers применяются только к `zap`; EVA сохраняет прежний control law.

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
- target/min/max не менялись между соседними model ticks;
- CO₂ slope доступен;
- overload sensors выключены;
- sample находится в заданных bounds и не является резким relative outlier.

оценки используют `room_load = co2_slope + ventilation_gain × opening_fraction` и `ventilation_gain = (room_load − co2_slope) / opening_fraction`, поэтому load sample не зависит от обязательного полного закрытия окна.

модель хранит provisional estimates, отдельные sample counts и confidence. Active controller читает только accepted last-known-good coefficients.

до набора `load/gain/tau` counts `4/6/4` используется history-derived ZAP bootstrap из сохранённого режима C: `room_load=2.597 ppm/min`, `ventilation_gain=6.681 ppm/min/fraction`, `tau=30.715 min`. После достижения порога новый набор повышает `accepted_model_generation`; его применение выполняется bumpless.

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
- при первом переходе на generation 7, при изменении target/min/max и при принятии новых room-model coefficients используется bumpless tracking;
- anti-windup удерживает integral на min/max, low-CO₂ ceiling и thermal cap;
- forecast assist ограничен 5% room-local диапазона, а не прежними 20%;
- `co2_guard = max(filtered_CO₂, forecast_CO₂)`;
- при `co2_guard <= target - deadband` CO₂ ceiling равен `min_position`, при `co2_guard >= target + deadband` — `max_position`, между границами применяется smoothstep;
- final recommendation равна минимуму PI demand, low-CO₂ ceiling и thermal cap.

## temperature protection ZAP

основная room temperature:

`sensor.sensor_zap_temperature`

fallback room temperature:

`sensor.radiator_left_zap_local_temperature`

radiator setpoint `state_attr('climate.radiator_left_zap', 'temperature')` является верхней границей вычисляемого comfort range. При временной недоступности entity используется last-known-good числовое значение.

thermal supervisor каждую минуту:

1. обновляет сглаженный signed room temperature slope с фактическим `dt`;
2. вычисляет `predicted_room_temperature_10m = room + slope × 10`;
3. вычисляет `temperature_range = TRV_setpoint − min_indoor`;
4. нормирует положение прогноза внутри этого диапазона;
5. нормирует outdoor deficit ниже `min_outdoor` на тот же вычисленный диапазон;
6. выбирает меньший normalized factor и линейно переводит его в opening cap.

`temperature_range = TRV_setpoint - min_indoor`

`room_factor = clamp((predicted_temperature_10m - min_indoor) / temperature_range, 0, 1)`

`outdoor_deficit = max(min_outdoor - outdoor_temperature, 0)`

`outdoor_factor = clamp(1 - outdoor_deficit / temperature_range, 0, 1)`

`opening_factor = min(room_factor, outdoor_factor)`

`temperature_cap = min_position + (max_position - min_position) × opening_factor`

следствия:

- отрицательный slope понижает прогноз и уменьшает room factor; положительный slope повышает прогноз и отпускает cap;
- на `min_indoor` room factor равен нулю; на TRV setpoint — единице;
- outdoor выше `min_outdoor` не уменьшает opening; deficit ниже порога уменьшается пропорционально тому же computed range;
- hardcoded indoor/outdoor temperature spans отсутствуют;
- `cooling_time_to_floor_minutes` остаётся диагностикой и не вводит скрытую управляющую уставку;
- при недоступном outdoor factor fail-safe равен нулю; при отсутствующем/невалидном TRV-to-floor range cap равен `min_position`;
- hard-safety full close остаётся отдельным слоем.

обязательный sanity case при `TRV=27°C`, `min_indoor=22°C`, room/predicted около `26.5°C`, outdoor `21°C` и `min_outdoor=18°C`: computed range=`5°C`, room factor находится около верхней части диапазона, outdoor factor=`1`, поэтому cap должен быть высоким, но пропорционально ниже `max_position`; он не должен быть ни `min_position`, ни искусственно равен `max_position`.

`final_recommendation = min(co2_demand, co2_ceiling, temperature_cap)`

`min_position` остаётся ventilation floor. Diagnostics публикуют setpoint/floor, computed range, signed slope, 10-minute forecast, normalized room/outdoor factors, time-to-floor и фактически binding deviation.

## actuator layer

`automation.ventilation_control_main`:

- срабатывает по `/30`;
- читает latest `sensor.ventilation_recommended_position`;
- при отличии от current position и выполненных safety/cooldown conditions копирует recommendation целиком;
- обновляет `input_datetime.ventilation_last_adjustment`.

fixed steps и дополнительные CO₂ conditions в actuator layer отсутствуют.

## safety ZAP

`input_boolean.ventilation_use_indoor_temperature` выбирает режим hard safety:

- `on` — primary `sensor.sensor_zap_temperature`, затем fallback `sensor.radiator_left_zap_local_temperature`, затем наружная температура;
- `off` — используется только наружная температура.

для внутренних источников применяется `input_number.ventilation_min_indoor_temp`; для outdoor-only и outdoor fallback — `input_number.ventilation_min_outdoor_temp`. Все `input_number`, `input_boolean` и `input_datetime` этого package не задают `initial`: пользовательские UI values и actuator cooldown timestamps восстанавливаются из Home Assistant state после restart.

`sensor.ventilation_zap_safety_temperature` публикует выбранное значение, source, threshold и fallback state. Если ни один разрешённый источник недоступен, automation закрывает окна fail-safe.

обычное управление блокируется, а safety automation закрывает окна при selected temperature на соответствующем пороге или ниже, overload любого ZAP actuator или недоступности `sensor.sensor_zap_co2` более двух минут.

в режиме `off` thermal cap отключён и защита температуры выполняется только по наружному hard threshold. В режиме `on` thermal cap использует ту же primary/fallback indoor chain; при смене source temperature slope сбрасывается без ложного скачка.

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
- `input_boolean.ventilation_use_indoor_temperature`;
- `input_number.ventilation_min_indoor_temp`;
- `input_number.ventilation_min_outdoor_temp`;
- `input_datetime.ventilation_last_adjustment`.

## ZAP diagnostics

- raw/filtered CO₂ и slope;
- current error и 10-minute forecast;
- recommendation и actual group position;
- room model confidence/load/gain/tau;
- equilibrium, automatic gains, integral и bounded forecast assist;
- room temperature, heating setpoint, predicted temperature и thermal cap;
- selected safety temperature, source, fallback state и оба temperature thresholds;
- overload/safety state.

**Device:** Drivent V2, grouped ZAP covers + single EVA cover

**Version:** 2.2.0