# Автоматизации TION 4S в Home Assistant

В этом разделе собраны безопасные шаблоны автоматизаций для интеграции `TionAPI/HA-tion`.

`HA-tion` создаёт климатическую сущность TION. Она поддерживает:

- `climate.turn_on` и `climate.turn_off`;
- `climate.set_hvac_mode` со значениями `off`, `fan_only`, `heat`;
- `climate.set_fan_mode` со скоростями `0`–`6`;
- `climate.set_temperature`;
- пресеты, в том числе `boost` и `away`, если они доступны конкретной модели и версии интеграции.

> TION — приточная вентиляция с подогревом воздуха, а не отопительный прибор. Не используйте бризер как основное отопление помещения.

Готовый общий YAML находится в [`examples/automations/tion-automations.yaml`](../examples/automations/tion-automations.yaml).

---

## 1. Перед использованием замените сущности

В примерах используются условные идентификаторы:

```yaml
climate.tion_breezer
sensor.living_room_co2
sensor.outdoor_temperature
binary_sensor.living_room_window
person.owner
notify.mobile_app_phone
```

Узнать свои entity ID можно здесь:

```text
Настройки → Устройства и службы → Сущности
```

Или:

```text
Инструменты разработчика → Состояния
```

---

## 2. Управление скоростью по CO₂

Главная задача — не переключать скорость при каждом колебании датчика. Поэтому используются:

- выдержка времени `for`;
- разные пороги повышения и снижения;
- режим автоматизации `restart`;
- одна команда за запуск.

Рекомендуемая логика:

| CO₂ | Скорость |
|---:|---:|
| ниже 650 ppm | 1 |
| 650–850 ppm | 2 |
| 850–1100 ppm | 3 |
| 1100–1400 ppm | 4 |
| выше 1400 ppm | 5 |

Ночью максимальную скорость разумно ограничить, например, третьей.

### Вариант с одной автоматизацией

```yaml
alias: TION — управление по CO₂
id: tion_co2_control
mode: restart

triggers:
  - trigger: state
    entity_id: sensor.living_room_co2
    for: "00:03:00"

conditions:
  - condition: template
    value_template: >-
      {{ states('sensor.living_room_co2') not in ['unknown', 'unavailable', 'none'] }}
  - condition: state
    entity_id: binary_sensor.living_room_window
    state: "off"

actions:
  - variables:
      co2: "{{ states('sensor.living_room_co2') | int(0) }}"
      night: "{{ now().hour >= 23 or now().hour < 7 }}"
      requested_speed: >-
        {% if co2 < 650 %} 1
        {% elif co2 < 850 %} 2
        {% elif co2 < 1100 %} 3
        {% elif co2 < 1400 %} 4
        {% else %} 5
        {% endif %}
      final_speed: >-
        {% if night %}
          {{ [requested_speed | int, 3] | min }}
        {% else %}
          {{ requested_speed | int }}
        {% endif %}

  - if:
      - condition: state
        entity_id: climate.tion_breezer
        state: "off"
    then:
      - action: climate.turn_on
        target:
          entity_id: climate.tion_breezer
      - delay: "00:00:03"

  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "{{ final_speed | string }}"
```

### Почему задержка после включения важна

Bluetooth-соединение и сам бризер могут не успеть принять вторую команду сразу после `turn_on`. Пауза в 2–5 секунд уменьшает вероятность потерянной команды.

---

## 3. CO₂ с гистерезисом

Для максимально спокойной работы лучше не рассчитывать скорость при каждом обновлении датчика, а использовать отдельные пороги.

Пример:

- перейти на скорость 4 при CO₂ выше 1100 ppm в течение 5 минут;
- вернуться на скорость 3 только после снижения ниже 950 ppm в течение 10 минут.

Так система не будет прыгать между скоростями около одного порога.

```yaml
alias: TION — скорость 4 при высоком CO₂
id: tion_co2_high_speed_4
mode: single
triggers:
  - trigger: numeric_state
    entity_id: sensor.living_room_co2
    above: 1100
    for: "00:05:00"
conditions:
  - condition: not
    conditions:
      - condition: state
        entity_id: climate.tion_breezer
        state: "off"
  - condition: time
    after: "07:00:00"
    before: "23:00:00"
actions:
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "4"
```

```yaml
alias: TION — вернуть скорость 3 после снижения CO₂
id: tion_co2_return_speed_3
mode: single
triggers:
  - trigger: numeric_state
    entity_id: sensor.living_room_co2
    below: 950
    for: "00:10:00"
conditions:
  - condition: not
    conditions:
      - condition: state
        entity_id: climate.tion_breezer
        state: "off"
actions:
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "3"
```

---

## 4. Ночной режим

Вариант без полного выключения вентиляции:

```yaml
alias: TION — ночной режим
id: tion_night_mode
mode: restart
triggers:
  - trigger: time
    at: "23:00:00"
actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.tion_breezer
    data:
      hvac_mode: fan_only
  - delay: "00:00:03"
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "2"
```

Утреннее восстановление:

```yaml
alias: TION — дневной режим
id: tion_day_mode
mode: restart
triggers:
  - trigger: time
    at: "07:00:00"
actions:
  - action: climate.turn_on
    target:
      entity_id: climate.tion_breezer
  - delay: "00:00:03"
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "3"
```

---

## 5. Управление подогревом по наружной температуре

Не включайте нагреватель летом без необходимости.

Пример логики:

- ниже `+5 °C` — режим `heat`, цель `18 °C`;
- выше `+8 °C` — режим `fan_only`;
- разница между порогами создаёт гистерезис.

```yaml
alias: TION — включить подогрев при холоде
id: tion_heater_enable_cold
mode: restart
triggers:
  - trigger: numeric_state
    entity_id: sensor.outdoor_temperature
    below: 5
    for: "00:10:00"
conditions:
  - condition: not
    conditions:
      - condition: state
        entity_id: climate.tion_breezer
        state: "off"
actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.tion_breezer
    data:
      temperature: 18
  - delay: "00:00:03"
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.tion_breezer
    data:
      hvac_mode: heat
```

```yaml
alias: TION — выключить подогрев при потеплении
id: tion_heater_disable_warm
mode: restart
triggers:
  - trigger: numeric_state
    entity_id: sensor.outdoor_temperature
    above: 8
    for: "00:15:00"
conditions:
  - condition: not
    conditions:
      - condition: state
        entity_id: climate.tion_breezer
        state: "off"
actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.tion_breezer
    data:
      hvac_mode: fan_only
```

---

## 6. Открытое окно

Одновременно держать окно открытым и подавать воздух через TION обычно неэффективно.

```yaml
alias: TION — пауза при открытом окне
id: tion_pause_window_open
mode: restart
triggers:
  - trigger: state
    entity_id: binary_sensor.living_room_window
    to: "on"
    for: "00:01:00"
actions:
  - action: climate.turn_off
    target:
      entity_id: climate.tion_breezer
```

Возобновление после закрытия:

```yaml
alias: TION — включить после закрытия окна
id: tion_resume_window_closed
mode: restart
triggers:
  - trigger: state
    entity_id: binary_sensor.living_room_window
    to: "off"
    for: "00:02:00"
conditions:
  - condition: state
    entity_id: person.owner
    state: home
actions:
  - action: climate.turn_on
    target:
      entity_id: climate.tion_breezer
  - delay: "00:00:03"
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "2"
```

Если важно восстановить предыдущую скорость, используйте helper `input_number` или отдельный script, сохраняющий состояние перед выключением.

---

## 7. Режим «Никого нет дома»

Полное выключение экономит фильтр и электроэнергию, но может привести к росту влажности и запахов. Поэтому часто лучше использовать скорость 1.

```yaml
alias: TION — режим отсутствия
id: tion_away_mode
mode: restart
triggers:
  - trigger: state
    entity_id: zone.home
    attribute: persons
    to: 0
    for: "00:15:00"
actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.tion_breezer
    data:
      hvac_mode: fan_only
  - delay: "00:00:03"
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "1"
```

Возвращение домой:

```yaml
alias: TION — возвращение домой
id: tion_home_mode
mode: restart
triggers:
  - trigger: numeric_state
    entity_id: zone.home
    attribute: persons
    above: 0
conditions:
  - condition: state
    entity_id: binary_sensor.living_room_window
    state: "off"
actions:
  - action: climate.turn_on
    target:
      entity_id: climate.tion_breezer
  - delay: "00:00:03"
  - action: climate.set_fan_mode
    target:
      entity_id: climate.tion_breezer
    data:
      fan_mode: "3"
```

---

## 8. Boost по кнопке или сценарию

Лучше реализовать через script — его можно вызвать из Lovelace, голосового ассистента или другой автоматизации.

```yaml
script:
  tion_boost_20_minutes:
    alias: TION — интенсивное проветривание 20 минут
    mode: restart
    sequence:
      - action: climate.turn_on
        target:
          entity_id: climate.tion_breezer
      - delay: "00:00:03"
      - action: climate.set_hvac_mode
        target:
          entity_id: climate.tion_breezer
        data:
          hvac_mode: fan_only
      - delay: "00:00:03"
      - action: climate.set_fan_mode
        target:
          entity_id: climate.tion_breezer
        data:
          fan_mode: "6"
      - delay: "00:20:00"
      - action: climate.set_fan_mode
        target:
          entity_id: climate.tion_breezer
        data:
          fan_mode: "2"
```

Если интеграция корректно предоставляет preset `boost`, вместо скорости 6 можно использовать `climate.set_preset_mode`.

---

## 9. Контроль недоступности

```yaml
alias: TION — уведомить о недоступности
id: tion_unavailable_notification
mode: single
triggers:
  - trigger: state
    entity_id: climate.tion_breezer
    to: unavailable
    for: "00:10:00"
actions:
  - action: notify.mobile_app_phone
    data:
      title: TION недоступен
      message: >-
        Бризер не отвечает более 10 минут. Проверьте Bluetooth,
        TION Remote, питание и RSSI.
```

Уведомление о восстановлении:

```yaml
alias: TION — уведомить о восстановлении
id: tion_recovered_notification
mode: single
triggers:
  - trigger: state
    entity_id: climate.tion_breezer
    from: unavailable
conditions:
  - condition: template
    value_template: "{{ trigger.to_state.state not in ['unknown', 'unavailable'] }}"
actions:
  - action: notify.mobile_app_phone
    data:
      title: TION снова доступен
      message: Соединение с бризером восстановлено.
```

---

## 10. Защита от слишком частых команд

Для BLE рекомендуется:

- не отправлять несколько команд без паузы;
- использовать `mode: restart` или `mode: queued` осознанно;
- не создавать несколько автоматизаций, одновременно меняющих скорость;
- объединять связанную логику в одну автоматизацию с `choose`;
- использовать гистерезис;
- добавлять `for` к шумным датчикам.

### Рекомендуемые режимы

| Сценарий | Режим |
|---|---|
| Реакция на CO₂ | `restart` |
| Последовательность команд | `queued` |
| Уведомление о недоступности | `single` |
| Boost с таймером | `restart` |

---

## 11. Проверка перед включением автоматизаций

1. Все сущности существуют.
2. Ручное управление TION стабильно.
3. Скорости `1`–`6` принимаются интеграцией.
4. У датчика CO₂ корректные единицы `ppm`.
5. Датчик окна использует `on` при открытом состоянии.
6. Уведомления работают.
7. Автоматизации не дублируют друг друга.
8. После включения проверьте `Трассировки` каждой автоматизации.

---

## Полезные ссылки

- [Официальный пример автоматизации HA-tion](https://github.com/TionAPI/HA-tion#automation-example)
- [Автоматизации Home Assistant в YAML](https://www.home-assistant.io/docs/automation/yaml/)
- [Условия автоматизаций](https://www.home-assistant.io/docs/automation/condition/)
- [Режимы выполнения автоматизаций](https://www.home-assistant.io/docs/automation/modes/)
