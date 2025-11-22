# Home Assistant Automations

Коллекция продвинутых автоматизаций и интеграций для Home Assistant.

## 📦 Автоматизации

| Пакет | Описание | Статус |
|-------|----------|--------|
| [humidity](packages/humidity/) | Адаптивное управление увлажнителем Polaris PUH-9105 | ✅ Работает |

---

## 🌡️ Adaptive Humidity Control

Научно обоснованная автоматизация управления увлажнителем воздуха с адаптацией под температуру помещения и погодные условия.

### Особенности

- **Научный подход**: Формулы основаны на стандартах WHO (40-60%), ASHRAE, ISO 7730
- **Dual-temperature adaptation**: Учет температуры помещения + наружной температуры
- **Гибридное управление**: Комбинация зонного (для больших отклонений) и инкрементального (для точной настройки) методов
- **8 уровней интенсивности**: Полный диапазон управления (0-7)
- **Детекция ручного управления**: Автоматическая остановка при изменениях пользователем
- **Auto-setup устройства**: Автоматическая настройка режима, отключение звука, включение блокировки/ионизации/подогрева
- **Мониторинг ошибок**: Уведомления через Telegram при критических ошибках

### Требования

- **Устройство**: Polaris PUH-9105 (или совместимые модели)
- **Интеграция**: [polaris-mqtt](https://github.com/samoswall/hacs-polaris) (HACS)
- **Датчики**:
  - Датчик температуры помещения
  - Датчик влажности помещения
  - Погодная интеграция (Met.no или другая)
- **Уведомления**: Telegram Bot (notify.zapgroup или измените на свой)

### Установка

1. **Клонировать репозиторий:**
   ```bash
   cd /config  # или ваша директория Home Assistant
   git clone https://github.com/zappbrannigan34/home-assistant.git temp-repo
   cp -r temp-repo/packages/humidity packages/
   rm -rf temp-repo
   ```

2. **Настроить entity IDs в файле `packages/humidity/humidity_control.yaml`:**

   Найдите и замените следующие entity IDs на ваши:

   ```yaml
   # Датчики помещения (строки ~20, 21)
   sensor.sensor_zap_temperature  # → ваш датчик температуры
   sensor.sensor_zap_humidity     # → ваш датчик влажности

   # Погода (строка ~22)
   weather.forecast_home_assistant  # → ваша погодная интеграция

   # Устройство Polaris (по всему файлу)
   number.humidifier_puh_9105_puh_2709_intensity
   switch.humidifier_puh_9105_puh_2709_power
   switch.humidifier_puh_9105_puh_2709_sound
   switch.humidifier_puh_9105_puh_2709_child_lock
   switch.humidifier_puh_9105_puh_2709_ioniser
   switch.humidifier_puh_9105_puh_2709_warm_stream
   switch.humidifier_puh_9105_puh_2709_backlight
   humidifier.humidifier_puh_9105_puh_2709_humidifier
   sensor.humidifier_puh_9105_puh_2709_error
   binary_sensor.humidifier_puh_9105_puh_2709_water_tank

   # Уведомления (строка ~446)
   notify.zapgroup  # → ваш сервис уведомлений
   ```

3. **Проверить конфигурацию:**
   ```yaml
   # В configuration.yaml должно быть:
   homeassistant:
     packages: !include_dir_named packages
   ```

4. **Перезапустить Home Assistant** или перезагрузить конфигурацию YAML

### 🔧 Настройка Entity IDs

#### Таблица всех используемых Entity IDs

| Entity ID | Тип | Назначение | Где найти |
|-----------|-----|-----------|-----------|
| `sensor.sensor_zap_temperature` | Sensor | Температура в комнате для расчета базовой влажности | Developer Tools → States → фильтр "temperature" |
| `sensor.sensor_zap_humidity` | Sensor | Текущая влажность в комнате | Developer Tools → States → фильтр "humidity" |
| `weather.forecast_home_assistant` | Weather | Наружная температура для защиты от конденсата | Settings → Integrations → Weather |
| `number.humidifier_puh_9105_puh_2709_intensity` | Number | Уровень интенсивности увлажнителя (0-7) | polaris-mqtt integration |
| `switch.humidifier_puh_9105_puh_2709_power` | Switch | Питание увлажнителя | polaris-mqtt integration |
| `switch.humidifier_puh_9105_puh_2709_sound` | Switch | Звуковые уведомления (OFF в автоматизации) | polaris-mqtt integration |
| `switch.humidifier_puh_9105_puh_2709_child_lock` | Switch | Блокировка от детей (ON в автоматизации) | polaris-mqtt integration |
| `switch.humidifier_puh_9105_puh_2709_ioniser` | Switch | Ионизация (ON в автоматизации) | polaris-mqtt integration |
| `switch.humidifier_puh_9105_puh_2709_warm_stream` | Switch | Подогрев пара (ON в автоматизации) | polaris-mqtt integration |
| `switch.humidifier_puh_9105_puh_2709_backlight` | Switch | Подсветка дисплея (OFF в автоматизации) | polaris-mqtt integration |
| `humidifier.humidifier_puh_9105_puh_2709_humidifier` | Humidifier | Основной объект увлажнителя (режим работы) | polaris-mqtt integration |
| `sensor.humidifier_puh_9105_puh_2709_error` | Sensor | Код ошибки устройства (01-05) | polaris-mqtt integration |
| `binary_sensor.humidifier_puh_9105_puh_2709_water_tank` | Binary Sensor | Статус бака с водой (ON = снят, OFF = на месте) | polaris-mqtt integration |
| `notify.zapgroup` | Notify Service | Сервис Telegram уведомлений | Settings → Integrations → Telegram |

#### Как найти свои Entity IDs

**Метод 1: Developer Tools (рекомендуется)**
1. Откройте Home Assistant
2. Перейдите в **Developer Tools** → **States**
3. Используйте фильтр для поиска:
   - `temperature` - датчики температуры
   - `humidity` - датчики влажности
   - `weather` - погодные интеграции
   - `humidifier` - увлажнители
4. Скопируйте полные entity_id (например: `sensor.living_room_temperature`)

**Метод 2: Entity Registry (продвинутый)**
```bash
# На машине с Home Assistant
cat /config/.storage/core.entity_registry | jq '.data.entities[] | select(.platform=="polaris_mqtt") | {entity_id, original_name}'
```

**Метод 3: Визуальный поиск**
1. Settings → Devices & Services
2. Найдите вашу интеграцию (polaris-mqtt, Met.no, Telegram)
3. Кликните на устройство → увидите все entities

#### Массовая замена Entity IDs

**Для Linux/macOS:**
```bash
cd /config/packages/humidity

# Резервная копия
cp humidity_control.yaml humidity_control.yaml.backup

# Замена датчиков комнаты
sed -i 's/sensor\.sensor_zap_temperature/sensor.YOUR_TEMPERATURE/g' humidity_control.yaml
sed -i 's/sensor\.sensor_zap_humidity/sensor.YOUR_HUMIDITY/g' humidity_control.yaml

# Замена погоды
sed -i 's/weather\.forecast_home_assistant/weather.YOUR_WEATHER/g' humidity_control.yaml

# Замена notify
sed -i 's/notify\.zapgroup/notify.YOUR_NOTIFY/g' humidity_control.yaml

# Замена Polaris (если ID отличается)
sed -i 's/humidifier_puh_9105_puh_2709/YOUR_HUMIDIFIER_ID/g' humidity_control.yaml
```

**Для Windows (PowerShell):**
```powershell
cd H:\packages\humidity

# Резервная копия
Copy-Item humidity_control.yaml humidity_control.yaml.backup

# Замена
(Get-Content humidity_control.yaml) `
  -replace 'sensor\.sensor_zap_temperature', 'sensor.YOUR_TEMPERATURE' `
  -replace 'sensor\.sensor_zap_humidity', 'sensor.YOUR_HUMIDITY' `
  -replace 'weather\.forecast_home_assistant', 'weather.YOUR_WEATHER' `
  -replace 'notify\.zapgroup', 'notify.YOUR_NOTIFY' `
  -replace 'humidifier_puh_9105_puh_2709', 'YOUR_HUMIDIFIER_ID' |
  Set-Content humidity_control.yaml
```

#### Примеры для разных конфигураций

**Вариант 1: Другая модель Polaris (например PUH-9003)**
```yaml
# Ваши entity IDs будут выглядеть так:
number.humidifier_puh_9003_xxxx_intensity
switch.humidifier_puh_9003_xxxx_power
# и т.д.

# Замените через sed/PowerShell:
# humidifier_puh_9105_puh_2709 → humidifier_puh_9003_xxxx
```

**Вариант 2: Погода от OpenWeatherMap**
```yaml
# Вместо:
weather.forecast_home_assistant

# Используйте:
weather.openweathermap  # или weather.home (по умолчанию от OpenWeatherMap)
```

**Вариант 3: Уведомления через мобильное приложение**
```yaml
# Вместо:
notify.zapgroup

# Используйте:
notify.mobile_app_iphone  # или ваше устройство
```

**Вариант 4: Датчики Aqara вместо Tuya**
```yaml
# Вместо:
sensor.sensor_zap_temperature
sensor.sensor_zap_humidity

# Используйте (примеры):
sensor.aqara_living_room_temperature
sensor.aqara_living_room_humidity
# или
sensor.0x00158d0001a2b3c4_temperature
sensor.0x00158d0001a2b3c4_humidity
```

#### Проверка корректности замены

После замены entity IDs проверьте конфигурацию:

1. **Через Home Assistant UI:**
   - Developer Tools → YAML → Check Configuration
   - Должно показать "Configuration valid"

2. **Через командную строку:**
   ```bash
   ha core check
   ```

3. **Тестовый запуск:**
   - Перезагрузите YAML конфигурацию или перезапустите HA
   - Включите `input_boolean.humidity_auto_enabled`
   - Проверьте логи на ошибки: Settings → System → Logs

### Использование

1. **Включить автоматизацию**: `input_boolean.humidity_auto_enabled` → ON
2. **Мониторинг**:
   - `sensor.adaptive_target_humidity` - текущая целевая влажность
   - `sensor.humidity_error` - отклонение от цели
   - `sensor.recommended_intensity` - рекомендуемая интенсивность
3. **Ручное управление**: При изменении настроек увлажнителя вручную автоматизация отключается автоматически

### Как это работает

**Математическая модель:**

```python
# Базовая целевая влажность (зависит от температуры помещения)
base_indoor = 75 - (1.25 × T_room)

# Ограничение по наружной температуре (защита от конденсата)
outdoor_limit = 50 - max(0, 10 - T_outdoor)

# Итоговая цель (с ограничениями WHO)
target = min(base_indoor, outdoor_limit, 60)
target = max(target, 40)
```

**Логика управления:**

- **Большое отклонение** (>10%): Максимальная интенсивность (7)
- **Среднее отклонение** (5-10%): Высокая интенсивность (5)
- **Малое отклонение** (2-5%): Средняя интенсивность (3)
- **Точная настройка** (0.5-2%): Инкрементальное изменение ±1
- **Переувлажнение** (<-2%): Снижение интенсивности

---

## 📁 Структура репозитория

```
home-assistant/
├── README.md
└── packages/
    └── humidity/
        └── humidity_control.yaml
    # Здесь будут добавляться другие автоматизации
```

## 🔧 Добавление новых автоматизаций

1. Создайте директорию в `packages/` для вашей автоматизации
2. Добавьте YAML файл с конфигурацией
3. Обновите таблицу в README.md
4. Создайте pull request или коммит

## 📄 Лицензия

Этот проект распространяется свободно для личного использования.

## 🤝 Вклад

Автоматизация разработана с помощью [Claude Code](https://claude.com/claude-code).
