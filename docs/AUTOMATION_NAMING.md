# Home Assistant Automation Naming Rules

⚠️ **ВАЖНО**: Эта документация создана потому что я ЧЕТВЁРТЫЙ РАЗ забываю эти правила!

---

## TL;DR - Быстрая справка

**entity_id НЕ задаётся явно в YAML!**
- ❌ НЕТ поля `entity_id:` в automation YAML
- ✅ entity_id генерируется из `alias` через slugify

**Правило:**
```yaml
alias: "Humidity Device Setup"  # → automation.humidity_device_setup
alias: "Инициализация..."       # → automation.initsializatsiia_... (ПЛОХО!)
```

---

## Доступные поля в automation YAML

```yaml
automation:
  - id: "unique_id_123"           # Внутренний ID (НЕ entity_id!)
    alias: "Friendly Name"        # Отображаемое имя → генерирует entity_id
    description: "Description"    # Описание (опционально)
    # НЕТ ПОЛЯ entity_id: !!!
```

**Поля (источник: официальная документация):**
- `id` - уникальный идентификатор для UI editor, debug traces
- `alias` - friendly name, **генерирует entity_id при создании**
- `description` - описание
- `name` - **НЕ существует для automations** (только для других entity types)

---

## Как генерируется entity_id

**Процесс: slugify (python-slugify)**

1. alias преобразуется в lowercase
2. Пробелы → underscores (_)
3. Удаляются спецсимволы
4. Только [a-z0-9_]
5. **Русские/non-Latin символы → транслитерация**

**Примеры:**

| alias | entity_id |
|-------|-----------|
| `"Humidity Device Setup"` | `automation.humidity_device_setup` ✅ |
| `"humidity device setup"` | `automation.humidity_device_setup` ✅ |
| `"Humidity-Device-Setup"` | `automation.humidity_device_setup` ✅ |
| `"Инициализация увлажнителя"` | `automation.initsializatsiia_uvlazhnitelia` ❌ |
| `"Управление интенсивностью"` | `automation.upravlenie_intensivnostiu` ❌ |

---

## Правила и ограничения

### ✅ МОЖНО:
- Английские буквы (любой регистр)
- Пробелы (→ underscores)
- Дефисы (→ underscores)
- Числа

### ❌ НЕЛЬЗЯ контролировать:
- Явно задать entity_id (такого поля нет)
- Изменить entity_id после создания через alias (entity_id создаётся ОДИН РАЗ)

### ⚠️ ВАЖНО:
- entity_id создаётся при ПЕРВОЙ загрузке automation
- Изменение alias НЕ меняет entity_id (после HA 0.105 - Feb 2020)
- Для смены entity_id нужно:
  1. Удалить automation entity из реестра
  2. Изменить alias в YAML
  3. Reload automations

---

## 🎯 KVAZIS PATTERN - BEST PRACTICE

**⚠️ КЛЮЧЕВОЕ ОТКРЫТИЕ:** `alias` в формате [a-z0-9_] становится entity_id **НАПРЯМУЮ** (БЕЗ slugify)!

### Правильное использование полей (по kvazis):

**`id`** - Описательное имя для ЛЮДЕЙ (можно на родном языке):
```yaml
id: "Инициализация увлажнителя"
id: "Управление интенсивностью увлажнителя"
id: "Гостиная, уведомление о проветривании"
```

**`alias`** - Технический код для entity_id (ТОЛЬКО английский, [a-z0-9_]):
```yaml
alias: humidity_device_setup
alias: humidity_control_main
alias: living_air_fresh
```

### 🔑 КЛЮЧЕВОЕ ПРАВИЛО

**alias ПРЯМО ПРИСВАИВАЕТСЯ как entity_id (если уже в формате [a-z0-9_])**

```yaml
alias: humidity_device_setup  → automation.humidity_device_setup
alias: humidity_control_main  → automation.humidity_control_main
```

**НЕТ трансформации! НЕТ slugify! Прямое присвоение!**

### Примеры из kvazis репозитория:

**living_air.yaml:**
```yaml
automation:
  - id: Гостиная, уведомление о проветривании
    alias: living_air_fresh
    # → automation.living_air_fresh
```

**living_humidity.yaml:**
```yaml
automation:
  - id: Увлажнение в гостиной
    alias: 05_gg_hum_auto
    # → automation.05_gg_hum_auto
```

### ✅ ПРАВИЛЬНЫЙ ПАТТЕРН (kvazis):

```yaml
- id: "Инициализация увлажнителя"
  alias: humidity_device_setup
  # id: человеческое описание (русский/любой язык)
  # alias: технический код (английский, становится entity_id)
  # → automation.humidity_device_setup

- id: "Управление интенсивностью увлажнителя"
  alias: humidity_control_main
  # → automation.humidity_control_main
```

### ❌ НЕПРАВИЛЬНЫЙ ПАТТЕРН (старый способ):

```yaml
- id: humidity_device_setup
  alias: "Humidity Device Setup"
  # id: технический код (неправильное использование поля)
  # alias: человеческое описание (будет slugified)
  # → automation.humidity_device_setup (работает, но использование полей задом наперед)

- id: humidity_device_setup
  alias: "Инициализация увлажнителя"
  # alias: русское описание → ТРАНСЛИТЕРАЦИЯ
  # → automation.initsializatsiia_uvlazhnitelia ❌❌❌
```

---

## Best Practices

### ✅ ПРАВИЛЬНО:
```yaml
- id: humidity_device_setup
  alias: "Humidity Device Setup"
  # → automation.humidity_device_setup

- id: humidity_control_main
  alias: "Humidity Intensity Control"
  # → automation.humidity_intensity_control
```

### ❌ НЕПРАВИЛЬНО:
```yaml
- id: humidity_device_setup
  alias: "Инициализация увлажнителя"
  # → automation.initsializatsiia_uvlazhnitelia (транслит!)
```

---

## Использование entity_id в action

**В reload/recovery automations:**

```yaml
action:
  - service: automation.trigger
    target:
      entity_id: automation.humidity_device_setup  # Должен совпадать с generated entity_id!
```

**Проблема:**
- alias: "Инициализация увлажнителя" → automation.initsializatsiia_uvlazhnitelia
- Но в action: `entity_id: automation.humidity_device_setup` ← НЕ СУЩЕСТВУЕТ!
- Результат: Spook warning "Unknown entity"

---

## Проверка entity_id

### Метод 1: Entity Registry
```bash
cat H:/.storage/core.entity_registry | jq '.data.entities[] | select(.entity_id | startswith("automation.")) | {entity_id, unique_id, alias: .original_name}'
```

### Метод 2: Developer Tools
1. Settings → Developer Tools → States
2. Фильтр: `automation.`
3. Найти automation и проверить entity_id

### Метод 3: Grep
```bash
grep -A2 '"entity_id":"automation\.' H:/.storage/core.entity_registry
```

---

## Источники

**Официальная документация:**
- https://www.home-assistant.io/docs/automation/yaml/
- https://developers.home-assistant.io/docs/entity_registry_index/

**Проверено:** 2025-01-22 (WebSearch + WebFetch)

**Slugify процесс:**
- python-slugify
- GitHub Issues: home-assistant/core#75141, home-assistant/frontend#23215

---

## История изменений

- **2025-11-22 (v2)**: Исправлена логика manual_control_detector и reload_recovery:
  - manual_control_detector: убран trigger на power, добавлен for: 2sec, изменены conditions (проверка current вместо parent_id)
  - reload_recovery: убрано condition helper="on", добавлено включение helper в action
  - Теперь helper корректно включается вручную и при reload/restart
- **2025-11-22**: Добавлен раздел "KVAZIS PATTERN". Обновлены все 5 automations в humidity_control.yaml по паттерну kvazis (id для описания, alias для entity_id)
- **2025-01-22**: Создан после ЧЕТВЁРТОГО раза забывания этих правил 🤦
