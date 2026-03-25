# Home Assistant Automations

Небольшой набор YAML‑пакетов для Home Assistant.
Каждый пакет — изолированная автоматизация, которую можно подключить целиком через `homeassistant: packages`.

---

## 📦 Пакеты

| Пакет | Описание | Документация |
|-------|----------|--------------|
| `humidity` | Управление увлажнителем Polaris PUH‑9105 (адаптивная цель по влажности, режимы AUTO/MANUAL). | [`packages/humidity/README.md`](packages/humidity/README.md) |
| `ventilation` | Trend-based CO2 вентиляция через Drivent V2 (зонный алгоритм, защита ресурса привода). | [`packages/ventilation/README.md`](packages/ventilation/README.md) |

> Новые пакеты добавляются в каталог `packages/` и в эту таблицу.

---

## 🚀 Как использовать пакеты

1. Убедитесь, что в `configuration.yaml` включена поддержка пакетов:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. Скопируйте нужный каталог из `packages/` этого репозитория в свой `/config/packages/`.

3. Откройте `README.md` внутри выбранного пакета и:
   - настройте `entity_id` под свои датчики/устройства;
   - следуйте шагам «Установка»/«Использование».

---

## 📁 Структура репозитория

```text
home-assistant/
├── README.md                   # Общая информация о проекте
├── dashboards/
│   ├── humidity_dashboard.yaml # Дашборд управления увлажнителем
│   └── ventilation_dashboard.yaml # Дашборд вентиляции (CO2 + Drivent)
├── docs/
│   ├── NAMING_CONVENTIONS.md   # Правила нейминга сущностей
│   └── HUMIDITY_CHANGELOG.md   # История изменений пакета humidity
└── packages/
    ├── humidity/
    │   ├── humidity_control.yaml  # YAML‑пакет управления увлажнителем
    │   └── README.md              # Подробная документация по пакету
    └── ventilation/
        ├── ventilation_control.yaml  # YAML‑пакет управления окном по CO2
        └── README.md                 # Подробная документация по пакету
```

---

## 📚 Документация

- Правила нейминга: [`docs/NAMING_CONVENTIONS.md`](docs/NAMING_CONVENTIONS.md)
- Changelog пакета humidity: [`docs/HUMIDITY_CHANGELOG.md`](docs/HUMIDITY_CHANGELOG.md)

---

## 🤝 Вклад

Если хотите добавить новый пакет:

1. Создайте подпапку в `packages/`.
2. Добавьте YAML‑файл(ы) и README пакета.
3. Обновите таблицу выше.
4. Откройте issue или отправьте PR с кратким описанием.