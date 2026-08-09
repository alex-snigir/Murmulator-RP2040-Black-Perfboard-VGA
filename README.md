*Read this document in English: [README_EN.md](README_EN.md)*

# Мурмулятор — RP2040 Black Clone, VGA-сборка на перфборде

Сборка Мурмулятора (переходной платы от Raspberry Pi Pico / клона YD-RP2040 к периферии) с VGA-выходом, PS/2-клавиатурой, двумя джойстик-портами (с поддержкой Bluetooth-геймпадов через BlueRetro), аудиовыходом и внешним питанием. Плата собирается на перфборде, схема разведена в KiCad.

## Разъёмы платы

| Разъём | Назначение |
|---|---|
| J1 | Внешнее питание (PSU), DC Jack |
| J2 | Отвод «сырого» +5V от PSU (до D1) — для внешней периферии (BlueRetro) |
| J3 | VGA (12-контактная гребёнка, Conn_02x06) |
| J4 | Клавиатура (PS/2, Mini-DIN-6) |
| J5 | Внешний разъём |
| J6 | Джойстик 1 (DE-9) |
| J7 | Джойстик 2 (DE-9) |
| J8 | Аудиовыход |

## Документация

- [`doc/murmulator-vga-project-summary.md`](doc/murmulator-vga-project-summary.md) — полное описание сборки (RU): доработка SD-модуля, VGA, PS/2-клавиатура, джойстики и интеграция BlueRetro, аудио, питание, чек-лист сборки.
- [`doc/murmulator-vga-project-summary_EN.md`](doc/murmulator-vga-project-summary_EN.md) — тот же документ на английском языке.

## Схемы (KiCad)

- [`schematics/rp2040_black_vga/`](schematics/rp2040_black_vga/) — исходные файлы проекта KiCad.
- [`schematics/rp2040_black_vga/schematics_rp2040_black_vga.pdf`](schematics/rp2040_black_vga/schematics_rp2040_black_vga.pdf) — принципиальная схема в PDF.

## Ключевые особенности сборки

- **SD-модуль** доработан: выпаяны AMS1117 и буфер 74VHCT125A, установлены перемычки — модуль работает как чистый 3.3V breakout без потери фронтов на высокой скорости SPI.
- **VGA** — пассивная резисторная R2R-лесенка на GPIO, без отдельного питания.
- **PS/2-клавиатура** — питание от Vout (после диода BAT54C), сигнальные линии CLK/DATA согласованы по уровню резистивно-зенерной схемой (5V → 3.3V).
- **Джойстики (Dendy, DE-9)** — сдвиговый регистр запитан от 3.3V, что снимает необходимость в защитных резисторах на DATA.
- **BlueRetro** — DIY-адаптер на ESP32 эмулирует оба джойстик-порта по Bluetooth HID (PS/4/5, Xbox, Wii/Switch и др.), протокол CLOCK/LATCH/DATA идентичен физическому Dendy-джойстику. Требует подключённого внешнего PSU.
- **Аудиовыход** — стереоканал (L/R) плюс подмешивание сигнала пищалки (BEEP_OUT).
- **Питание** — диод Шоттки D1 (1N5819) резервирует путь от PSU к узлу +5V_VOUT в обход маломощного встроенного BAT54C.

Подробности по каждому пункту — в [`doc/murmulator-vga-project-summary.md`](doc/murmulator-vga-project-summary.md).

## Ссылки

- Официальная документация и другие варианты сборки Мурмулятора: https://murmulator.ru/howto
- Классическая схема (референс): https://github.com/AlexEkb4ever/MURMULATOR_classical_scheme
- Базовая версия «Мурмулятор на макетной плате 7×9 с VGA»: https://murmulator.ru/mm-maket
