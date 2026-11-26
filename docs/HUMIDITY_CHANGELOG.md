# Humidity Control Automation - Changelog

История изменений автоматизации управления увлажнителем.

**Repository:** https://github.com/zappbrannigan34/home-assistant.git  
**File:** YAML‑пакет увлажнителя (например `packages/humidity/humidity_control.yaml`)

---

## v4.7.0 (2026-11-26)

**Version:** 4.7.0

### Изменения: защита от конденсата и воды из фильтра (`REFILL_PENDING >= 120 мл`), удаление `periodic_probe`, импульсный классификатор `low_water` и точные `history_stats`

1. **Полное удаление `periodic_probe` и двухфазное подтверждение заправки по объёму (`>= 120 мл`):**
   - Удалён таймер `periodic_probe` (`minutes: "/15"` / `"/30"`) и `power_on_55s`, которые включали пустой увлажнитель на стекшем со стенок бака конденсате и остаточной воде из фильтра (`15–50 мл`, `38–108 с` работы), вызывая циклические ложные `"✅ Бак заправлен"` → `"💧 Мало воды"` и сухой нагрев мембраны.
   - Введена неразрушающая стадия проверки заправки (`input_boolean.humidifier_*_refill_pending`, `input_number.humidifier_*_pending_baseline_ml`, `input_datetime.humidifier_*_pending_cycle_start`): при установке бака (`tank_replaced`), нажатии кнопки `ON` на корпусе, `/refill_*` в Telegram или переключении тумблера UI старые `water_consumed` и `cycle_start` **не затираются**, ТЭН `warm_stream` держится **выключенным**, а `"✅ Бак заправлен"` не отправляется до фактического испарения **`>= 120 мл`** воды (инвариантно к скорости `1..7` и паузам гистерезиса).
   - После испарения `>= 120 мл` (`humidity_refill_verified_commit`) старт цикла фиксируется задним числом (`pending_cycle_start`), `water_consumed` инициализируется фактически израсходованными `~120 мл` (через `event: humidity_refill_reset` с `initial_ml`, удалён конфликтный триггер `needs_refill -> off` на `water_consumed`), включается `warm_stream: on` и отправляется одно подтверждённое уведомление `"✅ Бак заправлен"`.
   - Для защиты от дребезга поплавка в `Standby` (`devtype 17`) добавлен счётчик отсечки `input_number.humidifier_*_tank_replaced_fails` (`< 2` попыток за 30 минут, `max: 10000`, сброс при подтверждённой заправке или новом цикле), а короткий осмотр непустого бака (`10–34 с`) возобновляет работу без сброса `water_consumed`.
2. **Импульсный классификатор `low_water` vs снятие бака руками (`5–6 с` против `55–146 с`) и взвешенная калибровка:**
   - В `polaris-mqtt` `binary_sensor.*_water_tank` подписан на `state/error/water` (тот же поплавок базы, срабатывает в пределах `±50 мс` от `low_water`, при реальном опустошении авто-сбрасывается в `'off'` за `5.15–6.07 с`, а при снятии бака руками остаётся `'on'` всё время отсутствия бака).
   - При `low_water` выполняется двухступенчатое ожидание `wait_for_trigger(2s -> on, затем 10s -> off)`: авто-сброс `< 10 с` распознаётся как реальное опустошение бака на базе (отправляется `"💧 Мало воды"` и запускается калибровка); удержание `'on'` `>= 10 с` распознаётся как снятие бака руками (`tank_lifted` ставит `tank_removed_latch = on`, без ложной калибровки и без спама `"💧 Мало воды"`).
   - В `humidifier_*_calibrate_rates` добавлены проверки `was_actively_on`, `total_hrs >= 3.0 ч`, согласованности `reconstructed_ml` с `model_consumed` (`±20%`, защита от простоев HA), окно `3600..7500 мл`, взвешивание коррекции по доле объёма каждого уровня (`share_k`) и монотонность ставок `r[1] <= ... <= r[7]`.
3. **Точный поуровневый учёт наработки и сервисные коды:**
   - Все 14 сенсоров `history_stats` (`Humidifier [Zap] Level 1..7 Hours`) переведены с `number.*_intensity*` (сохраняющего `7` в `Standby` на `MCU 1.0.9`) на `sensor.humidifier_*_effective_intensity` (`0` в паузах и авариях).
   - В `humidity_apply_area_control` убраны захардкоженные UUID устройств и хрупкий `age < 70`; нефатальные сервисные коды (`03 replace_filter`, `05 clean_tank`) уведомляют даже при `needs_refill == on` и не блокируют увлажнение.

---

## v4.6.1 (2026-11-26)

**Version:** 4.6.1

### Изменения: исправление заморозки счётчика калибровок на `11` и отмены калибровки при быстрой заправке

1. **Исправлен счётчик циклов калибровки (`input_number.*_calibration_cycles`, `humidifier_*_calibrate_rates`):**
   - В `input_number.set_value`, `input_text.*_last_calibration` и тексте уведомления запись изменена с `capped_cycles + 1` (`min(cycles, 10) + 1`, из-за чего после 11-го цикла счётчик навсегда застревал на `11`) на честный инкремент `next_cycle = cycles + 1`; ограничение `cc = min(cycles, 10)` оставлено строго для веса EMA-сглаживания ставок.
   - Потолок `max:` у `input_number.humidifier_calibration_cycles` и `input_number.humidifier_zap_calibration_cycles` увеличен с `20` до `10000`.
2. **Устранена отмена калибровки при заправке быстрее 3 минут (`delay: "00:00:10"` с проверкой закрытого бака):**
   - Прежняя пауза `delay: "00:03:00"` с проверкой `needs_refill == 'on'` и `refill_detected.current == 0` отменяла валидную калибровку каждый раз, когда пустой бак заправляли быстрее чем за 3 минуты после Telegram-уведомления.
   - Заменена на проверку того, что бак стоял на базе закрытым (`water_tank == 'off'` не менее `30 с` до `low_water`) и оставался закрытым `10 с` после `low_water` (`last_changed >= 40 с`): ложное срабатывание при раннем снятии непустого бака по-прежнему отсекается (`water_tank` размыкается за `1–2 с`), а калибровка сохраняется на `t + 10 с` до начала заправки.
   - Верхний порог `water_consumed` расширен с `7500` до `9000 мл`, границы поуровневых ставок уточнены до `0.45..1.35 * nominal` (сохраняя реальные откалиброванные малые скорости `~68–78 мл/ч` на уровне `1`), сброс `full_cycle_valid` защищён снимком `cycle_start_snap`.

---

## v4.6.0 (2026-11-25)

**Version:** 4.6.0

### Изменения: 100% управление из Home Assistant (`mode: "boost"`, скорости `1..7`), `child_lock: off` и поуровневый учёт воды

1. **Отказ от встроенной автоматики Polaris (`sleep` полностью удалён) и перевод 24/7 в `mode: "boost"` (`"5"`):**
   - В `script.humidity_apply_area_control`, `humidity_*_mode_enforcer` и пробе заправки `humidity_*_refill_detected` удалён `mode: "sleep"` (`4`), который включал внутренний авто-алгоритм увлажнителя и игнорировал `1..7`.
   - Рабочий режим — **`mode: "boost"` (`"5"`, программа `"Интенсивность"`, `limits.humidity = 0`, внутренний гигростат полностью отключён, `limits.speed = 1..7`, `switch.*_power` корректно отражает `'on'` в `polaris/switch.py:850`)**.
2. **Полный PI+D-регулятор `0..7` с гистерезисом включения/выключения (`sensor.recommended_intensity`, `sensor.humidity_zap_recommended_intensity`):**
   - При `error > 1.5` или `predicted_error > 1.5` немедленно задаётся максимальная скорость **`7`** (100% физическая мощность), около цели скорость плавно модулируется в диапазоне `1..7`, а пороги входа (`error >= 0.30` или `error >= 0.15 and predicted_error >= 0.45`) и выхода (`error <= -0.45` или `error <= -0.15 and predicted_error <= -0.55`) исключают дребезг `0 ↔ 1`.
   - `sensor.humidifier_effective_intensity` и `sensor.humidifier_zap_effective_intensity` переведены на реактивный `trigger:`-шаблон и честно показывают фактический `number.*_intensity*` (`1..7`, или `0` в `Standby`/ошибке).
3. **`child_lock` принудительно выключен (`off`) на обоих устройствах:**
   - При работе и холодном старте `child_lock`, `backlight` и `sound` удерживаются в `off` (`humidity_*_mode_enforcer` отслеживает `child_lock: on` на `10s`), что убирает мигание индикатора `ON/OFF` каждые 5 с в темноте (руководство стр. 21) и конфликт кода `02` на `devtype 17` (`zap`), сохраняя 100% радиотишину в `Standby` и при ошибках.
4. **Восстановление поуровневого учёта воды и калибровки (`1..7`):**
   - Восстановлены `input_text.humidifier_*_consumption_rates` (без `initial:`) и 14 `history_stats` сенсоров `level_1..7_hours` (при сохранении `active_hours`); удалены `input_number.humidifier_*_evaporation_rate_ml_h`.
   - Обновлён `dashboards/humidity_dashboard.yaml` (`yaxis3: [-0.5, 7.8]`, `Recommended/Effective/Device Intensity`, `Operating Mode`, `Child Lock`, `Current Rate`).

---

## v4.5.0 (2026-11-25)

**Version:** 4.5.0

### Изменения: честный предиктивный ON/OFF-контур `sleep`-режима (`0%` / `100%`) и прямая калибровка расхода (`ml/h`)

1. **Устранение фиктивной шкалы `0..7` и хардкода `else 6` (`packages/humidity/humidity_control.yaml`):**
   - В `mode: "sleep"` (`4`, единственный режим с полным гашением дисплея и кнопок) прошивка игнорирует `control/intensity` (`1..7`), всегда шлёт `state/intensity = 0` и физически испаряет при `warm_stream: on` с полной мощностью (`~500–550 мл/ч`, подтверждено ночным циклом `zap`: `5000 мл` за `9.054 ч` = `552.2 мл/ч`).
   - `sensor.recommended_intensity` (`eva`) и `sensor.humidity_zap_recommended_intensity` (`zap`) переведены в честный двухпозиционный предиктивный PD-гистерезис **`0` / `100` (`%`)** без интегрального насыщения (`integral_level`) и без зависимости от мёртвых `number.*_intensity*`: включение при `error >= +0.30%` или (`error >= +0.10%` и `predicted_error >= +0.50%`), выключение при `error <= -0.35%` или (`error <= 0.0%` и `predicted_error <= -0.35%`).
   - `sensor.humidifier_effective_intensity` и `sensor.humidifier_zap_effective_intensity` переведены на честные **`0%` / `100%`** (`unit_of_measurement: "%"`, без `state_class`, без хардкода `6`).
   - На триггерах `sensor.*recommended_intensity` в `humidity_control_main` и `humidity_zap_control_main` добавлен фильтр `to: ["0", "100"]`, исключающий лишние запуски при обновлении диагностических атрибутов.
2. **Прямая модель воды и калибровка `evaporation_rate_ml_h`:**
   - Удалены 14 мёртвых `history_stats` сенсоров `level_1..7_hours` и устаревшие JSON-таблицы `input_text.humidifier_*_consumption_rates`.
   - Добавлены 2 реальных `history_stats` сенсора наработки (`sensor.humidifier_active_hours`, `sensor.humidifier_zap_active_hours` по `humidifier == 'on'`) и 2 персистентных `input_number` без `initial:` (`input_number.humidifier_evaporation_rate_ml_h`, `input_number.humidifier_zap_evaporation_rate_ml_h`).
   - `humidifier_calibrate_rates` и `humidifier_zap_calibrate_rates` калибруют реальный расход `capacity / active_hours` (границы `300..620 мл/ч`, окно `2500..7500 мл`).
3. **Очистка дашборда (`dashboards/humidity_dashboard.yaml`):**
   - Серия графика `y3` и её ось переведены на `Mist Output` (`0–100%`, `range: [-5, 108]`).
   - В `Zap Diagnostics` и `Eva Diagnostics` удалён мёртвый ползунок `Intensity: 0` (`number.*_intensity*`) и нечитаемый JSON ставок; оставлены согласованные `Control Demand` (`0%/100%`), `Mist Output` (`0%/100%`), `Evaporation Rate` и `Active Hours`.

---

## v4.4.6 (2026-11-25)

**Version:** 4.4.6

### Изменения: надёжное обнаружение заправки на `MCU 1.0.9` (`eva`) и при верхнем доливе без снятия бака

1. **Проба воды `wait_for_trigger` `50s` и грейс `70s` (`humidity_refill_detected`, `humidity_zap_refill_detected`, `refill_probe_grace`, `mode_enforcer`):**
   - На `eva` (`devtype 70 / MCU 1.0.9`) задержка срабатывания поплавка `low_water` при пустом баке составляет `43.1s` (прежняя `14s`-проба ложно сбрасывала `needs_refill`, а при доливе без снятия бака или в `Standby` `MCU 1.0.9` не шлёт MQTT-событий).
   - Окно пробы переведено с фиксированного `14s` на `wait_for_trigger` (`timeout: 50s`) с мгновенным выходом при `low_water` / `off`, грейс-окно `refill_probe_grace` и `mode_enforcer` расширено до `70s`, `power_on_20s` увеличен до `55s`.
2. **Дневная автопроверка заправки и кнопка в Telegram:**
   - В `humidity_refill_detected` (`eva`, каждые 15 мин) и `humidity_zap_refill_detected` (`zap`, каждые 30 мин) добавлен дневной `periodic_probe` (`09:00–22:00`, при `needs_refill = on`, `available = on`, `humidifier = off`, `recommended >= 1`, не ранее чем через 10/15 мин после опустошения).
   - В уведомления о низком уровне воды добавлена inline-кнопка Telegram **`✅ Бак заправлен`** (`/refill_eva` и `/refill_zap`), запускающая немедленную пробу и старт, а `humidity_manual_refill_toggle` сразу триггерит `control_main`.

---

## v4.4.5 (2026-11-24)

**Version:** 4.4.5

### Изменения: 100% радиотишина в Standby/ошибках, устранение шторма `child_lock ↔ low_water`, диагностика и offline watchdog

1. **100% радиотишина в `Standby` (`mode == 0`) и при авариях (`low_water` / `needs_refill` / снятый бак):**
   - Из `script.humidity_apply_area_control` удалена `else:`-ветка подстройки `child_lock`/`sound`/`backlight` в `Standby`, а при выключении с активной ошибкой/пустым баком (`has_fault_or_empty`) отправляется только `switch.turn_off(power)` без команд на `child_lock` и вспомогательные переключатели.
   - Из `humidity_mode_enforcer` и `humidity_zap_mode_enforcer` удалена проверка `cl`/`snd`/`bl` при `humidifier == 'off'`, а `unexpected_power_on` сокращён с `25s` до `12s`.
   - Исключён циклический шторм `child_lock ↔ low_water` (вызывавший зависание `zap` `MCU 1.0.3` и команды в снятый бак `eva` `MCU 1.0.9`).
2. **Устранение 2-секундного ложного рестарта после `low_water` и корректный cold-start:**
   - В ветке холодного старта `script.humidity_apply_area_control` добавлены `standby_settled` (`age >= 6s`), `delay: 3s` и живая повторная проверка условий перед пуском; пробуждение выполняется через `switch.turn_on(power)` → `1s` → `humidifier.set_mode("sleep")`, а пост-стартовые переключатели (`sound`/`backlight`/`child_lock`/`warm_stream`/`ioniser`) срабатывают только при `humidifier == 'on'`.
3. **Защита автокалибровки и детектора заправки:**
   - В `humidity_refill_detected` и `humidity_zap_refill_detected` ужесточён дебаунс `tank_replaced` (`dur >= 15s`, `since_empty > 10s`), пробуждение переведено на `switch.turn_on(power)` перед `mode: "sleep"`, а `sound`/`backlight`/`child_lock` защищены условием `humidifier == 'on'`.
   - В `humidifier_calibrate_rates` и `humidifier_zap_calibrate_rates` после `delay: 3m` добавлена проверка `refill_detected.current == 0` и `water_tank == 'off' or last_changed >= 150s`.
4. **Эффективная интенсивность, дашборд и offline watchdog:**
   - Добавлены реактивные сенсоры `sensor.humidifier_effective_intensity` и `sensor.humidifier_zap_effective_intensity` (`6` в `sleep`, `0` в `Standby`, `unavailable` в офлайне).
   - В `dashboards/humidity_dashboard.yaml` график интенсивности переведён на `effective_intensity` с `connectgaps: false`, исправлена кнопка `15m` (`count: 15`, было `count: 1`), в `Diagnostics` добавлены `Effective Intensity` и `Online`.
   - Добавлена автоматизация `humidity_device_offline_watchdog` (`available == 'off'` на `10m` → `script.notify_dispatch` `p2_maintenance`; возврат после `>= 600s` → `script.notify_clear`).

---

## v4.4.4 (2026-11-24)

**Version:** 4.4.4

### Изменения: перевод уведомлений на единый диспетчер `packages/notifications`

1. Удалён локальный дубликат `script.humidity_send_notification` из `packages/humidity/humidity_control.yaml`.
2. Все 7 точек отправки уведомлений (предупреждение `<= 15%`, подтверждение заправки, ручной сброс заправки, калибровка `eva`/`zap`, ошибки устройств `eva`/`zap`) переведены на `script.notify_dispatch` (`p1_urgent` / `p2_maintenance` / `p3_info`) и `script.notify_clear` при подтверждении заправки.

---

## v4.4.3 (2026-11-23)

**Version:** 4.4.3

### Изменения: полное гашение экрана и кнопок через `mode: "sleep"` (`4`) + гистерезис включения

1. **Перевод рабочего режима с `home` (`6`) на `sleep` (`4`, `Ночной режим`) (`packages/humidity/humidity_control.yaml`):**
   - По официальной документации `PUH-9105` (стр. 9, 11, 20) и живому фотоподтверждению `backlight = false` (`0x1C`) гасит **только подсветку водяного бака**; в режиме `home` (`6`) цифровой экран и 6 сенсорных кнопок через 15 с лишь приглушаются. Единственный рабочий режим с полным гашением экрана и кнопок через 15 с тишины — `mode: "sleep"` (`4`).
   - Живой проверкой через MQTT на обеих ревизиях (`eva` `devtype 70 / MCU 1.0.9` и `zap` `devtype 17 / MCU 1.0.3`) подтверждено, что в `mode = "4"` (`sleep`) сохраняются `warm_stream: true`, `ioniser: true`, `child_lock: true`, `backlight: false`, `sound: false`, а `number.set_value(intensity)` игнорируется (`state/intensity = "0"`).
   - `script.humidity_apply_area_control`, `humidity_mode_enforcer`, `humidity_zap_mode_enforcer` и проба `tank_replaced` в `humidity_*_refill_detected` переведены на `mode: "sleep"` без отправки лишних `set_humidity` и `number.set_value(intensity)` (чтобы не будить экран).
2. **Устранение 10-секундного дребезга `on ↔ off` (`sensor.recommended_intensity`, `sensor.humidity_zap_recommended_intensity`):**
   - Добавлен гистерезис включения (`not has_on and error < 0.25 and predicted_error < 0.45 → 0`) и ускоренная разрядка `integral_level` при простое в перелёте (`0.60 * error * dt`), что исключило цикличное включение/выключение каждые 10 с около уставки и пробуждение дисплея.
3. **Учёт воды и full-cycle калибровка в режиме `sleep` (`water_consumed`, `hours_remaining`, `calibrate_rates`):**
   - При `power_on` и `state/intensity == "0"` (`sleep`) расход и `hours_remaining` считаются по откалиброванной ставке уровня `6`, а `humidifier_*_calibrate_rates` при `total_hrs < 0.5` масштабирует всю таблицу `1..7` по факту полного цикла (`5000 мл / model_consumed`).

---

## v4.4.2 (2026-11-22)

**Version:** 4.4.2

### Изменения: режим максимальной незаметности и блокировки локального управления (`Stealth & HA-Only Lockdown`), фиксация аппаратных отличий `devtype 70` vs `devtype 17`

1. **Принудительное отключение индикации/звука и 24/7 блокировка локальных кнопок (`packages/humidity/humidity_control.yaml`):**
   - В `script.humidity_apply_area_control` снято исключение `supports_child_lock` для `devtype 17` (`zap`): теперь `child_lock: on`, `backlight: off` и `sound: off` удерживаются **на обоих увлажнителях** (`devtype 70` и `devtype 17`) во всех фазах — при холодном старте (сразу после `switch.turn_on`, до 3-секундной задержки), во время работы, перед выключением (строго **до** `switch.turn_off(power_ent)`, не нарушая чистое выключение без команд после `mode = "0"`) и в установившемся `Standby` (`age > 15s`).
   - В ветке `tank_replaced` (`humidity_refill_detected`, `humidity_zap_refill_detected`) перед 15-секундной пробой безусловно отправляются `sound/backlight = off` и `child_lock = on`.
2. **Полноценный сторож отклонений (`humidity_mode_enforcer`, `humidity_zap_mode_enforcer`):**
   - Расширен набор триггеров: помимо `mode`, `humidity` и `warm_stream`, отслеживаются `ioniser: off`, `child_lock: off` (`10s`), `backlight: on` (`10s`), `sound: on` (`10s`), ручное изменение `intensity` (`32s`) и неожиданное локальное `on`/`off` (`25s`/`15s`).
   - В условиях добавлена блокировка на время активной 15-секундной пробы заправки (`refill_detected.current == 0`) и удержание `child_lock: on` / `backlight: off` / `sound: off` даже при `Standby` с пустым баком.
3. **Документирование аппаратных ревизий и протокола (`packages/humidity/README.md`, `AI_GUIDELINES.md`):**
   - Зафиксирована сравнительная спецификация `eva` (`devtype 70`, `MCU 1.0.9`) vs `zap` (`devtype 17`, `MCU 1.0.3`), таблица соответствия программ `polaris-mqtt` (0–7), поведение дисплея/подсветки/встроенного гигрометра и известные баги `polaris-mqtt v1.1.6`.

---

## v4.4.1 (2026-11-21)

**Version:** 4.4.1

### Изменения: полностью автоматическая заправка (zero-touch) и защита калибровки от раннего долива

1. **Автоматическое обнаружение заправки без кнопок и ручного включения (`packages/humidity/humidity_control.yaml`):**
   - В `humidity_refill_detected` (`eva`) и `humidity_zap_refill_detected` (`zap`) добавлен основной триггер `tank_replaced` (`binary_sensor.*_water_tank`: `on → off`, работает в т.ч. в Standby), с фильтром `dur >= 10 and (not nr_on or since_empty > 2 or dur > 75)`, исключающим 55-секундный Standby-артефакт Polaris.
   - После установки бака автоматически выполняется **тихая 15-секундная закрытая проба**: включается питание, приглушаются `sound` и `backlight` (если были включены), через `15 с` проверяются `humidifier == 'on'`, нормальный код ошибки (`['00', 'no_error', '02', 'child_lock']`) и `water_tank == 'off'`; при успехе неблокирующе вызывается `script.turn_on` на `script.humidifier_eva_confirm_refill` / `script.humidifier_zap_confirm_refill`, при неудаче питание выключается. Триггер `power_on_20s` сохранён как fallback.
   - В `script.humidity_apply_area_control` переменная `refill_probe_grace` дополнена проверкой `probe_running` (`(state_attr(refill_auto, 'current') | int(0)) > 0`), исключая преждевременное отключение во время 15-секундной пробы.

2. **Защита автокалибровки от раннего снятия бака (`humidifier_calibrate_rates`, `humidifier_zap_calibrate_rates`):**
   - Режим изменён на `mode: restart`, добавлена проверка нормального `trigger.from_state.state` (`['00', 'no_error', '02', 'child_lock']`), снимок счётчиков `h1..h7` захватывается в `variables:` до ожидания, затем выполняется `delay: "00:03:00"` с повторной проверкой `needs_refill == 'on'` и `water_tank == 'off'` перед записью новых ставок и сбросом `full_cycle_valid`.

3. **Очистка UI и текстов уведомлений (`dashboards/humidity_dashboard.yaml`, `packages/humidity/humidity_control.yaml`):**
   - Удалены ручные кнопки **«Refill Tank (100%) / Заправлен»** из `Zap Controls` и `Eva Controls`.
   - Тексты предупреждения `<= 15%` (`humidity_water_low_warning`) и ошибки `low_water` (`humidity_device_error`, `humidity_zap_device_error`) обновлены с указанием на полностью автоматическое возобновление работы.

---

## v4.4.0 (2026-11-21)

**Version:** 4.4.0

### Изменения: надёжные уведомления, честный `0%` пустого бака, верхний долив и защита калибровки

1. **Отказоустойчивые уведомления и ранний порог 15% (`packages/humidity/humidity_control.yaml`):**
   - Добавлен `script.humidity_send_notification`: отправка в Telegram (`continue_on_error: true`) с автоматическим fallback на `notify.mobile_app_zapphone`, если `notify.telegram_bot_8110302509_1002699581686` недоступен.
   - Добавлены helpers `input_boolean.humidifier_low_water_warned` / `humidifier_zap_low_water_warned` (без `initial:`) и автоматизация `humidity_water_low_warning`, отправляющая одно предупреждение за цикл при падении уровня `<= 15%`.
   - Ошибки устройств (`humidity_device_error`, `humidity_zap_device_error`) переведены на `script.humidity_send_notification` и сглаженную влажность `sensor.humidity_*_filtered`.

2. **Исправление ложной «заправки» через 55 с и поддержка верхнего долива:**
   - Удалён сломанный триггер `binary_sensor.*_water_tank` (`on → off` `> 30s`), который ложно срабатывал в Standby при автосбросе ошибки прошивкой Polaris.
   - При `needs_refill == 'on'` или ошибке `low_water`/`01` сенсоры `sensor.humidifier_water_level` и `sensor.humidifier_zap_water_level` принудительно показывают **`0%`** (`remaining_ml: 0`, `hours_remaining: 0.0`).
   - Добавлены скрипты `script.humidifier_confirm_refill`, `script.humidifier_eva_confirm_refill`, `script.humidifier_zap_confirm_refill` (event `humidity_refill_reset`) и кнопки **«Refill Tank (100%) / Заправлен»** в `Zap Controls` и `Eva Controls` ([dashboards/humidity_dashboard.yaml](dashboards/humidity_dashboard.yaml)) + автоматизация синхронизации ручного тумблера `humidity_manual_refill_toggle`.
   - `humidity_refill_detected` / `humidity_zap_refill_detected` переведены на автоподхват по **20 с работы** после физического включения увлажнителя без `low_water`, с 30-секундным окном `refill_probe_grace` в `script.humidity_apply_area_control`.

3. **Защита автокалибровки объёма и сброс завышенных ставок `zap`:**
   - Добавлен флаг полного цикла `input_boolean.humidifier_full_cycle_valid` / `humidifier_zap_full_cycle_valid`: калибровка запускается только при полном цикле в диапазоне расхода `3000..7500 мл`, коэффициент коррекции зажат в `[0.70, 1.35]`, а ставки per-level — в физические границы `[max(35.0, nom*0.60), nom*1.15]` (атрибут `cycle_start` вынесен в момент подтверждения заправки).
   - Сбитые ставки `zap` (`660.9 мл/ч` на 7-й скорости) сброшены к базовой шкале `70..480 мл/ч` (`calibration_cycles: 1`).

---

## v4.3.0 (2026-11-20)

**Version:** 4.3.0

### Изменения: сглаживание сигнала влажности и графиков (по аналогии с CO₂)

1. **Серверная фильтрация (`packages/humidity/humidity_control.yaml`):**
   - Добавлены `platform: filter` (`lowpass`, `time_constant: 18`, `precision: 2` — $\tau \approx 3\text{ мин}$ на 10.5-секундной сетке WirenBoard) и `platform: derivative` (`time_window: 00:05:00`) для обеих комнат:
     - `eva`: `sensor.humidity_filtered`, `sensor.humidity_slope`;
     - `zap`: `sensor.humidity_zap_filtered`, `sensor.humidity_zap_slope`.
   - `sensor.humidity_rate` / `sensor.humidity_zap_rate` переведены на расчёт по сглаженной влажности с диагностическими атрибутами `raw_humidity`, `filtered_humidity`, `derivative_slope_5m`, `filter_type: "ema_lowpass"`.
   - `sensor.humidity_error` / `sensor.humidity_zap_error` триггерятся от сглаженных сенсоров и считают ошибку с точностью `0.01%` (`round(2)`), устраняя ступенчатую «пилу» `±0.1%`, с атрибутами `raw_humidity`, `filtered_humidity`, `target_humidity`, `error_source: "filtered_humidity_error"`.
   - `sensor.recommended_intensity` / `sensor.humidity_zap_recommended_intensity` дополнены атрибутами `raw_humidity` и `filtered_humidity`, `predicted_error` и `overshoot_pct` округлены до `round(2)`.

2. **Дашборд и графики (`dashboards/humidity_dashboard.yaml`, вкладки `Zap` и `Eva`):**
   - На графиках `custom:plotly-graph` для кривых `Humidity` и `Error` применён многоступенчатый конвейер сглаживания: `median` (`window_size: 5` / `3`, `extended: true`) → `exponential_moving_average` (`alpha: 0.06` / `0.10`) → `sliding_window_moving_average` (`window_size: 5`, `extended: true`).
   - Gauge-виджет `Current` в обеих комнатах переведён на `sensor.humidity_zap_filtered` и `sensor.humidity_filtered`.
   - В панели `Zap Diagnostics` и `Eva Diagnostics` добавлены строки `Filtered Humidity`, `Raw Humidity` и `Slope (5m)`.

---

## v4.2.1 – REMOVE CYCLIC DESYNC LOOP, DEVTYPE 17 CHILD LOCK FIX & HARDWARE/PROTOCOL AUDIT (2026-11-20)

**Version:** 4.2.1

### Уточнение причины инцидента в комнате `zap` (`devtype 17`, `mcu_firmware 1.0.3`)

1. **Самоиндуцированное циклическое включение (`Case 2` из `v4.2.0`):**
   - Ветка `desync_needed` (`turn_on` → `3s` → `turn_off` каждые >180 с при перелёте `error <= -1.5%` и росте `rate >= 0.08%/мин`) сама кратковременно включала увлажнитель в режим `auto` (`23:47:57`, `23:51:47`, `23:55:00`, `23:58:16` UTC), вызывая щелчок реле.
2. **Неподдерживаемый `child_lock` для `devtype 17` (`mcu 1.0.3`):**
   - Официальный профиль Polaris `polaris_devices/devices-74B733E1-9775-4715-BDCC-D9A14B298AB2.json` (`type 17`) в отличие от `type 70` (`eva`, `mcu 1.0.9`) **не содержит** `child_lock` и имеет 1 счётчик `expendables` (`4392`, текущее значение `[2417]` = 1 975 ч наработки). Автоматизация принудительно слала `control/child_lock = "true"`, блокируя кнопку `ON/OFF` на панели.
3. **Протокол Syncleo/MQTT и электрика `PUH-9105`:**
   - Во всех реализациях (`samoswall/polaris-mqtt`, `DeKaN/ha-syncleo`, `pysyncleo`, `samoswall/Syncleo`) и в схемах Polaris `type 17`/`type 70` выключение — строго `control/mode = "0"` (`CmdMode(0)`), режима просушки/додува не существует.
   - При `state/mode = "0"` устройство отключает ТЭН тёплого пара (реле 220 В, 80 Вт — слышимый щелчок), но если 24 В силовой ключ (MOSFET/транзистор) ультразвуковой мембраны и вентилятора (30 Вт) пробит или замкнут, генерация тумана продолжается при `Standby` (`mode = 0`).

### Изменения

- Полностью удалена ветка `Case 2` (`turn_on → 3s → turn_off`) из `script.humidity_apply_area_control`.
- При выключении заранее гасятся `warm_stream`/`ioniser`, для питания (`turn_on`/`turn_off`) используются **только `switch.turn_off` / `switch.turn_on`** (`power_ent` — тот же MQTT-топик `control/mode`, без вызова `humidifier.turn_off`, т.к. `polaris/humidifier.py:207` присваивает `self._attr_is_on = "0"` [truthy строка → ложный `'on'` при повторном off]), а для `devtype == '17'` принудительно снимается и больше не включается `child_lock` (также в `humidity_mode_enforcer` и `humidity_zap_mode_enforcer`).
- В `dev_is_on` добавлена проверка `intensity > 0`.

---

## v4.2 – SINGLE-WRITER SHUTDOWN, MCU DESYNC FIX & POLLING POWER RELIABILITY (2026-11-20)

**Version:** 4.2.0

### Причина инцидента («интенсивность 0 / реле щёлкает / пар не останавливается»)

1. **Гонка `mode_enforcer` vs `control_main` + зависание MCU Polaris (`zap`, `devtype 17`, `mcu 1.0.3`):**
   - `switch.*_power` и `humidifier.*` в `polaris-mqtt` используют один топик `control/mode`.
   - При выключении (`control/mode = "0"`) прошивка за ~270 мс до `state/mode = "0"` кратковременно публикует переходный пакет `mode: boost`, `humidity: 30`, `warm_stream: false`.
   - `humidity_zap_mode_enforcer` без дебаунса и проверки `recommended >= 1` ловил этот переход, возвращал `mode: home` и спустя 1.2 с после выключения отправлял в уже выключенный контроллер пачку команд (`control/humidity = 75`, `control/warm_stream = true`, `control/intensity = 0`).
   - Из-за этого отключался ТЭН тёплого пара (щелчок реле, `state/mode = "0"`, `intensity = 0` на дашборде), но ультразвуковая мембрана и вентилятор в MCU зависали включёнными.
2. **Блокировка выключения в `eva` (`needs_refill`) и баг `polaris/switch.py`:**
   - В `script.humidity_apply_area_control` проверка `needs_refill == 'off'` стояла на внешнем `if` и блокировала даже команду выключения (`recommended == 0`).
   - В `polaris/switch.py` строка 850 не распознаёт режимы `"6"` (`home`) и `"7"` (`eco`), из-за чего `switch.*_power` оставался в `unknown`/`off`.

### Изменения

- **Однописательная схема по комнатам (`mode: queued`, `max: 2`):** `humidity_mode_enforcer` / `humidity_zap_mode_enforcer` переведены на триггеры `mode`/`humidity`/`warm_stream` с дебаунсом **`for: 15s`**, обязательным условием `recommended >= 1` + `is_state(hum, 'on')` и вызовом `automation.trigger` для `control_main` (без параллельного запуска отдельного скрипта).
- **`humidifier.*` как первичный источник питания:** во всех сенсорах (`recommended_intensity`, `water_consumed`) и скриптах используется `is_state(hum, 'on') or is_state(pwr, 'on')`; `humidifier.*` добавлен в trigger'ы template-сенсоров.
- **Безусловное чистое выключение и анти-desync (`script.humidity_apply_area_control`):**
  - При `recommended == 0` или неактивном `can_run` отправляется только `switch.turn_off` (+ `wait_template` 5 с и fallback `humidifier.turn_off`), никогда не отправляются `set_mode`, `set_humidity`, `warm_stream` или `intensity: 0`.
  - Если `dev_is_on == false`, `recommended == 0`, `can_run == true`, `error <= -1.5%`, `rate >= 0.08%/min` и `hum_ent` в `off` > 180 с — выполняется одиночный чистый UART-ресинк (`turn_on` → `3s` → `turn_off`).
  - При холодном старте через 3 с перепроверяется `recommended`: если цель уже снизилась до `0`, устройство тут же чисто выключается.
- **`script.humidity_setup_area_devices` и `script.humidity_enforce_area_mode`:** устранено безусловное включение при старте HA, вся работа делегирована в `script.humidity_apply_area_control`.

---

## v4.1 – TIGHT ±0.5% PI+D CONTROLLER & WARM STEAM ENFORCEMENT (2026-11-20)

**Version:** 4.1.0

### Изменения

#### 1. Плотный PI+D-контроллер удержания цели (`±0.5%` вместо `±2.0%`)

- Убрана зона «заморозки» (`current <= 2` внутри `±2.0%`), ограничение разгона `+3` и затянутое отключение до `-4.0%`.
- Введена самоподстраивающаяся интегральная составляющая `integral_level` (`0.0..7.0` по фактическому `dt`):
  - при `error > +1.5%` или `predicted_error > +1.5%` сразу выдаётся **`7`**;
  - при `error ≤ -1.0%` (или `-0.5%` с прогнозом `≤ -1.2%`) сразу выдаётся **`0`**;
  - в полосе `-1.0% .. +1.5%` интенсивность рассчитывается непрерывно (`integral_level + 1.8·error - 3.5·rate`, мин. `1` при `error > +0.5%`).
- Обновлена подсветка зоны нечувствительности на графиках в `dashboards/humidity_dashboard.yaml`: `±0.5%`, fallback `35`.

#### 2. Принудительный возврат `warm_stream` и вспомогательных свитчей

- Устранена потеря ~30% производительности из-за сброса `warm_stream` прошивкой Polaris при цикле питания `off → on`: `script.humidity_apply_area_control`, `script.humidity_enforce_area_mode` и `humidity_*_mode_enforcer` проверяют и включают `warm_stream`, `ioniser`, `child_lock` (и выключают `sound`, `backlight`) на всех активных увлажнителях комнаты.

#### 3. Отмена конденсатного урезания и расширение уставки

- Убрано урезание адаптивной цели по `outdoor_limit` (оставлена комнатная кривая `35–60%`), `min` у `input_number.humidity_target` и `input_number.humidity_zap_target` снижен с `30` до `25%`.

---

## v4.0 – MULTI-ROOM AREA CONTROL & PERSISTENT MANUAL TARGET (2026-11-20)

**Version:** 4.0.0

### Изменения

#### 1. Многокомнатная и много-устройственная архитектура по `area` (`zap` + `eva`)

- Добавлен полный независимый контур управления для комнаты **`zap`** (`sensor.humidity_zap_*`, `automation.humidity_zap_*`, `input_*_zap_*`, `humidifier.humidifier_puh_9105_puh_2709_humidifier_2`), при сохранении всех legacy entity IDs комнаты **`eva`**.
- Логика инициализации, контроля режима `home` и управления питанием/интенсивностью вынесена в переиспользуемые скрипты `script.humidity_setup_area_devices`, `script.humidity_enforce_area_mode`, `script.humidity_apply_area_control`, которые динамически обходят все устройства с доменом `humidifier.*` в `area_devices(area_name)`.
- `Recommended Intensity` (`eva` и `zap`) также определяет текущую активную интенсивность по всем включённым увлажнителям комнаты через `area_devices(...)`.
- Учтена особенность `polaris-mqtt` после рестарта в режиме `home` (`6`): если `switch.*_power` временно `unknown`, состояние питания безопасно определяется по `humidifier.*`.

#### 2. Ручное задание целевой влажности с сохранением между перезагрузками

- Добавлены персистентные helpers **без `initial:`** для обеих комнат:
  - `input_boolean.humidity_adaptive_target` / `input_boolean.humidity_zap_adaptive_target`
  - `input_number.humidity_target` / `input_number.humidity_zap_target` (30–70%)
- Добавлены сенсоры действующей цели `sensor.target_humidity` (`eva`) и `sensor.humidity_zap_target_humidity` (`zap`).
- Добавлена автоматизация `automation.humidity_manual_target_autoswitch`, автоматически выключающая адаптивный режим при ручном вводе значения в дашборде.
- Удалён `initial: 5000` у `input_number.humidifier_tank_capacity_ml` и синхронизирована live стейт-машина заправки (`humidifier_needs_refill` + `humidity_refill_detected` + фикс HA bug `#105309`).

#### 3. Многокомнатный дашборд (`dashboards/humidity_dashboard.yaml`)

- Дашборд разделён на две вкладки в стиле `sidebar` (по аналогии с `ventilation_dashboard.yaml`): **`Zap`** и **`Eva`**.
- Для каждой комнаты добавлена панель `Controls` (переключатель `Auto Control`, `Adaptive Target`, ручной ввод `Target Humidity`, `Effective Target`, `Adaptive Formula`), графики Plotly центрируются на действующей цели и отображают зону нечувствительности `±2.0%`.

---

## v3.1 – PER-LEVEL CONSUMPTION RATE CALIBRATION (2026-01-29)

**Version:** 3.1.0

### Изменения

#### 1. Автокалибровка расхода per-level (вместо калибровки ёмкости бака)

**Было:** калибровалась ёмкость бака (костыль — бак не резиновый, всегда 5000 мл).  
**Стало:** ёмкость бака = константа. Калибруются **ставки расхода** (мл/ч) для каждого из 7 уровней интенсивности.

- `input_text.humidifier_consumption_rates` — JSON с калиброванными ставками
- `input_text.humidifier_level_minutes` — JSON-счётчик минут работы на каждом уровне
- `automation.humidifier_track_level_minutes` — каждую минуту +1 к счётчику текущего уровня
- `automation.humidifier_calibrate_rates` — при опустошении:
  - `correction = capacity / model_consumed`
  - Для уровней с >5 мин работы — EMA-коррекция ставки
  - Уровни с <5 мин — без изменений (мало данных)
  - Сброс счётчика минут

#### 2. Дашборд

- Добавлено отображение оставшегося времени (`hours_remaining`)
- Добавлены строки: калиброванные ставки, минуты текущего цикла, циклы калибровки
- Убрана кнопка Resume Auto Control

#### 3. Сенсоры

- `sensor.humidifier_water_consumed` — использует калиброванные ставки вместо линейной формулы
- `sensor.humidifier_water_level` → `hours_remaining` использует калиброванную ставку текущего уровня
- Атрибут `consumption_rates` добавлен к water level sensor

---

## v9 – ARCHITECTURE REWRITE / HYBRID 2.0 (2025-11-24)

**Commit:** TBD  
**Version:** 2.0.1

> Эта версия представляет собой крупный рефакторинг логики и структуры пакета.  
> Все записи v1–v8 ниже относятся к **ветке 1.x** (старый файл с `input_boolean.humidity_auto_enabled` и другой логикой HYBRID).

### Основные изменения

#### 1. Новая архитектура AUTO/MANUAL (без `input_boolean`)

- **Удалён** helper `input_boolean.humidity_auto_enabled`:
  - Режим AUTO теперь определяется **фактом включённости**:
    - `automation.humidity_control_main` (основная логика управления).
- **MANUAL override**:
  - Любое внешнее изменение управляющих сущностей (пульт, кнопки, интеграции), не инициированное автоматизациями, приводит к:
    - отключению **только** `automation.humidity_control_main`.
  - В MANUAL автомат не трогает ни питание, ни интенсивность; устройство остаётся в том состоянии, в которое перевёл пользователь/интеграция.
- **Возврат в AUTO**:
  - Через `script.humidity_auto_resume`, который включает:
    - `automation.humidity_control_main`,
    - `automation.humidity_manual_override_detector`.
  - При включении `humidity_control_main` срабатывает `humidity_device_setup` (триггер по `automation.humidity_control_main → 'on'`) и повторно конфигурирует устройство.

#### 2. Детектор ручного управления (расширен и защищён)

**Automation:** `humidity_manual_override_detector`

- **Триггеры:** изменения всех ключевых сущностей управления:
  - `number.humidifier_puh_9105_puh_2709_intensity`
  - `switch.humidifier_puh_9105_puh_2709_power`
  - `humidifier.humidifier_puh_9105_puh_2709_humidifier` (attr `mode`)
  - `switch.humidifier_puh_9105_puh_2709_warm_stream`
  - `switch.humidifier_puh_9105_puh_2709_backlight`
  - `switch.humidifier_puh_9105_puh_2709_ioniser`
  - `switch.humidifier_puh_9105_puh_2709_sound`
- **Фильтры:**
  - `from_state != to_state`
  - `from_state.state` не в `['unavailable', 'unknown', 'none']`
  - `state_attr('automation.humidity_control_main', 'current') == 0`
  - `state_attr('automation.humidity_device_setup', 'current') == 0`
- **Действия:**
  - Отключает только `automation.humidity_control_main` → вход в MANUAL.

Автоматизации `humidity_control_main` и `humidity_device_setup` **перед своими service‑вызовами** временно выключают `humidity_manual_override_detector`, чтобы их собственные изменения не считались ручными.

#### 3. Инициализация устройства (device_setup 2.0)

**Automation:** `humidity_device_setup`

- **Триггер:**
  - `state: automation.humidity_control_main → 'on'`.
  - То есть:
    - при первом импорте пакета (после reload automations),
    - при старте HA (`initial_state: true` + включение автоматики),
    - при явном возврате в AUTO (через `script.humidity_auto_resume`).
- **Действия (как и раньше, но с защитой детектора):**
  - Временно отключает `humidity_manual_override_detector`.
  - Включает питание `switch.…power`.
  - Ставит режим `home` на `humidifier.…`.
  - Конфигурирует:
    - звук OFF,
    - подсветку OFF,
    - блокировку ON,
    - ионизатор ON,
    - подогрев ON.
  - Даёт задержку (около 20 секунд) для стабилизации MQTT.
  - Включает детектор обратно.

Таким образом, **device_setup больше не завязан на helper или power→off‑триггеры**, а просто стартует каждый раз при входе в AUTO.

#### 4. HYBRID‑алгоритм мощности (новая версия)

**Sensor:** `sensor.recommended_intensity`

- **Zoned control для крупных положительных ошибок:**

  ```text
  error > 10%          → intensity = 7
  5%   < error ≤ 10%   → intensity = 5
  2%   < error ≤ 5%    → intensity = 3
  ```

- **Incremental control вблизи цели и при переувлажнении:**

  ```text
  1%   < error ≤ 2%    → intensity = current_intensity + 1   (до max 7)
  error < −1%          → intensity = current_intensity − 1   (до min 0)
  −1% ≤ error ≤ 1%     → intensity не меняем (гистерезис)
  ```

- **Антидребезг по времени:**
  - В `humidity_control_main` добавлено условие:
    - менять интенсивность только если с момента `last_changed` у `number.…intensity`
      прошло > 300 секунд (5 минут).

**Итог:**  
исключён runaway `current+1` при малом недоувлажнении,  
+ аппаратно щадящий режим (не дёргаем интенсивность чаще, чем раз в 5 минут, и не дёргаем в зоне ±1%).

#### 5. Template‑сенсоры: переход на modern `template:` в packages

- Исторически пакет использовал legacy `sensor: - platform: template`.
- В v9:
  - template‑сенсоры (`Adaptive Target Humidity`, `Humidity Error`, `Recommended Intensity`)
    определяются через modern формат интеграции `template:`:
    ```yaml
    template:
      - sensor:
          - name: "Adaptive Target Humidity"
            unique_id: adaptive_target_humidity
            ...
    ```
  - Это актуально для современных версий HA и упрощает поддержку.

#### 6. Упрощение reload/restart‑логики

- Старые сценарии `humidity_reload_recovery` + helper больше не используются.
- Включённость AUTO обеспечивается за счёт:
  - `initial_state: true` у `humidity_control_main` и других ключевых автоматизаций,
  - явного вызова `script.humidity_auto_resume` при первом импорте и по желанию.

---

> ⚠️ **Ниже – changelog для ветки 1.x** (старый файл с helper’ом и предыдущей версией HYBRID).  
> Он сохранён для истории и понимания эволюции логики, но часть этих деталей больше не актуальна в 2.0.

---

## v8 - FIX INCREMENTAL RUNAWAY (2025-11-22)

**Commit:** 41b2e7b  
**Version:** 1.1.2

### Исправления:

**Incremental Logic Runaway Bug:**
- **Проблема:** Диапазон 0.5–2% использовал `current + 1` на КАЖДОМ цикле (5 мин).
- За 6 циклов достигал max=7 даже при error=1.5%!
- Intensity НЕ снижалась, когда error падал с 2% к 0.5%.
- **Root cause:** Incremental control применялся ВНЕ deadband (должен быть только для fine‑tuning).

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
error < -2   → current - 1 (incremental decrement)
```

**Rationale:**
- Диапазон 0.5–2% — это всё ещё "средняя ошибка", не зона точной подстройки.
- Incremental control нужен только для fine‑tuning в deadband или при переувлажнении.
- Zoned‑подход для 0.5–2% даёт более предсказуемое поведение.

### Production Testing (2025-11-22 12:49–12:58 UTC)

**Scenario 1: Small error (0.5–1%):**

- ✅ BEFORE: error=1.65% → recommended=7 (WRONG, runaway)
- ✅ AFTER: error=0.5% → recommended=1 (CORRECT)
- ✅ Intensity НЕ растёт, остаётся на 1

**Scenario 2: Approach to target:**

- ✅ Error: 0.5% → 0.2% → 0.1% → 0.0%
- ✅ Recommended: стабильно 1 (нет runaway)
- ✅ Current: 39.84% → 40.25% (плавное приближение к target ≈40.3%)

**Scenario 3: Overshoot (переувлажнение):**

- ✅ Error: −0.1% → −0.4% → −0.5% → −0.6%
- ✅ Recommended: 1 → 0 (при error < −0.5)
- ✅ Power: ON → OFF (12:55:48)
- ✅ Power остаётся OFF (v7‑fix сохранён, нет infinite loop)

### Files Changed (ветка 1.x):

- `packages/humidity/humidity_control.yaml` (строки 4, 137–158)

---

## v7 - ФИНАЛЬНОЕ РЕШЕНИЕ (2025-11-22)

**Commit:** 33fd799

### Исправления:

**1. Logic Deadlock в control_main:**

- **Проблема:** проверка готовности стояла **перед** управлением питанием.
- Если `power=off`, automation останавливалась → управление питанием не выполнялось.
- **Fix:** блок управления питанием перенесён **перед** проверкой готовности.
- Теперь automation сначала решает включить/выключить power, затем проверяет готовность.

**2. device_setup Infinite Loop:**

- **Проблема:** `device_setup` триггерился, когда `power → off for 5 sec`, и всегда включал power обратно, даже при `recommended=0`:
  - цикл: `control_main` выключает → `device_setup` включает → `control_main` выключает…
- **Fix:** добавлена `condition` по `recommended > 0` в начале `action`.
- Теперь `device_setup` останавливается, если `recommended=0`, и не трогает power.

---

## v6 (corrected) - Fix elif order (2025-11-22)

**Commit:** 6959277

### Исправления:

**recommended_intensity elif order bug:**

- **Проблема:** `elif error < -0.5` стояло **перед** `elif error < -2`.
- Ветка `error < -2` была недостижима (dead code).
- **Fix:** порядок изменён — сначала `error < -2`, затем `error < -0.5`.

**Правильная логика:**

```yaml
elif error > 0.5:     # → increment
elif error < -2:      # → decrement (check larger threshold first!)
elif error < -0.5:    # → 0 (then smaller threshold)
else:                 # → maintain
```

### Тесты:

- ✅ При error = −1.9% recommended корректно становится 0.

---

## v6 - Power Management (2025-11-22)

**Commit:** e42bb9a

### Новая функциональность:

**1. Power management в control_main:**

- Добавлена секция «УПРАВЛЕНИЕ ПИТАНИЕМ»:
  - Если `recommended=0` и `power=on` → выключить power.
  - Если `recommended>0` и `power=off` → включить power.

**2. Conditional power в device_setup:**

- `device_setup` начал проверять `recommended` перед включением:
  - Если `recommended>0` → включить power,
  - иначе → выключить power.

**3. Исправлен порядок `elif` в `recommended_intensity`:**

- Логика:  
  `error < −2` → decrement;  
  `−2 < error < −0.5` → 0;  
  `−0.5 < error < 0.5` → maintain.

---

## v5 - Detector Protection для control_main (2025-11-22)

**Commit:** dc269bb

### Исправления:

**control_main MQTT echo protection:**

- **Проблема:** `control_main` вызывают сервисы → MQTT даёт обновления → детектор считает это ручным управлением.
- **Fix:** `control_main` временно выключает детектор:
  1. `automation.turn_off` detector;
  2. `number.set_value` (изменение intensity);
  3. `delay: 20 sec` (ждём MQTT);
  4. `automation.turn_on` detector.

---

## v4 - Увеличен delay до 20 sec (2025-11-22)

**Commit:** 90c7208

### Исправления:

**device_setup delay increase:**

- **Проблема:** MQTT‑updates приходили до 14+ секунд после изменения.
- Delay 5 секунд был недостаточен.
- **Fix:** увеличен delay с 5 до 20 секунд.

---

## v3 - device_setup detector protection (2025-11-22)

**Commit:** eb2c61e

### Исправления:

**device_setup MQTT echo protection:**

- **Проблема:** `device_setup` меняет intensity → MQTT echoes → detector триггерится.
- **Fix:** `device_setup` временно выключает детектор:
  1. `automation.turn_off` detector в начале;
  2. все service calls;
  3. `delay: 5 sec`;
  4. `automation.turn_on` detector в конце.

---

## v2 - Detector logic fixes (2025-11-22)

### Исправления (ветка с helper’ом):

**manual_control_detector:**

- Убран trigger на `power`.
- Добавлен `for: 2 сек`.
- Изменены conditions (проверка `current` вместо `parent_id`).

**reload_recovery:**

- Убрано условие `helper="on"`.
- Добавлено включение helper в `action`.

---

## v1 - KVAZIS Pattern (2025-11-22)

### Изменения:

**Automation naming:**

- Применён kvazis pattern для всех 5 automations:
  - `id`: описательное имя для людей (русский),
  - `alias`: технический код для `entity_id` (английский, `[a-z0-9_]`).

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

Создан файл `AUTOMATION_NAMING.md` после четвёртого раза забывания правил именования automations 🤦

**Документировано:**

- Правила генерации `entity_id` (slugify),
- Доступные поля (`id`, `alias`, `description`),
- Best practices по kvazis‑pattern,
- Проверка `entity_id` через REST API.