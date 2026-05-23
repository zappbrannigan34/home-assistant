# Telegram notifications card

## Current state

Home Assistant Telegram notifications use the HA `telegram_bot` integration directly, not legacy `notify.zapgroup` services.

Telegram Bot API access from Russia requires a proxy. The proxy URL is stored in the HA Telegram config entry:

- file: `/config/.storage/core.config_entries`
- entry domain: `telegram_bot`
- field: `data.proxy_url`

Do not print proxy credentials in logs, docs, commits, or chat summaries.

## Targets

- Normal alerts: `notify.telegram_bot_8110302509_1002699581686`
- Emergency / leak alerts: `notify.telegram_bot_8110302509_1002715853345`

## Message pattern

```yaml
- action: telegram_bot.send_message
  data:
    entity_id: notify.telegram_bot_8110302509_1002699581686
    title: "..."
    message: >
      ...
```

Emergency/leak alerts use the emergency target instead:

```yaml
- action: telegram_bot.send_message
  data:
    entity_id: notify.telegram_bot_8110302509_1002715853345
    title: Утечка
    message: ...
```

## Frigate snapshot pattern

```yaml
- action: telegram_bot.send_photo
  data:
    entity_id: notify.telegram_bot_8110302509_1002699581686
    url: "http://ccab4aaf-frigate:5000/api/events/{{ event_id }}/snapshot.jpg"
    caption: "{{ object_label }} у двери {{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
    parse_mode: plain_text
```

## Verification

After edits:

1. `ha core check`
2. `automation.reload` for YAML-only automation/package changes
3. send muted safe tests (`disable_notification: true`) to both normal and emergency targets
4. verify actual Telegram history contains the messages/photos
5. check fresh HA logs for Telegram `ServiceValidationError`, timeout, network, or setup errors
