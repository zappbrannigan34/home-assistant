# Adaptive Humidity Control Automation (Multi-Room)

Автоматическое управление увлажнителями Polaris PUH‑9105 / PUH‑2709 по комнатам (`zap` и `eva`) на основе PD-регулятора с предиктивным демпфированием, поддержкой нескольких устройств в одной `area` и ручным/адаптивным заданием целевой влажности с сохранением между перезагрузками.

---

## 🤔 Зачем это нужно?

Зимой при проветривании и работе отопления влажность в комнате резко падает:

- холодный воздух с улицы содержит очень мало воды в абсолютном выражении;
- при нагреве до комнатной температуры его относительная влажность падает в разы (10–20% и ниже);
- результат – сухость, раздражение слизистых, статическое электричество.

Эта автоматизация решает задачу **поддержания комфортной и безопасной влажности** независимо в каждой комнате:

- с опорой на комнатную температуру (**35–60% RH** в адаптивном режиме, без урезания по уличному конденсату) либо с **ручной целевой уставкой (25–70% RH)**;
- с автоматическим переключением в ручной режим при изменении `input_number` на дашборде и **запоминанием уставки между перезагрузками** (все helpers без `initial:`);
- с привязкой устройств по **Home Assistant Area** (`area_devices('eva')`, `area_devices('zap')`) — в любую комнату можно добавить дополнительные увлажнители без переписывания автоматизаций;
- с активным удержанием **тёплого пара (`warm_stream`)**, ионизатора, блокировки и тихого ночного режима (звук/подсветка выкл) при каждом включении и в `mode_enforcer`;
- с виртуальным датчиком уровня воды и прямой автокалибровкой расхода (`ml/h`) на каждый увлажнитель по фактическим часам работы (`active_hours`);
- без вмешательства в окна и пакет `ventilation`: **CO₂ всегда в приоритете**, автоматизация влажности никогда не ограничивает проветривание.

---

## 🏠 Комнаты и привязка по Area

| Комната (Area) | Сенсоры комнаты | Увлажнители (по `area_id`) | Особенности |
|----------------|-----------------|-----------------------------|-------------|
| **`eva`** | `sensor.sensor_eva_humidity`, `sensor.sensor_eva_temperature` | `humidifier.humidifier_puh_9105_puh_2709_humidifier` и любые новые `humidifier.*` в area `eva` | Сохраняет legacy entity IDs (`sensor.recommended_intensity`, `sensor.humidity_error`, `automation.humidity_control_main` и др.); `devtype 70`, `mcu 1.0.9`, поддерживает `child_lock` |
| **`zap`** | `sensor.sensor_zap_humidity`, `sensor.sensor_zap_temperature` | `humidifier.humidifier_puh_9105_puh_2709_humidifier_2` и любые новые `humidifier.*` в area `zap` | Отдельный контур (`sensor.humidity_zap_*`, `automation.humidity_zap_*`, `input_*_zap_*`); `devtype 17`, `mcu 1.0.3`, также поддерживает `child_lock` (`state/child_lock`) |

### Механизм нескольких устройств в одной комнате (`area_devices`)

Три переиспользуемых скрипта работают с любым количеством увлажнителей внутри указанной `area`:

- `script.humidity_setup_area_devices` — инициализирует каждый увлажнитель комнаты; пропускает устройства в `Standby`, с активным `needs_refill`, `low_water` или снятым баком.
- `script.humidity_enforce_area_mode` — удерживает ручной режим интенсивности (`mode: boost`, скорости `1..7`), `warm_stream: on`, `ioniser: on` и `child_lock/backlight/sound: off` на всех включённых устройствах комнаты (в `Standby` соблюдается 100% радиотишина).
- `script.humidity_apply_area_control` — применяет целевую скорость (`0..7` от `recommended_intensity`) ко всем готовым увлажнителям комнаты независимо (ошибка или снятый бак на одном устройстве не останавливает остальные в той же `area`), соблюдая 0 MQTT-команд в `Standby`/ошибках, 3-секундный settle перед холодным стартом и пробуждение через `switch.turn_on(power)` перед `mode: boost`.
- `automation.humidity_device_offline_watchdog` — отслеживает `binary_sensor.*_available*` и отправляет `p2_maintenance` через `script.notify_dispatch` при офлайне `>= 10 мин` (с автосбросом через `script.notify_clear`).
- `automation.humidity_refill_detected` / `automation.humidity_zap_refill_detected` — запускает неразрушающую объёмную проверку заправки (`REFILL_PENDING >= 120 мл`, с `warm_stream: off`) по снятию/установке бака (`water_tank`: `on → off`, `>= 10 с`), нажатию физической кнопки `ON` на корпусе или кнопке Telegram `/refill_eva` / `/refill_zap` (без `periodic_probe`, с сохранением `water_consumed` при коротком осмотре бака `10–34 с`).

---

## 🎯 Целевая влажность: Адаптивная или Ручная

Для каждой комнаты предусмотрены два персистентных helper'а (без `initial:`):

| Комната | Переключатель адаптивности | Ручная уставка (25–70%) | Действующая цель | Адаптивная формула |
|---------|-----------------------------|--------------------------|------------------|--------------------|
| **`eva`** | `input_boolean.humidity_adaptive_target` | `input_number.humidity_target` | `sensor.target_humidity` | `sensor.adaptive_target_humidity` |
| **`zap`** | `input_boolean.humidity_zap_adaptive_target` | `input_number.humidity_zap_target` | `sensor.humidity_zap_target_humidity` | `sensor.humidity_zap_adaptive_target` |

- При включённом `Adaptive Target` (`on`) используется формула по комнатной температуре `clamp(75 - 1.25 * T_room, 35, 60)` без урезания по уличному конденсату.
- При изменении `Target Humidity` (`input_number`) автоматизация `automation.humidity_manual_target_autoswitch` автоматически выключает `Adaptive Target`, и контроллер переходит на ручную уставку без 1-tick задержки.
- Все `input_boolean`, `input_number`, `input_text` и `input_datetime` объявлены **без `initial:`**, поэтому выбранный режим, ручная уставка, ёмкость бака и калибровочные таблицы полностью сохраняются в `.storage` при перезагрузке HA.

---

## 🧮 Сглаживание сигнала и PI+D-регулятор (`0..7`)

Логика идентична для обеих комнат (`eva` и `zap`):

1. **Сглаживание входной влажности (`lowpass`)**: нативные сенсоры `sensor.sensor_eva_humidity` и `sensor.sensor_zap_humidity` (~10.5 с интервал) фильтруются через `platform: filter` (`lowpass`, `time_constant: 18`, `precision: 2`, эффективная постоянная $\tau \approx 3\text{ мин}$) в `sensor.humidity_filtered` и `sensor.humidity_zap_filtered`, плюс 5-минутная производная `sensor.humidity_slope` / `sensor.humidity_zap_slope` (`platform: derivative`, по аналогии с CO₂-вентиляцией). На графиках `dashboards/humidity_dashboard.yaml` используется многоступенчатое сглаживание (`median` → `exponential_moving_average` → `sliding_window_moving_average`), а карточка gauge `Current` и диагностика показывают сглаженную влажность, raw и slope.
2. **Скорость изменения** (`sensor.humidity_rate` / `sensor.humidity_zap_rate`, `%/мин`): 5-минутное окно по сглаженному сигналу `sensor.humidity_*_filtered` + EMA-сглаживание (`0.4 new + 0.6 old`).
3. **Ошибка регулирования** (`sensor.humidity_error` / `sensor.humidity_zap_error`): `effective_target − filtered_humidity` с точностью `0.01%` (`round(2)`), атрибутами `raw_humidity`, `filtered_humidity`, `error_source: filtered_humidity_error`.
4. **Прогноз ошибки на 5 минут**: `predicted_error = error − (rate × 5)`.
5. **100% HA-управляемый PI+D-закон (`0..7`) с гистерезисом** (`sensor.recommended_intensity` / `sensor.humidity_zap_recommended_intensity`):
   - Устройства работают в **`mode: "boost"` (`"5"`, программа «Интенсивность», `limits.humidity = 0`)** — встроенный гигростат и авто-режимы Polaris полностью отключены, интенсивность `1..7` задаётся только из HA через `number.*_intensity*`:
   - включение из `0` при `error >= +0.30%` или (`error >= +0.15%` и `predicted_error >= +0.45%`);
   - максимальная скорость **`7` (100% мощности)** при `error > +1.5%` или `predicted_error > +1.5%`;
   - плавная модуляция `1..7` около цели (`integral_level + 1.8*error - 3.5*rate`, slew-ограничение `±2` ступени);
   - выключение (`0`) при `error <= -0.45%` или (`error <= -0.15%` и `predicted_error <= -0.55%`).
6. **Фактический выход и учёт воды (`Effective Intensity`, `water_consumed`, `hours_remaining`)**:
   - `sensor.humidifier_effective_intensity` / `sensor.humidifier_zap_effective_intensity` отражают фактический `number.*_intensity*` (`1..7`, или `0` в `Standby`/ошибке/снятом баке).
   - `water_consumed` и `hours_remaining` считаются по поуровневой таблице ставок `input_text.humidifier_*_consumption_rates` (`1..7`, мл/ч) и калибруются при полном цикле по `level_1..7_hours` (`history_stats`).

---

## 💧 Виртуальный датчик уровня воды, уведомления и защита от ложных сбросов

Каждый увлажнитель отслеживается отдельно:

- **Стейт-машина заправки (`needs_refill`)**: при ошибке `low_water`/`01` включается `input_boolean.humidifier_*_needs_refill`, питание выключается, а `Water Level` принудительно показывает **`0%`** (`remaining_ml: 0`, `hours_remaining: 0.0`).
- **Надёжные уведомления (`script.notify_dispatch` / `script.notify_clear`) и порог 15%**: все уведомления об ошибках, предупреждении `<= 15%` (`humidity_water_low_warning`, 1 раз за цикл через `low_water_warned`), заправке и калибровке маршрутизируются через единый диспетчер `packages/notifications` (с учётом тихих часов, приоритетов `P1–P3`, fallback в `mobile_app` и автоочисткой при заправке).
- **Полностью автоматическая фиксация заправки с защитой от конденсата (`REFILL_PENDING >= 120 мл`, без `periodic_probe`)**:
  - После работы с `warm_stream: on` пар на стенках пустого 5-литрового бака конденсируется и вместе с остаточной водой из картриджа-фильтра (`15–50 мл`) стекает в поддон базы, чего хватает на `1–6 мин` работы на скорости `7` и до `40 мин` на скорости `1`. Поэтому слепой таймер `periodic_probe` полностью удалён, а подтверждение заправки привязано к **фактически испарённому объёму `>= 120 мл`**.
  - При установке бака (`tank_replaced` по `water_tank: on → off`, `dur >= 10 с`, с отсечкой дребезга `tank_replaced_fails < 2` и сохранением текущего цикла при коротком осмотре непустого бака `10–34 с`), нажатии физической кнопки `ON` на корпусе, кнопке Telegram `/refill_*` или переключении тумблера UI включается стадия `refill_pending`: увлажнитель работает по PI+D с выключенным ТЭНом (`warm_stream: off`), **не затирая** прежние `water_consumed`/`cycle_start` и **не отправляя** `"✅ Бак заправлен"`. Если это был конденсат, прибор тихо выключается по `low_water` без ложных уведомлений.
  - Как только испарено **`>= 120 мл`**, `humidity_refill_verified_commit` фиксирует `cycle_start = pending_cycle_start` задним числом, выставляет `water_consumed = ~120 мл`, включает `warm_stream: on`, очищает алерт и отправляет одно честное `"✅ Бак заправлен"`.
- **Импульсный классификатор `low_water` и поуровневая автокалибровка (`1..7`, `effective_intensity`):**
  - В `polaris-mqtt` `binary_sensor.*_water_tank` (`state/error/water`) срабатывает от того же поплавка базы: при реальном опустошении на базе авто-сбрасывается `on → off` за `5.15–6.07 с`, а при снятии бака руками держится `'on'` все `55–146 с`. Двухступенчатое ожидание `wait_for_trigger(2s + 10s)` точно отделяет сухой бак (`< 10 с`) от раннего снятия руками (`>= 10 с`, без ложной калибровки и без спама `"💧 Мало воды"`).
  - Сенсоры `history_stats` `level_1..7_hours` считают часы по `sensor.humidifier_*_effective_intensity` (исключая паузы `Standby`). При полном цикле (`3600..7500 мл`, `total_hrs >= 3.0 ч`, согласованность `±20%`) ставки корректируются пропорционально доле объёма каждого уровня с обеспечением монотонности `r[1] <= ... <= r[7]` и инкрементом `cycles + 1` (`max: 10000`).

---

## 🔧 Состав пакета (v4.7.0)

### Архитектурные гарантии и особенности `polaris-mqtt`

- **Single-writer управление:** `humidity_control_main` и `humidity_zap_control_main` работают в режиме `mode: queued` (`max: 2`). `humidity_mode_enforcer` и `humidity_zap_mode_enforcer` отслеживают дрейф `mode` (`boost`), `intensity` (`32s`), `warm_stream`, `ioniser`, `child_lock: on` (`10s`), `backlight: on` (`10s`), `sound: on` (`10s`) и неожиданное локальное включение/выключение (`12s`/`15s`) — с блокировкой на время пробы заправки — и лишь триггерят `control_main`, исключая гонки при выключении.
- **Безусловное чистое выключение:** при `recommended == 0` или блокировке (`needs_refill` / `water_tank` / ошибка, кроме активного `refill_probe_grace`) при отсутствии аварии заранее выключаются включённые `warm_stream`/`ioniser`/`backlight`/`sound`/`child_lock`, затем отправляется только `switch.turn_off` (`power_ent` на `control/mode = "0"`, с `wait_template` 5 с и повтором `switch.turn_off`), без циклического `turn_on/turn_off`, без команд после `power_ent` и без `humidifier.turn_off` (в `polaris/humidifier.py:207` `self._attr_is_on = "0"` — truthy строка, оставляющая сущность в `'on'`).
- **Режим `mode: "boost"` (`5`, 24/7, `child_lock: off`):** на **обеих** аппаратных ревизиях (`devtype 70` и `devtype 17`) во время работы удерживается ручной режим интенсивности **`mode: "boost"` (`control/mode = "5"`, `limits.humidity = 0`)** с `number.*_intensity* = 1..7`, `warm_stream: on`, `ioniser: on`, `backlight: off`, `sound: off`, **`child_lock: off`**. Блокировка от детей принудительно выключена, так как по инструкции (стр. 21) при активном `child_lock` кнопка `ON/OFF` мигает каждые 5 с в темноте, а на `devtype 17` (`zap`) код `02` (`child_lock`) может маскировать `01` (`low_water`). В `Standby` (`mode == 0`) и при ошибках соблюдается **100% радиотишина (`0` команд)**.
- **`humidifier.*` и `switch.*_power` в `boost` (`"5"`):** в `polaris/switch.py:850` режим `"5"` распознаётся как `'on'` (в отличие от `"6"`/`"7"`), поэтому оба объекта синхронно отражают питание.

---

## 🔬 Аппаратные ревизии Polaris PUH‑9105 (`devtype 70` vs `devtype 17`), индикация дисплея и протокол Syncleo/MQTT

### 1. Сравнение двух ревизий в доме

| Параметр в MQTT (`polaris/<mac>/state/*`) | `eva` (`781c3c873296`) | `zap` (`98f4abda1783`) |
|---|---|---|
| Модель в `polaris-mqtt` | `PUH-9105_PUH-2709` | `PUH-9105_PUH-2709` |
| Аппаратный тип (`state/devtype`) | **`70`** (новая ревизия) | **`17`** (ранняя ревизия) |
| Прошивка контроллера дисплея (`state/mcu_firmware`) | **`1.0.9`** (чип `0700...`) | **`1.0.3`** (чип `0200...`) |
| Прошивка Wi‑Fi модуля (`state/firmware`) | `2.19` | `2.19` |
| Счётчики обслуживания (`state/expendables`) | `[4392, 168]` (фильтр + `CLEAN` 168 ч базы) | `[4392]` (только фильтр, без `CLEAN`) |
| Блокировка кнопок (`CMD_CHILD_LOCK 0x1E`) | Поддерживается прошивкой и схемой | Поддерживается прошивкой (`state/child_lock = true`), но отсутствует в JSON-схеме |
| Управление `backlight` (`CMD_BACKLIGHT 0x1C`) | По паспорту управляет **только подсветкой бака** (остаётся `off` 24/7) | По паспорту управляет **только подсветкой бака** (остаётся `off` 24/7) |

### 2. Соответствие режимов `polaris-mqtt` заводским программам `PUH-9105`

| Код `state/mode` | Режим в `polaris-mqtt` | Программа в прошивке Polaris | `limits.speed` (интенсивность) | `limits.humidity` (цель) | Примечание |
|---|---|---|---|---|---|
| `"0"` | `off` | Выкл (`Standby`) | `0` | `0` | Безусловное отключение; экран полностью погашен; нельзя слать команды после `"0"` |
| `"1"` | `auto` | Авто | `0` (авто) | `0` | Таблица от температуры; экран светится (через 15 с лишь приглушается) |
| `"2"` | `comfort` | Здоровье | `0` (авто) | `0` | Цель `60%`; экран светится |
| `"3"` | `baby` | Детский | `0` (авто) | `0` | Цель `55%`; экран светится |
| **`"4"`** | **`sleep`** | **Ночной** | `0` (авто) | `0` | **Рабочий режим пакета (v4.4.3):** через 15 с тишины полностью гасит экран и 6 кнопок; разрешает `warm_stream: on`, `ioniser: on`, `child_lock: on` |
| `"5"` | `boost` | Интенсивность | `1..7` | `0` | Кратковременно мелькает в переходном пакете за ~270 мс до `mode = "0"` |
| `"6"` | `home` | Ручной (Custom) | `1..7` | `30..75` | Экран и все 6 кнопок остаются видны ночью в режиме приглушения (не используется с v4.4.3) |
| `"7"` | `eco` | Влажность | `0` (авто) | `30..75` | Автоматический гигростат; экран светится |

### 3. Особенности дисплея, датчиков и протокола

- **Почему в `mode = 6` экран оставался светящимся, а `backlight = false` не помогал:** по официальной инструкции `PUH-9105` (стр. 9, 11, 20) `backlight` (`0x1C`) управляет **только подсветкой водяного бака**. В режимах `1, 2, 3, 5, 6, 7` через 15 с неактивности экран лишь приглушается («энергосбережение»), но продолжает отображать влажность/температуру, шкалу и 6 сенсорных кнопок. **Единственный рабочий режим с полным гашением дисплея и кнопок через 15 с** — `mode = 4` (`sleep`, стр. 20 рис. 2); при этом любая входящая MQTT-команда сбрасывает 15-секундный таймер затухания, поэтому пакет в установившемся `sleep` не шлёт повторных команд. Живой проверкой подтверждено, что `number.set_value(intensity)` в `sleep` игнорируется (`state/intensity = 0`), а `warm_stream`, `ioniser` и `child_lock` работают штатно на обеих ревизиях (`devtype 70` и `devtype 17`).
- **Показания встроенного гигрометра на корпусе:** экран прибора всегда отображает собственный датчик в базе, который из-за нагрева от платы и ТЭНа тёплого пара (`27–28 °C`) занижает относительную влажность на `7–15%` по сравнению с настенными датчиками WirenBoard (`eva`: `35%` vs `41.8%`; `zap`: `21%` vs `36.3%`). Внутренний автопорог `sleep` при `27–28 °C` составляет `43–45%`, поэтому пока HA держит прибор включённым для достижения комнатной цели (`35–48%`), увлажнитель стабильно генерирует пар.
- **Особенности прошивки при заправке и выключении:** при переходе в `Standby` с пустым баком прошивка может через ~55 с сама сбросить `water_tank`, а при включении с сухим баком выставляет `low_water` примерно через `4.8 с` — оба факта учтены в фильтре `tank_replaced` и 15-секундной закрытой пробе.
- **Известные баги `samoswall/polaris-mqtt v1.1.6`:**
  1. `humidifier.py:207`: `turn_off()` присваивает `self._attr_is_on = "0"` (truthy строка → залипание `'on'`);
  2. `switch.py:850`: проверка активных режимов пропускает `"6"` (`home`) и `"7"` (`eco`), из-за чего `switch.*_power` может показывать `'off'`/`'unknown'` при работающем приборе;
  3. `sensor.py:661`: `clean_retain` индексирует второй элемент `expendables.split(",")[1]` и падает с `IndexError` на `devtype 17` (`[2384]`), оставляя сенсор `clean_retain_2` в `'unknown'`.

### Scripts

- `script.humidity_auto_resume` / `script.humidity_zap_auto_resume`
- `script.humidifier_confirm_refill` (`humidifier_eva_confirm_refill` / `humidifier_zap_confirm_refill`)
- `script.humidity_setup_area_devices`
- `script.humidity_enforce_area_mode`
- `script.humidity_apply_area_control`

### Automations

| ID / Alias | Назначение |
|------------|-----------|
| `humidity_manual_target_autoswitch` | Автовыключение адаптивного режима при вводе ручной уставки |
| `humidity_water_low_warning` | Раннее предупреждение при расчётном остатке воды `<= 15%` (1 раз за цикл) |
| `humidity_manual_refill_toggle` | Синхронизация `cycle_start` и `full_cycle_valid` при ручном сбросе `Needs Refill` |
| `humidity_device_setup` / `humidity_zap_device_setup` | Инициализация всех увлажнителей в `eva` / `zap` по `area` через `apply_area_control` |
| `humidity_mode_enforcer` / `humidity_zap_mode_enforcer` | Сторож HA-only управления, `home`, `backlight/sound: off`, `child_lock: on`, `intensity` и питания → триггерит `control_main` |
| `humidity_control_main` / `humidity_zap_control_main` | Очередное (`queued`) применение `recommended_intensity` ко всем устройствам `area` |
| `humidifier_calibrate_rates` / `humidifier_zap_calibrate_rates` | Взвешенная поуровневая автокалибровка при реальном опустошении (`water_tank` авто-сброс `< 10 с`, монотонность `r[1]..r[7]`, `cycles + 1`) |
| `humidity_refill_detected` / `humidity_zap_refill_detected` | Запуск объёмной проверки заправки (`REFILL_PENDING`) по `tank_replaced`, кнопке `ON` и `/refill_*` (без `periodic_probe`) |
| `humidity_refill_verified_commit` | Фиксация полной заправки после фактического испарения `>= 120 мл` воды с учётом стартового расхода |
| `humidity_device_error` / `humidity_zap_device_error` | Разделение реального `low_water` и снятия бака руками, тихий отбой конденсата и уведомления по сервисным кодам |

---

## 🚀 Установка и обновление

1. Скопируйте `packages/humidity/humidity_control.yaml` в `/config/packages/humidity/humidity_control.yaml` и `dashboards/humidity_dashboard.yaml` в `/config/dashboards/humidity_dashboard.yaml`.
2. Убедитесь, что увлажнители в HA привязаны к нужным Area (`eva`, `zap`).
3. Проверьте конфигурацию (`check_config`) и выполните **перезагрузку Home Assistant** при добавлении новых helpers/template-сенсоров.
4. Для новых helpers без `initial:` задайте стартовые значения один раз через UI или `input_number.set_value` — далее они сохраняются автоматически.

---

**Devices:** Polaris PUH‑9105 / PUH‑2709 (`eva`, `zap`)  
**Integration:** `polaris-mqtt` (MQTT)  
**Version пакета:** 4.7.0
