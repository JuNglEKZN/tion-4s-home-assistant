# TION 4S + Home Assistant

Русскоязычная база знаний по интеграции **TION Бризер 4S** с Home Assistant.

Проект объединяет:

- пошаговую настройку через Bluetooth и `TionAPI/HA-tion`;
- практические рекомендации сообщества `esphome_tion` за 2022–2026 годы;
- диагностику проблем Bluetooth;
- сравнение прямого Bluetooth, ESPHome BLE и UART;
- основу для будущих автоматизаций и карточек интерфейса.

> Это независимая документация сообщества. Проект не связан с производителем TION.

## Быстрый старт

1. [Подключение TION 4S через Bluetooth](docs/installation-bluetooth.md)
2. [FAQ и лучшие практики сообщества](docs/faq-community.md)
3. [Диагностика неисправностей](docs/troubleshooting.md)
4. [Способы интеграции и выбор архитектуры](docs/connection-options.md)

## Рекомендуемая архитектура для первого запуска

```text
TION 4S
   │ Bluetooth Low Energy
   ▼
Home Assistant Bluetooth
   │
   ▼
TionAPI/HA-tion
```

Начинайте со встроенного Bluetooth Home Assistant. Если сигнал слабый или соединение нестабильно, рассмотрите ESPHome Bluetooth Proxy либо отдельный ESPHome-модуль рядом с бризером.

## Важное ограничение Bluetooth

TION использует активное BLE-соединение. Простого обнаружения рекламных пакетов недостаточно. Bluetooth-адаптер или прокси должен поддерживать активные подключения.

## Источники

Документация подготовлена на основе:

- [TionAPI/HA-tion](https://github.com/TionAPI/HA-tion);
- [dentra/esphome-tion](https://github.com/dentra/esphome-tion);
- [Bluetooth в Home Assistant](https://www.home-assistant.io/integrations/bluetooth/);
- экспортированной истории сообщества [@esphome_tion](https://t.me/esphome_tion) за 2022–2026 годы;
- практической систематизации повторяющихся решений.

Сообщения сообщества используются как источник практического опыта, а не как официальная документация. Советы, относящиеся только к ESPHome или UART, отмечены отдельно.

## Структура проекта

```text
docs/
├── installation-bluetooth.md
├── faq-community.md
├── troubleshooting.md
└── connection-options.md
```

## Статус

Первая версия базы знаний сосредоточена на Bluetooth-подключении. Разделы автоматизаций, Lovelace и подробного UART-подключения будут развиваться отдельно.
