*Read this document in English: [README_EN.md](README_EN.md)*

# Мурмулятор — RP2040 Black Clone, VGA-сборка на перфборде

Сборка Мурмулятора (переходной платы от Raspberry Pi Pico / клона YD-RP2040 к периферии) с VGA-выходом, PS/2-клавиатурой, двумя джойстик-портами (с поддержкой Bluetooth-геймпадов через BlueRetro), аудиовыходом, аудиовходом для загрузки программ, слотом MicroSD и внешним питанием. Плата собирается на перфборде, схема разведена в KiCad.

![Murmulator PCB](doc/murmulator_perfboard_blueretro.jpg)

## Разъёмы платы

| Разъём | Назначение |
|---|---|
| J1 | Внешнее питание (PSU), DC Jack → шина `+5V_RAW` |
| J2 | Отвод шины `+5V_RAW` — для внешней периферии (BlueRetro) |
| J3 | VGA (12-контактная гребёнка, Conn_02x06) |
| J4 | Клавиатура (PS/2, Mini-DIN-6) |
| J5 | Внешний разъём (EXT, 40 пинов); пары пинов с джамперами подключают PS/2, звук и аудиовход к GPIO |
| J6 | Джойстик 1 (DE-9) |
| J7 | Джойстик 2 (DE-9) |
| J8 | Аудиовыход |
| J9 | Аудиовход (загрузка программ с TAP/магнитофона) |
| J10 | Отвод `+3V3` (JST-XH) |
| J11 | Отвод узла `+5V_VOUT` (тот же узел, что и клавиатура J4) |
| J12 | Второй вход питания от USB (модуль Micro USB) вместо DC-джека → через D1 на шину `+5V_RAW` |
| J13 | Селектор (перемычка): `+5V_RAW` → Vin (штатно) или → Vout |
| J14 | Перемычка в обход D1 |
| J15 | Модуль MicroSD (SPI) |
| J16 | Дубль J2 на другой стороне платы |

## Документация

- [`doc/murmulator-vga-project-summary.md`](doc/murmulator-vga-project-summary.md) — полное описание сборки (RU): доработка SD-модуля, VGA, PS/2-клавиатура, джойстики и интеграция BlueRetro, аудиовыход и аудиовход, разъём J5, питание, чек-лист сборки. Английская версия: [`doc/murmulator-vga-project-summary_EN.md`](doc/murmulator-vga-project-summary_EN.md).
- [`doc/rp2040-module-powerchain-analysis.md`](doc/rp2040-module-powerchain-analysis.md) — сравнение цепей питания YD-RP2040 (Black Clone) и официального Raspberry Pi Pico. Английская версия: [`doc/rp2040-module-powerchain-analysis_EN.md`](doc/rp2040-module-powerchain-analysis_EN.md).
- [`doc/Firmware/technocat-murmulator-keyboard-mapping.md`](doc/Firmware/technocat-murmulator-keyboard-mapping.md) — маппинг PS/2-клавиатуры на ZX Spectrum в прошивке Tecnocat v0.96.20, разбор режима «Cursor joystick». Английская версия: [`doc/Firmware/technocat-murmulator-keyboard-mapping_EN.md`](doc/Firmware/technocat-murmulator-keyboard-mapping_EN.md).
- [`doc/murmulator-ps2-modifiers-oscillograms.pdf`](doc/murmulator-ps2-modifiers-oscillograms.pdf) — осциллограммы PS/2-сигналов CLK/DATA для клавиш-модификаторов ([DOCX](doc/murmulator-ps2-modifiers-oscillograms.docx)). Английская версия: [PDF](doc/murmulator-ps2-modifiers-oscillograms_EN.pdf), [DOCX](doc/murmulator-ps2-modifiers-oscillograms_EN.docx).
- [`doc/murmulator_perfboard_front.jpg`](doc/murmulator_perfboard_front.jpg) — фото платы на перфборде (front).
- [`doc/murmulator_perfboard_back.jpg`](doc/murmulator_perfboard_back.jpg) — фото платы на перфборде (back).

## Схемы (KiCad)

- [`schematics/rp2040_black_vga/`](schematics/rp2040_black_vga/) — исходные файлы проекта KiCad.
- [`schematics/rp2040_black_vga/plot/rp2040_black_vga.pdf`](schematics/rp2040_black_vga/plot/rp2040_black_vga.pdf) — принципиальная схема в PDF.
- [`schematics/rp2040_black_vga/plot/rp2040_black_vga.net`](schematics/rp2040_black_vga/plot/rp2040_black_vga.net) — нетлист.

## Справочные материалы

- [`doc/YD-2040-2022-V1.1-Schematics_black_clone.pdf`](doc/YD-2040-2022-V1.1-Schematics_black_clone.pdf) — схема модуля YD-RP2040 (Black Clone).
- [`doc/YD-2040-2022-V1.1 - Powerchain.jpg`](<doc/YD-2040-2022-V1.1 - Powerchain.jpg>), [`doc/RP2040 Official Pico Module - Powerchain.jpg`](<doc/RP2040 Official Pico Module - Powerchain.jpg>) — фрагменты схем питания клона и официального Pico.
- [`doc/original_classic_murmulator_37NJU22_emul_RPPICO_card.pdf`](doc/original_classic_murmulator_37NJU22_emul_RPPICO_card.pdf) — оригинальная классическая схема Мурмулятора.
- [`doc/FT SD Module modification.jpg`](<doc/FT SD Module modification.jpg>) — доработка SD-модуля.

## Ключевые особенности сборки

- **SD-модуль** доработан: выпаяны AMS1117 и буфер 74VHCT125A, установлены перемычки — модуль работает как чистый 3.3V breakout без потери фронтов на высокой скорости SPI.
- **VGA** — пассивная резисторная R2R-лесенка на GPIO, без отдельного питания.
- **PS/2-клавиатура** — питание от Vout (после диода BAT54C), сигнальные линии CLK/DATA согласованы по уровню резистивно-зенерной схемой (5V → 3.3V).
- **Джойстики (Dendy, DE-9)** — сдвиговый регистр запитан от 3.3V, что снимает необходимость в защитных резисторах на DATA.
- **BlueRetro** — DIY-адаптер на ESP32 эмулирует оба джойстик-порта по Bluetooth HID (PS3/4/5, Xbox, Wii/Switch и др.), протокол CLOCK/LATCH/DATA идентичен физическому Dendy-джойстику. Требует питания на J1 или J12. Сборка адаптера на 3.3V-логике: https://github.com/alex-snigir/BlueRetro-3.3V-logic
- **Аудиовыход** — стереоканал (L/R) плюс подмешивание сигнала пищалки (BEEP_OUT).
- **Аудиовход** — двухкаскадный формирователь на транзисторах для загрузки программ с TAP-сигнала (магнитофон, телефон, онлайн-плеер TAP).
- **Питание** — шина `+5V_RAW` от DC-джека J1 или от Micro USB J12 (через диод Шоттки D1), селектор J13 подаёт её на Vin модуля (штатно, через встроенный BAT54C) или напрямую на Vout в обход маломощного BAT54C.

Подробности по каждому пункту — в [`doc/murmulator-vga-project-summary.md`](doc/murmulator-vga-project-summary.md).

## Корпус

![Murmulator PCB](doc/murmulator-vga-box1.jpg)
![Murmulator PCB](doc/murmulator-vga-box2.jpg)

### 3D-модели для печати

Файлы лежат в папке [`3d models/`](<3d models/>):

| Деталь | Файлы |
|---|---|
| Корпус (дно + крышка) | [`Box_Bottom_v2.STL`](<3d models/Box_Bottom_v2.STL>) + [`Box_Cover_v2.STL`](<3d models/Box_Cover_v2.STL>) |
| Только плоское дно, без крышки | [`Box_Bottom.STL`](<3d models/Box_Bottom.STL>) |
| Крепление для двух разъёмов DB9 (джойстики J6/J7) | [`DB9_x2_mount.STL`](<3d models/DB9_x2_mount.STL>) |
| Опора для SD-модуля | [`SD_module_support.STL`](<3d models/SD_module_support.STL>) |

Рядом лежат исходники SolidWorks (`.SLDPRT`) и проекты слайсера (`.3mf`) с теми же именами.


## Ссылки

- Официальная документация и другие варианты сборки Мурмулятора: https://murmulator.ru/howto
- Классическая схема (референс): https://github.com/AlexEkb4ever/MURMULATOR_classical_scheme
- Базовая версия «Мурмулятор на макетной плате 7×9 с VGA»: https://murmulator.ru/mm-maket
- BlueRetro-адаптер на 3.3V-логике (интеграция джойстик-портов): https://github.com/alex-snigir/BlueRetro-3.3V-logic
- Прошивка Tecnocat (Murmulator_rp2040): https://github.com/MadedCat/Murmulator_rp2040
