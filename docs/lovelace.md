# Lovelace для TION 4S

В этом разделе собраны карточки для управления TION 4S через `HA-tion`.

Основная сущность интеграции — `climate`. Поэтому для базового интерфейса достаточно штатных карточек Home Assistant. Пользовательские карточки нужны только для более компактного или визуально насыщенного интерфейса.

Готовый YAML находится в [`examples/lovelace/tion-dashboard.yaml`](../examples/lovelace/tion-dashboard.yaml).

---

## 1. Сначала найдите свои сущности

В примерах используются:

```yaml
climate.tion_breezer
sensor.living_room_co2
sensor.outdoor_temperature
binary_sensor.living_room_window
script.tion_boost_20_minutes
```

Замените их на реальные entity ID.

Проверить доступные атрибуты климатической сущности можно здесь:

```text
Инструменты разработчика → Состояния → climate.tion_breezer
```

Особенно важны:

- `hvac_modes`;
- `fan_modes`;
- `preset_modes`;
- `temperature`;
- `current_temperature`.

---

## 2. Самый простой вариант: Thermostat card

```yaml
type: thermostat
entity: climate.tion_breezer
name: TION 4S
features:
  - type: climate-hvac-modes
    hvac_modes:
      - fan_only
      - heat
      - "off"
```

Плюсы:

- штатная карточка;
- управление температурой;
- понятные режимы;
- не требует HACS.

Минус: занимает много места и не всегда удобно показывает скорости вентилятора.

---

## 3. Рекомендуемый штатный вариант: Tile card

```yaml
type: tile
entity: climate.tion_breezer
name: TION 4S
icon: mdi:air-filter
state_content:
  - state
  - temperature
features_position: bottom
features:
  - type: climate-hvac-modes
    hvac_modes:
      - fan_only
      - heat
      - "off"
  - type: climate-fan-modes
    style: icons
    fan_modes:
      - "1"
      - "2"
      - "3"
      - "4"
      - "5"
      - "6"
  - type: target-temperature
```

Если интерфейс не показывает числовые скорости как иконки, замените:

```yaml
style: icons
```

на:

```yaml
style: dropdown
```

Это наиболее совместимый вариант.

---

## 4. Компактная карточка управления

```yaml
type: vertical-stack
cards:
  - type: tile
    entity: climate.tion_breezer
    name: TION 4S
    icon: mdi:air-filter
    features:
      - type: climate-hvac-modes
        hvac_modes:
          - fan_only
          - heat
          - "off"
      - type: climate-fan-modes
        style: dropdown

  - type: grid
    columns: 3
    square: false
    cards:
      - type: tile
        entity: sensor.living_room_co2
        name: CO₂
        icon: mdi:molecule-co2
        vertical: true

      - type: tile
        entity: sensor.outdoor_temperature
        name: Улица
        icon: mdi:thermometer
        vertical: true

      - type: tile
        entity: binary_sensor.living_room_window
        name: Окно
        icon: mdi:window-closed-variant
        vertical: true
```

Такой блок хорошо подходит для мобильного дашборда.

---

## 5. Кнопка Boost

Если создан script `script.tion_boost_20_minutes`:

```yaml
type: button
entity: script.tion_boost_20_minutes
name: Проветрить 20 минут
icon: mdi:fan-plus
tap_action:
  action: perform-action
  perform_action: script.turn_on
  target:
    entity_id: script.tion_boost_20_minutes
hold_action:
  action: more-info
```

Можно добавить кнопку в сетку рядом с управлением TION.

---

## 6. Условное предупреждение об открытом окне

```yaml
type: conditional
conditions:
  - entity: binary_sensor.living_room_window
    state: "on"
card:
  type: markdown
  content: |
    ## Окно открыто
    TION остановлен или должен быть переведён в паузу.
```

Более компактный вариант:

```yaml
type: conditional
conditions:
  - entity: binary_sensor.living_room_window
    state: "on"
card:
  type: tile
  entity: binary_sensor.living_room_window
  name: Закройте окно
  icon: mdi:window-open-variant
  color: red
```

---

## 7. Предупреждение о недоступности TION

```yaml
type: conditional
conditions:
  - entity: climate.tion_breezer
    state: unavailable
card:
  type: markdown
  content: |
    ## TION недоступен
    Проверьте Bluetooth, питание, TION Remote и уровень сигнала.
```

---

## 8. Карточка качества воздуха

### Только штатные карточки

```yaml
type: vertical-stack
cards:
  - type: gauge
    entity: sensor.living_room_co2
    name: CO₂
    min: 400
    max: 2000
    needle: true
    severity:
      green: 400
      yellow: 800
      red: 1200

  - type: history-graph
    title: CO₂ за 12 часов
    hours_to_show: 12
    entities:
      - entity: sensor.living_room_co2
        name: CO₂
```

Пороговые значения здесь служат интерфейсной подсказкой, а не медицинским нормативом.

---

## 9. Полный штатный дашборд

```yaml
title: TION
views:
  - title: Воздух
    path: air
    icon: mdi:air-filter
    type: sections
    max_columns: 2
    sections:
      - type: grid
        cards:
          - type: heading
            heading: TION 4S
            icon: mdi:air-filter

          - type: tile
            entity: climate.tion_breezer
            name: Бризер
            features_position: bottom
            features:
              - type: climate-hvac-modes
                hvac_modes:
                  - fan_only
                  - heat
                  - "off"
              - type: climate-fan-modes
                style: dropdown
              - type: target-temperature

          - type: conditional
            conditions:
              - entity: climate.tion_breezer
                state: unavailable
            card:
              type: markdown
              content: |
                ## TION недоступен
                Проверьте Bluetooth и питание.

      - type: grid
        cards:
          - type: heading
            heading: Качество воздуха
            icon: mdi:molecule-co2

          - type: gauge
            entity: sensor.living_room_co2
            name: CO₂
            min: 400
            max: 2000
            needle: true
            severity:
              green: 400
              yellow: 800
              red: 1200

          - type: history-graph
            title: CO₂
            hours_to_show: 12
            entities:
              - sensor.living_room_co2
```

Sections layout доступен в современных версиях Home Assistant и удобен для адаптации под телефон, планшет и браузер.

---

## 10. Mushroom card через HACS

Для более компактного интерфейса можно использовать `Mushroom`.

Установка:

```text
HACS → Frontend → Mushroom
```

Пример:

```yaml
type: custom:mushroom-climate-card
entity: climate.tion_breezer
name: TION 4S
icon: mdi:air-filter
show_temperature_control: true
hvac_modes:
  - fan_only
  - heat
  - "off"
collapsible_controls: false
```

Дополнительные кнопки скоростей можно сделать через chips:

```yaml
type: custom:mushroom-chips-card
chips:
  - type: action
    icon: mdi:fan-speed-1
    tap_action:
      action: perform-action
      perform_action: climate.set_fan_mode
      target:
        entity_id: climate.tion_breezer
      data:
        fan_mode: "1"

  - type: action
    icon: mdi:fan-speed-2
    tap_action:
      action: perform-action
      perform_action: climate.set_fan_mode
      target:
        entity_id: climate.tion_breezer
      data:
        fan_mode: "3"

  - type: action
    icon: mdi:fan-speed-3
    tap_action:
      action: perform-action
      perform_action: climate.set_fan_mode
      target:
        entity_id: climate.tion_breezer
      data:
        fan_mode: "6"
```

Здесь три кнопки соответствуют условным уровням «тихо», «обычно» и «максимум», а не буквально скоростям 1, 2 и 3.

---

## 11. Карточка с быстрыми сценариями

Рекомендуется не отправлять несколько BLE-команд непосредственно из каждой кнопки. Лучше кнопкой запускать script.

```yaml
type: grid
columns: 3
square: false
cards:
  - type: button
    entity: script.tion_quiet
    name: Тихо
    icon: mdi:weather-night

  - type: button
    entity: script.tion_normal
    name: Обычно
    icon: mdi:fan

  - type: button
    entity: script.tion_boost_20_minutes
    name: Boost
    icon: mdi:fan-plus
```

Преимущество scripts:

- можно добавить задержки между командами;
- можно восстановить скорость после Boost;
- проще использовать один сценарий из нескольких мест;
- меньше риска конкурирующих BLE-команд.

---

## 12. Рекомендованная структура мобильного экрана

```text
[ TION: режим, температура, скорость ]
[ CO₂ ] [ Улица ] [ Окно ]
[ Тихо ] [ Обычно ] [ Boost ]
[ График CO₂ за 12 часов ]
[ Предупреждения — только при проблеме ]
```

Такой экран не перегружен, но даёт доступ ко всем повседневным действиям.

---

## 13. Что использовать

| Задача | Рекомендация |
|---|---|
| Самый простой интерфейс | Thermostat card |
| Современный штатный вариант | Tile card |
| Телефон | Tile + Grid |
| Планшет | Sections dashboard |
| Компактный дизайн | Mushroom |
| Сложные последовательности | Кнопки, запускающие scripts |
| Диагностика | Conditional + Markdown |

---

## 14. Проверка после добавления

1. Карточка показывает реальное состояние.
2. Режимы `fan_only`, `heat`, `off` доступны.
3. Скорости совпадают с `fan_modes` в атрибутах сущности.
4. Температура меняется без ошибки.
5. Кнопка Boost запускает script.
6. После каждого действия состояние обновляется.
7. Нет нескольких карточек, одновременно вызывающих конфликтующие scripts.

---

## Полезные ссылки

- [Dashboard cards Home Assistant](https://www.home-assistant.io/dashboards/cards/)
- [Tile card](https://www.home-assistant.io/dashboards/tile/)
- [Card features](https://www.home-assistant.io/dashboards/features/)
- [Mushroom](https://github.com/piitaya/lovelace-mushroom)
