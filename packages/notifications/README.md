# Notifications / Единая система уведомлений и аварийный контур

## Назначение

Централизованный пакет уведомлений для Home Assistant (`hass.zapbrannigan.org`), объединяющий:
1. Единый диспетчер доставки `script.notify_dispatch` с уровнями приоритета **`P0`–`P3`**, тихими часами (`23:00–08:00`) и автоматическим fallback (`Telegram → mobile_app`).
2. Единый отказоустойчивый контур протечек (`P0`): мгновенное перекрытие кранов ХВС/ГВС, проверка их фактического закрытия через 15 секунд, пробой беззвучного режима на телефонах (`alarm_stream`), голосовое оповещение на Яндекс.Станциях, прогрессивные повторы до подтверждения (`Ack`) и автообновление сообщения при высыхании.
3. Сторожевые таймеры (`watchdog`): отвал любого из 7 датчиков протечки (`> 65 мин` при живом Zigbee2MQTT, с учётом ~50-минутного интервала опроса датчиков) и падение самого моста `Zigbee2MQTT`.
4. Ежедневный тихий дайджест обслуживания в `10:00`: разряженные батарейки (с подтягиванием типа элементов из **Battery Notes**), offline-устройства, низкие чернила Canon BX110, пустые баки увлажнителей Polaris и заполнение диска HAOS.

## Каналы доставки

| Уровень | Назначение | Telegram | Mobile App (`zapphone`, `masy`) | Яндекс.Станции |
|---|---|---|---|---|
| `p0_emergency` | Протечка воды, отказ кранов | Срочный (`-1002715853345`), со звуком | Пробой DND (`channel: alarm_stream`, кнопка `Ack`) | Голосовой TTS |
| `p1_urgent` | Отвал датчиков протечки, Z2M offline | Срочный (`-1002715853345`), уважает тихие часы | Fallback при отказе Telegram | — |
| `p2_maintenance` | Дайджест 10:00 (батарейки, расходники, диск) | Обычный (`-1002699581686`), всегда без звука | Fallback при отказе Telegram | — |
| `p3_info` | Информационные события и восстановление | Обычный (`-1002699581686`), уважает тихие часы | Fallback при отказе Telegram | — |

## Создаваемые сущности

| Entity ID | Тип | Назначение |
|---|---|---|
| `input_boolean.notifications_quiet_hours_enabled` | `input_boolean` | Включение расписания тихих часов `23:00–08:00` (без `initial:`) |
| `input_boolean.notifications_dnd_manual` | `input_boolean` | Ручной режим «Не беспокоить» (без `initial:`) |
| `input_boolean.leak_emergency_active` | `input_boolean` | Флаг активной протечки (без `initial:`) |
| `input_boolean.leak_emergency_acknowledged` | `input_boolean` | Флаг подтверждения (`Ack`) пользователем (без `initial:`) |
| `input_boolean.z2m_bridge_offline_warned` | `input_boolean` | Флаг отправленного алерта об отвале Z2M (без `initial:`) |
| `input_number.notifications_battery_low_threshold` | `input_number` | Порог низкого заряда батареек (по умолчанию `20%`) |
| `input_number.notifications_disk_warn_threshold` | `input_number` | Порог предупреждения по диску HAOS (по умолчанию `85%`) |
| `binary_sensor.notifications_quiet_time` | `binary_sensor` | Текущее состояние тихого времени |
| `sensor.notifications_low_batteries` | `sensor` | Счётчик и список разряженных батареек (+ типы из Battery Notes) |
| `sensor.notifications_offline_leak_sensors` | `sensor` | Счётчик и список недоступных датчиков протечки |
| `sensor.notifications_offline_zigbee_ble_devices` | `sensor` | Счётчик и список недоступных батарейных датчиков |
| `script.notify_dispatch` | `script` | Единый роутер уведомлений |
| `script.notify_clear` | `script` | Очистка пушей `clear_notification` и `telegram_bot.edit_message` |

## Карточка для дашборда (Battery State Card + Battery Notes)

```yaml
type: custom:battery-state-card
title: "Батарейки дома"
sort_by_level: asc
collapse: 8
secondary_info: "{attributes.battery_type_and_quantity}"
filter:
  include:
    - name: entity_id
      value: "sensor.*_battery_plus"
  exclude:
    - name: entity_id
      value: "*zapphone*"
    - name: entity_id
      value: "*masy*"
```

## Использование `script.notify_dispatch` из других пакетов

```yaml
- action: script.notify_dispatch
  data:
    level: p2_maintenance
    title: "Заголовок"
    message: "Текст уведомления"
    tag: "my_alert_tag"
```

## Version

1.0.2 — добавлены прямые state-триггеры датчиков протечки в `sensor.notifications_offline_leak_sensors`, убран ложный 3-минутный стартовый обход в `leak_sensors_offline_watchdog`, выдержка отвала увеличена с 10 до 65 минут (с учётом ~50-мин паспортного интервала чек-ина Aqara).  
1.0.1 — исправлена проверка доступности `notify.telegram_bot_*` (штатный `state: unknown` разрешён), добавлен захват `response_variable: tg_resp` для fallback при сетевых ошибках прокси и `| default(...)` для всех опциональных полей.  
1.0.0 — первичная версия единого пакета уведомлений, аварийного контура протечек и дайджеста обслуживания.
