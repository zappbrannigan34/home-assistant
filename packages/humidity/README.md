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

- `script.humidity_setup_area_devices` — инициализирует каждый увлажнитель комнаты (`mode: sleep`, `warm_stream: on`, `ioniser: on`, `child_lock: on`, `sound: off`, `backlight: off`); пропускает устройства в `Standby`, с активным `needs_refill`, `low_water` или снятым баком.
- `script.humidity_enforce_area_mode` — удерживает ночной режим (`mode: sleep`) и блокировку на всех включённых устройствах комнаты (в `Standby` соблюдается 100% радиотишина).
- `script.humidity_apply_area_control` — применяет включение/выключение (`recommended_intensity`) ко всем готовым увлажнителям комнаты независимо (ошибка или снятый бак на одном устройстве не останавливает остальные в той же `area`), соблюдая 0 MQTT-команд в `Standby`/ошибках, 3-секундный settle перед холодным стартом и пробуждение через `switch.turn_on(power)` перед `mode: sleep`.
- `automation.humidity_device_offline_watchdog` — отслеживает `binary_sensor.*_available*` и отправляет `p2_maintenance` через `script.notify_dispatch` при офлайне `>= 10 мин` (с автосбросом через `script.notify_clear`).
- `automation.humidity_refill_detected` / `automation.humidity_zap_refill_detected` — обнаруживает заправку по снятию/установке бака (`water_tank`: `on → off`), по дневной автопроверке `09:00–22:00` (`periodic_probe`, на случай верхнего долива без снятия бака или молчания датчика `MCU 1.0.9` в `Standby`) или по кнопке Telegram `/refill_eva` / `/refill_zap`, с 50-секундной пробой `wait_for_trigger` (учитывает `43.1s` задержку поплавка `MCU 1.0.9`).

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

## 🧮 Сглаживание сигнала и предиктивный ON/OFF-регулятор (`0%` / `100%`)

Логика идентична для обеих комнат (`eva` и `zap`):

1. **Сглаживание входной влажности (`lowpass`)**: нативные сенсоры `sensor.sensor_eva_humidity` и `sensor.sensor_zap_humidity` (~10.5 с интервал) фильтруются через `platform: filter` (`lowpass`, `time_constant: 18`, `precision: 2`, эффективная постоянная $\tau \approx 3\text{ мин}$) в `sensor.humidity_filtered` и `sensor.humidity_zap_filtered`, плюс 5-минутная производная `sensor.humidity_slope` / `sensor.humidity_zap_slope` (`platform: derivative`, по аналогии с CO₂-вентиляцией). На графиках `dashboards/humidity_dashboard.yaml` используется многоступенчатое сглаживание (`median` → `exponential_moving_average` → `sliding_window_moving_average`), а карточка gauge `Current` и диагностика показывают сглаженную влажность, raw и slope.
2. **Скорость изменения** (`sensor.humidity_rate` / `sensor.humidity_zap_rate`, `%/мин`): 5-минутное окно по сглаженному сигналу `sensor.humidity_*_filtered` + EMA-сглаживание (`0.4 new + 0.6 old`).
3. **Ошибка регулирования** (`sensor.humidity_error` / `sensor.humidity_zap_error`): `effective_target − filtered_humidity` с точностью `0.01%` (`round(2)`), атрибутами `raw_humidity`, `filtered_humidity`, `error_source: filtered_humidity_error`.
4. **Прогноз ошибки на 5 минут**: `predicted_error = error − (rate × 5)`.
5. **Честный двухпозиционный закон (`0%` / `100%`) с гистерезисом** (`sensor.recommended_intensity` / `sensor.humidity_zap_recommended_intensity`):
   - В полностью тёмном режиме `mode: "sleep"` (`4`) аппаратные скорости `1..7` недоступны (устройство либо выключено в `Standby`, либо испаряет в `sleep + warm_stream` с полной производительностью `~500–550 мл/ч`), поэтому регулятор выдаёт честный запрос `0%` или `100%` (без фиктивной шкалы `0..7` и без интегрального перелёта):
   - когда контур выключен (`0%`) → включение (`100%`) при `error >= +0.30%` или (`error >= +0.10%` и `predicted_error >= +0.50%`);
   - когда контур включён (`100%`) → выключение (`0%`) при `error <= -0.35%` или (`error <= 0.0%` и `predicted_error <= -0.35%`).
6. **Фактический выход и учёт воды (`Mist Output`, `water_consumed`, `hours_remaining`)**:
   - `sensor.humidifier_effective_intensity` / `sensor.humidifier_zap_effective_intensity` показывают честный процент работы **`100%`** (активно испаряет в `sleep`) или **`0%`** (`Standby`/ошибка/снят бак).
   - `water_consumed` и `hours_remaining` считаются напрямую по откалиброванной ставке `input_number.humidifier_*_evaporation_rate_ml_h` (`eva: ~500 мл/ч`, `zap: ~530 мл/ч`).

---

## 💧 Виртуальный датчик уровня воды, уведомления и защита от ложных сбросов

Каждый увлажнитель отслеживается отдельно:

- **Стейт-машина заправки (`needs_refill`)**: при ошибке `low_water`/`01` включается `input_boolean.humidifier_*_needs_refill`, питание выключается, а `Water Level` принудительно показывает **`0%`** (`remaining_ml: 0`, `hours_remaining: 0.0`).
- **Надёжные уведомления (`script.notify_dispatch` / `script.notify_clear`) и порог 15%**: все уведомления об ошибках, предупреждении `<= 15%` (`humidity_water_low_warning`, 1 раз за цикл через `low_water_warned`), заправке и калибровке маршрутизируются через единый диспетчер `packages/notifications` (с учётом тихих часов, приоритетов `P1–P3`, fallback в `mobile_app` и автоочисткой при заправке).
- **Полностью автоматическая фиксация заправки (`tank_replaced` + 15 с проба, без ручных кнопок)**: Polaris PUH‑9105 использует съёмный бак (без верхнего долива), поэтому достаточно достать бак, залить воду и поставить его на место (даже когда прибор выключен / в Standby):
  1. **Детектор снятия и установки бака (`tank_replaced` в `humidity_*_refill_detected`)**: отслеживает переход `binary_sensor.*_water_tank` (`on → off`), отсеивая ложный ~55-секундный сброс в Standby фильтром `dur >= 10 and (not nr_on or since_empty > 2 or dur > 75)`. Далее автоматика запускает **тихую 15-секундную пробу** (`switch.turn_on` с приглушением `sound`/`backlight`, защищено от отключения через `probe_running` в `refill_probe_grace` внутри `humidity_apply_area_control`). Если за 15 с прибор работает и ошибка остаётся в норме (`00`/`no_error`/`02`/`child_lock`) при закрытом баке (`water_tank == 'off'`), неблокирующе запускается `script.humidifier_*_confirm_refill` (`100%`, `needs_refill → off`, уведомление, `humidity_control_main`); если бак пуст — питание сразу выключается;
  2. **Резервный канал (`power_on_20s`)**: если `needs_refill == 'on'` и прибор проработал 20 с без `low_water`, также срабатывает `script.humidifier_*_confirm_refill`;
  3. Скрипты `script.humidifier_eva_confirm_refill` / `script.humidifier_zap_confirm_refill` и тумблер `Needs Refill` (`humidity_manual_refill_toggle`) сохранены как сервисный резерв (ручные кнопки «Заправлен» убраны с дашборда).
- **Прямая автокалибровка расхода по фактическим часам работы (`active_hours`):** `history_stats` (`sensor.humidifier_active_hours` / `sensor.humidifier_zap_active_hours`) считает часы работы `humidifier == 'on'` от подтверждённого старта цикла (`cycle_start`). Автоматизации `humidifier_*_calibrate_rates` работают в `mode: restart`, проверяют нормальное предыдущее состояние (`trigger.from_state`), выдерживают **3 минуты задержки** и перепроверяют `needs_refill == 'on'` и `water_tank == 'off'`. При полном цикле (`2500..7500 мл`, `active_hours >= 4.0`) фактическая скорость рассчитывается как `capacity / active_hours`, сглаживается с предыдущей ставкой (`[0.75..1.35]`, границы `300..620 мл/ч`) и сохраняется в `input_number.humidifier_*_evaporation_rate_ml_h`.

---

## 🔧 Состав пакета (v4.4.3)

### Архитектурные гарантии и особенности `polaris-mqtt`

- **Single-writer управление:** `humidity_control_main` и `humidity_zap_control_main` работают в режиме `mode: queued` (`max: 2`). `humidity_mode_enforcer` и `humidity_zap_mode_enforcer` отслеживают дрейф `mode` (`sleep`), `warm_stream`, `ioniser`, `child_lock` (`10s`), `backlight` (`10s`), `sound` (`10s`) и неожиданное локальное включение/выключение (`25s`/`15s`) — с блокировкой на время пробы заправки — и лишь триггерят `control_main`, исключая гонки при выключении.
- **Безусловное чистое выключение:** при `recommended == 0` или блокировке (`needs_refill` / `water_tank` / ошибка, кроме активного `refill_probe_grace`) заранее подтверждается `child_lock: on` и выключаются включённые `warm_stream`/`ioniser`/`backlight`/`sound`, затем отправляется только `switch.turn_off` (`power_ent` на `control/mode = "0"`, с `wait_template` 5 с и повтором `switch.turn_off`), без циклического `turn_on/turn_off`, без команд после `power_ent` и без `humidifier.turn_off` (в `polaris/humidifier.py:207` `self._attr_is_on = "0"` — truthy строка, оставляющая сущность в `'on'`).
- **Режим `Full Blackout & HA-Only Lockdown` (`mode: "sleep"`, `4`, 24/7):** на **обеих** аппаратных ревизиях (`devtype 70` и `devtype 17`) во время работы удерживается ночной режим **`mode: "sleep"` (`control/mode = "4"`)** с `warm_stream: on`, `ioniser: on`, `child_lock: on`, `backlight: off`, `sound: off`, без отправки лишних `set_humidity` и `number.set_value(intensity)` (любая MQTT-команда будит экран в `sleep` ещё на 15 с). Через 15 с тишины цифровой экран и все 6 сенсорных кнопок полностью гаснут. В `Standby` (`age > 15s`, в т.ч. при пустом баке) удерживаются `child_lock: on`, `backlight: off`, `sound: off`.
- **`humidifier.*` и `switch.*_power` в `sleep` (`"4"`):** в `polaris/switch.py:850` режим `"4"` распознаётся как `'on'` (в отличие от `"6"`/`"7"`), поэтому оба объекта синхронно отражают питание.

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
| `humidifier_calibrate_rates` / `humidifier_zap_calibrate_rates` | Защищённая full-cycle автокалибровка расхода per-level при `low_water` (с 3-мин проверкой раннего долива) |
| `humidity_refill_detected` / `humidity_zap_refill_detected` | Полностью автоматическое обнаружение заправки по снятию/установке бака + тихая проба 15 с (и fallback 20 с работы) |
| `humidity_device_error` / `humidity_zap_device_error` | Обработка `low_water` и ошибок устройств + уведомление с fallback |

---

## 🚀 Установка и обновление

1. Скопируйте `packages/humidity/humidity_control.yaml` в `/config/packages/humidity/humidity_control.yaml` и `dashboards/humidity_dashboard.yaml` в `/config/dashboards/humidity_dashboard.yaml`.
2. Убедитесь, что увлажнители в HA привязаны к нужным Area (`eva`, `zap`).
3. Проверьте конфигурацию (`check_config`) и выполните **перезагрузку Home Assistant** при добавлении новых helpers/template-сенсоров.
4. Для новых helpers без `initial:` задайте стартовые значения один раз через UI или `input_number.set_value` — далее они сохраняются автоматически.

---

**Devices:** Polaris PUH‑9105 / PUH‑2709 (`eva`, `zap`)  
**Integration:** `polaris-mqtt` (MQTT)  
**Version пакета:** 4.5.0
