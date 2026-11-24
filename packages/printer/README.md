# Printer notifications / Уведомления принтера Canon MAXIFY BX110

## Назначение

Пакет отслеживает уровень чернил портативного принтера **Canon MAXIFY BX110** в Home Assistant и отправляет однократные уведомления в Telegram при низком остатке (`<= 15%`) и полном окончании (`0%`) с указанием точной модели и артикула картриджа для покупки.

## Устройства и расходники

| Объект | Значение |
|---|---|
| Устройство HA | `Canon BX110 series` (`f8ae1a07b487534def7d84df4f362e28`, интеграция `ipp`) |
| Статус принтера | `sensor.canon_bx110_series` |
| Чёрный картридж | `sensor.canon_bx110_series_black` — **Canon PGI-35 Black** (арт. `1509B001`) |
| Цветной картридж | `sensor.canon_bx110_series_color` — **Canon CLI-36 Color** (арт. `1511B001`) |
| Канал уведомлений | Общий диспетчер `script.notify_dispatch` (`p2_maintenance` для `<= 15%`, `p3_info` для `0%`) и `script.notify_clear` при замене |

## Создаваемые сущности

| Entity ID | Тип | Назначение |
|---|---|---|
| `input_boolean.canon_bx110_black_low_warned` | `input_boolean` | Флаг отправленного предупреждения `<= 15%` для чёрного (без `initial:`) |
| `input_boolean.canon_bx110_color_low_warned` | `input_boolean` | Флаг отправленного предупреждения `<= 15%` для цветного (без `initial:`) |
| `input_boolean.canon_bx110_black_empty_warned` | `input_boolean` | Флаг отправленного алерта `0%` для чёрного (без `initial:`) |
| `input_boolean.canon_bx110_color_empty_warned` | `input_boolean` | Флаг отправленного алерта `0%` для цветного (без `initial:`) |
| `automation.canon_bx110_ink_low_warning` | `automation` | Предупреждение при `<= 15%` (выдержка 1 мин, 1 раз за цикл картриджа) |
| `automation.canon_bx110_ink_empty_alert` | `automation` | Алерт при `0%` (выдержка 1 мин, 1 раз за цикл картриджа) |
| `automation.canon_bx110_ink_refill_reset` | `automation` | Автосброс флагов при замене картриджа (`> 20%`, выдержка 1 мин) |

## Защита от повторов и ночных рестартов

- Все `input_boolean` объявлены **без `initial:`**, чтобы Home Assistant восстанавливал их состояние из `.storage` после ночных перезапусков HAOS (04:25 МСК) и циклов сна/пробуждения принтера по Wi-Fi.
- При ручном запуске `automation.trigger` для `canon_bx110_ink_low_warning` или `canon_bx110_ink_empty_alert` цикл `repeat.for_each` безопасно проверяет оба картриджа (`['black', 'color']`) и отправляет уведомление только по тем, которые перешли порог и ещё не имеют взведённого флага.

## Установка и обновление

1. В серверном `configuration.yaml` включены пакеты (`homeassistant: packages: !include_dir_named packages`).
2. Скопировать `printer_notifications.yaml` в `/config/packages/printer/`.
3. Проверить конфигурацию (`/api/config/core/check_config`).
4. Перезагрузить только домены пакета (полный рестарт HA Core не требуется):
   - `input_boolean.reload`
   - `automation.reload`

## Version

1.1.0 — перевод отправки уведомлений и автоочистки на единый диспетчер `packages/notifications` (`script.notify_dispatch` / `script.notify_clear`), удалён локальный `script.printer_send_notification`.  
1.0.0 — первичная версия пакета уведомлений об уровне чернил Canon MAXIFY BX110.
